# LLM Ops Recipes

Three field-tested recipes for running LLM workloads in production, extracted from a live
multi-provider gateway that serves 33 models across NVIDIA NIM, Kilo, OpenRouter and
Pollinations channels, with scheduled jobs that must survive unattended nights.

Each recipe is one page: **Problem → Why it bites → Recipe (runnable code) → Checklist**.

| Recipe | What it solves |
|---|---|
| [001 — Sliding-window pacing](001-sliding-window-pacing.md) | Stop 429 cascades and get ~2x throughput on the same free-tier quota |
| [029 — The 15-second window](029-wechat-15s-window.md) | Reply inside WeChat's passive-reply window when your upstream takes 20–180s |
| [030 — Postmortem template](030-postmortem-template.md) | Every request hung 178–194s and your code was fine: the upstream proxy had a hard timeout |

## Why these three

They are the three failures that cost the most time in the first month of running LLM jobs:
silent hangs, rate-limit thrash, and a 3am incident whose root cause was not in your code.

## Numbers are real

- 40 RPM sliding window measured with a burst probe: 45 requests → 24 success / 20 rate-limited
- Stream idle timeout distinguishes connect-fail, idle-fail and slow-success
- 95K context served in 18.6s on the fast free tier; the slow one took 83s

## Related

- Free tools that run entirely in your browser (nothing uploaded): <https://csvtriage.com/box/>
- The full set of 30 recipes, plus 24 agent recipes and 150 developer prompts, is available
  as a paid pack: <https://csvtriage.com/p/llm-ops-cookbook.html>

## License

MIT for the recipe text and code samples in this repository.
