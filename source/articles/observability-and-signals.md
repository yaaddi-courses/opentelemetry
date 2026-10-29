# Observability and the telemetry signals

**Covers:** Observability, CPU, Trace, Metric, Log

## Is it working, and why not? {#overview}
A program deployed to many machines cannot be watched by looking at it. When something goes wrong, the questions come fast: what failed, for whom, since when and why? The ability to answer them from outside is called observability. It is built from data that the program sends out about itself, called telemetry, and three kinds of that data do most of the work. Let's meet them.

## Observability {#observability}
Observability is the ability to answer new questions about a running system, using data it already emits, without shipping new code only to investigate. Monitoring asks "is the thing I expected to go wrong going wrong?" Observability lets you ask questions you did not predict, such as "why are checkouts slow only for customers using one payment method?".

A system is not observable merely because it has dashboards. It is observable when the data it produces is rich enough that you can slice it by any detail: by user, by version, by region, by request. That is why the signals below carry details and not only totals.

## CPU {#cpu}
The CPU, or central processing unit, is the chip that carries out a program's instructions. It is a good example of a resource you want to observe. If a service's CPU is at 95 percent, requests queue and slow down. When you can see the CPU use over time, next to the request times, the cause of a slowdown often becomes visible. CPU time is also what a profile later shows being spent, line by line.

## Trace {#trace}
A trace is the record of one request as it travels through the system, possibly across many services, stitched together from all of its timed steps. It answers "what happened to this request, and where did the time go?"

A trace of a checkout might show the web server taking 5 milliseconds, the cart service 20, the payment service 900 and the database 15. You can see at once that the payment call is the slow part. A trace is the signal that follows one story from start to finish.

## Metric {#metric}
A metric is a numeric measurement recorded over time, such as requests per second, error rate or memory use. Metrics are combined, or aggregated, to show trends, and they are not meant to explain any single event. They are small and cheap, so you can keep them for months and draw graphs and alerts from them.

Metrics answer "how is the system doing, and is it getting worse?" They are the first signal to fire an alarm, and the one least able to tell you why.

## Log {#log}
A log is a separate, timestamped record of something that happened, usually a short message such as "payment declined for order 1042". Logs are the oldest and most familiar signal. They carry the most detail about an individual event, and you can read the actual words.

Their weakness is volume and structure. A busy system writes millions of lines, and free text is hard to search. Logging with named fields makes logs far more useful.

## A worked example {#example}
Customers report that checkout was slow between 14:00 and 14:10. Let's follow the three signals:

| Step | Signal | What you learn |
|---|---|---|
| 1 | Metric | The 95th-percentile request time rose from 300 ms to 2,500 ms at 14:02, and CPU on the payment service hit 98 percent |
| 2 | Trace | One slow request shows 2,300 of its 2,500 ms inside the payment service |
| 3 | Log | A log line from that service in the same trace says: "retrying card provider, attempt 3" |

The metric told you when and where. The trace showed which step. The log gave the reason. Any one of them alone would have left you guessing. Together, they took about five minutes.

Here is a quick check of proportion. The payment service accounted for 2,300 of 2,500 ms, which is 92 percent of the request. If you had fixed anything else, for example by making the web server twice as fast, you would have saved less than 3 ms.

OpenTelemetry, described later in this course, is a standard way to produce all of these signals so that you can send them to any tool.

## A common mistake {#mistake}
A common slip is to believe that having dashboards means having observability. If the data lacks details such as the version, the user group or the request id, you cannot ask new questions. Another slip is to rely on one signal. Each answers a different question.

## Key points {#key-points}
- Observability means being able to ask new questions about a running system from the data it emits.
- A trace follows one request, a metric tracks numbers over time and a log records one event.
- Use metrics to detect, traces to locate and logs to explain.
- Resources such as CPU use are worth watching next to request times.
