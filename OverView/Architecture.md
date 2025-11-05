# [Database Features](https://docs.starrocks.io/docs/introduction/Features/)
## [MPP framework](https://docs.starrocks.io/docs/introduction/Features/#mpp-framework)

Unlike the Scatter-Gather framework used by many other data analytics systems, the **MPP framework** can utilize more resources to process query requests.  
In the Scatter-Gather framework, _**only the Gather node can perform the final merge operation**_.  
_**In the MPP**_ framework, _**data is shuffled to multiple nodes for merge**_ operations.   
