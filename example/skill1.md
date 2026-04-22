---
name: zoral-workflow-to-plantuml
description: Convert a Zoral ProcessWorkflowConfiguration XML (plus its .layout file) into a detailed PlantUML sequence diagram. Use when the folder contains `<workflow>.xml` + `<workflow>.xml.layout` and the user asks to generate/update a UML diagram file.
---

# Zoral Workflow XML → PlantUML Sequence Diagram

สร้างไฟล์ `plant_uml.puml` จาก ProcessWorkflowConfiguration XML โดยให้แผนภาพอ่านง่าย ละเอียด และใช้ palette โทนฟ้า + ครีม/เหลืองเท่านั้น

---

## Inputs (required in the same folder)

| File | Purpose |
|---|---|
| `<workflow>.xml` | ProcessWorkflowConfiguration — items, flow, scripts, queries |
| `<workflow>.xml.layout` | positions/connections (optional hints for ordering) |
| `plant_uml_example` หรือ `plant_uml_example.puml` | *(optional)* style reference only |

## Output

- `plant_uml.puml` — PlantUML sequence diagram (UTF-8, `@startuml ... @enduml`)

---

## Procedure

### 1. Parse the XML workflow

อ่านไฟล์ XML แล้วเก็บข้อมูลต่อไปนี้:

- **GlobalVariables** จาก `<GlobalVariables><add name=... value=... />` → โชว์ค่าเริ่มต้น
- **EmbeddedRuntime** → `Dependency[Name=input].OutputJsonSchema` → input fields
- สำหรับแต่ละ `<Items>` child อ่าน:
  - `ItemId`, `Name`, `NextId`
  - ประเภท node: `MessageStartEvent`, `EndEvent`, `ComponentTask`, `ScriptTask`, `Gateway`, `SubDecisionTask`
  - `BoundaryEvents/BoundaryErrorEvent` → error route (`NextId`)
  - สำหรับ `Gateway`: อ่าน `GatewayConnections/Connection[ToId, IsElse, Condition/Script]`
  - สำหรับ `ComponentTask[ComponentName=AdwQuery]`: ใน `<Script>` หา `result.query = \`...\`` (GraphQL) และ `result.variables = {...}` → ดึง **table names** + **WHERE conditions** + **SELECT columns**
  - สำหรับ `ScriptTask`: สกัดเจตนาของสคริปต์ (filter rules, branching logic, globalVariable mutations)
  - `SubWorkflows/SubWorkflow` → sub-decision (เช่น `StatusCodeDecision`)

### 2. Build the flow graph

- เริ่มที่ `MessageStartEvent` แล้วเดินตาม `NextId`
- ที่ `Gateway` แยกเป็น **true branch** และ **else branch** (จาก `IsElse`)
- บันทึก error routes แยก (ทุก BoundaryErrorEvent ที่ชี้ไป StatusCode2000)
- จบที่ `EndEvent`

### 3. Group databases by DB box

ดูคำนำหน้า table ใน GraphQL:

| Prefix ใน query | DB Box |
|---|---|
| `lps_corporate_loan_*` | **LPS Corporate Loan DB** |
| `master_*` (เช่น `master_mas_product_matrix`, `master_mas_agricultural_p`, `master_backend_parameter`) | **Master DB** |
| SubWorkflow (เช่น `StatusCodeDecision`) | **Sub-Workflow** box แยก |

### 4. Compose the PlantUML file

ใช้ template ด้านล่าง อย่าละเลยสิ่งเหล่านี้:

- `autonumber "<b>[00]"` — ใส่เลขขั้นให้ทุก message
- แต่ละ ComponentTask/ScriptTask ห่อด้วย `group` + header `== STEP n: <ชื่อสื่อความหมาย> (Item #<id>) ==`
- ระบุใน note ของแต่ละ DB query:
  - `WHERE` clause พร้อมตัวแปร `$...` (ใช้ `<color blue>**$var**</color>` สำหรับตัวแปร input)
  - รายการ columns ที่ SELECT (สรุปเป็น bullet ถ้ายาว)
- Gateway ใช้ `alt / else` พร้อมเงื่อนไขใน header (เอาจาก `Condition/Script`)
- BoundaryErrorEvent สรุปท้ายไฟล์ในรูปโน้ต (อธิบายว่า step ไหนบ้าง → Item #StatusCode2000)

---

## Color palette (ข้อบังคับ)

**ห้ามใช้** สี green, red/coral, pink, หรือ gray เด็ดขาด

### 🔵 Blue tones
| Hex | ใช้กับ |
|---|---|
| `#E1FAFD` | กล่องหลัก zoral (อ่อนสุด) |
| `#D4EEFB` | group สำหรับ DB query tasks |
| `#D6EAF8` | else branch ของ alt |
| `#A6DEEE` | กล่อง DB (Master / LPS) |
| `#A9DCDF` | ParticipantBackground / ActorBackground |
| `#80BBEC` | boundary zoral + database เน้น |
| `#67C7E2` | สี default ของ database tables |
| `#2874A6` | LifeLineBorderColor / GroupBorderColor |

### 🟡 Cream / Yellow tones
| Hex | ใช้กับ |
|---|---|
| `#FFFDEB` | GroupBackgroundColor (พื้นหลัง group อ่อนสุด) |
| `#FCF3CF` | group ของ ScriptTask / logic in-memory |
| `#FFF2B8` | true branch ของ alt |
| `#F9E79F` | กล่อง Sub-Workflow |
| `#F7DC6F` | SubDecisionTask + note เตือน error |

---

## PlantUML structure template

```plantuml
@startuml
title <WorkflowName> (Detailed Sequence)

actor Requestor

box "zoral (Workflow Engine)" #E1FAFD
  boundary "zoral" as zoral #80BBEC
end box

box "Master DB" #A6DEEE
  database "<master_table_a>" as <alias_a> #67C7E2
  ' ...
end box

box "LPS Corporate Loan DB" #A6DEEE
  database "<lps_table_a>" as <alias_a> #67C7E2
  ' ...
end box

box "Sub-Workflow" #F9E79F
  participant "<SubName>\n(<CallerItem>)" as <subAlias> #F7DC6F
end box

hide footbox
autonumber "<b>[00]"

skinparam defaultFontName Comic Sans MS
skinparam sequence {
  ParticipantBorderColor      black
  ParticipantBackgroundColor  #A9DCDF
  ParticipantFontName         Comic Sans MS
  ParticipantFontSize         14
  ActorBackgroundColor        #A9DCDF
  ActorBorderColor            black
  ArrowFontName               Comic Sans MS
  LifeLineBorderColor         #2874A6
  GroupBackgroundColor        #FFFDEB
  GroupBorderColor            #2874A6
}

Requestor -> zoral : **<WorkflowName> (StartEvent #<id>)**
note over zoral #FCF3CF
  **Input payload**  ... (list input fields)
  **Global variables (initial)**  ... (list globals)
end note

== STEP 1: <task name> (Item #<id>) ==
group #D4EEFB <AdwQuery...> (Item #<id>)
  zoral -> <db> : Inquiry <table>
  activate <db>
    note right of <db>
      **WHERE** <col> = <color blue>**$<var>**</color>
      select: <columns>
    end note
  <db> --> zoral : <response>
  deactivate <db>

  alt Boundary Error
    zoral -> zoral : goto Item #<errorItem> (StatusCode2000)
    note right of zoral #F7DC6F
      set **globalVariables.statusCode = 2000**
    end note
  end
end

' ... repeat for every Item, respecting ItemId flow ...

== STEP n: Output (Item #<id>) ==
group #FCF3CF Output - ScriptTask (Item #<id>)
  zoral -> zoral : Post-processing & output assembly
  activate zoral
    note right of zoral
      (สรุป logic: filter, merge, flag handling, return shape)
    end note
  deactivate zoral
end

note over zoral #F7DC6F
  **BoundaryErrorEvent** รวม step ที่ error ทั้งหมด → Item #<StatusCode2000Id>
end note

zoral --> Requestor : **response** { status, data } (EndEvent #<id>)
@enduml
```

---

## Writing tips (ที่ทำให้อ่านง่ายกว่า example)

1. **หัว group** ให้ใส่ `Item #<id>` เสมอ เพื่อ cross-reference กลับ XML ได้
2. Query ที่ยิงหลายตารางใน ComponentTask เดียว → รวมไว้ใน `group` เดียว ไม่แยก (เช่น `AdwQueryFormAndSummary` ยิง 3 ตาราง)
3. โน้ต `note right of <db>` แสดง **WHERE** และ **SELECT** ครบ (column list ถ้ายาวสรุปเป็น bullet)
4. สำหรับ Gateway ใส่เงื่อนไขเป็น header ของ `alt/else` เช่น `alt #FFF2B8 globalVariables.mapping.length > 1`
5. Business rule ใน ScriptTask (เช่น filter priority, special customer codes) → แยก note เป็นบล็อกๆ มี heading เช่น `=== Filter rules ===`
6. ระบุ global variable ที่ mutate ในแต่ละ step อย่างชัดเจน (เช่น `globalVariables.selectMapping = picked row`)
7. BoundaryErrorEvent รวบไว้ท้ายไฟล์ในโน้ตเดียว ระบุ list ItemId ที่ catch

---

## Validation checklist (ก่อนส่งมอบ)

- [ ] ทุก `ItemId` ใน XML ปรากฏใน diagram (ยกเว้น StartEvent/EndEvent ที่ implicit)
- [ ] ทุก Gateway มี `alt/else` ครบตาม `GatewayConnections`
- [ ] ทุก `BoundaryErrorEvent` ถูกกล่าวถึง (อย่างน้อยใน summary note)
- [ ] ทุก DB ที่ reference ใน GraphQL query มี `database` declaration ใน box ที่ถูกต้อง
- [ ] ไม่มี color ที่ไม่อยู่ใน palette (grep `'#'` เช็ค)
- [ ] ไฟล์เปิด preview ใน VS Code (`Option+D`) ได้ไม่มี syntax error
- [ ] ไฟล์ชื่อลงท้ายด้วย `.puml` หรือ `.plantuml` เพื่อให้ extension auto-detect
