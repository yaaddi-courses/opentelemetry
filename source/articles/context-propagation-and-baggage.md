# Passing context between services

**Covers:** W3C Trace Context, traceparent, W3C, Baggage, Context propagation, RabbitMQ

## How a trace stays one trace {#overview}
When a request moves from one service to the next, each service creates its own spans. For them to be assembled into a single trace, the second service must learn the trace ID and the identity of the span that called it. Nothing makes this happen by itself. The information has to travel with the request, and the way that it does so is standardized.

## W3C {#w3c}
The W3C, or World Wide Web Consortium, is the organization that publishes web standards, including HTML. Its involvement means that the trace context format is not tied to any vendor, so services written in different languages and monitored by different tools can still pass context to each other.

## W3C Trace Context {#w3c-trace-context}
W3C Trace Context is the standard format for carrying a request's trace identity across a network call. It defines HTTP headers that hold the trace ID and the ID of the calling span, so that the receiving service can continue the same trace. Before it existed, each tracing product used its own header names, and a request that crossed from one product to another lost its trace.

## traceparent {#traceparent}
`traceparent` is the specific HTTP header, defined by the W3C standard, that carries the trace ID and the parent span ID from one service to the next. It has four fields separated by dashes: a version, the trace ID, the parent span ID and trace flags, one of which says whether the trace is being sampled.

A real value looks like `00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01`. A second header, `tracestate`, lets vendors add their own data alongside it.

## Context propagation {#context-propagation}
Context propagation is the mechanism of injecting the trace context into an outgoing request's headers on the sending side, and extracting it again on the receiving side. In HTTP, the instrumentation libraries do this for you, adding `traceparent` to every outgoing call and reading it from every incoming one.

It must work in every hop. If one service in the chain does not forward the header, the trace breaks into pieces: the later spans start a new trace with a new ID, and you cannot see the whole journey.

## Baggage {#baggage}
Baggage is user-defined key-value data that travels alongside the trace, in its own HTTP header called `baggage`, separate from `traceparent`. It lets you carry business context, such as a customer's plan or a test cohort name, through the whole request, so that every service can see it without each one looking it up again.

Baggage is not automatically added to spans. A service must read it and copy it into attributes if needed. It is also sent in clear text on every call, so never put secrets or personal data in it. Keep it small, because it adds to every request.

## RabbitMQ {#rabbitmq}
RabbitMQ is a popular open-source broker for messages: services hand messages to it, and it delivers them to other services. It matters here because a message queue breaks the direct chain of calls. The sender puts a message in the queue, and the receiver reads it later. Trace context does not travel across the queue by itself.

To keep the trace connected, the sender injects the context into the message's headers, and the consumer extracts it when it reads the message. Most OpenTelemetry libraries for messaging do this for you, and for others you add the few lines yourself.

## A worked example {#example}
This program reads a `traceparent` header and splits it into its parts.

```python
header = "00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01"
version, trace_id, parent_id, flags = header.split("-")

print(trace_id)
print(parent_id)
print(flags == "01")
# 0af7651916cd43dd8448eb211c80319c
# b7ad6b7169203331
# True
```

How to read it:

1. The header splits into four parts. The version is `00`.
2. The trace ID, 32 hexadecimal characters, identifies the whole request. Every service in the chain will use the same one.
3. The parent ID, 16 characters, is the span in the calling service. The receiving service starts its span as a child of it.
4. The flags `01` mean that the trace is sampled, so spans should be recorded.

Now follow a request through a queue:

| Step | What happens |
|---|---|
| 1 | The web service handles a request, with trace ID `0af7...`, and creates a producer span |
| 2 | It injects `traceparent` into the message headers and puts the message on RabbitMQ |
| 3 | A worker reads the message, extracts the header and starts a consumer span with the same trace ID |
| 4 | The trace now shows web request, queue send and worker processing as one trace |

Without step 2, the worker's spans would carry a new trace ID, and the trace would look as if the work had no cause. With a few minutes' work to inject and extract, the full path is visible.

## A common mistake {#mistake}
Assuming that context crosses a queue, a scheduled job or a thread pool automatically. It crosses HTTP calls when libraries are set up, and otherwise needs explicit code. Another mistake is putting secrets in baggage, which is copied into every outgoing request.

## Key points {#key-points}
- W3C Trace Context defines the `traceparent` header, with the trace ID, the parent span ID and flags.
- Context propagation injects the context into outgoing calls and extracts it on arrival.
- Baggage carries business key-value data alongside the trace. Keep it small and free of secrets.
- Across message queues such as RabbitMQ, inject and extract the context in the message headers.
