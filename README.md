# AI Tools Blocklist

A continuously-updated, machine-classified database of AI-tool domains for network filtering, data-loss prevention, and shadow-AI governance. Every entry is a fully-qualified domain that hosts a generative-AI product  chatbots, code assistants, image and voice generators, autonomous agents, and the long tail of niche vertical tools that general-purpose URL filters have never heard of.

**Homepage:** https://www.aitoolsblocklist.com 

---

## Why this exists

Blocking AI tools sounds like a solved problem until you try to operationalize it. The naive approach  a hand-maintained list of `openai.com`, `claude.ai`, and a dozen other household names  covers maybe two percent of what is actually running on a corporate or school network. The generative-AI ecosystem now numbers in the tens of thousands of distinct services, and dozens of new ones ship every single day. A static list assembled last quarter is already stale: it misses the essay-writer your students adopted last week and the code-exfiltration-as-a-service tool your developers found yesterday.

The problem is structural. Real coverage requires three things that no spreadsheet can provide:

1. **Discovery at scale.** New AI tools do not announce themselves to your firewall. They surface as freshly-registered domains, subdomains of existing SaaS platforms, and rebrands of last month's tool. Finding them means continuously crawling newly-registered domains  we process roughly 300,000 per day against a 102-million-domain corpus  and running each candidate through a classifier that can tell an AI product from a marketing page that merely mentions AI.

2. **Classification, not pattern-matching.** A regex on the string `ai` is worse than useless; it flags `airbnb.com`, `dubai-tourism.ae`, and `email.example.org` while missing `cursor.sh` and `perplexity.ai`. Each domain in this database is classified by content into one of 18 functional categories and 172 subcategories, with a confidence score you can threshold on.

3. **Freshness as a product, not a promise.** The dataset is rebuilt daily. Additions, removals, and re-classifications are exposed through a delta endpoint so your enforcement point never drifts more than 24 hours from ground truth.

If your organization has a compliance obligation  GDPR, CIPA, FERPA, HIPAA, SOC 2, or an internal acceptable-use policy  "we block the AI tools we happen to know about" is not a defensible control. This database is the control.

---

## What's in the data

Each record is a domain plus its classification metadata:

| Field | Description |
|-------|-------------|
| `domain` | Fully-qualified domain, e.g. `claude.ai` |
| `primary_category` | One of 18 top-level categories |
| `subcategory` | One of 172 fine-grained subcategories |
| `confidence` | Classifier confidence, 0.0â€“1.0 |
| `risk_tier` | `low` \| `medium` \| `high`, derived from data-handling behavior |
| `data_handling` | e.g. `cloud-processed`, `on-device`, `trains-on-input` |
| `first_seen` | When the domain first entered the dataset (ISO 8601) |
| `last_verified` | Last time the classification was re-confirmed |

The 18 top-level categories span the entire landscape: Text & Language, Image & Visual, Video, Audio/Voice/Music, Code & Development, Agents & Automation, Data & Analytics, Search & Knowledge, Productivity, Marketing/Sales, Customer Support, Business & Vertical, Education, Design & Creative, Models & Infrastructure, Security & Detection, Lifestyle, and Robotics & Embodied AI.

**Free sample:** a 500-row CSV is available with no signup  https://www.aitoolsblocklist.com/sample_databases/ai_tools_database_sample.csv

---

## Delivery formats

The same dataset ships in the exact shapes your infrastructure already consumes, so integration is a configuration change rather than an engineering project:

- **CSV / JSON**  the full classified dataset, for pipelines, SIEMs, and analytics.
- **REST API**  real-time single-domain lookup, filtered list retrieval, bulk classification, and a delta feed.
- **EDL (External Dynamic List)**  a plain newline-delimited domain list that next-generation firewalls poll on a schedule.
- **PAC file**  a `FindProxyForURL` function that routes matched domains to a block proxy.
- **Hosts file**  `0.0.0.0 domain` lines for endpoint-level and Pi-hole-style enforcement.
- **DNS RPZ**  a Response Policy Zone for resolver-level blocking (BIND, Unbound, Knot).

---

## REST API

Base URL and versioning:

```
https://api.aitoolsblocklist.com/v1/

GET  /v1/lookup/{domain}     # Single domain classification
GET  /v1/domains             # Full domain list with filters
GET  /v1/delta               # Changes since a timestamp
POST /v1/bulk-lookup         # Classify up to 1,000 domains
GET  /v1/taxonomy            # Category tree with IDs
GET  /v1/stats               # Database statistics
```

Authentication is a Bearer token in the `Authorization` header:

```bash
curl -H "Authorization: Bearer atb_your_api_key_here" \
     "https://api.aitoolsblocklist.com/v1/stats"
```

### Single-domain lookup

```bash
curl -H "Authorization: Bearer atb_your_api_key_here" \
     "https://api.aitoolsblocklist.com/v1/lookup/claude.ai"
```

```json
{
  "domain": "claude.ai",
  "in_blocklist": true,
  "primary_category": "Text & Language",
  "subcategory": "General assistants & chatbots",
  "confidence": 0.99,
  "risk_tier": "high",
  "data_handling": "cloud-processed",
  "status": "active",
  "first_seen": "2023-07-11T00:00:00Z",
  "last_verified": "2026-07-09T06:00:00Z"
}
```

A domain that is not an AI tool returns `"in_blocklist": false` with `200 OK`  the absence of a match is an answer, not an error.

### Bulk classification

```bash
curl -X POST "https://api.aitoolsblocklist.com/v1/bulk-lookup" \
     -H "Authorization: Bearer atb_your_api_key_here" \
     -H "Content-Type: application/json" \
     -d '{"domains": ["cursor.sh", "midjourney.com", "example.org"]}'
```

### Delta feed  stay current without re-downloading everything

```bash
curl -H "Authorization: Bearer atb_your_api_key_here" \
     "https://api.aitoolsblocklist.com/v1/delta?since=2026-07-08T00:00:00Z"
```

The delta response contains `added`, `removed`, and `reclassified` arrays. Poll it once a day and apply the diff to your enforcement point; you never move more than 24 hours out of sync, and you never re-transfer the full dataset.

### Filtering, pagination, and rate limits

The `/v1/domains` endpoint accepts query parameters that compose: `category`, `subcategory`, `min_confidence`, `risk_tier`, and `updated_since`. Results are cursor-paginated  each response carries a `next_cursor` you pass back to retrieve the following page, which keeps pagination stable even as the underlying dataset changes between requests.

```bash
# Only high-confidence Code & Development domains, 1,000 per page
curl -H "Authorization: Bearer atb_your_api_key_here" \
     "https://api.aitoolsblocklist.com/v1/domains?category=code-development&min_confidence=0.9&limit=1000"
```

Every response includes `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers. When you exhaust a window the API returns `429 Too Many Requests` with a `Retry-After` header  back off for the indicated number of seconds rather than hammering the endpoint. Responses are gzip-encoded when you send `Accept-Encoding: gzip`, which matters when you are pulling the full list.

---

## Integration

### Next-generation firewall (EDL)

Point the firewall at the hosted EDL endpoint and set a refresh interval. The firewall fetches the list on its own schedule; you write one policy rule that references the object.

```bash
# The EDL is a plain text file, one domain per line:
curl -H "Authorization: Bearer atb_your_api_key_here" \
     "https://api.aitoolsblocklist.com/v1/domains?format=edl" \
     -o ai-tools.edl
```

Configure the firewall's external-dynamic-list object to poll that URL hourly, then attach it to a security policy with an action of `deny`. No commits are needed for content updates  the firewall re-reads the list on its interval.

### DNS resolver (RPZ)

For DNS-level enforcement, consume the RPZ zone. In BIND, add a `response-policy` clause and pull the zone via AXFR or a scheduled `named` reload:

```
response-policy { zone "ai-tools.rpz"; };

zone "ai-tools.rpz" {
    type master;
    file "/etc/bind/ai-tools.rpz";
};
```

Regenerate the zone file from the delta feed on a cron and reload. Queries to any listed domain return `NXDOMAIN` (or a walled-garden address, if you prefer to redirect rather than drop).

### Endpoints / Pi-hole (hosts file)

```bash
curl -H "Authorization: Bearer atb_your_api_key_here" \
     "https://api.aitoolsblocklist.com/v1/domains?format=hosts" \
     -o /etc/pihole/ai-tools.hosts
```

Add the file as a local block source and run `pihole -g`. Every listed domain resolves to `0.0.0.0`.

### Proxy (PAC file)

```bash
curl -H "Authorization: Bearer atb_your_api_key_here" \
     "https://api.aitoolsblocklist.com/v1/domains?format=pac" \
     -o ai-tools.pac
```

The generated `FindProxyForURL` sends matched hosts to your block proxy and everything else `DIRECT`.

### CI / build pipeline

Because the API is stateless HTTP with a stable schema, it drops into any language. A minimal Python guard that fails a build if a dependency phones home to an AI service:

```python
import requests

API = "https://api.aitoolsblocklist.com/v1/bulk-lookup"
KEY = "atb_your_api_key_here"

def flagged(domains):
    r = requests.post(API, headers={"Authorization": f"Bearer {KEY}"},
                      json={"domains": domains}, timeout=10)
    r.raise_for_status()
    return [d["domain"] for d in r.json()["results"] if d["in_blocklist"]]

hits = flagged(["cursor.sh", "internal-cdn.example.org"])
if hits:
    raise SystemExit(f"AI-tool endpoints detected: {hits}")
```

---

## How classification works

Understanding the pipeline helps you set thresholds sensibly. A candidate domain moves through four stages before it earns a place in the dataset:

1. **Ingestion.** Newly-registered and newly-observed domains enter from certificate-transparency logs, zone-file deltas, and the crawl corpus. This is the funnel  hundreds of thousands of candidates per day, the vast majority of which are not AI tools.

2. **Content fetch and feature extraction.** Each candidate is fetched with a headless renderer so JavaScript-heavy single-page apps are captured, not just their empty HTML shells. We extract on-page text, meta tags, structured data, and API-surface hints.

3. **Classification.** The extracted content is scored against the 18-category, 172-subcategory taxonomy. The output is a category assignment plus a `confidence` value. A page that merely *mentions* AI scores low; a page that *is* an AI product  with a prompt box, a pricing tier for tokens or generations, a model selector  scores high. This is why `min_confidence` is the single most useful filter: set it to `0.9` for a low-false-positive enforcement list, or lower it to `0.6` when you would rather over-block and review.

4. **Verification and decay.** Classifications are re-confirmed on a rolling schedule; `last_verified` records when. Domains that go dark, park, or pivot away from AI are removed via the delta feed's `removed` array, so your list does not accumulate rot.

Because the whole pipeline is deterministic given its inputs, two lookups of the same domain on the same day return identical classifications  important when you are diffing enforcement state across sites.

## Update cadence and versioning

The dataset is rebuilt every 24 hours. The API is versioned under `/v1/`; breaking schema changes ship under a new version prefix, and the previous version is supported in parallel. `last_verified` on each record tells you exactly how fresh a given classification is.

---

## Standards and further reading

This project sits alongside the broader web-filtering and content-classification ecosystem. Useful non-commercial references:

- IETF DNS Response Policy Zones  https://datatracker.ietf.org/doc/html/draft-vixie-dns-rpz
- IANA root zone and TLD registry  https://www.iana.org/domains/root
- Mozilla PAC / `FindProxyForURL` specification  https://developer.mozilla.org/en-US/docs/Web/HTTP/Proxy_servers_and_tunneling/Proxy_Auto-Configuration_PAC_file
- ICANN registration data (WHOIS/RDAP)  https://www.icann.org/resources/pages/rdap-2019-08-05-en
- NIST guidance on protecting controlled unclassified information  https://csrc.nist.gov/publications/detail/sp/800-171/rev-2/final

---

## License

The dataset is a commercial subscription. A free 500-row sample is available at https://www.aitoolsblocklist.com/sample_databases/ai_tools_database_sample.csv for evaluation. Full access, an API key, and the complete daily feed are provided through https://www.aitoolsblocklist.com

Maintained by the team behind https://www.aitoolsblocklist.com  questions and integration help: `info@alpha-quantum.com`.
