# OpenTelemetry on Kubernetes

**Covers:** CRD, Dockerfile, SSH

## Instrumenting a whole cluster without editing every image {#overview}
In a cluster with dozens of services written in different languages, adding instrumentation to each by hand is slow, and it can get out of date. Kubernetes has a way to extend itself, and OpenTelemetry uses it to do the work for you. This article explains the mechanism, which needs three small terms to follow.

## CRD {#crd}
A CRD, or custom resource definition, is a way to add your own object types to Kubernetes. Kubernetes knows built-in kinds such as Pods and Deployments. With a CRD, a tool can teach it a new kind, such as `OpenTelemetryCollector` or `Instrumentation`, which you can then create and edit with the same commands and files as any other object.

The CRD only describes the new kind. Something has to act on the objects, and that something is a program called an operator, which watches for them and makes the real world match what they say.

## Dockerfile {#dockerfile}
A Dockerfile is a text file that lists the instructions Docker follows to build an image. It matters here because the classic way of adding instrumentation is to change the Dockerfile: install the OpenTelemetry packages, change the start command and rebuild every image. Doing that for 50 services is a large project, and each team has to cooperate.

## SSH {#ssh}
SSH, or Secure Shell, is an encrypted protocol for logging in to and running commands on another machine. It appears in this story because in the old way of working, an engineer might log in to each server by SSH to install an agent, a manual and error-prone step that does not scale to hundreds of short-lived containers. Kubernetes replaces that manual work with declarations.

## The OpenTelemetry Operator {#operator}
The OpenTelemetry Operator is a program that runs in your cluster and uses CRDs to manage OpenTelemetry for you. It does two main things:

- **Runs Collectors.** You create an `OpenTelemetryCollector` object that describes the Collector's configuration and deployment mode, such as a DaemonSet for agents or a Deployment for a gateway, and the Operator creates and updates the pods.
- **Injects auto-instrumentation.** You create an `Instrumentation` object that says which exporter endpoint and settings to use. Then you add an annotation to a pod or deployment, such as `instrumentation.opentelemetry.io/inject-python: "true"`, and the Operator adds the instrumentation to the pod when it starts, with no change to the image or the Dockerfile.

The language support covers Java, Python, Node.js, .NET and Go, with some differences in how mature each one is.

## A worked example {#example}
Instrument a Python deployment without touching its image.

First, describe the settings once:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: shop-instrumentation
spec:
  exporter:
    endpoint: http://otel-collector:4317
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
```

Then annotate the deployment:

```yaml
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
```

What happens:

1. The CRD makes `Instrumentation` a kind that Kubernetes accepts, so it stores the object.
2. The Operator sees the annotation on the new pods and adds an init container and environment variables that load the Python instrumentation.
3. The service starts, creates spans for web requests and database calls and sends them in OTLP to the Collector.
4. The sampler keeps 25 percent of traces, decided from the trace ID.

Compare the effort. The old way: edit the Dockerfile of each service, rebuild and redeploy, perhaps one hour for each of 50 services, which is 50 hours, plus coordination with each team. The Operator way: one `Instrumentation` object plus one annotation per deployment, about 5 minutes each, a total of about 4 hours.

The trade-offs are real. Auto-instrumentation is added at start-up, so pods restart when you add the annotation. It covers common libraries but not your business logic, so manual spans may still be needed. And the Operator is another component to run and keep updated.

## A common mistake {#mistake}
Expecting traces after adding the annotation to a running pod. The injection happens when a pod is created, so the pods must be restarted. Another mistake is forgetting to deploy the Collector that the endpoint refers to, so that the data has nowhere to go.

## Key points {#key-points}
- A CRD adds a new object type to Kubernetes. The OpenTelemetry Operator uses CRDs to manage Collectors and instrumentation.
- An annotation on a deployment can add auto-instrumentation with no Dockerfile change.
- The manual alternatives, rebuilding images or logging in by SSH, do not scale.
- Restart pods after adding the annotation, and run a Collector for the data to reach.
