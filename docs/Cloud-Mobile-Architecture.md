# Cloud + Mobile Architecture

> Status: initial architecture proposal for the `snowgemm/obscura` fork. This document records a staged direction; it does not claim that mobile builds or a hosted service already work.

## 1. Product goal

Make Obscura usable from phones and tablets without requiring the browser engine to run on every device:

- A responsive mobile-first web app (PWA) is the first client for Android, iOS, tablets, and desktop browsers.
- A cloud control plane authenticates users and manages browser sessions.
- Isolated Linux workers run the existing Obscura engine.
- The client receives page previews and sends explicitly authorized browser actions.
- Native Android/iOS shells are optional follow-on work, after the API and PWA are stable.

The cloud service is a separate product layer around the Apache-2.0 engine. It must not imply that upstream Obscura Cloud infrastructure or branding belongs to this fork.

## 2. Current repository baseline

The existing workspace contains ten Rust crates: CLI, CDP, browser lifecycle, V8/JavaScript runtime, DOM, network, SSRF policy, MCP, renderer, and embeddable library.

Current facts from this fork's `main`:

- The release workflow builds Linux x86_64/ARM64, macOS, and Windows desktop binaries; it has no Android or iOS release target.
- The CDP server already supports a bearer token through `OBSCURA_CDP_TOKEN`, origin checks, connection caps, and private-network blocking by default.
- CDP is a browser-control protocol, not a user-facing multi-tenant cloud API.
- The engine executes untrusted page JavaScript in-process. The repository's security policy explicitly says in-process hardening is not a substitute for OS-level isolation.
- The pinned dependency graph uses `deno_core 0.412.0` and `v8 150.4.0`. Cross-compilation support in a current V8 README is not proof that this exact workspace builds on mobile.

## 3. Proposed system

```text
Mobile / Tablet / Desktop
  Responsive PWA
       | HTTPS + authenticated WebSocket
       v
Cloud API (Rust)
  - authentication and authorization
  - session lifecycle and ownership
  - quotas, request validation, audit events
  - short-lived session capabilities
       |
       v
Session Orchestrator
  - creates and destroys isolated workers
  - applies CPU / memory / time / concurrency limits
  - routes only the owning user's session traffic
       |
       v
Obscura Worker (existing Rust engine)
  - private CDP endpoint
  - per-session profile and temporary storage
  - restricted outbound network
       |
       v
Web pages (untrusted content)
```

### Client

**First client: responsive PWA.** It gives the same UI to Android, iOS, tablets, and desktop without maintaining two native UI codebases before the service contract is stable. It should support:

- Sign-in and session list.
- Create, rename, stop, and delete a session.
- URL navigation, back/forward/reload, viewport controls.
- Live preview using a bounded stream of screencast frames.
- Basic browser actions and a clear connection/error state.
- Accessible touch targets, keyboard support, and reduced-motion settings.

A native Android client can follow if PWA limitations materially affect the experience. An iOS shell should only be considered after the cloud workflow is validated. The mobile client must never receive a worker's raw CDP token.

### Cloud API

A small Rust HTTP/WebSocket service is the control plane. Axum is a candidate because the repository already uses Tokio, but the dependency and version must be checked before adoption.

The API owns user identity, session ownership, authorization, quotas, lifecycle, and event logs. It issues short-lived, session-scoped capabilities and proxies permitted operations. It must not expose arbitrary CDP methods or a public worker address.

### Worker isolation

The first deployment target is Linux containers. Use one isolated worker container per active browser session for the MVP, then benchmark whether a safer and cheaper per-user worker model is viable.

Minimum container controls:

- Non-root UID; no privileged mode; no host PID/network namespaces.
- No Docker socket, host filesystem mounts, or unnecessary Linux capabilities.
- Read-only root filesystem where practical; bounded writable temp/profile storage.
- CPU, memory, process-count, file-descriptor, and wall-clock limits.
- Restricted egress with DNS/IP validation on every navigation, redirect, and subresource request.
- Block loopback, private, link-local, and cloud metadata ranges by default.
- Separate worker credentials and storage per session; destroy temporary state on teardown.
- Keep CDP on a private worker network. Require authentication even on that network.

A container is a minimum isolation boundary, not a guarantee against every kernel or V8 exploit. Stronger VM/microVM isolation should be evaluated before accepting arbitrary public workloads at scale.

### Data and persistence

Initial data model:

- `users`: identity subject and account status.
- `sessions`: owner, state, worker reference, created/last-active timestamps, expiry.
- `session_events`: bounded operational events; do not store page contents by default.
- `usage_counters`: session minutes, worker memory/time, and request counts.

PostgreSQL is the proposed durable store. Worker profiles and screenshots are ephemeral by default. Persistent cookies or browser profiles require an explicit user opt-in, encryption at rest, clear deletion controls, and documented retention. Never log passwords, cookies, authorization headers, or page secrets.

## 4. Security boundaries

1. Authenticate every API request and authorize every session against its owner.
2. Never expose raw CDP, worker ports, or long-lived worker secrets to clients.
3. Treat URLs, redirects, DNS answers, page scripts, downloads, and uploaded content as untrusted.
4. Enforce SSRF controls at the worker network boundary as well as in the engine.
5. Apply quotas before allocating a worker; enforce hard limits inside the worker/container too.
6. Use TLS externally and short-lived, scoped credentials internally.
7. Keep audit logs metadata-only by default, with retention limits.
8. Add rate limits, WebSocket message-size limits, idle timeouts, and cancellation.
9. Do not add proxy-provider integrations, stealth defaults, or paid APIs as prerequisites for the core product.
10. Provide an account/session deletion path and explain what data is ephemeral versus persistent.

## 5. Platform feasibility

### Android

Rust supports Android targets including `aarch64-linux-android`, and the V8 project documents Android cross-compilation. However, Obscura's exact pinned V8/deno_core versions and all native dependencies still need a reproducible build test. Android engine execution is not part of the first milestone because the cloud worker removes that requirement from the phone.

### iOS

The V8 project documents iOS device builds in a jitless configuration because of platform JIT restrictions. That makes a direct on-device engine port a separate compatibility project. The PWA client avoids making iOS V8 support a launch blocker.

### Compatibility promise

The initial promise is **mobile browser access**, not “every phone can run the engine locally.” Validate current Android Chrome and iOS Safari versions during device testing. Native app support and older-device support must be stated only after real tests.

## 6. Cost and deployment policy

- Local development can run without a paid cloud provider.
- A public hosted service will incur infrastructure, bandwidth, storage, domain, and possibly database costs.
- Start with Docker Compose on a developer machine; do not enable a paid cloud account, managed database, proxy service, or paid API without explicit approval.
- Keep deployment portable: OCI containers, environment-based configuration, and documented backup/restore.
- Publish a cost estimate only after measuring worker memory, session duration, bandwidth, and expected concurrency.

## 7. Delivery plan

### M0 — Baseline and reproducibility
- Run the existing CI/build/test path on the fork.
- Record the current upstream commit and dependency/security audit.
- Do not change engine behavior during baseline validation.

### M1 — Service contract
- Define API schemas, session state machine, error model, quotas, and WebSocket events.
- Add threat model and data-retention decisions.
- Add contract tests before implementing the UI.

### M2 — Single-user local cloud prototype
- Build the API and orchestrator for one local worker.
- Keep worker CDP private and authenticated.
- Verify create/navigate/preview/stop lifecycle and cleanup.

### M3 — Multi-user isolation
- Add authentication, per-user authorization, per-session worker isolation, quotas, and audit events.
- Test cross-session access denial, SSRF cases, resource exhaustion, and worker crash recovery.

### M4 — Mobile PWA
- Build responsive session management and live preview.
- Validate touch, keyboard, reconnect, low-bandwidth behavior, and mobile browser compatibility.

### M5 — Production hardening
- Add deployment/backup/monitoring documentation, rate limits, retention controls, and measured cost model.
- Consider native wrappers only if the PWA has demonstrated limitations.

## 8. Acceptance gates

No milestone is complete based on a successful compile alone.

- Existing engine tests and obstacle-course gate remain green.
- API tests prove user/session ownership and deny cross-user access.
- Workers cannot reach private or metadata IP ranges by default, including through redirects and DNS changes.
- A worker crash or timeout cannot leave a session marked as healthy.
- Session stop/delete removes worker resources and ephemeral data.
- Mobile UI is tested on real Android and iOS browsers at supported viewport sizes.
- No paid service is activated without explicit approval.

## 9. Open decisions

These are intentionally unresolved until the service contract and prototype provide evidence:

- Authentication mechanism and account recovery.
- Whether persistent browser profiles are needed in the first release.
- Exact API framework and database migration tool.
- Worker-per-session versus microVM isolation and its measured cost.
- Minimum supported Android/iOS browser versions.
- Whether a native app provides enough value beyond the PWA.
