---
# Mandatory fields.
title: Querying with Azure Data Explorer
titleSuffix: Azure Digital Twins
description: Understand the Azure Digital Twins query plugin for Azure Data Explorer
author: baanders
ms.author: baanders # Microsoft employees only
ms.date: 5/19/2021
ms.topic: conceptual
ms.service: digital-twins

# Optional fields. Don't forget to remove # if you need a field.
# ms.custom: can-be-multiple-comma-separated
# ms.reviewer: MSFT-alias-of-reviewer
# manager: MSFT-alias-of-manager-or-PM-counterpart
---

# Azure Digital Twins query plugin for Azure Data Explorer

The Azure Digital Twins plugin for [Azure Data Explorer (ADX)](/azure/data-explorer/data-explorer-overview) lets you run ADX queries that access and combine data across the Azure Digital Twins graph and ADX time series databases. Use the plugin to contextualize disparate time series data by reasoning across digital twins and their relationships to gain insights into the behavior of modeled environments.

For example, with this plugin, you can write a KQL query that...
1. selects digital twins of interest via the Azure Digital Twins query plugin,
2. joins those twins against the respective times series in ADX, and then 
3. performs advanced time series analytics on those twins.  

Combining data from a twin graph in Azure Digital Twins with time series data in ADX can help you understand the operational behavior of various parts of your solution. 

## Using the plugin

In order to get the plugin running on your own ADX cluster that contains time series data, start by running the following command in ADX in order to enable the plugin:

```kusto
.enable plugin azure_digital_twins_query_request
```

This command requires **All Databases admin** permission. For more information on the command, see the [.enable plugin documentation](/azure/data-explorer/kusto/management/enable-plugin). 

Once the plugin is enabled, you can invoke it within an ADX Kusto query with the following command. There are two placeholders, `<Azure-Digital-Twins-endpoint>` and `<Azure-Digital-Twins-query>`, which are strings representing the Azure Digital Twins instance endpoint and Azure Digital Twins query, respectively. 

```kusto
evaluate azure_digital_twins_query_request(<Azure-Digital-Twins-endpoint>, <Azure-Digital-Twins-query>) 
```

The plugin works by calling the [Azure Digital Twins query API](/rest/api/digital-twins/dataplane/query), and the [query language structure](concepts-query-language.md) is the same as when using the API. 

>[!IMPORTANT]
>The user of the plugin must be granted the **Azure Digital Twins Data Reader** role or the **Azure Digital Twins Data Owner** role, as the user's Azure AD token is used to authenticate. Information on how to assign this role can be found in [Concepts: Security for Azure Digital Twins solutions](concepts-security.md#authorization-azure-roles-for-azure-digital-twins).

For more information on using the plugin, see the [Kusto documentation for the azure_digital_twins_query_request plugin](/azure/data-explorer/kusto/query/azure-digital-twins-query-request-plugin).

To see example queries and complete a walkthrough with sample data, see [Azure Digital Twins query plugin for ADX: Sample queries and walkthrough](https://github.com/Azure-Samples/azure-digital-twins-getting-started/tree/main/adt-adx-queries) in GitHub.

## Using ADX IoT data with Azure Digital Twins

There are various ways to ingest IoT data into ADX. Here are two that you might use when using ADX with Azure Digital Twins:
* Historize digital twin property values to ADX with an Azure function that handles twin change events and writes the twin data to ADX, similar to the process used in [How-to: Integrate with Azure Time Series Insights](how-to-integrate-time-series-insights.md). This path will be suitable for customers who use telemetry data to bring their digital twins to life.
* [Ingest IoT data directly into your ADX cluster from IoT Hub](/azure/data-explorer/ingest-data-iot-hub) or from other sources. Then, the Azure Digital Twins graph will be used to contextualize the time series data using joint Azure Digital Twins/ADX queries. This path may be suitable for direct-ingestion workloads. 

### Mapping data across ADX and Azure Digital Twins

If you're ingesting time series data directly into ADX, you'll likely need to convert this raw time series data into a schema suitable for joint Azure Digital Twins/ADX queries.

An [update policy](/azure/data-explorer/kusto/management/updatepolicy.md) in ADX allows you to automatically transform and append data to a target table whenever new data is inserted into a source table. 

You can use an update policy to enrich your raw time series data with the corresponding **twin ID** from Azure Digital Twins, and persist it to a target table. Using the twin ID, the target table can then be joined against the digital twins selected by the Azure Digital Twins plugin. 

For example, say you created the following table to hold the raw time series data flowing into your ADX instance. 

```kusto
.createmerge table rawData (Timestamp:datetime, someId:string, Value:string, ValueType:string)  
```

You could create a mapping table to relate time series IDs with twin IDs, and other optional fields. 

```kusto
.createmerge table mappingTable (someId:string, twinId:string, otherMetadata:string) 
```

Then, create a target table to hold the enriched time series data. 

```kusto
.createmerge table timeseriesSilver (twinId:string, Timestamp:datetime, someId:string, otherMetadata:string, ValueNumeric:real, ValueString:string)  
```

Next, create a function `Update_rawData` to enrich the raw data by joining it with the mapping table. This will add the twin ID to the resulting target table. 

```kusto
.createoralter function with (folder = "Update", skipvalidation = "true") Update_rawData() { 
rawData 
| join kind=leftouter mappingTable on someId 
| project 
    Timestamp, ValueNumeric = toreal(Value), ValueString = Value, ... 
} 
```

Lastly, create an update policy to call the function and update the target table. 

```kusto
.alter table timeseriesSilver policy update 
@'[{"IsEnabled": true, "Source": "rawData", "Query": "Update_rawData()", "IsTransactional": false, "PropagateIngestionProperties": false}]' 
```

Once the target table is created, you can use the Azure Digital Twins plugin to select twins of interest and then join them against time series data in the target table. 

### Example schema

Here is an example of a schema that might be used to represent shared data.

| timestamp | twinId | modelId | name | value | relationshipTarget | relationshipID |
| --- | --- | --- | --- | --- | --- | --- |
| 2021-02-01 17:24 | ConfRoomTempSensor | dtmi:com:example:TemperatureSensor;1 | temperature | 301.0 |  |  |

Digital twin properties are stored as key-value pairs (`name, value`). `name` and `value` are stored as dynamic data types. 

The schema also supports storing properties for relationships, per the `relationshipTarget` and `relationshipID` fields. The key-value schema avoids the need to create a column for each twin property.

### Representing properties with multiple fields 

You may want to store a property in your schema with multiple fields. These properties are represented by storing a JSON object as `value` in your schema.

For instance, if you want to represent a property with three fields for roll, pitch, and yaw, the value object would look like this: `{"roll": 20, "pitch": 15, "yaw": 45}`.

## Next steps

* View the plugin documentation for the Kusto language in ADX: [azure_digital_twins_query_request plugin](/azure/data-explorer/kusto/query/azure-digital-twins-query-request-plugin)

* View sample queries using the plugin, including a walkthrough that runs the queries in an example scenario: [Azure Digital Twins query plugin for ADX: Sample queries and walkthrough](https://github.com/Azure-Samples/azure-digital-twins-getting-started/tree/main/adt-adx-queries) 

* Read about another strategy for analyzing historical data in Azure Digital Twins: [How-to: Integrate with Azure Time Series Insights](how-to-integrate-time-series-insights.md)
