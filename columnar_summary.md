## Introduction

Encoding is an important concept in columnar databases, like Redshift and Vertica, as well as database technologies that can ingest columnar file formats like Parquet or ORC. Particularly for the case of Redshift and Vertica—both of which allow one to declare explicit column encoding during table creation—this is an key concept to grasp. In this article, we will cover (i) how columnar data differs from traditional, row-based RDBMS storage; (ii) how column encoding works, generally; then we'll move on to discuss (iii) the different encoding algorithms and (iv) when to use them.

## Row Storage

Most databases store data on disk in sequential blocks. When data are queried, the disk is scanned and the data is retrieved. With traditional RDMBS (_e.g._, MySQL or PostgreSQL), data are stored in rows—multiple rows for each block. Consider the table `events`:

<table>
<thead>
<tr>
<th><code>id</code></th>
<th><code>user_id</code></th>
<th><code>event_type</code></th>
<th><code>ip</code> </th>
<th><code>created_at</code> </th>
<th><code>uri</code> </th>
<th><code>country_code</code> </th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>1</td>
<td>search</td>
<td>122.303.444</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B0%7D" alt="equation"></td>
<td>/search/products/filters=tshirts</td>
<td>US</td>
</tr>
<tr>
<td>2</td>
<td>1</td>
<td>view</td>
<td>122.303.444</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B1%7D" alt="equation"></td>
<td>/products/id=11212</td>
<td>US</td>
</tr>
<tr>
<td>3</td>
<td>1</td>
<td>add_to_cart</td>
<td>122.303.444</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>/add_to_cart/product_id=11212</td>
<td>US</td>
</tr>
<tr>
<td>4</td>
<td>3</td>
<td>search</td>
<td>121.321.484</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B1%7D" alt="equation"></td>
<td>/search/products/filters=pants</td>
<td>FR</td>
</tr>
<tr>
<td>5</td>
<td>3</td>
<td>search</td>
<td>121.321.484</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>/search/products/filters=“hammer pants”</td>
<td>FR</td>
</tr>
<tr>
<td>6</td>
<td>3</td>
<td>search</td>
<td>121.321.484</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B3%7D" alt="equation"></td>
<td>/search/products/filters=“parachute pants”</td>
<td>FR</td>
</tr>
<tr>
<td>7</td>
<td>9</td>
<td>checkout</td>
<td>121.322.432</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>/checkout/cart=478/status=confirmed</td>
<td>UK</td>
</tr>
</tbody>
</table>
With traditional RDBMS, these rows will be stored as tuples, with each new row delimited in some fashion (in this case, I'm using semicolon and a line break for illustrative purposes):
```sql
1,1,search,122.303.444,t_{0},/products/search/filters=tshirts,US; --row 1
2,1,view,122.303.444,t_{1},/products/id=11212,US; --row 2
...;
7,9,checkout,121.322.432,t_{2},/checkout/cart=478/status=confirmed,UK; --row 7
```
Indexing speeds up various operations because the database can skip over entire blocks very quickly, minimizing seek. Let's assume there was an index on `user_id`. Let's also assume that the 3 events for user 1 were stored in block a, the 3 events for user 3 were stored in block b, and the 1 event for user 9 was stored in block c. Finally, let's suppose we issue the following query (Query 1) to the database: 

```sql
select * 
from events 
where user_id = 3
```
In this case, our index would allow the database to skip the first block, fetch from the second block, and stop there in order to get the data it needs for this query.

Now, suppose we issue a more analytical query (Query 2) to this database: 
```sql
select 
    created_at::date as created_date
    , count(1) as search_events 
from events 
where event_type = 'search' 
group by 1 
order by 1
```

In this case, we don't really benefit from our index on `user_id`, and we don't have an index declared on `event_type`, so our query will scan all of the rows across all of the blocks for the information it needs, identifying predicate matches on rows 1, 4, 5, and 6, then performing the aggregation and group by. This would be a relatively expensive operation on a large data set.

## Columnar Storage

In columnar databases and in columnar file formats, data are transformed such that values of a particular column are stored adjacently on disk. Revisiting how our data is stored, this time in columnar format, we'd expect it to look like this:

```sql
1,2,3,4,5,6,7; --id
1,1,1,3,3,3,9; --user_id
search,view,add_to_cart,search,search,search,checkout; --event_type
...; --misc columns
US,US,US,FR,FR,FR,UK; --country_code
```
Again, the contiguous values in a column can be co-located in the same blocks. Let's assume that `id`, `user_id`, and `event_type` are located in block a, `ip` and `created_at` are located in block b, and `uri` and `country` are located in block c. 

Revisiting Query 2 above, we can see that the database would only need to access the data in blocks a and b in order to execute our query (and in both cases, we'd only need to scan a subset of the data in each block). Conversely, Query 1 would require that the database scan all blocks on disk to marshall the values of every column associated with the rows that match our filter predicate.

We can see, when comparing these two types of data stores, that the clear benefit with row-base storage comes from operations that query, update, insert, or delete many columns for _specific_ rows or ranges of rows, provided indexes are in place. This is why these database technologies remain so prominent for OLTP workloads. Alternatively, columnar storage favors queries that operate on a subset of columns, typically accompanied by aggregations and filters. The benefits of columnar databases are improved by introducing the notion of compression or encoding.

## Column Encoding

Because row-based storage may often store heterogeneous data adjacently, compression options are limited. In our example, in a given row, the `user_id`, an integer, is stored directly next to `event_type`, a character, meaning a compression algorithm that's particularly well suited for integers cannot be used on the entire row, and the same holds for a compression algorithm that's particularly well suited for strings. Column stores, however, are bound to store relatively homogeneous data adjacently—at the very least, the explicit type of data is the same. This fact allows us to apply type-specific encoding, reducing the data's impact on space and improving retrieval speeds when querying the data. Revisiting our columnar data stored on disk, let's assume that we apply type-specific encoding and ignore the specifics of implementation for the time being. 

Conceptually, our data would start like this:
```
1,2,3,4,5,6,7;
1,1,1,3,3,3,9;
search,view,add_to_cart,search,search,search,checkout;
...;
US,US,US,FR,FR,FR,UK;
```
and, once compressed, reduce to this:
```
1,2,3,4,5,6,7;
1,3,9;
search,view,add_to_cart,checkout;
...;
US,FR,UK;
```
Obviously, this is not the full story; but this simplistic example should convey how columnar storage coupled with some form of encoding can reduce the amount of data stored on disk, sometimes dramatically.

## Encoding Types

Let ![equation](https://latex.codecogs.com/gif.latex?%7CS%7C)  denote the cardinality of a column, where cardinality is defined as the number of elements in a set. For example, the set ![equation](https://latex.codecogs.com/gif.latex?S%3D%20%5C%7B%5Ctextup%7BUS%2C%20FR%2C%20UK%7D%5C%7D), derived from the tuple, ![equation](https://latex.codecogs.com/gif.latex?T%20%3D%20%28%5Ctextup%7BUS%2CUS%2CUS%2CFR%2CFR%2CFR%2CUK%7D%29), has cardinality ![equation](https://latex.codecogs.com/gif.latex?%7CS%7C%20%3D%203). Each of the encoding types listed below represents a _class_ of encoding. Which is to say, there may be implementations, or codecs, for each class that can be applied to a range of data types. (In the following examples, the visualizations are meant to convey the general idea of the encoding type and may not accurately represent the specifics of _how_ the data are stored on disk.)

### Runlength

Runlength encoding reformats the data to contain the value itself accompanied by the number of times that value is repeated on subsequent rows (_i.e._, the length of its run) until interrupted by a new value. While we might expect there to be ranges in the data where we have alternating values, row over row, for random or deterministic but ordered processes, we'd expect to see runs of repeated values.

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%201%20%5C%5C%201%20%5C%5C%201%20%5C%5C%203%20%5C%5C%203%20%5C%5C%203%20%5C%5C%209%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%20%281%2C3%29%20%5C%5C%20%283%2C3%29%20%5C%5C%20%289%2C1%29%20%5Cend%7Bpmatrix%7D)

### Dictionary

With dictionary encoding, the general idea is that a hash-map or dictionary is created, where the set of original values is mapped to an integer. Then all of the original data are replaced with the corresponding integer. Particularly in the case of strings, the integer replacements compress to relatively few bytes.

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%20%5Ctextup%7BUS%7D%20%5C%5C%20%5Ctextup%7BUS%7D%20%5C%5C%20%5Ctextup%7BUS%7D%20%5C%5C%20%5Ctextup%7BFR%7D%20%5C%5C%20%5Ctextup%7BFR%7D%20%5C%5C%20%5Ctextup%7BFR%7D%20%5C%5C%20%5Ctextup%7BUK%7D%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%200%20%5C%5C%200%20%5C%5C%200%20%5C%5C%201%20%5C%5C%201%20%5C%5C%201%20%5C%5C%202%20%5Cend%7Bpmatrix%7D%20%5Ccup%20%5Cbegin%7BBmatrix%7D%200%20%5CRightarrow%20%5Ctextup%7BUS%7D%20%5C%5C%201%20%5CRightarrow%20%5Ctextup%7BFR%7D%20%5C%5C2%20%5CRightarrow%20%5Ctextup%7BUK%7D%20%5Cend%7BBmatrix%7D)

### Delta

Delta encoding starts off by copying the first entry in the column. Subsequent values, however, are populated by taking a difference between the current value and the preceding value. We can see that this has the potential to take potentially large values (in terms of bytes) and reduce their size dramatically. In our simple example, to get the value associated with any point in the column, we simply sum all preceding values.

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%201%20%5C%5C%201%20%5C%5C%201%20%5C%5C%203%20%5C%5C%203%20%5C%5C%203%20%5C%5C%209%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%20%5Cmathbf%7B1%7D%20%5C%5C%201%20-%201%20%5C%5C%201%20-%201%20%5C%5C%203%20-%201%20%5C%5C%203%20-%203%20%5C%5C%203%20-%203%20%5C%5C%209%20-%203%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%20%5Cmathbf%7B1%7D%20%5C%5C%200%20%5C%5C%200%20%5C%5C%202%20%5C%5C%200%20%5C%5C%200%20%5C%5C%206%20%5Cend%7Bpmatrix%7D)

### Sliding-Window Codecs (LZ77, LZSS, LZO, DEFLATE, etc.)

The basic idea with this class of algorithm is that runs of characters previously seen are replaced with a _reference_ to said previously seen run. The reference itself is some sort of tuple containing the offset (_i.e._, starting position), the run length, and (optionally) the deviation. Consider the following stream of text:
```txt
foo bar
bar baz
foobarbaz
```
The encoded version might look something like this:
```txt
foo bar
(4,3)baz
(0,3)(4,3)(8,3)
```
While this example might not look like much space is conserved, the savings can be considerable on larger streams of data.

## Encoding Strategies

### Runlength

For data whose ![equation](https://latex.codecogs.com/gif.latex?%7CS%7C%20%5Cll%20N) we might reasonably expect to see many runs, making this a good choice of encoding. Typically, this might be a boolean, binary categorical (_e.g._, `male`, `female`), or low-cardinality categorical (_e.g._, `graduate degree`, `bachelors`, `some college`, `high school`) column.

### Dictionary

Dictionary encoding works well when the cardinality is a bit larger than what's appropriate for runlength encoding, and not too large so as to make a massive dictionary yielding slow lookups. Also, the values in the dictionary, in most implementations, have a fairly limited length, so larger varchars tend to be poor choices here. As a loose rule, I recommend a cardinality between 10 and 10,000. Dictionary encoding is particularly well suited for states, countries, job titles, etc.

### Delta

Delta encoding can be applied to a variety of data types. In our simplistic example, we can see that monotonically increasing or decreasing values would likely compress down dramatically. There are delta codecs that are meant to handle potentially large row-over-row differences—_e.g._, `delta32k` in Redshift..

### Sliding Window

This class of algorithm is very well suited for longer streams of text. For example, raw text from documents or support chats, browser agents, etc.

## Further Considerations

### Tuple Reconstruction

Wherever possible, the database is going to perform operations on a column-by-column basis rather than on tuples. This allows for tight, CPU-friendly `for` loops, improved parallelization, and vectorization. On the other hand, when a client renders a result set or when a result set is returned via JDBC or ODBC, the individual columns need to be reconstructed into tuples in order to display the results in a tabular format. To achieve this, modern columnar databases are likely to employ some form of late materialization or tuple reconstruction. 

The general idea is this: most execution engines will perform filters on the columns in the `where` clause, one by one, noting the row positions of the filter-predicate matches in each step, passing the positions along to the next filter operation. When it comes time to join—and assuming some filtering in the query is actual present—only a subset of rows from each table is used in the join. The join itself only requires the column(s) explicitly included in the join predicate. When filtering and joining is done, the execution engine is left with a number of arrays, one for each table in the join, together creating the positional references for the ultimate result set. At this point, the relevant attribute values from the columns across the tables are stitched together to create the tuples that make up the result set. 

To illustrate, suppose that we continue with our example `events` table and also add a user look-up table, `users`, so that our schema now looks like this:

[`events`]
<table style="width:100%">
<thead>
<tr>
<th><code>id</code></th>
<th><code>user_id</code></th>
<th><code>event_type</code></th>
<th><code>ip</code> </th>
<th><code>created_at</code> </th>
<th><code>uri</code> </th>
<th><code>country_code</code> </th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>1</td>
<td>search</td>
<td>122.303.444</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B0%7D" alt="equation"></td>
<td>/search/products/filters=tshirts</td>
<td>US</td>
</tr>
<tr>
<td>2</td>
<td>1</td>
<td>view</td>
<td>122.303.444</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B1%7D" alt="equation"></td>
<td>/products/id=11212</td>
<td>US</td>
</tr>
<tr>
<td>3</td>
<td>1</td>
<td>add_to_cart</td>
<td>122.303.444</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>/add_to_cart/product_id=11212</td>
<td>US</td>
</tr>
<tr>
<td>4</td>
<td>3</td>
<td>search</td>
<td>121.321.484</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B1%7D" alt="equation"></td>
<td>/search/products/filters=pants</td>
<td>FR</td>
</tr>
<tr>
<td>5</td>
<td>3</td>
<td>search</td>
<td>121.321.484</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>/search/products/filters=“hammer pants”</td>
<td>FR</td>
</tr>
<tr>
<td>6</td>
<td>3</td>
<td>search</td>
<td>121.321.484</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B3%7D" alt="equation"></td>
<td>/search/products/filters=“parachute pants”</td>
<td>FR</td>
</tr>
<tr>
<td>7</td>
<td>9</td>
<td>checkout</td>
<td>121.322.432</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>/checkout/cart=478/status=confirmed</td>
<td>UK</td>
</tr>
</tbody>
</table>

[`users`]
<table class="table table-striped table-bordered" style="width:100%" >
<thead>
<tr>
<th><code>id</code></th>
<th><code>created_at</code></th>
<th><code>age</code></th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B0%7D" alt="equation"></td>
<td>32</td>
</tr>
<tr>
<td>2</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B0%7D" alt="equation"></td>
<td>29</td>
</tr>
<tr>
<td>3</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B1%7D" alt="equation"></td>
<td>21</td>
</tr>
<tr>
<td>4</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>38</td>
</tr>
<tr>
<td>5</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>33</td>
</tr>
<tr>
<td>6</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B2%7D" alt="equation"></td>
<td>32</td>
</tr>
<tr>
<td>7</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B3%7D" alt="equation"></td>
<td>27</td>
</tr>
<tr>
<td>8</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B4%7D" alt="equation"></td>
<td>26</td>
</tr>
<tr>
<td>9</td>
<td><img src="https://latex.codecogs.com/gif.latex?t_%7B5%7D" alt="equation"></td>
<td>30</td>
</tr>
</tbody>
</table>

Now, let's suppose that we issue the following query.
```sql
select count(1) as search_events
from   events
join   users
on     events.user_id = users.id
where  events.event_type = 'search'
and    events.created_at < t_{3}
and    users.age < 30
```

Here's how our database might process this query, step by step:

Step 1—Filter. Create a bit string or array that references the rows in `events.event_type` that equal `'search'`. We will refer to this as our "position list."

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%20%5Ctextup%7Bsearch%7D%20%5C%5C%20%5Ctextup%7Bview%7D%20%5C%5C%20%5Ctextup%7Badd%5C_to%5C_cart%7D%20%5C%5C%20%5Ctextup%7Bsearch%7D%20%5C%5C%20%5Ctextup%7Bsearch%7D%20%5C%5C%20%5Ctextup%7Bsearch%7D%20%5C%5C%20%5Ctextup%7Bcheckout%7D%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%201%20%5C%5C4%20%5C%5C5%20%5C%5C6%20%5Cend%7Bpmatrix%7D)

Step 2—Reconstruct. We have to filter on our second condition against the `events` table, but we also know, from our previous step, that we only have to check our second condition on a subset of rows. In other words, before we filter on the entire `events.created_at` column, we can and should reduce our search to a subset of rows found in our position list.

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%201%20%5C%5C4%20%5C%5C5%20%5C%5C6%20%5Cend%7Bpmatrix%7D%20%5Crightarrow%20%5Cbegin%7Bpmatrix%7D%20t_%7B0%7D%20%5C%5C%20t_%7B1%7D%20%5C%5C%20t_%7B2%7D%20%5C%5C%20t_%7B1%7D%20%5C%5C%20t_%7B2%7D%20%5C%5C%20t_%7B3%7D%20%5C%5C%20t_%7B2%7D%20%5Cend%7Bpmatrix%7D%20%5Crightarrow%20%5Cbegin%7Bpmatrix%7D%20t_%7B0%7D%20%5C%5C%20t_%7B1%7D%20%5C%5C%20t_%7B2%7D%20%5C%5C%20t_%7B3%7D%20%5Cend%7Bpmatrix%7D)

Step 3—Filter. Now we can evaluate the second filter, `events.created_at < t_{3}`, again returning a position list considering both of the preceding filter predicates.

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%20t_%7B0%7D%20%5C%5C%20t_%7B1%7D%20%5C%5C%20t_%7B2%7D%20%5C%5C%20t_%7B3%7D%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%201%20%5C%5C%204%20%5C%5C%205%20%5Cend%7Bpmatrix%7D)

Step 4—Filter. Our final step before joining is obtaining a position list for the `users` table based on the filter `users.age < 30`.

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%2032%20%5C%5C29%20%5C%5C21%20%5C%5C38%20%5C%5C33%20%5C%5C32%20%5C%5C27%20%5C%5C26%20%5C%5C30%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%202%20%5C%5C3%20%5C%5C7%20%5C%5C8%20%5Cend%7Bpmatrix%7D)

Step 5—Join. We know that our join involves two columns: `events.user_id` and `users.id`. We also have two position lists from our filters that restrict the possible rows in play and, therefore, the values we have to match in the join. (In the join illustration, we have both the position index and the value itself, indicated by _i_ and _v_ like so: ![equation](https://latex.codecogs.com/gif.latex?%28i%2Cv%29))

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%20%281%2C1%29%20%5C%5C%284%2C3%29%20%5C%5C%285%2C3%29%20%5Cend%7Bpmatrix%7D%20%5CJoin%20%5Cbegin%7Bpmatrix%7D%20%282%2C2%29%20%5C%5C%20%283%2C3%29%20%5C%5C%20%287%2C7%29%20%5C%5C%20%288%2C8%29%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%20%284%2C3%29%20%5C%5C%285%2C3%29%20%5Cend%7Bpmatrix%7D%20%5Cwedge%20%5Cbegin%7Bpmatrix%7D%20%283%2C3%29%20%5Cend%7Bpmatrix%7D)

At this point, we have the row positions that we'll need from both tables in order to project any column from either table. Our original example `select count(1) from ...` would yield 2. We could easily turn this into something like `select users.age, events.country_code from ...` and have our row positions projected onto these two columns in order to extract the information we need, yielding:

<table class="table table-striped table-bordered">
<thead>
<tr>
<th><code>age</code></th>
<th><code>country_code</code></th>
</tr>
</thead>
<tbody>
<tr>
<td>21</td>
<td>FR</td>
</tr>
<tr>
<td>21</td>
<td>FR</td>
</tr>
</table>

This is the basic idea of late materialization in tuple reconstruction. There are early materialization approaches where the tuples are stitched together much earlier in the process, such that later operations are applied to the entire tuple. In many situations, late materialization improves performance. We can see, however, that it is associated with many join-like operations between the position lists and the values in the columns themselves. Good cost-based optimizers may weigh both late and early materialization plans to see which has a lower overall cost. In the case that large tables get filtered down to reasonably small result sets, late materialization may prove to be the faster option.

### Operations on Compressed Data

Column stores may have the ability to perform certain operations on compressed data, yielding even better performance in certain circumstances. In other cases, operations must first decompress the data before it can be used. One must be mindful of the columns they choose to compress, with particular consideration for _how_ the data will likely be used.

For columns that only ever appear in the projection or restriction, and which are not used for joins, the execution engine may be able to work on the compressed data. For example, if we use runlength encoding on a column, the average could be computed using strictly the compressed data using the following formula: 

![equation](https://latex.codecogs.com/gif.latex?%5Cfrac%7B%5Csum_%7Bj%3D1%7D%5EN%20v_%7Bj%7Do_%7Bj%7D%7D%7B%5Csum_%7Bj%3D1%7D%5EN%20o_%7Bj%7D%7D)

where ![equation](https://latex.codecogs.com/gif.latex?%28v_%7Bj%7D%2C%20o_%7Bj%7D%29) captures the value, _v_, of a given row and the number of occurrences, _o_, of that value on following rows.

More concretely, if we had a column of ages, our average could be computed like so:

![equation](https://latex.codecogs.com/gif.latex?%5Cbegin%7Bpmatrix%7D%2030%20%5C%5C%2030%20%5C%5C%2021%20%5C%5C%2031%20%5C%5C%2030%20%5C%5C%2027%20%5C%5C%2040%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cbegin%7Bpmatrix%7D%20%2830%2C%202%29%20%5C%5C%20%2821%2C%201%29%20%5C%5C%20%2831%2C%201%29%20%5C%5C%20%2830%2C%201%29%20%5C%5C%20%2827%2C%201%29%20%5C%5C%20%2840%2C%201%29%20%5Cend%7Bpmatrix%7D%20%5CRightarrow%20%5Cfrac%7B%2830*2&plus;21&plus;31&plus;30&plus;27&plus;40%29%7D%7B%282&plus;1&plus;1&plus;1&plus;1&plus;1%29%7D)

If a column is used in a join, however, the execution engine will almost certainly have to decompress it before it can be used. This is expensive and should be avoided if possible. For Redshift, similar complications make compressing sort and distribution keys a risky proposition.

### Column Metadata

Modern columnar databases, as well as columnar formats, will store column meta data in the file(s) along with the column data itself. This allows certain operations to avoid operating on the underlying data altogether, potentially improving performance. 

Let's suppose that each block on disk maps to one file and contains some number of columns. Furthermore, let's assume that each column is accompanied by the following summary statistics: min, max, and count. (Obviously, some stats only make sense for certain data types, but we'll ignore that detail for the time being.) If I were to issue the following query

```sql
select max(some_column)
from my_table
```

conceivably, we could get this figure by taking a `max` of the "max" summary statistic from this column. Similarly, if we issued 

```sql
select count(1)
from my_table
where some_column > x
```

we could quickly discard blocks of data whose "max" summary statistic did not exceed _x_.

*Note: For this reason, it's important to run `analyze` on tables regularly within Redshift, as it updates column meta data.*
