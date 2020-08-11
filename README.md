# PowerBI-Name Based Incremental Refresh
**Demonstration of how to use incremental refresh capabilities in Power BI to exclude individual files and tables from processing.**

The goal of this exercise is to exclude individual files from being processed by a dataset refresh in the Power BI service.  There may be times where you have an "append only" table (based on a directory of files) included in your Power BI model, and you'd like to avoid re-processing the static history each time your data is refreshed.  This is possible today in the Power BI Desktop application via "Include in report refresh" property assigned in Power Query (see image below), but it is not currently supported once a report is published.  To work around this limitation, it is possible to use incremental refresh to partition your table by file name, and achieve greater control over which data is included in a refresh.

![](/Images/IncludeInRefreshMenu.jpg)

In order to implement this strategy:
1. enable incremental refresh, and use properties to control how data is partitioned and when it is refreshed
1. create a function in Power Query that will accept parameters for start and end date and filter file list based on date range
1. invoke that function to create your data table, and leverage RangeStart and RangeEnd parameters for start and end dates
1. be sure to define empty table structure and add "try...otherwise..." block to M query to avoid empty dataset errors when no files are processed

**Definition of Function:**
The function below will query a storage container in Azure, and return a list of files matching the pattern "_YYYYMM.csv".  Note the variable Schema is used to define an empty table that matches the schema of the data table.  This empty table is returned when an error is thrown because no files exist that match the desired pattern.

![](/Images/FunctionDefinition.jpg)

```javascript
{
(StartDate as datetime, EndDate as datetime) =>
let
    start = Number.From (Date.From(StartDate)),
    end = Number.From(Date.From(EndDate))-1,
    //strDates = List.Distinct( List.Transform( {start..end}, each Text.Combine({ Text.From( Date.Year(Date.From(_))) , Text.End(Text.From( Date.Month(Date.From(_)) + 100), 2), Text.End( Text.From( Date.Day(Date.From(_)) + 100), 2) , ".csv" }))),
    strDates = List.Distinct( List.Transform( {start..end}, each Text.Combine({ Text.From( Date.Year(Date.From(_))) , Text.End(Text.From( Date.Month(Date.From(_)) + 100), 2) , ".csv" }))),
    Source = AzureStorage.Blobs("https://[].blob.core.windows.net/sampledata"),
    #"Filtered Rows" = Table.SelectRows(Source, each List.Contains( List.Transform( strDates, (x)=> Text.Contains( [Name],x) ), true)),
    Path = List.First( #"Filtered Rows"[Folder Path]),
    FileName = List.First( #"Filtered Rows"[Name]),
    #"Source File" = #"Filtered Rows"{[#"Folder Path"=Path,Name=FileName]}[Content],   
    #"Imported CSV" = Csv.Document(#"Source File",[Delimiter=",", Columns=16, Encoding=1252, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(#"Imported CSV", [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",{{"Region", type text}, {"Country", type text}, {"Item Type", type text}, {"Sales Channel", type text}, {"Order Priority", type text}, {"Order Date", type date}, {"Order ID", Int64.Type}, {"Ship Date", type date}, {"Units Sold", Int64.Type}, {"Unit Price", type number}, {"Unit Cost", type number}, {"Total Revenue", type number}, {"Total Cost", type number}, {"Total Profit", type number}, {"OrderDateKey", Int64.Type}, {"", type text}}),
    Schema = #table( type table [Region = text, Country = text, Item Type = text, Sales Channel = text, Order Priority = text, Order Date = date, Order Id = number, Ship Date = date, Units Sold = number, Unit Price = number, Unit Cost = number, Total Revenue = number, Total Cost = number, Total Profit = number, OrderDateKey = number, Column1 = text], {})
in
   try #"Changed Type" otherwise Schema

}
```

**Invocation of Function**
![](/Images/DataTableDefinition.jpg)

*Incremental Refresh Properties

![](/Images/IncrementalRefreshProperties.jpg)

After testing the refresh locally in Power BI Desktop, it's time to publish the report to the service, and process the dataset twice.  The first refresh will create and process all partitions of your table.  Subsequent refresh will only process the most recent partition based on the incremental refresh rules that you have defined.  In this example, I have created 2 source files as 'Sales_202006.csv' and 'Sales_202007.csv', and my incremental refresh was configured to only process last 1 day of data.  Since I am running this test on 20200811, I expect both files to be ignored on the 2nd refresh resulting in much faster load time without any data loss.  There are 2 easy ways to prove that things are working as expected.  The first is to look at the refresh history in the Power BI service to confirm that both refreshes did indeed completely successfully, and the 2nd ran much faster than the first.  The second test involves using SSMS to browse the partitions created for this dataset, and confirm that the data is stored in separate partitions (based on file name), and only the most recent partition was processed by the 2nd refresh (Note: this test requires Premium capacity which enables XMLA endpoints).

**Refresh History**
You'll see that the first time the dataset is processed in the Power BI service, all partitions are always processed, so you should expect the runtime to be similar to what you'd see if you did not use incremental refresh at all.  Subsequent Refreshes will occur much faster (~4 seconds in my case) since they will not find any file names matching the specified pattern, and will not load any new data (Note: original data will not be deleted as long as it does not fall outside of the range specified in incremental refresh properties.

![](/Images/RefreshHistory.jpg)<p>
![](/Images/Partitions.jpg)]

And that's all there is to it!  Going forward, scheduled refreshes will no longer need to continue re-processing the same static data, but as long as the naming conventions are kept consistent, new changes will always be picked up automatically.