<Outline :current="1" />

---

# Line one

<<< @/snippets/line-one.cjs#slide js {1}

<div class="caption">Fastify with Datadog. In ESM, line one moves into the command: <span class="mono">node &#45;&#45;import dd-trace/initialize.mjs</span></div>

---
clicks: 2
---

# <span class="red">3</span> ways it silently stops

<div class="graveyard">

esbuild #3348: "breaks if you bundle your code"

dd-trace-js #2160: express spans missing when bundled

next.js #88161: <span class="mono">serverExternalPackages</span> or no spans

</div>

<img class="limitation" src="/limitation.png" alt="opentelemetry instrumentation readme, limitations: modules are not included in a bundle" />

<div class="src">github.com/open-telemetry/opentelemetry-js · instrumentation README</div>

<img v-click="[1, 2]" class="evidence" src="/esbuild-problem.png" alt="esbuild issue 3348: breaks if you bundle your code ahead of time, closed as not planned" />

<style>
.graveyard p { color: var(--grey); }
.limitation { width: 720px; margin-top: 24px; }
.evidence { position: absolute; top: 40px; left: 50%; transform: translateX(-50%); height: 470px; border: 1px solid #000; }
</style>

