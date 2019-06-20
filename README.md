# Optimize Azure SQL Database Bulk Upsert scenarios

Azure SQL Database provides high availability out of the box, as clearly described in this article: <https://docs.microsoft.com/en-us/azure/sql-database/sql-database-high-availability>

Even if different service tiers are providing  this capability through different underlying implementations, generally speaking in Azure SQL Database things like minimal logging, simple or bulk logged recovery modes are just not available and every operation on persistent tables is fully logged.

Well known techniques can be leveraged to minimize the impact of fully logged database operations in traditional workloads: <https://docs.microsoft.com/en-us/azure/sql-database/sql-database-use-batching-to-improve-performance>

For other scenarios like bulk loading or bulk insert this can significantly impact performance if compared to on premises systems where minimal logging is available. Limits in log generation rate for both general purpose and mission critical service tiers are documented here (<https://docs.microsoft.com/en-us/azure/sql-database/sql-database-vcore-resource-limits-single-databases#business-critical-service-tier-for-provisioned-compute-tier>) and cannot be crossed.

This example demonstrates how to optimize a specific scenario where customers need to regularly update large datasets into Azure SQL Database, and then execute upsert activities that will either modify existing records if they already exists (by key) in a target table, or insert them if they don't.

Generally speaking, there 2 major approaches to achieve this:

1) Iterate on the dataset on the application tier, and for every row invoke a stored proc that will execute an INSERT/UPDATE operation depending on the existence of record with a certain key. This approach can work well if the amount of records to upsert is relatively small, otherwise roundtrips and log writes will significantly impact performance.

2) Leverage bulk insert techniques, like using SqlBulkCopy class in ADO.NET, to upload the entire dataset to Azure SQL Database, and then execute all the INSERT/UPDATE (or MERGE) operation within a single batch, to mininize roundtrips and log writes and maximize throughput. This approach can reduce overall execution times from hours to minutes/seconds, even if the incoming dataset is made of millions of records.

When using data integration services like Azure Data Factory, scenarios like #1 are usually provided out of the box, as described here: <https://docs.microsoft.com/en-us/azure/data-factory/connector-azure-sql-database#invoking-stored-procedure-for-sql-sink>

Implementing something like described in #2 instead does requires a bit of workaround, as it will depend more on specific scenario requirements that may vary on a customer by customer basis.

An example on how to implement such a scenario is what is provided in this article from now on.

We should start from an important point: previous paragraph mentioning minimal logging not available for Azure SQL Database has an exception, which is when you're bulk loading into a temporary table. With this approach, you can get much higher data loading throughput for larger dataset, although you must always consider that this can influence other activities that are equally leveraging tempdb (even if, in general, data loading scenarios are happening on different time windows compared to regular workloads).

In the article that can be found here (<https://docs.microsoft.com/en-us/azure/data-factory/connector-azure-sql-database#best-practice-for-loading-data-into-azure-sql-database>), it is recommended to leverage Database-scoped temporary table for the job, in order to let multiple sessions accessing the same temp table created in a different session.

This is correct, but it comes with a caveat: global temporary table in fact will get immediately deleted when there isn't any active session that keeps a reference to that (i.e. the session who created the table, or any other maintaining a reference to it).

When using a service like Azure Data Factory to orchestrate multiple activities required to execute an upsert scenario like described, you don't have an automatic construct that you can use to keep a session open across all tasks that compose a pipeline, so you'll need a simple workaround.

In this code repo, you'll find 3 separate pipelines:

1) A "OrchestrateDataLoad" which is the parent pipeline, just a wrapper that invokes the following two.

2) A "PrepareGlobalTempTable" that will invoke the "spCreateTempTable" stored procedure (see script.sql file) that creates the "##mytemptable" global temp table for data loading, and an accessory control table ("mycontroltable") that will be used to signal the completion of the overall data loading process. When invoked, the stored proc contains a loop that will periodically check control table (maintaining the underlying session active) and will exit when the next pipeline will conclude the upsert procedure. This pipeline has the "Wait for completion" property set to "false", to let the parent pipeline proceed with invoking the subsequent child pipeline.

3) A "CopyAndMerge" pipeline that is using a CopyData activity to bulk load source data into the global temp table, then will execute the "spMergeData" stored proc that effectively merges data into the final/persistent target table ("mytargettable"). Once completed, it signals back to the previous stored proc that the operation has finished by updating the control table.

This example could be easily improved to include the ability to execute multiple upsert pipelines in parallel hitting multiple target tables by extending control table and logic and manage table names and such.

