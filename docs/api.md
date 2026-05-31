# API Reference

Complete reference for `FableUltravisorClient`. All method signatures, parameters, and return shapes below are verified against the module source.

## Class: FableUltravisorClient

Extends `FableServiceProviderBase` from [fable-serviceproviderbase](https://fable-retold.github.io/fable-serviceproviderbase/).

```javascript
const libFableUltravisorClient = require('fable-ultravisor-client');

let _Client = new libFableUltravisorClient(_Fable,
{
	UltravisorURL: 'https://ultravisor.noc.example',
	UserName: 'engineer-alice',
	Password: 'hunter2'
});
```

`serviceType` is `'UltravisorClient'`. Each instance owns its own session cookie, so multiple clients with different URLs or users can coexist in one process.

### Module Exports

| Export | Description |
|--------|-------------|
| *(default)* | The `FableUltravisorClient` class |
| `.new(pFable, pOptions, pServiceHash)` | Legacy factory function that returns a new instance |

```javascript
// Legacy factory form
let _Client = libFableUltravisorClient.new(_Fable, { UltravisorURL: 'https://ultravisor.noc.example' });
```

## Constructor

```javascript
new FableUltravisorClient(pFable, pOptions, pServiceHash)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `pFable` | object | No | The Fable instance. Used for the settings fallback and logging. |
| `pOptions` | object | No | Configuration: `{ UltravisorURL, UserName, Password }` |
| `pServiceHash` | string | No | Service hash passed through to the provider base |

### Settings Resolution

For each setting, the constructor reads the constructor options first, then falls back to `fable.settings.UltravisorClient`, then to a default:

| Setting | Source order | Default |
|---------|-------------|---------|
| `UltravisorURL` | `pOptions.UltravisorURL` → `fable.settings.UltravisorClient.UltravisorURL` | `''` |
| `UserName` | `pOptions.UserName` → `fable.settings.UltravisorClient.UserName` | `''` |
| `Password` | `pOptions.Password` (if a string) → `fable.settings.UltravisorClient.Password` (if a string) | `''` |

The session cookie starts as `null` and is populated by `authenticate()`.

## Configuration Methods

### configure(pConfig)

Reconfigure the client at runtime. **Clears the current session cookie**, so you must re-authenticate after calling it.

```javascript
_Client.configure(
{
	UltravisorURL: 'https://ultravisor-staging.noc.example',
	UserName: 'engineer-bob',
	Password: 'correct-horse'
});
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `pConfig` | object | `{ UltravisorURL, UserName, Password }` — each property is applied only if it is a string |

If `pConfig` is not a non-null object, the call is a no-op. The session cookie is cleared regardless of which fields were provided. Returns nothing.

### isConfigured()

Returns `true` when the client has a non-empty `UltravisorURL`, which is the minimum needed to make a request.

```javascript
if (_Client.isConfigured())
{
	// safe to authenticate / dispatch
}
```

**Returns:** `boolean`

### getSessionCookie()

Returns the currently captured session cookie string, or `null` if not authenticated. Intended for diagnostics.

```javascript
let tmpCookie = _Client.getSessionCookie();
```

**Returns:** `string | null`

## Authentication

### authenticate(fCallback)

POSTs `{ UserName, Password }` to `/1.0/Authenticate` and captures the session cookie from the response's `Set-Cookie` header (the first cookie pair, before the first `;`). On success, the cookie is attached to every subsequent request.

```javascript
_Client.authenticate(
	function (pError)
	{
		if (pError)
		{
			console.error(pError.message);
			return;
		}
		// authenticated
	});
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `fCallback` | function | `function (pError)` |

**Failure modes** (each produces a callback error):

- `UltravisorURL` is not configured.
- `UltravisorURL` is not a valid URL.
- The response status is HTTP 400 or higher.
- The response body parses to `{ LoggedIn: false }` — the coordinator rejected the credentials. The error includes the coordinator's `Error` text when present.
- The response was HTTP 200 with no `LoggedIn: false` marker **and** no `Set-Cookie` header. This is treated as a failure so the caller never proceeds thinking it is authenticated when no session was captured.

On success the callback receives `null`. When a logger is available on the instance, a successful authentication is logged at `info` level.

## Generic Request

### request(pMethod, pPath, pBody, fCallback, pOptions)

Makes a single JSON HTTP round-trip to the coordinator. This is the low-level primitive that `dispatch()` and `getStatus()` build on; call it directly for endpoints without a dedicated method.

```javascript
_Client.request('GET', '/1.0/SomeEndpoint', null,
	function (pError, pResult)
	{
		// pResult is the parsed JSON body
	});
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `pMethod` | string | Yes | HTTP method (`GET`, `POST`, `PUT`, `PATCH`, …) |
| `pPath` | string | Yes | URL path on the coordinator |
| `pBody` | object \| null | No | Request body. Serialized to JSON and sent only when `pMethod` is `POST`, `PUT`, or `PATCH`. |
| `fCallback` | function | Yes | `function (pError, pResult)` |
| `pOptions` | object | No | `{ TimeoutMs }` to bound the socket timeout |

**Behavior:**

- Sends `Content-Type: application/json` and `Connection: keep-alive`. Attaches the `Cookie` header when a session cookie is present.
- Parses the response body as JSON (an empty body yields `{}`).
- On HTTP 400 or higher, the callback receives an `Error` using the body's `Error` field when present, otherwise `HTTP <status>`.
- On a JSON parse failure, the callback receives an `invalid JSON response` error.
- The callback fires exactly once (guarded against double-firing across response/error/timeout paths).

**Timeout:** `pOptions.TimeoutMs` bounds the socket. The default is `0` (infinite) to preserve long-running-operation behavior. When the timeout elapses before a response, the request is destroyed and the callback receives a `request timeout after <ms>ms` error.

## Work-Item Dispatch (Synchronous JSON)

### dispatch(pWorkItem, fCallback)

POSTs a work item to `/Beacon/Work/Dispatch` and waits for the synchronous JSON result envelope returned by the beacon handler.

```javascript
_Client.dispatch(
	{
		Capability: 'MeadowProxy',
		Action: 'Request',
		Settings: { Method: 'GET', Path: '/1.0/Books' },
		AffinityKey: 'customer-acme',
		TimeoutMs: 30000
	},
	function (pError, pResult)
	{
		console.log(pResult.Outputs);
	});
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `pWorkItem` | object | Yes | The [work item](#work-item-shape). Must be an object with a `Capability`. |
| `fCallback` | function | Yes | `function (pError, pResult)` |

**Validation** (each produces a callback error before any request is sent):

- `pWorkItem` is missing or not an object — `dispatch requires a work item object.`
- `pWorkItem.Capability` is missing — `dispatch work item requires Capability.`

**Socket timeout:** if the work item carries a positive numeric `TimeoutMs`, the underlying socket timeout is set to `TimeoutMs + 5000` (a 5-second grace over the coordinator-side cap) so a stuck dispatch does not hang forever. Otherwise the socket timeout is `0` (infinite).

**Result:** `pResult` is the parsed JSON body from the coordinator — typically an `Outputs` envelope produced by the beacon handler. When no beacons are registered, the coordinator returns HTTP 503 and the callback receives an error whose message contains `No Beacon workers are registered.`

## Work-Item Dispatch (Binary-Framed Streaming)

### dispatchStream(pWorkItem, pCallbacks, fCallback)

POSTs a work item to `/Beacon/Work/DispatchStream` and parses the framed `application/octet-stream` response, invoking the supplied callbacks as frames arrive. See [Binary Frame Protocol](binary-frame-protocol.md) for the wire format.

```javascript
_Client.dispatchStream(
	{ Capability: 'ImageProcessor', Action: 'Render', Settings: {}, TimeoutMs: 120000 },
	{
		onProgress: function (pProgress) { /* frame 0x01 */ },
		onBinaryData: function (pBuffer) { /* frame 0x02 */ },
		onError: function (pErrorNotice) { /* frame 0x05 */ }
	},
	function (pError, pResult)
	{
		// pResult.OutputBuffer present if final binary output streamed
	});
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `pWorkItem` | object | Yes | The [work item](#work-item-shape) to dispatch |
| `pCallbacks` | object \| null | No | `{ onProgress, onBinaryData, onError }`. Pass `null` to ignore intermediate frames. |
| `fCallback` | function | Yes | `function (pError, pResult)` — the terminal callback |

**Callbacks object:**

| Callback | Frame | Argument |
|----------|-------|----------|
| `onProgress(pProgress)` | `0x01` | Parsed JSON progress object (e.g. `{ Percent, Message, Step, TotalSteps }`) |
| `onBinaryData(pBuffer)` | `0x02` | A `Buffer` of intermediate binary data |
| `onError(pErrorNotice)` | `0x05` | Parsed JSON error notification (e.g. `{ Error }`) — non-fatal |

Each callback is optional. Malformed JSON in a progress, result, or error frame is silently ignored rather than aborting the stream.

**Behavior:**

- Errors immediately if `UltravisorURL` is unconfigured or invalid.
- A pre-stream HTTP 400+ response (before any frame) is read as JSON and surfaced as a callback error using its `Error` field when present.
- Frame `0x03` (final binary) chunks are accumulated; if any were received, they are concatenated into `pResult.OutputBuffer` (a `Buffer`).
- Frame `0x04` (result metadata) JSON becomes the base of `pResult`.
- The TCP-chunk boundary is handled: a frame split across multiple `data` events is buffered until its full payload arrives.
- If the stream ends without a result frame (`0x04`), the callback receives a `stream ended without result frame` error.
- The socket timeout is disabled (`0`) because streaming dispatch is expected to be long-running.
- The callback fires exactly once.

## Operation Trigger

### triggerOperation(pOperationHash, pParameters, fCallback)

POSTs to `/Operation/{hash}/Trigger` to run a pre-configured Ultravisor operation by hash. The hash is URL-encoded into the path. Handles both JSON and `application/octet-stream` responses transparently.

```javascript
_Client.triggerOperation('0xabc123operationhash',
	{ CustomerID: 42, ReportMonth: '2026-05' },
	function (pError, pResult)
	{
		if (Buffer.isBuffer(pResult.OutputBuffer))
		{
			// binary result
		}
		else
		{
			// JSON envelope
		}
	});
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `pOperationHash` | string | Yes | The operation hash; URL-encoded into the request path |
| `pParameters` | object | No | Seeded into the operation's `OperationState`. Defaults to `{}`. |
| `fCallback` | function | Yes | `function (pError, pResult)` |

**Request body sent:**

```javascript
{
	Parameters: pParameters || {},
	Async: false,
	TimeoutMs: (pParameters && pParameters.TimeoutMs) || 300000
}
```

The client always requests synchronous execution (`Async: false`) and waits for completion. The trigger timeout is taken from `pParameters.TimeoutMs` when present, otherwise `300000` (5 minutes).

**Response handling:**

- **`application/octet-stream`** — the body is buffered and the callback receives:

  ```javascript
  {
  	Success: true,
  	OutputBuffer: <Buffer>,
  	RunHash: <X-Run-Hash header, or ''>,
  	Status: <X-Status header, or 'Complete'>,
  	ElapsedMs: <parsed X-Elapsed-Ms header, or 0>
  }
  ```

- **Otherwise (JSON)** — the body is parsed. On HTTP 400+ the callback errors using the body's `Error` field. On HTTP < 400 with `Success` falsy, the callback errors with the first entry of `Errors` when present, otherwise `operation trigger failed`. On success the parsed JSON envelope is returned as `pResult` (typically including `Success`, `RunHash`, `Status`, `OperationState`, `TaskOutputs`, `Output`, `Errors`, `Log`, `ElapsedMs`).
- A JSON parse failure produces an `invalid response from trigger` error.

The socket timeout is disabled (`0`). The callback fires exactly once.

## Status

### getStatus(fCallback)

Reads the capabilities currently advertised by connected beacons. This is a thin wrapper over `request('GET', '/Beacon/Capabilities', null, fCallback)`.

```javascript
_Client.getStatus(
	function (pError, pResult)
	{
		console.log(pResult.Capabilities, pResult.BeaconCount);
	});
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `fCallback` | function | `function (pError, pResult)` |

**Result:** `pResult` is the `/Beacon/Capabilities` payload — `{ Capabilities, BeaconCount }` (the coordinator additionally returns an `ActionCatalog` array describing the available actions). `Capabilities` is the de-duplicated set of capability names across all registered beacons; `BeaconCount` is the number of registered beacons.

> The coordinator's `/Beacon/Capabilities` endpoint does not require a session, so `getStatus()` works as a health check even before `authenticate()`.

## Work Item Shape

A work item describes a unit of work for a beacon to perform:

```javascript
{
	Capability: 'SomeCapability',           // required
	Action: 'SomeAction',                   // required in practice
	Settings: { /* action-specific */ },    // payload for the action
	AffinityKey: 'stable-routing-string',   // routes to a sticky beacon
	TimeoutMs: 30000                         // coordinator-side cap
}
```

| Field | Type | Notes |
|-------|------|-------|
| `Capability` | string | **Required.** The named capability a beacon must advertise. |
| `Action` | string | Required in practice — the action within the capability. The coordinator defaults it to `'Execute'` if omitted. |
| `Settings` | object | Action-specific payload handed to the beacon provider. |
| `AffinityKey` | string | Routes repeated work to the same beacon, enabling source-file caching across work items. |
| `TimeoutMs` | number | Coordinator-side cap on execution time. The coordinator defaults it to `300000` if omitted. |

The same work-item shape is consumed by the beacon's providers on the other end of the wire. See [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) for how a beacon resolves `Capability:Action` to a handler and what an `Outputs` result envelope looks like.

## Callback Convention

Every asynchronous method uses the Node.js error-first callback convention:

```javascript
function (pError, pResult)
{
	if (pError)
	{
		// handle the error
		return;
	}
	// use pResult
}
```

`authenticate()` and `configure()`-adjacent flows that have no result pass only `pError`.
