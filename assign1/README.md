# International Football Match Performance Lakehouse

**CMPS 360 – Assignment 1: Hands-on Data Engineering**

## 1. Project overview

This project uses Databricks to build a small Lakehouse for analyzing historical men's international football match results. It follows the **Bronze → Silver → Gold** medallion architecture and uses **Databricks SQL**, **Delta Lake**, and a simple **star schema** to organize the data. The project also includes three simulated incremental data batches based on dates, data-quality checks, a quarantine area for problematic records, and four analytical queries that run on the Gold layer.

The central Gold fact table is **`gold.fact_matches`**, with one row per unique international match. The project focuses on match scoring trends, neutral versus non-neutral venues, tournament activity, and national-team victories.

## 2. Dataset and source

- **Dataset:** International Football Results, historical data beginning in 1872
- **Source / maintainer:** `martj42`, *international_results*
- **Dataset URL:** https://github.com/martj42/international_results / https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017
- **Dataset snapshot used for this assignment:** downloaded/accessed on **2026-10-10**
- **File format:** CSV (four files)

| Source file | Rows in assignment snapshot | Columns | Meaning |
|---|---:|---:|---|
| `results.csv` | 49,945 | 9 | Match dates, participants, scores, competition, location, neutral venue |
| `goalscorers.csv` | 47,914 | 8 | Recorded goals, players, minutes, own-goal and penalty flags |
| `shootouts.csv` | 700 | 5 | Penalty shootout winners and first-shooter information |
| `former_names.csv` | 36 | 4 | Historical team-name mappings and date ranges |

Download each CSV from the source repository's file list. For **exact reproducibility**, retain the four CSVs from the **same snapshot** used for this project in a `data/` folder in the submission repository (or provide a fixed version/commit). The upstream GitHub repository may be updated, so downloading its latest files later may produce different totals.

## 3. Software and repository contents

**Environment:** The project runs in a Databricks notebook using SQL and Python cells. It requires SQL-compatible compute, Delta Lake, and permission to create schemas, tables, and a Unity Catalog volume. The submitted SQL pipeline runs entirely in Databricks, so Google Colab and extra Python packages are not needed.

Where to find files?

- `football_analysis.ipynb` — The actual notebook with all the code and results.
- `match_performance_star_schema.png` — Gold star-schema diagram
- `results.csv`, `goalscorers.csv`, `shootouts.csv`, `former_names.csv` — The four CSV files used in the analysis contained in the "dataset_snapshots" folder.
 

## 4. Databricks setup

1. Open a Databricks workspace and attach compute capable of executing SQL and creating Delta tables/volumes.
2. Import the exported `football_analysis.ipynb` notebook into the Databricks workspace.
3. Obtain the four **same-snapshot** CSV files. Upload them to **one accessible Databricks directory**. This notebook's initial exploration cell expects all four CSVs to be in the same directory.
4. In the initial Python exploration cell, set `results_path` to the **actual uploaded path** of `results.csv`. The notebook derives the other three CSV paths from that directory.
5. Run the one-time setup cells to create the `bronze`, `silver`, `gold`, and `quarantine` schemas plus the Bronze `landing_zone` volume, then prepare the date-based input folders.
6. Confirm that the `volume_path` widget matches the landing-zone path reported by setup: `/Volumes/<your_catalog>/bronze/landing_zone`. The workspace/catalog name can differ from the original development environment.
7. Create the Bronze, Silver, Quarantine, and Gold Delta tables using the notebook's table-definition cells. Run setup **only when initializing a fresh workspace**, not before every batch.

**Notebook path note:** Databricks Volumes use catalog-qualified `/Volumes/...` paths. The current notebook includes a user-configured source upload path and a Volume widget; these must be configured for the new workspace. For the assignment's strict **relative-path / one-click run** requirement, the notebook should additionally be refactored to resolve its source files from the checked-out project and execute all batches in sequence without manual jumps. The README alone does **not** make the existing notebook fully `Run All` reproducible.

## 5. Ingestion strategy

- **Initial loading:** Historical files are split into three **simulated arrival batches**. The first batch acts as an initial historical load; the later two demonstrate incremental loading.
- **Incremental parameter/watermark:** `run_date` identifies the **arrival batch date**, not the actual football match date. Each run processes only files associated with that date.
- **Batch vs streaming:** Batch ingestion is appropriate because this is a static CSV snapshot; real-time streaming is not needed.
- **Pull vs push:** The notebook initiates reading files uploaded/downloaded from a public repository (**pull**).
- **Frequency:** On demand for this assignment; a real refresh could be scheduled after new source updates.
- **Reruns/recovery:** Bronze and Silver/Quarantine replace rows belonging to the selected load date; Gold uses `MERGE INTO` to update or insert by generated identifier. Audit and quality checks are rerun after each batch.

| Simulated `run_date` | Match-date range represented | Source files |
|---|---|---|
| `2026-10-07` | Before 2000 | All four CSVs |
| `2026-10-08` | 2000–2014 | `results`, `goalscorers`, `shootouts` |
| `2026-10-09` | 2015 onward | `results`, `goalscorers`, `shootouts` |

These are **simulation dates**, not dates on which the historical matches were played or necessarily when the dataset was published.

## 6. Execution order in the existing notebook

**Current implementation requires controlled, step-by-step execution; do not assume that clicking `Run All` will automatically loop through all three dates.** The chronological steps are:

1. **One-time setup:** Dataset inspection/documentation → Lakehouse schemas/landing volume → split the CSV snapshot into the three arrival-date folders → create Bronze/Silver/Quarantine/Gold tables.
2. **Batch 1 (`run_date = 2026-10-07`):** Bronze ingest all four files and audit → recreate cleaned and match-lookup staging views → validate and load Silver/Quarantine → merge the four Gold dimensions → merge `gold.fact_matches`.
3. **Batch 2 (`run_date = 2026-10-08`):** Bronze ingest only `results`, `goalscorers`, `shootouts` and audit → recreate staging views for that date → load Silver/Quarantine → merge Gold dimensions → merge `gold.fact_matches`.
4. **Batch 3 (`run_date = 2026-10-09`):** Repeat the Batch 2 steps using the third date.
5. Run the final Bronze/Silver/Quarantine reconciliation, fact-key and dimension-link checks, and the four SQL analyses. Review the included executed result tables, interpretations, and charts.

The `former_names.csv` reference file is loaded only during Batch 1. After switching batches, always verify the active `run_date` widget. Session-local SQL variables and `TEMPORARY VIEW`s can disappear when compute/session restarts, so recreate the staging views before any Silver load. Rerunning only `SELECT` verification cells is safe and does not alter tables.

## 7. Bronze, Silver, Quarantine, and Gold design

- **Bronze:** Preserves raw CSV strings and stores `_load_date`, `_load_id`, `_source_file`, and `_ingested_at`. `bronze.load_audit` records source and loaded row counts per batch.
- **Silver:** Converts valid dates/scores/flags, trims text, generates match and goal identifiers, checks values and relationships, and isolates rejected rows in Quarantine with a `rejection_reason`.
- **Gold:** One central fact table, **`fact_matches`**, with `match_id` as the logical key; measures include `total_goals`, `match_count`, `home_win`, `away_win`, and `draw`.
- **Dimensions:** `dim_date` (`date`), `dim_team` (`team_name`, referenced for both home and away), `dim_tournament` (`tournament`), and `dim_venue` (`venue_id`). Keys are **logical PK/FK relationships** validated by SQL; the table-creation statements do not enforce PK/FK constraints.

See [the star-schema diagram](match_performance_star_schema.png).

## 8. Four analytical questions (Gold tables only)

1. How did **average goals per international match** change across decades?
2. How does the **home-team win percentage** compare at neutral and non-neutral venues?
3. Which **10 non-friendly tournaments** hosted the most matches since 2000?
4. Which **10 national teams** recorded the most score-based victories since 2000?

The notebook includes a labeled SQL query, its result, a short interpretation, and a visualization for each question.

## 9. Quality checks and expected totals

| Source | Bronze rows | Silver rows | Quarantined rows |
|---|---:|---:|---:|
| Results | 49,945 | 49,945 | 0 |
| Goalscorers | 47,914 | 47,914 | 0 |
| Shootouts | 700 | 699 | 1 |
| Former names | 36 | 36 | 0 |

Expected **Gold `fact_matches` row count: 49,945**. SQL checks examine unique/non-null `match_id` values, match-outcome measures, and references to all four dimensions. One shootout associated with **Saare County versus Åland Islands (2011-06-29)** could not be matched uniquely and was quarantined. Some source goal records have identical-looking attributes; they were preserved where it was unsafe to determine whether they were truly duplicates.

## 10. Known limitations

- Coverage, historical definitions, and the number of matches in the upstream source may change over time. Results depend on the specific CSV snapshot used.
- Counting wins does not adjust for teams playing different numbers of matches. A home-team designation is not by itself proof of a causal home advantage.
- The match identifier is derived from match attributes rather than a stable source-issued match ID; later corrections to those attributes can make change detection harder.
- The three batches are a **simulation** based on historical match dates, not a real sequence of source updates.
- The notebook **currently relies on manual batch execution, a source-file upload path, and catalog-qualified Volume paths**. It needs a final end-to-end execution/relative-path pass to fully satisfy the assignment's reproducibility requirements.

## 11. Acknowledgments and academic integrity

- **Dataset:** `martj42/international_results`, https://github.com/martj42/international_results / https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017 . Dataset rights and attribution are governed by the upstream repository's license (CC0).

- **Course example:** The course's supplied Databricks medallion architecture example was used as a guide to setup → Bronze → Silver/Quarantine → Gold design and SQL structure. The retail domain and its specific tables were not used for the football analysis.

- **AI assistance:** ChatGPT (OpenAI) was used to help plan the workflow, adapt example SQL queries to the football dataset, write parts of the README and explanations, and fix notebook errors. The project author is responsible for reviewing and understanding the work, running and checking the code independently, verifying the results and conclusions, and following the course's academic integrity guidelines.
