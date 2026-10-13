# Generic Lead Validator — Application & Codebase Specification

> A source-agnostic rebuild of the Lead Validator. Same architecture, same pipeline, same four-color confidence tiers — but every field name, scoring rule, and enrichment check is driven by configuration, so any website intake form (mortgage, insurance, solar, legal, home services, B2B demo requests, etc.) can be validated without code changes.

| | |
|---|---|
| Version | v3.0 (generic) — derived from THL Lead Validator v2.0 |
| Stack | Node.js 18+ · Express 4 · Vanilla JS dashboard |
| Runtime | Single process, file-based storage (pluggable DB adapter) |
| Input | Any JSON/CSV lead from any web form, via webhook or batch |
| Output | Original lead + `validation_score`, `confidence_color`, `notes`, `_enrichment`, `_rule_hits` |

---

## 0. What Changed From the Original

| Original (mortgage-specific) | Generic version |
|---|---|
| Hard-coded field names (`"Credit Rating"`, `"Moving Date"`, `"Property City"`) | **Intake profiles** map any source's field names to a canonical schema |
| 18 hard-coded scoring signals in `scoreLead()` | **Rules engine** — scoring rules are JSON in config, editable in the dashboard |
| Tier thresholds hard-coded (74 / 49 / 25) | Tier thresholds configurable; defaults unchanged |
| Score capped at 100 | Choice of `capped` (original) or `normalized` (% of achievable points) |
| TrustedForm +8 and AI fraud −15 hard-coded | Both are ordinary rules with editable weights |
| One lead type | Many profiles, selected per request by `source`/`profile` |
| No regression testing | **Validation test harness** — fixture leads with expected tiers, run from dashboard, API, or CI |

Everything else — enrichment providers, parallel pipeline, webhook auth, outgoing forward, dashboard, logs, storage — behaves exactly as in v2.0.

---

## 1. Confidence Tiers

| Tier | Default range | Meaning | Suggested action |
|---|---|---|---|
| 🟢 GREEN | 75 – 100 | High confidence — identity, contact and intent all corroborate | Route immediately |
| 🔵 BLUE | 50 – 74 | Good confidence — minor gaps | Route same day |
| 🟠 AMBER | 26 – 49 | Low confidence — key signals missing or unverified | Nurture / secondary check |
| 🔴 RED | 0 – 25 | Very low confidence — likely bad data or fraud | Manual review / reject |

```json
"tiers": [
  { "name": "GREEN", "min": 75, "color": "#16A34A" },
  { "name": "BLUE",  "min": 50, "color": "#2563EB" },
  { "name": "AMBER", "min": 26, "color": "#D97706" },
  { "name": "RED",   "min": 0,  "color": "#DC2626" }
]
```
Tiers are evaluated top-down; the first tier whose `min` ≤ score wins.

---

## 2. File Structure

```
lead-validator/
├── server.js                 ← Express app, routes, pipeline orchestration
├── package.json              ← express, cors only
├── README.md
├── lib/
│   ├── storage.js            ← readJSON / writeJSON / addLog / deepMerge (DB-swappable)
│   ├── normalize.js          ← intake profile → canonical lead mapping
│   ├── rules.js              ← generic rules engine (operators, scoring, tiering)
│   ├── pipeline.js           ← processLead(): map → enrich → score → AI → store → forward
│   ├── testHarness.js        ← fixture runner for validation tests
│   └── providers/            ← one file per enrichment provider, common interface
│       ├── ipApi.js
│       ├── emailVerify.js    (Hunter.io)
│       ├── phoneVerify.js    (Numverify)
│       ├── melissaPerson.js
│       ├── melissaAddress.js
│       ├── trustedForm.js
│       ├── jornaya.js        (LeadiD)
│       ├── userAgent.js
│       ├── crmLookup.js      (Snowflake or any SQL REST)
│       ├── duplicate.js      (local dedupe)
│       └── aiReview.js       (Azure OpenAI / any chat-completions endpoint)
├── profiles/                 ← intake profiles (one JSON per form/source)
│   ├── default.json
│   └── mortgage-survey.json  ← the original THL mapping, kept as an example
├── tests/
│   └── fixtures/*.json       ← validation test cases
├── public/
│   └── index.html            ← single-page dashboard
└── data/                     ← auto-created
    ├── config.json
    ├── leads.json            (max 2,000)
    ├── logs.json             (max 500)
    └── test-runs.json        (max 50)
```

No build step, no bundler, no framework beyond Express.

---

## 3. Architecture

```
 Any website form ─┐
 Zapier / Make ────┼──► POST /webhook/lead?profile=<id>   (x-api-key)
 Direct API ───────┘          │
 CSV upload (dashboard) ─► POST /api/leads/validate
 Test fixtures ──────────► POST /api/tests/run
                              │
                     ┌────────▼─────────┐
                     │ 1. normalize()   │  intake profile → canonical lead
                     ├──────────────────┤
                     │ 2. enrich()      │  Promise.all(enabled providers)
                     │   ip · email ·   │  each: timeout, never throws,
                     │   phone · person │  returns { checked:false } on fail
                     │   address · TF · │
                     │   LeadiD · UA ·  │
                     │   CRM · dedupe   │
                     ├──────────────────┤
                     │ 3. scoreLead()   │  rules engine → score, tier, notes
                     ├──────────────────┤
                     │ 4. aiReview()    │  optional, sequential, gets score
                     │ 5. re-score      │  if AI rules exist (two-pass)
                     ├──────────────────┤
                     │ 6. store + log   │
                     │ 7. forwardLead() │──► outgoing webhook (CRM/any API)
                     └──────────────────┘
```

**Design rules carried over from v2.0**
- Single process; enrichment parallelized with `Promise.all`.
- A failed or slow provider never blocks a lead — rules depending on it simply don't fire.
- Only `/webhook/lead` requires the API key; `/api/*` is internal (protect at network layer).
- Secrets are masked (`••••••`) on `GET /api/config`; masked values are stripped before merge on save.
- Partial config updates are deep-merged.

**Fix carried in from the v2.0 notes:** AI review is now true two-pass — score without AI, call AI with that score, then re-score with AI results so AI rules actually affect the final number.

---

## 4. Canonical Lead Schema

Every intake profile maps its source fields onto this shape. Anything not mapped is preserved under `custom` so survey answers from any vertical remain scoreable.

```json
{
  "id": "uuid (auto)",
  "source": { "profile": "mortgage-survey", "site": "example.com", "aff_id": "420", "sub_id": "LO" },

  "identity": { "first_name": "", "last_name": "", "full_name": "", "dob": "" },
  "contact":  { "email": "", "phone": "" },
  "address":  { "line1": "", "city": "", "state": "", "zip": "", "country": "US" },

  "tracking": {
    "ip_address": "71.197.189.233",
    "user_agent": "Mozilla/5.0 (...)",
    "trusted_form_url": "https://cert.trustedform.com/...",
    "leadid_token": "266D10A3-54E2-354A-F6F2-162F989F129E",
    "unique_id": "108151247",
    "submitted_at": "ISO-8601"
  },

  "custom": {
    "<any survey question>": "<answer>"
  }
}
```

The original payload is always returned untouched; the canonical form is stored on the result as `_canonical` for traceability.

---

## 5. Intake Profiles (Field Mapping)

A profile tells the validator how to read one source. Paths on the left are canonical; values on the right are the incoming field names (first match wins, case-insensitive).

```json
{
  "id": "default",
  "description": "Generic web form",
  "match": { "header": "x-lead-profile", "query": "profile", "field": "form_id" },
  "map": {
    "identity.first_name":       ["first_name", "First Name", "fname", "firstName"],
    "identity.last_name":        ["last_name", "Last Name", "lname", "lastName"],
    "contact.email":             ["email", "Email", "email_address"],
    "contact.phone":             ["phone", "Phone", "phone_number", "mobile"],
    "address.line1":             ["address", "Address", "street"],
    "address.city":              ["city", "City", "Property City"],
    "address.state":             ["state", "State", "Property State"],
    "address.zip":               ["zip", "Zip", "postal_code", "Property Zip"],
    "tracking.ip_address":       ["ip_address", "IP ADDRESS", "ip"],
    "tracking.user_agent":       ["user_agent", "USER AGENT"],
    "tracking.trusted_form_url": ["trusted_form_url", "TRUSTED FORM", "xxTrustedFormCertUrl"],
    "tracking.leadid_token":     ["leadid_token", "LEAD ID COM", "universal_leadid"],
    "tracking.unique_id":        ["unique_id", "UNIQUE ID"],
    "source.aff_id":             ["aff_id", "AFF ID"],
    "source.sub_id":             ["sub_id", "CATEGIORY SUBID", "category_subid"],
    "source.site":               ["site", "Site Name"]
  },
  "custom": "passthrough",
  "transforms": {
    "contact.phone": "digits",
    "contact.email": "lowercase_trim",
    "address.state": "upper_state_code"
  },
  "rules": "default"
}
```

**Profile selection order:** `?profile=` query → `x-lead-profile` header → `form_id` field in payload → `default`.

**Available transforms:** `trim`, `lowercase_trim`, `digits`, `upper_state_code` (full name → 2-letter), `title_case`, `split_full_name` (fills first/last from one field), `percent_to_number`, `currency_to_number`.

---

## 6. Rules Engine

### 6.1 Rule shape

```json
{
  "id": "email_format",
  "label": "Valid email format",
  "group": "contact",
  "when": { "field": "contact.email", "op": "regex", "value": "^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$" },
  "points": 8,
  "note": "Valid email",
  "enabled": true
}
```

- `field` is a dot-path into `{ lead: <canonical>, enrichment: <_enrichment> }` — prefix `enrichment.` to score provider results, otherwise it reads the canonical lead.
- `points` may be negative (deductions).
- `note` is appended to `notes` when the rule fires.
- `requires` (optional) — provider name; if that provider didn't run (`checked:false`), the rule is **skipped**, not failed. Skipped rules don't count against the `normalized` maximum.

### 6.2 Operators

| op | Meaning |
|---|---|
| `present` / `absent` | Non-empty / empty |
| `equals`, `not_equals` | Case-insensitive string compare |
| `contains`, `starts_with` | Case-insensitive substring |
| `in`, `not_in` | Value in array |
| `regex` | JavaScript regex |
| `gte`, `lte`, `between` | Numeric (non-digits stripped first) |
| `is_true`, `is_false` | Boolean |
| `length_eq`, `length_gte` | String length |
| `all`, `any` | Compound — `value` is an array of nested conditions |

### 6.3 Exclusive groups

Rules in the same `exclusive` group award only the highest match (e.g., credit tiers, timeframe buckets):

```json
{ "id": "credit_excellent", "exclusive": "credit", "when": { "field": "custom.credit_rating", "op": "contains", "value": "excellent" }, "points": 25 }
```

### 6.4 Score modes

| Mode | Formula | Use when |
|---|---|---|
| `capped` (default, original behavior) | `clamp(Σ points, 0, 100)` | Weights already sized to ~100 |
| `normalized` | `clamp(Σ earned / Σ max achievable positive points × 100, 0, 100)` | Profiles have very different numbers of rules |

### 6.5 Default generic rule set (`rules: "default"`)

Applies to any web lead — no vertical-specific questions needed.

| id | Condition | Pts | Requires |
|---|---|---|---|
| `name_present` | first + last name present, alphabetic, length ≥ 2 | 5 | — |
| `name_not_junk` | name not in junk list (test, asdf, xxx…) and first ≠ last | 5 | — |
| `email_format` | regex valid | 8 | — |
| `email_deliverable` | `enrichment.email.deliverable` true | 12 | email |
| `email_disposable` | `enrichment.email.disposable` true | −15 | email |
| `phone_10_digit` | 10 digits, valid NANP area code | 7 | — |
| `phone_valid` | `enrichment.phone.valid` true | 10 | phone |
| `phone_mobile` | line type `mobile` | 3 | phone |
| `phone_voip` | line type `voip` | −5 | phone |
| `ip_geo_match` | IP state matches address state (city match = bonus) | 18 | ipApi |
| `ip_proxy` | `enrichment.ipApi.proxy` or `hosting` true | −15 | ipApi |
| `identity_verified` | Melissa result GE01/GE02 | 12 | melissaPerson |
| `address_deliverable` | Melissa DPV Y or S | 10 | melissaAddress |
| `trustedform_present` | URL starts with `https://cert.trustedform.com/` | 8 | — |
| `trustedform_verified` | cert retrieved, age < 72h, page URL on allowed domain | 6 | trustedForm |
| `leadid_present` | Jornaya token is valid GUID | 5 | — |
| `leadid_verified` | token authentic, not reused | 5 | jornaya |
| `ua_real_browser` | UA parses as real browser, not bot/headless | 4 | userAgent |
| `ua_bot` | headless/bot/curl UA | −20 | userAgent |
| `duplicate_recent` | same email or phone in last 30 days | −10 | duplicate |
| `existing_customer` | CRM match | 15 | crmLookup |
| `ai_fraud_flag` | `enrichment.aiReview.fraudFlags.length > 0` | −15 | aiReview |

### 6.6 Vertical example — `mortgage-survey` profile

The original THL signals live here as `custom.*` rules, added on top of the default set (`"rules": ["default", "mortgage"]`):

| id | Field | Condition | Pts |
|---|---|---|---|
| `credit_excellent` / `good` / `average` / `fair` | `custom.credit_rating` | contains | 25 / 15 / 10 / 5 (exclusive) |
| `timeframe_now` / `90d` / `3_12m` | `custom.moving_date` | contains "right away" / "within 90" / "3-6","6-12" | 20 / 12 / 8 (exclusive) |
| `has_realtor` | `custom.realtor` | equals Yes | 15 |
| `found_home` | `custom.found_home` | equals Yes | 10 |
| `down_10_plus` / `down_any` | `custom.down_payment` | gte 10 / gte 1 | 12 / 7 (exclusive) |
| `military` | `custom.military_service` | equals Yes | 10 |

To support a new vertical, copy this pattern: map the form's questions into `custom.*`, then write rules against them. No code changes.

---

## 7. Enrichment Providers

### 7.1 Common interface

```js
module.exports = {
  name: 'phone',
  timeoutMs: 8000,
  enabled: (cfg) => cfg.integrations.phone?.enabled && !!cfg.integrations.phone.apiKey,
  async run(lead, cfg, signal) {
    // must never throw; return { checked:false } on any failure
    return { checked: true, valid: true, carrier: '...', lineType: 'mobile' };
  },
  async test(cfg) { return { ok: true, message: 'Connected' }; }
};
```

`pipeline.js` loads every file in `lib/providers/`, runs the enabled ones in `Promise.all` with `AbortSignal.timeout(timeoutMs)`, and places each result at `_enrichment.<name>`.

### 7.2 Provider catalog

| Provider | Field(s) used | Returns | Timeout | Key needed |
|---|---|---|---|---|
| `ipApi` (ip-api.com) | ip, city, state | `{ match, cityMatch, stateMatch, geo, proxy, hosting, mobile, isp }` | 5s | No (free tier, HTTP, 45 req/min) |
| `email` (Hunter.io) | email | `{ deliverable, status, score, disposable, webmail }` | 8s | Yes |
| `phone` (Numverify) | phone | `{ valid, carrier, lineType, location }` | 8s | Yes |
| `melissaPerson` (Personator) | name, email, phone, address | `{ isValid, resultCodes, nameFull, addressLine1, ... }` | 8s | Yes |
| `melissaAddress` (Global Address) | address | `{ isDeliverable, dpv, resultCode, deliveryLine, ... }` | 8s | Yes |
| `trustedForm` | trusted_form_url | `{ present, verified, ageHours, pageUrl, ip, ... }` | 8s | Account for full verify |
| `jornaya` (LeadiD) | leadid_token | `{ present, authentic, reused, ageSeconds }` | 8s | Account |
| `userAgent` (local) | user_agent | `{ browser, os, device, isBot, isHeadless, isWebView }` | — | No |
| `duplicate` (local) | email, phone, unique_id | `{ isDuplicate, matchedId, daysAgo }` | — | No |
| `crmLookup` (Snowflake SQL REST or any) | email, phone | `{ isExistingCustomer, customerId, previousCount, lastActivity }` | 10s | Yes |
| `aiReview` (Azure OpenAI / chat-completions) | full lead + enrichment + score | `{ qualitySummary, fraudFlags[], nextBestAction, confidenceReason, riskLevel }` | 30s | Yes |

Adding a provider = drop a file in `lib/providers/`, add a config block, write rules against `enrichment.<name>.*`.

---

## 8. Validated Lead Response

The original payload is returned with these fields appended:

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Generated if absent |
| `validation_score` | int 0–100 | Final score |
| `confidence_color` | string | `GREEN` \| `BLUE` \| `AMBER` \| `RED` |
| `notes` | string | Fired-rule notes joined with ` · ` |
| `validated_at` | ISO-8601 | UTC |
| `profile` | string | Intake profile used |
| `_canonical` | object | Normalized lead (section 4) |
| `_enrichment` | object | Every provider result keyed by provider name |
| `_rule_hits` | array | `[{ id, points, fired, skipped }]` — full audit trail of the score |

```json
{
  "First Name": "Jane", "Email": "jane@example.com", "...": "...",
  "id": "8c1f...",
  "validation_score": 82,
  "confidence_color": "GREEN",
  "notes": "Valid email · Email deliverable · IP matches state · TrustedForm cert · Real browser",
  "validated_at": "2026-10-12T14:22:00.000Z",
  "profile": "default",
  "_rule_hits": [
    { "id": "email_format", "points": 8, "fired": true },
    { "id": "phone_valid", "points": 10, "fired": false, "skipped": true, "reason": "phone provider disabled" }
  ],
  "_enrichment": { "ipApi": { "checked": true, "match": true, "geo": "Seattle, WA" } }
}
```

---

## 9. Configuration — `data/config.json`

```json
{
  "apiKey": "uuid",
  "scoreMode": "capped",
  "tiers": [ "...see section 1..." ],
  "outgoing": { "enabled": false, "url": "", "apiKey": "", "onlyTiers": ["GREEN","BLUE"], "retryOnFail": true },
  "integrations": {
    "ipApi":          { "enabled": true },
    "email":          { "enabled": false, "apiKey": "" },
    "phone":          { "enabled": false, "apiKey": "" },
    "melissa":        { "enabled": false, "licenseKey": "", "personator": true, "addressVerify": true },
    "trustedForm":    { "enabled": true, "apiKey": "", "allowedDomains": [] },
    "jornaya":        { "enabled": false, "apiKey": "" },
    "userAgent":      { "enabled": true },
    "duplicate":      { "enabled": true, "windowDays": 30 },
    "crmLookup":      { "enabled": false, "account": "", "user": "", "password": "", "warehouse": "", "database": "", "schema": "", "table": "" },
    "aiReview":       { "enabled": false, "endpoint": "", "deploymentName": "", "apiVersion": "", "apiKey": "" }
  },
  "ruleSets": {
    "default":  [ "...section 6.5..." ],
    "mortgage": [ "...section 6.6..." ]
  }
}
```

| Env var | Default | Purpose |
|---|---|---|
| `PORT` | 3000 | Listen port |
| `DATA_DIR` | `./data` | Storage location |
| `LV_SECRETS_FROM_ENV` | false | Read API keys from env vars (`LV_HUNTER_KEY`, etc.) instead of config.json |

---

## 10. API Reference

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/webhook/lead` | `x-api-key` | Validate one lead from any source; store, log, forward |
| POST | `/api/leads/validate` | — | Batch validate (array or object); used by CSV upload |
| GET | `/api/leads` | — | All stored leads (`?tier=`, `?profile=`, `?q=` filters) |
| GET | `/api/leads/:id` | — | One lead |
| DELETE | `/api/leads/:id` · `/api/leads` | — | Delete one / all |
| GET · POST | `/api/config` | — | Read (masked) / partial update (deep-merge) |
| POST | `/api/config/regenerate-key` | — | New webhook key |
| GET · POST · DELETE | `/api/profiles[/:id]` | — | Manage intake profiles |
| POST | `/api/profiles/:id/preview` | — | Show canonical mapping for a sample payload (no scoring) |
| GET · POST | `/api/rules[/:set]` | — | Read / replace a rule set |
| POST | `/api/integrations/test/:name` | — | Live connection test for one provider |
| POST | `/api/outgoing/test` | — | Send a dummy GREEN lead to the outgoing URL |
| GET · DELETE | `/api/logs` | — | Event log (`?type=incoming|outgoing|test`) |
| POST | `/api/tests/run` | — | Run validation test fixtures (section 11) |
| GET | `/api/tests/runs` | — | Past test-run results |

**Webhook example**
```bash
curl -X POST "http://localhost:3000/webhook/lead?profile=default" \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -d '{"first_name":"Jane","last_name":"Smith","email":"jane@example.com",
       "phone":"206-555-0142","city":"Seattle","state":"WA","zip":"98101",
       "ip_address":"71.197.189.233",
       "trusted_form_url":"https://cert.trustedform.com/e7aacca6...",
       "leadid_token":"266D10A3-54E2-354A-F6F2-162F989F129E",
       "user_agent":"Mozilla/5.0 (Linux; Android 15; ...) Chrome/142.0 Mobile Safari/537.36"}'
```

Errors: `401 { "error": "Invalid API key" }`, `422 { "error": "Unknown profile" }`, `500 { "error": "<message>" }`.

---

## 11. Validation Test Harness

Lets you prove that a profile + rule set classifies leads the way you expect, before pointing live traffic at it — and re-prove it after every weight change.

### 11.1 Fixture format — `tests/fixtures/<name>.json`

```json
{
  "profile": "default",
  "offline": true,
  "cases": [
    {
      "name": "Clean consumer lead",
      "lead": { "first_name": "Jane", "last_name": "Smith", "email": "jane.smith@gmail.com",
                "phone": "2065550142", "state": "WA", "ip_address": "71.197.189.233",
                "trusted_form_url": "https://cert.trustedform.com/abc",
                "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/142.0" },
      "mockEnrichment": { "ipApi": { "checked": true, "match": true, "stateMatch": true } },
      "expect": { "tier": "GREEN" }
    },
    {
      "name": "Junk name + bot UA",
      "lead": { "first_name": "test", "last_name": "test", "email": "asdf@mailinator.com",
                "phone": "1111111111", "user_agent": "HeadlessChrome/120.0" },
      "expect": { "tier": "RED", "rulesFired": ["ua_bot"], "rulesNotFired": ["name_not_junk"] }
    },
    {
      "name": "Real person, out-of-state IP",
      "lead": { "first_name": "Ana", "last_name": "Lopez", "email": "ana@example.com",
                "phone": "3055550199", "state": "FL", "ip_address": "71.197.189.233" },
      "mockEnrichment": { "ipApi": { "checked": true, "match": false, "geo": "Seattle, WA" } },
      "expect": { "scoreMax": 49 }
    }
  ]
}
```

**Expectations available:** `tier`, `tierIn`, `scoreMin`, `scoreMax`, `rulesFired`, `rulesNotFired`, `notesContain`.

**`offline: true`** — providers are not called; `mockEnrichment` is used instead. Fast, free, deterministic — use in CI. Set `offline: false` to run against live provider APIs (costs credits).

### 11.2 Running

- **Dashboard** → Tests tab → pick fixture(s) → Run.
- **API** → `POST /api/tests/run` with `{ "fixtures": ["default-basics"], "offline": true }`.
- **CLI / CI** → `npm test` (exit code 1 on any failure).

### 11.3 Result

```json
{
  "runId": "uuid", "ranAt": "ISO-8601", "passed": 2, "failed": 1,
  "results": [
    { "name": "Real person, out-of-state IP", "pass": false,
      "expected": { "scoreMax": 49 }, "actual": { "score": 55, "tier": "BLUE" },
      "rule_hits": [ "..." ] }
  ],
  "tierDistribution": { "GREEN": 1, "BLUE": 1, "AMBER": 0, "RED": 1 }
}
```

### 11.4 Replay mode

`POST /api/tests/run { "replay": { "lastN": 500 } }` re-scores stored leads with the **current** rules (using their stored `_enrichment`, no new API calls) and reports how many would change tier. Use this before saving new weights.

---

## 12. Dashboard — `public/index.html`

Single file, Chart.js from CDN, no framework.

| Tab | Contents |
|---|---|
| Dashboard | Metric cards (Green / Blue / Amber / Red counts + avg score), donut chart, tier filter, search, sortable table, CSV drop zone, CSV export |
| Lead drill-down | AI analysis card · contact · address · tracking (IP, UA, TrustedForm, LeadiD, IDs) · `custom` answers · enrichment results per provider · **rule-hit breakdown** (each rule, points, fired/skipped) · export / delete |
| Profiles | List profiles, edit field mappings, paste a sample payload → live canonical preview |
| Rules | Edit rule sets in a table (field, op, value, points, note, enabled), add/remove rules, edit tier thresholds and score mode, **replay impact** before save |
| Tests | Fixture list, run, pass/fail table, tier distribution, history |
| Integrations | One card per provider: toggle, credentials, Save, Test |
| API Logs | Webhook URL, cURL example, event stream (incoming / outgoing / test) |
| Settings | Webhook API key (copy / regenerate), outgoing webhook, data management |

Panel is 340px on desktop and becomes a bottom sheet under 1100px wide.

---

## 13. Storage

| File | Cap | Contents |
|---|---|---|
| `config.json` | — | Settings, keys, rule sets |
| `leads.json` | 2,000 | Validated leads (newest first) |
| `logs.json` | 500 | Event log |
| `test-runs.json` | 50 | Test harness results |
| `profiles/*.json` | — | Intake profiles |

Writes go to a temp file then `renameSync` (atomic). Swap `lib/storage.js` for a Postgres/SQLite/Snowflake adapter for >500 leads/day; nothing else changes.

---

## 14. Security

| Concern | Implementation | Production recommendation |
|---|---|---|
| Webhook auth | UUID `x-api-key` | Add source IP allowlist |
| Internal API | Open | Reverse-proxy auth or VPN |
| Secrets | config.json, masked on read | `LV_SECRETS_FROM_ENV=true` or a secrets manager |
| Input | Profile mapping + transforms; unknown fields kept in `custom` | Add per-profile required-field check (422) |
| CRM SQL | Parameterized binds (fixed from v2.0 string interpolation) | Key-pair JWT auth |
| PII | Stored locally | Retention policy; purge job |
| CORS | Configurable origin list | Restrict to dashboard origin |
| TLS | Behind nginx | Terminate TLS at proxy |

---

## 15. Setup

```bash
# Node.js 18+
npm install
npm start          # http://localhost:3000
npm run dev        # node --watch
npm test           # run offline validation fixtures
```

First run creates `data/`, a fresh API key (printed to terminal), the `default` profile and rule set.

**Onboarding a new website form**
1. Profiles tab → New → paste one real submission → map fields until the preview looks right.
2. Rules tab → start from `default`; add `custom.*` rules for the form's own questions.
3. Tests tab → create fixtures: one known-good, one known-junk, one borderline per tier.
4. Run tests until they pass; run replay against any existing leads.
5. Point the form/Zapier at `/webhook/lead?profile=<id>` with the API key.
6. Optionally enable the outgoing webhook to push scored leads to your CRM.

---

## 16. Extending

- **New provider:** add `lib/providers/<name>.js` (section 7.1 interface) → config block → rules against `enrichment.<name>.*` → integration card appears automatically.
- **New operator:** add a function to the `OPERATORS` map in `lib/rules.js`.
- **New vertical:** new profile + rule set + fixtures. No code.
- **High volume:** chunked `Promise.allSettled` batches for CSV; queue (Service Bus / RabbitMQ) between webhook and pipeline above ~5,000/day.

---

## 17. Reference Links

| Resource | URL |
|---|---|
| ip-api | https://ip-api.com/docs |
| Hunter.io email verifier | https://hunter.io/api-documentation |
| Numverify | https://numverify.com/documentation |
| Melissa Personator | https://www.melissa.com/developer/personator-consumer |
| Melissa Global Address | https://www.melissa.com/developer/global-address |
| Melissa result codes | https://wiki.melissadata.com/index.php?title=Result_Code_Details |
| TrustedForm | https://developers.activeprospect.com/docs/trustedform/ |
| Jornaya (LeadiD) | https://www.verisk.com/marketing-solutions/ |
| Snowflake SQL REST API | https://docs.snowflake.com/en/developer-guide/sql-api/guide |
| Azure OpenAI REST | https://learn.microsoft.com/azure/ai-services/openai/reference |
