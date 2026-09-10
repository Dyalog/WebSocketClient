`WebSocketClient` exposes two kinds of method. **Instance** methods are called on a client instance (`ws.Connect`) and act on that instance's fields. **Shared** methods are called on the class itself (`WebSocketClient.Base64Encode`), though they can also be called through an instance; they do not depend on any instance's state.

Most methods return a 2-element `(rc msg)` result, where `rc` of `0` means success and `msg` describes what happened. `Connect` and `Close` also leave their result in the instance's `rc` and `msg` fields; `Send` does not.

## Operational Methods

### `New`

|--|--|
|Description|Shared method to create a new `WebSocketClient` instance.|
|Syntax|`ws←WebSocketClient.New args`|
|`args`| can be any of <ul><li>`''` meaning all setting are at their default values</li><li>a namespace whose variables are the settings to apply</li><li>a vector of settings in the order `URL OnWSReceive OnWSUpgrade Protocol Headers Params`</li></ul>|
|`ws`|the new `WebSocketClient` instance, or, if construction failed, a namespace reporting the failure|
|Example(s)|`ws←WebSocketClient.New (URL:'echo.websocket.org' ⋄ OnWSReceive:'OnMessage')`<br/>`ws←WebSocketClient.New 'echo.websocket.org' 'OnMessage'`<br/>`ws←WebSocketClient.New '' ⋄ ws.(URL OnWSReceive)←'echo.websocket.org' 'OnMessage'`|
|Details|`args` may be empty (resulting in an instance with all defaults), a namespace whose variables are the settings to apply, or a vector of positional settings in the order `URL`, `OnWSReceive`, `OnWSUpgrade`, `Protocol`, `Headers`, `Params`. A namespace containing a name that is not a public field signals an error reporting the invalid setting(s).|
|Note|`New` traps errors that `⎕NEW` would otherwise signal: if construction fails it returns a namespace with `rc`, `msg`, `Connected`, and `URL` in place of an instance, so check `Connected` or `rc` rather than assuming an instance came back. Set [`Debug`](settings-connect.md#debug) to `1` to have the error signalled instead.|

### `Connect`

|--|--|
|Description|Establish the connection and complete the WebSocket handshake.|
|Syntax|`(rc msg)←ws.Connect`|
|`rc`|`0` if the WebSocket is connected, non-zero otherwise|
|`msg`|`'Connected'`, `'Already connected'`, or a description of the failure|
|Example(s)|`ws←WebSocketClient.New (URL:'echo.websocket.org')`<br/>`ws.Connect`|
|Details|`Connect` resolves and initializes Conga if that has not happened yet, builds and sends the WebSocket upgrade request from `URL`, `Params`, `Headers`, and the other connect-related settings, follows up to `MaxRedirections` redirects, and - via [`OnWSUpgrade`](settings-eventhooks.md#onwsupgrade) or [`OnWSResponse`](settings-eventhooks.md#onwsresponse) - completes the handshake. On success it starts the listener thread, sets `Connected` to `1`, and returns `0 'Connected'`. If the instance already has a live connection, it returns `0 'Already connected'` without doing anything.|
|Note|Errors are trapped and returned as `¯1` and a `msg` beginning `'Unexpected '` unless [`Debug`](settings-connect.md#debug) is non-zero. On failure the connection is closed, the listener thread is terminated, and `Connected` is set back to `0`.|

### `Send`

|--|--|
|Description|Send a message, or one fragment of a message, over an established WebSocket.|
|Syntax|`(rc msg)←ws.Send data`<br/>`(rc msg)←ws.Send (data final)`|
|`data`|character data to send as a text message, or integer data to send as a binary message|
|`final`|`1` (the default) if this completes the message, `0` if further fragments follow|
|`rc`|`0` if the data was sent, non-zero otherwise|
|`msg`|`''` if the data was sent, otherwise a description of the failure|
|Example(s)|`ws.Send 'Hello'`<br/>`ws.Send ('First part' 0) ⋄ ws.Send ('last part' 1)`|
|Details|When `final` is `0` the message is left open and subsequent `Send` calls continue it - `WebSocketClient` sets the continuation opcode itself. Returns `0 ''` on success, `¯1 'No client connection has been established'` if there is no connection, or Conga's return code with `'Conga send failure: ...'`.|
|Note|All fragments of one message must be of the same datatype - mixing character and integer fragments fails with `'Datatype is not the same as previous fragment (...)'`. Unlike `Connect` and `Close`, `Send` does not update the instance's `rc` and `msg` fields.|

### `Close`

|--|--|
|Description|Close the WebSocket and stop the listener thread.|
|Syntax|`(rc msg)←ws.Close`|
|`rc`|always `0`|
|`msg`|`'Closed'`, `'Not listening'`, or `'Already closed'`|
|Example(s)|`ws.Close`|
|Details|`Close` signals the listener thread to stop and waits up to `WaitTime×1.1` milliseconds for it to terminate, killing it with `⎕TKILL` if it has not. It then clears `Connection` and returns `0 'Closed'`. If no listener is running it returns `0 'Not listening'`, and if the connection has already been closed, `0 'Already closed'`.|
|Note|Because `Close` stops the listener directly rather than through a Conga `Closed` event, the [`OnClose`](settings-eventhooks.md#onclose) hook is not called. Expunging the instance does **not** close the connection: the running listener thread holds a reference to the instance, so the class's destructor does not run and you are left with an orphaned listener on a live connection. Recover the reference with `⎕INSTANCES #.WebSocketClient` and call `Close` on it - see [Closing from your side](userguide.md#closing-from-your-side).|

### `Init`

|--|--|
|Description|Locate and initialize Conga without making a connection.|
|Syntax|`(rc msg)←ws.Init`|
|`rc`|`0` if Conga was initialized, non-zero otherwise|
|`msg`|`'Initialized'`, or a description of why Conga could not be initialized|
|Example(s)|`ws.Init`|
|Details|Performs the Conga resolution that `Connect` would otherwise do on first use, honouring [`CongaRef`](settings-conga.md#congaref) and [`CongaPath`](settings-conga.md#congapath), and sets the shared `LDRC` and `CongaVersion` fields. Returns `0 'Initialized'` on success, or a non-zero `rc` and a `msg` describing why Conga could not be initialized.|
|Note|Calling `Init` is optional - it is useful for checking the Conga setup, or `CongaVersion`, before attempting a connection.|

## Header Manipulation Methods

### `AddHeader`

|--|--|
|Description|Add a header to `Headers`, unless a header of that name is already defined.|
|Syntax|`{hdrs}←{name}ws.AddHeader value`<br/>`{hdrs}←ws.AddHeader (name value)`|
|`name`|the header name; if omitted, it is taken from the first element of the right argument|
|`value`|the header value; `''` adds nothing|
|`hdrs`|(shy) the updated `Headers` matrix|
|Example(s)|`'Accept'ws.AddHeader'application/json'`<br/>`ws.AddHeader 'Accept' 'application/json'`|
|Details|If a header of that name already exists, `AddHeader` leaves it alone - use `SetHeader` to overwrite. Header names are matched case-insensitively. A `value` of `''` adds nothing. The shy result is the updated `Headers` matrix.|
|Note|Signals a `FORMAT ERROR` if the current contents of `Headers` cannot be interpreted as headers. `SetHeader` and `RemoveHeader` do the same.|

### `SetHeader`

|--|--|
|Description|Set a header in `Headers`, overwriting any existing header of that name.|
|Syntax|`{hdrs}←{name}ws.SetHeader value`<br/>`{hdrs}←ws.SetHeader (name value)`|
|`name`|the header name; if omitted, it is taken from the first element of the right argument|
|`value`|the header value|
|`hdrs`|(shy) the updated `Headers` matrix|
|Example(s)|`'Accept'ws.SetHeader'text/plain'`|
|Details|Behaves like `AddHeader` except that an existing header of the same name is replaced rather than kept. Header names are matched case-insensitively and the shy result is the updated `Headers` matrix.|

### `GetHeader`

|--|--|
|Description|Retrieve a header's value from `Headers`, or from a header matrix supplied as the left argument.|
|Syntax|`value←{hdrs}ws.GetHeader name`|
|`hdrs`|a 2-column matrix of headers to search; if omitted, the instance's `Headers`|
|`name`|the header name to look up, or a nested vector of header names|
|`value`|the header value; `''` if a single `name` is not found, or `'∘???∘'` in place of each name not found when several are requested|
|Example(s)|`ws.GetHeader'Accept'`<br/>`ws.WSUpgradeResponse.headers ws.GetHeader 'Sec-WebSocket-Protocol'`|
|Details|With no left argument, `GetHeader` looks in the instance's `Headers`. Supplying `hdrs` - for example the `headers` element of [`WSUpgradeResponse`](settings-websocket.md#wsupgraderesponse) or [`ProxyResponse`](settings-proxy.md#proxyresponse) - searches that matrix instead. Names are matched case-insensitively, and a name that is not present returns `''`.|
|Note|`name` may be a nested vector of names, in which case the corresponding values are returned; any name that is not present appears in the result as `'∘???∘'` rather than being dropped.|

### `RemoveHeader`

|--|--|
|Description|Remove one or more headers from `Headers`.|
|Syntax|`{hdrs}←ws.RemoveHeader name`|
|`name`|the header name to remove, or a nested vector of header names|
|`hdrs`|(shy) the updated `Headers` matrix|
|Example(s)|`ws.RemoveHeader'Accept'`<br/>`ws.RemoveHeader 'Accept' 'User-Agent'`|
|Details|Removes every header whose name matches, case-insensitively. Removing a name that is not present is not an error. The shy result is the updated `Headers` matrix.|

## Informational Methods

### `Config`

|--|--|
|Description|Return the instance's current configuration - every public field and its value.|
|Syntax|`r←ws.Config`|
|`r`|a 2-column matrix of public field names and their current values|
|Example(s)|`ws.Config`<br/>`ws.Secret←0 ⋄ ws.Config`|
|Details|`r` is a 2-column matrix of field names and values, covering both instance and shared fields (`ValidFormUrlEncodedChars` is omitted). A field that cannot be read is shown as `'not set'`.|
|Note|While [`Secret`](settings-connect.md#secret) is `1` - the default - the `Auth` and `ProxyAuth` values are replaced with `'>>> Secret setting is 1 <<<'` so that credentials aren't inadvertently displayed or logged.|

### `Version`

|--|--|
|Description|Shared method returning the name, version number, and date of this `WebSocketClient`.|
|Syntax|`r←WebSocketClient.Version`|
|`r`|a 3-element vector of the name, version number, and date|
|Example(s)|`WebSocketClient.Version`|
|Details|`r` is a 3-element vector, for example `'WebSocketClient' '0.9.0' '2026-08-24'`.|

### `Documentation`

|--|--|
|Description|Shared method returning a pointer to this documentation.|
|Syntax|`r←WebSocketClient.Documentation`|
|`r`|a character vector naming the documentation website|
|Example(s)|`WebSocketClient.Documentation`|
|Details|Returns the character vector `'See https://dyalog.github.io/WebSocketClient/'`.|

### `ListenerThread`

|--|--|
|Description|A read-only property giving the thread number of the listener thread started by `Connect`.|
|Syntax|`r←ws.ListenerThread`|
|`r`|the thread number of the listener, or `⍬` if no listener is running|
|Example(s)|`ws.ListenerThread∊⎕TNUMS ⍝ is the listener still running?`|
|Details|`⍬` before `Connect` succeeds and after `Close` has terminated the listener. The listener thread is where [`OnWSReceive`](settings-eventhooks.md#onwsreceive), [`OnClose`](settings-eventhooks.md#onclose), and [`OnError`](settings-eventhooks.md#onerror) are called.|

## Utility Methods

### `GetEnv`

|--|--|
|Description|Shared method returning the value of an environment variable.|
|Syntax|`r←WebSocketClient.GetEnv var`|
|`var`|the name of an environment variable|
|`r`|the variable's value, or `''` if it is not set|
|Example(s)|`WebSocketClient.GetEnv'DYALOG'`|
|Details|Returns `''` if the variable is not set. This is the same lookup used by [`HeaderSubstitution`](settings-connect.md#headersubstitution) to substitute environment variables into header values.|

### `Base64Encode`

|--|--|
|Description|Shared method to Base64-encode data.|
|Syntax|`r←{cpo}WebSocketClient.Base64Encode w`|
|`cpo`|"code points only" - if supplied (with any value), character data is encoded as code points rather than being translated to UTF-8|
|`w`|the character or integer data to encode|
|`r`|the Base64-encoded character vector|
|Example(s)|`WebSocketClient.Base64Encode'userid:password'`|
|Details|Character data is translated to UTF-8 before encoding; integer data is encoded as-is. Supplying any left argument (`cpo`, "code points only") suppresses the UTF-8 translation.|
|Note|`Auth` is Base64-encoded automatically when it holds `(userid password)` or a `userid:password` string - see [`Auth`](settings-connect.md#auth).|

### `Base64Decode`

|--|--|
|Description|Shared method to decode Base64-encoded data.|
|Syntax|`r←{cpo}WebSocketClient.Base64Decode w`|
|`cpo`|"code points only" - if supplied (with any value), the decoded bytes are returned as code points rather than being translated from UTF-8|
|`w`|the Base64-encoded character vector to decode|
|`r`|the decoded data|
|Example(s)|`WebSocketClient.Base64Decode'dXNlcmlkOnBhc3N3b3Jk'`|
|Details|The decoded bytes are translated from UTF-8 unless a left argument (`cpo`) is supplied, in which case they are treated as code points.|

### `UrlEncode`

|--|--|
|Description|Shared method to URL-encode data for use in a query string.|
|Syntax|`r←{name}WebSocketClient.UrlEncode data`|
|`name`|the name to pair `data` with, when `data` is a single value|
|`data`|a character vector, an even number of name/value character vectors, a vector of name/value pairs, or a namespace of variables to encode|
|`r`|the URL-encoded character vector, for example `'name=fred&type=student'`|
|Example(s)|`WebSocketClient.UrlEncode 'name' 'fred' 'type' 'student'`<br/>`'name'WebSocketClient.UrlEncode'fred'`|
|Details|`data` may be a simple character vector, an even number of name/value character vectors, a vector of name/value pairs, or a namespace whose variables are the names and values. The result is a character vector such as `'name=fred&type=student'`, with anything outside the unreserved character set percent-encoded from its UTF-8 bytes.|
|Note|`Connect` applies this to [`Params`](settings-connect.md#params) itself, so `UrlEncode` only needs to be called directly when building a query string by hand.|

### `setDisplayFormat`

|--|--|
|Description|Refresh the `⎕DF` display form of an instance.|
|Syntax|`ws.setDisplayFormat ns`|
|`ns`|the instance (or namespace) whose display form is to be set - normally the instance itself|
|Example(s)|`ws.setDisplayFormat ws`|
|Details|Sets `⎕DF` to a summary of `rc`, `msg`, and the connection state, which is what you see when you display an instance:<br/><br/>`[ rc: 0 | msg: Connected | Connected to wss://echo.websocket.org ]`<br/><br/>`WebSocketClient` calls this itself whenever `URL`, `Connected`, `rc`, or `msg` changes, so it rarely needs calling directly. It is also used to give the namespace returned by a failed [`New`](public-methods.md#new) the same display form as a real instance.|

