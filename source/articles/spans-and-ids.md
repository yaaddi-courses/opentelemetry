# Spans, attributes and identifiers

**Covers:** Span, Span attribute, Span kind, Root span, Trace ID, Span ID, IDs, UUIDs

## The building blocks of a trace {#overview}
A trace is made of smaller pieces, and every piece carries labels that make it searchable and linkable. If you understand the pieces and the identifiers that tie them together, any trace view in any tool becomes readable. This article covers them from the smallest to the largest.

## Span {#span}
A span is one timed unit of work inside a trace, such as "handle this web request" or "run this database query". It has a name, a start time, a duration and a status, and it can carry extra details. A trace is a collection of spans, and a span can have a parent span, which forms a tree.

Think of the trace as a project and spans as the tasks: each has a start and an end, and some tasks are made of smaller ones.

## Span attribute {#span-attribute}
A span attribute is a key-value pair attached to a span to add searchable detail, such as `http.status_code=200` or `db.system=postgresql`. Attributes turn a bare timing into something you can filter and group: "show me all slow spans where the payment method was card and the region was Berlin".

Choose attributes that you will actually search by. Keep them small, and never put secrets or personal data in them.

## Span kind {#span-kind}
Span kind is a label that says what role a span played in a request. The kinds are:
- **Server:** handling an incoming call.
- **Client:** making an outgoing call.
- **Internal:** work inside the process, with no call across a boundary.
- **Producer:** sending a message to a queue.
- **Consumer:** receiving and processing a message.

The kind helps tools draw the picture correctly: a client span and the server span it calls are two views of the same call, and the gap between them is network time.

## Root span {#root-span}
The root span is the first span in a trace, the one with no parent. Its duration is roughly the total time of the whole request, since everything else happens inside it. When you look at a trace, you start at the root and read down.

## Trace ID {#trace-id}
The trace ID is the one identifier shared by every span of the same request. It is how a back end gathers scattered spans from many services and puts them back together as one trace. It is 16 bytes, shown as 32 hexadecimal characters, for example `0af7651916cd43dd8448eb211c80319c`.

Because it is shared, it is also the key for finding the related logs: if every log line carries the trace ID, you can jump from a slow trace straight to its logs.

## Span ID {#span-id}
The span ID uniquely identifies one span, and no other span has the same one. It is 8 bytes, or 16 hexadecimal characters. Each span records its own ID and its parent's ID, which is how the tree structure is rebuilt from a flat list.

## IDs {#ids}
IDs are the identifiers that single out one thing, such as a trace or a span. They are generated randomly, with enough bits that two will almost never match, so that services can create them independently, with no central counter.

## UUIDs {#uuids}
UUIDs, universally unique identifiers, are 128-bit random-looking numbers that are practically never repeated, usually written as 32 hexadecimal digits in groups, such as `550e8400-e29b-41d4-a716-446655440000`. A trace ID is also 128 bits, the same size, though it is written without dashes. You may meet UUIDs as order or user identifiers that are worth adding as attributes, so that a trace can be tied to a business record.

## A worked example {#example}
A checkout request makes four spans.

| Span | Parent | Kind | Duration | Attributes |
|---|---|---|---|---|
| `POST /checkout` | none (root) | server | 950 ms | `http.status_code=200` |
| `charge-card` | `POST /checkout` | client | 900 ms | `payment.method=card` |
| `POST /charge` (payment service) | `charge-card` | server | 890 ms | `http.status_code=200` |
| `INSERT order` | `POST /checkout` | client | 20 ms | `db.system=postgresql` |

All four share the trace ID `0af7651916cd43dd8448eb211c80319c`, and each has its own span ID.

Read it: the root span, `POST /checkout`, took 950 ms in total. Its child `charge-card` took 900 ms, which is 900 / 950, about 95 percent. The matching server span in the payment service took 890 ms, so about 10 ms (900 minus 890) went on the network and the connection. The database insert took 20 ms, only 2 percent.

The children's times, 900 and 20, add up to 920 ms. The remaining 30 ms of the root's 950 is the checkout service's own work. The answer to "why was the request slow?" is plain: the payment call.

## A common mistake {#mistake}
Putting a user's email or a card number into a span attribute. Telemetry is widely visible and often stored for a long time. Use an opaque id instead. Another mistake is creating a new trace for each step of one request, which breaks the link between them.

## Key points {#key-points}
- A span is one timed unit of work. Spans form a tree, and the root span has no parent.
- Attributes add searchable detail. Span kind says whether a span was a client, server, internal, producer or consumer.
- The trace ID is shared by all spans of one request. Each span has its own span ID.
- IDs are random and large enough that they do not collide.
