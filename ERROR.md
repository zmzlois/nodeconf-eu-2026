# Errors, corrections and decisions

Reference log for the talk. Every claim that turned out wrong, every method flaw, and every decision made to fix it lives here. Read this before editing `plans/` or writing slides. Append new entries; do not delete old ones, mark them resolved instead.

Status: `open` (found, not yet applied to plans) · `resolved` (plans updated) · `decided` (a choice, not a bug)

---

## 2026-09-28 — first plan review

Resolved the same day. Plan files were renamed to match build order: `01-design-system.md`, `02-benchmarks.md`, `03-animation.md`, `04-deck.md`.

Verified on this machine (Node v26.3.0, macOS arm64) and against upstream source.

### 1. Factual errors

| # | Where | Plan said | What is true | Evidence | Status |
|---|---|---|---|---|---|
| F1 | slide 25 notes, research §A | "a buggy subscriber cannot break the publisher" | publisher code continues, but the `nextTick` rethrow is an uncaught exception: the **process crashes** (exit 1) | local run: `publisher continued` then `Error: x`, `exit=1` | resolved |
| F2 | slide 14 | fastify per-request channels are onRequest / onResponse / onError | one tracingChannel: `tracing:fastify.request.handler:{start,end,asyncStart,asyncEnd,error}` | https://github.com/fastify/fastify/blob/main/lib/handle-request.js (L22), https://github.com/fastify/fastify/blob/main/docs/Reference/Hooks.md | resolved |
| F3 | slide 13 caption | "every `fetch()` in Node has been publishing since 2021" | global `fetch` did not exist in Node in 2021 (experimental 17.5, 2022). the undici PR is 2021, fetch came later | research-notes §D | resolved |
| F4 | slide 16 | "dd-trace is built on channels internally" | true, but dd-trace still **patches** with require-in-the-middle, then publishes. channels only separate the patch from the plugin | https://github.com/DataDog/dd-trace-js/blob/master/packages/datadog-instrumentations/src/http/client.js | resolved |
| F5 | 03-animation.md perf_hooks lane, slide 28 table | mark is stored in a C++ buffer (`node_perf.cc`), crosses JS/C++ | `mark()` → `enqueue()` → pushed onto `markEntryBuffer`, a **JS array** | https://github.com/nodejs/node/blob/main/lib/internal/perf/observe.js (L93, L420) | resolved |
| F6 | 03-animation.md acceptance | "boundary crossings in the other four lanes" | EventEmitter never crosses; after F5 only async_hooks and trace_events cross | follows from F5 | resolved |
| F7 | slide 13 snippet | subscribe to `undici:request:trailers` and print status | the status code arrives on `undici:request:headers` (`response.statusCode`); `trailers` only has `{ request, trailers }` | local run: `headers GET /hi 200`, `trailers keys [ 'request', 'trailers' ]` | resolved |
| F8 | 03-animation.md dc lane, hop 2 | "prototype check" | no check happens; the inactive prototype's `publish() {}` is simply called | local run: `Channel publish() {}` | resolved |

Confirmed correct (keep): prototype swap — idle `Channel` has `publish() {}`, after `subscribe` the prototype is `ActiveChannel` (slide 36b snippet works on 26.3). `fetch()` publishes `undici:request:create` (tested offline against a local server).

### 2. Benchmark method

| # | Issue | Fix | Status |
|---|---|---|---|
| B1 | tinybench times each iteration with `performance.now()`, which costs ~21 ns here, while an idle `publish()` takes about 1 ns. all idle cases would hit the same ~40M ops/s ceiling: you'd be measuring the timer | use Node core's pattern (`benchmark/diagnostics_channel/publish.js`): `hrtime.bigint()` around a 10M loop, write into a sink so V8 can't delete the loop as dead code, divide by N. drop tinybench | resolved |
| B2 | `watched` chart puts `ah-*` (await workload) and `trace-*` (perf.mark workload) on one axis | plot % slowdown vs each API's own baseline | resolved |
| B3 | numbers baked on 26.3; Node 26 becomes LTS 2026-10-27, so the room will be on 26.9+ | upgrade before baking; also makes `sqlite.db.query` / `crypto.fips.indicator` demoable | resolved |
| B4 | `node:bench` (26.9+) mentioned without a source | drop the mention | resolved |

### 3. Internal inconsistencies

| # | Issue | Fix | Status |
|---|---|---|---|
| I1 | overview says demos are "slides 19–22" (they are 23–24); 02-benchmarks.md says "slide 24" (it is 28) | fix refs | resolved |
| I2 | slide count stated as 42, then "38–40"; I count 43 with a/b slides. file numbers don't match workstream order (04-design-system is workstream 1) | recount once; renumber files in build order | resolved |
| I3 | slide 19 uses `tc.tracePromise` before slide 20 introduces `tracingChannel` | swap 19 and 20 | resolved |
| I4 | section 2 is "under the hood" but covers adoption; internals are in section 5 (36b) | rename section 2 (e.g. "Who already moved") or accept 36b as the payoff | decided: keep Lois's title; 36b is the payoff for "under the hood" |
| I5 | slide 25 sells GC as a feature; research §G and Sentry PR #24632 show it as a sharp edge (spans silently stopped) | present once, as a trade-off | resolved |
| I6 | slide 15 "nobody had to patch `http`" vs slide 18 "instrumentation-http still patches" | "nobody *has* to patch `http` any more" | resolved |
| I7 | permissions demo (24) uses `bindStore` while the message already carries `user`, so ALS adds nothing | `request:start` runs `runStores({ user }, handler)` once at request entry; `auth:check` subscribers read `als.getStore()`. add default-deny (no or throwing subscriber leaves `allowed` undefined) and keep a module-level reference to the channel (GC edge) | resolved |
| I8 | benchmarks (4) come before data flow (5): proof before mechanism | consider mechanism first: the animation predicts "no-op when idle", the bars confirm it | decided: keep Lois's order for now; swapping is still an option (move pages/05 before pages/04 and swap outline items 4 and 5) |

### 4. Copy vs the three-line rule

At 28px on a 980px canvas a line holds ~55 characters, so long bullets wrap to two lines. Slides 7, 8 (5 bullets), 14, 18, 25, 38, 39 fail the plan's own acceptance check. Status: resolved (all rewritten to three lines of ~55 characters or fewer; slide 27 tightened too; 19/20 layout is now magic-move).

Suggested rewrites:
- 25: "A bus, not a broker." / "Synchronous. In-process. Free when idle." / "No persistence. No backpressure. No wildcards."
- 18: drop "Half of the ecosystem has crossed." (no source)
- 8: keep three issues; the Lambda crash (dd-trace-js #3479) is not "silent"
- 4: "Line one of every *instrumented* Node app." ("every Node app in production" overclaims)
- 12: "Free" → "A no-op when nobody listens."
- 17: attribute to Sigrid Huemer; keep the source's inner quotes around 'from the outside'
- 19 layout: two-column code at 24px gives ~28 characters per column; stack the blocks or use magic-move

### 5. Missing

| # | Item | Status |
|---|---|---|
| M1 | "import first" does not fully go away: you still subscribe before the event fires (`fastify.initialization` fires once), but no longer before `require`. add a line to slide 38 or 39 | resolved |
| M2 | slide 13 needs venue network access; use a local `http.createServer` + `fetch('http://localhost')` (tested offline, works) | resolved |
| M3 | QR target is known; generate once (`npx qrcode -o public/qr.svg https://normal-people.com`) and drop the file-exists logic in `Qr.vue` | resolved |
| M4 | add a link-check step to verification: `curl` every URL in `research-notes.md` | resolved |
| M5 | slide 39 gaps: no wildcard or prefix subscribe (ties back to RabbitMQ topic exchanges); channels aren't shared across worker threads | resolved |

---

## 2026-09-28 — design prototype (6 slides)

Built cover, outline, statement (slide 5), code (slide 12), three lines (slide 25) and DataFlow `dc` (slide 36) to test the design system. Checked by exporting every click state to PNG.

| # | Found | Fix or decision | Status |
|---|---|---|---|
| D1 | the spec's vertical boundary at x=660 sits between columns 3 and 4, so the `dc` lane's last arrow would look like it crosses into C++. the picture would contradict the talk | horizontal boundary, JS row above, C++ row below; row change = crossing, drawn red. checked with the async_hooks lane | decided |
| D2 | `api="all"` stacking five lanes (900x1100 per spec) shrinks text to ~5px on a 16:9 slide | not built. options: a compact lane without notes, or drop slide 37 and let 33–36 carry it | open |
| D3 | code slide at 24px / line-height 1.5 overflowed: 11 lines under an `h1`, caption on top of code | line-height 1.4, 8 lines max, show a `#region` of the runnable file | resolved |
| D4 | `class="outline"` drew a black box: UnoCSS attributify treats it as the `outline` utility | renamed to `.section-list`; rule added to 01-design-system.md | resolved |
| D5 | `h1` max-width 18ch wrapped "What you actually built" onto two lines | 24ch | resolved |
| D6 | `pnpm export` needs a Chromium download | use installed Chrome via `--executable-path` | resolved |

---

## 2026-09-28 — full deck build (workflow, 5 builders + 5 reviewers)

| # | Found | Fix or decision | Status |
|---|---|---|---|
| F9 | research §E quoted esbuild #3348 as "require-in-the-middle isn't going to work in a bundler"; that sentence is not in the issue | verbatim: "breaks if you bundle your code ahead of time" (slide 8, research-notes) | resolved |
| F10 | research §F: userland reaches trace_events via `node.perf.usertiming`. on v26.3.0 marks and measures write only metadata; usertiming.js has no trace call | trace cases, slide 28 notes, DataFlow lane and 34b notes use `console.count` + `node.console`. recheck on 26.9+ | resolved |
| F11 | plan acceptance expected `dc-idle-guarded` among the fastest idle cases. measured 7.4 ns vs 2.1 ns unguarded (cheap message), level with `ee-idle` | slide 29 highlights `dc-idle`; notes: the guard pays off only when the message is expensive | resolved |
| F12 | research §D: instrumentation-undici subscribes to create/trailers only | it subscribes to five channels (create, sendHeaders, headers, trailers, error), per undici.ts | resolved |
| F13 | plan slide 4 used `NODE_OPTIONS="--import @sentry/node/preload"`, not on the cited Sentry page | `node --import ./instrument.mjs app.mjs`, as the page shows | resolved |
| F14 | plan slide 16 said the dd-trace excerpt shows `.publish`; the real file uses `startChannel.runStores(ctx, ...)` | kept verbatim, runStores highlighted, notes explain it publishes | resolved |
| B5 | perf-idle / perf-one differ 18% / 24% between two runs (GC with 1M retained marks); every other case within 3% | reported, not hidden. perf and trace cases run N = 1M, not 10M (10M marks ≈ 1.4 GB) | decided |
| B6 | `traceSync` costs ~360 ns, ~50x a plain publish, on 26.3 (`using` + DisposableStack) | slide 30 calls it the outlier; recheck on 26.9+ rebake | open |
| D7 | magic-move marks every token `.slidev-code-highlighted`, so the red rule repeated per token | global exclusion in style.css; local copies removed | resolved |
| D8 | Slidev's `p` line-height (24px) beat the deck's 1.3, cramping wrapped 28px text | `.slidev-layout p { line-height: 1.3 }` | resolved |
| M6 | repo link on slides 27 and 40 is a placeholder | speaker supplies the public repo URL | open |
| M7 | slide 8 notes keep a slot for Lois's own war story | speaker | open |

---

## 2026-09-28 — cursed section (plan only, see `plans/05-cursed-app.md`)

Lois asked for a new section 4, "cursed things you can build with it": a real application (chat, database, feature flags, permissions) on channels. Benchmarks move to 5, data flow to 6. Slide numbers in the entries above refer to the old numbering; the map old → new is in `plans/05-cursed-app.md`.

Checked on Node v26.3.0 against upstream main, plus a throwaway prototype in the session scratchpad (eight end-to-end checks pass, exit 0).

| # | Found | Fix or decision | Status |
|---|---|---|---|
| F15 | slide 25 notes, `demos/permissions.mjs`, research §A and §G say "an unreferenced channel object can be garbage collected, keep a module-level reference". overstated: `subscribe` and `bindStore` call `channels.incRef`, so a channel with a subscriber or a bound store is strongly held. DEP0163 was revoked for this reason (v24.8.0 / v22.20.0, "channel objects are now resistant to garbage collection when the channel has active subscribers"). only an unsubscribed, unreferenced channel is collected, and that loses nothing. verified with `--expose-gc` | reword to "with a subscriber it is strongly held (v22.20 / v24.8+)"; the chat demo creates `room:<name>` channels on the fly | open |
| F16 | plan assumed `channel.subscribe(fn)` is deprecated (DEP0163) | revoked; both forms are fine, no warning on 26.3 even with `--throw-deprecation`. use `dc.subscribe(name, fn)` on slides for readability only | resolved |
| F17 | `tracePromise`: the `end` message does not carry `result`; `asyncStart` / `asyncEnd` do. `traceSync` `end` carries `result`. not on a slide today, but the tracing snippet's notes should not claim otherwise | note in research §A | open |
| D9 | six sections instead of five; section 3 shrinks to divider + smell + one RabbitMQ exchange comparison; feature-flags and permissions demos are replaced by layers of `demos/chat/` | decided, pending Lois's confirmation of the 6-section shape (alternative: drop section 6 or merge 3 into 4) | decided |
| D10 | timing: 30 → 32 min before cuts. cut order: 41b, 38b, then fold fan-out (new 28) into rooms (new 27). never cut 31 (cluster), it sets up the close | decided |
| M8 | slide 31 "add a second process" claims cluster breaks fan-out. per-thread channels are verified (subscribe on main, publish in a Worker: nothing), the cluster run itself is not | verify with `demos/chat/cluster.mjs` before the slide exists | open |
| M9 | a subscriber that writes to an ended `res` throws `ERR_STREAM_WRITE_AFTER_END` asynchronously and kills the process, same as a throwing subscriber. `res.destroyed` alone is not enough | guard is `res.destroyed || res.writableEnded` | resolved |
| M10 | `Date.now()` as the history key collides within one millisecond and overwrites | monotonic `nextTs()` | resolved |
| M11 | prior art: no published chat/db/flags on channels found (2026-09-28). closest: evlog `evlog.event`, Fastify `fastify.initialization`. safe to claim "nobody has done this" with "that I could find" | notes on new slide 25 | resolved |

---

## 2026-09-28 — research: origin, vendor rewrite paths, goals

| # | Found | Fix | Status |
|---|---|---|---|
| F21 | research §D gave the Sentry blog title as "There Are Better Ways Than Monkey-Patching…" | real title: "No more monkey-patching: Better observability with tracing channels" | resolved |
| F22 | research §D cited sergical/js-tracing-channels-proposals | stale fork; cite getsentry/js-tracing-channels-proposals | resolved |
| F23 | notes and slides call TracingChannel experimental | Stable since v26.8.0 (#64525); store APIs, boundedChannel and built-in channel groups stay Experimental. Section 2 slide 16 says so | resolved |
| F24 | F4 said dd-trace patches with the npm require-in-the-middle | it uses its own fork (`packages/dd-trace/src/ritm.js`) + import-in-the-middle, and is cutting over to orchestrion | resolved in notes |
| F25 | "orchestrion-js is Rust/SWC" | rewritten in JS (2026-03), lives at nodejs/orchestrion-js | resolved in notes |
| F26 | assumed Qard opened diagnostics#134 and that import order / semver breakage were listed harms | Mike Kaufman (Microsoft) opened it; the four harms are brittleness, duplicated work, agents trampling each other, ESM loaders | resolved in notes |
| B4b | B4 dropped `node:bench` as unsourced | it exists: #65606, v26.9.0, behind `--experimental-bench` | resolved in notes |
| B6b | slow `traceSync` (B6) | fix #64251 merged 2026-09-19, not in v26.10.0; recheck on next release | open |
| R1 | Sentry tracker claims undici "ships TracingChannel natively" | wrong: `lib/core/diagnostics.js` uses plain `channel()` only; don't repeat it | noted |
| R2 | Sentry comment: "Node caps ... channels ... at 1024" | only an initial C++ buffer size; don't put on a slide | noted |
| D11 | section 2 expanded from 11 to 17 slides (origin, stable, who publishes, orchestrion, vendor table, goal); the OpenTelemetry slide folded into the vendor table. Every later slide number shifts by +6, on top of the cursed-section map in `plans/05-cursed-app.md` | decided (Lois, 2026-09-29: build all, trim after a timed rehearsal) |

---

## 2026-09-29 — data-flow research (slides 22–26, rendered numbering)

Full write-up with permalinks: `plans/research-dataflow-cpp.md`. Node source pinned at `d9274ce6238d6d0ab3b7870231e2d9ac1ab399ca`; runtime checks on v26.3.0.

| # | Found | Fix | Status |
|---|---|---|---|
| F27 | slide 22 notes: Orchestrion loads "through module.registerHooks". true on current Node only. Sentry v11 uses `registerHooks` when stable (24.13 / 25.1+), else `module.register` plus a `Module.prototype._compile` patch; Datadog uses `registerHooks` except on buggy versions; the reference loader `apm-js-collab/tracing-hooks` patches `Module.prototype._compile` for CJS | reword notes: "at build time, or at load time via module.registerHooks on current Node; older Node falls back to module.register plus a _compile patch" | open |
| F28 | slide 22 implies Orchestrion is the whole tool. `nodejs/orchestrion-js` is the transformer only (`@apm-js-collab/code-transformer`); it inlines `start.runStores` / `end.publish` / `asyncStart` / `asyncEnd` rather than calling `tracePromise`, names channels `orchestrion:<module>:<name>`, and has a `hasSubscribers` fast path | add to slide 22 notes | open |
| F29 | extends F17: `tracePromise` publishes `end` when `fn` returns its promise, before the awaited work finishes; `asyncStart` and `asyncEnd` fire back to back in the reaction (source TODO at lib/diagnostics_channel.js L614). spans must end on `asyncEnd` (Sentry does) | slide 23/24 notes | open |
| R2b | R2 "1024 channels cap": `kInitialChannelCapacity = 1024` is an initial size for C++-linked channels only, doubling since v26.7.0 (#64497). on 26.3 the table is a fixed 1024, still only for C++-linked channels | never on a slide | resolved |
| M12 | Node's C++ publishes on channels since v25.8.0 (#61869): `sqlite.db.query`, `crypto.fips.indicator`, and `node:permission-model:*` (documented in permissions.md, not in the built-in list). C++ checks a shared `Uint32Array` of subscriber counts that JS writes on subscribe. the permission channel fires on 26.3 (`node --permission`, denied read of /etc/hosts) | candidate beat "Even C++ publishes" for the Node-itself slide or the reveal; Lois decides | open |
| M13 | `http.server.request.start` is a plain `publish`, not `runStores` (lib/_http_server.js L1384); a vendor cannot `bindStore` request context and must `enterWith` in a subscriber | one line for "what I would still fix" | open |
| D12 | section 2 cut from 17 to 14 slides: Datadog code (21), Orchestrion (22) and the Sentry quote (24) removed. Orchestrion gets one line above the vendor table; the cut slides' quotes and sources moved into that slide's notes. `snippets/dd-trace.js` is now unused | decided (Lois, 2026-09-29) |

---

## 2026-09-29 — dcmq (plan: `plans/06-dcmq.md`, prototype: `plans/spikes/dcmq/`)

| # | Found | Fix or decision | Status |
|---|---|---|---|
| F30 | on macOS, `PRAGMA synchronous=FULL` is not power-loss safe: SQLite uses plain `fsync` unless `PRAGMA fullfsync=1`. measured ~17,900 vs ~197 commits/s. SIGKILL survival is unaffected (proofs 3, 4) | any slide saying "durable" means "survives a process crash"; add a `fullfsync` option (C6) | open |
| F31 | `PRAGMA data_version` changes only when another connection commits, never for the same connection's own writes (verified on node:sqlite, SQLite 3.53.2) | reset the topology cache by hand on local writes | resolved in prototype |
| F32 | a throwing subscriber on `tracing:dcmq:deliver:start` exits the process with code 1 even though dcmq catches handler errors | say it on the dcmq slides: dcmq protects handlers, not other people's subscribers | decided |
| F33 | the chat prototype crashed after being moved: `engine-wal.mjs` threw ENOENT inside a `db:set` subscriber because `data/` was missing, which killed the process (the F1 rule, in practice) | create `data/` at import; a real example for the fan-out slide notes | resolved |
| D14 | dcmq semantics follow RabbitMQ where cheap: drop-head default, delivery limit 20, `x-*` header filtering, unroutable is still confirmed and `mandatory` returns it on `dcmq:returned`. multi-queue publish stays all-or-nothing on reject (RabbitMQ partially delivers) | see C1–C8 in the plan | decided pending Lois |
| M14 | the chat plan pointed at a scratchpad path that does not survive the session | prototypes preserved in `plans/spikes/{chat,dcmq}`, runtime `data/` gitignored | resolved |
| D15 | Lois, 2026-09-29: "the chat app should run on dcmq". reverses the plan's recommendation to keep chat on raw channels. rooms cross processes through dcmq (one queue per node, bound to `room.#`); inside a node a relay republishes on raw `room:<name>` channels; flags and auth stay on raw channels. dcmq must be built before the chat app | decided |
| D13 | removed the close's "Publish, don't patch." and "What I would still fix" slides (deck 43, 44); the end slide is the whole close; their points moved into the end slide's notes. Note: `plans/05-cursed-app.md` calls the cursed section "the setup for that close", which no longer has a slide | decided (Lois, 2026-09-29) |
| R3 | my entries above were first numbered D12 and D13, colliding with Lois's D12 (section 2 cut) and D13 (close cut); renumbered to D14 and D15 on 2026-09-29 | resolved |

---

## 2026-09-29 — chat on dcmq (prototype: `plans/spikes/chat-on-dcmq/`, 27 checks pass)

| # | Found | Fix or decision | Status |
|---|---|---|---|
| F34 | a node down for more than its queue's `maxLength` (1000, drop-head) silently loses the oldest events: 1200 published while node 2 was down, 200 gone, publisher still got `true` | say it on the relay slide; the build raises the node queue to 10,000 and dead-letters dropped events (C5) | open |
| F35 | a node killed while holding a message breaks order on restart: the leased message comes back after `leaseMs`, behind newer ones (`2, 3, 1`). with the default 30 s lease it is 30 s late | relay uses `leaseMs: 2000` (its handler is synchronous and fast); document the reorder | open |
| F36 | two processes opening a fresh dcmq file at once: one died with `database is locked` on `PRAGMA journal_mode=WAL`; the busy timeout does not cover that pragma | dcmq change C9: retry the WAL pragma; the prototype's `cluster.mjs` starts workers one at a time | open |
| F37 | a restarted node reusing `seq = 1` would repeat ids and the idempotent relay would silently drop its new messages | ids are `${NODE_INDEX}.${max(Date.now(), last + 1)}` | resolved in prototype |
| F38 | history is arrival order (one global order, because a publish writes every node queue in one transaction), not `ts` order: `ts` went backwards 4 and 17 times in 300 concurrent posts | never sort history by `ts` | decided |
| F39 | a SIGKILL during a WAL append leaves a torn last line and replay throws at startup (original chat behaviour) | replay skips an unparseable last line | open |
| D16 | section 4's thesis changes: with the close's gap slide gone (D13) and the chat app on dcmq (D15), section 4 ends working. "channels inside a process, a broker between them" | `05-cursed-app.md` rewritten | decided |
| D14 | slides "Saying something, nobody listening" and "…one consumer" showed different second rows (guard vs traceSync) and ops/s, so channel.publish() looked like it dropped 475M → 144M when it goes 2 → 7 ns | merged into one "Cost per call" slide (`components/CostBars.vue`): ns per call, idle and one-listener bars per API, guard and traceSync in the notes. No re-bench; numbers unchanged. Deck is now 42 slides | resolved (Lois, 2026-09-29) |
| D15 | sections 1 and 2 cut to the slides with screenshots or real code. Section 1: divider, Line one, 3 ways it silently stops. Section 2: divider, 2020 (+ "Stable in Node 26.8, six years later."), Fastify, Where the vendors are. Cut slides' notes are folded into the notes of the section's last kept slide. Unused now: snippets/ritm.mjs, api.mjs, tracing.mjs, undici.mjs, before.mjs, after.mjs, dd-trace.js | decided (Lois, 2026-09-29) |

---

## 2026-09-29 — public chat on Vercel

| # | Found | Fix or decision | Status |
|---|---|---|---|
| D17 | Lois asked for the chat app on Vercel with a QR in section 3, per-session identities, and in-memory node:sqlite | deployed `plans/spikes/chat-on-dcmq/` as project `dcmq-chat` (team zmzlois-projects) to production, https://dcmq-chat.vercel.app, region dub1. production, not preview: previews sit behind Vercel login | decided |
| D18 | in-memory storage cannot be shared across processes | `DCMQ_FILE` defaults to `:memory:` (Vercel, single process); the local two-node demo sets a file, since two processes cannot share memory | decided |
| F40 | Vercel offers Node 24.x at most, not 26 | `engines.node: 24.x`; chat and dcmq tests pass on Node 24.4.0 and 26.3.0 | resolved |
| F41 | Vercel scales out automatically and in-memory state is per instance, so a busy room can split | the page shows the instance id; the "Your turn" notes say a second id means the room split, the "one thread" gap live | open, accepted |
| D19 | a public room on the big screen needs moderation, and the deck is itself deployed, so no token in deck source | signed identities (HMAC, `SESSION_SECRET`), admin only via `ADMIN_TOKEN`; moderator mode on the phone page via `#admin=<token>` (a fragment never reaches the server); token in `plans/spikes/chat-on-dcmq/.vercel/admin-token.txt`, not in git | decided |
| D20 | Lois: each gap on the overview slide deserves its own slide | eight gap slides after "You absolutely can build ~~RabbitMQ~~ dcmq", each: what breaks, the decision, the proof in red; sources in notes | decided |
| D16 | added "What is RabbitMQ?" and "What a broker must do" after "You absolutely can build dcmq" (deck 15, 16), before the eight gap slides. Facts from rabbitmq.com (amqp-concepts, protocols): broker definition, exchange → binding → queue flow, acks, AMQP 0-9-1 original core, AMQP 1.0 core since 4.0. The eight "must do" lines follow the gap slides' order. First inserted one slide too early (slide count off by one), moved within the same turn | decided (Lois, 2026-09-29) |
| D21 | Lois, 2026-09-29: guarantee one instance and one room: "lets do cloudflared". the public room runs as one process on the speaker's laptop (in-memory sqlite) behind a named Cloudflare tunnel; `CHAT_ROOM=nodeconf` locks the server to one room | decided |
| F42 | Cloudflare Quick Tunnels (trycloudflare.com) "do not support Server-Sent Events (SSE)" and cap at 200 in-flight requests (https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/): the chat's stream would break and a full room would exceed it | named tunnel on normal-people.com (already on Cloudflare DNS); SSE sent with `no-cache, no-transform` so the proxy does not compress or buffer it | open until the tunnel is verified |
| F42b | resolves F42: named tunnel `dcmq-chat` routes chat.normal-people.com to localhost:4300. five probes through Cloudflare arrived in 63–86 ms (two tunnel hops, from the laptop itself), no buffering; other rooms 404 | named tunnel verified 2026-09-29 | resolved |
| F43 | the launcher's public check failed although the tunnel worked: a DNS lookup made before the record existed left a negative cache entry on the laptop | launcher reports "tunnel connected" from cloudflared's own log; the public check is advisory. test from a phone on mobile data | resolved |
| D22 | dcmq-chat.vercel.app now redirects (307, path kept) to https://chat.normal-people.com, so the old URL cannot open a second room; the "Your turn" QR encodes chat.normal-people.com and its stage pane talks to localhost:4300 | decided |
| D17 | slide 16 "What is RabbitMQ?" rewritten as a table: jobs people give RabbitMQ/BullMQ, each mapped to the gap slide it hits (exact gap titles); "What a broker must do" folded into it and removed; the MQTT/STOMP caption dropped. Sources: rabbitmq.com (amqp-concepts, protocols, tutorial two), docs.bullmq.io | decided (Lois, 2026-09-29) |
| F27 | "No wildcards." slide: `*.orange.*` rendered as italic ".orange." (markdown emphasis) | patterns wrapped in mono spans with &#42; | resolved |
| F44 | phones on the speaker's Wi-Fi could not open chat.normal-people.com while the laptop browser could: the Wi-Fi resolver (128.65.200.80) cached NXDOMAIN from a lookup made before the record existed (negative TTL 1800 s from the zone's SOA); 1.1.1.1, 8.8.8.8 and 9.9.9.9 resolved it and Cloudflare served the page to an iPhone user agent. cloudflared's "stream canceled by remote" lines are browsers closing their event streams, not errors | expires on its own; never look up a hostname before creating its record. at the venue the resolver has never seen the name | resolved |
| D23 | Lois: "we can skip the admin token" | the launcher no longer reads or passes a token; a moderator exists only if ADMIN_TOKEN is set in the environment | decided |
| D18 | slide 16 table now names real users, each checked against a primary or vendor source: Vendure (BullMQ, Vendure docs + BullMQ "Used by"), Hemnet, FarmBot, Softonic (CloudAMQP use-case page), NRK (VMware Tanzu case study). Dropped for lack of a readable source: Microsoft lage and Datawrapper (listed by BullMQ, use not confirmed), Instagram (post returned 403). The gap column is our mapping, not the companies' words | decided (Lois, 2026-09-29) |
| D24 | Lois: slides 17–24 (the eight gaps) "don't mean much" as three-line text; each should show source code, titled "<gap> - <how we solved it>" | each is now a `class: gap` code slide: title, one grey "Before:" sentence, a `#region` imported from the running code (plans/spikes/chat-on-dcmq), the key lines highlighted, the measured proof as caption. regions: dcmq.mjs#disk/#ack/#slide/#wires, store.mjs#slide (claim, used twice: lease and work queue), router.mjs#slide, chat/bus.mjs#replay. long source lines reflowed to fit 17px code (behaviour unchanged, both test suites pass) | decided |
| D19 | slide 16: company column dropped (Lois: famous names or none). Discord and Riot Games: no source says they use RabbitMQ or BullMQ. Twitter built its own queue (Kestrel). Verified famous users named in the caption instead: Mozilla (Pulse, a managed RabbitMQ cluster, topic exchanges) and the original reddit.com (open-source code declares vote_link_q, vote_comment_q, …). Table is jobs → gaps again | decided (Lois, 2026-09-29) |
| D25 | Lois: "we should handle guarantee delivery" and put it on slide 17 | dcmq never deletes an unacked message: out of tries, drop-head and unroutable now park with a reason; `retryDead` / `purgeDead` / `dead`; lease heartbeat stops slow handlers being delivered twice (proof 13 now asserts delivered once). new slide 17 "No guarantee - only an ack deletes" shows `settle()`; "No disk" moved to 18. dcmq and chat tests pass on Node 26 and 24 | decided |
| F45 | my first insert of slide 17 and the ack highlight fix were overwritten 30 s later by a save of pages/03-rabbitmq.md from an editor holding an older copy | re-applied; reload the file in the editor before saving | resolved |
| F46 | the "No ack" caption said "twenty tries, then the dead-letter queue"; the prototype default is 5 tries and now parks | caption: "Out of tries, the message is parked, not lost." | resolved |
| D26 | Lois: remove every "Before: …" line on the gap slides, there is no "before" to handle | 9 lines removed from slides 17–25, `.gap .problem` style removed | decided |
| D27 | Lois: "No xxx" titles sound cringe; match the wording of slide 16 ("what people give it / what is required") | titles are now "<requirement from slide 16> - <how dcmq does it>", e.g. "Kept on disk - a row before publish returns"; all fit on one line | decided |
| D20 | removed the "Try it" slide (two-node live chat, deck 26). Only the slide; ChatDemo and the demo servers are untouched. "Your turn" (the public chat) stays. Backup of the page in the session scratchpad | decided (Lois, 2026-09-29) |
| D28 | Lois: slides 17–25 must say whether the code imports diagnostics_channel and which method it calls | a grey mono line above each excerpt: the file, whether it imports node:diagnostics_channel (red when it does), and the exact calls. verified against the source: dcmq.mjs and chat/relay.mjs import it; store.mjs (node:sqlite only), router.mjs and chat/bus.mjs do not. four of the nine fixes use no channel at all: lease, claim, routing, waiting queue | decided |
| D29 | Lois: move sections 4 and 5 to 3 and 4, and section 3 to 5, for flow | order is now problem, under the hood, benchmarks, data flow, rabbitmq/dcmq, close. changed slides.md import order, Outline.vue list, the three dividers and their "section n" notes. page files keep their names (03-rabbitmq.md is now section 5) so an editor holding a file open cannot resurrect an old name; rename later if wanted. the gap slides moved from 17–25 to 30–38 | decided |
| D30 | review of slides 29–41, Lois: "go ahead" | slide 40 rewritten as the payoff ("A bus inside a process. / A broker between them, in under 350 lines. / At-least-once. Idempotent, or wrong."); retried and one-worker slides merged into one with click highlights `{5,7|4,6,8,9}`; claim SQL reads `state='ready' OR (state='inflight' AND lease_until < ?)`; C1: publish returns true when stored (unroutable too), false on backpressure; channel lines in one format "file · diagnostics_channel or not / calls"; "dcmq:dead publish() parks" corrected to "announce"; "two slides on" replaced by a slide name; stale notes fixed (20 tries, maxlen loss, eight slides, repo placeholder); slide 29 rows in slide order, "wildcard". dcmq tests pass on Node 26 and 24, chat tests pass | decided |
| F15b | F15's overclaim (unreferenced channels get collected) was still in slide 40's notes | notes rewritten; F15 resolved for the deck | resolved |
| F28 | slide "Guaranteed delivery - only an ack deletes" / caption "never deleted" / code comment "the only way out": purgeDead() also deletes, and park() also takes a message out of the queue | title "Guaranteed delivery - no silent loss", caption "deleted only if you ask", comments reworded (behaviour unchanged, tests pass) | resolved |
| F29 | "Kept on disk" slide: createBroker() defaults to :memory: | note added: disk only with { file } | resolved |
| F30 | no mention that dcmq retries immediately, without backoff | note added | resolved |
| D31 | Lois: remove slide 39 ("What you actually built") | removed; the RabbitMQ section ends on "Your turn", then the end slide (now 39) | decided |
| D21 | removed the benchmark "Two questions" setup slide (deck 12); its notes (method, Node version caveat, repo link) moved into "Not the same animal" | decided (Lois, 2026-09-29) |
