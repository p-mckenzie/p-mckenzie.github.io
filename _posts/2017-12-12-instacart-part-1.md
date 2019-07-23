---
# Posts need to have the `post` layout
layout: post

# The title of your post
title: "Instacart pt. 1 - Feature Engineering"

# (Optional) Write a short (~150 characters) description of each blog post.
# This description is used to preview the page on search engines, social media, etc.
description: >
  A feature engineering approach to the Instacart Market Basket Analysis, hosted by Kaggle.
  The first of a 2-part series (part 2 available [here](https://p-mckenzie.github.io/2017/12/12/instacart-part-2/ "Instacart Part 2 - Modeling")).

# (Optional) Link to an image that represents your blog post.
# The aspect ratio should be ~16:9.
image: /assets/img/instacart/Instacart-delivery.jpg

# You can hide the description and/or image from the output
# (only visible to search engines) by setting:
# hide_description: true
# hide_image: true

categories: [descriptive]
tags: []
languages: [Python]
---
{% include /custom/bokeh_instacart_1.html %}
![]({{site.url}}/assets/img/instacart/Instacart-delivery.jpg){:.lead}

<!--more-->

* dummy list
{:toc}

# Contest Overview
[Instacart](https://www.instacart.com/ "Instacart.com"), an app that connects users to personal shoppers who pick and deliver their groceries for them, challenged the Kaggle community to use a set of 3 million orders for over 200,000 anonymized users to predict which previously purchased products will be in a user’s next order.

## Dataset
The data consists of large sample of order histories, where each user has between 4 and 100 orders in the dataset. 

For each unique order, we have the following information:
* Products in the order
* Time of day/day of week the order was placed
* Days since the user's previous order (up to a max of 30)
<br>
Additionally, each user's most recent order is either a training order, where the products are known (131,209 orders), or a test set, where the products are unknown (75,000 orders). It is these 75K orders that Kaggle scores for accuracy, using the mean F1 metric.

# Solution Approach
I wanted to fit a model to user-product pairs, to predict how likely a given user is to repurchase a certain product. 
These predictions could later be aggregated by user, to predict an entire cart.

Since the challenge is to predict reorders, the first task was to find every product that each user had ever ordered, prior to the training/test order. 
Each of these unique user-product pairs, I classified as either:
* **Positive class:** the user ordered the product in question in their most recent (ie training) order
* **Negative class:** the user had previously ordered the product in question, but did not order it in their most recent (ie training) order

# Process
The computations were completed in Python, relying primarily on the pandas library to group by individual users and iterate over their order history, keeping track of time displacements and the number of occurrences of various events.

The basic initial steps:

1. Reading in and joining the competition-provided datasets (available [here](https://www.kaggle.com/c/instacart-market-basket-analysis/data "Data - Kaggle's Instacart Market Basket Analysis")). 
```python
# read in data
orders = pd.read_csv('orders.csv')
train = pd.read_csv('order_products__train.csv')
prior = pd.read_csv('order_products__prior.csv')
#merge with orders
train = train.merge(orders, on='order_id', how='left')
prior = prior.merge(orders, on='order_id', how='left')
#make new dataframe
new = train.append(prior, ignore_index=True)
```
2. Finding the unique products each test user ever ordered, and adding them as potential products with unknown target in the user's test order
```python
#get unique set of user, product combinations, where the user_id is in the test set (unknowns to predict)
unique = new[new['user_id'].isin(orders[orders['eval_set']=='test']['user_id'])] \
						      .drop_duplicates(['user_id', 'product_id'])[['user_id', 'product_id']]
#merge with orders to get day of week, hour of day, order_id, etc 
#for the test order the product could potentially be in
unique = unique.merge(orders[orders['eval_set']=='test'], on='user_id', how='left')
```
3. Create target, based on which order the product was in
```python
new['target'] = 0
new.loc[new['eval_set'] == 'train', 'target'] = 1
new.loc[new['eval_set'] == 'test', 'target'] = 2
```
4. Remove user-product pairs where the product is FIRST ordered by the user in the train set, 
since we are only predicting re-orders in the test set, and have no history with which to learn 
how to predict these training points.
```python
new = new[~((new['target']==1) & (new['reordered']==0))]
```

## Result
After the merges, I had over 13 million unique user-product pairs, of various class.

| # of rows | Class | Set |
|-----------------:|:---------:|:------:|
|7,645,837| 0 | Train |
|828,824| 1 | Train |
|4,833,292| Unknown | Test |

# Features
Next, I wanted to generate a series of variables to explain the relationship between a user and a product.

## Product affinity
The first few measures were aimed at measuring how attached the user is to this particular product.

For example, what percentage of the user's orders contained this product? Intuitively, if the user has purchased the product in most of their orders, they should be more likely to purchase it again.

<div class="bk-root">
    <div class="bk-plotdiv" id="648c8d20-a3a6-4c23-8abe-7772c59a2004"></div>
</div> 
Indeed, we see this holds true. Products that were bought (target=1) tend to have higher perc_prod_support than products that were not (target=0).

I also investigated how many of the user’s consecutive orders (from most recent) have contained this product, believing a streak of several orders would be highly indicative of user-product affinity, and that this would capture more recent behavior that may not be captured in perc_prod_support. I found some separation here, though most products (regardless of purchase status) had a streak_length of zero.

## Seasonality factors
First, I wondered how many days have elapsed since the user last ordered the product in question. One would guess products that haven't been purchased for months aren't very likely to be purchased again.

<div class="bk-root">
    <div class="bk-plotdiv" id="2af49d86-e61c-417a-a037-8a8070afcc16"></div>
</div> 
Indeed, we see a much flatter distribution where the products were not bought, indicating a longer time displacement since the user last ordered the product is associated with a lower probability of purchase.

However, cans and dry goods typically have a longer shelf life than fresh fruits and vegetables. So, I wondered:
1. On average, how many days elapse between this user ordering this product?
2. On average, how many days elapse between ANY user ordering this product?
<br>
Using these numbers, is the user 'due' to order the product by their average displacement, or by the overall time displacement for this product?
<div class="bk-root">
    <div class="bk-plotdiv" id="8031e77b-a052-4530-8f85-f4dc0fe117c2"></div>
</div> 
That's a dramatic separation! The products that were bought are skewed to the right, towards a 100% due rate, while the products that were not bought are skewed to the left, with a low due rate.

## Switching behavior
However, I may be 'due' to order my breakfast cereal, but I order oatmeal instead. 
Anything from the breakfast aisle would be a reasonably suitable replacement, unless 
I'm very committed to this specific breakfast product. 

I then dug into the data to discover:
1. When the user ordered something from the product's aisle, was it this product?
2. When the user ordered something from the product's department, was it this product?

<div class="bk-root">
    <div class="bk-plotdiv" id="7c2afbb3-d62c-4221-ba2d-5f99fb4787b8"></div>
</div>
Looking at this ratio for departments we don't see quite the dramatic separation we enjoyed earlier, 
but there is still a visible difference between the distributions, indicating a higher intra-aisle or intra-department commitment is indicative of a higher product affinity.

## Data dictionary
In the end, I had 30 continuous variables:

|Attribute	|  Description	|
|-----:|:-----|
|avg_days_between_order|The average number of days between the user's orders|
|avg_order_size|The average number of products in all the user's orders|
|prev_ord_size|The number of products in the user's previous order|
|reordered_usr_avg|The average of 'reordered' across all of the user's entries in the dataset|
|num_orders_placed|The number of orders the user has placed|
|overall_avg_prod_disp|Average of usr_avg_prod_disp, across all users|
|overall_avg_aisle_disp|Average of usr_avg_aisle_disp, across all users|
|perc_aisle_support|The percentage of the user's orders which have something from the aisle|
|overall_avg_dept_disp|Average of usr_avg_dept_disp, across all users|
|perc_dept_support|The percentage of the user's orders which have something from the department|
|perc_prod_support|The percentage of the user's orders which contain the product|
|avg_ord_pos|average position of the product in the user's orders (normalized)|
|days_since_aisle|Number of days since the user has ordered from the product's aisle|
|days_since_department|Number of days since the user has ordered from the product's department|
|days_since_prod|Number of days since the user has ordered the product|
|order_aisle_displacement|Number of orders between the user placing an order from the aisle and the user ordering the product|
|orders_since_prod|Number of orders since the user last ordered the product|
|prod_aisle_ratio|ratio of how many orders had the product/how many orders had the aisle|
|prod_dept_ratio|ratio of how many orders had the product/how many orders had the department|
|usr_avg_aisle_disp|average number of days between the user ordering items from the product's aisle|
|usr_avg_dept_disp|average number of days between the user ordering items from the product's department|
|usr_avg_prod_disp|average number of days between the user ordering the product|
|streak_length|The number of consecutive orders (from most recent) the user has placed, which contain the product|
|prod_due_overall_perc|days_since_prod/overall_avg_prod_disp|
|prod_due_user_perc|days_since_prod/usr_avg_prod_disp|
|aisle_due_overall_perc|days_since_aisle/overall_avg_prod_disp|
|aisle_due_user_perc|days_since_aisle/usr_avg_prod_disp|
|dept_due_overall_perc|days_since_department/overall_avg_prod_disp|
|dept_due_user_perc|days_since_department/usr_avg_prod_disp|
|reorder_custom|Whether the product had been in the user's most recent order|

As the entirety of this project was completed on a laptop with limited memory, I chose to use only continuous variables, rather than relying on a large number of dummy-coded categories that would eat memory.

The full data dictionary along with code to complete the feature analysis can be found in my Git [repository](https://github.com/p-mckenzie/Instacart-market-basket-analysis "p-mckenzie/Instacart-market-basket-analysis") for this project.

# Next Steps
Now that the data has been manipulated into a useable form, my next post ([part 2]({{site.url}}/2017/12/12/instacart-part-2/ "Instacart Part 2 - Modeling") of my Instacart series), will describe the models I used to learn the user-product relationship.

<br>
Please note that part of this work was submitted as part of a group project for MIS 382N in the Fall of 2017, but the method, code, and commentary contained/described herein is my own independent work.
{:.faded}