---
name: dotnet-benchmark-designer
description: Expert in designing effective .NET performance benchmarks and instrumentation. Use when creating BenchmarkDotNet tests, designing custom benchmarks, setting up profiling, or choosing measurement approaches. Knows when BenchmarkDotNet isn't suitable.
license: MIT
metadata:
  author: Aaronontheweb
  source: https://github.com/Aaronontheweb/dotnet-skills
  copied_date: "2026-02-01"
---

# .NET Benchmark Designer

Expert guidance for designing effective .NET performance benchmarks and instrumentation.

## When to Use This Skill

Use this skill when:
- Creating BenchmarkDotNet benchmark tests
- Designing custom benchmarks for scenarios BenchmarkDotNet doesn't handle
- Setting up profiling (dotTrace, dotMemory, PerfView)
- Choosing between measurement approaches
- Establishing performance baselines

## Core Expertise Areas

### BenchmarkDotNet Mastery

- Benchmark attribute patterns and configuration
- Job configuration for different runtime targets
- Memory diagnostics and allocation measurement
- Statistical analysis configuration and interpretation
- Parameterized benchmarks and data sources
- Setup/cleanup lifecycle management
- Export formats and CI integration

### When BenchmarkDotNet Isn't Suitable

- Large-scale integration scenarios requiring complex setup
- Long-running benchmarks (>30 seconds) with state transitions
- Multi-process or distributed system measurements
- Real-time performance monitoring during production load
- Benchmarks requiring external system coordination
- Memory-mapped files or system resource interaction

### Custom Benchmark Design

- Stopwatch vs QueryPerformanceCounter usage
- GC measurement and pressure analysis
- Thread contention and CPU utilization metrics
- Custom metric collection and aggregation
- Baseline establishment and storage strategies
- Statistical significance and confidence intervals

### Profiling Integration

- JetBrains dotTrace integration for CPU profiling
- JetBrains dotMemory for memory allocation analysis
- ETW (Event Tracing for Windows) custom events
- PerfView and custom ETW providers
- Continuous profiling in benchmark scenarios

### Instrumentation Patterns

- Activity and DiagnosticSource integration
- Performance counter creation and monitoring
- Custom metrics collection without affecting performance
- Async operation measurement challenges
- Lock-free measurement techniques

## Benchmark Categories

| Category | Description | Tool |
|----------|-------------|------|
| **Micro-benchmarks** | Single method/operation | BenchmarkDotNet |
| **Component benchmarks** | Class or module-level | BenchmarkDotNet |
| **Integration benchmarks** | Multi-component interaction | Custom harness |
| **Load benchmarks** | Sustained performance under load | Custom + profiler |
| **Regression benchmarks** | Change impact measurement | BenchmarkDotNet CI |

## BenchmarkDotNet Example

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class SerializationBenchmarks
{
    private Player _player = null!;

    [GlobalSetup]
    public void Setup()
    {
        _player = new Player
        {
            Id = 1,
            Name = "Test Player",
            Attributes = new PlayerAttributes { Speed = 85, Strength = 90 }
        };
    }

    [Benchmark(Baseline = true)]
    public string SerializeWithNewtonsoftJson()
    {
        return JsonConvert.SerializeObject(_player);
    }

    [Benchmark]
    public string SerializeWithSystemTextJson()
    {
        return JsonSerializer.Serialize(_player);
    }
}
```

## Design Principles

1. **Minimize measurement overhead** - Don't let measurement affect results
2. **Proper warmup** - Ensure JIT compilation is complete
3. **Control environment** - GC, CPU affinity, thermal throttling
4. **Determinism** - Same inputs should produce consistent timings
5. **Statistical rigor** - Enough iterations for significance
6. **Baseline tracking** - Store and compare over time

## Common Anti-Patterns to Avoid

- Measuring in Debug mode or with debugger attached
- Insufficient warmup causing JIT compilation noise
- Shared state between benchmark iterations
- Console output or logging during measurement
- Synchronous blocking in async benchmarks
- Ignoring GC impact on allocation-heavy operations

## Measurement Strategy Selection

| Scenario | Approach |
|----------|----------|
| Isolated micro/component tests | BenchmarkDotNet |
| Integration or long-running | Custom harness |
| Bottleneck identification | Profiler-assisted |
| Production validation | Production monitoring |
