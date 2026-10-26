<h1 align="left">ANALYSIS DEDUCTIONS</h1>

In this report I analyse census microdata for England and Wales using five data mining techniques in KNIME. The dataset (`Census.csv`) holds 569,740 person records with 21 attributes covering geography, household, personal characteristics and work.

I built one workflow for each technique. For each one I explain what every file and step does, report the results I obtained, and state what I conclude. The project overview and setup instructions are in the [README](README.md).

## Table of contents

- [Results at a glance](#results-at-a-glance)
- [Basic statistics](#1-basic-statistics-basic-statisticsknwf)
- [Regression](#2-regression-regressionknwf)
- [Association rule mining](#3-association-rule-mining-association-rule-miningknwf)
- [Clustering](#4-clustering-clusteringknwf)
- [Classification](#5-classification-classificationknwf)
- [Overall conclusions](#overall-conclusions)
- [Limitations and next steps](#limitations-and-next-steps)

## Results at a glance

| Workflow | Question | Technique | Headline result |
|---|---|---|---|
| `Basic Statistics.knwf` | Q1 | Descriptive statistics and charts | Dataset profiled; figures to be added |
| `Regression.knwf` | Q3 | Linear regression and regression tree | Scores to be added |
| `Association Rule Mining.knwf` | Q4 | Apriori | 430,623 rules at 10% support |
| `Clustering.knwf` | Q5 | k-Means and k-Medoids | Silhouette 0.175 and 0.226 (weak separation) |
| `Classification.knwf` | Part 1 | Decision tree, k-NN, Naive Bayes | Decision tree best at 81.2% accuracy |

---

## 1. Basic statistics (`Basic Statistics.knwf`)

### What this workflow does

I used this workflow to get to know the data before modelling anything. It has two branches, each starting from its own reader.

**Branch 1: statistics, box plot and charts**
1. **CSV Reader** loads `Census.csv`.
2. **String To Number** converts the numeric columns, such as `No of hours`, from text into numbers so they can be summarised.
3. **Statistics** calculates summary statistics for the census attributes.
4. **Column Filter** keeps only `No of hours`, which feeds a **Box Plot** showing the spread, median and outliers of hours worked.
5. **Color Manager** nodes colour the data by category: by `Approximated Social Grade` for an **Interactive Histogram** of hours worked, and by `Economic Activity` for an **Interactive Pie Chart** showing how the population divides across economic activities.

**Branch 2: relationships**
1. A second **CSV Reader** and **String To Number** prepare the data in the same way.
2. **Crosstab** compares `Sex` against `Approximated Social Grade`, showing how the two combine.
3. A **Color Manager** colours by `Age(cat)`, which feeds a second **Interactive Histogram** of hours worked by age group.

### What I found

- The dataset is very large (569,740 records) and mostly categorical, so counts, proportions and cross-tabulations are my main descriptive tools. Means and medians only make sense for the numeric columns (`Age`, `Hours worked per week` and `No of hours`).
- Social grade, age group and economic activity are the lenses I used to look at working hours, because they are the groupings most likely to explain differences in them.
- The age histogram only draws the first 400,000 of the 569,740 rows. If the file is ordered in some way, for example by region, that view may not represent everyone, so I treat it as indicative rather than exact.
---

## 2. Regression (`Regression.knwf`)

### What this workflow does

I used this workflow to predict `No of hours` (hours worked) from the other attributes. It contains two parallel branches, one per model, built from the same steps.

1. **CSV Reader** loads the census file.
2. **Column Filter** keeps the 18 useful attributes and drops `Person ID`, `Age` and `Hours worked per week`, because the categorised versions carry the same information.
3. **Missing Value** handles records with gaps.
4. **String To Number** converts the remaining text columns to numbers so the models can use them.
5. **Partitioning** splits the data 70% for training and 30% for testing.
6. The two models differ at this point:
   - **Linear Regression Learner** fits a straight-line model with a constant term, and **Regression Predictor** applies it to the test data.
   - **Simple Regression Tree Learner** builds a tree of splits, and **Simple Regression Tree Predictor** applies it to the test data.
7. **Numeric Scorer** measures how close each model's predictions are to the real hours worked.

### What I found

- I evaluated both models on held-out test data, so the scores show how well they predict people they have not seen.
- The two models make different assumptions. Linear regression assumes each attribute shifts hours by a fixed amount, while the tree can capture interactions, such as occupation mattering differently at different ages.
- Most predictors are categories turned into numbers. For a linear model that implies an order and equal spacing between categories that does not really exist, so I am cautious about reading the coefficients literally.

---

## 3. Association rule mining (`Association Rule Mining.knwf`)

### What this workflow does

I used this workflow to find combinations of attributes that tend to occur together, for example a type of occupation that often goes with a certain number of hours. The steps run in this order.

1. **CSV Reader** loads the census file.
2. **Missing Value** removes any record with a missing value, leaving 267,419 records.
3. **String To Number** converts `No of hours` into a number so it can be grouped.
4. **Auto-Binner** sorts `No of hours` into four equal-sized bins (low to high hours), turning it into a category.
5. **Column Filter** keeps only the attributes I want to mine.
6. **One to Many** turns each attribute value into its own yes/no column, giving 111 items in total.
7. **Create Bit Vector** packs those yes/no columns into one record per person, which is the format the rule learner needs.
8. **Association Rule Learner (Borgelt)** runs the Apriori algorithm with a minimum support of 10% and a confidence threshold of 10%.
9. **Split Collection Column** splits each rule's "if" part (the antecedent) into separate columns so the rules are easier to read and filter.

### What I found

- My thresholds were permissive, and the workflow produced **430,623 rules**. That is far too many to read, and many will be trivial or repeat each other.
- A support of 10% means every rule rests on a combination found in at least about 26,700 people, so the rules describe common patterns rather than rare ones.
- Confidence alone can flatter rules whose outcome is simply very common. Ranking by **lift** (how much more often a pattern occurs than chance would predict) is the best way to find the interesting rules.
- Some attributes are derived from one another, such as `Age` and `Age(cat)`, and can produce rules that are true by construction. I would exclude these when interpreting the results.

---

## 4. Clustering (`Clustering.knwf`)

### What this workflow does

I used this workflow to see whether people fall into natural groups. I ran two clustering algorithms side by side and compared them.

**Branch 1: k-Means**
1. **CSV Reader** loads the census file.
2. **Column Filter** keeps the 18 useful attributes.
3. **Category To Number** converts categories into numbers so distances can be calculated.
4. **Missing Value** handles gaps.
5. **Row Sampling** draws a random sample of 2,700 records to keep the run fast.
6. **Normalizer** applies a Z-score so no attribute dominates just because its numbers are larger.
7. **k-Means** groups the records into 3 clusters, with up to 99 iterations.
8. **Silhouette Coefficient** scores how well separated the clusters are.

**Branch 2: k-Medoids**
1. The same loading, filtering, converting, missing-value, sampling and normalising steps are repeated.
2. **Distance Matrix Calculate** measures the Euclidean distance between every pair of sampled records.
3. **k-Medoids** groups the records into 3 clusters using that matrix, with up to 99 iterations.
4. **Silhouette Coefficient** scores this result in the same way.

### What I found

| Algorithm | k | Mean silhouette |
|---|---|---|
| k-Means | 3 | 0.175 |
| k-Medoids | 3 | 0.226 |

- Both scores are low. A silhouette close to 1 means tight, well-separated clusters, and values around 0.2 mean the groups overlap considerably. In my data, people do not split cleanly into three distinct types.
- k-Medoids scored higher than k-Means. A plausible reason is that medoids are real records and are less pulled around by outliers, which suits coded categorical data. The gap is modest (about 0.05) and each branch used its own random sample, so I treat it as a tendency rather than proof.
- I fixed k at 3 rather than tuning it. Testing other values and choosing the best silhouette would give a stronger conclusion.
- Converting categories into numbers and measuring Euclidean distance treats arbitrary codes as if they were measurements. Methods built for categorical data, such as k-modes or Gower distance, would probably separate the groups better.

---

## 5. Classification (`Classification.knwf`)

### What this workflow does

I used this workflow to predict a person's `Approximated Social Grade` from their other attributes. It has three parallel branches, one for each classifier, and each one follows the same preparation steps so the comparison is fair.

1. **CSV Reader** loads the census file.
2. **Row Filter** removes records where the social grade is "No code", because there is nothing to predict for them (569,740 → 445,637 records).
3. **Column Filter** keeps the 18 useful attributes and drops `Person ID`, `Age` and `Hours worked per week`.
4. **Category To Number** and **String To Number** convert the attributes into numbers.
5. **Missing Value** removes records with no `No of hours` (→ 265,168 records).
6. **Normalizer** rescales the attributes to a 0–1 range so none dominates.
7. **Partitioning** splits the data 70% for training (185,617 records) and 30% for testing (79,551 records), stratified so every social grade is represented in both parts.
8. The branches then differ:
   - **Decision Tree Learner** builds a tree using the Gini index, with no pruning and a minimum of 2 records per node. **Color Manager** and **Color Appender** attach colours to the grades, and **Decision Tree Predictor** classifies the test records.
   - **K Nearest Neighbor** classifies each test record by looking at its 3 nearest training records.
   - **Naive Bayes Learner** and **Naive Bayes Predictor** classify using probabilities calculated from the training data.
9. A **Scorer** at the end of each branch compares predictions with the real grades and reports accuracy and Cohen's kappa.

### What I found

| Model | Correct | Wrong | Accuracy | Cohen's kappa |
|---|---|---|---|---|
| Decision Tree | 64,559 | 14,992 | **81.2%** | 0.743 |
| k-Nearest Neighbour (k = 3) | 55,268 | 24,283 | 69.5% | 0.581 |
| Naive Bayes | 37,897 | 41,654 | 47.6% | 0.328 |

- The **decision tree is clearly the best model**. It beats k-NN by about 12 percentage points and Naive Bayes by about 33. Its kappa of 0.743 shows substantial agreement beyond chance, compared with moderate for k-NN and only fair for Naive Bayes.
- I think the tree does well because the approximated social grade is derived largely from occupation and related work characteristics, so a model that can split on `Occupation` and `Economic Activity` recovers much of it. This means the 81% partly reflects how the target is built, and I would not read it as social grade being predictable from unrelated traits.
- The tree used no pruning, which normally invites overfitting. However, the 81% comes from unseen test data, so it held up well. Pruning would still give a simpler tree that is easier to explain.
- **Naive Bayes** did worst. It assumes the attributes are independent given the grade, but census attributes are strongly related (occupation, industry and economic activity move together). It also modelled numeric category codes as if they followed a bell curve, which does not suit coded labels.
- **k-NN** suffers from the same coding problem, because distances between arbitrary category codes are not meaningful, and using only 3 neighbours makes it sensitive to noise.
- Accuracy alone hides which grades are being confused. I would check each Scorer's confusion matrix to see whether errors fall between neighbouring grades (more forgivable) or between opposite ends.

---

## Overall conclusions

1. **Social grade can be predicted well from work and household attributes.** The decision tree reached about 81% accuracy on unseen data, far ahead of k-NN and Naive Bayes.
2. **The population does not form natural clusters.** Silhouette scores of 0.18 and 0.23 indicate overlapping groups rather than three distinct types of person.
3. **Association rule mining at 10% support and confidence generates too many rules** (430,623) to use directly, so lift-based filtering is essential.
4. **Converting categories to numbers limits several of the methods** I used (k-NN, Naive Bayes, k-Means and linear regression). One-hot encoding or algorithms designed for categorical data are the clearest improvement.

## Limitations and next steps

- Around 22% of records had no social-grade code and many more had no `No of hours` (people not in work), so my conclusions apply to the working population with a recorded grade, not to everyone.
- I used a single 70/30 split. Cross-validation would show whether the results are stable.
- Next, I would tune k for clustering, prune the decision tree, try one-hot encoding, and report precision and recall for each social grade.
- For association rules, I would validate the strongest rules by lift and exclude attributes that are derived from each other.

---

## Related files

| File | Purpose |
|---|---|
| [`README.md`](README.md) | Project overview, dataset description and how to run the workflows |

KNIME and the KNIME logo are trademarks of KNIME AG. This project is not affiliated with or endorsed by KNIME.
