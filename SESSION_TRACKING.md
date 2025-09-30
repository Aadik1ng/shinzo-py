# Session Tracking and Replay (Python SDK)

The Shinzo Python SDK now includes session tracking capabilities for detailed debugging and UX analysis of MCP server interactions.

## Overview

Session tracking automatically captures:
- Tool calls with inputs and outputs
- Tool responses with timing information
- Errors with full tracebacks
- User inputs and system messages
- Session metadata

Data is sent to the Shinzo Platform backend for chronological replay, error search, and export.

## Installation

Make sure you have httpx installed (automatically included in dependencies):

```bash
pip install shinzo[dev]
```

## Usage

### Basic Setup

```python
from shinzo import instrument_server
import asyncio

# Instrument your MCP server
observability = instrument_server(
    server,
    {
        "server_name": "my-mcp-server",
        "server_version": "1.0.0",
        "exporter_endpoint": "http://localhost:8000/telemetry/ingest_http",
        "exporter_auth": {
            "type": "bearer",
            "token": "your-ingest-token"
        },
        "enable_argument_collection": True  # Required for input/output capture
    }
)

# Enable session tracking
await observability.instrumentation.enable_session_tracking(
    resource_uuid="resource-uuid-here",
    metadata={
        "user_id": "user-123",
        "environment": "production",
        "client_version": "1.0.0"
    }
)
```

### Session Lifecycle

```python
# Start session when client connects
async def on_connect():
    await observability.instrumentation.enable_session_tracking(
        resource_uuid,
        metadata={"client_id": "client-123"}
    )

# Tool calls are automatically tracked
# ...your MCP server handles requests...

# Complete session when client disconnects
async def on_disconnect():
    await observability.instrumentation.complete_session()
    await observability.shutdown()
```

### Checking Session Status

```python
session_tracker = observability.instrumentation.get_session_tracker()

if session_tracker and session_tracker.is_session_active():
    session_id = session_tracker.get_session_id()
    print(f"Active session: {session_id}")
```

### Manual Event Tracking

```python
from shinzo import SessionEvent, EventType
from datetime import datetime

session_tracker = observability.instrumentation.get_session_tracker()

if session_tracker:
    session_tracker.add_event(
        SessionEvent(
            timestamp=datetime.now(),
            event_type=EventType.USER_INPUT,
            metadata={
                "input_source": "cli",
                "command": "search"
            }
        )
    )
```

## Event Types

Available in the `EventType` enum:

- **TOOL_CALL**: When a tool is invoked
  - Includes: tool name, input data, metadata
- **TOOL_RESPONSE**: When a tool completes successfully
  - Includes: tool name, output data, duration, metadata
- **ERROR**: When a tool encounters an error
  - Includes: tool name, error details, traceback, duration
- **USER_INPUT**: Custom user interaction events
- **SYSTEM_MESSAGE**: Custom system events

## Privacy and Data Collection

Session tracking respects the `enable_argument_collection` config setting:
- When `True`: Full inputs and outputs are captured
- When `False`: Only metadata and timing information is captured

The backend provides privacy filtering to redact sensitive parameters.

## Backend Integration

Sessions connect to these endpoints:
- `POST /sessions/create` - Initialize session
- `POST /sessions/add_event` - Add events (batched automatically)
- `POST /sessions/complete` - Mark session complete

Events are automatically batched and flushed:
- Every 5 seconds
- When the queue reaches 10 events
- When the session completes

## Complete Example

```python
import asyncio
from mcp.server import Server
from shinzo import instrument_server

async def main():
    # Create MCP server
    server = Server("my-server")

    # Set up instrumentation
    observability = instrument_server(
        server,
        {
            "server_name": "my-mcp-server",
            "server_version": "1.0.0",
            "exporter_endpoint": "http://localhost:8000/telemetry/ingest_http",
            "exporter_auth": {
                "type": "bearer",
                "token": os.getenv("INGEST_TOKEN")
            },
            "enable_argument_collection": True
        }
    )

    # Enable session tracking
    await observability.instrumentation.enable_session_tracking(
        os.getenv("RESOURCE_UUID"),
        metadata={
            "client_id": "client-123",
            "environment": "production"
        }
    )

    # Register tools
    @server.call_tool()
    async def search(query: str) -> str:
        # Tool logic here
        return f"Results for: {query}"

    # Run server
    try:
        # Your server run logic
        await server.run()
    finally:
        # Clean up
        await observability.instrumentation.complete_session()
        await observability.shutdown()

if __name__ == "__main__":
    asyncio.run(main())
```

## API Reference

### McpServerInstrumentation

#### `async enable_session_tracking(resource_uuid: str, metadata: Optional[Dict[str, Any]] = None) -> None`
Enable session tracking for this server instance.

#### `get_session_tracker() -> Optional[SessionTracker]`
Get the current session tracker instance.

#### `async complete_session() -> None`
Complete and finalize the current session.

### SessionTracker

#### `add_event(event: SessionEvent) -> None`
Manually add an event to the session.

#### `get_session_id() -> str`
Get the unique session ID.

#### `is_session_active() -> bool`
Check if the session is currently active.

#### `async complete() -> None`
Complete the session (called automatically by complete_session).

### SessionEvent

```python
@dataclass
class SessionEvent:
    timestamp: datetime
    event_type: EventType
    tool_name: Optional[str] = None
    input_data: Optional[Dict[str, Any]] = None
    output_data: Optional[Dict[str, Any]] = None
    error_data: Optional[Dict[str, Any]] = None
    duration_ms: Optional[int] = None
    metadata: Optional[Dict[str, Any]] = None
```

## Error Handling

Session tracking is designed to fail gracefully:
- Failed session starts are logged but don't crash the server
- Failed event flushes retry by re-queuing events
- Network errors are caught and logged

```python
# Session tracking errors won't crash your server
await observability.instrumentation.enable_session_tracking(
    resource_uuid,
    metadata={"user": "test"}
)
# If this fails, the server continues without session tracking
```

## Troubleshooting

### Sessions not appearing in dashboard
- Verify `exporter_endpoint` points to the backend
- Check authentication configuration
- Ensure `enable_argument_collection` is `True` for inputs/outputs
- Check backend logs for ingestion errors

### High memory usage
- Set `enable_argument_collection` to `False` to reduce payload size
- Events auto-flush every 5 seconds to prevent buildup
- Call `complete_session()` promptly to flush remaining events

### Missing events
- Events are batched and flushed periodically (5 seconds)
- Call `complete_session()` to force final flush
- Check network connectivity to backend
- Look for error messages in console output

## Dependencies

Session tracking requires:
- `httpx>=0.25.0` - HTTP client for backend communication
- `asyncio` - For async operation

Both are automatically installed with the Shinzo SDK.
