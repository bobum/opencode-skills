---
name: dotnet-performance-analyst
description: Expert in analyzing .NET application performance data, profiling results, and benchmark comparisons. Use when interpreting profiler output, analyzing BenchmarkDotNet results, detecting regressions, or identifying bottlenecks.
license: MIT
metadata:
  author: Aaronontheweb
  source: https://github.com/Aaronontheweb/dotnet-skills
  copied_date: "2026-02-01"
---

# .NET Performance Analyst

Expert guidance for analyzing .NET application performance data, profiling results, and identifying bottlenecks.

## When to Use This Skill

Use this skill when:
- Interpreting JetBrains profiler output (dotTrace, dotMemory)
- Analyzing BenchmarkDotNet results
- Detecting performance regressions
- Identifying CPU, memory, or I/O bottlenecks
- Establishing performance baselines and budgets

## Core Expertise Areas

### JetBrains Profiler Analysis

**dotTrace CPU profiling:**
- Call tree analysis and hot path identification
- Thread contention detection
- Timeline profiling interpretation
- UI responsiveness analysis

**dotMemory analysis:**
- Memory allocation patterns
- GC pressure identification
- Memory leak detection
- Large Object Heap analysis

### BenchmarkDotNet Results Analysis

- Statistical interpretation: mean, median, standard deviation
- Percentile analysis and outlier identification
- Memory allocation analysis and GC impact
- Scaling analysis across different input sizes
- Cross-platform performance comparison
- CI/CD performance regression detection

### Baseline Management

- Establishing performance baselines from historical data
- Regression detection algorithms and thresholds
- Performance trend analysis over time
- Environmental factor normalization
- Statistical significance testing for changes
- Performance budget establishment

## Bottleneck Identification Patterns

| Type | Symptoms | Investigation |
|------|----------|---------------|
| **CPU-bound** | High CPU, hot methods | Call tree, algorithm complexity |
| **Memory-bound** | GC pressure, allocations | dotMemory, allocation tracking |
| **I/O-bound** | Waiting, low CPU | Async analysis, batching opportunities |
| **Lock contention** | Thread starvation | Synchronization analysis |
| **Cache misses** | Slow memory access | Data locality, access patterns |
| **JIT compilation** | Warmup overhead | Tier compilation analysis |

## Performance Metrics Interpretation

### Latency Analysis

| Percentile | Use Case |
|------------|----------|
| P50 (median) | Typical user experience |
| P95 | Most users' worst case |
| P99 | SLA compliance |
| P99.9 | Tail latency for critical systems |

### Throughput vs Latency Trade-offs

- Higher throughput often increases latency variance
- Batching improves throughput but increases individual latency
- Connection pooling affects both metrics

## Common Performance Issues

| Issue | Detection | Solution |
|-------|-----------|----------|
| **Sync-over-async** | Deadlocks, context switching | Use async all the way |
| **Boxing in hot paths** | Allocations in value types | Generic constraints |
| **String concatenation** | GC pressure in loops | StringBuilder |
| **LINQ in hot paths** | Allocations, delegate overhead | Explicit loops |
| **Exception handling** | Slow normal flow | Avoid for control flow |
| **Reflection** | Slow invocation | Compiled expressions, source gen |
| **LOH pressure** | Compaction pauses | ArrayPool, chunking |

## Regression Analysis Framework

### Statistical Confidence

```
Regression detected if:
  |new_mean - baseline_mean| > (baseline_stddev * threshold)

Typical thresholds:
  - Warning: 2 standard deviations (5% false positive)
  - Alert: 3 standard deviations (0.3% false positive)
```

### Environmental Factors to Normalize

- Hardware (CPU model, memory speed, disk type)
- OS version and configuration
- .NET runtime version
- Background processes and thermal state
- Time of day (for CI runners)

## Profiler Data Correlation

1. **Cross-reference CPU and memory** - High allocations often correlate with CPU time
2. **Correlate GC events with pauses** - Gen2 collections cause longest pauses
3. **Map thread contention to code** - Lock statements, ConcurrentDictionary hotspots
4. **Connect issues to code paths** - Use call stack attribution

## Performance Optimization Validation

### Before/After Comparison

1. Run baseline multiple times (10+ iterations)
2. Apply optimization
3. Run same iterations
4. Compare using statistical tests
5. Check for unintended side effects (memory, correctness)

### Multi-Metric Assessment

Don't optimize one metric at the expense of others:

| Metric | Acceptable Trade-off |
|--------|---------------------|
| Throughput | May increase memory usage |
| Latency | May decrease throughput |
| Memory | May increase CPU time |
| Startup | May increase steady-state memory |

## Actionable Recommendations Format

When reporting findings:

```markdown
## Finding: [Bottleneck Description]

**Impact**: [Quantified - e.g., "45% of CPU time", "2GB/hour allocations"]

**Root Cause**: [Specific code path or pattern]

**Recommendation**: [Concrete fix with code example]

**Expected Improvement**: [Estimated gain]

**Risk**: [Any trade-offs or risks]
```
