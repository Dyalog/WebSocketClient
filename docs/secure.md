## Secure Connections

A WebSocket connection is secured exactly as an HTTPS one is: TLS is negotiated first,
and the handshake and every frame after it travel inside it. `WebSocketClient` treats a
connection as secure if any of the following is true:

- [`URL`](settings-connect.md#url) begins `wss:` or `https:`
- `URL` specifies port `443` and no scheme at all
- [`Cert`](settings-conga.md#cert) or [`PublicCertFile`](settings-conga.md#publiccertfile)
  is set, whatever the URL says

In the ordinary case - a public server, no client certificate - there is nothing to
configure. `WebSocketClient` builds an anonymous certificate for the connection, and
`wss://` is all you need:

```
      ws←WebSocketClient.New 'wss://echo.websocket.org'
      ws.Connect
0 Connected
```

### The Default Does Not Validate the Server

[`SSLFlags`](settings-conga.md#sslflags) defaults to `32`, which tells Conga to accept
the server's certificate without checking it. The connection is encrypted, but nothing
establishes that the server on the other end is the one you meant to reach - an
expired, self-signed, or wrong-host certificate is accepted just as readily as a good
one:

```
      ws←WebSocketClient.New 'wss://expired.badssl.com'
      ws.Connect          ⍝ TLS succeeded; only the upgrade was refused
¯1 Unexpected server response: 200 OK
```

Setting `SSLFlags` to `0` asks for full validation, and the same connection is then
refused where it should be:

```
      ws.SSLFlags←0
      ws.Connect
1202 Conga failed to connect to "expired.badssl.com":  ERR_INVALID_PEER_CERTIFICATE  Remote certificate is invalid  1026
```

An application that cares who it is talking to should set `SSLFlags←0` and keep it
there. The permissive default exists because it lets test and development servers with
self-signed certificates work out of the box; the individual flag values, and how to
relax validation in a narrower way than "accept anything", are in the Conga User
Guide. [`Priority`](settings-conga.md#priority) similarly passes a GnuTLS priority
string straight through, for restricting the protocol versions and ciphers offered.

### Presenting a Client Certificate

Servers that authenticate clients by certificate rather than by password need one
supplied. There are three equivalent ways to do it:

```
      ws.Cert←cert                                 ⍝ an X509Cert instance you already have
      ws.Cert←'client.pem' 'client.key'            ⍝ public and private key files
      ws.(PublicCertFile PrivateKeyFile)←'client.pem' 'client.key'
```

The last two are the same thing said differently - a 2-element `Cert` is simply
shorthand for the two fields. Both files are required: supplying one without the other
fails with `'PublicCertFile is empty'` or `'PrivateKeyFile is empty'`, a file that is
not there gives `'Not found PublicCertFile "..."'`, and one that cannot be read as a
certificate gives `'Unable to decode PublicCertFile "..." as certificate'`. All of
these come back from `Connect` as an `rc` of `¯1` before any connection is attempted.

Because setting `Cert` or `PublicCertFile` makes the connection secure on its own, a
client certificate cannot be presented over a `ws:` connection by accident.

### Inspecting the Server's Certificate

Once a secure connection is up, [`PeerCert`](settings-status.md#peercert) holds the
server's certificate as an `X509Cert` instance:

```
      ws.PeerCert.Formatted.Subject
CN=echo.websocket.org
      ws.PeerCert.Formatted.(ValidFrom ValidTo)
```

`Formatted` also carries `Issuer`, `SerialNo`, `KeyLength`, `Extensions` and the rest -
see the Conga User Guide for what an `X509Cert` offers. This is worth reading in an
[`OnWSUpgrade`](settings-eventhooks.md#onwsupgrade) hook if you want to pin a
certificate or check an issuer yourself: the hook can refuse the connection by
returning a non-zero `rc`, though note that by then the TLS session is already
established.

`PeerCert` stays `''` on an insecure connection, and when a proxy is in use it holds
the end server's certificate rather than the proxy's - see
[The Two Connections Are Secured Separately](proxy.md#the-two-connections-are-secured-separately).
