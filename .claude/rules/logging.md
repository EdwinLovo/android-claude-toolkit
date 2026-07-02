---
name: logging
description: Use Timber for logging; never android.util.Log; no TAG constant
paths:
  - "**/*.kt"
---

# Logging

Use `timber.log.Timber` for all logging in production code. `android.util.Log` is banned outside test code.

## Basic usage

```kotlin
import timber.log.Timber

Timber.d("Refresh started")
Timber.w("Cache miss for key=%s", key)
Timber.e(exception, "Failed to load %s", resourceId)
```

- **No `TAG` constant** — Timber auto-derives the tag from the call site's class name via a stack trace
- **Use printf-style format args** (`%s`, `%d`) rather than string concatenation — Timber lazily evaluates them, skipping the toString() work in production builds where the log level is disabled
- **Pass the throwable as the first arg to `Timber.e(...)`** — do not just log `e.message`, the stack trace is what you actually want

## Tree setup

Plant a `Timber.DebugTree()` in your `Application.onCreate()` for debug builds. In release builds, plant a custom tree that forwards to your crash reporter (Crashlytics, Sentry) and swallows `Timber.d` / `Timber.v`:

```kotlin
override fun onCreate() {
    super.onCreate()
    if (BuildConfig.DEBUG) {
        Timber.plant(Timber.DebugTree())
    } else {
        Timber.plant(CrashlyticsTree())
    }
}
```

## Log levels

| Level | When |
|---|---|
| `Timber.v` | Fine-grained trace, off in release |
| `Timber.d` | Debug — dev-only diagnostics |
| `Timber.i` | Info — noteworthy but not a problem |
| `Timber.w` | Warning — unexpected but recoverable |
| `Timber.e` | Error — recoverable failure; include throwable |
| `Timber.wtf` | Assertion failed — something is very wrong |

## Hard rules

- **No `import android.util.Log`** in production code
- **No `Log.d(TAG, ...)` / `Log.e(TAG, ...)`** — replace with `Timber.d(...)` / `Timber.e(...)`
- **No `private const val TAG = "..."`** in a Kotlin file — Timber doesn't need it
- **Do not log user-identifiable data** (email, name, session token) at any level in release builds — the release tree should scrub or drop those. Prefer logging IDs and status codes.
- **No `println(...)` in production code** — use Timber

## Common violations

- `Log.d(TAG, "loaded ${item.name}")` → `Timber.d("loaded %s", item.name)`
- `Log.e(TAG, "boom", e)` → `Timber.e(e, "boom")` (exception first)
- `private const val TAG = "MyViewModel"` at the top of the file → delete
- `println("debug: $x")` → `Timber.d("debug: %s", x)` (or delete if it was a leftover)
- Logging inside a tight recomposition or a `LazyColumn` item — same rule as any perf-sensitive code: guard with a level check or move up
