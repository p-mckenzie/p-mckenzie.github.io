---
# Posts need to have the `post` layout
layout: post

# The title of your post
title: "Car reviews analysis"

# (Optional) Write a short (~150 characters) description of each blog post.
# This description is used to preview the page on search engines, social media, etc.
description: >
  A descriptive analysis using text and sentiment of over 42 thousand car reviews from Edmunds.com, courtesy of the
  [UCI Machine Learning Repository](http://archive.ics.uci.edu/ml/datasets/opinrank+review+dataset), for cars
  with model year 2007, 2008, and 2009.

# (Optional) Link to an image that represents your blog post.
# The aspect ratio should be ~16:9.
#image: /assets/img/insta.jpg

# You can hide the description and/or image from the output
# (only visible to search engines) by setting:
hide_description: false
# hide_image: true

categories: [descriptive]
tags: [text, scraping]
languages: [Python]
---
![]({{site.url}}/assets/img/Edmunds/header.jpg){:.lead}

<!--more-->

1. dummy list
{:toc}

# Overview
[Edmunds](https://edmunds.com/) is an online resource for automotive information, like used car listings and price comparisons.
However, they also host a wide range of [forums](https://forums.edmunds.com/) where car enthusiasts discuss
subjects like:
* [Hybrid cars](https://forums.edmunds.com/discussions/tagged/x/hybrids-evs)
* [Classic cars](https://forums.edmunds.com/discussions/tagged/x/classic-cars)
* [Safety](https://forums.edmunds.com/discussions/tagged/x/safety)

<br>
However, Edmunds also has car-specific forums where users can discuss the features of a specific vehicle, like
the Infiniti G37.

Kavita Ganesan & ChengXiang Zhai scraped Edmunds for over 42 thousand reviews for cars with model year 2007, 2008, and 2009,
and graciously made the dataset available at the [UCI Machine Learning Repository](http://archive.ics.uci.edu/ml/datasets/opinrank+review+dataset).
<br>

|Field|Description|
|:------------|:-------|
| Author | The screenname of the post's author|
| Text | The comment the author made about the car|
| Favorite | The author's favorite thing about the car (an additional text field) |
| Date | The date the review was left |
| Car | The vehicle make and model |
| Year | The year of the car in question|

The following is an exploration of the text of the reviews.

# Brand similarity
I first wanted to investigate which brands were discussed together, 
assuming many co-occurrences of brand names in reviews means that consumers associate the two brands strongly, 
and tend to compare one with the other.

For example, as `AmericanCarsStinks` writes about the Chevrolet Impala on 04/17/2007:
>This is my company car and all I have to say is this car is junk. Fortunately for me, I didn't have to pay a penny for this car. I have nothing good to say about it and a lot to complain about. My biggest complaint is the uncountable number of design "flaws". For example, when you open the trunk on a rainy or snowy day, water or snow spills into the trunk compartment! The car also has poor handling, cheap materials and a build quality that leaves much to be desired. **Spend your money on a Toyota, Honda or Nissan.**

`AmericanCarsStinks` seems to find Toyota, Honda and Nissan to be comparable brands.
Though the author seems to prefer the brands he lists to Chevrolet, I decided to rely 
only on the text of the reviews (rather than the car/manufacturer the review is written about), 
to isolate direct comparisons.

To capture this relationship between mentioned brands I used lift, a metric that attempts to measure whether terms appear together by chance or due to association.

$$ Lift = \frac{P(Brand A \, \bigcap \, Brand B)}{P(Brand A)*P(Brand B)} $$

If the lift score is greater than 1, there is an association between Brand A and Brand B that is stronger than
what would be expected by chance alone.

I computed pairwise lift scores for each brand combination in the dataset, and defined $$ dissimilarity = \frac{1}{lift} $$.

Using multidimensional scaling to translate the relative dissimilarity pairs into two-dimensional space, I generated the visualization below. 
Adjacent brands are mentioned together more often, while brands further away from each other were less likely to be mentioned together.

<iframe src="/assets/img/Edmunds/brand_similarity.html"
    sandbox="allow-same-origin allow-scripts"
    width="100%"
    height="620"
    scrolling="no"
    seamless="seamless"
    frameborder="0">
</iframe>

We can see some clustering behavior, as the luxury/performance segment of Lexus, BMW, and Infinity is centered in the top left,
and the american brands are concentrated to the bottom right. Additionally, Mini and Smart (well-known economy brands) are adjacent in the bottom right.

# Sentiment analysis
I next wanted to investigate how positively or negatively brands are viewed by the Edmunds community, rather than
simply in assocation with one another.

For this, I applied VADER (Valence Aware Dictionary and sEntiment Reasoner) (a Python [package](https://github.com/cjhutto/vaderSentiment) 
developed to extract sentiment from blocks of text) to each review. I chose to focus on the compound score, which
is a normalized, weighted composite score that takes into account the components of positive, negative, and neutral sentiment in the text.

C.J. Hutto classifies text with different compound scores as follows:
1. **Positive sentiment**: compound score >= 0.5
2. **Neutral sentiment**:  -0.5 < compound score < 0.5)
3. **Negative sentiment**: compound score <= -0.5

<br>
Grouping by brand, I looked at the mean and standard deviation in sentiment. 

<iframe src="/assets/img/Edmunds/sentiment.html"
    sandbox="allow-same-origin allow-scripts"
    width="100%"
    height="540"
    scrolling="no"
    seamless="seamless"
    frameborder="0">
</iframe>

Interestingly, the variation in sentiment between reviews increases as the average sentiment decreases,
perhaps indicating that the reviews with higher sentiment are less controversial than brands on the other end of the
specturm, like Kia and Chrysler. 

<iframe src="/assets/img/Edmunds/review_counts_brand.html"
    sandbox="allow-same-origin allow-scripts"
    width="100%"
    height="280"
    scrolling="no"
    seamless="seamless"
    frameborder="0">
</iframe>
Additionally, there was no noticeable relationship between the number of reviews
for each brand and the brand's average sentiment or variance in sentiment.

# Brand perception
Next, I wanted to crowdsource brand perception, to investigate which brands are associated with words 
like safety, performance, and price.

I decided to use lift again, this time to find which brands were most co-mentioned with 
**styling**, **quality**, **safety**, **performance**, **luxury**, **price**, and **handling**.

This time, I analyzed the number of times these attributes were mentioned in reviews for that particular manufacturer.
Revisiting our `AmericanCarsStinks` review for the Chevrolet Impala:
>This is my company car and all I have to say is this car is junk. Fortunately for me, I didn't have to pay a penny for this car. I have nothing good to say about it and a lot to complain about. My biggest complaint is the uncountable number of design "flaws". For example, when you open the trunk on a rainy or snowy day, water or snow spills into the trunk compartment! The car also has poor **handling**, cheap materials and a build **quality** that leaves much to be desired. Spend your money on a Toyota, Honda or Nissan.

`AmericanCarsStinks` has associated Chevrolet with **handling** and **quality**, though not in a positive way, 
as evidenced in a `compound` sentiment score of -0.7965 (discussed earlier in the [sentiment analysis](#sentiment-analysis) deep-dive).

Again, looking at pairwise lift scores, I found:
* If **safety** is your priority, a Volvo may be your best shot.
* The brand with the most **handling** mentions is BMW.
* If you're looking for a brand with superior **styling**, Cadillac is the way to go.
* If you're looking for a brand with great all-around **quality**, the crowd votes for Lexus.
* For the **price**-sensitive, Kia is the brand with highest co-occurrence of the term.
* Cadillac takes the top prize for both **styling** and **luxury** mentions.

<br>
Focusing in on brand-attribute pairs with `lift>1.8`, I generated the following visualization:

<iframe src="/assets/img/Edmunds/brand_associations_chain.html"
    sandbox="allow-same-origin allow-scripts"
    width="100%"
    height="650"
    scrolling="no"
    seamless="seamless"
    frameborder="0">
</iframe>
A thicker link between brand-attribute pairs indicates a larger lift value. 
Additionally we can see that all pairs have an average sentiment score above the dataset's average sentiment score,
(probably indicating the reviewer was pleased with the brand's attribute), except for **Kia-Price**,
which exhibits a slightly below-average sentiment score, probably indicating reviewers were disparaging of Kia vehicles,
though impressed with the low cost.

With this, we can see segments that were less clear in the brand map created in [Brand similarity](#brand-similarity),
which I will endearingly dub *Fancy* brands, in the right segment (BMW, Lexus, Cadillac...), 
and *Responsible* brands, in the left segment (Subaru, Volvo, Hyundai...).

# Minivan melee
I selected the minivan subset of cars for a closer, pairwise-examination.
Aggregating by brand produced at least 180 reviews per category,
plenty of opportunities for co-occcurrences, but the scarce 21 reviews for the *Dodge Caravan* don't provide enough
length for numerous comparisons, so I decided to use cosine similarity rather than lift.

Cosine similarity is a metric of similarity that is more robust to different document lengths. It maps two documents
to a score $$\in[0,1]$$, by finding an angle in many-dimensional space between two vectors, 
assembled from the content in each document. I chose to go with a pure word frequency approach.

For example, taking an imaginary corpus containing reviews for Car A and Car B, then counting the number of times
each word is used to describe either car:

|             |Car A|Car B|
|------------:|:-------:|:------:|
|acceleration | 11| 1 |
|adequate     | 2 | 1 |
|complaint    | 3 | 4 |
|cross        | 0 | 0 |
|dealership   | 3 | 5 |
|minivan      | 50| 29|
|offer        | 3 | 3 |
|performance  | 7 | 2 |
|quiet        | 19| 7 |
|smooth       | 30| 9 |

$$
\begin{aligned}
  \cos {\theta} &= \frac{\vec{A} \, \cdotp \vec{B}}{\rVert\vec{A}\lVert \, \rVert\vec{B}\lVert} \\[2em]
            &=  \frac{1,916}{\sqrt{3,962}*\sqrt{1,027}} \\[2em]
            &=  0.9498 \end{aligned}
$$

These example documents are highly similar, judging by the large cosine similarity score.

After removing stopwords and using lemmatization to decrease the dimensionality of the words used in different 
minivan reviews by a factor of 3, I applied pairwise cosine similarity to find how similarly different brands
were talked about in the reviews.

Each car pair has a cosine score, but I defined brands to have a **strong connection** if their $$\cos {\theta} > 0.7$$.
Looking at the pairs of similar cars, I found the Dodge Magnum was described most differently from other vehicles,
with only a single strong connection, while the Honda Odyssey had both the largest number of similar cars and the highest
average connection strength among them, as seen below.
<iframe src="/assets/img/Edmunds/minivan_cos_similarity.html"
    sandbox="allow-same-origin allow-scripts"
    width="100%"
    height="620"
    scrolling="no"
    seamless="seamless"
    frameborder="0">
</iframe>
While cosine similarity is more robust to different document lengths, we can still see that the Honda Odyssey
has both the largest number of strong connections and the largest number of reviews. In fact, sorting the
minivans by number of strong connections, we can see a decreasing trend in the number of reviews, indicating that
cars with more reviews tend to be similar to more other cars than those with few reviews.
<iframe src="/assets/img/Edmunds/minivan_counts.html"
    sandbox="allow-same-origin allow-scripts"
    width="100%"
    height="520"
    scrolling="no"
    seamless="seamless"
    frameborder="0">
</iframe>
This makes sense, because a larger number of reviews in high-dimensional word space make it more likely the car
has few zero entries in the vector, as we see borne out in the density of each word vector. Despite removing stopwords
and stemming to decrease the variety of words (and hence the dimensionality the vectors), the Honda Odyssey
has the largest number of reviews by almost a factor of 2, and only half of the entries in the corresponding 
word vector are zero, whereas the Dodge Caravan has only ~9% of its rows non-zero.

The more dense vector can catch similarities in more dimensions than cars with very few words in the sample set, 
allowing it to make larger cosine similarity scores on average.

# Code
The analysis was completed with Python 2.7, relying on Pandas. Visualizations built using Bokeh 0.12.13.

Code for the analysis and visualizations can be found in my [Example Projects Repository](https://github.com/p-mckenzie/example-projects).