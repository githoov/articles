## Exploration Patterns
In [Part 1](https://discourse.looker.com/t/mining-text-with-sql-and-looker-part-1) of this series, we stepped through some text-parsing approaches that transform raw text into a format that is more palatable for analysis. In this article, we'll move on to some basic exploration patterns that help us get a better feel of distributions and long-term trends in our data. This will help us choose the most appropriate analytical models, which we will cover in the next (and final) article in this series. First, let's get a better feel of our data.

### Simultaneous Chats
Part of allocating support staff to meet demand means determining when peak hours are. If our business were restricted to a single timezone, we might expect peak hours to be mid morning or early afternoon. As the company grows into more regions, however, this may exacerbate already busy times of day, or it might change when peak hours are entirely.

We will get to the bottom of this using a two-stage query. First, we'll take the `chat_event` table, which we created in our last post, and build a fact table that includes start and end times for each chat.

```sql
select ticket_id
  , min(message_at) as chat_start
  , max(message_at) as chat_end
  , datediff('minute'
      , min(message_at) 
      , max(message_at)
    ) as chat_duration_minutes  
from public.chat_event
group by 1
```
<img src="https://discourse.looker.com/uploads/default/original/2X/7/7a52ec61b90f68d066e18b26848915ce07c2efb7.png" width="690" height="55">

Next we'll join our fact table into table of incremented minutes. Again, we are purposfully fanning out our table. In this instance, when we join our sequence of minutes between chat start and end times, we can see how many and which chats were active at any point in time.

```sql
select minutes.minute
  , ticket_id
from public.minutes
left join public.chat_event_facts
on minutes.minute between chat_event_facts.chat_start and chat_event_facts.chat_end
```
With this result set, we can use standard dimensions and measures to view the distribution of chats by hour of day and day of week. Let's review some long-term trends. 

First, as we may have guessed, most chats come in during normal business hours, with the mode being 11:00am.

<img src="https://discourse.looker.com/uploads/default/original/2X/2/23a777c1502e2eab07f25574a5951b24986f3cc8.png" width="690" height="297">

For non-weekend days, this pattern holds pretty firmly.

<img src="https://discourse.looker.com/uploads/default/original/2X/0/0595b803b0a10969cf676c52e7a7095471989fdb.png" width="690" height="307">

We can also get very detailed and view a minute-by-minute breakdown of tickets.

<img src="https://discourse.looker.com/uploads/default/original/2X/b/b656cdf65c5fa66fec584ac21ce54f65fa48c8fe.png" width="690" height="308">

The 4-week moving average of hourly simultaneous chats is on an upward trend, but took a dip during the holidays.

<img src="https://discourse.looker.com/uploads/default/original/2X/7/7227ea2b3c91c99bea2756a8393758c65a583170.png" width="689" height="307">

### Chat Duration
Relying on the dimension we calculated in `chat_event_facts`, we can view a distribution of `chat_duration_minutes`. We can see that this is non-normal (it looks a lot like the shape of an Exponential distribution). When it comes time to do some analysis on this data, we might choose a distribution that's well-suited for modeling concurrent events or durations (_e.g._, Erlang or Poisson). We'll also need to make sure we use the moments associated with the appropriate distribution when doing further exploratory analysis, as the wrong choice of metric can be misleading.

<img src="https://discourse.looker.com/uploads/default/original/2X/4/45a16b51e6fe74a389e0e2fd847739f84da4c044.png" width="690" height="301">

### Cadence
As a crude measure of engagement, I'm going to define the time between question and response between different chat participants as "cadence." This way we can see if certain chat agents are slow to reply, within a normal range, or if they tend to rush conversations. Perhaps this will give us some indication of quality of support service. If so, perhaps we'll be able to discern what sort of load a typical support agent can maintain before service level takes a hit. 

Before we get to a ticket-level statistic, let's take a look at the distribution of time to reply (in seconds) between differing chat participants.

<img src="https://discourse.looker.com/uploads/default/original/2X/a/a1d546b91ef171dd497d77ddbc98b1cc92227282.png" width="690" height="329">

Once again, we've got skew in our distribution, which means a metric that makes use of the average might not work for us. Instead, let's use median, instead of mean, as our measure of center and interquartile range, instead of variance, as a measure for spread.


```sql
select ticket_id
  , is_looker
  , min(median_time_to_reply) as median_time_to_reply
  , min(third_quartile_time_to_reply) as third_quartile_time_to_reply
  , min(first_quartile_time_to_reply) as first_quartile_time_to_reply
from (select event_id
        , ticket_id
        , is_looker
        , median(difference_in_seconds) over(partition by ticket_id) as median_time_to_reply
        , percentile_cont(0.75) within group(order by difference_in_seconds) over(partition by ticket_id, is_looker) as third_quartile_time_to_reply
        , percentile_cont(0.25) within group(order by difference_in_seconds) over(partition by ticket_id, is_looker) as first_quartile_time_to_reply
      from (select event_id
              , ticket_id
              , created_at
              , participant
              , participant != chat_initiator as is_looker
              , next_message_participant
              , datediff('seconds', created_at, lead(created_at) over(partition by ticket_id order by created_at)) as difference_in_seconds
            from (select event_id
                    , ticket_id
                    , created_at
                    , participant
                    , first_value(participant) over(partition by ticket_id order by created_at rows between unbounded preceding and unbounded following) as chat_initiator
                    , lead(participant) over(partition by ticket_id order by created_at) as next_message_participant
                    , body
                  from public.chat_events)
            where participant != next_message_participant))
group by 1,2
```
When filtering only for messages by Looker chat agents, we can see that Adam has many chats with a relatively high median time to reply, while Annie skews toward shorter median times to reply. 

<img src="https://discourse.looker.com/uploads/default/original/2X/e/e09a6bece850d1c1019a5de227409800647deed2.png" width="690" height="308">

A similar pattern holds when looking at our measure of spread.
 
<img src="https://discourse.looker.com/uploads/default/original/2X/4/49b6ca24a6b7cc70f0b0b5049c01a60aee8183b2.png" width="690" height="306">

These metrics can be useful in identifying chat agents who need re-training, particularly when SLAs are in place and these differences are significant. Of course, without knowing more about the nature of the chats Adam and Annie field, respectively, we cannot rule out that Adam is an escalation analyst and, in fact, takes on more challenging tickets that require interfacing with other departments, etc.

### Chat Symmetry

```sql
      select ticket_id
        , round(1.0 * sum(case when participant != chat_initiator then 1 else 0 end) / nullif(sum(case when participant = chat_initiator then 1 else 0 end), 0), 1) as symmetry
      from (select event_id
              , ticket_id
              , participant
              , first_value(participant) over(partition by ticket_id order by created_at rows between unbounded preceding and unbounded following) as chat_initiator
            from public.chat_events)
      group by 1
```
<img src="https://discourse.looker.com/uploads/default/original/2X/c/c282a9d777d6fdbf5612f09acb2ffc31a52bba29.png" width="690" height="305">

As symmetry goes toward zero from 1, the chat is dominated by the customer, as it increases above 1, the chat skews toward the chat agent; 1 indicates perfect one-to-one chat symmetry.
