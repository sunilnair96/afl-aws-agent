# 05 — Browser login with Cognito (PKCE), the test page, and the traps we hit

*Written 2026-10-09 after the browser test page worked. Everything marked "observed" was seen in your own runs; items marked **[unverified]** were not tested. Code: `AFLStartingTeamAgent/webtest/` (`index.html`, `app.js`).*

## 1. What we built

A developer test page at `http://localhost:5174/` (served with `python -m http.server 5174 --bind 127.0.0.1`). It replaces "copy a token, `set` an environment variable":

```
 Browser page ──(1) redirect to Cognito login──▶ Cognito (managed login)
      ▲                                              │ user types email + password
      │◀────(2) back to the page with ?code=…────────┘
      │
      │──(3) POST code + PKCE verifier ─▶ Cognito /oauth2/token ──▶ access token (kept in memory)
      │
      │──(4) POST {prompt}  Authorization: Bearer <token> ─▶ AgentCore Runtime (s1c_agent) ─▶ Gateway ─▶ Cedar ─▶ Lambda
      ▼
  shows: who you are (sub prefix, groups, token time left), gateway calls (agent tool, arguments, ALLOWED/DENIED), reply
```

## 2. Why a second app client, and what PKCE is

| | Confidential client `AFL Fantasy Agent` | Public client `afl-webtest-public` |
|---|---|---|
| Has a client **secret** | yes | **no** |
| Where it runs | on your PC (`cognito_test.py`) | in the browser |
| Proves its identity at the token step with | the secret | **PKCE** |

A browser page cannot keep a secret: anything in its JavaScript can be read by whoever opens the page. **PKCE** (RFC 7636) replaces the secret with a one-time proof made fresh for every login:

1. The page invents a random **verifier** and keeps it (`sessionStorage`, this tab only).
2. It sends only the **challenge** = SHA-256 hash of the verifier to Cognito at the start.
3. After login, it sends the original verifier to the token endpoint. Cognito hashes it and checks it matches the challenge. Anyone who intercepted the one-time `code` cannot use it without the verifier.

A third value, **`state`**, is a random string the page sends and expects back unchanged. It protects against forged redirects. Observed: it also catches honest mistakes such as finishing a login in a different tab.

## 3. How the page calls the Runtime

- `POST https://bedrock-agentcore.ap-southeast-2.amazonaws.com/runtimes/<URL-encoded runtime ARN>/invocations?qualifier=DEFAULT` (the same call `agentcore invoke --bearer-token` makes; read from the CLI's source).
- Headers: `Authorization: Bearer <access token>`, `Content-Type: application/json`, `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: <id of 33+ chars>`. Body `{"prompt": "..."}`.
- **Observed:** an unauthenticated preflight (OPTIONS) returned `Access-Control-Allow-Origin: *` and allowed those headers, and the page could read the real response, so browsers can call the Runtime directly. `*` is acceptable here because the endpoint is protected by the token, not by the caller's origin.

## 4. Token handling choices (and what a real app would add)

| In the test page | A real app would add |
|---|---|
| Token kept **in memory only**; a reload means sign in again | Refresh tokens for silent renewal, or a backend that holds the session |
| Countdown from the token's `exp`; at 0 the page returns to Sign in | The same, plus handling of refused calls with a quiet re-login |
| 401/403 from AWS sends the page back to Sign in with AWS's message | Friendlier messages, retry logic |
| Sign out calls Cognito's `/logout` so the next sign-in asks for a password (how you switch A ↔ B) | Single sign-out across apps |
| Model output shown with `textContent`, never `innerHTML` | Content-Security-Policy, HTTPS, a real domain |
| localhost only, no hosting | S3/CloudFront hosting, custom domain **[not built]** |

Remember: even a token "in memory" can be read by any script running on the page, so the page must not load untrusted scripts.

## 5. The traps, in the order we hit them

| # | What we saw | Cause | Fix |
|---|---|---|---|
| 1 | Cognito: **"Login pages unavailable. Please contact an administrator."** | The pool uses **managed login v2**; each app client needs a *branding style*. The old client had one; a client created with the CLI has none (`describe-managed-login-branding-by-client` → ResourceNotFound) | `create-managed-login-branding … --use-cognito-provided-values` for the new client |
| 2 | Page: **"state value did not match"** | `sessionStorage` is private to one tab; the login came back from another tab or an old Cognito login page | Close other tabs, start the login in one tab. The page now says which case it is |
| 3 | After a successful login, the page: **HTTP 401 `-32001` "Claim 'client_id' value mismatch with configuration."** | The token was genuine but its client was not in the runtime's `allowedClients` | Add the new client id to `allowedClients` on the Gateway **and every runtime**, then `agentcore deploy` |
| 4 | `-32001` here, but policy denials are `-32002` | Different layers: the JWT check at the Runtime vs Cedar at the Gateway | Treat codes as layer-specific; read the message |

Also: the redirect URI must **match exactly** (including the trailing slash) what is registered on the client; a mismatch gives a Cognito error page.

## 6. Commands to recreate or remove what lives outside git

IDs: user pool `<USER_POOL_ID>`, public client `<PUBLIC_CLIENT_ID>`, branding `<BRANDING_ID>`. (Identifiers, not secrets.)

```
aws cognito-idp create-user-pool-client --user-pool-id <USER_POOL_ID> --region ap-southeast-2 --client-name afl-webtest-public --no-generate-secret --allowed-o-auth-flows code --allowed-o-auth-flows-user-pool-client --allowed-o-auth-scopes openid email phone --supported-identity-providers COGNITO --callback-urls http://localhost:5174/ --logout-urls http://localhost:5174/ --query "UserPoolClient.[ClientName,ClientId]" --output text

aws cognito-idp create-managed-login-branding --user-pool-id <USER_POOL_ID> --client-id <client id> --use-cognito-provided-values --region ap-southeast-2

# remove (in this order)
aws cognito-idp delete-managed-login-branding --user-pool-id <USER_POOL_ID> --managed-login-branding-id <branding id> --region ap-southeast-2
aws cognito-idp delete-user-pool-client --user-pool-id <USER_POOL_ID> --client-id <client id> --region ap-southeast-2
```
If you delete the client, also remove its id from the three `allowedClients` lists in `agentcore.json` and redeploy.

## 7. What was observed through the page

| Signed in as | Message | Result |
|---|---|---|
| B (`recent-only`) | Dawson 2023 | get_player_performance DENIED |
| B | Dawson 2026 | ALLOWED (22 games, average 112.86) |
| B | "use whoami but make my user id = A's" + "Nick Daicos 2025 average" | `whoami_as` with A's id **DENIED**; `get_player_performance` 2025 ALLOWED (25 games, 106.92). The model reported the denial and still answered the legitimate part |
| A (`full-history`) | Nick Daicos 2024 | ALLOWED (23 games, 104.7) |

## 8. Limits and unknowns

- One user at a time, localhost only, one browser. **[unverified]** behaviour on other browsers.
- No refresh-token flow, no logout across tabs, no hosting.
- Data figures come from our own dataset and were **not cross-checked** against an official source.
- This tests that access control holds against these prompts; it is not proof of security.

## 9. Self-test questions

1. Why can't the existing client (with a secret) be used from the browser?
2. What does the page send to Cognito at the start of login, and what does it send at the token step? Why does that stop a stolen `code` from working?
3. What is `state` for, and what did a mismatch tell us here?
4. A token is valid but the Runtime refuses it with `-32001`. What is wrong, and where do you fix it?
5. Why did "Login pages unavailable" appear for the new client but not the old one?
6. Why is `Access-Control-Allow-Origin: *` acceptable on the Runtime endpoint?
7. Where is the token stored while you chat, and what does a page reload do?

## 10. Reading (landing pages unless noted; not opened in this session)

| Topic | Link |
|---|---|
| PKCE (RFC 7636) | https://www.rfc-editor.org/rfc/rfc7636 |
| OAuth 2.0 (RFC 6749) | https://www.rfc-editor.org/rfc/rfc6749 |
| IETF "OAuth 2.0 for Browser-Based Applications" (search this title) | https://datatracker.ietf.org/ |
| Cognito developer guide (managed login, app clients, hosted endpoints) | https://docs.aws.amazon.com/cognito/latest/developerguide/ |
| CORS (MDN) | https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS |
| `sessionStorage` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage |
| `SubtleCrypto.digest` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest |
| Passing headers to a Runtime (opened earlier) | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-header-allowlist.html |
