---
# Posts need to have the `post` layout
layout: post

# The title of your post
title: "Instacart Part 2 - Modeling"

# (Optional) Write a short (~150 characters) description of each blog post.
# This description is used to preview the page on search engines, social media, etc.
description: >
  A modeling approach to the Instacart Market Basket Analysis, hosted by Kaggle, using engineered features.
  The second of a 2-part series (part 1 available [here](https://p-mckenzie.github.io/2017/12/12/instacart-part-1/ "Instacart Part 1 - Feature Engineering")).

# (Optional) Link to an image that represents your blog post.
# The aspect ratio should be ~16:9.
image: /assets/img/instacart/instacart-checkout.jpg

# You can hide the description and/or image from the output
# (only visible to search engines) by setting:
# hide_description: true
# hide_image: true

categories: [predictive]
tags: []
languages: [Python]

featured: true
hidden: true
---
{% include /custom/bokeh_instacart_2.html %}
![]({{site.url}}/assets/img/instacart/instacart-checkout.jpg){:.lead}

<!--more-->

* dummy list
{:toc}

# Synopsis
[Instacart](https://www.instacart.com/ "Instacart.com"), an app that connects users to personal shoppers who pick and deliver their groceries for them, challenged the Kaggle community to use a set of 3 million orders for over 200,000 anonymized users to predict which previously purchased products will be in a user’s next order.

After my feature engineering (described in [part 1]({{site.url}}/2017/12/12/instacart-part-1/ "Instacart Part 1 - Feature Engineering") of this series), the next step was to train a few models to use these features to predict the target, whether or not the user ordered the product in their most recent order, and then aggregate the predictions by user to generate a full order of maximum probability.

Note that feature engineering code as well as data analysis notebooks can be found in my Git [repository](https://github.com/p-mckenzie/Instacart-market-basket-analysis "p-mckenzie/Instacart-market-basket-analysis") for this project.

## Baseline
The easiest prediction to make to assume the user places the exact same order that they did last time. 
This produces a Kaggle score of **0.3118025**.

# Classification Models
The first step was to split my engineered features with known class by user_id, keeping 70% of the user_ids in the train set and reserving 30% of the user_ids for testing, to simulate the Kaggle validation environment (predicting on different users from the training set).

I built 4 different classifiers, using the parameters as follows:
1. Logistic Regression
```python
log = LogisticRegression()
log.fit(X_train,y_train)
```
2. Multilayer Perceptron Classifier
```python
mlp = MLPClassifier(hidden_layer_sizes=(10,40), activation='tanh',
                       solver='sgd', learning_rate='adaptive',
                      random_state=42, batch_size=500, learning_rate_init=.003,
                        momentum=.5, tol=.001, verbose=False,
                       early_stopping=True, validation_fraction=.2)
#fit to training data
mlp.fit(X_train, y_train)
```
3. XGBoost
```python
gbm = xgboost.XGBClassifier(objective='binary:logistic',
                            max_depth=10,
                            learning_rate=.2,
                            n_estimators=140,
                            subsample=.8,
                            colsample_bytree=.8)
gbm.fit(X_train, y_train,
                eval_set=[(X_test, y_test)],
                eval_metric='auc',
                early_stopping_rounds=2)
```
4. LightGBM
```python
gbm = lightgbm.LGBMClassifier(objective='binary:logistic',
                            num_leaves=40,
                            learning_rate=.2,
                            n_estimators=140,
                            feature_fraction=.8)
gbm.fit(X_train, y_train,
        eval_set=[(X_test, y_test)],
        eval_metric='auc',
        early_stopping_rounds=2, verbose=False)
```

## Comparison
Looking at AUC for my 30% reserved test set, I found all 4 models to be clustered around .57, with LightGBM the incremental winner.

<div class="bk-root">
    <div class="bk-plotdiv" id="9a876d76-21af-4bc7-aa45-7a0fb9cdd521"></div>
</div> 

Note that grid search results to arrive at these models and the code itself can be found in be found in my Git [repository](https://github.com/p-mckenzie/Instacart-market-basket-analysis "p-mckenzie/Instacart-market-basket-analysis") for this project, in the Preliminary Analysis notebook.

## Issues

<img src="{{site.url}}/assets/img/instacart/red-flag.jpg" height="150px">

Using the `predict()` method on the Kaggle validation set of almost 5 million user-product pairs, I found a huge issue:

| Model | + Predictions | - Predictions | + percentage |
|-----------------:|:---------:|:------:|:-----:|
|Logistic Regression|123,092|4,710,200|0.0254|
|MLP| 134,919 | 4,698,373 |0.0279|
|XGBoost| 140,879 | 4,692,413 |0.0291|
|LightGBM| 142,741 | 4,690,551 |0.0295|

In my training set, the positive class had made up about 10% of the rows. My models are predicting the positive class at a quarter of that rate!

My initial attempt to remedy this was to lower the global classification thresholds until I got the correct number of predictions, but I found this left different users with dramatically different cart sizes than they had historically ordered.

I then wondered if the global classification threshold was ill-suited to the problem I am trying to solve.

## Half-solution
To allow for more flexibility in my model, I could change to a user-specific classification metric.
The two logical thresholds I thought of were:
1. **Previous order size** - assume that the user will order the same number of products as they did in their previous order
2. **Average order size** - assume that the user will order the same number of products as they did on average, across all their previous orders

<div class="bk-root">
    <div class="bk-plotdiv" id="982e1098-a742-41a4-8836-0de71ab4d9fd"></div>
</div> 
<div class="bk-root">
    <div class="bk-plotdiv" id="b10e01ce-70db-4aae-bf65-0d85f95f7ee4"></div>
</div> 

I instead used the `predict_proba()` function to get probilities for each user-product pair, and selected the top-N products for each user, for each assumption/model combination.
However, I discovered that the average probabilities were incredibly high (just over 0.9 for LightGBM, as an example), leaving little separation to make it clear which products were measurably better.
In fact, the Kaggle submissions I attempted had scores far below the **0.3118025** baseline, ranging from **0.1054205** (Logistic Regression, average assumption) all the way up to **0.2065613** (MLPClassifier, average assumption).
Not a single classification model came close the baseline Kaggle score.

# Rethinking the Approach
My thought process was as follows:
1. Fundamentally what I want is to map the user-product features into a likelihood-to-buy ranking, from which I can select the top-N (depending on assumption) most likely products for each user.
2. The problem I ran into with the classification models I attempted was all the probabilities were clustered together around 90%, with little separation.
3. This lack of separation is probably causing the ranking system to be incorrect. 
4. What I need is a model that will force the values to spread out, by penalizing the difference between the predicted likelihood and the actual likelihood.
5. I also wanted to more heavily penalize large deviations over small deviations. 
6. Mean squared error is probably the metric to accomplish this relative penalization.

<br>

... I wanted to fit a linear regression to my classification problem.

<img class="gif" src="{{site.url}}/assets/img/instacart/shock.gif" height="180px">

Everybody remain calm. I wouldn't write about it if it didn't work! Just suspend disbelief and let's go.

# Regression Models
Using the same 70/30 split as before, I will instead validate models by first classifying the top-N (previous and average order size assumptions) per user, and comparing that classification to the real accuracy, again using AUC.

For the purposes of model selection, I chose to minimize RMSE, fitting the models to predict the target (1 - bought, 0 - not bought, as before).

### Linear Regression
As a proof-of-concept, let's just try a line. A simple multivariate linear regression from Sklearn.

```python
lin = LinearRegression()
lin.fit(X_train,y_train)
```
This produced an RMSE on the 30% reserved test set of **0.2697**. 

Even better, looking at my prediction range:

|Low|Mean|High|
|:-----:|:-----:|:-----:|
|-.2421|.0978|3.1557|

That looks like more of the separation I want.

Now, applying the previous order size assumption:
```python
def label_top_n(group):
    group['lin'] = 0
    #label the products where the prediction is in the top-N (N=prev_ord_size) for that user_id
    group.loc[group['lin_pred'].isin(group['lin_pred'] \
                               .nlargest(int(max(group['prev_ord_size'])))), 'lin'] = 1
    return group
```
After applying `label_top_n()`, I compared the linear predicted targets to the actual target values, using the same AUC metric I used while fitting classification models.

```python
from sklearn.metrics import roc_auc_score
roc_auc_score(labeled['target'].values, labeled['lin'].values)
```
Keep in mind the highest AUC on the test 30% I found using a classification model was **0.5891**, using a LightGBM classifier.

... but this simple linear prediction with the previous order size assumption produced an AUC of **0.6966**!

<img class="gif" src="{{site.url}}/assets/img/instacart/bandwagon.gif" height="200px">

I was convinced, and continued my analysis with more elaborate regressors.

## MLP
With the MLPRegressor instead of MLPClassifier this time, I was specifically concerned with tuning the shape, since I had found `tanh` to be the best activation function, and I decided to use an adaptive learning rate, so didn't feel a need to tune either of these parameters.

I tuned the shape with minibatch stochastic gradient descent, with `batch_size=500`, reserving 20% of the training data for validation.
Once the model converged to a single solution with lowest RMSE on validation set possible, I then found the out-of-sample RMSE on the reserved 30% testing data, and chose the shape with lowest RMSE.

<div class="bk-root">
    <div class="bk-plotdiv" id="fd7b49f0-de5d-4022-92aa-8496a4a65c74"></div>
</div>
(Inspiration for graph from [Matthew Harris](https://matthewdharris.com/2017/12/18/hypurrr-ameter-grid-search-with-purrr-and-future/))

The best model:
```python
mlp = MLPRegressor(hidden_layer_sizes=(40,10), activation='tanh',
                       solver='sgd', learning_rate='adaptive',
                      random_state=42, batch_size=500, learning_rate_init=.003,
                        momentum=.5, tol=.001, verbose=False,
                       early_stopping=True, validation_fraction=.2)
#fit to training data
mlp.fit(X_train, y_train)
```
This model produced an out-of-sample RMSE of **0.2674**, and an AUC (with previous order size assumption) of **0.7014**, a small lift over the linear model's AUC of **0.6966** with the same assumption.

## XGBoost
The go-to for Kaggle competitions. 
Due to the large dataset and my impatience, I chose a random forest approach with `colsample_bytree=0.8` and `subsample=0.8`.

I then used a grid search on the `learning_rate`, `max_depth` of each tree, and `n_estimators` for the overall forest, though the best results were found with `learning_rate=0.05`.

<div class="bk-root">
    <div class="bk-plotdiv" id="66b916c2-e614-44f1-92ee-12914340e7c8"></div>
</div>

The best model:
```python
gbm = xgb.XGBRegressor(objective='reg:linear',
                max_depth=10,
                learning_rate=.05,
                n_estimators=240,
                subsample=.8,
                colsample_bytree=.8)
gbm.fit(X_train, y_train,
    eval_set=[(X_val, y_val)],
    eval_metric='rmse',
    early_stopping_rounds=2, verbose=False)
```
This model produced an out-of-sample RMSE of **0.2645**, and an AUC (with previous order size assumption) of **0.7096**, better than both the linear and MLP regressors.

## LightGBM
Rather than spending more time on parameter-tuning XGBoost, I moved to LightGBM, which I've found to be much faster.

I again opted for the random forest approach with `feature_fraction=0.8`. 
I performed a similar grid search to the XGB approach, changing `learning_rate`, `num_leaves` of each tree (comparable to `max_depth` for XGBoost, since LightGBM grows trees leaf-wise), 
and `n_estimators` for the overall forest, though the best results were found with `learning_rate=0.1`.

<div class="bk-root">
    <div class="bk-plotdiv" id="e408aa0a-9ef2-4201-ac8f-1adbacb6dcda"></div>
</div> 
The best model:
```python
gbm = lgb.LGBMRegressor(objective='regression',
                num_leaves=40,
                learning_rate=.1,
                n_estimators=340,
                feature_fraction=.8)
gbm.fit(X_train, y_train,
    eval_set=[(X_val, y_val)],
    eval_metric='rmse',
    early_stopping_rounds=2, verbose=False)
```
This model produced an out-of-sample RMSE of **0.2645**, and an AUC (with previous order size assumption) of **0.7093**. 

XGBoost was slightly more accurate, but LightGBM performed the grid search in ~30 minutes while XGBoost performed a similar grid search in >8 hours.

## Model Comparison

<div class="bk-root">
    <div class="bk-plotdiv" id="7733d84c-1e6a-46f6-84c6-1a370b5f627a"></div>
</div>

As with the classifiers, LightGBM was victorious in AUC on the 30% testing set.

Note that grid searches to arrive at these models and the code itself can be found in be found in my Git [repository](https://github.com/p-mckenzie/Instacart-market-basket-analysis "p-mckenzie/Instacart-market-basket-analysis") for this project, in the Final Analysis notebook.

### Feature Importance
Checking relative importance on our two best-performing models, LightGBM and XGBoost:
<div class="bk-root">
    <div class="bk-plotdiv" id="d6af84e2-3261-4b76-ba7f-636f4c50cf71"></div>
</div>
We can see a lot of variation in feature importance, but some of the important variables are quite intuitive:
* reordered_usr_average, a measure of the user's tendency to reorder in general
* overall_avg_prod_disp, a measure of how often the user orders the product
* perc_prod_support, a measure of the user's product affinity

## Kaggle Submissions
Finally, I used the trained models to predict on the user-product pairs of unknown class, applied various assumptions, aggregated by user,
and fed the CSV files into Kaggle for scoring.
<div class="bk-root">
    <div class="bk-plotdiv" id="0ffa697c-31a0-4bd4-960d-09efbf5bf7f0"></div>
</div>
In general, we can see that the Average assumption performs better than the Previous assumpion, and that XGBoost and LightGBM are the best-performing models.

All models more than beat the baseline prediction of **0.3118025**, and XGBoost with the Average assumption performed the best out of all models, with a score of **0.3752033**.