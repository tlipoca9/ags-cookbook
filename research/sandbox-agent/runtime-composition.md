# Immutable-image runtime composition for Sandbox Agent

**Research ticket:** [Define the immutable-image runtime composition seam](https://github.com/tlipoca9/ags-cookbook/issues/21)

**Evidence cutoff:** 2026-09-01

**Pinned inputs:** Sandbox Agent `v0.4.2` / `d55b0dfb887cc92152f20f756995317c5f5c7709`; official `linux/amd64` child image `sha256:a7b9afc8c79fb075852d3ae73b86ae201b5ce683b1bf5f11988980cd2bca09a5`; image-bundled Codex CLI `0.116.0`; `agr` `v0.6.6` source snapshot `67f83b4b08037483e8a3b3c3000e788901b1c2ed`.

## Decision

Keep the mirrored Sandbox Agent image byte-for-byte unchanged and compose the missing runtime state in the AGS Tool:

1. Override the image command with an explicit `/bin/bash` bootstrap.
2. At each container start, create the writable `/home/sandbox` state directories and atomically write one **non-secret** Codex `config.toml` into `/home/sandbox/.codex`.
3. Inject only `HY3_API_KEY` as the provider credential in Tool `CustomConfiguration.Env`; never interpolate or copy it into the bootstrap script, Codex config, image, repository, or request body.
4. Put the OpenCode compatibility database at `/home/sandbox/.local/state/sandbox-agent/opencode-compat.db` and the coding workspace at `/home/sandbox/workspace`.
5. End the bootstrap with `exec /usr/local/bin/sandbox-agent server --no-token --host 0.0.0.0 --port 2468`.
6. On the native Sandbox Agent session path, create sessions with `agent: "codex"` and `mode: "auto"`, and omit the session-level `model` field. Codex obtains `hy3` and `hy3-tokenhub` from its own config.

This is a runtime composition seam, not a downstream image variant. The only long-lived configuration file contains no credential; the provider key remains an environment value inherited by Sandbox Agent's Codex child process.

The runtime composition is decided, but the current raw `/opencode` HTTP/SSE journey is **not** closed: Sandbox Agent `v0.4.2` does not propagate OpenCode-compatible request agent/mode metadata into the backing session's `agent_mode`. The native SDK can apply `codex`/`auto`; the pinned OpenCode compatibility layer cannot. The final scenario must either use the native session path, select a later immutable upstream image that closes this gap, or remain blocked. It must not claim that an `/opencode` request enabled `auto`.

The decision also depends on the already-selected **provisional** `--no-token` local-debug boundary. It does not promote that boundary to a production recommendation. The Shanghai validation must still prove AGS gateway enforcement and absence of a direct port-2468 bypass, as required by the [authentication decision](https://github.com/tlipoca9/ags-cookbook/blob/978b010fcb262d01c6f3c5142a99bdff866721e2/research/sandbox-agent/auth-boundary.md).

## Exact runtime contract

### Fixed paths and values

| Field | Required value |
| --- | --- |
| CCR image | `ccr.ccs.tencentyun.com/ags.dev/sandbox-agent:v0.4.2-full-amd64`, accepted only after its recorded manifest digest equals the approved amd64 child digest |
| Runtime identity | image user `sandbox` (`uid=1001`, `gid=1001`) |
| Runtime home / working directory | `/home/sandbox` |
| Bootstrap command | `Command: ["/bin/bash"]` |
| Bootstrap arguments | `Args: ["-eu", "-c", "<script below>"]` |
| Codex home | `/home/sandbox/.codex` |
| Codex config | `/home/sandbox/.codex/config.toml`, generated at start, non-secret |
| Provider secret | `HY3_API_KEY`, Tool environment only |
| OpenCode SQLite database | `/home/sandbox/.local/state/sandbox-agent/opencode-compat.db` |
| Coding workspace | `/home/sandbox/workspace` |
| Server process | `/usr/local/bin/sandbox-agent server --no-token --host 0.0.0.0 --port 2468` |
| Native session selection | `agent: "codex"`, `mode: "auto"`, model omitted |
| `/opencode` session selection | blocked on `v0.4.2`; no source-backed way to apply Codex `auto` mode |

The immutable image itself establishes the non-root user, `/home/sandbox` workdir, installed Bash, server binary, port, and default server arguments ([upstream Dockerfile](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docker/runtime/Dockerfile.full#L137-L159)). The [image decision](https://github.com/tlipoca9/ags-cookbook/blob/9b97c380511563eca66b6ed36f91212456a5ac9f/research/sandbox-agent/ccr-image.md) fixes the mirrored manifest and forbids adding a downstream layer.

### Tool environment

The Tool must supply exactly these runtime variables for this seam:

```json
[
  {"Name": "HOME", "Value": "/home/sandbox"},
  {"Name": "CODEX_HOME", "Value": "/home/sandbox/.codex"},
  {"Name": "OPENCODE_COMPAT_DB_PATH", "Value": "/home/sandbox/.local/state/sandbox-agent/opencode-compat.db"},
  {"Name": "HY3_API_KEY", "Value": "<read from the operator environment>"}
]
```

Do not also set `OPENAI_API_KEY`, `CODEX_API_KEY`, `OPENCODE_COMPAT_STATE`, a server token, or a session model. Redundant aliases create ambiguous fallback behavior, while `OPENCODE_COMPAT_DB_PATH` already has precedence over `OPENCODE_COMPAT_STATE` in `v0.4.2` ([adapter source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L644-L670)).

`CustomConfiguration.Env` is runtime delivery, not a managed secret store. `agr` accepts only `Name`/`Value` environment entries and an authorized Tool read can include the configuration. A real key must therefore be purpose-created, scoped and spend-capped, streamed from a protected operator environment to `agr tool create --custom-configuration -`, and revoked after validation. The key is “runtime-only” relative to the image, Git, files, args, requests, and evidence; it is still persisted in the Tool control-plane record under the currently inspected API ([Tool input](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/create/api.generated.go#L101-L106), [canonical authentication boundary](https://github.com/tlipoca9/ags-cookbook/blob/978b010fcb262d01c6f3c5142a99bdff866721e2/research/sandbox-agent/auth-boundary.md#exact-ags-environment-contract)).

### Bootstrap script

Use this exact non-secret script as the third item of `Args`:

```bash
umask 077

mkdir -p \
  /home/sandbox/.codex \
  /home/sandbox/.local/state/sandbox-agent \
  /home/sandbox/workspace

config_tmp="/home/sandbox/.codex/config.toml.tmp"
trap 'rm -f "$config_tmp"' EXIT HUP INT TERM

cat >"$config_tmp" <<'CODEX_CONFIG'
model_provider = "hy3-tokenhub"
model = "hy3"

[model_providers.hy3-tokenhub]
name = "Hy3 via TokenHub"
base_url = "https://tokenhub.tencentmaas.com/v1"
env_key = "HY3_API_KEY"
wire_api = "responses"
CODEX_CONFIG

mv "$config_tmp" /home/sandbox/.codex/config.toml
trap - EXIT HUP INT TERM
cd /home/sandbox

exec /usr/local/bin/sandbox-agent \
  server \
  --no-token \
  --host 0.0.0.0 \
  --port 2468
```

Do not add Tencent's newer `disable_response_storage = true` example setting. Codex `0.116.0` marks `ConfigToml` as `deny_unknown_fields`, and that release's configuration type/schema contains no such field ([configuration type](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/config/mod.rs#L1193-L1204)). The six TOML assignments above are the complete selected provider contract for this pinned binary.

The bootstrap deliberately invokes a shell because it must create one runtime file. AGS `Command` and `Args` remain string arrays with no documented direct environment interpolation; using an explicit shell is the prior decision's supported escape hatch for non-secret composition ([`agr` structured input](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/create/api.generated.go#L101-L106), [authentication decision](https://github.com/tlipoca9/ags-cookbook/blob/978b010fcb262d01c6f3c5142a99bdff866721e2/research/sandbox-agent/auth-boundary.md#command-and-argument-expansion-constraints)). The script never expands, reads, copies, logs, or tests `HY3_API_KEY`; the eventual Codex process reads it directly using `env_key`.

The temporary-file-and-rename sequence prevents Codex from observing a partially written configuration. `exec` makes Sandbox Agent PID 1 and preserves signal delivery. All server and authentication arguments are literal and visible for inspection; none contains a credential. Sandbox Agent `v0.4.2` exposes `--token` and `--no-token`, and absence of a supplied token selects disabled server authentication ([CLI flags](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/cli.rs#L41-L54), [server auth construction](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/cli.rs#L450-L466)). `--no-token` is retained explicitly so the Tool configuration states the provisional decision rather than inheriting an unauthenticated default silently.

A local launch of this exact bootstrap against the approved image digest succeeded without a real credential: `/v1/health` returned exact `{"status":"ok"}`, the process ran as uid/gid `1001`, the working directory was `/home/sandbox`, `config.toml` was mode `0600`, and the workspace/state directories were mode `0700`. After the first `/opencode/session` read initialized the adapter, the selected state directory contained `opencode-compat.db`, `opencode-compat.db-wal`, and `opencode-compat.db-shm`. This verifies local image behavior and path writability, not AGS command merging or pause persistence.

### CustomConfiguration shape

The resulting relevant Tool fields are:

```json
{
  "Image": "ccr.ccs.tencentyun.com/ags.dev/sandbox-agent:v0.4.2-full-amd64",
  "ImageRegistryType": "personal",
  "Command": ["/bin/bash"],
  "Args": ["-eu", "-c", "<exact bootstrap script above>"],
  "Env": [
    {"Name": "HOME", "Value": "/home/sandbox"},
    {"Name": "CODEX_HOME", "Value": "/home/sandbox/.codex"},
    {"Name": "OPENCODE_COMPAT_DB_PATH", "Value": "/home/sandbox/.local/state/sandbox-agent/opencode-compat.db"},
    {"Name": "HY3_API_KEY", "Value": "<read from the operator environment>"}
  ],
  "Ports": [
    {"Name": "http", "Port": 2468, "Protocol": "TCP"}
  ],
  "Resources": {
    "CPU": "<validated-cpu>",
    "Memory": "<validated-memory>",
    "Storage": "<validated-storage>"
  },
  "Probe": {
    "HttpGet": {"Path": "/v1/health", "Port": 2468, "Scheme": "HTTP"}
  }
}
```

The complete creation path must retain `agr tool create --persistent`. The Deployment must retain `EXCLUSIVE` affinity and `IdleAction: "PAUSE"`: `agr v0.6.6` describes persistent Tools as creating persistent sandboxes, describes `PAUSE` as preserving Instance state, and describes `EXCLUSIVE` as dedicating one non-migrating Instance per affinity ID ([Tool persistent flag](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/tool/create/api.generated.go#L116-L120), [Deployment lifecycle and affinity](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/create/api.generated.go#L59-L70)). These controls make the two `/home/sandbox` paths candidates for pause/resume continuity; only the live Shanghai journey can prove the service preserves their exact bytes.

### Session selection

Use the native Sandbox Agent TypeScript SDK session path:

```ts
const session = await client.createSession({
  agent: "codex",
  mode: "auto",
  cwd: "/home/sandbox/workspace",
  // model deliberately omitted
});
```

The SDK request type makes `agent` required and `model`/`mode` optional, and `createSession` explicitly calls `setSessionMode` when mode is present while calling `setSessionModel` only when model is present ([request type](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/sdks/typescript/src/client.ts#L168-L178), [create behavior](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/sdks/typescript/src/client.ts#L1123-L1159)). Omit `model` so the native Codex process loads `model = "hy3"` and `model_provider = "hy3-tokenhub"` from `CODEX_HOME/config.toml`, as required by the [provider decision](https://github.com/tlipoca9/ags-cookbook/blob/2aff9ac380677e612ae2baeddb8ff3bb976dfb9a/research/sandbox-agent/agent-provider.md#session-creation-behavior). `auto` is required because the acceptance journey writes and tests files; the default read-only mode cannot satisfy it.

The pinned OpenCode-compatible HTTP layer does not implement the same contract. When it creates the backing session, it hard-codes `agent_mode: None` ([backing-session creation](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/opencode_compat.rs#L511-L540)). On message submission it normalizes the request's `agent` field and stores it as `last_agent` metadata, but the eventual `ensure_backing_session` call still cannot apply that value as the backing session's mode ([message handling](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/opencode_compat.rs#L4128-L4173), [runtime metadata and backing call](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/opencode_compat.rs#L4219-L4244)).

For completeness, an `/opencode` prompt model selector of `{"providerID":"codex","modelID":"codex"}` resolves the backing agent to Codex while `backing_model_for_agent` returns `None`, so it does not force a Codex model override ([agent resolution](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/opencode_compat.rs#L1047-L1076), [model sentinel behavior](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/opencode_compat.rs#L1185-L1201)). That sentinel solves only agent selection and model omission; it does not solve `auto` mode. Therefore the [HTTP/SSE decision](https://github.com/tlipoca9/ags-cookbook/blob/d44eb69a76af8a26fb635257362300246b83554f/research/sandbox-agent/http-sse.md) remains useful for transport behavior but cannot be composed into the final golden path unchanged.

Switching to the native SDK is also not a drop-in persistence substitution: its client accepts a `SessionPersistDriver` and writes the session record through that driver ([start option](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/sdks/typescript/src/client.ts#L155-L166), [session persistence call](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/sdks/typescript/src/client.ts#L1132-L1147)). The server-side OpenCode SQLite database does not by itself satisfy native-SDK client persistence. The transport follow-up must therefore decide the event and persistence mechanism together, not only change one request body.

## Why the persistence paths are coupled

Sandbox Agent's OpenCode adapter defaults its SQLite database to `/tmp/sandbox-agent-opencode.db`, but first checks `OPENCODE_COMPAT_DB_PATH`; it creates the database if missing and enables WAL and foreign keys ([database selection](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L644-L670)). On initialization it migrates the database and rebuilds its in-memory session/message projection from the `sessions`, metadata and event rows ([rebuild source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L200-L305)). Therefore the SQLite file is necessary for compatibility-session recovery, while `/home/sandbox/workspace` is separately necessary for recovering the actual user files.

Both live under the image user's writable home so the runtime does not need root, a downstream `VOLUME`, or an assumed `/workspace` directory. Recovering only one path is a failure:

- database without workspace yields a remembered session whose files are gone;
- workspace without database yields files with no recoverable OpenCode-compatible session projection.

The acceptance check must resume the same affinity-owned Instance, look up the captured session id, read its pre-pause messages, and compare exact workspace bytes before sending the follow-up. Health or an idle status alone does not prove either persistence contract.

## Codex `0.116.0` compatibility result

**Accepted and locally route-verified.** The selected configuration is not merely compatible with a newer Codex probe; it is accepted by the exact Codex binary frozen in the approved image.

At OpenAI Codex tag `rust-v0.116.0` (commit `38771c9082535aa16b4c4d0395d3532f32f656ff`):

- `ConfigToml` accepts `model`, `model_provider`, and user-defined `model_providers` ([configuration type](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/config/mod.rs#L1193-L1204), [provider map](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/config/mod.rs#L1310-L1313)).
- `hy3-tokenhub` is a valid non-reserved provider id and the selected provider is resolved from the user-defined map ([validation and selection](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/config/mod.rs#L1959-L1987), [resolution](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/config/mod.rs#L2377-L2397)).
- `ModelProviderInfo` accepts `base_url`, `env_key`, and `wire_api`; `responses` is the only accepted wire value and `chat` is rejected ([provider type and wire parser](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/model_provider_info.rs#L31-L130)).
- The provider reads a non-empty API key from the named environment variable instead of requiring it in the file ([API-key lookup](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/model_provider_info.rs#L193-L212)).
- `CODEX_HOME` selects the configuration directory and must already exist, which is why the bootstrap creates it before starting Sandbox Agent ([home resolution](https://github.com/openai/codex/blob/38771c9082535aa16b4c4d0395d3532f32f656ff/codex-rs/core/src/config/mod.rs#L2957-L2966)).

### Exact-image probe

A credential-free local probe used the official image by its approved amd64 child digest. It first verified:

```text
codex-cli 0.116.0
```

It then mounted the exact TOML above as `CODEX_HOME/config.toml`, set `HY3_API_KEY` to the literal placeholder `not-a-real-credential`, and ran one non-writing `codex exec` request. Codex reported:

```text
model: hy3
provider: hy3-tokenhub
```

and the intentionally invalid credential received HTTP `401` from:

```text
https://tokenhub.tencentmaas.com/v1/responses
```

This proves the image-pinned binary parses the selected configuration, resolves the custom provider and model, reads the requested environment variable, and constructs the required Responses route. It also closes the earlier version gap: the previous provider report's probe used Codex `0.144.5`, while this probe used the immutable image's exact `0.116.0` binary.

No real key was used, printed, written to the repository, or retained. Consequently the probe does **not** prove authenticated `hy3` output, Codex tool calls through Sandbox Agent ACP, or TokenHub behavior after the initial request.

The probe also emitted `Model metadata for 'hy3' not found. Defaulting to fallback metadata; this can degrade performance and cause issues.` Configuration acceptance is decided, but context-window, compaction and token-budget behavior remain a live compatibility gate.

## Rejected alternatives

| Alternative | Why rejected |
| --- | --- |
| Add `config.toml` in a downstream Docker layer | Violates the immutable mirrored-image decision and creates a second image identity for non-secret data that can be supplied at runtime. |
| Put TOML in `Env` and ask Codex to parse it | Codex `0.116.0` loads `CODEX_HOME/config.toml`; no source-backed inline-config environment contract was found. |
| Put `HY3_API_KEY` in the TOML, bootstrap script, `Args`, or image environment | Creates durable disclosure in files, Tool argv/config literals, Git or image metadata; `env_key` exists specifically to keep the credential out of provider configuration. |
| Use direct `Args` containing `$HY3_API_KEY` | AGS exposes exec-style string arrays and no interpolation contract. An explicit shell would expand it into observable process/script state, which is unnecessary and unsafe. |
| Leave OpenCode state in `/tmp` | The adapter's default path is not the selected pause/resume durability seam and can recover neither exact `/home/sandbox` workspace bytes nor a documented persistent database location. |
| Use `/workspace` | The immutable image creates and runs as `sandbox` in `/home/sandbox`; `/workspace` is not part of the pinned image contract. |
| Set a real TokenHub/OpenAI model id as a session or OpenCode backing-model override | Risks validation against Sandbox Agent's static Codex catalog and contradicts the provider decision to select `hy3` inside native Codex configuration. The `codex`/`codex` compatibility sentinel documented above is not a real model override. |
| Use the image default command implicitly | Hides the provisional auth choice, cannot create the runtime config/state directories, and relies on unverified image command/argument merge behavior. |

## Residual validation gates

The route is specified, but promotion to a public tutorial still requires the Shanghai prototype to establish:

1. **Actual AGS exec semantics:** with non-secret canaries, prove the Tool starts `/bin/bash -eu -c <script>`, Sandbox Agent becomes PID 1 with the exact literal server arguments, and `CODEX_HOME`/`OPENCODE_COMPAT_DB_PATH` reach the relevant child/runtime. The CLI and SDK prove the arrays' shape, not the service's final OCI merge behavior.
2. **Authenticated provider journey:** use a purpose-created Guangzhou TokenHub key to complete a real `codex`/`auto` turn through the native Sandbox Agent session path, including a deterministic file edit and test. Do not treat the successful 401 route probe as authenticated compatibility.
3. **`hy3` metadata behavior:** measure normal and longer turns under Codex's fallback model metadata; verify context, compaction, tool calling and error handling are acceptable for the documented scenario.
4. **Persistence:** under `--persistent`, `EXCLUSIVE` affinity and `PAUSE`, prove the exact SQLite database (including WAL-managed state) and `/home/sandbox/workspace` bytes survive, then recover the same session and perform an incremental follow-up.
5. **Gateway/auth boundary:** prove missing or invalid `X-Access-Token` fails, loopback `agr deployment proxy` succeeds, and no route bypasses the gateway to port `2468`. Otherwise `--no-token` remains blocked.
6. **Secret visibility and cleanup:** verify Tool read/redaction behavior with a synthetic canary, capture only allowlisted evidence, delete Deployment and Tool, revoke the provider key, and confirm no secret-bearing files or logs remain.
7. **Transport-path decision:** reconcile the native SDK's correct `codex`/`auto` behavior with the desired raw HTTP/SSE tutorial, including where session records and event cursors persist. The acceptable outcomes are to use the native session path with an evidenced `SessionPersistDriver` and event contract, pin a later unmodified upstream image whose HTTP compatibility layer applies the mode, or keep the scenario blocked. A downstream patch and a false claim that OpenCode `agent: "auto"` changed Codex mode are both outside this decision.
8. **Resource sizing:** validate CPU, memory and local storage values against the large immutable image, SQLite WAL behavior and the two-turn workspace; do not fill the placeholders from guesswork.

No remaining source-level uncertainty blocks the immutable-image runtime composition itself. The end-to-end scenario remains blocked by the native-session versus raw-HTTP transport decision plus empirical Deployment, credential, lifecycle and sizing checks.

## Validation performed

```bash
# Canonical ticket and decision inputs
gh issue view 21 --repo tlipoca9/ags-cookbook --json number,title,body,url,labels,assignees,state,comments
git show 2aff9ac380677e612ae2baeddb8ff3bb976dfb9a:research/sandbox-agent/agent-provider.md
git show 9b97c380511563eca66b6ed36f91212456a5ac9f:research/sandbox-agent/ccr-image.md
git show 978b010fcb262d01c6f3c5142a99bdff866721e2:research/sandbox-agent/auth-boundary.md
git show d44eb69a76af8a26fb635257362300246b83554f:research/sandbox-agent/http-sse.md

# Pinned first-party source
git show d55b0dfb887cc92152f20f756995317c5f5c7709:docker/runtime/Dockerfile.full
git show d55b0dfb887cc92152f20f756995317c5f5c7709:server/packages/opencode-adapter/src/lib.rs
git show d55b0dfb887cc92152f20f756995317c5f5c7709:server/packages/sandbox-agent/src/cli.rs
git show refs/tags/rust-v0.116.0:codex-rs/core/src/model_provider_info.rs
git show refs/tags/rust-v0.116.0:codex-rs/core/src/config/mod.rs
git show 67f83b4b08037483e8a3b3c3000e788901b1c2ed:internal/commands/tool/create/api.generated.go
git show 67f83b4b08037483e8a3b3c3000e788901b1c2ed:internal/commands/deployment/create/api.generated.go

# Exact-image, no-real-credential compatibility probe
podman pull --platform linux/amd64 \
  docker.io/rivetdev/sandbox-agent@sha256:a7b9afc8c79fb075852d3ae73b86ae201b5ce683b1bf5f11988980cd2bca09a5
podman run --rm --platform linux/amd64 \
  --entrypoint /home/sandbox/.local/share/sandbox-agent/bin/codex \
  docker.io/rivetdev/sandbox-agent@sha256:a7b9afc8c79fb075852d3ae73b86ae201b5ce683b1bf5f11988980cd2bca09a5 \
  --version

# A temporary CODEX_HOME contained the exact TOML from this report.
# HY3_API_KEY was the literal placeholder "not-a-real-credential".
# `codex exec` selected hy3/hy3-tokenhub and received expected HTTP 401
# from https://tokenhub.tencentmaas.com/v1/responses.

# The exact bootstrap script was also launched against the same image digest.
# Health, uid/gid, cwd, file modes, literal server args, and creation of the
# selected SQLite/WAL files were checked before the disposable container stopped.
```

## Primary-source inventory

- [Canonical provider decision](https://github.com/tlipoca9/ags-cookbook/blob/2aff9ac380677e612ae2baeddb8ff3bb976dfb9a/research/sandbox-agent/agent-provider.md)
- [Canonical immutable-image decision](https://github.com/tlipoca9/ags-cookbook/blob/9b97c380511563eca66b6ed36f91212456a5ac9f/research/sandbox-agent/ccr-image.md)
- [Canonical authentication decision](https://github.com/tlipoca9/ags-cookbook/blob/978b010fcb262d01c6f3c5142a99bdff866721e2/research/sandbox-agent/auth-boundary.md)
- [Canonical HTTP/SSE decision](https://github.com/tlipoca9/ags-cookbook/blob/d44eb69a76af8a26fb635257362300246b83554f/research/sandbox-agent/http-sse.md)
- [Sandbox Agent `v0.4.2` source](https://github.com/rivet-dev/sandbox-agent/tree/d55b0dfb887cc92152f20f756995317c5f5c7709)
- [OpenAI Codex `rust-v0.116.0` source](https://github.com/openai/codex/tree/38771c9082535aa16b4c4d0395d3532f32f656ff)
- [`agr` Tool and Deployment command schema snapshot](https://github.com/TencentCloudAgentRuntime/ags-cli/tree/67f83b4b08037483e8a3b3c3000e788901b1c2ed)
- [Tencent Cloud TokenHub Codex guide](https://cloud.tencent.com/document/product/1823/133532)
