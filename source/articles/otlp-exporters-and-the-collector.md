# OTLP, exporters and the Collector

**Covers:** OTLP, Exporter, Collector

## How telemetry leaves your application {#overview}
Once the application has produced spans, metrics and logs, they have to get to a place where people can look at them. Three pieces make up that journey. A format says how the data is written. An exporter in the application sends it. And, optionally, a separate program in the middle receives, adjusts and forwards it. If you understand these three, you can explain almost every OpenTelemetry setup you will meet.

## OTLP {#otlp}
OTLP, the OpenTelemetry Protocol, is the standard format for sending telemetry between applications, collectors and back ends. It carries traces, metrics and logs in one family of messages, and it can travel over gRPC, on port 4317 by default, or over HTTP, on port 4318. Nearly every monitoring product can now receive OTLP directly.

Because it is a common language, you can change the destination without touching the application, as long as the new destination speaks OTLP.

## Exporter {#exporter}
An exporter is the component that sends processed telemetry to its final destination, such as a back end, a Collector or the console for local debugging. It belongs to the SDK. You choose and configure it in the application's setup: which protocol, which address and how often to send.

Common exporters are OTLP, which is the default choice, the console exporter, which prints data on the screen, and exporters for specific tools. Telemetry is sent in batches and not one item at a time, so the application is not slowed.

## Collector {#collector}
The Collector is an optional but recommended standalone program that sits between your applications and the back end. It receives telemetry, can process it and sends it on. It runs as its own process and is not part of your code.

Why add it? Applications can send data quickly to something nearby and move on, and the Collector handles retries and buffering if the back end is slow. You can change back ends, thin out the data, remove sensitive fields or send a copy to a second destination by changing the Collector's configuration, with no change to any application. It also keeps the API keys for the back end in one place, and not in every service.

## A worked example {#example}
Let's configure an application to send to a Collector, and the Collector to send on to a back end.

In the application, using standard environment variables:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
export OTEL_SERVICE_NAME=checkout
```

In the Collector's configuration:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
processors:
  batch:
exporters:
  otlphttp:
    endpoint: https://backend.example.com
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp]
```

This is how the data flows.

1. The application's exporter sends spans in OTLP to `localhost:4317`, which takes under a millisecond.
2. The Collector receives them, batches them and forwards them over HTTP to the back end.
3. If the back end is unreachable for a minute, the Collector holds the data and retries. The application is not slowed.

Now compare two ways of changing the back end across 30 services. Without a Collector, each service's exporter points to the back end, so all 30 must be reconfigured and redeployed. With a Collector, only its `endpoint` line changes. That is 1 change in place of 30.

Check the protocol numbers too. gRPC on 4317 and HTTP on 4318 are the defaults, so if the data does not arrive, first confirm that the port in the endpoint matches the protocol the listening side expects.

## A common mistake {#mistake}
A common slip is to send directly to a distant back end from every application, with long timeouts, so a slow back end slows the applications. Send to a nearby Collector. Another slip is to mix up the ports, such as sending gRPC to the HTTP port.

## Key points {#key-points}
- OTLP is the standard OpenTelemetry format, sent over gRPC or HTTP.
- An exporter in the SDK sends data out. Choose the OTLP exporter in most cases.
- A Collector sits between applications and back ends to buffer, process and route data.
- With a Collector, changing a back end is one configuration change.
