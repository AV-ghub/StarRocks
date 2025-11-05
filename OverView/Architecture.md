# [Database Features](https://docs.starrocks.io/docs/introduction/Features/)
## [MPP framework](https://docs.starrocks.io/docs/introduction/Features/#mpp-framework)

Unlike the Scatter-Gather framework used by many other data analytics systems, the **MPP framework** can utilize more resources to process query requests.  
In the Scatter-Gather framework, _**only the Gather node can perform the final merge operation**_.  
_**In the MPP**_ framework, _**data is shuffled to multiple nodes for merge**_ operations.   

## [Fully vectorized execution engine](https://docs.starrocks.io/docs/introduction/Features/#fully-vectorized-execution-engine)

The fully vectorized execution engine makes more efficient use of CPU processing power because this engine _**organizes and processes data in a columnar manner**_.  
The vectorized execution engine also makes _**full use of SIMD instructions**_.   
This engine can complete _**more data operations with fewer instructions**_.   
Tests against standard datasets show that this engine enhances the overall performance of operators by _**3 to 10 times**_.   

StarRocks uses the Operation on _**Encoded Data technology to directly execute operators on encoded strings**_, without the need for decoding.   
This noticeably reduces SQL complexity and increases the query speed by more than _**2 times**_.  

## [Separation of storage and compute](https://docs.starrocks.io/docs/introduction/Features/#separation-of-storage-and-compute)

Computing and storage are decoupled and can be _**scaled independently**_.   
Computing can _**dynamically scale within seconds**_, improving resource utilization, especially when there are noticeable traffic peaks and valleys.  

## [Cost-based optimizer](https://docs.starrocks.io/docs/introduction/Features/#cost-based-optimizer)

_**Execution engines alone cannot deliver superior performance because the complexity**_ of execution plans may vary by several orders of magnitude in multi-table join query scenarios. The more the associated tables, the more the execution plans, which makes it NP-_**hard to choose an optimal plan**_.   
_**Only a query optimizer excellent enough**_ can choose a relatively optimal query plan for efficient **multi-table analytics**.

The CBO enables StarRocks to deliver _**better multi-table join query performance**_ than competitors, especially in complex multi-table join queries.

## [Real-time, updatable columnar storage engine](https://docs.starrocks.io/docs/introduction/Features/#real-time-updatable-columnar-storage-engine)

StarRocks' storage engine guarantees the _**atomicity, consistency, isolation, and durability (ACID)**_ of each data ingestion operation.  
For a data loading transaction, the _**entire transaction either succeeds or fails**_.   
Concurrent transactions do not affect each other, providing _**transaction-level isolation**_.  

StarRocks' storage engine uses the _**Delete-and-insert pattern**_, which allows for efficient _**Partial Update and Upsert**_ operations.   
The storage engine can quickly _**filter data using primary key indexes**_, eliminating the need for Sort and Merge operations at data reading.   
The engine can also make _**full use of secondary indexes**_.   
It delivers fast and predictable query performance even on _**huge volume of data updates**_.

## [Intelligent materialized view](https://docs.starrocks.io/docs/introduction/Features/#intelligent-materialized-view)

StarRocks' materialized views _**automatically update data**_ according to the data changes in the base table without requiring additional maintenance operations.   
In addition, the _**selection of materialized views is also automatic**_.   
If StarRocks identifies a suitable materialized view (MV) to improve query performance, it will _**automatically rewrite the query to utilize the MV**_. 

Raw data can be used to _**create a normalized table based on an external MV**_.   
A _**denormalized table can be created from normalized**_ tables through asynchronous materialized views.    
_**Another MV can be created from normalized tables**_.










