# Session-by-Session Execution Plan — v2

**Version:** 2, dated 2026-10-05. It replaces `SESSION_PLAN.md` (v1, 27 sessions), which stays unchanged for reference.
**Inputs:** the v1 session plan, `AGENT_HANDOFF_2026-10-05.md`, decisions D-001 to D-029, and the planning answers in §3.
**Sessions:** 31 in total: 14 in Phase 0 (checking that the approach works before building) and 17 in production.

The v1 rule still holds. Model, framework, algorithm, latency, privacy and hardware feasibility must be proven before the production Android app is built. The starting point has changed. The hardest Phase 0 question in v1 was how to turn a statement into transactions. That is now answered on the Mac, with a different method from the one v1 assumed. Every other Phase 0 question is still open. One new question is added: whether the chosen method still works on an Android phone.

---

## 1. What changed from v1

1. **The extraction method is decided (D-028), and it is not the one v1 assumed.** v1 planned ML Kit OCR followed by Gemma 4 E4B structured extraction on LiteRT-LM. The benchmark chose a different method. Each page image goes to a general vision-language model (Qwen3.5-4B) in one request. The runtime enforces the JSON schema, and a rules step fixes the year. This scored 97.2% strict accuracy on 66 synthetic statements. Gemma 4 E4B scored 81.6% with the same method. ML Kit OCR, column reordering and the vision table-analysis fallback are dropped.
2. **Accuracy was measured with llama.cpp and GGUF weights on a Mac, but the target is LiteRT-LM on Android.** Moving to the phone runtime is now the largest open risk. Three things are unproven:
   - whether Qwen3.5-4B and its vision encoder run on LiteRT-LM at all;
   - whether LiteRT-LM can enforce the JSON schema;
   - whether LiteRT's 4-bit weights match the accuracy of the `Q4_K_M` reference.

   New session S7 and new gate G1.4 cover this. llama.cpp built for Android is the fallback, because its accuracy is already measured (§5).
3. **The latency gates are set again after measurement.** v1 required under 60 s for a 12-page statement and under 5 s per page. Those targets assumed OCR plus text extraction. On a phone, a 4B vision model reads thousands of image tokens and writes thousands of JSON tokens per page, so those targets will not hold. S8 measures the real cost and the architecture decision (S14) sets new targets. Ingestion becomes a background job that can resume after interruption.
4. **Each capability picks its own model, based on measurement.** There is no vendor constraint. For merchants, insights, reasoning and routing, the choice is between two setups:
   - the extraction model in text mode, so only one model stays loaded;
   - Gemma 4 E4B with FunctionGemma on LiteRT-LM.
5. **The open extraction gaps from the handoff become planned work:**
   - compact JSON output;
   - recovery of pages cut off at the token limit;
   - statements that run from December into January;
   - real statements;
   - statement metadata and balance validation. The frozen `json-period` prompt returns transactions and the statement period; S3 checks whether it also returns opening and closing balances.
6. **Production adds a Kotlin port of the extraction core (S17).** It is checked against the Python pipeline by replaying recorded model outputs through both. This replaces v1's OCR and column-reordering sessions.
7. **The handoff's repository conventions replace v1's `feasibility/` and `tools/` layout** (§4).
8. **Scope:** Android only, and the plan ends with the demo and acceptance, as in v1. iOS comes later (§12).

---

## 2. Starting point (as of 2026-10-05)

| Area | Status | Evidence |
|---|---|---|
| Choice of extraction method | **Done** on the Mac | D-028; `docs/ON_DEVICE_EXTRACTION_REPORT_V2.md` |
| Synthetic extraction corpus and ground truth | **Done, frozen** | 66 statements, 48 layouts, 2,001 transactions; `data/gt/`; dev split of 8 templates |
| Strict scorer; 95% intervals that resample whole layouts | **Done** | D-019 to D-024; `docs/EVAL_DESIGN_REVIEW.md` |
| `statement_extractor` Python pipeline | **Done** (817 tests pass) | D-029; `make test` |
| Year step (year on the row, else the statement-period year, else the model's year) | **Done** | `docs/YEAR_HANDLING_OPTIONS.md` |
| Model comparison: Qwen3.5 0.8B/2B/4B, Gemma 4 E2B/E4B, OCR specialists | **Done** | Report v2 |
| Real statements | **Not started**: `data/real_val/` is empty; collection protocol and notebook are ready | `docs/real_val_protocol.md`, `notebooks/try_real_statements.ipynb` |
| Compact JSON output | **Not done**: the 8,192-token limit is a workaround | Handoff §8 |
| Recovery of cut-off pages | **Not done**: such pages are flagged in `result.warnings`, not retried | Handoff §8 |
| December-to-January statements | **Not done** | Handoff §8 |
| Statement metadata and balance validation | **Not done** | — |
| Runtime parity (MLX, LiteRT-LM) | **Not re-run** | Handoff §8 |
| Android | **Not started**: the bench app has never compiled; no SDK, NDK or device | Handoff §8 |
| iOS | Bench app runs on an iPhone 16 (Qwen3.5-0.8B on llama.cpp; Gemma 4 E2B on LiteRT-LM at 18 s/page and 1.9 GB peak). Reference only. | Handoff §8 |
| D-029 refactor | **Not committed** (about 300 paths) | Handoff §9 |
| Merchants, cross-account, insights, routing, reasoning, voice, privacy | **Not started** | — |

---

## 3. Planning decisions (recorded as D-030 in Session 1)

| # | Decision | Consequence |
|---|---|---|
| P1 | Android is the only target in this plan, and LiteRT-LM is the primary runtime. iOS follows later with similar models and pipeline. | Production is Kotlin with Jetpack Compose. A second runtime (llama.cpp through JNI) is allowed only if the architecture decision shows LiteRT-LM cannot meet the extraction gates. |
| P2 | There is no constraint on model vendor. | Each capability uses the best model that runs on the target device, by measurement. |
| P3 | Ingestion latency is measured first and the targets are set afterwards, in the architecture decision. | v1's G0.4 and G1.3, and the ingestion part of G6.2, become values set by S14. |
| P4 | The plan still ends with the demo and acceptance. | v1 Sessions 26–27 become S30–S31, adapted to ingestion that takes minutes rather than seconds. |
| P5 | The D-028 extraction method and the frozen items (§4, rule 4) are the baseline. | Changes happen only through named variants and new D-entries. |

---

## 4. How to use this plan

1. Run Sessions 1–14 before any production session. S1–S5 need only the Mac. Start them while Android devices and real statements are being obtained.
2. Paste each session's **Prompt** into a fresh coding session at the repository root. The prompts expect these files to exist:
   - `docs/EXECUTION_PLAN.md`
   - `docs/SESSION_PLAN.md` (v1)
   - `docs/SESSION_PLAN_V2.md` (this file)
   - `docs/AGENT_HANDOFF_2026-10-05.md`
   - `docs/STATE.md`
   - `docs/DECISIONS.md`

   Session 1 puts them in place.
3. **Every session follows the handoff's repository conventions:**
   - Use `uv` for all Python package management. `make test` and `make lint` must pass before a session ends.
   - `docs/STATE.md` is append-only. Add one dated entry at the end of every session.
   - `docs/DECISIONS.md` gets a numbered entry for any change to schema, metrics, prompts, protocol or architecture. The next number is **D-030**.
   - Reports are new markdown files in `docs/`. A correction goes in a new document; existing reports are not edited.
   - Superseded outputs move into `_superseded_<date>/` folders. They are never deleted.
   - Runs can be resumed and only add data. Each run id names the model, quantization, runtime, device, input path, decoding and prompt variant.
   - Never reset, stash or clean the working tree.
4. **Frozen items:**
   - prompt sha256 `44511ccf…a78968`;
   - schema sha256 `c28864f9…8ad8`;
   - the `ExtractorConfig` defaults;
   - llama.cpp commit `f3f1a8f`;
   - `data/gt/`;
   - the 8-template dev split.

   Do not edit any of them. New wording becomes a named variant. New ground truth goes in a new versioned directory. A default changes only through a D-entry. The D-entry must be backed by a full run that was designed on the dev split and also reported on the other 40 templates.
5. **Where Phase 0 work lives:**
   - Python code in `research/` (scripts, configs, tests);
   - raw outputs in `results/`;
   - reports in `docs/`;
   - the disposable Android test harness in `apps/android-bench/`.

   Phase 0 must not create production feature modules, navigation, Compose screens, Room integration or reusable UI. S15 creates the production app at `apps/android/`.
6. **Use one scorer.** On-device runs and other-runtime runs save their raw model outputs. They are scored by replaying those outputs through the Python `statement_extractor`. Do not write a second scorer during Phase 0.
7. **Gate numbers come from the artifact that ships.** An accuracy number counts for a gate only if it comes from the same weights file, quantization, runtime and backend family that will ship. Mac GGUF runs can be used to screen candidates and design prompts. They never count as evidence for an on-device claim.
8. For every measurement, record the device, OS, runtime and model versions, model hashes, quantization, backend, run count, warm-up policy and the raw numbers. Report every run, not only the best ones.
9. Freeze test and holdout corpora before tuning. Never tune prompts, rules or weights on a holdout.
10. Session 14 ends with one outcome: **PROCEED**, **PROCEED WITH CHANGES** or **STOP/PIVOT**. Session 15 may start only after PROCEED or PROCEED WITH CHANGES, and only once every required change is in the target architecture.
11. Phase 0 proves only minimal working slices. A production feature counts as working only after it exists and its acceptance tests have been rerun on it.

---

## 5. Extraction architecture options (decided in Session 14)

| Option | Extraction | Downstream text tasks | Runtimes in the app | Evidence today | Main risk |
|---|---|---|---|---|---|
| **A** (preferred) | Qwen3.5-4B on LiteRT-LM | The same model in text mode, or Gemma 4 E4B | LiteRT-LM only | 97.2%, measured only on llama.cpp with `Q4_K_M` weights | Qwen3.5's vision encoder or other layers may not convert to LiteRT or run on LiteRT-LM. Schema enforcement may be missing. LiteRT's 4-bit scheme differs from `Q4_K_M`. |
| **B** | Gemma 4 E4B on LiteRT-LM | Gemma 4 E4B, with FunctionGemma for routing | LiteRT-LM only | 81.6% (Mac, 4-bit) | 15.6 points below Qwen. It needs an improvement, such as a Gemma-specific prompt or fine-tuning, proven on real statements. |
| **C** | Qwen3.5-4B on llama.cpp built for Android (JNI) | Gemma 4 E4B and FunctionGemma on LiteRT-LM, or Qwen on llama.cpp | Two runtimes, or llama.cpp alone | The same GGUF file that scored 97.2% | Android GPU-backend speed, native library size, and the memory and upkeep cost of two runtimes |
| **D** | Qwen3.5-2B on runtime A or C | As above | As above | 89.2%, below the 90% G1.1 threshold | Accuracy. Use only if A and C fail on memory or latency. |

**Order of investigation:** S7 tests option A first. If A is blocked, S7 runs options C and B on the same subset of statements, and S14 chooses. This plan recommends C over B unless B reaches G1.1 and G1.4 with evidence.

---

## 6. Phases and critical path

- **Phase 0A — Mac: consolidation and extraction gaps (Sessions 1–5).** No device needed.
- **Phase 0B — Android runtime feasibility (Sessions 6–8).** Needs a Samsung S25 Ultra.
- **Phase 0C — Feasibility of the features after extraction (Sessions 9–13):** merchants, cross-account links, insights, routing, reasoning, voice and privacy.
- **Exit gate (Session 14):** architecture decision.
- **Phase 1 — Production foundation and ingestion (Sessions 15–19).**
- **Phase 2 — Merchant resolution (Sessions 20–21).**
- **Phase 3 — Cross-account intelligence (Sessions 22–23).**
- **Phase 4 — Proactive insights (Sessions 24–25).**
- **Phase 5 — Conversational Q&A and voice (Sessions 26–28).**
- **Production privacy and security, across all phases (Session 29).**
- **Phase 6 — Demo and acceptance (Sessions 30–31).**

**Critical path:** 1 → 2 → 6 → 7 → 8 → 14 → 15 → 16 → 18 → 19 → 20.
- Real statements (S4) join the critical path once the data arrives. S14 cannot conclude PROCEED without them.
- S3, S4 and S5 run on the Mac alongside the device sessions.
- S9–S12 can start once S6 and S7 have shown which models run on the device.
- S17 (Kotlin port) can run alongside S16.
- After S20, merchant UI, cross-account work and insights can overlap.
- Q&A can start after S18.
- Everything converges at S30 → S31.
- S29 can run any time after S16 and must finish before S30.

---

## 7. Gate contract v2

Session 1 turns this table into the machine-readable evaluation contract, which is the source of truth. Where v1 had an exploratory threshold and a stricter later one, the stricter one is used for approval. Thresholds marked *proposed* are fixed in S1, before any run uses them.

**Metric note (recorded as D-031):** the primary extraction metric is the existing strict per-transaction accuracy. A transaction counts only if its date, amount, direction and payee line are all correct, which is stricter than v1's detection F1. Transaction-detection precision, recall and F1 are also reported, so v1's thresholds can still be checked. Confidence intervals resample whole layouts (synthetic data) or whole statements and institutions (real data).

| Gate | v2 criterion | Change from v1 | Status on 2026-10-05 |
|---|---|---|---|
| G0.1 | The selected text and reasoning model decodes at **≥15 tokens/s** on the S25 Ultra. Gemma 4 E4B with MTP is measured either way. The extraction model's prefill and decode speeds feed the latency model and have no fixed threshold. | Generalized from Gemma 4 E4B only | Not measured on Android |
| G0.2 | Strict accuracy **≥85%** on the 100-page real transcribed corpus. Exploratory only; approval needs G1.1 and G1.4. | Metric is now strict accuracy. Method is now the D-028 vision model; OCR is gone. | Synthetic only: 97.2% (Mac reference). Real: not run. |
| G0.3 | Merchant category accuracy **≥80%** on the 200-descriptor exploratory set. 80% to under 85% triggers contingency work. | None | Not started |
| G0.4 | **Set again in S14** from S8's measurements. v1's under 60 s per 12 pages is not assumed. | Targets set after measurement | Not measured |
| G0.5 | Peak memory (RSS) of the models that must be loaded together is **under 8 GB** on a 12 GB device. This includes the vision encoder and projector and the KV cache at the configured context. Otherwise, a load-on-demand design with measured swap time. | Covers the extraction model | Not measured on Android |
| G1.1 | Strict accuracy **≥90%** on 10 real statements from at least 5 institutions, **measured on the artifact that will ship**. The Mac reference is reported alongside. | Metric; must be measured on device | Not run |
| G1.2 | Opening balance plus transactions equals closing balance within **$1** on **≥80%** of test statements | Depends on S3's metadata extraction | Not built |
| G1.3 | **Set again in S14:** p95 time per page, total for a 12-page statement, time until the first transaction appears, battery used per statement, and no thermal shutdown | Targets set after measurement; battery and thermal added | Not measured |
| **G1.4** (new) | **Runtime parity:** the shipping artifact's strict accuracy on the 66-statement synthetic corpus is within **2.0 points** (*proposed*) of the Mac reference (97.2%), using paired, layout-resampled intervals, and is **≥90%** absolute | New | Not run |
| **G1.5** (new) | **No rows lost silently:** every page that hits the token limit is detected, and S2's recovery policy recovers it. Zero unrecovered cut-off pages on the synthetic and stress corpora. On real data, any unrecovered page is shown to the user. | New | Partly met: cut-off pages are flagged but not retried. 3 pages were cut off at the 4,096-token limit. |
| **G1.6** (new) | **December-to-January statements:** zero wrong-year rows caused by the year step on the rollover corpus, and no regression on the main corpus | New | Not handled |
| G2.1 | The selected text model's merchant category accuracy is **≥85%** on the frozen 500-descriptor holdout | The model is chosen by S9 | Not started |
| G2.2 | The model beats the rules baseline correctly on **≥30%** of comparison transactions | None | Not started |
| G2.3 | Resolving 100 unique descriptors in batches takes **under 30 s**. Per-descriptor time is also reported against the G6.2 target of under 2 s. | None | Not started |
| G3.1 | Transfer-detection precision **≥90%** | None | Not started |
| G3.2 | Transfer-detection recall **≥80%** for approval (75% is the formal minimum) | None | Not started |
| G3.3 | Cross-account analysis of 3 accounts × 6 months takes **under 2 minutes** | None | Not started |
| G4.1 | At least **4 of 6** insight types find relevant, non-obvious results | None | Not started |
| G4.2 | 10 reviewers rate relevance and actionability at a mean of **≥3.5 out of 5** | None | Not started |
| G5.1 | Routing accuracy **≥85%** on 200 frozen holdout queries, scoring both function choice and arguments | The candidates now include the text model's own tool calling | Not started |
| G5.2 | The selected reasoning model is factually correct on **≥90%** of known-answer queries. Citation accuracy is scored separately. | No longer tied to Gemma 4 E4B | Not started |
| G5.3 | Voice lookup round trip **≤3 s**. Approval uses the stricter target: under 2 s from the end of speech to the start of spoken output. | None | Not started |
| G5.4 | Word error rate on financial terms **≤15%** | None | Not started |
| G6.2 (pre-production subset) | Ingestion: **the S14 values**. Merchant resolution under 2 s per descriptor. Lookup under 500 ms. Reasoning under 5 s. Voice lookup under 2 s. Voice reasoning under 5 s. | Ingestion targets set after measurement | Not started |
| G6.3 (pre-production subset) | The test harnesses send **zero packets containing financial data**, account numbers or descriptors, and need no network during inference. Model files are delivered without the app holding the INTERNET permission, and the runtimes do not log prompts or outputs. | Model delivery and runtime logs added | Not started |
| G6.4 | Human evaluation averages **≥3.5 out of 5** across relevance, accuracy, surprise and actionability | None | Not started |
| G6.1 | The complete demo runs on both target devices without crashing. **Checked only in S31**, once the demo exists. | None | — |

---

## 8. Blockers and long-lead items

| Item | Needed by | Owner | Notes |
|---|---|---|---|
| A Samsung S25 Ultra (12 GB) and an Android development setup | S6 | You | The Pixel 9 Pro is a secondary target. Without it, G6.1 is checked on a single device and the ADR must say so. |
| Real statements: 100 transcribed pages (50 Canadian, 50 US) plus 10 statements from at least 5 institutions, kept separate from a real dev partition | S4, S7 | You | Collection and labeling follow `docs/real_val_protocol.md`, extended in S1. Holdout labels are entered twice by hand, without filling them in from model output first. |
| Real merchant descriptors (200 exploratory plus 500 holdout) | S9 | You | Taken from the real statements plus synthetic data. Holdouts must not overlap other partitions. |
| Recorded speech with financial terms (quiet and noisy) | S12 | You | — |
| 10 human reviewers | S11, S31 | You | Without them, G4.2 and G6.4 stay unproven, and S14 cannot conclude an unconditional PROCEED. |
| LiteRT-LM support for Qwen3.5 with vision | S7 | Resolved by S7 | The largest technical unknown in the plan |
| Model licenses for app distribution | S1 | S1 | Qwen3.5, Gemma 4, FunctionGemma and the speech-recognition candidates |
| A route for getting the models (3–4 GB) onto the phone without the app needing internet access | S13, S29 | S13 | S13 checks the current Play Asset Delivery and Play on-device AI options against current documentation. Demo builds can load models over adb. |

---

## 9. Sessions

### Session 1: Consolidation, plan decisions, and evaluation contract v2

**Phase:** 0A — Mac
**Prerequisites:** None

**Scope:**
- Commit the D-029 refactor as one commit, after `make test` and `make lint` pass.
- Put the plan documents in `docs/`.
- Record D-030 (the planning decisions) and D-031 (how v1's gates map to the strict scorer).
- Write a machine-readable evaluation contract for every gate in §7, with each gate's current evidence status.
- Write schemas, manifests and a validator for the corpora that do not exist yet. Extend the real-statement protocol.
- Review model licenses.

**Prompt:**
> Read `docs/AGENT_HANDOFF_2026-10-05.md`, `docs/STATE.md`, `docs/DECISIONS.md`, `docs/EVAL_DESIGN_REVIEW.md`, `docs/real_val_protocol.md`, `docs/dataset_card.md`, and `docs/SESSION_PLAN_V2.md` §3–§8. In `docs/EXECUTION_PLAN.md`, read the Decision Gates, Tasks 0.2–0.4, Task 6.5, the Fine-Tuning Strategy and Appendices B–C.
>
> **Step 0.** The D-029 refactor is not committed. Do not reset, stash or clean the working tree. Run `make test` (expect 817 passed) and `make lint`. If both pass, commit the working tree as one commit that describes the D-029 layout change, with nothing else in it. If either fails, stop and report. In a separate commit, add `docs/SESSION_PLAN_V2.md`, and also `docs/SESSION_PLAN.md`, `docs/EXECUTION_PLAN.md` and the handoff if they are missing.
>
> Record D-030 with planning decisions P1–P5 from §3 of the plan. Record D-031, which maps v1's extraction gates onto the existing strict scorer:
> - strict per-transaction accuracy is the primary metric;
> - transaction-detection precision, recall and F1 are also reported, so v1's thresholds stay checkable;
> - confidence intervals resample whole layouts for synthetic data, and whole statements or institutions for real data.
>
> Finalize the proposed thresholds for G1.4–G1.6 in D-031 before any run uses them.
>
> Write `research/configs/evaluation_contract.yaml` and `docs/EVALUATION_CONTRACT.md`. Cover every gate in §7 with:
> - the metric formula;
> - matching rules (reuse the existing row matcher);
> - the confidence-interval method and minimum sample size;
> - warm-up and repetition rules for timing;
> - the hardware metadata to record;
> - the fail action;
> - the current evidence status, linked to the evidence, or "none".
>
> Record the latency gates (G0.4, G1.3 and the ingestion part of G6.2) as "set in S14", with no threshold.
>
> Define schemas and manifests for the corpora that do not exist yet:
> - 100 real transcribed pages (50 Canadian, 50 US);
> - 10 real statements from at least 5 institutions;
> - a real dev partition, separate from both;
> - 200 exploratory merchant descriptors and a separate 500-descriptor holdout;
> - 200 routing holdout queries, plus separate training and few-shot data;
> - known-answer reasoning and citation cases;
> - cross-account fixtures that include hard negatives;
> - cases for the six insight types;
> - recorded speech containing financial terms.
>
> Add `research/scripts/validate_corpora.py`. It checks schema, expected counts and SHA-256 hashes, and checks that no statement or descriptor appears in both a dev partition and a holdout. Wire it into `make test` for the corpora that exist.
>
> Extend the real-statement protocol. Following the repository's conventions, this may need to be a new protocol document. It must cover the 100-page and 10-statement sets, scans and photos, de-identification, double-entered holdout labels with no pre-filling from model output, and collecting merchant descriptors. Real statements stay out of source control.
>
> Write `docs/MODEL_LICENSES.md` for Qwen3.5-4B and 2B, Gemma 4 E4B, FunctionGemma and the speech-recognition candidates. For each, give the license, the terms for commercial use and for redistribution inside an app, and any attribution required.
>
> Report missing data as a blocker with an owner. Never replace it silently with synthetic data. Append an entry to `docs/STATE.md`.

**Key outputs:**
- The D-029 commit
- D-030 and D-031
- `research/configs/evaluation_contract.yaml`, `docs/EVALUATION_CONTRACT.md`
- `research/scripts/validate_corpora.py` and corpus manifests
- The extended real-data protocol
- `docs/MODEL_LICENSES.md`

**Done when:** D-029 is committed and `make test` passes afterwards. Every v2 gate has a contract row and an evidence status. Every missing corpus has a schema, a manifest stub and an owner. Licenses are reviewed.

---

### Session 2: Extraction output efficiency — compact JSON, cut-off recovery, token budgets

**Phase:** 0A — Mac
**Prerequisites:** Session 1

**Scope:**
- A named variant that keeps the frozen prompt and schema but enforces JSON with no whitespace, run on the full corpus.
- Detection of pages cut off at the token limit, and a policy for recovering them.
- A token count for each page and each candidate model (image tokens in, JSON tokens out), plus lower-resolution variants to show the trade-off between speed and accuracy.

**Prompt:**
> Read the handoff §2, §6 and §8; `docs/RUNNING.md`; D-028 and later in `docs/DECISIONS.md`; `statement_extractor/prompts.py`, `config.py` and `pipeline.py`; and §4 and gate G1.5 of the plan. On a phone, the time to generate output tokens will dominate ingestion. Measure and reduce output tokens without touching anything frozen.
>
> **(1) Compact JSON.** Add a named variant that keeps the `json-period` prompt text and schema but enforces JSON with no whitespace, through the grammar and `response_format` path. Confirm that the prompt and schema hashes are unchanged. Run the variant on all 66 statements with Qwen3.5-4B and the frozen decoding settings. If time allows, also run Qwen3.5-2B and Gemma 4 E4B. Report:
> - strict accuracy against the D-028 result, with a paired, layout-resampled confidence interval;
> - output tokens per page (p50, p95 and maximum);
> - how many pages go over 2,048, 4,096 and 8,192 tokens.
>
> **(2) Cut-off recovery.** Detect a cut-off page from the runtime's finish reason, not only from JSON that fails to parse. Evaluate at least three recovery methods:
> - retrying with a higher token limit that still fits the context;
> - splitting the page image into overlapping horizontal bands and removing duplicate rows at the overlaps deterministically;
> - a request that continues from where the output stopped.
>
> Build a small stress corpus of dense pages with the existing template printer, in a new versioned ground-truth directory; do not touch `data/gt/`. Test on this corpus and on the 3 pages that were cut off at 4,096 tokens. Choose a policy that loses no rows silently. Any page that cannot be recovered must appear as a page-level warning.
>
> **(3) Input cost.** Count image tokens per page at 200 DPI for each candidate model. Add named variants at lower resolution or with a pixel cap, for example 150 DPI. Report the trade-off between accuracy and tokens, first on the dev split and then on the other 40 templates. Do not change any default.
>
> Propose any default change (compact output, `max_tokens`, recovery policy) as a D-entry backed by a full run.
>
> Write `docs/EXTRACTION_TOKEN_BUDGET.md`, a table per model of image tokens, output tokens and cut-off rates; S8 turns it into latency estimates. Write `docs/CUTOFF_RECOVERY.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- Named variants in `statement_extractor/`
- A stress corpus with versioned ground truth
- Runs in `results/`
- `docs/EXTRACTION_TOKEN_BUDGET.md` and `docs/CUTOFF_RECOVERY.md`
- D-entries

**Done when:** Compact JSON's accuracy and token savings are measured with confidence intervals. A recovery policy meets G1.5 on the synthetic and stress corpora. Every candidate model has a token budget.

---

### Session 3: Extraction correctness gaps — year rollover, statement metadata, balance validation, page boundaries

**Phase:** 0A — Mac
**Prerequisites:** Session 1

**Scope:**
- A rule for statements that run from December into January, tested on a new corpus.
- A named request that extracts statement metadata (institution, masked account number, opening and closing balances), if the frozen schema lacks these fields.
- Balance validation (G1.2) as a trigger for review.
- An evidence check of each v1 edge case against the D-028 method.

**Prompt:**
> Read the handoff §2 and §8; `docs/YEAR_HANDLING_OPTIONS.md`; `statement_extractor/year_resolution.py`, `dates.py`, `parsing.py` and `types.py`; `docs/EXECUTION_PLAN.md` Tasks 1B.3 and 1C.1–1C.3; and gates G1.2 and G1.6.
>
> **(1) Statements that run from December into January.** Use the existing template printer to generate a new versioned rollover corpus. It should hold statements that run from December into January, and at least one that covers February 29. Include templates that print the year on each row and templates that do not. Extend the year step: a row without a printed year takes the year from the statement period that matches its month, and otherwise falls back as it does today. Replay all recorded outputs through the new rule and confirm that nothing changes on the existing corpus. Then run the model on the rollover corpus. G1.6 requires zero wrong-year rows caused by the year step.
>
> **(2) Statement metadata.** Check what the frozen schema already returns. If it lacks any of these fields, design a separate named `statement-metadata` prompt and schema:
> - institution, account type, masked account number and currency;
> - opening and closing balances;
> - card fields such as previous balance, new balance and payment due.
>
> Run it on the first page, and on the last page where needed. For synthetic statements, take ground truth from the generator's own values and write it to a new versioned directory; never type it by hand. Design on the dev split and report on the other 40 templates. Reduce account numbers to their last four digits before anything is stored.
>
> **(3) Balance validation (G1.2).** Implement two checks:
> - for the statement, opening balance plus credits minus debits equals the closing balance, within $1;
> - for each row, where a balance column exists, the running balance is consistent.
>
> Measure both on the synthetic corpus:
> - the false-alarm rate: how often correctly extracted statements fail the check;
> - the detection rate: the share of statements with at least one extraction error that the check flags.
>
> The check triggers review. It never corrects anything automatically.
>
> **(4) Page boundaries and v1 edge cases.** Using recorded outputs, measure how often each of these happens:
> - rows split across two pages;
> - carried-forward or opening-balance rows extracted as transactions;
> - rows duplicated at a page break;
> - multi-line descriptions, fees and foreign-currency notes;
> - cheque images and promotional pages;
> - pending versus posted rows.
>
> For each v1 edge case, record with evidence whether the D-028 method already handles it or needs a rule. Add rules to `statement_extractor` only where failures exist, with tests.
>
> Record a D-entry for each rule and schema. Write `docs/EXTRACTION_GAPS_REPORT.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- The rollover corpus and its ground truth, plus the updated year step
- The `statement-metadata` variant and its ground truth
- The balance validator
- `docs/EXTRACTION_GAPS_REPORT.md`
- D-entries

**Done when:**
- G1.6 passes on synthetic data.
- Metadata accuracy is reported field by field.
- G1.2 has a synthetic baseline with false-alarm and detection rates.
- Every v1 edge case is marked "handled", "needs a rule" or "not applicable", with evidence.

---

### Session 4: Real-statement validation of the reference pipeline

**Phase:** 0A — Mac
**Prerequisites:** Session 1, plus Session 3 for G1.2. You must have collected and labeled the real corpora first.

**Scope:**
- First accuracy numbers on real statements with the Mac reference configuration: G0.2, G1.1 and G1.2.
- Comparison runs for the fallback models.
- A list of the error types on real statements, sorted by cause.

**Prompt:**
> Read `docs/real_val_protocol.md` and its extension from S1, `docs/EVALUATION_CONTRACT.md`, the handoff, and `notebooks/try_real_statements.ipynb`. Before any run, check the real corpora's manifests and hashes with `research/scripts/validate_corpora.py`. If a holdout is incomplete, stop and report.
>
> Run the reference configuration on three sets: the 100-page set (G0.2, exploratory), the 10-statement set from at least 5 institutions (G1.1 and G1.2), and the scans and photos subset. The reference configuration is:
> - Qwen3.5-4B with `Q4_K_M` weights, on llama.cpp `f3f1a8f`;
> - the frozen defaults, plus any defaults that S2 or S3 changed through a D-entry.
>
> Also run Qwen3.5-2B and Gemma 4 E4B, as evidence for the fallbacks. Score with the strict scorer and report:
> - precision, recall and F1;
> - error rates for each field (date, year, amount, direction, payee);
> - results per institution, with confidence intervals that resample statements;
> - cut-off pages and balance-check outcomes.
>
> Sort the errors on real statements by cause and compare them with the synthetic errors. Design any fix on the real dev partition only, as a named variant, then run it once on the holdouts. Never use a holdout for design.
>
> Ten statements give wide confidence intervals. Say so plainly, and do not claim a pass when the lower bound is far below 90%. These are Mac reference numbers, not on-device evidence (plan §4, rule 7). Write `docs/REAL_STATEMENT_VALIDATION.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- Real-data runs in `results/` (gitignored)
- `docs/REAL_STATEMENT_VALIDATION.md`, including the error breakdown by cause

**Done when:** G0.2, G1.1 and G1.2 have Mac-reference results with confidence intervals, or are recorded as blocked by missing data. Real-data errors are sorted by cause. No holdout was used for design.

---

### Session 5: Synthetic scenario statements and fixtures for later features (v1 Session 2)

**Phase:** 0A — Mac
**Prerequisites:** Session 1

**Scope:**
- The 18 Appendix C PDFs, generated with the existing template printer, with exact ground truth.
- All ten seeded Appendix C scenarios.
- Fixtures for testing cross-account links, insights, routing, reasoning and privacy.
- Extracted versions of the 18 PDFs, so later evaluations can measure how extraction errors carry through.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Task 6.1 and Appendix C, the evaluation contract, `research/configs/template_print.yaml`, and the existing statement generator in `research/`. Reuse that generator; do not write a second one.
>
> Generate the 18 Appendix C PDFs: TD chequing, Scotiabank Visa and RBC Visa, January–June 2026. Match Appendix C's exact volumes, balances and descriptors, and seed all of its scenarios:
> - price changes;
> - a duplicate charge;
> - a transfer between accounts;
> - a refund-and-rebuy chain;
> - a forgotten ("zombie") subscription;
> - micro-charges;
> - telling Uber from UberEats;
> - PayPal and Patreon ambiguity;
> - point-of-sale descriptor traps.
>
> Include multi-page tables, repeated headers, promotional inserts and varied date formats, all generated from a documented seed. If these institutions need new templates, add them outside the dev split and outside the 66-statement corpus.
>
> Produce canonical ground truth for transactions, statement metadata, merchants, cross-account links, insights, query answers and redaction. Add hard negatives, so that no detector can pass by marking every candidate as positive. Check the values printed in each PDF, and its balances, against the ground truth.
>
> Then run the reference extraction pipeline on all 18 PDFs and store the extracted transactions next to the ground truth. Later sessions evaluate each feature twice:
> - on ground-truth transactions, to measure the feature itself;
> - on extracted transactions, to measure how extraction errors carry through.
>
> Report extraction accuracy on these PDFs as an extra held-out check. Freeze the release and hash it. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `research/scripts/generate_demo_data.py`, a thin wrapper around the existing generator
- `data/demo_v1/*.pdf`
- `data/demo_v1/ground_truth.json` and `data/demo_v1/extracted_reference.json`
- `data/demo_v1/manifest.json`

**Done when:**
- The 18 PDFs and all fixtures reproduce byte for byte from the seed, after normalizing any timestamps the PDF library embeds.
- All ten Appendix C scenarios are checked by tests.
- Balance and schema checks pass.
- Accuracy of the extracted transactions against ground truth is recorded.
- The release is hashed.

---

### Session 6: Android toolchain and LiteRT-LM runtime benchmark (v1 Session 3)

**Phase:** 0B — Android device
**Prerequisites:** Session 1; a Samsung S25 Ultra (and a Pixel 9 Pro where available)

**Scope:**
- Install a pinned Android SDK and NDK, and get `apps/android-bench` to compile for the first time.
- Check which LiteRT-LM features actually exist: image input, constrained decoding and backend choice.
- Benchmark Gemma 4 E4B and FunctionGemma: load time, prefill and decode speed, MTP, heat, memory, running both models at once, and loading them on demand.
- Check whether Qwen3.5 is available for LiteRT-LM, for use in S7.

**Prompt:**
> Read the handoff §8 ("Phones"), `apps/android-bench/`, `apps/ios-bench/` (for the measurement design already used on iOS), `docs/ENV.md`, `docs/EXECUTION_PLAN.md` Tasks 0.1 and 0.6, and gates G0.1 and G0.5.
>
> The Android bench app has never compiled. Install and pin the SDK, NDK, build tools, Gradle and the LiteRT-LM Android release, and record every version in `docs/ENV.md`. Make `apps/android-bench` build and run on the S25 Ultra. Keep it a disposable harness: it needs only the controls to run and export benchmarks.
>
> Check which LiteRT-LM APIs actually exist; do not trust the pseudocode in the execution plan. Check image input, streaming, sampling controls, constrained decoding or structured output, and backend selection (CPU, GPU, and NPU where offered).
>
> Benchmark Gemma 4 E4B (INT4, GPU):
> - cold and warm load time;
> - prefill at 512, 1,024, 2,048, 4,096 and 8,192 tokens;
> - decode tokens per second with MTP on and off;
> - peak RSS;
> - CPU and GPU thermal status every 10 s during five minutes of continuous inference.
>
> Benchmark FunctionGemma 270M (dynamic INT8, CPU): load time and p50/p95 classification latency.
>
> Load both models together, and measure total RSS and how much FunctionGemma slows down. Then implement a minimal load-on-demand manager that unloads one model before loading the other, and measure swap time and failure behavior.
>
> Also check whether a LiteRT-LM artifact for Qwen3.5-4B or 2B exists, or whether the current converter supports these models. Record what you find for S7, but do not convert yet.
>
> Export raw and summary JSON to `results/android_bench/`, including every run. Write `docs/ANDROID_RUNTIME_BENCHMARK.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `apps/android-bench` building with pinned versions
- An updated `docs/ENV.md`
- `results/android_bench/*.json`
- `docs/ANDROID_RUNTIME_BENCHMARK.md`

**Done when:** The bench builds from a clean checkout. G0.1 is marked pass or fail for Gemma 4 E4B. G0.5 passes, or load-on-demand is measured. LiteRT-LM's support for image input, constrained decoding and each backend is documented with evidence.

---

### Session 7: Extraction model on Android — artifact, constrained decoding, accuracy parity

**Phase:** 0B — Android device
**Prerequisites:** Sessions 2 and 6

**Scope:**
- Option A: get or convert Qwen3.5-4B and 2B for LiteRT-LM, including the vision encoder and projector, and find a way to enforce the JSON schema.
- Accuracy parity against the D-028 reference (G1.4), and the effect of rendering pages on the device instead of the Mac.
- Options C and B on the same subset if option A is blocked.
- A table comparing the candidates, for the architecture decision.

**Prompt:**
> Read D-028, the handoff §2 and §6, `docs/ANDROID_RUNTIME_BENCHMARK.md`, `docs/EXTRACTION_TOKEN_BUDGET.md`, and the plan's §5 options and gate G1.4. Establish which extraction options run on the S25 Ultra, and how accurate each one is, in the order below.
>
> **Option A: Qwen3.5-4B on LiteRT-LM.**
> - Get or convert a `.litertlm` artifact for Qwen3.5-4B, and for 2B, including the vision encoder and projector. Use the official LiteRT conversion tooling.
> - Check that the converter and runtime support every architectural feature the model uses. If any layer type is not supported, stop work on option A and record exactly what failed and with which versions.
> - Record the quantization scheme; it will differ from `Q4_K_M`.
> - Work out how the frozen JSON schema can be enforced: native constrained decoding, an equivalent grammar, or not at all. If nothing can enforce it, measure free decoding plus the existing `json_repair`, and compare with the free-decoding `*_screen` tables.
>
> **Accuracy parity.**
> 1. Run the full 66-statement corpus on the artifact that will ship. Use desktop LiteRT-LM if it supports this artifact and image input; otherwise use the device.
> 2. Run a subset of at least 16 statements on the device's GPU backend, including all 8 dev templates, to confirm the backend gives the same accuracy.
> 3. Feed page images rendered by the Mac pipeline, so rendering stays constant. Save the raw outputs and score them by replaying them through the Python pipeline.
> 4. Report strict accuracy against the D-028 reference with paired, layout-resampled confidence intervals (G1.4).
> 5. Repeat the subset with images that `PdfRenderer` renders on the device at 200 DPI, to measure any difference caused by rendering.
>
> **If option A fails or misses G1.4,** run the same subset for two more options:
> - Option C: the Qwen3.5-4B GGUF on llama.cpp built for Android, on the CPU and on the available GPU backend. Record the llama.cpp commit used, and whether `f3f1a8f` builds for Android.
> - Option B: Gemma 4 E4B on LiteRT-LM, with the D-028 method.
>
> Improving option B, through a Gemma-specific prompt variant or fine-tuning, is out of scope for this session. Record it as a contingency for the architecture decision.
>
> **Real statements.** When S4's real corpora are available, run the leading artifact once on the 10-statement holdout and on the 100-page set. If the data arrives after this session ends, do this as a short follow-up run under the same protocol before S14.
>
> Write `docs/ANDROID_EXTRACTION_PARITY.md` with a table of candidates. For each option, give the artifact hash, quantization, runtime and backend, schema enforcement, accuracy with confidence interval, rendering parity, peak RSS and blockers. Append an entry to `docs/STATE.md`, and record a D-entry for any new run protocol.

**Key outputs:**
- Conversion scripts, if used (`research/scripts/convert_litert.py`)
- Hashed artifacts (gitignored)
- Notes on building llama.cpp for Android, if option C is tested
- Results and `docs/ANDROID_EXTRACTION_PARITY.md`

**Done when:** Option A is proven or ruled out, with evidence. At least one option has on-device accuracy evidence on the synthetic corpus. G1.4 is marked pass or fail for every option tested.

---

### Session 8: On-device ingestion timing, memory, heat and battery — setting the latency targets

**Phase:** 0B — Android device
**Prerequisites:** Sessions 3 and 7

**Scope:**
- A minimal on-device ingestion pipeline for each viable candidate, timed stage by stage.
- Throttling under sustained heat, battery use, and peak memory.
- Proof that ingestion runs in the background and resumes after the app's process is killed.
- A latency model, a ranked list of speed-ups, and proposed targets for G0.4, G1.3 and the ingestion part of G6.2.

**Prompt:**
> Read `docs/ANDROID_EXTRACTION_PARITY.md`, `docs/EXTRACTION_TOKEN_BUDGET.md`, `docs/EXTRACTION_GAPS_REPORT.md`, `docs/EXECUTION_PLAN.md` Task 0.5, and gates G0.4, G0.5, G1.3 and G6.2. Do not assume v1's targets of under 60 s per 12-page statement and under 5 s per page. This session produces the evidence for setting new ones.
>
> In `apps/android-bench`, build a minimal ingestion pipeline for each candidate that passed S7:
> - `PdfRenderer`, one page at a time, at 200 DPI and at any lower-resolution variant from S2;
> - S3's metadata request;
> - the per-page extraction request, with the chosen output format and cut-off policy;
> - raw outputs saved for replay scoring.
>
> Measure:
> - time per page for each stage (rendering, image preprocessing, vision encoding, prefill and decode), plus the total, at p50 and p95;
> - the time until the first transaction appears on screen;
> - end-to-end time for the 12-page statement;
> - throttling from heat, by running three 12-page statements back to back;
> - battery drop;
> - peak RSS, including the KV cache at the configured context. If the token budget allows, also test a smaller context.
>
> Check that ingestion works in the background. A long-running WorkManager worker, or a foreground service with a notification, must finish a 12-page import with the screen off and with the app in the background. If the process is killed partway through a statement, the import must resume from the last finished page. Record which Android 14 and 15 foreground-service type applies, and its limits.
>
> Fit a latency model: time per page ≈ fixed cost + image tokens × prefill cost per token + output tokens × decode cost per token. Check its predictions against the measurements. Rank the available speed-ups by time saved and accuracy lost, using the measurements from S2 and S7:
> - compact output;
> - lower resolution;
> - the 2B model;
> - the NPU backend;
> - a smaller context.
>
> Propose new values for G0.4, G1.3 and the ingestion part of G6.2, plus targets for time to first transaction and for battery use. The architecture decision will adopt or change them. Repeat everything on the Pixel 9 Pro if one is available. Write `docs/ANDROID_INGESTION_TIMING.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- The ingestion test code in `apps/android-bench`
- `results/android_bench/ingestion_*.json`
- `docs/ANDROID_INGESTION_TIMING.md`

**Done when:** Every candidate has measured per-page and per-statement times, heat data and battery data. Background ingestion is shown to resume after the process is killed. The proposed targets are backed by measurements.

---

### Session 9: Choosing the text model; merchant and cross-account evaluation (v1 Session 5)

**Phase:** 0C — Features after extraction
**Prerequisites:** Sessions 5, 6 and 7

**Scope:**
- Choose the model for the text tasks: the extraction model in text mode (only one model loaded) or Gemma 4 E4B, plus how models are loaded and swapped.
- Compare a rules baseline with the model on the exploratory and holdout descriptor sets.
- Build detectors for transfers, duplicates, refunds and subscription migrations, tested on S5's fixtures.
- Run every evaluation on both ground-truth and extracted transactions.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 0.3, 2.1–2.3, 3.1–3.4 and 6.5, the Fine-Tuning Strategy, gates G0.3, G2.1–G2.3 and G3.1–G3.3, `docs/ANDROID_RUNTIME_BENCHMARK.md`, and `docs/ANDROID_EXTRACTION_PARITY.md`. Build disposable evaluators in `research/`. Do not build Room storage or any UI.
>
> The text-model candidates are the models that S6 and S7 showed run on the device:
> - the selected extraction model in text-only mode, which keeps a single model loaded;
> - Gemma 4 E4B on LiteRT-LM;
> - any other model that runs.
>
> Mac GGUF runs may be used to screen candidates. Gate numbers must come from the artifact that ships (plan §4, rule 7).
>
> **Merchant resolution.** Build a rules baseline from regular expressions and keywords. Build a model-based resolver that handles point-of-sale prefixes, payment intermediaries, marketplace sub-brands, amounts, dates and institution context, one descriptor at a time and in batches of ten.
>
> Evaluate on the 200 exploratory descriptors. Freeze the prompts, then run the separate 500-descriptor holdout once. Report:
> - accuracy of merchant name, category, subscription flag and point-of-sale handling;
> - calibration, and where the model beats the rules;
> - p50 and p95 latency on the device;
> - total time for 100 unique descriptors.
>
> 80% triggers contingency work; approval needs 85% category accuracy on the holdout.
>
> **Cross-account detection.** Using S5's fixtures, build deterministic candidate generation followed by model confirmation for transfers, duplicates, refunds and subscription migrations. Include hard negatives, and score each link type separately.
>
> Run every evaluation twice: on ground-truth transactions and on S5's extracted transactions, so the effect of extraction errors is visible. Report transfer precision and recall, and the runtime for 3 accounts × 6 months.
>
> Recommend one text model, and a loading policy for it next to the extraction model: both loaded together, or loaded on demand with a measured swap time. Keep all raw predictions. Write `docs/MERCHANT_CROSSACCOUNT_FEASIBILITY.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `research/merchant_eval/` and `research/cross_account_eval/`
- `results/merchant_metrics.json` and `results/cross_account_metrics.json`
- `docs/MERCHANT_CROSSACCOUNT_FEASIBILITY.md`

**Done when:** G0.3, G2.1–G2.3 and G3.1–G3.3, and the merchant latency target in G6.2, have reproducible evidence. A text model and its loading policy are recommended with measurements. Each failure is traced to the prompt, the rules, the model or candidate generation.

---

### Session 10: Routing and reasoning feasibility (v1 Session 6)

**Phase:** 0C — Features after extraction
**Prerequisites:** Sessions 5 and 6 (Session 9 is recommended, for the choice of text model)

**Scope:**
- Compare FunctionGemma 270M (base, few-shot and, if needed, fine-tuned) with the text model's own tool calling.
- Fine-tune, convert and verify on the device, when needed.
- Evaluate reasoning: factual accuracy and citation accuracy, using known structured records.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Task 0.4, Tasks 5A.1–5A.4, the Fine-Tuning Strategy, Appendix B, gates G5.1–G5.2, and `docs/ANDROID_RUNTIME_BENCHMARK.md`. Use disposable tools only: no database access objects, conversation storage or chat UI.
>
> Implement the full 11-function schema, and an evaluator that scores:
> - the query's route class;
> - function selection;
> - argument extraction;
> - outputs with several calls;
> - invalid JSON;
> - latency.
>
> Compare FunctionGemma 270M on LiteRT-LM (base, few-shot and, when needed, fine-tuned) with the S9 text model's own tool calling. Fine-tuning FunctionGemma depends on its base accuracy:
> - below 70%: fine-tuning is required;
> - 70% to under 85%: evaluate fine-tuning as the preferred contingency;
> - 85% or more: document whether the base or few-shot model is enough.
>
> If fine-tuning is needed:
> 1. Generate at least 500 training pairs with the 40/20/25/15 distribution.
> 2. Train with the plan's LoRA configuration.
> 3. Convert the result to a dynamic-INT8 `.litertlm` file.
> 4. Verify the converted model on the device.
>
> The scripts must actually run, not just exist. Keep the frozen 200-query holdout separate from all training and few-shot data. Score every candidate on exactly that holdout; 85% or more is required.
>
> Build a reasoning evaluator for the selected text model. It gives the model only fixture records from S5, and checks every amount and date the model cites against those records. Score:
> - factual correctness;
> - citation precision and recall;
> - unsupported claims;
> - refusal when the data is insufficient;
> - on-device latency against the 5 s target.
>
> Write `docs/ROUTING_REASONING_FEASIBILITY.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `research/scripts/generate_training_data.py`, `finetune_functiongemma.py` and `convert_model.py`
- `research/query_reasoning_eval/`
- `results/routing_reasoning_metrics.json`
- `docs/ROUTING_REASONING_FEASIBILITY.md`

**Done when:** G5.1 and G5.2 have evidence from the holdout. The selected router and its fallback are explicit. Any training and conversion path has actually been run.

---

### Session 11: Insight logic and human-evaluation feasibility (v1 Session 7)

**Phase:** 0C — Features after extraction
**Prerequisites:** Sessions 5 and 9

**Scope:**
- Plain-code versions of all six insight generators, tested on fixtures.
- Narration by the selected text model, using only the totals it is given.
- A blinded human-evaluation protocol, and its results.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 4.1–4.6, the insight evaluation in Task 6.5, Appendix A5, and gates G4.1, G4.2 and G6.4. Build disposable evaluators in SQL, Python or Kotlin, and minimal adapters for the selected text model. Do not build production InsightEngine storage or Compose cards.
>
> Evaluate the six insight types on S5's canonical fixtures, using both ground-truth and extracted transactions:
> - subscription audit;
> - spending anomaly;
> - fee and interest tracker;
> - micro-charge totals;
> - largest transactions;
> - income regularity.
>
> Check all arithmetic in plain code before the model narrates anything. A narrative may cite only the amounts and dates it was given, and any recommendation it cannot support is flagged.
>
> Define a blinded protocol for 10 reviewers who rate relevance, accuracy, surprise and actionability from 1 to 5. It needs randomized presentation, anchors for each rating, a way to handle disagreements, and anonymized stored ratings. Run the protocol if reviewers are available. If they are not, mark the human gates unproven, which blocks an unconditional PROCEED. Report coverage and ratings for each insight type. Write `docs/INSIGHT_FEASIBILITY.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `research/insight_eval/`
- `docs/INSIGHT_HUMAN_EVAL_PROTOCOL.md`
- `results/insight_metrics.json`
- `docs/INSIGHT_FEASIBILITY.md`

**Done when:** G4.1, G4.2 and G6.4 are marked pass or fail with the raw reviewer evidence, or are explicitly unresolved because no reviewers were available. Every number in the narratives matches the fixtures.

---

### Session 12: Offline speech recognition, word error rate and voice latency (v1 Session 8)

**Phase:** 0C — Features after extraction
**Prerequisites:** Sessions 1, 6 and 10

**Scope:**
- Check Moonshine's model files, license, Android runtime, streaming support and offline use. Also check any audio-capable model that runs on LiteRT-LM.
- Measure word error rate on financial terms, and the latency of the whole voice pipeline.
- Measure memory under the loading policy chosen in S9.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 5B.1–5B.2, the voice latency budget, risks R5 and R8, and gates G5.3–G5.4 and the voice targets in G6.2. Build a disposable push-to-talk test in `apps/android-bench`, plus an evaluator that replays audio files. Do not build production chat or waveform UI.
>
> Check the actual Moonshine v2 Base files: the license, model format, Android-compatible runtime, CPU architecture support, streaming behavior, and whether it starts without a network connection. Do not assume a `.litertlm` version exists. Test ONNX Runtime Mobile or whichever runtime the documentation names. If a model that LiteRT-LM supports accepts audio input, include it as a speech-recognition candidate.
>
> Replay the frozen audio corpus, with both noisy and quiet samples. Compute the standard word error rate, and the rate on financial terms alone: RRSP, chequing, amortization, merchant names, dates and amounts.
>
> Measure each stage separately and the whole pipeline end to end:
> - detecting the end of speech;
> - the final transcription;
> - routing;
> - the fixture lookup;
> - reasoning by the text model;
> - the start of the first spoken word.
>
> Measure memory with the S9 loading policy in effect. G5.3 requires 3 s or less; use under 2 s for a lookup as the stricter approval target. If Moonshine fails on availability, error rate, latency, size or privacy, benchmark at least one offline alternative and document the architecture change. Write `docs/VOICE_RUNTIME_DECISION.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- Voice test code in `apps/android-bench`
- `results/voice_metrics.json`
- `docs/VOICE_RUNTIME_DECISION.md`

**Done when:** G5.3, G5.4 and the voice targets in G6.2 have measured evidence. The runtime and model files are proven available. The fallback and its privacy trade-offs are explicit.

---

### Session 13: Offline operation, privacy and model isolation (v1 Session 9)

**Phase:** 0C — Features after extraction
**Prerequisites:** Sessions 5–12

**Scope:**
- Run all test harnesses offline and inspect their network traffic.
- Check temporary page images, stored raw model outputs, runtime logs, and that each inference session is isolated from the others.
- Find how the models reach the phone without the app needing internet access.
- Define the privacy controls the production app needs, and reusable probes to check them.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Privacy Architecture, the privacy validation in Task 6.5, Model Isolation, and gate G6.3. Test only the Phase 0 harnesses and minimal adapters. Do not claim anything about production-app security.
>
> Run representative cases from ingestion, merchants, routing, reasoning, insights and voice with airplane mode on or networking blocked. Capture the traffic of the app process with tools on the device and on the network. Assert zero packets containing financial data, full account numbers, descriptors, prompts or outputs. List every dependency, including LiteRT-LM, any llama.cpp build and the speech-recognition runtime, with its network behavior and telemetry.
>
> Check that:
> - rendered page images and other temporary files are private to the app and deleted after use;
> - stored raw model outputs are private to the app;
> - runtime and app logs contain no prompts, outputs or raw financial data at release log levels;
> - account-number redaction works on adversarial fixtures;
> - model files stay read-only;
> - inference sessions keep no user data in the weights or between tests;
> - repeated prompts cannot recover earlier fixtures.
>
> Document the limits of these checks. Process memory cannot be proven clean without instrumentation, and traffic from the operating system must be told apart from the app's own traffic.
>
> Find out, from current documentation, how 3–4 GB of model files can reach the device without the app holding the INTERNET permission. Examine Play Asset Delivery and Google Play's on-device AI delivery, where the Play Store downloads the files instead of the app, and record their current size limits. Also document sideloading over adb for demo builds.
>
> Produce reusable privacy probes and a checklist of production controls:
> - no INTERNET permission;
> - no analytics or crash-reporting SDKs;
> - SQLCipher with an Android Keystore key;
> - a full data wipe;
> - model isolation;
> - a model-delivery path;
> - final acceptance checks with tcpdump or Wireshark.
>
> Write `docs/PRODUCTION_PRIVACY_REQUIREMENTS.md`. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `research/privacy_tests/` and probes in `apps/android-bench`
- `results/privacy_validation.json`
- `docs/PRODUCTION_PRIVACY_REQUIREMENTS.md`

**Done when:** The pre-production part of G6.3 passes, or it blocks approval. Offline inference is proven for each selected runtime. A way to deliver the models is identified. Remaining risks, and the tests the production app must rerun, are documented.

---

### Session 14: Architecture decision and feasibility report (v1 Session 10)

**Phase:** 0 — Exit gate
**Prerequisites:** Sessions 1–13. G1.1 must have been measured on the shipping artifact (S7, or its follow-up run).

**Scope:**
- Bring together the evidence for every gate, without re-running anything or changing any holdout.
- Choose the extraction option (A, B, C or D), the text model, the router, speech recognition, how models are loaded, the supported devices, how models are delivered, and the fallbacks.
- Adopt the new latency targets as concrete numbers.
- Issue one formal outcome.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md`, `docs/SESSION_PLAN_V2.md`, `docs/EVALUATION_CONTRACT.md`, every Phase 0 report in `docs/`, and the raw result manifests. Do not write production code.
>
> Produce an architecture decision record and a feasibility report. Give every gate in §7 its own row with: the threshold, the measured result, the dataset and its hash, the hardware, the confidence interval or variance, PASS, FAIL or UNPROVEN, and the fail action.
>
> Explicitly reconcile:
> - G0.2's exploratory 85% with G1.1's approval threshold of 90%;
> - G0.3's 80% contingency trigger with G2.1's approval threshold of 85%;
> - G3.2's formal 75% with the conservative 80% used for approval;
> - G5.3's 3 s with the stricter 2 s lookup target;
> - the Mac reference with the shipping artifact (G1.4);
> - synthetic statements with real ones.
>
> Decide:
> - the extraction option (A, B, C or D from plan §5), with its artifact hash, runtime, backend and output enforcement;
> - the text model and router;
> - the speech-recognition runtime;
> - which models are loaded together and which on demand;
> - the minimum device requirements;
> - how models are delivered.
>
> Adopt numeric targets for G0.4, G1.3 and the ingestion part of G6.2, starting from S8's proposals. Record why v1's values were not adopted.
>
> Choose exactly one outcome:
> - **PROCEED:** every gate critical to the architecture passes, and no unresolved dependency threatens production.
> - **PROCEED WITH CHANGES:** feasibility is proven only with named changes. Examples: llama.cpp alongside LiteRT-LM (option C), the 2B model, models loaded on demand, a different speech-recognition model, support limited to some institutions, a manual review step, a fine-tuned FunctionGemma, or narrower device support. Update the target architecture, and the assumptions of every later session, in the report.
> - **STOP/PIVOT:** one or more critical capabilities still fail, and no tested fallback meets the product goal. For example, no option reaches G1.1 on real statements with the shipping artifact.
>
> G6.1 stays deferred. The human gates cannot be called passed without completed human reviews. Include a signed decision checklist, and link every claim to raw evidence. Record the decision as a D-entry. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `docs/ADR-001-phase0-architecture.md`
- `docs/PHASE0_FEASIBILITY_REPORT.md`
- `results/gate_summary.json`
- A D-entry for the decision

**Done when:** One outcome is recorded. Every gate can be traced to its evidence. Required changes are concrete. Latency targets are numbers. Production scaffolding is allowed only after PROCEED or PROCEED WITH CHANGES.

---

### Session 15: Production Android scaffolding (v1 Session 11)

**Phase:** 1 — Foundation
**Prerequisites:** Session 14 outcome is PROCEED or PROCEED WITH CHANGES

**Scope:**
- A Kotlin and Compose project at `apps/android/`, with min SDK 28, target SDK 35 and the arm64-v8a ABI.
- These modules:
  - `:app`
  - `:core:database`
  - `:core:ml`, which wraps the runtime(s) the architecture decision selected
  - `:core:extraction`, pure Kotlin with no Android dependencies
  - `:feature:ingestion`, `:feature:merchants`, `:feature:crossaccount`, `:feature:insights`, `:feature:chat` and `:feature:settings`
- Hilt dependency injection.
- A shell with four tabs, using Material 3.
- No INTERNET permission.
- No ML Kit.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` and `docs/ADR-001-phase0-architecture.md`. Stop if Session 14 did not authorize production.
>
> Scaffold the production Kotlin and Compose app at `apps/android/`, separate from `apps/android-bench/`. Use only the runtimes, backends, model-loading policy, supported devices and fallbacks that the decision record selected. Configure:
> - Room with SQLCipher;
> - the pinned LiteRT-LM version;
> - if the decision record chose option C, a pinned llama.cpp built through CMake and wrapped with JNI;
> - coroutines and Flow;
> - Hilt;
> - Navigation Compose;
> - Material 3.
>
> Do not add ML Kit.
>
> Create the modules listed above. `:core:extraction` must not depend on Android. Model files must not be packed into the APK; they load from the delivery location the decision record chose. Build a four-tab shell (Import, Transactions, Insights, Chat) with placeholders. Omit the INTERNET permission and set `usesCleartextTraffic` to false. Keep Phase 0 harness code out of the production dependencies. Append an entry to `docs/STATE.md`.

**Key outputs:**
- Gradle and settings files, and the modules
- `MainActivity.kt`, navigation and theme
- The manifest
- Native build configuration, if option C was chosen

**Done when:** A clean checkout builds. The app launches with a four-tab shell in light and dark themes. Dependency versions are pinned. The decision record's constraints are reflected in the code.

---

### Session 16: Database, SQLCipher, model metadata and extraction provenance (v1 Session 12)

**Phase:** 1 — Foundation
**Prerequisites:** Session 15

**Scope:**
- All the execution plan's entities and tables, database access objects, indices and the 53-category taxonomy.
- SQLCipher with a key protected by the Android Keystore.
- Model metadata, and a record of exactly how each page was extracted.
- Storage for statement metadata, and per-page status so ingestion can resume.

**Prompt:**
> Read the exact SQLite schema, the category taxonomy, the Privacy Architecture and the Model Update Strategy in `docs/EXECUTION_PLAN.md`, plus the decision record and `docs/EXTRACTION_GAPS_REPORT.md`.
>
> Implement Room entities for these tables, with exact constraints, indices, cents and date conventions, cascades and model-version fields:
> - accounts, statements and transactions;
> - merchant resolutions and merchant maps;
> - cross-account links and the transactions they link;
> - insights and insight transactions;
> - conversations and messages;
> - categories;
> - model metadata.
>
> Extend the schema beyond v1:
> - Statement metadata from S3: institution, account type, account number reduced to its last four digits, currency, period, and opening and closing balances.
> - A per-page ingestion status (pending, done, cut-off, failed or flagged), so an import can resume.
> - A provenance record for every extracted page: model artifact hash, runtime and version, backend, prompt variant and its sha256, schema sha256, render resolution, decoding settings and warnings.
> - The raw model output for each page, encrypted, with the retention policy the decision record sets, so pages can be audited and reprocessed.
>
> Implement the database access objects that ingestion, merchants, cross-account, insights and chat need.
>
> Configure SQLCipher with a key that cannot be extracted: an Android Keystore wrapping or derivation design that actually works on Android. Do not assume `SecretKey.encoded` returns usable key bytes without verifying it.
>
> Seed all 53 categories. Test encryption, migrations, cascades and queries. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `core/database` entities, access objects, `AppDatabase`, the seeder, the Hilt database module and tests

**Done when:**
- The encrypted database opens, and cannot be read without its key.
- The taxonomy is seeded exactly once.
- Access-object and cascade tests pass.
- Each extracted page can be traced to its model, prompt, schema and runtime.

---

### Session 17: Kotlin extraction core, checked by golden replay (new)

**Phase:** 1 — Foundation
**Prerequisites:** Session 15, and the extraction decisions from Sessions 2, 3 and 14

**Scope:**
- Export a set of reference test files that do not depend on any programming language, from the Python pipeline.
- Port the deterministic part of the pipeline to `:core:extraction`: prompts and schemas (byte-identical), JSON repair, parsing, dates, the year step, metadata, balance validation and normalization.
- Show that the port matches Python exactly on every recorded output.

**Prompt:**
> Read the handoff §2, §4 and §6; D-028 and every later extraction D-entry; and all of `statement_extractor/` (`prompts.py`, `parsing.py`, `json_repair.py`, `dates.py`, `year_resolution.py`, `types.py`, and S3's metadata and validation code), with its tests.
>
> First write `research/scripts/export_replay_fixtures.py`. It exports a set of test files that do not depend on any programming language:
> - the exact prompt text and schema bytes for every variant the decision record selected, with their sha256 values;
> - the recorded raw model outputs from the D-028 reference runs, S2's compact-output and stress runs, S3's rollover and metadata runs, and S7's on-device runs;
> - for each raw output, the Python pipeline's canonical result after parsing, year resolution and validation.
>
> Exclude real statements. Version and hash the fixture set; a future Swift port will reuse it.
>
> Port the deterministic pipeline to `:core:extraction` in pure Kotlin, with no Android dependencies. The port covers:
> - loading prompts and schemas, byte-identical to the originals, with tests that check the pinned hashes;
> - building requests for the selected runtime;
> - JSON repair and parsing;
> - date parsing and the year step, including the rollover rule;
> - metadata parsing and balance validation;
> - normalizing rows to whole cents with a direction.
>
> Port the strict scorer into a test-only source set.
>
> **Golden replay:**
> - Every fixture must reproduce the Python canonical output exactly.
> - Re-scoring the reference runs in Kotlin must reproduce the published numbers: 97.2% for Qwen3.5-4B, 89.2% for 2B and 81.6% for Gemma 4 E4B, plus the S2, S3 and S7 results.
>
> Any mismatch is a bug in the port, not a tolerance. Append an entry to `docs/STATE.md`.

**Key outputs:**
- `research/scripts/export_replay_fixtures.py` and a versioned fixture set
- The `core/extraction` sources and tests

**Done when:** `./gradlew :core:extraction:test` passes with exact agreement on every fixture. Prompts and schemas are pinned by hash. The fixture set is versioned and recorded in a D-entry.

---

### Session 18: Production import, page rendering and on-device model execution (replaces v1 Sessions 13–14)

**Phase:** 1A — Ingestion
**Prerequisites:** Sessions 15–17

**Scope:**
- Importing PDFs and images through the system file picker, plus optional camera capture.
- Rendering pages one at a time with `PdfRenderer`, at the resolution the decision record chose.
- `LlmEngineManager`: the runtime the decision record chose, the loading policy, output enforcement, the cut-off policy and thermal back-off.
- Background ingestion that can resume, with progress and a live preview.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 1A.1 and 1B.1, the decision record, `docs/ANDROID_EXTRACTION_PARITY.md` and `docs/ANDROID_INGESTION_TIMING.md`. ML Kit OCR, column reordering and vision table analysis are superseded (plan §11); do not build them.
>
> **Import and rendering.**
> - Import PDFs and images through `ACTION_OPEN_DOCUMENT`, with optional camera capture that guides cropping.
> - Render one page at a time with `PdfRenderer`, at the resolution the decision record chose, and recycle bitmaps strictly.
> - Use exactly the preprocessing validated in S7. Send image files as given.
> - Write temporary page images only to app-private storage, and delete them after use.
>
> **`LlmEngineManager`,** in `:core:ml`, implements the runtime or runtimes and the loading policy from the decision record. It must:
> - load models from the delivery location and verify their hashes;
> - use the selected backend;
> - decode greedily with the fixed seed;
> - apply the selected output enforcement, the token limits and S2's cut-off policy;
> - support cancellation;
> - slow down when `PowerManager` reports high thermal status.
>
> Record the provenance fields from S16 on every page result.
>
> **Background ingestion.** Run ingestion as a background job that can resume, using the mechanism proven in S8:
> - save each finished page to the encrypted database as a checkpoint;
> - continue after the app's process is killed;
> - show a notification with progress;
> - allow cancellation.
>
> Build the Import screen and its ViewModel, with progress page by page and a live preview of extracted rows.
>
> **Regression test.** Add a test that runs a fixed set of fixture PDFs on the device and compares the raw outputs and parsed rows with S7's parity outputs, within the documented tolerance. Append an entry to `docs/STATE.md`.

**Key outputs:**
- The import screen and ViewModel
- The renderer and the camera-capture flow
- `LlmEngineManager` and its dependency-injection setup
- The background ingestion worker
- Device regression tests

**Done when:** On-device outputs reproduce S7's within tolerance. An import survives being sent to the background and having its process killed. A 30-page PDF imports without running out of memory. No network is needed.

---

### Session 19: Normalization, validation, de-duplication and ingestion orchestration (v1 Session 15)

**Phase:** 1C — Ingestion
**Prerequisites:** Sessions 16–18

**Scope:**
- Only the edge-case handling that S3 and S4 showed is needed.
- Merging rows across page boundaries; de-duplicating pending versus posted rows and overlapping statements.
- Balance validation that puts problem statements into a review state.
- A minimal screen to fix flagged rows.
- `IngestionOrchestrator`, running from PDF to encrypted database.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 1B.3 and 1C.1–1C.3, `docs/EXTRACTION_GAPS_REPORT.md`, `docs/REAL_STATEMENT_VALIDATION.md` and the decision record. Implement only the edge-case handling that S3 and S4 evidence marks as needed. For each v1 edge case, link the evidence for the choice from the code or the report.
>
> Build `IngestionOrchestrator`, which runs these steps in order:
> 1. render the page;
> 2. send the metadata request;
> 3. send the extraction request for each page;
> 4. parse and repair the output;
> 5. apply the year step;
> 6. normalize: convert to cents, derive a signed amount from the direction, and clean the descriptor while keeping the raw descriptor;
> 7. merge rows across page boundaries;
> 8. de-duplicate pending versus posted rows and overlapping statements;
> 9. validate balances;
> 10. insert into the database in one transaction, with the statement's status.
>
> Put statements and pages into a review state with actionable detail when a balance or running-balance check fails, when a page is cut off, or when a page has low confidence. Add a minimal screen where the user edits or confirms the rows on flagged pages. Never drop rows silently, and never correct amounts automatically.
>
> Integration tests run the whole flow from PDF to database on fixtures. They repeat G1.1–G1.6 as regression tests for the implementation: on synthetic data on the JVM by replay, and on a subset on the S25 Ultra. Append an entry to `docs/STATE.md`.

**Key outputs:**
- Normalizer, page-boundary merger and de-duplicator
- Balance validator and review screen
- The orchestrator and its progress model
- Integration tests

**Done when:** The full flow from PDF to database passes integration tests on the fixtures, repeating G1.1–G1.6 as regression tests. Flagged statements reach the review screen with the reason they were flagged.

---

### Session 20: Production merchant resolution engines (v1 Session 16)

**Phase:** 2 — Merchant resolution
**Prerequisites:** Sessions 16, 18 and 19

**Scope:**
- A rules baseline with at least 100 rules.
- A model-based resolver using the text model the decision record chose.
- Preprocessing of descriptors, batching and caching, and storage of results.
- One interface shared by both resolvers.
- Loading the text model after ingestion, under the chosen loading policy.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 2.1–2.3, `docs/MERCHANT_CROSSACCOUNT_FEASIBILITY.md` and the decision record.
>
> Implement a rules baseline of at least 100 regular expressions. Implement a production resolver using the text model the decision record chose. It must:
> - work in batches of ten descriptors, with stable response indexing;
> - return a confidence and its reasoning;
> - preprocess point-of-sale descriptors;
> - use context about payment intermediaries;
> - handle Amazon and Uber sub-brands.
>
> Load the model under the decision record's loading policy, swapping it with the extraction model when needed. Use the fallback tested in S9 when a batch fails, and cache resolutions across statements.
>
> Store the resolutions, the mapping to transactions, the model version, and the raw descriptor so it stays visible. Keep the baseline honest: do not cripple the rules beyond their documented limits. Append an entry to `docs/STATE.md`.

**Key outputs:** The resolver interface, the rules and model-based resolvers, the preprocessor, the service and tests.

**Done when:** Regression tests of the production resolver meet G2.1 and G2.3, and compute the G2.2 comparison rather than hard-coding it.

---

### Session 21: Merchant review and comparison UI (v1 Session 17)

**Phase:** 2 — Merchant resolution
**Prerequisites:** Session 20

**Scope:**
- A review queue for confidence below 0.70, with accept and correct actions.
- Corrections that persist and become few-shot examples.
- A side-by-side comparison of the rules baseline and the model, with statistics.
- A transaction list with details.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 2.4–2.5.
>
> Build a review queue for resolutions with confidence below 0.70. Each entry shows the original descriptor, the suggestion, its category, confidence and reasoning, with accept and edit actions and a picker for the taxonomy.
>
> Persist verified corrections. Use the most recent 20 as few-shot examples, without contaminating any frozen evaluation holdout.
>
> Build an accurate side-by-side comparison of the rules baseline and the model. Highlight unresolved descriptors and disagreements, and compute the summary statistics.
>
> Add the production transaction list, grouped by date, showing the merchant, category and account, with links back to the source page and the raw descriptor. Append an entry to `docs/STATE.md`.

**Key outputs:** The merchant screens, components and ViewModel, with UI tests.

**Done when:** Corrections persist, and affect only the future prompts they are allowed to affect. Comparison numbers match the underlying records. Every transaction can be traced to its raw descriptor and source page.

---

### Session 22: Production cross-account detection (v1 Session 18)

**Phase:** 3 — Cross-account intelligence
**Prerequisites:** Sessions 16, 18 and 20

**Scope:**
- Detectors for transfers, duplicates, refunds and subscription migrations.
- SQL filters that find candidates first.
- Confirmation by the text model.
- Storage of links, and the orchestration that runs the detectors.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 3.1–3.4 and `docs/MERCHANT_CROSSACCOUNT_FEASIBILITY.md`.
>
> Implement efficient SQL filters, and detectors that can each be tested on their own:
> - transfers: matching amounts within $1 and dates within 3 days, across accounts;
> - duplicates: candidates with the same merchant, amount and date;
> - refunds: credits that match an earlier debit within 30 days;
> - subscription migrations: a subscription that stops while a new one starts, matching the timing and category criteria and within ±20% in amount.
>
> Use the selected text model for confirmation, at low temperature, only after the plain-code step has generated candidates.
>
> Store each link with the role of each transaction, the confidence, the reasoning, whether the user confirmed it, and the model version. Never merge financial records automatically. Append an entry to `docs/STATE.md`.

**Key outputs:** The four detectors, the orchestrator, database queries, and fixture and integration tests.

**Done when:** Production fixture tests repeat G3.1–G3.3, with an error report for each link type and no shortcuts specific to the fixtures.

---

### Session 23: Cross-account UI and net cashflow (v1 Session 19)

**Phase:** 3 — Cross-account intelligence
**Prerequisites:** Session 22

**Scope:**
- Animated links between accounts, color-coded by type.
- An explanation for each link, with confirm and reject actions.
- An account overview.
- Cashflow totals that exclude internal transfers.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 3.5–3.6.
>
> Build the cross-account view in Compose:
> - animated curved (Bézier) links in the specified green, red, blue and orange type colors;
> - accessible detail cards showing confidence and reasoning;
> - actions for the user to confirm or reject each link.
>
> Implement gross and net income and expense totals. They exclude both sides of confirmed internal transfers, while keeping all the original transactions.
>
> Add an account overview, and deterministic cashflow tests, including the seeded $500 transfer. Append an entry to `docs/STATE.md`.

**Key outputs:** The cross-account UI and ViewModel, the cashflow calculator and cards, and tests.

**Done when:** Links render smoothly on the target devices, confirmations persist, and cashflow totals on the fixtures are exact.

---

### Session 24: Production insight engine (v1 Session 20)

**Phase:** 4 — Proactive insights
**Prerequisites:** Sessions 16 and 20

**Scope:**
- An engine that accepts insight generators as plug-ins.
- Generators for subscriptions, anomalies, fees, micro-charges, largest transactions and income regularity.
- Ranking.
- Narration by the text model, checked against the computed data.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 4.1–4.6, Appendix A5, and `docs/INSIGHT_FEASIBILITY.md`.
>
> Implement all six generators, computing everything in SQL before any narration. Require at least three months of data before annualizing any amount. Keep the structured data behind each insight, and the transactions linked to it.
>
> Narrate with the selected text model, and check every amount and date in the narration against that data. Rank insights by financial impact multiplied by a surprise factor, and store the model version. Reuse the approved prompts, without including holdout answers. Append an entry to `docs/STATE.md`.

**Key outputs:** The engine and its interface, the six generators, the narration checker, storage and regression tests.

**Done when:** All six generators run, the arithmetic tests pass, and the regression tests repeat G4.1 and the checks that narration matches the data.

---

### Session 25: Insight display UI (v1 Session 21)

**Phase:** 4 — Proactive insights
**Prerequisites:** Session 24

**Scope:**
- The top five insight cards, appearing one after another with an animation.
- Cards that expand to show the transactions behind them.
- Labels for financial impact.
- Swipe to dismiss, and restore.
- An empty state.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Task 4.7.
>
> Build the Insights screen:
> - the top five ranked cards, each sliding and fading in after a delay of `index * 300ms`;
> - an icon for each insight type, and the financial impact;
> - the transactions behind each card, shown on expansion;
> - the remaining insights in an expandable section below.
>
> Implement swipe to dismiss, and restore, using `is_dismissed`. Regenerate insights after each ingestion. Add accessibility semantics, and empty, error and loading states. Append an entry to `docs/STATE.md`.

**Key outputs:** The insight screen, cards and ViewModel, with UI tests.

**Done when:** Ranking, dismiss and restore work correctly. Cards show their supporting data. Animations stay smooth on the target devices.

---

### Session 26: Production query router and function executor (v1 Session 22)

**Phase:** 5A — Conversational Q&A
**Prerequisites:** Sessions 16 and 18, and the router the decision record selected

**Scope:**
- The selected router: a FunctionGemma artifact, or the text model's own tool calling.
- The full schema of 11 functions.
- An SQL executor.
- Formatting of lookup results.
- A fallback to the text model when the router's confidence is low.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Task 5A.1, Appendix B, `docs/ROUTING_REASONING_FEASIBILITY.md` and the decision record.
>
> Integrate the router the decision record selected: the base, few-shot or fine-tuned FunctionGemma artifact, or the text model's own tool calling. Record its exact version and hash, and apply the approved latency and loading strategy.
>
> Implement:
> - routing into LOOKUP, COMPARISON, REASONING and SYNTHESIS;
> - strict parsing of tool calls;
> - all 11 functions, with parameterized queries;
> - normalization of relative dates;
> - a fallback to the text model when confidence is low;
> - human-readable formatting of lookup results.
>
> Do not regenerate the frozen holdout or train on it. Append an entry to `docs/STATE.md`.

**Key outputs:** The router, the schema and call models, the executor, the formatter and tests.

**Done when:** Every function returns the correct fixture data, and the production integration repeats G5.1 and the lookup-latency regression tests.

---

### Session 27: Reasoning, conversation and tool loop (v1 Session 23)

**Phase:** 5A — Conversational Q&A
**Prerequisites:** Session 26; Sessions 22 and 24 for full context

**Scope:**
- Building the model's context from function results.
- Answers from the text model, limited to the evidence it was given.
- Conversations with several turns that persist.
- Resolving follow-up questions.
- A function-calling loop with at most three iterations.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 5A.2–5A.4 and `docs/ROUTING_REASONING_FEASIBILITY.md`.
>
> Build the model's context from explicit function results and the last six messages, with account numbers redacted. The selected text model must answer only from the evidence provided. Persist conversations, messages and tool calls.
>
> Resolve follow-up questions through the conversation history. Implement a function-calling loop limited to three iterations, with a strict allowlist, argument validation, timeouts and graceful failure. Add citation checking, and a response for when there is not enough data. Append an entry to `docs/STATE.md`.

**Key outputs:** The reasoning engine, the conversation manager, the tool loop, and tests for grounding and for multi-turn conversations.

**Done when:**
- Known-answer production tests repeat G5.2.
- Citations point to real records.
- Follow-up questions persist.
- Malformed or repeated tool calls fail safely.

---

### Session 28: Production voice and chat UI (v1 Session 24)

**Phase:** 5B — Voice and chat
**Prerequisites:** Sessions 12 and 27

**Scope:**
- The offline speech-recognition runtime the decision record chose.
- Push-to-talk.
- The pipeline from voice to query to spoken answer.
- A text and voice chat UI, permissions, and a way to correct the transcript.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 5B.1–5B.3 and `docs/VOICE_RUNTIME_DECISION.md`.
>
> Integrate the selected offline speech-recognition runtime; do not assume Moonshine's API. Implement:
> - push-to-talk capture at 16 kHz;
> - events for partial results, final results and silence;
> - resource handling that is safe across the Android lifecycle;
> - the RECORD_AUDIO permission.
>
> Show the transcript so the user can confirm or correct it before it is sent. Pass the final text to the ConversationManager, and speak the answer with Android text-to-speech, following the decision record's model-loading policy.
>
> Build the chat UI:
> - accessible message bubbles and text input;
> - the microphone state with a waveform;
> - the partial transcript and a typing indicator;
> - cancellation;
> - an indicator on voice messages.
>
> Append an entry to `docs/STATE.md`.

**Key outputs:** The voice input and Q&A engines, the chat screen, its components and ViewModel, and tests.

**Done when:**
- Text and voice conversations work offline.
- The app recovers from lifecycle and permission failures.
- Acceptance tests repeat G5.3–G5.4 and the voice targets in G6.2.

---

### Session 29: Production privacy and security (v1 Session 25)

**Phase:** Cross-cutting
**Prerequisites:** Sessions 15 and 16; must finish before Session 30

**Scope:**
- Hardening of the Keystore and SQLCipher setup.
- Enforcing that the app has no network access.
- Redacting account numbers.
- Wiping all data, or one account's data.
- Model isolation, integrity and versioning.
- Handling of rendered page images and raw outputs.
- Runtime log levels.
- The model-delivery path.
- The Settings UI.

**Prompt:**
> Read the Privacy Architecture, Model Isolation and Model Update Strategy sections of `docs/EXECUTION_PLAN.md`, plus `docs/PRODUCTION_PRIVACY_REQUIREMENTS.md` and the decision record.
>
> **Keys and network.**
> - Harden how the SQLCipher key is protected by the Keystore, and verify that no key or plaintext is ever logged.
> - Keep the INTERNET permission absent, block cleartext traffic, and include no telemetry, analytics or crash-reporting SDKs.
> - Add probes to debug and demo builds that confirm network calls fail.
>
> **Model runtimes.**
> - Set the LiteRT-LM log level, and the llama.cpp log level if it is used, so release builds never log prompts or outputs.
> - Verify the hash of every model artifact before loading it.
> - Implement the delivery path the decision record chose.
>
> **Data handling.**
> - Redact account numbers of 8 to 19 digits before they are stored or logged, keeping only the last four digits for display.
> - Delete rendered page images after use.
> - Store raw model outputs encrypted, under the decided retention policy.
>
> **Wipe.**
> - A full wipe removes, in one operation, the database, temporary and cache files, rendered pages, raw PDFs, raw outputs, preferences and keys.
> - A per-account wipe removes that account's data through database cascades.
> - Both need a confirmation screen.
>
> **Models.** Verify that model files are read-only, that inference sessions are isolated, and that every record derived from a model stores the model's version and hash.
>
> Append an entry to `docs/STATE.md`.

**Key outputs:**
- Security configuration for the app
- The redaction module and wipe manager
- The Settings screen
- Security tests and checklist

**Done when:** Tests pass for encryption, redaction, wiping, the dependency audit, model isolation and integrity, and offline operation. The Phase 0 privacy probes are adapted for final acceptance.

---

### Session 30: Demo orchestration and data refresh (v1 Session 26)

**Phase:** 6 — Demo polish
**Prerequisites:** Sessions 19, 21, 23, 25, 28 and 29

**Scope:**
- The six-beat demo: Drop → Parse → Detective → Reveal → Conversation → Proof.
- A network overlay, a privacy-proof screen, and polished callouts comparing rules with the model.
- Demo data refreshed from S5, not created for the first time.
- A demo-preparation run for ingestion, which now takes minutes rather than seconds.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Tasks 6.1–6.4 and Appendix C, `docs/ANDROID_INGESTION_TIMING.md` and the decision record. Refresh the S5 generator only as far as the implemented schemas require. Keep S5's frozen release unchanged, and create a separately versioned demo release. Bundle the 18 PDFs through the asset strategy the decision record chose.
>
> Ingestion takes minutes per statement, so the demo works like this:
> - **Demo preparation.** The production ingestion pipeline ingests 17 statements on the demo device ahead of time. Record the timestamps and provenance of that run, and show them on the Proof screen.
> - **The Parse beat.** One short statement is ingested live through the same pipeline, with progress page by page and rows appearing as they are extracted.
> - **Fallback.** If live ingestion fails, the presenter can switch to the prepared result, which is clearly labeled as such.
>
> Implement a demo orchestrator that the presenter can interrupt, covering Drop → Parse → Detective → Reveal → Conversation → Proof. Add a `TrafficStats` network overlay with zero and non-zero states, a privacy-proof screen, and polished callouts comparing rules with the model on the seeded traps. Do not hard-code analytical results; the demo must run the production pipelines. Append an entry to `docs/STATE.md`.

**Key outputs:**
- The demo orchestrator, data loader and preparation runner
- The network overlay and privacy screen
- A refreshed demo manifest and assets

**Done when:** All six beats run, both manually and through the orchestrator. Seeded scenarios are produced by the real feature logic. Preparation runs are recorded with provenance. Refreshed data stays consistent with the canonical ground truth.

---

### Session 31: Post-implementation regression and acceptance (v1 Session 27)

**Phase:** 6 — Final validation
**Prerequisites:** Sessions 19–30

**Scope:**
- Implementation acceptance only. No first-time feasibility work and no training.
- Regression on every gate, including G1.4–G1.6 and the latency targets from the decision record.
- Stability, privacy and demo acceptance on both devices.

**Prompt:**
> Read `docs/EXECUTION_PLAN.md` Task 6.5, `docs/EVALUATION_CONTRACT.md`, the decision record and all production modules. Do not start any first-time feasibility work, and do not train on holdouts. Adapt the frozen evaluators to the production interfaces, and run them without changing corpora, prompts or thresholds to improve the results.
>
> **Repeat every test:**
> - extraction on synthetic data and real holdouts (G1.1), and runtime parity (G1.4);
> - cut-off recovery (G1.5) and December-to-January statements (G1.6);
> - balance validation;
> - ingestion latency, battery and heat, against the decision record's targets (G0.4, G1.3 and G6.2);
> - merchants, cross-account links and insights;
> - routing, and reasoning with citations;
> - voice;
> - privacy.
>
> **Compare and confirm:**
> - Compare production results with the Phase 0 baselines and flag any regression.
> - Confirm the Kotlin extraction core still reproduces the golden replay fixtures exactly.
> - Run the complete demo repeatedly on the Samsung S25 Ultra and the Pixel 9 Pro. G6.1 requires no crashes on either.
> - Validate:
>   - G6.2 with end-to-end production timings;
>   - G6.3 with packet capture filtered to the app (tcpdump or Wireshark), plus the model-delivery path;
>   - G6.4 with the defined human protocol.
> - Verify that model versions are recorded, that wipes work, that no holdout leaked into training or prompts, and that no financial data appears in logs or traffic.
>
> Produce a release acceptance report with PASS or FAIL for each gate, the change from the Phase 0 baselines, known limitations and anything that blocks release. A failed gate is never waived silently; record an owner, a mitigation and an explicit decision. Append an entry to `docs/STATE.md`.

**Key outputs:**
- The production `EvaluationRunner` and test adapters
- `results/acceptance/*.json`
- `docs/FINAL_ACCEPTANCE_REPORT.md`

**Done when:** Every production acceptance gate, including G1.4–G1.6 and G6.1–G6.4, has traceable evidence. Regressions are reported. No late session was used to discover basic model or framework feasibility for the first time.

---

## 10. Dependency graph

```text
PHASE 0A — Mac only (no device)
S1 Consolidation + evaluation contract
 ├─► S2 Output efficiency (compact JSON, cut-off recovery, token budgets)
 ├─► S3 Correctness gaps (rollover, metadata, balance, page boundaries)
 ├─► S4 Real statements            ◄── real corpora (you); S3 needed for G1.2
 └─► S5 Scenario PDFs + fixtures for later features

PHASE 0B — Android device (S25 Ultra)
S1 ─► S6 Toolchain + LiteRT-LM benchmark
S2 + S6 ─► S7 Extraction artifact + accuracy parity (options A → C/B)
S3 + S7 ─► S8 Ingestion timing, memory, heat, battery → proposed latency targets

PHASE 0C — Features after extraction
S5 + S6 + S7 ─► S9 Text model choice, merchants, cross-account
S5 + S6 (+S9) ─► S10 Routing + reasoning
S5 + S9 ─► S11 Insights + human evaluation
S6 + S10 ─► S12 Voice
S5–S12 ─► S13 Offline, privacy, model delivery

EXIT GATE
S1–S13 (+ G1.1 measured on the shipping artifact) ─► S14 Decision: PROCEED / PROCEED WITH CHANGES / STOP-PIVOT

PRODUCTION — blocked until S14 authorizes it
S14 ─► S15 Scaffolding ─► S16 Database
S15 ─► S17 Kotlin extraction core (golden replay)
S16 + S17 ─► S18 Import + on-device model ─► S19 Orchestration ─► S20 Merchant engine
S20 ─► S21 Merchant UI
S20 ─► S22 Cross-account engine ─► S23 Cross-account UI
S20 ─► S24 Insight engine ─► S25 Insight UI
S16 + S18 ─► S26 Router
S26 + S22 + S24 ─► S27 Reasoning ─► S28 Voice/chat
S16 ─► S29 Privacy/security
S19 + S21 + S23 + S25 + S28 + S29 ─► S30 Demo
S19–S30 ─► S31 Regression and acceptance
```

---

## 11. Where each v1 item went

### Sessions

| v1 session | Where it is now |
|---|---|
| 1 Evaluation contract and corpora | **S1.** The extraction corpus, scorer and ground truth already exist; S1 adds the remaining corpora and maps the gates. |
| 2 Synthetic statements and fixtures | **S5**, reusing the existing template printer |
| 3 LiteRT-LM runtime | **S6** for the text models; **S7–S8** for the extraction model |
| 4 OCR, vision, extraction, timing | **Mostly done** by D-028, which chose the method. The rest moved to S2 (output efficiency), S3 (correctness gaps), S4 (real statements), S7 (on-device parity) and S8 (timing). |
| 5 Merchants and cross-account | **S9**, which also chooses the text model |
| 6 Routing and reasoning | **S10**, which also tests the text model's own tool calling |
| 7 Insights and human evaluation | **S11** |
| 8 Voice | **S12** |
| 9 Offline and privacy | **S13**, which adds model delivery, rendered pages and runtime logs |
| 10 Architecture decision | **S14**, which adds the option A–D choice and the new latency targets |
| 11 Scaffolding | **S15**: no ML Kit; adds `:core:extraction` |
| 12 Database | **S16**: adds provenance, statement metadata and per-page status |
| 13 PDF import and OCR | **S18**, with OCR and column reordering removed |
| 14 LiteRT-LM and structured extraction | **S17** (Kotlin port) and **S18** (model execution) |
| 15 Normalization, validation, orchestration | **S19**, keeping only the edge cases that evidence shows are needed |
| 16–25 | **S20–S29**, unchanged in substance. "E4B" now means the text model the decision record chose. |
| 26 Demo | **S30**, which adds the demo-preparation run and a live single-statement Parse beat |
| 27 Acceptance | **S31**, which adds G1.4–G1.6, the new latency targets and golden-replay parity |

### Sections of `EXECUTION_PLAN.md` that are superseded or amended

These are the execution-plan sections that v1 sessions pointed to. Wherever `EXECUTION_PLAN.md` conflicts with this plan or with `docs/DECISIONS.md`, the decisions log wins.

| Section | Status |
|---|---|
| Task 0.2 (ML Kit OCR plus E4B extraction quality) | **Superseded** by the D-028 method. G0.2 now measures strict accuracy with the D-028 method. |
| Task 0.5 and the ingestion-timing gates (G0.4, G1.3, ingestion part of G6.2) | **Amended:** measured in S8, and the targets are set in S14 |
| Task 1A.2 (ML Kit OCR) | **Removed** |
| Task 1A.3 (column ordering, removal of headers and footers) | **Removed.** The vision model reads the layout. Page-boundary handling follows S3's evidence. |
| Task 1B.1 (model manager; vision table-analysis fallback) | **Model manager kept**, using the runtime the decision record chose. **Table-analysis fallback removed.** |
| Task 1B.2 and Appendix A1–A2 (extraction prompts) | **Replaced** by the frozen `json-period` prompt and schema plus S3's `statement-metadata` variant |
| Tasks 1B.3 and 1C.1–1C.3 (normalization, edge cases, de-duplication, balance) | **Kept**, trimmed to the cases S3 and S4 showed are needed |
| Gemma 4 E4B named as the model for merchants, insights and reasoning | **Amended** to the text model the decision record chose (S9) |
| FunctionGemma named as the router | **Amended** to the router the decision record chose (S10) |
| ML Kit bundled as a dependency | **Removed** |
| Appendix C and Tasks 6.1–6.5 | **Kept** |

---

## 12. Deferred: iOS

iOS is out of scope for this plan. You said it can follow later with similar models and pipeline. To keep that path cheap:

- The golden replay fixtures from S17 do not depend on any programming language. A Swift port of the extraction core can use the same test.
- Prompts and schemas are files pinned by hash, not code, so iOS uses the same bytes.
- The provenance schema in S16 records the runtime and backend, so results from iOS and Android can be compared.
- `apps/ios-bench/` stays as it is. Its measurements are reference points, not gate evidence.
- Known constraints to plan for:
  - Qwen3.5-4B has never been loaded on an iPhone.
  - The increased-memory entitlement that the 4B model probably needs requires a paid Apple developer account.
  - Whether LiteRT-LM runs on iOS with the selected artifact must be checked when iOS planning starts.

Plan iOS as its own Phase 0 after S31. Expect it to repeat S6–S8 (runtime, parity, timing) and S17 (porting the core to Swift).

---

## 13. Assumptions in this plan

These come from v1 or the handoff and were not separately confirmed. Change them here if they are wrong.

1. The target devices are the Samsung S25 Ultra (12 GB) and the Pixel 9 Pro, as in v1.
2. The real corpora are defined as in v1: 100 transcribed pages (50 Canadian, 50 US), and 10 statements from at least 5 institutions.
3. The plan files and `EXECUTION_PLAN.md` live in `docs/`, and the production app lives in `apps/android/`.
4. The thresholds for the new gates are proposals until S1 fixes them in D-031: G1.4 allows a drop of 2.0 points with at least 90% absolute accuracy, and G1.6 requires zero year errors caused by the year step.
5. The frozen `json-period` schema may not return opening and closing balances. S3 checks this before designing the metadata request.
6. The existing template printer can produce the Appendix C demo statements (TD, Scotiabank, RBC). If it cannot, S5 adds templates outside the extraction corpus.
7. The order of investigation, and the recommendation of option C over option B, are recommendations. S14 makes the actual choice.
