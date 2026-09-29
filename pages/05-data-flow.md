<Outline :current="4" />

---
clicks: 4
---

# EventEmitter

<DataFlow api="ee" />

---
clicks: 4
---

# perf_hooks

<DataFlow api="perf_hooks" />

---
clicks: 4
---

# trace_events

<DataFlow api="trace_events" />

---
clicks: 4
---

# async_hooks

<DataFlow api="async_hooks" />

---
clicks: 4
---

# diagnostics_channel

<DataFlow api="dc" />

<div v-click="4" class="caption">It does nothing until someone cares.</div>

---
clicks: 2
---

# The reveal

<v-switch>
<template #0>

<<< @/snippets/prototype-swap.mjs#slide js

</template>
<template #1>

<<< @/snippets/prototype-swap.mjs#slide js

</template>
<template #2>

```js {2}
class Channel {
  publish() {}
}

class ActiveChannel {
  publish(data) {
    for (const fn of this._subscribers) fn(data)
  }
}
```

</template>
</v-switch>

<div v-click="[1, 2]" class="mono reveal-output">Channel<br>ActiveChannel</div>

<div v-click="2" class="src reveal-src">https://github.com/nodejs/node/blob/main/lib/diagnostics_channel.js</div>

<div class="caption">When nobody has subscribed to a channel, calling publish() on it runs a function with nothing inside.</div>

<style>
.reveal-output { position: absolute; top: 72px; right: 64px; font-size: 24px; line-height: 1.4; color: var(--red); text-align: right; }
.reveal-src { top: 72px; bottom: auto; }
:deep(.slidev-code) { --slidev-code-font-size: 18px; }
</style>
