# Sandbox Agent Deployment runtime credential and authentication boundary

**Research date:** 2026-09-01

**Ticket:** [Define the runtime credential and authentication boundary](https://github.com/tlipoca9/ags-cookbook/issues/2)

**Scope:** planned, local-debug-oriented Sandbox Agent Deployment tutorial in `ap-shanghai`; no production ingress design

**Evidence snapshots:** Sandbox Agent `v0.4.2` / `d55b0dfb887cc92152f20f756995317c5f5c7709`; `agr v0.6.6` / `67f83b4b08037483e8a3b3c3000e788901b1c2ed`

## Status vocabulary

- **Verified in source/help:** the cited first-party implementation or locally installed `agr v0.6.6` exposes the behavior.
- **Provisional tutorial decision:** safe enough only for the controlled validation described here, subject to the Shanghai gates below.
- **Unverified:** no public contract or live `ap-shanghai` evidence was found; the tutorial must not state it as guaranteed.

## Decision summary

Use three deliberately separate credential domains:

1. **Model-provider credential:** inject only the provider variable selected by the provider-contract ticket into the Sandbox Tool's `CustomConfiguration.Env`. For the expected OpenAI-compatible/Codex path this is `OPENAI_API_KEY`; Codex also recognizes `CODEX_API_KEY`. Claude paths use `ANTHROPIC_API_KEY` or `CLAUDE_API_KEY`. Sandbox Agent checks these environment names, and its own documentation warns that the agent and everything else in the sandbox can read an injected token. Do not inject aliases redundantly. [Sandbox Agent credential table and exposure warning](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L7-L45) [credential lookup order in source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/agent-credentials/src/lib.rs#L270-L319)
2. **AGS Deployment credential:** let `agr deployment proxy` acquire the Deployment-scoped credential, keep it in memory, refresh it, and inject it as `X-Access-Token`. Do not put a Deployment token in the image, Tool, provider environment, request examples, or evidence. The proxy is explicitly local-debug-only and binds to loopback by default. [`agr` proxy command](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L40-L70) [`agr` token acquisition contract](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/controlplane/sdk.go#L295-L304) [`agr` gateway-header injection](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L150-L164)
3. **Sandbox Agent server credential:** start the server with explicit `--no-token` for this provisional local-debug path and rely on the AGS Deployment-token gateway at external ingress. Sandbox Agent otherwise supports one global token checked as an exact `Authorization: Bearer` value, but that is a different credential from `X-Access-Token`. [`--token` versus disabled auth](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/cli.rs#L450-L455) [Bearer validation](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router/support.rs#L21-L53)

The third decision is **provisional, not an assertion that `--no-token` is generally safe**. It becomes tutorial-ready only if Shanghai validation proves all three conditions: an unauthenticated request to the public Deployment endpoint fails at the gateway, a request through the loopback `agr` proxy succeeds, and the tutorial user has no direct/internal route to port `2468` that bypasses the gateway. If any condition fails or cannot be established, server-token composition remains unresolved and the prototype stays blocked.

This is not a production or multi-tenant pattern. Upstream requires a backend boundary that authenticates the caller, authorizes workspace/sandbox/session access, and applies rate limits and request logging. [Sandbox Agent security guidance](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/security.mdx#L7-L19)

## Recommended layered-auth contract

| Layer | Credential/header | Who obtains it | Where it lives | What it protects | Tutorial decision |
| --- | --- | --- | --- | --- | --- |
| Provider | Selected provider variable, normally `OPENAI_API_KEY` for the expected path | Human operator from provider | Local parent-shell environment, then persisted as Tool `CustomConfiguration.Env`, then inherited by server/agent processes | Model API spend and access | Purpose-created, spend-capped, preferably short-lived key only; no shared organization master key |
| AGS control plane | `TENCENTCLOUD_SECRET_ID`, `TENCENTCLOUD_SECRET_KEY`, optional `TENCENTCLOUD_TOKEN` | Human/STS system | Local `agr` credential resolution only | Tool/Deployment operations and Deployment-token acquisition | Never inject into the Tool or sandbox for this scenario |
| AGS data plane | Deployment token in `X-Access-Token` | `agr deployment proxy` via `AcquireDeploymentToken` | Proxy memory and upstream request header | Access to the target Deployment endpoint | Required; never print or persist |
| Sandbox Agent | Optional global Bearer in `Authorization` | Tutorial operator/application | Server argument plus client header | Sandbox Agent `/v1` API surface reached after AGS ingress | Disabled provisionally with explicit `--no-token`; do not invent a token for the golden local-debug path |
| End-user/application | Session/JWT/API-key scheme chosen by application | Application backend | Backend/client contract | User identity, workspace/session authorization, limits, audit | Out of scope for this tutorial, mandatory in production |

The AGS control-plane variables are separate from the Deployment token. `agr` documents `TENCENTCLOUD_SECRET_ID`, `TENCENTCLOUD_SECRET_KEY`, and `TENCENTCLOUD_TOKEN` as CLI credential inputs, while its Deployment proxy calls the control plane to acquire a Deployment token for the specified Deployment. [`agr` credential inputs](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/README.md#L131-L164) [`agr` Deployment proxy acquisition](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L142-L169)

`agr deployment proxy` preserves incoming application headers for Deployment traffic and overwrites/injects `X-Access-Token`; therefore a future client can carry a distinct Sandbox Agent `Authorization` Bearer through the proxy if double authentication is later required. This composition is source-verified, but it is deliberately not the golden-path decision because it adds secret provisioning and client handling that the local-debug path does not need. [`PreserveHeaders: true`](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L160-L170) [header clone plus gateway-token injection](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L537-L552)

## Exact AGS environment contract

### What `agr` actually exposes

For a custom Tool, `agr v0.6.6` accepts this concrete shape:

```json
{
  "Command": ["sandbox-agent"],
  "Args": ["server", "--no-token", "--host", "0.0.0.0", "--port", "2468"],
  "Env": [
    {"Name": "OPENAI_API_KEY", "Value": "<read-from-parent-environment>"}
  ]
}
```

The exact injection seam is `agr tool create --custom-configuration <JSON|@file|->`; `-` reads the JSON object from stdin. `CustomConfiguration` contains `Command`, `Args`, and `Env`, and each environment entry is only a `Name`/`Value` pair. No secret-reference, `valueFrom`, or managed-secret field appears in the `agr v0.6.6` command schema or AGS SDK model. This is a statement about the inspected CLI/API surface, not a claim that no other Tencent Cloud secret product exists. [`agr v0.6.6` Tool input definition](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/create/api.generated.go#L101-L106) [official AGS SDK `CustomConfiguration`](https://github.com/TencentCloud/tencentcloud-sdk-go/blob/0779918cdca6f337ec4cfc735c3c2743692bc0e9/tencentcloud/ags/v20250920/models.go#L570-L596) [official AGS SDK `EnvVar`](https://github.com/TencentCloud/tencentcloud-sdk-go/blob/0779918cdca6f337ec4cfc735c3c2743692bc0e9/tencentcloud/ags/v20250920/models.go#L1279-L1286)

The Deployment itself references a Tool and has no environment override in `agr deployment create`; therefore a Deployment-managed instance receives this tutorial credential through the Tool template, not through `deployment create` and not through `agr instance exec --env`. [`agr v0.6.6` Deployment create fields](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/create/api.generated.go#L31-L80) [cookbook explanation of PID 1 environment scope](https://github.com/TencentCloudAgentRuntime/ags-cookbook/blob/af7c77addc82baefc76276a9a5eec1aee47aed9f/examples/envd-oci-env/README.md#L108-L137)

`CustomConfiguration.Env` is **not a secret store**. It persists a clear `Value` in Tool configuration; `agr tool get -o json` includes `CustomConfiguration`, so an authorized control-plane reader may retrieve it. The value is also present in the sandbox process environment. [`agr` canonical Tool JSON includes `CustomConfiguration`](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/get/command.go#L135-L146) [Sandbox Agent exposure warning](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L36-L45)

### Safest available tutorial injection procedure

The public tutorial should show placeholders only. During controlled validation, the operator exports the real provider key interactively or through an approved local secret loader, disables shell tracing, and streams generated JSON directly to `agr` stdin. This keeps the value out of the literal command, shell history, process argv, temporary files, repository, and Docker build context. It **does not** prevent persistence in the AGS Tool or access from inside the sandbox.

```bash
set +x
: "${OPENAI_API_KEY:?set a purpose-created, spend-capped validation key}"

TOOL_ID=$(
  python3 - <<'PY' |
import json
import os
import sys

json.dump(
    {
        "Image": "<pinned-ccr-image>",
        "ImageRegistryType": "personal",
        "Command": ["sandbox-agent"],
        "Args": ["server", "--no-token", "--host", "0.0.0.0", "--port", "2468"],
        "Env": [
            {"Name": "OPENAI_API_KEY", "Value": os.environ["OPENAI_API_KEY"]},
        ],
        "Ports": [{"Name": "http", "Port": 2468, "Protocol": "TCP"}],
        "Resources": {"CPU": "<validated-cpu>", "Memory": "<validated-memory>"},
        "Probe": {
            "HttpGet": {"Path": "/v1/health", "Port": 2468, "Scheme": "HTTP"}
        },
    },
    sys.stdout,
    separators=(",", ":"),
)
PY
  agr tool create \
    --region "$AGR_REGION" \
    --tool-name "$SANDBOX_AGENT_TOOL_NAME" \
    --tool-type custom \
    --persistent \
    --role-arn "$AGR_ROLE_ARN" \
    --network-configuration '{"NetworkMode":"PUBLIC"}' \
    --custom-configuration - \
    --output json \
    --jq '.Data.ToolId'
)

unset OPENAI_API_KEY
```

Use the exact provider variable selected by the provider ticket in place of `OPENAI_API_KEY`; never add all possible aliases. Upstream recognizes the variable names listed above and starts adapter processes as children of the server; Rust `Command` inherits the parent environment unless explicitly cleared, and the adapter only adds its launch-specific environment. [credential variables](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L26-L34) [adapter spawn code](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/acp-http-adapter/src/process.rs#L73-L104)

After validation, delete the Deployment and Tool, wait for deletion to complete, revoke/rotate the provider credential at its issuer, and clear it from the shell. Tool deletion alone is not evidence that provider-side revocation happened.

### Command and argument expansion constraints

Treat `Command` and `Args` as an exec-style string vector, not a shell script. `agr` parses JSON into the AGS SDK's string arrays and submits them; neither the inspected CLI schema nor SDK model defines environment interpolation. Therefore an argument such as `"$OPENAI_API_KEY"` must be assumed to reach the process literally. This is source-derived and still requires one non-secret Shanghai probe before publication. [`agr` structured `Command`/`Args` example](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/create/api.generated.go#L101-L106) [AGS SDK array fields](https://github.com/TencentCloud/tencentcloud-sdk-go/blob/0779918cdca6f337ec4cfc735c3c2743692bc0e9/tencentcloud/ags/v20250920/models.go#L579-L587)

If shell expansion is intentionally needed, make the shell explicit, for example `Command: ["/bin/sh"]` and `Args: ["-c", "exec ..."]`. Do not use that mechanism for secrets: expansion into `sandbox-agent --token "$SANDBOX_AGENT_TOKEN"` would put the resolved token in the server process argument vector, while leaving `"$SANDBOX_AGENT_TOKEN"` in direct `Args` would be literal under the contract above. The no-token golden path avoids both failure modes.

Do not rely on image `ENTRYPOINT`/`CMD` merge behavior for the tutorial. Set `Command` and `Args` explicitly and validate the resulting process command with non-secret arguments in Shanghai.

## Threat and exposure table

| Asset/surface | Who can plausibly read it | Exposure created by the contract | Required mitigation/evidence |
| --- | --- | --- | --- |
| Local provider key before creation | Operator account, shell children, local debugging/EDR tools | Exported environment can be read by processes with sufficient local privilege | Use a dedicated shell, `set +x`, no shared terminal recording, approved secret loader, immediate `unset` |
| Shell history and process argv | Shell history readers, same-host process observers | Inline JSON, `--arg "$KEY"`, or literal `--token` can expose values | stdin generator reads `os.environ`; no secret-bearing flags or command strings |
| Git, worktree, issue/PR text | Repository readers and forks | Permanent, replicated disclosure | Placeholders only; never stage `.env`, rendered JSON, transcripts, screenshots, or copied API responses containing values |
| Docker build context/image/history | Registry readers and image consumers | `ARG`, `ENV`, `RUN`, or `COPY` can persist credentials in layers/history | Provider key is runtime Tool input only; `.dockerignore` must exclude local secret files; inspect Dockerfile/history without a real key |
| AGS Tool control-plane record | Principals authorized to describe/read Tools, service operators under platform policy | Clear `Env[].Value` persists with the Tool and can be returned in `tool get -o json` | Purpose-created capped/short-lived key, least-privilege CAM access, no secret-bearing Tool JSON in evidence, prompt Tool deletion plus provider revocation; Shanghai must verify actual visibility/redaction |
| Sandbox Agent process environment | Sandbox Agent, child agents, same-UID processes, workload with environment/proc access | Provider key is intentionally available to the workload | One trust domain per instance; never execute untrusted prompts/code against a valuable shared key; prefer scoped gateway key |
| Agent output, errors, server logs, CLS | Anyone with output/log access | Prompted code or diagnostics can print environment and headers | Test only key presence via boolean/credential-discovery signals; ban `env`, `printenv`, `/proc/*/environ`, key-prefix/length output, header dumps, and secret-bearing debug logs |
| Deployment token | `agr` process memory and upstream gateway request | Compromise permits target Deployment access until expiry | Use proxy auto-acquisition; never call raw acquisition for tutorial evidence; never log/persist; bind proxy to loopback |
| Loopback proxy | Local host users/processes and, if rebound, network peers | Any local caller can ride the proxy's injected Deployment token while it runs | Default `127.0.0.1`; never use `--address 0.0.0.0`; stop immediately after test; dedicated validation host if local users are untrusted |
| Public Deployment endpoint | Anyone who learns endpoint, subject to gateway | Gateway is the only provisional server ingress auth when `--no-token` | Prove missing/invalid `X-Access-Token` is rejected and no bypass is available before unblocking |
| Sandbox Agent global Bearer alternative | Every holder and process-argument observer on server | One static server-wide secret, no per-user/session authorization | Not used in golden path; if later required, provision separately, rotate, send as `Authorization`, and still add backend authz/rate limits/audit |
| Affinity ID/session/workspace IDs | Tutorial readers if captured | Not provider credentials, but may reveal or resume a session depending on service behavior | Mask in published evidence and keep only as long as the resume test needs it |

The provider exposure and shared-key risk are explicit upstream: everything in the sandbox can access the token, and a single exfiltrated shared organization key exposes the organization's budget; upstream recommends scoped per-tenant gateway keys for externally exposed production use. [Sandbox Agent credential strategies](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L36-L49) [shared-key warning](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L197-L216)

The proxy-specific boundaries are source-backed: it is documented as local-debug-only, defaults to `127.0.0.1`, warns on non-loopback binding, and its verbose HTTP log records method/path rather than credentials. [`agr` local proxy contract](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L40-L70) [loopback and warning behavior](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L108-L127) [secret-safe request logging implementation](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L150-L163)

## Exact redaction and documentation rules

These are normative for the planned tutorial and its validation artifacts.

1. **Placeholders only in Git and docs.** Use `<provider-api-key>`, `<masked-deployment-token>`, `<masked-affinity-id>`, and structurally fake IDs such as `sdt-replace-me`; do not use a real-looking prefix plus a real suffix. Repository contribution guidance already requires placeholder-only `.env.example` files and forbids real secrets. [cookbook contribution guide](https://github.com/TencentCloudAgentRuntime/ags-cookbook/blob/af7c77addc82baefc76276a9a5eec1aee47aed9f/CONTRIBUTING.md#L52-L77)
2. **Never commit a rendered secret-bearing Tool request.** A checked-in template must omit the provider entry or use a non-secret marker. The real object exists only in the stdin pipe and AGS control plane.
3. **No secret-bearing command literals.** Do not show a provider variable assigned to anything except an explicit redaction marker, inline `Env[].Value`, `--secret-key`, raw `X-Access-Token`, or an actual `Authorization` credential in a command, transcript, screen recording, screenshot, CI annotation, or issue comment.
4. **No shell tracing during secret handling.** Run `set +x` before loading/generating/injecting credentials. Do not use `env`, `set`, `export -p`, `printenv`, `ps e`, `/proc/*/environ`, or commands that print request bodies.
5. **Do not capture broad Tool JSON.** `agr tool get -o json` includes `CustomConfiguration`; validation must query only non-secret fields with a reviewed `--jq` projection or use a sanitized text summary. [`agr` Tool JSON shape](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/get/command.go#L135-L146)
6. **Do not capture provider values indirectly.** Evidence may say `credential detected: true` or show a successful minimal model turn. It must not show value, prefix, suffix, length, hash, encoded form, or exception text containing request headers.
7. **Use proxy auto-acquisition.** Do not run `AcquireDeploymentToken` in the tutorial's local-debug proof, because raw API output contains the token. `agr`'s proxy contract says the returned credential stays in memory and must not be logged or persisted. [`agr` token handling requirement](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/controlplane/sdk.go#L295-L304)
8. **Redact identifiers as defense in depth.** Mask account IDs, role ARNs beyond a synthetic example, Tool/Deployment/Instance IDs, request IDs, timestamps when identifying, affinity IDs, hostnames containing real resource IDs, and absolute local config paths.
9. **Sanitize before capture, then scan.** Capture only allowlisted output. Before commit, search tracked and untracked files plus the staged diff for forbidden variable assignments, credential headers, common key/token prefixes, and unmasked resource IDs. A scan complements review; it is not proof of absence by itself.
10. **Cleanup is part of redaction.** Stop the proxy, delete Deployment and Tool with waited deletion, revoke/rotate the validation key at the provider, `unset` the variable, remove any local transcript, and confirm `git status --short` and `git diff --cached` are empty except for intended documentation during research.

## Alternatives considered

### A. Recommended provisional path: AGS gateway only, server `--no-token`

**Why:** no extra long-lived server secret, no token in server argv, works naturally with `agr deployment proxy`, and matches upstream's statement that an infrastructure-secured sandbox provider often does not need the server token. It remains conditional on proving that the AGS gateway cannot be bypassed in the tutorial's reachable topology. [Sandbox Agent remote-server token guidance](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/quickstart.mdx#L163-L202)

### B. Double authentication: AGS `X-Access-Token` plus Sandbox Agent Bearer

**Composition:** client supplies `Authorization: Bearer <server-token>`; `agr` preserves it and adds/overwrites `X-Access-Token`. Sandbox Agent validates its own Bearer after gateway transit. [`agr` header preservation](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L537-L552) [Sandbox Agent Bearer validation](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router/support.rs#L21-L53)

**Why not golden path now:** there is no inspected Sandbox Agent environment option for its global token, direct `Args` do not provide documented environment expansion, a shell wrapper would resolve the token into process argv, clients must manage a second credential, and the Tool health probe's authenticated-header capability is not present in the inspected `agr` probe schema. This can be revisited only with an image entrypoint that reads a protected secret source without exposing it in argv plus verified probe behavior.

### C. Provider key in image or checked-in file

Rejected. It creates durable disclosure in Git/build context/image layers and is unnecessary because AGS exposes runtime `Env`. The cookbook's existing secret guidance also excludes `.env` from Git and Docker build context and forbids Dockerfile `COPY`, `ARG`, or `ENV` for keys. [existing cookbook runtime-secret pattern](https://github.com/TencentCloudAgentRuntime/ags-cookbook/blob/af7c77addc82baefc76276a9a5eec1aee47aed9f/examples/harness-nix-volume/README.md#L89-L97)

### D. Raw Deployment-token handling

Rejected for the local-debug tutorial. The proxy already acquires, refreshes, and injects the credential, while raw acquisition risks terminal/transcript exposure. Production clients may need direct token acquisition, but production ingress/client design is explicitly outside this ticket. [`agr` proxy acquisition and injection](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L142-L169)

### E. Per-tenant model gateway or managed secret delivery

Preferred production direction, not part of the tutorial implementation. Upstream recommends scoped gateway keys with budgets because exfiltration then exposes only the tenant allowance. No verified AGS managed-secret reference for `CustomConfiguration.Env` was found in `agr v0.6.6`; if Tencent Cloud provides one, it should replace clear Tool values after separate verification. [upstream gateway recommendation](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L36-L55)

## Evidence gaps and validation required in Shanghai

The public prototype remains **blocked** until every security-critical item below has recorded, redacted evidence.

1. **Gateway enforcement:** against the real `ap-shanghai` Deployment endpoint, prove no/malformed/expired `X-Access-Token` is rejected and the proxy-acquired token succeeds. Record status/shape only, never response headers or token values.
2. **No gateway bypass:** inventory all addresses/routes visible to the tutorial operator and prove port `2468` is not directly reachable around the Deployment gateway. If an internal/direct path is reachable or cannot be assessed, do not publish `--no-token` as safe.
3. **Tool value visibility:** with a synthetic canary, determine exactly which `agr tool get/list`, Tencent Cloud API, console, audit, and log surfaces return `Env[].Value`, and which CAM permissions can read them. Current source proves `agr` includes returned `CustomConfiguration`; service-side masking and at-rest controls are unverified. [`agr` Tool result mapping](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/get/command.go#L135-L146)
4. **Runtime delivery:** inject a non-secret canary through Tool `Env` and prove it reaches Sandbox Agent PID 1 and the chosen agent child without printing the value. Then use the purpose-created provider key and prove only a successful minimal turn, not key contents.
5. **Command semantics:** set a non-secret value and prove direct `Args: ["$CANARY"]` remains literal; separately prove the explicit `Command`/`Args` launch vector. Do not infer expansion behavior from Docker or Kubernetes.
6. **Probe/auth interaction:** prove `/v1/health` works with the provisional `--no-token` Tool probe. If double auth becomes necessary, verify whether AGS probe headers are supported or design a separate non-sensitive local probe before enabling Bearer.
7. **Proxy secrecy:** run normal and `--verbose` proxy modes with synthetic credentials and inspect stdout/stderr, local process state, and any configured logs for token/header leakage. Source logs method/path only and genericizes credential errors, but live packaging and service logs must be checked. [`agr` request/error log behavior](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L150-L176) [`agr` generic credential errors](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L520-L527)
8. **Rotation/revocation:** update or replace the Tool with a synthetic credential, observe which paused/running instances retain the old environment, then delete resources and verify provider revocation. No rotation semantics for Deployment-managed instances were established by this research.
9. **Least privilege:** identify and document the minimum CAM actions for Tool/Deployment management and token acquisition, plus who may describe Tool configuration. Do not infer these permissions from successful administrator credentials.
10. **Egress and provider scope:** verify the chosen provider endpoint is reachable and that the validation key's spend cap, expiration, revocation, and organization scope match the provider ticket. A long-lived shared organization key does not satisfy this contract. [upstream shared-key warning](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L197-L216)

## Production caveats

- The AGS Deployment token authenticates access to a Deployment endpoint; it does not supply application user identity, workspace/session authorization, rate limiting, audit policy, or model-budget isolation. Upstream says those controls belong in a backend before sandbox-bound requests. [Sandbox Agent security guidance](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/security.mdx#L7-L19)
- Sandbox Agent's optional token is one server-wide shared Bearer, not a tenant/session authorization system. Even if enabled, production still needs the backend controls above. [server token implementation](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router/support.rs#L21-L53)
- `CustomConfiguration.Env` provides runtime delivery, not secret management. Production should use scoped, revocable, short-lived gateway credentials or a verified managed-secret integration, isolate tenants at an appropriate sandbox boundary, restrict Tool-read permissions, and define rotation behavior. [upstream scoped-gateway recommendation](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L36-L55)
- `agr deployment proxy` is an operator debugging convenience, not production ingress. Its loopback listener automatically rides a Deployment credential, so any process able to call it can use that access while the proxy is running. [`agr` local-debug warning and bind default](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L40-L70)
- Provider keys must be assumed exfiltratable by agent-generated code. Spend caps and short lifetime reduce impact but do not make the sandbox a secret enclave. [Sandbox Agent exposure statement](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx#L36-L45)

## Implications for the blocked prototype

The prototype can proceed to a **controlled Shanghai validation only** with the stdin injection procedure, a purpose-created spend-capped and preferably short-lived provider key, explicit `--no-token`, loopback-only `agr deployment proxy`, and the redaction rules above. It cannot yet be promoted to customer documentation.

Promotion remains blocked on:

- gateway enforcement and no-bypass evidence for the provisional `--no-token` choice;
- actual Tool value visibility/redaction and least-privilege CAM evidence;
- environment delivery, command literal behavior, probe behavior, and key rotation/revocation evidence;
- the separate provider ticket choosing the exact environment variable, endpoint, model, key scope, and revocation procedure;
- redacted end-to-end pause/resume validation showing no secret in stdout, stderr, logs, screenshots, saved requests, Git, or staged changes.

If Shanghai cannot prove the no-bypass condition, the implementation-ready specification must not silently switch to a server token. It must instead keep authentication unresolved until a non-argv server-secret delivery and authenticated health-probe design is evidenced.

## Primary sources

- [Sandbox Agent v0.4.2](https://github.com/rivet-dev/sandbox-agent/tree/d55b0dfb887cc92152f20f756995317c5f5c7709)
- [Sandbox Agent LLM credential guidance](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/llm-credentials.mdx)
- [Sandbox Agent security guidance](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/security.mdx)
- [Sandbox Agent server auth source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router/support.rs)
- [`agr v0.6.6` source](https://github.com/TencentCloudAgentRuntime/ags-cli/tree/67f83b4b08037483e8a3b3c3000e788901b1c2ed)
- [`agr` Tool custom-configuration input](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/create/api.generated.go#L101-L106)
- [`agr` Deployment proxy implementation](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go)
- [Tencent Cloud AGS SDK models](https://github.com/TencentCloud/tencentcloud-sdk-go/blob/0779918cdca6f337ec4cfc735c3c2743692bc0e9/tencentcloud/ags/v20250920/models.go)
- [Tencent Cloud Sandbox Tool data structures](https://cloud.tencent.com/document/product/1814/124823)
- [Tencent Cloud CreateDeployment API](https://cloud.tencent.com/document/product/1814/136841)
- [Existing deployment-cookbook local proxy/token pattern](https://github.com/TencentCloudAgentRuntime/ags-cookbook/blob/af7c77addc82baefc76276a9a5eec1aee47aed9f/examples/deployment-cookbook/httpbin/simple/README.md#L138-L225)
