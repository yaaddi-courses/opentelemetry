# The API and the SDK, and two ways to instrument

**Covers:** API, SDK, No-op, Manual instrumentation, Auto-instrumentation

## Why OpenTelemetry is split in two {#overview}
OpenTelemetry deliberately separates the part your code talks to from the part that does the work. That separation lets libraries add telemetry without forcing a particular setup on the people who use them. It also determines how telemetry gets into your code in the first place: by hand, or automatically. This article explains both ideas.

## API {#api}
An API, or application programming interface, is a defined way for one program to ask another for something. In OpenTelemetry, the API is the small set of calls that your code uses to create telemetry, such as "start a span called checkout" or "increase this number by one". The API does not decide where the data goes or how it is processed. It only describes what you can ask for.

Because it is small and stable, it is safe to depend on. A library author can add API calls to their library, and the application that uses it decides what happens to them.

## SDK {#sdk}
The SDK, or software development kit, is the implementation of the API. It is the part that actually samples, batches and exports the data. The application's owner configures it: where to send the data, how much to sample, what resource information to attach. The SDK is chosen once, in the application's startup code.

The split has a payoff. A library that is written against only the API works with any SDK, and it costs almost nothing when no SDK is present.

## No-op {#no-op}
A no-op, short for "no operation", is the safe default when an API call is made and no SDK has been configured: the call silently does nothing instead of failing. Code that was instrumented with the API keeps working, and it just produces no data.

That is why a library can safely include instrumentation, and why you may see no telemetry if you forgot to set up the SDK. If data is missing, check first that the SDK is configured.

## Manual instrumentation {#manual-instrumentation}
Manual instrumentation is telemetry code that you write by hand to capture your own business logic, the parts of the application that no general library can know about. For example, you wrap a "calculate discount" function in a span and record the discount size as an attribute.

It gives you the most meaningful data, because you choose what to measure and what to call it. The price is effort and the need to keep the code up to date. Use it for the business operations that matter most to you.

## Auto-instrumentation {#auto-instrumentation}
Auto-instrumentation adds telemetry with no source code edits, usually by attaching an agent that modifies the code as it loads, or by wrapping popular libraries. In some languages, you start the application with an extra command and instantly get spans for incoming web requests, outgoing calls and database queries.

It is the fastest way to start and covers the common frameworks. It cannot see your own business logic, and it may produce more data than you need. A good approach is to start with auto-instrumentation for broad coverage and add manual instrumentation where the detail matters.

## A worked example {#example}
A Python service uses both approaches.

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
opentelemetry-instrument --service_name shop python app.py
```

With those three commands, auto-instrumentation records a span for each web request and each database call, with no change to `app.py`.

Now the team adds one manual span around the part that matters:

```text
from opentelemetry import trace

tracer = trace.get_tracer("shop")

def apply_discount(order):
    with tracer.start_as_current_span("apply_discount") as span:
        span.set_attribute("order.items", len(order))
        return 0.1 if len(order) > 3 else 0.0
```

What happens:

1. The code depends only on the API (`opentelemetry.trace`). It would work with any SDK.
2. When started through `opentelemetry-instrument`, an SDK is configured, so the span is recorded and exported.
3. If the same code runs in a test with no SDK, the calls are no-ops: nothing is recorded and nothing breaks.

Compare the effort. Auto-instrumentation gave useful spans for 100 percent of web requests in a couple of minutes. The manual span took five lines, and added the one piece of business detail, the number of items, that makes the discount step understandable.

## A common mistake {#mistake}
Putting SDK configuration inside a shared library. Only the application should configure the SDK. Another mistake is relying only on auto-instrumentation and then being surprised that business steps are invisible. Add manual spans for them.

## Key points {#key-points}
- The API is what code calls. The SDK does the actual work and is configured by the application.
- With no SDK, API calls are no-ops, so instrumented code is always safe.
- Manual instrumentation captures your own logic. Auto-instrumentation covers common libraries without code changes.
- Combine the two: start automatic, add manual where the detail matters.
