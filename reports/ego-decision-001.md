# DecisionStudy baseline — System-1 / browser decision models

**Task id:** ego-decision-001  
**Date:** 2026-09-22 (Africa/Nairobi, EAT / UTC+3)  
**Scope:** Public docs and public repos only. No paid TypeSafe API calls; no scraping of private/paid content. Does **not** cover ego-lite browser shell architecture.

---

## Executive summary

| Project | Plain-terms identity |
| --- | --- |
| **TypeSafe Jev** | Closed, hosted “System One” decision API: send unstructured `state` + typed questions (`choice` / `noul` / `score`); get calibrated probabilities back via `POST /v1/systemone`. No public weights. |
| **Open-Jev** (`Zefan-Cai/Open-Jev`) | Independent open research stack inspired by Jev: MIT code + Apache-2.0 LoRA/decision-head packages on Qwen; same `/v1/systemone`-shaped local server. Explicitly does **not** claim to reproduce TypeSafe RLCD, private data, or advertised speedups. |
| **convaiinnovations/laya** | Open Apache-2.0 encoder-based System-1 family (ModernBERT / mmBERT, ~322–421M). Python SDK (`pip install laya`); ~33 ms/question claims on T4 GPU. General typed decisions, not a browser agent by itself. |
| **cklxx/laya-browser** | Hugging Face release: Laya fine-tuned as a **browser-agent decision head**, with a local `/v1/systemone`-compatible server and a patch so `browser-use/jev-ultrafast` can point at localhost instead of TypeSafe. |
| **browser-use/jev-ultrafast** | MIT browser agent that uses TypeSafe Jev (hosted) for operation/target choice and a small LLM for `TYPE_TEXT`. Chrome/Chromium via Browser Harness / CDP. Published end-to-end demos (e.g. Google Flights ~7.1 s); **not** an ONNX-in-browser model. |

---

## 1. TypeSafe Jev (closed)

### What it is

TypeSafe AI’s flagship **System One** model class: unstructured state in, typed probabilistic decisions out (no free-form text generation). Public launch post (2026-09-15) frames Jev as optimized with **RLCD** (Reinforcement Learning for Calibrated Decisions) and parallel sampling rather than autoregressive token generation.

### API / integration surface (public)

Verified from anonymous OpenAPI at `https://api.typesafe.ai/openapi.json` and public how-to docs:

| Item | Public value |
| --- | --- |
| Evaluate | `POST https://api.typesafe.ai/v1/systemone` |
| List models | `GET https://api.typesafe.ai/v1/models` |
| Auth | `Authorization: Bearer <API_KEY>` |
| Body | `model`, `state`, `questions` (map of named questions) |
| Question types | `choice` (criteria map), `noul` (yes/no probability), `score` (ordered rubric) |
| Response | `model`, `answers` (same keys), `usage` (`input_tokens`, `output_tokens`) |
| Model aliases (docs) | `jev-latest`, pinned versions such as `jev-1.13.0`, `jev-preview` |

Documented limits from public how-to (`jevtypesafeai.com/how-to-use`): ~64k tokens state+questions; 32k for state + longest question; rate limits **250,000 tokens/s** and **1,200 req/min**; choice criteria up to **255** options; score criteria **2–10** levels.

SDKs referenced in public ecosystem pages: Python `typesafe-sdk`, JS `@typesafe-ai/sdk` (not re-verified line-by-line for this report).

### License / pricing surface (public)

- **Weights / method:** Closed proprietary hosted service. No public model weights, training code, or full RLCD method disclosure found in public docs.
- **Official launch pricing** (TypeSafe blog, 2026-09-15): **$0.042 / 1M input tokens**; **output tokens free**.
- **Docs snippet** (`docs.typesafe.ai/llms-full.txt` search hit): example price tuple for jev-1.12 as of Sep 2026 matches **$0.042 / $0.00**.
- **Third-party / reseller pages** (e.g. `jevtypesafeai.com/pricing`) advertise higher metered rates ($0.25–$0.42/M) as a convenience layer — treat as **non-canonical** relative to TypeSafe’s own $0.042 figure.
- Access: early-access / console API keys (waitlist / sign-in) per TypeSafe site; this research did **not** create accounts or call paid endpoints.

### Hardware expectations

- For **API consumers**: HTTPS client only; TypeSafe hosts inference. Public launch claims **70–500 ms** end-to-end response times (vendor range; measurement conditions partially disclosed as West Coast laptop → their service).
- **No public customer-facing GPU/CPU self-host recipe** for closed Jev was found. Underlying cluster hardware for TypeSafe’s service is **not published** in the sources reviewed.

### Browser / CDP fit

Jev itself is a **decision API**, not a browser controller. Browser fitness is via integrators (notably `browser-use/jev-ultrafast`), which send DOM-derived `state` + dynamic `choice` questions to `/v1/systemone`.

### Open questions / unknowns

- Exact self-host availability, enterprise license terms, and SLA — not verified from free public pages alone.
- Whether OpenAPI `info.version` `0.2.0` maps 1:1 to product versioning of `jev-1.13.x`.
- Live model list contents require an authenticated `GET /v1/models` (not called).

---

## 2. Open-Jev (`Zefan-Cai/Open-Jev`)

### What it is

Independent open research project: “open probability decisions” with Qwen3.5-2B / 9B (and 27B in progress) LoRA adapters **plus a trained scalar decision head**, served with a Jev-shaped local API. Project site and README state it is **inspired by** TypeSafe Jev and **does not** reproduce proprietary RLCD, private weights/data, or TypeSafe’s advertised speedups.

**Primary repo:** https://github.com/Zefan-Cai/Open-Jev (public, not archived; ~154★ as of fetch time)  
**Site:** https://zefan-cai.github.io/open-jev/  
**HF:** collection `ZefanCai/open-jev`; models `Open-Jev-2B`, `Open-Jev-9B`; dataset `ZefanCai/Open-Jev`

**Note on naming:** Many similarly named community forks exist (`daseinlabs/open-jev`, `SiliconLabAI/OpenJev`, `openjev/openjev` on HF, etc.). This baseline treats **`Zefan-Cai/Open-Jev` + zefan-cai.github.io** as the primary “Open-Jev” lineage matching the DecisionStudy target.

### API / integration surface

- Local server: `python -m jev.server --checkpoint ... --device cuda:0 ...` (default examples open **http://127.0.0.1:8791**).
- Python client: `jev.client.Client().ask(state=..., questions=...)`.
- HTTP shape documented as **`POST /v1/systemone`**-compatible (partial hosted-API compatibility claimed in README capability table).
- Question types: Choice / Noul / Score with calibrated probabilities; no autoregressive answer generation for the decision path.

### License

- **Original source code:** MIT (`LICENSE` in repo, Copyright 2026 Open-Jev contributors).
- **Trained LoRA adapters / decision heads:** Apache-2.0 under model cards, subject to **pinned upstream Qwen** terms.
- Dataset / third-party engines: retain their own licenses (see `THIRD_PARTY_NOTICES.md`).

### Hardware

- Published inference workflow targets **Linux + GPU** (`cuda:0` in docs).
- Latency report / site: warm **H100** loopback measurements for Open-Jev-2B (heterogeneous vs hosted HTTPS Jev; not matched-hardware).
- 27B training described as running on **four H100 GPUs** (training in progress as of README notes dated ~2026-09-21 UTC).
- CPU-only serving of the published Qwen-LoRA packages is **not** presented as the primary measured recipe.

### Browser / CDP fit

- README lists **community adapters** including “Browser DOM” (state/action interfaces); external browser environments are **not bundled**.
- Open-Jev is **not** itself a Chromium CDP agent. Browser use would require a separate harness sending structured state into `/v1/systemone`.

### Relation to closed Jev

| Dimension | Closed Jev | Open-Jev |
| --- | --- | --- |
| Ownership | TypeSafe proprietary | Independent research |
| Interface idea | System One / typed probs | Compatible-shaped API |
| Method claim | RLCD (closed) | Own LoRA + scalar head; **no** RLCD reproduction claim |
| Weights | Hosted only | Open adapters + need Qwen base |
| Speed claims | Vendor 70–500 ms E2E | Local ms figures on H100; honest caveats vs HTTPS |

### Open questions / unknowns

- Full parity of response schema edge-cases vs TypeSafe OpenAPI (e.g. confidence fields, usage metering) — “partial compatibility” only.
- Final 27B checkpoint quality — pending per README.
- Whether any production browser agent ships with Open-Jev as default — not found as a first-party product.

---

## 3. convaiinnovations/laya

### What it is

Open-weight **non-autoregressive System 1 decision engine** from Convai Innovations: bidirectional encoder + decision head; typed `choice` / `score` / `noul` over a `state` in one forward pass; trained with an RLCD-style proper-scoring-rule recipe (as described by the authors). Multilingual routing across 100+ languages via a built-in `Router`.

**Hub:** https://huggingface.co/convaiinnovations/laya (Apache-2.0 card)  
**Code:** https://github.com/NandhaKishorM/laya  
**Docs/marketing:** https://laya.convaiinnovations.com/  
**PyPI:** `pip install laya`

Checkpoints (bundled under the main HF repo / subfolders):

| Checkpoint | Backbone | Params | Context (default) |
| --- | --- | --- | --- |
| English root | ModernBERT-large | ~421M | 512 |
| multilingual | mmBERT-base | ~322M | 1024 (encoder up to 8k claimed) |
| typed-decisions | ModernBERT-large | ~421M | 1024 |

### API / integration surface

- SDK: `laya.load(...)`, `agent.predict(state, questions)`, `Router(preload=True).predict(...)`.
- Same three primitives as Jev-shaped System One (`choice` / `score` / `noul`).
- Presets in SDK: router / guard / moderation / triage question helpers (README).
- **Not** a hosted TypeSafe-compatible HTTP service by default; local Python inference. Community/adjacent projects add `/v1/systemone` servers (see laya-browser).

### License

- **Apache License 2.0** — confirmed via GitHub `LICENSE` and HF model card `license: apache-2.0`.

### Hardware

- Author-measured latency on **Tesla T4**: e.g. multilingual **32.8 ms** (1 question), **72.3 ms** (10 questions batched); English **39.5 ms** (1 q).
- CPU with preload: README cites **193–464 ms** per request (vs multi-second cold swaps if not preloaded).
- Fine-tune notebook targets free **2×T4** (Kaggle).
- Weights download sizes publicly cited ~647–808 MB per checkpoint (selective subfolder download).

### Browser / CDP fit

- Base Laya is a **general decision model**, not a browser/CDP stack.
- Authors and third parties note **near-chance** out-of-box browser element selection without domain fine-tuning (see laya-browser).
- High-cardinality choices (>~20 options at default `head_max_len`) degrade sharply vs Jev’s 255-option support — material for browser UIs with dozens of DOM candidates unless `head_max_len` / shortlisting is tuned.

### Open questions / unknowns

- Head-to-head numbers vs Jev on Laya’s site cite **third-party** Jev figures (authors state no TypeSafe API access) — treat cross-vendor accuracy deltas cautiously.
- Official HTTP `/v1/systemone` server in main Laya package vs only in forks — mainline README emphasizes Python SDK; systemone server path is prominent in **laya-browser**.

---

## 4. cklxx/laya-browser

### What it is

Public Hugging Face project that **fine-tunes Laya** into a usable decision head for **browser-use/jev-ultrafast**-style steps: operation + element target (+ related typed questions) with format transforms so element options fit the encoder head budget.

**URL:** https://huggingface.co/cklxx/laya-browser  
**No separate GitHub org repo** named `cklxx/laya-browser` was found; code ships **inside the HF repo** under `code/`.

Checkpoints described: `v10` (ModernBERT-large 421M), `v10s` / `v11s` (mmBERT-base 322M). Live-suite success rates claimed on a **16-task × 3** browser suite (e.g. v10s **62%** in README table; treat as author-reported, single-lab).

### API / integration surface

1. Direct: `laya.load("laya-browser/v10s")` then `predict(state, questions)` with format-v2/v3 conventions.
2. Drop-in TypeSafe-shaped server:

```bash
python apps/systemone_server.py 8791 /path/to/laya-browser/v10s 999
# then point jev-ultrafast at localhost (via project patch):
# TYPESAFE_BASE_URL=http://127.0.0.1:8791
```

3. Optional TileLang fast path (`uv sync --extra fast`) for fused CUDA kernels.

### How it relates to Chromium CDP

- **laya-browser does not implement CDP itself.**
- CDP / browser control remains in **jev-ultrafast + Browser Harness**.
- README extras for live suite: Chromium with **`--remote-debugging-port=9222`**, apply `code/jev-ultrafast.patch`, run suite scripts.
- Integration pattern: local System One HTTP server ↔ patched agent that redirects TypeSafe calls; CDP remains the observation/action channel.

**Important stock-code note:** Upstream `browser-use/jev-ultrafast` `jev_ultrafast/model.py` (fetched) hardcodes `https://api.typesafe.ai/v1/systemone`. Redirect via `TYPESAFE_BASE_URL` is described as coming from **laya-browser’s patch**, not from stock env vars in `.env.example`.

### License

- HF card: **`license: apache-2.0`** (same as Laya). Mind2Web used for training under its own license (training-only note on card).

### Hardware

- Trained/verified on **RTX 4070 Ti SUPER (16 GB)**.
- v10s: ~0.65 GB weights, ~1.5 GB VRAM with CUDA graphs; **17–23 ms** per 3-question browser step (author).
- Verify example: **35 ms** stock / **28 ms** TileLang fast path on a 65-option step (2026-09-21 clean env claim).
- Separate local **Qwen3-8B-AWQ** (sglang) used as text helper / DAgger teacher in their pipeline — not required solely for choice/noul heads if text is supplied elsewhere.

### Open questions / unknowns

- Longevity of the patch against upstream jev-ultrafast (114 open issues on ultrafast; fast-moving).
- Generalization beyond the 16-task suite / Mind2Web-derived training mix.
- Whether a standalone GitHub mirror will appear.

---

## 5. browser-use/jev-ultrafast

### What it is

MIT-licensed **browser agent** from Browser Use: one natural-language goal → dynamic indexed DOM action space → **TypeSafe Jev** chooses operation + target in one System One request → optional small OpenAI-compatible LLM generates text only for `TYPE_TEXT`.

**Repo:** https://github.com/browser-use/jev-ultrafast (~17k★ at fetch time)  
**License:** MIT (Copyright 2026 Browser Use)

### Speed claims (evidence-backed)

From repo `README.md` and `docs/performance.md` (primary):

| Claim | Detail |
| --- | --- |
| Demo video | Google Flights Zürich→London **7.073 s** at 1× (includes model + browser + waits) |
| Matched A/B | Median **9.450 s → 7.092 s** (−25%); CDP calls **1,092 → 101**; Jev requests **22 → 17**; 3/3 pairs verified |
| Other smokes | Wikipedia article open **2.798 s**; local hotel filter **1.896 s** |
| Median Jev latency in recording | **178 ms** (hosted API RTT, not local encoder) |

Authors explicitly limit statistical strength (sign-test p = 0.25; live web variance).

### Hardware / ONNX-in-browser

| Topic | Finding |
| --- | --- |
| Decision model compute | **Hosted TypeSafe API** by default — no local GPU required for Jev itself |
| Text helper | External OpenAI-compatible HTTP API (`TEXT_MODEL_*`) |
| Browser | Chrome/Chromium via **Browser Harness** (CDP); local browser process |
| ONNX / WebGPU / in-browser weights | **Not claimed** in README or performance.md for this repo |
| Local GPU decision heads | Only via third-party replacements (e.g. laya-browser server), not stock |

Separate community projects (e.g. PyPI `edgejev`) discuss ONNX CPU ports of Laya-like models; that is **out of scope** for stock jev-ultrafast and was not treated as a claim of this target.

### CDP / browser agent integration

- `jev_ultrafast/browser.py` + `snapshot.js`: atomic DOM snapshot, indexed controls, freshness guards, `observe()` / `act()`.
- Depends on **browser-harness** for CDP connection; doctor command `browser-harness --doctor`.
- Community notes (secondary) mention `BU_CDP_URL` for pointing at a separate Chromium — **not** documented in the main README excerpt reviewed; mark as **community-reported** until confirmed in harness docs.
- MVP limits (README): no full a11y-name algorithm; weak shadow DOM/frames/canvas/uploads/new tabs/nested scroll support; owned tabs share Chrome profile by default.

### API / env surface (stock)

From `.env.example` and `model.py`:

- `TYPESAFE_API_KEY`, `TYPESAFE_MODEL` (default `jev-latest`)
- Hardcoded POST to `https://api.typesafe.ai/v1/systemone`
- `TEXT_MODEL_API_KEY`, `TEXT_MODEL_BASE_URL`, `TEXT_MODEL`, `TEXT_MODEL_REASONING`

### Open questions / unknowns

- Cloud “Browser Use Ultrafast” waitlist product vs open-source parity.
- Exact Browser Harness telemetry defaults (`BH_TELEMETRY`) — discussed on secondary review sites; not fully audited here.
- Cost of a Flights run: token counts published; dollar total depends on TypeSafe billing (input $0.042/MTok official) plus text-helper charges.

---

## Comparison table

| Dimension | Closed Jev | Open-Jev (Zefan-Cai) | Laya | laya-browser | jev-ultrafast |
| --- | --- | --- | --- | --- | --- |
| **Openness** | Closed hosted API | MIT code + Apache adapters; need Qwen base | Apache-2.0 weights + code | Apache-2.0 HF release | MIT agent; depends on closed Jev **or** local replacement |
| **Speed signals** | Vendor 70–500 ms E2E; third-party ~236–276 ms p50 | Local H100 loopback ms (workload-dependent); mixed vs HTTPS Jev | ~33 ms/q on T4 (author) | 17–35 ms/step local GPU (author) | ~7.1 s E2E Flights; median Jev RTT ~178 ms in demo |
| **Browser fitness** | API only | Community DOM adapters; no CDP stack | Weak OOB for DOM choice; needs FT | Designed for jev-ultrafast browser steps | First-class CDP browser agent |
| **License** | Proprietary SaaS | MIT (+ Apache models, Qwen terms) | Apache-2.0 | Apache-2.0 | MIT |
| **Hardware** | Cloud (vendor) | GPU (H100 class for published timings) | GPU preferred (T4+); CPU slower | Consumer GPU (4070 Ti SUPER class) | Browser + network; GPU optional if local decision head |

### Laya vs Open-Jev vs closed Jev (decision-model layer)

| | Closed Jev | Open-Jev | Laya |
| --- | --- | --- | --- |
| Architecture signal | Proprietary System One | Qwen LoRA + scalar decision head | Bidirectional encoder (ModernBERT/mmBERT) + head |
| Cardinality | Up to 255 choices (docs) | Dynamic candidates; large-context workloads slower locally | Default head budgets struggle at 50–77 options unless tuned |
| Multilingual | Not clearly benchmarked publicly | Growing controls; not Laya’s 51-lang sweep | Explicit multilingual checkpoint + router |
| Self-host | No public weights | Yes (GPU) | Yes (GPU/CPU) |
| Browser-ready OOB | Via agents calling API | DIY | Near-chance until fine-tuned (laya-browser) |

---

## Practical constraints for plugging into a browser agent

1. **API shape convergence:** Closed Jev, Open-Jev server, and laya-browser’s `systemone_server` all orbit `state` + typed `questions` → typed `answers`. Stock jev-ultrafast currently **hardcodes** TypeSafe’s URL; local backends need a patch or fork (`TYPESAFE_BASE_URL` pattern).
2. **GPU vs cloud:** Hosted Jev → no local GPU for decisions, but network RTT (~150–300 ms class) sits in the agent loop. Local Laya/Open-Jev → VRAM + driver stack; can cut decision latency to tens of ms on GPU.
3. **CPU:** Laya documents hundreds of ms on CPU when preloaded; Open-Jev’s published path is GPU-centric. CPU may be acceptable for low-QPS demos, not for “ultrafast” loops.
4. **ONNX-in-browser:** **Not verified** as a capability of any of the five primary targets. Do not assume WebGPU/ONNX Runtime Web deployment without a separate project (e.g. community ONNX exporters).
5. **High-cardinality DOM:** Browser pages often expose dozens of interactive elements. Jev’s 255-option support is a practical fit; Laya-family models need larger `head_max_len`, format transforms (as in laya-browser v2/v3), or shortlisting.
6. **Dual-model pattern:** jev-ultrafast still needs a **text LLM** for typing. Replacing only the decision head does not remove that dependency.
7. **CDP safety:** Default harness may attach to a real Chrome profile; prefer isolated Chromium + remote debugging port for experiments.
8. **Licensing stack:** Combining MIT agent + Apache decision head + Qwen base + Mind2Web-trained weights requires respecting **each** license — especially Qwen terms for Open-Jev.

---

## Sources (primary preferred)

### TypeSafe Jev
- https://api.typesafe.ai/openapi.json
- https://typesafe.ai/blog/introducing-system-one-models-and-jev
- https://www.jevtypesafeai.com/how-to-use
- https://docs.typesafe.ai/llms-full.txt (via public search hit; pricing snippet)
- https://apis.io/apis/typesafe-ai/system-one-api/

### Open-Jev
- https://github.com/Zefan-Cai/Open-Jev (README, LICENSE)
- https://zefan-cai.github.io/open-jev/
- https://huggingface.co/ZefanCai/Open-Jev-9B (and 2B / collection)

### Laya
- https://github.com/NandhaKishorM/laya (README, LICENSE)
- https://huggingface.co/convaiinnovations/laya
- https://laya.convaiinnovations.com/
- https://pypi.org/project/laya/

### laya-browser
- https://huggingface.co/cklxx/laya-browser (README / model card)

### jev-ultrafast
- https://github.com/browser-use/jev-ultrafast (README, LICENSE, docs/performance.md, .env.example, jev_ultrafast/model.py)
- https://github.com/browser-use/browser-harness (referenced dependency)

### Secondary (used cautiously)
- https://jevaiguide.com/jev-api/ , https://jevaiguide.com/jev-pricing/
- https://jevtypesafeai.com/pricing (reseller-style metering; not treated as TypeSafe official $0.042)
- https://mrjev.com/projects/browser-use-jev-ultrafast/ (community review; CDP tip)
- DeepWiki pages for jev-ultrafast (secondary navigation only)

---

## Explicitly NOT verified from public sources

- Live authenticated TypeSafe model catalog / account quotas / enterprise contracts.
- Internal TypeSafe training data, RLCD implementation details, or cluster hardware SKUs.
- Whether stock jev-ultrafast accepts `TYPESAFE_BASE_URL` **without** the laya-browser patch (code fetch shows hardcoded URL).
- ONNX / WebGPU / in-browser execution for Jev, Open-Jev, Laya, laya-browser, or jev-ultrafast **as shipped**.
- Independent reproduction of Laya vs Jev accuracy tables (Laya authors mark Jev columns as third-party).
- Independent reproduction of laya-browser’s 16-task suite percentages.
- Open-Jev 27B final quality and TREC results marked pending upstream.
- Exact `BU_CDP_URL` / harness telemetry defaults beyond secondary blogs.
- Any private or paid documentation behind TypeSafe login walls.

---

*End of ego-decision-001 baseline. Charter: DecisionStudy System-1 / browser decision models only.*
