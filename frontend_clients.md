# Frontent Clients

The Frontend Clients provide the user-facing interface for interacting with the Chat, Media, and Authentication services. Clients should be **lightweight**, **cross-platform**, and **developer-friendly**, reflecting the modular philosophy of the backend.

- Deliver a clean, responsive user experience for chat and media interaction.
- Provide multiple form factors (Web, Desktop, CLI) with shared logic.
- Support end-to-end encryption, token-based authentication, and offline resilience.
- Maintain a headless client mode for automated testing and integration.

## Authentication Model

All clients must authenticate **directly with the Authentication Service** rather than the Chat Service. Once a JWT is obtained, clients use it for all subsequent API calls to other services. This maintains clear separation of concerns and aligns with standard OAuth2 flows.

**Authentication Flow:**

1. User logs in via `/oauth/token` or signup flow.
2. JWT and refresh tokens are stored securely on the client.
3. Subsequent requests to Chat or Media APIs include the `Authorization: Bearer <token>` header.
4. Tokens are refreshed automatically before expiry.

## Web Application (Primary Target)

The web client is the first implementation and serves as the reference frontend.

It communicates directly with:

- The **Authentication Service** for login, signup, and token refresh.
- The **Chat Service** for rooms, messages, and sync events.
- The **Media Service** for upload, download, and preview of attachments.

**Implementation Notes**

- Could be funny using HTMX, though Vue or React will probably be better.
- Persist local session tokens in secure browser storage.
- Provide a minimal but extensible UI, focusing on functional correctness before aesthetics.
- Include developer tooling such as a `debug view` for inspecting API calls and event logs. This is mainly for our sake rather than the users.

## Desktop Application

For a native experience, the desktop client should wrap the web client using **Wails** or **Tauri**:

- **Wails** offers better Go integration for local logic or encryption handling.
- **Tauri** provides a smaller footprint, strong security sandboxing and third degree burns from the borrow checker.

This allows:

- Reuse of the web UI with additional system-level capabilities (e.g., notifications, file system access).
- Native integration for OS tray presence or quick-launch.

## Headless Client

The headless client is intended for:

- Automated testing of services.
- Load and reliability simulations.
- Continuous integration checks.
- Potential bot frameworks.

It interacts with the same APIs but without a graphical interface. This mode is critical for verifying long-polling reliability, token expiration handling, and chat sync integrity.

## CLI Client

A terminal-based client built using frameworks like **Charm.sh** would:

- Offer distilled chaos, developer-focused UX.
- Allow debugging and quick administrative access.
- Demonstrate the protocol’s simplicity and extensibility.

Although not a priority for MVP, it’s valuable for testing and developer engagement. 

