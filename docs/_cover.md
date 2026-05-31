# Fable Ultravisor Client

> An HTTP client for the Ultravisor beacon coordinator

Authenticate, dispatch work to remote beacons, and trigger operations — synchronous JSON, binary-framed streaming, and operation triggering, packaged as a Fable service.

- **Authentication** -- POST credentials, capture the session cookie, attach it automatically
- **Synchronous Dispatch** -- Submit a work item, block on a single JSON result envelope
- **Streaming Dispatch** -- Progress events, intermediate binary, and final output over a framed octet-stream
- **Operation Triggering** -- Run a configured operation by hash; JSON or raw binary, handled transparently

[Get Started](quickstart.md)
[API Reference](api.md)
[GitHub](https://github.com/fable-retold/fable-ultravisor-client)
