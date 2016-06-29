## Introduction
Few SQL dialects support robust table-generating functions. Some common use cases for table-generating functions are creating a sequence of numbers or dates, un-nesting a JSON object or array into rows, or splitting a string on a delimiter into rows. 

In this brief article, I'll demonstrate the latter example: how to take a string containing _n_ distinct comma-separated values and split it into _n_ unique rows.

## Setup
Suppose we have a table of products containing `id`, `name`, and `tags`.

### _products_
<table style="width:100%">
  <tr>
    <td>id</td>
    <td>name</td> 
    <td>tags</td>
  </tr>
  <tr>
    <td>123</td>
    <td>Scrunch Cloth Pants Set</td> 
    <td>Pants, Throwback, 80s</td>
  </tr>
  <tr>
    <td>1287</td>
    <td>Le Suit Citrus Breeze Skirt Suit</td> 
    <td>Skirts, Dresses, Business Casual</td>
  </tr>
  <tr>
    <td>486</td>
    <td>MaxStudio Plaid Romper</td> 
    <td>Jumpsuits, Casual, Rompers</td>
  </tr>
</table>
Our goal is to un-nest the tags to create a product-tag mapping table. I am assuming a table of numbers already exists in the database (though this can be created using **[this pattern](https://discourse.looker.com/t/sql-generate-series-for-mysql-and-redshift/482)**).

## Approach
Two functions are useful here: first, the `split_part` function, which takes a string, splits it on some delimiter, and returns the first, second, ... , _n_th value specified from the split string; second, `regexp_count`, which tells us how many times a particular pattern is found in our string.

Our strategy is to (i) determine how many instances of our delimiter (", ") are found for each string; (ii) add one to that value, telling us how many values are separated by our delimiter; (iii) join in our numbers table on an inequality such that we fan out our tagged products _n_ times, where _n_ is the the value returned from (ii) above for each product; (iv) use the `split_part` function and our sequence of numbers to get the 1st, 2nd, _n_th tags for each product.

## SQL

```SQL
select row_number() over(order by 1) as product_tag_id
  , products.id as product_id
  , split_part(products.tags, ', ', numbers.num) as tag_name
from products
join numbers
on numbers.num <= regexp_count(products.tags, ',\\s') + 1      
```
<table style="width:100%">
<tr> <td> product_tag_id </td>	<td> product_id </td>	<td> tag_name </td>
<tr> <td> 1 </td>	<td> 123 </td>	<td> Throwback </td>
<tr> <td> 2 </td>	<td> 123 </td>	<td> Pants </td>
<tr> <td> 3 </td>	<td> 123 </td>	<td> 80s </td>
<tr> <td> 4 </td>	<td> 1287 </td> <td> 	Skirts </td>
<tr> <td> 5 </td>	<td> 1287 </td> <td> 	Dresses </td>
<tr> <td> 6 </td>	<td> 1287 </td> <td> 	Business Casual </td>
<tr> <td> 7 </td>	<td> 486 </td>	<td> Casual </td>
<tr> <td> 8 </td>	<td> 486 </td>	<td> Jumpsuits </td>
<tr> <td> 9 </td>	<td> 486 </td>	<td> Rompers </td>
</table>

That's it! Super easy!
