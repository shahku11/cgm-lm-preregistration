# Preregistration

**Status: declared for runs that have NOT yet been performed.** Nothing in this document
retroactively covers a result already in `docs/results/`. See §0.

**Version:** 1 · **Created:** 2026-08-28

Bind a result to this document with `python -m cgm_lm.preregistration`, which records this
file's SHA-256, the commit that introduced it, whether the working copy still matches that
commit, and whether an OpenTimestamps receipt exists. Every field is computed; none is
assertable by the caller.

---

## 0. What this document cannot do

The repository's history is 23 commits over four days (2026-08-24 → 27). **No threshold in any
result published before this document has version-control evidence of predating the result it
judges.** Two specifics, both stated against interest:

- `construct_validity_summary.py` recorded `"declared_before_analysis": True` as a hard-coded
  literal, in the same commit (`76eb259`) as the result JSON it certified. That literal is now
  replaced by a computed record (`docs/ERRATA.md` E4).
- One threshold *was* changed after a near-miss, disclosed at `cgm_lm/omop.py`: the duplication
  audit used raw concordance, which put genuinely-independent retinopathy at 0.891 against a
  0.95 gate, so the statistic became Cohen's κ and the threshold moved to 0.9. The change is
  correct and the disclosure is exemplary. It is still, definitionally, post-hoc.

A git commit date comes from `GIT_COMMITTER_DATE` and can be backdated; a GPG signature covers
the backdated value. So signing evidences *authorship*, not *priority*. Priority requires an
external anchor:

```bash
ots stamp docs/PREREGISTRATION.md
```

Commit the resulting `.ots` receipt. Anyone can then verify the file's SHA-256 was anchored
into Bitcoin at a given time without trusting this repository. **Do this before submitting any
run whose gates are declared below.** Push to a public remote as a second, independent
timestamp, and register on OSF or AsPredicted before anything enters a manuscript.

---

## 1. Transfer grid rerun (`transfer_v2`) — the registered representation gate

**Motivation.** The `transfer_v1` gate declared itself passed when *any* of 48 correlated
intervals excluded zero. Corrected, 1 of 48 survives Holm
(`docs/results/transfer_v1_multiplicity.json`). Pooling across the three disjoint-subject sites
gives +0.0051 [+0.0031, +0.0070] — but that cell was chosen *after* observing it passed, so it
is exploratory. This section makes it confirmatory.

**Primary endpoint, declared now.** One test.

| field | value |
|---|---|
| encoder | `state_event` |
| arm | `pretrained` |
| target | `hyper` (prolonged hyperglycemia in hours [18,24)) |
| comparator | handcrafted summary features (`transfer_eval.handcrafted_features`) |
| statistic | per-site Δ AUROC, pooled by fixed-effect inverse-variance across UAB, UCSD, UW |
| test | two-sided, α = 0.05, no multiplicity adjustment (one test) |
| direction requirement | Δ > 0 |
| **effect size to beat** | **Δ ≥ 0.0031** (the lower bound of the exploratory estimate) |
| seeds | SSL pretraining seeds 17, 42, 73; all three carried into reporting |
| aggregation across seeds | median of the three per-seed pooled estimates; all three reported |

**Pooling rule.** Across sites only. Cells within a site share test rows and share the
comparator, so pooling them counts one piece of evidence several times. An arm is pooled **only
when every site is present** — a pool over the subset of sites that passed would condition the
estimate on passing.

**Secondary family.** The 8 (encoder × arm) pooled estimates × 2 targets = 16 tests, Holm at
FWER 0.05, via `cgm_lm.multiplicity.holm` over exact bootstrap p-values.

**Exploratory.** All 48 per-cell tests, reported in full with raw and Holm-adjusted p. Never a
gate. Labelled `exploratory` in the schema.

**Stopping rule.** The grid is enumerated, not sampled. No cell may be added or dropped after
any result is seen. If a cell fails to complete, `--expected-cells` fails the aggregation loudly
and the incomplete grid is reported as incomplete.

**Analysis code path.** `cgm_lm.transfer_eval` → `cgm_lm.transfer_summary.aggregate` →
`cgm_lm.transfer_summary.recorrect_summary`. Bootstrap B = 2000.

**B-escalation rule.** Any test that is the prespecified primary, or whose `p_normal` falls
within a factor of 10 of its Holm boundary, is recomputed at B = 20,000. Declared here so the
escalation is not itself a selection.

**Prediction, recorded so it can be wrong.** The pooled primary will land between +0.003 and
+0.008 and will clear the gate; `hypo` will remain null. If `hyper` also comes out null under
three pretraining seeds, the correct conclusion is that `transfer_v1`'s single seed was
favourable, and that must be reported as the finding.

---

## 2. Decoder conditioning ablation — the central open question

**Motivation.** `CGM_LM_PIPELINE_UPGRADE_2026-08-20.md` records full-minus-zero-prefix
−0.0033, CI [−0.0065, −0.0007]: the fact-conditioned decoder is *worse* than a zeroed encoder.
The training-time arms have never been run.

**Arms.** `signal_and_facts`, `signal_only`, `facts_only` (`carina_conditioning_ablation.sbatch`).

**Two-stage design, declared before submission.**

- **Stage A**: 3 arms × seed 42, `DEC_EPOCHS=4`, all subjects, 1,536 generation samples.
- **Decision rule**: proceed to Stage B only if `signal_only`'s claim-verification pass rate has
  a subject-clustered lower bound above the deterministic-rule comparator.
- **Stage B**: seeds 314 and 2718 for all three arms.

Stage A is not a pilot whose result is discarded. If the decision rule fails, **Stage A is the
result** and is reported as such.

**Gate, inherited unchanged** from `cgm_lm/evaluate.py:signal_ablation_summary`, which has a
dated documentary antecedent in `CGM_LM_PIPELINE_UPGRADE_2026-08-20.md` ("at least two
percentage points") — the strongest pre-registration evidence in the repository:

- signal-dependence effect ≥ **0.02**, and
- paired subject-clustered lower bound > 0.

**The internal positive control.** In the `signal_only` arm `data.py` sets `decoder_prompt=""`,
so generation runs from the query prefix alone. `evaluate.py`'s zero-signal arm therefore has no
prompt *and* no prefix and **must** collapse. If it does not, the measurement apparatus is
broken and no arm's result may be reported.

**Prediction.** `facts_only` will match `signal_and_facts` within 0.02, confirming the decoder
is a fact-conditioned paraphraser. If so, the contribution claim becomes verification +
composition + safety governance, stated permanently, and Phase 5's three
"beyond-templates" capabilities become the deliverable.

---

## 3. ReCoCa objective ablation

**Arms.** `recoca_cgm`, `coca`, `clip`, `caption_only`, plus two added here:

- `no_numeric_grounding` (`w_numeric=0`, `w_counterfactual=0`, otherwise `recoca_cgm`) — the
  numeric and counterfactual losses are the repository's own contribution over SleepLM and no
  existing ablation isolates them.
- `jepa_latent` (`w_recon=0` plus the `cgm_lm/jepa.py` objective) — SleepLM argues *for* raw
  reconstruction as a regulariser; GluoFM and CGM-JEPA argue *against* it and for masked-latent
  prediction. The two literatures disagree and one corpus can settle it.

Array becomes `0-17` (6 variants × 3 seeds).

**Loss scale, fixed before submission and recorded in `objective_manifest`.**
`w_caption` is set so that `w_caption × mean(caption_CE) ≈ w_contrastive × mean(contrastive)` at
the end of representation warm-up, measured from the 1-epoch smoke run's per-term means.

`w_counterfactual` is **not** raised to compensate: `numeric_counterfactual_loss` is a hinge at
`margin=0.1`, so its value is bounded above by 0.1 and scaling a saturated hinge changes
nothing. The lever is `--counterfactual-margin`.

**Guardrails, not outcomes.** `cf_acc` and `align_acc` below the `custom`-variant baseline are
reported as *objective interference*, not as a representation finding. Otherwise "caption hurts
the representation" and "caption starved the counterfactual optimiser" are indistinguishable.

**Naming.** Arms are reported by what they contain (`recoca_cgm + cgm_aux`, …). `PRETRAINING_VARIANTS`
overrides only 3 of ~6.2 total loss weight, so the arm labelled `caption_only` retains 3.0 of
numeric/counterfactual weight and must not be published as "caption only".

---

## 4. Event-proposal rerun (`event_proposal_v4`)

**Motivation.** `event_proposal.py` trains on 10 of 22 scenarios but builds its validation set
over all 22, and selects the operating threshold and checkpoint with a utility containing
`0.1 × val_stress_f1` — tuning on the 12 stress families the holdout exists to keep unseen, and
stress F1 is the promotion criterion.

**Fix.** Threshold and checkpoint selection see `TRAIN_SCENARIOS` only; stress metrics are
recorded under a `reported_not_used` key; an assertion in `run()` enforces the subset relation.

**Array.** All 7 arms × 3 seeds = 21 tasks. **Not** a subset. Re-running only the arms expected
to win is itself selection: a leak-free threshold could promote `cgmformer`/default or `raw`,
and choosing not to look is how that stays invisible.

**Promotion gate, now baseline-relative** rather than the absolute constants 0.75/0.60, which
sat *below* the deterministic baseline they replace (0.914/0.857) so an arm could pass while
being worse than the rule:

| requirement | threshold |
|---|---|
| seeds | ≥ 3 |
| proposal F1 | ≥ the production detector's F1 on the same benchmark |
| stress F1 | ≥ the production detector's stress F1 |
| stress F1 vs baseline | paired case-level bootstrap lower bound > 0 |
| negative abstention | ≥ 0.85 |
| signal dependence | 3/3 seeds |
| quality gate | 3/3 seeds |

**Expected consequence, computed in advance from the v3 per-seed numbers:** the baseline-relative
thresholds alone move `raw` from 1/3 to 0/3 and leave the three promotions unchanged. The
*threshold-leak* fix changes every test number, so no v3 row survives comparison and none will be
printed in the same table as a v4 row.

---

## 5. Generator de-fingerprinting (`noncircular_benchmark_v3`)

Three generators. Fitting and threshold-locking happen on **A only**.

- **A** — `noncircular_benchmark_v2`, imported unchanged. Existing artifacts remain valid as a
  named comparator.
- **B** — independent synthetic: AR(1)/OU background around a per-case mean drawn from the
  empirical AI-READI distribution of subject mean glucose, continuous circadian amplitude and
  phase, heteroscedastic noise, `onset ~ Uniform(window)`, and amplitude support wider than and
  only partly overlapping A's finite `replicate % k` ladder.
- **C** — real-trace overlay: a synthetic excursion injected into a real AI-READI day whose
  target window is verified event-free by a **published-threshold predicate written for this
  purpose**. That predicate must not call `production_detector`; using the production detector
  to certify a background clean reintroduces the circularity the module exists to avoid.

**Registered gate on the generalisation gap:**

- stress F1 on C ≥ the deterministic baseline's stress F1 on C, **and**
- F1(A) − F1(C) ≤ **0.10**.

**Reported alongside:** the candidate-day rejection rate for arm C. A heavily filtered "real"
background is not representative and the filter rate is what tells a reader how much.

**Matching.** Primary criterion becomes `IoU ≥ 0.30 AND onset error ≤ 6 slots`. The v2 rule
(`IoU ≥ 0.10 OR onset ≤ 6`) counted a prediction entirely disjoint from truth as a true
positive; it is retained as `matching_v1_loose` so old numbers stay reconstructible. An IoU sweep
at {0.10, 0.30, 0.50} is published so the sensitivity is visible rather than a single threshold.

---

## 6. Aim 2 / CGM1 physiologic calibration

**The power statement, declared before the data is opened.** The participant-level n is **55**,
not the number of curves — curves are replicates within participants and carry no independent
information about a participant-level phenotype. At n = 55, α = 0.05 two-sided, 80% power detects
**r ≈ 0.37**; with Bonferroni over 10 feature×phenotype pairs, **r ≈ 0.45**. Aim 2's calibration
is powered only for large effects.

**Therefore: exactly two primary pairs, named now.**

1. iAUC above pre-meal baseline vs insulin sensitivity
2. time to peak vs a β-cell function index

Everything else is exploratory and carries **no Tier-3 consequence**.

**Replicate-stability gate, before any language.** A feature must reach **ICC(2,1) ≥ 0.5** across
repeated administrations of the identical standardized meal. A feature whose within-person SD is
comparable to its between-person SD cannot support Tier-3 wording however well it correlates with
a phenotype.

**Model.** `feature ~ meal_type (fixed) + (1 | participant)` for the descriptive layer, then
`phenotype ~ feature + prespecified covariates`, leave-participant-out.

**Tier-3 registry.** Every permitted Tier-3 sentence maps to a licensing calibration entry with
`replication_status`. The audit's default inverts: a mechanism-phrase match is a violation
**unless** `evidence_tier == 3` **and** a registry entry matches **and** that entry is not
`unreplicated`.

---

## 7. What is *not* declared here

These require data or approvals not in hand, and declaring their analysis now would be
declaring an analysis of data whose schema is unknown:

- Tier-2 meal/insulin endpoints (gated on the Garmin download and the public-dataset adapters)
- external-cohort scoring (gated on a locked external cohort; the freeze-then-score-once
  sequence in `docs/SLEEPLM_ALIGNMENT_AND_RUN_PLAN.md` steps 5–7 governs it)
- both human studies (gated on IRB; `docs/HUMAN_EVALUATION_PROTOCOL.md` locks the instruments
  and still lacks a sample size — that power calculation is a prerequisite, not a formality)

---

## Amendments

None. Amendments append a dated section here with a stated reason and are never edited in
place, following the convention `docs/ERRATA.md` sets. An amendment made after seeing a result
must say so.
