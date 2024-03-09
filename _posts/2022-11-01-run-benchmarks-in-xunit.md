---
title: Run BenchmarkDotNet in xUnit
author: valdis
date: 2022-11-01 10:00:00 +0200
categories: [.NET, C#, Performance, xUnit, Testing]
tags: [.net, c#, performance, xunit, testing]
---

If you ever need to run [BenchmarkDotNet](https://benchmarkdotnet.org/articles/overview.html) as part of the unit test (because sometimes it's easier to just write unit tests instead of a dedicated app), you can use this wrapper around benchmark runner.

```csharp
[Trait("Category", "Unit")]
public class PerformanceTests
{
    private readonly ITestOutputHelper _output;

    public PerformanceTests(ITestOutputHelper output)
    {
        _output = output;
    }

    [Fact]
    public void TestPerformance__HeavyMethod__ShouldAllocateNothing()
    {
        var logger = new AccumulationLogger();

        var config = ManualConfig.Create(DefaultConfig.Instance)
            .AddLogger(logger)
            .WithOptions(ConfigOptions.DisableOptimizationsValidator);

        BenchmarkRunner.Run<HeavyBenchmarks>(config);

        // write benchmark summary
        _output.WriteLine(logger.GetLog());
    }
}

[MemoryDiagnoser]
public class HeavyBenchmarks
{
    [Benchmark]
    public void ProcessRequest()
    {
        var sut = new HeavyClass();
        sut.HeavyMethod();
    }
}
```

Happy unit benchmarking!

[*eof*]
