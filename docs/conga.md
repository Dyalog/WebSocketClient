## Conga Usage

Conga is a shared resource. It's not unusual for an application to have more than one Conga-using component like `WebSocketClient`, `HttpCommand`, `Jarvis`, `isolate` and so on. Each of these needs to use Conga without treading on the others.

!!! note "Use Conga instead of DRC"

> If you have more than one Conga-using component in your application, you should use the `Conga` namespace (from the conga workspace) in preference to the `DRC` namespace. The `Conga` namespace supports multiple Conga roots which is what you should use in this type of situation. The `DRC` namespace is kept largely for backwards compatibility for older applications.

`WebSocketClient` uses three user-settable fields that allow you to configure how it locates Conga: [`LDRC`](settings-conga.md#ldrc), [`CongaRef`](settings-conga.md#congaref) and [`CongaPath`](settings-conga.md#congapath). There is also a fourth, read-only, field - [`CongaVersion`](settings-conga.md#congaversion) that reports Conga's version once it has been located and initialized.
All of these fields are _shared_, class-level, fields, so however many instances you create, they resolve Conga once between them.
The resolution is done under `:Hold`, so instances started on separate threads at the same time cannot race each other into initializing Conga twice.

### Default Behavior

`Connect` initializes Conga on first use, and [`Init`](public-methods.md#init) does
the same thing without connecting - useful for checking the setup, or the Conga
version, up front:

```
      ws←WebSocketClient.New ''
      ws.Init
0 Initialized
      ws.CongaVersion
3 6 1703
      ws.LDRC ⍝ reference to the initialized Conga library
#.WebSocketClient.[LIB]
```

The default behavior for [`Connect`](./public-methods.md#connect) or [`Init`](./public-methods.md#init) in a workspace that doesn't already have Conga, is to copy the `Conga` namespace from the `conga` workspace into the `WebSocketClient` class and run `Conga.Init 'WebSocketClient'` thereby creating a Conga root named `WebSocketClient`, and [`LDRC`](./settings-conga.md#ldrc) is a reference to the initialized Conga library.

### Playing Nicely With Others

When your application has more than one Conga-using component, you'll want to use the `Conga` namespace from `conga` workspace to create a Conga root for each component. `HttpCommand`, `Jarvis`, `WebSocketClient` and `WebSocketServer` all behave similarly in their use of `LDRC`, `CongaRef`, and `CongaPath`. Other tools like [`isolate`](https://docs.dyalog.com/20.0/files/Parallel_Language_Features.pdf) will have different ways to specify where to find Conga. Let's suppose you have a hypothetical application that uses `Jarvis` as a web service, `HttpCommand` to access resources on the net, `WebSocketClient` for real-time full-duplex communications, and `isolate` to run computations in parallel.

```
⍝ get all the components we'll be using
      'Conga' ⎕CY 'conga'
      'isolate' 'll' ⎕CY 'isolate'
      ]load HttpCommand -nol
      ]get https://raw.githubusercontent.com/Dyalog/Jarvis/refs/heads/master/Source/Jarvis.dyalog
      ]get https://raw.githubusercontent.com/Dyalog/WebSocketClient/refs/heads/master/Source/WebSocketClient.aplc

⍝ tell each component which Conga to use
      (HttpCommand Jarvis WebSocketClient).CongaRef←#.Conga
⍝ isolate is configured through isolate.Config rather than a field - left at its
⍝ default it creates its own root, named "isolate", in #.DRC

⍝ run each of the components (this will initialize Conga for each one)
      HttpCommand.Get 'dyalog.com'
[rc: 0 | msg:  | HTTP Status: 200 "OK" | ≢Data: 22860]
       ⍳ ll.Each 3 4 5 ⍝ isolate's "parallel each"
 1 2 3  1 2 3 4  1 2 3 4 5
      Jarvis.Run''
2026-09-09 @ 14.27.17.142 - Starting  Jarvis  1.22.6
2026-09-09 @ 14.27.17.145 - Local Conga v3.7 reference is #.[LIB]
2026-09-09 @ 14.27.17.147 - Jarvis starting in "JSON" mode on port 8080
2026-09-09 @ 14.27.17.150 - Serving code in #
2026-09-09 @ 14.27.17.151 - Click http://192.168.223.117:8080 to access web interface
 #.[Jarvis]  0  Server started
      ws←WebSocketClient.New 'wss://echo.websocket.org'
      ws.Connect
0  Connected
>>> Request served by 4d896d95b55478

⍝ check all the Conga roots created
      ((Jarvis HttpCommand WebSocketClient).LDRC #.DRC).RootName
 Jarvis  HttpCommand  WebSocketClient   isolate
```

As you can see, each component has its own Conga root which can be manipulated independently of the other components. `isolate` is the odd one out: it takes its Conga through `isolate.Config 'drc' ⍵` rather than a field, and expects an already-initialized instance. Left at its default it copies `Conga` into `#` and runs `Conga.Init 'isolate'` itself.

### Finding ~~Nemo~~ Conga { #finding-conga }

`WebSocketClient` attempts to locate Conga as follows:

1. If [`LDRC`](./settings-conga.md#ldrc) is already set and usable, nothing further happens.
1. **[`CongaRef`](settings-conga.md#congaref), if you have set it.** It accepts a reference to a `Conga`
   or `DRC` namespace, a reference to an already-initialized Conga instance (what
   `Conga.Init` returns), or a character vector naming one, such as `'#.Conga'`. If
   `CongaRef` is set and cannot be resolved, initialization fails rather than falling through
   to the searches below.
1. **A `Conga` or `DRC` namespace in `##` or `#`.** The namespace containing the class
   is searched before the root, and `Conga` is searched before `DRC`.
1. **The `conga` workspace.** `Conga` (then `DRC`) is copied into the class itself, so
   `#.WebSocketClient.Conga` is where a copied Conga ends up. If
   [`CongaPath`](settings-conga.md#congapath) is set, only that folder is used and a
   path that does not exist or is not a folder is reported as such; otherwise the
   `ws/` folder of the Dyalog installation is tried, and then the current working
   directory.

[`CongaPath`](settings-conga.md#congapath) is not only used for the workspace copy in
step 4. Whenever `WebSocketClient` initializes a `Conga` or `DRC` namespace itself - on
any of the routes above - `CongaPath` is passed to `Init` as the location of Conga's
shared libraries, so it applies however Conga was found. It is ignored only when
[`CongaRef`](settings-conga.md#congaref) is an already-initialized Conga instance, which
brings its own libraries with it.

Whichever route succeeds, `CongaVersion` is set from the resolved library, and
`LDRC` is left pointing at the Conga `LIB` instance (or the `DRC` namespace).

### Conga Version Requirements

[`CongaVersion`](settings-conga.md#congaversion) is set once initialization succeeds, and
[`Init`](./public-methods.md#init) is a convenient way to read it before attempting a
connection. Most of `WebSocketClient` works with any Conga version that supports
WebSockets; [connecting through a proxy](proxy.md) additionally requires Conga
`3.4.1626` or later, and `Connect` stops with `'Conga version 3.4.1626 or later is
required to use a proxy'` on anything older.

For what Conga itself offers - including the `SSLValidation` flags and `X509Cert` - see
the [Conga User Guide](https://docs.dyalog.com/20.0/files/Conga_User_Guide.pdf).
