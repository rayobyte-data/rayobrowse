# Changelog

## [Fixes] - 2026-07-05

### Fixed

- **Major stealth improvement** — addressed increased detection signals from demo.fingerprint.com/playground to significantly improve fingerprint coverage.
- **No more crashes from port-scan probes (Issue #6)** — some websites probe `localhost:9222` directly, which could crash the entire browser process. A new origin-check middleware now rejects those requests before they can do any damage. No configuration changes required.
- **`console.log()` output restored** — console messages were silently swallowed under CDP world isolation; logging now works as expected.
- **`ignore_https_errors` now works in daemon mode (Issue #4)** — this option was previously accepted but silently dropped. It's now fully wired through and applies via CDP's `Security.setIgnoreCertificateErrors`.

### Added

- **Custom IP lookup service (Issue #5)** — you can use your own custom IP checking service with any URL that returns your IP as plain text, useful for avoiding rate limits or for privacy. A hostname without a scheme is automatically prefixed with `https://`. See README.
- **Extension loading via the daemon (Issue #4)** — two supported paths:
  - Local/native mode: pass extension paths through `--load-extension` in `launch_args`.
  - Docker mode: mount the extension directory (`docker -v /local/addon:/addon`), then reference the mounted path.
- **Stable VNC port** — VNC no longer increments across ports (`6080`, `6081`, `6082`, ...). It's always served on `:6080` with a token-based URL. A `vnc` parameter was also added to `create_browser()` for convenience.
- **Optional unauthenticated VNC** — pass `vnc_no_auth=true` alongside `vnc=true` to skip the token-protected route entirely and get back a VNC URL on a dedicated port with no protection at all. Opt-in only; the default `vnc=true` behavior (stable, token-protected `:6080`) is unchanged.

---

## [Chromium 146.0.7680.219] - 2026-07-05

### Changed

- **Chromium updated to 146.0.7680.219** — both `linux/amd64` and `linux/arm64` Docker images rebuilt with Chromium v146 build 18.
  Pull: `docker pull rayobyte/rayobrowse:latest` or pin with `rayobyte/rayobrowse:chromium-146`.

---

## [Chromium 146.0.7680.169] - 2026-06-29

### Changed

- **Chromium updated to 146.0.7680.169** — both `linux/amd64` and `linux/arm64` Docker images rebuilt with Chromium v146 build 16.
  Pull: `docker pull rayobyte/rayobrowse:latest` or pin with `rayobyte/rayobrowse:chromium-146`.

---

## [0.2.2] - 2026-05-22

### Changed

- **CPU Performance Increase** - rayobrowse's performance has now been increased. Various benchmarks confirm it is more efficient than vanilla Chrome, while still retaining its important fingerprint stealth. 

---

## [0.2.1] - 2026-04-30

### Changed

- **Major anti-fingerprinting update** - all tests pass at demo.fingerprint.com/playground inside the Docker environment hosted on a GPU-less Linux server when using a Windows fingerprint.

---

## [0.2.0] - 2026-04-17

### Breaking

- **`GET /connect` is now HTTP-only.**  Previously `/connect` upgraded to a
  WebSocket and proxied CDP through the daemon.  It now responds with a plain
  `text/plain` body containing the CDP WebSocket URL (`ws://<host>/cdp/<id>`)
  plus `x-session-id` and (optional) `x-vnc-url` response headers — identical
  to the cloud gateway.  Attempting to send an `Upgrade: websocket` request
  to `/connect` now returns HTTP 400.
- **Clients must upgrade.**  Use `rayobrowse>=2.2.0` (Python) or
  `rayobrowse@0.3.0` (Node) — both of which now always issue an HTTP GET and
  return the real CDP URL to hand to Playwright/Puppeteer.  Old SDKs that
  hardcoded `ws://localhost:9222/connect?…` will fail.

### Added

- **Cloud-parity `/api/*` routes**: `POST /api/browser/create`,
  `POST /api/browser/close`, `GET /api/browser/{id}/status`,
  `GET /api/browsers`, `POST /api/browser/status/bulk`.  Response bodies are
  `camelCase` and match the cloud gateway exactly.
- **`keepAlive=` and `maxLifetime=` query params on `/connect`** —
  client-controlled browser lifecycle.  `keepAlive=true` persists the browser
  across CDP disconnects; `maxLifetime` (seconds) hard-caps session lifetime.
- **`sessionId=` query param on `/connect`** — reconnect to an existing
  browser without creating a new one.
- **`vnc=true` query param on `/connect`** — causes `x-vnc-url` to appear in
  the response headers for live session viewing.

### Deprecated

- Legacy routes `POST /browser`, `GET /browser/{id}`, `DELETE /browser/{id}`,
  `GET /browsers` still work but log a one-shot deprecation warning the first
  time they are called.  Migrate to the `/api/*` equivalents.

### Migration

Before (old SDK + old daemon):

```python
from rayobrowse import create_browser
ws_url = create_browser(headless=False)
# ws_url was ws://localhost:9222/connect?headless=false&...
# Playwright would upgrade it to a WebSocket and the daemon created the browser on connect.
```

After (`rayobrowse>=2.2.0`):

```python
from rayobrowse import Rayobrowse
client = Rayobrowse(endpoint="http://localhost:9222")  # or cloud URL — same code
cdp_url = client.connect_url(headless=False, os="windows", vnc=True)
# cdp_url is ws://localhost:9222/cdp/<browser_id>, ready for Playwright.
# client.last_vnc_url  -> noVNC URL if vnc=True
# client.last_session_id -> the new session id
```

The `create_browser()` compat shim from `rayobrowse` still works identically;
only the URL scheme returned changed (you no longer get a `/connect?...` URL,
you get a direct `/cdp/...` URL — which Playwright uses the same way).

## [0.1.33] - 2026-03-27

### Added
- **Per-browser VNC isolation** — each browser now gets its own display and noVNC port (6080–6119), eliminating cross-session visual interference when running multiple browsers simultaneously. Docker Compose now exposes the full port range.
- **Native mode** — SDK client now supports `STEALTH_BROWSER_MODE=native` for running the daemon as a local Python process without Docker.
- **`debug` parameter** — `create_browser(debug=True)` generates a diagnostic report on the daemon host for troubleshooting.
- **Chromium v145 and v146 support** — fingerprint patches ported to latest Chromium versions with multi-version compatibility.

### Improved
- **ClientRect fingerprinting** — `getClientRects()` and `getBoundingClientRect()` now return noise-adjusted values matching real hardware variance.
- **Canvas GPU noise model** — GPU fingerprints use a realistic noise model with optimized text rendering, improving canvas fingerprint diversity.
- **Docker security hardening** — container now runs as a non-root user with Chrome's setuid sandbox, removing the need for `--cap-add=SYS_ADMIN`.
- **colorDepth accuracy** — display server upgraded to support depth 32, so `screen.colorDepth` now correctly reports 32 to match real Windows environments.
- **CDP connection stability** — WebSocket ping/pong keepalive prevents idle CDP connections from being dropped.

### Fixed
- **VNC cleanup** — VNC teardown now completes reliably even if the browser kill step hangs.
- **`keepAlive` flag** — browsers with `keepAlive` enabled are no longer prematurely killed during disconnect cleanup.

---

## [0.1.32] - 2026-03-10

### Improved
- **Page visibility and focus** — all browser windows now report as visible (`document.visibilityState === "visible"`) and focused, matching the state of a real foreground tab. When running multiple browsers on a single server, anti-bot systems can detect background or hidden pages; this update ensures every session appears as an active, human-attended tab.
- **MAJOR: WebGL rendering pipeline** — GPU fingerprints now match real hardware profiles instead of exposing SwiftShader (software renderer) metadata. Spoofed WebGL values are also returned with realistic query latency, defeating timing-based detection that flagged instant in-memory responses as synthetic.

### Fixed
- **Locale normalization** — geo-derived locales are now normalized to Chrome-supported values (e.g. `en-SG` → `en-GB`), fixing `Intl` locale mismatches between the main thread and Web Workers.
- **Worker language consistency** — `entropy.languages` is now set for every session, ensuring `navigator.languages` returns the same value inside Workers as it does on the main page.

---

## [0.1.26] - 2026-02-21

### Added
- **`/connect` endpoint** — WebSocket endpoint that auto-creates a stealth browser on connection. Third-party tools (OpenClaw, Scrapy, Puppeteer, Selenium) can now connect via a static CDP URL like `ws://localhost:9222/connect?headless=true&os=windows` without calling the REST API. Browser is automatically cleaned up on disconnect.
- **OpenClaw integration** — full setup guide and configuration examples for running OpenClaw AI agents with rayobrowse browsers. See [`integrations/openclaw/`](integrations/openclaw/).
- **Scrapy integration** — complete tutorial and ready-to-run example project for `scrapy-playwright` + rayobrowse. Includes spiders for JS-rendered sites. See [`integrations/scrapy/`](integrations/scrapy/).

---

## [0.1.25] - 2026-02-20

### Added
- **ARM64 (Apple Silicon / AWS Graviton) support** — Docker image now ships as a multi-architecture build supporting both x86_64 (amd64) and ARM64. Docker automatically pulls the correct image for your platform. Both architectures are tested in CI/CD for each release.

---

## [0.1.24] - 2026-02-20

### Added
- **Remote / Cloud mode** — new `STEALTH_BROWSER_DAEMON_MODE=remote` option
  turns rayobrowse into a publicly accessible cloud browser backend. Your server
  creates browsers via REST API (authenticated with API key), and end users
  connect directly over CDP WebSocket with zero middleman.

### Fixed
- **Docker image cache issue** — fixed a bug where fonts and the Chromium binary
  were being downloaded twice on first startup.

---

## [0.1.11] - 2026-02-19

### Added
- **Selenium and Puppeteer support** — in addition to Playwright, rayobrowse
  now has ready-to-run examples for Selenium and Puppeteer. See the
  [`examples/`](examples/) folder.
- **`AGENTS.md`** — end-to-end setup guide for AI agents and LLM-driven
  pipelines, covering setup, verification, and common failure modes.
- **`examples/verify_setup.py`** — non-interactive E2E test script; run it
  to confirm your installation is working correctly.
- **Terms acceptance via environment variable** — `STEALTH_BROWSER_ACCEPT_TERMS=true`
  replaces the previous interactive prompt, making setup compatible with
  automated and headless environments.

---

## [0.1.11] - 2026-02-18

### Added
- **Docker-first architecture** — rayobrowse now runs entirely inside a Docker
  container. No system dependencies on the host beyond Docker and Python. This
  enables running on any x64 machine (ARM64 coming soon), while making it
  easier to develop and deploy to production.
- **Daemonized** — the system has been daemonized so that only a lightweight
  Python SDK is needed to connect to the daemon which manages the browsers. All
  connections from your automation library (Playwright, etc.) go through a
  simple CDP socket.
- **noVNC viewer** — watch live browser sessions in real time at
  `http://localhost:6080`. No VNC client required.
