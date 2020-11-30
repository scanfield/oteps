# Allow GAUGE Distributions

Allow GAUGE points to have the DISTRIBUTION data type.

## Motivation

Consider a server that maintains many small resources; for example a
per-user queue with millions of users or a cache with many small
entries. It would be too costly to emit a gauge with a field for each
user, but would be useful to export a distribution of queue-length
across all users.

## Explanation

Pseudocode for the example above:

```
fn ValueObserverCb(d *DistributionObserverResult) {
  for user, queue in self.PerUserQueues {
    d->Record(queue.Length())
  }
}
```

Each time this callback is invoked, we produce a Distribution of the
current queue lengths. In some circumstances, it can be useful for the
user code to maintain the distribution separately (perhaps it can do
some incremental calculation). In that case, we'd want to support an API like:

```
fn ValueObserverCb(d *DistributionObserverResult) {
    d->Set(self.PerUserQueueLengths)
}
```

(I think we'd also need/want a sync version of the above).

With a sufficiently complex monitoring system, this allows the user to
ask questions like "what is the 95p queue length across all users").


## Internal details

There are two changes here; one is how to represent the histogram in
the wire protocol and the other is API.

For the protocol, I think this suggests separating kind
(`CUMULATIVE/DELTA/GAUGE`) from data type
(`INT/DOUBLE/HISTOGRAM/SUMMARY`). If we wanted to fit into the way the
proto is currently organized, we'd instead add a `HistogramGauge` type
to Metric::data.

In the API, we would need to provide a Distribution-like object for
use in the observer above. In a perfect world this would be the same
object that you can use in the sync API.

## Trade-offs and mitigations

This slightly increases the API surface. 

## Prior art and alternatives

Monarch supported this; I don't think Prometheus does because AFAICT the instrumentation classes are Counter/Gauge/Histogram.

## Open questions

Exemplars within these? For the queue length example above, it could
be useful to keep track of an example user per-bucket.

If they are configurable, where do we configure buckets for these histograms?

## Future possibilities
