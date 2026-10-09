# Harness fails with "You must provide a ResourceOauth2ReturnUrl" although `defaultReturnUrl` is set (harness v7)

## Summary
A harness with an `agentcore_gateway` tool using `outboundAuth.oauth` (`grantType: AUTHORIZATION_CODE`,
`defaultReturnUrl: http://localhost:8080`) fails at tool load. `GetResourceOauth2Token` is rejected with
`ValidationException: You must provide a ResourceOauth2ReturnUrl to proceed with this flow`, even though the
deployed harness config contains `defaultReturnUrl` and the workload identity allow-lists the same URL.

## Environment
- `@aws/agentcore` CLI 0.31.1 (latest on npm at time of testing)
- AWS CLI 2.37.4, Windows 11
- Region: ap-southeast-2
- Date: 2026-10-04

## Setup
- Inbound: harness `authorizerType: CUSTOM_JWT` (Cognito user pool, app client allow-listed).
- Tool: `agentcore_gateway` pointing at a Gateway with `CUSTOM_JWT` authorizer and a Cedar policy engine.
- Outbound auth on the tool (confirmed via `aws bedrock-agentcore-control get-harness`, harnessVersion 7, runtime
  endpoint DEFAULT liveVersion 7):

```json
"outboundAuth": {
  "oauth": {
    "providerArn": "arn:aws:bedrock-agentcore:ap-southeast-2:<account>:token-vault/default/oauth2credentialprovider/AFLUserOAuth",
    "scopes": ["openid", "email", "phone"],
    "grantType": "AUTHORIZATION_CODE",
    "defaultReturnUrl": "http://localhost:8080"
  }
}
```

- Workload identity updated to allow the URL:

```
aws bedrock-agentcore-control update-workload-identity \
  --name harness_AFLStartingTeamAgent_teamHarness-<id> \
  --allowed-resource-oauth2-return-urls http://localhost:8080 --region ap-southeast-2
```
  Response lists `allowedResourceOauth2ReturnUrls: ["http://localhost:8080"]`.

## Steps to reproduce
1. Deploy the harness above (v7 live).
2. `agentcore invoke --harness teamHarness --bearer-token "<valid Cognito access token>" --prompt "..." --verbose`

## Actual result
```
{"type":"error","errorType":"runtimeClientError","message":"Failed to load tool 'AFLGatewayJWT'
(type=HarnessToolType.agentcore_gateway): Failed to start MCP client: the client initialization failed:
unhandled errors in a TaskGroup (1 sub-exception): An error occurred (ValidationException) when calling the
GetResourceOauth2Token operation: You must provide a ResourceOauth2ReturnUrl to proceed with this flow"}
```
Reproduced on 4 invokes after v7 went live (including a fresh session each time).

## Expected result
The harness passes `defaultReturnUrl` as `resourceOauth2ReturnUrl` to `GetResourceOauth2Token` and returns the
OAuth authorization URL for user consent (3-legged flow).

## Notes
- Config matches the `update-harness` API docs for `defaultReturnUrl`.
- Unclear whether the harness runtime ignores `defaultReturnUrl` or the CLI/CDK deploy path drops it; the
  deployed `get-harness` output does contain it.
