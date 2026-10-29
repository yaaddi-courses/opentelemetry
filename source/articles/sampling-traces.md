# Sampling: keeping the traces that matter

**Covers:** Sampling, Head-based sampling, Tail-based sampling, IP

## You cannot keep everything {#overview}
A busy system can produce millions of traces a day. Storing, indexing and searching all of them costs a great deal, and most of them are ordinary, successful requests that nobody will ever look at. Sampling deals with this: you keep a fraction, and try to make sure that the fraction includes the traces that matter. The big question is when to decide.

## Sampling {#sampling}
Sampling is deliberately keeping only a fraction of the traces instead of every one, trading some completeness for lower storage cost and lighter queries at high volume. A rate of 10 percent keeps one trace in ten and drops the other nine.

Sampling is a trade-off, and has to be designed with care. Keep too little and rare problems disappear from the data. Keep too much and the cost is high. It applies to whole traces: you keep or drop every span of a trace together, since half a trace is hard to read. Metrics are not usually sampled, because they are already summaries.

## Head-based sampling {#head-based-sampling}
Head-based sampling decides whether to keep a trace at its very start, before the request has finished. The decision is made once, at the first span, and passed along to every other service through the trace flags in `traceparent`, so that all services agree.

It is simple and cheap. No data needs to be held while waiting. But it cannot favour interesting traces, because at the start you do not yet know whether the request will fail or be slow. With a 10 percent rate, you keep 10 percent of the errors too, and may miss the one failure you needed.

A common choice is a ratio-based sampler that derives the decision from the trace ID, so that every service reaches the same answer on its own.

## Tail-based sampling {#tail-based-sampling}
Tail-based sampling decides whether to keep a trace only after it has finished, when the whole picture is known. That makes it possible to keep every trace that contained an error, every trace slower than a threshold and a small share of the rest.

The price is that the system must hold all the spans of each trace in memory until it is complete, and that all spans of a trace must arrive at the same place. That is why it is run in a central Collector, the gateway, and why it uses much more memory than head-based sampling.

## IP {#ip}
An IP address, or Internet Protocol address, is the number that identifies a device on a network. It matters to sampling in one way: for tail-based sampling to work, every span of a trace must reach the same Collector instance, and a common technique is to route by the trace ID, using a load-balancing exporter that picks the instance for each trace. The Collectors behind it are found by their IP addresses or by a name that resolves to them.

## A worked example {#example}
A hundred traces, with every 25th one containing an error (4 errors). Compare the two ways of keeping about 10 percent.

```python
traces = [{"id": i, "error": i % 25 == 0} for i in range(1, 101)]
head = [t for t in traces if t["id"] % 10 == 0]
tail = [t for t in traces if t["error"] or t["id"] % 10 == 0]
print(len(head), sum(t["error"] for t in head))
print(len(tail), sum(t["error"] for t in tail))
# 10 2
# 12 4
```

How to read it:

1. The head rule keeps traces whose id ends in 0, 10 in all, without looking at the content. By chance, 2 of the 4 error traces (ids 50 and 100) were kept, and the other 2 (ids 25 and 75) were lost.
2. The tail rule also keeps any trace with an error. It keeps 12 traces, the 10 from before plus the 2 extra error traces, and all 4 errors are saved.
3. Both keep about a tenth of the traffic. The tail version spends two more traces to keep every error.

Now the cost side. Suppose the system produces 10,000 traces a second, each with 20 spans, and traces take up to 30 seconds to finish. Tail-based sampling must hold up to 10,000 x 30 = 300,000 traces, or 6 million spans, in memory at once. Head-based sampling holds none. If a span takes 1 KB, the tail buffer is about 6 GB. That is the hidden cost of the extra insight, and the reason to run it only in a well-sized gateway.

Many teams combine them: head-sample a modest share at the start to limit the volume, then use tail-based rules to guarantee that errors and slow traces are kept.

## A common mistake {#mistake}
Using head-based sampling at 1 percent and then wondering why the rare error cannot be found. If an error is 0.1 percent of requests, a 1 percent sample keeps about 1 error in 1,000. Use tail-based rules for what you must not lose. Another mistake is sampling spans independently, which cuts traces into incomplete pieces.

## Key points {#key-points}
- Sampling keeps a fraction of traces to control cost. Keep or drop whole traces.
- Head-based sampling decides at the start. It is cheap and blind to errors.
- Tail-based sampling decides at the end, so it can keep errors and slow traces, at the cost of memory.
- Tail-based sampling needs all spans of a trace in one Collector, so route by trace ID.
