---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2018-08-22T16:12:00Z"
description: '.NET Core, BenchmarkDotNet: for vs foreach performance'
images: ["/wp-content/uploads/2017/07/dotnetcore.png"]
published: true
tags: ["benchmarkdotnet", "for", "foreach"]
title: '.NET Core, BenchmarkDotNet: for vs foreach performance'
url: /2018/08/22/net-core-benchmarkdotnet-for-vs-foreach-performance/
---

So what is faster: looping through a **List<>** with **for** or with **foreach**?

Today I'll show you how to use [BenchmarkDotNet](https://benchmarkdotnet.org) with .Net Core to answer that question.

Let's start:

## 1. Create a folder for your new project
---
Open a command prompt an run:

``` shell
mkdir benchmark.for
```

## 2. Create the project
---

``` shell
cd benchmark.for
dotnet new console
```

## 3. Add the references to BenchmarkDotNet
---

``` shell
dotnet add package BenchmarkDotNet -v 0.11.0
dotnet restore
```

## 4. Replace the contents of Program.cs with the following code
---

``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

namespace benchmark
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // Use BenchmarkRunner.Run to Benchmark your code
            var summary = BenchmarkRunner.Run<ForEachVsFor>();
        }
    }

    // We are using .Net Core we are adding the CoreJobAttribute here.
    [CoreJob(baseline: true)]
    [RPlotExporter, RankColumn]
    public class ForEachVsFor
    {
        private static Random random = new Random();
        private List<int> list;

        public static List<int> RandomIntList(int length)
        {
            int Min = 1;
            int Max = 10;
            return Enumerable
                .Repeat(0, length)
                .Select(i => random.Next(Min, Max))
                .ToList();
        }

        // We wil run the the test for 3 diff list sizes
        [Params(10, 100, 1000)]
        public int N;

        [GlobalSetup]
        public void Setup()
        {
            list = RandomIntList(N);
        }

        // Foreach is ~2 times slower than for
        [Benchmark]
        public void Foreach()
        {
            int total = 0;
            foreach (int i in list)
            {
                total += i;
            }
        }

        // For is ~2 times faster than foreach
        [Benchmark]
        public void For()
        {
            int total = 0;
            for (int i = 0; i < list.Count; i++)
            {
                total += list[i];
            }
        }
    }
}
```

## 5. Run the application and interpret the results
---
Run the following command:

``` powershell
dotnet run -c release
```

after a while you should expect an output like the following:

``` shell
// * Summary *

BenchmarkDotNet=v0.11.0, OS=Windows 10.0.17134.228 (1803/April2018Update/Redstone4)
Intel Core i5-6300U CPU 2.40GHz (Skylake), 1 CPU, 4 logical and 2 physical cores
Frequency=2437501 Hz, Resolution=410.2562 ns, Timer=TSC
.NET Core SDK=2.1.302
  [Host] : .NET Core 2.1.2 (CoreCLR 4.6.26628.05, CoreFX 4.6.26629.01), 64bit RyuJIT
  Core   : .NET Core 2.1.2 (CoreCLR 4.6.26628.05, CoreFX 4.6.26629.01), 64bit RyuJIT

Job=Core  Runtime=Core

  Method |    N |        Mean |      Error |      StdDev |      Median | Scaled | Rank |
-------- |----- |------------:|-----------:|------------:|------------:|-------:|-----:|
 Foreach |   10 |    45.76 ns |  3.0062 ns |   8.8167 ns |    43.15 ns |   1.00 |    1 |
         |      |             |            |             |             |        |      |
     For |   10 |    18.14 ns |  0.4040 ns |   0.3968 ns |    18.04 ns |   1.00 |    1 |
         |      |             |            |             |             |        |      |
 Foreach |  100 |   503.16 ns |  9.9973 ns |  24.8968 ns |   501.49 ns |   1.00 |    1 |
         |      |             |            |             |             |        |      |
     For |  100 |   149.56 ns |  3.6317 ns |  10.1238 ns |   147.41 ns |   1.00 |    1 |
         |      |             |            |             |             |        |      |
 Foreach | 1000 | 2,611.51 ns | 51.7677 ns | 122.0225 ns | 2,596.29 ns |   1.00 |    1 |
         |      |             |            |             |             |        |      |
     For | 1000 | 1,406.77 ns | 28.3726 ns |  82.3140 ns | 1,391.04 ns |   1.00 |    1 |
```

So as you can see **for** performs much better than **foreach** when looping through a **List<>**!!!

Try changing the **int** type for a class and then working with it inside the loop and check the results. You'll know what to do next time you need your application to run fast!

Please find all the code used in this post [here](https://github.com/cmendible/dotnetcore.samples/tree/main/benchmarkdotnet.for).

Hope it helps!