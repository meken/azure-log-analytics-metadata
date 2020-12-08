# Accessing table metadata with Log Analytics

[Azure Log Analytics](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/data-platform-logs) is a tool in the Azure portal to edit and run log queries from data collected by Azure Monitor Logs and interactively analyze their results. These logs are typically stored in a tabular format and queried through a language called [Kusto Query Language (KQL)](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/).

Although the Azure portal provides the schema information in a visual way, sometimes you'd like to have programmatic access to this information. KQL provides schema management operations to get metadata information through the [`.show table schema` command](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/show-table-schema-command), however, this and other KQL management commands are not available in Log Analytics context. In this document we'll explore the alternative options.

Before we start exploring various options, make sure that you've captured the Log Analytics Workspace id for your experiments. In the examples below we'll be using the `ADFActivityRun` table which would only be present if you've configured an Azure Data Factory to send its diagnostics to the workspace. You can replace that with any other table you'd like (for example the more general `AzureMetrics`). In this document an example will be provided to get the list of available tables within a workspace too.

```bash
WORKSPACE_ID="..."  # Log Analytics workspace id
```

## Naive approach

The first option that you could try is the Azure CLI command [`az monitor log-analytics`](https://docs.microsoft.com/en-us/cli/azure/ext/log-analytics/monitor/log-analytics?view=azure-cli-latest) (which is at the moment of this writing in preview), that runs a Log Analytics query.

```bash
$ az monitor log-analytics query -w $WORKSPACE_ID \
    --analytics-query "ADFActivityRun | limit 1" \
    --query "keys([0])" -o tsv
TableName
TenantId
SourceSystem
TimeGenerated
ResourceId
OperationName
...
```

This is a very simple method that would provide at least the column names for a table. However, the column types are not available and it wouldn't work if you had an empty table either.

## REST to rescue

All Azure services have well documented REST APIs, this applies to [Log Analytics Workspaces](https://docs.microsoft.com/en-us/rest/api/loganalytics/dataaccess/metadata/get) too. We could just run `curl` (or a similar tool) to get the relevant information. But, we'd then have to deal with the authentication and all, luckily Azure CLI provides the [rest](https://docs.microsoft.com/en-us/cli/azure/reference-index?view=azure-cli-latest#az_rest) command which uses the same identity as the CLI user to make the web request, simplifying things.

```bash
$ az rest -u https://api.loganalytics.io/v1/workspaces/$WORKSPACE_ID/metadata \
    --query "tables[?name=='ADFActivityRun'].columns[].[name, type]" -o tsv
TenantId        string
SourceSystem    string
TimeGenerated   datetime
ResourceId      string
OperationName   string
...
```

### And the tables?

If you don't know the name of the table that you're looking for, you could also retrieve that information through this same method.

```bash
$ az rest -u https://api.loganalytics.io/v1/workspaces/$WORKSPACE_ID/metadata \
    --query "tables[].name" -o tsv
AADDomainServicesAccountLogon
AADDomainServicesAccountManagement
AADDomainServicesDirectoryServiceAccess
AADDomainServicesLogonLogoff
...
```
