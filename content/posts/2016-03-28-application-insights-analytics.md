---
author: Carlos Mendible
categories:
- Azure
- DevOps
date: "2016-03-28T12:01:42Z"
description: Application Insights Analytics
image: /wp-content/uploads/2016/03/query-1.png
# tags: ["ApplicationInsights"]
title: Application Insights Analytics
---
For the last couple of months I've been using **<a href="https://azure.microsoft.com/en-us/documentation/articles/app-insights-overview/" target="_blank">Application Insights</a>** to monitor, detect and diagnose performance issues in our applications. Last week I noticed that the _Analytics_ button on the <a href="https://azure.microsoft.com/en-us/documentation/articles/app-insights-overview/" target="_blank">Application Insights</a> blade was working:

<a href="/wp-content/uploads/2016/03/Analytics.png" rel="attachment wp-att-2421"><img class="size-medium wp-image-2421 aligncenter" src="wp-content/uploads/2016/03/Analytics-300x67.png" alt="Analytics" width="300" height="67" srcset="/wp-content/uploads/2016/03/Analytics-300x67.png 300w, /wp-content/uploads/2016/03/Analytics-250x56.png 250w, /wp-content/uploads/2016/03/Analytics.png 586w" sizes="(max-width: 300px) 100vw, 300px" /></a>

Once you click it, your browser will open the **Application Insights Analytics** page with the following taxonomy:

<a href="http://carlos.mendible.com/wp-content/uploads/2016/03/Analytics_Page.png" rel="attachment wp-att-2441"><img class="size-medium wp-image-2441 aligncenter" src="http://carlos.mendible.com/wp-content/uploads/2016/03/Analytics_Page-300x217.png" alt="Analytics Page" width="300" height="217" srcset="/wp-content/uploads/2016/03/Analytics_Page-300x217.png 300w, /wp-content/uploads/2016/03/Analytics_Page-768x555.png 768w, /wp-content/uploads/2016/03/Analytics_Page-250x181.png 250w, /wp-content/uploads/2016/03/Analytics_Page.png 921w" sizes="(max-width: 300px) 100vw, 300px" /></a>

  * A left panel (named SCHEMA) which shows all the tables you can use to query the telemetry from your applications.
  * A top panel where you must write your queries.
  * And a bottom panel to show the query results or charts.

<p style="text-align: left;">
  Yes you read it right: **You can query your telemetry!**
</p>

To write your queries you must use the <a href="https://azure.microsoft.com/en-us/documentation/articles/app-analytics-queries" target="_blank">Insights Query Language</a> and start with a source table followed by a series of operators separated by |.

The following example is a simple query that will render a pie chart with the average _Requests/Sec_ served by role instances (servers):

``` sql
    metrics 
    | where name == "Requests/Sec"
    | project value, cloud_RoleInstance 
    | summarize avg(value) by cloud_RoleInstance
    | render piechart
``` 

  * Line 1 specifies the _metrics_ table as the source of the query.
  * Line 2 uses the _where_ operator to filter the table.
  * Line 3 selects the columns to include using the _project_ operator.
  * Line 4 calculates the _average_ of the metric by _cloud_RoleInstance_ (server)
  * Line 5 renders the results as a pie chart.

You must be thinking: do I have to learn another language? The short answer is yes! But it's nothing you won't be able to handle and once you get used to it, you will unleash the full power of the queries and gain a better understanding of your applications. For instances now we can answer the following questions:

  * What method of our application throws more exceptions?
  * What exceptions are related to a given user?
  * What pages are being used and which ones are not?

To learn much more you can visit the <a href="https://azure.microsoft.com/en-us/documentation/articles/app-analytics-queries" target="_blank">Queries in Analytics</a> page which is a great resource and is well organized.

Hope it helps.