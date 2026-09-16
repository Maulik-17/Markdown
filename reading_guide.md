# Protein function prediction: guide to the collected literature

Prepared 16 September 2026. Project constraint: CNN models, principally MobileNet and EfficientNet.

This is a first-pass literature study of the supplied collection, with closer checks of the methods and benchmark tables most relevant to the project. It is not a reproduction of the experiments or a systematic search of all research published to date. Publication status refers to the supplied version. Reported results below are authors' results, not independently reproduced results. Recommendations are our interpretation.

The actual folder is `references/`. At the final inventory it contains 20 research/review PDFs and one dissertation preview (`out.pdf`). A project plan (`initial_plan.pdf`, with its LaTeX source) was also present and read earlier in the session, but was no longer present at the final check. The plan is not a research paper. This review did not modify or remove any original files.

## 1. What are we trying to predict?

A protein is a chain of amino acids. Its sequence influences its structure, binding partners and activities, but its biological role also depends on its surroundings. A useful function predictor should say which activities or roles are supported by the evidence, rather than merely give a protein a single generic class.

Gene Ontology (GO) describes three aspects:

| Aspect | Question | Example |
|---|---|---|
| Molecular function (MF) | What activity does it perform? | ATP binding or a particular catalytic activity |
| Biological process (BP) | What larger process does it contribute to? | DNA repair |
| Cellular component (CC) | Where is it active or located? | A membrane or protein complex |

One protein can have multiple annotations. GO prediction is therefore usually **multi-label classification**, not a one-class-only problem. GO also organizes terms into relationships; a specific annotation can imply broader annotations through appropriate relationships. See the [GO overview](https://www.geneontology.org/docs/ontology-documentation/).

Missing annotations are not proof that a protein lacks a function. They can mean that nobody has established or recorded it yet. This distinction affects training labels and evaluation. See [GO annotation guidance](https://www.geneontology.org/docs/go-annotations/).

The collection contains several different prediction problems:

| Problem | Input | Output | Representative local sources |
|---|---|---|---|
| Direct function annotation | One protein, optionally with context | GO terms or function scores | DeepGO-SE, TransFew, multi-task DNN |
| Function description matching | Protein plus candidate text | Compatibility score | ProtNote |
| Function description generation | Protein sequence | Natural-language description | Prot2Text-V2 |
| Binary interaction prediction | Two proteins, sometimes a network | Interaction / no interaction | EResCNN |
| Interaction-type prediction | Two proteins and network context | One or more interaction types | DL-PPI, HI-PPI |
| Structure prediction | Sequence and evolutionary information | Three-dimensional coordinates | AlphaFold |
| Specific activity/binding prediction | Sequence/structure | A selected function or binding site | DeepRNN, PiCAP/CAPSIF2 |

**Project implication:** predicting that A interacts with B does not establish that A performs B's molecular function. Partners may share a process while having different activities. An interaction model needs a separate, evaluated function-inference stage before it becomes a function annotation system. A predicted structure is likewise evidence for investigating function, rather than a direct experimental determination of function.

## 2. Recommended reading sequence

Read the collection in conceptual order rather than filename order. After each paper, explain its input, target, method, evaluation and main limitation without looking at the PDF.

1. **Field and terminology:** the Proteomics review, then the 2021 review. Learn GO, annotation evidence, sequence/structure/network inputs and evaluation.
2. **Direct prediction:** multi-task DNN and DeepRNN, then DeepGO-SE. Understand multi-label outputs, learning from sequence, and the distinction between a prediction and an experimentally tested candidate.
3. **CNN and interaction methods:** EResCNN, DL-PPI and HI-PPI. Compare their different targets and the information available to each model.
4. **Modern representations and labels:** ESM, TransFew, ProtNote and Prot2Text-V2. Understand what newer approaches gain from pretraining, label relationships and text.
5. **Structure and context:** AlphaFold, the structure review, the complex atlas and PiCAP/CAPSIF2. Understand when structure adds evidence and what its predictions cannot establish.
6. **Broader context:** the two language-model reviews, host-pathogen review, attributed graph embedding and dissertation preview.

A good first group discussion can cover just the Proteomics review, EResCNN, DL-PPI and DeepGO-SE. Together they expose the most consequential design decisions.

## 3. Notes on every research document

Page references below use PDF page order. Section and figure/table names are included where more useful than a page number.

### 1. Deep learning methods for protein function prediction

Source: [Boadu, Lee and Cheng, Proteomics 2025](../references/PMIC-25-2300471.pdf). Accepted in 2024; journal citation is 2025.

- **Question:** What data and deep-learning approaches support functional annotation?
- **Main idea:** Organize methods around sequences, structures, interaction networks, domains and other information. Figure 1 gives the overall input-to-feature-to-function workflow.
- **What to learn:** Data sources, GO targets, evaluation metrics, and the differences between CNN, graph and language-model approaches.
- **Project relevance:** Best starting overview. Use it to justify the biological task before discussing MobileNet or EfficientNet.
- **Reading caution:** It is a review, not a controlled head-to-head benchmark. Trace performance claims to the original experiments before citing them as a baseline.

### 2. A Review of Deep Learning Techniques for Protein Function Prediction

Source: [Aggarwal and Hasija, INCET 2021](../references/1_Review_DeepLearning_ProteinFunctionPrediction.pdf), six pages.

- **Question:** How can deep learning address the gap between known sequences and known functions?
- **Main idea:** An introductory survey connecting protein function classification with developments in deep learning.
- **What to learn:** Why sequence similarity and neural representations are useful, and why automated annotation is needed.
- **Project relevance:** Accessible historical introduction; read before the more technical papers if the team is new to biology or AI.
- **Reading caution:** Its account of the frontier belongs to 2021. Do not present its model survey as the current state of the field.

### 3. Deep Recurrent Neural Network for Protein Function Prediction from Sequence

Source: [Xueliang Leon Liu, supplied manuscript](../references/2_DeepRNN_ProteinFunctionPrediction.pdf). Read abstract, Results/Model Architecture, functional prediction experiments and experimental validation.

- **Question:** Can raw sequence support recognition of selected functions, including proteins distant from familiar families?
- **Method:** LSTM recurrent networks learn directly from amino-acid sequences. The work studies four selected functional classification tasks and searches for additional candidates.
- **Evidence:** The manuscript reports experimental validation for some ferritin-like iron-sequestering predictions. This is especially relevant to the project's ambition to identify functions.
- **Project relevance:** Learn how a computational screen leads to a manageable candidate list and then biological testing. The experimental workflow is useful even though the network is not a CNN.
- **Limitation:** Success on selected functions is not a demonstration of accurate prediction across the full GO. Check out-of-class evaluation separately from in-class results.

### 4. Multi-task Deep Neural Networks in Automated Protein Function Prediction

Source: [Rifaioglu et al., arXiv version, May 2017](../references/3_MultiTaskDNN_ProteinFunctionPrediction.pdf). Read methods, Tables 3–4 and Conclusions.

- **Question:** Can shared learning and GO hierarchy improve prediction across many function labels?
- **Method:** Hierarchical, multi-task deep networks and a prediction procedure that considers parent GO terms.
- **Evidence:** Results depend on the amount of training data per label. Adding annotations with less reliable evidence helps some weakly performing categories but harms some strong ones.
- **Project relevance:** A CNN backbone still needs a properly designed multi-label output and annotation policy. Shared features and hierarchy are relevant independently of the backbone.
- **Limitation:** Older datasets and evaluation choices prevent direct comparison with newer published scores. “More annotated examples” does not automatically mean better biological evidence.

### 5. Biological structure and function emerge from scaling unsupervised learning to 250 million protein sequences

Source: [Rives et al., supplied December 2020 bioRxiv version](../references/4_ESM_BiologicalStructureFunction.pdf). Read abstract and representation/downstream evaluation sections.

- **Question:** What biological information can be learned from sequences without explicit function labels?
- **Method:** Large-scale protein language modeling produces contextual representations. The supplied version describes training on 250 million sequences and 86 billion amino acids.
- **Evidence:** Learned representations contain information relevant to structure, remote homology and downstream prediction tasks.
- **Project relevance:** Explains why modern papers use pretrained embeddings instead of relying entirely on manually designed descriptors. An embedding-based comparator can help evaluate the quality of CNN inputs.
- **Limitation:** This is not itself a complete general GO annotation pipeline. Pretraining scale does not establish performance on your dataset. The original ESM work should not be conflated with later ESM2 models.

### 6. Highly accurate protein structure prediction with AlphaFold

Source: [Jumper et al., Nature 2021](../references/5_AlphaFold_Nature2021.pdf). Read opening results, architecture description and confidence discussion.

- **Question:** Can sequence and evolutionary information predict accurate three-dimensional protein structures?
- **Method:** AlphaFold integrates biological and geometric constraints, multiple-sequence alignment information and learned structure prediction.
- **Evidence:** Strong CASP14 structure-prediction performance.
- **Project relevance:** Structures can supply contact or distance maps for a CNN and help examine possible active sites.
- **Limitation:** A structure benchmark measures coordinate accuracy, not GO annotation accuracy. Structural uncertainty, missing biological context and conformational changes matter when reasoning about function.

### 7. A Comprehensive Review of Transformer-based language models for Protein Sequence Analysis and Design

Source: [Ghosh et al., arXiv July 2025 version](../references/6_Review_TransformerLMs_ProteinSeq.pdf).

- **Question:** How are transformer language models used across protein analysis and design?
- **Main idea:** Survey sequence modeling, function and structure tasks, binding, and protein generation.
- **Project relevance:** Provides context for explaining what a computationally efficient CNN approach is competing with and which features could later be borrowed.
- **Reading caution:** Distinguish prediction from generation and each task's benchmark. The generic “JOURNAL OF LATEX CLASS FILES” header is a template artifact; use the visible arXiv version metadata for this copy.

### 8. Prot2Text-V2: Protein Function Prediction with Multimodal Contrastive Alignment

Source: [Fei et al., October 2025 arXiv v3; copy marked NeurIPS 2025](../references/7_Prot2Text-V2.pdf). Read Figure 1, Methods, evaluation and limitations.

- **Question:** Can sequences be translated into free-form functional descriptions?
- **Method:** ESM sequence encoder, a nonlinear projector and LLaMA-3.1-8B-Instruct. Hybrid Sequence-level Contrastive Alignment Learning aligns protein and text representations, followed by instruction tuning with LoRA.
- **Evidence:** Training uses about 250,000 curated Swiss-Prot entries; evaluation includes low-homology conditions.
- **Project relevance:** Defines a possible later output interface, but it is a different problem from predicting fixed GO labels with a CNN.
- **Limitation:** Fluent or textually similar output is not sufficient biological validation. Check correctness of individual functional claims, particularly for unfamiliar proteins.

### 9. Protein function prediction as approximate semantic entailment (DeepGO-SE)

Source: [Kulmanov et al., Nature Machine Intelligence 2024](../references/8_DeepGO-SE_NatMachIntell.pdf). Figure 1 and Tables 1–3, PDF pp. 3–4; dataset methods on p. 5.

- **Question:** Can a predictor use the logical knowledge encoded in GO, beyond treating labels as unrelated classes?
- **Method:** ESM2 embeddings plus multiple approximate models of GO axioms, with aggregated function predictions. Additional variants incorporate interaction information.
- **Evidence:** On its Swiss-Prot benchmark, DeepGO-SE reports MF Fmax 0.554, BP 0.432 and CC 0.721. The dataset contains 77,647 proteins and uses a sequence-similarity-based split.
- **Project relevance:** Strong example of direct function prediction and an evaluation protocol to study. Ontology-aware outputs may matter as much as choosing the backbone.
- **Limitation:** Information sources differ across model variants. Adding PPIs improves some aspects but does not uniformly improve MF. These scores cannot be compared numerically with binary PPI accuracy.

### 10. Improving protein function prediction by learning and integrating representations of protein sequences and function labels (TransFew)

Source: [Boadu and Cheng, March 2024 bioRxiv version](../references/9_TransFew_biorxiv.pdf). Read architecture, rare-term evaluation and Conclusions.

- **Question:** How can prediction improve for function labels with few annotated proteins?
- **Method:** ESM2-t48 represents sequences; BioBERT and a graph-convolutional autoencoder represent label definitions and GO relationships; cross-attention combines these representations.
- **Evidence:** Authors report improved prediction of rare terms as well as overall performance.
- **Project relevance:** Shows why a single aggregate score is insufficient. Evaluate common and rare functions separately even if the final model remains CNN-based.
- **Limitation:** Few-shot prediction of sparsely observed labels differs from zero-shot prediction of completely unseen labels. A MobileNet replacement would be a new experiment, not the published architecture.

### 11. ProtNote: a multimodal method for protein-function annotation

Source: [Char et al., October 2024 bioRxiv version](../references/10_ProtNote_biorxiv.pdf). Figure 1 and Table 1 on p. 2; Methods on pp. 2–4.

- **Question:** Can a model score a protein against a textual function description, including functions unseen during supervised training?
- **Method:** Frozen ProteInfer protein representations and frozen E5 text embeddings are combined to predict protein–function compatibility. It is not simply ESM plus a classifier.
- **Evidence:** The supplied version evaluates supervised GO annotation and separate zero-shot settings, including newly introduced GO labels and EC annotations. It reports both macro and micro mean average precision.
- **Project relevance:** A useful example of coupling a protein encoder to label semantics. It also highlights class imbalance and the distinction between new proteins and new labels.
- **Limitation:** Inspect sequence duplication and random-split construction in Table 1. Zero-shot benchmark success does not experimentally establish an entirely new biochemical activity.

### 12. Protein Language Models: Applications and Perspectives

Source: [Leclercq and Droit, Journal of Proteome Research 2026](../references/11_PLM_ApplicationsPerspectives.pdf), pp. 507–524.

- **Question:** Where are protein language models useful, and what limits their use?
- **Main idea:** Review applications of learned sequence representations across proteomics and discuss future directions.
- **Project relevance:** Use after ESM to place modern representations in context. Its discussion of sequence-only limitations motivates evaluating biological context rather than assuming larger models solve every task.
- **Limitation:** A review's broad application claims are not controlled evidence that any particular model beats your CNN on the selected dataset. Date and database version matter for quoted coverage statistics.

### 13. Prediction of protein-protein interactions based on ensemble residual convolutional neural network (EResCNN)

Source: [Gao et al., Computers in Biology and Medicine 2023](../references/1-s2.0-S0010482522011799-main.pdf). Methods pp. 2–4; Tables 1–2 on p. 5.

- **Question:** Does combining sequence descriptors and different classifiers improve binary PPI prediction?
- **Method:** Six descriptors—PseAAC, AC, PsePSSM, EBGW, MMI and conjoint triad—form a 1,686-dimensional pair representation. A residual CNN predictor is ensembled with LightGBM, XGBoost, random forest and Extra-Trees.
- **Evidence:** Five-fold benchmark accuracies are 95.34% for yeast, 87.89% for H. pylori and 98.61% for Human–Y. pestis.
- **Project relevance:** Closest collected example for descriptor-based CNN interaction prediction. Separate representation, neural architecture and ensembling contributions.
- **Limitation:** A standalone MobileNet is not directly equivalent to this ensemble. Audit negative sampling and protein/homology overlap. The printed MCC equation contains a nonstandard denominator factor; use a verified metric implementation rather than transcribing it.

### 14. DL-PPI: a method on prediction of sequenced protein–protein interaction based on deep learning

Source: [Wu et al., BMC Bioinformatics 2023](../references/s12859-023-05594-5.pdf). Dataset p. 5; architecture pp. 6–11; Table 1 p. 13.

- **Question:** How can sequence features and interaction-network information improve PPI predictions, including unfamiliar proteins?
- **Method:** Sequence encoding, multi-scale Inception-style convolution, graph-isomorphism-network feature learning and attention/pair prediction components.
- **Targets:** The main STRING/SHS benchmarks use seven interaction types: activation, binding, catalysis, expression, inhibition, post-translational modification and reaction. A separate yeast experiment uses binary labels.
- **Evidence:** SHS27k micro-F1 changes from 89.12% with random partitioning to 72.95% with BFS partitioning. See the full table below.
- **Project relevance:** Split design and network context can change conclusions substantially.
- **Limitation:** A pure sequence model and a graph-informed model use different information. Do not reduce all its datasets to a single “interacts / does not interact” task.

### 15. A novel link prediction algorithm for protein-protein interaction networks by attributed graph embedding

Source: [Nasiri et al., Computers in Biology and Medicine 2021](../references/1-s2.0-S0010482521005667-main.pdf).

- **Question:** Can network structure and protein attributes jointly improve missing-edge prediction?
- **Method:** Modified DeepWalk, with feature selection and weighting to influence attributed-network representation learning.
- **Project relevance:** Explains what graph context adds beyond a sequence-only classifier and supplies conceptual context for network-based baselines.
- **Limitation:** Missing-edge prediction in an existing graph is different from annotation of an isolated, newly discovered protein. Determine what neighborhoods and attributes are available at inference time before comparing methods.

### 16. Prediction of protein–protein interaction based on interaction-specific learning and hierarchical information (HI-PPI)

Source: [Tang et al., BMC Biology 2025](../references/s12915-025-02359-9.pdf). Read Figure 1, Results and Methods.

- **Question:** Can network hierarchy and pair-specific features improve interaction prediction?
- **Method:** Structural and sequence representations feed a hyperbolic graph model; a gated interaction network models pairs. Structural features include contact-map-derived information.
- **Evidence:** The paper compares methods on shared splits and reports repeated experiments; its abstract reports micro-F1 improvements over the next-best method in its experiments.
- **Project relevance:** Highlights the importance of protein-pair information and a fair shared-split comparison.
- **Limitation:** Uses richer structural/network input than a simple sequence image. Its published numbers should not be ranked against DL-PPI without checking identical splits and preprocessing.

### 17. Comprehensive review and assessment of machine learning approaches for host-pathogen protein-protein interaction prediction

Source: [Noor and Qamar, Briefings in Bioinformatics 2026](../references/bbag051.pdf), published 10 February 2026.

- **Question:** What helps predict interactions across host and pathogen proteins?
- **Main idea:** Review transfer learning, hybrid/ensemble methods and the challenges of sparse data, imbalance and pathogen evolution.
- **Project relevance:** Especially useful if selecting the Human–Y. pestis benchmark or testing cross-organism generalization.
- **Limitation:** Host-pathogen interactions form a specific domain. Evidence from them is not automatically transferable to general GO annotation or within-species PPI prediction.

### 18. Protein structure prediction via deep learning: an in-depth review

Source: [Meng et al., Frontiers in Pharmacology 2025](../references/fphar-16-1498662.pdf), published 3 April 2025.

- **Question:** How has deep learning changed protein structure prediction?
- **Main idea:** Review structure-prediction approaches and their applications, including remaining limitations in complex cases, dynamics and computational cost.
- **Project relevance:** Background for deciding whether predicted structures are a practical input to a CNN.
- **Limitation:** This is a structure review, not evidence that MobileNet/EfficientNet predicts function accurately. Structure confidence and function confidence require different evaluation.

### 19. Atlas of predicted protein complex structures across kingdoms

Source: [Nature Communications 2026](../references/s41467-026-70884-4.pdf), DOI 10.1038/s41467-026-70884-4.

- **Question:** What can large-scale complex modeling reveal about interactions and conserved biology?
- **Method/evidence:** The paper describes 1.1 million ColabFold/AlphaFold2-based predicted interaction structures, including 181,671 classified as high confidence, with selected experimental support and downstream analyses.
- **Project relevance:** A possible structural resource and an example of connecting computational hypotheses with experiments.
- **Limitation:** Predicted structures and confidence filters do not make every complex an experimentally verified positive. Training and testing on related predicted resources can create circular evidence or overlap.

### 20. Predictions from deep learning propose substantial protein–carbohydrate interplay

Source: [Canner, Schnaar and Gray, 2026](../references/canner-et-al-2026-predictions-from-deep-learning-propose-substantial-protein-carbohydrate-interplay.pdf). Read protein-level versus residue-level Results and dataset construction.

- **Question:** Which proteins bind carbohydrates, and which residues participate?
- **Method:** Equivariant graph neural networks: PiCAP for protein-level binding prediction and CAPSIF2 for binding-site residues, using structural/sequence information.
- **Evidence:** Authors report approximately 90% balanced accuracy for protein-level prediction and approximately 0.57 Dice for independent residue-level prediction. Their proteome-wide binding proportions are predictions.
- **Project relevance:** A concrete, narrower functional task illustrates how to define an experimentally meaningful endpoint.
- **Limitation:** Likely nonbinding proteins are an assumed negative class. Accuracy on the test set does not prove all large-scale predicted binders actually bind carbohydrates.

### 21. Interpretable Machine Learning for Sequence–Property Mapping and Guided Biological Sequence Design

Source: [Akash Pandey, June 2026 dissertation preview](../references/out.pdf).

- **What is available:** A 24-page, preview-watermarked extract with front matter, abstract, contents and introductory material. The contents refer to chapters beyond the supplied pages.
- **Main idea:** Sequence–property prediction, interpretable representations and guided design; the abstract describes B-factor prediction, silk properties and the COLOR architecture.
- **Project relevance:** Context for interpretability and relating learned features to biological properties.
- **Limitation:** This is not a complete dissertation in the local collection. The abstract's numerical claims cannot be fully audited against the missing experimental chapters. It is peripheral to GO annotation.

## 4. Scores worth studying—and how to interpret them

These are separate benchmark panels, not a combined leaderboard. Scores depend on label definitions, data release, split, input information, negative sampling and evaluation implementation.

### EResCNN: binary PPI classification

Source: Table 2, PDF p. 5; dataset descriptions p. 2. Five-fold cross-validation.

| Dataset | Positive / negative pairs reported | Accuracy | MCC |
|---|---:|---:|---:|
| S. cerevisiae | 5,594 / 5,594 | 95.34% | 0.9086 |
| H. pylori | 1,458 / 1,458 | 87.89% | 0.7581 |
| Human–Y. pestis | 2,067 / 2,067 | 98.61% | 0.9723 |

The six-descriptor fusion and full ensemble are part of these results. Global redundancy filtering does not by itself establish that proteins or homologous families are separated between folds.

### DL-PPI: interaction-type prediction

Source: Table 1, PDF p. 13. All entries are micro-F1 percentages.

| Dataset | Random | BFS | DFS |
|---|---:|---:|---:|
| SHS27k | 89.12 | 72.95 | 78.07 |
| SHS148k | 92.49 | 68.87 | 85.45 |
| STRING | 94.85 | 77.53 | 92.76 |

BFS and DFS denote breadth-first and depth-first graph traversal partition schemes. They change which interactions occur in the test set. They are not automatically interchangeable with a strict sequence-cluster split or complete protein-disjoint evaluation. Audit actual membership.

### DeepGO-SE: direct function prediction

Source: Tables 1–3, PDF pp. 3–4. Swiss-Prot benchmark, base DeepGO-SE model.

| GO aspect | Fmax | AUPR |
|---|---:|---:|
| Molecular function | 0.554 | 0.552 |
| Biological process | 0.432 | 0.401 |
| Cellular component | 0.721 | 0.730 |

These measure GO annotation performance; 0.554 Fmax cannot be interpreted as 55.4% ordinary accuracy or compared with EResCNN's 95.34% accuracy.

### Metric vocabulary

- **Precision:** Of the predictions called positive, how many are supported by test labels?
- **Recall:** Of the test positives, how many did we retrieve?
- **F1:** Harmonic mean of precision and recall. Micro aggregation favors frequent labels; macro aggregation gives labels equal weight under the specified implementation.
- **Fmax:** Best F score over a threshold sweep, using the benchmark's exact protein-centric definition. It is a reporting metric; deployment thresholds should be chosen on validation data.
- **AUPR / average precision:** Precision–recall summaries. Specify the calculation and averaging: trapezoidal PR area and average precision are not numerically identical in general.
- **MCC:** A classification correlation measure using all four confusion-matrix counts.
- **Smin:** An ontology-information-weighted error measure used in function evaluation; lower is better.
- **Balanced accuracy:** Average class recall in binary classification; helpful when classes have different sizes.
- **Dice:** Overlap measure used here for residue-level binding-site predictions.

## 5. What the literature implies for MobileNet and EfficientNet

MobileNet emphasizes efficient depthwise-separable convolution; EfficientNet studies coordinated scaling of network depth, width and resolution. These are architecture motivations, not evidence of biological accuracy. See the original [MobileNet paper](https://arxiv.org/abs/1704.04861) and [EfficientNet paper](https://proceedings.mlr.press/v97/tan19a.html). Record the exact MobileNet variant when designing experiments.

The most defensible project question is: **Can an efficient CNN produce useful protein function predictions at a lower computational cost under a carefully controlled evaluation?** This is a proposed research question, not a result established by the collected papers.

For direct function annotation, a possible experimental pipeline is:

```text
one protein sequence
  -> documented numerical representation
  -> MobileNet or EfficientNet encoder
  -> pooled features
  -> K function scores, one sigmoid output per GO label
  -> validation-selected decision rule and GO-consistency handling
```

For a PPI project, the input is instead a protein pair and the output is either an interaction probability or multiple interaction-type scores. Inferring GO functions from that output is an additional task requiring its own labels and held-out evaluation.

### Input representations need their own experiments

| Candidate representation | Potential benefit | Main question to test |
|---|---|---|
| Residue-by-feature matrix | Preserves position along the sequence | Does convolution along the feature axis have useful meaning? |
| Evolutionary PSSM | Encodes conservation across related sequences | Are database-search cost, coverage and database version acceptable? |
| Contact/distance map | Has a meaningful two-dimensional residue-pair interpretation | Are structures reliable, and does resizing remove useful detail? |
| Reshaped descriptor vector | Simple fixed-size input | Does arbitrary layout create artificial local relationships? |
| Protein-language-model embeddings | Rich pretrained sequence information | Is the extra representation cost allowed by the CNN constraint? |

PSSM, PseAAC and conjoint triad are different kinds of representations. A PSSM is a position-by-amino-acid matrix; composition descriptors usually produce vectors. PSSM generation involves evolutionary sequence searches, not merely a local arithmetic conversion of one sequence.

Do not assume that reshaping or tiling numbers into a 224×224 RGB image preserves biology. Input resolution and channel handling depend on the chosen implementation; 224×224×3 is not an immutable property of every CNN architecture. ImageNet pretraining is a hypothesis to test against random initialization. Rotations and flips should not be borrowed automatically from photography because they can change the meaning of sequence-based inputs.

Reasonable future ablations include the same representation with a simple MLP, a small 1D CNN, MobileNet and EfficientNet; the same backbone with different representations; and pretrained versus randomly initialized weights. If the project permits non-CNN comparators, include sequence-similarity transfer and a frozen protein-embedding baseline. These are experimental suggestions, not work already performed.

## 6. Corrections to the existing initial plan

Source: `initial_plan.pdf` and `initial_plan.tex`, inspected earlier in this session. They were no longer present in `references/` at the final check; the observations below refer to the version read earlier. This review did not modify or remove them.

| Existing assumption | Correction needed |
|---|---|
| Interaction prediction serves as the project endpoint | Your stated objective is function prediction. Either predict GO labels directly or define and evaluate a second stage that infers functions from interactions. |
| Interacting proteins do the same job | They may participate in the same process while performing different molecular activities. Also distinguish physical binding from functional association. |
| Every dataset is binary interaction / non-interaction | DL-PPI's principal STRING/SHS experiments predict seven interaction types; its yeast setting is different. |
| EResCNN is a CNN baseline using a few descriptors | It uses six descriptors and an ensemble containing a residual CNN plus four tree classifiers. |
| Descriptor-to-image conversion is a neutral formatting step | It is a modeling decision that may introduce artificial spatial relationships or discard information. |
| ImageNet weights will transfer successfully | This requires an ablation against training from scratch under the same evaluation. |
| Published numbers form one scoreboard | Scores must be separated by task and compared only under matching protocols. |

The immediate literature-study deliverable should therefore be agreement on the prediction target, input information and evaluation—not installation or full model training.

## 7. How to evaluate a credible function-prediction study

Before implementation, record these decisions:

1. **Target:** GO MF, BP and/or CC; selected functional classes; or PPI types. Define which labels are actually predicted.
2. **Evidence:** Dataset release, GO version, evidence-code policy and treatment of unannotated labels. “Reviewed” and “experimentally verified” are not synonyms.
3. **Split:** Prevent identical sequences and close homologs from creating an artificially easy test. Consider sequence-cluster or temporal evaluation and report how it was constructed.
4. **PPI-specific leakage:** Audit reversed duplicate pairs, shared proteins, homologs and graph edges used during feature construction. Specify whether both, one or neither member of a test pair has appeared in training.
5. **Preprocessing:** Fit learned normalization or feature selection on training data. Freeze label vocabulary and selection rules without using test outcomes.
6. **Comparison:** Keep data, input information, tuning budget and splits consistent. Report multiple seeds where practical and document uncertainty.
7. **Metrics:** Use task-appropriate precision–recall and F metrics; inspect rare labels and sequence-similarity strata. Do not rely only on overall accuracy.
8. **Efficiency:** Measure the whole pipeline, including sequence searches or embeddings, as well as network parameters, inference time and memory.
9. **Biological follow-up:** Rank candidate functions with supporting domains, conserved residues, structural context or other independent evidence. State which predictions need experiments.

Calling a function experimentally identified requires relevant empirical evidence. A high model score, an attractive attention map or a plausible text description alone does not provide that evidence.

## 8. A reusable paper-reading worksheet

For each paper, prepare answers to these questions:

1. What exact biological question is being answered?
2. What constitutes one example: protein, pair, residue, graph or protein–text pair?
3. What information is available at training and prediction time?
4. How are positive, negative and missing labels defined?
5. What does the architecture add beyond the input representation?
6. How were train, validation and test examples separated?
7. Which figure or table provides the central evidence, and what metric is used?
8. Which ablation supports the claimed contribution?
9. What kind of unfamiliar protein or function was actually tested?
10. What can we reuse within a CNN project, and what claim remains unproven?

The group should be able to explain why PPI prediction and GO prediction differ; why random splits can be optimistic; why rare terms are difficult; why structure is useful but not sufficient; and what experiment would demonstrate that MobileNet or EfficientNet adds value.

## 9. Scope and unresolved checks

This guide covers every local PDF at an orientation level, with targeted methods/results review for the central papers. EResCNN p. 5, DL-PPI p. 13 and DeepGO-SE p. 3 were also rendered and visually checked to verify key tables. Other notes rely on extracted text and document metadata; this is not a claim to have audited every equation or supplement.

No datasets were downloaded, model code executed, benchmark reproduced or scientific result experimentally validated. Repository availability and current publication status of preprints were not independently audited. The supplied dissertation is incomplete. A broader current-literature search may reveal additional methods beyond this collection.

The key unresolved project decision is whether the required endpoint is **direct protein function annotation** or **interaction prediction followed by function inference**. The stated objective supports the former; the existing plan describes the latter. Keep the two distinct during discussion with the project supervisor.
