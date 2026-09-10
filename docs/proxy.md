## Connecting Through a Proxy Server

Many networks do not let an application open an outbound connection directly, and
require it to go through an HTTP proxy instead. `WebSocketClient` supports this by
_tunnelling_: it connects to the proxy and asks it, with an HTTP `CONNECT` request, to
open a connection to the real server on your behalf, and then runs the entire
WebSocket conversation through the tunnel the proxy hands back.

Setting [`ProxyURL`](settings-proxy.md#proxyurl) turns tunnelling on. All the other proxy-related settings are ignored if
`ProxyURL` is empty.

```
      ws←WebSocketClient.New ''
      ws.(URL OnWSReceive)←'wss://echo.websocket.org' 'OnMessage'
      ws.ProxyURL←'http://proxy.example.com:8080'
      ws.Connect
0  Connected
```

!!! note "Proxy support relies on Conga version `3.4.1626` or later."

### What `Connect` Does Differently

When `ProxyURL` is set, `Connect` inserts four steps ahead of the handshake it would
otherwise perform:

1. It connects to the host and port in `ProxyURL` rather than the one in
   [`URL`](settings-connect.md#url), using TLS if `ProxyURL` is `https:`.
1. It sends `CONNECT host:port HTTP/1.1` - the host and port taken from `URL` - along
   with [`ProxyHeaders`](settings-proxy.md#proxyheaders) and any
   `Proxy-Authorization` header it has derived from your settings.
1. It waits for the proxy's reply and records it in
   [`ProxyResponse`](settings-proxy.md#proxyresponse). Anything other than a status of
   `'200'` ends the attempt.
1. If `URL` is secure, it starts TLS with the _end server_ over the established
   tunnel.

From there the WebSocket upgrade request goes out exactly as it would have done on a
direct connection, and once the handshake completes the proxy is invisible - `Send`,
`Close`, and your `OnWSReceive` hook all behave identically.

**Give `URL` a scheme when you are proxying.** While a direct (non-proxied) connection may not require a `wss:` or `ws:` scheme for `WebSocketClient` to connect, when using a proxy, be sure to supply the scheme in `URL`.

### Authenticating With the Proxy

Proxy credentials are separate from the credentials the end server sees:
[`ProxyAuth`](settings-proxy.md#proxyauth) and
[`ProxyAuthType`](settings-proxy.md#proxyauthtype) build the `Proxy-Authorization`
header on the `CONNECT` request, while [`Auth`](settings-connect.md#auth) and
[`AuthType`](settings-connect.md#authtype) build the `Authorization` header on the
WebSocket upgrade request inside the tunnel. Setting one has no effect on the other,
and an application talking to an authenticated server through an authenticated proxy
sets all four.

There are three ways to specify credentials for the proxy. If you specify credentials in more than one way, the order of precedence is as follows:

- Setting [`ProxyAuth`](settings-proxy.md#proxyauth) and [`ProxyAuthType`](settings-proxy.md#proxyauthtype) takes precedence over
- Setting a `Proxy-Authorization` header in `ProxyHeaders` which takes precedence over
- Supplying credentials in the `ProxyURL`.

`ProxyAuth` follows the same rules as `Auth`: given `(userid password)`, or a string
containing a `:`, with `ProxyAuthType` either empty or case-insensitively matching `'basic'`, the credentials are
Base64-encoded for you and `ProxyAuthType` becomes `'Basic'`.

```
      ws.ProxyURL←'http://proxy.example.com:8080'
      ws.ProxyAuth←'proxyuser' 'proxypassword'    ⍝ encoded for you as Basic
```

Credentials can also be embedded in `ProxyURL` itself, which is convenient when the
proxy address arrives from a configuration file or an environment variable:

```
      ws.ProxyURL←'http://proxyuser:proxypassword@proxy.example.com:8080'
```

The two are not additive. If `ProxyAuth` is set it wins, and credentials in the URL
are ignored; the URL form is used only when `ProxyAuth` is empty. A
`Proxy-Authorization` header you place in `ProxyHeaders` yourself is likewise
overwritten by `ProxyAuth`, though it survives if only the URL form is present.

Both settings that keep credentials out of your source apply to the proxy as well as
to the end server:

- [`HeaderSubstitution`](settings-connect.md#headersubstitution) is applied to the
  `CONNECT` request's headers, including the `Proxy-Authorization` header built from
  `ProxyAuth`, so a proxy credential can be held in an environment variable:

  ```
        ws.HeaderSubstitution←'${' '}'
        ws.(ProxyAuthType ProxyAuth)←'BEARER' '${PROXY_TOKEN}'
  ```

  Substitution happens as the request is sent, but the automatic Basic encoding
  happens earlier, when the header is built - so `('${USER}' '${PASS}')` would encode
  the placeholders rather than the values. Use the single-string form, as above, or
  do the lookup yourself with [`GetEnv`](public-methods.md#getenv).

- [`Secret`](settings-connect.md#secret) masks `ProxyAuth` alongside `Auth` in
  `Config`, and masks the `Proxy-Authorization` header wherever headers are
  displayed.

### The Two Connections Are Secured Separately

There are two independent hops when you tunnel, and it is worth being clear about
which settings apply to which:

| Hop              | Secured when                                       | Certificate settings used                                                                                                                                                                                                        |
| ---------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| You → proxy      | `ProxyURL` begins `https:`                         | None - an anonymous client certificate                                                                                                                                                                                           |
| You → end server | `URL` is secure, or `Cert`/`PublicCertFile` is set | [`Cert`](settings-conga.md#cert), [`SSLFlags`](settings-conga.md#sslflags), [`Priority`](settings-conga.md#priority), [`PublicCertFile`](settings-conga.md#publiccertfile), [`PrivateKeyFile`](settings-conga.md#privatekeyfile) |

So a client certificate is presented to the end server, never to the proxy; there is
currently no way to authenticate to a proxy with a certificate rather than with
`ProxyAuth`. Errors raised while securing the proxy hop are prefixed `PROXY: ` to
distinguish them from the end-server ones.

In practice most proxies are addressed as plain `http:` even when the target is
`wss:`, and that is not the weakness it looks like: a `CONNECT` tunnel is opaque, so
the TLS session is negotiated end-to-end with the real server through it and the proxy
sees only encrypted bytes. `https:` on `ProxyURL` encrypts the `CONNECT` request
itself - which matters mainly because that request carries your proxy credentials.

### When the Proxy Refuses

A proxy that declines to open the tunnel answers the `CONNECT` request with an
ordinary HTTP response, and `Connect` leaves the whole of it in `ProxyResponse` for
you to look at. This is the first place to check whenever a proxied connection fails
where a direct one would have worked:

```
      ws.Connect
¯1  Proxy CONNECT response failed, ProxyResponse has the response from the proxy server
      ws.ProxyResponse.(status message)
407  Proxy Authentication Required
      ws.ProxyResponse.headers ws.GetHeader 'Proxy-Authenticate'
Basic realm="corporate-proxy"
```

`407` means the credentials were missing, wrong, or in a scheme the proxy does not
accept - and, as above, the `Proxy-Authenticate` header tells you which scheme it
wants. `403` usually means the credentials were fine but the proxy's policy forbids
the destination host or port; many proxies allow `CONNECT` only to port `443`, which
is a common reason for an otherwise correct `ws://` connection to be refused where
`wss://` succeeds.

Some proxies explain themselves in the response body. When the reply carries a
`Content-Length`, `Connect` reads on for that body and leaves what it gets in
`ProxyResponse.payload` - as the enclosed Conga event rather than as text, so the body
itself is `4⊃⊃ws.ProxyResponse.payload`.

`ProxyResponse` is populated whenever the proxy replies at all, successful or not, so
it is also available for inspection after a connection that worked. It stays `''` if
no reply ever arrived.

**The proxy gets one second to reply.** The wait for the `CONNECT` response is fixed
at 1000 ms and is not governed by [`WaitTime`](settings-conga.md#waittime). A proxy
that authenticates against a slow directory service can exceed it, giving
`'Proxy CONNECT wait failed: ...'` or, if it answers with something unexpected,
`'Proxy CONNECT did not respond with HTTPHeader event: ...'`. Both leave `ProxyResponse`
unset, which distinguishes them from a proxy that answered and said no.

Note also that redirections are followed _inside_ the tunnel. If the end server
redirects, `Connect` returns to the start and dials the proxy again for the new `URL`,
issuing a fresh `CONNECT` each time - so `MaxRedirections` bounds the number of
`CONNECT` requests as well.
