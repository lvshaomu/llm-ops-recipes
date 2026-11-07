# LLM Ops Cookbook — Recipe 001: Sliding-window pacing beats 429 cooldowns

**Part A · Throughput** · code: Node.js

## Problem

Free and cheap tiers give you a rate limit (e.g. 40 RPM) with a **sliding window**, not a fixed per-second quota. Naive senders either crawl (1 request every 2s "to be safe") or trip 429s and eat a 60–120s cooldown. Measured on a live 40-RPM free tier: burst 45 requests → 24 success / 20 rate-limited / 1 failed upstream in 12.8s. The limit is enforced over a rolling minute, and the penalty for tripping it is much larger than the reward for bursting.

## Why it bites

- **Cooldowns are channel-wide.** One 429 can freeze every model on that provider for 120s — you pay for one greedy request with a minute of downtime.
- **The window slides.** Requests you made 59s ago still count against you now. Fixed-interval senders slowly drift over the edge.
- **Reactive pacing wastes the budget.** After a 429 you back off 60s+ while your real entitlement was ~1.5 requests/second all along.

## Recipe

Pace client-side below the measured limit, and only fall back to cooldown when the server insists.

```js
// pacer.js — proactive sliding-window pacer for an N-RPM tier
class Pacer {
  constructor(rpm, { safety = 0.85 } = {}) {
    this.minIntervalMs = (60_000 / rpm) / safety; // 40 RPM -> ~1.59s at 85%
    this.last = 0;
  }
  async wait() {
    const now = Date.now();
    const due = this.last + this.minIntervalMs;
    if (now < due) await sleep(due - now);
    this.last = Date.now();
  }
}
const sleep = (ms) => new Promise(r => setTimeout(r, ms));

// usage
const pacer = new Pacer(40);
for (const item of jobs) {
  await pacer.wait();
  await send(item);            // 429s become rare, not routine
}
```

When a 429 does land (shared quota, someone else on the same key), cool down **that model only**, not the whole channel, and re-probe with a single cheap request before resuming the batch.

```js
const cooldowns = new Map();          // model -> untilTs
async function send(model, payload) {
  const until = cooldowns.get(model) || 0;
  if (Date.now() < until) await sleep(until - Date.now());
  try {
    return await call(model, payload);
  } catch (e) {
    if (e.status === 429) {
      cooldowns.set(model, Date.now() + 60_000);   // this model only
      throw e;
    }
    throw e;
  }
}
```

## Measure it first (30-line burst probe)

```js
// probe-rate.js — find the real limit in one run
const t0 = Date.now(); let ok = 0, limited = 0;
const results = await Promise.allSettled(
  Array.from({length: 45}, (_, i) => call('your-model', tinyPrompt)
    .then(() => ok++).catch(e => { if (e.status === 429) limited++; throw e; }).catch(() => {}))
);
console.log({ ok, limited, secs: ((Date.now()-t0)/1000).toFixed(1) });
// ok=24 limited=20  => entitlement ~= 40 RPM sliding window, burst penalty confirmed
```

## Checklist

- [ ] Measured the real limit with a burst probe (never trust docs)
- [ ] Pacer set to 80–90% of entitlement, min-interval style
- [ ] 429 cooldown applies per model, not per channel
- [ ] Burst probe re-run monthly (limits change without notice)
