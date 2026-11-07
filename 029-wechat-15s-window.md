# LLM Ops Cookbook — Recipe 029: The 15-second window: slow upstreams vs messaging UX
Part: Messaging Integrations · Language: Node.js (TypeScript)

## Problem
WeChat's passive reply API demands a response within ~5 seconds per attempt, with up to three automatic retries spanning roughly 15 seconds total. A typical LLM chain — retrieval, rerank, generation — routinely takes 20–180 seconds, so the first two retries hit an empty handler and the third arrives after the user has already seen "no response." Most integrations silently drop the reply because they treat each webhook delivery as independent, losing the correlation between the original `MsgId` and its redeliveries. The result: users receive nothing, logs show 200 OK, and the failure is invisible until support tickets pile up.

## Why it bites
- **Idempotency key mismatch**: WeChat retries use the same `MsgId` but your handler generates a new request ID, so deduplication caches miss and you either double-process or drop the retry.
- **Empty-reply semantics**: Returning `""` or `"success"` on the first hit satisfies the 5s SLA but teaches the platform your bot is healthy, suppressing alerts while the real work times out in the background.
- **Streaming cutoff blindness**: Streaming tokens after 4s looks fast locally, but the HTTP response isn't flushed until the stream ends; WeChat sees a hanging connection and retries anyway.
- **Cache stampede on redelivery**: If the first attempt spawns an async job but doesn't store its handle, the retry spawns a duplicate job, doubling upstream cost and latency.

## Recipe
```typescript
// recipe-029-wechat-15s-window.ts
// Minimal Express handler for WeChat passive reply with 15s retry window
// Requires: npm i express ioredis @types/express @types/node
// Placeholders: REDIS_URL, LLM_API_KEY, LLM_ENDPOINT

import express, { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';
import crypto from 'crypto';

const app = express();
app.use(express.json({ limit: '1mb' }));

const redis = new Redis(process.env.REDIS_URL ?? 'redis://localhost:6379', {
  maxRetriesPerRequest: 1,
  enableReadyCheck: true,
  lazyConnect: true,
});

const LLM_ENDPOINT = process.env.LLM_ENDPOINT ?? 'https://api.example.com/v1/chat/completions';
const LLM_API_KEY = process.env.LLM_API_KEY ?? 'PLACEHOLDER_KEY';
const SYNC_WINDOW_MS = 4000;          // must reply within this to beat first retry
const MAX_TOTAL_MS = 14500;           // hard ceiling before 3rd retry arrives
const CACHE_TTL_SEC = 30;             // covers 3 retries + margin

interface WeChatMsg {
  ToUserName: string;
  FromUserName: string;
  CreateTime: number;
  MsgType: string;
  Content: string;
  MsgId: string;                      // idempotency key across retries
}

async function callLLM(prompt: string, signal: AbortSignal): Promise<string> {
  const ctrl = new AbortController();
  signal.addEventListener('abort', () => ctrl.abort(signal.reason));

  const res = await fetch(LLM_ENDPOINT, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${LLM_API_KEY}`,
    },
    body: JSON.stringify({
      model: 'gpt-4o-mini',
      messages: [{ role: 'user', content: prompt }],
      stream: false,                  // streaming adds flush latency; avoid for sync path
      max_tokens: 300,                // keeps generation < 2s typical
      temperature: 0.3,
    }),
    signal: ctrl.signal,
  });

  if (!res.ok) throw new Error(`LLM ${res.status}: ${await res.text()}`);
  const data = await res.json() as { choices: { message: { content: string } }[] };
  return data.choices[0]?.message?.content?.trim() ?? '';
}

function shortFallback(prompt: string): string {
  // Deterministic, <200 chars, no upstream call — used when cache miss & time critical
  const hash = crypto.createHash('sha256').update(prompt).digest('hex').slice(0, 8);
  return `[思考中…] 收到："${prompt.slice(0, 50)}" (ref:${hash})`;
}

app.post('/wechat', async (req: Request, res: Response, next: NextFunction) => {
  const msg = req.body as WeChatMsg;
  if (!msg?.MsgId || msg.MsgType !== 'text') return res.send('success'); // non-text: ack fast

  const cacheKey = `wechat:reply:${msg.MsgId}`;
  const lockKey = `wechat:lock:${msg.MsgId}`;

  // 1. Fast path: cached final reply exists → return immediately (<5ms)
  const cached = await redis.get(cacheKey);
  if (cached) return res.send(cached);

  // 2. Acquire single-flight lock for this MsgId (NX + EX = 12s covers 2 retries)
  const gotLock = await redis.set(lockKey, '1', 'NX', 'EX', 12);
  if (!gotLock) {
    // Another request (retry) is processing; wait briefly for its result
    for (let i = 0; i < 8; i++) {      // 8 * 250ms = 2s max wait
      await new Promise(r => setTimeout(r, 250));
      const maybe = await redis.get(cacheKey);
      if (maybe) return res.send(maybe);
    }
    // Lock holder stalled; fall through to generate fallback so user gets something
  }

  // 3. We are the elected processor — start timer
  const deadline = Date.now() + SYNC_WINDOW_MS;
  const abortCtrl = new AbortController();
  const timeoutId = setTimeout(() => abortCtrl.abort(new Error('sync window')), SYNC_WINDOW_MS);

  try {
    // 3a. Try real LLM with tight deadline
    let reply: string;
    if (Date.now() + 2000 < deadline) { // only call if >2s headroom
      reply = await callLLM(msg.Content, abortCtrl.signal);
    } else {
      throw new Error('insufficient headroom');
    }

    // 3b. Store final reply for retries (TTL covers full 15s window)
    await redis.set(cacheKey, reply, 'EX', CACHE_TTL_SEC);
    return res.send(reply);
  } catch (err) {
    // 4. Any timeout/error → emit short fallback, cache it, still 200 OK
    const fallback = shortFallback(msg.Content);
    await redis.set(cacheKey, fallback, 'EX', CACHE_TTL_SEC);
    return res.send(fallback);
  } finally {
    clearTimeout(timeoutId);
    // Release lock only if we still own it (prevents race with retry waiter)
    await redis.eval(`if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`, 1, lockKey, '1');
  }
});

// Health check for load balancer
app.get('/healthz', (_req, res) => res.send('ok'));

const PORT = process.env.PORT ?? 3000;
app.listen(PORT, () => console.log(`WeChat handler listening on :${PORT}`));
```

## Checklist
- [ ] Redis `SET NX EX` lock TTL ≥ 12s (covers two retries) and cache TTL ≥ 30s (covers full window + clock skew)
- [ ] Sync window constant (`SYNC_WINDOW_MS`) ≤ 4000ms measured end-to-end including TLS handshake to LLM
- [ ] Fallback reply ≤ 200 chars, deterministic, contains zero PII, and renders correctly in WeChat chat bubble
- [ ] Load test with `wrk -t4 -c100 -d30s --latency` simulating 3 retries per `MsgId`; p99 < 4.5s on first attempt
- [ ] Alert on `wechat:lock:*` keys older than 15s (stuck processor) and on cache-miss rate > 5% (upstream degradation)
- [ ] Document `MsgId` correlation in runbook: how to replay a dropped reply by re-posting to `/wechat` with original `MsgId`
