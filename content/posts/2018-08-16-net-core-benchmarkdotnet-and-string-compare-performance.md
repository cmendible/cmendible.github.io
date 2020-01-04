---
author: Carlos Mendible
categories:
- dotNetCore
crosspost_to_medium: true
date: "2018-08-16T11:27:39Z"
description: .NET Core, BenchmarkDotNet and string compare performance
image: /wp-content/uploads/2017/07/dotnetcore.png
published: true
# tags: benchmarkdotnet string
title: .NET Core, BenchmarkDotNet and string compare performance
---

You have to choose between using **string.compare** or **==** to compare strings. How would you know which method performs faster?

Today I'll show you how to use [BenchmarkDotNet](https://benchmarkdotnet.org) with .Net Core to answer that question.

Let's start:

## 1. Create a folder for your new project
---
Open a command prompt an run:

``` shell
mkdir benchmark
```

## 2. Create the project
---

``` shell
cd benchmark
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
            var summary = BenchmarkRunner.Run<StringCompareVsEquals>();
        }
    }

    // We are using .Net Core so add we are adding the CoreJobAttribute here.
    [CoreJob(baseline: true)]
    [RPlotExporter, RankColumn]
    public class StringCompareVsEquals
    {
        private static Random random = new Random();
        private string s1;
        private string s2;

        public static string RandomString(int length)
        {
            const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
            return new string(Enumerable.Repeat(chars, length)
                .Select(s => s[random.Next(s.Length)]).ToArray());
        }

        // We wil run the the test for 2 diff string lengths: 10 & 100
        [Params(10, 100)]
        public int N;


        // Create two random strings for each set of params
        [GlobalSetup]
        public void Setup()
        {
            s1 = RandomString(N);
            s2 = RandomString(N);
        }

        // This is the slow way of comparing strings, so let's benchmark it.
        [Benchmark]
        public bool Slow() => s1.ToLower() == s2.ToLower();

        // This is the fast way of comparing strings, so let's benchmark it.
        [Benchmark]
        public bool Fast() => string.Compare(s1, s2, true) == 0;
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

 Method |   N |     Mean |     Error |    StdDev |   Median | Scaled | Rank |
------- |---- |---------:|----------:|----------:|---------:|-------:|-----:|
   Slow |  10 | 229.6 ns | 11.590 ns | 32.500 ns | 240.8 ns |   1.00 |    1 |
        |     |          |           |           |          |        |      |
   Fast |  10 | 195.2 ns | 12.870 ns | 37.948 ns | 205.5 ns |   1.00 |    1 |
        |     |          |           |           |          |        |      |
   Slow | 100 | 531.9 ns | 10.320 ns | 14.467 ns | 534.7 ns |   1.00 |    1 |
        |     |          |           |           |          |        |      |
   Fast | 100 | 148.2 ns |  3.132 ns |  7.683 ns | 145.1 ns |   1.00 |    1 |
```

Now you know what method is faster when comparing strings!

Please find all the code used in this post [here](https://github.com/cmendible/dotnetcore.samples/tree/master/benchmarkdotnet).

Hope it helps!