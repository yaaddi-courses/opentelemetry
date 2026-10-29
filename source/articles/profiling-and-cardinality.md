# Continuous profiling and the cardinality trap

**Covers:** Continuous profiling, Cardinality

## One new signal and one old mistake {#overview}
Traces tell you which step was slow, but not which line of code made it slow. A newer signal fills that gap. On the other side, the most common way to ruin a metrics system is one innocent-looking label. This article covers the new signal first and the trap second, since both are about getting useful detail at a price you can afford.

## Continuous profiling {#continuous-profiling}
Continuous profiling is OpenTelemetry's newest signal. It repeatedly samples which line of code is running on the CPU at that moment, all the time and in production, so that you can see exactly where the processor's time goes. Every few milliseconds the profiler looks at the program's call stack and records it. After thousands of samples, the functions that appear most often are the ones that use the most CPU time.

The result is usually drawn as a flame graph: a wide bar is a function that takes a lot of time, and bars above it show what it called. Sampling like this has low overhead, typically a few percent of CPU, which is why it can run all the time.

Its strength comes when it is linked to traces. A trace says that the `calculate-price` span took 800 ms. The profile, filtered to that span, says that 600 of those milliseconds were spent in one sorting function. Profiling measures CPU time, so it explains slow computation. It does not explain waiting, such as time spent on a slow database.

## Cardinality {#cardinality}
Cardinality is the number of different combinations of label values that a metric can take. A metric with a label for the HTTP status (5 values) and one for the route (20 values) has up to 5 x 20 = 100 combinations. Each combination is a separate time series that the system must store and index.

The trap is attaching a label with many possible values, such as a user id, an order id or a full URL. If you have 50,000 users, the metric suddenly has 100 x 50,000 = 5 million time series. Memory use, storage cost and query time grow with it, and the monitoring system may slow down or reject the data, often when you need it most. This is called a cardinality explosion.

The rule: labels on metrics should have a small, bounded set of values. Put high-variety detail on spans and logs, which are built for it, and not on metrics.

## A worked example {#example}
How many time series does a request counter create?

```python
users = 50_000
status = 5
routes = 20
print(status * routes)
print(status * routes * users)
# 100
# 5000000
```

How to read it:

1. With labels for status and route only, there are 5 x 20 = 100 time series. That is cheap and easy to graph.
2. If someone adds a `user_id` label, each of the 100 combinations is multiplied by 50,000 users. That gives 5,000,000 series.
3. If each series costs about 3 KB of memory in the monitoring system, 5 million series use about 15 GB, against 300 KB for the original 100. That is a 50,000-fold increase for one label.

The fix is to remove `user_id` from the metric and record it as an attribute on spans, where a search such as "show traces for user 12345" works well.

Now a profiling example in words. A checkout request is slow, and the trace shows the `price` span at 800 ms. A flame graph for that span shows `sort_items` taking 75 percent of the CPU samples, which is 600 of the 800 ms. Replacing the algorithm with a built-in sort drops the span to 250 ms, and the profile confirms that `sort_items` is now below 5 percent. The cost was a profiler adding about 2 percent to CPU use.

## A common mistake {#mistake}
Using a metric label for anything that grows with your user base. Check each label and ask how many values it can take. Another mistake is expecting a profile to explain a slow database call. If the code is waiting, it is not using the CPU, so it will not appear. Use the trace for that.

## Key points {#key-points}
- Continuous profiling samples the CPU all the time to show which code uses it, usually as a flame graph.
- Link profiles to traces to go from a slow span to the guilty function.
- Cardinality is the number of distinct label combinations, and each one is a separate series.
- Keep metric labels small and bounded. Put variable detail on spans and logs.
