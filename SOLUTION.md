# Solution Steps

1. Start by defining the real operational goals of the offline-first redesign: readable cached work orders, offline-capable field actions, automatic sync, and explicit conflict handling.

2. Introduce an architecture section that shifts the frontend from online-first fetching to local-first rendering, where IndexedDB is the durable client store and the server is synchronized in the background.

3. Choose the core supporting tools and justify them with trade-offs: IndexedDB plus Dexie for durable storage, TanStack Query for query orchestration, and a service worker via Workbox or vite-plugin-pwa for shell caching.

4. Design the local persistence schema in detail, including stores for work-order summaries, full work orders, outbox items, photo blobs, and sync metadata, plus retention rules for pruning old data.

5. Describe the read path for app startup, list pages, and detail pages so the UI hydrates from local data immediately and refreshes from the network only when connectivity exists.

6. Design the outbox model for offline mutations, including optimistic local updates, durable queue records, per-work-order ordering, coalescing rules, and transaction boundaries so local updates and queue inserts succeed together.

7. Specify retry and idempotency behavior, including exponential backoff with jitter, handling for 4xx versus 5xx errors, and a required idempotency key/client mutation id on every write request.

8. Write explicit conflict-resolution rules per field instead of generic last-write-wins, covering dispatcher-owned fields, status transitions, checklist items, notes, photos, and sign-off.

9. Plan the photo pipeline carefully for the 250 MB storage budget, including compression, thumbnail generation, upload queueing, eviction rules, and the rule that unsynced photos are never auto-deleted.

10. Define the technician UX for sync state with both global and per-work-order indicators, plus accessible announcements for queued, syncing, synced, failed, and conflict states.

11. Add an accessibility section that covers ARIA live regions, focus management on route and modal changes, semantic HTML for grouped controls, non-color sync indicators, and touch-target sizing.

12. Document the responsive mobile-first layout approach so the most common field actions stay reachable on phones while still scaling to tablet widths.

13. Include a security section that explains token handling trade-offs in an offline web app, ideally preferring HttpOnly refresh cookies and treating cached data as sensitive.

14. Finish with a staged MVP delivery plan, testing strategy, and operational metrics so the team can ship the redesign incrementally and measure whether it reduces technician downtime.

