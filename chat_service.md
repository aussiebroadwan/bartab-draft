# Chat Service

The Chat Service handles all core chat functionality including room creation, message distribution, and event synchronisation. It acts as the central event hub for users once authenticated via the Authentication Service.

## Endpoints

For the MVP of the chat service it should at least handle the following:

- `POST /v1/invites/redeem` *New User*
- `GET /v1/rooms` 
- `POST /v1/rooms` *admin-only*  needs room name
- `GET /v1/rooms/{id}` 
- `DELETE /v1/rooms/{id}` *admin-only*
- `GET /v1/rooms/{id}/messages` 
- `POST /v1/rooms/{id}/messages` 
- `GET /v1/sync?timeout=30000`  *Long-Poll*
- `GET /healthz` Liveness → 200 OK
- `GET /readyz` Readiness when the DB and storage drivers are ready
- `GET /.well-known` Publish limits, capabilities and auth/media service locations

These are the bare minimum endpoints required to have a functional system. This doesn't include reactions or other settings. By default, for the MVP, all users will join every channel. This will change but require additional endpoints:

- `GET /v1/me` 
- `POST /v1/rooms/{id}/join` 
- `POST /v1/rooms/{id}/leave` 
- `GET /v1/rooms/{id}/members` 
- `DELETE /v1/rooms/{id}/members/{user_id}`  *admin-only (ban/kick)*
- `POST /v1/rooms/{id}/messages/{event_id}/react` 
- `DELETE /v1/rooms/{id}/messages/{event_id}/react` 
- `POST /v1/rooms/{id}/typing`  ephemeral timeout ~3s
- `POST /v1/rooms/{id}/read-receipts` required latest event id

## Live Messages

Originally I wanted to use web sockets or even SSE but after reading the [Matrix specification](https://spec.matrix.org/latest/), I think Long Polling might be the best/easiest solution (unless a good argument can be made for the other options). 

My reasoning for Long Polling over SSE are as follows:

- Long Polling doesn't need sticky sessions or connection draining logic
- Each request should last within the timeout
- SSE has a tendency to have connections dropped after a varying time period during idle streams, so Long Polling naturally handles this issue.
- Long Polling will support batching per request rather than waiting for single live events.
- It will automatically fix itself on failover and in-flight pod fails without custom reconnection logic.

We could add SSE later if we need its performance increases. But for now we will implement a simple **GET** request on the `/v1/sync` endpoint. A timeout parameter can be set though the default will be a `30` second timeout, we should absolutely have a maximum timeout value and a minimum timeout to protect itself against attacks.

```other
GET /v1/sync?
    since=<event_id>&
    timeout=30s
```

You may have noticed a `since` parameter in the request template above. It is **required**, the server should buffer the events in the event stream and send in a batch events since that `event_id`. These event IDs can be received by the latest event received when loading the history messages and continuing on from the latest.

The timeout parameter is how long the server will hold the connection if there are no new events in the even stream. If there are events buffered then it will send as a batch and return immediately. This should have events from all rooms streamed to allow the user clients to handle all server events. Perhaps, we should have a `filter` parameter for room IDs to only receive specific room events.

## Historic Messages

To take stress off the `sync` endpoint, there should be a dedicated endpoint `/v1/rooms/{id}/messages` which should be paginated. The batching and paginated logic in this will be quite complicated and versatile:

```other
GET /rooms/{id}/messages?
    from=<event_id>&
    dir=[b|f]&
    limit=50
```

The `from` parameter should be the latest event we want to start paginating from, without this parameter it will be based of the `dir` parameter. We can either paginated `[b]ackwards` or `[f]orwards`, this will be from the `from` event and if not from an event ID it will be from the top of the room message stream or the bottom. By default, there will be a limit of `50` messages batched and there should be a maximum message `limit` so we don't open a window to crash the server.

Though these are events ordered from the event stream AOL, we probably should have reactions and read receipts aggregated per message. This will reduce inaccurate client calculations and less messages to store, though I don't really know the best way to handle this effectively.

## Encryption

Initially, the chat system will support **unencrypted message events** using the `m.plain` type. This allows us to validate core messaging logic; event storage, synchronisation, and history pagination without the complexity of cryptographic key management.

Once the plain event system is stable and verified, the next phase will introduce **end-to-end encryption (E2EE)** following the `Olm` and `Megolm` protocols. These will require beer, new event types and endpoints for key exchange and encrypted payload delivery. The [End-to-End Encryption implementation guide](https://matrix.org/docs/matrix-concepts/end-to-end-encryption) by Matrix is a good resource for this.

> **Note:** Come back to this and flesh it out when we have a solid system to implement E2EE to.