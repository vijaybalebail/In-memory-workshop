# Workshop Introduction and Overview

## Introduction to Oracle In-Memory

Oracle Database In-Memory adds in-memory functionality to Oracle Database for transparently accelerating analytic queries by orders of magnitude, enabling real-time business decisions. Using Database In-Memory, businesses can instantaneously run analytics and reports that previously took hours or days. Businesses benefit from better decisions made in real-time, resulting in lower costs, improved productivity, and increased competitiveness.

 Oracle Database In-Memory accelerates both Data Warehouses and mixed workload OLTP databases and is easily deployed under any existing application that is compatible with Oracle Database. No application changes are required. Database In-Memory uses Oracle’s mature scale-up, scale-out, and storage-tiering technologies to cost effectively run any size workload. Oracle’s industry leading availability and security features all work transparently with Oracle Database In-Memory, making it the most robust offering on the market.



## **Dual-Format Database**



**Row Format vs. Column Format** 

  ![ ](https://github.com/vijaybalebail/learning-library/raw/master/data-management-library/database/in-memory/intro/images/DBIM.png)



Oracle Database has traditionally stored data in a row format. In a row format database, each new transaction or record stored in the database is represented as a new row in a table. That row is made up of multiple columns, with each column representing a different attribute about that record. A row format is ideal for online transaction systems, as it allows quick access to all of the columns in a record since all of the data for a given record are kept together inmemory and on-storage. 

A column format database stores each of the attributes about a transaction or record in a separate column structure. A column format is ideal for analytics, as it allows for faster data retrieval when only a few columns are selected but the query accesses a large portion of the data set. 

But what happens when a DML operation (insert, update or delete) occurs on each format? A row format is incredibly efficient for processing DML as it manipulates an entire record in one operation (i.e. insert a row, update a row or delete a row). A column format is not as efficient at processing DML, to insert or delete a single record in a column format all the columnar structures in the table must be changed. That could require one or more I/O operations per column. Database systems that support only one format suffer the tradeoff of either sub-optimal OLTP or sub-optimal analytics performance. 

Oracle Database In-Memory (Database In-Memory) provides the best of both worlds by allowing data to be simultaneously populated in both an in-memory row format (the buffer cache) and a new in-memory columnar format (in Memory pool )  a dual-format architecture. 



## **Database In-Memory and Performance**



There are four basic architectural elements of the column store that enable orders of magnitude faster analytic query processing:

1. *Compressed columnar storage*: Storing data contiguously in compressed column units allows an analytic query to scan only data within the required columns, instead of having to skip past unneeded data in other columns as would be needed for a row major format. Columnar storage therefore allows a query to perform highly efficient sequential memory references while compression allows the query to optimize its use of the available system (processor to memory) bandwidth.
2. Vector processing. Modern CPUs feature highly parallel instructions known as SIMD or vector instructions(e.g. Intel AVX). These instructions can process multiple values in one instruction –for instance, they allow multiple values to be compared with a given value (e.g. find saleswith State = “California”) in one instruction. Vector processing of compressed columnar data further multiplies the scan speed obtained via columnar storage,resulting in scan speeds exceeding tens of billions of rows per second, per CPU core.



1. *In-Memory Storage Indexes*: The IM column store for a given table is divided into units known as In-Memory Compression Units(IMCUs) that typically represent a large number of rows (typically several hundred thousand). Each IMCU automatically records the min and max values for the data within each column in the IMCU, as well as other summary information regarding the data. This metadata serves as an In-Memory Storage Index: For instance, it allows an entire IMCU to be skipped during a scan when it is known from the scan predicates that no matching value will be found within the IMCU.
2. *In-Memory Optimized Joins and Reporting*: As a result of massive increases in scan speeds, the Bloom Filteroptimization (introduced earlier in Oracle Database 10g) can be commonly selected by the optimizer. With the Bloom Filter optimization, the scan of the outer (dimension) table generates a compact bloom filter which can then be used to greatly reduce the amount of data processed by the join from the scan of the inner (fact) table. Similarly, an optimization known as Vector Group Bycan be used to reduce a complex aggregation query on a typical star schema into a series of filtered scans against the dimension and fact tables.

## In-Memory Architecture

The following figure shows a sample IM column store. The database stores the sh.sales table on disk in traditional row format. The SGA stores the data in columnar format in the IM column store, and in row format in the database buffer cache.

[![ ](https://github.com/vijaybalebail/learning-library/raw/master/data-management-library/database/in-memory/intro/images/arch.png)](https://github.com/vijaybalebail/learning-library/blob/master/data-management-library/database/in-memory/intro/images/arch.png)

## More Information on In-Memory

Database In-Memory Channel [![ ](https://github.com/vijaybalebail/learning-library/raw/master/data-management-library/database/in-memory/intro/images/inmem.png)](https://www.youtube.com/channel/UCSYHgTG68nrHa5aTGfFH4pA)

Oracle Database Product Management Videos on In-Memory [![ ](https://github.com/vijaybalebail/learning-library/raw/master/data-management-library/database/in-memory/intro/images/youtube.png)](https://www.youtube.com/channel/UCr6mzwq_gcdsefQWBI72wIQ/search?query=in-memory)

Please proceed to the next lab.

## Acknowledgements

- **Authors/Contributors** - Andy Rivenes, Senior Principal Product Manager, In-Memory
- **Last Updated By/Date** - Kay Malcolm, March 2020
- **Workshop Expiration Date** - March 31, 2021

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues).

<details class="details-reset details-overlay details-overlay-dark" id="jumpto-line-details-dialog" style="box-sizing: border-box; display: block;"><summary data-hotkey="l" aria-label="Jump to line" role="button" style="box-sizing: border-box; display: list-item; cursor: pointer; list-style: none;"></summary></details>

- © 2020 GitHub, Inc.
- [Terms](https://github.com/site/terms)
- [Privacy](https://github.com/site/privacy)
- [Security](https://github.com/security)
- [Status](https://githubstatus.com/)
- [Help](https://docs.github.com/)



- [Contact GitHub](https://github.com/contact)
- [Pricing](https://github.com/pricing)
- [API](https://docs.github.com/)
- [Training](https://services.github.com/)
- [Blog](https://github.blog/)
- [About](https://github.com/about)