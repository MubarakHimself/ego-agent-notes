# Research synthesis (Firstmate) — 2026-09-22

Captain brief: understand public ego-lite + peers + decision models; reverse-engineer understanding, not steal code; no stack until evidence.

## What ego (lite) actually is

- **Closed product:** Chromium-based browser (DMG), Spaces UI, injects `globalThis.ego`. Not in the public git tree.
- **Open (MIT):** `package/ego-browser` Node/TS CDP harness + `skills/ego-browser` agent skill.
- **Platforms:** macOS shipping; Windows beta signals; Linux roadmap.
- **No Electron claim** in the public tree.

## Decision-model layer

| Option | Open? | Fit |
|--------|-------|-----|
| TypeSafe Jev | Closed SaaS API | Fast hosted `/v1/systemone`; needs API key/money |
| Laya + laya-browser | Apache-2.0 | Local System-1; browser-tuned head; can replace Jev in jev-ultrafast |
| Open-Jev | MIT/Apache + Qwen | Local `/v1/systemone`; needs GPU for serious use |

## Peer architecture families (evidence)

1. **Extension in your Chrome** — Nanobrowser (Apache-2.0)
2. **CDP remote-drive** — Vercel agent-browser (Rust), browser-use / jev-ultrafast (Python)
3. **Desktop shell embed** — Tauri + native webview, or Tauri + CEF (heavy); Electron not required by studied peers

## Implication

Building a personal tool can reuse MIT ego-browser *patterns* and open decision heads without cloning the closed Chromium app. Stack choice = which family above + which decision head — captain call next.

Full reports in `reports/`.
