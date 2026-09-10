These fields are set by `WebSocketClient` rather than by you: they report what happened during `Connect`, what the listener thread is doing, and what the server said. They are public so that they can be read at any time - `Connect` and `Close` return the important ones as their result as well, but the fields remain available afterwards, which is where you look when something did not work.

`Connect` clears every one of them back to its default before it begins, so what you find in them always describes the most recent attempt and never a previous one. Nothing is cleared when a connection ends, so the fields survive for inspection after `Close` or after the listener has stopped.

Three further fields are cleared by `Connect` in the same way but are documented alongside the settings they belong with: [`MsgState`](settings-websocket.md#msgstate), [`WSUpgradeResponse`](settings-websocket.md#wsupgraderesponse), and [`ProxyResponse`](settings-proxy.md#proxyresponse). The [`ListenerThread`](public-methods.md#listenerthread) property is described with the public methods.

### `rc`

|--|--|
|Description|The return code of the last `Connect` or `Close`.|
|Default|`¯1`|
|Example(s)|`:If 0≠ws.rc ⋄ ⎕←ws.msg ⋄ :EndIf`|
|Details|`0` means the operation succeeded. `Connect` sets `rc` to `¯1` for the failures it detects itself, or to the Conga return code when a Conga call is what failed; `Close` sets it to `0`. Both also return `(rc msg)` as their result, so assigning that result is usually more convenient than reading the fields.|
|Note|[`Send`](public-methods.md#send) does **not** update `rc` and `msg` - its returned `(rc msg)` is the only report of a failed send. See [Sending Partial Messages](partial.md#sending-partial-messages).|

### `msg`

|--|--|
|Description|The message accompanying `rc`, explaining what happened.|
|Default|`''`|
|Example(s)|`ws.msg`<br/>`Connected`|
|Details|`'Connected'` after a successful `Connect` and `'Closed'` after a successful `Close`. On failure it describes the problem - for example `'Conga failed to connect to "example.com": ...'`, or `'Unexpected DOMAIN ERROR at OnUpgrade[4]'` when a hook called from `Connect` signalled an error and [`Debug`](settings-connect.md#debug) is `0`. `Connect` also returns `0 'Already connected'` if the instance already has a live connection, in which case nothing is re-negotiated.|

### `Connected`

|--|--|
|Description|Whether the WebSocket is currently connected.|
|Default|`0`|
|Example(s)|`:If ws.Connected ⋄ ws.Send data ⋄ :EndIf`|
|Details|Set to `1` once the upgrade handshake has completed and the listener thread has been started, and back to `0` by the listener as it terminates - whether it stopped because the server closed the connection, because `Close` asked it to, or because an error ended it. `Connect` resets `Connected` to `0` before each attempt.|
|Note|`Connected` is the reliable test of whether the WebSocket is usable, but it is the listener that clears it. In the rare case where `Close` has to `⎕TKILL` a listener that did not stop within `WaitTime×1.1` milliseconds, the field is left as it stood; `ws.ListenerThread∊⎕TNUMS` is then the better check.|

### `Connection`

|--|--|
|Description|The name of the Conga object for this connection.|
|Default|`''`|
|Example(s)|`ws.LDRC.Describe ws.Connection`|
|Details|Set when `Connect` successfully creates the Conga client, and reset to `''` by `Close` and by a failed `Connect`. It is the handle to pass to Conga's own functions - `Describe`, `GetProp`, and so on - if you need to interrogate the connection directly.|
|Note|Unlike the other fields on this page, `Connection` is read as well as written: `Connect` checks it to decide whether the instance is already connected, so overwriting it will confuse the instance.|

### `ErrorInfo`

|--|--|
|Description|The `⎕DMX` namespace captured when an error terminated the listener thread.|
|Default|`''`|
|Example(s)|`ws.ErrorInfo.EM`<br/>`DOMAIN ERROR`|
|Details|The listener runs inside an error trap, so an error on that thread does not suspend it - the error is recorded here, the connection is closed, `Connected` is set to `0`, and the listener ends. This covers errors in your [`OnWSReceive`](settings-eventhooks.md#onwsreceive), [`OnClose`](settings-eventhooks.md#onclose), and [`OnError`](settings-eventhooks.md#onerror) hooks, in the UTF-8 translation of an incoming message, and in Conga itself. `ErrorInfo` is `''` until something is caught, so a listener that stopped with `ErrorInfo` still `''` stopped for an ordinary reason rather than an error.|
|Note|Setting [`Debug`](settings-connect.md#debug) to a non-zero value disables the trap, so that errors suspend the listener thread and can be examined in the debugger instead. See [Two things to be careful about](userguide.md#two-things-to-be-careful-about).|

### `LastWaitResponse`

|--|--|
|Description|The most recent non-timeout result of Conga's `Wait` on the listener thread.|
|Default|`''`|
|Example(s)|`3⊃ws.LastWaitResponse ⍝ the event name`|
|Details|A 4-element vector of the Conga return code, the Conga object name, the event name (`'WSReceive'`, `'Closed'`, `'Error'`, ...), and the event's data. It is updated for every event the listener sees except `'Timeout'`, so it survives as a record of the last thing that actually happened on the connection - most usefully the `'Closed'` or `'Error'` event that ended it.|

### `Data`

|--|--|
|Description|The unparsed data of a response Conga could not interpret as HTTP.|
|Default|`''`|
|Details|Populated only in the specific case where Conga reports an `HTTPHeader` event whose data it has not been able to break into version, status, message and headers. `Connect` then fails with `rc` of `¯1` and `msg` of `'Conga failed to parse the response HTTP header'`, leaving the raw data here for inspection. It normally stays `''`.|

### `HttpStatus`

|--|--|
|Description|The HTTP status of the last non-WebSocket response received during `Connect`.|
|Default|`⍬`|
|Example(s)|`ws.HttpStatus`<br/>`301`|
|Details|An integer. Set when the server answers the upgrade request with an ordinary HTTP response rather than a `101` - in practice a redirection (`301`, `302`, `303`, `307`, `308`), which `Connect` follows, or any other status, which fails with `'Unexpected server response: ...'`. `Connect` resets it to `⍬` at the start of each attempt, so `⍬` after a successful connection means the handshake was answered directly.|
|Note|The status of a successful upgrade is not recorded here - it is `WSUpgradeResponse.status`.|

### `HttpMessage`

|--|--|
|Description|The HTTP reason phrase accompanying `HttpStatus`.|
|Default|`''`|
|Example(s)|`ws.HttpMessage`<br/>`Moved Permanently`|
|Details|Set and reset alongside `HttpStatus`.|

### `HttpVersion`

|--|--|
|Description|The HTTP version of the response that set `HttpStatus`.|
|Default|`''`|
|Example(s)|`ws.HttpVersion`<br/>`HTTP/1.1`|
|Details|Set and reset alongside `HttpStatus`.|

### `HttpHeaders`

|--|--|
|Description|The headers of the response that set `HttpStatus`.|
|Default|`''`|
|Example(s)|`ws.HttpHeaders ws.GetHeader 'Location'`|
|Details|A 2-column matrix of header names and values, suitable as the left argument to [`GetHeader`](public-methods.md#getheader). Set and reset alongside `HttpStatus`.|

### `Redirections`

|--|--|
|Description|A vector of namespaces, one per redirection followed during `Connect`.|
|Default|`⍬`|
|Example(s)|`⊃ws.Redirections.URL ⍝ where we were first sent`|
|Details|Each namespace records the state **before** that redirection was followed: `URL` is the URL that produced the response, and `HttpVersion`, `HttpStatus`, `HttpMessage`, and `HttpHeaders` are the response itself. [`URL`](settings-connect.md#url) is then updated to the `Location` header and the request retried, up to [`MaxRedirections`](settings-connect.md#maxredirections) times. When a proxy is in use, each redirection also means a fresh `CONNECT`.|
|Note|`Redirections` accumulates as an attempt proceeds and is cleared at the start of the next one, so it always describes a single `Connect`.|

### `Secure`

|--|--|
|Description|Whether the connection to the server is secure.|
|Default|`⍬`|
|Details|Set from the parsed [`URL`](settings-connect.md#url) during `Connect` - `1` for `wss:`/`https:`, or when a [`Cert`](settings-conga.md#cert) or [`PublicCertFile`](settings-conga.md#publiccertfile) has been supplied. `Secure`, `Host`, `Port`, and `Path` always describe the end server, never the proxy, and are set before the handshake, so they show what `Connect` was aiming at even when the attempt failed.|

### `Host`

|--|--|
|Description|The host name parsed from `URL`, lower-cased.|
|Default|`''`|
|Example(s)|`ws.Host`<br/>`echo.websocket.org`|
|Details|Any credentials and port in the URL are removed; see `Secure` above.|

### `Port`

|--|--|
|Description|The port parsed from `URL`, or the default for the scheme.|
|Default|`⍬`|
|Example(s)|`ws.Port`<br/>`443`|
|Details|`80` or `443` if the URL did not give a port explicitly. This is the port named in the `CONNECT` request when connecting through a proxy, which is worth checking if a proxy refuses the tunnel.|

### `Path`

|--|--|
|Description|The resource path parsed from `URL`.|
|Default|`''`|
|Example(s)|`ws.URL←'wss://example.com/api/socket'`<br/>`ws.Path`<br/>`/api/socket`|
|Details|Always begins with `/`, and is `/` if the URL gave no path at all. Spaces are converted to `%20`. The query string is not included - `Connect` builds that separately from the URL's own query string and [`Params`](settings-connect.md#params), and appends it to `Path` when it sends the upgrade request.|

### `PeerCert`

|--|--|
|Description|The server's certificate, on a secure connection.|
|Default|`''`|
|Example(s)|`ws.PeerCert.Formatted`|
|Details|Read from Conga once the upgrade handshake has completed, and only when the connection is secure - it stays `''` otherwise. When a proxy is in use this is still the end server's certificate, since TLS is negotiated through the tunnel; the proxy's own certificate is not retained.|

!!! note
    `Connect` clears these fields only when it is going to attempt a connection. If the instance is already connected it returns `0 'Already connected'` immediately, leaving every status field as the live connection left it - so a second `Connect` is safe and does not disturb what you are looking at.
