This section describes in more depth the details of how `WebSocketClient` works.

## WebSocket Handshake

A WebSocket connection begins life as an ordinary HTTP request. The client asks the
server to change protocols, and a server willing to do so answers `101 Switching
Protocols`; from that point the socket carries WebSocket frames rather than HTTP.
`Connect` performs this exchange for you, but it is worth knowing what it sends, what
it does with the answer, and where the answer is left for you to look at.

### What Is Sent

Conga composes the upgrade request, and `WebSocketClient` supplies three things to put
in it: the path (the path from [`URL`](settings-connect.md#url), plus a query string
built from the URL's own and from [`Params`](settings-connect.md#params)), the host,
and the headers.

The headers are assembled in this order:

1. [`Headers`](settings-connect.md#headers), as you have built it up with
   `AddHeader`, `SetHeader`, and friends.
1. `Sec-WebSocket-Extensions` from [`Extensions`](settings-websocket.md#extensions)
   and `Sec-WebSocket-Protocol` from [`Protocol`](settings-websocket.md#protocol) -
   both added only if you have not set that header yourself.
1. `Authorization`, from [`Auth`](settings-connect.md#auth) and
   [`AuthType`](settings-connect.md#authtype) if they are set (overwriting any
   `Authorization` header you set directly), or from credentials embedded in the URL
   if they are not.
1. [`HeaderSubstitution`](settings-connect.md#headersubstitution) is applied to the
   result, replacing delimited environment-variable references with their values.
1. Headers with empty values are dropped.

Conga adds the protocol's own mandatory headers - `Upgrade`, `Connection`,
`Sec-WebSocket-Key`, and `Sec-WebSocket-Version` - so you neither need to nor should
set those yourself.

### What Comes Back

`Connect` then waits [`WaitTime`](settings-conga.md#waittime) milliseconds for a
single Conga event, and what arrives decides the outcome:

| Event        | Meaning                                                                                                                     |
| ------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `WSUpgrade`  | The server upgraded, and Conga has already validated the response ([`AutoUpgrade`](settings-connect.md#autoupgrade) is `1`) |
| `WSResponse` | The server responded and it is yours to validate (`AutoUpgrade` is `0`)                                                     |
| `HTTPHeader` | An ordinary HTTP response - a redirection, or a refusal                                                                     |
| `Timeout`    | Nothing arrived within `WaitTime`; `Connect` returns `100 'Conga connection timed out'`                                     |
| `Error`      | Conga reported an error, which becomes the `rc`                                                                             |
| `Closed`     | The server closed the socket instead of answering; `Connect` returns `'Socket closed by server'`                            |

On either of the first two, the response is parsed into
[`WSUpgradeResponse`](settings-websocket.md#wsupgraderesponse) before your hook sees
it, and it stays there after `Connect` returns:

```
      ws.Connect
0 Connected
      ws.WSUpgradeResponse.(version status message)
 HTTP/1.1  101  Switching Protocols
      ws.WSUpgradeResponse.headers
 upgrade               websocket
 connection            Upgrade
 sec-websocket-accept  vKfjfb62aH5uFI4KM/un/ixCh3k=
 date                  Sat, 05 Sep 2026 19:01:09 GMT
 server                Fly/ec1a4f957c (2026-08-31)
      ws.WSUpgradeResponse.headers ws.GetHeader 'upgrade'
websocket
```

`status` is a number, `headers` is a 2-column matrix that
[`GetHeader`](public-methods.md#getheader) will search for you when passed as its left
argument, and `payload` holds anything that followed the headers - normally empty.
`WSUpgradeResponse` is `''` if the handshake never got as far as a response.

### Vetting the Handshake Yourself

Even with `AutoUpgrade` left at `1`, an [`OnWSUpgrade`](settings-eventhooks.md#onwsupgrade)
hook gets to see the parsed response and can veto the connection by returning a
non-zero `rc`, which `Connect` returns as its own result. This is where to check that
the server agreed to what you asked for - a sub-protocol, most usefully, since a
server is free to ignore the request and speak its own dialect instead:

```
     ∇ (rc msg)←client OnUpgrade response
[1]   ⍝ refuse the connection unless the server agreed to our sub-protocol
[2]    (rc msg)←0 ''
[3]    :If 'chat'≢response.headers client.GetHeader 'sec-websocket-protocol'
[4]        (rc msg)←¯1 'server did not accept the chat sub-protocol'
[5]    :EndIf
     ∇
```

```
      ws.(Protocol OnWSUpgrade)←'chat' 'OnUpgrade'
```

Setting `AutoUpgrade` to `0` goes further: Conga hands over the response without
validating it, `WebSocketClient` calls your
[`OnWSResponse`](settings-eventhooks.md#onwsresponse) hook, and only if that returns
`0` does it accept the upgrade. The hook is then responsible for whatever checking the
`WSUpgrade` path would have done for you, so leave `AutoUpgrade` at `1` unless you
have a specific reason not to.

Both hooks run on the thread that called `Connect`, and errors in them are trapped -
`Connect` returns `¯1` and a `msg` beginning `'Unexpected '` rather than suspending,
unless [`Debug`](settings-connect.md#debug) is non-zero.

### Redirections

A server that answers with `301`, `302`, `303`, `307`, or `308` arrives as an
`HTTPHeader` event, and `Connect` starts again against the `Location` header - up to
[`MaxRedirections`](settings-connect.md#maxredirections) times. Each hop is recorded
in [`Redirections`](settings-status.md#redirections) as a namespace holding the `URL`
that was tried and the response it produced, so a connection that ended up somewhere
unexpected can be traced afterwards. A redirection without a `Location` header, or one
too many hops, ends the attempt.

Any other HTTP status is a refusal: `Connect` returns
`¯1 'Unexpected server response: ...'` with the status and message, and
[`HttpStatus`](settings-status.md#httpstatus),
[`HttpMessage`](settings-status.md#httpmessage), and
[`HttpHeaders`](settings-status.md#httpheaders) hold the response for inspection.

### When the Handshake Fails Quietly

Not every server that declines to upgrade says so in HTTP. Asking
`echo.websocket.org` for a sub-protocol it does not support, for example, gets no
response at all - the server simply closes the socket:

```
      ws.Protocol←'chat'
      ws.Connect
1119 Socket closed by server
```

and with a short `WaitTime` the same attempt ends as
`100 'Conga connection timed out'` instead, because `Connect` gave up before the close
arrived. Either result, with `WSUpgradeResponse` still `''`, points at the request
rather than at the network: a header the server dislikes, a sub-protocol or extension
it will not speak, or a path it does not serve WebSockets on.

## When a Connection Fails

`Connect` reports a failure rather than signalling one, so a connection that did not
happen leaves you with a result to interpret and a set of status fields to read. The
fields are described in [Status-related fields](settings-status.md); this section is
about which of them to look at, and when.

[`Connected`](settings-status.md#connected) is the dependable test. `Connect`'s `rc`
is `0` on success, but a handful of validation failures - a URL that cannot be parsed,
headers that cannot be interpreted - currently report the problem in `msg` while
leaving `rc` at `0`:

```
      ws.URL←'ftp://example.com'
      ws.Connect
0 Invalid protocol: ftp
      ws.Connected
0
```

So test `Connected` (or check that `msg` is `'Connected'`) rather than testing `rc`
alone.

### Reading the Message

Failures fall into a few groups, and the message says which:

| Message                                                                                                                            | What went wrong                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| `'No URL specified'`<br/>`'URL is not a simple character vector'`<br/>`'Headers are not character'`<br/>`'Improper header format'` | Settings were rejected before anything was attempted                                               |
| `'Invalid protocol: ...'`<br/>`'No host specified'`<br/>`'Invalid host/port: ...'`<br/>`'Invalid port: ...'`                       | The URL could not be parsed                                                                        |
| `'Could not initialize Conga ...'`<br/>`'neither Conga nor DRC were successfully copied'`                                          | Conga could not be located - see [Playing Nicely With Others](conga.md#playing-nicely-with-others) |
| `'Conga failed to connect to "..." ...'`                                                                                           | The TCP or TLS connection never came up                                                            |
| `'Unexpected server response: ...'`                                                                                                | The server answered with HTTP rather than upgrading                                                |
| `'Conga connection timed out'`                                                                                                     | Nothing arrived within [`WaitTime`](settings-conga.md#waittime)                                    |
| `'Socket closed by server'`                                                                                                        | The server closed the connection instead of answering                                              |
| `'Unexpected ... at ...'`                                                                                                          | A hook called from `Connect` signalled an error                                                    |

The connection-level messages carry Conga's own text, which is usually specific enough
to act on:

```
      ws.URL←'wss://no-such-host.invalid' ⋄ ws.Connect
1106 Conga failed to connect to "no-such-host.invalid":  ERR_INVALID_HOST  Host identification not resolved
      ws.URL←'ws://127.0.0.1:9' ⋄ ws.Connect
1111 Conga failed to connect to "127.0.0.1":  ERR_CONNECT_DATA  Unable to connect to host data port
      ws.URL←'wss://expired.badssl.com' ⋄ ws.SSLFlags←0 ⋄ ws.Connect
1202 Conga failed to connect to "expired.badssl.com":  ERR_INVALID_PEER_CERTIFICATE  Remote certificate is invalid
```

`ERR_INVALID_HOST` is a name that did not resolve, `ERR_CONNECT_DATA` a host that
resolved but refused the connection, and `ERR_INVALID_PEER_CERTIFICATE` a certificate
that failed the validation asked for by
[`SSLFlags`](settings-conga.md#sslflags) - see [Secure Connections](secure.md).

### Where to Look Next

Which field holds the detail depends on how far the attempt got:

| Symptom                                                  | Look at                                                                                                                                         |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| The server answered with HTTP                            | [`HttpStatus`](settings-status.md#httpstatus), [`HttpMessage`](settings-status.md#httpmessage), [`HttpHeaders`](settings-status.md#httpheaders) |
| Conga could not parse the response as HTTP               | [`Data`](settings-status.md#data), which holds the unparsed event data                                                                          |
| The handshake completed but something about it was wrong | [`WSUpgradeResponse`](settings-websocket.md#wsupgraderesponse)                                                                                  |
| The connection ended up somewhere unexpected             | [`Redirections`](settings-status.md#redirections)                                                                                               |
| A proxy is in use                                        | [`ProxyResponse`](settings-proxy.md#proxyresponse) - see [When the Proxy Refuses](proxy.md#when-the-proxy-refuses)                              |
| The connection was made and then died                    | [`ErrorInfo`](settings-status.md#errorinfo) and [`LastWaitResponse`](settings-status.md#lastwaitresponse)                                       |

A worked example of the third row - a server answering `200 OK` to an upgrade request
because the path serves ordinary HTTP - looks like this:

```
      ws.URL←'wss://example.com'
      ws.Connect
¯1 Unexpected server response: 200  OK
      ws.(HttpStatus HttpMessage)
200  OK
```

Nothing is cleared when a connection ends, so all of these survive for as long as you
need them; `Connect` clears them only when it is about to make a fresh attempt.

### Turning Off the Safety Net

[`Debug`](settings-connect.md#debug) has two useful values while diagnosing:

- `1` disables the error trapping everywhere, so an error inside `Connect`, inside the
  listener, or inside one of your hooks suspends where it happened instead of being
  reduced to `msg` or `ErrorInfo`. This is how to develop a hook.
- `2` stops `Connect` just before the Conga client is created, which is the moment to
  inspect the headers, secure parameters, and options that are about to be used.

`Debug` is a shared field, so setting it affects every instance.
