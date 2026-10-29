# Resources: describing who produced the telemetry

**Covers:** Resource attribute, service.name, K8s, Resource detector, OS

## Not every detail belongs to a single request {#overview}
A span attribute describes one request: which route, which status. But some facts do not change from request to request: which service this is, which version, which machine, which cluster. Repeating them on every span would waste space and invite mistakes. OpenTelemetry attaches them once, to the whole process, as a resource.

## Resource attribute {#resource-attribute}
A resource attribute is a piece of metadata, set once per process, that describes the thing producing the telemetry, such as which service, which host or which Kubernetes pod, as opposed to anything about a single request. Every span, metric and log from that process carries the same resource attributes.

They let you answer questions such as "do the errors come from one version only?" or "is one host slower than the rest?" without anyone having to tag each span by hand. Typical attributes: service name, service version, deployment environment, cloud region, host name and process id.

## service.name {#service-name}
`service.name` is the specific resource attribute that tells you which service produced a piece of telemetry. It is the most important one. Dashboards and trace views group and filter by it, and when it is missing, the SDK uses a default like `unknown_service`, which makes every service look the same. Always set it, to a short, stable name such as `checkout` or `payment-api`.

## K8s {#k8s}
K8s is a shorthand for Kubernetes: the letter K, eight letters and the letter s. It is written this way because the full name is long. You will see it in tool names, labels and attribute names, for example `k8s.pod.name` and `k8s.namespace.name`, which identify the pod and the namespace in which a service runs.

## OS {#os}
OS stands for operating system. It is the layer of software that controls a computer's hardware and starts other programs. Resource attributes include facts about it, such as `os.type`, for example linux or windows, and `os.version`. They help to explain a problem that appears only on one type of system.

## Resource detector {#resource-detector}
A resource detector is a piece of SDK setup that fills in resource attributes automatically by reading them from the environment, such as the Kubernetes pod name, the cloud region, the host name or the operating system. Instead of typing these values by hand, you enable a detector, and it looks them up when the application starts.

Detectors save effort and avoid typos. Use the ones that match where your code runs, such as a Kubernetes detector in a cluster and a cloud detector on a cloud platform. You can also supply values through the standard environment variable `OTEL_RESOURCE_ATTRIBUTES`.

## A worked example {#example}
A service runs in three pods. Set the resource in an environment variable, with no code change.

```bash
export OTEL_SERVICE_NAME=checkout
export OTEL_RESOURCE_ATTRIBUTES="service.version=2.4.1,deployment.environment=production"
```

Add a detector in the SDK and the pod name, the cluster and the operating system come for free. A span from one pod then carries these resource attributes:

| Resource attribute | Value |
|---|---|
| `service.name` | `checkout` |
| `service.version` | `2.4.1` |
| `deployment.environment` | `production` |
| `k8s.pod.name` | `checkout-7d9f-abc12` |
| `os.type` | `linux` |

Use them to investigate. The slow requests are 3 percent of traffic. Group them by `k8s.pod.name`: two of the three pods have 0.5 percent slow requests, and the third has 10 percent. The problem is one pod, probably a bad node. Group by `service.version` and you might find that the slow requests all come from 2.4.1, which points to the release.

Count what you saved: if each span had to carry these five values by hand, then 1 million spans a day, at about 100 bytes of extra data each, would add 100 MB of repeated text a day, and any typo would split your data into separate groups. As a resource, they are written once per process.

## A common mistake {#mistake}
Forgetting `service.name`, so that every service appears as "unknown_service" and the traces cannot be separated. Another mistake is putting request-specific values, such as a user id, into resource attributes. Those belong on spans.

## Key points {#key-points}
- Resource attributes describe the process that produces telemetry, and are set once.
- `service.name` is the most important. Always set it.
- Names such as `k8s.pod.name` and `os.type` describe where the code runs.
- Resource detectors fill in many of these automatically.
