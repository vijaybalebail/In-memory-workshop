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

## Step 1: BULK LOADS and In-Memory tables.
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





## Step 3: In-Memory workload Query Performance.

  In the previous section, we disused how In-Memory transparently loads data into In-Memory pool. In-memory operations are mainly CPU and memory bond and have little Query Performance impact due to DML/ Bulk-Loads on the underlying table.
  To demonstrate this, Run the following script.
  The script will run 3 queries
  - Base RUN  : run queries without any load.
  - Batch RUN : run queries while, loading 10000 rows to Lineorder.
  - DML RUN   : run queries while inserting 500 rows to lineorder and commiting after each row.
  - ALL       : run queries while, batch and DML operating are happening.
````
<copy>
  DECLARE
  PROCEDURE wait_macro as
  cnt1 number:=1;
  BEGIN
    while cnt1 >= 1 loop
     dbms_lock.sleep (5);
     SELECT count(1) into cnt1 FROM dba_scheduler_running_jobs srj
     WHERE srj.job_name like 'QRUN%';
   end loop;
  end;
  PROCEDURE BACKGROUND_QUERY ( job_name in varchar2) AS
  BEGIN
  Begin
  dbms_scheduler.drop_job('JOB_NAME');
  EXCEPTION WHEN OTHERS THEN NULL;
  END ;
  dbms_scheduler.create_job (
    job_name => JOB_NAME,
    job_type => 'PLSQL_BLOCK',
    job_action => 'BEGIN QUERY_PERFORMANCE('''||job_name||''',400); END;',
    auto_drop =>  TRUE,
    enabled => true    
   );
  commit;
  END;
BEGIN
  execute immediate 'truncate table run_time ';

   BACKGROUND_QUERY('QRUN_BASE');
   WAIT_MACRO();

   BACKGROUND_QUERY('QRUN_BATCH');
   EXECUTE IMMEDIATE 'insert /*+ ssappend */ into lineorder select * from lineorder2 where rownum <=10000';
   commit;   
   WAIT_MACRO();

   BACKGROUND_QUERY('QRUN_DML');
   for c1 in ( select * from lineorder2 where rownum < 100 ) Loop
      insert into lineorder values (c1."LO_ORDERKEY",c1.LO_LINENUMBER,c1.LO_CUSTKEY,c1.LO_PARTKEY,c1.LO_SUPPKEY,
	   c1.LO_ORDERDATE,c1.LO_ORDERPRIORITY,c1.LO_SHIPPRIORITY,c1.LO_QUANTITY, c1.LO_EXTENDEDPRICE,c1.LO_ORDTOTALPRICE,
	   c1.LO_DISCOUNT, c1.LO_REVENUE,c1.LO_SUPPLYCOST,c1.LO_TAX,c1.LO_COMMITDATE,c1.LO_SHIPMODE ) ;
   end loop;
   commit;
   WAIT_MACRO();

   BACKGROUND_QUERY('QRUN_ALL');
   for c1 in ( select * from lineorder2 where rownum < 100 ) Loop
      insert into lineorder values (c1."LO_ORDERKEY",c1.LO_LINENUMBER,c1.LO_CUSTKEY,c1.LO_PARTKEY,c1.LO_SUPPKEY,
	   c1.LO_ORDERDATE,c1.LO_ORDERPRIORITY,c1.LO_SHIPPRIORITY,c1.LO_QUANTITY, c1.LO_EXTENDEDPRICE,c1.LO_ORDTOTALPRICE,
	   c1.LO_DISCOUNT, c1.LO_REVENUE,c1.LO_SUPPLYCOST,c1.LO_TAX,c1.LO_COMMITDATE,c1.LO_SHIPMODE ) ;
   end loop;
   commit;
   EXECUTE IMMEDIATE 'insert /*+ append * / into lineorder select * from lineorder2 where rownum <=10000';
   commit;
   WAIT_MACRO();
END;
/
</copy>
````


## Conclusion

In this lab you had an opportunity to try out Oracle’s In-Memory performance claims with queries that run against a table with over 23 million rows (i.e. LINEORDER), which resides in both the IM column store and the buffer cache. From a very simple aggregation, to more complex queries with multiple columns and filter predicates, the IM column store was able to out perform the buffer cache queries. Remember both sets of queries are executing completely within memory, so that’s quite an impressive improvement.

These significant performance improvements are possible because of Oracle’s unique in-memory columnar format that allows us to only scan the columns we need and to take full advantage of SIMD vector processiong. We also got a little help from our new in-memory storage indexes, which allow us to prune out unnecessary data. Remember that with the IM column store, every column has a storage index that is automatically maintained for you.

## Acknowledgements

- **Author** - Vijay Balebail , Andy Rivenes
- **Last Updated By/Date** - Oct 2020.

See an issue?  Please open up a request [here](https://github.com/oracle/learning-library/issues).
