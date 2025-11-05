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

