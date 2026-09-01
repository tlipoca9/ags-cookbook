# Reproducible CCR image contract for Sandbox Agent

**Research ticket:** [Define the reproducible CCR image contract](https://github.com/tlipoca9/ags-cookbook/issues/7)

**Evidence cutoff:** 2026-09-01

**Target:** AGS Deployment in `ap-shanghai`, `linux/amd64`

## Decision

Mirror the **official, already-built `linux/amd64` child manifest** of Sandbox Agent `v0.4.2` full into CCR. Do not rebuild `Dockerfile.full`, do not run the install script, and do not carry a downstream patch.

| Contract field | Required value |
| --- | --- |
| Upstream release | `v0.4.2` (latest stable release at the evidence cutoff; newer `v0.5.0-rc.*` releases are prereleases) [E1] |
| Upstream commit | `d55b0dfb887cc92152f20f756995317c5f5c7709` [E2] |
| Upstream source recipe | `docker/runtime/Dockerfile.full` at that commit [E3] |
| Authoritative source image | `docker.io/rivetdev/sandbox-agent@sha256:a7b9afc8c79fb075852d3ae73b86ae201b5ce683b1bf5f11988980cd2bca09a5` (`linux/amd64` child, not the tag or multi-platform index) [E4] |
| Discovery tag/index only | `rivetdev/sandbox-agent:0.4.2-full` / `sha256:e356e73cbf2c2bd318e052f91d8180a1d54a9a97672384b9eb5476929c945d7e`; these are not the deployment pin [E4] |
| CCR publication tag | `ccr.ccs.tencentyun.com/ags.dev/sandbox-agent:v0.4.2-full-amd64` |
| Platform | exactly `linux/amd64` [E4] |
| Runtime user | `sandbox` (`uid=1001`, `gid=1001` in the inspected image) [E3, E8] |
| `HOME` / workdir | `/home/sandbox` [E3, E5] |
| Entrypoint | `['sandbox-agent']` [E3, E5] |
| Image-default arguments | `['server', '--host', '0.0.0.0', '--port', '2468']` [E3, E5] |
| Port | `2468/tcp` [E3, E5] |
| Readiness endpoint | unauthenticated image default: `GET /v1/health`, require HTTP 200 and exact JSON `{"status":"ok"}` [E6, E8] |
| Image-level `HEALTHCHECK` | none; configure the AGS probe against `/v1/health` [E5] |
| Downstream changes | none |

The source **child digest** is the reproducibility boundary. `0.4.2-full`, `0.4.x-full`, and `full` are tags and must never substitute for that digest during source acquisition. The first-party registry's SLSA statement binds this exact child digest to commit `d55b0df...`, `Dockerfile.full`, the official repository, and release workflow input `0.4.2`; it also records the resolved builder base-image digests. [E7]

### Runtime authentication is deliberately not decided here

The immutable image contract preserves the exact upstream arguments above. In v0.4.2, if `--token` is absent the server constructs disabled authentication; omission is therefore an unauthenticated server, even though the literal `--no-token` flag is not present. [E9] Do **not** normalize the image arguments to add `--no-token`, select a token policy, or imply that this default is safe for AGS.

The scenario's AGS Tool `Command`/`Args` override is a deployment-time security decision blocked on [Define the runtime credential and authentication boundary](https://github.com/tlipoca9/ags-cookbook/issues/2). That decision must be applied without changing or republishing this image.

## Why mirror rather than build

The official full image is the only current upstream artifact that simultaneously gives the scenario an immutable runtime payload and all supported agents preinstalled. Upstream explicitly documents `0.4.2-full` as the versioned full image and its release workflow builds `linux/amd64` from `Dockerfile.full`. [E3, E10, E11]

A fresh source build of the tagged Dockerfile is **not bit-for-bit reproducible now**. It resolves moving inputs during the build: `node:22-alpine`, `node:22-bookworm-slim`, Debian package indexes, a musl-cross `latest` URL, `rustup target add`, and `sandbox-agent install-agent --all`. [E3] Agent installation without version overrides also consults current/latest native-agent endpoints and a live ACP registry; `--all` cannot be combined with the per-agent version overrides. [E12] Rebuilding would therefore produce a different agent set or bytes even from the same Git commit.

A no-op downstream Dockerfile such as `FROM rivetdev/sandbox-agent@sha256:a7b9...` is source-pinned but unnecessary: it can create a new manifest/config while adding no behavior. A registry-to-registry copy with digest preservation retains the exact audited runtime manifest.

## Contents and resource-relevant properties

The image must retain the complete upstream full payload, not a hand-selected agent subset:

- Sandbox Agent `0.4.2`, including the embedded Inspector UI. [E3, E8]
- All seven agent IDs reported installed by `/v1/agents`: Claude, Codex, OpenCode, Amp, Pi, Cursor, and the built-in mock agent. The upstream `--all` list contains the same seven IDs. [E8, E12]
- Native binaries observed in the immutable image: Claude Code `2.1.84`, Codex CLI `0.116.0`, and OpenCode `1.3.2`. [E8]
- ACP payloads observed in the image: Claude adapter `@zed-industries/claude-agent-acp@0.23.0`, Codex adapter `@zed-industries/codex-acp@0.10.0`, Amp adapter binary SHA-256 `83f88b1a4fc4f18a078f93f815c5d5148a73a9f4ce37bc92b3214f0445b2f35a`, `pi-acp@0.0.23`, Cursor adapter `@blowmage/cursor-agent-acp@0.1.0` with `cursor-agent@1.0.3`, plus the OpenCode/native and built-in mock launchers. The source digest, not these human-readable versions alone, pins all transitive files. [E8]
- Runtime packages intentionally installed by the upstream full recipe: `bash`, `ca-certificates`, `curl`, and `git`, on `node:22-bookworm-slim`; registry config reports Node `22.22.2`, and inspection reports npm `10.9.7`. [E3, E5, E8]
- The full recipe does not install the optional desktop runtime or `ffmpeg`; the scenario must not claim either capability from this image contract. [E3, E10]

Registry metadata reports **546,706,174 bytes compressed** for the amd64 child. Local image inspection reports **1,464,832,488 bytes unpacked**, 11 filesystem layers, and approximately 952 MiB under the preinstalled Sandbox Agent data directory. These properties are relevant to CCR pull time, AGS cold start, node disk pressure, and pre-cache planning; they are not a CPU/RAM sizing recommendation. [E4, E5, E8]

The image is non-root and its default writable project location is under `/home/sandbox`. The Shanghai prototype must create its test project there (for example `/home/sandbox/workspace`) rather than assuming `/workspace` exists or that root-owned paths are writable. [E3, E5, E8]

## Publication specification

No production copy was performed during this research. A publisher with normal CCR credentials should execute the equivalent of:

```bash
SOURCE='docker://docker.io/rivetdev/sandbox-agent@sha256:a7b9afc8c79fb075852d3ae73b86ae201b5ce683b1bf5f11988980cd2bca09a5'
DEST='docker://ccr.ccs.tencentyun.com/ags.dev/sandbox-agent:v0.4.2-full-amd64'

skopeo copy \
  --override-os linux \
  --override-arch amd64 \
  --preserve-digests \
  "$SOURCE" "$DEST"
```

`--preserve-digests` must fail the publication if CCR cannot retain the selected manifest rather than silently changing it. After publication, inspect the CCR reference and record the destination manifest digest. The expected digest is `sha256:a7b9...`; if CCR changes representation despite a supported copy path, stop and review rather than blessing an unrecorded digest.

### Fallback Dockerfile specification (not recommended)

If a future repository policy requires a Dockerfile artifact, the smallest acceptable recipe is:

```dockerfile
FROM docker.io/rivetdev/sandbox-agent@sha256:a7b9afc8c79fb075852d3ae73b86ae201b5ce683b1bf5f11988980cd2bca09a5
```

Build only with `docker buildx build --platform linux/amd64`. Do not add `USER`, `WORKDIR`, `ENTRYPOINT`, `CMD`, packages, labels, or agent installation. This fallback changes the destination image identity and must receive a distinct `-ags.N` tag plus a separately recorded digest. It is inferior to manifest copying because it adds no capability.

### Source-build reconstruction (audit only, not publication)

For upstream recipe auditing, not for producing the cookbook pin:

```bash
git clone https://github.com/rivet-dev/sandbox-agent.git
cd sandbox-agent
git checkout --detach d55b0dfb887cc92152f20f756995317c5f5c7709
test "$(git rev-parse refs/tags/v0.4.2^{commit})" = \
  d55b0dfb887cc92152f20f756995317c5f5c7709

docker buildx build \
  --platform linux/amd64 \
  --file docker/runtime/Dockerfile.full \
  --build-arg TARGETARCH=amd64 \
  --load \
  --tag sandbox-agent:audit-v0.4.2-full-amd64 \
  .
```

A successful build validates buildability only. Its digest is not expected to equal the official digest because of the moving inputs listed above, and it must not be pushed under the contract tag.

## Verification checklist

### Before copy

- [ ] `gh release view v0.4.2 --repo rivet-dev/sandbox-agent` reports a published, non-prerelease release. [E1]
- [ ] `git rev-parse refs/tags/v0.4.2^{commit}` equals the full approved commit. The current tag is lightweight and unsigned, so do not describe it as a signed source tag. [E2]
- [ ] Docker Hub tag metadata still maps `0.4.2-full` to the recorded index and amd64 child digest; fail on drift. [E4]
- [ ] Fetch and archive the amd64 SLSA statement; verify its subject digest, repository, revision, workflow, and Dockerfile path. [E7]
- [ ] `skopeo inspect --raw` of the child reports OCI image manifest and exactly the recorded config/layer digests. [E5]

### After copy / before release

- [ ] CCR tag resolves to one `linux/amd64` image and no ARM or unknown/attestation platform entries.
- [ ] Destination child manifest digest equals the approved recorded digest.
- [ ] Config is unchanged: user `sandbox`, workdir `/home/sandbox`, entrypoint/args and port as in the contract, and no image `HEALTHCHECK`. [E5]
- [ ] Run as the image user; assert `id -u=1001`, `id -g=1001`, `sandbox-agent --version` is `0.4.2`, and the current directory is `/home/sandbox`. [E8]
- [ ] Start with the immutable image-default args only in an isolated local test; `GET /v1/health` returns HTTP 200 and exact `{"status":"ok"}`. [E6, E8]
- [ ] `GET /v1/agents` reports all seven IDs installed. Do not exercise paid agents or provide credentials in this smoke test. [E8]
- [ ] Confirm compressed/unpacked sizes have not unexpectedly grown and document the observed CCR pull time in the Shanghai prototype. [E4, E8]
- [ ] Run vulnerability/license scanning under the cookbook's publication policy; this research did not perform those governance checks.
- [ ] Only after the runtime-auth ticket resolves, validate the selected Tool `Command`/`Args` and probe behavior through the AGS gateway.

## Tag, digest, and upgrade policy

1. Never publish or document `latest`, `full`, `0.4.x-full`, or any other moving channel. Upstream's release machinery intentionally moves channel tags for latest stable releases. [E11]
2. Treat `v0.4.2-full-amd64` as immutable after its first successful CCR publication. Never force-update or rebuild it.
3. Record both provenance values in the build appendix: upstream Git commit and upstream amd64 child digest. Record the post-copy CCR digest too.
4. Customer commands may use the immutable CCR version tag because existing deployment-cookbook examples use pinned CCR version tags, but maintainers must verify its recorded digest before validation and publication. Existing repository validation already models a tag-plus-recorded-digest invariant for image artifacts. [E13]
5. An upgrade is a new review and new tag, not an in-place rebuild:
   - choose a published stable upstream release (no RC unless a product decision explicitly approves one);
   - resolve its tag to a full commit;
   - inspect its full-image amd64 child and SLSA provenance;
   - diff `Dockerfile.full`, supported-agent list, CLI auth semantics, routes/OpenAPI, and release workflow from the previous commit;
   - inventory agent/runtime versions and image size from the new immutable image;
   - copy by child digest to `v<version>-full-amd64`;
   - rerun every checklist item and the complete Shanghai pause/resume scenario;
   - update all English/Chinese image references and the recorded digest atomically.
6. If no signed tag or signed provenance is introduced upstream, retain the limitation plainly; do not infer cryptographic source authenticity from a digest alone.

## Rejected alternatives

| Alternative | Reason rejected |
| --- | --- |
| Rebuild upstream `Dockerfile.full` and push the result | Same commit does not fix moving base tags, apt repositories, musl-cross `latest`, toolchain downloads, the live ACP registry, or latest native-agent downloads. It cannot promise identical bytes. [E3, E12] |
| Build from `node:22-bookworm-slim` plus `releases.rivet.dev/.../install.sh` | The official install script downloads the binary but does not verify a checksum/signature, and a channel such as `0.4.x` moves. Recreating the full agent layer still hits the mutable installation paths. [E14, E12] |
| Use minimal `0.4.2` and lazy-install an agent | Upstream says skipped agents are installed lazily; that makes first session startup network-dependent and changes bytes/runtime behavior after deployment. The scenario requires a reliable pause/resume baseline. [E10] |
| Pin only `rivetdev/sandbox-agent:0.4.2-full` | A registry tag is mutable by design even when convention says it should not move. It is discovery metadata, not a content pin. [E4, E11] |
| Use the multi-platform index digest | It includes ARM64 and two attestation manifests. The approved target is one `linux/amd64` runtime manifest. [E4, E7] |
| Choose `v0.5.0-rc.3` | It is a prerelease, while v0.4.2 is the latest stable release. No scenario requirement establishes a need for RC-only behavior. [E1] |
| Patch entrypoint, auth, health, user, or workdir downstream | Upstream already supplies the required server bind, port, non-root user/workdir, and health route. Authentication is a deployment-time decision, not evidence of an image defect. [E3, E6, E9] |
| Install only the eventual golden-path agent | [Choose the baseline agent and model-provider contract](https://github.com/tlipoca9/ags-cookbook/issues/6) is unresolved. Pruning would create a downstream image variant and new support burden without reducing the already-published source payload. |

## Implications for the blocked Shanghai prototype

[Validate the Shanghai pause-and-resume journey](https://github.com/tlipoca9/ags-cookbook/issues/3) can now pin the image payload, port, user, workdir, and health endpoint. It should budget for a ~547 MB compressed pull, pre-cache or allow an adequate cold-start window, use a project path under `/home/sandbox`, and probe `/v1/health`. [E4, E5, E8]

The prototype remains blocked on two independent runtime choices rather than image work:

- the AGS Tool authentication override from [Define the runtime credential and authentication boundary](https://github.com/tlipoca9/ags-cookbook/issues/2); and
- the one agent/provider path from [Choose the baseline agent and model-provider contract](https://github.com/tlipoca9/ags-cookbook/issues/6).

No downstream patch is justified by current primary evidence. If the prototype later proves an AGS-specific incompatibility, capture the failing command, response, and immutable image digest before opening a patch decision; do not mutate this tag.

## Residual unknowns

1. CCR digest preservation and AGS pull behavior have not been tested because this ticket forbids publishing and registry credentials. This is the first publisher/prototype gate.
2. Cold-start, CPU, RAM, ephemeral-disk headroom, and pause/resume timings are platform measurements, not inferable from OCI metadata; issue 3 must measure them in `ap-shanghai`.
3. Upstream `v0.4.2` is a lightweight unsigned tag. The first-party image includes SLSA provenance, but this research did not find or verify a cryptographic signature over that statement. [E2, E7]
4. Upstream's full build resolved agent versions at its 2026-03-26 build time. The digest freezes those bytes, but rebuilding the recipe later will not reconstruct them without separately pinning every transitive artifact. [E3, E8, E12]
5. Vulnerability, third-party license, and redistribution review for the bundled proprietary agents is outside this technical reproduction ticket and must be completed before a public CCR release.

## Evidence ledger

All repository line links use the immutable upstream commit.

- **E1 — official release metadata:** [v0.4.2 release](https://github.com/rivet-dev/sandbox-agent/releases/tag/v0.4.2); `gh release list --repo rivet-dev/sandbox-agent` reported v0.4.2 as Latest and v0.5.0-rc.1 through rc.3 as prereleases on 2026-09-01.
- **E2 — official Git reference and commit:** [Git ref API](https://api.github.com/repos/rivet-dev/sandbox-agent/git/ref/tags/v0.4.2) resolves directly to commit `d55b0df...`; [commit](https://github.com/rivet-dev/sandbox-agent/commit/d55b0dfb887cc92152f20f756995317c5f5c7709). Local `git cat-file -t v0.4.2` returned `commit`, confirming a lightweight rather than annotated/signed tag.
- **E3 — upstream full recipe:** [`docker/runtime/Dockerfile.full`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docker/runtime/Dockerfile.full), especially [build inputs](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docker/runtime/Dockerfile.full#L1-L96) and [runtime contract](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docker/runtime/Dockerfile.full#L137-L159).
- **E4 — first-party registry metadata:** [Docker Hub tag API](https://hub.docker.com/v2/repositories/rivetdev/sandbox-agent/tags/0.4.2-full) reports index `sha256:e356...`, amd64 child `sha256:a7b9...`, amd64 compressed size 546,706,174 bytes, ARM child, and attestation entries. `skopeo inspect --raw docker://rivetdev/sandbox-agent:0.4.2-full` independently returned the same OCI index members.
- **E5 — immutable image manifest/config inspection:** `skopeo inspect --raw` and `skopeo inspect --config` against `docker://rivetdev/sandbox-agent@sha256:a7b9...` reported an OCI image manifest, 11 layers, `linux/amd64`, user/workdir/entrypoint/Cmd/port shown in the contract, Node `22.22.2`, and no `Healthcheck` field.
- **E6 — health implementation:** upstream router registers `/v1/health` and returns `{"status":"ok"}` ([route](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router.rs#L183-L186), [handler](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/router.rs#L637-L649)); upstream quickstart uses that endpoint ([docs](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/quickstart.mdx#L198-L202)).
- **E7 — first-party SLSA statement:** OCI manifest `sha256:ceb2495b09c28b1b26b85ad5607df524dbfc9c4e93fee3c1114abcc9f310e71f`, attached in the official index, contains `application/vnd.in-toto+json` statement layer `sha256:a412ca...`. `skopeo copy`/JSON inspection reported subject `pkg:docker/rivetdev/sandbox-agent@d55b0df-full-amd64` digest `a7b9...`, source `https://github.com/rivet-dev/sandbox-agent`, revision/workflow SHA `d55b0df...`, workflow `.github/workflows/release.yaml`, Dockerfile path `Dockerfile.full`, and resolved Dockerfile/Node/Rust image digests.
- **E8 — local, credential-free runtime inspection of the immutable official image:** `podman pull/run --platform linux/amd64 docker.io/rivetdev/sandbox-agent@sha256:a7b9...` reported Sandbox Agent 0.4.2, uid/gid 1001, Node 22.22.2, npm 10.9.7, native and ACP payload versions/hashes listed above, unpacked size 1,464,832,488 bytes, agent data ~952 MiB; a local server returned exact health JSON and `/v1/agents` reported all seven installed. No model credentials or paid calls were used.
- **E9 — v0.4.2 authentication behavior:** [CLI token flags](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/cli.rs#L41-L54) and [`run_server`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/cli.rs#L450-L466) show that a supplied token enables auth and absence selects `AuthConfig::disabled()`.
- **E10 — upstream Docker guidance:** [published full image and exact tag](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/deploy/docker.mdx#L10-L22), [custom full install recipe](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/deploy/docker.mdx#L78-L99), and [optional/lazy agent behavior](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/docs/quickstart.mdx#L219-L227).
- **E11 — official image release flow:** workflow builds full amd64 with `Dockerfile.full` ([release workflow](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/.github/workflows/release.yaml#L173-L223)); release code creates versioned and moving channel manifests ([`scripts/release/docker.ts`](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/scripts/release/docker.ts#L27-L60)).
- **E12 — agent installation semantics:** all seven IDs ([source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/agent-management/src/agents.rs#L20-L80)); CLI version overrides conflict with `--all` ([source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/sandbox-agent/src/cli.rs#L315-L325)); default registry URL and latest native download paths ([source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/agent-management/src/agents.rs#L15-L18), [native installers](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/server/packages/agent-management/src/agents.rs#L1283-L1523)).
- **E13 — cookbook image conventions:** deployment-cookbook uses a versioned CCR tag and updates image/build/tutorial references together ([httpbin build guide](../../examples/deployment-cookbook/httpbin/dockerfiles/README.md)); envd validation explicitly forbids `latest`, records the pushed manifest digest, and fails on mismatch ([`.env.example`](../../examples/envd-oci-env/.env.example#L28-L34)).
- **E14 — official install script:** it selects a versioned URL, downloads with `curl`, marks executable, installs, and runs `--version`, but contains no checksum/signature verification ([source](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/scripts/release/static/install.sh#L14-L15), [download/install](https://github.com/rivet-dev/sandbox-agent/blob/d55b0dfb887cc92152f20f756995317c5f5c7709/scripts/release/static/install.sh#L69-L102)).
