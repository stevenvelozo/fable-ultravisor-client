# Quick Start

This guide walks through configuring the client, authenticating against an Ultravisor coordinator, and dispatching work.

## Prerequisites

- Node.js
- An Ultravisor coordinator reachable over HTTP or HTTPS
- Valid credentials (a `UserName` and `Password`) for that coordinator
- At least one [beacon](https://stevenvelozo.github.io/ultravisor-beacon/) registered with the coordinator that advertises the capability you intend to dispatch

## Installation

```bash
npm install fable-ultravisor-client
```

## 1. Construct the Client

The client is a Fable service. It is constructed with a Fable instance and an options object.

```javascript
const libFableUltravisorClient = require('fable-ultravisor-client');

let _Client = new libFableUltravisorClient(_Fable,
{
	UltravisorURL: 'https://ultravisor.noc.example',
	UserName: 'engineer-alice',
	Password: 'hunter2'
});
```

`UltravisorURL` is the only setting required to make requests. `UserName` and `Password` are used by `authenticate()`; `Password` defaults to an empty string if omitted.

### Settings Fallback

If a setting is not passed in the options object, the client falls back to `fable.settings.UltravisorClient`. This lets you configure the client through Fable's settings chain instead of (or in addition to) the constructor:

```javascript
const libFable = require('fable');

let _Fable = new libFable(
{
	Product: 'MyApp',
	UltravisorClient:
	{
		UltravisorURL: 'https://ultravisor.noc.example',
		UserName: 'engineer-alice',
		Password: 'hunter2'
	}
});

// No options needed — settings come from fable.settings.UltravisorClient
let _Client = new libFableUltravisorClient(_Fable, {});
```

Options passed to the constructor take precedence over the `fable.settings.UltravisorClient` fallback.

## 2. Authenticate

`authenticate()` POSTs the configured credentials to `/1.0/Authenticate` and captures the session cookie from the response. Every subsequent request attaches that cookie automatically.

```javascript
_Client.authenticate(
	function (pError)
	{
		if (pError)
		{
			console.error('Authentication failed:', pError.message);
			return;
		}
		console.log('Authenticated. Session cookie captured.');
	});
```

Authentication is strict: the client treats a missing session cookie as a failure even on an HTTP 200, and surfaces the coordinator's `LoggedIn: false` rejection as an error. See [API Reference](api.md) for the exact failure modes.

## 3. Dispatch a Work Item (Synchronous JSON)

Once authenticated, dispatch a work item and wait for the JSON result. The work item names a `Capability` and `Action` that a registered beacon can perform.

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
		if (pError)
		{
			console.error('Dispatch failed:', pError.message);
			return;
		}
		console.log('Outputs:', pResult.Outputs);
	});
```

`Capability` is required. `AffinityKey` routes repeated work to the same beacon (useful for caching). `TimeoutMs` is the coordinator-side cap on how long the work may run.

If no beacons are registered, the coordinator responds with HTTP 503 and the callback receives an error containing the text `No Beacon workers are registered.`

## 4. Stream a Work Item (Binary Frames)

For long-running work that emits progress or binary output, use `dispatchStream()`. It POSTs to `/Beacon/Work/DispatchStream` and parses the framed response, invoking your callbacks as frames arrive.

```javascript
_Client.dispatchStream(
	{
		Capability: 'ImageProcessor',
		Action: 'Render',
		Settings: { Width: 1920, Height: 1080 },
		AffinityKey: 'render-farm-a',
		TimeoutMs: 120000
	},
	{
		onProgress: function (pProgress)
		{
			console.log('Progress:', pProgress.Percent, pProgress.Message);
		},
		onBinaryData: function (pBuffer)
		{
			// Intermediate binary chunk (frame type 0x02)
			console.log('Got intermediate chunk:', pBuffer.length, 'bytes');
		},
		onError: function (pErrorNotice)
		{
			// Non-fatal error notification (frame type 0x05)
			console.warn('Stream error notice:', pErrorNotice.Error);
		}
	},
	function (pError, pResult)
	{
		if (pError)
		{
			console.error('Stream failed:', pError.message);
			return;
		}
		// pResult.OutputBuffer is a Buffer if final binary output streamed
		console.log('Final result:', pResult);
	});
```

The callbacks object is optional — pass `null` if you only care about the final result. See [Binary Frame Protocol](binary-frame-protocol.md) for the frame format.

## 5. Trigger an Operation

To run a pre-configured Ultravisor operation by hash, use `triggerOperation()`. The parameters seed the operation state. The response is handled transparently whether it is JSON or a raw binary buffer.

```javascript
_Client.triggerOperation('0xabc123operationhash',
	{
		CustomerID: 42,
		ReportMonth: '2026-05'
	},
	function (pError, pResult)
	{
		if (pError)
		{
			console.error('Operation failed:', pError.message);
			return;
		}

		if (Buffer.isBuffer(pResult.OutputBuffer))
		{
			// Binary result (e.g. a generated file)
			console.log('Binary output:', pResult.OutputBuffer.length, 'bytes');
		}
		else
		{
			console.log('JSON result:', pResult);
		}
	});
```

## 6. Check Beacon Status

`getStatus()` reads the capabilities currently advertised by connected beacons. It does not require authentication on the coordinator side.

```javascript
_Client.getStatus(
	function (pError, pResult)
	{
		if (pError)
		{
			console.error(pError.message);
			return;
		}
		console.log('Capabilities:', pResult.Capabilities);
		console.log('Beacon count:', pResult.BeaconCount);
	});
```

## Next Steps

- [API Reference](api.md) — Every method with verified signatures and options
- [Binary Frame Protocol](binary-frame-protocol.md) — The wire format behind streaming dispatch
