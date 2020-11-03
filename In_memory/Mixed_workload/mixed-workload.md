# In-Memory Mixed workload

## Introduction

In a OLTP production environment, it is common to expect the datbase to run 1000s of transactions with very little toleration to poor throughput. In such environments, It is a common strategy to make a redundant copy of data in a data wherehouse system where analytic queries are run.

Also, if there are reports running of OLTp production database, we usually have several indexes on ensure that the reports run efficiently and not cause some unexpected heavy load on the OTLP system. More indexes we have in a OLTP system, the slower the DML operations happens due to the extra operations and logging of INSERT,DELETE operations on the underlying indexes.

 Now with in-memory , these queries can run without the need of additional reporting indexes. This helps in reducing the storage space and also improve performance of DML operations.

 ![](../images/lessIndexs.png)

Watch a preview video of querying the In-Memory Column Store

[](youtube:U9BmS53KuGs)

###  DML and In-Memory tables

It is clear by now that the IM column store can dramatically improve performance of all types of queries but very few database environments are read-only. For the IM column store to be truly effective in modern database environments it has to handle both bulk data loads AND online transaction processing (OLTP).
We will test how the Oracle Database In-Memory is the only in-memory column store that can handle both bulk data loads and online transaction processing today.

## Step 1: BLUK LOADS and In-Memory tables.
Bulk data loads occur most commonly in Data Warehouse environments and are typically conducted as a direct path load. A direct path load parses the input data, converts the data for each input field to its corresponding Oracle data type, and then builds a column array structure for the data. These column array structures are used to format Oracle data blocks and build index keys. The newly formatted database blocks are then written directly to the database, bypassing the standard SQL processing engine and the database buffer cache.
Once the load operation (direct path or non-direct path) has been committed, the IM column store is instantly aware it does not have all of the data populated for the object. The size of the missing data will be visible in the BYTES_NOT_POPULATED column of V$IM_SEGMENTS. If the object has a PRIORITY specified on it then the newly added data will be automatically populated into the IM column store. Otherwise the next time the object is queried, the background worker processes will be triggered to begin populating the missing data, assuming there is free space in the IM column store.


1.	Using SQL*Plus, connect as LABUSER.
   ````
   <copy> CONNECT  ssb/Ora_DB4U@localhost:1521/orclpdb </copy>
   ````

2.	Create a new SALES1 table as an empty copy of PART with INMEMORY .
````
<copy> CREATE TABLE PART1 INMEMORY AS SELECT * FROM  PART WHERE 1=2; </copy>
````

3.	Load data into PART1 via a bulk load operation (non-direct path). Do NOT COMMIT the transaction yet.
````
<copy> INSERT INTO PART1
SELECT * FROM PART WHERE ROWNUM <=10000;
 </copy>
````
4.	Check if the SALES1 table got populated into the IM column store by querying V$IM_SEGMENTS. What do you observe?
````
<copy> SELECT * FROM V$IM_SEGMENTS WHERE SEGMENT_NAME = 'PART1';
 </copy>
````
````
SQL> SELECT * FROM V$IM_SEGMENTS WHERE SEGMENT_NAME = 'PART1';

no rows selected
````

5.	As the data was not committed, the IM column store will not be able to see the changes and hence cannot be populated to the column store. COMMIT the changes so they are visible to the column store.

````
<copy> COMMIT;
 </copy>
````
````
SQL> COMMIT;

Commit complete.
````
6.	Recheck V$IM_SEGMENTS.
````
<copy> SELECT * FROM V$IM_SEGMENTS WHERE SEGMENT_NAME = 'PART1';
 </copy>
````
````
SQL> SELECT * FROM V$IM_SEGMENTS WHERE SEGMENT_NAME = 'PART1';

no rows selected
````

7.	Notice that SALES1 still did not get populated even though you had performed a COMMIT. Why?
    This is because the initial population is only triggered by querying the table via full table scan, using DBMS_INMEMORY.POPULATE or by specifying the PRIORITY clause, none of which was done in this case.

8.	Next, check the following session statistics:

-	IM populate segments requested. The number of times the session has requested for In-Memory population of a segment.
-	IM transactions. Transactions issued by the session to In-Memory tables (i.e. the number of times COMMITs have been issued).
-	IM transactions rows journaled. Count of rows logged in the Transaction Journal by the session. Transaction Journal is discussed in the DML section.
````
<copy> SELECT DISPLAY_NAME, VALUE  
FROM V$MYSTAT m, V$STATNAME n
WHERE m.STATISTIC# = n.STATISTIC#
AND n.DISPLAY_NAME IN (
'IM populate segments requested',
'IM transactions',
'IM transactions rows journaled');
'
 </copy>

DISPLAY_NAME					      VALUE
------------------------------------------   --------
IM transactions 					  	    1
IM transactions rows journaled		      10000
IM populate segments requested  			    0
````

9.	In the above output, observe the following:
-	IM populate segments requested. Segment population requested should be 0, as the column store population has not been triggered.
-	IM transactions. There was one COMMIT issued.
-	IM transactions rows journaled. The total number of rows loaded in the session so far is 10000.

10.	Let’s manually trigger a population using DBMS_INMEMORY.POPULATE procedure.
````
<copy>
EXEC DBMS_INMEMORY.POPULATE('SSB','PART1',NULL); </copy>

SQL> EXEC DBMS_INMEMORY.POPULATE('SSB','PART1',NULL);
PL/SQL procedure successfully completed.
````
11.	Check the population status now. You may need to run the below query a few times until you see an entry for PART1 with BYTES_NOT_POPULATED as 0.
````
<copy>
COL SEGMENT_NAME FORMAT A20
SELECT SEGMENT_NAME, POPULATE_STATUS, BYTES, BYTES_NOT_POPULATED
FROM V$IM_SEGMENTS WHERE SEGMENT_NAME = 'PART1'; </copy>

SEGMENT_NAME	   POPULATE_      BYTES BYTES_NOT_POPULATED
-------------------- --------- ---------- -------------------
PART1		   COMPLETED     458752		            0
````

12.	Using V$IM_SEGMENTS_DETAIL, check the number of extents and blocks for SALES1, both on disk and in-memory. Note that the number of blocks loaded in-memory (i.e. mapped to IMCUs) may be different from on-disk blocks as only the used blocks get populated into the column store.
````
<copy>
COL OBJECT_NAME FORMAT A20

SELECT a.OBJECT_NAME, b.MEMBYTES, b.EXTENTS, b.BLOCKS,
b.DATABLOCKS, b.BLOCKSINMEM, b.BYTES
FROM V$IM_SEGMENTS_DETAIL b, DBA_OBJECTS a
WHERE a.DATA_OBJECT_ID = b.DATAOBJ
AND a.OBJECT_NAME =  'PART1'; </copy>

SQL> COL OBJECT_NAME FORMAT A10
SQL> SELECT a.OBJECT_NAME, b.MEMBYTES, b.EXTENTS, b.BLOCKS, b.DATABLOCKS, b.BLOCKSINMEM, b.BYTES FROM V$IM_SEGMENTS_DETAIL b, DBA_OBJECTS a WHERE a.DATA_OBJECT_ID = b.DATAOBJ AND a.OBJECT_NAME =  'PART1';

OBJECT_NAM   MEMBYTES	 EXTENTS     BLOCKS DATABLOCKS BLOCKSINMEM BYTES
---------- ---------- ---------- ---------- ---------- ----------- ------
PART1	  1179648	       7	     56	    50	    50 458752
````

	From the above output, observe that PART1 so far has 7 extents, 56 allocated on-disk blocks (BLOCKS) and 50 used blocks (DATABLOCKS), out of which all 50 have been loaded to the IM column store (BLOCKSINMEM).
  Note: If the table does not have PRIORITY ENABLED and you execute another Direct Path Load, then those segments will be populated either during the next query or when you execute DBMS_INMEMORY.REPOPULATE.

## Step 2: DML and In-Memory tables

In this scenario, you will load a few additional rows into SALES1 and observe the column store population.

1.	Load a few rows into SALES1 via IAS (Insert as Select). Commit the changes so they become visible to the IM column store.
````
<copy>
INSERT INTO part1
SELECT *
FROM SALES
WHERE ROWNUM <=1000;
COMMIT; </copy>
````

2.	Check the session statistics once again and observe the change.
````
<copy>
SELECT DISPLAY_NAME, VALUE  FROM V$MYSTAT m, V$STATNAME n
WHERE m.STATISTIC# = n.STATISTIC#
AND n.DISPLAY_NAME IN (
'IM populate segments requested',
'IM transactions',
'IM transactions rows journaled'); </copy>

DISPLAY_NAME					      VALUE
------------------------------------------   --------
IM transactions 					  	    2
IM transactions rows journaled		      11000
IM populate segments requested  			    1
````
	From the above output, the IM transactions are now incremented to 2 (one additional due to the recent COMMIT) and the number of IM transactions rows journaled are now 11000 (reflecting the newly added 1000 rows on top of 10000).

3.	Check if the rows that just got inserted resulted in new extents being added to the on-disk table.
````
<copy>
COL OBJECT_NAME FORMAT A20
SELECT a.OBJECT_NAME, b.MEMBYTES, b.EXTENTS, b.BLOCKS,
b.DATABLOCKS, b.BLOCKSINMEM, b.BYTES
FROM V$IM_SEGMENTS_DETAIL b, DBA_OBJECTS a
WHERE a.DATA_OBJECT_ID = b.DATAOBJ
AND a.OBJECT_NAME =  'PART1';
</copy>

SQL> COL OBJECT_NAME FORMAT A10

SQL> SELECT a.OBJECT_NAME, b.MEMBYTES, b.EXTENTS, b.BLOCKS, b.DATABLOCKS, b.BLOCKSINMEM, b.BYTES FROM V$IM_SEGMENTS_DETAIL b, DBA_OBJECTS a WHERE a.DATA_OBJECT_ID = b.DATAOBJ AND a.OBJECT_NAME =  'PART1';

OBJECT_NAM   MEMBYTES	 EXTENTS     BLOCKS DATABLOCKS BLOCKSINMEM BYTES
---------- ---------- ---------- ---------- ---------- ----------- ------
PART1	  1179648	       7	     56	    50	    50 458752
````

As you observe from the above output, no new extents were added as only a few rows were loaded (1000 rows). The table hasn’t grown its on-disk footprint, and still has 7 extents, 50 DATABLOCKS and 50 BLOCKSINMEM.
If the table would have grown its on-disk footprint, the rows in the new extents will not be automatically populated into the column store, unless a non-default PRIOIRTY is specified or the table gets accessed, or the procedure DBMS_INMEMORY.PROPULATE is used.
If the new rows get added to existing extents/blocks, the new rows should be populated to the IM column store via the trickle repopulate process (discussed later), even when a default PRIORITY is specified on the table.

4.	Check the V$IM_SEGMENTS view and observe the value in BYTES_NOT_POPULATED column. Do you think a 0 in this column indicate that all the new rows have been added to the column store ?

````
<copy>
COL SEGMENT_NAME FORMAT A20

SELECT SEGMENT_NAME, POPULATE_STATUS, BYTES, BYTES_NOT_POPULATED
FROM V$IM_SEGMENTS WHERE SEGMENT_NAME = 'PART1';
</copy>

SEGMENT_NAME	   POPULATE_      BYTES BYTES_NOT_POPULATED
-------------------- --------- ---------- -------------------
PART1		   COMPLETED     458752		            0
````
BYTES_NOT_POPULATED only indicates the bytes in the new extents that got added to the segment and have not been populated to the column store. If the new rows get inserted into the existing extents, BYTES_NOT_POPULATED will still be 0 and new rows may still be not populated. This how the V$IM_SEGMENTS view behaves currently.
Next, check if the newly added rows got populated to the IM column store using V$IM_HEADER view. This view contains details about all IMCUs (incl. split IMCUs) for all segments that are loaded into the column store.

5. You may have to run the below query a few times in order to for TRICKLE_REPOPULATE to be set to 1.

````
<copy>

COL OBJECT_NAME FORMAT A10

SELECT b.OBJECT_NAME, a.PREPOPULATED, a.REPOPULATED, a.TRICKLE_REPOPULATED, a.NUM_DISK_EXTENTS, a.NUM_BLOCKS, a.NUM_ROWS, a.NUM_COLS
FROM V$IM_HEADER a, DBA_OBJECTS b
WHERE a.OBJD = b.DATA_OBJECT_ID
AND b.OBJECT_NAME =  'SALES1';
</copy>
SQL> COL OBJECT_NAME FORMAT A10  SELECT b.OBJECT_NAME, a.PREPOPULATED, a.REPOPULATED, a.TRICKLE_REPOPULATED, a.NUM_DISK_EXTENTS, a.NUM_BLOCKS, a.NUM_ROWS, a.NUM_COLS FROM V$IM_HEADER a, DBA_OBJECTS b WHERE a.OBJD = b.DATA_OBJECT_ID AND b.OBJECT_NAME =  'SALES1';

OBJECT_NAM PREPOPULATED REPOPULATED TRICKLE_REPOPULATED NUM_DISK_EXTENTS  NUM_BLOCKS   NUM_ROWS	NUM_COLS

---------- ------------ ----------- ------------------- -------------------------- ---------- ----------

PART1		      0 	  1		      1 	       7 	50	11000	       7
````

Below are the observations from the above output:
-	There is only one IMCU (evident from the presence of only a single row in the output).
- 	The number of on disk rows mapped to this IMCU is 11000, which means that the newly added 1000 rows were also recorded in the journal.
- A total of 50 disk extents have been mapped to this IMCU.
- The REPOPULATED flag and the TRICKLE_REPOPULATED flag for this IMCU has been set to 1 (indicating that the trickle repopulate process has synchronized the changes).
- The trickle repopulate process must have populated the last 1000 rows.





## Step 2: In-Memory Indexing Storage Indexing
  In the *Introduction and Overview*, we saw how min-max and dictionary based pruning could work as Index. We will now query the table and filter based on a where condition.

5.  Let's look for a specific order in the LINEORDER table based on the order key.  Typically, a full table scan is not an efficient execution plan when looking for a specific entry in a table.  

    ````
    <copy>
    set timing on

    select  lo_orderkey, lo_custkey, lo_revenue
    from    LINEORDER
    where   lo_orderkey = 5000000;

    set timing off

    select * from table(dbms_xplan.display_cursor());

    @../imstats.sql
    </copy>
    ````



6.  Think indexing lo_orderkey would provide the same performance as the IM column store? There is an invisible index already created on the lo_orderkey column of the LINEORDER table. By using the parameter OPTIMIZER_USE_INVISIBLE_INDEXES we can compare the performance of the IM column store and the index. Let's see how well the index performs.  

    ````
    <copy>
    alter session set optimizer_use_invisible_indexes=true;

    set timing on

    Select  /* With index */ lo_orderkey, lo_custkey, lo_revenue
    From    LINEORDER
    Where   lo_orderkey = 5000000;

    set timing off

    select * from table(dbms_xplan.display_cursor());

    @../imstats.sql
    </copy>
    ````
    We observe that , when index is available on the filter column of the query, the optimizer chose INDEX SCAN over INMEMORY FULL TABLE SCAN.


7.  Analytical queries have more than one equality WHERE clause predicate. What happens when there are multiple single column predicates on a table? Traditionally you would create a multi-column index. Can storage indexes compete with that?  

    Let’s change our query to look for a specific line item in an order and monitor the session statistics:

    To execute the query against the IM column store type.  

    ````
    <copy>
    set timing on

    select
    lo_orderkey,
    lo_custkey,
    lo_revenue
    from
    LINEORDER
    where
    lo_custkey = 5641
    and lo_shipmode = 'XXX AIR'
    and lo_orderpriority = '5-LOW';

    set timing off

    select * from table(dbms_xplan.display_cursor());

    @../imstats.sql

    exit
    </copy>
    ````
    You can see that the In-Memory SCAN  is used even when there is a INDEX on lo_orderkey. In fact, INMEMORY replaces multiple indexes on the Database.
    This not only speeds up Query with fewer indexes, but also improve DML and load performance due to fewer indexes.

    ![](images/num6.png)   


## Step 3: In-Memory Joins and In-Memory Aggregation

Up until now we have been focused on queries that scan only one table, the LINEORDER table. Let’s broaden the scope of our investigation to include joins and parallel execution. This section executes a series of queries that begin with a single join between the  fact table, LINEORDER, and a dimension table and works up to a 5 table join. The queries will be executed in both the buffer cache and the column store, to demonstrate the different ways the column store can improve query performance above and beyond the basic performance benefits of scanning data in a columnar format.

1. Let's switch to the Part2 folder and log back in to the PDB.

   ````
   <copy>

   sqlplus ssb/Ora_DB4U@localhost:1521/orclpdb
   <copy>    
   ````

   ![](images/num1.png)

2. Join the LINEORDER and DATE_DIM tables in a "What If" style query that calculates the amount of revenue increase that would have resulted from eliminating certain company-wide discounts in a given percentage range for products shipped on a given day (Christmas eve 1996).  In the first one, execute it against the IM column store.  

   ````
   <copy>
   set timing on
   set pages 9999
   set lines 100

   SELECT SUM(lo_extendedprice * lo_discount) revenue
   FROM   lineorder l,
   date_dim d
   WHERE  l.lo_orderdate = d.d_datekey
   AND    l.lo_discount BETWEEN 2 AND 3
   AND    l.lo_quantity < 24
   AND    d.d_date='December 24, 1996';

   set timing off

   select * from table(dbms_xplan.display_cursor());

   @../imstats.sql
   </copy>
   ````

   ![](images/num2.png)

   The IM column store has no problem executing a query with a join because it is able to take advantage of Bloom Filters.  It’s easy to identify Bloom filters in the execution plan. They will appear in two places, at creation time and again when it is applied. Look at the highlighted areas in the plan above. You can also see what join condition was used to build the Bloom filter by looking at the predicate information under the plan.

3. Let's run against the buffer cache now.  

   ````
   <copy>
   set timing on

   select /*+ NO_INMEMORY */
   sum(lo_extendedprice * lo_discount) revenue
   from
   LINEORDER l,
   DATE_DIM d
   where
   l.lo_orderdate = d.d_datekey
   and l.lo_discount between 2 and 3
   and l.lo_quantity < 24
   and d.d_date='December 24, 1996';

   set timing off

   select * from table(dbms_xplan.display_cursor());

   @../imstats.sql
   </copy>
   ````

   ![](images/num3.png)

4. Let’s try a more complex query that encompasses three joins and an aggregation to our query. This time our query will compare the revenue for different product classes, from suppliers in a certain region for the year 1997. This query returns more data than the others we have looked at so far so we will use parallel execution to speed up the elapsed times so we don’t need to wait too long for the results.  

   ````
   <copy>
   set timing on

   SELECT d.d_year, p.p_brand1,SUM(lo_revenue) rev
   FROM   lineorder l,
       date_dim d,
       part p,
       supplier s
   WHERE  l.lo_orderdate = d.d_datekey
   AND    l.lo_partkey = p.p_partkey
   AND    l.lo_suppkey = s.s_suppkey
   AND    p.p_category = 'MFGR#12'
   AND    s.s_region   = 'AMERICA'
   AND    d.d_year     = 1997
   GROUP  BY d.d_year,p.p_brand1;

   set timing off

   select * from table(dbms_xplan.display_cursor());

   @../imstats.sql
   </copy>
   ````

   ![](images/num4a.png)

   ![](images/num4b.png)

   The IM column store continues to out-perform the buffer cache query but what is more interesting is the execution plan for this query:

   In this case, we noted above that three join filters have been created and applied to the scan of the LINEORDER table, one for the join to DATE_DIM table, one for the join to the PART table, and one for the join to the SUPPLIER table. How is Oracle able to apply three join filters when the join order would imply that the LINEORDER is accessed before the SUPPLER table?

   This is where Oracle’s 30 plus years of database innovation kick in. By embedding the column store into Oracle Database we can take advantage of all of the optimizations that have been added to the database. In this case, the Optimizer has switched from its typically left deep tree to create a right deep tree using an optimization called ‘swap_join_inputs’. *Your instructor can explain ‘swap_join_inputs’ in more depth should you wish to know more. What this means for the IM column store is that we are able to generate multiple Bloom filters before we scan the necessary columns for the fact table, meaning we are able to benefit by eliminating rows during the scan rather than waiting for the join to do it.*

## Step 4: In-Memory Join Group

   A new In-Memory feature called Join Groups was introduced with the Database In-Memory Option in Oracle Database 12.2.  Join Groups can be created to significantly speed up hash join performance in an In-Memory execution plan.  Creating a Join Group involves identifying up-front the set of join key columns (across any number of tables) likely to be joined with by subsequent queries.  

    For the above example, we can create a In-Memory Join Group on the join column l.lo_orderdate = d.d_datekey.
    If you observe a HASH JOIN plan for inmemory, there are 2 operations. One is *JOIN FILTER CREATE* for  Bloom filter. The other is *JOIN FILTER USE* to consume it. Pre-creating the *Join Group* increases the performance of queries by using the prebuilt Join groups.
   ````
     CREATE INMEMORY JOIN GROUP  JoinGroup (lineorder(lo_orderdate),date_dim (d_datekey)) ;
   ````

## Step 5: In-Memory Expressions
   In-Memory Expressions (IM expressions) provide the ability to materialize simple deterministic expressions and store them in the In-Memory column store (IM column store) so that they only have to be calculated once, not each time they are accessed. They are also treated like any other column in the IM column store so the database can scan and filter those columns and take advantage of all Database In-Memory query optimizations like SIMD vector processing and IM storage indexes.

   There are actually two types of IM expressions, a user-defined In-Memory virtual column (IM virtual column) that meets the requirements of an IM expression, and automatically detected IM expressions which are stored as a hidden virtual column when captured.

   ### 1. User-defined virtual column.

   The automatically detected IM expressions are captured in the new Expression Statistics Store (ESS). IM expressions are fully documented in the In-Memory Guide.


 ````
 <copy>
 set timing on
 set autotrace traceonly
 SELECT lo_shipmode, SUM(lo_ordtotalprice),
 SUM(lo_ordtotalprice - (lo_ordtotalprice*(lo_discount/100)) + lo_tax) discount_price
 FROM LINEORDER
 GROUP BY lo_shipmode
 ORDER BY lo_shipmode;
 set timing off

 select * from table(dbms_xplan.display_cursor());

 @../imstats.sql
 </copy>
 ````
 And the session statistics does not have any IM Scan EU ... stats.

 Notice the following expression in the query:

````
(lo_ordtotalprice - (lo_ordtotalprice*(lo_discount/100)) + lo_tax)

````

This expression is simply an arithmetic expression to find the total price charged with discount and tax included, and is deterministic. We will create a virtual column and re-populate the LINEORDER table to see what difference it makes.

 To make sure that we populate virtual columns in the IM column store we need to ensure that the initialization parameter INMEMORY_VIRTUAL_COLUMNS is set to ENABLE or MANUAL. The default is MANUAL which means that you must explicitly set the virtual columns as INMEMORY enabled. Set it to enable at table level.
````
 SQL> <copy> show parameter INMEMORY_VIRTUAL_COLUMNS
       alter system set inmemory_virtual_columns=enable;
       show parameter INMEMORY_VIRTUAL_COLUMNS </copy>

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
inmemory_virtual_columns             string      MANUAL

SQL> alter system set inmemory_virtual_columns=enable;
System altered.
````
Next, we will add a virtual column to the LINEORDER table for the expression we identified above and re-populate the table:

````
<copy>
alter table lineorder no inmemory;
alter table lineorder add v1 as (lo_ordtotalprice - (lo_ordtotalprice*(lo_discount/100)) + lo_tax)
alter table lineorder inmemory ;
select /*+ noparallel */ count(*) from lineorder ;
<\copy>
````

Now let's re-run our query and see if there is any difference:
````
<copy>
set timing on
set autotrace traceonly
SELECT lo_shipmode, SUM(lo_ordtotalprice),
SUM(lo_ordtotalprice - (lo_ordtotalprice*(lo_discount/100)) + lo_tax) discount_price
FROM LINEORDER
GROUP BY lo_shipmode
ORDER BY lo_shipmode;
set timing off

select * from table(dbms_xplan.display_cursor());

@../imstats.sql
</copy>
````
Notice the statistics that start with "IM scan EU ...". IM expressions are stored in the IM column store in In-Memory Expression Units (IMEUs) rather than IMCUs, and the statistics for accessing IM expressions all show "EU" for "Expression Unit".

This simple example shows that even relatively simple expressions can be computationally expensive, and the more frequently they are executed the more savings in CPU resources IM expressions will provide.

 ### 2. Automatic IM expressions.
  Optimizer tracks expressions and stores them in a repository called the Expression Statistics Store (ESS). Along with the expression and other information, the number of times the expression was evaluated and the cost of evaluating the expression are tracked. This is exposed in the ALL|DBA|USER_EXPRESSION_STATISTICS view and Database In-Memory uses this information to determine the 20 most frequently accessed expressions in either the past 24 hours (i.e. LATEST snapshot) or since database creation (i.e. CUMULATIVE snapshot). The following is an example for the LINEORDER table in my test system for the LATEST snapshot.

  In order to populate automatically detected IM expressions the INMEMORY_EXPRESSIONS_USAGE parameter must be set to either ENABLE (the default) or DYNAMIC_ONLY.
  ````
  <copy> show parameter INMEMORY_EXPRESSIONS_USAGE </copy>

  NAME                                 TYPE        VALUE
  ------------------------------------ ----------- ------------------------------
  inmemory_expressions_usage           string      ENABLE
  ````
 #### Let us restore the lineorder table and drop the virtual column.
 ````
 <copy>
 alter table lineorder no inmemory;
 ALTER TABLE lineorder DROP COLUMN v1;
 alter table lineorder inmemory ;
 select /*+ noparallel */ count(*) from lineorder ; </copy
 ````

Since 18c, there are 2 ways to capture the expressions automatically. They are either during a window or current.
For the window function, we use dbms_inmemory_admin.ime_open_capture_window() to start.
dbms_inmemory_admin.ime_close_capture_window() to stop capturing the inmemory expression and run
dbms_inmemory_admin.ime_capture_expressions('WINDOW') to automatically generate automatic In-Memory expressions.
The other alternative is to simple run dbms_inmemory_admin.ime_capture_expressions('CURRENT') which will capture the expressions from the shared pool.

  ````
  <copy>
  set timing on
  set autotrace traceonly
  SELECT lo_shipmode, SUM(lo_ordtotalprice),
  SUM(lo_ordtotalprice - (lo_ordtotalprice*(lo_discount/100)) + lo_tax) discount_price
  FROM LINEORDER
  GROUP BY lo_shipmode
  ORDER BY lo_shipmode;
  set timing off

  dbms_inmemory_admin.ime_capture_expressions('CURRENT');

  select * from table(dbms_xplan.display_cursor());

  @../imstats.sql
  </copy>
  ````

Now check if the expression is captured.
````
  select
````

Now rerun the query and verify you see "IM scan EU ..." instead on only "IM scan CU ..." statistics.

## Step 6: In-Memory Optimized Arithmetic

Using the native binary representation of numbers rather than the full precision Number format means that certain aggregation and arithmetic operations are tens of times faster than in previous version of In-Memory as we are
able to take advantage of the SIMD Vector processing units on CPUs

![](images\IMOptimizedArithmetic.png)

In-Memory Optimized Arithmetic are available in 18c and encodes the NUMBER data type as a fixed-width native
integer scaled by a common exponent. This enables faster calculations using SIMD hardware. The Oracle Database
NUMBER data type has high fidelity and precision. However, NUMBER can incur a significant performance overhead
for queries because arithmetic operations cannot be performed natively in hardware. The In-Memory optimized
number format enables native calculations in hardware for segments compressed with the QUERY LOW compression
option.
Not all row sources in the query processing engine have support for the In-Memory optimized number format so the
IM column store stores both the traditional Oracle Database NUMBER data type and the In-Memory optimized number
type. This dual storage increases the space overhead, sometimes up to 15%.
In-Memory Optimized Arithmetic is controlled by the initialization parameter INMEMORY_OPTIMIZED_ARITHMETIC.
The parameter values are DISABLE (the default) or ENABLE. When set to ENABLE, all NUMBER columns for tables that
use FOR QUERY LOW compression are encoded with the In-Memory optimized format when populated (in addition to
12 | ORACLE DATABASE IN-MEMORY WITH ORACLE DATABASE 19C
the traditional Oracle Database NUMBER data type). Switching from ENABLE to DISABLE does not immediately drop
the optimized number encoding for existing IMCUs. Instead, that happens when the IM column store repopulates
affected IMCUs.

#### Set
````
SQL> <copy> show parameter INMEMORY_OPTIMIZED_ARITHMETIC
      alter system set INMEMORY_OPTIMIZED_ARITHMETIC=enable;
      show parameter INMEMORY_OPTIMIZED_ARITHMETIC </copy>
      SQL> show parameter INMEMORY_OPTIMIZED_ARITHMETIC

      NAME                                 TYPE        VALUE
      ------------------------------------ ----------- ------------------------------
      inmemory_optimized_arithmetic        string      ENABLE

      SQL> alter system set INMEMORY_OPTIMIZED_ARITHMETIC=enable;
      System altered.
 ````
Now reload lineorder
````
<copy>
alter table lineorder no inmemory;
alter table lineorder inmemory ;
select /*+ noparallel */ count(*) from lineorder ; </copy
````


## Conclusion

In this lab you had an opportunity to try out Oracle’s In-Memory performance claims with queries that run against a table with over 23 million rows (i.e. LINEORDER), which resides in both the IM column store and the buffer cache. From a very simple aggregation, to more complex queries with multiple columns and filter predicates, the IM column store was able to out perform the buffer cache queries. Remember both sets of queries are executing completely within memory, so that’s quite an impressive improvement.

These significant performance improvements are possible because of Oracle’s unique in-memory columnar format that allows us to only scan the columns we need and to take full advantage of SIMD vector processiong. We also got a little help from our new in-memory storage indexes, which allow us to prune out unnecessary data. Remember that with the IM column store, every column has a storage index that is automatically maintained for you.

## Acknowledgements

- **Author** - Vijay Balebail , Andy Rivenes
- **Last Updated By/Date** - Oct 2020.

See an issue?  Please open up a request [here](https://github.com/oracle/learning-library/issues).
