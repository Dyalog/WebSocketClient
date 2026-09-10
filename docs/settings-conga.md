## Conga-related fields

### `BufferSize`

|--|--|
|Description|The buffer size (in bytes) passed to Conga's `Clt` (client) constructor for the underlying connection.|
|Default|`200000`|
|Example(s)|`ws.BufferSize←1000000`|
|Details|The connection is created in Conga's `'http'` mode, where `BufferSize` limits the size of the HTTP headers Conga will accept - in practice, the headers of the server's response to the WebSocket upgrade request. `BufferSize` must be set before `Connect` is called; changing it afterwards has no effect on an already-established connection.|
|Note|`BufferSize` does not limit or divide up WebSocket messages. A message arrives in several pieces only if the sender fragmented it - see [Partial Messages](partial.md) - and raising `BufferSize` will not change how a message is delivered.|

### `WaitTime`

|--|--|
|Description|The timeout, in milliseconds, passed to Conga's `Wait` function while listening for WebSocket events, and used to bound how long `Close` waits for the listener thread to terminate.|
|Default|`5000`|
|Example(s)|`ws.WaitTime←10000 ⍝ wait up to 10 seconds`|
|Details|Each iteration of the listen loop calls `LDRC.Wait` with a timeout of `WaitTime` milliseconds; a timeout is not treated as an error and the loop simply waits again. `Close` signals the listener to stop and then polls for up to `WaitTime×1.1` milliseconds before forcibly killing the listener thread.|

### `Cert`

|--|--|
|Description|An `X509Cert` instance to use for the client certificate when connecting over `wss` (HTTPS/TLS). If empty, `PublicCertFile` and `PrivateKeyFile` are used instead.|
|Default|`⍬`|
|Example(s)|`ws.Cert←cert ⍝ cert is a previously-created X509Cert instance`|
|Details|`Cert` may also be set to a 2-element vector `(PublicCertFile PrivateKeyFile)` as a shorthand for setting those two fields directly. Supplying either `Cert` or `PublicCertFile` causes the connection to be treated as secure even if the URL scheme does not indicate it.|

### `SSLFlags`

|--|--|
|Description|The SSL/TLS validation flags passed to Conga as `SSLValidation` when creating the secure connection.|
|Default|`32` (accept the server certificate without checking it)|
|Example(s)|`ws.SSLFlags←0 ⍝ perform full certificate validation`|
|Details|See the Conga User Guide for the full list of `SSLValidation` flag values and how they may be combined.|

### `Priority`

|--|--|
|Description|The GnuTLS priority string passed to Conga when creating the secure connection.|
|Default|`'NORMAL:!CTYPE-OPENPGP'`|
|Example(s)|`ws.Priority←'NORMAL:!CTYPE-OPENPGP'`|
|Details|See the Conga User Guide and GnuTLS documentation for the syntax of priority strings.|

### `PublicCertFile`

|--|--|
|Description|Path to a file containing the client's public certificate, used when `Cert` is not set to an `X509Cert` instance.|
|Default|`''`|
|Example(s)|`ws.PublicCertFile←'client.pem'`|
|Details|If `PublicCertFile` is supplied, `PrivateKeyFile` must be supplied as well (and vice versa) - `WebSocketClient` reads and decodes the certificate from these two files to build the `X509Cert` instance used for the connection.|

### `PrivateKeyFile`

|--|--|
|Description|Path to the file containing the private key corresponding to `PublicCertFile`.|
|Default|`''`|
|Example(s)|`ws.PrivateKeyFile←'client.key'`|
|Details|Used together with `PublicCertFile`; see `PublicCertFile` above.|

### `LDRC`

|--|--|
|Description|A shared reference to the Conga (or DRC) namespace/instance that `WebSocketClient` uses to make all Conga calls, set once Conga has been located and initialized.|
|Default|unset|
|Example(s)|`ws.LDRC.Names'.'`|
|Details|`LDRC` is set automatically during `Initialize` and should not normally be set directly by user code. It is shared across all instances of `WebSocketClient`. However, it can be used to call Conga functions that aren't otherwise exposed through the `WebSocketClient` API.|

### `CongaPath`

|--|--|
|Description|A shared field giving the path to a user-supplied Conga workspace and the platform-specific shared libraries that go with it.|
|Default|`''`|
|Example(s)|`WebSocketClient.CongaPath←'/opt/conga/'`|
|Details|`CongaPath` serves two purposes. When Conga has to be copied from a workspace, it names the folder to copy from - and only that folder is searched; if `CongaPath` is empty, `WebSocketClient` falls back to the Dyalog installation's `ws/` folder and then the current working directory. `CongaPath` is *also* passed to `Init` whenever `WebSocketClient` initializes a `Conga` or `DRC` namespace itself, telling Conga where to load its shared libraries from, so it applies however Conga was located - not just when one is copied. It is ignored only when [`CongaRef`](#congaref) is an already-initialized Conga instance. See [Finding Conga](./conga.md#finding-conga).|

### `CongaRef`

|--|--|
|Description|A shared field letting the user supply a specific reference to a Conga or DRC library, instead of `WebSocketClient` locating and/or copying one itself.|
|Default|`''`|
|Example(s)|`WebSocketClient.CongaRef←#.Utils.Conga ⍝ Conga is a reference to an already-initialized Conga namespace`|
|Details|`CongaRef` may be a character vector naming a namespace, a reference to the `Conga` or `DRC` namespace, or a reference to an already-initialized Conga instance. See [Finding Conga](./conga.md#finding-conga) for more information.|

### `CongaVersion`

|--|--|
|Description|A shared, read-mostly field set to the Conga library version once Conga has been initialized.|
|Default|`''`|
|Details|`CongaVersion` is set from `LDRC.Version` during `Initialize` and is subsequently used internally - for example, to check that the Conga version in use is recent enough to support proxy connections.|
