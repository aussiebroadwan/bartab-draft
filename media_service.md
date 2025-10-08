# Media Service

The media service should handle all media for the chat service. This means all media will be uploaded for messages and act like a CDN to server to other users. This service should use the **Authentication Service** to handle access. Additionally,

- Run an Asynchronous Thumbnail Engine.
- Media Store Adaptors with the default being the `Local` file system.
- Cache media using a configurable caching logic.

> **Note:** Should we allow a public flag on media?

## Endpoints

For MVP of the media service, there won't be encryption or multi-node setups. IDs of files should be ULID for lexicographically sortable 128-bit IDs that are Base32 encoded.

- `POST /v1/media` 
- `GET /v1/media/{id}` 
- `GET /v1/media/{id}`
- `HEAD /v1/media/{id}` *Cheap Preflights*
- `HEAD /v1/media/{id}/thumb` *Cheap Preflights*
- `DELETE /v1/media/{id}` *admin hard-delete*
- `POST /v1/media/{id}/quarantine` *admin soft-delete*
- `GET /healthz` Liveness → 200 OK
- `GET /readyz` Readiness when the DB and storage drivers are ready
- `GET /.well-known` Publish limits and capabilities

The preflight HEAD endpoints act as their GET counterparts but only return the headers without the body data. This will allow clients to pull metadata like the size, mime, resolution, and hash without needing to pull the whole media data. This could also allow a mark for the server to start to cache the media data for if the client then does a full GET request.

## Uploading Media

By default, media should be private and require a valid Authentication token requiring the JWT to be validated to the Authentication service. If the media post contains the `public` flag then the media can be GET without an auth token. When creating media the process will follow these steps after auth validating (and rate limiting).

1. Stream the media to a temporary file. 
2. Validate the data with the SHA256 hash, and MIME content type.
3. Check if the file is a duplicate by the `ETag="sha256:XXX..."` and public flag match.
    1. On Hit, return the existing media ID.
4. On Miss, move the temporary file to the configured storage driver. Default to `internal` file system.

Though not currently planned, it would be a good idea to have a retention policy configuration option to help cleaning up expired data, trying to access the expired media should return a `410 Gone`.

## Well Known Data

Each service has a `GET /.well-known` endpoint which contains the common resources to make it easier to use and integrate with other systems. The following is a rough draft of the Media Service well known data:

```json
{
  "base_uri": "https://media.example.com/v1/media",
  "auth_required": true,
  "max_upload_size": 10000000,
  "storage_mode": "local"
}
```

The storage mode for MVP should be `local` for the local file system. But later we may want to have S3 integrations and have a `storage_mode=s3` which will probably also require a `region=ap-southeast-2` . Currently, I drafted a default max upload size of 10MB but we may want to reduce or increase this depending on configurations.