# PBMC Immune Profiling Pipeline

A reproducible Python and SQLite pipeline for analyzing immune-cell count data: data validation and cleaning, a normalized relational database, relative cell-population frequencies, responder vs. nonresponder statistics, baseline cohort summaries, and an interactive Streamlit dashboard.

The whole workflow runs from a Makefile.

**Live dashboard:** https://teikotechnical-kfayrucj3gzdoa22b5wvwg.streamlit.app/

## What it does

- Validates and cleans the input data
- Loads it into a normalized SQLite database
- Computes relative immune-cell frequencies per sample
- Compares responders and nonresponders with a nonparametric test and multiple-testing correction
- Summarizes the baseline melanoma cohort
- Presents the results in an interactive dashboard (Streamlit + Plotly)

## Project structure

```
.
├── data/
│   ├── cell-count.csv
│   └── cell-count-cleaned.csv
│
├── output/
│   ├── cleaning_report.txt
│   ├── relative_frequencies.csv
│   ├── part3_data.csv
│   ├── part3_statistics.csv
│   ├── part4_samples_by_project.csv
│   ├── part4_subjects_by_response.csv
│   ├── part4_subjects_by_sex.csv
│   └── part4_average_b_cells.csv
│
├── clean_data.py
├── load_data.py
├── analysis.py
├── dashboard.py
├── schema.sql
├── Makefile
├── requirements.txt
└── README.md
```

| File | Description |
|---|---|
| `clean_data.py` | Validates and cleans the input CSV |
| `schema.sql` | Defines the normalized SQLite schema |
| `load_data.py` | Creates and populates the SQLite database |
| `analysis.py` | Runs the frequency, statistical and baseline analyses |
| `dashboard.py` | Runs the interactive Streamlit dashboard |
| `Makefile` | Reproducible setup, pipeline and dashboard commands |
| `requirements.txt` | Python dependencies |

## Requirements

Python 3, SQLite, pandas, NumPy, SciPy, statsmodels, Plotly and Streamlit. The project can be run locally or in GitHub Codespaces.

## Setup

```bash
make setup
```

## Running the pipeline

```bash
make pipeline
```

This runs, in order:

```
clean_data.py  ->  load_data.py  ->  analysis.py
```

### Step 1: Data cleaning

`clean_data.py` validates the input dataset and writes `data/cell-count-cleaned.csv`. It checks:

- Required columns
- Missing values
- Categorical values
- Numeric values
- Negative cell counts
- Invalid ages and treatment timepoints
- Duplicate sample identifiers
- Completely duplicated rows

Missing response values are kept, because response is an outcome variable and should not be imputed or inferred. A summary is written to `output/cleaning_report.txt`.

### Step 2: Database loading

`load_data.py` builds `cell-count.db` from the cleaned dataset each time the pipeline runs. The schema has three tables:

```
subjects
   |
   +--< samples
           |
           +--< cell_counts
```

- **subjects:** `subject`, `project`, `condition`, `age`, `sex`
- **samples:** `sample`, `subject`, `treatment`, `response`, `sample_type`, `time_from_treatment_start`
- **cell_counts** (long format): `sample`, `population`, `count`

This separates subject-level metadata, sample-level metadata and cell-population measurements, with foreign-key relationships between them.

### Step 3: Analysis

`analysis.py` generates the results below and saves them under `output/`.

## Analysis

### Relative cell frequencies

For each sample, the five immune-cell populations are summed to a total count (B cell + CD8 T cell + CD4 T cell + NK cell + monocyte), and each population's relative frequency is

```
percentage = population count / total count x 100
```

The result is saved to `output/relative_frequencies.csv` (columns: `sample`, `total_count`, `population`, `count`, `percentage`) and covers:

- 10,500 samples
- 52,500 population-level records
- 5 immune-cell populations per sample

### Responders vs. nonresponders

The comparison is restricted to:

- Condition: melanoma
- Sample type: PBMC
- Treatment: miraclib
- Response: yes or no

This gives 1,968 samples from 656 subjects, with 3 timepoints per subject (0, 7 and 14), including 993 responder samples and 975 nonresponder samples.

Relative frequencies are compared separately for B cells, CD4 T cells, CD8 T cells, NK cells and monocytes. A two-sided Mann-Whitney U test is used for each population, and because five populations are tested, p-values are adjusted with the Benjamini-Hochberg false discovery rate (FDR) procedure. Results are saved to `output/part3_statistics.csv`.

| Population | Responder median (%) | Nonresponder median (%) | Raw p-value | Adjusted p-value |
|---|---|---|---|---|
| B cell | 9.43 | 9.79 | 0.0557 | 0.1393 |
| CD4 T cell | 30.22 | 29.66 | 0.0133 | 0.0667 |
| CD8 T cell | 24.73 | 24.60 | 0.6391 | 0.6391 |
| Monocyte | 19.61 | 19.94 | 0.1632 | 0.2039 |
| NK cell | 14.51 | 14.80 | 0.1211 | 0.2018 |

At an FDR threshold of 0.05, none of the five populations is significant after multiple-testing correction. CD4 T cells have a raw p-value below 0.05, but the adjusted p-value is 0.0667.

**Repeated measurements.** Each of the 656 subjects contributes three samples. The comparison above treats samples as the unit of analysis. For subject-level inference that accounts for the repeated measurements, a mixed-effects or other repeated-measures model would be more appropriate.

### Baseline melanoma summary

Baseline is defined as `time_from_treatment_start = 0`. The baseline cohort is restricted to melanoma, PBMC samples and miraclib treatment, which gives 656 samples.

| Project | Samples |
|---|---|
| prj1 | 384 |
| prj3 | 272 |

| Response | Subjects |
|---|---|
| No | 325 |
| Yes | 331 |

| Sex | Subjects |
|---|---|
| Female | 312 |
| Male | 344 |

### Average B-cell count

For melanoma, male, responder (response = yes) subjects at timepoint 0, the average B-cell count is **10206.15**. This calculation includes all sample types and treatments. It is saved to `output/part4_average_b_cells.csv`.

## Interactive dashboard

```bash
make dashboard
```

The dashboard (Streamlit + Plotly) shows:

- Relative cell frequencies
- Responder vs. nonresponder distributions
- Statistical results
- Baseline sample summaries
- The average B-cell calculation

The responder vs. nonresponder view can be filtered by cell population and timepoint. The statistics it shows correspond to the overall sample-level comparison across all three timepoints.

## Reproducibility

```bash
make setup
make pipeline
make dashboard
```

The database is rebuilt from the cleaned CSV on every `make pipeline`, so downstream results always reflect the current cleaned input.

## Output files

| File | Description |
|---|---|
| `cleaning_report.txt` | Data validation and cleaning summary |
| `relative_frequencies.csv` | Relative frequencies of each cell population |
| `part3_data.csv` | Dataset for the responder vs. nonresponder comparison |
| `part3_statistics.csv` | Mann-Whitney U and FDR results |
| `part4_samples_by_project.csv` | Baseline samples by project |
| `part4_subjects_by_response.csv` | Baseline subjects by response |
| `part4_subjects_by_sex.csv` | Baseline subjects by sex |
| `part4_average_b_cells.csv` | Average B-cell count for the baseline subgroup |

## Notes

- Missing response values are preserved in the cleaned dataset and not imputed. Response-specific analyses exclude samples without a response.
- Statistical conclusions come from sample-level analyses and should be interpreted in light of repeated measurements from the same subjects.