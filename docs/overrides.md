`WebSocketClient` allows you to override its default behavior for specific events. The events correspond to Conga events which are documented in the Conga User Guide. Most can be left to their default behavior, but `OnWSReceive` should be set as the default behavior is to just output received messages to the session, which is not particularly useful in a real-world application.

There are two ways to override an event's default behavior:

### Write a Function
* Set the appropriate setting for the event to the name of a function to be called when that event occurs.

### Override a Method
* Create a derived class based on `WebSocketClient` and write overrides for the events you want to customize.

## Overridable Events

### `(rc msg) ← OnWSUpgrade waitData`
|--|--|
| Description | The maximum number of bytes that `HttpCommand` will accept for the HTTP status and headers received from the host.|
| Default |`200000`|
|Example(s)|`h.BufferSize←50000 ⍝ set a lower threshold`|
|Details|By default, when using Conga's HTTP mode, as `HttpCommand` does, `BufferSize` applies only to the data received by the `HTTPHeader` event. The intent is to protect against a maliciously large response from the host.  If the data received by the `HTTPHeader` event exceeds `BufferSize`, `HttpCommand` will exit with a return code of 1135.|

### `OnWSResponse`

### `OnWSReceive`

### `OnClose`

### `OnError`