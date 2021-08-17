# Minmux

A minimalistic multiplexing specification.

## What, Why, and How?

Establishing reliable connections between programs over a network can be rather expensive. TCP requires a three-way handshake, setting up encryption requires additional roundtrips, and the whole thing might be routed through [tor](https://www.torproject.org). In addition to connection setup time, each tcp connection requires keeping track of a bunch of state, usually tracked by the operating system. For these reasons, it is often desirable to create at most one tcp connection between two programs.

Within such a single connection, there might however be data streams that are logically independent. If one of them is closed, the others should continue working. If one of them needs to be throttled because the receiving program can't keep up processing the data, that doesn't necessarily mean that the other data streams have to be throttled as well. And finally, sending large amounts of data along one data stream should not mean that the others become unusable until the data transfer is done. Minmux provides a generic approach for maintaining separate data streams that have independent flow control (backpressure), don't starve each other, and can be opened and closed without interfering.

The actual mechanisms for providing these features are fairly simple and lightweight. Minmux provides a namespace of unidirectional logical streams, each with a numeric identifier. All data is encapsulated in packets with a light header, this header addresses the stream to which the data pertains. There are different packet types, one for each possible action, each pertaining to a single logical stream. The actions come pairs; for each action that can be applied to an outgoing unidirectional logical stream, there is a corresponding action that can be performed by the other peer on the incoming end.

Flow control works with a credit-based scheme: Initially, no data may be sent down a stream. An endpoint can give some *credit* to the other endpoint for a stream,he credit indicates how much data may be written. Once credit reaches zero, no more data may be sent until new credit has been received. The unit of credit is not specified, it could be individual bytes, or higher-level, application-specific concepts.

For minmux to function correctly, it is mandatory that credit is only given for data that can be handled immediately. Since all streams are multiplexed over a single actual data stream, all received data must be processed immediately, lest it block some supposedly independent stream. The credit mechanism makes sure that the other endpoint never overwhelms the receiver's ability to handle the incoming data quickly enough as to not be delaying other streams. In practice, this usually means that the incoming data is simply copied somewhere else in memory for later processing, so that further packets on the underlying connection can be handled immediately.

Besides issuing credit exclusively up to an amount of data that can actually be handled immediately, there is a second factor for maintaining correctness: the minmux packets must be small enough relative to the bandwidth of the underlying channel. Suppose the underlying connection only transferred one byte per second, but you wrote a packet of 1800 bytes to a stream. This would block all other information for half an hour! This example is obviously exaggerated, but the same applies in realistic situations. Writing a few contiguous gigabytes to a substream over a tcp connection will also delay the other streams. Protocols built upon minmux need to ensure on their own that this does not happen.

Small packets alone don't guarantee fairness, sending a large number of small packets to the same stream could still starve out other streams. A minmux implementation is thus required to fairly interleave packets that pertain to different streams, the simplest solution being round-robin.

Note that minmux does not provide facilities for negotiating which stream should be used by whom or for what, all it offers is a flat namespace of streams. The intention is that other protocols can specify that they need, for example, two bidirectional streams and one unidirectional stream, the first two controlling transmission by byte, and the latter controlling transmissions by some specific encoding of a valid position in the game of go. Then a static mapping from these high-level streams to the logical streams of minmux can be specified ahead of time. Any failure to adhere to that specification, e.g., data being written to an unassigned stream, should be treated as an error and result in the connection being terminated, because clearly either an endpoint is faulty or the endpoints do not agree on a single protocol.

It is of course possible to multiplex dynamically assigned streams, but the mapping to minmux streams must be done in a higher-level component of the protocol stack. If you choose to do so, know that a stream id is safe to reuse if and only if the sending endpoint has promised to not send anything again on this stream *and* the receiving endpoint has indicated that it will not grant anymore credit for this stream.

## Specifics

The protocol assigns to each endpoint a role in order to break symmetry: the endpoint that initiated the connection is the *proactive* endpoint, the other one is the *reactive* endpoint.

There are `2^64` unidirectional logical streams, each being identified by a number between `0` and `2^64 - 1`. The proactive endpoint reads from even-numbered streams and writes to odd-numbered streams, the reactive endpoint reads from odd-numbered streams and writes to even-numbered streams.

These are the different kinds of packets:

```rust
enum Packet {
  // The first pair of packets handles the writing of data.

  // Allow the other endpoint to write more items to some stream.
  GiveCredit {
    id: u64, // The stream they are allowed to write more to.
    amount: u64, // How many more items they are allowed to write.
  },

  // Write some items to a stream.
  Write {
    id: u64, // The stream to write to.
    amount: u64, // How many items will be written.
    data: Bytes, // The encoding of the items.
  },
  // You are not allowed to write more items to a stream than you have pending
  // credit on that stream. When writing items, reduce your available credit
  // on the upstream by the number of items that were written.
  // When you receive a `Write` packet whose `amount` exceeds the credit that
  // was available on the stream, this is an error and the connection should be
  // terminated.

  // The next pair of packets allows to communicate that a stream will not be
  // used anymore, allowing both endpoints to reclaim resources.

  // Indicates that the endpoint will not give any more credit than a certain
  // limit.
  StopRead {
    id: u64, // The stream on which reading will stop.
    amount: u64, // How much more credit this endpoint might still give at most.
  },
  // If this endpoint sends anymore packets pertaining to the stream after the
  // maximal amount of credit has been given, that is an error (unless the
  // stream id has been dynamically reassigned, in which case the new stream
  // should be considered to be a completely separate object from the old stream
  // that just happened to have had the same id).
  //
  // An endpoint is allowed to tighten the deadline, e.g., first sending a
  // `StopRead` with an amount of `42`, then giving `3` credit, and then sending
  // a `StopRead` with an amount of `7`. It is however not allowed to increase
  // the amount again. In particular, once the point at which it promised to
  // stop interacting with a stream has been reached, there is no way of
  // resuming interaction again.

  // Indicates that the endpoint will not send anymore items than a certain
  // limit.
  StopWrite {
    id: u64, // The stream on which writing will stop.
    amount: u64, // How many more items this endpoint might still send at most.
  },
  // This works analogously to `StopRead`: The endpoint may not write after it
  // has written the specified amount of items, and it may reduce the number of
  // outstanding items with subsequent `StopWrite` packets but it cannot extend
  // it.

  // The remaining packets are not critical to the correctness of a minmux
  // session, a minimal implementation is free to ignore them. They do however
  // smooth over some resource management questions and can help with the
  // efficiency of a system.

  // Forgo some credit without using it to send items.
  ForgoCredit {
    id: u64, // The stream on which to decrease the credit this endpoint has.
    amount: u64, // How much credit to give up on.
  },

  // Kindly ask the other endpoint to forgo some of its credit.
  Oops {
    id: u64, // The stream on which to give up credit.
    maximum: u64, // How much credit they are fine to keep.
  },
  // Once credit has been issued, an endpoint must uphold the promise of being
  // able to receive that many items. If this is inconvenient, tough.
  // But perhaps the other endpoint is friendly enough to voluntarily forgo its
  // credit, it certainly cannot hurt to ask.
  // In general, an endpoint should answer with a `ForgoCredit` packet, because
  // the overall system performance probably benefits. It is however also free
  // to ignore it - in particular it may have concurrently performed writes
  // using that credit.

  // The last two packets allow to inform the other endpoint of minimum
  // resource requirements, allowing it to prepare for the coming load.
  //
  // Neither packet has any direct impact on the protocol, they are merely
  // informative. An endpoint is not required to actually consume the requested
  // resources, and it can update requests at any point to any value.

  // Inform the other endpoint that it will need to issue at least some more
  // credit so that this endpoint can meaningfully complete its work.
  RequestCredit {
    id: u64, // The stream on which more credit is required.
    base: u64, // The available credit as of sending this packet.
    amount: u64, // How much more credit is required.
  },

  // Informed the other endpoint that it will need to write at least some more
  // items so that this endpoint can meaningfully complete its work.
  RequestItems {
    id: u64, // The stream on which more items are required.
    base: u64, // The amount of unconsumed credit the other endpoint has as of
               // sending this packet.
    amount: u64, // How many more items are required.
  }
}
```

## Encodings

Each packet begins with a header that encodes the packet type and the id of the stream the packet pertains to. The header begins with a single byte. If the last six bits are all set to `1`, then this byte is followed by a [VarU64](https://github.com/AljoschaMeyer/varu64) indicating the stream id. Otherwise, the last six bits encode the stream id directly. The packet type depends on the first two bits and the parity of the stream id, as given in the following table. A `0` in the "Stream id parity" indicates that the parity is even if the packet is encoded by the proactive endpoint and odd if the packet is encoded by the reactive endpoint. A `1` indicates the opposite parity. If this seems confusing, remember that the parity of a stream id and the role of the endpoint determine whether that endpoint reads from or writes to that stream.

| First two bits | Stream id parity | Packet |
|---|---|---|
| 00 | 0 | GiveCredit |
| 00 | 1 | Write |
| 01 | 0 | StopRead |
| 01 | 1 | StopWrite |
| 10 | 0 | Oops |
| 10 | 1 | ForgoCredit |
| 11 | 0 | RequestItems |
| 11 | 1 | RequestCredit |

Each such header is then followed by packet-specific data:

| Packet | Additional data |
|---|---|
| GiveCredit | The `amount` of credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| Write | The `amount` of items, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64), followed by the encoding of that many items. |
| StopRead | The `amount` of items, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| StopWrite | The `amount` of credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| Oops | The `maximum` of credit to retain, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| ForgoCredit | The `amount` of credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| RequestItems | The `base` of unconsumed credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64), followed by the `amount` of requested items, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| RequestCredit | The `base` of available credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64), followed by the `amount` of requested credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
