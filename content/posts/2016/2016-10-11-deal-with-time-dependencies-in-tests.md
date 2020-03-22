---
author: Carlos Mendible
categories:
- dotnet
date: "2016-10-11T07:37:35Z"
description: Deal with time dependencies in Tests
image: /wp-content/uploads/2016/10/SystemDateTime.jpg
tags: ["DateTime", "UnitTests", "xUnit"]
title: Deal with time dependencies in Tests
---
Las week we discovered that some of our test would"randomly" fail depending of the time of the day. After investigating the issue we found that the culprit was that the service being tested was taking decisions based on the current system time (DateTime.Now) leading to different outcomes through the day. So how **do we deal with time dependencies in tests**?

My first thought was creating an IClock interface and injecting an implementation to the service. It seemed like an overkill and then I googled enough to find a short and very simple solution in Ayende's 2008 article: <a href="https://ayende.com/blog/3408/dealing-with-time-in-tests" target="_blank">Dealing with time in tests</a>

I'll use .NET Core and xUnit to illustrate the problem and the proposed solution.

Now let's start:

## 1. Create a test project
---
Open a command prompt and run the following commands 
    
``` powershell
mkdir remove.datetime.dependency.test
cd remove.datetime.dependency.test
dotnet new -t xunittest
```
## 2. Create a SystemDateTime helper class
---
This is the helper class a proposed by Ayende
    
``` csharp
namespace MyServices
{
    using System;

    /// <summary>
    /// Helper class that will return DateTime.Now, but can be changed to deal with tests.
    /// https://ayende.com/blog/3408/dealing-with-time-in-tests
    /// </summary>
    public static class SystemDateTime
    {
        
        /// <summary>
        /// Gets a function that when evaluated returns a local date and time.   
        /// </summary>
        /// <returns>A function that when evaluated returns a local date and time.</returns>
        public static Func<DateTime> Now = () => DateTime.Now;
    }
}
```

## 3. Create a Service with a dependency on time
The service will have two methods: one will use System.DateTime.Now and one will use the helper created on step 2
    
``` csharp
namespace MyServices
{
    using System;

    /// <summary>
    /// A time dependent on DateTime.Now 
    /// </summary>
    public class TimeDependentService
    {

        /// <summary>
        /// Tells if it's morning.
        /// </summary>
        /// <returns>true if it's before 12</returns>
        public bool OldIsMorning()
        {
            return DateTime.Now.Hour < 12;
        }

        /// <summary>
        /// Tells if it's morning using the helper by Ayende. 
        /// </summary>
        /// <returns>true if it's before 12</returns>
        public bool IsMorning()
        {
            return SystemDateTime.Now().Hour < 12;
        }
    }
}
```

## 4. Create the tests
---
Replace the content of Tests.cs with the following code 
    
``` csharp
namespace Tests
{
    using System;
    using MyServices;
    using Xunit;

    public class Tests
    {
        /// <summary>
        /// If this test is run after 12PM it will fail cause of the dependency on the system time.
        /// </summary>
        [Fact]
        public void Will_Fail_After_Noon()
        {
            var svc = new TimeDependentService();
            Assert.True(svc.OldIsMorning());
        }

        /// <summary>
        /// This test will run OK no matter the systems time, cause we are using the Ayende 
        /// approach to tests with time dependencies.
        /// </summary>
        [Fact]
        public void Will_Run_OK_No_Matter_The_Real_Time()
        {
            // With Ayende's helper we can for the time to 11 AM
            SystemDateTime.Now = () => new DateTime(2016, 10, 10, 11, 0, 0);
            var svc = new TimeDependentService();
            Assert.True(svc.IsMorning());
        }
    }
}
```

## 5. Run the tests
---  

``` powershell
dotnet test
```
    
No matter the system time the test: **Will_Run_OK_No_Matter_The_Real_Time** will always pass, while the **Will_Fail_After_Noon** test will fail any time after 12 PM.

You can get the code here: <https://github.com/cmendible/dotnetcore.samples/tree/master/remove.datetime.dependency.test>

Hope it helps!