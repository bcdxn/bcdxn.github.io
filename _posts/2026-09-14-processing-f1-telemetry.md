---
title: "Processing 120,000 Telemetry Events for Fun"
date: 2026-09-14 07:30:00 -0400
categories: [blog]
tags: [go, golang, f1, livetiming, websockets, streaming]
excerpt: "A look at the small Go pipeline behind an F1 live timing recorder, replay server, and terminal dashboard — with a closer look at the in-memory fanout doing the busy work."
image:
  path: /assets/images/banners/2026-09-14.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-09-14.jpg
---

[![f1-banner](/assets/images/banners/2026-09-14.jpg)](https://unsplash.com/photos/a-red-and-white-race-car-speeds-around-a-track-EwUZ8hjWXSk)

I spent the last few days building a small F1 live timing system in Go. It connects to the F1 SignalR service, records raw messages to SQLite, replays them through a local WebSocket server, and feeds a terminal dashboard that reconstructs timing state in memory.

The project captured each session from the Monza race weekend (FP1, FP2, FP3, Qualifying, and the Grande Prix). In total, my client processed **120,413 stream events**. Obviously their public event broadcast is a tiny subset of the [raw event data processed](https://aws.amazon.com/blogs/media/real-time-storytelling-the-aws-architecture-behind-formula-1-track-pulse/) by F1 themselves.

That number is just large enough to make the implementation interesting, but small enough that the right architecture is not a distributed system. Instead, I implemented an in-memory broadcaster using goroutines and channels.

Here is the resulting TUI built using [Charmbracelet's Bubble Tea framework](https://github.com/charmbracelet/bubbletea) that consumes the telemetry data:

<video src="/assets/videos/2026-09-14-race-telemetry.mp4" autoplay loop muted width="100%">
</video>

## The Architecture

At a high level, the pipeline looks like this:

![architecture](/assets/images/2026-09-14-vehicle-telemetry.png){: .img-max-height}

The live timing client has one job: negotiate a SignalR connection, subscribe to the requested streams, and expose ordered raw messages on a Go channel. It does not understand timing data and it does not merge reference messages with deltas. I left that to the consumers. This allows the client to A, be very fast and B, preserve the exact messages from the origin SignalR server allowing for replay and mocking.

The recorder subscribes to the broadcaster with a blocking policy and writes each raw payload to SQLite with its sequence number and receive timestamp. SQLite simply gives me a durable capture without making the live path depend on a more complicated event format.

The replay server reads those rows back in order and speaks enough of the SignalR protocol for the same client to connect to it (it can basically act as a mock which helped me to test out the TUI when a real session was not in progress). It can preserve the original timing or accelerate it with a `speed` query parameter.

The TUI uses a reducer to hold the latest raw value for each stream, merges incoming reference and delta messages, then projects that data into timing rows, weather, race control messages, and team radio state. The UI never needs to know whether its input came from a live WebSocket or a local replay server.

This separation gives each layer a small responsibility:

- the client handles transport
- the broadcaster handles delivery
- the recorder handles durability
- the replay server handles deterministic playback
- the reducer handles application state
- the TUI handles presentation

The broadcaster is the most interesting part because it sits directly on the boundary between a fast producer and consumers that have very different speeds.

## Counting the Weekend

The recordings contain raw SignalR rows, not one row per logical F1 stream update. A single payload can contain multiple data stream events. e.g., `WeatherData`, `Position.z`, and `TimingAppData` can all be packaged together in a single SignalR payload.

Counting invdividual data stream events over the race weekend yielded:

| Session           | Stream events |
| ----------------- | ------------: |
| FP1               |        17,510 |
| FP2               |        19,026 |
| FP3               |        15,321 |
| Qualifying        |        10,640 |
| Race              |        57,916 |
| **Weekend total** |   **120,413** |

## A Broadcaster With One Owner

The public API is simple:

```go
type Broadcaster struct {
    ctx      context.Context
    source   <-chan livetiming.Event
    commands chan command
    done     chan struct{}
}

func New(ctx context.Context, source <-chan livetiming.Event) *Broadcaster {
    broadcaster := &Broadcaster{
        ctx:      ctx,
        source:   source,
        commands: make(chan command),
        done:     make(chan struct{}),
    }
    go broadcaster.run()
    return broadcaster
}
```

`New` starts one goroutine. That goroutine owns the mutable subscriber map and the sequence counter. Subscribers do not acquire a mutex to register, remove themselves, or receive an event. They send commands to the run loop, and the run loop processes those commands alongside source messages:

```go
func (b *Broadcaster) run() {
    state := fanoutState{subscribers: make(map[*subscriber]struct{})}

    defer close(b.done)
    defer func() {
        for subscriber := range state.subscribers {
            close(subscriber.channel)
        }
    }()

    for {
        select {
        case <-b.ctx.Done():
            return
        case request := <-b.commands:
            state.handleCommand(request)
        case sourceEvent, ok := <-b.source:
            if !ok {
                return
            }
            state.publish(b, sourceEvent)
        }
    }
}
```

This is a useful pattern when there is one natural owner for a small piece of mutable state. The map is not shared between goroutines, so we don't need to worry about race conditions on update. The sequence number is assigned in the same place that decides which subscribers see the event, so all subscribers observe the same ordering.

> **<i class="fas fa-lightbulb"></i>** The broadcaster is not trying to be a general-purpose message bus. It is a short, in-process handoff between one ordered source and a few consumers.
> {: .notice--info}

## Copying Is Part of the Contract

The input event contains a byte slice. Byte slices are references to backing arrays, so passing that slice to several consumers would make ownership ambiguous (i.e. changes by one consumer would reflect in the others and create possible race conditions).

The broadcaster copies the source payload when it creates the shared event, then copies it again for each subscriber:

```go
func (s *fanoutState) publish(b *Broadcaster, sourceEvent livetiming.Event) {
    event := Event{
        ReceivedAt: time.Now(),
        Data:       append([]byte(nil), sourceEvent.Data...),
        Err:        sourceEvent.Err,
    }
    if len(event.Data) > 0 {
        s.sequence++
        event.Sequence = s.sequence
    }

    for subscriber := range s.subscribers {
        delivery := event
        delivery.Data = append([]byte(nil), event.Data...)
        if subscriber.policy == Drop {
            s.tryDeliver(subscriber, delivery)
            continue
        }
        s.deliver(b, subscriber, delivery)
    }
}
```

That is an allocation cost, but it buys a simple ownership rule: every subscriber owns the `Event` value it receives.

The sequence number also has a deliberate detail. It increments for messages containing real live-timing data, but not for an empty control or error event. Consumers can use it to identify the order of actual payloads without treating a terminal error notification as another timing message.

## Blocking and Dropping

Subscribers do not all have the same delivery requirements. The recorder must not quietly skip a live timing payload because SQLite is briefly slow. In theory the TUI shouldn't care if it misses a few messages, however because the messages don't carry full state, and are simply deltas, the reducer actually needs every event to reconstruct the state accurately. This could have a more sophisticated error handling where the stream is restarted, getting a fresh reference state and continuing to stream new deltas. For now, I've simply opted to use the `Block` style delivery requirement for both recorder and the TUI.

```go
func (s *fanoutState) deliver(b *Broadcaster, subscriber *subscriber, event Event) {
    select {
    case subscriber.channel <- event:
    case <-subscriber.ctx.Done():
        delete(s.subscribers, subscriber)
        close(subscriber.channel)
    case <-b.ctx.Done():
    }
}
```

With a blocking subscriber, a full buffer applies backpressure to the broadcaster and eventually to the source. That is the correct default for the recorder: it makes slowness visible instead of turning it into silent data loss.

There are also consumers where dropping is an acceptable tradeoff. Those subscribers use a non-blocking send:

```go
func (s *fanoutState) tryDeliver(subscriber *subscriber, event Event) {
    select {
    case subscriber.channel <- event:
    default:
        subscriber.dropped.Add(1)
    }
}
```

The dropped count is an atomic counter because it can be read by a consumer while the broadcaster is still publishing. The drop is not hidden: the TUI can expose a degraded state, and callers can inspect `Subscription.Dropped()` to decide what to do next.

This policy is more useful than a global choice between "always block" and "always drop." Durability-oriented consumers can slow the producer. Best-effort consumers allow the producer to keep moving. The behavior is selected at subscription time.

## It's Fast Enough

There is no queue service, serialization hop, lock around every subscriber, or intermediate event model on the hot path. The broadcaster does a small amount of work per source message:

1. copy the raw payload;
2. assign one sequence number;
3. copy the payload once per subscriber;
4. send through each subscriber's channel.

The data remains raw until a consumer needs to interpret it. This allows the recorder to preserve the original message, while allowing the TUI to decode and merge it. Decoding once in the broadcaster would force every consumer to share an application-specific representation and would add work even for consumers that only want to archive or forward the bytes.

The single owner goroutine is also a practical optimization for this workload. There are only a handful of subscribers, and the source is already ordered. Avoiding lock contention keeps the common path straightforward. More importantly, it makes correctness cheap: ordering, membership, cancellation, and channel closure all happen in one place.

Ultimately the TUI is snappy and since all consumers are local there's no need for more sophisticated distributed fanout mechanisms.

## Closing Thoughts

The final number of events was smaller than the title I first had in mind when I set out to build the TUI and write this post, but 120,413 events with peaks of 20 events per second still made for interesting experiement.

The broader lesson is architectural. Separate concerns of transport, fanout, storage, replay, state reduction, and presentation. Then make the API boundary between them clear and simple; in my case that turned out to be a channel owned by a single go routine.

[Check out the code](https://github.com/bcdxn/f1)

Box! Box!
