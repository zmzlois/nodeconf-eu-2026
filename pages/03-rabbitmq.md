<Outline :current="5" />

---
layout: statement
---

# Topics. Publishers.<br>Subscribers. Fan-out.<br><span class="red">I smelled RabbitMQ.</span>

---

# Feature flags

<<< @/demos/feature-flags.mjs#slide js {4}

<div class="caption">Subscribers run in the same tick. Writing back is a reply.</div>

---

# Permissions with context

<<< @/demos/permissions.mjs#slide js {7}

<div class="caption">The caller never passes the user. It rides in the store.</div>

---

# You absolutely can build <s>RabbitMQ</s> <span class="red">dcmq</span> out of <span class="mono">diagnostics_channel</span>

Built on <span class="mono">node:diagnostics_channel</span> and <span class="mono">node:sqlite</span>.

<span class="red">255 lines. </span>

---

# What is RabbitMQ?

<div class="flow mono">producer → exchange → queue → consumer</div>

<table class="jobs">
<tr><th>what people give it</th><th class="red">what's required</th></tr>
<tr><td>a payment webhook must reach its handler</td><td>Guaranteed delivery, at least once.</td></tr>
<tr><td>count every vote in the background</td><td>Kept on disk and acked when done.</td></tr>
<tr><td>resize uploads across three workers</td><td>One worker per job, retried if it dies.</td></tr>
<tr><td>a slow API that must not be flooded</td><td>A cap on work in flight.</td></tr>
<tr><td>route CI events by topic</td><td>Routing by pattern and wildcard matching.</td></tr>
<tr><td>a restarted worker catches up</td><td>Messages wait until it is back.</td></tr>
<tr><td>workers on many machines</td><td>Works across processes.</td></tr>
</table>

<style>
.flow { font-size: 24px; margin: 0 0 12px; }
.jobs { font-size: 20px; line-height: 1.25; margin-top: 12px; }
.jobs th, .jobs td { padding: 4px 24px 4px 0 !important; border-color: #000; text-align: left; }
.jobs td:last-child { font-weight: 700; white-space: nowrap; }

</style>

---
class: gap
---

# Guaranteed delivery - no silent loss

<div class="dc mono">dcmq.mjs · imports <span class="red">node:diagnostics_channel</span><br>announce: channel('dcmq:dead').publish() · wake: channel('dcmq:queue:q').publish()</div>

<<< @/plans/spikes/chat-on-dcmq/dcmq/dcmq.mjs#guarantee js {2,4}

<div class="caption">Out of tries, overflowed or unroutable: parked with a reason, deleted only if you ask.</div>

---
class: gap
---

# Kept on disk - a row before publish returns

<div class="dc mono">dcmq.mjs · imports <span class="red">node:diagnostics_channel</span> · the rows are node:sqlite<br>tracingChannel('dcmq:publish').traceSync() · wake: channel('dcmq:queue:q').publish()</div>

<<< @/plans/spikes/chat-on-dcmq/dcmq/dcmq.mjs#disk js {3-4}

<div class="caption">SIGKILL the publisher: 100 sent, 100 there.</div>

---
class: gap
---

# Acked when done - awaits the handler

<div class="dc mono">dcmq.mjs · imports <span class="red">node:diagnostics_channel</span><br>tracingChannel('dcmq:deliver').tracePromise() · channel('dcmq:error').publish()</div>

<<< @/plans/spikes/chat-on-dcmq/dcmq/dcmq.mjs#ack js {3,5,8}

<div class="caption">Resolve acks. Throw nacks. When out of tries the message is parked not lost.</div>

---
class: gap
---

# One worker retried if it dies 

<div class="dc mono">store.mjs · node:sqlite<br>DatabaseSync.prepare(sql).get() · a setInterval() renews the claim every 10 s</div>

<<< @/plans/spikes/chat-on-dcmq/dcmq/store.mjs#slide js {5,7|4,6,8,9}

<div class="caption">Killed mid-handler: back with deliveries = 2. Three processes, 300 messages, no duplicates.</div>

---
class: gap
---

# A cap on work in flight

<div class="dc mono">dcmq.mjs · woken by channel('dcmq:queue:q').subscribe(), see Works across processes</div>

<<< @/plans/spikes/chat-on-dcmq/dcmq/dcmq.mjs#slide js {1}

<div class="caption">Prefetch 2: never more than 2 running.</div>

---
class: gap
---

# Routing by pattern 

<div class="dc mono">router.mjs · channel names are exact, so routing happens before any channel</div>

<<< @/plans/spikes/chat-on-dcmq/dcmq/router.mjs#slide js {4-6,8}

<div class="caption">lazy.# and *.orange.* route like RabbitMQ.</div>

---
class: gap
---

# Messages wait - a queue per node

<div class="dc mono">bus.mjs · the queue waits in node:sqlite<br>relay.mjs · imports <span class="red">node:diagnostics_channel</span>, then channel('room:&lt;name&gt;').publish()</div>

<<< @/plans/spikes/chat-on-dcmq/chat/bus.mjs#replay js {1,4}

<div class="caption">Node 2 down, 3 posts, restart: all 3, in order.</div>

---
class: gap
---

# Works across processes - one SQLite file

<div class="dc mono">dcmq.mjs · imports <span class="red">node:diagnostics_channel</span><br>this process: channel('dcmq:queue:q').subscribe() · others: setInterval()</div>

<<< @/plans/spikes/chat-on-dcmq/dcmq/dcmq.mjs#wires js {6-7}

<div class="caption">Channel: 4 ms. Disk: 30 ms.</div>

---

# Your turn

<div class="your-turn">
  <Qr label="chat.normal-people.com" href="https://chat.normal-people.com" src="/chat-qr.svg" />
  <ChatDemo :nodes="[{ label: 'chat.normal-people.com', base: 'http://localhost:4300' }]" :controls="false" />
</div>

<style>
.your-turn { display: grid; grid-template-columns: 260px 1fr; gap: 48px; align-items: start; }
.your-turn .qr img { width: 260px !important; height: 260px !important; }
</style>

