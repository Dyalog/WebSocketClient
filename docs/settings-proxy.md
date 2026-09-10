These fields are only used if `ProxyURL` is set to connect through a proxy server. Note that when using a proxy server, [`URL`](settings-connect.md#url) must be fully qualified with a leading scheme (`ws://`, `wss://`, `http://` or `https://`).

### `ProxyURL`

|--|--|
|Description|The address of an HTTP proxy server to tunnel the WebSocket connection through.|
|Default|`''`|
|Example(s)|`ws.ProxyURL←'http://proxy.example.com:8080'`|
|Details|If non-empty, `Connect` first connects to `ProxyURL` and issues an HTTP `CONNECT` request for `URL`'s host and port, then continues the WebSocket handshake over the resulting tunnel (upgrading it to TLS first if `URL` is secure). Using a proxy requires Conga version `3.4.1626` or later - `Connect` fails immediately if `CongaVersion` is older.|

### `ProxyAuth`

|--|--|
|Description|Credentials to authenticate with the proxy server: one of `''` (none), a token character vector, or a 2-element vector `(userid password)`.|
|Default|`''`|
|Example(s)|`ws.ProxyAuth←'proxyuser' 'proxypassword'`|
|Details|Combined with `ProxyAuthType` to build the `Proxy-Authorization` header sent with the `CONNECT` request, following the same automatic Basic-encoding rules as [`Auth`](settings-connect.md#auth). If `ProxyAuth` is not set, credentials embedded in `ProxyURL` (e.g. `http://user:pass@proxyhost`) are used instead.|

### `ProxyAuthType`

|--|--|
|Description|The authentication scheme used to build the `Proxy-Authorization` header, along with `ProxyAuth`.|
|Default|`''`|
|Example(s)|`ws.ProxyAuthType←'BASIC'`|
|Details|Works the same way as [`AuthType`](settings-connect.md#authtype), but applies to the proxy `CONNECT` request rather than the WebSocket upgrade request.|

### `ProxyHeaders`

|--|--|
|Description|HTTP headers to be sent with the proxy `CONNECT` request. Accepts the same formats as `Headers`.|
|Default|`0 2⍴⊂''`|
|Example(s)|`ws.ProxyHeaders←('Proxy-Connection' 'Keep-Alive')`|
|Details|`ProxyHeaders` is merged with any `Proxy-Authorization` header derived from `ProxyAuth`/`ProxyAuthType` (or from credentials embedded in `ProxyURL`) before `Connect` issues the `CONNECT` request.|

### `ProxyResponse`

|--|--|
|Description|A namespace holding the proxy server's response to the `CONNECT` request.|
|Default|`''`|
|Example(s)|`ws.ProxyResponse.status` `ws.ProxyResponse.headers ws.GetHeader 'Content-Type'`|
|Details|Populated whenever the proxy replies to `CONNECT`, whether or not the request succeeds - it is cleared at the start of each `Connect`, so it remains `''` if the current attempt received no response. The namespace has the elements `version`, `status`, `message`, `headers`, and `payload`; `version`, `status`, `message` and `headers` are character data, `status` being the HTTP status as text, for example `'200'`. `payload` is `''` unless the response carries a `Content-Length` greater than `0`, in which case `Connect` waits for the body and encloses the resulting Conga event into it - the body text itself is `4⊃⊃ProxyResponse.payload`. If `status` is not `'200'`, `Connect` fails with `msg` of `'Proxy CONNECT response failed, ProxyResponse has the response from the proxy server'`.
