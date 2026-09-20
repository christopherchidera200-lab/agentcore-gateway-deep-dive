AgentCore Gateway Deep Dive — Pizza Shop MCP Gateway
Terraform-built Amazon Bedrock AgentCore Gateway that exposes a Pizza Shop's tools (menu, orders, promotions) as an MCP endpoint, layered progressively with authentication, Cedar policy authorization, request/response interception, and outbound API-key identity for a third-party backend.
Based on the AWS Workshop Studio lab: AgentCore Gateway Deep Dive.
Architecture
```
                         ┌─────────────────────────┐
  MCP client / agent ──▶ │   AgentCore Gateway      │
  (Bearer JWT)           │   (protocol: MCP)        │
                         └───────────┬─────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                 ▼
             get-menu Lambda  create-order Lambda   promotions target
             (no external      (no external          (OpenAPI schema,
              creds needed)     creds needed)          outbound API key
                                                        → API Gateway →
                                                        promotions Lambda)

  Cognito User Pool ── issues JWTs, scoped per client (get_menu / create_order)
  Cedar Policy Engine ── authorizes each tool call against principal scope + input
  Interceptor Lambda ── runs on REQUEST/RESPONSE, can inspect/transform traffic
```
Environment
WSL2 (Ubuntu) on Windows 11 — not native PowerShell
Terraform 1.16.2, AWS CLI 2.34.45, `jq`, `curl`
AWS account: sandbox, region `us-east-1`
IAM user with AdministratorAccess (workshop convenience — scope this down for anything beyond a throwaway sandbox)
What this builds, module by module
M0–M1 — Bootstrap & Gateway concepts
`bootstrap.tf`, `providers.tf`. Standard Terraform init against `aws` (6.47.0) and `awscc` (1.85.0) providers — `awscc` is required because AgentCore Gateway/Policy resources aren't in the classic `aws` provider yet.
M2 — First tool, no auth
`gateway.tf`, `lambda-get-menu.tf`, `lambda-create-order.tf`. Gateway created with `authorizer_type = "NONE"`. Tools registered as Lambda-backed MCP targets with inline JSON-schema tool definitions. Callable directly with `curl` and an empty `Authorization: Bearer` header — useful for seeing the raw MCP `tools/list` / `tools/call` JSON-RPC shape before anything else gets involved.
Gotcha: `list-tools` / `get-menu` fail with `jq: not found` on a fresh WSL box — `sudo apt install -y jq` fixes it. Not an AgentCore issue, just a missing local dependency.
M3 — JWT auth via Cognito
`cognito-module3.tf`. Gateway redeployed with `authorizer_type = "CUSTOM_JWT"`, pointed at a Cognito User Pool's discovery URL, scoped to `gateway/invoke`. Calling without a token now returns `-32001 Invalid Bearer token`; `make get-token` fetches a client-credentials token and `list-tools` succeeds with it attached.
Note on `redeploy-gateway`: replacing the `awscc_bedrockagentcore_gateway` resource gets a new gateway ID and URL every time (it's not an in-place update for auth changes). Each redeploy in this lab required grabbing the new URL from `make list-tools` output before the next call would work.
M4 — Fine-grained scopes + Cedar policies
`cognito-module4.tf`, `gateway-policies.tf`. Two additional Cognito clients created — `client1` (scope: `gateway/get_menu` only) and `client2` (scopes: `gateway/get_menu` + `gateway/create_order`) — to demonstrate scope-based authorization.
Once a Policy Engine is attached to the gateway, all tool calls are denied by default until a Cedar policy explicitly permits them — this is a real trap: enabling the policy engine with zero policies attached silently breaks every tool call with `Tool Execution Denied: ... denied by default`, even for a client that has the right Cognito scope. The fix is either a broad `permit_all` policy or (better, as done here) per-tool policies:
```
permit(
  principal,
  action == AgentCore::Action::"get-menu___get-menu",
  resource == AgentCore::Gateway::"<gateway-arn>"
);

permit(
  principal,
  action == AgentCore::Action::"create-order___create-order",
  resource == AgentCore::Gateway::"<gateway-arn>"
)
when {
  principal.hasTag("scope") &&
  principal.getTag("scope") like "*gateway/create_order*"
};

forbid(
  principal,
  action == AgentCore::Action::"create-order___create-order",
  resource == AgentCore::Gateway::"<gateway-arn>"
)
when {
  context.input.pizzaId == 5
};
```
That last policy blocks ordering pizza ID 5 (Pineapple) regardless of scope — a concrete example of Cedar authorizing on tool input, not just identity, which is the interesting part of this module.
Gotcha: the Cedar policy statements embed the gateway's ARN literally. Every `redeploy-gateway` (which changes the gateway ID) requires re-`terraform apply`-ing the policies too, since Terraform recomputes the ARN string in the `cedar.statement` — this repo's `terraform apply` handles that as a normal in-place policy update, not a recreate.
M5 — Interceptors
`lambda-interceptor.tf`. A Lambda wired into `interceptor_configurations` on `REQUEST` and `RESPONSE` interception points, with `pass_request_headers = true`.
Bug hit and fixed: after changing the interceptor's `handler` from `index.handler` to `index2.handler` via Terraform, the next agent run failed with `Connection to the MCP server was closed` / `MCPClientInitializationError: the client session is not running`. The interceptor Lambda's session-handling logic didn't match the handler AgentCore was invoking mid-session — the fix was reverting to the handler the gateway's live session was expecting and doing a full `redeploy-gateway` rather than an in-place Lambda code update, since the gateway appears to cache session/handler state across a plain `terraform apply`.
M6 — Outbound identity (API key credential provider)
`promotions-backend.tf`. A separate promotions backend (API Gateway HTTP API → Lambda) that requires `x-api-key`. AgentCore Gateway holds that key via `aws_bedrockagentcore_api_key_credential_provider` and injects it as an outbound header when the `promotions` OpenAPI-schema target calls the backend — the MCP client never sees or handles the backend's API key at all. This is the pattern for wrapping any API-key-protected third-party service as an MCP tool without exposing the key to callers.
M7 — Strands agent
`src/agent/agent.py`. A Strands-based conversational agent consuming all three tools (`get-menu`, `create-order`, `promotions`) via the gateway's MCP endpoint, run with `make run-agent-client2` (client2 has both `get_menu` and `create_order` scopes).
Bug hit, not yet root-caused: immediately after a `redeploy-gateway`, the first agent session's tool calls failed with HTTP 403 followed by `RuntimeError: Connection to the MCP server was closed` / `MCPClientInitializationError`, on every tool call in that session — menu, promotions, and order alike. Killing and restarting `make run-agent-client2` (same client, same token flow) fixed it immediately with no other changes. Likely a stale MCP session/connection established against the gateway's previous ID or a brief propagation delay right after `awscc_bedrockagentcore_gateway` finishes its replace — worth watching for if you see this: retry a fresh session before assuming something's actually broken.
M8–M9 — Observability & cleanup
CloudWatch log/trace delivery pipelines (`gateway-observability.tf`) route both gateway application logs and X-Ray traces through `aws_cloudwatch_log_delivery`. Full teardown via `make destroy`; log deliveries must be destroyed before the gateway itself (handled by `redeploy-gateway`'s explicit `-target` ordering, and by `make destroy`'s dependency graph).
Cleanup
```bash
make destroy
```
If it errors on a dependency still detaching (e.g. a Cognito domain or log delivery), wait ~15s and re-run — this is a known AgentCore/Terraform timing issue, not a broken config.
Known rough edges
`redeploy-gateway` always assigns a new gateway ID/URL — nothing about the gateway identity is stable across an auth-config change.
The policy engine denies by default the moment it's attached, before any policies exist.
MCP client sessions can go stale right after a gateway replace; a fresh session usually resolves it.
