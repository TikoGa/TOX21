Tox21 Enrichment-Conditioned Graph Neural Network
A multitask molecular toxicity prediction pipeline for the 12 Tox21 endpoints. The model combines atom–bond graph representations, global physicochemical descriptors, SMARTS-defined functional-group features, and training-derived functional-group enrichment profiles. Local graph message passing is integrated with global multi-head self-attention through an enrichment-conditioned gating mechanism.
Research status: This repository contains an experimental research implementation. Results below describe the saved run in tox21_GNNEnrichedFG.ipynb; they are not a claim of external validation or superiority over published benchmarks.

Highlights
- 12-task prediction: nuclear receptor and stress-response assays in Tox21.
- Two duplicate-label strategies: consensus aggregation and majority voting.
- Scaffold-based evaluation: 80% train / 10% validation / 10% test, with Bemis–Murcko scaffolds separated across partitions.
- Multimodal molecular input: atom features, bond features, five physicochemical descriptors, 19 functional-group counts, and a 12-dimensional enrichment profile.
- Leakage-aware enrichment features: enrichment statistics are computed from training labels; training examples use scaffold-grouped cross-fitting.
- Missing-label masking: unknown assay labels are excluded from the loss and evaluation.
- Imbalance-aware optimization: task-specific positive-class weights and validation-derived thresholds.
Model overview
For each molecule, the pipeline constructs a graph where atoms are nodes and bonds are edges. The model then processes the molecule through the following components:
1. Atom encoder: five atom features are projected to a 128-dimensional hidden space.
2. Global feature projections: five molecular descriptors and 19 functional-group counts are separately projected and fused with the atom embeddings.
3. Enrichment context: a 12-dimensional vector summarizes training-set functional-group enrichment for the 12 endpoints.
4. Three enrichment-conditioned graph layers: each layer combines edge-aware GINE message passing with four-head global self-attention. The enrichment context conditions a learned gate that fuses local and global atom representations.
5. Graph readout: global mean pooling and global max pooling are concatenated and passed through a multilayer perceptron.
6. Multitask output: 12 endpoint-specific heads return logits for binary toxicity classification.
Molecular features
Atom features (5): normalized atomic number, normalized atom degree, aromaticity, normalized formal charge, and normalized total hydrogen count.

Bond features (6): single, double, triple, and aromatic bond indicators, conjugation, and ring membership. Each chemical bond is represented as two directed edges.

Physicochemical descriptors (5): log1p(MolWt), MolLogP, number of hydrogen-bond donors, number of hydrogen-bond acceptors, and log1p(TPSA). Descriptor scaling statistics are fitted on the training partition and applied to validation and test data.

Functional groups (19): alcohol, ketone, carboxylic acid, ester, ether, primary amine, secondary amine, tertiary amine, amide, nitro, sulfide, nitrile, halogen, aromatic ring, phenol, aniline, sulfonamide, phosphate, and alkene. SMARTS-matched counts are transformed with log1p.

Enrichment profiles: for each endpoint, the active rate among compounds containing a functional group is divided by the overall active rate, with Laplace smoothing (alpha=1). Groups with fewer than 10 labeled examples are treated as unavailable; enrichment ratios are clipped to [0.25, 4] and transformed with log2. Training profiles are generated using scaffold-grouped cross-fitting; validation and test profiles use statistics derived from training data only.
Tox21 endpoints
Endpoint	Assay / pathway label
NR-AR	Androgen receptor
NR-AR-LBD	Androgen receptor ligand-binding domain
NR-AhR	Aryl hydrocarbon receptor
NR-Aromatase	Aromatase
NR-ER	Estrogen receptor
NR-ER-LBD	Estrogen receptor ligand-binding domain
NR-PPAR-gamma	Peroxisome proliferator-activated receptor gamma
SR-ARE	Antioxidant response element
SR-ATAD5	ATAD5-associated stress response
SR-HSE	Heat shock response element
SR-MMP	Mitochondrial membrane potential
SR-p53	p53-associated stress response


Dataset processing
The notebook reads an SDF file with RDKit, converts valid molecules to canonical isomeric SMILES, and normalizes SDF property names before mapping the 12 target properties.

In the saved run:
Processing step	Count
Valid SDF records	11,761
Invalid SDF records skipped	3
Unique canonical SMILES	8,041
Redundant records associated with duplicates	3,720
Duplicate SMILES groups	2,952
Conflicting molecule–endpoint label pairs	513
Unique molecules with at least one conflicting endpoint	403


Two target aggregation paradigms are built:
- Consensus: retain an endpoint label only if all observed binary labels for that molecule agree; conflicting entries become missing.
- Majority voting: choose the more frequent observed binary label; ties and unobserved endpoints remain missing.
Both versions contained 8,041 unique molecules before splitting. The notebook's saved split contains 6,433 train, 804 validation, and 804 test molecules per paradigm. Scaffold groups are assigned to only one partition. A fixed seed of 42 is used for reproducibility.

Missing labels are represented by a mask. Their placeholder values do not contribute to the binary cross-entropy loss or evaluation metrics.
Training configuration
Setting	Value
Hidden dimension	128
Graph layers	3
Global attention heads	4
Per-head dimension	32
Graph-layer dropout	0.1
Readout dropout	0.2
Batch size	16
Maximum epochs	60
Optimizer	AdamW
Initial learning rate	1e-4
Weight decay	1e-4
Learning-rate scheduler	ReduceLROnPlateau, maximize validation macro ROC-AUC; factor 0.5, patience 5, minimum LR 1e-6
Early stopping	Patience 15, minimum improvement 1e-4
Positive-class weighting	Square root of negative/positive ratio, clipped to [1, 20]
Gradient clipping	Maximum norm 5.0
Random seed	42
Mixed precision	Enabled only when CUDA is available
Checkpoint selection	Best validation macro ROC-AUC
Classification thresholds	Per-endpoint thresholds selected on validation data by Youden's index


Results from the saved notebook run
The following values are test-set macro averages across the 12 tasks. Threshold-independent metrics (ROC-AUC and PR-AUC/average precision) summarize discrimination; the other metrics use endpoint-specific thresholds selected using the validation set.
Metric	Consensus	Majority voting
Accuracy	0.7713	0.8006
Balanced accuracy	0.7415	0.7689
F1-score	0.2747	0.2969
Precision	0.1824	0.1981
Recall	0.7109	0.7323
Macro ROC-AUC	0.8396	0.8531
Macro PR-AUC (average precision)	0.4000	0.4130


Majority-voting model: per-task test discrimination
Endpoint	Labeled test samples	Positives	Negatives	ROC-AUC	PR-AUC
NR-AR	743	17	726	0.7569	0.4570
NR-AR-LBD	688	15	673	0.8589	0.4431
NR-AhR	677	60	617	0.8715	0.5952
NR-Aromatase	603	28	575	0.8954	0.3705
NR-ER	628	55	573	0.7762	0.4674
NR-ER-LBD	706	16	690	0.9264	0.4733
NR-PPAR-gamma	663	17	646	0.8655	0.1994
SR-ARE	564	78	486	0.8300	0.4616
SR-ATAD5	724	20	704	0.8580	0.2623
SR-HSE	639	41	598	0.8533	0.3983
SR-MMP	600	68	532	0.8888	0.5273
SR-p53	699	52	647	0.8560	0.3009
Macro average	—	—	—	0.8531	0.4130


The best test ROC-AUC was 0.9264 for NR-ER-LBD. The highest PR-AUC was 0.5952 for NR-AhR. PR-AUC varies with the positive-label prevalence, so it should be interpreted alongside the number of positive test examples for each task.

Interpretation note: Majority voting improved the macro averages in this saved run, but the differences are descriptive. The notebook does not provide repeated-seed or repeated-split confidence intervals, and this experiment alone cannot establish statistically reliable superiority. Precision is low for several rare endpoints, so predicted active compounds should be treated as candidates for prioritization and experimental follow-up, not as confirmed toxicants.
Requirements
The notebook was authored for Google Colab and uses Google Drive to access the SDF file. Core dependencies include:
- Python
- RDKit
- PyTorch
- PyTorch Geometric
- NumPy and pandas
- scikit-learn
- Matplotlib and seaborn
The first notebook cell installs rdkit and torch-geometric. The remaining packages may already be present in Colab; in other environments, install compatible versions of the above libraries for your Python and PyTorch/CUDA configuration.
How to run
1. Clone or download this repository and open tox21_GNNEnrichedFG.ipynb in Google Colab.
2. Place tox21_data.sdf in your Google Drive at MyDrive/GNNFG/tox21_data.sdf, or edit the DATA_PATH variable in the notebook to point to the SDF location available in your environment.
3. Run the notebook cells in order. When prompted, authorize Google Drive access.
4. The notebook builds the consensus and majority-voting datasets, creates scaffold splits, computes molecular features and enrichment profiles, trains both models, and evaluates predictions.
5. Review the generated output tables and figures. The split CSV files are written to the notebook's current working directory. Model checkpoints are saved using the configured names GNNFG21_consensus_best.pt and GNNFG21_majority_best.pt.
The SDF dataset is not embedded in this README. Ensure that you have the appropriate right to use and redistribute the dataset before committing it to a public repository.
Expected artifacts
The notebook writes or displays artifacts including:
- consensus_train.csv, consensus_validation.csv, consensus_test.csv
- majority_train.csv, majority_validation.csv, majority_test.csv
- Best model checkpoints: GNNFG21_consensus_best.pt and GNNFG21_majority_best.pt
- Training curves, validation metrics, per-task test metrics, and comparison plots
Exact output availability depends on which cells have been executed and where the notebook is run.
Reproducibility and limitations
- A fixed random seed of 42 is used, but exact bitwise reproducibility is not guaranteed across hardware, CUDA/PyTorch versions, or library configurations.
- The reported results come from one saved scaffold partition and one set of model runs.
- The README reports observed notebook metrics; it does not claim external validation or superiority to DeepTox, random forests, gradient boosting, or graph-only baselines.
- Component ablations are needed to quantify the independent contribution of descriptors, functional groups, global attention, and enrichment conditioning.
- Predictions are intended for research and prioritization. They should not be used as a substitute for experimental toxicology or regulatory assessment.
