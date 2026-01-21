# Analysis scripts

## Requirements
- Python 3.x
- pandas

## Input data
- ./Cuda-Q/cudaq_issues_raw.csv
- ./qskit/github_issues.csv

## Scripts

### 01_amount_of_issues.py
Purpose: Create dataset overview statistics (Table 1).
Inputs:
- cudaq_issues_raw.csv
- github_issues.csv
Processing:
- Normalizes column names (lowercase, strip spaces)
- Removes accidental duplicated header rows (caused by appended exports)
- Deduplicates by IssueID (keeps last occurrence)
- Parses CreatedAt timestamps
- Normalizes Status (open/closed)
- Filters Qiskit issues to gpu_relevant == True (accepts variants like True/1/yes/X)
Outputs:
- Console summary (N total, N per repo, createdAt range, open/closed counts)
- table1_dataset_overview.csv


## 02_basic.py 

**Purpose:**  
Compute the core descriptive statistics used in the paper (Step C), based on the manually coded GitHub issue datasets.

**Inputs (CSV):**
- `./Cuda-Q/cudaq_issues_raw.csv`
- `./qskit/github_issues.csv`

**Inclusion rules:**
- CUDA-Q: include all issues.
- Qiskit Aer: include only GPU-relevant issues (filtered via `gpu_relevant`, e.g., marked with `X`).

**Processing (high-level):**
- Normalizes column names (strip/BOM removal, lowercase, spaces → underscores).
- Removes accidental embedded header rows (e.g., `IssueID == "IssueID"`).
- Deduplicates by issue ID (keeps the last occurrence if files were appended).
- Creates a repo-unique issue key (`project#issueid`) to avoid collisions across repositories.
- Cleans label strings (strip/uppercase for CTClass).
- Normalizes B-subtypes to `B1`/`B2` (and handles missing values if present).
- Optionally computes 95% Wilson confidence intervals for proportions (if `statsmodels` is installed).

**Outputs (CSV):**
- `c_ctclass_overall.csv`, `c_ctclass_by_project.csv`
- `c_stacklayer_overall.csv`, `c_stacklayer_by_project.csv`
- `c_bugtype_overall.csv`, `c_bugtype_by_project.csv`
- `c_b_subtype_overall.csv`, `c_b_subtype_by_project.csv`

**How to run:**
```bash
python 02_basic.py
```

### 03_cross.py

**Purpose:**  
Generate the main cross-tabulations used for the paper’s “story” analysis (Step D), i.e., how CTClass varies across stack layers, bug types, and projects.

**Inputs (CSV):**
- `./Cuda-Q/cudaq_issues_raw.csv`
- `./qskit/github_issues.csv`

**Inclusion rules:**
- CUDA-Q: include all issues.
- Qiskit Aer: include only GPU-relevant issues (`gpu_relevant == "X"`).

**Processing (high-level):**
- Normalizes column names (strip, lowercase, BOM removal, spaces → underscores).
- Removes embedded header rows (checks `issueid == "IssueID"` and `project == "Project"`).
- Deduplicates by `(project, issueid)` (keeps last occurrence).
- Creates a repo-unique issue key `uid = project#issueid`.
- Cleans labels (`ctclass` uppercased; `stacklayer`, `bugtype`, `project` stripped).
- Produces cross-tabs as **counts** and **row-wise percentages** (percentages sum to ~100 per row, rounding to 1 decimal).

**Outputs (CSV):**
- StackLayer × CTClass:
  - `d_layer_x_ctclass_overall_counts.csv`
  - `d_layer_x_ctclass_overall_pct.csv`
  - `d_layer_x_ctclass_by_project_counts.csv`
  - `d_layer_x_ctclass_by_project_pct.csv`
- BugType × CTClass:
  - `d_bugtype_x_ctclass_overall_counts.csv`
  - `d_bugtype_x_ctclass_overall_pct.csv`
  - `d_bugtype_x_ctclass_by_project_counts.csv`
  - `d_bugtype_x_ctclass_by_project_pct.csv`
- Project × CTClass:
  - `d_project_x_ctclass_overall_counts.csv`
  - `d_project_x_ctclass_overall_pct.csv`
- Label audit (sanity check):
  - `d_audit_unique_labels.csv` (unique label counts for stacklayer/bugtype overall and per project)

**How to run:**
```bash
python 03_cross.py
```

### 04.py — Effect sizes for key cross-tabs (Step D, optional)

**Purpose:**  
Compute effect sizes (Cramér’s V) and significance estimates for the three main cross-tabulations used in the paper:
- Project × CTClass  
- StackLayer × CTClass  
- BugType × CTClass

**Inputs (CSV):**
- `./Cuda-Q/cudaq_issues_raw.csv`
- `./qskit/github_issues.csv`

**Inclusion rules:**
- CUDA-Q: include all issues.
- Qiskit Aer: include only GPU-relevant issues (`gpu_relevant == "X"`).

**Processing (high-level):**
- Normalizes column names and removes embedded header rows.
- Deduplicates by `(project, issueid)` and creates a repo-unique key (`project#issueid`).
- Builds the three contingency tables and computes:
  - χ² statistic (uncorrected) and expected cell counts
  - **Cramér’s V** as effect size
  - Permutation p-value (5,000 permutations, seed=0) when expected counts are small (or when SciPy is not available)
  - Fisher’s exact test for 2×2 tables (only when applicable)

**Output (CSV):**
- `e_effect_sizes.csv` (one row per test, incl. `cramers_v`, `p_perm`, and `min_expected`)

**How to run:**
```bash
python 04.py
```

### make_fig1.py — Figure 1 (CTClass distribution)

**Purpose:**  
Generate *Figure 1* as a publication-ready 2-panel plot:
- (A) Overall CTClass distribution (counts + percentages)
- (B) CTClass distribution by project as 100% stacked bars (percentages + per-project N)

**Inputs (CSV):**  
(Produced by prior analysis steps)
- `c_ctclass_overall.csv` (from Step C / `02.py`)
- `d_project_x_ctclass_overall_pct.csv` (from Step D / `03_cross_tabs.py`)
- `d_project_x_ctclass_overall_counts.csv` (from Step D / `03_cross_tabs.py`)

**Processing (high-level):**
- Validates totals (overall count matches `total` column within tolerance).
- Validates that per-project percentages sum to ~100%.
- Enforces fixed CTClass order A/B/C.
- Applies consistent color mapping and panel labels.

**Outputs:**
- `figures/fig1_ctclass.pdf`
- `figures/fig1_ctclass.png` (300 dpi)

**How to run:**
```bash
python make_fig1.py
```

### make_fig2.py — Figure 2 (CTClass distribution by stack layer)

**Purpose:**  
Generate *Figure 2* as a publication-ready plot showing the **CTClass distribution per stack layer** as **100% stacked horizontal bars** (percentages + per-layer N). Layers are ordered to highlight where **runtime-only (C)** dominates.

**Inputs (CSV):**  
(Produced by prior analysis steps / cross-tabs)
- `d_layer_x_ctclass_overall_pct.csv`
- `d_layer_x_ctclass_overall_counts.csv`

**Processing (high-level):**
- Validates that A+B+C percentages per layer sum to ~100% (tolerance 99.8–100.2).
- Computes total N per layer from counts and appends it to y-axis labels (`Layer (N=...)`).
- Sorts layers by **C share descending** (C-heavy layers on top).
- Uses fixed CTClass stacking order **A → B → C** and consistent color mapping.
- Adds in-bar percentage labels only for segments ≥ 10% (C labels in white for contrast).
- Applies publication formatting (gridlines, spines, legend below plot).

**Outputs:**
- `figures/fig2_layer_x_ctclass.pdf`
- `figures/fig2_layer_x_ctclass.png` (300 dpi)

**How to run:**
```bash
python make_fig2.py
```



### make_fig3.py — Figure 3 (CTClass distribution by bug type)

**Purpose:**  
Generate *Figure 3* as a publication-ready plot showing the **CTClass distribution per bug type** as **100% stacked horizontal bars** (percentages + per-bugtype N). Bug types are ordered to highlight where **potentially pre-execution preventable (B)** is strongest.

**Inputs (CSV):**  
(Produced by prior analysis steps / cross-tabs)
- `d_bugtype_x_ctclass_overall_pct.csv`
- `d_bugtype_x_ctclass_overall_counts.csv`

**Processing (high-level):**
- Validates that A+B+C percentages per bug type sum to ~100% (tolerance 99.8–100.2).
- Computes total N per bug type from counts and appends it to y-axis labels (`BugType (N=...)`).
- Sorts bug types by **B share descending** (B-heavy types on top).
- Uses fixed CTClass stacking order **A → B → C** and consistent color mapping.
- Adds in-bar percentage labels only for segments ≥ 10% (C labels in white for contrast).
- Applies publication formatting (gridlines, spines, legend below plot).

**Outputs:**
- `figures/fig3_bugtype_x_ctclass.pdf`
- `figures/fig3_bugtype_x_ctclass.png` (300 dpi)

**How to run:**
```bash
python make_fig3.py
```

### make_fig4.py — Figure 4 (B1 vs B2 subtype distribution)

**Purpose:**  
Generate *Figure 4* as a publication-ready **2-panel plot** for **CTClass=B subtypes**:
- (A) Overall distribution of **B1 vs B2**
- (B) Distribution of **B1 vs B2 by project** (with per-project N)

This figure separates **metadata/config constraints (B1)** vs **contracts/typestate-like constraints (B2)**.

**Inputs (CSV):**  
(Produced by prior analysis steps)
- `c_b_subtype_overall.csv`
- `c_b_subtype_by_project.csv`

**Processing (high-level):**
- Robustly supports **WIDE** (columns `B1`, `B2`) and **LONG** formats (detects subtype column names like `ctsubtype_norm`, `b_subtype`, etc.).
- Auto-detects whether inputs are **counts or percentages**:
  - If B1+B2 ≈ 100 → treat as percentages
  - Else treat as counts and convert to percentages; derives N
- Validates that B1+B2 sums to ~100% overall and per project (tolerance 99.8–100.2).
- Orders projects with **CUDA-Q first**, then **Qiskit Aer (GPU)** (fallback to original order if names don’t match).
- Adds in-bar percentage labels only for segments ≥ 10%.
- Applies publication formatting and panel labels (A)/(B), plus legend below the full figure.

**Outputs:**
- `figures/fig4_b_subtype.pdf`
- `figures/fig4_b_subtype.png` (300 dpi)

**How to run:**
```bash
python make_fig4.py
```
