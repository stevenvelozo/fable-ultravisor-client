# Binary Frame Protocol

The streaming dispatch endpoint, `/Beacon/Work/DispatchStream`, returns a chunked `application/octet-stream` response composed of length-prefixed binary frames. This protocol carries progress events, intermediate binary chunks, a final binary output, a result envelope, and non-fatal error notices over a single HTTP response.

The protocol is versioned `binary-frames-v1`. The coordinator advertises it on the response with the header `X-Ultravisor-Protocol: binary-frames-v1`, alongside `Content-Type: application/octet-stream` and `Transfer-Encoding: chunked`.

This page documents the format as implemented by this client's parser in `dispatchStream()` and produced by the coordinator's frame writer.

## Frame Format

Every frame is a 5-byte header followed by a payload:

```
+--------+------------------------+-------------------+
| 1 byte | 4 bytes                | N bytes           |
| type   | payload length (uint32 | payload           |
|        | big-endian)            |                   |
+--------+------------------------+-------------------+
```

- **Byte 0** — the frame *type* (one of the codes below), read as an unsigned 8-bit integer.
- **Bytes 1–4** — the payload length `N`, an unsigned 32-bit integer in **big-endian** byte order.
- **Bytes 5 … 5+N** — the payload. Its interpretation (JSON text or raw bytes) depends on the type.

Frames are written back-to-back on the stream. There is no delimiter between frames; the length prefix is what bounds each one.

## Frame Types

| Code | Name | Payload | Client handling |
|------|------|---------|-----------------|
| `0x01` | Progress | JSON, e.g. `{ Percent, Message, Step, TotalSteps }` | Parsed and passed to `onProgress`. |
| `0x02` | Intermediate binary data | Raw bytes | Passed to `onBinaryData` as a `Buffer`. |
| `0x03` | Final binary output | Raw bytes | Accumulated; concatenated into `pResult.OutputBuffer`. |
| `0x04` | Result metadata | JSON result envelope | Parsed and used as the base of `pResult`. |
| `0x05` | Error | JSON, e.g. `{ Error }` | Parsed and passed to `onError`. Non-fatal — the stream continues. |

There is no on-the-wire frame for "end of stream"; the transport-level end of the chunked response signals completion. (On the coordinator side, an internal `end` signal simply closes the response.)

## How the Client Parses the Stream

`dispatchStream()` maintains a growing buffer and drains complete frames from the front of it:

1. Incoming `data` chunks are appended to an internal buffer.
2. While the buffer holds at least 5 bytes (a full header), the payload length is read from bytes 1–4.
3. If the buffer does not yet contain the full `5 + N` bytes, parsing pauses and waits for more data — this is how a frame **split across two TCP chunks** is handled correctly.
4. Once a full frame is present, its type and payload are sliced off, the buffer is advanced past it, and the frame is dispatched by type.

This means callers do not need to worry about chunk boundaries: a single frame may arrive in pieces, or several frames may arrive in one chunk, and the parser handles both.

## Terminal Result

When the response ends:

- If a result frame (`0x04`) was seen, its parsed JSON becomes `pResult`. If any final-binary frames (`0x03`) were received, their bytes are concatenated and attached as `pResult.OutputBuffer` (a `Buffer`). The terminal callback fires with `(null, pResult)`.
- If the stream ends **without** a result frame, the terminal callback fires with an error whose message contains `stream ended without result frame`.

Malformed JSON inside a `0x01`, `0x04`, or `0x05` frame is silently ignored — a bad progress, result, or error frame does not abort the stream.

## Error Responses Before the Stream

If the coordinator rejects the request before streaming begins (HTTP 400 or higher — for example a missing `Capability`, or HTTP 503 when no beacons are registered), it returns a normal JSON body rather than frames. The client detects the non-2xx status, reads the JSON body, and surfaces it as a callback error using the body's `Error` field when present. No frame parsing occurs in that case.

## Encoding a Frame

A frame is built by writing the type byte, the big-endian length, then the payload. For a JSON-payload frame the payload is the UTF-8 bytes of the JSON string; for a binary-payload frame it is the raw bytes. The following mirrors the encoder used in the module's tests:

```javascript
const libBuffer = require('buffer').Buffer;

function encodeFrame(pTypeCode, pPayload)
{
	let tmpPayloadBuffer = libBuffer.isBuffer(pPayload) ? pPayload : libBuffer.from(pPayload);
	let tmpHeader = libBuffer.alloc(5);
	tmpHeader.writeUInt8(pTypeCode, 0);
	tmpHeader.writeUInt32BE(tmpPayloadBuffer.length, 1);
	return libBuffer.concat([ tmpHeader, tmpPayloadBuffer ]);
}

// A progress frame (type 0x01) with a JSON payload:
let tmpProgressFrame = encodeFrame(0x01, JSON.stringify({ Percent: 25, Message: 'working' }));

// A final-binary frame (type 0x03) with raw bytes:
let tmpBinaryFrame = encodeFrame(0x03, libBuffer.from([ 0xBE, 0xEF ]));
```

## Worked Example

A typical streamed dispatch might place these frames on the wire, in order:

1. `0x01` — `{ "Percent": 25, "Message": "working" }` (progress)
2. `0x02` — raw intermediate bytes
3. `0x03` — raw final-output bytes
4. `0x04` — `{ "Success": true, "RunHash": "r1" }` (result metadata)

After the response ends, the client delivers:

- one `onProgress({ Percent: 25, Message: 'working' })` call,
- one `onBinaryData(<Buffer>)` call for the intermediate bytes,
- and a terminal `fCallback(null, { Success: true, RunHash: 'r1', OutputBuffer: <Buffer> })`, where `OutputBuffer` holds the concatenated `0x03` bytes.

## Relationship to the Beacon

The frames are produced by the Ultravisor coordinator, not by the beacon directly. As a beacon executes a work item, it reports progress and emits output to the coordinator (over WebSocket or HTTP); the coordinator re-encodes those events as `binary-frames-v1` frames on the open `DispatchStream` response to this client. See [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) for how a beacon reports progress and returns results on its side of the coordinator.
