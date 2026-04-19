---
name: porkbun
description: Interface with the Porkbun API for domain registration, DNS management, SSL retrieval, URL forwarding, glue records, DNSSEC, and account operations. Uses a locally cached OpenAPI spec queried with jq.
version: 1.0.0
author: agentskills
tags: [porkbun, dns, domains, ssl, registrar, api]
---

# Porkbun API Skill

Agent-friendly interface to the full Porkbun API (~32 endpoints). The skill ships with a cached copy of Porkbun's official OpenAPI spec and tells the agent how to query it with `jq`, handle credentials safely, and apply appropriate caution to write operations.

## How this skill works

This skill is a thin pointer to Porkbun's official OpenAPI spec. Instead of duplicating endpoint documentation in markdown (which drifts), the spec **is** the reference:

- `spec.json` — cached copy of `https://porkbun.com/api/json/v3/spec` (~90 KB)
- `spec.meta` — sidecar with sha256, size, version, and timestamps for the cached spec

The agent queries `spec.json` with `jq` on demand. **Never read the full `spec.json` into context** — it's ~20K tokens. Always extract the specific endpoint or schema needed.

Throughout this document, `<SKILL_DIR>` refers to the directory this `SKILL.md` lives in (provided to you when this skill is invoked).

## Prerequisites

```bash
# jq is required — the skill uses it to query the cached spec
brew install jq              # macOS
apt-get install jq           # Debian/Ubuntu

# curl is assumed present (standard on macOS/Linux)
```

## First action on every invocation: freshness check

Before executing the user's task, perform this check. It is fast, cached, and fails open.

### Step 1 — skip if recently checked

Read `<SKILL_DIR>/spec.meta`. If `checked_at` is less than 6 hours old, skip the freshness check entirely and proceed to the user's task.

```bash
jq -r '.checked_at' "<SKILL_DIR>/spec.meta"
```

### Step 2 — fetch and hash

Porkbun does not serve `ETag` or `Last-Modified` headers (`cache-control: no-cache`), so the only reliable comparison is a body hash. Fetch the spec and compare:

```bash
curl -sSL https://porkbun.com/api/json/v3/spec -o /tmp/porkbun-spec-candidate.json
CANDIDATE_SHA=$(shasum -a 256 /tmp/porkbun-spec-candidate.json | awk '{print $1}')
CACHED_SHA=$(jq -r '.sha256' "<SKILL_DIR>/spec.meta")
```

If the fetch fails (network error, non-200), **log one line and proceed with the cached spec**. Never block the user's task on a failed update check.

### Step 3 — update `checked_at` if unchanged

If `CANDIDATE_SHA == CACHED_SHA`, the spec is current. Update only the `checked_at` timestamp in `spec.meta` and proceed to the user's task:

```bash
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
jq --arg now "$NOW" '.checked_at = $now' "<SKILL_DIR>/spec.meta" > "<SKILL_DIR>/spec.meta.tmp" \
  && mv "<SKILL_DIR>/spec.meta.tmp" "<SKILL_DIR>/spec.meta"
rm /tmp/porkbun-spec-candidate.json
```

### Step 4 — if changed, ask the user

If `CANDIDATE_SHA != CACHED_SHA`, **do not overwrite the cache automatically**. Tell the user:

> Porkbun's API spec has changed since your local cache (sha256 differs). Want me to update `spec.json`? Changes could be new endpoints, renamed fields, or nothing material. I can show you the diff first if you'd like.

Offer a diff preview using:

```bash
diff <(jq -S . "<SKILL_DIR>/spec.json") <(jq -S . /tmp/porkbun-spec-candidate.json) | head -100
```

Only on explicit user approval, perform the update:

```bash
mv /tmp/porkbun-spec-candidate.json "<SKILL_DIR>/spec.json"
SHA=$(shasum -a 256 "<SKILL_DIR>/spec.json" | awk '{print $1}')
SIZE=$(wc -c < "<SKILL_DIR>/spec.json" | tr -d ' ')
VERSION=$(jq -r '.info.version' "<SKILL_DIR>/spec.json")
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
printf '{\n  "source_url": "https://porkbun.com/api/json/v3/spec",\n  "sha256": "%s",\n  "size_bytes": %s,\n  "spec_version": "%s",\n  "fetched_at": "%s",\n  "checked_at": "%s"\n}\n' \
  "$SHA" "$SIZE" "$VERSION" "$NOW" "$NOW" > "<SKILL_DIR>/spec.meta"
```

If the user declines the update, clean up and proceed with the cached spec:

```bash
rm /tmp/porkbun-spec-candidate.json
```

## Querying the spec with jq

The cached spec follows the OpenAPI 3.0 structure. Common queries:

```bash
# List every endpoint path
jq -r '.paths | keys[]' "<SKILL_DIR>/spec.json"

# Full definition of one endpoint
jq '.paths["/dns/create/{domain}"]' "<SKILL_DIR>/spec.json"

# Just the request body schema for an endpoint
jq '.paths["/dns/create/{domain}"].post.requestBody' "<SKILL_DIR>/spec.json"

# All endpoints matching a prefix (e.g. DNS)
jq -r '.paths | keys[] | select(startswith("/dns"))' "<SKILL_DIR>/spec.json"

# A named component schema
jq '.components.schemas.DnsRecord' "<SKILL_DIR>/spec.json"

# API-wide metadata: auth, servers, rate limits (embedded in description)
jq '.info' "<SKILL_DIR>/spec.json"
jq '.servers' "<SKILL_DIR>/spec.json"
jq '.components.securitySchemes' "<SKILL_DIR>/spec.json"
```

Rule of thumb: start from the endpoint list, pick the right one, then pull just that endpoint's full definition. Avoid reading large slices of the spec at once.

## Authentication

Porkbun requires an API key + secret API key. Get them at https://porkbun.com/account/api.

### Credential resolution order

1. **Environment variables** (preferred):
   - `PORKBUN_API_KEY`
   - `PORKBUN_SECRET_KEY`
2. **Interactive prompt** (fallback): if env vars are missing, ask the user for them and export for the session only. Never write credentials to disk, never echo them back, never include them in command output the user sees.

```bash
# Check for env vars first
if [ -z "$PORKBUN_API_KEY" ] || [ -z "$PORKBUN_SECRET_KEY" ]; then
    # Prompt user, then export for this session
    export PORKBUN_API_KEY="<from user>"
    export PORKBUN_SECRET_KEY="<from user>"
fi
```

### Header auth vs body auth

Porkbun supports two auth methods — prefer headers so secrets don't appear in shell history:

```bash
# Preferred: header auth (read and write endpoints)
curl -sS -X POST https://api.porkbun.com/api/json/v3/dns/retrieve/example.com \
  -H "X-API-Key: $PORKBUN_API_KEY" \
  -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY"

# Body auth (required by some SDK examples — equivalent, but leaks secrets into shell history)
curl -sS -X POST https://api.porkbun.com/api/json/v3/dns/retrieve/example.com \
  -H "Content-Type: application/json" \
  -d "{\"apikey\":\"$PORKBUN_API_KEY\",\"secretapikey\":\"$PORKBUN_SECRET_KEY\"}"
```

Note: header auth takes effect **only when no body credentials are present**. Don't mix.

### Verify credentials work

Use `/ping` — it echoes back the public IP on success, returns `INVALID_API_KEYS_001` on bad credentials:

```bash
curl -sS -X POST https://api.porkbun.com/api/json/v3/ping \
  -H "X-API-Key: $PORKBUN_API_KEY" \
  -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" | jq
```

### Per-domain API Access (common footgun)

Every domain in a Porkbun account has an **API Access toggle** that must be enabled in the Porkbun UI (Domain Management → per-domain settings) before the API will accept operations against it. If you see `INVALID_DOMAIN` on a domain that clearly exists in the account's `listAll` output, that's almost always the cause. Tell the user to toggle it on and retry.

## Safety tiers

Every write has real-world consequences — money, DNS changes, or resolution breakage. Before executing a write, classify it and apply the matching confirmation pattern.

| Tier | Examples | Agent behavior |
|---|---|---|
| **Safe (read)** | `/ping`, `/ip`, `/domain/listAll`, `/domain/getNs`, `/domain/getUrlForwarding`, `/domain/getGlue`, `/dns/retrieve*`, `/dns/getDnssecRecords`, `/ssl/retrieve`, `/pricing/get`, `/marketplace/getAll`, `/apikey/retrieve` | Execute without prompting. |
| **Routine write** | `/dns/create`, `/dns/edit`, `/dns/delete`, `/dns/editByNameType`, `/dns/deleteByNameType`, `/domain/updateAutoRenew`, `/domain/addUrlForward`, `/email/setPassword` | Show a one-line summary of the change (domain, record type, before/after) and get a yes/no confirmation. |
| **High-risk** | `/domain/create` (spends money), `/domain/updateNs`, `/domain/deleteUrlForward`, `/dns/createDnssecRecord`, `/dns/deleteDnssecRecord`, `/domain/createGlue`, `/domain/updateGlue`, `/domain/deleteGlue`, `/apikey/request`, `/account/invite` | Show the exact cost (if monetary) and impact (what can break), and require explicit typed confirmation like `yes, register example.com for $9.73`. |

Unknown / newly-added endpoints: treat as **routine write** by default. Confirm first.

## Error handling

Branch on the `code` field, not `message`. Messages are for display; codes are stable.

| Code | Handling |
|---|---|
| `INVALID_API_KEYS_001` | Credentials are wrong. Re-prompt the user; don't retry with the same keys. |
| `INVALID_DOMAIN` | Almost always "API Access not enabled for this domain." Confirm via `/domain/listAll`, then tell the user to toggle API Access in the Porkbun UI. |
| `DOMAIN_NOT_AVAILABLE` | Registration failed because the domain is taken. Do not retry. |
| `INSUFFICIENT_FUNDS` | Registration blocked. Tell the user to top up their account balance; do not retry. |
| `RATE_LIMIT_EXCEEDED` | Back off using `X-RateLimit-Reset` header (unix timestamp) or `ttlRemaining` field. |
| `INVALID_TYPE` / `INVALID_RECORD_ID` | Input error. Re-check against the spec; surface the problem to the user. |

Full error code list lives in the spec under `.info.description` — search for "Error codes".

## Rate limiting

Two fixed-limit endpoints (`/apikey/request`, `/apikey/retrieve`) return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers on every call. Other endpoints are rate-limited but without per-response headers.

On HTTP 429:

```bash
# Extract reset time and sleep until then
RESET=$(curl -sI ... | grep -i '^x-ratelimit-reset:' | awk '{print $2}' | tr -d '\r')
NOW=$(date -u +%s)
sleep $(( RESET - NOW + 1 ))
```

Prefer `ttlRemaining` from the JSON body when headers aren't present.

## Common workflows

### List all domains in the account

```bash
curl -sS -X POST https://api.porkbun.com/api/json/v3/domain/listAll \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" \
  -d '{"start":"0","includeLabels":"yes"}' | jq
```

### Retrieve DNS records for a domain

```bash
curl -sS -X POST https://api.porkbun.com/api/json/v3/dns/retrieve/example.com \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" | jq
```

### Add an A record (routine write — confirm first)

Full schema: `jq '.paths["/dns/create/{domain}"].post.requestBody' <SKILL_DIR>/spec.json`

```bash
curl -sS -X POST https://api.porkbun.com/api/json/v3/dns/create/example.com \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"type":"A","name":"www","content":"1.2.3.4","ttl":"600"}' | jq
```

### Check domain availability and pricing

```bash
curl -sS -X POST https://api.porkbun.com/api/json/v3/domain/checkDomain/example.com \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" | jq
```

### Retrieve the Porkbun-issued SSL bundle for a domain

```bash
curl -sS -X POST https://api.porkbun.com/api/json/v3/ssl/retrieve/example.com \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" | jq
```

Response contains `certificatechain`, `privatekey`, and `publickey` — treat as secrets.

### Update nameservers (high-risk — require typed confirmation)

Changing nameservers takes the domain off Porkbun's DNS and can break resolution for hours if the new NS records are wrong. Require the user to type `yes, switch nameservers for example.com`:

```bash
curl -sS -X POST https://api.porkbun.com/api/json/v3/domain/updateNs/example.com \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ns":["ns1.example.net","ns2.example.net"]}' | jq
```

### Register a new domain (high-risk — spends money)

Two-step: check price, then register with the price in pennies. Require the user to type `yes, register <domain> for $<amount>`:

```bash
# Step 1: check availability and price (read — safe)
RESP=$(curl -sS -X POST https://api.porkbun.com/api/json/v3/domain/checkDomain/newdomain.com \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY")
echo "$RESP" | jq

# Step 2: register (spends real money — confirm first)
PRICE_USD=$(echo "$RESP" | jq -r '.response.price')
PRICE_PENNIES=$(printf '%.0f' "$(echo "$PRICE_USD * 100" | bc)")
curl -sS -X POST https://api.porkbun.com/api/json/v3/domain/create/newdomain.com \
  -H "X-API-Key: $PORKBUN_API_KEY" -H "X-Secret-API-Key: $PORKBUN_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"cost\":$PRICE_PENNIES,\"agreeToTerms\":\"yes\"}" | jq
```

## Resources

- [Porkbun API dashboard (get keys)](https://porkbun.com/account/api)
- [Live OpenAPI spec](https://porkbun.com/api/json/v3/spec)
- [Porkbun API docs](https://porkbun.com/api/json/v3/documentation)
- [Porkbun support](https://kb.porkbun.com/)
