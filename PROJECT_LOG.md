# Project Decision Log

This file keeps track of the main decisions made while rebuilding the project. It is not meant to repeat the full notebook. The goal is to show how the analysis changed as the data revealed more about what could actually be supported.

## 2026-10-07: Rethinking the unit of analysis

### Where the project started

The first version tried to segment customers using their historical department mix.

That idea sounded reasonable at first, but the customer histories in the dataset turned out to be too incomplete for that interpretation.

### What the data showed

Observed orders per user were very limited:

- 54.9% of users had only one observed order
- 78.4% had two observed orders or fewer

A later coverage check showed that these were not complete customer histories. For about 93.6% of users, the highest observed `order_number` was larger than the number of orders actually present in the dataset.

The median user had a maximum observed `order_number` of 10, but only one order was available in the sample.

### Decision

The customer was dropped as the primary unit of segmentation.

The main analysis was reframed around individual orders, with the goal of identifying recurring shopping missions from basket composition.

Customer IDs were kept only for a secondary analysis using consecutive orders that were actually observed in the data.

## 2026-10-08: Representing each basket

### Question

Should an order be represented using raw product counts or department proportions?

### Decision

Use department proportions.

### Why

Raw counts would make large baskets look more similar simply because they contain more products.

Using proportions makes baskets of different sizes comparable based on composition instead of volume.

Basket size was kept separately for later behavioral profiling.

### Result

The modeling table contains:

- 200,000 orders
- 21 department features
- one row per order
- department proportions that sum to approximately 1 for every order

## 2026-10-10: Handling compositional basket data

### Problem

Each row is a composition because all department shares add up to 1.

This means that increasing one department share automatically reduces the space available to the others. That can affect correlations, distances, and PCA.

The matrix is also sparse because most baskets only contain products from a few of the 21 departments.

### Representations compared

Two versions were tested:

1. Raw department proportions
2. Hellinger transformed proportions

PCA was fitted separately to both.

The raw representation concentrated more variance in the first components:

- Raw PC1: about 30.45%
- Hellinger PC1: about 19.22%
- Raw PC1 to PC3: about 59.19%
- Hellinger PC1 to PC3: about 42.99%

Even though the variance was distributed differently, the main PCA structure stayed very similar. The absolute correlations between corresponding loading vectors for the first five components ranged from about 0.965 to 0.990.

### Scaling decision

Standard scaling was not used.

All features already came from the same proportional scale, and forcing each department to unit variance could give rare departments more influence than intended.

### Decision

Use the Hellinger transformed proportions for the primary clustering analysis.

Raw proportions were kept for interpretation and sensitivity checks.

### Why

The Hellinger transformation keeps zero values, reduces the dominance of very large basket shares, and gives secondary departments more influence without replacing the main structure already present in the data.

## 2026-10-10: Keeping small baskets

### Problem

Department proportions are much more sensitive to individual products when baskets are very small.

For example, adding one product to a basket with only one or two items changes its composition much more than adding one product to a basket with twenty items.

Orders containing three products or fewer represent 17.13% of the dataset.

### Decision

Keep small baskets in the main analysis, but test how much they affect the final clustering.

### Why

Small baskets are still real shopping events. Removing them automatically could erase valid focused purchases such as beverage heavy or produce heavy trips.

The better approach was to keep them first and then test whether the main profiles survived without them.

## 2026-10-10: Choosing the PCA representation

### Question

Should K Means run directly on all 21 Hellinger features or on a reduced PCA representation?

### Approach

Three PCA representations were compared:

- 7 components
- 10 components
- 13 components

The comparison did not rely only on explained variance. Cluster separation, balance, stability, and interpretability were considered together.

### Result

The 7 component representation consistently produced better silhouette scores than the 10 and 13 component alternatives.

Adding more lower variance components did not improve cluster separation.

### Decision

Use 7 principal components for the primary clustering model, then compare the result with direct clustering on all 21 Hellinger features as a robustness check.

## 2026-10-10: Choosing the number of clusters

### Search

For each PCA representation, K Means solutions from 2 to 10 clusters were tested using:

- silhouette score
- inertia
- cluster balance
- interpretability in the original department space

The 2 cluster solution had the highest silhouette score, but it was too coarse for the goal of finding differentiated shopping missions.

Within the 7 component representation, silhouette performance flattened around 7 to 10 clusters.

### Candidate comparison

The 6, 7, and 8 cluster solutions were inspected more closely.

Moving from 6 to 7 clusters separated broader mixed baskets into clearer patterns.

The 8 cluster solution added a distinct pantry and produce profile with elevated canned goods and dry goods while keeping roughly the same geometric separation as the nearby alternatives.

### Decision

Use the following primary model:

- Hellinger transformed department proportions
- 7 principal components
- 8 K Means clusters
- 50 centroid initializations for the final model

The final cluster sizes range from about 5.3% to 21.2% of all orders.

The silhouette score is about 0.171. That is modest, so the clusters should be treated as overlapping shopping patterns rather than perfectly separated natural groups.

## 2026-10-10: Checking against a dominant department baseline

### Question

Is K Means finding multi department basket structure, or is it mostly reproducing the department with the largest share in each order?

### Adjustment

A simple `idxmax` baseline was not enough because many baskets have ties for the largest department share.

About 23.6% of orders contain a tie, so those cases were separated before evaluating the baseline.

### Findings

Among orders with a unique dominant department:

- Cluster 0: dairy eggs, 87.8%
- Cluster 1: produce, 98.1%
- Cluster 2: frozen, 34.5%
- Cluster 3: household, 28.1%
- Cluster 4: produce, 77.2%
- Cluster 5: snacks, 72.5%
- Cluster 6: beverages, 92.6%
- Cluster 7: produce, 72.2%

Several clusters share produce as their most common dominant department but have clearly different broader basket profiles.

Clusters 2 and 3 are also much more mixed than the specialized produce, dairy, snack, or beverage patterns.

### Decision

Keep the 8 cluster solution. The baseline confirmed that the model captures more than a simple dominant department rule.

## 2026-10-10: Robustness checks

Three checks were used before interpreting the final clusters.

### Direct Hellinger clustering

K Means was fitted directly to all 21 Hellinger features without PCA.

Results:

- Direct Hellinger silhouette: 0.107
- PCA based silhouette: 0.171
- Adjusted Rand Index between both partitions: 0.469

The two approaches share some structure, but PCA gives K Means a cleaner space for separation.

PCA was kept in the final pipeline.

### Removing small baskets

The full PCA and K Means pipeline was repeated after removing orders with three products or fewer.

This retained 165,743 orders, or 82.87% of the dataset.

Results:

- 7 component PCA variance retained: 68.26%
- silhouette without small baskets: 0.125
- ARI versus the original assignments: 0.529

Most cluster profiles remained very similar after filtering. The best profile correlations were:

- Cluster 0: 0.91
- Cluster 1: 0.99
- Cluster 2: 0.98
- Cluster 3: 0.58
- Cluster 4: 0.99
- Cluster 5: 1.00
- Cluster 6: 0.88
- Cluster 7: 0.99

Cluster 3 was the main exception. It also had the highest concentration of small baskets at about 47%.

### Random seed stability

The final model was fitted again using five different random seeds with 50 centroid initializations each.

ARI values ranged from 0.944 to 0.981, with an average of about 0.966 relative to the primary solution.

This showed that the final partition was highly reproducible once enough centroid initializations were used.

### Decision

Retain the PCA based 8 cluster model.

Most profiles are stable enough to interpret confidently. Cluster 3 should be treated with more caution because it is more sensitive to small baskets.

## 2026-10-15: Naming the shopping missions

The cluster names were added only after the model and robustness checks were complete.

The goal was to keep the labels descriptive and avoid inventing customer intent that the data do not observe directly.

Final labels:

| Cluster | Shopping mission |
| --- | --- |
| 0 | Dairy Core |
| 1 | Produce Core |
| 2 | Frozen Variety |
| 3 | Household Pantry |
| 4 | Fresh Essentials |
| 5 | Snack Basket |
| 6 | Beverage Basket |
| 7 | Pantry Staples |

Names such as weekly restock, party run, healthy shopper, or family basket were avoided because the dataset does not provide enough evidence to support those interpretations.

## 2026-10-15: Behavioral profiling

The clustering model used only department composition, so contextual order variables were used afterward to see whether the missions also differed in other ways.

The strongest differences appeared in:

- basket size
- number of departments per order
- mean item reorder rate

Across clusters:

- median basket size ranged from 4 to 13 products
- average department count ranged from 2.83 to 6.58
- mean item reorder rate ranged from 46.24% to 65.93%

Average order hour changed very little, ranging only from about 13.28 to 13.73. Days since the previous observed order and the share of first orders were also fairly similar between missions.

### Interpretation

The missions differ mainly in what is bought and in the scope of the basket, not in strong time of day patterns.

## 2026-10-15: Mission persistence across consecutive orders

Because customer histories are incomplete, the longitudinal analysis was restricted to pairs of consecutively numbered orders that were both present in the dataset.

The final sample contained:

- 10,236 users with at least one valid transition
- 11,318 consecutive order transitions

The same shopping mission was repeated in 35.2% of transitions.

Every mission had a repeat rate above its chance baseline, with lift ranging from 1.73 to 5.04.

### Interpretation

Shopping missions show real persistence, but they are not permanent customer segments.

Most transitions still move to a different mission, which supports the idea that the same person can shop for different reasons across visits.

This is why the project stays centered on order level missions rather than assigning one fixed segment to each customer.

## 2026-10-17: Final interpretation boundary

The final model supports the idea that recurring basket structures exist and that some departments tend to appear together in consistent ways.

It also shows that these patterns differ in basket size, department diversity, and reorder behavior.

The project does not claim that the clusters reveal true customer intent or permanent customer types.

Important limits remain:

- customer histories are incomplete
- the silhouette score is modest, so missions overlap
- Cluster 3 is more sensitive to small baskets
- the clustering works at the department level rather than the individual product level
- price, spend, promotions, and other economic variables are not available
- K Means assigns every order to one group even when some baskets may sit between multiple patterns

The final interpretation is therefore simple: the model provides useful summaries of recurring shopping patterns, not definitive labels for people.
