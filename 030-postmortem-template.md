# LLM Ops Cookbook — Recipe 030: Postmortem template: the 180s timeout that wasn't yours
Part 6: Observability & Reliability | Language: Markdown (template) + Bash (generator)

## Problem
Our inference API started returning 504s after exactly 178–194 seconds of silence. Application logs showed the model worker completing in < 2 s, but the HTTP response never arrived. Packet captures revealed the upstream corporate proxy (Envoy, managed by another team) silently dropped connections at a hard 180 s idle timeout. Our client library had no idle timeout configured, so it waited indefinitely while the proxy had already severed the TCP session. The outage lasted 47 minutes until we correlated proxy access logs with our request IDs.

## Why it bites
- **Silent proxy termination**: No FIN/RST reaches the client; the socket appears open until the next read/write attempt.
- **Retry storms**: Automatic retries (default 3×) multiplied load on the model workers, turning a latency spike into a capacity incident.
- **Observability gap**: Our distributed traces stopped at the API gateway; the proxy hop was invisible, so the 180 s cliff looked like application slowness.
- **Configuration drift**: The proxy timeout was lowered from 300 s to 180 s during a security hardening sprint without cross-team notification.

## Recipe
### 1. Reusable postmortem template (12 fields)
Save as `postmortem-template.md` and copy for every incident.

```markdown
<!-- POSTMORTEM TEMPLATE — fill every section, delete this comment -->
# Incident: {{INCIDENT_ID}} — {{ONE_LINE_SUMMARY}}

**Date**: {{YYYY-MM-DD}}  
**Duration**: {{START_UTC}} → {{END_UTC}} ({{DURATION_MIN}} min)  
**Severity**: {{SEV_1|SEV_2|SEV_3}}  
**Author**: {{YOUR_NAME}}  
**Reviewers**: {{COMMA_SEPARATED}}

---

## 1. Timeline (UTC)
| Time | Event | Source |
|------|-------|--------|
| {{T1}} | {{EVENT_1}} | {{LOG_LINK_1}} |
| {{T2}} | {{EVENT_2}} | {{LOG_LINK_2}} |
| ... | ... | ... |

## 2. Impact
- **User-facing**: {{IMPACT_DESCRIPTION}}
- **Requests affected**: {{REQUEST_COUNT}} ({{PERCENTAGE}}% of traffic)
- **Revenue/SLA**: {{DOLLAR_OR_SLA_IMPACT}}

## 3. Root Cause
{{ROOT_CAUSE_NARRATIVE}}

## 4. Contributing Factors
- {{FACTOR_1}}
- {{FACTOR_2}}
- ...

## 5. Detection
- **How detected**: {{ALERT_OR_MANUAL}}
- **Time to detect**: {{TTD_MIN}} min
- **Gap**: {{DETECTION_GAP_DESCRIPTION}}

## 6. Resolution
- **Action taken**: {{MITIGATION_STEPS}}
- **Time to resolve**: {{TTR_MIN}} min
- **Owner**: {{RESOLVER}}

## 7. Action Items (prevent recurrence)
| # | Action | Owner | Due | Tracking Ticket |
|---|--------|-------|-----|-----------------|
| 1 | {{ACTION_1}} | {{OWNER_1}} | {{DUE_1}} | {{TICKET_1}} |
| 2 | {{ACTION_2}} | {{OWNER_2}} | {{DUE_2}} | {{TICKET_2}} |

## 8. Lessons Learned
### What went well
- {{GOOD_1}}
### What went poorly
- {{BAD_1}}
### Lucky breaks
- {{LUCK_1}}

## 9. Metrics Snapshot
| Metric | Value | Baseline |
|--------|-------|----------|
| p99 latency | {{P99}} ms | {{BASE_P99}} ms |
| Error rate | {{ERR_RATE}}% | {{BASE_ERR}}% |
| Throughput | {{THROUGHPUT}} rps | {{BASE_TPUT}} rps |

## 10. Architecture Diagram Link
{{DIAGRAM_URL}}

## 11. Related Incidents
- {{RELATED_1}}
- {{RELATED_2}}

## 12. Appendix
- Log queries: {{LOG_QUERIES}}
- Dashboards: {{DASHBOARD_LINKS}}
- Chat transcript: {{CHAT_EXPORT}}
```

### 2. One-shot generator (Bash)
Run `./new-postmortem.sh INC-2024-042 "Proxy 180s idle timeout"` to create a pre-filled file.

```bash
#!/usr/bin/env bash
# new-postmortem.sh — generate a postmortem from template
# Usage: ./new-postmortem.sh <INCIDENT_ID> "<ONE_LINE_SUMMARY>"
set -euo pipefail

INCIDENT_ID="${1:?incident id required}"
SUMMARY="${2:?summary required}"
DATE="$(date -u +%F)"
FILE="postmortem-${INCIDENT_ID}.md"

sed -e "s/{{INCIDENT_ID}}/${INCIDENT_ID}/g" \
    -e "s/{{ONE_LINE_SUMMARY}}/${SUMMARY}/g" \
    -e "s/{{YYYY-MM-DD}}/${DATE}/g" \
    postmortem-template.md > "${FILE}"

echo "Created ${FILE} — open and fill remaining placeholders."
```

### 3. Client-side fix (Python snippet)
Set an idle timeout **strictly below** the upstream proxy’s 180 s.

```python
# http_client.py — reusable async client with safe defaults
import httpx

# 170 s < 180 s proxy timeout; 5 s connect, 300 s total request
DEFAULT_TIMEOUT = httpx.Timeout(
    connect=5.0,
    read=170.0,      # idle read timeout
    write=10.0,
    pool=5.0,
)

client = httpx.AsyncClient(timeout=DEFAULT_TIMEOUT, limits=httpx.Limits(max_keepalive_connections=20))

async def safe_post(url: str, json: dict) -> httpx.Response:
    # Retries only on network errors, not on 5xx (handled upstream)
    return await client.post(url, json=json, timeout=DEFAULT_TIMEOUT)
```

## Checklist
- [ ] Postmortem template committed to `docs/incidents/postmortem-template.md`
- [ ] Generator script `new-postmortem.sh` executable and in PATH
- [ ] All HTTP clients (Python, Go, TypeScript) configure idle timeout ≤ 170 s
- [ ] Proxy timeout documented in architecture decision record (ADR)
- [ ] Alert on `proxy_idle_timeout_exceeded` counter (if exposed) or synthetic 175 s probe
- [ ] Next incident review includes "proxy timeout" as a mandatory checklist item
