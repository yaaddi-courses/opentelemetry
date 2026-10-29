# Where to run the Collector: agents and gateways

**Covers:** Agent pattern, Gateway pattern, DaemonSet

## One program, two places to put it {#overview}
The Collector is a single program, but you can deploy it in more than one way. Where you put it decides how close it is to your applications, how much it costs to run and what it can do. Two patterns cover most setups, and many real systems use both together.

## DaemonSet {#daemonset}
A DaemonSet tells Kubernetes to start one copy of a pod on each node in the cluster. When a new node joins, Kubernetes starts a copy there automatically, and when a node leaves, its copy goes too. It is the standard way to run something that must be present on every machine, such as a log shipper or a monitoring agent.

## Agent pattern {#agent-pattern}
In the agent pattern, one Collector instance runs on every host, typically as a DaemonSet in Kubernetes, and the applications on that host send their telemetry to it. The Collector is as close to the applications as possible, often reached through `localhost` or the node's own address.

Advantages:
- **Short, fast hop:** applications hand data off quickly and carry on.
- **Local buffering:** if the network or the next stage has a problem, the agent holds the data nearby.
- **Host information:** it can add details about the node, such as its name, cloud region and labels.
- **Host metrics:** it can read the machine's own CPU, memory and disk statistics.

The drawback is that you have many instances, one per node, so configuration changes must be rolled out to all of them, and each one uses a little memory and CPU on every node.

## Gateway pattern {#gateway-pattern}
In the gateway pattern, a small central group of Collectors, run as an ordinary service behind a load balancer, receives telemetry from many applications or from many agents. Instead of a Collector on every host, you have a few in one place.

Advantages:
- **One place to configure:** sampling, filtering, redaction and the back end's credentials are in one fleet.
- **Cross-cutting decisions:** tail-based sampling needs all spans of a trace in one place, which only a central tier can provide.
- **Fewer connections to the back end:** a few Collectors talk to it, not hundreds.

The drawbacks are that applications send data over the network, so there is more distance, and that the gateway needs capacity planning, since it must handle the total volume.

## Gateway patterns {#gateway-patterns}
In the plural, the term covers variations: a single gateway for a small system, several gateways per region and layered designs in which agents on each host send to a gateway, which sends on to the back end. The layered design is the one most large systems use.

## A worked example {#example}
A cluster has 20 nodes and 200 application pods. Compare the options.

| | Agent only | Gateway only | Agents plus gateway |
|---|---|---|---|
| Collectors to run | 20 (one per node) | 3 (central) | 20 + 3 = 23 |
| Hop from application | `localhost`, fast | Across the network | `localhost` |
| Connections to the back end | 20 | 3 | 3 |
| Tail-based sampling possible | No: spans of one trace may reach different agents | Yes | Yes, at the gateway |
| Host metrics | Yes | No | Yes |
| Change the back end's address | Update 20 | Update 3 | Update 3 |

Use an agent on each node for local collection and host details, and send everything onward to the gateway for sampling and delivery. The agents' configuration stays almost fixed, and only the gateway changes when policy does.

Watch the capacity at the gateway. If the 200 pods produce 500 spans per second each, that is 100,000 spans per second in total. If one gateway Collector instance handles about 40,000 spans per second, you need at least 3 instances (100,000 divided by 40,000 is 2.5, rounded up) and should run 4 or 5 to leave room for a burst or the loss of an instance.

To start, ask whether you need host-level features or central decisions. If only the first, run agents. If only the second, run a gateway. If both, combine them.

## A common mistake {#mistake}
Running a single gateway instance, which becomes a single point of failure for all telemetry. Run at least two behind a load balancer. Another mistake is attempting tail-based sampling across agents, which splits traces across instances. Do it at a gateway that sees whole traces.

## Key points {#key-points}
- A DaemonSet runs one pod on every Kubernetes node.
- The agent pattern puts a Collector on each host. It is fast and close, and it knows the host.
- The gateway pattern uses a small central fleet. It is the place for sampling and credentials.
- Large systems usually use agents that send to a gateway.
