---
name: zoral-xml-to-plantuml
description: Generate or update PlantUML-based `requirement*.md` and matching `.puml` files from Zoral Decision Engine XML/XML.layout workflows, including sequence diagrams, groupings for zoral/internal/external/database, detailed script notes, activate/deactivate timeline markers, and per-table ADW grouping when queries are present.
---

# Zoral XML To PlantUML

Use this skill when the user wants to:

- generate PlantUML from Zoral Decision Engine `.xml` and `.xml.layout`
- create or update `requirement*.md`
- create matching `.puml` files from markdown PlantUML blocks
- summarize workflow logic, gateway branches, script logic, and ADW table usage
- keep output in the style already used in this workspace

## Expected Inputs

Work from a matched pair:

- `workflow_name.xml`
- `workflow_name.xml.layout`

The basename must match. Use the XML as the source of step definitions and scripts. Use the layout file as the source of true flow order and branch wiring.

## Default Deliverables

Unless the user asks otherwise, create:

- `requirement01.md`
- `requirement01.puml`

Inside the workflow folder.

If the user asks for richer variants, you may also create:

- `requirement02.md` for more detailed script notes
- `requirement03.md` for per-table ADW grouping

The `.puml` should match the PlantUML block inside the markdown file exactly.

## Authoring Workflow

1. Inspect the XML and layout pair.
2. Inventory the workflow items:
   - `MessageStartEvent`
   - `EndEvent`
   - `ScriptTask`
   - `Gateway`
   - `SubProcessTask` / `SubProcessStartTask`
   - `SubDecisionTask`
   - `ComponentTask`
   - `GlobalErrorEvent`
3. Extract subworkflow mappings from `<SubWorkflow ... WorkflowName="...">`.
4. Extract database usage from `ComponentTask` scripts when `ComponentName = AdwQuery`.
5. Use `.xml.layout` to reconstruct the actual sequence and branch structure.
6. Summarize only the script logic that materially changes flow, payloads, scores, mappings, filtering, dedupe, mutation payloads, or result shaping.
7. Write the markdown file.
8. Copy the PlantUML block into the matching `.puml` file without modification.

## Markdown Structure

Use this section order unless the user requests a different format:

1. `Requirement Notes`
2. `Source Files`
3. `XML To PlantUML Mapping Rule`
4. `Table Grouping Rule`
5. `Lookup Summary From XML And XML.layout`
6. `Important Script Summary`
7. `High-Level Flow From XML.layout`
8. `Gateway Summary`
9. `Error / Status Pattern`
10. `Example PlantUML`

If a section does not apply, say so explicitly instead of omitting it.

## Mapping Rules

- Map `SubProcessTask` or `SubProcessStartTask` to workflow calls in the sequence diagram.
- If `WorkflowName` starts with `INT_`, put it in `zoral_internal`.
- If `WorkflowName` starts with `EXT_`, put it in `zoral_external`.
- Map `SubDecisionTask` such as `StatusCodeDecision` to a control or lookup step.
- Map `ComponentTask` with `ComponentName = AdwQuery` to database calls.
- Map `ComponentTask` like `Lookup` or `GlobalError` to control/error handling steps in zoral.
- Convert `Gateway` into `alt` branches using the real conditions from XML.
- Use `.xml.layout`, not step names alone, to decide actual order.

## Diagram Style Rules

Generate a sequence diagram by default.

Always include:

- `actor Requestor`
- `box "Zoral"` with a boundary participant
- `box "Zoral Internal"` when any `INT_` workflow is called
- `box "Zoral External"` when any `EXT_` workflow is called
- database boxes when direct ADW/query tables are used

Arrow conventions:

- `->` for requests, calls, and self-steps
- `-->` for responses and returns

Timeline conventions:

- Always add `activate` and `deactivate` around internal/external/database calls
- Always add notes near `activate`/`deactivate` blocks describing input and output
- Self-steps may stay as `zoral -> zoral` with a detailed note

Formatting conventions:

- `hide footbox`
- use the same visual style already used in this workspace
- keep the diagram readable rather than exhaustive

## Color Palette Rules

Use **only blue tones**. Do not use yellow, cream, green, red, coral,
pink, or gray as fills. Black is allowed for borders only.

**Group blocks** (every `group ... end` in the sequence) MUST use the blue
tone `#D4EEFB` as the background. Apply this uniformly regardless of whether
the group wraps a `ComponentTask`, `ScriptTask`, `SubDecisionTask`, gateway
body, or subworkflow call. No other color is allowed on `group`.

Example:

```plantuml
group #D4EEFB ComponentTask AdwQueryMapping
  ...
end

group #D4EEFB ScriptTask setMappingFilters
  ...
end

group #D4EEFB call StatusCodeDecision SubWorkflow
  ...
end
```

Approved blue palette (use these and nothing else):

| Hex       | Purpose                                                        |
| --------- | -------------------------------------------------------------- |
| `#E1FAFD` | Zoral box                                                      |
| `#A6DEEE` | DB boxes / SubDecision box                                     |
| `#80BBEC` | zoral boundary / SubDecision participant                       |
| `#67C7E2` | database participants                                          |
| `#A9DCDF` | ActorBackground / ParticipantBackground                        |
| `#D4EEFB` | **group background** (mandatory) + `alt` true branch header    |
| `#D6EAF8` | `alt` else branch header                                       |
| `#EBF5FB` | info note background                                           |
| `#AED6F1` | warning / error note background                                |
| `#FBFDFF` | GroupBackgroundColor in `skinparam sequence` (off-white blue)  |
| `#2874A6` | LifeLineBorderColor / GroupBorderColor                         |

Forbidden (do not introduce these hex codes):

- any cream / yellow such as `#F9E79F`, `#F7DC6F`, `#FFF2B8`, `#FCF3CF`, `#FFFDEB`
- any green, red, coral, pink, gray

When updating an existing file:

1. Sweep all `group` lines and normalize them to `group #D4EEFB ...`.
2. Grep for any hex outside the approved blue palette and replace it.
3. Re-grep after the edit to confirm zero non-blue hex codes remain.

## Detailed Note Rules

Detailed notes are expected. Do not leave notes vague.

For workflow calls, include:

- key input fields
- important header mapping
- important output fields
- special routing rules

For script steps, summarize:

- what data is read
- what filtering or branching happens
- what fields are merged, deduped, transformed, or calculated
- what globals are updated
- special-case rules
- what shape the output takes

Good note topics include:

- formula evaluation
- multi-form routing
- dedupe by ID
- grouping logic
- mutation payload construction
- score aggregation
- status code resolution
- date formatting
- branch conditions

Avoid copying raw code unless a tiny fragment is necessary.

## ADW Table Grouping Rules

When direct query tables are present, split them by table instead of using a single generic `AdwQuery` participant.

Group by prefix:

- `disbursement_*` -> `DBM`
- `master_*` -> `Master`
- `lps_corporate_loan_*` -> `Corporate Loan`
- `lps_collateral_*` -> `Collateral`
- `lps_document_service_*` -> `Document service`
- `lps_retail_loan_*` -> `Retail Loan`
- `cif_*` -> `CIF`

If a workflow has no direct ADW query in that layer, say so explicitly in `Table Grouping Rule` and `Table Inventory From XML`.

## Script Summary Heuristics

Prioritize scripts that:

- shape final response data
- build mutation/query payloads
- merge answer or schema structures
- calculate CRR/CSR/score/result grade
- filter records
- dedupe by identifiers
- select between corp/retail/DBM branches
- enrich result metadata such as dates or status

Do not over-document tiny pass-through steps.

## High-Level Flow Heuristics

Write the flow as numbered steps from the layout, for example:

1. `StartEvent(1)` -> `SomeTask(3)` -> `SomeGateway(4)`
2. If a gateway branches, describe both paths in prose.
3. Mention where branches rejoin.
4. Mention final status/output path.
5. Mention global error path if present.

## Error And Status Rules

If the workflow contains:

- hard-stop error scripts, list them and summarize their returned `status.code`
- a lookup/status decision step, mention how `statusCode` is passed into it
- a global error path, describe the exact chain such as `GlobalErrorEvent -> GlobalError -> ErrorResponse -> End`

If no explicit error flow exists, say that clearly.

## Output Rules

The markdown should be reusable as a template for later workflows.

The `.puml` file must:

- be extracted from the markdown PlantUML block
- stay in sync with the markdown
- use the exact same content as the code block

## Suggested Skeleton

```md
# WORKFLOW_NAME - requirement01

## Requirement Notes

## Source Files

## XML To PlantUML Mapping Rule

## Table Grouping Rule

## Lookup Summary From XML And XML.layout

## Important Script Summary

## High-Level Flow From XML.layout

## Gateway Summary

## Error / Status Pattern

## Example PlantUML

```plantuml
@startuml
...
@enduml
```
```

## Quality Checklist

Before finishing, check:

- the XML and XML.layout basename match
- the flow order follows layout, not guesswork
- every important branch is represented
- `activate` / `deactivate` is present for called workflows and database calls
- note blocks explain important input/output and logic
- ADW tables are grouped by real table name when applicable
- every `group` line uses `#D4EEFB` (no other color allowed on `group`)
- the `.puml` matches the markdown code block exactly

## Local Style Alignment

Match the style already used in this workspace:

- concise section headings
- Thai explanatory prose is acceptable and preferred when the user writes Thai
- detailed but readable notes
- sequence-diagram-first approach
- practical summaries instead of code dumps
