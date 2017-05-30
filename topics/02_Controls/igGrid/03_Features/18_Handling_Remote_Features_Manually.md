﻿<!--
|metadata|
{
	"fileName": "handling-remote-features-manually",
	"controlName": "igGrid",
	"tags": ["Grids","How to"]
}
|metadata|
-->

# Handling Remote Features Manually (igGrid)

## In this topic

- [Overview](#overview)
- [Handling Remote Paging](#paging)
- [Handling Remote Filtering](#filtering)
- [Handling Remote Sorting/GroupBy](#sorting)
- [Handling Remote Summaries](#summaries)
- [Related Content](#related-content)

## <a id="overview"></a> Overview

Remote features in the `igGrid` require a backend implementation for handling their specific remote requests and returning the resulting data.
The following features have remote capabilities:
  - Paging
  - Filtering
  - Sorting
  - GroupBy 
  - Summaries

They can be enabled via the related **type** option of the feature. For example:

```js
features: [
	{ name: "Paging", type: "remote" },
	{ name: "Sorting", type: "remote" },
	{ name: "Filtering", type: "remote" },
     ...
]
```

When you're using the Grid ASP.NET MVC wrapper the remote requests initiated by these features can be processed out of the box by adding to the related Action the `GridDataSourceActionAttribute`.
This is an action filter attribute that you can use to decorate the MVC Action that returns your grid data.
For example:

```csharp
[GridDataSourceAction]
public ActionResult GetGridData()
{
     ...
  return View(quryableData);
}
```

It handles incoming requests by the various remote grid features and returns the processed JSON data back to grid.

> **Note:** Use `GridDataSourceAction` attribute when you configure grid in the View. If you're using `GridModel` class and configure the grid in the Controller you should use the `GridModel.GetData` instance method to return data back to the browser. 

We recommend you to use the above methods in order to take full advantage of their remote capabilities with the least amount of effort.

However in some cases you may not have access to the MVC wrappers (for example if you're using Ignite UI in an ASP.NET project) or you may want to build a custom logic for handling those requests.
This topic will guide you through the process manually handling `igGrid` features remote requests and sending back a response in a JSON format that they can understand.

## <a id="paging"></a> Handling Remote Paging

This section will guide you through the process of configuring and handling remote Paging.

Steps:

1. Client-side configuration
2. Server-side configuration and processing of the request.

### Client-side configuration

The following table lists the options that affect the remote Paging and that should be set when you're manually handling the response from your backend.

Options| Description | Required |
-------|-------------|---------|
[responseDataKey](%%jQueryApiUrl%%/ui.igGrid#options:responseDataKey) | The property in the responses where data records are held.| Yes.
[dataSource](%%jQueryApiUrl%%/ui.igGrid#options:dataSource)| Can be any valid data source accepted by `$.ig.DataSource`, or an instance of an `$.ig.DataSource` itself. | Yes. Should be set to the url for the backend that will handle the remote requests.
[recordCountKey](%%jQueryApiUrl%%/ui.iggridpaging#options:recordCountKey) | The property in the response data, when using remote data source, that will hold the total number of records in the data source.| Yes
[type](%%jQueryApiUrl%%/ui.iggridpaging#options:type) | Sets the type of Paging. | Yes. Should be set to "remote" to enable remote Paging.
[pageSizeUrlKey](%%jQueryApiUrl%%/ui.iggridpaging#options:pageSizeUrlKey) | The name of the encoded URL parameter that will hold the currently requested page size. | No.  If not set OData conventions are used by default.
[pageIndexUrlKey](%%jQueryApiUrl%%/ui.iggridpaging#options:pageIndexUrlKey) | The name of the encoded URL parameter that will hold the currently requested page index.| No.  If not set OData conventions are used by default.

Example configuration:

```js
responseDataKey: "Records",
dataSource: "http://<server>/grid/GetData",
features: [
{ name: "Paging", type: "remote", recordCountKey: "TotalRecordsCount" }
]
```

The request generated by a paging request with the example setup would be:

```js
http://<server>/grid/GetData?$skip=0&$top=25
```

### Server-side configuration

The backend to which the request is send should do the following:

1. Read the query string parameters that contain the paging information (`$skip` and `$top`)
2. Process the data of the grid to get only the data for the current page based on the parameters.
3. Return JSON data that contains the processed grid data and the total records count of the grid, for example:

```js
{Records: <data>, TotalRecordsCount: totalCountOfAllRecords}
```
Note that the property holding  the data should match the `responseDataKey` option defined on the client-side and the property holding the total count should match the `recordCountKey` option of the Paging feature.

The below example shows how to apply the above steps in a ASP.NET MVC application:

**In C#:**

```csharp
public JsonResult GetData() {
			IEnumerable<Order> orders = RepositoryFactory.GetOrderRepository().Get().Take(200);
			IQueryable data = orders.AsQueryable();
      int totalCount = data.Count();
      if (Request.QueryString["$top"] != null && Request.QueryString["$skip"] != null) {
				data = ApplyPaging(Request.QueryString, data);
			}

      JsonResult result = new JsonResult();
			result.JsonRequestBehavior = JsonRequestBehavior.AllowGet;
			result.Data = new { Records = data, TotalRecordsCount = totalCount };
			return result;
}
private IQueryable ApplyPaging(NameValueCollection queryString, IQueryable data)
{
			int recCount = Convert.ToInt32(queryString["$top"]);
			int startIndex = Convert.ToInt32(queryString["$skip"]);

			data = data.Skip(startIndex).Take(recCount);
			return data;
}
```

## <a id="filtering"></a> Handling Remote Filtering

This section will guide you through the process of configuring and handling remote Filtering.

Steps:

1. Client-side configuration
2. Server-side configuration and processing of the request.

### Client-side configuration

The following table lists options that affect the remote Filtering and that should be set when you're manually handling the response from your backend.

Options| Description | Required |
-------|-------------|---------|
[dataSource](%%jQueryApiUrl%%/ui.igGrid#options:dataSource)| Can be any valid data source accepted by `$.ig.DataSource`, or an instance of an `$.ig.DataSource` itself. | Yes. Should be set to the url for the backend that will handle the remote requests.
[type](%%jQueryApiUrl%%/ui.iggridfiltering#options:type) | Sets the type of Filtering. | Yes. Should be set to "remote" to enable remote Filtering.
[filterExprUrlKey](%%jQueryApiUrl%%/ui.iggridfiltering#options:filterExprUrlKey)| URL key name that specifies how the filtering expressions will be encoded for remote requests, e.g. `&filter('col') = startsWith`. Default is OData.| No. If not set OData conventions are used by default.

Example configuration:

```js
dataSource: "http://<server>/grid/GetData",
features: [
{ name: "Filtering", type: "remote", filterExprUrlKey: "filter" }
...
]
```

The request generated by a filtering request with the example setup would be, for example:

```js
http://<server>/grid/GetData?filter(OrderID)=equals(10273)
```

### Server-side configuration

The backend to which the request is send should do the following:

1. Read the query string parameters that contain the filtering information
2. Process the data of the grid and return the filtered data.

The below example shows how to apply the above steps in a ASP.NET MVC application:

**In C#:**

```csharp

public JsonResult GetData() {
			IEnumerable<Order> orders = RepositoryFactory.GetOrderRepository().Get().Take(200);
			IQueryable data = orders.AsQueryable();
      var filterExprs = Request.QueryString.AllKeys.Where(x => x.Contains("filter"));
			if (filterExprs.Count() != 0)
			{
				data = ApplyFilterExpr(Request.QueryString, data);
			}
      JsonResult result = new JsonResult();
			result.JsonRequestBehavior = JsonRequestBehavior.AllowGet;
			result.Data = data;
			return result;
}
private IQueryable ApplyFilterExpr(NameValueCollection queryString, IQueryable data)
{
			List<FilterExpression> exprs = GetFilterExpressions(queryString);
			StringBuilder builder = new StringBuilder();
			int count = 0;

			for (int i = 0; i < exprs.Count; i++)
			{
				if (count != 0 && count <= exprs.Count - 1)
				{
					builder.Append(exprs[i].Logic.ToLower() == "AND".ToLower() ? " AND " : " OR ");
				}
				count++;

				string condition = exprs[i].Condition;
				string expr = exprs[i].Expr;
				string colKey = exprs[i].Key;
				var dt = DateTime.Now;
				
				switch (condition.ToLower())
				{
					case "startswith":
						 builder.Append(colKey + ".StartsWith(\"" + expr + "\")");
						break;
					case "contains":
						 builder.Append(colKey + ".Contains(\"" + expr + "\")");
						break;
					case "endswith":
						 builder.Append(colKey + ".EndsWith(\"" + expr + "\")");
						break;
					case "equals":
						if (colKey == "ShipName") {
							//col type is string
							builder.Append(colKey + " == \"" + expr + "\"");
						}
						else
						{
							//col type is number
							builder.Append(colKey + " == " + expr);
						}
						
						break;
					case "doesnotequal":
						if (colKey == "ShipName")
						{
							//col type is string
							builder.Append(colKey + " != \"" + expr + "\"");
						}
						else
						{
							//col type is number
							builder.Append(colKey + " != " + expr);
						}
						break;
					case "doesnotcontain":
						 builder.Append("! " + colKey + ".Contains(\"" + expr + "\")");
						break;
					case "lessthan":
						 builder.Append(colKey + " < " + expr);
						break;
					case "greaterthan":
						 builder.Append(colKey + " > " + expr);
						break;
					case "lessthanorequalto":
						 builder.Append(colKey + " <= " + expr);
						break;
					case "greaterthanorequalto":
						 builder.Append(colKey + " >= " + expr);
						break;
					case "on":
						dt = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc).AddMilliseconds(double.Parse(expr)).ToUniversalTime();
						 builder.Append("(" + colKey + ".Value.Day == " + dt.Day + " AND " + colKey +
							".Value.Year == " + dt.Year + " AND " +colKey + ".Value.Month == " + dt.Month + ")");
						break;
					case "noton":
						dt = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc).AddMilliseconds(double.Parse(expr)).ToUniversalTime();
						 builder.Append("!("+colKey + ".Value.Day == " + dt.Day + " AND " + colKey +
							".Value.Year == " + dt.Year + " AND " + colKey + ".Value.Month == " + dt.Month + ")");
						break;
					case "after":
						dt = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc).AddMilliseconds(double.Parse(expr)).ToUniversalTime();
						 builder.Append("((" + colKey + ".Value.Year > " + dt.Year + " OR (" +
							colKey + ".Value.Month > " + dt.Month + " AND " + colKey + ".Value.Year == " + dt.Year + ") OR (" +
							colKey + ".Value.Day > " + dt.Day + " AND " + colKey + ".Value.Year == " + dt.Year + " AND " +
							colKey + ".Value.Month == " + dt.Month + ")))");
						break;
					case "before":
						dt = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc).AddMilliseconds(double.Parse(expr)).ToUniversalTime();
						builder.Append("((" + colKey + ".Value.Year < " + dt.Year + " OR (" +
									   colKey + ".Value.Month < " + dt.Month + " AND " + colKey + ".Value.Year == " + dt.Year + ") OR (" +
									   colKey + ".Value.Day < " + dt.Day + " AND " + colKey + ".Value.Year == " + dt.Year + " AND " +
									   colKey + ".Value.Month == " + dt.Month + ")))");
						break;
					case "today":
						builder.Append("(" + colKey + ".Value.Day == " + DateTime.Now.Day + " AND " + colKey +
						 ".Value.Year == " + DateTime.Now.Year + " AND " + colKey + ".Value.Month == " + DateTime.Now.Month + ")");
						break;
					case "yesterday":
						DateTime yesterday = DateTime.Now.AddDays(-1);
						builder.Append("(" + colKey + ".Value.Day == " + yesterday.Day + " AND " + colKey +
						 ".Value.Year == " + yesterday.Year + " AND " + colKey + ".Value.Month == " + yesterday.Month + ")");
						break;
					case "thismonth":
						builder.Append("(" + colKey + ".Value.Year == " + DateTime.Now.Year + " AND " + colKey + ".Value.Month == " + DateTime.Now.Month + ")");
						break;
					case "lastmonth":
						builder.Append("(" + colKey + ".Value.Year == " + (DateTime.Now.Year - 1) + " AND " + colKey + ".Value.Month == " + (DateTime.Now.Month - 1) + ")");
						break;
					case "nextmonth":
						builder.Append("(" + colKey + ".Value.Year == " + (DateTime.Now.Year - 1) + " AND " + colKey + ".Value.Month == " + (DateTime.Now.Month + 1) + ")");
						break;
					case "thisyear":
						builder.Append(colKey + ".Value.Year == " + DateTime.Now.Year);
						break;
					case "lastyear":
						builder.Append(colKey + ".Value.Year == " + (DateTime.Now.Year - 1));
						break;
					case "nextyear":
						builder.Append(colKey + ".Value.Year == " + (DateTime.Now.Year + 1));
						break;
					default:
						break;
				}
			}
			if (builder.Length > 0) {
				data = data.Where(builder.ToString(), new object[0]);
			}
		
			return data;
}

```

## <a id="sorting"></a> Handling Remote Soring/GroupBy

This section will guide you through the process of configuring and handling remote Sorting. The same steps apply for GroupBy as the only applied data operation for GroupBy is to sort the data.

Steps:

1. Client-side configuration
2. Server-side configuration and processing of the request.

### Client-side configuration

The following table lists options that affect the remote Sorting and that should be set when you're manually handling the response from your backend.

Options| Description | Required |
-------|-------------|---------|
[dataSource](%%jQueryApiUrl%%/ui.igGrid#options:dataSource)| Can be any valid data source accepted by `$.ig.DataSource`, or an instance of an `$.ig.DataSource` itself. | Yes. Should be set to the url for the backend that will handle the remote requests.
[type](%%jQueryApiUrl%%/ui.iggridsorting#options:type) | Sets the type of Sorting. | Yes. Should be set to "remote" to enable remote Sorting.
[sortUrlKey](%%jQueryApiUrl%%/ui.iggridsorting#options:sortUrlKey)| URL param name which specifies how sorting expressions will be encoded in the URL. Example: ?sort(col1)=asc.| No. If not set OData conventions are used by default.
[sortUrlKeyAscValue](%%jQueryApiUrl%%/ui.iggridsorting#options:sortUrlKeyAscValue) | URL param value for ascending type of Sorting. Example: ?sort(col1)=asc.| No. If not set OData conventions are used by default.
[sortUrlKeyDescValue](%%jQueryApiUrl%%/ui.iggridsorting#options:sortUrlKeyDescValue)| URL param value for descending type of Sorting. Example: ?sort(col1)=desc.| No. If not set OData conventions are used by default.


Example configuration:

```js
dataSource: "http://<server>/grid/GetData",
features: [
	{
		name: "Sorting",
		type: "remote",
		sortUrlKey: 'sort',
		sortUrlKeyAscValue: 'asc',
		sortUrlKeyDescValue: 'desc'
	}
...
]
```

The request generated by a sorting request with the example setup would be, for example:

```js
http://<server>/grid/GetData?sort(OrderID)=asc
```

### Server-side configuration

The backend to which the request is send should do the following:

1. Read the query string parameters that contain the sorting information
2. Process the data of the grid and return the sorted data.

The below example shows how to apply the above steps in a ASP.NET MVC application:

**In C#:**

```csharp
public JsonResult GetData() {
			IEnumerable<Order> orders = RepositoryFactory.GetOrderRepository().Get().Take(200);
			IQueryable data = orders.AsQueryable();
      var sortExprs = Request.QueryString.AllKeys.Where(x => x.Contains("sort"));
			if (sortExprs.Count() != 0) {
				data = ApplySorting(Request.QueryString, data);
			}
      JsonResult result = new JsonResult();
			result.JsonRequestBehavior = JsonRequestBehavior.AllowGet;
			result.Data = data;
			return result;
}
private IQueryable ApplySorting(NameValueCollection queryString, IQueryable data)
		{
			List<SortExpression> sortExpressions = BuildSortExpressions(queryString, "sort", false);

			string orderBy = "OrderBy";
			string orderByDescending = "OrderByDescending";
			foreach (SortExpression expr in sortExpressions)
			{

				data = ApplyOrder(data, expr.Key, expr.Mode == SortMode.Ascending ? orderBy : orderByDescending);
				orderBy = "ThenBy";
				orderByDescending = "ThenByDescending";
			}
			return data;
			
		}
public List<SortExpression> BuildSortExpressions(NameValueCollection queryString, string sortKey, bool isTable)
{
			List<SortExpression> expressions = new List<SortExpression>();
			List<string> sortKeys = new List<string>();
			foreach (string key in queryString.Keys)
			{
				if (!string.IsNullOrEmpty(key) && key.StartsWith(sortKey))
				{
					SortExpression e = new SortExpression();
					e.Key = key.Substring(key.IndexOf("(")).Replace("(", "").Replace(")", "");
					e.Logic = "AND";
					e.Mode = queryString[key].ToLower().StartsWith("asc") ? SortMode.Ascending : SortMode.Descending;
					expressions.Add(e);
					sortKeys.Add(key);
				}
			}
			if (sortKeys.Count > 0 && isTable)
			{
				foreach (string sortedKey in sortKeys)
				{
					queryString.Remove(sortedKey);
				}
				string url = Request.Url.AbsolutePath;
				string updatedQueryString = "?" + queryString.ToString();
				Response.Redirect(url + updatedQueryString);
			}
			return expressions;
}
```

## <a id="summaries"></a> Handling Remote Summaries

This section will guide you through the process of configuring and handling remote Summaries.

Steps:

1. Client-side configuration
2. Server-side configuration and processing of the request.

### Client-side configuration

The following table lists options that affect the remote Summaries and that should be set when you're manually handling the response from your backend.

Options| Description | Required |
-------|-------------|---------|
[responseDataKey](%%jQueryApiUrl%%/ui.igGrid#options:responseDataKey) | The property in the responses where data records are held.| Yes.
[dataSource](%%jQueryApiUrl%%/ui.igGrid#options:dataSource)| Can be any valid data source accepted by `$.ig.DataSource`, or an instance of an `$.ig.DataSource` itself. | Yes. Should be set to the url for the backend that will handle the remote requests.
[type](%%jQueryApiUrl%%/ui.iggridsummaries#options:type) | Sets the type of summaries. | Yes. Should be set to "remote" to enable remote Summaries.
[summariesResponseKey](%%jQueryApiUrl%%/ui.iggridsummaries#options:summariesResponseKey)| Result key by which we get data from the result returned by remote data source. | No. Default is "summaries".
[summaryExprUrlKey](%%jQueryApiUrl%%/ui.iggridsummaries#options:summaryExprUrlKey) | Set key in GET Request for summaries - used only when type is remote. | No. Default is "summaries".

Example configuration:

```js
dataSource: "http://<server>/grid/GetData",
responseDataKey: "Records",
features: [
	{ name: "Summaries", type:"remote"}
...
]
```

The request generated by a summaries request with the example setup would be, for example:

```js
http://<server>/grid/GetData?summaries(OrderID)=count,min,max,sum,avg
```

### Server-side configuration

The backend to which the request is send should do the following:

1. Read the query string parameters that contain the summary information
2. Process the data of the grid and to pupulate the summaries.
3. Return JSON data that contain the proccesed grid data and the calculated summaries for the grid, for example:

```
{
    Records: <data>,
    "Metadata": {
        "Summaries": {
            "OrderID": {
                "count": 100
            }
        }
    }
}
```

## <a id="related-content"></a> Related Content

The following topics provide additional information related to this topic.
-   [Filtering](igGrid-Filtering.html)
-   [Paging](igGrid-Paging.html)
-   [Column Summaries](igGrid-Column-Summaries.html)
-   [Sorting](igGrid-Sorting.html)
-   [Column Grouping](igGrid-GroupBy.html)