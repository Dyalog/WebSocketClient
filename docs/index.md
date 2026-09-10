`WebSocketClient` is an APL-based, cross-platform utility that can be used to communicate with WebSocket servers.

## What is a WebSocket?

A WebSocket is a continuous, two-way connection between a client and an HTTP server that stays open rather than closing after every interaction like a standard web request. By completing a one-time "handshake" to open this persistent channel, both the client and the server can instantly send and receive data at the exact same time without the lag and overhead of constantly polling the server for updates. WebSockets are useful for any application that requires real-time interaction, such as live chat applications, multiplayer games, collaborative documents, and live stock tickers.

## Terminology

`WebSocketClient` is a class written in Dyalog APL that implements a WebSocket client. Since it's both the name of the class and the name of the underlying technology, we'll use the following conventions:

- "`WebSocketClient`" (in the APL font) refers to the class.
- "WebSocket" (in a non-APL font) refers to the technology.
- "`ws`" is the name we'll use in this documentation to refer to an instance of `WebSocketClient`. Obviously, you can name it whatever you like in your application.

## Obtaining `WebSocketClient`

You can obtain `WebSocketClient` in any of the following ways:

- Clone or download the zip file from the [`WebSocketClient`](https://github.com/dyalog/WebSocketClient) repository
- Download the `WebSocketClient.aplc` file from the [latest release of `WebSocketClient`](https://github.com/dyalog/WebSocketClient/releases/latest).
- Use Tatin to load `WebSocketClient` - `]TATIN.LoadPackages WebSocketClient`<br>Note: You will need to activate Tatin before you can use the Tatin user commands.

## Your First `WebSocketClient`

[WebSocket.org](https://websocket.org) has a public server that you can use to test WebSocket connections in real time. After having obtained `WebSocketClient` you can do the following:

```
      ws←WebSocketClient.New 'wss://echo.websocket.org'
      ws.Connect
0  Connected
>>> Request served by 4d896d95b55478
```

First we created a new instance `WebSocketClient` with the server's URL. `echo.websocket.org` is a publicly available server for testing WebSockets. Next we created the WebSocket connection using the `Connect` function. Running `Connect` will start a "listener" thread to receive any incoming messages. By default `WebSocketClient` will display the messages it receives to the session prefixed by `>>> `. Once connected, `echo.websocket.org` will echo back whatever it receives.

```
     ws.Send 'hello world'
>>> hello world
```

Finally, we can close the WebSocket using the `Close` function.

```
      ws.Close
0  Closed
```

## Further Reading

- For general information about WebSockets see [WebSocket.org](https://websocket.org).
- For information about Conga's WebSocket support see the [Conga User Guide](https://docs.dyalog.com/20.0/files/Conga_User_Guide.pdf)
