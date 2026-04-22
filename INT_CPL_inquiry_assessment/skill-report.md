---
name: zoral-query-review-report
description: Audit Zoral Decision Engine workflow XML to find SELECTed columns that are never read by downstream ScriptTask/Gateway, plus AdwQuery patterns that over-fetch (full-row fetch used only as `.length`, leading-wildcard `NOT LIKE`, JSON `property` blob over-pulling, cartesian IN-list expansion). Produce a Markdown review report with per-query column tables, ranked performance risks, and a prioritized patch plan to relieve memory pressure.
---

# Zoral Query Review Report

Use this skill when the user wants to:

- review a Zoral workflow XML for `AdwQuery` over-fetch and unused passthrough columns
- diagnose memory-full / OOM symptoms in a Zoral flow
- produce a Thai-language Markdown review report (`report.md` style) with per-query column usage tables and a ranked patch plan
- decide which columns are safe to remove from a `SELECT` and which need caller verification first

## Expected Inputs

Work from the workflow XML pair in a folder:

- `workflow_name.xml`
- `workflow_name.xml.layout` (optional — use only if order matters for the report)

Optionally also a generated `plant_uml.puml` from the `zoral-xml-to-plantuml` skill — useful as a flow reference.

## Default Deliverable

Create one file inside the workflow folder:

- `report.md`

The report is review-only — it does not modify the XML.

## Authoring Workflow

1. Inventory every `ComponentTask` whose `ComponentName = AdwQuery`.
2. For each AdwQuery, extract:
   - the GraphQL alias (e.g. `listsOtherType`)
   - the table name
   - the `where` clause
   - the full SELECT column list
3. For each SELECTed column, grep the XML for downstream reads of the form `steps.<TaskName>.data.<alias>...<column>` (and `.property.<key>` when the column is a JSON blob). A column referenced **only** inside the AdwQuery itself (e.g. inside its own `where`) does not count as "used in flow".
4. Classify each column as:
   - ✅ used in flow (read by ScriptTask / Gateway / another AdwQuery as input)
   - ❌ passthrough only (forwarded to caller but never read internally)
5. Scan AdwQuery results for the over-fetch anti-patterns in §"Performance Risk Heuristics".
6. Score each risk High / Medium / Low and write the report.

## Markdown Structure

Use this section order. If a section does not apply, say so explicitly.

1. Header block — `Source`, `Purpose`, `Date` (today's date in `YYYY-MM-DD`)
2. `1. Summary` — counts table + TL;DR bullets
3. `2. Per-query Column Usage` — one subsection per AdwQuery sub-query, with a column table
4. `3. Performance / Memory Risks` — ranked High → Low, each with Location, Problem, Recommendation
5. `4. Recommended Patches` — numbered, each with Effort, Impact, Pre-req
6. `5. Monitoring suggestions`
7. `6. Validation ก่อนลบ column`
8. `7. Quick wins สรุป` — priority table P0 → P3 with expected memory reduction

## Column Table Format

Use one of two layouts depending on column count:

**Few columns (≤ 10) — full row table:**

| Column | ใช้ใน flow? | Action |
|--------|-----|--------|
| `column_name` | ❌ | ตรวจสอบ caller → อาจลบ |
| `other_col` | ✅ | keep |

**Many columns (> 10) — split keep / passthrough:**

ใช้ใน flow : `col_a`, `col_b` (Item #N เอาไปเป็น input)

Passthrough : `col_x`, `col_y`, `col_z`

**N/M passthrough**

Always state the ratio (e.g. **9/11 ไม่ใช้ใน flow**) so the reader sees the impact at a glance.

## Performance Risk Heuristics

Flag every match against the patterns below. Each goes into §3 with a colored severity tag.

### 🔴 HIGH — `.length`-only full-row fetch
A query whose result is consumed only by `?.length > 0` / `.length === 0` etc. Pulling every row + every selected column to answer "is there any?" is wasted bandwidth.

**Recommend :** convert to `_aggregate { count }` or `limit: 1` selecting only a primary key.

### 🔴 HIGH — leading-wildcard `_nlike` / `_like`
Patterns like `_nlike: "%CAP%"` or `_like: "%foo%"` cannot use a btree index → full scan.

**Recommend :** if the domain values are enumerable, switch to `_nin: [...]` / `_in: [...]`. Otherwise add a functional/trigram index, or precompute the predicate column.

### 🟠 MEDIUM — many sub-queries in one ComponentTask
A single AdwQuery firing 4+ sub-queries against large tables creates a memory hotspot during result merge.

**Recommend :** split into 2–3 ComponentTask steps so each peak is smaller. Quantify (`5 tables × ~N rows × M columns`).

### 🟠 MEDIUM — over-broad WHERE filtered later in script
Query selects a wider set than needed, then a downstream ScriptTask filters most rows away. Look for `IN $list` clauses where `$list` is much wider than what is actually consumed.

**Recommend :** narrow the WHERE if the narrowing values are known at query time, or reorder the flow so the narrowing step runs first.

### 🟡 LOW — cartesian potential on multi-IN WHERE
`form_code IN [...] AND section_code IN [...]` with both lists large → cartesian expansion.

**Recommend :** narrow one list, or paginate with `limit` + cursor.

### 🟡 LOW — JSON `property` blob always pulled
Tables expose a `property` jsonb column. If `grep` finds no `<alias>.property.*` reads, the entire blob is wasted memory.

**Recommend :** drop `property` from the SELECT for tables where script never reads it. Keep it for tables that do (list both lists in the report).

## Patch Plan Conventions

Number each patch and record three fields:

- **Effort** — wall-clock estimate (`30 นาที`, `1–2 ชั่วโมง`, `ครึ่งวัน`, `1 วัน`)
- **Impact** — what gets reduced (`payload ต่อ row ~20–40%`, `peak memory ของ Item #N`, `DB CPU`)
- **Pre-req** — what must be verified first (`ตรวจกับ frontend ว่า column ไหนใช้`, `regression test`, `feature flag`)

Order patches by ratio of impact to risk, not by complexity. The standard ordering is:

1. Aggregate-rewrite for `.length`-only queries (zero risk, immediate win)
2. Drop `property` blob from non-reader tables
3. Drop passthrough columns after caller verification
4. Replace leading-wildcard predicates
5. Split heavy multi-sub-query ComponentTask
6. Reorder flow to enable narrower WHERE
7. Pagination / cursor-based chunked fetching

## Quick Wins Table

Always end the report with a P0 → P3 table that estimates memory reduction:

| Priority | Patch | Expected memory reduction |
|----------|-------|---------------------------|
| P0 | Aggregate rewrite | ~5–15% |
| P0 | Drop unused `property` | **~20–40%** |
| P1 | Drop passthrough columns | ~10–20% |
| P2 | Predicate rewrite | DB CPU only |
| P3 | Split / reorder / paginate | peak memory hotspot |

State the combined P0 + P1 estimate in prose so the user has a single number to act on.

## Validation Rules Before Deletion

The report MUST tell the reader to verify these before removing any column:

1. `grep -r "<column_name>" <caller-frontend-source>` — caller may be reading the field
2. Inspect Zoral metrics on recent workflow responses — confirm what the caller actually consumes
3. Stage with a feature flag, remove one table at a time, then roll out
4. When in doubt, fall back to the safer Patch (drop only `property` blobs from non-reader tables)

## Style Alignment

Match `INT_CPL_inquiry_assessment/report.md` as the canonical example:

- Thai prose for explanations and recommendations; English for code, identifiers, and table names
- Severity emojis (🔴🟠🟡) on risk subsection headings
- `#`/`##`/`###` heading hierarchy as in the structure above
- Code fences for GraphQL fragments and recommended rewrites — never paste full sub-query bodies, only the relevant lines
- Cite line numbers from the source XML (`Item #N (line A–B)`) so the reader can jump straight to the source
- Always include both the anti-pattern snippet and the recommended rewrite for HIGH risks

## Quality Checklist

Before finishing, check:

- every `AdwQuery` ComponentTask appears in §2 with a column table
- every column is classified ✅ or ❌ — no "unknown" leftovers
- every HIGH/MEDIUM risk in §3 has Location, Problem, Recommendation
- every patch in §4 has Effort + Impact + Pre-req
- the date in the header is today's date (`YYYY-MM-DD`)
- the report does not delete or modify the source XML
- caller-verification reminder appears in §6 before any "remove column" recommendation
