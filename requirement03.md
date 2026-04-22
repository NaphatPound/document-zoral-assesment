# INT_CPL_inquiry_assessment_schema_0 - requirement03

## Requirement Notes

ไฟล์นี้เป็น version ที่แตก `Zoral ADW` ออกเป็น `database per table` และจัดกลุ่มตาม prefix ของ table เพื่อใช้เป็น template สำหรับ workflow ถัดไปที่ต้องการเห็น dependency ฝั่ง ADW ชัดขึ้น

## Source Files

- `INT_CPL_inquiry_assessment_schema_0.xml`
- `INT_CPL_inquiry_assessment_schema_0.xml.layout`
- `requirement03.md`

## Table Grouping Rule

- `disbursement_*` -> `DBM`
- `master_*` -> `Master`
- `lps_corporate_loan_*` -> `Corporate Loan`
- `lps_collateral_*` -> `Collateral`
- `lps_document_service_*` -> `Document service`
- `lps_retail_loan_*` -> `Retail Loan` (inferred from XML)
- `cif_*` -> `CIF` (inferred from XML)

## Table Inventory From XML

### Corporate Loan

- `lps_corporate_loan_trn_corp_assessment_answer`
- `lps_corporate_loan_corp_loan_request_header`
- `lps_corporate_loan_corp_loan_request`

### Retail Loan

- `lps_retail_loan_loan_request_header`
- `lps_retail_loan_loan_request`

### DBM

- `disbursement_trn_disbursement`
- `disbursement_trn_corp_disbursement`
- `disbursement_trn_change_disbursement`

### Master

- `master_fn_get_interest_rate`
- `master_mas_index`

### CIF

- `cif_mas_assessment_answer`

### Not Used In This Workflow

- `lps_collateral_*`
- `lps_document_service_*`

## Example PlantUML

```plantuml
@startuml

' http://plantuml.com/PlantUML_Language_Reference_Guide.pdf

title Zoral : INT_CPL_inquiry_assessment_schema (table-group version)
'''''''''''''''''''''''''''''''''''''''''''''''''
' define diagram participants
'''''''''''''''''''''''''''''''''''''''''''''''''

actor Requestor

box "Zoral" #BFE2FF
boundary "Zoral" as zoral #67C7E2
end box

box "Zoral Internal" #A6DEEE
boundary "Zoral Internal" as zoral_internal #67C7E2
end box

box "Zoral External" #A6DEEE
boundary "Zoral External" as zoral_external #67C7E2
end box

box "Corporate Loan" #BFE2FF
database "lps_corporate_loan_trn_corp_assessment_answer" as db_corp_answer #67C7E2
database "lps_corporate_loan_corp_loan_request_header" as db_corp_header #67C7E2
database "lps_corporate_loan_corp_loan_request" as db_corp_request #67C7E2
end box

box "Retail Loan" #BFE2FF
database "lps_retail_loan_loan_request_header" as db_retail_header #67C7E2
database "lps_retail_loan_loan_request" as db_retail_request #67C7E2
end box

box "DBM" #BFE2FF
database "disbursement_trn_disbursement" as db_disbursement #67C7E2
database "disbursement_trn_corp_disbursement" as db_corp_disbursement #67C7E2
database "disbursement_trn_change_disbursement" as db_change_disbursement #67C7E2
end box

box "Master" #BFE2FF
database "master_fn_get_interest_rate" as db_interest_rate #67C7E2
database "master_mas_index" as db_mas_index #67C7E2
end box

box "CIF" #BFE2FF
database "cif_mas_assessment_answer" as db_cif_answer #67C7E2
end box

'''''''''''''''''''''''''''''''''''''''''''''''''
' formatting options
'''''''''''''''''''''''''''''''''''''''''''''''''
hide footbox

skinparam defaultFontName Comic Sans MS
skinparam sequence {
ParticipantBorderColor black
ParticipantBackgroundColor #A9DCDF
ParticipantFontName Comic Sans MS
ParticipantFontSize 15
ActorBackgroundColor #A9DCDF
ActorBorderColor black
ArrowFontName Comic Sans MS
}

'''''''''''''''''''''''''''''''''''''''''''''''''
' describe sequence logic
'''''''''''''''''''''''''''''''''''''''''''''''''
== Get CIF Data ==

Requestor -> zoral : INT_CPL_inquiry_assessment_schema
activate zoral
note right of zoral
  Version นี้เน้นแตก ADW เป็นราย table
  และจัดกลุ่มตาม prefix ของ table name
end note

group Fetch CIF And Customer Flags
  zoral -> zoral_internal : Call INT_CIF_corporate_transaction_inquiry
  activate zoral_internal
  note right of zoral_internal
    Input:
    - header.applicationId
  end note
  zoral_internal --> zoral : response cifData
  note right of zoral
    Script `setCifData`
    - set globalVariables.cifData
    - ใช้ customerCode ตัดสิน history flow
  end note
  deactivate zoral_internal
  note right of zoral_internal
    Output:
    - cifData
  end note

  zoral -> zoral : isSavingCoopOrAgriCoop
  note right of zoral
    ตัดสินว่าจะ query answer history หรือไม่
    ตาม assessmentType + customerCode
  end note

  alt if customer requires master answer history
    zoral -> db_cif_answer : AdwQueryMasterAnswers
    activate db_cif_answer
    note right of db_cif_answer
      Table:
      - cif_mas_assessment_answer

      Filter:
      - cif_no
      - is_show_history (บาง assessment type)
    end note
    db_cif_answer --> zoral : response masAnswer
    note right of zoral
      Script `setMasAnswer`
      - globalVariables.masAnswer = result[]
    end note
    deactivate db_cif_answer
    note right of db_cif_answer
      Output:
      - master answer history
    end note
  end
end group

== Assessment Type Routing ==

alt assessmentType in ["CREATE_LOAN", "CHANGE_LOAN", "OVERDRAFT"]
  alt corporate flow
    zoral -> db_corp_header : AdwQueryCorpLoanRequestHeader
    activate db_corp_header
    note right of db_corp_header
      Table:
      - lps_corporate_loan_corp_loan_request_header

      Filter:
      - loan_request_id
    end note
    db_corp_header --> zoral : response corp header
    note right of zoral
      header part ของ corp loan
    end note
    deactivate db_corp_header
    note right of db_corp_header
      Output:
      - corp loan request header
    end note

    zoral -> db_corp_request : AdwQueryCorpLoanRequestHeader
    activate db_corp_request
    note right of db_corp_request
      Table:
      - lps_corporate_loan_corp_loan_request

      Filter:
      - loan_request_id
      - loan_id
    end note
    db_corp_request --> zoral : response corp request
    note right of zoral
      Script `setCorpLoan`
      - set request + requestHeader
      - ถ้าไม่พบข้อมูล throw 2000
    end note
    deactivate db_corp_request
    note right of db_corp_request
      Output:
      - corp loan request
    end note

  else retail flow
    zoral -> db_retail_header : AdwQueryLoanRequestHeader
    activate db_retail_header
    note right of db_retail_header
      Table:
      - lps_retail_loan_loan_request_header

      Filter:
      - loan_request_id
    end note
    db_retail_header --> zoral : response retail header
    note right of zoral
      header part ของ retail loan
    end note
    deactivate db_retail_header
    note right of db_retail_header
      Output:
      - retail loan request header
    end note

    zoral -> db_retail_request : AdwQueryLoanRequestHeader
    activate db_retail_request
    note right of db_retail_request
      Table:
      - lps_retail_loan_loan_request

      Filter:
      - loan_request_id
      - loan_id
    end note
    db_retail_request --> zoral : response retail request
    note right of zoral
      Script `setLoan`
      - set request + requestHeader
      - ถ้าไม่พบข้อมูล throw 2000
    end note
    deactivate db_retail_request
    note right of db_retail_request
      Output:
      - retail loan request
    end note
  end

else assessmentType in ["CREATE_DBM", "CHANGE_DBM", "AUTO_DBM"]
  zoral -> db_corp_request : AdwQueryLoanRequestByLoanAccount
  activate db_corp_request
  note right of db_corp_request
    Table:
    - lps_corporate_loan_corp_loan_request

    Filter:
    - loan_account
    - order_by loan_amount desc
  end note
  db_corp_request --> zoral : response corpLoanDBM
  note right of zoral
    ข้อมูล corp loan candidate
  end note
  deactivate db_corp_request
  note right of db_corp_request
    Output:
    - corp DBM candidate
  end note

  zoral -> db_retail_request : AdwQueryLoanRequestByLoanAccount
  activate db_retail_request
  note right of db_retail_request
    Table:
    - lps_retail_loan_loan_request

    Filter:
    - loan_account
    - order_by sum_loan_purpose_amount desc
  end note
  db_retail_request --> zoral : response loanDBM
  note right of zoral
    Script `setLoanDBM`
    - เทียบ retail vs corp
    - เลือก request ที่วงเงินสูงกว่า
    - ถ้าไม่พบข้อมูลเลย set isInCBS = true
  end note
  deactivate db_retail_request
  note right of db_retail_request
    Output:
    - retail DBM candidate
  end note

  alt requestLoanId exists in LPS
    zoral -> db_corp_header : AdwQueryLoanRequestHeaderDBM
    activate db_corp_header
    note right of db_corp_header
      Table:
      - lps_corporate_loan_corp_loan_request_header
    end note
    db_corp_header --> zoral : response corp header
    note right of zoral
      corp header candidate
    end note
    deactivate db_corp_header
    note right of db_corp_header
      Output:
      - corp request header
    end note

    zoral -> db_retail_header : AdwQueryLoanRequestHeaderDBM
    activate db_retail_header
    note right of db_retail_header
      Table:
      - lps_retail_loan_loan_request_header
    end note
    db_retail_header --> zoral : response retail header
    note right of zoral
      Script `setLoanHeaderDBM`
      - เลือก header จาก retail/corp
      - retail เป็น default ถ้ามีทั้งสองฝั่ง
    end note
    deactivate db_retail_header
    note right of db_retail_header
      Output:
      - retail request header
    end note
  else lookup account info from CBS
    zoral -> zoral_external : Call EXT_cbs_account_information_inquiry
    activate zoral_external
    note right of zoral_external
      Input:
      - payload.accountNo = input.payload.loanAccount
    end note
    zoral_external --> zoral : response accountInfo
    note right of zoral
      ใช้ accountInfo สำหรับ market code
      และกรณี loan อยู่ใน CBS
    end note
    deactivate zoral_external
    note right of zoral_external
      Output:
      - productType
      - accountSubtype
      - marketCode
    end note
  end

  alt normal DBM flow
    zoral -> db_disbursement : queryTrnDisbursement
    activate db_disbursement
    note right of db_disbursement
      Table:
      - disbursement_trn_disbursement
    end note
    db_disbursement --> zoral : response disbursement
    note right of zoral
      Script `setViewModeDBM`
      - REQUEST / DOCUMENT / SIGNATURE = edit
      - stage อื่น = view mode
    end note
    deactivate db_disbursement
    note right of db_disbursement
      Output:
      - disburse_stage
    end note

    opt corp DBM branch
      zoral -> db_corp_disbursement : queryTrnCorpDisbursement
      activate db_corp_disbursement
      note right of db_corp_disbursement
        Table:
        - disbursement_trn_corp_disbursement
      end note
      db_corp_disbursement --> zoral : response corp disbursement
      note right of zoral
        Script `setViewModeCorpDBM`
        - ใช้กติกา stage เดียวกับ DBM ปกติ
      end note
      deactivate db_corp_disbursement
      note right of db_corp_disbursement
        Output:
        - corp disburse_stage
      end note
    end

  else change DBM flow
    zoral -> db_change_disbursement : queryTrnChangeDisbursement
    activate db_change_disbursement
    note right of db_change_disbursement
      Table:
      - disbursement_trn_change_disbursement
    end note
    db_change_disbursement --> zoral : response change disbursement
    note right of zoral
      Script `setViewModeDBM`
      - ใช้ result ของ change disbursement
    end note
    deactivate db_change_disbursement
    note right of db_change_disbursement
      Output:
      - change disburse_stage
    end note

    opt corp change DBM branch
      zoral -> db_change_disbursement : queryTrnChangeCorpDisbursement
      activate db_change_disbursement
      note right of db_change_disbursement
        Table:
        - disbursement_trn_change_disbursement

        ใช้ table เดียวกัน
        แต่คนละ component name
      end note
      db_change_disbursement --> zoral : response corp change disbursement
      note right of zoral
        Script `setViewModeChangeDBM`
        - ใช้ corp change stage สำหรับตัดสิน view mode
      end note
      deactivate db_change_disbursement
      note right of db_change_disbursement
        Output:
        - corp change disburse_stage
      end note
    end
  end

  zoral -> zoral_internal : Call INT_DBM_customer_major_loan_verify
  activate zoral_internal
  note right of zoral_internal
    Input:
    - cifId
    - loanAccount
  end note
  zoral_internal --> zoral : response verify result
  note right of zoral
    ใช้ verify result สำหรับ DBM condition
  end note
  deactivate zoral_internal
  note right of zoral_internal
    Output:
    - isPassCondition
  end note

else assessmentType == "REVIEW"
  zoral -> db_corp_request : AdwQueryLoanRequestByDisburseNo
  activate db_corp_request
  note right of db_corp_request
    Table:
    - lps_corporate_loan_corp_loan_request

    Filter:
    - disburse_no
  end note
  db_corp_request --> zoral : response corp review data
  note right of zoral
    corp review candidate
  end note
  deactivate db_corp_request
  note right of db_corp_request
    Output:
    - corp review loan
  end note

  zoral -> db_retail_request : AdwQueryLoanRequestByDisburseNo
  activate db_retail_request
  note right of db_retail_request
    Table:
    - lps_retail_loan_loan_request

    Filter:
    - disburse_no
  end note
  db_retail_request --> zoral : response retail review data
  note right of zoral
    Script `setLoanDBMByDisburseNo`
    - resolve request ของ review flow
    - ใช้ข้อมูลจาก corp/retail ที่ query ได้
  end note
  deactivate db_retail_request
  note right of db_retail_request
    Output:
    - retail review loan
  end note
end

== Market Code And Interest Rate ==

zoral -> zoral_external : Call EXT_get_market_code_detail
activate zoral_external
note right of zoral_external
  Build payload จาก request / requestHeader
  หรือ accountInfo จาก CBS เมื่อ isInCBS = true
end note
zoral_external --> zoral : response marketCode detail
note right of zoral
  Script `map_list_interest_rate_code`
  - collect market interest code หลาย key
  - append LN0100, LN0200, LN0800
  - dedupe list
end note
deactivate zoral_external
note right of zoral_external
  Output:
  - interest code candidates
end note

alt if interest rate code list exists
  zoral -> db_interest_rate : fn_get_interest_rate
  activate db_interest_rate
  note right of db_interest_rate
    Table:
    - master_fn_get_interest_rate
  end note
  db_interest_rate --> zoral : response interest rate
  note right of zoral
    ค่า rate จะถูก parse และนำไปจับคู่กับ desc
  end note
  deactivate db_interest_rate
  note right of db_interest_rate
    Output:
    - decimal_data
  end note

  zoral -> db_mas_index : mas_index
  activate db_mas_index
  note right of db_mas_index
    Table:
    - master_mas_index
  end note
  db_mas_index --> zoral : response interest desc
  note right of zoral
    Script `map_interest_rate`
    - combine interestRate + interestRateTypeDesc
    - push เข้า globalVariables.interestRateList
  end note
  deactivate db_mas_index
  note right of db_mas_index
    Output:
    - interest_desc
  end note

else integrate answer from CBS
  zoral -> zoral_internal : Call INT_CPL_inquiry_assessment_integrate_CBS
  activate zoral_internal
  note right of zoral_internal
    Input:
    - cifId
    - option = 1
  end note
  zoral_internal --> zoral : response assessment data
  note right of zoral
    Script `mappingUser`
    - set customerStatus

    Script `mapping`
    - build query/variables สำหรับ answer lookup
  end note
  deactivate zoral_internal
  note right of zoral_internal
    Output:
    - integrated assessment context
  end note

  zoral -> db_corp_answer : AdwQueryAnswers
  activate db_corp_answer
  note right of db_corp_answer
    Table:
    - lps_corporate_loan_trn_corp_assessment_answer

    Filter:
    - application_id
    - cif_no
    - assessment_type
    - loan_id (บาง assessment type)
  end note
  db_corp_answer --> zoral : response saved answers
  note right of zoral
    Script `setAnswers`
    - reuse masAnswer ถ้ายัง valid
    - REVIEW สามารถ restore request/requestHeader
    - ถ้าไม่เข้าเงื่อนไข จะ fallback มาที่ saved answer
  end note
  deactivate db_corp_answer
  note right of db_corp_answer
    Output:
    - saved assessment answer
  end note
end

== Assessment Schema And Output ==

zoral -> zoral : setAnswers
note right of zoral
  จุดนี้ answer ถูก resolve แล้ว
  เพื่อใช้ตัดสิน first-time flow หรือ view mode
end note

opt if view mode flow is required
  zoral -> zoral_internal : Call INT_CPL_inquiry_cap_product_config
  activate zoral_internal
  note right of zoral_internal
    Input:
    - staffId
    - workflowId
    - loanRequestId
  end note
  zoral_internal --> zoral : response cap product config
  note right of zoral
    ใช้ screenMode สำหรับคำนวณ isIntegrate
  end note
  deactivate zoral_internal
  note right of zoral_internal
    Output:
    - screenMode
  end note
end

alt if answer/schema data is available
  zoral -> zoral_internal : Call INT_CPL_inquiry_assessment
  activate zoral_internal
  note right of zoral_internal
    Input:
    - accountType
    - groupType
    - businessType
    - marketCode
    - loanAmount
  end note
  zoral_internal --> zoral : response assessment schema
  note right of zoral
    Script chain:
    - checkFormSelect
    - mappingForm
    - AddSection
    - AddCalculate
    - PointerCount
    - MappingSchema
  end note
  deactivate zoral_internal
  note right of zoral_internal
    Output:
    - form / section / topic / option / summary / mapping
  end note

else no answer or no matching form
  zoral -> zoral_internal : Call StatusCodeDecision
  activate zoral_internal
  note right of zoral_internal
    Input:
    - globalVariables.statusCode
  end note
  zoral_internal --> zoral : response status
  note right of zoral
    fallback status สำหรับ final response
  end note
  deactivate zoral_internal
  note right of zoral_internal
    Output:
    - status object
  end note
end

group Global Error Pattern
  note right of zoral
    Global error path ใน XML:
    GlobalErrorEvent(92)
    -> GlobalError(91)
    -> ErrRespones(90)
    -> ErrEnd(93)
  end note
end group

zoral -> zoral : Output
note right of zoral
  Final builder:
  - property
  - schema
  - answer
  - calculate
  - dataIntegrate
  - variableList
  - isIntegrate / isViewMode
end note

zoral --> Requestor : return response
deactivate zoral
note right of zoral
  กลุ่ม table ที่ไม่มีใช้ใน workflow นี้:
  - lps_collateral_*
  - lps_document_service_*
end note

@enduml
```

## Reusable Guideline

- ถ้าต้องการมุมมอง business flow ใช้ `INT_CPL_inquiry_assessment_schema_0.md`
- ถ้าต้องการมุมมอง script-note ใช้ `requirement02.md`
- ถ้าต้องการมุมมอง dependency ฝั่ง ADW แบบราย table ใช้ `requirement03.md`
- ถ้ามี `AdwQuery` ที่ query หลาย table ใน component เดียว ให้แตกเป็นหลาย database participant แบบในไฟล์นี้
- ถ้ามี table prefix ใหม่ ให้เพิ่ม group ใหม่ตาม prefix นั้น หรือใส่เป็น inferred group พร้อม note
