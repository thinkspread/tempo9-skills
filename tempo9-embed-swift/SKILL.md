---
name: tempo9-embed-swift
description: Embed the Tempo9 engine in-process in a Swift app instead of running a server — own the session lifetime correctly, warm up, switch models, pair the Metal kernels with the engine build, and fall back gracefully. Use when building a macOS/iOS app that ships a local model, when a second engine appears in memory, or when in-process output comes out as garbage.
---

# Embedding Tempo9 in a Swift app

## Overview

Tempo9Kit lets an app hold the engine in its own process — no server, no
localhost, no second process holding a copy of the weights. That is the right
choice for a shipping app, and it moves a set of responsibilities onto you
that a server would have handled. This skill is those responsibilities.

## Packages

The SDK exposes **one public library: `Tempo9`**. An app writes
`import Tempo9` and gets `LocalSession` — chat, tool calls, vision, prefix
cache. The engine binding and the C shim beneath it are deliberately
internal: exporting them would make every symbol below the SDK a permanent
API promise.

```swift
.package(url: "https://github.com/thinkspread/tempo9", branch: "main")
```

(Pin a version once the first release is tagged.)

```swift
import Tempo9
```

`LocalSession`, `SamplingConfig`, `EngineStats` and `ImageEmbeddings` all
come from that one import. (They did not always: the three engine types are
re-exposed by the SDK precisely because `stream(config:)` is public while its
parameter type was not nameable from outside the package.)

The engine is **not in the package source**. It ships as a binary with
Tempo9's first release, which is not out yet: until then the package
compiles, and an app that imports it cannot link.

## Declare macOS 26, or Metal turns itself off

A SwiftPM executable records its declared platform as its SDK version, and
the Metal runtime compiler finds the Metal 4 headers the engine's kernels use
by that SDK version. Declared lower, the engine's tensor-API probe kernel
fails to compile ("use of undeclared identifier 'mpp'"), the whole Metal
backend switches itself off, and the engine runs on the CPU with one log
line to show for it. Measured on an M5 Pro, 2026-09-29, three runs of each,
Qwen3.5-9B Q4_K_S: a 1,004-token prompt took 14 s to prefill that way against
0.8 s with the platform declared, and decode ran at 23 tok/s against 52.

```swift
platforms: [.macOS("26.0")],
```

Confirm it at startup rather than inferring it from speed: the engine logs
`Metal backend READY` when the backend is up, and `Metal backend disabled`
with the reason when it is not.

## One engine, shared, for the process lifetime

A session takes **minutes** to load and holds gigabytes resident. Anything
that constructs one per request reloads the model per request. Own it in one
place:

```swift
actor EngineHost {
    static let shared = EngineHost()
    private var session: LocalSession?
    private var loadedKey: String?
    private var loading: Task<LocalSession, Error>?   // see below — required
    private var loadingKey: String?
    private var loadError: String?

    /// The session as-is, no loading. A caller that only wants to *borrow*
    /// the engine must never trigger a multi-gigabyte load as a side effect.
    func current() -> LocalSession? { session }
}
```

### An actor is not enough — this bug ships two engines

**Actors serialise only the stretches between suspension points.** The load
looks safe inside an actor and is not:

```swift
// WRONG
let fresh = try LocalSession(...)
await fresh.warmUp()      // <- suspends; the actor is re-entrant here
session = fresh           // <- a second caller already passed the nil check
```

A second caller arriving during `warmUp()` finds `session` still nil — it is
assigned only *after* warm-up returns — and builds a whole second engine.
Two models resident on a machine sized for one; the app becomes the first
thing the OS reclaims under memory pressure, and exits cleanly with no crash
report.

The log said so plainly and was read as noise: two "engine load: start"
lines, a second "loading the model" *while* the first was warming, and two
"ready" lines 160 ms apart.

Fix: publish the **in-flight task** and let later callers await it.

```swift
if let loading, loadingKey == key { return try await loading.value }

let task = Task<LocalSession, Error> {
    onPhase("loading the model")
    let fresh = try LocalSession(modelName: name, graphPath: graph,
                                 ggufPath: gguf, maxLength: maxLength)
    await fresh.warmUp(onPhase: onPhase)
    return fresh
}
loading = task; loadingKey = key
```

### Switching models: drop the old one first

Holding two models needs twice the resident set. Set `session = nil` and
cancel any in-flight load *before* starting the new one.

### Remember the load error

Cache the failure. Without it a UI that asks per frame retries a load that
will fail identically every time, and shows nothing useful.

## Report the phases

Loading and warming up take comparable time and look identical from outside —
a single "loading…" spinner appears hung. Pass an `onPhase` callback and say
which stage is running.

Warming at load also removes the multi-second first question, which is the
larger half of the keep-warm trade (see `tempo9-agent-loop`).

## Pair the Metal kernels with the engine build

**The failure this prevents is silent garbage, not an error.**

The Metal kernels are compiled from source at runtime. A binary linked
against one engine build while compiling another checkout's kernels produces
a chimera — in one measured case the output was literally `!!!!`.

Set `AS_METAL_KERNEL_DIR` explicitly, in priority order, and log loudly if
neither exists rather than letting the engine guess:

```swift
// 1) kernels shipped inside the app bundle
// 2) the kernels staged next to the engine archive you linked
// 3) neither -> warn; output cannot be trusted
if let kdir { setenv("AS_METAL_KERNEL_DIR", kdir.path, 0) }
else { log("no engine-matched Metal kernel directory; output may be unreliable") }
```

`AS_GEMM_BACKEND` selects `metal` or `cpu`. CPU is markedly slower; it is a
setting for when the GPU is contended, not a default.

## Check the files before touching the engine

Validate that the model and graph files exist **before** constructing a
session. A load that fails and is then called again can take the process
down, and no amount of well-written fallback logic survives that.

```swift
guard FileManager.default.fileExists(atPath: graphPath) else {
    log("skipping primary engine: graph missing -> fallback")
    session = nil; return
}
```

## Degrade honestly

Ship a fallback path, switch to it on a written-down rule, and say so in the
UI:

> Rule: run the primary engine first; on load failure **or two consecutive
> generation failures**, switch to the fallback and announce it. The fallback
> cold-starts per request and is obviously slower — that is what a failure
> mode should look like, and it should not be mistaken for normal.

```swift
if let session, failures < 2 {
    do    { ...; failures = 0; return result }
    catch is CancellationError { throw CancellationError() }   // not a failure
    catch { failures += 1; guard fallback != nil else { throw error } }
}
return try await fallback.run(...)
```

Cancellation is not an engine failure. Re-throw it before the failure counter.

## Read the counters

`session.stats()` returns engine counters — including prefix-cache hit
tokens. There is no per-request field, so when the engine is used serially,
take the **difference** around the call:

```swift
let before = session.stats()?.prefixCacheHitTokens ?? 0
let reply  = try await session.stream(...)
let after  = session.stats()?.prefixCacheHitTokens ?? before
record(input: reply.promptTokens, output: reply.completionTokens,
       cacheHit: max(0, after - before))
```

Report only what you can count. Mixing an estimated figure into a measured
total makes the whole total unciteable — under-report instead.

## Do not call `exit()` while a worker thread is still finishing

An in-process host exposes lifetime problems that a server hides, and this one
is worth knowing before you meet it:

> `exit()` right after the last reply ran static destructors while a worker
> thread was still erasing that request — the registry's mutex failed with
> "lock failed: Invalid argument".

The fix was to leak the singleton rather than destroy it at exit. The general
rule for a host: **any function-local static that a worker thread touches is
suspect at process exit.** Drain or detach before exiting, or accept the leak
— a leak at exit costs nothing, and a destructor racing a live worker costs a
crash on the way out, which users read as "it crashed".

## One table, counted

Wrap the shared session in something that also counts what is in flight —
concurrency is the feature, so the peak is a number you want:

```swift
public final class AgentTable: @unchecked Sendable {
    /// nil for an offline table: prompts can be built and checked with no
    /// engine loaded; the first real request throws.
    public let session: LocalSession?
    public private(set) var inflight = 0
    public private(set) var peakInflight = 0
}
```

The `nil` case earns its keep: it lets prompt assembly, schema construction
and isolation checks run in a plain unit test with no model in memory.

## Never run the in-process engine and a separate server together

Two engines on one machine evict each other's pages continuously and both
crawl. If the app embeds the engine, it must not also spawn a server; if it
exposes an API, it should serve it *from the same session it already holds*.

## Checklist

- [ ] one shared session, created on first use
- [ ] in-flight load published as a Task, not guarded by the actor alone
- [ ] old session dropped before loading a different model
- [ ] load errors cached, not retried per frame
- [ ] phase callbacks wired to the UI
- [ ] the executable declares macOS 26, and the log says `Metal backend READY`
- [ ] `AS_METAL_KERNEL_DIR` set from an engine-matched directory, or a loud warning
- [ ] model/graph files checked before constructing a session
- [ ] fallback rule written down; cancellation excluded from the failure count
- [ ] cache-hit counted as a delta around the call
- [ ] no second engine anywhere on the machine
