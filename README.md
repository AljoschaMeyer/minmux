# Minmux

A minimalistic multiplexing specification.

## What, Why, and How?

Establishing reliable connections between programs over a network can be expensive. TCP requires a three-way handshake, setting up encryption requires additional roundtrips, and the whole thing might be routed through [tor](https://www.torproject.org). In addition to connection setup time, each tcp connection requires keeping track of state, usually managed by the operating system. For these reasons, it is often desirable to create at most one tcp connection between two programs.

Within such a single connection, there might however be data streams that are logically independent. If one of them is closed, the others should continue working. If one of them needs to be throttled because the receiving program cannot keep up processing the data, that does not imply that the other data streams have to be throttled as well. And finally, writing large amounts of data to one stream should not mean that the other streams become unusable until the data transfer is done. Minmux provides a generic approach for maintaining separate logical data streams with independent flow control over a single, byte-oriented connection.

To do so, minmux provides a namespace of unidirectional logical streams, each with a numeric identifier. All data is encapsulated in packets with a light header; this header addresses the stream to which the data pertains. There are different packet types, one for each possible action, each pertaining to a single logical stream. The actions come in pairs; for each action a sending endpoint can apply to an outgoing unidirectional logical stream, there is a corresponding action the receiving endpoint can perform on the incoming end.

Flow control works with a credit-based scheme. A receiving endpoint can grant stream-specific *credit* to the sending endpoint. Writing data to a stream consumes credit; while credit for a stream is zero, the sending endpoint may not write to that stream. The unit of credit is not specified, credit could represent the permission to write individual bytes, or higher-level, application-specific concepts.

For minmux to function correctly, it is crucial that endpoints grand credit only for data which they can immediately process — granting credit is a binding promise that up to a certain amount of data will be read when it arrives. Because all logical streams are multiplexed over a single underlying stream, all received data must be processed immediately, lest it block some supposedly independent stream. The credit mechanism makes sure that the sending endpoint never overwhelms the receiver's ability to handle the incoming data sufficiently fast to not delay other streams. In practice, this usually means that incoming data is simply copied elsewhere in memory for later processing, so that further packets on the underlying connection can be handled without delay.

A minmux implementation must ensure that many writes to some logical stream do not starve out writes to other streams by fairly interleaving packets pertaining to different streams. A single write of a large amount of data could however still block other streams. Minmux does not provide any mechanism guarding against this situation, it is the responsibility of writers to split data into small (relative to the bandwidth of the underlying communication channel) chunks so that the fair interleaving suffices to maintain independence of all streams.

Note that minmux does not provide facilities for negotiating which stream should be used by whom or for what, all it offers is a flat namespace of streams. The intention is that other, higher-level protocols can specify that they need, for example, two bidirectional streams and one unidirectional stream, the first two controlling transmission by byte, and the latter controlling transmissions by some specific encoding of a valid position in the game of go. Then a static mapping from these high-level streams to the logical streams of minmux can be specified ahead of time. Any failure to adhere to that specification, e.g., data being written to an unassigned stream, should be treated as an error and result in the connection being terminated, because clearly either an endpoint is faulty or the endpoints do not agree on a single protocol.

It is of course possible to multiplex dynamically assigned streams, but the mapping to minmux streams must be done in a higher-level component of the protocol stack. If you choose to do so, know that a stream id is safe to reuse if and only if the sending endpoint has promised to not send anything again on this stream *and* the receiving endpoint has indicated that it will not grant any more credit for this stream.

## Specifics

The protocol assigns to each endpoint a role in order to break symmetry: the endpoint that initiated the connection is the *proactive* endpoint, the other one is the *reactive* endpoint.

There are `2^64` unidirectional logical streams, each identified by a number between `0` and `2^64 - 1`. The proactive endpoint reads from even-numbered streams and writes to odd-numbered streams, the reactive endpoint reads from odd-numbered streams and writes to even-numbered streams.

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
    data: Bytes, // The encoding of the items.
  },
  // You are not allowed to write more items to a stream than you have pending
  // credit on that stream. When writing items, reduce your available credit
  // on the stream by the number of items that were written.
  //
  // Minmux does not specify how a receiving endpoint can determine the number
  // of items that have been written, an endpoint simply receives a bunch of
  // bytes. This conversion must be specified in a higher level in the protocol
  // stack.
  //
  // When you receive a `Write` packet sending more items than there was credit
  // available on the stream, this is an error and the connection should be
  // terminated.

  // The next pair of packets allows to communicate that a stream will not be
  // used anymore, allowing both endpoints to reclaim resources.

  // Indicate that the endpoint will not give more credit than a certain limit.
  StopRead {
    id: u64, // The stream on which reading will stop.
    amount: u64, // How much more credit this endpoint will still grant at most.
  },
  // If an endpoint grants more credit for a stream then a previously announced
  // limit, that is an error (unless the stream id has been dynamically
  // reassigned, in which case the new stream should be considered to be a
  // completely separate object from the old stream which just happens to have
  // the same id).
  //
  // An endpoint is allowed to tighten the deadline, e.g., first sending a
  // `StopRead` with an amount of `42`, then giving `3` credit, and then sending
  // a `StopRead` with an amount of `7`. It is however not allowed to increase
  // the amount again. In particular, once the point at which it promised to
  // stop interacting with a stream has been reached, there is no way of
  // resuming interaction again.

  // Indicates that the endpoint will not write more items than a certain limit.
  StopWrite {
    id: u64, // The stream on which writing will stop.
    amount: u64, // How many more items this endpoint will still write at most.
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
  // able to read that many items. If this is inconvenient, tough.
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

  // Inform the other endpoint that it will need to write at least some more
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
| Write | The data to write. |
| StopRead | The `amount` of items, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| StopWrite | The `amount` of credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| Oops | The `maximum` of credit to retain, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| ForgoCredit | The `amount` of credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| RequestItems | The `base` of unconsumed credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64), followed by the `amount` of requested items, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |
| RequestCredit | The `base` of available credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64), followed by the `amount` of requested credit, encoded as a [VarU64](https://github.com/AljoschaMeyer/varu64). |

Note that the header of a `Write` packet is followed by bytes whose format is not governed by the minmux specification. Determining when the packet ends and how much credit it consumes must be done according to some higher-level specification.
