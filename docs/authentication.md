## Headers and Authentication

The WebSocket upgrade request is an HTTP request, and anything an HTTP request can
carry, it can carry - which is how most servers expect to be told who you are.

### Building `Headers`

[`Headers`](settings-connect.md#headers) is held as a 2-column matrix of name and
value, but it accepts several shapes and normalizes whatever you give it the first
time it is used:

```
      ws.Headers←'Accept: text/plain',(⎕UCS 10),'X-Trace: 42' ⍝ text, one per line
      ws.Headers←('Accept' 'text/plain')('X-Trace' '42') ⍝ name/value pairs
      ws.Headers←'Accept' 'text/plain' 'X-Trace' '42'    ⍝ flat, alternating
      ws.Headers←2 2⍴'Accept' 'text/plain' 'X-Trace' '42' ⍝ the matrix itself
```

All four produce the same thing. A shape that cannot be read as headers signals
`FORMAT ERROR` from the header methods, and stops `Connect` with a `msg` of
`'Improper header format'`.

In practice the methods are easier than the field:
[`AddHeader`](public-methods.md#addheader) adds a header unless it is already there,
[`SetHeader`](public-methods.md#setheader) overwrites,
[`RemoveHeader`](public-methods.md#removeheader) deletes, and
[`GetHeader`](public-methods.md#getheader) reads:

```
      'Accept'ws.AddHeader'text/plain'
      'Accept'ws.AddHeader'application/json'  ⍝ ignored - Accept is already set
      'Accept'ws.SetHeader'application/json'  ⍝ this one replaces it
      ws.GetHeader'accept'
application/json
```

Names are matched case-insensitively throughout, so the case you use to look one up or
remove it does not matter - though `SetHeader` stores the name as you spell it.

Two things happen to your headers on the way out, both covered under
[WebSocket Handshake](troubleshooting.md#what-is-sent): headers with empty values are dropped, and
Conga adds the protocol's own (`Upgrade`, `Connection`, `Sec-WebSocket-Key`,
`Sec-WebSocket-Version`), which you should not set yourself.

### Identifying Yourself to the Server

There are three ways to supply credentials, and if more than one is used they take
precedence in this order:

1. [`Auth`](settings-connect.md#auth) with [`AuthType`](settings-connect.md#authtype)
1. an `Authorization` header you set yourself
1. credentials embedded in the URL - `wss://userid:password@host`

For Basic authentication, give `Auth` the pair and let `WebSocketClient` do the
encoding. It recognises `(userid password)`, or a single string containing a colon,
and - if `AuthType` is empty or `'BASIC'` - Base64-encodes it and sets `AuthType` to
`'Basic'` for you:

```
      ws.Auth←'brian' 'secret'
      ws.Connect
0 Connected
      ws.AuthType     ⍝ filled in by Connect
Basic
```

For token schemes, set both fields and nothing is transformed - the header value is
simply `AuthType`, a space, and `Auth`:

```
      ws.(Auth AuthType)←'eyJhbGciOi...' 'Bearer'
```

The `Authorization` header built this way is added to the request, not to `Headers`,
so it never appears in the instance's own header matrix.

### Keeping Credentials Out of Your Source

Two settings help with the awkward fact that credentials tend to end up in code and in
displays.

[`HeaderSubstitution`](settings-connect.md#headersubstitution) sets a pair of
delimiters; any delimited name found in a header - or in `Auth`, which becomes one -
is replaced with the value of that environment variable as the request is built:

```
      ws.HeaderSubstitution←'${' '}'
      ws.(AuthType Auth)←'Bearer' '${MY_API_TOKEN}'
```

The token itself lives in the environment, and the substitution happens too late for
the real value ever to be stored in the instance. A name with no matching environment
variable is left alone rather than blanked, which is worth remembering when a server
rejects credentials that look right: `ws.GetEnv 'MY_API_TOKEN'` tells you whether the
variable is actually set.

[`Secret`](settings-connect.md#secret), which defaults to `1`, masks `Auth` and
`ProxyAuth` in [`Config`](public-methods.md#config) and hides `Authorization` and
`Proxy-Authorization` values wherever headers are displayed:

```
      ws.Config
 ...
 Auth       >>> Secret setting is 1 <<<
 ...
```

Set it to `0` while debugging if you need to see what is actually being sent.
