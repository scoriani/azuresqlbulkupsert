# Optimize Azure SQL Database Bulk Upsert scenarios

Azure SQL Database provides high availability out of the box, as clearly described in this article: <https://docs.microsoft.com/en-us/azure/sql-database/sql-database-high-availability>

Even if different service tiers are providing  this capability through different underlying implementations, generally speaking in Azure SQL Database things like minimal logging, simple or bulk logged recovery modes are just not available and every operation on persistent tables is fully logged.

Well known techniques can be leveraged to minimize the impact of fully logged database operations in traditional workloads: <https://docs.microsoft.com/en-us/azure/sql-database/sql-database-use-batching-to-improve-performance>

For other scenarios like bulk loading or bulk insert this can be more challenging. 
