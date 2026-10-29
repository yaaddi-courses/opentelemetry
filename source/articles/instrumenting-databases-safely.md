# Instrumenting database calls without leaking data

**Covers:** SQL, SELECT, MySQL, PII

## Useful detail and private detail sit close together {#overview}
Database calls are often the slowest part of a request, so they are the first thing you want to see in a trace. But the text of a query often contains the very data you must protect: names, emails, account numbers. Good database instrumentation records enough to find slow queries and nothing that identifies a person. In this article we go through the pieces and how to strike that balance.

## SQL {#sql}
SQL, or Structured Query Language, is the standard language for asking questions of a relational database. A query says what you want, for example "all orders for customer 42 from this month", and the database finds it. Most applications build SQL text from user input and send it to the database, and each such call is a good candidate for a span.

## SELECT {#select}
When you only want to look at data, without changing it, you write a `SELECT`. It is the statement that fetches rows from a table. Other common statements are `INSERT` to add rows, `UPDATE` to change them and `DELETE` to remove them. If a span names the kind of statement, `SELECT` or `INSERT`, that says what sort of work was done without showing any of the values. Telemetry usually records it in the `db.operation` attribute.

## MySQL {#mysql}
MySQL is a widely used open-source relational database, and it is one of many database products. In telemetry, you record which product you used in the `db.system` attribute, with the value `mysql`, so a trace tool can show the right label and group the spans. PostgreSQL, SQL Server and many others work the same way, each with its own standard value.

## PII {#pii}
PII stands for personally identifiable information: data that can identify a person, such as a name, an email address, a phone number or a card number. Laws such as the GDPR limit how it may be stored and shared, and a leak can be costly. Many people see telemetry and it is kept for a long time, so it is a risky place for PII.

The risk in database spans is the full query text, `db.statement`. A query like `SELECT * FROM users WHERE email = 'ana@example.com'` has the user's email inside it. If you write that into a span, you copy the email to every system that stores traces.

## Safe habits {#safe-habits}
- Record the operation and the table, not the values.
- Use parameterized queries, in which values are sent separately from the SQL text. The statement then contains placeholders such as `?`, and the span can safely store it.
- If you must record the statement, sanitize it by replacing literal values with placeholders. Many instrumentation libraries do this by default or have a switch.
- Never put the result rows, user names or tokens into attributes.
- Use a Collector processor to delete or hash any attribute that should not be there, as a second line of defence.

## A worked example {#example}
This program removes literal values from a query before it is recorded.

```python
import re

query = "SELECT * FROM users WHERE email = 'ana@example.com' AND id = 42"
safe = re.sub(r"'[^']*'", "?", query)
safe = re.sub(r"\b\d+\b", "?", safe)
print(safe)
# SELECT * FROM users WHERE email = ? AND id = ?
```

Let's read it.

1. The first replacement turns any text in single quotes into `?`. The email is gone.
2. The second replaces whole numbers with `?`. The id 42 is gone.
3. What remains is the shape of the query. It is safe to store and still useful, because you can group all queries of this shape and find the slow ones.

A database span built with the standard attributes then looks like this:

| Attribute | Value |
|---|---|
| `db.system` | `mysql` |
| `db.operation` | `SELECT` |
| `db.statement` | `SELECT * FROM users WHERE email = ? AND id = ?` |
| duration | 420 ms |

Now put it to use. Group spans by `db.statement`. If this one shape accounts for 60 percent of the database time, an index on `email` is the first thing to try. If the values had been recorded, 10,000 requests would have produced 10,000 different statements, which is 10,000 groups of one, and the pattern would be invisible. Real emails would also be sitting in the trace database.

This regular-expression approach is a teaching example. Real sanitizers handle more cases, such as numbers in strings and comments, so prefer the one in your instrumentation library.

## A common mistake {#mistake}
A common slip is to enable full statement capture in production without checking what it records. A debugging setting then becomes a privacy incident. Another slip is trusting a sanitizer without testing it on your own queries.

## Key points {#key-points}
- Record `db.system` and `db.operation` on database spans. Name the product, such as MySQL, with the standard value.
- The query text can contain PII, so use placeholders instead of values.
- Parameterized queries are safer for the database and for your telemetry.
- Add a Collector processor as a second line of defence.
