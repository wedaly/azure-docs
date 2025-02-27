---
title: Azure SQL indexer
titleSuffix: Azure Cognitive Search
description: Set up a search indexer to index data stored in Azure SQL Database for full text search in Azure Cognitive Search.

manager: nitinme
author: HeidiSteen
ms.author: heidist
ms.service: cognitive-search
ms.topic: how-to
ms.date: 02/28/2022
---

# Index data from Azure SQL

In this article, learn how to configure an [**indexer**](search-indexer-overview.md) that imports content from Azure SQL Database or an Azure SQL managed instance and makes it searchable in Azure Cognitive Search. 

This article supplements [**Create an indexer**](search-howto-create-indexers.md) with information that's specific to Azure SQL. It uses the REST APIs to demonstrate a three-part workflow common to all indexers: create a data source, create an index, create an indexer. Data extraction occurs when you submit the Create Indexer request.

## Prerequisites

+ An [Azure SQL database](/azure/azure-sql/database/sql-database-paas-overview) with data in a single table or view. Use a table if you want the ability to [index incremental updates](#CaptureChangedRows) using SQL's native change detection capabilities.

+ Read permissions. Azure Cognitive Search supports SQL Server authentication, where the user name and password are provided on the connection string. Alternatively, you can [set up a managed identity and use Azure roles](search-howto-managed-identities-sql.md) to omit credentials on the connection.

+ A REST client, such as [Postman](search-get-started-rest.md) or [Visual Studio Code with the extension for Azure Cognitive Search](search-get-started-vs-code.md) to send REST calls that create the data source, index, and indexer. 

<!-- Real-time data synchronization must not be an application requirement. An indexer can reindex your table at most every five minutes. If your data changes frequently, and those changes need to be reflected in the index within seconds or single minutes, we recommend using the [REST API](/rest/api/searchservice/AddUpdate-or-Delete-Documents) or [.NET SDK](search-get-started-dotnet.md) to push updated rows directly.

Incremental indexing is possible. If you have a large data set and plan to run the indexer on a schedule, Azure Cognitive Search must be able to efficiently identify new, changed, or deleted rows. Non-incremental indexing is only allowed if you're indexing on demand (not on schedule), or indexing fewer than 100,000 rows. For more information, see [Capturing Changed and Deleted Rows](#CaptureChangedRows) below. -->

## Define the data source

The data source definition specifies the data to index, credentials, and policies for identifying changes in the data. A data source is defined as an independent resource so that it can be used by multiple indexers.

1. [Create or update a data source](/rest/api/searchservice/create-data-source) to set its definition: 

   ```http
    POST https://myservice.search.windows.net/datasources?api-version=2020-06-30
    Content-Type: application/json
    api-key: admin-key

    {
        "name" : "myazuresqldatasource",
        "type" : "azuresql",
        "credentials" : { "connectionString" : "Server=tcp:<your server>.database.windows.net,1433;Database=<your database>;User ID=<your user name>;Password=<your password>;Trusted_Connection=False;Encrypt=True;Connection Timeout=30;" },
        "container" : { "name" : "name of the table or view that you want to index" }
    }
   ```

1. Set "type" to `"azuresql"` (required).

1. Set "credentials" to a connection string:

   + You can get the connection string from the [Azure portal](https://portal.azure.com). Use the `ADO.NET connection string` option.

   + You can specify a managed identity connection string that does not include database secrets with the following format: `Initial Catalog|Database=<your database name>;ResourceId=/subscriptions/<your subscription ID>/resourceGroups/<your resource group name>/providers/Microsoft.Sql/servers/<your SQL Server name>/;Connection Timeout=connection timeout length;`.

    To use this connection string, follow the instructions for [Setting up an indexer connection to an Azure SQL Database using a managed identity](search-howto-managed-identities-sql.md).

## Add search fields to an index

In a [search index](search-what-is-an-index.md), add fields to accept values from corresponding fields in the SQL database. Ensure that the search index schema is compatible with source schema, with [equivalent data types](#TypeMapping).

1. [Create or update an index](/rest/api/searchservice/create-index) to define search fields that will store data:

    ```http
    POST https://[service name].search.windows.net/indexes?api-version=2020-06-30
    Content-Type: application/json
    api-key: [Search service admin key]
    {
        "name": "mysearchindex",
        "fields": [{
            "name": "id",
            "type": "Edm.String",
            "key": true,
            "searchable": false
        }, 
        {
            "name": "description",
            "type": "Edm.String",
            "filterable": false,
            "searchable": true,
            "sortable": false,
            "facetable": false,
            "suggestions": true
        }
      ]
    }
    ```

1. Create a document key field ("key": true) that uniquely identifies each search document. This is the only field that's required. Typically, the table's primary key is mapped to the index key field. The document key must be unique and non-null. The values can be numeric in source data, but in a search index, a key is always a string.

1. Create additional fields for more searchable content. See [Create an index](search-how-to-create-search-index.md) for details.

<a name="TypeMapping"></a>

### Mapping data types

| SQL data type | Cognitive Search field types | Notes |
| ------------- | -------------------------------- | --- |
| bit |Edm.Boolean, Edm.String | |
| int, smallint, tinyint |Edm.Int32, Edm.Int64, Edm.String | |
| bigint |Edm.Int64, Edm.String | |
| real, float |Edm.Double, Edm.String | |
| smallmoney, money decimal numeric |Edm.String |Azure Cognitive Search does not support converting decimal types into Edm.Double because doing so would lose precision |
| char, nchar, varchar, nvarchar |Edm.String<br/>Collection(Edm.String) |A SQL string can be used to populate a Collection(Edm.String) field if the string represents a JSON array of strings: `["red", "white", "blue"]` |
| smalldatetime, datetime, datetime2, date, datetimeoffset |Edm.DateTimeOffset, Edm.String | |
| uniqueidentifer |Edm.String | |
| geography |Edm.GeographyPoint |Only geography instances of type POINT with SRID 4326 (which is the default) are supported |
| rowversion |Not applicable |Row-version columns cannot be stored in the search index, but they can be used for change tracking |
| time, timespan, binary, varbinary, image, xml, geometry, CLR types |Not applicable |Not supported |

## Configure and run the Azure SQL indexer

Once the index and data source have been created, you're ready to create the indexer. Indexer configuration specifies the inputs, parameters, and properties controlling run time behaviors.

1. [Create or update an indexer](/rest/api/searchservice/create-indexer) by giving it a name and referencing the data source and target index:

    ```http
    POST https://[service name].search.windows.net/indexers?api-version=2020-06-30
    Content-Type: application/json
    api-key: [search service admin key]
    {
        "name" : "[my-sqldb-indexer]",
        "dataSourceName" : "[my-sqldb-ds]",
        "targetIndexName" : "[my-search-index]",
        "disabled": null,
        "schedule": null,
        "parameters": {
            "batchSize": null,
            "maxFailedItems": 0,
            "maxFailedItemsPerBatch": 0,
            "base64EncodeKeys": false,
            "configuration": {
                "queryTimeout": "00:05:00",
                "disableOrderByHighWaterMarkColumn": false
            }
        },
        "fieldMappings": [],
        "encryptionKey": null
    }
    ```

1. Under parameter configuration, you can set a timeout for SQL query execution. In the example above, the timeout is 5 minutes. The second configuration setting is "disableOrderByHighWaterMarkColumn". It causes the SQL query used by the [high water mark policy](#HighWaterMarkPolicy) to omit the ORDER BY clause.

1. [Specify field mappings](search-indexer-field-mappings.md) if there are differences in field name or type, or if you need multiple versions of a source field in the search index.

1. See [Create an indexer](search-howto-create-indexers.md) for more information about other properties.

An indexer runs automatically when it's created. You can prevent this by setting "disabled" to true. To control indexer execution, [run an indexer on demand](search-howto-run-reset-indexers.md) or [put it on a schedule](search-howto-schedule-indexers.md).

## Check indexer status

To monitor the indexer status and execution history, send a [Get Indexer Status](/rest/api/searchservice/get-indexer-status) request:

```http
GET https://myservice.search.windows.net/indexers/myindexer/status?api-version=2020-06-30
  Content-Type: application/json  
  api-key: [admin key]
```

The response includes status and the number of items processed. It should look similar to the following example:

```json
    {
        "status":"running",
        "lastResult": {
            "status":"success",
            "errorMessage":null,
            "startTime":"2022-02-21T00:23:24.957Z",
            "endTime":"2022-02-21T00:36:47.752Z",
            "errors":[],
            "itemsProcessed":1599501,
            "itemsFailed":0,
            "initialTrackingState":null,
            "finalTrackingState":null
        },
        "executionHistory":
        [
            {
                "status":"success",
                "errorMessage":null,
                "startTime":"2022-02-21T00:23:24.957Z",
                "endTime":"2022-02-21T00:36:47.752Z",
                "errors":[],
                "itemsProcessed":1599501,
                "itemsFailed":0,
                "initialTrackingState":null,
                "finalTrackingState":null
            },
            ... earlier history items
        ]
    }
```

Execution history contains up to 50 of the most recently completed executions, which are sorted in the reverse chronological order so that the latest execution comes first.

<a name="CaptureChangedRows"></a>

## Indexing new, changed, and deleted rows

If your SQL database supports [change tracking](/sql/relational-databases/track-changes/about-change-tracking-sql-server), a search indexer can pick up just the new and updated content on subsequent indexer runs. 

To enable incremental indexing, set the "dataChangeDetectionPolicy" property in your data source definition. This property tells the indexer which change tracking mechanism is used on your table or view. 

For Azure SQL indexers, there two change detection policies: 

+ "SqlIntegratedChangeTrackingPolicy" (applies to tables only)

+ "HighWaterMarkChangeDetectionPolicy" (works for tables and views)

### SQL Integrated Change Tracking Policy

We recommend using "SqlIntegratedChangeTrackingPolicy" for its efficiency and its ability to identify deleted rows.

Database requirements:

+ SQL Server 2012 SP3 and later, if you're using SQL Server on Azure VMs
+ Azure SQL Database or SQL Managed Instance
+ Tables only (no views).
+ On the database, [enable change tracking](/sql/relational-databases/track-changes/enable-and-disable-change-tracking-sql-server) for the table
+ No composite primary key (a primary key containing more than one column) on the table

Change detection policies are added to data source definitions. To use this policy, create or update your data source like this:

```http
POST https://myservice.search.windows.net/datasources?api-version=2020-06-30
Content-Type: application/json
api-key: admin-key
    {
        "name" : "myazuresqldatasource",
        "type" : "azuresql",
        "credentials" : { "connectionString" : "connection string" },
        "container" : { "name" : "table name" },
        "dataChangeDetectionPolicy" : {
            "@odata.type" : "#Microsoft.Azure.Search.SqlIntegratedChangeTrackingPolicy"
    }
```

When using SQL integrated change tracking policy, do not specify a separate data deletion detection policy. The SQL integrated change tracking policy has built-in support for identifying deleted rows. However, for the deleted rows to be detected automatically, the document key in your search index must be the same as the primary key in the SQL table. 

> [!NOTE]  
> When using [TRUNCATE TABLE](/sql/t-sql/statements/truncate-table-transact-sql) to remove a large number of rows from a SQL table, the indexer needs to be [reset](/rest/api/searchservice/reset-indexer) to reset the change tracking state to pick up row deletions.

<a name="HighWaterMarkPolicy"></a>

### High Water Mark Change Detection policy

This change detection policy relies on a "high water mark" column in your table or view that captures the version or time when a row was last updated. If you're using a view, you must use a high water mark policy. 

The high water mark column must meet the following requirements:

+ All inserts specify a value for the column.
+ All updates to an item also change the value of the column.
+ The value of this column increases with each insert or update.
+ Queries with the following WHERE and ORDER BY clauses can be executed efficiently: `WHERE [High Water Mark Column] > [Current High Water Mark Value] ORDER BY [High Water Mark Column]`

> [!NOTE]
> We strongly recommend using the [rowversion](/sql/t-sql/data-types/rowversion-transact-sql) data type for the high water mark column. If any other data type is used, change tracking is not guaranteed to capture all changes in the presence of transactions executing concurrently with an indexer query. When using **rowversion** in a configuration with read-only replicas, you must point the indexer at the primary replica. Only a primary replica can be used for data sync scenarios.

Change detection policies are added to data source definitions. To use this policy, create or update your data source like this:

```http
POST https://myservice.search.windows.net/datasources?api-version=2020-06-30
Content-Type: application/json
api-key: admin-key
    {
        "name" : "myazuresqldatasource",
        "type" : "azuresql",
        "credentials" : { "connectionString" : "connection string" },
        "container" : { "name" : "table or view name" },
        "dataChangeDetectionPolicy" : {
            "@odata.type" : "#Microsoft.Azure.Search.HighWaterMarkChangeDetectionPolicy",
            "highWaterMarkColumnName" : "[a rowversion or last_updated column name]"
        }
    }
```

> [!NOTE]
> If the source table does not have an index on the high water mark column, queries used by the SQL indexer may time out. In particular, the `ORDER BY [High Water Mark Column]` clause requires an index to run efficiently when the table contains many rows.

<a name="convertHighWaterMarkToRowVersion"></a>

##### convertHighWaterMarkToRowVersion

If you're using a [rowversion](/sql/t-sql/data-types/rowversion-transact-sql) data type for the high water mark column, consider setting the `convertHighWaterMarkToRowVersion` property in indexer configuration. Setting this property to true results in the following behaviors: 

* Uses the rowversion data type for the high water mark column in the indexer SQL query. Using the correct data type improves indexer query performance.

* Subtracts one from the rowversion value before the indexer query runs. Views with one-to-many joins may have rows with duplicate rowversion values. Subtracting 1one ensures the indexer query doesn't miss these rows.

To enable this property, create or update the indexer with the following configuration:

```http
    {
      ... other indexer definition properties
     "parameters" : {
            "configuration" : { "convertHighWaterMarkToRowVersion" : true } }
    }
```

<a name="queryTimeout"></a>

##### queryTimeout

If you encounter timeout errors, set the `queryTimeout` indexer configuration setting to a value higher than the default 5-minute timeout. For example, to set the timeout to 10 minutes, create or update the indexer with the following configuration:

```http
    {
      ... other indexer definition properties
     "parameters" : {
            "configuration" : { "queryTimeout" : "00:10:00" } }
    }
```

<a name="disableOrderByHighWaterMarkColumn"></a>

##### disableOrderByHighWaterMarkColumn

You can also disable the `ORDER BY [High Water Mark Column]` clause. However, this is not recommended because if the indexer execution is interrupted by an error, the indexer has to re-process all rows if it runs later, even if the indexer has already processed almost all the rows at the time it was interrupted. To disable the `ORDER BY` clause, use the `disableOrderByHighWaterMarkColumn` setting in the indexer definition:  

```http
    {
     ... other indexer definition properties
     "parameters" : {
            "configuration" : { "disableOrderByHighWaterMarkColumn" : true } }
    }
```

### Soft Delete Column Deletion Detection policy

When rows are deleted from the source table, you probably want to delete those rows from the search index as well. If you use the SQL integrated change tracking policy, this is taken care of for you. However, the high water mark change tracking policy doesn’t help you with deleted rows. What to do?

If the rows are physically removed from the table, Azure Cognitive Search has no way to infer the presence of records that no longer exist.  However, you can use the “soft-delete” technique to logically delete rows without removing them from the table. Add a column to your table or view and mark rows as deleted using that column.

When using the soft-delete technique, you can specify the soft delete policy as follows when creating or updating the data source:

```http
    {
        …,
        "dataDeletionDetectionPolicy" : {
           "@odata.type" : "#Microsoft.Azure.Search.SoftDeleteColumnDeletionDetectionPolicy",
           "softDeleteColumnName" : "[a column name]",
           "softDeleteMarkerValue" : "[the value that indicates that a row is deleted]"
        }
    }
```

The **softDeleteMarkerValue** must be a string in the JSON representation of your data source. Use the string representation of your actual value. For example, if you have an integer column where deleted rows are marked with the value 1, use `"1"`. If you have a BIT column where deleted rows are marked with the Boolean true value, use the string literal `"True"` or `"true"`, the case doesn't matter.

If you are setting up a soft delete policy from the Azure portal, don't add quotes around the soft delete marker value. The field contents are already understood as a string and will be translated automatically into a JSON string for you. In the examples above, simply type `1`, `True` or `true` into the portal's field.

## FAQ

**Q: Can I use Azure SQL indexer with SQL databases running on IaaS VMs in Azure?**

Yes. However, you need to allow your search service to connect to your database. For more information, see [Configure a connection from an Azure Cognitive Search indexer to SQL Server on an Azure VM](search-howto-connecting-azure-sql-iaas-to-azure-search-using-indexers.md).

**Q: Can I use Azure SQL indexer with SQL databases running on-premises?**

Not directly. We do not recommend or support a direct connection, as doing so would require you to open your databases to Internet traffic. Customers have succeeded with this scenario using bridge technologies like Azure Data Factory. For more information, see [Push data to an Azure Cognitive Search index using Azure Data Factory](../data-factory/v1/data-factory-azure-search-connector.md).

**Q: Can I use a secondary replica in a [failover cluster](/azure/azure-sql/database/auto-failover-group-overview) as a data source?**

It depends. For full indexing of a table or view, you can use a secondary replica. 

For incremental indexing, Azure Cognitive Search supports two change detection policies: SQL integrated change tracking and High Water Mark.

On read-only replicas, SQL Database does not support integrated change tracking. Therefore, you must use High Water Mark policy. 

Our standard recommendation is to use the rowversion data type for the high water mark column. However, using rowversion relies on the `MIN_ACTIVE_ROWVERSION` function, which is not supported on read-only replicas. Therefore, you must point the indexer to a primary replica if you are using rowversion.

If you attempt to use rowversion on a read-only replica, you will see the following error: 

"Using a rowversion column for change tracking is not supported on secondary (read-only) availability replicas. Please update the datasource and specify a connection to the primary availability replica.Current database 'Updateability' property is 'READ_ONLY'".

**Q: Can I use an alternative, non-rowversion column for high water mark change tracking?**

It's not recommended. Only **rowversion** allows for reliable data synchronization. However, depending on your application logic, it may be safe if:

+ You can ensure that when the indexer runs, there are no outstanding transactions on the table that’s being indexed (for example, all table updates happen as a batch on a schedule, and the Azure Cognitive Search indexer schedule is set to avoid overlapping with the table update schedule).  

+ You periodically do a full reindex to pick up any missed rows.