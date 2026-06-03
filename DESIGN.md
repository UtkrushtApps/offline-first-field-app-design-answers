# Frontend System Design Document

## 1. Executive summary

I would redesign the mobile web frontend as an offline-first client where:

- the app shell and today’s work orders are available without connectivity
- technician actions update the UI immediately, even when offline
- every write is persisted locally to an outbox and replayed when connectivity returns
- conflicts are resolved by explicit per-field rules instead of generic last-write-wins
- photo capture works within a strict 250 MB device budget
- sync state is always visible and understandable to technicians

The current React + Axios + `useEffect` approach is too network-coupled. The new design makes local storage the durable client source of truth, with the server becoming the remote authority that the client syncs against.

Recommended additions:

- IndexedDB via Dexie for durable structured local persistence
- TanStack Query for query orchestration and invalidation on top of the local store + network sync model
- Service worker via Workbox or `vite-plugin-pwa` for shell caching and background sync hooks
- a persisted outbox/sync engine for offline mutations

This gives the team a path to ship an MVP quickly while leaving room for stronger conflict handling and operational observability.

---

## 2. Goals, non-goals, constraints

### Goals

1. Work orders loaded today remain readable offline.
2. Status updates, checklist changes, notes, photo attachments, and sign-off succeed offline.
3. Local actions survive tab close, app restart, and device reconnection.
4. Sync is automatic, retried safely, and idempotent.
5. Conflicts with dispatcher changes are resolved by clear rules.
6. UX makes offline, queued, syncing, synced, and failed states obvious.
7. Accessibility is preserved across all dynamic updates.

### Non-goals

- Building a full collaborative editor for free-form notes in v1.
- Supporting multiple signed-in users on the same device simultaneously.
- Offline login as a first-class feature in MVP.
- Arbitrary historical archive download for all work orders.

### Constraints

- Existing stack: React 18, Vite, React Router 6, Axios.
- Mobile web, often low connectivity, intermittent connectivity, or full offline.
- ~5,000 technicians, so operational simplicity matters.
- Device storage for app-managed photos capped at 250 MB.

---

## 3. High-level architecture

### Proposed client architecture

1. **UI layer**: React routes and components.
2. **Domain layer**: work-order repository, mutation creators, conflict resolution helpers.
3. **Local persistence**: IndexedDB stores for work orders, summaries, photos, outbox items, and sync metadata.
4. **Sync engine**: foreground/background process that drains the outbox when connectivity is available.
5. **Network layer**: Axios client with auth handling and idempotency headers.
6. **Service worker**: caches app shell/static assets, optionally supports background sync where available.

### Core design principle

The UI should read from local persistent state first, not from live network responses first.

That means:

- screen rendering should work from IndexedDB
- network fetches refresh IndexedDB, then the UI reacts to local changes
- user actions update local state optimistically and enqueue remote work

This is the key shift from online-first to offline-first.

---

## 4. Library and pattern choices with trade-offs

## 4.1 IndexedDB via Dexie

**Choice:** use IndexedDB with Dexie.

**Why:**

- supports much larger data than `localStorage`
- async and non-blocking
- works well for structured records, indexes, and queues
- can store `Blob`s for photo uploads

**Why not `localStorage`:**

- too small for photos and detailed work orders
- synchronous API hurts performance
- poor fit for queryable records and queue processing

**Why not Cache API as primary data store:**

- great for request/response caching, poor for relational querying and queue state
- awkward for conflict metadata and per-record sync state

**Why Dexie instead of raw IndexedDB:**

- much lower implementation complexity
- transactions and queries are easier to reason about
- reduces risk for a team currently using basic Axios hooks

## 4.2 TanStack Query

**Choice:** add TanStack Query.

**Why:**

- gives a standard query/mutation model
- enables stale/fresh state, refetch triggers, retries, invalidation, and suspense integration
- lets us keep UI hooks simple while moving sync complexity into repositories/services

**Why not keep plain `useEffect` fetching:**

- each screen would reinvent caching, retries, refetching, and race handling
- hard to scale once offline, background sync, and optimistic updates are added

**Important nuance:** TanStack Query should not be the durable offline store. IndexedDB is the durable store. TanStack Query is the in-memory orchestration layer above it.

## 4.3 Service worker with Workbox

**Choice:** add a service worker using Workbox or `vite-plugin-pwa`.

**Why:**

- caches app shell and static assets for offline startup
- can help with navigation fallback and stale-while-revalidate asset strategy
- can use Background Sync where supported

**Why not rely on browser default caching alone:**

- inconsistent for shell reliability
- no explicit offline navigation strategy
- no controllable versioning or cleanup

---

## 5. Local persistence strategy and schema

## 5.1 What is stored locally

Persist locally:

- authenticated user profile needed for rendering
- work-order summaries for list views
- full work-order records for orders opened today
- outbox mutation queue
- photo blobs and thumbnails
- sync metadata and per-record dirty/conflict flags

Do **not** persist more than needed:

- avoid storing unrelated historical data indefinitely
- avoid storing raw API responses that contain fields unused by the app

## 5.2 IndexedDB schema

Suggested Dexie schema:

```ts
import Dexie, { Table } from 'dexie'

export interface WorkOrderRecord {
  id: string
  serverUpdatedAt: string
  lastViewedAt: string
  dirty: boolean
  hasConflict: boolean
  locallyDeletedPhotoIds: string[]
  data: WorkOrder
}

export interface WorkOrderSummaryRecord {
  id: string
  scheduledDay: string
  status: WorkOrderStatus
  priority: 'low' | 'medium' | 'high' | 'critical'
  title: string
  assignedTo: string
  locationAddress: string
  scheduledTime: string
  serverUpdatedAt: string
}

export interface OutboxItem {
  id: string
  workOrderId: string
  type:
    | 'status.set'
    | 'notes.append'
    | 'checklist.set'
    | 'photo.add'
    | 'signoff.submit'
  payload: unknown
  baseVersion: string
  idempotencyKey: string
  dependsOn?: string
  status: 'pending' | 'processing' | 'failed' | 'dead_letter'
  attemptCount: number
  nextAttemptAt: number
  createdAt: number
  lastError?: string
}

export interface PhotoBlobRecord {
  id: string
  workOrderId: string
  blob: Blob
  thumbnailBlob: Blob
  mimeType: string
  sizeBytes: number
  capturedAt: string
  uploaded: boolean
  outboxItemId: string
}

export interface SyncStateRecord {
  key: string
  value: unknown
}

class FieldOpsDb extends Dexie {
  workOrders!: Table<WorkOrderRecord>
  workOrderSummaries!: Table<WorkOrderSummaryRecord>
  outbox!: Table<OutboxItem>
  photoBlobs!: Table<PhotoBlobRecord>
  syncState!: Table<SyncStateRecord>

  constructor() {
    super('fieldops')
    this.version(1).stores({
      workOrders: '&id, serverUpdatedAt, lastViewedAt, dirty, hasConflict',
      workOrderSummaries: '&id, scheduledDay, status, scheduledTime',
      outbox: '&id, workOrderId, status, nextAttemptAt, createdAt',
      photoBlobs: '&id, workOrderId, uploaded, capturedAt',
      syncState: '&key'
    })
  }
}
```

## 5.3 Retention policy

To satisfy the requirement that work orders loaded today remain readable offline:

- all work orders assigned for today are prefetched and stored locally
- any work order opened today is pinned for at least 7 days
- summaries older than 14 days are pruned
- full work-order details older than 7 days with no pending outbox entries are pruned
- synced photos are eligible for local eviction after 14 days or sooner when storage pressure is high
- unsynced photos are never auto-deleted

---

## 6. Read path and cache hydration

## 6.1 App startup

1. Service worker serves app shell.
2. App initializes auth state.
3. App hydrates screen data from IndexedDB immediately.
4. If online, background refresh begins.
5. UI indicates whether data is fresh, stale, or offline-only.

## 6.2 Work-order list flow

- read summaries from IndexedDB first
- render immediately
- if online, fetch latest summaries for today
- merge server response into IndexedDB
- update list UI from local store changes

## 6.3 Work-order detail flow

- read full work order from IndexedDB if present
- if not present and offline, show an offline-unavailable message only for never-opened records
- if online, fetch latest detail and update IndexedDB
- merge pending local mutations into the displayed record so queued changes appear immediately

### Important UX distinction

- **Previously loaded today**: fully readable offline
- **Never loaded on device**: not available offline

This matches the stated requirement without pretending the device has data it never fetched.

---

## 7. Outbox and sync queue design

## 7.1 Why an outbox

An outbox is the durable list of mutations that have been accepted by the UI but not yet confirmed by the server.

Without this, offline actions are lossy after reload and impossible to retry safely.

## 7.2 Mutation flow

For every field action:

1. validate locally
2. apply optimistic update to local IndexedDB record
3. create outbox item with idempotency key and base version
4. mark work order as dirty
5. announce queued state in UI
6. sync engine submits when online
7. on success, mark outbox item complete and reconcile local record with server response
8. on conflict or permanent validation failure, mark conflict/failed state and inform the user

## 7.3 Queue item format

Every outbox item should include:

- stable client-generated `id`
- `idempotencyKey`
- `workOrderId`
- mutation type
- payload
- `baseVersion` based on last known server version or `updatedAt`
- `attemptCount`
- `nextAttemptAt`
- current processing status

## 7.4 Per-work-order ordering

Order matters.

Example:

- set status to `in_progress`
- complete checklist item
- upload photo
- set status to `completed`
- submit sign-off

These should be processed in order for the same work order. The sync engine can process different work orders concurrently, but should preserve ordering within a single work order.

## 7.5 Coalescing rules

To reduce queue size and server load:

- multiple pending status changes for the same work order collapse to the latest valid status
- checklist toggles for the same checklist item collapse to the latest boolean value
- notes do **not** collapse if modeled as append events
- photo uploads never collapse
- sign-off never collapses into anything else and always stays last

---

## 8. Retry, backoff, idempotency

## 8.1 Retry policy

Recommended retry behavior:

- network error / timeout: retry with exponential backoff + jitter
- 5xx: retry with exponential backoff + jitter
- 429: retry using server `Retry-After` if provided, otherwise backoff
- 4xx validation error: do not blind-retry, move to failed/conflict state
- 401: pause sync, require token refresh or re-auth

Example backoff:

- attempt 1: immediate or 2 seconds
- attempt 2: 10 seconds
- attempt 3: 30 seconds
- attempt 4: 2 minutes
- attempt 5: 10 minutes
- then capped retries until manual attention or dead-letter threshold

## 8.2 Idempotency strategy

Every mutation request should carry a client-generated idempotency key, for example:

- header: `Idempotency-Key`
- body field: `clientMutationId`

This protects against:

- reconnect duplicates
- browser retry duplicates
- service worker replay duplicates
- user double taps

For photo uploads, the idempotency key should be stored alongside the local blob so the same photo is not re-sent as a duplicate after reload.

## 8.3 Sync triggers

Run sync when:

- app starts
- app regains connectivity
- tab becomes visible
- user pulls to refresh or taps retry sync
- periodic foreground timer while app is open, for example every 30 to 60 seconds
- Background Sync event if supported

The design must not rely exclusively on Background Sync because iOS Safari support is limited.

---

## 9. Conflict-resolution rules by field

A generic last-write-wins policy is too destructive. Use field-specific rules.

## 9.1 Versioning model

Each mutation is sent with:

- `baseVersion`: the server `updatedAt` or record version the client last synced against
- `clientMutationId`: unique idempotency token

If the server detects the base version is stale, it can either:

- merge automatically using the rules below, or
- return `409 Conflict` with the server record and conflict metadata

## 9.2 Field rules

| Field | Rule | Rationale | UX |
|---|---|---|---|
| `assignedTo` | Dispatcher authoritative | Technician should not edit assignment | Show banner if reassigned while offline |
| `scheduledTime` | Dispatcher authoritative | Scheduling is dispatch-owned | Refresh local card and notify technician |
| `location` | Dispatcher authoritative | Operational accuracy | Replace local field with server value |
| `priority` | Dispatcher authoritative | Business priority belongs to dispatch | Replace local field with server value |
| `status` | State-machine merge, forward progress preferred, `signed_off` final | Prevent silent rollback of completed work | Show explicit conflict if server rejects local transition |
| `notes` | Prefer append-only notes/events, not whole-document replace | Avoid clobbering multi-actor edits | Render separate note entries with author/time |
| `checklist[i].completed` | Latest value per checklist item, but reject if checklist item deleted/changed by server | Independent toggles merge well per item | Highlight invalidated item if removed by dispatcher |
| `photos` | Additive, dedupe by idempotency key or content hash | Two actors adding photos do not conflict semantically | Show all accepted photos |
| `signOffSignature` | Terminal and immutable once accepted | Legal/audit importance | If sign-off rejected, force resolution before resubmission |

## 9.3 Status conflict rules in detail

Suggested precedence:

1. `signed_off` is final. No downgrade without a separate explicit reopen workflow.
2. If server is already `signed_off`, queued local status changes become no-ops and are marked resolved.
3. If technician moved status forward offline and dispatcher changed schedule/priority only, accept technician status.
4. If technician set `completed` offline and dispatcher set `pending_parts` online from the same earlier base version, server should reject automatic merge and return conflict because intent differs materially.
5. If technician tries a transition that is invalid from the latest server state, return conflict.

This favors safety over magic.

## 9.4 Notes conflict rule

I would change the product model from a single mutable `notes` blob to **append-only notes/events**.

Why:

- whole-text merging is brittle on mobile
- append-only preserves auditability
- dispatch and technician edits can coexist naturally

If the backend cannot change immediately, the frontend MVP can still queue technician note appends with a server endpoint that appends text blocks rather than replacing the entire field.

## 9.5 Conflict UX

When conflict happens:

- do not silently discard the technician action
- keep the local pending item visible
- show a conflict badge on the work order
- provide a simple resolution panel describing:
  - what the technician attempted
  - what changed on the server
  - what rule was applied
  - whether the action was kept, dropped, or needs user review

MVP should minimize manual conflict resolution by making server rules explicit.

---

## 10. Photo upload queuing within 250 MB

Photos are the hardest offline payload because they are large and user-generated.

## 10.1 Budget allocation

Proposed 250 MB budget:

- 180 MB for unsynced/synced original photo blobs
- 20 MB for thumbnails/previews
- 20 MB for work-order metadata + outbox + headroom
- 30 MB reserved safety margin to avoid quota edge failures

## 10.2 Capture pipeline

On capture:

1. read image file
2. normalize orientation
3. resize longest edge to about 1600 px for upload unless business needs full resolution
4. compress to JPEG/WebP depending on browser support, target roughly 1.5 to 2.0 MB typical
5. generate small thumbnail for gallery
6. store original upload blob + thumbnail in IndexedDB
7. enqueue `photo.add`
8. render thumbnail immediately with local object URL

## 10.3 Limits

- max 8 photos per work order, matching current UI
- soft reject files above a large threshold before compression, for example 25 MB
- show remaining local photo storage budget in the capture UI when below 20%
- block new capture only when the app cannot safely store the photo without risking unsynced data loss

## 10.4 Eviction policy

Never evict unsynced photos.

When storage pressure is high:

1. delete local copies of already-synced photos older than 14 days
2. if still over budget, delete synced full-size blobs but keep thumbnails
3. if still over budget, prompt the user before blocking new captures

## 10.5 Upload ordering

Photo uploads can be large and slow. The sync engine should:

- prioritize non-photo metadata mutations first so critical status/checklist actions sync quickly
- upload photos with limited concurrency, for example 1 to 2 at a time
- keep sign-off blocked until required photos are synced only if the business process requires it; otherwise sign-off can queue independently

---

## 11. UX patterns for sync state

Technicians should never wonder whether the app saved their action.

## 11.1 Global sync indicators

Add a top or sticky global sync banner/chip with states:

- Online, all changes synced
- Offline, changes will sync later
- Syncing 3 changes
- 2 changes need attention

This should be visible but lightweight.

## 11.2 Per-record indicators

Each work-order card/detail page should show:

- `Available offline`
- `Queued changes`
- `Syncing`
- `Conflict needs review`
- `Last synced 2m ago`

## 11.3 Interaction principles

- actions stay enabled offline unless physically impossible
- never disable a status button solely because the network is unavailable
- after tap, immediately show `Saved offline` or `Queued for sync`
- provide a manual `Retry sync` action
- keep an outbox screen or expandable panel for advanced transparency

## 11.4 Example sync status component

```tsx
function SyncBanner({ online, queueCount, failedCount }: Props) {
  let message = 'All changes synced'
  if (!online && queueCount > 0) message = `${queueCount} changes saved offline`
  else if (!online) message = 'Offline'
  else if (failedCount > 0) message = `${failedCount} changes need attention`
  else if (queueCount > 0) message = `Syncing ${queueCount} changes`

  return (
    <section className='sync-banner' aria-label='Sync status'>
      <p aria-live='polite'>{message}</p>
      {failedCount > 0 && <button type='button'>Review</button>}
    </section>
  )
}
```

---

## 12. Accessibility strategy

Offline-first UI adds many dynamic states, so accessibility must be intentional.

## 12.1 Live regions

Use the existing polite and assertive announcers:

- polite announcements for successful local saves and sync progress
- assertive announcements for failed sync, conflicts, or storage-limit errors

Examples:

- polite: `Status changed to In Progress. Saved offline.`
- polite: `Photo added. Upload will continue when online.`
- assertive: `Sign-off could not sync because the work order was updated by dispatch.`

## 12.2 Focus management

- on route changes, move focus to the page heading
- when opening conflict resolution panels or error banners, move focus to the panel heading
- after resolving a conflict, return focus to the triggering control

## 12.3 Semantic HTML

- use `fieldset` and `legend` for grouped status actions
- use proper `button` elements for actions, not clickable `div`s
- use actual `label` + `input` relationships for forms and checklist items
- keep badges supplemental, not the only source of meaning

## 12.4 Non-visual sync state

Do not rely on color alone.

Every status indicator should include text, for example:

- green dot + `Synced`
- gray dot + `Available offline`
- amber dot + `Queued`
- red dot + `Needs attention`

## 12.5 Touch and motion

- minimum 48 x 48 px targets
- avoid auto-scrolling focus jumps during sync
- keep animations subtle and not required for meaning

---

## 13. Responsive mobile-first layout

Primary layout principles:

- single-column default for phone widths
- sticky sync banner above content
- bottom navigation remains reachable by thumb
- key work-order actions near the bottom on detail pages
- photo gallery uses responsive grid
- on tablets, optional two-pane layout for list/detail split view

Suggested layout order on mobile detail screen:

1. page header
2. sync/conflict banner
3. job metadata card
4. status actions
5. checklist
6. notes
7. photos
8. sign-off CTA

This keeps the highest-frequency tasks reachable first.

---

## 14. Token handling and security trade-offs

Offline web apps have unavoidable security trade-offs because durable offline storage and browser-executable JavaScript do not provide perfect secret protection.

## 14.1 Preferred model

Best option if backend can support it:

- short-lived access token in memory only
- refresh token in `HttpOnly`, `Secure`, `SameSite=Lax` cookie
- cached work-order data in IndexedDB
- when offline and access token expires, user can continue reading cached data and queueing actions, but sync waits until connectivity returns and refresh succeeds

This reduces XSS exposure compared with storing bearer tokens in `localStorage`.

## 14.2 If current backend must keep bearer tokens

For MVP, if bearer tokens must remain client-managed:

- keep access token lifetime short
- store the minimum needed token set
- clear tokens on logout and remote wipe events
- use strong CSP and dependency hygiene to reduce XSS risk
- treat offline cached data as sensitive and avoid storing more PII than necessary

## 14.3 Data-at-rest trade-off

Web apps cannot guarantee device-at-rest encryption beyond the OS/browser. If the business requires stronger protection:

- use MDM/device policy requirements
- consider a re-auth gate before showing cached sensitive data after long inactivity
- minimize locally stored customer data

## 14.4 Service worker security

- never cache authenticated API responses blindly in the HTTP cache
- cache app shell and safe static assets explicitly
- keep sensitive business data in IndexedDB under app control, not in opaque caches

---

## 15. Sync engine behavior

## 15.1 Pseudocode

```ts
async function processOutbox() {
  if (!navigator.onLine) return

  const items = await db.outbox
    .where('status')
    .anyOf(['pending', 'failed'])
    .toArray()

  const ready = items
    .filter(item => item.nextAttemptAt <= Date.now())
    .sort((a, b) => a.createdAt - b.createdAt)

  const groups = groupBy(ready, item => item.workOrderId)

  await Promise.all(
    Object.values(groups).map(async group => {
      for (const item of group) {
        const blocked = item.dependsOn && await db.outbox.get(item.dependsOn)
        if (blocked && blocked.status !== 'complete') continue

        try {
          await sendMutation(item)
          await markComplete(item)
        } catch (error) {
          await handleSyncError(item, error)
          break
        }
      }
    })
  )
}
```

## 15.2 Mutation application pattern

```ts
async function queueStatusChange(workOrderId: string, status: WorkOrderStatus) {
  await db.transaction('rw', db.workOrders, db.outbox, async () => {
    const record = await db.workOrders.get(workOrderId)
    if (!record) throw new Error('Work order not cached')

    record.data.status = status
    record.dirty = true

    await db.workOrders.put(record)
    await db.outbox.put({
      id: crypto.randomUUID(),
      workOrderId,
      type: 'status.set',
      payload: { status },
      baseVersion: record.serverUpdatedAt,
      idempotencyKey: crypto.randomUUID(),
      status: 'pending',
      attemptCount: 0,
      nextAttemptAt: Date.now(),
      createdAt: Date.now()
    })
  })
}
```

This transaction is important: the local optimistic update and the outbox entry must succeed or fail together.

---

## 16. Backend contract expectations

The frontend can implement a lot, but offline reliability is much better if the backend supports:

- versioned work-order responses via `updatedAt` or `version`
- idempotent mutation endpoints
- append-note endpoint rather than whole-note replacement
- batch fetch for today’s work orders
- optional delta sync endpoint since a timestamp/version
- conflict response payloads on `409`

Example conflict response:

```ts
{
  code: 'WORK_ORDER_CONFLICT',
  workOrderId: 'wo_123',
  serverVersion: '2026-06-03T12:05:00Z',
  serverRecord: { ... },
  conflictFields: ['status'],
  message: 'Status changed by dispatch while offline'
}
```

---

## 17. Staged MVP delivery plan

## Phase 1: Offline read MVP

- add service worker for shell caching
- add IndexedDB store
- cache today’s work-order summaries and opened details
- render from local store first
- basic offline banner

**Outcome:** technicians can open already-loaded work orders with no network.

## Phase 2: Offline write MVP for small mutations

- add outbox for status, checklist, and notes
- optimistic local updates
- retry + idempotency
- per-record queued indicators

**Outcome:** most daily actions work offline and sync later.

## Phase 3: Photo queueing

- local blob storage
- image compression + thumbnail generation
- storage budget management
- upload retry and progress states

**Outcome:** photo capture no longer blocks on connectivity.

## Phase 4: Sign-off and conflict UX

- offline sign-off queueing
- conflict detection and resolution UI
- dispatcher-owned field rules

**Outcome:** full field workflow works offline end-to-end.

## Phase 5: Hardening and optimization

- delta sync
- background sync where supported
- metrics dashboards
- dead-letter queue tooling
- token/security improvements

---

## 18. Testing strategy

## Unit tests

- reducers/merge functions
- outbox coalescing
- retry scheduling
- conflict-rule decisions
- storage budget calculations

## Integration tests

- open online, reload offline, still read record
- queue status/checklist/photo offline, reconnect, verify sync order
- duplicate replay sends only one server mutation
- stale base version returns conflict and shows correct UI

## Manual/device tests

- airplane mode transitions
- flaky 2G throttling
- iOS Safari background/resume
- Android Chrome storage pressure
- screen reader announcements for sync/conflict states

---

## 19. Observability and success metrics

Track:

- percentage of work-order opens served from local cache
- outbox queue depth over time
- median time from local action to server confirmation
- sync success rate by mutation type
- conflict rate by field
- photo upload failure rate
- storage budget warnings triggered
- technician-reported lost-work incidents

Primary business KPI:

- reduction in technician productivity lost to connectivity issues

---

## 20. Final recommendation

I would ship this as an IndexedDB-backed offline-first architecture with a persisted outbox, explicit conflict rules, and service-worker shell caching.

The most important design decisions are:

1. **local durable state first**
2. **optimistic writes backed by a persisted outbox**
3. **idempotent replayable mutations**
4. **field-specific conflict rules**
5. **clear sync UX**
6. **careful photo budget management**

That combination solves the real technician problem: the app remains useful in basements, rural areas, and intermittent coverage zones instead of becoming blocked by connectivity.
