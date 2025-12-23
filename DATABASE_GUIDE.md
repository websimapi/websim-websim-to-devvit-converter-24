# WebSim to Reddit (Redis) Database Migration Guide

This project uses a custom bridge to map the **WebSim Database (Socket-based)** to **Reddit's Redis storage**. This allows games built with WebSim's `websim.collection` or `WebsimSocket` APIs to work seamlessly on Reddit with persistent data.

## Architecture Overview

The migration works through a three-tier pipeline:

1.  **Frontend (Polyfill):** A client-side script (`websim_polyfills.js`) replaces the global `websim` and `WebsimSocket` objects.
2.  **Bridge (Express API):** An Express.js server (`src/server/index.ts`) acts as a secure intermediary between the Webview and Reddit's Redis.
3.  **Storage (Redis):** Data is stored in Reddit's server-side Redis instance using Hashes and Sorted Sets.

## How Data is Mapped

### Collections ➔ Redis Hashes
Every WebSim "collection" (e.g., `room.collection('posts')`) is mapped to a **Redis Hash**.
- **Hash Key:** The collection name (e.g., `posts`).
- **Fields:** The unique record IDs.
- **Values:** JSON-stringified record objects.

### The Registry Pattern
Because Redis doesn't allow "listing all keys" efficiently in a production environment, we use a **System Registry Key** (`sys:registry`).
- Every time a new collection is created, its name is added to a Redis Sorted Set.
- On app startup, the server reads this registry to find all active collections and "hydrate" the client.

## The Generic Pipeline

### Getters (`getList`, `filter`)
- **Hydration:** When the game loads, it calls `/api/init`. The server fetches every record from every collection in the registry and sends it to the client.
- **Local Cache:** The client stores this in `window._genericDB`. Calls to `getList()` or `filter()` are served instantly from this local cache for high performance.
- **Filtering:** The `.filter({ key: value })` method is polyfilled to perform client-side filtering on the cached data.

### Setters (`create`, `update`, `delete`)
- **Optimistic Updates:** When you call `.create()`, the polyfill immediately updates the local cache and triggers any active `.subscribe()` callbacks so the UI feels instant.
- **Async Persistence:** It then sends a POST request to `/api/save` (or `/api/delete`). The Express server validates the request and commits the change to Reddit's Redis.

## Leaderboards & Routing
If your WebSim game uses a leaderboard:
1.  It likely calls `websim.collection('scores').getList()`.
2.  Our bridge returns the full list of scores stored in Redis.
3.  Your existing JS code likely handles the sorting (e.g., `scores.sort((a,b) => b.points - a.points)`). This remains unchanged and works perfectly.

## Current Limitations & Unimplemented Features

While the bridge is highly compatible, there are a few areas where it differs from native WebSim:

1.  **Cross-Client Real-time Sync:** 
    - *WebSim:* If User A posts, User B sees it instantly.
    - *Our Bridge:* Currently, real-time sync is **local only**. User B will see User A's changes only after a page refresh or if you implement a polling mechanism.
    - *Future Fix:* Implementing a long-polling or Server-Sent Events (SSE) route in the Express server.

2.  **Server-Side Ownership Enforcement:**
    - *WebSim:* Automatically prevents users from editing records they didn't create.
    - *Our Bridge:* The client-side polyfill mimics this behavior, but the Express server currently accepts any valid save request. For production apps with competitive leaderboards, you should add `context.userId` validation in `src/server/index.ts`.

3.  **Complex Queries:**
    - The `.filter()` polyfill only supports basic equality checks (e.g., `{ type: 'warrior' }`). Range queries or complex logic passed to filter may not work unless handled in your game's JS.

4.  **Binary Uploads:**
    - `websim.upload()` is stubbed to return `blob:` URLs. These are temporary and will not persist across sessions. For permanent asset uploads, you must use a dedicated S3-compatible storage or Reddit's media uploaders (not currently bridged).

## Implementation Details for Developers

If you need to manually interact with the database from the server-side, you can use the `redis` object in `src/server/index.ts`:

```typescript
// Example: Manual high-score check on the server
const score = await redis.hGet('leaderboard', userId);