# humanize-cli

A CLI for rewriting AI-generated text into more natural, human-styled prose — built to be measurably better than shallow tools like QuillBot, Grammarly's rewriter, and SmallSEOTools, and evaluated honestly against real AI-detection services rather than marketing claims.

> **Status: planning stage.** This repo currently contains the design doc (`plan.md`); the pipeline, models, and CLI described below are not yet implemented. Contributions welcome — see [Roadmap](#roadmap).

---

## Why this exists

Most existing "humanizer" tools do shallow synonym substitution and light sentence reordering. That's easy to beat with a model that operates at the paragraph level, models sentence-length variance ("burstiness"), and enforces a meaning-preservation floor so rewrites don't drift from the original content just to dodge a detector.

This project is built around three commitments:

1. **Measure, don't claim.** No fixed "95% human" number ships anywhere — not in the CLI, not in this README, not in marketing copy — without a detector name, detector version, and a benchmark date attached. See [`plan.md` §0](./plan.md#0-scope--success-definition) and [§7.6](./plan.md#76-result-integrity-safeguards-no-fabricated-or-stale-numbers).
2. **Fidelity is a hard constraint, not a nice-to-have.** Every rewrite is checked against a semantic-similarity floor before it's returned. Meaning-preserving rewrites that only partially evade detection beat garbled ones that fully evade it.
3. **Detectors drift, so the project does too.** AI-detection tools retrain against exactly this kind of software. This project ships with a benchmark harness that re-verifies performance on a fixed cadence and flags decay automatically, instead of letting a stale claim sit on a page.

---

## What it does (once built)

```
humanize rewrite input.txt --register casual --strength medium --report
```

- Rewrites AI-generated text at the paragraph level (not sentence-by-sentence)
- Preserves meaning within a configurable fidelity floor
- Targets natural sentence-length variance and reduces common AI stock phrasing
- Optionally reports diagnostics: fidelity score, burstiness stats, per-detector pass/fail estimate, model checkpoint version, and benchmark date

Full CLI spec: [`plan.md` §6](./plan.md#6-cli-design).

---

## How it's evaluated

Every claim this project makes about "better than X" is backed by a versioned, reproducible benchmark:

- A fixed, held-out test set (`B1`) of AI-generated text across domains
- Outputs from this tool and from named competitor tools (QuillBot, Grammarly, SmallSEOTools) scored on the *same* set
- Scored on three axes, always reported together — a high evasion score achieved by damaging the text is not counted as a win:
  1. **Detector pass rate** — per detector (GPTZero, Originality.ai, Copyleaks, Turnitin, ZeroGPT), reported individually, not blended into one number
  2. **Semantic fidelity** — NLI entailment / BERTScore vs. the original
  3. **Human-rated naturalness** — blind panel scoring
- Results carry provenance (checkpoint hash, detector version, benchmark date) wherever they're shown, and a report older than the refresh cadence fails to publish rather than shipping stale

Run `humanize benchmark` to reproduce these numbers yourself once the harness lands. Full details: [`plan.md` §7](./plan.md#7-benchmark--evaluation-harness).

### A note on ZeroGPT
ZeroGPT has a documented high false-positive rate on human-written text. This project reports results against it for completeness but doesn't treat wins against it as a meaningful signal on their own.

---

## Non-goals

- **Permanent, universal detector evasion.** Detectors are adversarially retrained against tools like this one. This project commits to a refresh cadence, not a one-time fix — see [`plan.md` §9](./plan.md#9-milestones).
- **Evasion at the cost of meaning.** Aggressive settings will always trade off against fidelity; the fidelity floor is enforced in code, not left as a UI suggestion.

---

## Project layout (planned)

```
.
├── plan.md              # Full technical design doc — architecture, dataset, training, benchmark
├── humanize/             # CLI + pipeline library (not yet implemented)
├── benchmarks/           # Fixed benchmark set (B1) + scoring harness
└── training/             # Fine-tuning scripts, dataset prep
```

---

## Roadmap

See [`plan.md` §9](./plan.md#9-milestones) for the full milestone breakdown:

- [ ] **v0.1** — Rule-based post-processor (no fine-tuned model), CLI skeleton, benchmark harness
- [ ] **v0.5** — First fine-tuned model (LoRA) + fidelity reranker, benchmarked against named competitors
- [ ] **v1.0** — Scaled training set, full `--register` / `--strength` options, clear benchmark win on all three axes
- [ ] **v1.1+** — Preference-tuning pass, quarterly refresh cadence in production

---

## Read the full plan

The complete architecture, model selection, dataset construction plan, training pipeline, CLI spec, benchmark design, cost estimates, and open risks live in [`plan.md`](./plan.md).

---

## A word on use case

This tool is meant to help make AI-assisted writing read more naturally — varied, idiomatic, less formulaic. If your use case is specifically evading academic-integrity or content-provenance checks, know that up front: that's a different liability and ethics surface than "improve my drafting," and it affects what this project will and won't support. See [`plan.md` §10](./plan.md#10-open-risks-to-revisit).
