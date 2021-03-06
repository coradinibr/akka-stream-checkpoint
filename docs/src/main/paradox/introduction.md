# Introduction

## Motivation

[Reactive Streams](http://www.reactive-streams.org/) are the key to build asynchronous, data-intensive applications with 
no predetermined data volumes. By enabling non-blocking backpressure, they boost the resiliency of your systems by design.
@extref[Akka Streams](akka-docs:scala/stream/index.html) is a widely used toolkit that internally implements the 
Reactive Streams JVM specification, whilst offering a much more user-friendly API.

But how do you tune and debug such applications? Two main challenges make monitoring Akka Streams application very different
(and somehow trickier) then traditional applications.

1. **non-linearity**: Akka Streams stages are not functions. Each stage's inputs and outputs are uncorrelated: zero/one input
can produce multiple (potentially infinite) outputs. Conversely, multiple inputs can produce fewer (potentially zero) outputs.
This makes measuring key metrics such as throughput and latency more challenging, as there is no generic way to determine 
when an event has been processed.

2. **backpressure**: Akka Streams stages are backpressure-enabled. This means that, in a multi-stage pipeline, the slower
stage will determine the throughput of the whole pipeline. This is the whole point of Reactive Streams, allowing your
pipeline to go "as fast as possible, but not faster". However, this gets in the way of effectively looking for bottlenecks,
as there is no clear way of understanding what is backpressuring and what's not.

Whilst Akka Streams doesn't currently provide a comprehensive solution to these issues, it exposes a low-level API that 
allows to perform custom actions when _push_ and _pull_ signals are sent to a stage. This library leverages this API. 
For more details on this, check out @extref[this section](akka-docs:/stream/stream-customize.html) of the Akka Streams 
documentation.

## What this library does

_Akka Streams Checkpoint_ attempts to overcome the two challenges above. It allows to create generic, pass-through stages
that can be placed at meaningful points in your pipeline. 

### Throughput
Throughput is the measurement of how many events are processed by your pipeline in a unit of time. Because of non-linearity,
measuring the throughput of a whole pipeline might not necessarily be helpful. Each section of your pipeline can have a 
different throughput, and it is up to the developer to determine what's meaningful and what's not.

A _checkpoint_ placed in a pipeline will be notified each time an element is pushed out of it, allowing the backend to record
the event accordingly.

@@@ note

Throughputs can also be aggregated to record "relative" throughputs. A ratio between throughputs measured at two different
points in your pipeline will give you an idea of how many events are dropped/conflated (if the ratio is lower than 100%)
or how many are generated/expanded (if the ratio is higher than 100%). Dashboard tools like [Grafana](https://grafana.com/) 
and [Datadog](https://www.datadoghq.com/) allow to compose series.

@@@

### Pull-push and push-pull latency
Each _checkpoint_ is a linear stage that emits downstream all the elements which are pushed from upstream. This means that 
for each pull from downstream, there will be a push from upstream.
Latency is being measured:

- _between a push and the last pull_: this will give us a reading of how much time the upstream takes to produce an element after 
it's been requested
- _between a pull and the last push_: this will give us a reading of how time the downstream takes to request an element after
the previous element has been emitted

### Backpressure ratio
A _checkpoint_ can be in one of two working states:

- _backpressuring_: the checkpoint is waiting for a pull from the downstream
- _NOT backpressuring_: the checkpoint is waiting for a push from the upstream

The _backpressure ratio_ metric measures the percentage of time spent by the checkpoint in the backpressuring state. A high
backpressure ratio indicates that a bottleneck is present downstream of the checkpoint.

### Liveness
Backpressured systems can suffer from liveness issues. These can have a variety of causes, ranging from an unresponsive 
third party, all the way to @extref[code bugs](akka-docs:/stream/stream-graphs.html#graph-cycles-liveness-and-deadlocks).
In case of a liveness issue, no demand will flow from downstream to upstream. This will cause all the metrics 
previously described to fall silent. To solve this issue, each _checkpoint_ keeps a basic flag counter that represents
if the _checkpoint_ is currently backpressured. Because of its simplicity, this metric becomes meaningful in case of 
liveness issues.

### Failures / Completions
Failures and completions are lifecycle termination events for any Akka Stream stage. Failures are erroneous terminations, whilst
completions are successful terminations.
Akka Streams offers a way to track these events using @extref[monitor stages](akka-docs:/stream/operators/index.html#watching-status-stages)
However for convenience purposes, the same events are tracked by `akka-stream-checkpoint` as 2 distinct metrics. 

## What this library does NOT

### Trace context
_Akka Streams Checkpoint_ is not a solution to allow tracing of context in Akka Streams applications. However, the
checkpoints provided by this library are completely transparent and should be able to coexist with any pipeline model,
including trace context.