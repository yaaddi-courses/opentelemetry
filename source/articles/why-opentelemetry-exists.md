# Why OpenTelemetry exists

**Covers:** Vendor lock-in, OpenTracing, OpenCensus, CNCF, OpenTelemetry, ELK

## A standard to end a trap {#overview}
Collecting telemetry used to mean choosing a vendor and then living with the choice. Each monitoring product came with its own code library, and that library was written into your application. Switching meant rewriting the instrumentation across every service. The industry's answer was an open standard, and it took several attempts and a merger to arrive. In this article we tell that story and explain the terms in it.

## Vendor lock-in {#vendor-lock-in}
Vendor lock-in is the trap of becoming so dependent on one company's product that switching is very costly. In telemetry, it happens when proprietary code from a monitoring vendor is embedded in your application. Moving to another vendor then means changing code in every service, retesting and redeploying, which can take months.

Lock-in is not only a technical problem. It weakens your position in price talks, because the vendor knows you cannot easily leave, and it slows you down if a better tool appears.

## ELK {#elk}
ELK stands for Elasticsearch, Logstash and Kibana, a popular open-source stack for collecting and searching logs. Logstash gathers and transforms the log lines, Elasticsearch stores and indexes them, and Kibana draws dashboards and searches. It shows that open tools existed for logs, while traces and metrics were scattered across many incompatible products.

## OpenTracing {#opentracing}
OpenTracing was an open standard for distributed tracing: a vendor-neutral set of interfaces that your code could call, with the actual tracer plugged in underneath. It was a step against lock-in, because you could change tracers without changing your instrumentation. It covered only traces, though, and it left some details open, which meant that implementations differed.

## OpenCensus {#opencensus}
OpenCensus was a project that began at Google for collecting both traces and metrics, and it came with libraries for several languages. It took a different approach from OpenTracing, which meant that the community had two competing standards. A company had to pick one, and tools often supported only one.

## CNCF {#cncf}
The CNCF, or Cloud Native Computing Foundation, is a neutral non-profit organization, part of the Linux Foundation, that hosts open infrastructure projects such as Kubernetes and Prometheus. No single company owns it, so vendors can trust a project that lives there. It is the home of OpenTelemetry, which gives the project governance and a roadmap that no vendor controls.

## OpenTelemetry {#opentelemetry}
OpenTelemetry is the open-source standard and set of tools for producing traces, metrics and logs from software. It was formed in 2019 by merging OpenTracing and OpenCensus, so that the community had one standard instead of two. It provides APIs and libraries for the main programming languages, a common data format and a separate program that forwards the data.

You instrument your code once, using OpenTelemetry, and then you can send the data to any compatible back end, commercial or open source. Changing the destination becomes a matter of configuration and not a code rewrite. Today it is one of the most active projects in the CNCF, and nearly every monitoring vendor supports it.

## A worked example {#example}
Suppose a company has 40 services. Let's compare the cost of changing monitoring vendors.

| | With a proprietary agent | With OpenTelemetry |
|---|---|---|
| Instrumentation in code | Vendor A's library in all 40 services | OpenTelemetry in all 40 services |
| To move to vendor B | Replace the library in 40 services, retest, redeploy | Change the destination address in configuration |
| Effort, assuming 3 days per service | 40 × 3 = 120 days | A few days, in one place |
| Risk of data gaps during the move | High | Low: send to both for a week |

The numbers are an illustration, but the structure is real. Work that grew with the number of services becomes work done once.

Now the history. OpenTracing (traces only) and OpenCensus (traces and metrics) merged in 2019. After the merger, neither was maintained on its own, and the community moved to OpenTelemetry. If you find an old project using one of them, plan a migration.

## A common mistake {#mistake}
A common slip is to believe that adopting OpenTelemetry means abandoning commercial tools. It does not. You still choose a back end, and many vendors build on OpenTelemetry. The gain is that the choice stays open. Another slip is to use OpenTracing or OpenCensus in new projects. They are retired.

## Key points {#key-points}
- Vendor lock-in made switching monitoring tools very costly.
- OpenTracing and OpenCensus merged in 2019 to form OpenTelemetry.
- OpenTelemetry lives in the CNCF, so no single vendor controls it.
- Instrument once, then change the destination by configuration.
