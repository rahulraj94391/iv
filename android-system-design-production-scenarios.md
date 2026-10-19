# Android System-Design Production Scenarios (SDE2)

These are design-flavored production problems: race conditions, concurrency, sync, real-time and SDK design issues. Interviewers usually raise them as follow-ups in **mobile system design** or **Android domain** rounds ("…and what happens if 5 calls get a 401 at the same time?").

This file is a companion to `android-production-issues-interview.md`. It **only contains scenarios that aren't already in that file**. The one exception is the worked example below, which expands Q34 there into a full answer template.

Tags:
- `[Interview]` = reported as an interview question or design topic (Glassdoor, LeetCode, interview-prep sites).
- `[Article]` = a documented real-world issue from engineering blogs or issue trackers.
- `[Field]` = a common industry issue I'm adding from experience; no single source found.

How to answer each one: **Scenario → Why it breaks (root cause) → Design fix → Edge cases → How you'd test it.**

---

## Worked example: 5 parallel calls all get `401 Unauthorized`  `[Article]`

**Scenario:** The access token expires. Five requests are in flight, so all five get a 401 and each triggers its own refresh.

**Why it breaks:**
- You get 5 refresh calls. With **refresh-token rotation**, the first refresh consumes the refresh token, so calls 2–5 fail and the user is **force-logged out**.
- `@Synchronized` on the `Authenticator` silently fails if DI hands out **different instances**, because the lock isn't shared.

**Design:**
- An **Interceptor** only attaches the header. An **Authenticator** only handles 401s. A `TokenRepository` (a singleton) owns refresh and storage.
- **Single-flight refresh:** take a shared `Mutex` or `synchronized` lock. Inside the lock, **compare the failed request's token with the stored token**. If they differ, someone already refreshed, so just retry with the new token.
- If the refresh itself returns 401/invalid, **log out once** (guard it with an atomic flag). Don't let 5 callbacks each show a "session expired" dialog.
- Return `null` from `authenticate()` after N retries (check `response.priorResponse`) to avoid infinite loops.
- Optional: **proactive refresh** shortly before `expiresAt`, so most requests never see a 401.

```kotlin
class TokenAuthenticator(private val repo: TokenRepository) : Authenticator {   // singleton!
    override fun authenticate(route: Route?, response: Response): Request? {
        if (responseCount(response) >= 2) return null
        val failedToken = response.request.header("Authorization")?.removePrefix("Bearer ")
        val newToken = synchronized(repo) {
            val current = repo.accessToken
            if (current != null && current != failedToken) current          // already refreshed by another thread
            else repo.refreshBlocking() ?: run { repo.logoutOnce(); null }  // single refresh
        } ?: return null
        return response.request.newBuilder().header("Authorization", "Bearer $newToken").build()
    }
}
```

**Follow-ups to expect:**
- What if the refresh call itself goes through the same OkHttpClient? (Use a separate client with no authenticator, or you get recursion or a deadlock.)
- What about the multi-process case (a `:sync` process with its own token copy)?
- How does WebSocket auth expiry fit in?
- **Test:** MockWebServer firing 5 parallel requests. Assert `/refresh` is hit exactly once.

---

## A. Network layer

### A1. Multiple screens fire the same API at the same time  `[Article]`
Home, Profile and a badge widget all call `GET /user` on launch, and you get 3 identical requests.
- **Fix:** **in-flight request coalescing** in the repository. Use a keyed `ConcurrentHashMap<Key, Deferred<T>>` (remove the entry on completion), or expose one `shareIn`/`stateIn` flow from the repository. The Dropbox **Store** library was built for exactly this: duplicate requests were a major source of their data usage.
- **Edge case:** start the coroutine `LAZY` and register it *before* it runs. Otherwise the body can start before the dedupe map rejects it.
- **Follow-up:** which scope owns the shared flow? (Application or repository scope, not ViewModel scope.)

### A2. Search-as-you-type shows results for an older query  `[Article]`
The user types "ph", then "phone". The "ph" response is slower and arrives last, overwriting the right results.
- **Why:** `debounce` only reduces how many requests you send. It doesn't guarantee responses come back in order.
- **Fix:** `debounce` + `distinctUntilChanged` + **`flatMapLatest`**, which cancels the previous request. As a backstop (cancellation is cooperative), drop any result whose `query != currentQuery`.
- **Test:** control the response order. Resolve the newer request first, then confirm the older one doesn't replace it.

### A3. User B sees User A's data after logout/login  `[Field]`
- **Why:** the OkHttp HTTP cache, image cache, Room tables or in-memory singletons aren't keyed by user. Or an **in-flight request from User A completes after logout** and writes to the DB.
- **Fix:** user-scoped storage (DB per user, or a `userId` column). Clear caches on logout. **Cancel a session-scoped `CoroutineScope`** on logout. Tag writes with a `sessionId` and drop any write whose session no longer matches.

### A4. A burst of startup calls hits rate limits (HTTP 429)  `[Field]`
On app foreground, 15+ calls fire (config, profile, feed, offers, notifications…). The backend throttles, or a cold start on 2G stalls.
- **Fix:** prioritize critical calls and defer the rest. **Batch/aggregate endpoints** (BFF). Honor `Retry-After`. Use a client-side concurrency limit (`Dispatcher.maxRequestsPerHost`). Use ETag/`If-None-Match` so unchanged data costs a 304.

### A5. Cancellation leaves inconsistent state  `[Field]`
The user leaves a screen in the middle of an operation, `viewModelScope` is cancelled, and the API succeeded but the local DB write never ran (or half ran).
- **Fix:** the repository owns long-lived work in an external/application scope. Put critical commits in `withContext(NonCancellable)`. Use a Room `@Transaction` for multi-table writes. For user-intent actions that must finish, use WorkManager.

---

## B. Local data & concurrency

### B1. `SQLiteDatabaseLockedException` / "database is locked"  `[Article]`
- **Why:** writes from **separate connections or processes** (a WorkManager or service running in `android:process=":remote"`, two Room instances, a SQLCipher DB opened twice) collide. Within one connection, writes queue up. Across connections, they throw.
- **Fix:** **one Room instance** (a singleton). Avoid multi-process access, or route through a ContentProvider. Set `PRAGMA busy_timeout`, use WAL mode, and keep transactions short.
- **Follow-up:** the Room connection pool shrinks to 1 on some Samsung WAL setups, leading to "unable to grant a connection for 30 seconds" stalls.

### B2. Lost update on read-modify-write (cart quantity, wallet balance, counters)  `[Field]`
Two coroutines read `qty=1`, both write `qty=2`, and one tap is lost.
- **Fix:** an atomic SQL `UPDATE cart SET qty = qty + 1`, a `@Transaction`, a `Mutex` / single-writer actor, or a server-authoritative quantity with an optimistic UI.

### B3. Optimistic UI rollback  `[Field]`
You show "Liked" or "Added to cart" immediately, and the server later rejects it (out of stock, limit hit).
- **Fix:** keep a pending state plus the previous value, roll back on failure, and reconcile with the server response. Don't let a late failure overwrite a newer user action (version or sequence the mutation).

---

## C. Offline-first & sync

### C1. Offline edits on two devices conflict  `[Interview]`
"Design offline-first sync with conflict resolution" is a very common prompt (for example, a Splitwise-style app).
- **Options:** last-writer-wins (simple, loses data), **conflict copies** (lossless, user decides), field-level merge, or server-side version numbers / vector clocks.
- **Flow:** **push local changes first, then pull** changes since the last sync token. Paginate large change sets. The server is the source of truth.
- **Follow-ups:** tombstones for deletes, clock skew (use server versions, not device time), and how to show sync state in the UI.

### C2. Offline action queue: duplicates, ordering, poison items  `[Interview]` `[Article]`
The user creates an expense offline, edits it, then deletes it, and the queue replays all three on reconnect.
- **Duplicates:** use a **client-generated UUID** per entity or mutation so server retries are idempotent.
- **Ordering:** run the queue FIFO per entity. Don't send an "update" before the "create" is acknowledged. Collapse create+update+delete into a no-op.
- **Poison item:** a single 400 shouldn't block the whole queue forever. Use a retry cap, move it to a dead-letter state, and surface it to the user.
- **Durability:** store the queue in Room and drain it with WorkManager (`NetworkType.CONNECTED`).

### C3. Duplicate or skipped items while paginating a live feed  `[Article]`
New posts are inserted at the top while the user scrolls. With **offset** pagination the window shifts, so items show up twice or get skipped. Duplicate keys in Compose `LazyColumn` can even **crash** the app.
- **Fix:** **cursor/keyset pagination** (`after=lastId`) on a stable, indexed sort key. Use a keyed `PagingSource`. Use Paging 3 + `RemoteMediator` with Room as the single source. Dedupe by ID on insert (`OnConflictStrategy.REPLACE`).
- **Follow-up:** how do you show "5 new posts" without jumping the user's scroll position?

### C4. Push and pull race and overwrite newer data with older  `[Field]`
A silent FCM push says "order delivered", and at the same time a periodic sync returns an older snapshot ("out for delivery") that lands last.
- **Fix:** every record carries a `version`/`updatedAt` from the server. Only write if `incoming.version > local.version`. Use a single sync pipeline (the push only *triggers* a sync and doesn't write data).

---

## D. Real-time (chat, live tracking)

### D1. WebSocket reconnect storm after a server deploy  `[Article]`
Every client reconnects at the same instant (thundering herd). Exponential backoff *without jitter* still produces synchronized waves and keeps the radio awake.
- **Fix:** capped exponential backoff **+ jitter**, and reset the backoff on a successful open. **Guard against double scheduling**: `onFailure` and `onClosed` can both fire, so make sure you start only one reconnect. Reconnect only while in the foreground or when there's a reason to. Listen to `ConnectivityManager` callbacks instead of blind retries.

### D2. Chat messages out of order or duplicated after reconnect  `[Interview]` `[Article]`
- **Ordering:** a **server-assigned sequence number** per conversation. Client timestamps are only hints.
- **Resume:** on reconnect, send `lastAckedSeq`, and the server replays everything after it.
- **Duplicates:** the send is retried because the ACK was lost. Use a **client message ID** (idempotency key) and dedupe on both server and client (unique index in Room).
- **Follow-ups (seen in WhatsApp-style design rounds):** single, double and blue ticks; the recipient is offline (store-and-forward plus FCM wake-up); the app is killed; online presence when backgrounded; group-chat schema changes; why not send chat messages over FCM.

### D3. Live location (ride or delivery tracking) jumps or goes backwards  `[Interview]` `[Field]`
- **Why:** location updates arrive out of order or in bursts after a network gap, and GPS jitter.
- **Fix:** sequence or timestamp each update and drop stale ones. Interpolate or animate between points, snap to road on the server, adapt the update frequency to speed and battery, and use a foreground service with a location type for the driver app.

---

## E. Background transfers

### E1. Large upload fails on a Wi-Fi ↔ cellular switch or process death  `[Interview]` `[Article]`
For example, KYC video or document upload in fintech apps, or media upload at Amazon.
- **Fix:** **chunked + resumable** uploads (for example, the tus protocol, 5–25 MB chunks). Persist `uploadSessionId` and committed chunks in Room. On retry, **ask the server for its committed offset**, then continue. Run it in WorkManager with a network constraint (UNMETERED for big files). Use idempotent chunks with checksums, and backoff with jitter.
- **Follow-ups:** progress notifications, pausing on low battery, and expiry of the server upload session.

### E2. File downloader: the same file is downloaded twice, or partial files are corrupt  `[Interview]`
"Design a file downloader library" is a frequently asked library-design question.
- **Fix:** dedupe by URL or key (a second caller attaches to the in-flight download). Write to a `.tmp` file and rename it atomically when done. Resume with HTTP `Range`. Verify size or checksum. Check free disk space first. Set concurrency limits and priorities.

---

## F. SDK / library design follow-ups

### F1. Analytics events lost when the app is killed, or duplicated on retry  `[Interview]` `[Article]`
"Design an analytics library" is a common prompt.
- **Why:** events batched **in memory** die with the process. Retrying after a lost ACK duplicates them.
- **Fix:** **write each event to disk first** (Room or a file queue), batch-upload from disk, **delete only after the server ACKs**, and retry via WorkManager. Give each event a UUID so the server can dedupe. Cap the store size with an eviction policy. Flush when the app goes to the background.

### F2. Image loading: wrong image in a recycled row, duplicate downloads, OOM  `[Interview]`
"Design Glide/Picasso" and "implement an LRU cache" are both reported in interviews.
- **Wrong image:** a late callback lands on a recycled ViewHolder. Tag the view with the request key, **cancel on detach/rebind**, and check the key before setting the bitmap.
- **Stampede:** 20 rows request the same URL, so coalesce in-flight requests by key.
- **Memory:** an LRU memory cache sized by bytes (a fraction of `maxMemory`), plus a disk cache, then the network. Downsample to the view's size and release on `onTrimMemory`. Tie loads to the lifecycle.

### F3. Duplicate push notifications  `[Article]`
- **Why:** the server resends after a redeploy (send-state held in memory), two push SDKs parse the same payload, or a push and an in-app sync both post a notification.
- **Fix:** a stable notification **tag/ID** per entity so it updates instead of adding a second notification. FCM `collapse_key`. Client-side dedupe by message ID. On the server: idempotency keys or an outbox pattern.

### F4. A remote config / feature flag changes in the middle of a session  `[Field]`
The flag flips while the user is mid-checkout, half the screens use the old flow and half the new one, or the config fetch blocks startup.
- **Fix:** fetch in the background and **activate on the next launch** (or at a safe boundary). Freeze flag values per session or flow. Ship safe in-app defaults. Never block the first frame on a config fetch. Keep a kill-switch flag separate and read it immediately.

---

## G. Fintech-specific design (PhonePe / Navi / PayPal)

### G1. Polling a pending transaction's status without hammering the backend  `[Field]`
- **Fix:** push-first (FCM or socket), with polling as a fallback. Back off on poll intervals (2s, 4s, 8s… capped) with a max duration, then fall back to "We'll notify you". Stop polling when the screen isn't visible, and resume on return. Use a single poller per transaction ID (no duplicate pollers after rotation or recreation).

### G2. Balance or history is stale after a transaction  `[Field]`
- **Fix:** event-driven invalidation of the balance and history cache when the transaction succeeds. Show "updating…" instead of a wrong number. The server is authoritative. Never compute the balance on the client.

### G3. The same account is logged in on two devices, and one changes the PIN or password  `[Field]`
- **Fix:** server-side session revocation, plus a push or socket "session invalidated" event. The other device gets a 401 with a specific error code, so it goes to forced logout, not a token refresh. This ties back to the worked example: tell "token expired" apart from "session revoked".

---

## Sources

- [velog – Race condition fix with Mutex + token comparison (Korean)](https://velog.io/@jini_1514/Race-Condition-해결기)
- [ProAndroidDev – Handling token expiration in Ktor](https://proandroiddev.com/handling-token-expiration-in-ktor-automatic-token-refresh-for-api-calls-e79badd379dc)
- [Kotlin Slack – OkHttp Authenticator @Synchronized not working (different instances)](https://slack-chats.kotlinlang.org/t/468520/hey-all-i-m-trying-to-implement-the-okhttp-authenticator-to-)
- [Kotlin Slack – Concurrent 401s, double refresh with Ktor](https://slack-chats.kotlinlang.org/t/461638/wave-i-m-using-the-auth-client-plugin-with-bearer-provider-i)
- [Nango – Concurrency with OAuth token refreshes](https://nango.dev/blog/concurrency-with-oauth-token-refreshes)
- [ProAndroidDev – From double-click prevention to in-flight deduplication](https://proandroiddev.com/from-double-click-prevention-to-in-flight-deduplication-499976d3a00d)
- [Dropbox Tech – Store: loading Android data with coroutines](https://dropbox.tech/mobile/store-grand-re-opening-loading-android-data-with-coroutines)
- [ProAndroidDev – flatMap vs flatMapLatest](https://proandroiddev.com/understanding-flatmap-vs-flatmaplatest-a-deep-dive-061994b7ffc4)
- [How to test async search for stale-result bugs](https://macmyths.com/how-to-test-async-search-for-stale-result-bugs/)
- [Dataford – Offline-first sync with conflict resolution (interview question)](https://dataford.io/questions/how-would-you-design-offline-first-synchronization-for-a-mobile-app-with-conflict-resolution-and-eventual-consistency)
- [TechInterview – Design a mobile file sync system](https://www.techinterview.org/post/3233475071/design-mobile-file-sync-system/)
- [PracHub – Mobile system design interview guide 2026 (offline sync)](https://prachub.com/resources/mobile-system-design-interview-guide-2026-ios-android-offline-sync-and-trade-offs)
- [OpenReplay – API pagination patterns (offset vs cursor)](https://blog.openreplay.com/api-pagination-patterns/)
- [Dev Genius – Pagination in modern systems](https://blog.devgenius.io/pagination-in-modern-systems-from-offset-to-cursor-tokens-and-bidirectional-navigation-1c281b5a86b8)
- [OMR – Mobile WebSocket reconnection & chaos engineering](https://omr.it.com/blog/persistent-websocket-reconnection-mobile-chaos-engineering/)
- [OMR – Reconnection storm & idempotency keys](https://omr.it.com/blog/stateful-websocket-reconnect-idempotency-keys/)
- [Coder – WebSocket backoff + double-schedule guard commit](https://git.dragonfruit.dev/mirror/coder/commit/0d8a0af2b5246b4b2acac2e8a89005cb0931a1bd)
- [Outcome School – Android system design interviews (WhatsApp, Uber, downloader, analytics)](https://outcomeschool.com/blog/android-system-design-interviews)
- [Filestack – Large file uploads on slow mobile connections](https://blog.filestack.com/mobile-file-upload-large-files-slow-connections/)
- [FastPix – Resumable uploads for Android](https://fastpix.com/docs/upload-videos/set-up-resumable-uploads-for-android)
- [Segment – Analytics-Android disk queue](https://www.twilio.com/docs/segment/connections/sources/catalog/libraries/mobile/android)
- [Matomo SDK – Don't keep events in memory (GitHub issue)](https://github.com/matomo-org/matomo-sdk-android/issues/16)
- [ProAndroidDev – Mobile system design exercise: image library](https://proandroiddev.com/mobile-system-design-exercise-image-library-83999eb0ad3c)
- [ProAndroidDev – Mobile system design exercise: caching library](https://proandroiddev.com/mobile-system-design-exercise-caching-library-8b3b5f1433cd)
- [Glassdoor – "Design an image caching library" (Android engineer)](https://static.glassdoor.es/Interview/android-engineer-interview-questions-SRCH_KO0,16_IP944.htm)
- [Outcome School – How image loading libraries solve slow loading](https://outcomeschool.com/blog/android-image-loading-library-solve-the-slow-loading-issue)
- [OneSignal – Duplicated notifications](https://documentation.onesignal.com/docs/duplicated-notifications)
- [Debugging FCM duplicate push after redeploy](https://braindetox.kr/en/posts/fcm_duplicate_push_debug_2026.html)
- [TO THE NEW – SQLite locking and transactions in Android](https://www.tothenew.com/blog/sqlite-locking-and-transaction-handling-in-android/)
- [Google Issue Tracker – Room "database is locked" / multi-process](https://issuetracker.google.com/issues/185414040)
