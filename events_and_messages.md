# Events and Messages

The Chat Service is built around an Append Only Log (AOL) of immutable events. Every action that occurs on the server - message sends, joins, leaves, reactions, and state updates - is represented as a discrete event.

> **Note:** This design make the system deterministic and also is a precursor design to set it up for a Federation server pattern later.

Each event is a JSON object containing metadata and content fields. All events are immutable and stored in lexicographically ordered **ULIDs**, ensuring monotonic sort order and distributed uniqueness.

```json
{
  "event_id": "01HZX1WNYVK2T69GPKJZV8Q5E3",
  "type": "m.room.message",
  "sender": "user_ulid",
  "room_id": "room_ulid",
  "timestamp": "2025-10-07T00:00:00Z",
  "content": { ... }
}
```

| Field       | Type    | Description                                        |
| ----------- | ------- | -------------------------------------------------- |
| `event_id`  | ULID    | Unique, lexicographically sortable ID for ordering |
| `type`      | String  | Event type name                                    |
| `sender`    | ULID    | The user who generated the event                   |
| `room_id`   | ULID    | Target Room                                        |
| `timestamp` | RFC3339 | UTC Timestamp of the event                         |
| `content`   | Object  | Event specific payload                             |

> **Note:** The type is prefixed with the namespace of `m` this is an artefact from the Matrix Specification. We may want to set our own one instead.

## Event Types

Below are some drafts of event types we may use:

**Room State Events**

- `m.room.create`: Initial creation event for a room
- `m.room.name`: Setting the room name
- `m.room.member`: Member state events join, leave, ban, and kick

**Message Events**

- `m.room.message`: Text or media message
- `m.room.redaction`: Removes content from a previous event
- `m.room.reaction`: Reaction event referencing another message
- `m.room.read_receipt`: Indicates a user has read up to a given event
- `m.typing`: Ephemeral typing indicator (not stored on disk, but sent to other members as a UX action).

## Message Payloads

The `m.room.message` event type is a bit complex when it comes to its payload as it depends on the message type. Our priority is to get the `msgtype=m.plain` working first which only requires a `text` field. But once we have a working system we will implement the `msgtype=m.encrypted` type which requires the below.

```json
{
  "type": "m.room.message",
  ...
  "content": {
    "msgtype": "m.encrypted",
    "algorithm": "...",
    "ciphertext": "...",
    "session_id": "...",
    "device_id": "..."
  }
}
```

The example message above would be what you would expect from a "normal message" (as we plan on doing E2EE). The content object will change on based on the `msgtype`. Another example is for sending a image.

```json
{
  "type": "m.room.message",
  ...
  "content": {
    "msgtype": "m.image",
    "url": "https://media.example.com/v1/media/XXXXXXXXXXXXXXX",
    "alt_text": "...",
    "info": {
      "mimetype": "image/jpeg",
      "w": 640,
      "h": 480
    }
  }
}
```