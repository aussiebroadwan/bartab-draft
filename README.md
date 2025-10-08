# BarTAB Chat - Design Draft

The project aims to build a lightweight group chat system that is self hostable, modular, and developer-friendly. Each server runs as an independent instance, capable of managing its own users, rooms, and media, while remaining minimal and performant.

At its core, the platform is built around three principles:

- **Append Only Logs (AOL):** for reliable event synchronisation.
- **Client-side cryptography:** for privacy and integrity with E2EE using Olm/Megolm. Though for MVP we might just do plaintext to ensure we have a working system.
- **Composable micro services:** that communicate over clean, versioned APIs.

> **Note:** I don't typically like micro services, but I think the way this is split will work well for future multi-tenant implementing.

## Key Concepts

- **Bot-First Architecture:** Bots are first-class citizens. A server SDK allows bots or automation services to interact with the same APIs as clients.
- **Media as BLOBs:** Media uploads are stored through a dedicated Media Service that supports thumbnails, caching, and future CDN integrations.
- **Simple IAM & ACLs:** Lightweight identity and access control built on JWTs, with planned OpenID Connect (OIDC) integration.
- **Efficient Sync:** Long polling via `/sync` replaces WebSockets or SSE to simplify scaling, recovery, and stateless operation.

Deploy as Docker Containers, will need to support SQLite3 DB first then probably add a Postgres driver later. Ideally we can plan some form of database backup scheme as well for the SQLite3 driver.

Having a docker container deployment model, we can allow server bots to be containers running in the same stack. This could allow bots to be directly connected to the server on an internal virtual network across containers, though this being said it would be greatly important to have external access as well.

## Core Components

The system is composed of three cooperating services, each independently deployable but tightly integrated through the Authentication Service.

1. **Authentication Service**: Handles identity, token issuance, and permission scopes. It is the root of trust for all other services and provides public keys for JWT validation via a `.well-known/jwks.json` endpoint.
2. **Chat Service**: Manages rooms, messages, events, and synchronisation. Implements an **Append Only Log** for message consistency and supports long polling through `/v1/sync` for real-time updates.
3. **Media Service**: Handles upload, storage, caching, and delivery of media assets. Acts like a simple CDN and integrates authentication for private or public access modes.

All services expose **health**, **readiness**, and **.well-known** endpoints for operational monitoring and discovery.

![Image.png](assets/example_arch.png)

The authentication service is a core part. The two other services shouldn't be able to run without it. It should have a public endpoint `jwks.json` which contains public keys to validate JWT keys. Consider The following sequence chart.

![Image.png](assets/example_sequence.png)

## Request for Discussion (RFD) Process

I've been following some of the design decisions that [Oxide](https://oxide.computer/) do. I like the idea of new features or architectural ideas begining as a **Request for Discussion (RFD)**:

1. Create a new markdown file in `/rfd/` (e.g. `RFD-0001-feature-x.md`).
2. Share it in the dev channel for review and async discussion.
3. Iterate until accepted or superseded.

[Oxide Example](https://rfd.shared.oxide.computer/rfd/0001)

> **Note**: This might be up for vote if people want to do this, otherwise we will just chill with GH issues.

## Project Maturity

This repository is a **draft workspace**. Once we finalise architecture and key decisions:

- The production repo will be created.
- Documentation will move to `/docs/`.
- A static docs site (VitePress or Fumadocs) will publish everything.
- CI/CD, linting, and migrations will be automated.

Until then, focus is on:

- Reviewing architecture and service definitions.  
- Refining the developer experience and deployment model.