---
title: "Explainable outlier detection with decision trees and ruleminer"
date: "2022-06-12"
categories: ["association rules mining"]
tags: ["decision trees"]
---

This is a note on an extension of the [ruleminer package](https://github.com/DeNederlandscheBank/ruleminer/tree/main) to convert the content of decision trees into rules to provide an approach to unsupervised and explainable outlier detection.

Here is a way to use decision trees for unsupervised outlier detection. For an arbitrary data set (in the form of a dataframe) for each column a decision tree is trained to predict that column by using the other columns as features. For target columns with integers a classifier is used and for columns with floating numbers a regressor is used. This provides an unsupervised approach resulting in decision trees that predict every column from the other columns.

The paths in the decision trees basically contain data patterns and rules that are embedded in the data set. These paths can therefore be treated as association rules and applied within the framework of association rules mining. Outliers are those values that could not be predicted accurately with the decision trees, and these are exceptions to the rules. The resulting rules are in a human readable format so this provides transparent and explainable rules representing patterns in the data set.

Training and using decision trees can of course easily be done with scikit-learn. What I have added in ruleminer package is source code to extract decision paths from arbitrary scikit-learn decision trees (classifiers and regressors) to convert them into rules in a ruleminer object. The extracted rules can then be treated like any other set of rules and can be applied to other data sets to calculate rule metrics and find outliers.

### Example with the iris data

Here is an example. I ran a AdaBoostClassifier on the iris data set in scikit-learn package and fit an ensemble of 25 trees with depth 2 (this will provide if-then rules where the antecedent of the rule contains a maximum of two conditions):

```python
base, estimator = DecisionTreeClassifier, AdaBoostClassifier

regressor = estimator(
    base_estimator = base(
        random_state=0, 
        max_depth=2),
    n_estimators=25,
    random_state=0)
regressor = regressor.fit(X, Y)
```

Here X is the features of the iris data set and Y is the target. We now have an ensemble (or forest) of decision trees. The first decision tree in the ensemble looks like this:

![](images/index.png)

The first line in the white (non leaf) nodes contain the conditions of the rules. To extract the rules from this tree I have provided an utility function in the ruleminer package that can be used in the following way:

```python
# derive expression from tree
ruleminer.tree_to_expressions(regressor[0], features, target)
```

This results in the following set of rules (in the syntax of ruleminer rules):

```
{'if (({"petal width cm"} <= 0.8)) then ({"target"} == 0)',
 'if (({"petal width cm"} > 0.8) & ({"petal width cm"} <= 1.75)) then ({"target"} == 1)',
 'if (({"petal width cm"} > 0.8) & ({"petal width cm"} > 1.75)) then ({"target"} == 2)'}
```

For each leaf in the tree the decision path is converted to a rule, each node contains the condition in the rule. To get the best decision tree in an ensemble (according to, for example, the highest absolute support) we generate a miner per decision tree in the ensemble:

```python
ensemble_exprs = ruleminer.fit_ensemble_and_extract_expressions(
    dataframe,
    target = "target", 
    max_depth = 2)

miners = [
    RuleMiner(
        templates=[{'expression': expr} for expr in exprs], 
        data=df) 
    for exprs in ensemble_exprs
]
```

From this we extract the miner with the highest absolute support

```python
max(miners, key=lambda x: miner.rules['abs support'].sum())
```

resulting in the following output of the rules

| idx | id | group | definition | status | abs support | abs exceptions | confidence | encodings |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 0 | 0 | if({"petal width cm"}<=0.8)then({"target"}==0) |  | 50 | 0 | 1.000000 | {} |
| 1 | 1 | 0 | if(({"petal width cm"}>0.8)&({"petal width cm"}>1.75))then({"target"}==2) |  | 45 | 1 | 0.978261 | {} |
| 2 | 2 | 0 | if(({"petal width cm"}>0.8)&({"petal width cm"}<=1.75))then({"target"}==1) |  | 49 | 5 | 0.907407 | {} |

With maximum depth of two, we see that three rules are derived that are confirmed by 144 samples in the data set, and six samples were found that do not satisfy the rules (outliers).

### Example with insurance undertakings data

If we apply this to the example data set I have used earlier:

```python
df = pd.DataFrame(
    columns=[
        "Name",
        "Type",
        "Assets",
        "TP-life",
        "TP-nonlife",
        "Own funds",
        "Excess",
    ],
    data=[
        ["Insurer1", "life insurer", 1000.0, 800.0, 0.0, 200.0, 200.0],
        ["Insurer2", "non-life insurer", 4000.0, 0.0, 3200.0, 800.0, 800.0],
        ["Insurer3", "non-life insurer", 800.0, 0.0, 700.0, 100.0, 100.0],
        ["Insurer4", "life insurer", 2500.0, 1800.0, 0.0, 700.0, 700.0],
        ["Insurer5", "non-life insurer", 2100.0, 0.0, 2200.0, 200.0, 200.0],
        ["Insurer6", "life insurer", 9001.0, 8701.0, 0.0, 300.0, 200.0],
        ["Insurer7", "life insurer", 9002.0, 8802.0, 0.0, 200.0, 200.0],
        ["Insurer8", "life insurer", 9003.0, 8903.0, 0.0, 100.0, 200.0],
        ["Insurer9", "non-life insurer", 9000.0, 8850.0, 0.0, 150.0, 200.0],
        ["Insurer10", "non-life insurer", 9000.0, 0, 8750.0, 250.0, 199.99],
    ],
)
df.index.name="id"
df[['Type']] = OrdinalEncoder(dtype=int).fit_transform(df[['Type']])
df[['Name']] = OrdinalEncoder(dtype=int).fit_transform(df[['Name']])
```

The last two lines converts the string data to integer data so it can be used in the decision trees. We then fit an ensemble of trees with maximum depth of one (for generating the most simple rules):

```python
expressions = ruleminer.fit_dataframe_to_ensemble(df, max_depth = 1)
```

This results in 41 expressions that we can evaluate with ruleminer selecting the rules that have confidence of 75% and minimum support of two:

```python
templates = [{'expression': solution} for solution in expressions]
params = {
    "filter": {'confidence': 0.75, 'abs support': 2},
    "metrics": ['confidence', 'abs support', 'abs exceptions']
}
r = ruleminer.RuleMiner(templates=templates, data=df, params=params)
```

This results in the following rules with metrics (the data error that we added in advance (insurer 9 is a non life undertaking reporting life technical provisions) is indeed found):

| idx | id | group | definition | status | abs suppor | abs exceptions | confidence | encodings |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 0 | 0 | if({"TP-life"}>400.0)then({"TP-nonlife"}==0.0) |  | 6 | 0 | 1.000000 | {} |
| 1 | 1 | 0 | if({"TP-life"}<=400.0)then({"Type"}==1) |  | 4 | 0 | 1.000000 | {} |
| 2 | 2 | 0 | if({"TP-life"}>400.0)then({"Type"}==0) |  | 5 | 1 | 0.833333 | {} |

There is a relationship between the constraints set on the decision tree and the structure and metrics of the resulting rules. The example above showed the most simple rules with maximum depth of one, i.e. only one condition in the if-part of the rule. It is also possible to set the decision tree parameter min\_samples\_leaf to guarantee a minimum number of samples in a leaf. In the association rules terminology this corresponds to selecting rules with a certain maximum absolute support or exception count. Setting the minimum samples per leaf to one results in rules with a maximum number of exceptions of one and this yields in our case the same results as maximum depth of one:

```python
expressions = ruleminer.fit_dataframe_to_ensemble(df, min_samples_leaf = 1)
```

The parameter min\_weight\_fraction\_leaf allows for a weighted minimum fraction of the input samples required to be at a leaf node. This might be applicable in cases where you have weights (or levels or importance) of the samples in the data set.

The rules that can be found with ensembles of decision trees are all of the form "if A and B and ... then C", where A, B and C are conditions. Rules containing numerical operations and rules with complex conditions in the consequent of the rule cannot be found in this way. Furthermore, if the maximum depth size of the decision tree is too large then resulting rules, although as such human readable, become less explainable. They might however point to unknown exceptions in the data that could not be captured with the supervised approach (predefined expressions with regexes). Taking into account these drawbacks, this approach has potential to be used in larger data sets.

The notebook of the examples above can be found [here](https://github.com/DeNederlandscheBank/ruleminer/blob/main/notebooks/Generating%20rules%20with%20decision%20trees.ipynb).
