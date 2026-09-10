These settings relate to the WebSocket protocol itself - the two `Sec-WebSocket-` headers that shape the upgrade request, and the parsed upgrade response that the server sends back.

### `Protocol`

|--|--|
|Description|The value(s) for the `Sec-WebSocket-Protocol` header, used to request one or more application sub-protocols from the server.|
|Default|`''`|
|Example(s)|`ws.Protocol←'chat, superchat'`|
|Details|If non-empty, `Connect` merges `Protocol` into the request headers (via the same add-unless-already-defined logic as `AddHeader`) when building the WebSocket upgrade request; leading, trailing, and redundant embedded blanks are removed first. Setting a `Sec-WebSocket-Protocol` header directly via `Headers`/`AddHeader`/`SetHeader` takes priority over `Protocol`. `Protocol` may also be supplied as the fourth element of the constructor's argument vector - `⎕NEW WebSocketClient (url onWSReceive onWSUpgrade protocol)`.|
|Note|The server selects at most one of the offered sub-protocols and names it in the `Sec-WebSocket-Protocol` header of its response - inspect `WSUpgradeResponse.headers` (or `GetHeader` in a hook function) to find out which, if any, was accepted.|

### `Extensions`

|--|--|
|Description|The value(s) for the `Sec-WebSocket-Extensions` header, used to request WebSocket extensions (such as `permessage-deflate`) from the server.|
|Default|`''`|
|Example(s)|`ws.Extensions←'permessage-deflate'`|
|Details|If non-empty, `Connect` merges `Extensions` into the request headers the same way as `Protocol` when building the WebSocket upgrade request. Setting a `Sec-WebSocket-Extensions` header directly via `Headers`/`AddHeader`/`SetHeader` takes priority over `Extensions`.|
|Note|Requesting an extension does not implement it - `WebSocketClient` neither negotiates nor applies extension semantics itself. Only offer an extension if the application is prepared to deal with the messages the server then sends.|

### `WSUpgradeResponse`

|--|--|
|Description|A namespace holding the parsed server response to the WebSocket upgrade request.|
|Default|`''`|
|Example(s)|`ws.WSUpgradeResponse.status` `ws.WSUpgradeResponse.headers ws.GetHeader 'Sec-WebSocket-Protocol'`|
|Details|Set whenever the server responds to the upgrade request, whether `AutoUpgrade` is `1` or `0`, and regardless of whether [`OnWSUpgrade`](settings-eventhooks.md#onwsupgrade)/[`OnWSResponse`](settings-eventhooks.md#onwsresponse) is set. It is cleared, like the [status fields](settings-status.md), at the start of each `Connect`, so it remains `''` if the current attempt received no upgrade response. The namespace has the elements `version`, `status`, `message`, `headers`, and `payload`. `status` is the HTTP status as an integer, for example `101`; `headers` is a 2-column matrix of header names and values, suitable as the left argument to `GetHeader`; `version`, `message`, and `payload` are character vectors.|
|Note|`WSUpgradeResponse` is also the right argument passed to `OnWSUpgrade`/`OnWSResponse`, so a hook function does not need to read the field to see the response. It is retained on the instance so the response can still be examined after `Connect` returns.|

### `MsgState`

|--|--|
|Description|A namespace holding the message currently being received - the buffer in which `WebSocketClient` reassembles fragmented messages.|
|Default|`(buffer:'' ⋄ payload:'' ⋄ final:¯1 ⋄ opcode:¯1)`|
|Example(s)|`ws.MsgState.buffer` `ws.MsgState.final`|
|Details|A WebSocket message may be sent as a series of fragments, and Conga reports each one separately. `WebSocketClient` accumulates them here so that hook functions do not have to. The namespace has four elements: `buffer` is everything received so far for this message, including the current segment; `payload` is the current segment alone; `final` is `1` when the segment just received completes the message and `0` when more is coming; and `opcode` is the message's type, taken from its first frame - `1` for text, `2` for binary. `MsgState` is the right argument passed to the [`OnWSReceive`](settings-eventhooks.md#onwsreceive) hook.|
|Note|The same namespace is reused for every message. Once a message is complete - and, if a hook is set, once that hook has returned - `buffer` and `payload` are reset to `''` and `final` and `opcode` to `¯1`. Between messages, therefore, `final` of `¯1` means "no message in progress". `Connect` clears the same four elements at the start of each attempt. Copy anything you need to keep rather than retaining the reference.|
