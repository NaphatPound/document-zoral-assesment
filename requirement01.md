## Requirement Notes

เอกสารนี้ใช้ PlantUML ด้านล่างเป็น `example reference` สำหรับ workflow กลุ่มเดียวกัน และใช้คู่ไฟล์ `INT_DBM_customer_verify_0.xml` กับ `INT_DBM_customer_verify_0.xml.layout` เป็น source of truth สำหรับ lookup โครงสร้างจริงของ flow

## Source Files

- `INT_DBM_customer_verify_0.xml`
- `INT_DBM_customer_verify_0.xml.layout`
- `requirement01.md`

## XML To PlantUML Mapping Rule

- `SubProcessTask` ให้ map เป็นการ call workflow ใน sequence diagram
- ถ้า `WorkflowName` ขึ้นต้นด้วย `EXT_` ให้จัดอยู่ในกลุ่ม `zoral_external`
- ถ้า `WorkflowName` ขึ้นต้นด้วย `INT_` ให้จัดอยู่ในกลุ่ม `zoral_internal`
- `SubDecisionTask` เช่น `LookUp -> StatusCodeDecision` ให้ใช้เป็น decision / error lookup step
- `ComponentTask` ที่ `ComponentName = AdwQuery` ให้ map เป็น `database`
- `Gateway` ให้แปลงเป็น `alt`, `opt`, `loop` หรือ branch control ตามชื่อ gateway และเส้นทางใน `.xml.layout`
- `ScriptTask` ที่ขึ้นต้นด้วย `Error` ให้สรุปเป็น hard stop note พร้อม `status.code`
- `ScriptTask` ที่ขึ้นต้นด้วย `Pre`, `Post`, `Set`, `Validate`, `Check` ให้ตีความเป็น preparation, validation, post-process หรือ condition note
- ใช้ `.xml.layout` เป็นตัวตัดสินลำดับจริงของ flow และการเชื่อม branch ไม่ใช่ดูจากชื่อ step อย่างเดียว
- เวลาเขียน sequence diagram ให้ใช้ `->` สำหรับ request/call และ `-->` สำหรับ response/return
- ใส่ `activate` / `deactivate` ทุกครั้งที่มี sub process, external service, internal workflow หรือ database call เพื่อให้เห็น timeline

## Lookup Summary From XML And XML.layout

### Main Workflow Inventory

- `3` `ExtCbsPersonalProfileInquiry` -> `EXT_cbs_personal_profile_inquiry`
- `4` `LookUp` -> `StatusCodeDecision`
- `9` `ExtDopaCardStatus` -> `EXT_dopa_card_status_by_laser_inquiry`
- `12` `IntColContractInquiry` -> `INT_COL_contract_inquiry`
- `13` `ExtBankruptInquiry` -> `EXT_bankrubt_inquiry`
- `21` `OtherExtBankruptInquiry` -> `EXT_bankrubt_inquiry`
- `26` `ExtCddInquiry` -> `EXT_cdd_inquiry`
- `41` `CbsPersonalProfileInquiry` -> `EXT_cbs_personal_profile_inquiry`
- `45` `subprocess_INT_DBM_account_verify` -> `INT_DBM_account_verify`
- `48` `subprocess_INT_DBM_cif_verify` -> `INT_DBM_cif_verify`
- `70` `IntAuditEvent` -> `INT_audit_event_publish`
- `71` `DBMCustomerMajorLoanVerify` -> `INT_DBM_customer_major_loan_verify`
- `74` `DBMCustomerBranchVerify` -> `INT_DBM_customer_branch_verify`
- `78` `DBMCustomerEndPeriod` -> `INT_DBM_customer_end_period_verify`
- `81` `DBMCustomerAccountMaturityDateVerify` -> `INT_DBM_customer_account_maturity_date_verify`
- `84` `IntColRequestSheetListInquiry` -> `INT_COL_request_sheets_list_inquiry`
- `94` `INT_RTL_loan_information_inquiry` -> `INT_RTL_loan_information_inquiry`
- `98` `INT_DBM_change_and_close_commitment_verify` -> `INT_DBM_change_and_close_commitment_verify`
- `105` `INT_DBM_collateral_verify` -> `INT_DBM_collateral_verify`

### Component Inventory

- `89` `QueryBackendParameter` -> `AdwQuery`
- `110` `GlobalError` -> `GlobalError`

### High-Level Flow From Layout

1. `StartEvent(1)` -> `ExtCbsPersonalProfileInquiry(3)` -> `ValidateCitizenId(7)` -> `ExtDopaCardStatus(9)` -> `ValidateDopa(10)`
2. ผ่านชุด pre-condition หลัก -> `DBMCustomerMajorLoanVerify(71)` -> `DBMCustomerBranchVerify(74)` -> `DBMCustomerEndPeriod(78)` -> `DBMCustomerAccountMaturityDateVerify(81)`
3. หลังจากผ่าน pre-condition -> `IntColContractInquiry(12)` -> `checkContractStatus(87)` -> `CheckContractInfo(65)` -> `ExtBankruptInquiry(13)` -> loop LED (`16/17/18/20/21`)
4. จาก loop LED ไปตรวจ `CDD / Rehabilitation` ผ่าน `ExtCddInquiry(26)` และ gateway `27`, `29`
5. จากนั้นเข้า loop ตรวจ `Collateral Status` ผ่าน `32`, `33`, `34`, `84`, `85`
6. ต่อด้วย `Live Status` และการไล่ `owner / guarantor` ผ่าน `37`, `39`, `40`, `41`, `42`, `43`, `63`, `64`
7. ช่วงท้ายเป็น `INT_DBM_account_verify(45)` -> `INT_DBM_cif_verify(48)` -> `QueryBackendParameter(89)` -> `INT_RTL_loan_information_inquiry(94)`
8. branch ท้าย flow จะต่อไป `INT_DBM_change_and_close_commitment_verify(98)` หรือ `INT_DBM_collateral_verify(105)` ก่อนจบที่ `PreAuditEvent(68)` -> `IntAuditEvent(70)` -> `MessageResponse(5)` -> `End(2)`

### Error / Hard Stop Pattern Seen In XML

- error script ที่พบ เช่น `Error8032`, `Error8002`, `Error8004`, `Error8014`, `Error8015`, `Error8094`, `Error8095`, `Error8096`, `Error8097`, `Error8113`, `Error8117`, `Error8118`, `Error9000`, `Error9100`
- pattern โดยรวมคือ error script ส่วนมากจะวิ่งกลับ `LookUp(4)` เพื่อ resolve `status.code` ก่อนส่ง `MessageResponse(5)`
- ถ้าเป็น global exception จะเข้าทาง `GlobalErrorEvent(108)` -> `GlobalError(110)` -> `ErrorResponse(111)` -> `GlobalErrorEnd(112)`

## Example PlantUML

```plantuml
@startuml

' http://plantuml.com/PlantUML_Language_Reference_Guide.pdf

title Zoral : INT_DBM_customer_verify
'''''''''''''''''''''''''''''''''''''''''''''''''
' define diagram participants
'''''''''''''''''''''''''''''''''''''''''''''''''

actor Requestor
box "Zoral" #BFE2FF
boundary "Zoral" as zoral#67C7E2

box "Zoral Internal" #A6DEEE
boundary "Zoral" as zoral_internal #67C7E2
end box


box "Zoral External" #A6DEEE
boundary "Zoral" as zoral_external #67C7E2
end box


box "Zoral ADW" #BFE2FF
database lookup_message as lookup #67C7E2
end box



'''''''''''''''''''''''''''''''''''''''''''''''''
' formatting options
'''''''''''''''''''''''''''''''''''''''''''''''''
hide footbox

'autonumber

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
== Pre-Screening Customer and Relate Person  ==

Requestor -> zoral: INT_DBM_customer_verify
activate zoral

group requested payload validation
zoral -> zoral: validation input param
alt If got any validation error
note right of zoral 
	Hard stop. To return status.code = 9300
end note
end alt
end group

 zoral -> zoral_external : Call EXT_cbs_personal_profile_inquiry 
    activate zoral_external 

          note right of zoral_external 
            Set cifId = input.cifId and needCustomerReg = true
          end note

    zoral_external--> zoral : response  cifData
    deactivate zoral_external

   alt If got any endpoint's error
      note right of zoral 
        Hard stop. To return status.code = 9100
      end note
    end alt


    alt If  cifData.customerRegistrationDetail.citizenId != input.citizenId
      note right of zoral 
        Hard stop. To return status.code = 8032
      end note
    end alt
== DOPA Verify (ตรวจสอบ DOPA )==

group  'DOPA' Verifyfor requester

   zoral -> zoral_external : Call **EXT_DOPA_card_status_by_laser_inquiry**
    activate zoral_external 
    zoral_external--> zoral :  response dopaResponse
    deactivate zoral_external
   
    alt if any connection error
            note right of zoral
              Hard stop and return status.code  9100
            end note
    end alt

       alt if dopaResponse.code !=  "0"
           note right of zoral
            Hard stop and return status.code  8002
          end note
        end alt
        
end group

== Major  Loan Verify (ตรวจสอบสินเชื่อรายใหญ่)==
group Major  Loan Verify

zoral -> zoral_internal : Call **Zoral: INT_DBM_customer_major_loan_verify **
    activate zoral_internal 
    zoral_internal--> zoral :  response 
    deactivate zoral_internal

alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
 end alt

alt if  response.isPassCondition = false
  note right of zoral 
          Hard stop and return status.code  8094 (เนื่องจากขณะนี้ระบบ LPS ยังไม่รองรับการขอเบิกสินเชื่อสำหรับลูกค้ารายใหญ่)
  end note
end alt

end group

== Branch of Ownership Verify (ตรวจสอบการเบิกต่างสาขา)==

group Branch of Ownership  Verify

zoral -> zoral_internal : Call **Zoral: INT_DBM_customer_branch_verify - **
    activate zoral_internal 
    zoral_internal--> zoral :  response 
    deactivate zoral_internal

alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
 end alt

alt if  response.isPassCondition = false
  note right of zoral 
          Hard stop and return status.code  8095 (เนื่องจากสินเชื่อรายย่อยไม่สามารถเบิกเงินกู้ต่างสาขาได้)
  end note
end alt

end group

== End of Period Date for Disbusement Verify (ตรวจสอบ วันที่สิ้นสุดการเบิกจ่ายเงินกู้)==
group  End of Period Date Verify

zoral -> zoral_internal : Call **Zoral: INT_DBM_customer_end_period_verify  - **
    activate zoral_internal 
    zoral_internal--> zoral :  response 
    deactivate zoral_internal

alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
 end alt

alt if  response.isPassCondition = false
  note right of zoral 
          Hard stop and return status.code  8096 (เนื่องจากสิ้นสุดระยะเวลาการเบิกจ่ายเงินกู้ของสินเชื่อนี้แล้ว)
  end note
end alt

end group


== Account Maturity Date Verify (ตรวจสอบ วันที่สิ้นสุดการเบิกจ่ายเงินกู้)==
group Account Maturity Date Verify

zoral -> zoral_internal : Call **Zoral: INT_DBM_customer_account_maturity_date_verify **
    activate zoral_internal 
    zoral_internal--> zoral :  response 
    deactivate zoral_internal

alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
 end alt

alt if  response.isPassCondition = false
  note right of zoral 
          Hard stop and return status.code  8097 (เนื่องจากสินเชื่อหมดอายุสัญญาแล้ว)
  end note
end alt

end group


== LED Verify (ตรวจสอบ LED )==

zoral -> zoral_internal : Call INT_COL_contract_inquiry
activate zoral_internal

	note right of zoral_internal
		set accountNo = input.loanAccount
	end note

zoral_internal--> zoral :  response  contractInfo
deactivate zoral_internal

alt #FFF3BB if INT_COL_contract_inquiry return status.code = 2000
      note right of zoral
        Hard stop and return status.code  8113
      end note
end alt

alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
 end alt


 
group  'LED' Verify  for requester,  owner and guarantor

 

 zoral -> zoral_external : Call EXT_Bankrupt_inquiry
      activate zoral_external 
      note right of zoral_external
            Check 'LED' for requester set cifId = input.cifId
 end note
      zoral_external--> zoral :  response requesterBankrupt
  deactivate zoral_external

alt If got any endpoint's error
        note right of zoral 
            Hard stop. To return status.code = 9100
        end note
      end alt
   

alt If  requesterBankrupt found at least one item of requesterBankrupt.responseData[] / found any Bankrupt info(s)
           note right of zoral
            Hard stop and return status.code  8004
          end note
end alt



loop For each contractInfo.contractInfo[]

  loop For each  collaterals[].collateral
    loop For each cif[]

   alt IF cif[].relatedType ="owner" or  cif[].relatedType = "guarantor"
   
   note right of zoral 
      declare Int guarantorAll = 0;
      declare boolean isGuarantorLed = false;
   end note
   alt  if cif[].relatedType = "guarantor"
      note right of zoral: guarantorAll = guarantorAll +1
   end alt

      zoral -> zoral_external : Call EXT_Bankrupt_inquiry
      activate zoral_external 
      zoral_external--> zoral :  response otherBankrupt
      deactivate zoral_external
    
      alt If got any endpoint's error
        note right of zoral 
            Hard stop. To return status.code = 9100
        end note
      end alt
      alt if any zoral sub process error
        note right of zoral
          Hard stop and return status.code regarding EXT_Bankrupt_inquiry
        end note
      end alt

      


        alt IF cif[].relatedType="owner" and found any item of otherBankrupt.responseData[] and otherBankrupt.responseData[].blacklistCode not in  "18" , "7" , "14"  and  "20" 
            note right of zoral 
              -  Hard stop. To return status.code = 8012
           end note

        else  ELSE IF  cif[].relatedType ="guarantor"  and  contractInfo.collateralType != "1301"  and found any item of otherBankrupt.responseData[]
              note right of zoral 
               -  Hard stop. To return status.code = 8013
            end note
        
        else  ELSE IF cif[].relatedType = "guarantor" and  contractInfo.collateralType =  "1301"  and Not found any item of otherBankrupt.responseData[]
             note right of zoral 
               - add guarantorSuccess= guarantorSuccess+1
         end note
      else  ELSE IF cif[].relatedType = "guarantor" and  contractInfo.collateralType =  "1301"  and  found any item of otherBankrupt.responseData[]
             note right of zoral 
               - Filter out this guarantor bankrupt from cif[] <b>(Refer to LPS-11391)</b>
         end note

         end alt
   end alt

  end loop

  alt IF contractInfo.collateralType  = "1301"  and  guarantorSuccess < 5
    note right of zoral 
              -  Hard stop. To return status.code = 8014
      end note
  end alt

  alt  IF contractInfo.collateralType  = "1301"  and  guarantorSuccess >= 5 and guarantorSuccess != guarantorAll
    note right of zoral 
      isGuarantorLed = true
    end note
  end alt

end loop 
end loop

end group


== CDD Verify (ตรวจสอบ CDD)==

group  VALIDATE   "CDD" for requester 

note right of zoral
            Check 'CDD' for requester set cifId = input.cifId
 end note

 zoral -> zoral_external : Call EXT_CDD_inquiry
      activate zoral_external 
      zoral_external--> zoral :  response cddResponse
  deactivate zoral_external

  alt If got any endpoint's error
        note right of zoral 
            Hard stop. To return status.code = 9100
        end note
      end alt
      alt if any zoral sub process error
        note right of zoral
          Hard stop and return status.code regarding EXT_CDD_inquiry
        end note
      end alt

alt If cddResponse found at least one of cddResponse.responseData[].levelType has "block" or cddResponse.responseData[].levelType has "risk"
           note right of zoral
            Hard stop and return status.code 8061
          end note
 end alt
 end group

 
== REHABILITATION Verify (ตรวจสอบ กฟก)==


group  VALIDATE  "REHABILITATION" for requester

 note right of zoral 
    Check
      - cifData.customerRegistrationDetail.rehabilitationCustomer.rehabilitationCustomerFlag = true 
      - cifData.customerDetail.restrictionList[].restrictionCode = "ZDSB"  
  end note

  alt if  rehabilitationCustomerFlag = true and restrictionCode = "ZDSB"  
    
    
    
    note right of zoral 
                -  Hard stop. To return status.code = 8015
        end note
  end alt
end group


== Collateral Status Verify (ตรวจสอบ สถานะหลักประกัน)==

group   Collateral Status Verify 
  loop For each contractInfo.contractInfo[]
    loop For each collaterals[].collateral

 
      alt If collateral.collateralStatus in notAllowCollateralStatusList
      zoral -> lookup : Query lookup_code = 8016
          activate lookup 
          lookup--> zoral :  response lookupMessage
      deactivate lookup

        note right of zoral        
                        <b>***notAllowCollateralStatusList</b> <font color=red><strike>"CMS004"</strike></font>, <font color=orange>"CMS011"</font><font color=red><strike>"CMS005"      
                              - MS to replace  lookupMessage.description_th paramerter "<collateralStatus>" with collateral.collateralStatusNameTh
                              - MS to set status.desc and status.header
                              - Hard stop. To return status.code = 8016
                end note
        end alt
 
group VALIDATE  "REQUEST STATUS"  (ตรวจสอบสถานะใบงานหลักประกัน)
      zoral -> zoral_internal : Call Zoral : INT_COL_request_sheets_list_inquiry 
          activate zoral_internal 
          zoral_internal--> zoral :  response  sheets[]
      deactivate zoral_internal
  alt #FAE6C9 if status.code = '2000'
    note right of zoral        
      skip loop sheets[]
    end note
  end alt
  loop For each sheets[]
      alt #FDDE77 if data.sheets[].sheetsType = 'COE'
        alt #lightgreen If sheets[].requestStatus  != <color orange>'COE009'(ใบคำขอแก้ไขหลักประกันสมบูรณ์)</color>  <color red><strike>'COL009'(หลักประกันเสร็จสมบูรณ์) </strike></color> <color red><strike>and sheets[].requestStatus  !='COL010' (ใบคำขอถูกยกเลิก) </strike></color>
              note right of zoral        
                                - Hard stop. To return status.code = 8098
                  end note
        end alt
    end alt             
  end loop
end group
    
    
    end loop
  end loop
end group


== Live Status Verify (ตรวจสอบการมีชีวิต)==

group  VALIDATE  "LIVE STATUS" for requester , owner, quarantor
  note right of zoral 
    Check Live Status  for  requester
  end note
  alt if  cifData.customerDetail.dateOfDeath have value
            note right of zoral 
                        - Hard stop. To return status.code = 8017
             end note
   end alt

 note right of zoral 
    Check Live Status  for  owner and guarantor 
  end note

  loop For each contractInfo.contractInfo[]
    loop For each collaterals[].collateral
    loop For each cif[]

        alt if  cif[].relatedType="owner"  and cif[].relatedType="guarantor" 
            
              zoral -> zoral_external : Call EXT_cbs_personal_profile_inquiry 
              activate zoral_external 

                note right of zoral_external
                  Set input.cifId = <strike> <color red> cif[].userId </color> </strike> cif[].cifId and input.needCustomerReg = false **(LPS-13559)**
                end note

              zoral_external--> zoral : response  cifRelatePerson
              deactivate zoral_external

                alt If got any endpoint's error
                  note right of zoral 
                       Hard stop. To return status.code = 9001
                  end note
               end alt

              alt if any zoral sub process error
                note right of zoral
                  Hard stop and return status.code regarding EXT_cbs_personal_profile_inquiry
                end note
              end alt

              group Prepare  guarantorAll 
                note right of zoral
                  Count all guarantor by:
                     - data.contractInfo.collaterals[].collateral.cif[].relatedType ="guarantor" then keep number of all guarantor into  guarantorAll
                end note 
              end group

              alt IF cif[].relatedType="owner" and cifRelatePerson.customerDetail.dateOfDeath have value
                  note right of zoral 
                          -  Hard stop. To return status.code = 8018
                  end note
             

              else ELSE IF contractInfo.collateralType  != 1301 and cif[].relatedType="guarantor"  and cifRelatePerson.customerDetail.dateOfDeath have value
                note right of zoral 
                              -  Hard stop. To return status.code = 8019
                      end note
              

              else ELSE IF contractInfo.collateralType  = 1301  and cif[].relatedType="guarantor"  and cifRelatePerson.customerDetail.dateOfDeath have value

                 note right of zoral 
                      - Add guarantorDeath= guarantorDeath+1
                      - Set  death detail
                                cifId = cifRelatePerson.customerRegistrationDetail.customerId
                                customerTitle = cifRelatePerson.customerDetail.customerTitle 
                                customerFirstName = cifRelatePerson.customerDetail.customerFirstName 
                                customerLastName = cifRelatePerson.customerDetail.customerLastName  
                      - Add  death detail into guarantorDeath[]
                 end note
            end alt
             
      end alt
    end loop
 
    alt IF contractInfo.collateralType  = 1301  and  guarantorDeath > 0
      note right of zoral 
                - guarantorStillAlive = guarantorAll -  guarantorDeath 
     end note
      alt IF guarantorStillAlive < guarantorAll and  guarantorStillAlive >= 5 

       note right of zoral 
        - set status.code = 8021
      end note
      else Else  If guarantorStillAlive <  5
        note right of zoral 
                 -  Hard stop. To return status.code = 8020
       end note

      end alt
      
    end alt

 end loop
  end loop
end group

== Deposit Verify (ตรวจสอบบัญชีเงินฝาก)==

group  Deposit Verify for requester

  zoral -> zoral_external : Call INT_DBM_account_list_inquiry 
    activate zoral_external 

          note right of zoral_external 
            Set cifId = input.cifId  
                  accountType = 'LOAN_ACCOUNT' 
                  accountClass = "D"
          end note

    zoral_external--> zoral : response  data.accountList[]
    deactivate zoral_external

               alt If got any endpoint's error
                  note right of zoral 
                       Hard stop. To return status.code = 9100
                  end note
               end alt

  alt if  Not found any item of accountList[]

      note right of zoral 
             - Hard stop. To return status.code = 8022
        end note
  end alt
end group

== Customer Information Verify (ตรวจสอบข้อมูลลูกค้า)==

group  Customer Information Verify  for requester

 zoral -> zoral_internal : Call INT_DBM_customer_Information_verify 
    activate zoral_internal 

          note right of zoral_internal 
            Set cifId = input.cifId 
          end note

    zoral_internal--> zoral : response  custInfoVerify
    deactivate zoral_internal

    
               alt If got any endpoint's error
                  note right of zoral 
                       Hard stop. To return status.code = 9100
                  end note
               end alt

 alt If custInfoVerify.hasCustomerInfo = false
      note right of zoral 
             - Hard stop. To return status.code = 8027
        end note
  end alt
end group

== Publish Event Log ==
alt  if status.code IN [1000, 8021]
zoral -> zoral_internal: Call INT_audit_event_publish to <b>Publish Event Code 3000A (DBM_PRE_CONDITION)
activate zoral_internal
zoral_internal --> zoral: Return response
deactivate zoral_internal
alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
end alt
else #lightgreen Else, if status.code IN [<color #orange><b>8002</b></color>, 8004, 8012, 8013, 8014, 8015, 8016, 8017, 8018, 8019, 8020, 8022, 8027, 8032, 8061, 8094, 8095, 8096, 8097, 8098]
zoral -> zoral_internal: Call INT_audit_event_publish to <b>Publish Event Code 3000B (DBM_PRE_CONDITION)
activate zoral_internal
zoral_internal --> zoral: Return response
deactivate zoral_internal
alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
end alt
else Else
zoral -> zoral_internal: Call INT_audit_event_publish to <b>Publish Event Code 3000C (DBM_PRE_CONDITION)
activate zoral_internal
zoral_internal --> zoral: Return response
deactivate zoral_internal
alt if any connection error
        note right of zoral
          Hard stop and return status.code  9100
        end note
end alt
end alt

zoral --> Requestor: return response
deactivate zoral 

@enduml
```

## Reusable Guideline For Next Files

- ใช้ example ด้านบนเป็น style reference ไม่จำเป็นต้องคัดทุกบรรทัด
- ก่อนเขียน diagram ทุกครั้ง ให้ lookup จากทั้ง `.xml` และ `.xml.layout` ที่ชื่อไฟล์ตรงกันก่อน
- เริ่มจากแตก inventory ของ `SubProcessTask`, `Gateway`, `ScriptTask`, `ComponentTask`
- แยก participant อย่างน้อยเป็น `Requestor`, `zoral`, `zoral_internal`, `zoral_external`, `database`
- ถ้า step เป็น external service หลายตัว แยกชื่อ participant ตาม `WorkflowName` ได้ แต่ยังจัดอยู่ใต้กลุ่ม `zoral_external`
- ใช้ชื่อจาก XML จริงก่อน แล้วค่อยเติม business label ใน note ถ้าจำเป็น
- ถ้าเห็น gateway ชื่อขึ้นต้น `Loop` ให้พิจารณา map เป็น `loop`
- ถ้าเห็น gateway ชื่อขึ้นต้น `Validate` หรือ `Check` ให้พิจารณา map เป็น `alt`
- ถ้าเห็น script ชื่อขึ้นต้น `Pre` หรือ `Post` ให้พิจารณาเขียนเป็น note หรือ processing step มากกว่าจะเปิด participant ใหม่
- ถ้าเป็น `AdwQuery` ให้ระบุ table / intent ที่ query ในข้อความลูกศรหรือ note
- ถ้าเป็น error flow ให้สรุปเป็น hard stop note พร้อม `status.code` แทนการวาดทุก error script แบบละเอียดเสมอไป
- ถ้ามี audit step ช่วงท้าย ให้ระบุ event publish logic แยกเป็นอีก group หนึ่งเสมอ
- พยายามรักษา diagram ให้อ่านเป็น business flow ก่อน แล้วค่อยเติม detail เชิงเทคนิค

## Reusable Output Structure

- `Requirement Notes`
- `Source Files`
- `XML To PlantUML Mapping Rule`
- `Lookup Summary From XML And XML.layout`
- `Example PlantUML`
- `Reusable Guideline For Next Files`

## Quick Authoring Checklist

- ชื่อไฟล์ `.xml` และ `.xml.layout` ต้องเป็นคู่เดียวกัน
- lookup `WorkflowName` ของทุก `SubProcessTask` ก่อนเขียน diagram
- ใช้ `.xml.layout` เพื่อเช็กทางแยกและ loop
- ใช้ `activate` / `deactivate` ให้ครบ
- ใช้ `->` สำหรับ request และ `-->` สำหรับ response
- ถ้ามี hard stop หลายจุด ให้รวมเป็น pattern note ได้
- ตรวจว่ามี path success, validation error, endpoint error, global error ครบหรือไม่
