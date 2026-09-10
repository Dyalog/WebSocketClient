## "Hook" Functions

A hook lets your application replace `WebSocketClient`'s default behavior for a particular WebSocket event. Each field holds the **name** of an APL function, as a character vector - not a reference to one. Setting a hook to `''` (the default) restores the built-in behavior for that event.

The name is resolved when the event occurs, not when the field is set, by evaluating it in `##` - the namespace in which the `WebSocketClient` class resides. The function must therefore be visible from there; a name like `'utils.OnMessage'` is resolved relative to `##` as well. A name that cannot be found produces a `VALUE ERROR` at event time.

All hook functions are called dyadically, with `client` as the left argument - a reference to the `WebSocketClient` instance itself, passed so that the hook function has access to the instance's public fields and methods. The right argument is the data associated with the Conga event. All hooks except `OnWSReceive` return a 2-element `(rc msg)` result, where `rc` of `0` means "carry on".

`OnWSUpgrade` and `OnWSResponse` are called on the thread that called `Connect`, and `Connect` traps errors - a `⎕SIGNAL` or unexpected error inside either hook makes `Connect` return `rc` of `¯1` and `msg` of `'Unexpected ... '` rather than suspending. `OnWSReceive`, `OnClose`, and `OnError` are called on the listener thread (see `ListenerThread`), whose wait loop is wrapped in a single error trap. An error in any of the three does not suspend the thread, but it does end the listener: `⎕DMX` is recorded in `ErrorInfo`, the connection is closed, and `Connected` is set to `0`. A hook that needs to survive a message it cannot handle must therefore trap the error itself. Set [`Debug`](settings-connect.md#debug) to `1` while developing a hook to disable trapping everywhere, so that errors suspend where they occur rather than being reduced to a field.

### `OnWSUpgrade`

|--|--|
|Description|The name of a function to be called when `AutoUpgrade` is `1` and the server has returned a WebSocket upgrade response, giving the application a chance to validate or reject the upgrade before `Connect` completes.|
|Default|`''`|
|Example(s)|`ws.OnWSUpgrade←'OnUpgrade'`|
|Signature|``|
|Details|Called as `(rc msg)←client OnWSUpgrade WSUpgradeResponse`. The right argument is the namespace described under [`WSUpgradeResponse`](settings-websocket.md#wsupgraderesponse) - `WebSocketClient` parses the server's raw response before calling the hook, so `version`, `status`, `message`, `headers`, and `payload` are already split out. Return `rc` as `0` to accept the upgrade; return a non-zero `rc` and a useful `msg` to reject it, in which case `Connect` closes the connection, terminates the listener thread, sets `Connected` to `0`, and returns the hook's `rc` and `msg`. If `OnWSUpgrade` is not set, the upgrade is accepted unconditionally.|
|Note|Because `AutoUpgrade` is `1`, the WebSocket is already upgraded when `OnWSUpgrade` is called. Returning a non-0 return code will close the connection.|

### `OnWSResponse`

|--|--|
|Description|The name of a function to be called instead of `OnWSUpgrade` when `AutoUpgrade` is `0`, giving the application a chance to inspect the server's (unaccepted) HTTP response before the handshake is completed.|
|Default|`''`|
|Example(s)|`ws.OnWSResponse←'OnResponse'`|
|Signature|``|
|Details|Called as `(rc msg)←client OnWSResponse WSUpgradeResponse`, with the same, already-parsed right argument as `OnWSUpgrade`. Return `0` to let `WebSocketClient` perform the `WSAccept` call - if Conga rejects it, `Connect` fails with Conga's return code and a `msg` of `'Conga WSAccept failed: ...'`. Return any non-zero `rc` (with a useful `msg`) to reject the connection. If `OnWSResponse` is not set, the `WSAccept` call is performed by default.|
|Note|Unlike `OnWSUpgrade`, the WebSocket is not connected when `OnWSResponse` is called. Returning a non-0 return code will close the connection without having upgraded it to a WebSocket connection. Leave the `WSAccept` call to `WebSocketClient` - performing it in the hook and then returning `0` causes it to be issued twice.|

### `OnWSReceive`

|--|--|
|Description|The name of a function to be called for each received WebSocket message segment, replacing the default behavior of displaying complete messages in the session prefixed by `>>> `.|
|Default|`''`|
|Example(s)|`ws.OnWSReceive←'OnMessage'`|
|Signature|``|
|Details|Called as `client OnWSReceive MsgState`, where [`MsgState`](settings-websocket.md#msgstate) is the namespace in which `WebSocketClient` assembles the message currently being received - `buffer` is everything received so far, `payload` is just this segment, `final` says whether the message is now complete, and `opcode` is `1` for a text message or `2` for a binary one. Character data is decoded from UTF-8 before the hook is called. The hook returns no result. If `OnWSReceive` is not set, `WebSocketClient` displays each complete message in the session with the prefix `>>> `.|
|Note|The hook is called for **every** segment, not only the last one, so a hook that wants whole messages must test `MsgState.final` and ignore the rest. Reassembly itself is done for you - `MsgState.buffer` already holds the complete message when `final` is `1`.|
|Note|`MsgState` is a single namespace reused for every message, and `WebSocketClient` clears it as soon as the hook returns for a final segment. Take a copy of anything the hook needs to keep or hand to another thread; retaining the reference itself will not work.<br/><br/>As `MsgState` is a namespace, you can also stash any additional message-related information within.|

### `OnClose`

|--|--|
|Description|The name of a function to be called when the WebSocket is closed by the server or the network, allowing the application to supply its own return code and message for the close event.|
|Default|`''`|
|Example(s)|`ws.OnClose←'OnClosed'`|
|Signature|``|
|Details|Called as `(rc msg)←client OnClose waitData`. `waitData` is the result of Conga's `Wait` function and is a 4-element array of [1] the Conga return code, [2] the Conga object name of the connection, [3] the event (`'Close'`), [4] data, if any - the same value left in `LastWaitResponse`. Once the hook returns, the listener closes the Conga connection and terminates, and `Connected` is set to `0`. If `OnClose` is not set, the default result is `0 'WebSocket Closed'`.|
|Note|`OnClose` reports a close initiated by the other end. Calling `Close` yourself signals the listener to stop directly, so `OnClose` is not called in that case.|

### `OnError`

|--|--|
|Description|The name of a function to be called when Conga reports a WebSocket error while listening.|
|Default|`''`|
|Example(s)|`ws.OnError←'OnErr'`|
|Signature|``|
|Details|Called as `(rc msg)←client OnError waitData`. `waitData` is the result of Conga's `Wait` function and is a 4-element array of [1] the Conga return code, [2] the Conga object name of the connection, [3] the event (`'Error'`), [4] data, if any. As with `OnClose`, the listener closes the connection and terminates once the hook returns. If `OnError` is not set, the default result is `0 'WebSocket Error: ',⍕waitData`.|
|Note|`OnError` covers errors reported on an established WebSocket. Errors raised while connecting are reported through `Connect`'s own `rc` and `msg` instead.|
