# Optimize Azure SQL Database Bulk Upsert scenarios

Azure SQL Database provides high availability out of the box, as clearly described in this article: <https://docs.microsoft.com/en-us/azure/sql-database/sql-database-high-availability>

Even if different service tiers are providing  this capability through different underlying implementations, generally speaking in Azure SQL Database things like minimal logging, simple or bulk logged recovery modes are just not available and every operation on persistent tables is fully logged.

Well known techniques can be leveraged to minimize the impact of fully logged database operations in traditional workloads: <https://docs.microsoft.com/en-us/azure/sql-database/sql-database-use-batching-to-improve-performance>

For other scenarios like bulk loading or bulk insert this can significantly impact performance if compared to on premises systems where minimal logging is available. 

This example demonstrates how to optimize a specific scenario where customers need to regularly update large datasets into Azure SQL Database, and then execute upsert activities that will either modify existing records if they already exists (by key) in a target table, or insert them if they don't.

Generally speaking, there 2 major approaches to achieve this:

1) Iterate on the dataset on the application tier, and for every row invoke a stored proc that will execute an INSERT/UPDATE operation depending on the existance of record with a certain key. This approach can work well if the amount of records to upsert is relatively small, otherwise roundtrips and log writes will significantly impact performance.

2) Leverage bulk insert techniques, like using SqlBulkCopy class in ADO.NET, to upload the entire dataset to Azure SQL Database, and then execute all the INSERT/UPDATE (or MERGE) operation within a single batch, to mininize roundtrips and log writes and maximize throughput. This approach can reduce overall execution times from hours to minutes/seconds, even if the incoming dataset is made of millions of records.

When using data integration services like Azure Data Factory, scenarios like #1 are usually provided out of the box, as described here: <https://docs.microsoft.com/en-us/azure/data-factory/connector-azure-sql-database#invoking-stored-procedure-for-sql-sink>

Implementing instead something like described in #2 requires a bit of workaround, as it will depend more on specific scenario requirements that may vary on a customer by customer basis.

An example on how to implement such a scenario is what is provided in this article from now on.

