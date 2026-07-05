*A self-hosted Chromium stealth browser for web scraping and automation.*

[Docs](https://docs.rayobrowse.com) · [Quick Start](#quick-start) · [Fingerprints](#what-rayobrowse-handles) · [Self-Hosted and Cloud](#self-hosted-and-cloud)

rayobrowse is a Chromium-based browser built to look like a real user in
server-side automation. It gives each session a coherent fingerprint across
browser APIs, graphics, fonts, language, timezone, WebRTC, automation surfaces,
and network-facing behavior.

It exposes standard CDP, so you can use it from Playwright, Puppeteer,
Selenium, Scrapy, OpenClaw, or your own automation code without learning a new
browser API.

Rayobyte uses rayobrowse at billion-page-per-month scale on major websites for its [web scraping API](https://rayobyte.com/products/web-scraping-api/). We
have been in the web scraping industry for 11 years, and this browser is part
of the infrastructure we rely on ourselves.

## Quick Start

Self-hosted rayobrowse is free and unlimited. Docker is the only required
runtime dependency.

```bash
git clone https://github.com/rayobyte-data/rayobrowse.git
cd rayobrowse
docker compose up -d
curl http://localhost:9222/health
```

### Connect with Playwright

`/connect` is an HTTP endpoint. It starts a browser and returns a CDP WebSocket
URL as plain text.

```python
# pip install httpx playwright && playwright install
import httpx
from playwright.sync_api import sync_playwright

r = httpx.get(
    "http://localhost:9222/connect",
    params={"os": "windows", "headless": "false", "vnc": "true"},
    timeout=120,
)
r.raise_for_status()
cdp_url = r.text.strip()
vnc_url = r.headers.get("x-vnc-url") or "http://localhost:6080/vnc.html"

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(cdp_url)
    context = browser.contexts[0] if browser.contexts else browser.new_context()
    page = context.pages[0] if context.pages else context.new_page()
    page.goto("https://browserscan.net/")
    print(page.title())
    print(f"To view your browser in VNC go to: {vnc_url}")
    input("Press Enter to close the browser...")
    browser.close()
```

The browser is cleaned up when the CDP session closes. For long-running
sessions, reconnects, and explicit lifecycle control, use the SDK or the
session management APIs.

For SDKs, session reuse, proxies, and integration guides, use
[docs.rayobrowse.com](https://docs.rayobrowse.com).

### `/connect` parameters

`/connect` accepts these query params in addition to `os`, `headless`, and
`vnc` shown above:

| Param                 | Type      | Description                                                                                                                                                                    |
| --------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy`                | string    | `http://user:pass@host:port`                                                                                                                                                 |
| `ignore_https_errors`  | bool      | Ignore invalid/expired/self-signed TLS certs (uses CDP under the hood).                                                                                                       |
| `ip_service`           | string    | Custom IP-lookup URL used to derive timezone/locale instead of the built-in provider rotation. A bare hostname (no `http(s)://`) is automatically treated as `https://`.      |
| `vnc`                  | bool      | Start a live noVNC viewer for the session. Returned in the `x-vnc-url` response header, always on a stable `:6080`, token-protected.                                          |
| `vnc_no_auth`          | bool      | Requires `vnc=true`. Skips the token-protected route and exposes the viewer on its own dedicated port with **no authentication at all**. Opt-in only — see warning below.     |
| `extension`            | string    | Path to an unpacked extension directory to load (must already be mounted into the container in Docker mode). Comma-separate multiple paths. See below.                        |
| `sessionId`            | string    | Reconnect to an existing browser instead of creating a new one.                                                                                                                |
| `keepAlive`            | bool      | Keep the browser alive after the CDP connection drops (skip auto-cleanup).                                                                                                     |
| `maxLifetime`          | int       | Max seconds the browser may live, regardless of activity.                                                                                                                       |

```python
# Ignore HTTPS errors + use a custom IP lookup service
r = httpx.get(
    "http://localhost:9222/connect",
    params={
        "os": "windows",
        "ignore_https_errors": "true",
        "ip_service": "https://api64.ipify.org",
    },
    timeout=120,
)
```

> **`vnc_no_auth` warning:** anyone who can reach the exposed port gets full
> view/control of that browser session with no login of any kind. Only use
> this on a trusted network (e.g. an SSH tunnel or a private VPC) — never
> expose it directly to the public internet.

### Loading a browser extension

Mount your unpacked extension into the container, then point `/connect` at
the mounted path with `extension=`:

```yaml
# docker-compose.yml
services:
  rayobrowse:
    volumes:
      - /local/path/to/addon:/addon:ro
```

```python
r = httpx.get(
    "http://localhost:9222/connect",
    params={"os": "windows", "headless": "false", "extension": "/addon"},
    timeout=120,
)
r.raise_for_status()
cdp_url = r.text.strip()

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(cdp_url)
    context = browser.contexts[0] if browser.contexts else browser.new_context()
    # Confirm the extension loaded via its background service worker
    print([sw.url for sw in context.service_workers])
    browser.close()
```

Pass multiple comma-separated paths (`extension=/addon1,/addon2`) to load more
than one extension.

## Detection Coverage

rayobrowse is built for protected, real-world targets and common fingerprint
test suites.


| System                          | Status                                     |
| ------------------------------- | ------------------------------------------ |
| Cloudflare                      | ✅                                          |
| Akamai                          | ✅                                          |
| PerimeterX / HUMAN              | ✅                                          |
| demo.fingerprint.com/playground | ✅                                          |
| BrowserScan                     | ✅                                          |
| PixelScan                       | ✅                                          |
| CDP clients                     | ✅ Playwright, Puppeteer, Selenium, raw CDP |


## What rayobrowse handles

Each session gets a coherent device profile across browser APIs, graphics,
networking, automation surfaces, and OS-level behavior. Covered signals include:

- User agent, browser version, client hints, platform, and OS metadata.
- Windows, Linux, macOS, and Android fingerprint support.
- Viewport, screen size, color depth, device scale factor, and media queries.
- WebGL vendor, renderer, extensions, parameters, shaders, and GPU consistency.
- Canvas output, text rendering, image rendering, and graphics behavior.
- AudioContext and audio processing fingerprints.
- OS-matched fonts and font rendering behavior.
- Timezone, locale, language, Accept-Language, and UI language.
- Navigator fields, permissions, plugins, MIME types, hardware concurrency,
memory, touch support, and related browser surfaces.
- WebRTC, DNS leak behavior, proxy alignment, and network-facing attributes.
- Automation surfaces, CDP artifacts, launch flags, process behavior, and
headless/headful consistency.
- Human-like mouse movement and click timing.

## Architecture

rayobrowse runs as a Docker container with:

- A Chromium-based browser binary.
- The stealth fingerprint engine.
- An HTTP daemon for `/connect`, browser close, status, health, and CDP proxy
endpoints.
- Optional noVNC viewing for live debugging.

Your automation code runs on the host and connects over CDP. You do not need a
host Chromium install, GPU setup, Xvfb, font packages, or custom browser flags.

## Why Docker Only

The hardest version of this problem is running a stealth browser reliably at
scale on Linux servers without GPUs. That is where scraping teams actually run
production workloads, so that is where we put our effort.

Docker lets rayobrowse ship the browser binary, patched runtime, fonts,
graphics stack, fingerprint engine, daemon, and VNC support as one reproducible
environment. You can run the same browser locally, in CI, on bare metal, or
across a cloud fleet without rebuilding a fragile desktop browser setup on
every machine.

## Self-Hosted and Cloud

Self-hosted rayobrowse is free and unlimited. Run it on your own machines, use
your own proxies, and get the same core stealth engine in every release. Local
use does not require registration.

rayobrowse Cloud is in early access for teams that want managed scaling,
orchestration, proxy integration, VNC access, observability, and fewer browser
fleet problems to own. The local and cloud connection model is intentionally
the same: call `/connect`, receive a CDP URL, automate the browser.

At scale, running browsers well means servers, proxies, updates, monitoring,
zombie cleanup, queueing, and debugging tools. We have optimized that stack so
managed rayobrowse Cloud should be cheaper for many teams than building and
maintaining their own fleet.


|                                                                                                    | Self-Hosted (Free)                                           | Cloud                       |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | --------------------------- |
| **Core Browser Engine**                                                                            |                                                              |                             |
| Full stealth engine (50+ spoofed signals)                                                          | ✅                                                            | ✅                           |
| Fingerprint spoofing (UA, WebGL, Canvas, fonts, timezone, audio, and more) from real-world devices | ✅                                                            | ✅                           |
| Patent-pending Canvas Fingerprint Spoofing with real canvas data instead of detectable noise       | ❌ Uses noise-based canvas spoofing, works on 90% of websites | ✅ Real canvas profiles      |
| CDP compatible: Playwright, Puppeteer, Selenium, and more                                          | ✅                                                            | ✅                           |
| Desktop fingerprints: Windows, Linux, macOS                                                        | ✅                                                            | ✅                           |
| Headless and headful modes                                                                         | ✅                                                            | ✅                           |
| Proxy support                                                                                      | ✅ Bring your own                                             | ✅ Managed or bring your own |
| Human mouse movement                                                                               | ✅                                                            | ✅                           |
| **Infrastructure and Operations**                                                                  |                                                              |                             |
| Central dashboard: start, stop, VNC view, traffic usage, and more                                  | ❌                                                            | ✅                           |
| Orchestration: auto-scaling, zombie cleanup, OOM protection                                        | ❌                                                            | ✅                           |
| Session queuing at capacity                                                                        | ❌                                                            | ✅                           |
| Automatic engine and software updates                                                              | ❌                                                            | ✅                           |
| **Proxy and Network**                                                                              |                                                              |                             |
| Managed proxy: datacenter, ISP, and residential                                                    | ❌                                                            | ✅                           |
| Proxies tuned for browser sessions                                                                 | ❌                                                            | ✅                           |
| **Monitoring and Debugging**                                                                       |                                                              |                             |
| Live VNC from anywhere                                                                             | ❌                                                            | ✅                           |
| Usage analytics and cost reporting                                                                 | ❌                                                            | ✅                           |
| Budget and proxy cost alerts                                                                       | ❌                                                            | Soon                        |
| Session replay                                                                                     | ❌                                                            | Soon                        |
| Remote DevTools                                                                                    | ❌                                                            | Soon                        |
| **Advanced Features**                                                                              |                                                              |                             |
| Real-world fingerprints captured like anti-bot vendors capture them                                | ✅ Limited set                                                | ✅ Tens of thousands         |
| Android mobile fingerprints                                                                        | ❌                                                            | ✅                           |
| Persistent browser profiles                                                                        | ❌                                                            | Beta                        |
| Smart Cache                                                                                        | ❌                                                            | Soon                        |
| **Collaboration**                                                                                  |                                                              |                             |
| Team accounts with role-based access                                                               | ❌                                                            | Soon                        |
| Audit logs                                                                                         | ❌                                                            | Soon                        |


Cloud is currently in early access. Read
[docs.rayobrowse.com](https://docs.rayobrowse.com) or contact
[sales@rayobyte.com](mailto:sales@rayobyte.com) if you want access.

## License

The repository wrapper, examples, docs, and configuration files are MIT
licensed. The rayobrowse Browser Binary has a separate license that allows most
commercial and hobby use, while restricting illegal or unethical use and
certain commercial resale categories.

See `[LICENSE](LICENSE)` and
`[BROWSER_BINARY_LICENSE.md](BROWSER_BINARY_LICENSE.md)`.

## Legal and Ethics

Use rayobrowse only for legal and ethical automation and web data collection.
Do not use it for credential attacks, fraud, spam, unauthorized access,
harassment, malware, or other harmful activity.

Rayobyte is a proud partner of the [EWDCI](https://ethicalwebdata.com/) and
takes ethical web scraping seriously.

## Support

- Docs: [docs.rayobrowse.com](https://docs.rayobrowse.com)
- Bugs and feature requests: open a GitHub issue.
- Cloud early access and commercial inquiries:
[sales@rayobyte.com](mailto:sales@rayobyte.com)

