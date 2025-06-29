# Azure SQL MI & Azure SQL Columnstore Index & Other Compression Options

# To columnstore, or not to columnstore that is the question.

Optimizing storage on Azure SQL Managed Instance systems has become an essential responsibility for data engineers, especially since increasing storage capacity also necessitates allocating additional vCores. 
This documentation explores various strategies for conserving storage space, allowing teams to delay the need for costly disk expansions until all other options have been considered.

# Test Case 1: Columnstore Index

Utilizing columnstore indexes can lead to substantial storage savings for tables, often achieving up to a tenfold reduction in space requirements under default configurations. 
However, it is important to note that columnstore indexes are not suitable for all scenarios; tables with frequent updates and deletions may experience suboptimal performance. 
Regular table maintenance is also crucial to ensure that row groups are properly closed and storage remains optimized. 
Additionally, nonclustered indexes can be employed to facilitate rapid data retrieval for smaller datasets. For further details, please refer to the resources below: 
Reference:  <a href="https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-design-guidance?view=sql-server-ver17" target="_blank">Columnstore indexes - design guidance</a>
For instance, the following table illustrates a comparison between rowstore and columnstore formats, highlighting the dramatic reduction in storage size achieved with columnstore. Particularly for columns containing similar data types or 
integers, the space savings can be significant, often resulting in exponential compression benefits.


Converting table to a clustered column store:

```CREATE CLUSTERED COLUMNSTORE INDEX CCI_Sales_Summary ON dbo.Sales_Summary;```

Important Considerations:
All existing nonclustered indexes will remain unless explicitly dropped.
Primary key and unique constraints that rely on the clustered index will be dropped unless they are supported by a nonclustered index.
You cannot have both a clustered B-tree index and a clustered columnstore index on the same table.


## Rowstore: 
name	rows	reserved	data	index_size	unused
VicSummaryChange4	51200	76296 KB	76216 KB	8 KB	72 KB
![image](https://github.com/user-attachments/assets/c741b35d-0224-4db3-b15d-36eb4112eb36)




## Columnstore: 
name	rows	reserved	data	index_size	unused
VicSummaryChange4	51200	328 KB	168 KB	0 KB	160 KB
![image](https://github.com/user-attachments/assets/8be19a45-6399-49db-aaa3-2ae831a9dac5)


# Test Case 2: Table & Index ROW & PAGE Level Compression

Another effective strategy involves implementing row-level or table-level compression, which can yield significant storage savings particularly for tables 
that contain multiple indexes. To assess the potential space savings, begin by executing the following SQL command:
``` 
EXEC sp_estimate_data_compression_savings 
    @schema_name = 'schema_name',
    @object_name = 'table_name',
    @index_id = NULL, -- NULL = all indexes
    @partition_number = NULL, -- NULL = all partitions
    @data_compression = 'PAGE'; -- PAGE or ROW
```

Once you have confirmed that the anticipated savings are substantial, you can apply compression to either index or table objects using the appropriate commands:

## Index: 
```ALTER INDEX index_name ON schema.table_name REBUILD WITH (DATA_COMPRESSION = PAGE);```

## Table: 
```
ALTER TABLE schema.table_name
REBUILD 
WITH (DATA_COMPRESSION = PAGE);  --Or ROW 
```

# Test Case 3: Rebuild Heap Tables
Although the potential space savings may not be as substantial as those achieved through columnstore or compression techniques, it is worthwhile to evaluate heap tables for fragmentation, a factor often overlooked. 
The following script identifies heap tables exhibiting significant fragmentation, providing a clear indication of candidates that would benefit from a rebuild.

```
SELECT 
    s.name AS SchemaName,
    t.name AS TableName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM 
    sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') AS ips
    JOIN sys.tables t ON ips.object_id = t.object_id
    JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE 
    ips.index_id = 0 -- 0 = heap
    AND ips.page_count > 1000
    AND ips.avg_fragmentation_in_percent > 20
ORDER BY 
    ips.avg_fragmentation_in_percent DESC;
```

To evaluate the current storage footprint of your tables, execute the following query:
```EXEC sp_spaceused 'schema_name.table_name';```

After performing the recommended rebuild operation using the corresponding command provided below, re-run the query to compare post-rebuild space utilization and assess the effectiveness of the fragmentation remediation.

```ALTER TABLE schema_name.table_name REBUILD;```

## Finally: 
To ensure optimal results and compatibility within your environment, it is essential to conduct thorough testing at each stage of the process.


***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. 
THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS 
FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) 
to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is 
embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the 
Sample Code.***
