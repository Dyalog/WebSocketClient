## Partial Messages

A WebSocket message does not have to arrive, or be sent, in one piece. The protocol
allows a message to be split into a sequence of _fragments_, each carried in its own
frame, with only the last one marked as "final". `WebSocketClient` exposes this
directly rather than hiding it: the [`OnWSReceive`](settings-eventhooks.md#onwsreceive)
hook is called once per segment, and [`Send`](public-methods.md#send) lets you mark a
message as incomplete.

A message arrives in pieces only because the sender chose to fragment it - a common
way to stream a large or open-ended payload without having to know its length up
front. Fragmentation is a decision made by whoever sent the message, so whether you
ever see a partial one depends entirely on the server you are talking to.

### Receiving Partial Messages

`OnWSReceive` is called once per segment, with
[`MsgState`](settings-websocket.md#msgstate) as its right argument:

- `MsgState.buffer` is everything received for this message so far, including the
  segment that has just arrived.
- `MsgState.payload` is that segment on its own.
- `MsgState.final` is `1` if the message is now complete, `0` if more is coming.
- `MsgState.opcode` is the message's type - `1` for text, `2` for binary - taken from
  its first frame.

Reassembly is done for you, so a hook that wants nothing but whole messages only has
to wait for `final` and read `buffer`:

```
     ∇ client OnMessage state
[1]   ⍝ ignore everything until the message is complete
[2]    :If state.final
[3]        Handle state.buffer
[4]    :EndIf
     ∇
```

Each client instance has its own `MsgState`, so several connections running at once
need no special handling - each accumulates independently.

#### Consuming a message as it arrives

The reason to look at the non-final segments is to avoid holding a large message in
memory, or to start work on it before it has finished arriving. A hook can deal with
each segment and then empty the buffer; `WebSocketClient` appends the next segment to
whatever it finds there, so draining it keeps the message from accumulating:

```
     ∇ client OnChunk state
[1]   ⍝ deal with each segment as it arrives rather than buffering the whole message
[2]    Received+←≢state.buffer       ⍝ ... or write it out, feed a parser, and so on
[3]    state.buffer←0⍴state.buffer   ⍝ drop what we have dealt with
[4]    :If state.final
[5]        ⎕←'message complete - ',(⍕Received),' elements'
[6]        Received←0
[7]    :EndIf
     ∇
```

Read `buffer` rather than `payload` here: after the first drain the two are the same,
but on any segment you have not drained, `buffer` is the part you still owe work.
Emptying it with `0⍴` rather than `''` preserves the datatype, so the technique works
for binary messages as well as text.

!!! note "Know Your Server"

> Character payloads are translated from UTF-8 as each segment arrives, before it is
> added to the buffer. A server is permitted by RFC 6455 to split a text message in
> the middle of a multi-byte character, and if one does, that translation fails with a
> `DOMAIN ERROR`, which ends the listener and leaves the error in `ErrorInfo` - see
> [Two things to be careful about](userguide.md#two-things-to-be-careful-about) in the
> Usage Guide. In practice servers fragment text on
> character boundaries; if you are dealing with one that does not, have the server
> send binary (`opcode` `2`) messages and do the UTF-8 translation yourself once the
> message has been reassembled.

### Sending Partial Messages

`Send` takes an optional second element saying whether the data completes the message:

```
      ws.Send data            ⍝ a complete message - final defaults to 1
      ws.Send data final      ⍝ final←0 leaves the message open
```

Sending `final←0` leaves the message open; each subsequent `Send` appends another
fragment, and the one you send with `final←1` closes it. `WebSocketClient` tracks this
for you and sets the WebSocket opcodes itself - the first fragment is sent as text or
binary, and the rest as continuations - so all you have to supply is the data and the
flag.

To stream a large payload in fixed-size chunks:

```
     ∇ (rc msg)←ws SendChunked payload;size;chunk
[1]   ⍝ send payload as a series of fragments of at most 32768 elements
[2]    (rc msg)←0 ''
[3]    size←32768
[4]    :While size<≢payload                    ⍝ more than one fragment still to go?
[5]        (chunk payload)←(size↑payload)(size↓payload)
[6]        :If 0≠⊃(rc msg)←ws.Send chunk 0     ⍝ 0 - the message continues
[7]            {}ws.Send''1                    ⍝ abandon the message
[8]            :Return
[9]        :EndIf
[10]   :EndWhile
[11]   (rc msg)←ws.Send payload 1              ⍝ 1 - the last fragment
     ∇
```

**Check the result of every fragment.** Unlike `Connect` and `Close`, `Send` does not
leave its result in the instance's `rc` and `msg` fields, so the returned `(rc msg)` is
the only report you get. A failure partway through leaves the message half-sent.

**All fragments of a message must be the same datatype.** Character data is sent as a
text message and integer data as a binary one; mixing them within a message is
rejected with `'Datatype is not the same as previous fragment (...)'` and nothing is
sent. Note that you do not need to UTF-8 encode character data yourself - Conga does
that as it sends each fragment.

**Nothing else can go out on that connection until the message is closed.** A
fragmented message owns the connection from its first fragment to its last: anything
else you send in between becomes part of it. In particular, `Send` keeps its
fragmentation state on the instance, so two threads sending on the same client at the
same time will interleave into a single garbled message. If your application sends
from more than one thread, serialize the sends - or give each thread its own client
instance.

**Close out a message you have abandoned.** If you stop partway through - because a
fragment failed, or because the data ran out - the instance still believes a message is
open, and the next `Send` will be treated as a continuation of it. Sending an empty
final fragment ends the message and clears that state. The empty fragment still has to
match the datatype of the fragments already sent, so use `''` to close out a text
message and `⍬` to close out a binary one:

```
      ws.Send ''1     ⍝ end an abandoned text message
      ws.Send ⍬ 1     ⍝ end an abandoned binary message
```
