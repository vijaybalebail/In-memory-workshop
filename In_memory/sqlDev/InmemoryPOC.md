# In-Memory Tool

## Introduction

Oracle In-Memory is relatively new. There are many expert and novice DBAs alike who would like to test their application SQL statements In-Memory and in buffer and understand the inner workings.

If we are doing a benchmark of Database with In-Memory enabled and disabled, it is not quiet obvious how Individual performance gains have contributed to the overall performance, In this Lab, we will look at a tool to run individual queries and see how it behaves with In-Memory and without it.

The tool has been developed using Sql\*Developer "User defined reports". The scripts can be plugged-in to any Sql\*Developer and the user can quickly connect to the schema and run the monitoring and diagnostic reports.  

The major capability of the tool is

1)  to run any give sql statement with in-memory ENABLED and  DISABLED and display the 2 different sql plans and stats. All the statistics that changed during the execution of the sql statement are displayed side-by-side. The unique features is that it can keep the same sql_id for the sql statment as the tool will not alter the statement in any way. (like adding a HINT or count in front of the sql).  Also, it only fetches the first 10 rows and stops. This will ensure you are only counting the DB time and not network time to move large amount of data.

2) It also has  provision to add multiple session level information to run statements like "alter session set nls\_date\_format ...;alter session....; " This will help if the sql queries expect date to be of a certain format so that you need not add to_date syntax. Also, you can add other tunable parameters like parallel degrees,APPROX\_FOR\_AGGREGATION,etc.

3) Show barchart of execution times when run in Buffer and In-Memory.

4) Show barchart of space it would occupy in buffer compared to In-Memory.

5) Show Barchart of different memory pools in SGA and PieChart of memory used by In-Memory tables.

## Install

The SQL In-Memory POC tool is loaded as a User defined report in Sql\*Developer. You can download Sql\*Developer in the test environment.

1. Download the XML file from [your Browser](https://raw.githubusercontent.com/vijaybalebail/In-memory-workshop/master/In_memory/sqlDev/InMemoryPOC.xml). Then right click and choose "save as"  to save the XML file.



![](images/saveAs.png)



2. Open Sql*Developer Click on Views--> reports
![](images/viewReports.png)

3. In the reports window, Right click and select "Open Report". Open the xml file saved in step 1.
![](images/openReport.png)

That is it. You have installed the In-Memory POC tool. Next, we we need to connect to the test environment.

4. Ensure have a connection in Sql\*Developer as the user which is running the query.
   To add a New Connection, you can click on the "+" icon on the upper left corner under connections window frame.
    ![](images/newConnection.png)

    You can add the connection information to connect. In our case, we will run Queries as SSB user for PDB orclpdb.
    ![](images/SSBconnection.png)
    "Test" your connection and Click Save to save the connection alias.

5. Next. We need to install some DB objects for the tool to work, we need to install the following objects.
	*   global temporary table called run_stats
	*   view called stats
	*   Plsql package called runstats_pkg

 Expand the "User Defined report" --> "InMemory POC tool" --> "Load Runstats".
 When you run in for the first time, you will get the following message displayed.

 ````
  As a sys user please grant access to following tables and run again
  grant select on SYS.V_$STATNAME to SSB;
  grant select on SYS.V_$MYSTAT to SSB;
  grant select on SYS.V_$TIMER to SSB;
  grant select on SYS.V_$LATCH to SSB;
  grant select on SYS.V_$SESSTAT to SSB;
  Table Created
  ORA-01031: insufficient privileges
  package recompiled in user SSB
  ORA-24344: success with compilation error
 ````

To add privileges, connect to sql\*plus session as sys user of ORCLPDB and grant the privileges.
````
  <copy>
  connect / as sysdba
  alter session set container=ORCLPDB;
  grant select on SYS.V_$STATNAME to SSB;
  grant select on SYS.V_$MYSTAT to SSB;
  grant select on SYS.V_$TIMER to SSB;
  grant select on SYS.V_$LATCH to SSB;
  grant select on SYS.V_$SESSTAT to SSB;
  </copy>
````
After granting the privileges, re-run the "Load Runstats"
After about 10 seconds, you should get the following display.

````
  temp table run_stats is already created
  view recreated
  package recompiled in user SSB
  Package body recompiled in user SSB
  You are ready to run the report
````
At this point, you have successfully imported the user defined report and installed the package.

## Verify the objects are Loaded.
Click on the report "in-memory-objects"
This displays the tables that have the property INMEMORY setup and loaded into In-Memory pool. In our test ensure you see LINEORDER,PART,CUSTOMER,DATE_DIM,SUPPLIER. If there are not show here, you can run the report "Load all inmemory tables".

![](images/objectSizeBar.png)

The chart shows the space it used in In-Memory compared to space occupied in buffer. Notice, that space in In-Memory is less than in buffer. Further, due to various compression levels, you could tune the compression level to fill the pool more economically.

## Run SQL Query
Now that the tool is installed, Click on "In-Memory Query vs  Buffer Cache".
In the popup , enter the following query to see the performance difference, SQL Plan and stats.

````
<copy>
select
lo_orderkey,
lo_custkey,
lo_revenue
from
LINEORDER
where
lo_custkey = 5641
and lo_shipmode = 'XXX AIR'
and lo_orderpriority = '5-LOW'; </copy>
````
![](images/enterQuery.png)
 Click "Apply"

 We will see the tool display the following .
 ````
     query string is :
    select
    lo_orderkey,
    lo_custkey,
    lo_revenue
    from
    LINEORDER
    where
    lo_custkey = 5641
    and lo_shipmode = 'XXX AIR'
    and lo_orderpriority = '5-LOW'

    IN MEMORY PLAN
    Plan hash value: 4017770458

    ----------------------------------------------------------------------------------------
    | Id  | Operation                  | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
    ----------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT           |           |     1 |    44 |  9459   (2)| 00:00:01 |
    |*  1 |  TABLE ACCESS INMEMORY FULL| LINEORDER |     1 |    44 |  9459   (2)| 00:00:01 |
    ----------------------------------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------

       1 - inmemory("LO_CUSTKEY"=5641 AND "LO_SHIPMODE"='XXX AIR' AND
                  "LO_ORDERPRIORITY"='5-LOW')
           filter("LO_CUSTKEY"=5641 AND "LO_SHIPMODE"='XXX AIR' AND
                  "LO_ORDERPRIORITY"='5-LOW')

    BUFFER CACHE PLAN
    Plan hash value: 4017770458

    -------------------------------------------------------------------------------
    | Id  | Operation         | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
    -------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT  |           |     1 |    44 | 91044   (1)| 00:00:04 |
    |*  1 |  TABLE ACCESS FULL| LINEORDER |     1 |    44 | 91044   (1)| 00:00:04 |
    -------------------------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------

       1 - filter("LO_CUSTKEY"=5641 AND "LO_SHIPMODE"='XXX AIR' AND
                  "LO_ORDERPRIORITY"='5-LOW')

    .09 sec in memory, 1.02 sec in SGA ----
    11 X times faster


    Run #  01 ran in .09  seconds
    Run #  02 ran in 1.05  seconds
    ###########################################################################
     Statistics                               | inmemory       | buffer        
    ###########################################################################
    CPU used by this session................ |              9 |            106
    HSC Heap Segment Block Changes.......... |              0 |              2
    IM SubCU-MM CUs Examined................ |              6 |              0
    IM SubCU-MM CUs Selected................ |              3 |              0
    IM SubCU-MM SubCUs Eliminated........... |          2,648 |              0
    IM SubCU-MM SubCUs in Selected CUs...... |          3,108 |              0
    IM scan CUs columns accessed............ |              8 |              0
    IM scan CUs columns theoretical max..... |            748 |              0
    IM scan CUs current..................... |             44 |              0
    IM scan CUs invalid or missing revert to |             16 |              0
    IM scan CUs memcompress for query low... |             44 |              0
    IM scan CUs no cleanout................. |             44 |              0
    IM scan CUs pcode pred evaled........... |             10 |              0
    IM scan CUs pcode selective done........ |              4 |              0
    IM scan CUs predicates applied.......... |             90 |              0
    IM scan CUs predicates optimized........ |            126 |              0
    IM scan CUs predicates received......... |            132 |              0
    IM scan CUs pruned...................... |             42 |              0
    IM scan CUs readlist creation accumulate |              9 |              0
    IM scan CUs readlist creation number.... |             44 |              0
    IM scan CUs split pieces................ |             65 |              0
    IM scan bytes in-memory................. |  1,386,345,809 |              0
    IM scan bytes uncompressed.............. |  2,134,966,644 |              0
    IM scan delta - only base scan.......... |             44 |              0
    IM scan rows............................ |     22,417,090 |              0
    IM scan rows optimized.................. |     21,365,119 |              0
    IM scan rows projected.................. |              1 |              0
    IM scan rows valid...................... |      1,051,971 |              0
    IM scan segments disk................... |              0 |              1
    IM scan segments minmax eligible........ |             44 |              0
    IM simd compare calls................... |             18 |              0
    IM simd compare selective calls......... |              1 |              0
    IM simd decode symbol calls............. |              3 |              0
    IM simd decode unpack calls............. |              3 |              0
    IM simd decode unpack selective calls... |              3 |              0
    buffer is not pinned count.............. |              2 |             19
    calls to get snapshot scn: kcmgss....... |              2 |             17
    calls to kcmgcs......................... |              2 |             30
    consistent changes...................... |              0 |              2
    consistent gets......................... |         25,154 |        333,545
    consistent gets examination............. |              0 |              1
    consistent gets examination (fastpath).. |              0 |              1
    consistent gets from cache.............. |         25,154 |        333,545
    consistent gets pin..................... |         25,154 |        333,544
    consistent gets pin (fastpath).......... |         25,154 |        333,544
    db block changes........................ |              0 |              4
    db block gets........................... |              0 |              2
    db block gets from cache................ |              0 |              2
    db block gets from cache (fastpath)..... |              0 |              2
    enqueue releases........................ |              1 |              6
    enqueue requests........................ |              1 |              6
    execute count........................... |              2 |             17
    index fetch by key...................... |              0 |              1
    lob writes.............................. |              0 |              1
    lob writes unaligned.................... |              0 |              1
    logical read bytes from cache........... |    206,061,568 |  2,732,417,024
    no work - consistent read gets.......... |         25,152 |        333,514
    non-idle wait count..................... |              3 |              2
    opened cursors cumulative............... |              2 |             17
    parse count (hard)...................... |              0 |              1
    parse count (total)..................... |              1 |             14
    parse time cpu.......................... |              0 |              1
    parse time elapsed...................... |              0 |              1
    recursive calls......................... |              9 |            123
    recursive cpu usage..................... |             10 |            105
    redo entries............................ |              0 |              2
    redo size............................... |              0 |            340
    session cursor cache hits............... |              2 |             13
    session logical reads - IM.............. |        311,540 |              0
    session pga memory...................... |        262,144 |     -1,769,472
    session uga memory...................... |            -40 |             40
    sorts (memory).......................... |              1 |              2
    table fetch by rowid.................... |              0 |              1
    table scan blocks gotten................ |         25,152 |        333,513
    table scan disk IMC fallback............ |      1,809,733 |              0
    table scan disk non-IMC rows gotten..... |              0 |     23,996,826
    table scan rows gotten.................. |      2,861,704 |     23,996,976
    table scan rs2.......................... |              1 |              0
    table scans (IM)........................ |              1 |              0
    table scans (short tables).............. |              1 |              3
    undo change vector size................. |              0 |            136
    workarea executions - optimal........... |              3 |              4
    workarea memory allocated............... |             11 |            -13
    ###########################################################################
 ````
 From the report generated, observe the SQL PLAN operation "*TABLE ACCESS INMEMORY FULL*" for INMEMORY plan and "*TABLE ACCESS FULL*" For table scan operation.

 Next, You can look at the relevant DB stats to see the optimizations.

 Note that the first time you run of the query, it might appear slower as the runstat package may not be in the shared pool.
Rerun the report again and compare.

Some sql queries will need to set session parameters like Parallel Degree, Hint and invisible indexes, optimizer tuning parameters and NLS date formats. We already tested one query by enabling invisible index in our test. We can now

## Top InmemorySQL report

![](images/topSQLwithInMemory.png)

SELECT /*+ 11VECTOR_TRANSFORM */ d.d_year,  p.p_brand1,SUM(lo_revenue) rev
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
