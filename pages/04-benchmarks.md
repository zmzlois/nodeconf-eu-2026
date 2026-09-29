<Outline :current="3" />

---

# Not the same animal

| | you publish? | consumer gets | crosses C++? |
|---|---|---|---|
| diagnostics_channel | yes | sync callback | no |
| EventEmitter | yes, per instance | sync callback | no |
| perf_hooks | yes, <span class="mono">mark()</span> | deferred observer | no, JS array |
| trace_events | no userland API | a file | yes |
| async_hooks | no, it sees everything | callbacks per resource | yes |

<div class="caption">async_hooks watches you. trace_events records the runtime. diagnostics_channel lets a library speak.</div>

<style>
table { font-size: 20px; line-height: 1.3; }
th, td { padding: 6px 16px 6px 0; border-color: #000; }
td:first-child { font-weight: 700; }
</style>

---

# Cost per call

<CostBars highlight="dc">
  <template #caption>
    <div class="caption">A channel costs 2 ns idle, 7 ns with a listener. Cheapest either way.</div>
  </template>
</CostBars>

---

# Cost of watching everything

<CostBars set="watch">
  <template #caption>
    <div class="caption">Turn on a hook and every await pays, not just yours.</div>
  </template>
</CostBars>

<div class="als">Since Node 24, AsyncLocalStorage no longer uses async_hooks.</div>

<style>
.als { color: var(--red); font-size: 24px; margin-top: 8px; }
</style>

---

# Memory

<BenchBars group="memory" highlight="mem-dc-one">
  <template #caption>
    <div class="caption">perf_hooks keeps every mark. trace_events keeps a file.</div>
  </template>
</BenchBars>

