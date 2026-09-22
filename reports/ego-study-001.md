# ego-study-001 — Deep study: citrolabs/ego-lite

**Task id:** `ego-study-001`  
**Kind:** scout · **Project:** `ego-agent`  
**Date:** 2026-09-22 (Africa/Nairobi)  
**Repo:** https://github.com/citrolabs/ego-lite  
**Local clone:** `/workspace/studies/ego-lite` @ `dca7003` (shallow; merge of PR #407)  
**Site:** https://lite.ego.app  

**Captain lock:** reverse-engineer *understanding* from public code/commits/PRs — do **not** steal code; no proprietary binary decompile; research-first; do **not** pick Electron or any final tech stack yet.

**Supersedes:** `egostudy-first-recon-001.md` (consolidated into this report).

---

## 1. Repo map — skills vs package/ego-browser vs binary

| Layer | Path / artifact | In git? | Role |
|---|---|---|---|
| Closed **ego lite browser** | DMG from `cdn.ego.app` / https://lite.ego.app | **No** | Chromium-based product; Spaces UI; injects `globalThis.ego`; ships bundled `ego-browser` helper inside the app bundle |
| Open **ego-browser harness** | `package/ego-browser/` | Yes (MIT) | Node/TS CDP helper runtime agents script against |
| Agent **skill** | `skills/ego-browser/` | Yes (MIT) | `SKILL.md`, install script, API refs, site `learnings/` |
| Native bridge **docs** | `docs/native-*.md`, `docs/local-runtime-development.md` | Yes | Contract for closed bindings the open harness consumes |
| Skills **spec** | `spec/agent-skills-spec.md` | Yes | Agent-skills format |
| Plugin packaging | `.agents/`, `.claude/`, `.codex/`, `.claude-plugin/` | Yes | Host-specific skill/plugin packaging |
| Root pointer | `install.md` → `skills/ego-browser/references/install.md` | Yes | Agent install recipe |

**Explicit separation (public):**

- `README.md`: “The contents of this repository are released under the MIT License. The ego lite browser is a separate, free download.”
- `AGENTS.md`: repo is the open harness + skill — **not** the browser. App provides `globalThis.ego`; app’s `ego-browser` binary embeds this runtime.
- `package/ego-browser/README.md`: “The Node.js helper layer that runs inside the `ego-browser` Chromium browser.”

**Data flow (open side):**

```text
Agent (skill / CLI)
  → ego-browser nodejs <<'EOF' … EOF   (or --sdk-path local dist)
  → package/ego-browser runtime (runMain / installEgoSdk)
  → helpers → browser-runtime CDP
  → globalThis.ego (closed) → Ego Lite + Chromium
```

**Key open modules** (`package/ego-browser/src/`): `index.ts`, `run.ts`, `helpers.ts`, `public-api-schema.ts`, `page-model.ts`, `browser-runtime.ts`, `cdp-eval.ts`, `element-resolver.ts`, `page-ref-registry.ts` / `page-ledger.ts`, `driver/*` (nav, pointer, keyboard, observe, waits, files, downloads…), `learning/*`, `env.ts`, `state.ts`.

---

## 2. Chromium evidence (public only — no binary work)

Public docs and scripts repeatedly identify the product as Chromium-based:

| Evidence | Cite |
|---|---|
| Stack diagram: Ego Lite CLI → local SDK → “Ego Lite and Chromium” | `docs/local-runtime-development.md` |
| “helper layer that runs inside the `ego-browser` Chromium browser” | `package/ego-browser/README.md` |
| Skill: “ego-browser (ego-lite) is a Chromium browser…” | `skills/ego-browser/SKILL.md` frontmatter |
| Install script: “A Chromium app bundle may contain multiple versions; prefer … Versions/Current/Helpers/…” | `skills/ego-browser/scripts/install.sh` (`find_ego_browser_in_app`) |
| API refs mention Chromium download id / suggested filename | `skills/ego-browser/references/api.md` |
| E2E PR title references Chromium 154 PDF viewer | merged PR #420 |

**Not found in public tree:** Electron packaging, CEF branding, or Chromium source forks. No claim that the closed app is Electron. Captain lock honored: stack choice deferred.

---

## 3. Spaces (task spaces)

Isolated browsing contexts with ownership `agent` | `user`.

**Native (`docs/native-bindings-api.md`):**

- Create: `ego.createTaskSpace(name, profileId?)` → numeric `id`
- Select (local to this Node invocation): `ego.useTaskSpace(id)`
- List / claim user Spaces: `listTaskSpaces`, `claimTaskSpace(id, name?)`
- Lifecycle: `closeTaskSpace`, `completeTaskSpace` (leave Space for user), `handOffTaskSpace` / `takeOverTaskSpace`
- Progress UI: `setAgentTaskState(state)`
- While user-in-control: mutating APIs + raw CDP blocked → `EGO_TASK_SPACE_USER_IN_CONTROL` (reason keys for permissions / `manual_takeover`)

**Open harness (`AGENTS.md`, `page-model.ts`, skill):**

- Scripts use `taskSpace(nameOrId)` → `TaskSpace` + labeled `Page` objects
- `task.pages()` / `task.tabs()`, `task.handOff()`, `takeOverTaskSpace(spaceId)`
- v1 global helpers kept only for compatibility

---

## 4. Chrome profile import

Two related but distinct ideas:

1. **First-run product UX (closed app):** `README.md` — on first launch, ask whether to migrate Chrome data (logins, cookies, extensions, bookmarks). Data stays on device.
2. **Agent install path:** `skills/ego-browser/references/install.md` — after DMG install, user completes onboarding in GUI: “Choose to import data from Chrome or another browser as needed.” Script only downloads/installs/launches on macOS; migration is GUI onboarding, not open-source logic in this repo.
3. **Runtime profile selection (documented native API):** `ego.listProfiles()` → `{ id, name, isDefault }`; pass `profile.id` into `ego.createTaskSpace(name, profileId)`. Profile is fixed at Space creation. Internal “Ego authentication profile” is not exposed.

Open repo does **not** contain Chrome profile-copy implementation — only docs + profile IDs for task creation.

---

## 5. CDP / ego-browser API

**Native CDP bridge:** `ego.sendCDPMessage(json)`, `ego.onCDPMessage`, `ego.onSendCDPMessageError` (`docs/native-bindings-api.md`). Requires selected agent-owned task space.

**Open harness:**

- `browser-runtime.ts` — CDP transport over `ego.sendCDPMessage`, session attach/cache (2s TTL), 10k event queue, dialog tracking
- `cdp-eval.ts` — `cdp()` / `js()`
- Higher-level: snapshot, click/fill/type, nav, waits, screenshots, downloads, site learnings
- v2 source of truth: `public-api-schema.ts`; agent docs: `skills/ego-browser/SKILL.md`, `references/api.md`
- Escape hatches: `page.cdp` / `task.cdp`, `page.evaluate`; clearing-state docs warn that some CDP commands are profile-wide (`references/clearing-state.md`)

**Design intent (README):** code-composed JS helpers in one pass vs CLI tool-call loops → fewer tokens / faster complex tasks (vendor claim; not independently verified here).

**Local SDK override:** `ego-browser nodejs --sdk-path <dist/out/index.js>` (`docs/local-runtime-development.md`) — temporary; still needs installed app for native bindings.

---

## 6. Mac / Win / Linux evidence

| Platform | Evidence | Status |
|---|---|---|
| **macOS arm64 / x64** | README DMG badges; `install.sh` Darwin-only; E2E prefers macOS app helper path | **Shipping** |
| **Windows** | README: “Windows closed beta is coming soon”; merged PRs #409 (keyboard.paste clipboard on Windows), #408 (absolute paths for screenshot/fetch saveAs), #413 (PowerShell/cmd invocation docs) | **In progress / beta-oriented** in open harness; full product install still pointed at website |
| **Linux** | README + roadmap link; install.md: non-macOS → website | **Roadmap**, not shipping in public install script |

Install script hard-fails off Darwin: `uname -s` must be `Darwin` (`skills/ego-browser/scripts/install.sh`, `references/install.md`).

---

## 7. Licenses

| Asset | License |
|---|---|
| Everything in this git repository | **MIT** — `LICENSE` © 2026 CitroLabs |
| Ego lite browser binary / Chromium product | **Not** under that MIT grant; separate free download (terms of the product, not in this tree) |

Implication for later personal tooling: reuse patterns and MIT harness/skill under license terms; do not treat the closed browser as open source. Do not paste proprietary binaries into our repos.

---

## 8. Notable commits / PRs (design intent)

Shallow clone HEAD: `dca7003`. Themes from recent history + `gh pr list --state merged`:

| Theme | Examples | Intent signal |
|---|---|---|
| **v2 API / TaskSpace baseline** | PR #386 / #385 / #384 promote v2; #392 / #402 beta-dev | Hard cut toward TaskSpace/Page v2 as the agent surface |
| **Snapshot + iframe + ref lifetime** | #375, #358, #335, #378, #377, #364 | Engine+SDK work so snapshots/refs survive actions, evaluate, iframe lifecycle |
| **Playwright-ish downloads** | #353 | Download UX aligned with familiar automation APIs |
| **Windows readiness** | #409, #408, #413 | Clipboard paste, path rules, shell invocation — preparing Win agents |
| **Plugin packaging for many hosts** | #391, #397 | Ship skill/plugins to Claude/Codex/etc. directories |
| **Docs / i18n** | #407, #406, #383 | Positioning + install clarity; architecture notes in CONTRIBUTING |
| **Dialog / task-space safety** | #415, #419, #423 | Fail fast on JS dialogs; validate locators; clearer Page lifecycle errors |

Commits worth citing for Spaces/snapshots specifically: `feat(ego-browser): add frame-aware subtree snapshots` (`45038fc`), `fix(ego-browser): preserve Page ref identity…` (`4d91094`), `docs(ego-browser): clarify task space reuse` (`155b98b`).

---

## 9. Open questions

1. **Exact Chromium fork/patch set** for “strongest Snapshot” (iframes) — only claimed in marketing + native requirement docs; implementation is closed.
2. **Windows/Linux ship timeline and install artifacts** — README/roadmap only; no public Win/Linux install script in repo.
3. **How Chrome migration maps onto `listProfiles()` ids** after import (Default vs Profile N) — undocumented beyond first-run UX.
4. **Bundled vs repo `ego-browser` version skew** — app embeds a build of this runtime; `--sdk-path` exists because they can diverge.
5. **Security model of claiming user Spaces** — docs say get user approval; enforcement details opaque outside error codes.
6. **Whether a personal tool should depend on ego lite as host vs reimplement Spaces elsewhere** — out of scope for this scout (no stack pick per captain lock); hand to later synthesis with DecisionStudy / BrowserPeers.

---

## 10. Pointers for sibling scouts

- **DecisionStudy (`ego-decision-001`):** Laya/Jev models — EgoStudy does not own that.
- **BrowserPeers (`ego-peers-001`):** compare Browser-Use / Vercel agent-browser / Atlas / Comet at product level; EgoStudy only notes README comparison table exists.

---

## Sources

- Local: `/workspace/studies/ego-lite` @ `dca7003`
- Paths cited above; GitHub `gh pr list` for merged PR metadata
- Prior notes: `egostudy-first-recon-001.md`

**Out of scope / not done:** proprietary DMG download, binary reverse engineering, Electron/stack recommendation.
