# Android Production-Issue Interview Questions (SDE2)

For PhonePe, Navi, PayPal, Amazon, and Google Android SDE2/L4 loops.

These questions usually come up in the **Android domain / specialization round**, the **hiring-manager round** ("tell me about a hard prod bug"), and **mobile system design** follow-ups ("what breaks in production?").

`[Seen]` = a real reported interview question (Glassdoor / LeetCode Discuss).
`[Common]` = a recurring real-world production issue from engineering blogs and Medium. These are a frequent basis for scenario questions.

---

## 0. The answer framework (use it for every question below)

1. **Detect & scope:** Crashlytics / Play Vitals / Sentry. Which app version, OS version, OEM, device RAM? What % of users? Did it start with a release or a server change?
2. **Mitigate first:** Halt the staged rollout, flip a remote-config / feature flag kill-switch, roll back the server change, or ship a hotfix.
3. **Root-cause:** Read the stack trace (deobfuscate it with `mapping.txt`), ANR traces, breadcrumbs/logs, and reproduce it on the same device class (a device farm if needed).
4. **Fix + verify:** Add a unit/instrumentation test, test the minified release build, and watch the crash-free rate for that cohort after rollout.
5. **Prevent:** Lint rule, CI check (Macrobenchmark, StrictMode in debug), alerting threshold, and a postmortem.

> Interviewers grade the *process* (scope → mitigate → RCA → prevent) more than the specific fix.

---

## 1. Crashes

1. **[Seen]** A fully rolled-out version crashes at startup because a **third-party SDK fails to initialize**. What do you do now, and how do you prevent it in future? *(Mitigate with a kill-switch or hotfix. Wrap SDK init in try/catch, init lazily or off the main thread, use App Startup, use staged rollouts, monitor crash rate per version.)*
2. **[Seen]** Tell me about a crash or ANR you caught through monitoring. How did you find the root cause?
3. **[Common]** The crash-free rate dropped from 99.8% to 99.2% after a release. Walk me through the investigation.
4. **[Common]** **`TransactionTooLargeException`** spikes, mostly when users background the app, and the **stack trace has no app code**. *(Binder buffer is ~1 MB per process. Large Bundle in `onSaveInstanceState` or fragment args. Fix: pass IDs instead of objects, keep state in a ViewModel/DB. Tooling: TooLargeTool.)*
5. **[Common]** A crash happens **only in release builds** with stack traces like `a.b.c()`. *(R8/ProGuard: upload `mapping.txt`, retrace. Reflection, Gson/serialization, or JNI classes stripped, so add keep rules. Test the minified build in CI.)*
6. **[Common]** A crash happens **only on specific OEMs/devices** and you can't reproduce it. *(Segment by model / OS / RAM / GPU. Firebase Test Lab or a device farm. Check OEM ROM quirks, WebView version, native `.so` ABI splits.)*
7. **[Common]** **`ForegroundServiceStartNotAllowedException`** after the targetSdk bump to 31+ (and FGS type errors on 34+). *(No starting an FGS from the background without an exemption. Use WorkManager expedited work. Declare `foregroundServiceType` + permissions.)*
8. **[Common]** **`ForegroundServiceDidNotStartInTimeException`**: `startForeground()` wasn't called within a few seconds of `startForegroundService()`.
9. **[Common]** **`IllegalStateException: Can not perform this action after onSaveInstanceState`** from fragment transactions or dialogs after an API callback. *(Lifecycle-aware collection with `repeatOnLifecycle`. Don't commit after `onStop`.)*
10. **[Common]** **NPE / `lateinit` not initialized after process death.** The user comes back hours later and the app crashes. *(Singletons and static state are gone. Use `SavedStateHandle`. Test with "Don't keep activities" or `adb shell am kill`.)*
11. **[Common]** **`ConcurrentModificationException` / race conditions** seen only under real-world load. *(Shared mutable state across coroutines. Use Mutex, immutable state, StateFlow.)*
12. **[Common]** **Native crash (SIGSEGV) with a tombstone** from a third-party `.so`. How do you symbolicate it and decide what to do?
13. **[Common]** **WebView crashes or blank screens** after a Chrome/WebView update on users' devices that you didn't ship.
14. **[Common]** The **backend changed a field type or returned null**, and the app crashes on JSON parsing for all users at once with no app release. *(Defensive parsing, nullable models, API contract tests, server-driven kill-switch, version-gated responses.)*

## 2. ANRs & jank

15. **[Seen]** Why does an ANR happen, and **how can a memory leak lead to an ANR** or a slow app? *(Leak → heap pressure → frequent GC pauses on the main thread → jank → ANR.)*
16. **[Common]** The ANR rate in Play Vitals crossed the bad-behavior threshold (0.47%). Play will reduce visibility. What's your plan? *(Read ANR traces in Vitals/Crashlytics. Look for the main thread blocked on I/O, locks, binder calls, or `SharedPreferences`.)*
17. **[Common]** ANRs point to **`QueuedWork.waitToFinish` / `SharedPreferences.apply()`** in `onStop`/`onPause`. Why does `apply()`, which is "async", cause ANRs? *(Pending writes are flushed on the main thread during lifecycle transitions. Fix: move to DataStore, or commit on a background thread.)*
18. **[Common]** ANRs from a **BroadcastReceiver** doing work in `onReceive` (about a 10 s limit). *(Use `goAsync()` or hand off to WorkManager.)*
19. **[Common]** **Deadlock / lock contention** between the main thread and a background thread (for example, a `synchronized` singleton initialized on both).
20. **[Common]** **Scroll jank in a RecyclerView or LazyColumn** on low-end devices. *(DiffUtil on the main thread, nested layouts, image decoding, missing `key`/stability in Compose, excess recomposition. Tools: Perfetto, JankStats, Layout Inspector recomposition counts.)*
21. **[Common]** ANRs that occur **only in production and can't be reproduced locally**. How do you get data? *(ANR traces, main-thread watchdog, breadcrumbs, `ApplicationExitInfo` API on Android 11+.)*

## 3. Memory / OOM

22. **[Seen]** What are memory leaks in Android? How would you **deliberately cause one**, and how would you detect it? *(Activity context in a singleton, static View, unregistered listener, Handler/inner class, binding not cleared in `onDestroyView`, `GlobalScope` coroutine. Tools: LeakCanary, heap dump in the Profiler.)*
23. **[Common]** **`OutOfMemoryError` when loading images**, concentrated on low-RAM devices. *(Downsample with Glide/Coil, `inSampleSize`, hardware bitmaps, cache size, `onTrimMemory`.)*
24. **[Common]** Memory keeps growing over a long session (a power user scrolls for 30 minutes). How do you find what's retained?
25. **[Common]** A **Fragment's view leaks** when using ViewPager or the Navigation back stack.

## 4. Startup & performance regressions

26. **[Common]** Cold start regressed from 1.2 s to 2.5 s in the last release. How do you find which change caused it? *(Macrobenchmark in CI, Perfetto trace, too many `ContentProvider`/SDK auto-inits in `Application.onCreate`, Baseline Profiles, lazy DI graph.)*
27. **[Common]** A startup crash or slowness happens **only on first launch after install/update** (DB migration, cache warm-up, dex optimization).
28. **[Common]** A **Room migration failure** in production crashes all upgrading users. *(Missing migration path, `fallbackToDestructiveMigration` dangers, migration tests with `MigrationTestHelper`.)*
29. **[Common]** APK/AAB size grew, and install conversion dropped in tier-2/3 markets. What do you do?

## 5. Background work, OEMs & notifications

30. **[Common]** **Push notifications aren't received on Xiaomi/Oppo/Vivo/OnePlus**, but work on Pixel. *(OEM battery killers, autostart off, the app treated as force-stopped. FCM high-priority messages, guide users to battery-optimization settings, dontkillmyapp.com.)*
31. **[Common]** WorkManager jobs (for example, syncing transactions) are delayed by hours or never run on some devices. *(Doze, App Standby buckets, OEM restrictions, constraints, expedited work quotas.)*
32. **[Common]** Battery drain complaints and a Play Vitals **excessive wake-lock / wake-up** warning.
33. **[Common]** Exact alarms broke after targeting Android 12/14 (`SCHEDULE_EXACT_ALARM` permission changes).

## 6. Networking, auth & data consistency

34. **[Common]** **Users get logged out randomly.** *(Token-refresh race: parallel 401s each trigger a refresh. Fix: synchronized OkHttp `Authenticator` / Mutex, single-flight refresh.)*
35. **[Common]** **The app breaks for everyone because an SSL-pinned certificate rotated or expired.** How do you avoid that outage? *(Pin multiple keys/backup pins, pin the public key not the cert, remote-config override.)*
36. **[Common]** **Retry storm:** a backend blip causes all clients to retry at once and makes the outage worse. *(Exponential backoff + jitter, circuit breaker, server-driven retry-after.)*
37. **[Common]** The app shows **stale or inconsistent data** between screens or after offline usage. *(Single source of truth such as Room + Flow, cache invalidation, conflict resolution.)*
38. **[Common]** **Wrong time-based behavior**, such as an OTP or offer expiry, because the device clock is wrong or the time zone changed. *(Use server time / `SystemClock.elapsedRealtime`.)*

## 7. Fintech-specific (PhonePe / Navi / PayPal focus)

39. **[Seen – PayPay/fintech loops]** **A user taps "Pay" twice, or the network retries, and gets debited twice.** How do you prevent it? *(Client-generated idempotency key, persisted across retries and process death. Disable the button while the request is in flight. Server dedupes by key.)*
40. **[Seen – fintech system design]** **The payment request timed out, but the backend actually succeeded.** The user retries. How do you avoid a double debit and show the correct status? *("Pending" state, poll a status API, idempotency, reconciliation, never assume failure on timeout.)*
41. **[Common]** **Process death in the middle of a payment**, for example while the user is inside a UPI app via intent or on a 3DS/bank WebView, and the result callback is lost. *(Persist the transaction ID, recover state on relaunch, query the status from the server.)*
42. **[Common]** The SMS OTP auto-read (SMS Retriever / User Consent API) stops working on some devices or after an OS update.
43. **[Common]** **Sensitive data leaked** through logs, screenshots or the recents screen, or an unencrypted cache. *(`FLAG_SECURE`, strip logs in release with R8, EncryptedSharedPreferences/Keystore.)*
44. **[Common]** Root/emulator detection or Play Integrity blocks legitimate users after an update (false positives). How do you roll it out safely?

## 8. Release & process (common hiring-manager follow-ups)

45. **[Seen-style]** **"Tell me about the hardest production bug you have solved."** Have 2 STAR stories ready: one crash/ANR, one data or concurrency bug.
46. How do you design a **staged rollout + kill-switch** strategy so a bad build reaches only 1–5% of users?
47. A bug is in a version you **can't force users off**, since old versions stay in the wild. *(Force-update / soft-update via remote config, server-side workaround, API versioning.)*
48. How do you decide whether to **hotfix now or wait** for the next release train? *(Impact × % users × severity, especially money-related bugs.)*
49. Which **metrics and alerts** would you set up for a new feature launch? *(Crash-free users, ANR rate, startup p50/p90, API error rate, funnel conversion.)*

---

## What to prepare from this list

- **2–3 personal war stories** mapped to sections 1, 2 and 7, told with the framework in section 0.
- Be able to explain the **mechanism** behind: Binder 1 MB limit, process death vs. config change, the `SharedPreferences.apply()` ANR, how R8 breaks reflection, Doze/App Standby, OkHttp `Authenticator`.
- Know the **tools by name**: Crashlytics, Play Vitals, LeakCanary, Perfetto, Android Studio Profiler, Macrobenchmark, Baseline Profiles, StrictMode, `ApplicationExitInfo`, retrace, Firebase Test Lab.

---

## Sources

- [Glassdoor – Android Engineer interview questions (third-party SDK startup crash scenario)](https://static.glassdoor.es/Interview/android-engineer-interview-questions-SRCH_KO0,16_IP929.htm)
- [Glassdoor – Sr Android Developer interview questions](https://static.glassdoor.com.br/Interview/sr-android-developer-interview-questions-SRCH_KO0,20_IP170.htm)
- [Glassdoor – ClusterDev interview (memory leak → ANR question)](https://www.glassdoor.co.nz/Interview/ClusterDev-Interview-E3982671-RVW90871648.htm)
- [Glassdoor – PayPay interview (multiple payment clicks)](https://www.glassdoor.co.uk/Interview/PayPay-Interview-E3735809-RVW104939940.htm)
- [LeetCode Discuss – Fintech system design interviews are different](https://leetcode.com/discuss/post/8420715/fintech-system-design-interviews-are-dif-dgyj/)
- [LeetCode Discuss – PayPal SDE2 Android](https://leetcode.com/discuss/interview-experience/5993279/)
- [LeetCode Discuss – Amazon Android SDE2 Toronto](https://leetcode.com/discuss/interview-experience/712272/amazon-android-sde-2-toronto-canada-june2020-offer)
- [LeetCode Discuss – Google L4 Android Framework, Oct 2025](https://leetcode.com/discuss/post/7563717/title-google-l4-swe-iii-android-framewor-jb9v/)
- [LeetCode Discuss – Google L4 Android domain interview 2026](https://leetcode.com/discuss/post/7636153/in-office-google-interview-2026-l4-andro-mh1z/)
- [Roundz – PhonePe SDE2 interview experience](https://roundz.substack.com/p/interview-experience-174-phonepe-sde2)
- [PitchnHire – Android developer interview questions](https://www.pitchnhire.com/interview-questions/android-developer)
- [Medium / ShopBack Engineering – Handle TransactionTooLargeException](https://medium.com/shopback-engineering/handle-transactiontoolargeexception-237961bd5ef8)
- [ProAndroidDev – Where does TransactionTooLargeException come from?](https://proandroiddev.com/where-does-transactiontoolargeexception-come-from-if-i-didnt-do-anything-wrong-792843835d4f)
- [Gen Digital (Avast) – How we fought with ANR rate in Android Vitals](https://www.gendigital.com/blog/insights/research/how-we-fought-with-anr-rate-in-android-vitals)
- [Skeelo Engineering – Reducing ANR rates with DataStore](https://engineering.skeelo.com/enhancing-user-experience-reducing-anr-rates-with-datastore-in-skeelo-android-app-0a276daa3c2a)
- [Dev Genius – Memory leaks & LeakCanary for interviews](https://blog.devgenius.io/memory-leaks-in-android-leakcanary-a-practical-guide-for-interviews-f79e462d14cf)
- [AppsOnAir – How to debug ANR issues step by step](https://www.appsonair.com/blogs/how-to-debug-anr-issues-step-by-step)
- [Android Developers – Foreground service troubleshooting](https://developer.android.com/develop/background-work/services/fgs/troubleshooting)
- [Android Developers – Josh startup time case study](https://developer.android.com/stories/apps/josh)
- [Embrace – Twitter Android startup regressions](https://embrace.io/?p=2450)
- [OneSignal – Manufacturers interfere with reliable notifications](https://onesignal.com/blog/manufacturers-interfere-with-reliable-notifications/)
- [Appunite – Fixing a mobile app crash for power users](https://www.appunite.com/blog/fixing-a-mobile-app-crash-for-our-power-users)
- [ProAndroidDev – Debugging native crashes in Android apps](https://proandroiddev.com/debugging-native-crashes-in-android-apps-2b86fd7113d8)
