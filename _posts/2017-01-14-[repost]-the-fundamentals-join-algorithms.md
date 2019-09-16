---
layout: post
title: "[repost] The fundamentals: join algorithms"
subtitle: ""
author: "kokosing"
header-img: "img/header.jpg"
catalog: true
tags:
  - java
  - presto
---

Origin Link: https://github.com/prestosql-rocks/site/blob/master/_posts/2017-01-14-the-fundamentals-join-algorithms.md

---

In previous [post](/internals/the-fundamentals-types-of-joins-in-sql/) I
explained how join works from the user point of view. Now it is the right time
to go one step deeper and learn how things are actually calculated. This a
very broad topic, so today we are going to just touch upon every join
algorithms used in Presto. To understand what they do and when they are used

You know already that on one axis join can be `INNER`, `OUTER`, `SEMI` etc.
Join execution is an independent axis. It means that type of join you are
using may not influence the way how it is executed. Actually, when you
change the type from `INNER` to `OUTER` not much changes in the plan and the
same join operators will be executed.

## Join sides

Join joins two tables, in terms of semantics, their appearance order in SQL does
not matter. Although in regards to the join algorithm, it may have a crucial
importance. It is common among different database internals publications that
relations used in join are called:

 * build table - is typically used to create an in-memory index which will be
 * asked for every probe table row. 
Usually, it has to be read completely before any of the probe table row is
streamed through the join.

 * probe table - is streamed through the join once build table is built.

Presto also follows this convention as well. If join reordering is disabled (no
cost-based or statistics-based optimizations are used), then left table is
a probe table and right table is a build table.

First advice caution: since build table is used to build an in-memory index,
rather use smaller tables as build tables. It means joins tables in below 
order (and not the other way round):

~~~ sql
SELECT * FROM huge_table_of_facts f JOIN small_table_of_dimmensions d ON f.x = d.x; 
~~~

## Join algorithms

Let's browse the joins execution types - the algorithms:
 
### Nested loop join
 
This is the simplest way of how join can be executed. It is a naive algorithm which uses two nested loops, one loop over one relation and second over another. For each row pairs join condition is evaluated. 
Pairs which satisfy the condition are returned. [^nested_loop]

As with everything, there is a good side and a bad side of this approach.
The good side is that it is easy, easy to understand and easy to execute.
The bad side of this algorithm is that it has to read one relation (build table) completely for every single row of other relation (probe table).
This gives `O(n^2)` algorithm time complexity [^time_complexity], which in regards to query execution means that when execution hits here, you experience that everything has just stopped. 
In most cases it is not acceptable, but there is one case where it is unavoidable unfortunately.
Do you remember `Cross join`? It unconditionally joins two relations, every single row from one relation is joined with every single row from the other. 
The only difference between `Cross join` and `Nested loop` is the join condition, which in case of `Cross join` is always satisfied.
This is the case where `Nested loop` join is used in Presto.
It will be used whenever you have a `CrossJoin` in the plan, as in the plan below:

~~~ sql
presto:tiny> explain select n.nationkey from nation n, region;
                                                       Query Plan                                                       
------------------------------------------------------------------------------------------------------------------------
 - Output[nationkey] => [nationkey:bigint] {rows: ?, bytes: ?}                                                          
     - RemoteExchange[GATHER] => nationkey:bigint {rows: 50, bytes: ?}                                                  
         - CrossJoin => [nationkey:bigint] {rows: 50, bytes: ?}                                                         
             - TableScan[tpch:tpch:nation:sf0.01, originalConstraint = true] => [nationkey:bigint] {rows: 25, bytes: ?} 
                     nationkey := tpch:nationkey                                                                        
             - RemoteExchange[REPLICATE] =>  {rows: 5, bytes: ?}                                                        
                 - TableScan[tpch:tpch:region:sf0.01, originalConstraint = true] => [] {rows: 5, bytes: ?}              
                                                                                                                        
(1 row)
~~~

Notice that `Cross join` does not need a join condition (as it is always satisfied - `TRUE`), so implementation of `Nested loop` in Presto does not have it.

Here there is no special in-memory index used for build table. It is just stored as it is.

I think there are two rules of the thumb here:

 * generally try to avoid a `CrossJoin`. It is expensive to execute (`O(n^2)` time complexity) as well as it is expensive to in regards to memory and network (it has to store or stream all the results somewhere). 

 * try to use it only for small tables. Take a look at this example. 1M of rows in one table and 2M of rows in the other table. Let's say that each row on average contains 128 bytes. In this case `Nested Loop` join will process 384MB of data, but it will output over 32GB of data, two orders of magnitude more.

### Hash join

The main drawback of `Nested Loop` is a very common problem in computer science. 
The problem can be stated as follows: how to find an element which satisfies the given condition in an array of elements and how to do it fast?
Hm.... it is not much better. But, if we simplify condition to equality condition then it is much simpler. 
Then the problem is: how to fast find an element in an array of elements? 
Yay... this problem has even its own name and it is called just `search` problem. 
`Nested loop` is using [linear search algorithm](https://en.wikipedia.org/wiki/Linear_search), 
but what if use an [hash table](https://en.wikipedia.org/wiki/Hash_table) in place of an array. 
Then we will get a hash join.

Once the build table is read, the hash table is created. 
Hash table is build for values in all the build table columns used in equality 
join conditions like `left.x = right.y AND left.z = right.w`.
Each of such equality conditions are called join equi criteria.  Then the probe
table is streamed. For each row of it, from columns which are listed in equi
criteria a hash value is calculated.  When a hash exists in the hash table,
then pairs with all the build table rows with same hash are checked if
join condition is satisfied, if so then such pairs are returned.

`Hash join` is the most common algorithm which is used for join in Presto. 
Any `INNER`, `OUTER` join which contains equi criteria will use it.

The time complexity in worst case is still `O(n^2)` as with `Nested loop` join,
but in common case it is expected to be about `O(n)`.

From the above you see that `Hash join` can be properly used only with at least
one equality condition that exists in join condition.  Otherwise all rows will
get the same hash, and you will almost get the same execution as with `Nested
loop`.
Equi criteria is also used in join data distribution, but this is a story for
another post.

### Merge join

Although this join method is not implemented in Presto, I think it would not be
good to omit this well known algorithm.

I am not sure how well do you feel with things like algorithms and data
structures, but there is another important algorithm in Computer Science. It is
called [merge sort](https://en.wikipedia.org/wiki/Merge_sort).
As name suggests it has two phases: merge and sort.  Sort is irrelevant for us
right now (well... maybe it isn't, but let take it off the table for a while).
Merge phase has a nice property, you can merge two sorted arrays into a sorted
array with `O(n)` time complexity [^time_complexity].

Ok, so how we can take advantage of it in join? Once a build table is read, it
is sorted by columns which appeared in equi criteria.  Then a probe table has
to be read and sorted.  After that, both tables are merged. However we do not
create a third sorted table as it is regular merge-sort algorithm, but evaluate
a join condition for each pairs with the same order position. If it is
satisfied then we return such pair as join result.

As you noticed this algorithm requires both build and probe tables to read
completely before doing the actual join.  This is unacceptable in the world of
`BIG DATA`.  So why do I talk about it?  Because you can never say never.
Notice that in case when by some accidence you have both build and probe tables
already sorted, this algorithm could be much faster than `Hash join` as it does
not need to calculate any hash value and do random data access.  To me it is
possible that Presto connector could return already sorted data and `Merge
join` could nicely leverage this. There is also a possibility to use it when data
is too big to fit into a hash table and it has to be spilled to disk. Though,
as usual, data spill is out of the todays topic, I hope to devote to it a separate post.

### Semi join

It is very similar to `Hash join`, but with some performance and semantic related modifications. 
 
 * It does not need to store all the build table in hash table as it does not need duplicated rows.
 * When a join condition is met, then the probe table row is simply returned

This method will be used whenever you will see `SemiJoin` in the output of the
`EXPLAIN` (I removed the boilerplate from the below query output):

```sql
presto:tiny> EXPLAIN SELECT nationkey FROM nation WHERE regionkey IN (SELECT regionkey FROM region);
                                      Query Plan                           
-------------------------------------------------------------------------------------------------
 - Output[...]
      ...
         - SemiJoin[...]
              ...
                 - ScanProject[..]
              ...
                 - ScanProject[...]
```

### Index join

It requires a connector to implement an [index provider](https://github.com/prestodb/presto/blob/master/presto-spi/src/main/java/com/facebook/presto/spi/connector/ConnectorIndexProvider.java).
Index provider is an entity which can return matching rows to given condition. 
It is assumed that it is able to do it quickly.

In that case build table is not used to build any complex data structures (index).
When a probe table is scanned, then the index is questioned to return matching build table rows to given probe table rows. 
It is performed in batched way i.e. index is asked for every `n` probe table rows.
And I am sorry, but I am not sure what happens next... probably a small hash table is created. 
Anyway, I will leave more details of `Index join` to a separate article.

As far as I recall, no open-sourced connectors is using this feature so you may not spot this in your life with Presto.

## Conclusion

Possibly there are more join algorithms, but in the case of Presto this is all
you need so far I think.  I hope you now have a deeper understanding what is
happening under the hood when your query contains join.
So far we covered join syntax (see the previous 
[article](/internals/the-fundamentals-types-of-joins-in-sql/)) and now join algorithms. 
Since Presto is a distributed system, in the next post I am going to cover the
way how data distributed during the join execution. So stay tuned, things are
getting more complicated and so more interesting.

[^nested_loop]:
    Here you can also read what Wikipedia can say about [Nested loop join](https://en.wikipedia.org/wiki/Nested_loop_join).

[^merge_join]:
    More information about [merge join](https://en.wikipedia.org/wiki/Sort-merge_join)

[^hash_join]:
    Go here to read about [hash join](https://en.wikipedia.org/wiki/Hash_join)

[^time_complexity]:
    Time complexity is a very broad and deep area of knowledge. To read more about you may want to visit [Wikipedia](https://en.wikipedia.org/wiki/Time_complexity)
    If you do not want to dive into this topic, just remember that:
     
    - `O(n^2)` means slow
    - `O(log(n) * n)` - neither slow or fast
    - `O(n)` - fast
    - `O(log(n))` - blazing fast
    - `O(1)` - indefinitely fast

