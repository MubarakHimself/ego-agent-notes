# Peer agent-browser stacks — evidence baseline

**Task:** `ego-peers-001`  
**Research date:** 2026-09-22 (Africa/Nairobi, UTC+3)  
**Crew:** BrowserPeers / Firstmate  
**Method:** Primary sources only (official READMEs, docs, project pages). No code theft; no private repos; no invented metrics.  
**Scope note:** Captain directive — do not assume Electron. Embedding models are documented as evidenced by real projects.

---

## 1. Nanobrowser

**Repo:** https://github.com/nanobrowser/nanobrowser  
**License:** Apache License 2.0 ([README](https://github.com/nanobrowser/nanobrowser/blob/master/README.md))  
**Site:** https://nanobrowser.ai/

### What it is

Nanobrowser is an open-source AI web-automation tool packaged as a **Chrome/Edge browser extension**. It runs multi-agent workflows (Planner / Navigator, etc.) locally in the user’s browser using the user’s own LLM API keys, positioned as a free alternative to OpenAI Operator. Acknowledge stack: README credits Browser Use, Puppeteer/Agent-E, LangChain, and a Chrome extension boilerplate.

### Architecture shape

**Browser extension** (Manifest/extension model) — not Chromium-embedded, not a separate remote Playwright process. Agent logic runs in the extension context and automates the tabs of the host Chromium browser the human already has open.

### Agent + human shared browsing

**Same browser / same session.** The agent operates in the user’s Chrome/Edge; the human watches (and can intervene in) the same tabs. Side panel chat provides status and control. There is no separate mirrored remote browser by default — the shared surface *is* the user’s browser.

### Cross-platform reality

- **Official:** Chrome, Edge (full support per README / Chrome Web Store listing).  
- **Not supported:** Firefox, Safari, other Chromium variants (Opera, Arc, etc.) — may work but untested.  
- Packaging: Chrome Web Store + GitHub release zip (`Load unpacked`) + build-from-source (Node ≥ 22.12, pnpm ≥ 9.15.1).

### Speed / latency / process model (evidence-backed)

- Process model: extension + existing browser process; no separate Playwright daemon.  
- Latency numbers: **unknown** from primary docs (not published as benchmarks in README).  
- Maintainers emphasize local execution / privacy over cloud round-trips for the agent loop; LLM API latency still dominates wall-clock (inference, not local IPC).

### Notable limitations (maintainers / store)

- Chromium-extension-only OS/browser matrix.  
- Chrome Web Store build may lag GitHub releases (README “Important Note”).  
- Local models need cleaner, more specific prompts (README).  
- Cost-effective LLM configs “may produce less stable outputs” (README).  
- Release notes (v0.1.13) cite fixes for infinite loops on sandboxed iframes (e.g. Amazon) and recursion in clickable-element discovery — evidence of DOM/iframe edge cases.

---

## 2. Vercel agent-browser (`vercel-labs/agent-browser`)

**Repo:** https://github.com/vercel-labs/agent-browser  
**License:** Apache-2.0 ([README](https://github.com/vercel-labs/agent-browser/blob/main/README.md))  
**Docs/site:** https://agent-browser.dev

### What it is

A **CLI for AI agents** to drive a real browser via compact accessibility-tree snapshots and element refs (`@e1`, `@e2`, …). Marketed as a fast native Rust CLI. Primary loop: `open` → `snapshot` → `click`/`fill` by ref → re-snapshot. Also offers MCP mode, dashboard, cloud providers, and optional AI `chat`.

### Architecture shape

**Remote-drive via CDP** (not Electron; not an extension):

1. **Rust CLI** — parses commands, talks to daemon.  
2. **Rust daemon** — “Pure Rust daemon using direct CDP, no Node.js required” (README Architecture).  
3. Browser: Chrome for Testing by default; `--engine lightpanda` optional; Safari/iOS via WebDriver provider; connect to existing Chrome with `--cdp` / `--auto-connect`.

(Earlier community writeups described Playwright under a Node daemon; the **current official README** states direct CDP Rust daemon. Treat Playwright-under-the-hood claims as historical/third-party unless re-verified against the tree.)

### Agent + human shared browsing

Several evidenced modes:

| Mode | Evidence |
|------|----------|
| Headed local window | `--headed` shows browser for debugging |
| Pair browsing / stream | WebSocket viewport stream + mouse/keyboard/touch input (“human can watch and interact alongside an AI agent”) |
| Dashboard live viewport | `dashboard start` — live JPEG frames + activity feed |
| Attach to user’s Chrome | `--auto-connect`, `--cdp`, `--profile` (Chrome profile reuse / persistent profiles) |
| Tab pinning on shared CDP | `--pin-tab` so multi-session agents don’t steal each other’s tabs |
| Observation-only | Default stream is viewable; input optional via WS protocol |

Not a single “takeover” product UX like Skyvern’s stream buttons, but **same-session attach + bidirectional stream** are first-class.

### Cross-platform reality

Native Rust binaries: macOS ARM64/x64, Linux ARM64/x64, Windows x64. Install via npm, Homebrew, Cargo, or from source (Node 24+, pnpm 11+, Rust). Linux: `install --with-deps`. iOS Simulator / real device via Appium XCUITest provider (macOS + Xcode). Cloud providers: Browserless, Browserbase, Browser Use Cloud, Kernel, AWS AgentCore.

### Speed / latency / process model (evidence-backed)

From official README (not third-party blogs):

- Daemon persists between commands; idle timeout default **1 hour** then save/close.  
- Default action timeout **25s** (below CLI’s 30s IPC read timeout) to avoid EAGAIN.  
- Design goal: avoid dumping full DOM/tool schemas into agent context; snapshot refs are the AI-facing API.  
- Third-party token benchmarks exist (e.g. CLI vs Playwright MCP); **treat as secondary** — not reproduced here.

### Notable limitations

- Chrome-first; Firefox/WebKit not the default CDP path (Safari via separate WebDriver path).  
- Annotated screenshots / a11y audits: CDP path; Safari/WebDriver lacks `--annotate`.  
- `--allowed-domains` rejects CDP auto-connect / profile / restore paths (containment can’t be installed early).  
- Overlay-blocked clicks require dismiss + fresh snapshot (documented).  
- Accessibility quality of the page becomes the agent API surface (ARIA roles/names).

---

## 3. browser-use

**Repo:** https://github.com/browser-use/browser-use  
**License:** MIT (open-source library; cloud services have separate ToS)  
**Docs:** https://docs.browser-use.com  
**Related:** [Browser Harness](https://github.com/browser-use/browser-harness) (CLI), Cloud API, Rust beta agent (`browser_use.beta`)

### What it is

An open-source **browser agent framework** (Python library + CLI/Harness + optional hosted cloud). The agent loop observes browser state (DOM/a11y/screenshot), calls an LLM for structured actions, executes tools, repeats until `done`. Positions: (1) fully hosted cloud agent, (2) CLI for coding agents, (3) local Python `Agent`.

### Architecture shape

**Remote-drive / CDP-attached Chromium** (framework + runtime split):

- Local: launch or attach to Chrome/Chromium (CDP); `Browser.from_system_chrome()` reuses system profile.  
- Cloud: managed browsers exposing CDP + live preview.  
- Agent framework owns sense–think–act; browser process is separate infrastructure (local or cloud).  
- README/docs describe evolution toward typed CDP clients / harness; community architecture notes (secondary) describe event-bus + watchdogs — verify against current tree if building on internals.

### Agent + human shared browsing

| Mode | Evidence |
|------|----------|
| Real Chrome profile | Docs: `Browser.from_system_chrome()` — cookies/logins; note: may need Chrome fully closed first |
| Visible local browser | `headless=False` / headed profile so human can watch |
| CLI attach | Browser Harness CLI attaches to running Chrome via CDP (tabs, cookies, extensions, logins) |
| Cloud live preview | Docs: `live_view_url` — iframe embed; **URL is a credential** (anyone with it can interact) |
| Observation vs control | Live preview can be interactive (treat as takeover-capable remote view) |

True “same window as human typing” is strongest on **local CDP attach / system Chrome**; cloud is a mirrored remote session with live view.

### Cross-platform reality

Python ≥ 3.11 (docs examples use 3.12). System Chrome paths documented for macOS, Windows, Linux. Cloud for headless/serverless. Separate TS/JS harness repos exist for JS agents.

### Speed / latency / process model (evidence-backed)

- Chrome memory + parallel agents called out as hard in README → recommend Cloud for production scale.  
- Cloud pricing cited in product copy (e.g. browser-hour rates on marketing pages) — **verify current pricing** before planning; not treated as a speed metric here.  
- Step loop latency dominated by LLM + page load; no official ms-level IPC numbers found in README.

### Notable limitations

- Profile sync (cloud): cookies, **not** localStorage / IndexedDB / extensions (docs / FAQ).  
- CAPTCHA: cloud stealth helps; “no configuration guarantees every CAPTCHA” (FAQ).  
- Google Search may block automated browsers; docs suggest DuckDuckGo for real-browser demos.  
- Open-source agent vs hosted Cloud API are different APIs (docs agent instructions).

---

## 4. Playwright / CDP patterns used by agent products

### What this layer is

**Chrome DevTools Protocol (CDP)** is the low-level control plane for Chromium. Playwright and Puppeteer are higher-level libraries that speak CDP (Playwright also abstracts Firefox/WebKit). Agent products sit **above** this layer: they choose how to *observe* (a11y tree, DOM, screenshots) and *act* (refs, selectors, coordinates).

Evidence:

- Lightpanda: CDP underlies Puppeteer/Playwright; choosing abstraction level ≠ choosing a different wire protocol ([blog](https://lightpanda.io/blog/posts/cdp-vs-playwright-vs-puppeteer-is-this-the-wrong-question)).  
- serp.fast “agent browser stack”: frameworks (Browser Use, Stagehand, Skyvern, Playwright) vs infrastructure (Browserbase, Steel, Kernel, …) connect over **CDP endpoints** — swap runtime ≈ connection string ([guide](https://serp.fast/guides/agent-browser-stack)).  
- Vercel agent-browser: snapshot + refs via Accessibility API over CDP.  
- Stagehand (Browserbase): Playwright-style APIs + a11y trimming; “runs as an extension next to the browser” for latency ([README](https://github.com/browserbase/stagehand)).  
- Skyvern: Playwright-compatible SDK + vision/LLM; AGPL-3.0 ([repo](https://github.com/Skyvern-AI/skyvern)).  
- Commercial CUA-style products (e.g. Operator lineage): screenshot + mouse/keyboard — often Playwright or OS-level input; treat product claims as secondary unless primary docs are open.

### Architecture shape

| Pattern | Observation | Action | Typical products |
|---------|-------------|--------|------------------|
| A11y snapshot + refs | Compact tree | Click/fill by `@ref` | agent-browser, Stagehand observe |
| DOM / indexed elements | Annotated DOM | Tool actions | browser-use (classic loop) |
| Vision / CUA | Screenshots | Coordinates / keys | Operator-class, Skyvern vision path |
| Hybrid | A11y + screenshot | Refs + fallback vision | Many production agents |
| Extension-in-browser | Same as host page | chrome.debugger / scripting | nanobrowser |

### Agent + human shared browsing (pattern level)

- **Attach CDP to user’s Chrome** → same session (agent-browser `--auto-connect`, browser-use real browser, Stagehand `userDataDir`).  
- **Cloud browser + Live View** → mirrored session; HITL pause/resume (Browserbase [HITL template](https://browserbase.com/templates/agent-with-human-in-loop); Skyvern take/cede control in browser stream UI — [PR #3054](https://github.com/Skyvern-AI/skyvern/pull/3054)).  
- **Extension** → same session, side-panel UX (nanobrowser).

### Cross-platform / speed posture

- Playwright: Chromium + Firefox + WebKit; heavier Node/browser install.  
- Raw CDP / Chrome-only CLIs: leaner agent context, Chrome-centric.  
- Managed CDP clouds: latency = network + provider; local CDP: process-local WebSocket.

### Licenses (representative)

| Project | License | URL |
|---------|---------|-----|
| Playwright | Apache-2.0 | https://github.com/microsoft/playwright |
| Stagehand | MIT | https://github.com/browserbase/stagehand |
| Skyvern | AGPL-3.0 | https://github.com/Skyvern-AI/skyvern |

### Notable limitations (pattern level)

- Bot detection / `navigator.webdriver` / fingerprinting on automation Chrome.  
- Cross-browser Playwright ≠ identical CDP semantics.  
- Token cost of dumping full a11y trees (motivation for snapshot filters / CLI designs).  
- AGPL (Skyvern) vs MIT/Apache for redistribution constraints.

---

## 5. CEF / Tauri / webview alternatives (evidenced projects only)

### 5.1 wrymium (Tauri + CEF)

**Repo:** https://github.com/gxcsoccer/wrymium  
**License:** MIT  

**What:** CEF-powered WebView backend for [wry](https://github.com/tauri-apps/wry), bringing **consistent Chromium** to Tauri apps with Rust (vs Electron’s Node). Includes `cdp-test` example and a “Claude Desktop clone” example with Browse WebView.

**Architecture:** Tauri 2 → patched `tauri-runtime-wry` → wrymium-as-wry → `cef` / `cef-dll-sys`. Explicit contrast table vs Electron and stock Tauri webviews.

**Shared browsing:** Infrastructure for embedding Chromium + CDP in a desktop shell — **not** a full agent-browser product. CDP bridge example evidences agent-driveability of the embedded view.

**Platforms / speed:** README benchmarks on Apple M2 Max (bundle ~257 MB vs Electron ~247 MB; app binary 4.6 MB vs ~49 MB; main-process memory 198 vs 285 MB; IPC ~0.48 ms). Total memory higher than Electron in their table due to CEF process isolation. Cross-platform CEF packaging complexity implied by tooling (`cargo wrymium`).

**Limitation:** Bundle size still Chromium-class; CEF subprocess / bundler complexity vs stock Wry.

### 5.2 OpenHuman CEF notes (historical — important)

**Docs:** https://tinyhumans.gitbook.io/openhuman/developing/cef  

**Evidence:** OpenHuman **formerly** shipped CEF via forked `tauri-runtime` specifically because **stock Tauri webviews (WKWebView / WebView2 / WebKitGTK) do not expose CDP**. They used CDP for embedded WhatsApp/Slack/etc. scanners.  

**Current status (same page, warning banner):** Historical. Desktop shell **no longer ships CEF**; builds on stock Tauri Wry; project forbids restoring CEF/CDP-scanner assumptions. Kept as design background.

**Crew takeaway:** Real project evidence that (a) CDP is the reason teams consider CEF over native webviews, and (b) CEF maintenance cost can push teams **back** to stock webviews when CDP-in-webview is no longer required.

### 5.3 AIAnytime/agent-browser (Tauri + native WebView)

**Repo:** https://github.com/AIAnytime/agent-browser  
**License:** MIT  

**What:** Desktop “agentic browser”: React UI + Tauri Rust backend + Node ReAct agent + **Tauri WebView** (platform native webview — **not** CEF per README architecture table).

**Shared browsing:** Embedded webview + sidebar chat in one window — human navigates; agent uses DOM tools (`search_dom`, `click_button`, …) on the same view.

**Platforms:** Tauri targets (macOS/Windows/Linux via native webviews). Consistency across OS engines is the classic Tauri tradeoff (WebKit vs WebView2 vs WebKitGTK).

**Speed:** Unknown (no benchmarks in README).  
**Limitation:** Demo/education-scale; OpenAI-centric; native webview ≠ full Chrome CDP automation surface.

### 5.4 tauri-agent-plugin

**Repo:** https://github.com/byeongsu-hong/tauri-agent-plugin  

**What:** Plugin so agents/CLI/MCP can drive a **Tauri v2 app** (snapshot, find, click, eval). Default Wry; optional **`cef` feature** when the app supplies a CEF runtime. Adopting docs describe guest instrumentation + optional VNC/noVNC advertisement for human/QA view.

**Shared browsing:** Agent drives the app’s own webview(s); VNC path is observation/control for QA, not a browser product.

---

## Evidence comparison table

| Dimension | Nanobrowser | Vercel agent-browser | browser-use | Playwright/CDP agent pattern | CEF/Tauri peers |
|-----------|-------------|----------------------|-------------|------------------------------|-----------------|
| **Embedding / control model** | Chrome **extension** in host browser | **CDP remote-drive** (Rust daemon); optional attach | **CDP** local or cloud + Python/TS agent loop | Framework over CDP/Playwright | **CEF-in-Tauri** (wrymium, historical OpenHuman); **native WebView** (AIAnytime); plugin for app webview |
| **Electron?** | No | No | No | Optional (apps can expose CDP) | Explicitly positioned as Electron alternative (wrymium) |
| **Shared human+agent browsing** | Same Chrome session + side panel | Headed, WS stream pair-browse, dashboard, profile/CDP attach | System Chrome / CDP attach; cloud live preview (interactive URL) | Attach vs cloud Live View vs HITL pause | Same embedded webview; CDP for drive; OpenHuman retired CEF |
| **Takeover / HITL** | Human in same browser (implicit) | Stream input + pin-tab; confirm-actions gates | Live preview can interact; pause patterns via custom tools | Browserbase HITL template; Skyvern take/cede | VNC/QA view (plugin); not standardized |
| **Platforms** | Chrome, Edge | macOS/Linux/Windows native; iOS via Appium; cloud providers | macOS/Win/Linux + cloud | Playwright multi-browser; CDP Chrome-centric | Tauri OS matrix; CEF = Chromium consistency at pack cost |
| **Speed posture (documented)** | Local extension; LLM-bound | Persistent daemon; compact snapshots; 25s default timeout | Local vs cloud scale; Chrome RAM called out | A11y trim / extension-next-to-browser (Stagehand claim) | wrymium IPC ~0.48 ms; CEF bundle ~250 MB class |
| **License** | Apache-2.0 | Apache-2.0 | MIT (lib) | Playwright Apache-2.0; Stagehand MIT; Skyvern AGPL-3.0 | wrymium MIT; AIAnytime MIT |
| **Primary packaging** | Extension store / zip | npm / brew / cargo CLI | pip/uv library + CLI + cloud | npm Playwright; SDKs | Tauri desktop apps / crates |

---

## Open questions (no recommendation)

1. **Same-session UX goal:** For BrowserPeers, is the target “agent drives *my* logged-in Chrome” (extension / CDP attach) or “agent owns an isolated browser with optional live view” (cloud / headed daemon)? Peers prove both; product shape differs sharply.

2. **CDP requirement vs shell weight:** OpenHuman’s CEF→Wry retreat asks: do we *need* CDP inside an embedded shell, or is attaching to system/Chrome-for-Testing enough? wrymium shows CEF+Tauri is viable but Chromium-bundle-sized.

3. **Observation API:** Snapshot+refs (agent-browser, Stagehand) vs annotated DOM (browser-use) vs vision (Skyvern/CUA) — which failure modes matter for Firstmate tasks (iframes, canvas, shadow DOM)? Nanobrowser release notes already hit iframe/sandbox loops.

4. **Token / context budget:** Official agent-browser design centers compact refs; third-party MCP-vs-CLI token numbers are suggestive but **unverified here**. Need an in-house measurement protocol if this is a gate.

5. **License constraints:** AGPL (Skyvern) vs Apache/MIT peers — redistribution and SaaS copyleft risk for crew packaging.

6. **Human takeover protocol:** Skyvern/Browserbase have explicit take/cede or askHuman; agent-browser has stream+confirm-actions; nanobrowser relies on same-tab presence. What semantics does BrowserPeers need (observation-only vs dual control vs exclusive lock)?

7. **Anti-bot / authenticity:** Real Chrome profile vs Chrome for Testing vs cloud stealth — peers disagree; CAPTCHA guarantees are universally soft in primary docs.

8. **Firefox/Safari:** Only Playwright-class and Stagehand/Playwright paths strongly cover non-Chromium; extension and CDP-CLI peers are Chromium-first. Is multi-engine in scope?

9. **Process model for desktop app:** If Firstmate ships a desktop shell, evidence supports Tauri+native webview **or** Tauri+CEF **or** “no embed — drive external Chrome.” Electron is **not** required by any of the studied open stacks.

10. **Maintenance reality:** CEF forks (OpenHuman historical, wrymium patches) and extension store review lag (nanobrowser) are operational costs, not just technical ones — how does the crew budget them?

---

## Source index (primary)

| Stack | Sources |
|-------|---------|
| Nanobrowser | https://raw.githubusercontent.com/nanobrowser/nanobrowser/master/README.md ; https://nanobrowser.ai/ ; Chrome Web Store listing |
| Vercel agent-browser | https://raw.githubusercontent.com/vercel-labs/agent-browser/main/README.md ; https://agent-browser.dev |
| browser-use | https://raw.githubusercontent.com/browser-use/browser-use/main/README.md ; https://docs.browser-use.com/open-source/customize/browser/real-browser ; https://docs.browser-use.com/cloud/browser/live-preview ; https://docs.browser-use.com/open-source/browser-use-cli |
| Playwright/CDP peers | https://github.com/browserbase/stagehand ; https://github.com/Skyvern-AI/skyvern ; https://serp.fast/guides/agent-browser-stack ; https://lightpanda.io/blog/posts/cdp-vs-playwright-vs-puppeteer-is-this-the-wrong-question ; https://browserbase.com/templates/agent-with-human-in-loop |
| CEF/Tauri | https://raw.githubusercontent.com/gxcsoccer/wrymium/main/README.md ; https://tinyhumans.gitbook.io/openhuman/developing/cef ; https://raw.githubusercontent.com/AIAnytime/agent-browser/main/README.md ; https://github.com/byeongsu-hong/tauri-agent-plugin |

---

*End of evidence report. No product recommendation — table + open questions only.*
