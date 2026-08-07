The field calls this **Automated Function Prediction (AFP)**, and it's framed as *hierarchical multi-label classification over a DAG*, not simulation.

Functions are described using **Gene Ontology (GO)** terms. GO is a concept hierarchy structured as a directed acyclic graph, where nodes are functional terms connected by relational ties, with more specific terms as descendants of broader ones, split into three sub-ontologies: Molecular Function, Biological Process, Cellular Component. Your model takes a protein and outputs a confidence score for each of ~40,000 GO terms, subject to a consistency constraint — if you predict a term, you must predict all its ancestors.

The current standard recipe:

- **Input**: amino-acid sequence → embeddings from a frozen protein language model (ESM-2, ProtT5). Optionally a predicted 3D structure → graph neural network over the residue contact graph. Optionally protein-protein interaction networks and literature text.
- **Head**: a classifier over GO terms, with a loss that respects label hierarchy and extreme class imbalance.
- **Ensemble**: blend the learned predictor with retrieval — nearest neighbours in embedding space, plus BLAST homology transfer. In CAFA 5, protein language model embedding similarity and non-experimental annotation baselines outperformed traditional BLAST and Naïve baselines in most cases, and essentially every top method is a hybrid.

For a small team, I'd argue against attacking full GO prediction on day one. Narrower, better-defined targets: **EC number prediction** (enzyme class), **subcellular localization**, **binding-site residue prediction**, or **protein-protein interaction prediction**. Each has cleaner labels and a sharper success criterion.

## Topics to study, and sources

**Domain (the part you're missing).** Amino acids, the four levels of protein structure, the central dogma, post-translational modifications, enzymes vs. binders vs. transporters vs. structural proteins. Read Alberts, *Molecular Biology of the Cell*, chapter 3 only — don't read the book. Supplement with Lehninger's *Principles of Biochemistry* chapters 3–6.

**Gene Ontology and annotation.** The DAG structure, evidence codes, annotation propagation, and — critically — the *open-world assumption* (absence of an annotation does not mean absence of the function). Sources: GO Consortium documentation, Ashburner et al. 2000, and Clark & Radivojac 2013 for the information-theoretic evaluation framework.

**Sequence homology.** BLAST, HMMER, profile HMMs, multiple sequence alignments, Pfam/InterPro domains. Compeau & Pevzner's *Bioinformatics Algorithms* is the right level.

**Protein language models.** Rives et al. 2021 (ESM-1b), Lin et al. 2023 (ESM-2), Elnaggar et al. (ProtTrans). Your transformer background transfers directly — these are masked language models over a 20-token alphabet.

**Structure prediction.** Jumper et al. 2021 (AlphaFold2) — read the methods and supplement, it's a genuinely mathematical paper. Then AlphaFold3, ESMFold.

**Geometric deep learning.** This is where your linear algebra and analysis background is worth the most: SE(3)-equivariance, group representations, GVP-GNNs, message passing on point clouds. Bronstein et al.'s *Geometric Deep Learning* proto-book is the reference.

**Reviews to orient yourself.** Boadu et al. 2025 in *PROTEOMICS* ("Deep learning methods for protein function prediction") is the best single survey. Then the CAFA5 analysis paper (De Paolis Kaluza et al., bioRxiv 2026) for where the field actually stands.

## What's actually being pursued

- **pLM embeddings + retrieval.** FANTASIA applies protein language models to ~1000 animal proteomes and assigns functions to virtually all proteins, including up to 50% left unannotated by homology-based methods.
- **Structure-based, interpretable prediction.** DPFunc uses domain information to guide structure-based prediction and detects the key residues or regions responsible for a predicted function — relevant to you, since your professor's other project is explicitly about explainability.
- **Multimodal generative models.** ESM3 reasons jointly over sequence, structure, and function tokens; it generated esmGFP, a fluorescent protein roughly 500 million years of evolutionary distance from any known natural GFP, whose activity was confirmed in the lab.
- **Dynamics-aware models.** A live acknowledgment that a single static structure is insufficient. DeepJump, a flow-matching model for conformational dynamics, reports roughly 1000× computational acceleration while recovering long-timescale behaviour — this is the *learned* version of what you wanted to brute force.
- **The dark proteome.** A 2026 Nature study identified more than 1,700 previously unrecognized protein-like molecules from over 7,000 non-canonical open reading frames, most under 50 amino acids, for which the team coined the term "peptideins" because their biological roles remain unclear.

**And here's the concrete open problem** — the most useful thing I found for you. CAFA5 introduced a "partial knowledge" setting: predicting *new* functions for proteins that already have some annotations. Comparing the best no-knowledge and partial-knowledge models per aspect, performance drops by 55.6%, 47.8%, and 59.7% — yet this setting accounts for 69.7% of all accumulated annotations. The authors explicitly flag the opportunity for models that condition on already-known annotations to predict the remaining ones.

That is a stated, quantified gap in a benchmark paper published three months ago, and it's a *structured prediction* problem — conditional information gain over a DAG. It's about as good a fit for someone with your background as you'll find.

## Does this require experimental data

Yes, but you don't need to generate it. You **consume** experimental data — GO annotations carry evidence codes, and only experimentally-validated ones count as ground truth. Experimental knowledge of protein function remains extremely limited, with fewer than 0.5% of proteins characterized.

A purely computational contribution (new method + rigorous benchmark) is publishable without a wet lab. But two warnings:

- **Use temporal splits, never random splits.** CAFA's whole design is prospective — predictions are locked before ground-truth annotations accumulate, ensuring evaluation data didn't exist during training and reducing leakage. Random splits will give you a beautiful, meaningless number.
- **Absence of a label ≠ absence of the function.** Your negatives are unreliable by construction. This is the single biggest methodological trap in the field.

If you want to claim you've *discovered* a function rather than predicted one, you need a collaborator with a lab. CAFA does exactly this — computational predictions driving targeted screens.

## Where to get data

| Purpose | Resource |
| Sequences + annotations | UniProtKB (Swiss-Prot reviewed, TrEMBL unreviewed) |
| Function labels + DAG | Gene Ontology, GOA |
| Experimental structures | RCSB PDB |
| Predicted structures | AlphaFold DB (~214M) |
| Domains, families, folds | InterPro, Pfam, CATH, SCOP |
| Interactions | STRING, BioGRID, IntAct |
| Human expression + localization | Human Protein Atlas, neXtProt |
| Pathways | Reactome, KEGG |
| Enzyme reactions | BRENDA, Rhea |
| Mass spectrometry | PRIDE, PeptideAtlas |
| Ready-made benchmark | CAFA5 Kaggle dataset |

One caveat worth knowing: AlphaFold DB drifts out of sync with UniProt. Comparing the 2025 UniProt release against AlphaFold's human proteome models, 631 of 20,504 sequences differed — a 3.08% discrepancy. Check accession versions rather than assuming they match.

Concrete first step: download the CAFA5 Kaggle data, compute ESM-2 embeddings, and reproduce the embedding-similarity baseline. It's a weekend of work, it forces you to confront GO propagation and the Fmax/Smin metrics hands-on, and it gives you a number to beat. The CAFA organizers explicitly suggest developers use these baselines as a starting point for models.

A deeper investigation could map the architectures of all ten top-performing CAFA5 teams against each other, trace which specific design choices drove the gains over CAFA4, and survey what has been published on the partial-knowledge setting since the benchmark paper appeared.I've laid out the answers above — the research suggestion is there if you want the deeper architectural comparison.
