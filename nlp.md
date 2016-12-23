# Introduction
I recently had the opportunity to showcase Snowflake at JOIN, Looker's first user conference. I used my time to highlight a few Snowflake features that I find particularly useful, as someone who does analytics. The presentation demonstrated simultaneous workloads that share the same data as well as analytically intensive SQL patterns against large-scale, semi-structured data.

I thought I'd refactor my presentation as a series of blog entries to share some useful insights and interesting patterns with a broader audience. In Part 1, I'm going to build an interactive model that analyzes tweet similarity using Snowflake and Looker. In Part 2, I'm going to step through simultaneous workloads using sentiment analysis and our tweet similarity. (Note: if you haven't read our [previous blog](https://www.snowflake.net/blog/happy-tweet-sad-tweet-building-naive-bayes-classifier-sql) on sentiment analysis using Naïve Bayes, I highly recommend you do so.)

# Part 1 - Tweet Similarity
## Overview
In this article, we'll compare the language in various documents to see how similar they are to one another. This is not just an academic exercise; document similarity has many interesting real-world applications. For example, a law firm may need to parse thousands or even millions of documents during discovery to build a cohesive paper trail that might otherwise be impractical to do by hand. Alternatively, an e-commerce company might want to mine product reviews to identify people with similar preferences or feelings about specific products to build or improve a recommendation engine. To keep things simple, we'll use Twitter data. It's relatively simple to demonstrate on (no more than 140 characters); it's general enough to be understood by most people; and it's just kind of fun. We'll start by going over a few basic natural-language-processing techniques which we'll use to conduct this analysis, then we'll see them in action using Snowflake as the backend and Looker as the front end. (Note: the implementation in its entirety  is available on [Github]().)

There are a number of approaches and considerations when determining document similarity. To avoid a deep dive into natural language processing from which we may never escape (or, more likely, not be able to cover in a single blog post), let's focus on some simple methods. We'll use [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity) to determine the similarity of tweets. We'll familiarize ourselves with constructing vectors that capture our tweets using a [bag-of-words](https://en.wikipedia.org/wiki/Bag-of-words_model) approach. Lastly, we'll scale our results so that uncommon words in our documents don't get buried by more commonly occurring words using [inverse document frequency](https://en.wikipedia.org/wiki/Tf%E2%80%93idf).

## A Few NLP Basics
### Cosine Similarity
Consider two non-zero vectors, $\vec{a}$ and $\vec{b}$. The cosine of the angle $\theta$ between these two vectors is indicative of similar or dissimilar orientation in the n-dimensional space that they occupy. In our case, the metric $\cos(\theta)$ lies on the interval $[0, 1]$, where $0$ indicates the vectors are orthogonal or perfectly dissimilar, and $1$ indicates the vectors are identical. We use the following formula to calculate cosine similarity of the vectors $\vec{a}$ and $\vec{b}$:

$$\large \cos({\theta}) = \frac{\vec{a} \cdot \vec{b}}{\|\vec{a}\|\cdot\|\vec{b}\|} = \frac{\sum_{i=1}^{n}a_{i}b_{i}}{\sqrt{\sum_{i=1}^{n}a_{i}^{2}} \sqrt{\sum _{i=1}^{n}b_{i}^{2}}}$$

In Figure 2, we can see that as the angle between these two vectors approaches $0^{\circ}$, the similarity increases; as the angle approaches 90º, they become more dissimilar.

![image](https://cloud.githubusercontent.com/assets/2467394/20046852/41be8b7a-a463-11e6-917f-89db6021a23a.png)


### Bag of Words
It may not be immediately obvious how vectors and angles relate to documents of words—it's deceptively simple, actually. We'll want to create a "bag of words", which will be a bridge between vectors in n-dimensional space and our documents. 

Imagine we had a list-like data structure that contained an index for every word in the English language as well as an empty slot or placeholder for the number of times that word actually occurs in our document.

```
[(a, ø), ..., (aardvark, ø), (aardwolf, ø), ..., (and, ø), (andalusia, ø), ..., (zygote, ø)]
```

To create our bag of words, we would simply count up the number of times each word in our vocabulary appears in our document (also known as "term frequency") and update the placeholder accordingly. For illustrative purposes, considering the following document:

"I think we have a biological imperative to fear and avoid spiders. I simply cannot understand people who have pet spiders."

Our representation of this document might look like this:
```
document_1 = [(a, 1), ..., (aardvark, 0), (aardwolf, 0), ..., (and, 1), (andalusia, 0), ..., (spiders, 2), ..., (zygote, 0)]
```

Let's suppose that we had a second document already in a bag-of-words form:
```
document_2 = [(a, 1), ..., (aardvark, 1), (aardwolf, 0), ..., (and, 0), (andalusia, 0), ..., (spiders, 1), ..., (zygote, 0)]]
```

Recall that our similarity metric will ultimately rely on an inner product between our document vectors. Notice, also, that if we join our two vectors on the word indexes and perform our element-wise multipliation, that any zero element cancels out a corresponding non-zero element (_e.g._, multiplying the element for "aardvark" in document 1 with that of document 2 yields 0). 

There are two implications here: First, we can store our data in a sparse format. In other words, we just need the actual words in our document and the number of times each occurs. All of the omitted words from our corpus are assumed to occur 0 times in our document. The second implication is that, when comparing two vectors, we only care about the elements that return from an inner join—_i.e._, we only care about words that appear in both documents. Considering this, our first document's representation might look like this:

```
document_1 = [(a, 1), (avoid, 1), (biological, 1), ..., (spiders, 2), ..., (understand, 1)]
```

Or, if you prefer to think of things in tabular format:

|document_id|word|occurrences|
|---|---|---|
|1|and|1|
|1|avoid|1|
|1|biological|1|
|...|...|...|
|1|spiders|2
|
|...|...|...|
|1|understand|1|

### Term Frequency - Inverse Document Frequency (tf-idf)
Before we start making any calculations, we should consider the lack of weighting in this scheme so far. As we can see, there are many words that appear in many or most documents. If we were to proceed without addressing this, these commonly encountered words might dominate our similarity metric, leading us to conclude that many documents are similar, when, in fact, they share little in common outside of these frequently occurring words. To address this, we may want to add a penalty to our term frequency (_i.e._, the "occurrences" from above). There are a few approaches we can take here, but we'll rely on the following to keep matters simple:

$$\large \text{idf}(t, D) = \log\frac{N}{1 + \{d \; \in \; D \;: \;t \; \in \; d\}}$$

Where $N$ is the total number of documents in the corpus, and $\{d \; \in \; D \;: \;t \; \in \; d\}$ is the number of documents in which the term $t$ appears (we add 1 to the denominator to avoid a divide-by-zero error). 

With the above tools in hand, we have all of our building blocks to calculate document similarity. Let's step through the actual implementation of these steps. The next part will rely largely on SQL transformations; however, I also want to emphasize out the usefulness of Looker for this type of task, both for its ability to handle complex, often dependent, transformations in its modeling layer, and for its ability to facilitate _ad hoc_ exploration through its web UI.


## Implementation
### Twitter Data
Let's get a feel of what our Twitter data actually look like. The Twitter API returns a JSON response that's fairly rich and complex (see below). In our table, we simply store two columns: the datetime of the tweet (`created_at`) and the actual JSON response (`tweet`) in its entirety. For the `tweet` column, Snowflake is making use of its [variant](linktodocs) data type, which columnarizes the data upon ingestion, improving query performance against semi-structured data.

```
{
   "contributors":null,
   "coordinates":null,
   "created_at":"Mon Oct 31 05:23:51 +0000 2016",
   "entities":{
      "hashtags":
        [
            { "indices": [ 62, 70 ], 
              "text": "nofilter" } 
        ]
      ],
      "symbols":[
      ],
      "urls":[
      ],
      "user_mentions":[
      ]
   },
   "favorite_count":0,
   "favorited":false,
   "filter_level":"low",
   "geo":null,
   "id":792960253834858496,
   "id_str":"792960253834858496",
   "in_reply_to_screen_name":null,
   "in_reply_to_status_id":null,
   "in_reply_to_status_id_str":null,
   "in_reply_to_user_id":null,
   "in_reply_to_user_id_str":null,
   "is_quote_status":false,
   "lang":"en",
   "place":null,
   "retweet_count":0,
   "retweeted":false,
   "source":"<a href=\"http://twitter.com/download/android\" rel=\"nofollow\">Twitter for Android</a>",
   "text":"Hey! Everything's fine.",
   "timestamp_ms":"1477891431660",
   "truncated":false,
   "user":{
      "contributors_enabled":false,
      "created_at":"Wed May 16 00:27:18 +0000 2012",
      "default_profile":false,
      "default_profile_image":false,
      "description":"I am an example person",
      "favourites_count":2914,
      "follow_request_sent":null,
      "followers_count":620,
      "following":null,
      "friends_count":497,
      "geo_enabled":true,
      "id":581377531,
      "id_str":"581377531",
      "is_translator":false,
      "lang":"en",
      "listed_count":1,
      "location":"Springfieild",
      "name":"Sam",
      "notifications":null,
      "profile_background_color":"ABB8C2",
      "profile_background_image_url":null,
      "profile_background_image_url_https":null,
      "profile_background_tile":true,
      "profile_banner_url":null,
      "profile_image_url":null,
      "profile_image_url_https":null,
      "profile_link_color":"ABB8C2",
      "profile_sidebar_border_color":"FFFFFF",
      "profile_sidebar_fill_color":"EFEFEF",
      "profile_text_color":"333333",
      "profile_use_background_image":true,
      "protected":false,
      "screen_name":"SamTheMan",
      "statuses_count":34490,
      "time_zone":"Pacific Time (US & Canada)",
      "url":null,
      "utc_offset":-25200,
      "verified":false
   }
}
```

We can traverse the variant's path using the single-colon (`:`) syntax; and we can reference array elements with a zero-based index using bracket (`[0]`) syntax. Issuing the following SQL:

```sql
select tweet:id as id
  , tweet:text as body
  , tweet:lang as language
  , tweet:user:screen_name as screen_name
  , tweet:entities:hashtags as hashtags
  , tweet:entities:hashtags[0]:text as first_hashtag
from twitter.data.tweets
limit 1
```

yields:

|id|body|language|screen_name|hashtags|first_hashtag|
|---|---|---|---|---|---|
|792960253834858496|Hey! Everything's fine.|en|SamTheMan|[{ "indices": [ 62, 70 ], "text": "nofilter" } ]|nofilter|

### Preliminaries
There are a few things we need to do in order to get things rolling. 

First, we will make use of Looker's [derived tables](https://looker.com/docs/data-modeling/learning-lookml/derived-tables). For those who are unfamiliar, these will effectively string together a series of common table expressions (CTEs). There's some unique syntax involved. For instance, when we embed `${bag_of_words.SQL_TABLE_NAME}` in a block of SQL, Looker knows to replace this with a reference to the previously defined CTE, `bag_of_words`.

Second, we need two [JavaScript UDFs](https://docs.snowflake.net/manuals/sql-reference/udf-js.html). The first, `split_text`, is detailed in [Martin's blog on Tweet Sentiment](https://www.snowflake.net/blog/happy-tweet-sad-tweet-building-naive-bayes-classifier-sql), so I won't go over that. The second, `word_count`, takes an array as input and returns an array of key-value pairs, where the key is the word and the value is the number of times that word appears in the document—effectively, our bag of words.

```sql
create or replace function word_count(WORDS variant)
  returns variant
  language javascript
    as '
      var arr = WORDS
      var obj = {};
      for (var i = 0, j = arr.length; i < j; i++) {
         obj[arr[i]] = (obj[arr[i]] || 0) + 1;
      }
      return obj
     ';
```

Let's take a look at how these work together:
```sql
select word_count(split_text('foo bar baz, my friends. foo for good measure.'))
```

yields:
```
{ "bar": 1, "baz": 1, "foo": 2, "for": 1, "friends": 1, "good": 1, "measure": 1, "my": 1 }
```

Third, we'll need to periodically compute the total number of documents or tweets in the data set, which is used later for our inverse document frequency calculation:
```sql
- view: total_documents
  derived_table:
    sql: |
        select count(1) as total
        from twitter.data.tweets
  sql_trigger_value: select current_date
```

And lastly we'll need to periodically compute stats about our corpus:
```sql
- view: corpus
  derived_table:
    sql: |
        select value::string as word
          , count(*) as occurrences
          , ln((select total from ${total_documents.SQL_TABLE_NAME})/ (1 + count(distinct tweet:id))) as idf
        from twitter.data.tweets,
        lateral flatten(split_text(tweet:text::string))
        where tweet:lang::string = 'en'
        and value::string not in (select this from stop_list)
        group by 1;
  sql_trigger_value: select current_date
```

Note a few things about the above SQL:
- We're only analyzing English-language tweets.
- We're removing common "stop words" from our analysis.
- The `corpus` transformation makes use of a [lateral view](https://docs.snowflake.net/manuals/sql-reference/functions/flatten.html), which may be unfamiliar depending on the SQL dialects you have used in the past.
- We've opted set these derived tables to materialize nightly because (1) our analysis doesn't hinge on up-to-the-minute statistics on the corpus and (2) these would be computationally impractical to include in our series of transformations that are computed on the fly.

### An Interactive Model
For the final steps, we'll place a series dependent SQL transformations into Looker derived tables. 

##### Step 1. Calculate the bag of words for all tweets.
This step will yield a result set that looks like [Table 1]() presented in the bag-of-words section.
```sql
- view: bag_of_words
  derived_table:
    sql: |
      select tweet:id::string as tweet_id
        , key as word
        , value as occurrences
      from twitter.data.tweets,
      lateral flatten(word_count(split_text(tweet:text::string)))
      where key::string not in (select this from stop_list)
      and tweet:lang::string = 'en'
  fields:
```

##### Step 2. Join in the corpus, mapping each word to its inverse document frequency.
 For each word in each tweet, we join to the corpus to bring in the inverse document frequency.
```sql
- view: similarity_prep
  derived_table:
    sql: |
      select bag_of_words.*
        , corpus.idf
      from ${bag_of_words.SQL_TABLE_NAME} as bag_of_words
      left join ${corpus.SQL_TABLE_NAME} as corpus
      on bag_of_words.word = corpus.word
  fields:
```

##### Step 3. Calculate cosine similarity between filtered tweet and all others.
The last transformation we need will take a `tweet_id` as filter input, inject that into the SQL, and compute the cosine similarity of that tweet with all others in the data set. This gives us a result set that we can expose and explore in Looker.
```sql
- explore: similarity
- view: similarity
  derived_table:
    sql: |
      select v1.tweet_id as original_tweet_id
        , v2.tweet_id as similar_tweet_id
        , sum(v1.value * v2.value * power(v1.idf, 2)) / (sqrt(sum(power(v1.value, 2) * power(v1.idf, 2))) * sqrt(sum(power(v2.value, 2) * power(v1.idf, 2)))) as cosine
      from ${similarity_prep.SQL_TABLE_NAME} as v1 
      join ${similarity_prep.SQL_TABLE_NAME} as v2 
      on v1.word = v2.word
      and {% condition tweet_id %} v1.tweet_id {% endcondition %}
      and v1.tweet_id != v2.tweet_id
      group by 1,2
  fields:
    
# filter-only fields

    - filter: tweet_id
      type: string
      suggest_explore: tweet
      suggest_dimension: id

# dimensions

    - dimension: original_tweet_id
      type: number
      hidden: true
      sql: ${TABLE}.original_tweet_id
      
    - dimension: similar_tweet_id
      type: number
      hidden: true
      sql: ${TABLE}.similar_tweet_id
  
    - dimension: cosine
      type: number
      value_format_name: decimal_2
      sql: ${TABLE}.cosine
```

The last syntax note I'll make is that Looker provides the ability to inject user-input parameters into SQL before it is sent to Snowflake. This form of SQL injection is handy when we want to include filters to reduce the amount of data scanned during these series of transforms, and to let users update the underlying SQL transformation without writing any code. So, in the UI, we'll expose a filter for Tweet ID. And when a user inputs an actual ID, say, 792960253834858496, the following code:
```sql
and {% condition tweet_id %} v1.tweet_id {% endcondition %}
```
gets transformed to:
```sql
and v1.tweet_id = 792960253834858496
```
If no ID is supplied on the front-end, it defualts to:
```sql
and 1=1
```

### Playing With Actual Data
Below, I've embedded two Looker queries. The first allows you to search tweets for certain keywords and identify similar tweets. The second performs the cosine similarity calculation on your user input; in other words, your input into the text fields is treated as the tweet against which all others are compaired.
<iframe here>
