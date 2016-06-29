####Introduction
At Looker, we recommend a few rules of thumb for Redshift table architecture and optimization (see [this][1] Discourse article written by Segah Meer). These rules of thumb, however, are just that—unless initially validated and subsequently monitored over time, they may have potentially detrimental impacts on query performance. This article provides a complimentary analysis to the above-sited article on table design, and is intended for the DBA, Data Engineer, or technical Analyst who can monitor cluster performance.

####The Setting
Consider a table of leads, moderate in size (144,073 rows). Suppose that converted leads map to accounts. We may frequently want to ask, "With which account is a particular lead associated?" In this case, because of the need to frequently join to account, we may declare the foreign key `converted_account_id` as the distribution key for this table. This can be done during the initial `CREATE TABLE` statement, or, as in our case, declared in a persistent derived table (PDT) definition within Looker using `distkey:`.


####Diagnostics
The following query can provide us with insight into how well our keys are behaving:

```
SELECT "schema" AS schema
	, "table" AS "table"
	, diststyle AS "distribution style"
	, "size" AS "table size" --in megabytes
	, unsorted AS "unsorted"
	, tbl_rows AS "total rows"
	, skew_sortkey1 AS "skew_sortkey"
	, skew_rows AS "skew_rows"
FROM svv_table_info
WHERE schema = 'looker_scratch' 
    AND "table" ILIKE 'lkr$%'
  AND skew_rows IS NOT NULL
ORDER BY skew_rows DESC
```

Inspecting the entry for our Lead PDT:

![image](https://discourse.looker.com/uploads/default/original/1X/2b1d83058cd75a58628e05b5c3a611684a8112b5.png)

We can see that the distribution style is set to **key** (distributing on `converted_account_id`), there are no issues with the table being unsorted, and that the skew in rows is 548.71. The latter metric tells us how well, or badly, the table is distributed and, therefore, how well it utilizes all of the nodes in the cluster when executing a variety of queries. Per the AWS Redshift [documentation][2], skew rows is defined as "[The] ratio of the number of rows in the slice with the most rows to the number of rows in the slice with the fewest rows." As the number departs from 1, skew increases. For the case of our Lead PDT, a skew of 543 might have a dramatic impact on performance.

####Benchmarks

To test the impact of skew, I've chosen two separate queries: the first queries the Lead table alone and contains an aggregate and group by. 
```
SELECT postal_code
	, COUNT(*) 
FROM looker_scratch.lkr$qcfr1a0086a5m1kiuzc_lead 
GROUP BY 1
```
The second query aggregates and groups by the same column, but forces a join to the Account table.
```
SELECT l.postal_code
	, COUNT(*) 
FROM looker_scratch.lkr$qcfr1a0086a5m1kiuzc_lead AS l 
LEFT JOIN salesforce._account AS a 
ON a.id = l.converted_account_id 
GROUP BY 1
```
After these benchmark queries (*N<sub>1</sub>* = *N<sub>2</sub>* = 40) are executed against the Lead table which is distributed on `converted_account_id`, we then execute them against the same table but with an **even** distribution style. The query execution times are logged, processed, and read into R for analysis.

While there doesn't seem to be a meaningful impact on query times for single-table queries across skewed or unskewed tables, the skewed table was 44% slower to execute, on average, than the evenly distributed table when considering the queries that forced a join. This may suggest that if distribution style isn't chosen wisely or well executed, it could have an impact on query performance, particularly when the query requires  network-intensive operations, such as joins.

![boxplot](https://discourse.looker.com/uploads/default/original/1X/8b37a74337544a5ccfd8b19a08bb3866ab2716c4.png)


----------

Here are the psql commands to execute the SQL scripts and pipe the query times into a txt file (note that each SQL script has as its first like `\timing on`).

```
psql -h <your_host> -p 5439 -d <your_database> -U <your_username> -f group_benchmark.sql | grep "^Time:" >> group_timing.txt

psql -h <your_host> -p 5439 -d <your_database> -U <your_username> -f group_join_benchmark.sql | grep "^Time:" >> group_join_timing.txt
```

The R code to reproduce the visualization and run a t-test:

```
library(ggplot2)

df <- read.csv(file = 'path_to_your_file', header = FALSE)

names(df) <- c('time', 'query_group')

ggplot(df, aes(factor(query_group), time)) + geom_boxplot() + ggtitle("Query Times by Distribution Style") + scale_x_discrete("Query Group", labels=c("Group By (skew)", "Group By (even)", "Group By & Join (skew)", "Group By & Join (even)")) + scale_y_continuous("Query Time (in milliseconds)")

sub.df <- df[1:80,]

t.test(sub.df$time ~ sub.df$query_group)
```

  [1]: https://discourse.looker.com/t/optimizing-redshift-performance/207
  [2]: http://docs.aws.amazon.com/redshift/latest/dg/r_SVV_TABLE_INFO.html
