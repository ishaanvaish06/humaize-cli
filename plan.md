# AI Humanizer CLI — Build Plan

## 0. Scope & success definition

Before writing code, pin down what "success" actually means, because this determines architecture, budget, and timeline. "95% human, against everything, forever" is not a spec — it's not falsifiable and it decays the moment a detector retrains. The spec below is the falsifiable version of the same ambition.

- **Primary metric (versioned, not absolute)**:
  > *"≥95% pass rate on internal benchmark B1 (§7), measured against detector set {GPTZero vX, Originality.ai vY, Copyleaks vZ, Turnitin vW, ZeroGPT}, model checkpoint pinned, re-verified on a fixed cadence (§7.4)."*
  This number is meaningless without the detector versions and the date it was measured — the plan tracks both.
- **Secondary metrics, required alongside the primary one** (a high evasion number achieved by wrecking the text is not a win):
  1. Semantic fidelity ≥ fixed floor (NLI entailment / BERTScore threshold, set in §5 step 3)
  2. Human-rated naturalness ≥ baseline competitor average on the same benchmark
- **Explicit non-goal**: permanent, universal evasion of all present and future detectors. Detectors retrain adversarially against tools like this one; the plan is built around a refresh loop (§7.4, §9), not a one-time solve.
- **Deliverable**: a CLI (`humanize`) that takes text/file input, runs it through the rewrite pipeline, and outputs humanized text plus a diagnostic report (fidelity score, per-detector pass/fail estimate, burstiness stats, checkpoint version, benchmark date).

---

## 1. Architecture overview

```
Input text
   │
   ▼
[1] Preprocessing & chunking (paragraph-aware)
   │
   ▼
[2] Style-transfer rewrite (fine-tuned LLM, LoRA)
   │
   ▼
[3] Burstiness / structure post-processor
   │
   ▼
[4] Fidelity filter (NLI/BERTScore reranker) ── reject/retry loop
   │
   ▼
[5] Detector-score estimator (local proxy classifiers)
   │
   ▼
Output text + diagnostics report
```

Two build tracks, ship Track A first:

- **Track A (v1, ships fastest)**: SFT-tuned 7–8B open model (LoRA) doing paragraph-level style transfer, with rule-based burstiness/discourse post-processing and a fidelity reranker. No RL yet.
- **Track B (v2)**: Add a DPO/RLHF pass using an ensemble of local detector-proxy classifiers as reward signal, to close the gap that pure imitation-learning leaves.

---

## 2. Model selection

| Component | Choice | Why |
|---|---|---|
| Base LLM | Mistral-7B-Instruct or Llama-3-8B-Instruct | Strong instruction-following, cheap to LoRA fine-tune on a single GPU, permissive-enough license for commercial use (verify current license terms before shipping) |
| Fine-tuning method | QLoRA (4-bit base + LoRA adapters) | Fits on a single 24–48GB GPU, fast iteration |
| Fidelity scorer | Cross-encoder NLI model (e.g., a `roberta-large-mnli`-class model) + BERTScore | Cheap, no API dependency, good at catching meaning drift |
| Detector proxies | Small classifiers you train yourself (see §5) on AI-vs-human text, mimicking perplexity/burstiness signals | Needed for local iteration without burning API calls against real detector services during training |
| Burstiness engine | Rule-based + statistical (sentence-length sampler conditioned on human corpus distribution) | Deterministic, fast, doesn't need a model call |

Do **not** start with RLHF against live third-party detector APIs — it's slow, rate-limited, expensive, and gives competitors a paper trail of your training queries. Build local proxy classifiers first; only spot-check against real detector APIs at eval time (§7).

---

## 3. Dataset plan

### 3.1 Paired corpus construction
- Generate AI text across many prompts/genres/registers using 3+ different source LLMs (avoid single-model fingerprint bias).
- For each AI text, obtain a **true human rewrite** of the same content — ideally commissioned from human writers (Upwork/Scale AI-style task, or your own writing team), not scraped. This gives real paired (AI, human) data rather than loosely topic-matched pairs.
- Supplement with naturally-paired data where available (e.g., press releases vs. journalist rewrites of the same facts — check licensing).
- Target: 20–50K pairs for a usable LoRA v1; 100K+ for a robust general release. Prioritize register/domain diversity over raw volume.

### 3.2 Detector-proxy training set
- Assemble a large AI-vs-human classification set (public academic AI-detection datasets + your own generations) to train local proxy classifiers that approximate what real detectors key on (perplexity, burstiness, phrase frequency).
- Keep this dataset separate from the paraphraser's training data to avoid leakage between "what fools our proxy" and "what fools the real thing."

### 3.3 Data hygiene
- De-duplicate, filter for length/quality, strip PII, verify licensing on any scraped source text.
- Hold out 10% of paired data, never trained on, for eval only.

---

## 4. Transformation targets (what the model should learn to do)

- Increase sentence-length variance (burstiness) to match human distributions.
- Reduce/replace AI-tell stock phrases ("it's important to note," "in conclusion," "furthermore," "delve into").
- Break overly symmetric paragraph/sentence structure.
- Introduce natural hedges, asides, mild redundancy, idiomatic phrasing appropriate to register.
- Vary sentence openers (avoid repetitive subject-first starts).
- Preserve all factual content and argument structure — this is the hard constraint, not a nice-to-have.

---

## 5. Training pipeline

1. **SFT pass**: fine-tune base model on paired corpus (AI-in → human-out), LoRA rank 16–64, standard causal LM loss on the target side.
2. **Local eval**: score checkpoints on held-out set using (a) proxy detector pass rate, (b) NLI/BERTScore fidelity, (c) burstiness delta vs. human reference distribution.
3. **Reranking layer**: at inference, sample k candidate rewrites (k=4–8), score each with the fidelity filter + proxy detector, keep the best that passes a fidelity floor (e.g., NLI entailment score above threshold). Reject-and-retry if none pass.
4. **(v2) DPO pass**: build preference pairs from reranked candidates (best vs. worst per prompt) and run DPO to sharpen the SFT model's default output toward higher-scoring rewrites without needing sampling+reranking at inference time.
5. **Continuous refresh loop**: every quarter (or when public detectors update), refresh the proxy classifiers on new detector behavior and re-run the DPO pass. Version the model (`v1.0`, `v1.1`, …) so CLI users can pin a version.

---

## 6. CLI design

```
humanize rewrite <input.txt|--stdin> [options]

Options:
  --register <academic|casual|marketing|technical>   Target style (default: casual)
  --strength <light|medium|aggressive>                Evasion vs. fidelity tradeoff
  --model <path|version>                              Model checkpoint to use
  --report                                             Print diagnostics (fidelity score,
                                                        burstiness stats, proxy detector estimate)
  --out <file>                                         Write output to file instead of stdout
  --max-retries <n>                                    Reranking retry budget (default 4)

humanize benchmark                                     Run the internal eval suite (§7)
humanize eval <file>                                    Score a single text against fidelity +
                                                         proxy detectors, no rewrite
```

Implementation notes:
- CLI wraps the inference pipeline (§1) as a library; keep the library decoupled from the CLI so it can later back an API/web product.
- `--strength` maps to reranking thresholds (aggressive = lower fidelity floor, more perturbation; light = high fidelity floor, minimal change).
- `--report` output should be structured (JSON option) so it can be scripted/CI'd.

---

## 7. Benchmark & evaluation harness

Build this **before** heavy training, so you have a fixed, falsifiable target throughout — not the flat "95%" claim, but a number you can actually reproduce and defend.

### 7.1 Fixed test set ("B1")
- A few hundred AI-generated texts spanning domains (essays, marketing copy, technical docs, blog posts), held out from all training data.
- Version this set (`B1.0`, `B1.1`…) — never quietly change it, or your historical scores become incomparable.

### 7.2 Baseline capture
- Run QuillBot, Grammarly, and SmallSEOTools outputs on B1 (manually or via API/UI) to establish baseline numbers on the same three axes as your own model.
- Re-capture baselines periodically too — competitors update their tools as well.

### 7.3 Scoring, per detector, per checkpoint
Score every output (baselines + yours) and record it as a row, not a single blended number:

| checkpoint | detector | detector_version | date | pass_rate | fidelity_score | naturalness_score | burstiness_delta |
|---|---|---|---|---|---|---|---|

- **Detector pass rate**: run against each of GPTZero, Originality.ai, Copyleaks, Turnitin, ZeroGPT *separately* — don't average them into one number until each is individually visible. Note ZeroGPT specifically has a documented high false-positive rate on human text, so treat wins against it as a weak signal, not a headline number.
- **Fidelity**: NLI entailment + BERTScore vs. original AI draft.
- **Naturalness**: small blind human panel, Likert scale.
- **Burstiness delta**: sentence-length variance vs. human reference corpus.
- Spend real detector-API calls here only (eval), not inside the training loop (§5 uses local proxy classifiers for that).

### 7.4 Per-detector calibration presets
A single `--strength` knob isn't enough once you're tracking multiple detectors with different weightings (perplexity-heavy vs. burstiness-heavy vs. phrase-frequency-heavy). Add presets:

```
--target-detector <gptzero|originality|copyleaks|turnitin|zerogpt|balanced>
```

Each preset biases the reranker (§5 step 3) toward the signal that detector weights most, without dropping below the global fidelity floor. `balanced` (default) optimizes the average across all five.

### 7.5 Decay-tracking dashboard
`humanize benchmark` should:
1. Re-run the full B1 scoring table (§7.3) against the *current* pinned model checkpoint.
2. Diff against the last recorded run per detector, and flag any detector where pass rate has dropped below the last-known value by more than a set threshold (e.g., 10 points) — this is your early warning that a detector updated and your model needs a refresh, rather than finding out from user complaints.
3. Output both a human-readable table and a `--json` machine-readable report for CI.

Run this: every training iteration during development, and on a fixed cadence post-launch (start monthly; stretch to quarterly once decay rate is well understood). Never let the "95%" number on your website be older than the last benchmark run — timestamp it.

### 7.6 Result-integrity safeguards (no fabricated or stale numbers)

The benchmark is only useful if it can't be gamed — including accidentally, by rounding up, cherry-picking, or letting a number go stale. Build these in as hard rules, not conventions:

- **No hand-entered scores.** Every number in the scoring table (§7.3) is written by the scoring code directly, never typed in manually. If a detector API call fails, the row records `error`, not a guess or an old value carried forward.
- **Every reported score carries its provenance.** Checkpoint hash, detector name + version, benchmark set version, and timestamp travel with the number wherever it's displayed — CLI `--report` output, the dashboard, and any external marketing copy. A number with no provenance attached should not leave the system.
- **Full-set reporting, not cherry-picked subsets.** `humanize benchmark` always scores and reports against the entire fixed set B1, and always reports all five detectors, not just the ones you're currently winning on. If you want a "best case" number for marketing, it must be labeled as such and shown alongside the full-set average, never in place of it.
- **Failure and rejection cases are counted, not dropped.** When the fidelity filter (§5 step 3) rejects all k candidates for a given input (no rewrite cleared the floor), that counts as a failure in the pass-rate denominator — it does not get silently excluded from the average.
- **Freshness enforcement.** Any dashboard, CLI output, or external claim displaying a pass-rate number must show the benchmark date. `humanize benchmark` should hard-fail (non-zero exit) if asked to publish/export a report older than the refresh cadence in §7.5, so a stale number can't ship by accident.
- **Independent re-verification.** Periodically (e.g., every other benchmark cycle), have someone other than the person who tuned the model run the benchmark from a clean checkout, to catch pipeline bugs that happen to inflate scores (e.g., a reranker accidentally leaking detector-set examples into training, or a fidelity check that's silently no-op'ing).
- **No overriding real API results with the local proxy.** The proxy detector classifiers (§5) are for fast training-loop iteration only; they must never substitute for a real detector-API score in any reported or published number. Reports should visibly distinguish `proxy_estimate` fields from `verified_api` fields, and only `verified_api` numbers are eligible for external claims.

---

## 8. Infrastructure & cost estimate

| Item | Estimate |
|---|---|
| LoRA fine-tuning (7–8B model, QLoRA) | Single A100/H100 (40–80GB), a few hours to ~1 day per run — a few hundred dollars on rented cloud compute |
| Proxy classifier training | CPU/small-GPU job, cheap, hours |
| Human data collection (paired corpus) | Largest cost driver — budget per-pair rate × 20–50K pairs; consider starting smaller (5–10K) for v0.1 and scaling |
| Detector API eval spend | Keep to eval-only usage (§7), not training-loop usage, to control cost |
| Inference serving | Single GPU sufficient for CLI/local use; scale out only if you productize as a hosted service |

---

## 9. Milestones

1. **v0.1**: Rule-based post-processor only (burstiness + phrase-swap), no fine-tuned model — ships fast, gives you a CLI skeleton and eval harness to validate against baselines.
2. **v0.5**: SFT LoRA model on ~5–10K paired examples + fidelity reranker. Run full benchmark (§7) vs. QuillBot/Grammarly/SmallSEOTools.
3. **v1.0**: Scaled dataset (20K+ pairs), reranking pipeline tuned, `--register` and `--strength` options live, benchmark shows clear win on all three axes (evasion, fidelity, naturalness) vs. named competitors.
4. **v1.1+**: DPO pass (Track B), quarterly refresh cadence established.

---

## 10. Open risks to revisit

- **License terms** on base model choice — confirm commercial-use terms before shipping a paid product.
- **Detector drift** — plan the refresh cadence into the roadmap now, not as a surprise later.
- **Fidelity/evasion tradeoff** — `--strength aggressive` will always risk meaning drift; keep the fidelity floor enforced in code, not just as a UI suggestion, so the CLI can't silently produce garbled output.
- **Use-case clarity** — if downstream users are specifically evading academic-integrity or content-provenance checks, that's a distinct liability/ethics surface from "make my AI-assisted draft read naturally." Worth deciding explicitly which product this is, since it affects ToS, marketing copy, and which features (if any) you gate.
