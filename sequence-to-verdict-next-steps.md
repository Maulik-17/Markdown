# Sequence-to-Verdict — What to Do Next

*Decision notes, September 2026. Source: project report "Sequence to Verdict" (Ch. 1, 9–13).*

## The verdict, in short

The project tested a narrow question: does reformatting a validated sequence descriptor (conjoint triad + PseAAC) as an image and running it through a pretrained ImageNet backbone produce a *usable* PPI predictor? On curated, single-species data (H. pylori, S. cerevisiae), **yes** — test accuracy 0.54–0.69, AUC up to 0.80, well above chance. On larger, more diverse STRING-derived data (SHS27k/SHS148k), **the approach breaks down** — test AUC of 0.50–0.59 is a genuine generalization failure, not just a lower score. That split is what everything below turns on.

## Results at a glance

| Comparison | Gap |
|---|---|
| vs. EResCNN, same-species (H. pylori / S. cerevisiae) | trails by 18.9–41.1 pts accuracy |
| vs. DL-PPI, STRING data (SHS27k / SHS148k) | trails by 36.5–44.6 pts Micro-F1; AUC ≈ 0.50–0.59 |
| Cross-species recall (4 unseen species) | 0.51–0.62 for most models — S. cerevisiae-trained models generalize better than H. pylori-trained ones |

## Why it underperforms

Three causes are traceable, not mysterious: (1) neither training stage saves the best-validation checkpoint — every reported number comes from the last epoch, sometimes past the point of overfitting; (2) fine-tuning the backbone mostly hurt, because the descriptor images have none of the spatial structure (edges, textures) ImageNet filters expect, so adapting them has little to work with; (3) no descriptor choice (conjoint triad, PseAAC, or combined) wins consistently — it's dataset-dependent, so "combine everything" isn't a free win.

## The decision

The narrow research question is already answered honestly, so the real choice isn't "keep tuning this recipe" — it's **which of two directions to invest in next**:

**A. Consolidate and report what's already true.** The approach works as a usable (not competitive) predictor on curated data. This is a legitimate, honestly-measured result on its own terms — closing it out costs little and gives you a clean, defensible writeup.

**B. Chase the generalization gap on diverse data.** The AUC ≈ 0.5 result on SHS27k/SHS148k is the one finding that actually threatens the project's premise. Two explanations are on the table and unresolved: STRING data is more taxonomically diverse, so protein-disjoint splits produce harder train/test separation by construction; or random-sampled negatives in STRING are simply easier than the biologically-plausible hard negatives elsewhere. Until this is disentangled, you don't know if the architecture has a hard ceiling here or if it's fixable.

Recommendation: do the cheap, high-confidence fixes first (below) regardless of which direction you lean, because they're needed either way — a "final" report with unselected checkpoints and unseeded single runs isn't trustworthy yet. Then treat the generalization investigation as the real fork: it decides whether this architecture family is worth more time, or whether the honest conclusion is a scoped negative result.

## Next actions, ranked by effort vs. payoff

**Do first — cheap, directly fixes acknowledged flaws:**
- Add best-checkpoint selection (track/restore lowest validation loss instead of last epoch) across Stages H–J. The infrastructure already exists; this alone should modestly tighten every headline number and make the frozen-vs-fine-tuned comparison fair for the first time.
- Fix the random seed and rerun the four core dataset/backbone combos 3–5× each to turn single-run numbers into confidence intervals. This tells you which findings (e.g. "fine-tuning hurts") are real versus noise.

**Do second — moderate effort, decides the real fork:**
- Run a dedicated experiment isolating STRING's taxonomic diversity from its negative-sampling scheme (e.g. evaluate on a diversity-matched subset of SHS27k, or swap in hard negatives) to find out why AUC collapses to ~0.5 there. This is the single most informative next experiment — it tells you whether direction B is worth pursuing at all.

**Lower priority — polish, unlikely to change the headline conclusion:**
- Progressive/gentler backbone unfreezing recipes (marginal gains at best, per Ch. 9–10 evidence).
- A systematic descriptor-pairing sweep beyond conjoint-triad + PseAAC (e.g. with AC or MMI) — worth doing only after the generalization question is settled, since it won't fix the STRING-data failure.

**Don't bother with:** more fine-tuning variants aimed at closing the same-species gap to EResCNN/DL-PPI specifically — the report's own evidence (descriptor images lack the spatial structure the backbone expects) suggests this is an architecture-level ceiling, not a tuning problem.
