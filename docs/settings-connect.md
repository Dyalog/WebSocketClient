## Connect-related fields

### `URL`

|--|--|
|Description|The WebSocket (or HTTP) server URL to connect to.|
|Default|`''`|
|Example(s)|`ws.URL←'wss://echo.websocket.org'`|
|Details|`URL` must be a non-empty simple character vector. The scheme may be `ws:`, `wss:`, `http:`, `https:`, or omitted. If omitted, `WebSocketClient` will `wss:`/`https:` (or a bare port of `443`) causes the connection to be treated as secure. On a redirect, `URL` is updated to the response's `Location` header value and the prior `URL` and response details are recorded in `Redirections`.|

### `Params`

|--|--|
|Description|Request parameters to be appended to the URL's query string.|
|Default|`''`|
|Example(s)|`ws.Params←'name' 'fred' 'type' 'student'`|
|Details|`Params` may be a simple character vector, a flat array of name/value pairs, or a namespace of variables. It is then properly formatted, if necessary, and appended to any query string already present in `URL`.|

### `ValidFormUrlEncodedChars`

|--|--|
|Description|A shared, read-only constant listing the characters that may appear in a query string without being percent-encoded.|
|Default|`'&=ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~*+~%'`|
|Example(s)|`WebSocketClient.ValidFormUrlEncodedChars`|
|Details|When `Params` is a simple character vector, `Connect` treats it as already encoded and leaves it alone if every character it contains is in this set; otherwise it is passed through [`UrlEncode`](public-methods.md#urlencode). `Params` given as pairs or as a namespace is always encoded. The field is read-only and is omitted from [`Config`](public-methods.md#config).|

### `Headers`

|--|--|
|Description|HTTP headers to be sent in the WebSocket upgrade request. `Headers` can be a 2-column matrix of name/value pairs, a vector of name/value pairs (`'hdr1' 'value1' 'hdr2' 'value2'`) or a vector of pairs of names/values (`('hdr1' 'value1')('hdr2' 'value2')`).|
|Default|`0 2⍴⊂''`|
|Example(s)|`ws.Headers←('X-Custom-Header' 'value') ('Accept' '*/*')`|
|Details|`Headers` is usually more conveniently maintained with `AddHeader`, `SetHeader`, `RemoveHeader`, and `GetHeader` rather than being set directly. Headers with empty values are dropped before the upgrade request is sent, and any header set via `Auth`/`AuthType`, `Protocol`, or `Extensions` is merged in alongside those already in `Headers`.|

### `Auth`

|--|--|
|Description|Credentials to authenticate with: one of `''` (none), a token character vector, or a 2-element vector `(userid password)`.|
|Default|`''`|
|Example(s)|`ws.Auth←'myuserid' 'mypassword'`|
|Details|If `Auth` is set, it takes priority over an `Authorization` header set directly, which in turn takes priority over credentials embedded in the URL (`wss://userid:password@host`). If `Auth` is `(userid password)` or contains a `:`, and `AuthType` is `''` or `'BASIC'` (case-insensitive), the credentials are Base64-encoded automatically and `AuthType` is set to `'Basic'`.|

### `AuthType`

|--|--|
|Description|The authentication scheme used to build the `Authorization` header, along with `Auth`.|
|Default|`''`|
|Example(s)|`ws.AuthType←'BEARER'`|
|Details|Typical values are `''`, `'BASIC'`, `'BEARER'`, `'TOKEN'`. `AuthType` and `Auth` are combined as `'AuthType Auth'` to form the `Authorization` header value - see `Auth` above for the automatic Basic-auth encoding rules.|

### `Origin`

|--|--|
|Description|The intended value of the `Origin` header.|
|Default|`'null'`|
|Details|In the current implementation, `Origin` is not automatically added to the request headers by `Connect`. If a server requires an `Origin` header, set it explicitly with `AddHeader`/`SetHeader` (or via `Headers`).|

### `HeaderSubstitution`

|--|--|
|Description|A 2-element vector of `(beg end)` delimiter strings used to substitute environment variable values into header names/values.|
|Default|`''`|
|Example(s)|`ws.HeaderSubstitution←'${' '}'`<br/>`ws.AuthType←'BEARER`<br/>`ws.Auth←'${MY_TOKEN}'`|
|Details|When non-empty, any header text matching `beg`, followed by a letter and any further characters, followed by `end`, is treated as the name of an environment variable; that portion of the header is replaced with the variable's value (via `GetEnv`) before the WebSocket upgrade request is sent. If the environment variable is not set, the matched text is left unchanged. This lets secrets be kept out of source code.|

### `Secret`

|--|--|
|Description|Whether `Config` should hide credential values.|
|Default|`1`|
|Example(s)|`ws.Secret←0 ⋄ ws.Config`|
|Details|When `1`, `Config` replaces the `Auth` and `ProxyAuth` values in its result with `'>>> Secret setting is 1 <<<'` so credentials aren't inadvertently displayed or logged. Set to `0` to see the actual values in `Config`'s result.|

### `AutoUpgrade`

|--|--|
|Description|Whether to automatically accept the server's WebSocket upgrade response.|
|Default|`1`|
|Example(s)|`ws.AutoUpgrade←0`|
|Details|When `1`, Conga's `WSAutoUpgrade` option is set so the handshake completes without user intervention (`OnWSUpgrade` is still called, as a last chance to validate/reject it). When `0`, the server's response is left as a normal HTTP response for inspection via `onWSResponse`, and `WSAccept` is called to manually complete the handshake. When connecting through a proxy, the `WSAutoUpgrade` option is instead applied to the tunnelled connection after the proxy `CONNECT`/TLS handshake completes.|

### `MaxRedirections`

|--|--|
|Description|The maximum number of redirect "hops" that `Connect` will follow.|
|Default|`2`|
|Example(s)|`ws.MaxRedirections←5`|
|Details|If the server responds with an HTTP redirection status (`301`, `302`, `303`, `307`, or `308`) and a `Location` header, `Connect` follows it, recording each hop as a namespace appended to `Redirections`. `Connect` fails once the number of redirections would exceed `MaxRedirections`.|

### `Debug`

|--|--|
|Description|Whether to disable error trapping so that errors suspend where they occur, rather than being reported as a return code and message.|
|Default|`0`|
|Example(s)|`ws.Debug←1 ⍝ let errors suspend`<br/>`ws.Debug←2 ⍝ also stop just before the Conga client is created`|
|Details|`Debug` is a **shared** field - setting it on any instance, or on the class itself (`WebSocketClient.Debug←1`), affects every instance. When `0`, `Connect` and `New` trap all errors: `Connect` returns `rc` of `¯1` and a `msg` of `'Unexpected ...'` naming the error and the line it occurred on, and a failed `New` returns a namespace with `rc`, `msg`, `Connected`, and `URL` in place of an instance. Any non-zero value disables that trapping, so the error suspends and can be examined in the debugger.|
|Note|`Debug←2` additionally stops just before the Conga client is created, displaying `Stopped for debugging... (Press Ctrl-Enter)` - useful for inspecting `Secure`, `Host`, `Port`, `Path`, and the headers that `Connect` has built. Because errors in a [hook function](settings-eventhooks.md) called from `Connect` are otherwise trapped and reduced to a message, setting `Debug` to `1` is the usual way to debug one.|
