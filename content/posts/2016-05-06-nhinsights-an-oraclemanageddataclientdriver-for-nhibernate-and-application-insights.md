---
author: Carlos Mendible
categories:
- Azure
- dotNet
date: "2016-05-06T07:18:05Z"
description: NHInsights an OracleManagedDataClientDriver for NHibernate and Application
  Insights
image: /wp-content/uploads/2016/05/download.png
# tags: ApplicationInsights NHibernate Oracle OracleManagedDataClientDriver Telemetry
title: NHInsights an OracleManagedDataClientDriver for NHibernate and Application
  Insights
---
Let's talk about **<a href="https://github.com/cmendible/NHInsights" target="_blank">NHInsights</a> an OracleManagedDataClientDriver for NHibernate and Application Insights**

If you use Oracle and NHibernate and you are trying to use Application Insights to diagnose issues with your database calls you will notice that for ASP.NET applications <a href="https://azure.microsoft.com/en-us/documentation/articles/app-insights-dependencies/" target="_blank">the out-of-the-box dependency monitor currently reports calls to these types of dependencies</a>:

  * SQL databases
  * ASP.NET web and WCF services that use HTTP-based bindings
  * Local or remote HTTP calls
  * Azure DocumentDb, table, blob storage, and queue

So how can you track your queries as a dependency?

Well I checked the <a href="https://azure.microsoft.com/en-us/documentation/articles/app-insights-api-custom-events-metrics/#track-dependency" target="_blank">Track Dependency API</a> which is really simple and also remembered that some time ago I had used <a href="https://github.com/MiniProfiler/dotnet" target="_blank">MiniProfiler</a> together with NHibernate and managed to track SQL Server response times.

I downloaded <a href="https://github.com/MiniProfiler/dotnet" target="_blank">MiniProfiler</a> source code and learned how they implemented their <a href="https://github.com/MiniProfiler/dotnet/blob/master/StackExchange.Profiling/Data/ProfiledDbCommand.cs" target="_blank">ProfiledDbCommand</a> and also the <a href="https://github.com/MRCollective/MiniProfiler.NHibernate/blob/master/MiniProfiler.NHibernate/Drivers/MiniProfilerSql2008ClientDriver.cs" target="_blank">MiniProfilerSql2008ClientDriver</a>.

Based on those two great pieces of code and the original <a href="https://github.com/nhibernate/nhibernate-core/blob/master/src/NHibernate/Driver/OracleManagedDataClientDriver.cs" target="_blank">OracleManagedDataClientDriver</a> code I managed to create <a href="https://github.com/cmendible/NHInsights" target="_blank">NHInsights</a>.

What's the secret? to create a driver capable of returning a profiled DbCommand (i.e. <a href="https://github.com/cmendible/NHInsights/blob/master/src/NHInsights/Infrastructure/InsightsDbCommand.cs" target="_blank">InsightsDbCommand</a>):

``` csharp
public class OracleManagedDataClientDriver : ReflectionBasedDriver, IEmbeddedBatcherFactoryProvider
{
    ...

    public override IDbCommand CreateCommand()
    {
       var command = base.CreateCommand();

       command = new **InsightsDbCommand**((DbCommand)command, (DbConnection)command.Connection) as IDbCommand;

       return command;
    }
}
```

Go ahead and try it! To start sending your queries as dependencies just install the nuget package:

``` powershell
Install-Package NHInsights
```

And then reference the NHInsights.OracleManagedDataClientDriver in your NHibernate configuration:

``` xml
<hibernate-configuration xmlns="urn:nhibernate-configuration-2.2">
  <session-factory>
    ...
    <property name="connection.driver_class">NHInsights.OracleManagedDataClientDriver, NHInsights, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null</property>
    ...
  </session-factory>
</hibernate-configuration>
```

Hope it helps!