<p align="center"> <a href="https://www.knime.com/"> <img src="https://cdn.simpleicons.org/knime" alt="KNIME logo" width="120"> </a> </p> <h1 align="center">UK Census Analysis</h1> <p align="center"><em>Built with <a href="https://www.knime.com/">KNIME Analytics Platform</a></em></p>

A data mining project applying five analytical techniques to census microdata for **England and Wales**, built as [KNIME Analytics Platform](https://www.knime.com/) workflows. Each technique has its own workflow, and the findings are written up in [`ANALYSIS DEDUCTIONS.md`](ANALYSIS%20DEDUCTIONS.md).

## Contents

| Question | Technique | Workflow | What it does |
|---|---|---|---|
| Q1 | Basic statistics | `Basic Statistics.knwf` | Summary statistics, box plot, histograms, pie chart and a crosstab |
| Q3 | Regression | `Regression.knwf` | Predicts hours worked with linear regression and a regression tree |
| Q4 | Association rule mining | `Association Rule Mining.knwf` | Finds attribute combinations that commonly occur together (Apriori) |
| Q5 | Clustering | `Clustering.knwf` | Groups people with k-Means and k-Medoids, compared by silhouette score |
| Part 1 | Classification | `Classification.knwf` | Predicts approximated social grade with a decision tree, k-NN and Naive Bayes |

The write-up of findings is in `ANALYSIS DEDUCTIONS.md`.

## Dataset

The census data holds **569,740 records, one per person**, with 21 columns and the region codes (`E12000001`–`E12000009`, `W92000004`) show that it covers England and Wales.

| Group | Columns |
|---|---|
| Geography and household | Region, Residence Type, Family Composition, Population Base |
| Person | Sex, Age, Age(cat), Marital Status, Student, Country of Birth, Health, Ethnic Group, Religion |
| Work | Economic Activity, Occupation, Industry, Hours worked per week, Hours worked per week(cat), No of hours, Approximated Social Grade |
| Identifier | Person ID |

The modelling workflows drop `Person ID`, `Age` and `Hours worked per week` and use the categorised versions instead.

## Methods and headline results

### Classification (`Classification.knwf`)
- **Target:** `Approximated Social Grade`. Rows coded "No code" are removed (569,740 → 445,637), then rows with a missing `No of hours` (→ 265,168).
- **Split:** 70% train (185,617) / 30% test (79,551), stratified on the target.
- **Models and test accuracy:**

| Model | Settings | Accuracy | Cohen's kappa |
|---|---|---|---|
| Decision Tree | Gini index, no pruning, min 2 records per node | **81.2%** | 0.743 |
| k-Nearest Neighbour | k = 3, unweighted | 69.5% | 0.581 |
| Naive Bayes | default thresholds | 47.6% | 0.328 |

### Regression (`Regression.knwf`)
- **Target:** `No of hours` (hours worked).
- **Split:** 70% train / 30% test.
- **Models compared:** Linear Regression (with constant term) and a Simple Regression Tree, each scored with Numeric Scorer.

### Clustering (`Clustering.knwf`)
- **Preparation:** missing values handled, categories converted to numbers, Z-score normalisation, 2,700-row sample.
- **Algorithms (k = 3, up to 99 iterations):** k-Means and k-Medoids (Euclidean distance).
- **Mean silhouette coefficient:** k-Means 0.175, k-Medoids 0.226.

### Association rule mining (`Association Rule Mining.knwf`)
- **Preparation:** `No of hours` binned into 4 equal-frequency bins, missing values removed (267,419 records), each attribute value turned into a yes/no item (111 items).
- **Algorithm:** Apriori (Borgelt) with minimum support 10% and a 10% threshold on rule confidence.
- **Output:** 430,623 rules, with antecedents split into separate columns for easier filtering.

### Basic statistics (`Basic Statistics.knwf`)
Descriptive statistics for the census attributes, a box plot, histograms of `No of hours` by social grade and by age group, a pie chart of economic activity, and a crosstab of sex against social grade.

## Requirements

- [KNIME Analytics Platform](https://www.knime.com/downloads) (free and open source)
- Any extensions KNIME offers to install when you import a workflow

## Getting started

1. Clone the repository:
   ```bash
   git clone https://github.com/Veckey5/UK-CENSUS-ANALYSIS.git
   ```
2. Open KNIME and choose `File → Import KNIME Workflow…`, then select a `.knwf` file.
3. The CSV Reader nodes still point to a path on the original author's computer. Open each one and set it to your copy of `Census.csv`.
4. Run the workflow with `Shift + F7` and inspect each node's output.

## Data licence

The census data is published by the Office for National Statistics and normally released under the [Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/).
