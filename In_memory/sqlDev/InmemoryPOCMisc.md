## Introduction
   This lab introduces complimentary database features that can be combined with In-Memory to provide additional performance gains to analytic queries.

## Approximate Query Processing and In-Memory.

While InMemory is highly efficient in query filtering, min-max pruning and In-Memory indexing, the query can further improve performance using "Approximate Query Processing" as a complimentary feature since 12c. They are features to improve performance of sort operations like DISTINCT, RANK, PERCENTILE, MEDIAN . These operations happen in the Process Global Area (PGA) of a user sessions. Since 12c, oracle has "approximate query processing" features that can make sorting operations faster but can be 95% or more accurate. For more information about this feature check out the  **[documentation.](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/query-optimizer-concepts.html#GUID-6273DFAC-7C4D-4540-AE11-B6973F237323)**
This type of approximation is useful in certain Dashboards, BI applications where data needs to be real time, but small variance in dataset does not distort the visualizations of data.

In 12.2 there are several APPROX functions introduced:
- APPROX_COUNT_DISTINCT_DETAIL
- APPROX_COUNT_DISTINCT_AGG
- TO_APPROX_COUNT_DISTINCT
- APPROX_MEDIAN
- APPROX_PERCENTILE
- APPROX_PERCENTILE_DETAIL
- APPROX_PERCENTILE_AGG
- TO_APPROX_PERCENTILE

 “Approximate query processing is primarily used in data discovery applications to return quick answers to explorative queries. Users typically want to locate interesting data points within large amounts of data and then drill down to uncover further levels of detail. For explorative queries, quick responses are more important than exact values.”

The interesting part is that you can utilize the Approximate functions without changing code. There are three initialization parameters introduced to control which functions should be treated as an approximate function during run time.

The initialization parameters are:

- approx_for_aggregation (TRUE|FALSE)
- approx_for_count_distinct (TRUE|FLASE)
- approx_for_percentile  (ALL| NONE | PERCENTILE_CONT | PERCENTILE_CONT DETERMINISTIC | PERCENTILE_DISC | PERCENTILE_DISC DETERMINISTIC | ALL DETERMINISTIC )

To test this, let use first run the below query from sql-Dev tool "InMemory Query vs Buffer"
````
<copy>
select lo_custkey, median(lo_ordtotalprice) from lineorder
group by  lo_custkey ;
</copy>
IN MEMORY PLAN
Plan hash value: 4230308389

-------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name      | Rows  | Bytes |TempSpc| Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |           | 80504 |   864K|       | 39801   (3)| 00:00:02 |
|   1 |  SORT GROUP BY              |           | 80504 |   864K|   459M| 39801   (3)| 00:00:02 |
|   2 |   TABLE ACCESS INMEMORY FULL| LINEORDER |    23M|   251M|       |  2045  (12)| 00:00:01 |
-------------------------------------------------------------------------------------------------

Run #  01 ran in 67.8  seconds
Run #  02 ran in 96.19  seconds
````
InMemory query still took less time than buffer. Now run the same query with approx_for_percentile set in session level.
Run the same SQL query  sql-Dev tool "InMemory Query vs Buffer with sessions"

For the query string, enter the same query.
<copy>
select lo_custkey, median(lo_ordtotalprice) from lineorder
group by  lo_custkey ;
</copy>
And for the second parameter "alter_session_statement" enter the following.
<copy>
ALTER SYSTEM SET approx_for_count_distinct = TRUE;
ALTER SESSION SET approx_for_aggregation = TRUE;
ALTER SESSION SET APPROX_FOR_PERCENTILE=all;
</copy>
````
Plan hash value: 3675673598

-------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name      | Rows  | Bytes |TempSpc| Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |           | 80504 |   864K|       | 39801   (3)| 00:00:02 |
|   1 |  HASH GROUP BY APPROX       |           | 80504 |   864K|   459M| 39801   (3)| 00:00:02 |
|   2 |   TABLE ACCESS INMEMORY FULL| LINEORDER |    23M|   251M|       |  2045  (12)| 00:00:01 |
-------------------------------------------------------------------------------------------------

Run #  01 ran in 28.6  seconds
Run #  02 ran in 56.69  seconds
````
Observe that the plan has HASH GROUP BY APPROX instead of "SORT GROUP BY"
And the InMemory query tool about 28 seconds which is lesser than the previous InMemory runs.
This feature also speeds up In Buffer runs too. This feature is useful in BI repoers, when we need to build visualization, and summation reports where 95-99% accuracy of data does not alter the overall information convayed.   

## InMemory Materialized Views and Query Rewrite.

The best SQL plans are the once where we don't have to do any work to get the result. Oracle Materialized Views has been one of the solutions for BI Dashboards and Data warehouse reports for a while. Oracle Introduced automatic query rewrite that automatically use Materialized Views when available to speed the query.
As we observed InMemory is very efficient in Data filtering and Vector Join plans for speeding the joins. However, if these joins and aggradations are precomputed, it would help us avoid accessing the underlying table until the data is altered. Adding Materialized View Logs can help capture the delta and refresh faster. However, it could have a slight impact on CPU resources during DML operations.
With the support of these Materialized Views be loaded into InMemory pool, we can now access the pre computed joins and aggressions from InMemory.

Let us test this out.
For the same query we ran earlier, we can create a MV and see the performance difference. From sql*prompt or sql*developer worksheet, run the following creation script.
````
<copy>
DROP Materialized view MV_median ;
CREATE MATERIALIZED VIEW MV_median INMEMORY PRIORITY HIGH
BUILD IMMEDIATE
--REFRESH FORCE ON COMMIT
ENABLE QUERY REWRITE
AS
select lo_custkey, median(lo_ordtotalprice) from lineorder
group by  lo_custkey ;
</copy>
````

Run the sql report "In-Memory Query vs in Buffer Cache" and pass the below query.
````
<copy>
select lo_custkey, median(lo_ordtotalprice) from lineorder
group by  lo_custkey ;
</copy>
````
The output below.
````
query string is :
select lo_custkey, median(lo_ordtotalprice) from lineorder
group by  lo_custkey

IN MEMORY PLAN
Plan hash value: 835373166

---------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |           | 80000 |   859K|     3  (34)| 00:00:01 |
|   1 |  MAT_VIEW REWRITE ACCESS INMEMORY FULL| MV_MEDIAN | 80000 |   859K|     3  (34)| 00:00:01 |
---------------------------------------------------------------------------------------------------

BUFFER CACHE PLAN
Plan hash value: 835373166

------------------------------------------------------------------------------------------
| Id  | Operation                    | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |           | 80000 |   859K|    54   (2)| 00:00:01 |
|   1 |  MAT_VIEW REWRITE ACCESS FULL| MV_MEDIAN | 80000 |   859K|    54   (2)| 00:00:01 |
------------------------------------------------------------------------------------------

0 sec in memory, 0 sec in SGA ----
0 X times faster


Run #  01 ran in 0  seconds
Run #  02 ran in .02  seconds
````
The query that had taken 28 seconds earlier when doing direct path load query. InMemory takes about 20 millisecond to get data from Buffer compared to less than a millisecond for InMemory Materilized view.
MVs with query rewrite is powerful as a feature. But when these MVs are inmemory the performance is much more.

For more information about this, check out Oracle **[documentation.](https://docs.oracle.com/en/database/oracle/oracle-database/19/dwhsg/basic-query-rewrite-materialized-views.html#GUID-DB76286B-8557-446B-A6CC-BC987C378076)**

## Result Caching and InMemory

We saw that materialized views can rewrite any SQL queries that use the underlying join condition or aggregation. Result Caching is a layer above this. This can save the result at a SQL statement level and for each bind value.  This can run over MATERIALIZED views or just over Tables. While it is not part of InMemory workshop, it is none the less one of the features that can improve performance of BI reports and dashboards type of applications. For more information check out the **[documentation.](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/tuning-result-cache.html#GUID-FA30CC32-17AB-477A-9E4C-B47BFE0968A1)**.
