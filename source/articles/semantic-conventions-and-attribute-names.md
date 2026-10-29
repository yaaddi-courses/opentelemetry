# Semantic conventions: agreeing on names

**Covers:** Semantic conventions, http.status_code, db.system, db.operation

## Why names matter more than they seem {#overview}
Telemetry is only useful if different teams, tools and languages mean the same thing by the same label. One team records the web status as `status`, another as `http_status` and a third as `responseCode`. A dashboard that must work for all three has to know every spelling. The cure is a shared list of names, and OpenTelemetry provides one.

## Semantic conventions {#semantic-conventions}
Semantic conventions are OpenTelemetry's standard names for common attributes, so that different teams, libraries and languages describe the same things in the same way. They cover web requests, databases, messaging, cloud platforms, Kubernetes and more. Names are written in lowercase, with dots as separators, grouped by area: `http.request.method`, `db.system`, `messaging.system`, `k8s.pod.name`.

The conventions also specify the type and the allowed values. An attribute meant to hold a number must be a number, and the standard says which units to use. The benefit is practical: a dashboard written once for `http.status_code` works for every service that follows the standard, in any language.

Conventions change over time. For example, newer versions of the HTTP convention use `http.response.status_code` and `http.request.method`. When you read older material, check which version of the convention it follows.

## http.status_code {#http-status-code}
`http.status_code` is the attribute that holds the HTTP response status code, such as 200, 404 or 500. When you set it on server and client spans, you can count errors, find all requests that returned 500 and compare failure rates across versions. Instrumentation libraries usually set it for you. Treat codes in the 500 range as server errors, and for server spans the instrumentation marks them as errors automatically.

## db.system {#db-system}
`db.system` is the attribute that names the database product, such as `postgresql`, `mysql`, `redis` or `mongodb`. With it, a trace view can show an icon, and you can ask "which database is the slowest?" or "how much time do we spend in each kind of store?" It must use the standard value for each product, because a different spelling breaks the grouping.

## db.operation {#db-operation}
`db.operation` is the attribute that names the kind of database operation, such as `SELECT`, `INSERT` or `UPDATE`. Together with the system name, it describes a database span at a safe level of detail. A span with `db.system=postgresql` and `db.operation=SELECT` shows that a read happened, without revealing the values in the query.

## A worked example {#example}
Suppose three teams record the same events. Let's compare the two situations.

| Without conventions | With conventions |
|---|---|
| Team A: `status=500` | `http.status_code=500` |
| Team B: `responseCode=500` | `http.status_code=500` |
| Team C: `db=pg`, `op=read` | `db.system=postgresql`, `db.operation=SELECT` |
| A dashboard needs 3 rules per concept | A dashboard needs 1 rule per concept |

Count the saving. A dashboard that shows the error rate, the slowest database and the busiest operation needs 3 concepts. With three naming styles, that is 9 separate rules to write and maintain. With one standard, it is 3, and it works for a fourth team on day one.

Code that sets these attributes by hand uses the standard names:

```text
span.set_attribute("http.status_code", 200)
span.set_attribute("db.system", "postgresql")
span.set_attribute("db.operation", "SELECT")
```

A query then works across all services: "show spans where `db.system` is `postgresql` and the duration is over 500 ms, grouped by `db.operation`". If the result shows that `SELECT` is 90 percent of the slow spans, the team knows to look at read queries and indexes first.

## A common mistake {#mistake}
A common slip is to invent your own names for something that already has a standard one, such as `http_code`. It works for you and breaks every shared dashboard and alert. Another slip is to mix versions of the conventions in one system without noticing, so half of the services use the old name and half use the new one.

## Key points {#key-points}
- Semantic conventions are the shared attribute names, types and units, such as `http.status_code`.
- `db.system` names the database and `db.operation` names the kind of operation.
- A shared vocabulary lets one dashboard or alert work for every service.
- Check which version of a convention you follow, since the names evolve.
