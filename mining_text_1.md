## Parsing Text
### Introduction
Some of the most interesting and valuable data available to us takes the form of raw text. The mark of a good data practitioner rests on her ability to wrangle unstructured text and turn it into something digestible and hopefully insightful. In a series of articles, I'm going to step through a few patterns that I believe are particularly useful. To illustrate, I'm going to walk through how we process and analyze Zendesk data at Looker.

### The Setting
One can obtain Zendesk data using a provider such as [Fivetran](https://fivetran.com/) or [RJ Pipeline](https://rjmetrics.com/product/pipeline/), or they can ETL the data themselves using the [Zendesk API](https://developer.zendesk.com/rest_api/docs/core/introduction). For the purpose of this walkthrough, I'll be demonstrating on data brought into a Redshift cluster by Fivetran.

Our primary table of interest is `tickets`. The table contains a rather large number of useful attributes; but for the first part of this walkthrough, I'm going to focus primarily on the text and a few other core attributes. Those core attributes are: `id`, which denotes a unique row or chat; `created_at`, which indicates the time the chat record was created; `via_channel`, which identifies the source of the ticket, be it email or chat; and `description`, which contains the body of the entire chat in a single cell. Here's an example of what this might look like:
### _tickets_
<table style="width:100%">
  <tr>
    <td><b>id</b></td>
    <td><b>created_at</b></td> 
    <td><b>via_channel</b></td>
    <td><b>description</b></td>
  </tr>
  <tr>
    <td>1</td>
    <td> 2016-01-04 21:15:26</td> 
    <td>chat</td>
    <td>Chat started on 2016-01-04 08:33 PM UTC (08:33:20 PM) *** Donald Chamberlin joined the chat *** (08:33:20 PM) Donald Chamberlin: Hello (08:33:39 PM) *** Annie Looker joined the chat *** (08:33:43 PM) Annie Looker: Hi Donald (08:33:46 PM) Annie Looker: How's it going? (08:33:49 PM) Donald Chamberlin: Hi Annie (08:33:52 PM) Donald Chamberlin: I'm doing well, thanks. (08:34:20 PM) Donald Chamberlin: I'm having an issue with my dashboard. One of the element's wheel keeps spinning. (08:34:22 PM) Annie Looker: Ah! I can help you with that (08:34:50 PM) Donald Chamberlin: Never mind. I figured it out. Thanks anyway! *** Donald Chamberlin left the chat ***</td>
  </tr>
</table>

The format of this data is somewhat useful; however, there are insights we can only glean when digging into the body of the text. For instance, what's the typical chat duration? Are there consistent outliers? Does time-to-reply or chat cadence make a difference on the outcome and sentiment of the chat? How many overlapping chats do we tend to have during peak hours? In order to get more utility out of these data, we may want to split our raw text into a stream of conversational messages.

### The Splitting Algorithm
Redshift is one of a few SQL dialects that do not support table-generating functions. This is an issue that I addressed in a [previous post](https://discourse.looker.com/t/splitting-strings-into-rows-in-the-absence-of-table-generating-functions). We're going to build upon the pattern presented in that article, adapting it for our unique case. Our immediate goal is to take a single row, one chat, and turn it into _N_ rows: one row for each message.

The first thing we need to do is identify a character, or string of characters, that separates one message from another. We can see that, once the chat gets going, each message starts with the time surrounded by parentheses—_e.g._, (08:34:22 AM).

In my previous article, I use the `split_part` function and a fan-out join on a number table to get the 1st, 2nd, _N_th element in a comma-separated list, turning each element into a row. We'll use something similar here; however, notice that our splitting character follows a pattern. So, as far as literal characters go, each delimiter truly differs from the next. 

One workaround may be to replace our pattern with a constant and split on that character. This would work, but would leave us without our chat times to play with later down the line.  So, what we'll do is make use of `regexp_replace` and capture groups to find our delimiter pattern and replace it with itself as well as a unique character that will never show up in a message. Here's the SQL:

```sql
select id
  , regexp_replace(description, '(\\([0-9:\\s]+(AM|PM)\\))', ('\\ 1' || substring(md5(id), 1, 8))) as message
from zendesk._tickets
where via_channel = 'chat'  -- let's restrict our analysis to chats only; no emails
```

<img src="https://discourse.looker.com/uploads/default/original/2X/7/76b0fad7c4ca46c6bd855bf3dfb94beabdd70c27.png" width="689" height="53">

Let's walk through this regular expression:
- We have one capture group that we're considering, denoted by the outer-most set of `()`
- Our pattern is surrounded by literal parentheses, which we must escape: `\\( ... \\)`
- The time component will contain the numbers 0-9, a ":", and a space at least once: `[0-9:\\s]+`
- This is followed by either AM or PM: `(AM|PM)`
Putting it all together, again, we get `( \\( [0-9:\\s]+(AM|PM) \\) )`  (Additional spaces added for legibility.)

Redshift only supports capture-group back references in select regex functions. Thankfully, it's available in the one that we need. (**Note**: This feature is not documented anywhere by AWS.) So, the first argument in our `regexp_replace` function is the text we're operating on; the second is the pattern we're trying to identify (above); the third is the replacement string. In our case, we're going to replace the pattern with itself, denoted by &#92;&#92;1, concatenated with our unique split character. In this case, I'm using a the first 8 characters of an MD5 hash of the `id` as my split character, though one can use any string they like. 

(**WARNING** : Long replacement strings may cause the text to exceed Redshift's possible `VARCHAR` column limit of 65535. Make use of `substring(md5(id), 1, n)` if need be.)

### Keeping Time
Notice that the value of the `created_at` column differs from the first timestamp within the body of the chat. This is because the record was created after the completion of the chat. To accurately assign a starting date and time to this chat, we should probably extract the timestamp within the body of the text. We'll add that pattern to our query above:

```sql
select id
  , regexp_substr(description, '[0-9\\s:-]+(AM|PM)\\sUTC') as start_at
  , regexp_replace(description, '(\\([0-9:\\s]+(AM|PM)\\))', ('\\ 1' || substring(md5(id), 1, 8))) as message
from zendesk._tickets
where via_channel = 'chat'
```
<img src="https://discourse.looker.com/uploads/default/original/2X/4/4af7ba5dea45996267ee5c67e0e19c20fa253d1d.png" width="690" height="52">

### Fan It Out

Next, borrowing from our previous article, we're going to fan each row out _N_ times for _N_ messages within the body of the chat. To do this, we'll make use of a number table and the `regexp_count` function. We'll rely on our first regular expression to locate and count the occurrences of our message delimiter, then join on that number plus one.

```sql
select *
from (select id
        , regexp_substr(description, '[0-9\\s:-]+(AM|PM)\\sUTC') as start_at
        , regexp_replace(description, '(\\([0-9:\\s]+(AM|PM)\\))', ('\\ 1' || substring(md5(id), 1, 8))) as message
        , description
      from zendesk._tickets
      where via_channel = 'chat') as chat
join public.numbers
on numbers.num <= regexp_count(description, '(\\([0-9:\\s]+(AM|PM)\\))') + 1
```
<img src="https://discourse.looker.com/uploads/default/original/2X/4/44e564308c7329c20a7dd6444ed7aa03c3f3a641.png" width="690" height="324">

### Splitting Up

The next step is take our fanned out result set and split the body of the message on our split character, `substring(md5(id), 1, 8)`, and grab the 1st, 2nd, _N_th split element:

```sql
select id as ticket_id
  , split_part(message, substring(md5(id), 1, 8), num::int) as message_body
from (select id
        , regexp_substr(description, '[0-9\\s:-]+(AM|PM)\\sUTC') as start_at
        , regexp_replace(description, '(\\([0-9:\\s]+(AM|PM)\\))', ('\\ 1' || substring(md5(id), 1, 8))) as message
        , description
      from zendesk._tickets
      where via_channel = 'chat') as chat
join public.numbers
on numbers.num <= regexp_count(description, '(\\([0-9:\\s]+(AM|PM)\\))') + 1
```
<img src="https://discourse.looker.com/uploads/default/original/2X/7/7d20b97fbf016587fafe7316d81048b678a88a07.png" width="689" height="134">

We're already in a good place. We've split our single text blob into a message stream. However, we should probably do some clean up and make sure we get our message times in there. Let's start by coming up with a regular expression to extract and/or remove the chatter's name as well as the time. We'll just re-use our pattern for time: `\\([0-9:\\s]+(AM|PM)\\)`. But we need something for names. This is tricky—there are a lot of variations to names:

- First Last
- First last
- First
- First Middle Last
- First M. Last
- First O'Leary

I could go on; instead, I'll provide you with a regular expression that seems to work well for our data:

`[A-Z]{1}[a-z]+\\s[A-Za-z-]+\\s[A-Z]{1}[a-z]+:|[A-Z]{1}[a-z]+\\s[A-Za-z]+:|[A-Z]{1}[a-z]+:|[A-Z]{2}:`

With our time and name regular expressions in hand, we'll simply `regexp_replace` these patterns out of our message body. We'll also want to use these patterns to extract this information and bring it in as additional columns.

```sql
select id as ticket_id
  , num::int as message_sequence
  , (nullif(regexp_replace(start_at, '[0-9:]{2}\\s(AM|PM)\\sUTC', ''), '')::date || ' ' || regexp_substr(split_part(message, substring(md5(id), 1, 8), num::int), '([0-9:]{2}){3}[0-9]{2}\\s(AM|PM)'))::timestamp as message_at
  , replace(regexp_substr(split_part(message, substring(md5(id), 1, 8), num::int), '[A-Z]{1}[a-z]+\\s[A-Za-z-]+\\s[A-Z]{1}[a-z]+:|[A-Z]{1}[a-z]+\\s[A-Za-z]+:|[A-Z]{1}[a-z]+:|[A-Z]{2}:'), ':', '') as chatter
  , regexp_replace(split_part(message, substring(md5(id), 1, 8), num::int), '(\\([0-9:\\s]+(AM|PM)\\))|[A-Z]{1}[a-z]+\\s[A-Za-z-]+\\s[A-Z]{1}[a-z]+:|[A-Z]{1}[a-z]+\\s[A-Za-z]+:|[A-Z]{1}[a-z]+:|[A-Z]{2}:', '') as message_body
from (select id
        , regexp_substr(description, '[0-9\\s:-]+(AM|PM)\\sUTC') as start_at
        , regexp_replace(description, '(\\([0-9:\\s]+(AM|PM)\\))', ('\\ 1' || substring(md5(id), 1, 8))) as message
        , description
      from zendesk._tickets
      where via_channel = 'chat') as chat
join public.numbers
on numbers.num <= regexp_count(description, '(\\([0-9:\\s]+(AM|PM)\\))') + 1
```
<img src="https://discourse.looker.com/uploads/default/original/2X/6/6dabb923babdb053f0774a2ae1d4df144742eba5.png" width="690" height="284">

This is some dense SQL, but it's far less code than one might have to write using an imperative programming language. Before we move on to analytical patterns, let's consider the tradeoffs of using this pattern in a relational database versus making it part of the ETL process.

## Performance Test: Spark vs. Redshift

Before coming up with the above SQL pattern, I actually wrote a [Spark job](https://github.com/llooker/zendesk_chat_events) that splits chat text into a message stream. I thought it'd be useful to compare the performance of these two approaches. We're currently running a 10-node [dc1.large](https://aws.amazon.com/redshift/pricing/) Redshift cluster for internal reporting and analytics. We also have a 3-node [m3.xlarge](https://aws.amazon.com/ec2/instance-types/) Spark cluster running on EMR for various ETL and machine-learning tasks. Here are the stats for a single worker node in each one of our tools:

<table style="width:100%">
  <tr>
    <td><b>Tool</b></td>
    <td><b>Machine Type</b></td> 
    <td><b>vCPU</b></td>
    <td><b>Memory (GiB)</b></td>
    <td><b> Storage (SSD GB)</b></td>
  </tr>
  <tr>
    <td>Redshift</td>
    <td>dc1.large</td>
    <td>2</td>
    <td>15</td>
    <td>160</td>
  </tr>
    <td>Spark</td> 
    <td>m3.xlarge</td> 
    <td>4</td>
    <td>15</td>
    <td>80</td>
</table>

I'm a fairly patient person; however, the Redshift task was slow enough for me to ditch the rigor of a reasonable sample size! The mean completion time for the Redshift job was 3,720 seconds (_N_ = 2), while the mean completion time for the Spark job was 389 seconds (_N_ = 11).

There are a number of factors that, at least at first, seem to make the Spark job more favorable than the in-database approach:

1. It's faster;
1. It's cheaper;
1. It can easily be adapted to be an incremental job;
1. It can be complimented with various machine-learning and natural-language-processing tasks in the transform stage (_viz_., [Spark's Python API](http://spark.apache.org/docs/latest/api/python/) + [NLTK](http://www.nltk.org/));
1. We can declare explicit column encoding on the table into which we are copying (we issue a standard `create table` rather than a `create table as select ... `).

The points above, however, assume that our in-database approach is locked into using Looker PDTs/CTAS statements. Of course, this is not necessarily the case. We could easily script an `unload` job that transforms data inflight to S3 and then copies our data back into a dedicated table. In which case, we'd expect our differential in execution times to decrease substantially. Moreover, we'd be able to make our job incremental and even declare explicit column encoding. 

I'll also mention that cost in this situation is sort of comparing apples to oranges: our Spark cluster is used for one-off tasks, while our Redshift cluster serves an entire organization with on-demand analytics and reporting. Lastly, SQL is much more accessible than Java, Scala, or Python for many analysts. There's a convenience implication with leaving the transformation in SQL, particular when Looker makes it trivial to schedule this job to run when no one's using the cluster.

The gist of this performance comparison is this: simply put, Spark is likely faster than Redshift for batch jobs of this nature. Relational databases are not meant to wrangle unstructured data. However, we can see that, as far as an analyst's time is concerned, the pattern presented in this article lowers the barriers to deeper chat analysis dramatically.

In the next article, I'll walk through some useful analyses that build upon our chat events table.

**Example LookML Model**:
https://gist.github.com/githoov/0b9aea1ca87ff155e54a

**Public Spark Application**:

https://github.com/llooker/zendesk_chat_events
