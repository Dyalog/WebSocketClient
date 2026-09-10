## The Basic Steps

1. Write a function that will be called whenever your WebSocket receives a message - you'll assign the function name to the `OnWSReceive` setting of your client.
1. Create an instance of `WebSocketClient`.
1. Configure the instance.
1. Run `ws.Connect` to connect to the WebSocket server and start listening for messages.
1. Close the WebSocket

Each of these steps is described in more detail below.

### 1. Write a function for `OnWSReceive`.

A WebSocket exists to deliver messages to you, so the first thing to decide is what your application should do with a message when it arrives. `WebSocketClient`'s default behavior is to display received messages in the session prefixed by `>>> ` which can be useful when experimenting, but not much use in a real application. Instead you write a "hook" function and assign its name to the [`OnWSReceive`](settings-eventhooks.md#onwsreceive) setting. The hook is called on the listener thread for each message segment that arrives, with the client instance as its left argument and [`MsgState`](settings-websocket.md#msgstate) - the namespace in which `WebSocketClient` assembles the incoming message - as its right argument.

### 2. Create an instance of `WebSocketClient`.

Use the shared [`New`](public-methods.md#new) method (`ws←WebSocketClient.New args`) rather than `⎕NEW`, because `New` traps the errors that `⎕NEW` would signal. `args` may be:

- `''` - an instance with all settings at their default values.
- a namespace whose variables are the settings to apply, for example `(URL:'echo.websocket.org' ⋄ OnWSReceive:'OnMessage')`.
- a vector of settings in the positional order<br/>[`URL`](settings-connect.md#url) [`OnWSReceive`](settings-eventhooks.md#onwsreceive) [`OnWSUpgrade`](settings-eventhooks.md#onwsupgrade) [`Protocol`](settings-websocket.md#protocol) [`Headers`](settings-connect.md#headers) [`Params`](settings-connect.md#params)<br/>for example `'echo.websocket.org' 'OnMessage'`.

  If construction fails, `New` returns a namespace reporting the failure instead of an instance, so check its `rc` or `Connected` before going on. Setting [`Debug`](settings-connect.md#debug) to `1` has the error signalled instead.

###3. Configure the instance.
Any setting not supplied to `New` can be assigned directly to the instance at any time before `Connect` is called. For instance:

```
      ws.(URL OnWSReceive)←'wss://echo.websocket.org' 'OnMessage'
```

The [`ws.Config`](public-methods.md#config) method returns a 2-column matrix of every public field and its current value, which is a convenient way to check what you've set.

###4. Run `ws.Connect`.
[`Connect`](public-methods.md#connect) initializes Conga if that hasn't happened yet, builds and sends the WebSocket upgrade request from your settings, follows any redirections, and completes the handshake. It returns `(rc msg)` - `0 'Connected'` on success - and also leaves the result in the instance's `rc` and `msg` fields. On success it starts the listener thread, which sits in a Conga wait loop calling your `OnWSReceive` hook whenever a message is received until the WebSocket is closed. From then on you can [`Send`](public-methods.md#send) messages, and when you're done, [`Close`](public-methods.md#close) shuts the WebSocket down and stops the listener.

###5. Close the WebSocket

A WebSocket connection ends in one of two ways - your application closes it, or the
other end does. Either way the listener thread stops and
[`Connected`](settings-status.md#connected) drops back to `0`, which is the reliable
test of whether the WebSocket is still usable. Nothing else announces itself: a
connection that has gone away does not interrupt your code, and `Send` simply fails
rather than signalling an error.

```
      ws.Send 'anyone there?'
¯1 No client connection has been established
```

#### Closing from your side

[`Close`](public-methods.md#close) signals the listener thread to stop, closes the
Conga connection, and clears [`Connection`](settings-status.md#connection):

```
      ws.Close
0 Closed
```

Like `Connect`, `Close` returns `(rc msg)` and leaves the same pair in the instance's
[`rc`](settings-status.md#rc) and [`msg`](settings-status.md#msg) fields. It is safe
to call more than once - a second call returns `0 'Already closed'` - and an instance
that was never connected returns `0 'Not listening'`.

`Close` may take up to [`WaitTime`](settings-conga.md#waittime) milliseconds (5
seconds by default) to return, because that is how long the listener can be sitting
in a Conga `Wait` before it notices that it has been asked to stop. `Close` polls for
`WaitTime×1.1` milliseconds and then terminates the thread with `⎕TKILL`, so it
always returns; if a prompt shutdown matters more to your application than an idle
listener waking rarely, lower `WaitTime`.

Because `Close` stops the listener directly rather than by way of a Conga `Closed`
event, your [`OnClose`](settings-eventhooks.md#onclose) hook is **not** called - a
close you asked for is not news to your application.

Always close a connection you are finished with. It is tempting to assume that
expunging the instance is enough - the class has a destructor that closes the
connection and kills the listener - but the destructor does not run while the
listener thread is still going, and the listener is what keeps the instance alive.
Expunging the name therefore leaves you with an orphaned listener: a thread still
waiting on a live connection, belonging to an instance you no longer have a
reference to.

The reference is recoverable. `⎕INSTANCES` returns every live instance of the class,
so you can find the orphan and close it properly:

```
      ws.Connected
1
      ⎕EX 'ws'
1
      ⎕TNUMS              ⍝ the listener is still running
0 1
      inst←⎕INSTANCES #.WebSocketClient
      ≢inst
1
      (⊃inst).URL
wss://echo.websocket.org
      (⊃inst).Close
0 Closed
      ⎕TNUMS
0
```

Once `Close` has stopped the listener, nothing is holding the instance any longer and
it is discarded - the destructor runs, and `⎕INSTANCES` comes back empty.

If several instances are live, `⎕INSTANCES` gives you all of them, so read `URL` or
`Config` to tell them apart - or simply close the lot:

```
      {}(⎕INSTANCES #.WebSocketClient).Close
```

#### Closed by the server

A close can equally come from the other end - the server shutting down, an idle
timeout, or a proxy dropping the connection. The listener sees Conga's `Closed`
event, calls your `OnClose` hook if you have set one, and then terminates, setting
`Connected` to `0`. The event itself is left in
[`LastWaitResponse`](settings-status.md#lastwaitresponse) for inspection afterwards.

An application that needs to react to this - to reconnect, to warn the user, to stop
queueing work that can no longer be sent - either checks `Connected` before it relies
on the connection, or sets an `OnClose` hook:

```
     ∇ (rc msg)←client OnClosed waitData
[1]   ⍝ waitData is Conga's Wait result: (return code) (object name) 'Close' (data)
[2]    ⎕←'WebSocket to ',client.URL,' was closed by the server'
[3]    Reconnect←1   ⍝ ... and let the rest of the application know
[4]    (rc msg)←0 'Closed by server'
     ∇
```

```
      ws.OnClose←'OnClosed'
```

The hook is called on the listener thread, as `OnWSReceive` is, so the cautions in
[Two things to be careful about](#two-things-to-be-careful-about) apply to it too:
keep it short, and trap anything it might signal. [`OnError`](settings-eventhooks.md#onerror) is its
counterpart for a connection that Conga reports an error on - it is called in the
same way, and the listener ends after it in the same way.

After the server has closed the connection there is nothing left for `Close` to do,
and it reports `0 'Already closed'`.

#### Connecting again

A closed instance is not a spent one. Calling [`Connect`](public-methods.md#connect)
again negotiates a fresh connection using the same settings, so an instance can be
opened and closed as often as your application needs:

```
      ws.Close
0 Closed
      ws.Connect
0 Connected
```

`Connect` clears the status fields before it begins, so anything you want to know
about the connection that has just ended - [`ErrorInfo`](settings-status.md#errorinfo)
after a listener that stopped on an error, `LastWaitResponse` after a close - has to
be read before you reconnect.

## More about `OnWSReceive`

The functionality of `OnWSReceive` and the format of the message payload is entirely up to the specifications of your application. Let's say we've connected to a ficticious online chat service where the payload format is JSON and contains the message sender's id and the message they sent - something like `{"id":"Daffy","msg":"Quack!"}` and you want to display the sender and message to your APL session.

```
     ∇ client OnMessage state;json
[1]   ⍝ state is WebSocketClient's MsgState namespace containing buffer, payload, final, opcode
[2]   ⍝ client is a reference to the instance in case you need its fields or methods
[3]   ⍝ Since WebSocketClient reassembles fragmented messages for us, there is
[4]   ⍝ nothing to do until the message is complete
[5]    :If state.final
[6]        :Trap 11 ⍝ in case the message isn't JSON
[7]            json←⎕JSON state.buffer
[8]            ⎕←json.id,' says "',json.msg,'"'
[9]        :Else
[10]           ⎕←'*** JSON import failed on: "',state.buffer,'"'
[11]       :EndTrap
[12]   :EndIf
     ∇
```

Since `echo.websocket.org` simply echoes back what it receives, we can test our hook function...

```
      ws←WebSocketClient.New (URL:'echo.websocket.org' ⋄ OnWSReceive:'OnMessage')
      ws.Connect
0 Connected
*** JSON import failed on: "Request served by 4d896d95b55478"

```

The first response from `echo.websocket.org` is informational and not JSON. But now we can send a JSON payload to `echo.websocket.org` and it will echo it back...

```
      ws.Send ⎕JSON (id:'Daffy' ⋄ msg:'Quack!')
0
Daffy says "Quack!"
      ws.Close
0 Closed

```

!!! note "Partial Messages"
The WebSocket protocol allows for messages to be sent one or more _fragments_. See [Partial Messages](./partial.md) for more information.

### Two things to be careful about

**Trap your own errors.** The listener runs the whole of its wait loop inside a single
error trap, and your `OnWSReceive` hook runs inside that loop. An error in the hook
does not suspend the listener thread, but it does end it: the listener records `⎕DMX`
in the instance's `ErrorInfo` field, closes the Conga connection, and sets `Connected`
to `0`. The same applies to an error anywhere else on that thread - in `OnClose` or
`OnError`, in the UTF-8 translation of an incoming message, or in Conga itself.

So a hook that fails on one awkward message takes the WebSocket down with it, and if
you want to survive that message you have to trap it yourself:

```
     ∇ client OnMessage state
[1]   :If state.final
[2]       :Trap 0
[3]           Handle state.buffer
[4]       :Else
[5]           ⎕←'bad message ignored: ',⊃⎕DMX.DM
[6]       :EndTrap
[7]   :EndIf
     ∇
```

What the trap in the listener buys you is that a failure is reported rather than
silent. A stopped listener says why:

```
      ws.Connected
0
      ws.ErrorInfo.EM
DOMAIN ERROR
      ws.ErrorInfo.DM
 DOMAIN ERROR  OnMessage[3] json←⎕JSON state.buffer  ∧
```

`ErrorInfo` is `''` until something is caught, and `ws.ListenerThread∊⎕TNUMS` tells
you whether the listener is still running. Between them they distinguish the three
ways a listener can be gone: closed normally by the server, ended by an error with
`ErrorInfo` set, or - with [`Debug`](settings-connect.md#debug) non-zero, which turns
the trap off - suspended in the debugger, which is how you want to develop a hook in
the first place.

`MsgState` is left exactly as the failing hook saw it, since the reset that normally
follows a final segment is skipped, so it is worth inspecting alongside `ErrorInfo`
when working out what the hook choked on.

**Do not keep the `MsgState` reference.** It is one namespace reused for every
message, and `WebSocketClient` clears it as soon as your hook returns for a final
segment. A hook that queues the reference for another thread, or stores it for later,
will find it empty or holding some later message. Copy out what you need:

```
[3]        Queue,←⊂state.(buffer opcode)   ⍝ the values, not the namespace
```

## More about `Send`

[`Send`](public-methods.md#send) puts a message on the WebSocket. What the server
makes of it is between you and the server, but two things are decided by
`WebSocketClient`: whether the message goes out as text or as binary, and what you get
told about it.

### Text or Binary

The WebSocket protocol has two kinds of message, and the datatype of the argument
chooses between them - character data is sent as a text message (opcode `1`), integer
data as a binary one (opcode `2`):

```
      ws.Send 'hello ⍺⍵'      ⍝ text
0
      ws.Send 72 73 74        ⍝ binary
0
```

There is nothing to set and nothing to encode. Conga translates character data to
UTF-8 on the way out and back from UTF-8 on the way in, so a message with APL glyphs
in it needs no special handling at either end.

Binary data must be integers that fit in a byte. Anything else is refused by Conga
rather than by `WebSocketClient`, which is worth recognising when you see it:

```
      ws.Send 1.5 2.5
1004 Conga send failure: 1004
```

An empty message is legal, and is sent as an empty text message.

### Check the Result

`Send` returns `(rc msg)`, `0 ''` when the data went out:

```
      ws.Send 'hello'
0
```

Unlike [`Connect`](public-methods.md#connect) and [`Close`](public-methods.md#close),
`Send` does **not** leave its result in the instance's `rc` and `msg` fields - those
still describe the last connect or close - so the result has to be taken from `Send`
itself:

```
      ⎕←(rc msg)←ws.Send payload
```

The three failures worth handling separately are:

| Result                                             | Meaning                                               |
| -------------------------------------------------- | ----------------------------------------------------- |
| `¯1 'No client connection has been established'`   | There is no connection - it was closed, or never made |
| `1004 'Conga send failure: 1004'` (any Conga `rc`) | Conga refused or could not send the data              |
| `¯1 '... occurred trying to send'`                 | An APL error was trapped while sending                |

A `0` from `Send` means the data reached Conga, not that the server processed it or
agreed with it - a WebSocket send has no reply. If your protocol has one, it arrives
later, on the listener thread, through your `OnWSReceive` hook.

### Sending a Message in Pieces

`Send` takes an optional second element saying whether the data completes the message,
which lets a large or open-ended message be sent as a sequence of fragments:

```
      ws.Send ('part one ' 0)   ⍝ 0 - more to come
0
      ws.Send ('part two' 1)    ⍝ 1 - that was the last of it
0
```

The receiving end sees one message, `'part one part two'`. `WebSocketClient` sets the
continuation opcodes itself, which is why every fragment of one message has to be of
the same datatype:

```
      ws.Send ('abc' 0)
0
      ws.Send (1 2 3) 1
¯1 Datatype is not the same as previous fragment (1)
```

Note that the message is still open after that failure - the fragment was rejected,
not the message. [Sending Partial Messages](partial.md#sending-partial-messages)
covers this, and how to abandon a message you have started, in more detail.
