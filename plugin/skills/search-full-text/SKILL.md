---
name: search-full-text
model: claude-sonnet-4-6
description: Executes full-text searches against FamilySearch AI-transcribed
  historical document images per the research plan. Uses the fulltext_search
  MCP tool with Lucene-style operators (+/-/"…"/?/*). Uniquely surfaces
  witnesses, neighbors, heirs, sureties, appraisers, and other non-principal
  mentions that indexed search misses. Logs every search including nil results
  and passes promising records to record-extraction. Use when the user says
  "full-text search", "search for witnesses mentioning [person]", "search
  newspapers for [person]", "find [person] in deeds/probate/court minutes",
  when a plan item targets FamilySearch full-text search, when looking for FAN
  club mentions, when searching pre-1850 US records, or Latin American notarial
  records. Do NOT use for structured indexed search (use search-records),
  Ancestry/MyHeritage/FindMyPast/FindAGrave/Newspapers.com (use
  search-external-sites), planning searches (use research-plan), or analyzing
  found records (use record-extraction).
allowed-tools:
  - fulltext_search
---

# Search Full-Text

**Narration:** Read `researcher_profile.narration_guidance` from `research.json` and apply it as your narration style. If absent, default to a one-line preamble per action.

`fulltext_search` searches raw AI transcript text of ~1.95 billion document images — not structured indexes. Finds people mentioned anywhere (witnesses, heirs, appraisers), not just principals. No fuzzy matching, no abbreviation expansion; default operator is OR.

| | Indexed (`record_search`) | Full-text (`fulltext_search`) |
|---|---|---|
| Source | Structured fields | Raw transcript text |
| Fuzzy/nicknames | Auto-applied | None — exact match only |
| Abbreviations | Wm→William auto | Must search separately |
| Default | Fuzzy on all terms | OR — require `+` explicitly |
| Unique strength | Principal lookup | Non-principal mentions |

Results are derivative (original → image → AI transcript; ~10% error rate). Always verify against the original image.

## Steps

### 1. Identify the plan item

Read `plans[]` in `research.json` for the next item with `status: "planned"` targeting full-text search. Ad-hoc searches use `plan_item_id: null` in the log.

### 2. Evaluate coverage

FTS covers ~6,665 collections as of mid-2026 — not all FamilySearch collections are searchable. Read collection descriptions; titles mislead about scope. Read `references/online-search-literacy.md` for the full evaluation checklist.

### 3. Choose search philosophy

**Less is more.** Every extra required term risks missing transcription variants.
- Uncommon surname → `+Surname` only, filter after
- Common surname → `+Surname +Associate` or `+Surname +Keyword`
- Very common surname → multiple required terms or phrase search

### 4. Determine strategy

Read `references/search-strategies.md` for the full catalog. Quick reference:

| Research goal | Query approach |
|---|---|
| Witness/appraiser/heir | `+Surname` in Name field |
| Narrative records (deeds, probate) | `+GivenName +Surname` in Keywords |
| FAN cluster | `+TargetSurname +AssociateSurname` |
| Kinship determination | `+Surname +"daughter of"` |
| Enslaved persons | Enslaver surname + slavery keywords (see reference) |

### 5. Construct the query

Read `references/query-syntax.md` for full operator rules. Critical rules:

- **Always `+` to require terms.** OR default returns millions of results.
- **Name first, place after.** Place in query hits collection metadata → false positives.
- **Search abbreviations explicitly.** William → also Wm; Thomas → also Thos.
- **Mine `research.json` for known spelling variants** before querying — FTS won't auto-expand what prior records already show.
- Phrases tolerate one intervening word: `"Ezekiel Pearce"` matches "Ezekiel John Pearce."
- Wildcards `?`/`*`: min 3 literal chars, cannot open a word or appear inside quotes.

```
fulltext_search({ keywords: "+Patrick +Flynn" })           # require both
fulltext_search({ keywords: '+"Patrick Flynn"' })          # phrase
fulltext_search({ keywords: "+Flynn +Brennan" })           # FAN cluster
fulltext_search({ keywords: "+Fl?nn +Patrick" })           # wildcard for HTR
fulltext_search({ dgsNumber: "4057677", keywords: "+Flynn" }) # single volume
fulltext_search({ nlQuery: "KD96-TV2" })                   # tree person ID only
```

Use `nlQuery` only for natural-language requests or tree person IDs — not as a fallback when `keywords` returns few results.

### 6. Execute and iterate

**Decision by hit count:**
- 0 → Step 10
- 1–50 → review all
- 50–500 → add Year/RecordType filter
- >500 → add second required term or place filter

Read `references/transcription-quirks.md` for HTR error patterns and coverage gaps.

### 7. Triage results

For each result: Does the target name appear in textDocument? Is the context right (witness, will clause, deed party) or a false positive (cross-column, place name)? Is place/date consistent? List results with match quality and role. Let the user confirm which to examine in detail.

### 8. Retain results and write the log entry

**Every search gets a log entry — no exceptions.** Follow `references/research-log-protocol.md`.

**a. Result sidecar** (`results/<log_id>.json`) — write verbatim response for non-nil results:
```json
{
  "log_id": "log_008",
  "tool": "fulltext_search",
  "retrieved": "2026-05-04T16:00:00Z",
  "returned_count": 5,
  "payload": { "...": "verbatim fulltext_search response" }
}
```
`returned_count` must equal payload count. Write in ~40-result chunks for large payloads. Zero-result searches write no sidecar.

**b. Log entry** in `research.json`:
```json
{
  "id": "log_008",
  "plan_item_id": "pli_010",
  "performed": "2026-05-04T16:00:00Z",
  "tool": "fulltext_search",
  "query": { "keywords": "+Flynn +\"Last Will and Testament\"", "recordPlace1": "Pennsylvania", "recordPlace2": "Schuylkill", "yearFrom": 1870, "yearTo": 1890 },
  "outcome": "positive",
  "results_examined": 5,
  "results_available": 47,
  "results_ref": "results/log_008.json",
  "notes": "47 Schuylkill will hits 1870–1890; 5 examined. Thomas Flynn's will (1881) names wife Mary and children Patrick, John, Margaret.",
  "external_site": null
}
```

**c.** If `returned_count` mismatches after one retry, set `results_ref: null`, note the failure, and tell the user plainly.

### 9. Update plan item status

- `completed` — search executed (regardless of outcome)
- `skipped` — unnecessary (question already answered by a prior search)

### 10. Handle nil results

1. Log with `outcome: "negative"`.
2. Iterate variants — read `references/search-strategies.md` and `references/online-search-literacy.md`. Log each retry separately.
3. Verify coverage exists for the locality/period before treating absence as negative evidence.
4. Check for fallback plan items; suggest search-records or re-plan.
5. **Do NOT run diagnostic queries** (`+Smith`, `+Jones`) to verify FTS is working. The tool's response is authoritative; diagnostic probes inflate cost and rationalize genuine negative findings away.

### 11. Queue cross-reference searches

Suggest sub-searches for named non-target persons (witnesses, executors, heirs, neighbors), distinctive landmarks, slaveholder ↔ enslaved pairs, powers of attorney, and marginal annotations.

### 12. Pass to extraction

For each promising record, invoke record-extraction and pass the transcript text from the FTS result as context.

### 13. Present results

Summarize what was searched and found. Highlight non-principal mentions (FTS's unique value). Show log entries and plan progress. Suggest next steps: continue plan items, pursue cross-reference sub-searches, or (if no results) try search-records or re-plan.

## Important rules

- **Always `+` to require terms.** OR returns millions of irrelevant results.
- **Name first, place after.** Place in query → metadata false positives.
- **Verify against the original image.** FTS results are derivative (~10% error).
- **A nil result does not prove absence.** Try variants; log exact parameters.
- **Log every search**, including nil results. The log is the GPS audit trail.
- **Let the user confirm before extraction.** Never fabricate results.
- **Write only to `log` and `plans`.** Source entries and assertions belong to record-extraction.
- **No extra fields on plan items.** Schema enforces `additionalProperties: false`.
