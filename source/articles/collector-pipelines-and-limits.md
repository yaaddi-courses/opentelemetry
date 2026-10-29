# Collector pipelines: receivers, processors and memory limits

**Covers:** Receiver, Processor, RAM, OOM

## Inside the Collector {#overview}
The Collector is built from small parts connected in lines called pipelines. Data enters at one end, passes through some steps and leaves at the other. The parts are few and simple, but the way you arrange them decides how reliable the Collector is under load. In this article we look at the first two stages, and then at the resource problem that most often brings a Collector down.

## Receiver {#receiver}
A receiver is the first stage of a pipeline. It accepts incoming telemetry and turns it into the Collector's internal form. The most common one is the OTLP receiver, which listens for data from applications. Others read different formats, such as those from Prometheus, Jaeger or Kafka, or read the host's own statistics. A pipeline can have several receivers, and one receiver can feed several pipelines.

## Processor {#processor}
A processor is the middle stage of a pipeline. It transforms, filters, batches or otherwise changes the data as it passes. Processors run in the order in which you list them, so the order matters. Useful ones include:

- **batch:** groups data into larger chunks before sending, which is more efficient.
- **memory_limiter:** refuses new data when the Collector is close to its memory limit.
- **filter:** drops data you do not want, such as health-check spans.
- **attributes:** adds, changes or removes attributes, for example to delete a sensitive field.

## RAM {#ram}
RAM, or random-access memory, is the fast working memory in which a program keeps its data. A Collector holds telemetry in RAM while it processes and batches it, and while it waits to send it on. If data arrives faster than it leaves, the amount held grows, and so does the RAM use. The Collector has no natural limit unless you give it one.

## OOM {#oom}
OOM stands for out of memory. It is the state in which a process needs more memory than is available and gets killed. In containers, the system stops the program as soon as it passes its memory limit, and often restarts it. A Collector killed this way loses all the telemetry it was holding at that moment. Then it has to start again, usually into the same flood of data, so it may be killed again and again.

The usual cause is a burst of traffic, or a slow back end that lets data pile up. The defence is the memory limiter processor, which sheds load before the limit is reached.

## A worked example {#example}
Here is a pipeline that protects itself:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 400
    spike_limit_mib: 100
  batch:
exporters:
  otlphttp:
    endpoint: https://backend.example.com
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp]
```

Let's read it in order.

1. The OTLP receiver accepts spans from the applications.
2. The `memory_limiter` is the first processor. It checks every second. If memory use passes the soft limit of 300 MiB (400 minus the 100 spike allowance), it starts refusing new data. At 400 MiB, it forces memory to be freed.
3. The `batch` processor groups what is left.
4. The exporter sends it to the back end.

The container running this Collector should have a memory limit higher than 400 MiB, for example 512 MiB. If the container's limit were also 400, the system would kill the Collector at the same moment the limiter acted, and the limiter would never help.

Now the arithmetic for a burst. Suppose spans arrive at 50 MB per second and the back end slows so that nothing leaves. The Collector reaches 300 MiB in about 6 seconds (300 divided by 50). Without the limiter, it would reach a 512 MiB container limit in about 10 seconds, be killed and lose everything it held. With the limiter, it refuses new data at 300 MiB, the senders retry later and the Collector stays alive.

Put the memory limiter first. Processors that come before it do not benefit from its protection.

## A common mistake {#mistake}
A common slip is to leave out the memory limiter, or to put it after the batch processor. The Collector then has no protection, and the first big burst kills it. Another slip is setting the limiter above the container's limit, so the system acts first.

## Key points {#key-points}
- A pipeline has receivers, processors and exporters. Processors run in the order listed.
- A Collector holds data in RAM, and the use grows when data arrives faster than it leaves.
- An out-of-memory kill loses buffered data and may repeat.
- Put `memory_limiter` first, and keep its limit below the container's limit.
