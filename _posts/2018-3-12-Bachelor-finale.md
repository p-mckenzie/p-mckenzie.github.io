---
# Posts need to have the `post` layout
layout: post

# The title of your post
title: "Twitter takes on the Bachelor (finale)"

# (Optional) Write a short (~150 characters) description of each blog post.
# This description is used to preview the page on search engines, social media, etc.
description: >
  A descriptive analytics look at twitter reactions to the Bachelor season 22 (Arie Luyendyk Jr.) 
  finale and After the Final Rose. **PG-13 post, due to strong language contained in some tweets.**

# (Optional) Link to an image that represents your blog post.
# The aspect ratio should be ~16:9.
image: /assets/img/bachelor-finale/header.jpg

# You can hide the description and/or image from the output
# (only visible to search engines) by setting:
hide_description: false
hide_image: false

categories: [descriptive]
tags: [text]
languages: [Python]
---
![]({{site.url}}/assets/img/bachelor-finale/header.jpg){:.lead}

<!--more-->

* dummy list
{:toc}
<br>
For those unfamiliar, the Bachelor is a television show on ABC that follows one man (the Bachelor) as he
dates >20 women, eliminating a few each week, until the finale, when he (usually) proposes to the last woman
standing. 

Season 22, with bachelor Arie Luyendyk Jr., had rather a dramatic ending, as Arie proposed to the 'winner', 
Becca Kufrin, but 2 months later (before the show was finished airing) broke up with her to be with
the '1st runner-up', Lauren Burnham, but only after checking with Lauren that she would take him back.

ABC decided to broadcast the entirety of the brutal break-up, "uncut and unedited", on Monday night.
On Tuesday night, Lauren, Arie, and Becca were all reunited on After the Final Rose to address the controversy,
and Becca was announced as the next Bachelorette (to star on a gender-swapped version of the Bachelor, also produced
by ABC).

Needless to say, twitter had some strong feelings about the events of Monday and Tuesday night, which I decided
to analyze.

Bachelor Nation (the term for viewers/fans of the show) expressed their feelings rather colorfully, so once again 
this post contains strong language. You've been warned.

# Data Acquisition
I used [python-twitter](https://python-twitter.readthedocs.io/en/latest/index.html), 
a wrapper for the twitter API, to extract over 650,000 tweets which mentioned one of
either `Arie`, `bachelor`, or `bachelorette` between February 27th, 2018, and March 10th, 2018.

This process took over 12 hours, due to the rate-limiting property of the twitter API. I parsed
the tweets to extract the time, user, and text of the tweet, as well as the number of favorites and retweets
the tweet has, and stored each tweet as a row in a Pandas dataframe.

The scraper is available in my example projects GitHub [repo](https://github.com/p-mckenzie/example-projects),
and can be used with any query to the twitter API (with a few modifications). It nicely parses the tweets
while waiting for the rate-limit to refresh, saving processing time. 

# Data Cleaning
As anybody who's used twitter knows, tweets are usually a mess. I decided I was interested in the words used, and 
so decided to remove punctuation, links, emoji, etc.

If you don't have a favorite regular expression, I would suggest:
```python 
re.sub(r"[^ -~]+", '', x)
``` 
Simply, this
expression will delete any character which is not between a space or a tilde (inclusive), otherwise known
as every character which is in the ASCII table. This can prevent errors in saving/loading data with odd 
characters, and was a real life-saver during this project.

Next, a few items to parse:
1. The time column, which Twitter returns as GMT, and I converted to my own time zone (central time) by subtracting 6 hours.
```python
df['time'] = df['time'].apply(lambda x: datetime.strptime(x, '%Y-%m-%d %H:%M:%S')-timedelta(hours=6))
```
2. The **RT @...**, which starts each retweeted tweet, and I was not interested in, since this post does not focus on network effects.
```python
df['text_clean'] = df['text'].apply(lambda x:re.sub(r"RT @[\w\d]+: ", '', x))
```
3. Any columns without text (or without text after removing the retweet shout-out) were removed, since I'm concerned only with the words people used.
```python
df = df[~df['text_clean'].isnull()]
```
4. For tidiness, I removed URLs, and lowercased everything.
```python
df['text_clean'] = df['text_clean'].apply(lambda x:re.sub(r"http[\w:\/\.]+", '', x).lower())
```
5. I replaced single quotes (ignoring apostrophes) with double quotes, using a handy regular expression I used in my [Jane Austen text analysis]({{site.url}}/2018/01/11/Jane-Austen/), before removing all punctuation except for apostrophes.
```python
df['text_clean'] = df['text_clean'].apply(lambda x:re.sub(r"((?<=[^\w])\'|\'(?=[^\w]))", '"', x)) #replace ' with "
df['text_clean'] = df['text_clean'].apply(lambda x:re.sub(r"[^\w ']", ' ', x)) #replace punctuation with spaces
```
6. I was only interested in tweets where the text actually mentions the topic (rather than a tweet from @Becca_fan9000), and so I filtered accordingly.
```python
df = df[df['text_clean'].str.contains('arie')|df['text_clean'].str.contains('bachelor')]
```
7. Lastly, I was interested in the sentiment each tweet expressed, and so used [VADER](https://github.com/cjhutto/vaderSentiment) again.
```python
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
analyser = SentimentIntensityAnalyzer()
df['sent'] = df['text_clean'].apply(lambda x:analyser.polarity_scores(x)['compound'])
```

# Lay of the Land
My first step was to find the average sentiment in the tweets over the course of the two evenings 
(broadcast times enclosed by black lines).
![]({{site.url}}/assets/img/bachelor-finale/sentiment-and-volume.JPG)
You could probably guess from looking at the graphs, but Arie breaking up with Becca 
(as Chris Harrison, the host, pointed out several times) "unedited and uncut!" was broadcast
around 21:30 on Monday evening. This is evident in the spike in tweets sent, and the drop
in sentiment the tweets contained.

We can also see that After the Finale Rose didn't inspire quite as many tweets as the Finale's
drama did, but seems to have left Twitter at least slightly happier about the show,
as evidenced by the higher sentiment on Tuesday over Monday, 
in the tails to the right of both graphs.

# Couples
I next decide to perform a similar analysis, this time pitting tweets that mentioned Arie
with Becca (his first fiancée, who he broke up with)
against tweets that mentioned Arie with Lauren (the girl he left Becca for).
![]({{site.url}}/assets/img/bachelor-finale/couples.JPG)
We can see dramatic dips in sentiment on Monday, first directed at Becca and Arie at 21:00 
(around the time that Arie telling Lauren he was choosing Becca aired).
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">ARIE CHOSE WRONG. LAUREN DESERVED BETTER. I HATE THIS SHOW</p>&mdash; scotty ✴️ (@Scottgrayson16) <a href="https://twitter.com/Scottgrayson16/status/970857163450208256?ref_src=twsrc%5Etfw">March 6, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

... and next directed at Lauren and Arie, around 21:30 (around the time that Arie's brutal breakup with 
Becca aired). And yes, ABC really did air the proposal and the breakup less than half an hour apart.
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">I hope Lauren dumps him <a href="https://twitter.com/Ariejr?ref_src=twsrc%5Etfw">@Ariejr</a> and what an ass you made of yourself !!! Becca was way too good for you and that was a D*** move !!!!!! <a href="https://twitter.com/ABCBachelorThe?ref_src=twsrc%5Etfw">@ABCBachelorThe</a></p>&mdash; SheSheMurphyD (@boandosh) <a href="https://twitter.com/boandosh/status/970869856752021506?ref_src=twsrc%5Etfw">March 6, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Just a guess, but it seems like people don't like what Arie did.

# Words
Before playing word association with Arie's name, I first wanted to see what words were used most.

I parsed each tweet, keeping only words that are in the dictionary (or at least a dictionary,
acquired [here](http://www.sil.org/linguistics/wordlists/english/wordlist/wordsEn.txt)).
```python
with open("english_dictionary.txt") as word_file:
    english_words = set(word.strip().lower() for word in word_file)
    
def is_english_word(word):
    return word in english_words
```
I also used `nltk`'s stopwords functionality to remove basic words, and added a few of my own 
which seemed relevant:
```python
cachedStopWords = nltk.corpus.stopwords.words("english")
cachedStopWords += ['arie', 'becca', 'lauren', 'bachelor', 'finale']
```
Lastly, I found each word in all the tweets, parsed them, counted each unique word,
and used `nltk`'s part of speech tagger.
```python
words = pd.Series([word for word in re.findall(r"[a-z']+", ' '.join(df['text_clean'])) if is_english_word(word) and word not in cachedStopWords]).value_counts().reset_index()
words.columns = ['word', 'count']
words['pos'] = pd.DataFrame(nltk.pos_tag(words['word']))[1]
words.head()
```

|Part of speech|Words|
|:----|:----:|
|Nouns|**love**<br>season<br>watch<br>men<br>women|
|Verbs|get<br>**go**<br>think<br>know<br>want|
|Adjectives|amp<br>I'm<br>new<br>last<br>**leave**|
|Adverbs|right<br>ever<br>never<br>even<br>really|
{:.flip-table}

Rather fittingly (or sadly), love is the word most mentioned overall. 

For those who did not watch, Arie broke up with Becca and then followed her around the house,
while she cried and repeatedly told him to leave. Probably the high mentions of **leave** and **go**
are the twitterverse expressing frustration that Arie wouldn't leave the poor girl alone
to cry.

# Word Assocation
I say Arie Luyendyk Jr., Twitter says...? Let's find out.

To find a strong relationship between two words I used lift, a metric I've used before
that indicates whether words are associated if their mutual lift "score" is greater than 1.
It measures whether words are more likely to appear together than apart.

Note that in calculating lift I decided to drop duplicates on the text column, effectively
ignoring the impact of retweets on this calculation, as I didn't want a large number of 
retweets to skew the numbers and instead decided to focus on unique associations here.
```python
def calc_lift(a, b, df):
    df = df.drop_duplicates('text_clean')
    total_size = len(df)
    filter_a = df[df['text_clean'].str.contains(a)]
    num_a = len(filter_a)
    num_b = len(df[df['text_clean'].str.contains(b)])
    num_a_b = len(filter_a['text_clean'][filter_a['text_clean'].str.contains(b)])
    return total_size*float(num_a_b)/float(num_a*num_b)
	
print calc_lift('arie', 'tool', df[df['day']==5]), calc_lift('becca', 'goddess', df[df['day']==5])
### 1.7390005610282486, 2.14683135172573
```
Those are both strong relationships in tweets on Monday.

I then defined a list of words that I thought might appear, relying on my own opinions and
words found in the previous section, and calculated pairwise lift scores on Monday (and then on Tuesday)
for each person-descriptor pair.
```python
people = ['arie', 'becca', 'lauren']
words = ['tool', 'goddess', 'queen', 'asshole', 'happy', 'cute', 'yay', 'hate',
         'handsome', 'beautiful', 'hot', 'forgive', 'blame',
        'love', 'couple', 'mess', 'fake', 'staged', 'angry', 'real', 'true', 'leave', 'poor']
lift_mon = pd.DataFrame(columns=people, index=words)
lift_tue = pd.DataFrame(columns=people, index=words)
for word, series in list(lift_mon.iterrows()):
    for person in series.index:
        lift_mon[person].loc[word] = calc_lift(person, word, df[df['day']==5])
        lift_tue[person].loc[word] = calc_lift(person, word, df[df['day']==6])
```
Using this analysis, we can find the person most associated with each descriptor on each day:

|Descriptor|Monday|Tuesday|
|:----|:----:|:----:|
|tool   |   Arie|Arie|
|goddess   |   Becca|Becca|
|queen    |  Becca|Becca|
|asshole  |      Arie|Arie|
|happy    |     Becca|Becca|
|cute     |     Becca|Becca|
|yay      |      **Arie**|**Becca**|
|hate      |     Arie|Arie|
|handsome  |    Becca|Arie|
|beautiful  |   Becca|Becca|
|hot       |   Lauren|Becca|
|forgive  |      Arie|Lauren|
|blame    |      Arie|Arie|
|love     |    Lauren|Lauren|
|couple    |   Lauren|Lauren|
|mess       |    Arie|Lauren|
|fake        |   Arie|Lauren|
|staged    |    Becca|Becca|
|angry    |     Becca|Arie|
|real     |    Lauren|Lauren|
|true     |      Arie|Lauren|
|leave    |      Arie|Arie|
|poor     |     Becca|Becca|

The only really notable change between Monday and Tuesday is the switch of **yay** from Arie to Becca, probably indicating
people celebrating Becca's becoming the Bachelorette instead of Arie's getting engaged initially.

Lastly, for each character, I plotted the percentage mentions of their top-5 attributes
over the two screenings. For example, for Arie, 25% of the tweets mentioning him at 22:00 on Monday
also mentioned **leave**.
![]({{site.url}}/assets/img/bachelor-finale/arie.JPG)
This is certainly because he hung around while Becca repeatedly told him to leave
after he broke up with her.

On Tuesday, there is very little cohesion other than a spike around 19:30 when all of twitter
exploded over the fact that Lauren would **forgive** him.

As for Lauren, she sees a similar **forgive** spike at the same time on Tuesday.
![]({{site.url}}/assets/img/bachelor-finale/lauren.JPG)
The only other very prominent pattern, across both days, is the high percentage of people
commenting how in **love** she is.

Lastly, Becca (crowned new Bachelorette on Tuesday around 20:30):
![]({{site.url}}/assets/img/bachelor-finale/becca.JPG)
Somewhat sadly (in retrospect), Becca has a spike in **happy** around 21:00 on Monday (when Arie chooses her), followed by
a 20-ish minute delay spike in **poor**, certainly when the breakup aired.

Later that night, we can clearly see Bekah M. (a former contestant on Arie's season) send out a tweet that quickly takes 
over Becca's mentions:
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">hahahahahaha <a href="https://twitter.com/ariejr?ref_src=twsrc%5Etfw">@ariejr</a> is the biggest fucking tool i’ve ever seen. becca is a queen. a goddess. thank the LORD he’s out of her life</p>&mdash; bekah martinez ♡ (@whats_ur_sign_) <a href="https://twitter.com/whats_ur_sign_/status/970881191682351104?ref_src=twsrc%5Etfw">March 6, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
On Tuesday, we can see the twitterverse happy with Becca becoming bachelorette as words like **happy** and **yay** become
prominent while **poor** dies off early in the broadcast window.

# Conclusion
This project was mainly an excuse for me to learn to use the twitter API, and I'm pleased 
to have added another tool to my skill set. Regardless of how you feel about the drama
that unfolded, I think we all wish Arie and Lauren well.

More importantly, we also hope Becca finds love that is reciprocated on the Bachelorette.

![]({{site.url}}/assets/img/bachelor-finale/becca.gif)

# Code
All analysis was completed using Python 2.7, relying primarily on the pandas, re, and nltk libraries. 
Visualizations created with matplotlib.
The data was extracted from twitter using python-twitter, a python-based wrapper for the twitter API.

All code can be found in my Example Projects git [repository](https://github.com/p-mckenzie/example-projects).