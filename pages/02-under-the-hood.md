<Outline :current="2" />

---
layout: fact
clicks: 1
---

<div>
<div class="huge">2020</div>

Stephen Belanger opens <span class="mono">nodejs/node</span> #34895.

<span class="red">Stable in Node 26.8, six years later.</span>

</div>

<img v-click="1" class="evidence" src="/create-diagnostics.png" alt="nodejs/node pull request 34895, lib: create diagnostics_channel module, opened by Qard on Aug 23, 2020" />

<style>
.evidence { position: absolute; top: 40px; left: 50%; transform: translateX(-50%); height: 470px; border: 1px solid #000; }
</style>

---
clicks: 1
---

# Fastify

<v-switch>
<template #0>

<<< @/snippets/fastify-publish.js#slide js {8}

<div class="caption">Fastify publishes. <span class="mono">lib/handle-request.js</span></div>
</template>
<template #1>

<<< @/snippets/sentry-fastify.ts#slide ts {2}

<div class="caption">Sentry subscribes. No patch, no import order.</div>
</template>
</v-switch>

<style>
:deep(.slidev-code) { --slidev-code-font-size: 20px; }
.caption { position: static; margin-top: -12px; }
</style>

---

# Where the vendors are

<div class="orchestrion"><span class="red">Orchestrion</span> rewrites library source to call TracingChannel.</div>

<table class="vendors">
<tr><th></th><th>today</th><th>next</th></tr>
<tr><td>Datadog</td><td>~120 patched, 17 rewritten</td><td>orchestrion by default</td></tr>
<tr><td>New Relic</td><td>~50 libraries rewritten</td><td>node core still patched</td></tr>
<tr><td>Sentry</td><td>channels by default, v11</td><td>more native libraries</td></tr>
<tr><td>OpenTelemetry</td><td>undici only</td><td>http on channels, opt-in PR</td></tr>
<tr><td>Elastic</td><td>undici only</td><td>orchestrion proof of concept</td></tr>
<tr><td>AppSignal, Instana</td><td>patches</td><td>nothing announced</td></tr>
<tr><td>Better Stack</td><td>eBPF from the kernel, no patches</td><td>nothing announced</td></tr>
</table>

<style>
.orchestrion { font-size: 24px; margin-bottom: 16px; }
.vendors { font-size: 18px; line-height: 1.2; }
.vendors th, .vendors td { padding: 4px 24px 4px 0 !important; border-color: #000; text-align: left; }
.vendors td:first-child { font-weight: 700; }
</style>

