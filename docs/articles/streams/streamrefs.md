---
uid: stream-ref
title: StreamRefs - Reactive Streams over the network
---

# StreamRefs - Reactive Streams over the network

Stream references, or “stream refs” for short, allow running Akka Streams across multiple nodes within an Akka Remote boundaries.

Unlike heavier “streaming data processing” frameworks, Akka Streams are not “deployed” nor automatically distributed. Akka stream refs are, as the name implies, references to existing parts of a stream, and can be used to create a distributed processing framework or introduce such capabilities in specific parts of your application.

Stream refs are trivial to make use of in existing clustered Akka applications, and require no additional configuration or setup. They automatically maintain flow-control / back-pressure over the network, and employ Akka’s failure detection mechanisms to fail-fast (“let it crash!”) in the case of failures of remote nodes. They can be seen as an implementation of the Work Pulling Pattern, which one would otherwise implement manually.

> [!NOTE]
> A useful way to think about stream refs is: “like an `IActorRef`, but for Akka Streams’s `Source` and `Sink`”. Stream refs refer to an already existing, possibly remote, `Sink` or `Source`. This is not to be mistaken with deploying streams remotely, which this feature is not intended for.

## Stream References

The prime use case for stream refs is to replace raw actor or HTTP messaging between systems where a long running stream of data is expected between two entities. Often times, they can be used to effectively achieve point to point streaming without the need of setting up additional message brokers or similar secondary clusters.

Stream refs are well suited for any system in which you need to send messages between nodes and need to do so in a flow-controlled fashion. Typical examples include sending work requests to worker nodes, as fast as possible, but not faster than the worker node can process them, or sending data elements which the downstream may be slow at processing. It is recommended to mix and introduce stream refs in Actor messaging based systems, where the actor messaging is used to orchestrate and prepare such message flows, and later the stream refs are used to do the flow-controlled message transfer.

Stream refs are not persistent, however it is simple to build a recover-able stream by introducing such protocol on the actor messaging layer. Stream refs are absolutely expected to be sent over Akka remoting to other nodes within a cluster, and as such, complement and do not compete with plain Actor messaging. Actors would usually be used to establish the stream, by means of some initial message saying “I want to offer you many log elements (the stream ref)”, or alternatively in the opposite way “If you need to send me much data, here is the stream ref you can use to do so”.

Since the two sides (“local” and “remote”) of each reference may be confusing to simply refer to as “remote” and “local” – since either side can be seen as “local” or “remote” depending how we look at it – we propose to use the terminology “origin” and “target”, which is defined by where the stream ref was created. For SourceRefs, the “origin” is the side which has the data that it is going to stream out. For SinkRefs the “origin” side is the actor system that is ready to receive the data and has allocated the ref. Those two may be seen as duals of each other, however to explain patterns about sharing references, we found this wording to be rather useful.

### Source Refs - offering streaming data to a remote system

A `SourceRef` can be offered to a remote actor system in order for it to consume some source of data that we have prepared locally.

In order to share a `Source` with a remote endpoint you need to materialize it by running it into the `StreamRefs.SourceRef`. That sink materializes the `ISourceRef<T>` that you can then send to other nodes. Please note that it materializes into a Task so you will have to use the continuation (either `PipeTo` or async/await pattern).

[!code-csharp[StreamRefsDocTests.cs](../../../src/core/Akka.Docs.Tests/Streams/StreamRefsDocTests.cs?name=data-source-actor)]

The origin actor which creates and owns the `Source` could also perform some validation or additional setup when preparing the source. Once it has handed out the `ISourceRef<T>` the remote side can run it like this:

[!code-csharp[StreamRefsDocTests.cs](../../../src/core/Akka.Docs.Tests/Streams/StreamRefsDocTests.cs?name=source-ref-materialization)]

The process of preparing and running a `ISourceRef<T>` powered distributed stream is shown by the animation below:

![source ref](/images/source-ref-animation.gif)

> [!WARNING]
> A `ISourceRef<T>` is by design “single-shot”. i.e. it may only be materialized once. This is in order to not complicate the mental model what materializing such value would mean. While stream refs are designed to be single shot, you may use them to mimic multicast scenarios, simply by starting a `Broadcast` stage once, and attaching multiple new streams to it, for each emitting a new stream ref. This way each output of the broadcast is by itself an unique single-shot reference, however they can all be powered using a single `Source` – located before the `Broadcast` stage.

### Sink Refs - offering to receive streaming data from a remote system

They can be used to offer the other side the capability to send to the origin side data in a streaming, flow-controlled fashion. The origin here allocates a Sink, which could be as simple as a `Sink.ForEach` or as advanced as a complex sink which streams the incoming data into various other systems (e.g. any of the Alpakka provided Sinks).

> [!NOTE]
To form a good mental model of `SinkRef`s, you can think of them as being similar to “passive mode” in FTP.

[!code-csharp[StreamRefsDocTests.cs](../../../src/core/Akka.Docs.Tests/Streams/StreamRefsDocTests.cs?name=data-sink-actor)]

Using the offered `ISinkRef<>` to send data to the origin of the `Sink` is also simple, as we can treat the `ISinkRef<>` just as any other sink and directly runWith or run with it.

[!code-csharp[StreamRefsDocTests.cs](../../../src/core/Akka.Docs.Tests/Streams/StreamRefsDocTests.cs?name=sink-ref-materialization)]

The process of preparing and running a `ISinkRef<>` powered distributed stream is shown by the animation below:

![sink ref](/images/sink-ref-animation.gif)

> [!Warning]
> A `ISinkRef<>` is *by design* “single-shot”. i.e. it may only be materialized once. This is in order to not complicate the mental model what materializing such value would mean. If you have an use case for building a fan-in operation accepting writes from multiple remote nodes, you can build your `Sink` and prepend it with a `Merge` stage, each time materializing a new `ISinkRef<>` targeting that `Merge`. This has the added benefit of giving you full control how to merge these streams (i.e. by using “merge preferred” or any other variation of the fan-in stages).

## Configuration

### Stream reference subscription timeouts

All stream references have a subscription timeout, which is intended to prevent resource leaks in situations in which a remote node would requests the allocation of many streams yet never actually run them. In order to prevent this, each stream reference has a default timeout (of 30 seconds), after which the origin will abort the stream offer if the target has not materialized the stream ref in time. After the timeout has triggered, materialization of the target side will fail pointing out that the origin is missing.

Since these timeouts are often very different based on the kind of stream offered, and there can be many different kinds of them in the same application, it is possible to not only configure this setting globally (`akka.stream.materializer.stream-ref.subscription-timeout`), but also via attributes:

```csharp
Source.Repeat("hello")
    .RunWith(StreamRefs.SourceRef<string>()
        .AddAttributes(StreamRefAttributes
            .SubscriptionTimeout(TimeSpan.FromSeconds(5)))
    , materializer);

StreamRefs.SinkRef<string>()
    .AddAttributes(StreamRefAttributes
        .SubscriptionTimeout(TimeSpan.FromSeconds(5)))
    .RunWith(Sink.Ignore<string>(), materializer);
```
