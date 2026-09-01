# Raw HTTP and SSE golden path for Sandbox Agent on AGS

Status: **candidate sequence specified; two release/runtime blockers require Shanghai validation before it can be published as reliable**

This report answers [“Specify the raw HTTP and SSE golden path”](https://github.com/tlipoca9/ags-cookbook/issues/4). It targets Sandbox Agent `v0.4.2` (`d55b0dfb887cc92152f20f756995317c5f5c7709`) and the locally inspected `agr` `v0.6.6` (`67f83b4b08037483e8a3b3c3000e788901b1c2ed`). No cloud Deployment or credential was used.

## Decision summary

Use two first-party HTTP surfaces on one Sandbox Agent server:

1. `/v1/health`, `/v1/agents`, and `/v1/fs/file` for server readiness, real agent discovery, and file evidence.
2. `/opencode/global/health`, `/opencode/session`, `/opencode/event`, `/opencode/session/status`, and `/opencode/project/current` for the OpenCode-compatible session and event flow.

Create the session once with `POST /opencode/session`, save the returned `id`, and always address that ID directly. Open SSE **before** submitting each turn. A turn succeeds only after a matching `session.idle` event, no matching `session.error`, and successful post-turn message/file inspection. The server does not end the event stream on turn completion; the client must close it after the terminal event so AGS can become idle and pause.

Do not use `/opencode/agent` as proof that a requested coding-agent binary is installed: Sandbox Agent `v0.4.2` returns one compatibility placeholder named `Sandbox Agent`. Use `GET /v1/agents?config=true` and match the parameterized `$AGENT_ID` instead.

Do not claim durable SSE offsets. `Last-Event-ID` is an implementation-only, best-effort replay cursor into a process-local 4,096-event ring. It is suitable for a transient reconnect to the same running server process, not as the recovery mechanism across an AGS lifecycle boundary. Across pause/resume, recover from the captured session ID and the HTTP read models, then open a fresh stream.

Two facts currently prevent calling the sequence unconditionally reliable through `agr deployment proxy`:

- `POST /opencode/session/{id}/prompt_async` is not actually asynchronous in release `v0.4.2`: it awaits the synchronous prompt handler, ignores that handler’s response/error, and only then returns `204`.
- `agr deployment proxy` `v0.6.6` waits at most 30 seconds for upstream response headers. The synchronous real-agent prompt waits for the ACP turn response. A turn longer than 30 seconds can therefore surface locally as an ambiguous `502 Bad Gateway` even while SSE may have received progress.

The Shanghai prototype must validate a real selected-provider turn below 30 seconds or move to a Sandbox Agent build with a genuinely fire-and-forget prompt endpoint (and preserved error observability). It must not document `prompt_async` as the workaround on `v0.4.2`.

## Contract and evidence map

### Canonical Sandbox Agent OpenAPI

The release OpenAPI is [`docs/openapi.json` at `v0.4.2`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/openapi.json). Relevant operations and schemas are:

| Operation | Method/path | Request | Success evidence |
| --- | --- | --- | --- |
| `get_v1_health` | `GET /v1/health` | none | `HealthResponse`, required string `status`; release handler returns `{"status":"ok"}` |
| `get_v1_agents` | `GET /v1/agents?config=true` | optional `config`, `no_cache` query booleans | `AgentListResponse.agents[]`; each `AgentInfo` requires `id`, `installed`, `credentialsAvailable`, `capabilities` |
| `get_v1_fs_file` | `GET /v1/fs/file?path=...` | required string query `path` | `200` file bytes |

The release router mounts `/v1` and `/opencode` separately and gives the OpenCode adapter the same server token ([`router.rs` lines 313–337](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router.rs#L313-L337)). The health implementation fixes the healthy value to `ok` ([`router.rs` lines 637–648](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router.rs#L637-L648)).

### OpenCode-compatible OpenAPI

Sandbox Agent documents `/opencode` as experimental and lists session, message, and SSE coverage ([OpenCode compatibility documentation](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/opencode-compatibility.mdx)). The official generated OpenCode artifact was removed from the tree before `v0.4.2`; the last first-party artifact remains available at [`resources/agent-schemas/artifacts/openapi/opencode.json`](https://github.com/rivet-dev/sandbox-agent/blob/ef3e811c94b792415f5a41ff3fadb17518772f9d/resources/agent-schemas/artifacts/openapi/opencode.json). Release source still mounts these routes ([`opencode-adapter/src/lib.rs` lines 697–762](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L697-L762)).

Relevant generated operations/schemas:

| Operation ID | Method/path | Schema detail used here |
| --- | --- | --- |
| `global.health` | `GET /global/health` | JSON object; release source returns `healthy: true` and `version` |
| `app.agents` | `GET /agent` | `Agent[]` (compatibility metadata only) |
| `project.current` | `GET /project/current?directory=...` | `Project`, requiring `id`, `worktree`, `time`, `sandboxes` in the artifact |
| `session.create` | `POST /session?directory=...` | optional `title`, `parentID`, `permission`; returns `Session` |
| `session.get` | `GET /session/{sessionID}` | returns `Session`; `404` is specified |
| `session.list` | `GET /session` | returns `Session[]` |
| `session.status` | `GET /session/status` | object keyed by session ID, values `SessionStatus` (`idle`, `busy`, or `retry`) |
| `event.subscribe` | `GET /event?directory=...` | `text/event-stream`, each data JSON matches `Event` |
| `session.prompt` | `POST /session/{sessionID}/message` | body requires `parts[]`; optional `messageID`, `model`, `agent`, `system`, `variant`; returns `{info: AssistantMessage, parts: Part[]}` |
| `session.prompt_async` | `POST /session/{sessionID}/prompt_async` | same prompt body; specifies `204` |
| `session.messages` | `GET /session/{sessionID}/message` | returns `[{info: Message, parts: Part[]}]` |

`TextPartInput` requires `{"type":"text","text":"..."}`. The model selector requires both `providerID` and `modelID`. `Session` requires `id`, `slug`, `projectID`, `directory`, `title`, `version`, and `time`. The completion and failure event schemas are:

```json
{"type":"session.idle","properties":{"sessionID":"<session-id>"}}
{"type":"session.error","properties":{"sessionID":"<session-id>","error":{}}}
```

`message.updated` carries `properties.info`; `message.part.updated` carries `properties.part` and optionally `properties.delta`.

There are release-source divergences that the tutorial must respect:

- `/opencode/agent` is a fixed placeholder, not installed-agent discovery ([lines 897–916](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L897-L916)).
- `/opencode/project/current` is compatibility metadata whose project ID is generated when the adapter process starts ([lines 1427–1447](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L1427-L1447)); it is not durable project proof.
- `session.init` can create an absent client-named session in release source, but its generated description says “Initialize session,” not “create session,” and its generated required fields disagree with the optional runtime body. This report therefore does not use it as an undocumented deterministic-create shortcut.
- `/session/{id}/diff` and `/file/content` are stub-like in this release. Use canonical `GET /v1/fs/file` for result evidence.

## Access modes and required headers

Assume the Sandbox Agent server token is enabled. If the credential/authentication ticket chooses a no-token server, omit only the `Authorization` header; the AGS data-plane credential remains required in production.

### Local tutorial access through `agr deployment proxy`

Start the proxy in its own terminal:

```bash
export AGR_REGION=ap-shanghai
export DEPLOYMENT_ID='dpl-replace-me'
export LOCAL_PORT=18080
export SANDBOX_AGENT_PORT=2468

agr deployment proxy "$DEPLOYMENT_ID" \
  "$LOCAL_PORT:$SANDBOX_AGENT_PORT" \
  --region "$AGR_REGION"
```

Use `BASE_URL=http://127.0.0.1:18080`. The caller sends:

- `Authorization: Bearer $SANDBOX_AGENT_TOKEN` when Sandbox Agent auth is enabled;
- `Accept: application/json` for JSON reads;
- `Content-Type: application/json` and `Accept: application/json` for JSON writes;
- `Accept: text/event-stream` and `Cache-Control: no-cache` for SSE.

Do **not** send `X-Access-Token` or the affinity header to the local proxy. The CLI acquires Deployment tokens, replaces `X-Access-Token`, captures the configured affinity response header, and reuses it. It preserves application `Authorization` headers. The first successful request binds an exclusive affinity session; copy the `Affinity ID: ...` line printed by the proxy.

This is first-party CLI behavior, not an assumption: the command promises HTTP/SSE/WebSocket forwarding and affinity capture/reuse ([`command.go` lines 40–68](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go#L40-L68)); the proxy injects `X-Access-Token` while preserving business headers ([`proxy.go` lines 138–167](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L138-L167)); and its test asserts simultaneous `X-Access-Token`, `Authorization`, and SSE forwarding ([`deployment_test.go` lines 38–122](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/deployment_test.go#L38-L122)).

### Production direct-domain access

The proxy is local-debug tooling, not production ingress. Acquire a short-lived Deployment token, then call:

```text
https://2468-{deployment-id}.{region}.agents.{data-plane-domain}
```

With the repository default domain, this is:

```text
https://2468-{deployment-id}.ap-shanghai.agents.tencentags.com
```

Every direct request needs:

```http
X-Access-Token: <short-lived Deployment token>
X-Tencent-Agr-Affinity-Id: <captured exclusive affinity ID>
Authorization: Bearer <Sandbox Agent server token>   # if enabled
```

The first direct request omits the affinity header and captures it from the response; all subsequent requests return it. Acquire the Deployment token with:

```bash
agr api call AcquireDeploymentToken \
  --region "$AGR_REGION" \
  --request '{"DeploymentId":"'"$DEPLOYMENT_ID"'"}' \
  --output json
```

The data-plane domain, `X-Access-Token` header, and token response shape are established by this repository’s [deployment simple example](../../examples/deployment-cookbook/httpbin/simple/README.md#5-access-the-production-data-plane). The default affinity header and exclusive reuse convention are established by the [affinity example](../../examples/deployment-cookbook/httpbin/affinity/README.md#6-exclusive-one-dedicated-instance-per-affinity-id).

## Ordered candidate golden path

The examples below show local proxy URLs. Add the direct-domain headers above for production. All provider-specific values remain parameters until the baseline-agent/provider ticket is resolved.

### 0. Set non-secret parameters

```bash
export BASE_URL='http://127.0.0.1:18080'
export SANDBOX_AGENT_TOKEN='replace-with-runtime-server-token'
export AGENT_ID='replace-with-selected-agent-id'
export PROVIDER_ID='replace-with-selected-opencode-provider-id'
export MODEL_ID='replace-with-selected-model-id'
export WORKDIR='/workspace'
export PROJECT_DIR='/workspace/http-sse-golden'
export SESSION_TITLE='AGS raw HTTP SSE golden path'
export FIRST_MESSAGE_ID='msg_ags_phase_1'
export FOLLOWUP_MESSAGE_ID='msg_ags_phase_2'
```

Never place real token values in command literals, Git, screenshots, captured response bodies, or the report. The examples use the environment only.

### 1. Establish affinity and verify both API surfaces

Canonical readiness:

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/v1/health" | tee health.json

jq -e '.status == "ok"' health.json
```

Compatibility readiness:

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/opencode/global/health" | tee opencode-health.json

jq -e '.healthy == true and (.version | type == "string")' opencode-health.json
```

The first response should cause the proxy terminal to print the affinity ID. Save it outside Git:

```bash
export AFFINITY_ID='replace-with-proxy-output'
```

Failure evidence: `401` means the Sandbox Agent bearer token is absent/wrong; `502` before a response header is a proxy/cold-start/upstream failure; a non-JSON success response means the wrong port/image/path.

### 2. Discover the real agent and record compatibility metadata

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/v1/agents?config=true" | tee agents.json

jq -e --arg id "$AGENT_ID" '
  [.agents[] | select(.id == $id and .installed == true)] | length == 1
' agents.json
```

Whether `credentialsAvailable` must be true depends on the selected credential path; assert it only after that ticket defines the contract.

Record, but do not use as installation proof, the OpenCode compatibility agent list:

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/opencode/agent" | tee opencode-agents.json
```

### 3. Inspect the project context and create exactly one session

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  --url-query "directory=$WORKDIR" \
  "$BASE_URL/opencode/project/current" | tee project-before.json

jq -e --arg workdir "$WORKDIR" '.worktree == $workdir' project-before.json
```

Create once and capture the returned ID directly; never select “the newest” session:

```bash
jq -n --arg title "$SESSION_TITLE" '{title: $title}' > create-session.json

curl --fail-with-body --silent --show-error \
  --request POST \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --url-query "directory=$WORKDIR" \
  --data-binary @create-session.json \
  "$BASE_URL/opencode/session" | tee session.json

export SESSION_ID="$(jq -er '.id | select(startswith("ses_"))' session.json)"
jq -e --arg id "$SESSION_ID" --arg title "$SESSION_TITLE" --arg dir "$WORKDIR" \
  '.id == $id and .title == $title and .directory == $dir' session.json
```

“Deterministic” here means one non-retried creation request and immediate capture of its returned ID. `session.create` has no client idempotency key. If the response is lost, do not blindly retry: list by exact title and directory, require exactly one match, and stop for manual reconciliation if zero or multiple sessions match.

`--url-query` URL-encodes `directory` without moving `--data-binary` out of the JSON request body. It requires curl 7.87 or newer; an explicitly URL-encoded query is equivalent.

### 4. Open SSE before submitting the first turn

Start this in a separate terminal or background process:

```bash
curl --fail-with-body --silent --show-error --no-buffer \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: text/event-stream' \
  --header 'Cache-Control: no-cache' \
  --url-query "directory=$WORKDIR" \
  "$BASE_URL/opencode/event" | tee events-phase-1.sse
```

Wait until a complete SSE record contains `{"type":"server.connected"...}` before submitting. The wire form is:

```text
id: <decimal-runtime-event-id>
data: {"type":"server.connected","properties":{}}

```

For each later record, persist the last complete decimal `id:` separately from `data:`. Heartbeats and keepalive comments may have no ID and must not advance the cursor.

### 5. Submit one real, deterministic coding task

Recommended request body, with the selected model injected by `jq`:

```bash
jq -n \
  --arg messageID "$FIRST_MESSAGE_ID" \
  --arg providerID "$PROVIDER_ID" \
  --arg modelID "$MODEL_ID" \
  --arg projectDir "$PROJECT_DIR" \
  '{
    messageID: $messageID,
    model: {providerID: $providerID, modelID: $modelID},
    parts: [{
      type: "text",
      text: ("Create " + $projectDir + " as a small Python project. Implement counter.py with a tested next_value(integer) function, add unittest coverage, and run python -m unittest -v. Only after tests pass, write " + $projectDir + "/result.json as exactly {\"phase\":1,\"value\":2,\"tests\":\"passed\"}. Do not modify files outside that project directory.")
    }]
  }' > prompt-phase-1.json
```

The contract-correct submission is:

```bash
curl --fail-with-body --silent --show-error \
  --request POST \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --url-query "directory=$WORKDIR" \
  --data-binary @prompt-phase-1.json \
  "$BASE_URL/opencode/session/$SESSION_ID/message" | tee prompt-phase-1-response.json
```

Do not automatically retry this POST after a timeout or `502`: `messageID` is useful correlation, but no first-party source states that it makes prompt execution idempotent.

`POST .../prompt_async` is deliberately excluded from the golden command. Although operation `session.prompt_async` says it returns immediately, release source awaits `oc_session_prompt`, ignores its result, and only then returns `204` ([lines 2647–2656](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L2647-L2656)).

### 6. Detect completion without treating stream closure as success

Process SSE records in order and filter by the exact `$SESSION_ID`:

1. `session.status` with `properties.status.type == "busy"` is progress, not success.
2. `message.part.updated` can be rendered or logged; `delta` is an incremental chunk when present.
3. `permission.asked` or `question.asked` means the unattended golden path is blocked. Do not wait forever; record the request and fail this scenario unless the provider ticket deliberately adds an interaction policy.
4. A matching `session.error` is a failed turn, even if an `idle` follows.
5. The first matching `session.idle` after this message submission is the turn-completion marker.
6. EOF, a transport error, heartbeat, `server.connected`, HTTP `200`, or the message POST response is not the SSE completion marker.

Release source emits `session.status` and then `session.idle` when status becomes idle ([lines 2999–3037](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L2999-L3037)). Its real-agent integration test opens the stream before prompting, rejects stream EOF before idle, fails on `session.error`, and treats `session.idle` as terminal ([`real-agent.test.ts` lines 57–113](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/tests/opencode-compat/real-agent.test.ts#L57-L113)).

After terminal idle/error, stop the SSE `curl`. The endpoint is intentionally long-lived: it emits JSON heartbeat data at 30 seconds and SSE keepalive comments at 15 seconds, and closes only when its broadcaster closes ([lines 984–1040](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L984-L1040)). Leaving it open keeps an active Deployment connection and prevents a clean idle-pause observation.

### 7. Inspect the session and concrete result

First require HTTP status idle:

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/opencode/session/status" | tee status-phase-1.json

jq -e --arg id "$SESSION_ID" '.[$id].type == "idle"' status-phase-1.json
```

Then inspect persisted messages and require both the correlated user message and a nonempty assistant result:

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/opencode/session/$SESSION_ID/message" | tee messages-phase-1.json

jq -e --arg messageID "$FIRST_MESSAGE_ID" '
  any(.[]; .info.id == $messageID and .info.role == "user") and
  any(.[]; .info.role == "assistant" and ([.parts[]? | select(.type == "text") | .text] | join("") | length > 0))
' messages-phase-1.json
```

Finally inspect real file bytes through canonical `get_v1_fs_file`:

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/octet-stream' \
  --url-query "path=$PROJECT_DIR/result.json" \
  "$BASE_URL/v1/fs/file" | tee result-phase-1.json

jq -e '. == {phase:1, value:2, tests:"passed"}' result-phase-1.json
```

The assistant’s prose is supporting evidence; the exact result file is the acceptance evidence. A reported idle status alone is insufficient after a crash/restart because the adapter rebuild initializes loaded sessions as idle before replaying durable envelopes.

### 8. Close every connection and observe AGS pause

Stop the SSE curl and the local proxy. Wait at least the configured idle timeout, then inspect Instances with the same command used by existing deployment examples:

```bash
agr instance list --tool-id "$SANDBOX_AGENT_TOOL_ID" --region "$AGR_REGION"
```

Require the same affinity-owned Instance to reach `PAUSED`. Do not poll through the Deployment data-plane endpoint, because each request can reset idle accounting or resume the Instance.

### 9. Resume the same affinity, then look up the captured session ID

Restart the proxy explicitly with the saved affinity ID:

```bash
agr deployment proxy "$DEPLOYMENT_ID" \
  "$LOCAL_PORT:$SANDBOX_AGENT_PORT" \
  --region "$AGR_REGION" \
  --affinity-id "$AFFINITY_ID"
```

Call readiness to wake the paused exclusive Instance, then address the captured ID directly:

```bash
curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/v1/health" | jq -e '.status == "ok"'

curl --fail-with-body --silent --show-error \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: application/json' \
  "$BASE_URL/opencode/session/$SESSION_ID" | tee session-after-resume.json

jq -e --arg id "$SESSION_ID" --arg title "$SESSION_TITLE" \
  '.id == $id and .title == $title' session-after-resume.json
```

Also repeat the phase-one message and file reads before sending a follow-up. This distinguishes real session/workspace recovery from merely creating a fresh healthy server.

Sandbox Agent stores OpenCode session metadata and event envelopes in SQLite and rebuilds its projection at initialization ([lines 200–305](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L200-L305)). The default database is `/tmp/sandbox-agent-opencode.db`; `OPENCODE_COMPAT_DB_PATH` or `OPENCODE_COMPAT_STATE` can relocate it ([lines 644–670](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L644-L670)). The image/deployment contract must place this database on storage that the chosen AGS `PAUSE` path demonstrably preserves.

### 10. Open a fresh stream and submit one incremental follow-up

Do not carry a pre-pause `Last-Event-ID` into this lifecycle test. Open a fresh stream, wait for `server.connected`, and submit:

```bash
jq -n \
  --arg messageID "$FOLLOWUP_MESSAGE_ID" \
  --arg providerID "$PROVIDER_ID" \
  --arg modelID "$MODEL_ID" \
  --arg projectDir "$PROJECT_DIR" \
  '{
    messageID: $messageID,
    model: {providerID: $providerID, modelID: $modelID},
    parts: [{
      type: "text",
      text: ("Continue the existing project in " + $projectDir + ". Add a tested increment_by(integer, step) function without rewriting next_value, run python -m unittest -v, and only after tests pass replace result.json with exactly {\"phase\":2,\"value\":5,\"tests\":\"passed\"}.")
    }]
  }' > prompt-phase-2.json
```

Use the same `POST /opencode/session/$SESSION_ID/message` and completion rules. Then require:

- the same session ID and original phase-one messages remain;
- a user message with ID `$FOLLOWUP_MESSAGE_ID` exists;
- a later assistant message has nonempty content;
- `$PROJECT_DIR/result.json` is exactly `{"phase":2,"value":5,"tests":"passed"}`;
- source/tests still include `next_value` and now include `increment_by`.

This proves incremental continuity, not just filesystem existence.

## SSE reconnect and offset rules

### Same running process

On an unexpected transport disconnect before a terminal event:

1. Save only the last fully parsed numeric `id:`.
2. Reconnect promptly with the same URL, auth, affinity, and `Last-Event-ID: <id>`.
3. Deduplicate by numeric event ID.
4. Continue waiting for a matching `session.error` or `session.idle`.
5. Reconcile with `GET /opencode/session/status` and `GET /opencode/session/{id}/message` after reconnect.

Example:

```bash
curl --fail-with-body --silent --show-error --no-buffer \
  --header "Authorization: Bearer $SANDBOX_AGENT_TOKEN" \
  --header 'Accept: text/event-stream' \
  --header 'Cache-Control: no-cache' \
  --header "Last-Event-ID: $LAST_EVENT_ID" \
  --url-query "directory=$WORKDIR" \
  "$BASE_URL/opencode/event"
```

The implementation parses a decimal `Last-Event-ID` and silently treats invalid values as absent ([lines 3607–3612](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L3607-L3612)). It returns retained events whose IDs are strictly greater than the cursor ([lines 308–335](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L308-L335)).

### Limits: no durable or gap-detecting cursor

- The replay log contains at most 4,096 events and lives in memory.
- Event numbering starts from `1` when the adapter process starts.
- Durable SQLite event envelopes rebuild messages/status, but are not used to rebuild the SSE ring or its IDs.
- If the requested ID has fallen out of the ring, the server sends whatever newer retained events exist and does not emit an explicit replay-gap error.
- The release adapter computes replay before subscribing its live receiver, leaving a small unacknowledged subscribe race; clients must reconcile HTTP read models.
- Heartbeat events have no ID.

Therefore `Last-Event-ID` is best-effort loss reduction, not exactly-once delivery. Never use a stale pre-restart cursor as the sole pause/resume mechanism. The reliable lifecycle key is `(Deployment affinity ID, OpenCode session ID)` plus HTTP inspection.

## Failure modes and required handling

| Observation | Meaning/action |
| --- | --- |
| `/v1/health` `401` | Wrong/missing Sandbox Agent bearer token. Do not confuse it with the Deployment `X-Access-Token`. |
| Direct-domain gateway rejection | Missing/expired `X-Access-Token`, wrong Deployment token, or missing/wrong affinity. Reacquire the short-lived token; do not change affinity. |
| Proxy `502 Bad Gateway` during first request | Cold start or upstream failed to return headers within the proxy’s 30-second bound. Retry readiness only after checking Instance state. |
| Proxy `502` during message POST | Ambiguous execution because the prompt is not idempotent and may still have produced events. Do not retry automatically; inspect SSE/status/messages/files. |
| SSE response is not `text/event-stream` | Wrong route/port, auth/gateway error, or proxy failure. Treat as fatal. |
| SSE EOF before terminal event | Not completion. Reconnect with the last ID only to the same process, then reconcile HTTP state. |
| `session.error` for the target | Failed turn. Preserve the event and inspect status/messages; `idle` after error does not convert it to success. |
| `permission.asked` / `question.asked` | Golden path requires interaction. Fail unattended validation unless a reviewed reply policy exists. |
| Target absent from `/v1/agents` or `installed != true` | Wrong image or installation failure. `/opencode/agent` cannot override this evidence. |
| Session ID missing after resume | Wrong affinity, unpreserved SQLite path, fresh/replaced Instance, or adapter restoration failure. Do not create a replacement session. |
| Session exists but result file is missing | Session metadata persistence alone did not prove workspace persistence. Fail resume acceptance. |
| Project ID changed after resume | Expected if the adapter process restarted; project compatibility metadata is process-generated. Use session/file evidence. |
| Long-lived SSE remains open | Deployment has an active connection and may not reach idle pause. Close SSE and proxy before waiting. |

The `agr` proxy’s fixed response-header timeout and `502` error mapping are in [`proxy.go` lines 138–179](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go#L138-L179). Sandbox Agent explicitly blocks waiting for the real `session/prompt` response ([`opencode-adapter/src/lib.rs` lines 2202–2240](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs#L2202-L2240)).

## Unresolved live-validation points

These cannot be resolved from source without calling a real Deployment, which this ticket prohibited:

1. **Prompt duration versus proxy timeout:** measure first and follow-up turns with the selected real agent/model in `ap-shanghai`; verify whether each POST returns headers within 30 seconds. A `502` plus later idle is still not an acceptable documented success contract.
2. **Gateway behavior on a long POST:** confirm whether the production direct-domain gateway has its own response-header/body timeout and whether client disconnect cancels the Sandbox Agent request.
3. **Pause stream termination:** source proves the application stream is indefinite, but no first-party AGS source here specifies the exact socket/HTTP error seen if an Instance pauses or resumes while SSE is open. The tutorial should close SSE before pause rather than promise a particular termination signal.
4. **Pause storage semantics:** prove that the exact configured SQLite path and `/workspace` survive the Deployment’s `EXCLUSIVE + PAUSE` lifecycle on the selected image.
5. **Runtime memory versus restart:** observe whether event IDs continue or reset after the selected pause mode. The tutorial remains correct by reconnecting fresh and reconciling HTTP state either way.
6. **Affinity response timing:** confirm the configured affinity header is present on the first health response and that `agr deployment proxy` prints it before the tutorial asks the user to stop the proxy.
7. **Provider interaction events:** establish that the selected task does not produce permission/question requests, or specify and test an explicit safe reply flow in the provider ticket.
8. **OpenAPI drift:** the release canonical OpenAPI omits `/opencode`, while the last generated OpenCode artifact predates `v0.4.2`. Pin and archive the exact runtime contract used by the final image.

## Implications for the blocked Shanghai prototype

The prototype can proceed to live validation with the ordered sequence above, but the public tutorial remains blocked on message submission semantics:

- Pin the exact Sandbox Agent and `agr` versions during validation.
- Configure `EXCLUSIVE` affinity and `PAUSE`; capture the affinity ID from the first health request.
- Put `OPENCODE_COMPAT_DB_PATH` on a path proven to survive pause, rather than relying silently on `/tmp`.
- Run SSE before each prompt, filter terminal events by the captured session ID, and close SSE immediately after terminal detection.
- Use the synchronous `session.prompt` operation for observable HTTP errors, but record the 30-second proxy risk. Do not present `prompt_async` as immediate or error-preserving on `v0.4.2`.
- If a selected real task cannot reliably return message-response headers within 30 seconds, the implementation ticket needs an upstream pin containing a true async enqueue/`204` behavior or a narrowly reviewed downstream fix. Shortening the task until it “usually” fits is not a reliable protocol contract.
- Validate recovery by direct session lookup and exact file bytes before sending the follow-up; do not infer success from health, project metadata, or an idle status alone.

## Evidence assertions checklist

A successful validated run must retain masked evidence for all of the following:

- [ ] `agr version -o json` and pinned Sandbox Agent version/commit.
- [ ] Proxy reports the expected direct domain and one saved affinity ID.
- [ ] `/v1/health.status == "ok"` and `/opencode/global/health.healthy == true`.
- [ ] Exactly one `/v1/agents` item matches `$AGENT_ID` and is installed.
- [ ] Session-create response supplies the captured ID, expected title, and `/workspace` directory.
- [ ] SSE was connected before each prompt and shows target-specific busy/progress followed by exactly one terminal idle, with no target-specific error.
- [ ] Message read model contains each fixed user message ID and a nonempty assistant result.
- [ ] Phase-one result file has the exact expected JSON.
- [ ] All connections were closed and the same affinity-owned Instance reached `PAUSED`.
- [ ] Proxy restart used `--affinity-id`; direct lookup found the same session and phase-one file before follow-up.
- [ ] Phase-two result and source/tests prove an incremental change while preserving phase one.
- [ ] No real token, provider key, affinity ID, Deployment ID, or unmasked hostname-specific secret appears in committed evidence.

## Source inventory

- Sandbox Agent release commit: [`d55b0dfb887cc92152f20f756995317c5f5c7709`](https://github.com/rivet-dev/sandbox-agent/tree/d55b0dfb887cc92152f20f756995317c5f5c7709)
- Canonical Sandbox Agent OpenAPI: [`docs/openapi.json`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/openapi.json)
- OpenCode adapter implementation: [`server/packages/opencode-adapter/src/lib.rs`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/opencode-adapter/src/lib.rs)
- OpenCode compatibility documentation: [`docs/opencode-compatibility.mdx`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/opencode-compatibility.mdx)
- Last checked-in generated OpenCode OpenAPI: [`opencode.json` at `ef3e811`](https://github.com/rivet-dev/sandbox-agent/blob/ef3e811c94b792415f5a41ff3fadb17518772f9d/resources/agent-schemas/artifacts/openapi/opencode.json)
- Real-agent completion test: [`real-agent.test.ts`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/tests/opencode-compat/real-agent.test.ts)
- `agr` proxy implementation at inspected commit: [`internal/dataplane/proxy/proxy.go`](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/dataplane/proxy/proxy.go)
- `agr` Deployment proxy command: [`internal/commands/deployment/proxy/command.go`](https://github.com/TencentCloudAgentRuntime/ags-cli/blob/67f83b4b08037483e8a3b3c3000e788901b1c2ed/internal/commands/deployment/proxy/command.go)
- Existing cookbook proxy/direct-domain convention: [`httpbin/simple`](../../examples/deployment-cookbook/httpbin/simple/README.md) and [`httpbin/affinity`](../../examples/deployment-cookbook/httpbin/affinity/README.md)
