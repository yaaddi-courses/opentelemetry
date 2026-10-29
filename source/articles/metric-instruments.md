# Metric instruments: counters, gauges and histograms

**Covers:** Counter, Gauge, Histogram, Cumulative temporality, Delta temporality

## Choosing the right shape for a number {#overview}
A metric is a number measured over time, but numbers come in different shapes. The number of requests served only grows. The number of users online now goes up and down. The time that requests take is not one number at all, but a spread. OpenTelemetry has an instrument for each shape, and it also gives you a choice about how each report is sent. If you pick correctly, your graphs make sense. Let's take the instruments first.

## Counter {#counter}
A counter is a metric instrument whose value only goes up, such as the total number of requests served, the number of errors or the bytes sent. It never decreases, and it resets only when the process restarts.

You rarely look at the raw total. What matters is the rate, the increase per second or per minute, which a monitoring tool calculates from the total. "We served 1.2 million requests today" is less useful than "requests per second doubled at 14:00".

## Gauge {#gauge}
A gauge is a metric instrument for a current value that can rise or fall at any time, such as memory in use, the length of a queue or the temperature of a machine. It tells you the state right now. If you read it twice, you get two different numbers, and neither is "the total".

Use a gauge when the question is "how much is there at this moment?" Use a counter when the question is "how many have there been?"

## Histogram {#histogram}
A histogram is a metric instrument that records the distribution of many values, such as request latency. It counts how many values fall into each range, called a bucket. It lets you see not only the average but how spread out the values are, and it lets you calculate percentiles such as the 95th.

The average hides the worst cases. A histogram shows that 90 percent of requests are fast and 1 percent are very slow. The buckets are chosen in advance, so choose boundaries that match your goals, such as one at the response time you promise.

## Cumulative temporality {#cumulative-temporality}
Now the choice about reporting. With cumulative temporality, each report shows the running total since the start. If a counter was reported three times, the reports might say 5, 12 and 16. The back end subtracts to find what happened in each interval.

Its strength is resilience: if a report is lost, the next one still contains everything. Its weakness is that when a process restarts, the total drops back to zero, and the back end must detect and correct the reset.

## Delta temporality {#delta-temporality}
With delta temporality, each report shows only the change since the last report, and the count starts again after each one. The same counter would report 5, 7 and 4. It uses less memory in the application, because the application need not remember the whole history of every combination of labels.

The risk is that if a report is lost, that part of the count is lost for good. Which choice to make usually depends on what the back end supports, because some prefer cumulative values and others prefer deltas.

## A worked example {#example}
These two programs show the difference between the two ways of reporting, and what a histogram records.

```python
reports = [5, 7, 4]
cumulative = []
total = 0
for n in reports:
    total += n
    cumulative.append(total)
print(cumulative)
print(reports)
# [5, 12, 16]
# [5, 7, 4]
```

The counter saw 5, then 7, then 4 new requests in three intervals. Cumulative reports are the running totals, 5, 12 and 16. Delta reports are the new amounts alone, 5, 7 and 4.

```python
latencies_ms = [12, 48, 95, 230, 410, 55, 700, 88, 1200, 64]
bounds = [50, 100, 500, 1000]
counts = [0] * (len(bounds) + 1)
for value in latencies_ms:
    for i, bound in enumerate(bounds):
        if value <= bound:
            counts[i] += 1
            break
    else:
        counts[-1] += 1
print(counts)
print(sum(latencies_ms) / len(latencies_ms))
# [2, 4, 2, 1, 1]
# 290.2
```

Here is how to read the histogram.

1. The buckets are up to 50 ms, up to 100, up to 500, up to 1,000 and above 1,000.
2. Two requests were under 50 ms, four were between 51 and 100, two were between 101 and 500, one was between 501 and 1,000 and one was above 1,000.
3. The average is 290.2 ms, but the histogram shows that 6 of 10 requests (2 + 4) finished within 100 ms. The one request over a second pulls the average up.

A gauge would be the right instrument for "number of requests being handled right now", which rises and falls. A counter would be right for "total requests", which only grows.

## A common mistake {#mistake}
A common slip is to use a gauge for something that only grows, or a counter for something that falls, so the graphs are meaningless. Another is to look at the average of a latency and ignore the distribution. Use a histogram for latency.

## Key points {#key-points}
- A counter only goes up, so look at its rate. A gauge reports a current value. A histogram records the spread.
- Cumulative reports the running total, and delta reports the change since the last report.
- Match the instrument to the question you want to answer.
- Use histograms for latency, and look at percentiles.
