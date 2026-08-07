# Deep-Dive: CAFA 5 Top-Method Architectures and the Partial-Knowledge Evaluation Frontier

## TL;DR
- **The CAFA 5 top methods are, almost universally, ensembles of frozen protein-language-model (pLM) embeddings (ProtT5, ESM-2, Ankh) feeding lightweight classifier heads, fused with classical BLAST/DIAMOND homology transfer and — in the very top solutions — literature/text mining. Genuine architectural novelty concentrated in (a) DAG-aware loss functions and post-processing, and (b) using non-experimental (electronic) GO annotations and text as *features* rather than targets. Structure (AlphaFold2) was used by only a few teams and was not the dominant driver.**
- **The paper's headline open problem is the newly introduced "partial-knowledge" (PK) setting — predicting *new* annotations for proteins that already have some experimental annotations in the same GO aspect. PK accounts for 69.7% of accumulated annotations, and the best model's performance drops 55.6% (BPO), 47.8% (CCO), 59.7% (MFO) versus the no-knowledge setting. Essentially no CAFA 5 team conditioned on already-known annotations — this is the clearest low-compute, math-heavy opening for a new entrant.**
- **Everything needed to work on this is public: the Kaggle data, the ProtT5/ESM-2/Ankh embeddings, the CAFA-evaluator-PK fork, the information-accretion code, and a Zenodo release of the CAFA 5 evaluation data. CAFA 6 is live on Kaggle — hosted by Iowa State with Northeastern, UniProt and ISCB, $50,000 prize pool, entry deadline 26 January 2026 — so a method targeting PK can be validated against a fresh blind benchmark.**

## Key Findings

1. **Sequence-only pLM embeddings + simple heads dominated.** The winning recipe across the leaderboard was: take frozen embeddings from one or more pLMs (ProtT5-XL-U50, ESM-2 up to 3B, Ankh, ProtBERT), optionally concatenate taxonomy one-hot vectors and text features, and train a shallow head (MLP, logistic regression, ridge, or gradient-boosted trees). No team needed to fine-tune the pLM.
2. **The difference between "very good" and "1st place" was auxiliary signal, not the backbone.** GOCurator (1st) and PROTGOAT (4th) added literature text mining; the 3rd-place solution engineered non-experimental (IEA) annotations as features; InterLabelGO+ (6th) and StarFunc (5th) added novel losses and homology/structure fusion respectively.
3. **DAG structure was exploited at three points:** (i) in the *loss* (ZLPR ranking loss, IA-weighted F1, soft-F1); (ii) in *model construction* (conditional-probability modeling, GCN stacking over the GO graph); and (iii) in *post-processing* (parent≥child propagation, threshold calibration).
4. **The gains over CAFA 4 are attributed primarily to pLM embeddings and, secondarily, to the influx of ML practitioners from Kaggle (22.2× more teams), with growth of the annotation database a real but partial contributor and AlphaFold/structure explicitly *not* claimed as a CAFA 5 driver.**
5. **The PK setting is a wide-open research frontier.** The organizers derived a conditional-information-accretion framework for it and flag explicitly that model developers may build models that exploit already-known annotations within a GO aspect to predict remaining functions. No top CAFA 5 method did this.
6. **A cluster of adaptable prior/parallel work exists** — positive-unlabeled learning (PU-GO), weak-label/incomplete-label learning (ProWL), matrix-completion-style annotation replenishment, and the structured-output "open world" evaluation theory (Jiang et al. 2014) — but none has been evaluated on the CAFA 5 PK benchmark specifically.

## Details

### 1. The evaluation metric (needed to read everything below)

CAFA scores are **information-accretion (IA)–weighted, protein-centric F-measures**, maximized over a decision threshold τ ∈ [0,1] (`Fmax`, also written `wFmax`). Predictions are DAG-propagated (a term's score is raised to at least its children's) before scoring.

Information accretion (Clark & Radivojac, Bioinformatics 2013, 29(13):i53–i61, doi:10.1093/bioinformatics/btt228) assigns each term v a weight equal to the "surprise" of observing it given its parents are already present:

  ia(v) = −log₂ Pr(v | Pa(v)),

where Pa(v) are the parents of v in the ontology, modeled as a Bayesian network over the DAG. The total information of an annotation set T is i(T) = Σ_{v∈T} ia(v). Two IA-weighted quantities act as information-theoretic analogs of recall and precision:
- **remaining uncertainty** ru(τ) = Σ IA of true terms the model *missed* (false negatives), and
- **misinformation** mi(τ) = Σ IA of predicted terms that are *wrong* (false positives).

The scalar summary is the **semantic distance** Smin = min_τ √(ru(τ)² + mi(τ)²) (lower is better). CAFA 5 reports micro- and macro-averaged variants and uses B = 5,000 bootstrap samples for confidence intervals (earlier rounds used 10,000).

Biologically: GO has three aspects — **Molecular Function (MFO)** (what the protein does biochemically, e.g. "kinase activity"), **Biological Process (BPO)** (the larger program, e.g. "cell cycle"), and **Cellular Component (CCO)** (where it acts). BPO is the largest and hardest (>21,000 terms in the CAFA 5 training set) with the lowest Fmax; MFO is smallest and easiest.

### 2. Top CAFA 5 method architectures

CAFA 5 ran on Kaggle April–August 2023. The assessment (Ramola/De Paolis Kaluza et al., "Advances in Protein Function Prediction from the Fifth CAFA Challenge," bioRxiv 2026, doi:10.64898/2026.04.27.716980) reports: "During the four-month challenge, 1,625 teams from 78 countries submitted at least one prediction, resulting in a total of 2,850 models … a 22.2-fold increase in participating teams compared to CAFA 4 and a 17.6-fold increase in submitted models." The GO structure was frozen at version **2023-01-01**; training annotations came from **UniProt release 2023_03 / GOA**, with 142,246 experimentally-annotated training proteins and a ~141,865-sequence "super-test."

The table summarizes what the top solutions did. Ranks are from the Kaggle public/final leaderboard as reported in the solution write-ups; note the assessment paper re-evaluates with IA-weighted Fmax/Smin on its own benchmark, so its internal ranking may differ slightly.

| Rank / Team | pLM embeddings | Classifier head / core model | Structure? | PPI / text / taxonomy / domains | Loss & DAG handling | Post-processing | Notable trick |
|---|---|---|---|---|---|---|---|
| **1 — GOCurator** (Zhu Lab, Fudan; based on NetGO 3.0) | ESM-2 (LR-ESM), plus sequence/InterPro/BLAST-KNN components | Diverse ensemble (learning-to-rank over component methods) | Yes (structure-based component) | **Text mining via GORetriever** (2-stage deep information-retrieval matching GO definitions to protein literature); InterPro; taxonomy | Component-wise; ensemble weighting | Standard propagation | Literature-driven GO retrieval as a first-class component; per Yan et al. (Bioinformatics 2024, 40(Suppl_2):ii53, doi:10.1093/bioinformatics/btae401), "Incorporating GORetriever into NetGO3.0 successfully improves the wFmax performance (from 0.587 to 0.604)" |
| **2 — U900 / ProtBoost** (Chervov, Vakhrushev, Fironov) | ProtT5 (primary) + ESM-2, concatenated | **Py-Boost** (GPU multi-target gradient boosting, ~4,500 targets/run) + MLP + logistic regression | No | Taxonomy one-hot; QuickGO/UniProt **electronic (IEA) annotations as stacking features** | BCE; **conditional-probability modeling (CondProbMod)** | Two-direction "soft" propagation (weighted 0.7/0.3 parent/child), averaged | **GCN stacking** over the GO DAG combining base models + electronic annotations; final CAFA-metric 0.58240 (per Chervov et al., arXiv:2412.04529: "with a score of 0.58240. Our score was significantly higher than those of the third and fourth place models, which scored 0.57276 and 0.56245, respectively") |
| **3 — Tito** | ProtT5 / ESM-2 | NN | No | **Non-experimental GO labels preprocessed via CNN and used as features** (not targets); taxonomy | BCE | Propagation | Treated train/val/test as a temporal (time-series) split to match the prospective test |
| **4 — PROTGOAT** (Chua et al., bioRxiv 2024.04.01.587572) | **Five PLMs**: ESM-2-3B, ESM-2-650M, ProtT5, ProtBERT, Ankh | Simple dense NN per aspect | No | **TF-IDF of associated literature abstracts** (final text embedding dim 10,279); taxonomy | BCE | IA/frequency-based term selection (1,500 BPO, 800 CCO, 800 MFO) | Ensemble of diverse PLMs + literature is the whole thesis |
| **5 — StarFunc / Zhang & Freddolino** (bioRxiv 2024.05.15.594113) | ESM-2 (via SPROF-GO in the CAFA5 build; InterLabelGO in the final version) | Template + deep-learning fusion | **Yes — AlphaFold2 structural alignment (Foldseek/AFDB templates)** | **PPI-based component; Pfam family component**; BioLiP templates | Component fusion | Consensus weighting | Ablation: removing InterLabelGO and the PPI component caused the biggest performance drops |
| **6 — InterLabelGO+** (Liu, Zhang, Freddolino, Bioinformatics 2024, 40(11):btae655) | **ESM-2 (last 3 hidden layers, mean-pooled, dim 2560 each)** | 3 parallel MLPs → concat → MLP head; fused with **AlignmentKNN** (DIAMOND homology) via dynamic sequence-identity weighting | No | Homology only | **Novel composite loss:** IA-weighted protein-centric F1 × GO-centric F1 × **ZLPR** (zero-bounded log-sum-exp pairwise rank) loss | Enforce parent prob ≥ max child prob | Loss ablation is the paper's core contribution; ESM-2 gave +40.9% (BPO), +12.8% (MFO), +3.7% (CCO) wFmax over a no-pLM variant |
| **~14 — Zoltan** | Combined embeddings | NN | No | Taxonomy | — | — | Downsizing training set to most-annotated proteins; selecting top-frequency targets per sub-ontology |
| **~19 — Hoehndorf Lab** | Ontology-aware | Geometric ("ball") ontology embeddings; "approximate semantic entailment" (Nat Mach Intell 2024, 6:220) | No | Ontology axioms | Geometric constraints | — | **Zero-shot prediction** of terms absent from training via translating ontology axioms into geometric restrictions |

**The single most important cross-cutting observation** (made explicitly in the ProtBoost paper): the primary component of essentially all top models was pretrained pLM embeddings, and the "tricks that data-scientists brought that domain bioinformaticians had not been using" were (i) large-scale ensembling/stacking (GCN stacking, blending 5 PLMs), (ii) treating electronic/IEA annotations and literature text as *features* rather than as targets or noise, and (iii) aggressive threshold/propagation calibration to the exact CAFA metric.

#### Deep dives on the most technically novel pieces

**ProtBoost's conditional-probability modeling (CondProbMod).** For a target term (node) whose parents are all zero in a training protein, the label is set to NaN and excluded from the loss. The model therefore learns Pr(node | at least one parent present) — a conditional probability. At inference, unconditional probabilities are reconstructed recursively from root to leaves:

  P_mod(node) = P_model(node) · (1 − Π_{p∈parents}(1 − P_mod(p))).

Ensembling the conditional model with the ordinary (unconditional) model gave a substantial boost. This is the *closest thing in the entire CAFA 5 field to reasoning about conditional structure* — but it conditions on the model's own predicted parents, not on externally-known annotations, so it is **not** a PK method.

**ProtBoost's GCN stacking.** The GO DAG is the graph; each protein instantiates its own node features (29 per node: 4 features from each of 5 base models = 20, plus 1 electronic-annotation feature, plus an 8-dim trainable per-term embedding). Message passing runs in three edge directions (forward, reverse, undirected), 8 layers with residual connections, BCE loss summed over proteins × nodes. This lifted the score from 0.59 to 0.62 in their ablation.

**InterLabelGO+'s loss.** The composite loss is the *product* of three terms: an IA-weighted protein-centric F1 loss, an IA-weighted GO-centric F1 loss, and the ZLPR loss

  L_zlpr = log(e^(−s₀) + Σ_{i∈pos} e^(−sᵢ)) + log(e^(s₀) + Σ_{j∈neg} e^(sⱼ)),

which pushes the logits of all positive (present) terms above all negatives with a learned threshold s₀ = 0, capturing label co-occurrence without treating each GO term as independent. Ablations showed ZLPR > BCE, and combining ZLPR with both F1 variants was best. IA(q) here is defined empirically as log₂[(1 + |proteins with parent term(s) of q|)/(1 + |proteins with term q|)]. In late-submission testing InterLabelGO+ reached wFmax ≈ 0.5764, which would have placed ~3rd.

### 3. What drove the gains over CAFA 4

The assessment paper does a head-to-head evaluation restricted to proteins in the intersection of all target sets (CAFA 2–5) and terms common to all four GO graph structures (CAFA 1 excluded for GO incompatibility). Its attribution, as best the literature exposes it:

- **Protein language model embeddings — the main technical driver.** The paper introduces a **pLM-embedding-similarity baseline** and states it provides a stronger "floor" than the traditional BLAST and Naïve baselines in most cases, explicitly crediting "the development of protein models outside the function prediction space." Every top method rests on pLMs. InterLabelGO+'s controlled ablation (ESM-2 vs no-ESM-2: +40.9% BPO wFmax) is the cleanest quantitative isolation available.
- **Database growth — real but partial.** The paper uses BLAST/Naïve baselines built on look-up databases of increasing size (the BLAST reference at t0 was 137,372 experimentally-annotated proteins: 84,562 Swiss-Prot + 52,810 TrEMBL) as a proxy. It reports that "the BLAST and Naive baselines using CAFA 5 look-ups outperformed their earlier counterparts," i.e. some improvement is simply more annotations to transfer. Datasets were also rebuilt on UniProt releases 2023_05, 2024_02, 2024_04, and 2024_06 to trace performance as annotations grew. (Consistent with the CAFA 2/3 finding that database updates explain part but not all of the gains.)
- **Structure / AlphaFold — not a demonstrated CAFA 5 driver.** The paper did *not* build a structure baseline and refers to AlphaFold2's potential contribution in the *future* tense. Only a few teams (StarFunc/#5) used structure, and there via template alignment (Foldseek/AFDB), not learned 3D features.
- **The Kaggle format itself.** The 22.2-fold increase in teams and broadened geographic participation (CAFA 3 had co-authors from 21 countries with none from South America, sub-Saharan Africa, or Oceania; all three regions were represented in CAFA 5) is presented as a first-order cause of the field-wide jump: it brought in ML practitioners whose ensembling/feature-engineering practices (text-as-features, stacking) had not been standard in bioinformatics.
- **Genuine modeling innovation.** The paper's framing — top methods beat all baselines including the strong pLM baseline — implies real modeling value-add beyond embeddings and data. I could not extract a single verbatim sentence isolating a percentage for "innovation alone"; treat the decomposition as qualitative. (This is a limitation of what is currently public.)

Absolute performance is high for MFO/CCO and lower for BPO; the CAFA co-chair stated pre-publication that CAFA 5 was nearing a mean Fmax ≈ 0.65 across the three aspects, with MFO approaching ≈0.8.

### 4. The partial-knowledge (PK) setting

**Definitions.** CAFA has long used **no-knowledge (NK)** benchmarks (proteins with no prior experimental annotation in *any* aspect at t0) and **limited-knowledge (LK)** (annotations in some other aspect but not the target aspect). CAFA 5 adds **partial-knowledge (PK)**: proteins that *already have* experimental annotation(s) *within the same aspect* at t0 and gain *new* ones within that aspect by t_e. This is the annotation-completion problem.

**Why it matters (the headline numbers).** As unannotated proteins become rarer, PK dominates. The assessment states: "This setting accounts for nearly 70% of accumulated annotations and will increasingly dominate future evaluations as annotation databases grow and fully unannotated proteins become rarer" — precisely, **69.7% of accumulated annotations**. Adding PK expanded the CAFA 5 evaluation by **229% in annotations and 508% in proteins**. Yet performance collapses: comparing the best NK model to the best PK model per aspect, Fmax drops **55.6% (BPO), 47.8% (CCO), 59.7% (MFO)**; comparing LK vs PK, drops are **62.0%, 47.7%, 54.6%**. Smin worsens correspondingly except MFO LK-vs-PK, where the top model's Smin actually *improves* by 11.9%. The organizers attribute the drop to PK terms being deeper in the DAG (rarer, higher-IA, harder) *and* to no team having built a model that conditions on known annotations.

**The information-theoretic framework for PK (Clark & Radivojac 2013, extended in the CAFA 5 Appendix A).** In the base framework the ontology is a Bayesian network; each node v carries ia(v) = −log₂ Pr(v | Pa(v)), and an annotation set's information is additive over nodes. For NK/LK, where the set of known terms T₀ = ∅, PK reduces to the standard unconditional IA. For **PK, IA is redefined as the *conditional* information of the newly added terms T₁ = T_te \ T_t0 given the already-known set T₀ = T_t0**: the paper computes Pr(T₁ | T₀) over the ontology Bayesian network and scores only the newly-discovered terms, weighting them by conditional information. Concretely, the evaluation *excludes* the terms already known at t0 (per protein) and evaluates only the "newly added" terms — this is exactly what the `-known` option in CAFA-evaluator-PK implements. Remaining uncertainty, misinformation, and Smin are all recomputed on this conditioned term set.

> **Caveat / gap:** the exact closed-form equations for Pr(T₁|T₀) and the PK-specific ru/mi/Smin in Appendix A could not be extracted verbatim (bioRxiv rate-limited full-text/PDF access throughout this research). The structure above — conditioning newly-added terms on the known set, collapsing to standard IA when T₀=∅ — is confirmed from the main-text metric section and the evaluator implementation, but a reader implementing it should read Appendix A directly.

**Prospective sub-evaluation.** The paper additionally evaluated on the stricter subset of 1,568 proteins (23,317 propagated annotations) whose annotations came from publications released *after* t0 — a true prospective test that guards against publication/text-mining leakage. A preliminary evaluation was also run on 10,304 annotations for 499 proteins. Dataset versions: t0 = GOA release 2023-07-12 and GO ontology 2023-07-27; known annotations at t0 filtered on UniProt 2023_03; propagation over is-a, part-of, regulates, negatively/positively regulates; experimental evidence codes only.

### 5. Methods that address (or could address) annotation completion / PK

No published method yet reports results on the CAFA 5 PK benchmark specifically. The adaptable body of work:

- **Positive-unlabeled (PU) learning — PU-GO.** Zhapa-Camacho, Tang, Kulmanov & Hoehndorf, "Predicting protein functions using positive-unlabeled ranking with ontology-based priors," Bioinformatics 2024, 40(Suppl_1):i401–i409 (doi:10.1093/bioinformatics/btae237). Treats unlabeled (protein, GO) pairs as *potential positives* rather than negatives, using empirical risk minimization with class priors derived from the GO hierarchy. Directly targets the false-negative problem that PK exposes and is the most on-point recent method; "PU-GO+Diamond outperforms all methods in the class-centric AUC evaluation across all subontologies" (trained on Swiss-Prot 2023_03, tested on 2024_01).
- **Weak-label / incomplete-label learning — ProWL and ProWL-IF** (Yu et al., "Protein Function Prediction with Incomplete Annotations," IEEE/ACM TCBB; PMID 26356025). Explicitly "replenish the missing functions of proteins" — i.e. annotation completion — under the assumption that a protein's known label set is a subset of the truth. Older (pre-pLM) but conceptually the exact PK formulation.
- **Structured-output "open world" theory — Jiang, Clark, Friedberg & Radivojac 2014 (Bioinformatics 30(17):i609–i616, doi:10.1093/bioinformatics/btu472)** and **Dessimoz, Škunca & Thomas 2013 (Trends Genet 29(11):609–610).** The theoretical foundation for why incomplete knowledge biases evaluation and how structured-output learning over the DAG should treat it; this is the intellectual root of the PK metric.
- **Label-space / correlation methods.** InterLabelGO's ZLPR loss, GO-term label-space dimensionality reduction (LSDR), and Domain2GO (domain→GO conditional-probability mapping) all model label correlations that a PK model would exploit — a protein's known terms are the strongest possible prior for its unknown terms.
- **Graph / network approaches.** ProtBoost's CondProbMod and GCN stacking, plus PPI/graph methods (DeepGraphGO, GOBeacon), provide the machinery to propagate from known to unknown terms; none has been pointed at PK.

The clear conceptual gap: a model that takes the **known term set T₀ as an explicit input** (e.g., as an additional feature vector, a conditioning mask, or a graph prior) and predicts the residual T₁. This is architecturally natural for transformers (condition on T₀ via cross-attention or a learned "known-terms" embedding added to the sequence embedding) and is essentially untouched.

### 6. Practical entry points

**Code & baselines**
- **CAFA-evaluator (BioComputingUP)** — official evaluator; Piovesan et al., Bioinformatics Advances 2024, doi:10.1093/bioadv/vbae043.
- **CAFA-evaluator-PK (github.com/claradepaolis/CAFA-evaluator-PK)** — adds the `-known` (per-protein known-annotation exclusion) and `-toi` (terms-of-interest) options that operationalize PK. Use `-B` with `-B_pct 100` for bootstrap; `-ia` for the IA file; `-prop max/fill` and `-norm cafa/pred/gt` for propagation/normalization.
- **InformationAccretion (github.com/claradepaolis/InformationAccretion)** — computes IA from a GOA dataset + OBO graph (`ia` CLI).
- **ProtBoost (github.com/btbpanda/CAFA5-protein-function-prediction-2nd-place)** — full 2nd-place pipeline incl. the GPU CAFA-metric implementation adopted by organizers.
- **InterLabelGO (github.com/QuanEvans/InterLabelGO)** — 6th-place; clean composite-loss reference implementation + web server.
- **GORetriever (github.com/ZhuLab-Fudan/GORetriever)** — the text-retrieval component of the 1st-place GOCurator.
- **PROTGOAT (github.com/zongmingchua/cafa5)** — 4th-place multi-PLM + TF-IDF ensemble.
- **PU-GO** — reference for the PU-learning formulation (repo linked from the Bioinformatics 2024 paper).

**Data**
- **CAFA 5 Kaggle competition data** (`kaggle.com/competitions/cafa-5-protein-function-prediction/data`): sequences, GO OBO (2023-01-01), IA weights, train terms.
- **Precomputed embeddings on Kaggle:** ProtT5 (`sergeifironov/t5embeds`), ESM-2 and Ankh (competition discussion datasets 462419 / 466703) — this removes most of the compute barrier.
- **CAFA 5 evaluation data (Zenodo record 20186533)** — the version used by CAFA-evaluator-PK, including known-annotation and ground-truth files.
- **Dataset versions to reproduce the assessment:** GO ontology 2023-01-01 (challenge) / 2023-07-27 (t0); UniProt 2023_03 (training/known) and GOA 2023-07-12 (t0 look-ups); grow-over-time releases 2023_05, 2024_02, 2024_04, 2024_06. Propagate over is-a, part-of, regulates, negatively/positively regulates; keep only experimental evidence codes (EXP, IDA, IPI, IMP, IGI, IEP, IC, TAS).

**CAFA 6.** Per Kaggle's official launch announcement (@kaggle, 15 Oct 2025): "CAFA 6 Protein Function Prediction hosted by @IowaStateU … $50,000 Prize Pool ⏰ Entry Deadline: Jan 26, 2026," with partners Northeastern, UniProt, and ISCB. This means a PK-focused method developed now can be validated on a fresh prospective blind benchmark. (The exact CAFA 6 evaluation-setting details, including whether PK is scored as a separate track, were not confirmable from public pages at the time of writing.)

**Tractable sub-problems for a small, math-strong, low-compute team**
1. **Conditional annotation completion (the flagship gap).** Build a model p(T₁ | embedding, T₀) that explicitly consumes the known term set T₀. Frozen public embeddings + a modest transformer/MLP head that cross-attends to a learned embedding of T₀. Evaluate directly with CAFA-evaluator-PK. This is the single highest-value, lowest-compute target.
2. **PU-learning with GO-hierarchy priors, evaluated in PK.** Port PU-GO's empirical-risk-minimization formulation to the PK benchmark; the class-prior-from-DAG idea is mathematically clean and needs no wet lab.
3. **Calibration / propagation theory.** The soft parent/child propagation and threshold selection are currently heuristic (e.g. ProtBoost's 0.7/0.3 blend). A principled treatment — DAG-consistent probability calibration, or exact optimization of IA-weighted Fmax under hierarchy constraints — is a pure-math contribution with immediate leaderboard payoff.
4. **Extending conditional information accretion.** The Bayesian-network conditional-IA derivation invites analysis: estimator variance, behavior under DAG mis-specification, connections to structured-output loss design. Theory-first, compute-light.
5. **Label-correlation / matrix-completion modeling of the (protein × term) matrix**, conditioned on observed entries — a natural fit for a linear-algebra-strong team, and directly the PK problem.

## Recommendations

**Stage 0 (weeks 1–2, orientation).** Reproduce a strong sequence-only baseline: download the Kaggle data + the precomputed ProtT5/ESM-2 embeddings, train a 2-layer MLP head with an IA-weighted loss, and score it with CAFA-evaluator-PK in NK, LK, *and* PK modes. Benchmark to change your plan: if your NK Fmax is not within ~0.02 of the ~0.55–0.60 range of top solutions, fix the pipeline before proceeding.

**Stage 1 (the bet: PK).** Build the conditional model p(T₁ | x, T₀). Concretely: embed the sequence (frozen pLM), embed the known-term set T₀ (sum/attention over learned GO-term embeddings — reuse ProtBoost's 8-dim per-term embeddings or learn your own), fuse, and predict residual terms. Train with an IA-weighted, DAG-aware loss (start from InterLabelGO's ZLPR × F1). **Success threshold:** meaningfully close the reported NK→PK gap (47–60% Fmax drop). Because the current field baseline is "no one conditions on T₀," any solid gain is publishable.

**Stage 2 (theory & robustness).** Add the PU-learning prior (class priors from the DAG) and a principled calibration/propagation step. Validate that gains hold on the *prospective* subset (post-t0 publications) to rule out text-leakage artifacts. Then enter CAFA 6 to get a fresh blind evaluation.

**What would change the plan.** If CAFA 6's public rules score PK separately and heavily, prioritize Stage 1 immediately and skip Stage 0 refinements. If your compute cannot even hold the embedding matrices in memory, drop to the frequency-truncated term set (top ~1,500 BPO / 800 MFO / 800 CCO, as PROTGOAT did) — this is the standard low-resource move and costs little Fmax. If you cannot beat AlignmentKNN/BLAST on PK, that itself is an informative negative result worth reporting, because it would show homology transfer is a strong PK baseline the field has not characterized.

## Caveats
- **Leaderboard ranks vs assessment ranks.** The rank labels (1st GOCurator, 2nd U900/ProtBoost, etc.) come from the Kaggle final leaderboard and the individual solution papers. The assessment paper re-scores with IA-weighted Fmax/Smin on its own benchmark; its internal top-10 ordering and per-aspect winners could not be extracted verbatim (bioRxiv rate-limited full-text access throughout). Treat the per-aspect "who won" question as unresolved from public sources here.
- **Appendix A equations not verified verbatim.** The exact conditional-IA and PK ru/mi/Smin formulas should be read directly from the preprint's Appendix A before implementation; the description here is reconstructed from the main text and the evaluator code.
- **"Decomposition" of gains is qualitative.** The paper supports pLM embeddings as the leading driver and database growth as partial, and explicitly does not credit structure, but I found no single quantitative variance-decomposition; the InterLabelGO ESM-2 ablation is the cleanest isolated number and is method-specific.
- **CAFA 6 specifics are provisional.** The competition is live but detailed evaluation-setting documentation (especially whether PK is a scored track) was not confirmable from accessible pages.
- **Some figures are from secondary sources.** Several architectural details of the 1st/3rd/5th solutions come from the ProtBoost survey and the GORetriever/StarFunc papers rather than primary write-ups; where a claim is load-bearing (e.g. StarFunc's PPI ablation) it is attributed to the source that reported it.