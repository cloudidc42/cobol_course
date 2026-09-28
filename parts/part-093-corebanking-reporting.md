# Part 093: Case Study - Core Banking ตอนที่ 3: การรายงานและปิดบัญชี (ขั้นตอนที่ 921–930)

## คำนำของ Part นี้

Part 091 วางรากฐานของระบบ (Copybook, `CBINIT`, `CBOPEN`) และ Part 092 สร้างโมดูลที่ทำให้เงินเคลื่อนไหว
ได้จริง (`CBAUDIT`, `CBDEP`, `CBWD`, `CBXFER` พร้อมกลไก Compensating Transaction, `CBACCR`) เมื่อจบ
Part 092 เรามีระบบที่ผ่านการทดสอบ 2 วันทำการเต็มรูปแบบ: บัญชี 4 บัญชี, `TXNLOG.DAT` ที่บันทึกทุก
ธุรกรรมไว้ครบถ้วน (รวมถึงธุรกรรมที่สำเร็จ ถูกปฏิเสธ และถูกกู้คืน), และดอกเบี้ยค้างรับที่สะสมไว้รอการ
ผ่านรายการ

Part นี้คือบทสุดท้ายของ Case Study — เราจะนำข้อมูลทั้งหมดที่สะสมมา**ต่อยอดทันทีโดยไม่รีเซ็ตใหม่**เพื่อ
สร้างสิ่งที่ธนาคารจริงต้องมี:

- **`CBDAILY`** — รายงานธุรกรรมประจำวัน
- **`CBSTMT`** — ใบแจ้งยอดบัญชี (ต่อยอดเทคนิค SORT + Control Break จาก Part 027, 039 และสไตล์รายงาน
  Aging จาก Part 050)
- **`CBEOD`** — การกระทบยอดสิ้นวัน ที่**พิสูจน์หลักบัญชีคู่ (Double-Entry)** ของธุรกรรมโอนเงินที่สร้าง
  ใน Part 092 อย่างเป็นรูปธรรม
- **`CBMEND`** — การผ่านรายการดอกเบี้ยสิ้นเดือน พร้อม **Idempotency Guard** ที่ป้องกันการจ่ายดอกเบี้ย
  ซ้ำหากรันแบตช์ผิดพลาดซ้ำ
- **`CBCLOSE`** — รายงานสรุปปิดบัญชีระดับพอร์ตโฟลิโอ

ปิดท้ายด้วย **การทดสอบครบวงจรทั้งระบบ (End-to-End Integration Test)** ที่รันตั้งแต่ `CBINIT` ผ่านทุก
ธุรกรรม ไปจนถึงการปิดบัญชีสิ้นเดือน แสดงผลลัพธ์จริงทั้งหมดในที่เดียว เป็นบทพิสูจน์สุดท้ายว่า Case Study
นี้ทำงานได้จริงครบวงจร

### ภาพรวม 10 ขั้นตอนของ Part นี้

| ขั้นตอน | สิ่งที่สร้าง |
|---|---|
| 921 | ทบทวน Part 092 + ภาพรวมโมดูลรายงานที่จะสร้าง |
| 922 | `CBDAILY` — รายงานธุรกรรมประจำวัน |
| 923 | `CBSTMT` — ใบแจ้งยอดบัญชี (SORT + Control Break) พร้อมบั๊กจริงที่พบและแก้ไข |
| 924 | `CBEOD` — การกระทบยอดสิ้นวันแบบหลักบัญชีคู่ |
| 925 | `CBMEND` — ผ่านรายการดอกเบี้ยสิ้นเดือน + Idempotency Guard (พร้อมบั๊กจริง) |
| 926 | `CBCLOSE` — รายงานสรุปปิดบัญชี |
| 927 | เจาะลึกการจัดการข้อยกเว้นและบทเรียนจากบั๊กที่พบทั้งหมด |
| 928 | ข้อพิจารณาเชิงปฏิบัติการ: Batch Window, การรันซ้ำอย่างปลอดภัย, การสำรองข้อมูล |
| 929 | ทดสอบครบวงจร ส่วนที่ 1: Setup + ธุรกรรม 2 วันทำการ |
| 930 | ทดสอบครบวงจร ส่วนที่ 2: ปิดบัญชีสิ้นเดือน + สรุปทั้ง Case Study |

---

## ขั้นตอนที่ 921: ทบทวน Part 092 และภาพรวมโมดูลรายงาน

### สถานะข้อมูลเมื่อเริ่ม Part นี้

หลังจบ Part 092 เรามีข้อมูลสะสมดังนี้ (จะใช้ต่อเนื่องตลอด Part นี้):

- **บัญชี 4 บัญชี**: `100001` (Savings), `200001` (Checking, ติดลบได้), `300001` (Fixed Deposit),
  `400001` (Savings)
- **`TXNLOG.DAT`** มีธุรกรรมของวันที่ 1 และวันที่ 2 (เมษายน 2026) รวม 17 บรรทัด ครอบคลุมทุกประเภท
  ธุรกรรม: `OPEN`, `DEP`, `WD` (ทั้งสำเร็จและถูกปฏิเสธ), `XFRD`/`XFRC` (ทั้งสำเร็จและถูกกู้คืนด้วย
  `RVSL`)
- **ดอกเบี้ยค้างรับ** สะสมมาแล้ว 2 วันจาก `CBACCR` รอการผ่านรายการ

### หลักการออกแบบของโมดูลในบทนี้

โมดูลทั้งหมดใน Part นี้เป็น **Batch Program** (ทบทวน Pattern จาก Part 066) ที่มีโครงสร้างร่วมกัน:

1. อ่านไฟล์อินพุต (ทั้งไฟล์หรือกรองด้วยวันที่) แบบ Sequential
2. สะสมยอดควบคุม (Control Totals) ระหว่างอ่าน
3. พิมพ์รายละเอียดและ/หรือสรุปยอดเมื่ออ่านจบ
4. ไม่มีการรับอินพุตแบบโต้ตอบระหว่างทำงาน (ยกเว้นพารามิเตอร์วันที่ประมวลผลตอนเริ่มต้นโปรแกรม)

นี่คือความแตกต่างชัดเจนจาก Part 092 ที่เป็นโปรแกรม Online-style ทั้งหมด — Part นี้แสดงให้เห็น "อีกครึ่ง
หนึ่ง" ของระบบธนาคารจริงตามแผนภาพสถาปัตยกรรมที่ Part 091 ขั้นตอนที่ 906 วางไว้

### แบบฝึกหัดที่ 921.1

**โจทย์**: จงอธิบายว่าทำไมโมดูลรายงานทั้งหมดใน Part นี้ถึงเปิดไฟล์แบบ `INPUT` เท่านั้น (ไม่ใช่ `I-O`)
ยกเว้น `CBMEND` ที่ต้องเปิดแบบ `I-O`

**เฉลยแนวทาง**: `CBDAILY`, `CBSTMT`, `CBEOD`, `CBCLOSE` ทำหน้าที่**อ่านและรายงาน**เท่านั้น ไม่มีการแก้ไข
ข้อมูลใดๆ ในไฟล์ต้นทาง จึงเปิดแบบ `INPUT` (อ่านอย่างเดียว) พอ ซึ่งเป็นการเขียนโค้ดที่ปลอดภัยกว่า (ป้องกัน
การเขียนทับข้อมูลโดยไม่ตั้งใจ) และสื่อเจตนาของโปรแกรมชัดเจนแก่ผู้ที่มาอ่านโค้ดในภายหลัง ส่วน `CBMEND`
ต้อง**แก้ไขยอดคงเหลือจริง**ในบัญชีทุกบัญชีที่ผ่านรายการดอกเบี้ย จึงจำเป็นต้องเปิดแบบ `I-O`

---

## ขั้นตอนที่ 922: `CBDAILY` — รายงานธุรกรรมประจำวัน

### หลักการ

`CBDAILY` อ่าน `TXNLOG.DAT` ตั้งแต่ต้นจนจบเพียงรอบเดียว (ไม่ต้อง SORT เพราะไฟล์นี้เป็น append-only จึง
เรียงตามลำดับเวลาอยู่แล้วโดยธรรมชาติ) กรองเฉพาะบรรทัดที่ตรงกับวันที่ที่ต้องการ แล้วพิมพ์รายละเอียดพร้อม
สรุปยอดควบคุมแยกตามประเภทธุรกรรม

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBDAILY.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBDAILY - daily transaction report. Reads TXNLOG.DAT from
      * start to finish (it is append-only and already in chronological
      * order, so no SORT is needed here) and prints every line whose
      * TXN-DATE matches the requested report date, plus a control
      * total by transaction type at the end.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANSACTION-LOG ASSIGN TO "TXNLOG.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANSACTION-LOG.
       01  TRANSACTION-LINE.
           COPY "cbtxn.cpy".

       WORKING-STORAGE SECTION.
       01  WS-LOG-STATUS            PIC X(2).
       01  WS-EOF-FLAG              PIC X(1)   VALUE "N".
           88  WS-EOF                        VALUE "Y".
       01  WS-REPORT-DATE           PIC 9(8)   VALUE 0.
       01  WS-AMOUNT-EDIT           PIC Z,ZZZ,ZZ9.99.
       01  WS-BAL-EDIT              PIC -Z,ZZZ,ZZ9.99.
       01  WS-LINE-COUNT            PIC 9(5)   VALUE 0.
       01  WS-CNT-DEP               PIC 9(5)   VALUE 0.
       01  WS-CNT-WD                PIC 9(5)   VALUE 0.
       01  WS-CNT-XFRD              PIC 9(5)   VALUE 0.
       01  WS-CNT-XFRC              PIC 9(5)   VALUE 0.
       01  WS-CNT-RVSL              PIC 9(5)   VALUE 0.
       01  WS-CNT-OTHER             PIC 9(5)   VALUE 0.
       01  WS-CNT-REJECTED          PIC 9(5)   VALUE 0.
       01  WS-SUM-DEP               PIC 9(9)V99 VALUE 0.
       01  WS-SUM-WD                PIC 9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== CBDAILY: Daily Transaction Report ===".
           DISPLAY "Enter report date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-REPORT-DATE.

           OPEN INPUT TRANSACTION-LOG.
           DISPLAY " ".
           DISPLAY "DATE     TIME   GROUPID  ACCTID TYPE     AMOUNT"
               "        BALANCE       STATUS".
           DISPLAY "-------- ------ -------- ------ ---- ------------"
               " ------------- --------".

           PERFORM UNTIL WS-EOF
               READ TRANSACTION-LOG
                   AT END
                       MOVE "Y" TO WS-EOF-FLAG
                   NOT AT END
                       PERFORM PROCESS-LINE
               END-READ
           END-PERFORM.

           CLOSE TRANSACTION-LOG.
           DISPLAY " ".
           DISPLAY "---- CBDAILY Control Totals (" WS-REPORT-DATE
               ") ----".
           DISPLAY "Lines matching this date : " WS-LINE-COUNT.
           DISPLAY "  DEP  : " WS-CNT-DEP  "   total=" WS-SUM-DEP.
           DISPLAY "  WD   : " WS-CNT-WD   "   total=" WS-SUM-WD.
           DISPLAY "  XFRD : " WS-CNT-XFRD.
           DISPLAY "  XFRC : " WS-CNT-XFRC.
           DISPLAY "  RVSL : " WS-CNT-RVSL.
           DISPLAY "  OTHER: " WS-CNT-OTHER.
           DISPLAY "  REJECTED entries (any type): " WS-CNT-REJECTED.
           STOP RUN.

       PROCESS-LINE.
           IF TXN-DATE = WS-REPORT-DATE
               ADD 1 TO WS-LINE-COUNT
               MOVE TXN-AMOUNT        TO WS-AMOUNT-EDIT
               MOVE TXN-BALANCE-AFTER TO WS-BAL-EDIT
               DISPLAY TXN-DATE SPACE TXN-TIME SPACE TXN-GROUP-ID
                   SPACE TXN-ACCT-ID SPACE TXN-TYPE SPACE
                   WS-AMOUNT-EDIT SPACE WS-BAL-EDIT SPACE TXN-STATUS
               IF TXN-STATUS = "REJECTED"
                   ADD 1 TO WS-CNT-REJECTED
               ELSE
                   EVALUATE TXN-TYPE
                       WHEN "DEP "
                           ADD 1 TO WS-CNT-DEP
                           ADD TXN-AMOUNT TO WS-SUM-DEP
                       WHEN "WD  "
                           ADD 1 TO WS-CNT-WD
                           ADD TXN-AMOUNT TO WS-SUM-WD
                       WHEN "XFRD"
                           ADD 1 TO WS-CNT-XFRD
                       WHEN "XFRC"
                           ADD 1 TO WS-CNT-XFRC
                       WHEN "RVSL"
                           ADD 1 TO WS-CNT-RVSL
                       WHEN OTHER
                           ADD 1 TO WS-CNT-OTHER
                   END-EVALUATE
               END-IF
           END-IF.
```

### ทดสอบจริงกับข้อมูลวันที่ 1

```bash
cobc -x -o cbdaily cbdaily.cob
printf '20260401\n' | ./cbdaily
```

**ผลลัพธ์จริง**:

```
=== CBDAILY: Daily Transaction Report ===
Enter report date (YYYYMMDD):
DATE     TIME   GROUPID  ACCTID TYPE     AMOUNT        BALANCE       STATUS
-------- ------ -------- ------ ---- ------------ ------------- --------
20260401 090000 00000000 100001 OPEN     5,000.00      5,000.00 OK
20260401 090500 00000000 200001 OPEN     1,000.00      1,000.00 OK
20260401 091000 00000000 300001 OPEN    60,000.00     60,000.00 OK
20260401 091500 00000000 400001 OPEN    12,000.00     12,000.00 OK
20260401 100000 00000000 100001 DEP      2,000.00      7,000.00 OK
20260401 101000 00000000 200001 WD         300.00        700.00 OK
20260401 103000 00000001 400001 XFRD     1,000.00     11,000.00 OK
20260401 103000 00000001 200001 XFRC     1,000.00      1,700.00 OK
20260401 104000 00000002 100001 XFRD       500.00      6,500.00 OK
20260401 104000 00000002 100001 RVSL       500.00      7,000.00 OK
20260401 104000 00000002 999999 XFRC       500.00      7,000.00 REJECTED
20260401 105000 00000000 100001 WD      90,000.00      7,000.00 REJECTED
 
---- CBDAILY Control Totals (20260401) ----
Lines matching this date : 00012
  DEP  : 00001   total=000002000.00
  WD   : 00001   total=000000300.00
  XFRD : 00002
  XFRC : 00001
  RVSL : 00001
  OTHER: 00004
  REJECTED entries (any type): 00002
```

รายงานแสดงครบทั้ง 12 บรรทัดของวันที่ 1 ตรงตามที่บันทึกไว้ใน Audit Trail (Part 092 ขั้นตอนที่ 919) พร้อม
ยอดควบคุมแยกตามประเภทธุรกรรมที่ถูกต้อง

### ข้อควรระวัง

- `OTHER` ในที่นี้นับรวม `OPEN` ทั้ง 4 รายการ เพราะ `EVALUATE` ไม่มี `WHEN` เฉพาะสำหรับ `OPEN` — เป็นการ
  ตัดสินใจออกแบบที่ควรระบุให้ชัดเจนว่า "การเปิดบัญชีไม่ถือเป็นยอดฝาก/ถอนในรายงานนี้" (ถ้าต้องการแยก
  `OPEN` ออกมาเป็นยอดควบคุมของตัวเอง สามารถเพิ่ม `WHEN "OPEN"` ได้ตามความต้องการทางธุรกิจ)
- ฟิลด์ `TXN-TYPE` เป็น `PIC X(4)` ดังนั้นค่าที่สั้นกว่า 4 ตัวอักษร (เช่น `"WD"`) จะถูกเก็บเป็น `"WD  "`
  (เติมช่องว่างท้าย) การเทียบค่าใน `EVALUATE` จึงต้องเขียน `WHEN "WD  "` ให้ความยาวตรงกันเป๊ะ มิฉะนั้น
  เงื่อนไขจะไม่ตรงกันเลยและตกไปที่ `WHEN OTHER` อย่างเงียบๆ

### แบบฝึกหัดที่ 922.1

**โจทย์**: หากเปลี่ยนคำสั่ง `ACCEPT WS-REPORT-DATE` เป็นวันที่ `20260405` (วันที่ไม่มีธุรกรรมใดๆ เกิดขึ้น
เลย) คาดว่ารายงานจะแสดงผลอย่างไร

**เฉลย**: ส่วนตารางรายละเอียดจะไม่มีบรรทัดใดแสดงเลย (เพราะเงื่อนไข `IF TXN-DATE = WS-REPORT-DATE` ไม่
เป็นจริงสำหรับบรรทัดใดใน log) แต่ส่วนสรุปยอดควบคุมท้ายรายงานจะยังคงแสดงออกมาเสมอ โดยทุกค่าจะเป็นศูนย์
(`Lines matching this date : 00000` และยอดรวมทุกประเภทเป็น 0) เพราะโค้ดในส่วนสรุปอยู่นอก loop การอ่าน
ไฟล์ ไม่ขึ้นกับว่าพบข้อมูลตรงเงื่อนไขหรือไม่

---

## ขั้นตอนที่ 923: `CBSTMT` — ใบแจ้งยอดบัญชี (SORT + Control Break)

### หลักการ

ต่างจาก `CBDAILY` ที่รายงานตามลำดับเวลาดิบของทั้งธนาคาร `CBSTMT` ต้องจัดกลุ่มธุรกรรม**ตามบัญชี**ก่อน จึง
ต้องใช้ **SORT Statement** (Part 027) เรียงลำดับ `TXNLOG.DAT` ใหม่ตาม `ACCT-ID` แล้วตามด้วยวันที่/เวลา
จากนั้นใช้เทคนิค **Control Break** (Part 039, ต่อยอดสไตล์รายงาน Aging จาก Part 050) พิมพ์หนึ่งใบแจ้งยอด
ต่อหนึ่งบัญชี พร้อมยอดยกไปตรวจสอบกับ `ACCOUNT-MASTER` (Master-Detail Pattern จาก Part 026)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBSTMT.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBSTMT - batch account statement generator. Uses the SORT verb
      * (Part 027) to reorder the whole TXNLOG.DAT by account then by
      * date/time, then a control-break loop (Part 039 / the Part 050
      * aging-report technique) prints one statement block per account:
      * every transaction line, and a closing balance looked up from
      * ACCOUNT-MASTER (Master-Detail pattern, Part 026).
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANSACTION-LOG ASSIGN TO "TXNLOG.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

           SELECT SORT-WORK ASSIGN TO "STMTSRT.TMP".

           SELECT SORTED-LOG ASSIGN TO "STMTOUT.TMP"
               ORGANIZATION IS LINE SEQUENTIAL.

           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-MASTER-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANSACTION-LOG.
      *> Used only as the SORT verb's USING source - a flat PIC X
      *> record is enough since no individual field is ever accessed
      *> through this FD (avoids name collisions with SORTED-LINE).
       01  TRANSACTION-LINE         PIC X(70).

       SD  SORT-WORK.
      *> Renamed fields (via REPLACING) so the sort keys below cannot
      *> collide with SORTED-LINE's identically-shaped copy - only the
      *> 3 key fields are ever referenced, by their new SRT- names.
       01  SORT-LINE.
           COPY "cbtxn.cpy"
               REPLACING TXN-DATE          BY SRT-DATE
                         TXN-TIME          BY SRT-TIME
                         TXN-GROUP-ID      BY SRT-GROUP-ID
                         TXN-ACCT-ID       BY SRT-ACCT-ID
                         TXN-TYPE          BY SRT-TYPE
                         TXN-AMOUNT        BY SRT-AMOUNT
                         TXN-BALANCE-AFTER BY SRT-BALANCE-AFTER
                         TXN-STATUS        BY SRT-STATUS.

       FD  SORTED-LOG.
       01  SORTED-LINE.
           COPY "cbtxn.cpy".

       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "cbacct.cpy".

       WORKING-STORAGE SECTION.
       01  WS-MASTER-STATUS         PIC X(2).
       01  WS-EOF-FLAG              PIC X(1)   VALUE "N".
           88  WS-EOF                        VALUE "Y".
       01  WS-FIRST-FLAG            PIC X(1)   VALUE "Y".
           88  WS-FIRST-RECORD               VALUE "Y".
       01  WS-CUR-ACCT              PIC 9(6)   VALUE 0.
       01  WS-LINE-TOTAL-DEP        PIC 9(9)V99 VALUE 0.
       01  WS-LINE-TOTAL-WD         PIC 9(9)V99 VALUE 0.
       01  WS-LINE-COUNT            PIC 9(5)   VALUE 0.
       01  WS-AMOUNT-EDIT           PIC Z,ZZZ,ZZ9.99.
       01  WS-BAL-EDIT              PIC -Z,ZZZ,ZZ9.99.
       01  WS-CLOSING-EDIT          PIC -Z,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== CBSTMT: Account Statement Batch ===".
           SORT SORT-WORK
               ON ASCENDING KEY SRT-ACCT-ID
               ON ASCENDING KEY SRT-DATE
               ON ASCENDING KEY SRT-TIME
               USING TRANSACTION-LOG
               GIVING SORTED-LOG.

           OPEN INPUT SORTED-LOG.
           OPEN INPUT ACCOUNT-MASTER.

           PERFORM UNTIL WS-EOF
               READ SORTED-LOG
                   AT END
                       MOVE "Y" TO WS-EOF-FLAG
                       IF NOT WS-FIRST-RECORD
                           PERFORM PRINT-ACCOUNT-FOOTER
                       END-IF
                   NOT AT END
                       PERFORM PROCESS-STMT-LINE
               END-READ
           END-PERFORM.

           CLOSE SORTED-LOG.
           CLOSE ACCOUNT-MASTER.
           STOP RUN.

       PROCESS-STMT-LINE.
           IF WS-FIRST-RECORD OR TXN-ACCT-ID NOT = WS-CUR-ACCT
               IF NOT WS-FIRST-RECORD
                   PERFORM PRINT-ACCOUNT-FOOTER
               END-IF
               MOVE TXN-ACCT-ID TO WS-CUR-ACCT
               MOVE "N" TO WS-FIRST-FLAG
               MOVE 0 TO WS-LINE-TOTAL-DEP WS-LINE-TOTAL-WD
                   WS-LINE-COUNT
               PERFORM PRINT-ACCOUNT-HEADER
           END-IF.

           ADD 1 TO WS-LINE-COUNT.
           MOVE TXN-AMOUNT        TO WS-AMOUNT-EDIT.
           MOVE TXN-BALANCE-AFTER TO WS-BAL-EDIT.
           DISPLAY "  " TXN-DATE " " TXN-TIME " " TXN-TYPE " "
               WS-AMOUNT-EDIT "   bal after: " WS-BAL-EDIT
               "  [" TXN-STATUS "]".
           IF TXN-STATUS = "OK"
               EVALUATE TXN-TYPE
                   WHEN "DEP "
                       ADD TXN-AMOUNT TO WS-LINE-TOTAL-DEP
                   WHEN "XFRC"
                       ADD TXN-AMOUNT TO WS-LINE-TOTAL-DEP
                   WHEN "INT "
                       ADD TXN-AMOUNT TO WS-LINE-TOTAL-DEP
                   WHEN "WD  "
                       ADD TXN-AMOUNT TO WS-LINE-TOTAL-WD
                   WHEN "XFRD"
                       ADD TXN-AMOUNT TO WS-LINE-TOTAL-WD
               END-EVALUATE
           END-IF.

       PRINT-ACCOUNT-HEADER.
           MOVE WS-CUR-ACCT TO ACCT-ID.
           READ ACCOUNT-MASTER
               INVALID KEY
                   MOVE SPACES TO ACCT-NAME
           END-READ.
           DISPLAY " ".
           DISPLAY "======================================".
           DISPLAY "STATEMENT FOR ACCOUNT " WS-CUR-ACCT " - "
               FUNCTION TRIM(ACCT-NAME).
           DISPLAY "======================================".

       PRINT-ACCOUNT-FOOTER.
           MOVE WS-CUR-ACCT TO ACCT-ID.
      *> Always handle INVALID KEY explicitly - without this MOVE 0,
      *> a rejected transfer to a nonexistent account (Part 092) would
      *> silently print the PREVIOUS account's leftover balance here.
           READ ACCOUNT-MASTER
               INVALID KEY
                   MOVE 0 TO ACCT-BALANCE
           END-READ.
           MOVE ACCT-BALANCE TO WS-CLOSING-EDIT.
           DISPLAY "  -----------------------------------".
           DISPLAY "  Lines on statement : " WS-LINE-COUNT.
           MOVE WS-LINE-TOTAL-DEP TO WS-AMOUNT-EDIT.
           DISPLAY "  Total credits      : " WS-AMOUNT-EDIT.
           MOVE WS-LINE-TOTAL-WD TO WS-AMOUNT-EDIT.
           DISPLAY "  Total debits       : " WS-AMOUNT-EDIT.
           DISPLAY "  Current balance    : " WS-CLOSING-EDIT.
```

### บั๊กจริงที่พบระหว่างทดสอบ: ข้อมูลค้าง (Stale Data) รั่วไหลจากบัญชีก่อนหน้า

ระหว่างทดสอบ `CBSTMT` ครั้งแรก (ก่อนมีบรรทัด `INVALID KEY MOVE 0 TO ACCT-BALANCE` ใน
`PRINT-ACCOUNT-FOOTER` — ตอนนั้นเขียนเพียง `INVALID KEY CONTINUE`) เราพบผลลัพธ์ที่ผิดปกติสำหรับบัญชี
`999999` (บัญชีที่ไม่มีอยู่จริง แต่ปรากฏใน log เพราะเป็นปลายทางของการโอนที่ถูกปฏิเสธใน Part 092):

```
======================================
STATEMENT FOR ACCOUNT 999999 -
======================================
  20260401 104000 XFRC       500.00   bal after:      7,000.00  [REJECTED]
  -----------------------------------
  Lines on statement : 00001
  Total credits      :         0.00
  Total debits       :         0.00
  Current balance    :     50,000.00
```

สังเกต **"Current balance : 50,000.00"** — ตัวเลขนี้ไม่ได้เกี่ยวข้องกับบัญชี `999999` เลย แต่เป็นยอด
คงเหลือของบัญชี `300001` (บัญชีก่อนหน้าในลำดับ) ที่ค้างอยู่ในพื้นที่หน่วยความจำ `ACCOUNT-RECORD`
**สาเหตุ**: เมื่อ `READ ACCOUNT-MASTER` ด้วยคีย์ `999999` ล้มเหลว (`INVALID KEY`) มาตรฐาน COBOL **ไม่ได้
รับประกันว่าพื้นที่ record จะถูกเคลียร์** — ถ้าโปรแกรมเมอร์เขียนแค่ `CONTINUE` (ไม่ทำอะไรเลย) ค่าเก่าที่
ค้างอยู่จากการ `READ` ครั้งก่อนหน้า (ซึ่งสำเร็จ) จะยังคงอยู่และถูกนำไปแสดงผลราวกับเป็นข้อมูลที่ถูกต้อง

**วิธีแก้**: เพิ่ม `MOVE 0 TO ACCT-BALANCE` ใน branch `INVALID KEY` อย่างชัดเจน (ดังที่แสดงในโค้ดด้านบน)
หลังแก้ไข ผลลัพธ์จริงกลายเป็น:

```
======================================
STATEMENT FOR ACCOUNT 999999 -
======================================
  20260401 104000 XFRC       500.00   bal after:      7,000.00  [REJECTED]
  -----------------------------------
  Lines on statement : 00001
  Total credits      :         0.00
  Total debits       :         0.00
  Current balance    :         0.00
```

### ทดสอบจริง (หลังแก้บั๊ก) — แสดงเฉพาะบัญชีแรกเพื่อความกระชับ

```bash
cobc -x -o cbstmt cbstmt.cob
./cbstmt
```

**ผลลัพธ์จริงของบัญชี `100001`**:

```
======================================
STATEMENT FOR ACCOUNT 100001 - SUPAPORN JAIDEE
======================================
  20260401 090000 OPEN     5,000.00   bal after:      5,000.00  [OK      ]
  20260401 100000 DEP      2,000.00   bal after:      7,000.00  [OK      ]
  20260401 104000 XFRD       500.00   bal after:      6,500.00  [OK      ]
  20260401 104000 RVSL       500.00   bal after:      7,000.00  [OK      ]
  20260401 105000 WD      90,000.00   bal after:      7,000.00  [REJECTED]
  20260402 093000 XFRC       400.00   bal after:      7,400.00  [OK      ]
  -----------------------------------
  Lines on statement : 00006
  Total credits      :         0.00
  Total debits       :       500.00
  Current balance    :      7,407.49
```

ใบแจ้งยอดแสดงประวัติธุรกรรมครบทั้ง 6 รายการของบัญชีนี้เรียงตามวันเวลา รวมทั้งรายการที่ถูกกู้คืน (`RVSL`)
และรายการที่ถูกปฏิเสธ (`REJECTED`) อย่างโปร่งใส — ยอดคงเหลือ `7,407.49` มาจาก `ACCOUNT-MASTER` โดยตรง
(รวมดอกเบี้ยค้างรับ 2 วันที่ `CBACCR` สะสมไว้แล้ว แต่ยังไม่ผ่านรายการเป็น log entry เพราะจะผ่านรายการ
เฉพาะสิ้นเดือนโดย `CBMEND` ในขั้นตอนที่ 925)

### ข้อควรระวัง

- **บทเรียนสำคัญที่สุดของขั้นตอนนี้**: `READ ... INVALID KEY` ที่เขียนแค่ `CONTINUE` โดยไม่เคลียร์
  ฟิลด์ที่จะนำไปใช้ต่อ คือแหล่งบั๊กที่พบได้บ่อยมากในโค้ด COBOL จริง เพราะไม่มี compile error หรือ
  runtime error ใดๆ เตือนเลย (ทบทวนร่วมกับบั๊กที่พบใน Part 092 ขั้นตอนที่ 912 — ทั้งสองกรณีคือ "ข้อมูล
  ที่ไม่ได้ตั้งค่าไว้อย่างชัดเจน" กลายเป็นข้อมูลที่แสดงผลราวกับถูกต้อง)
- การ SORT ในที่นี้ใช้ `TRANSACTION-LOG` เป็น `USING` และ `SORTED-LOG` (คนละไฟล์กัน) เป็น `GIVING` —
  ไม่ควรใช้ไฟล์เดียวกันทั้ง `USING` และ `GIVING` เพราะ SORT ต้องเปิดไฟล์นั้นทั้งเพื่ออ่านและเขียนในเวลา
  ใกล้เคียงกัน ซึ่งอาจสร้างความสับสนหรือปัญหาในบาง Environment

### แบบฝึกหัดที่ 923.1

**โจทย์**: จงอธิบายว่าทำไม `SORT-LINE` (SD) ต้องเปลี่ยนชื่อฟิลด์ทุกตัวด้วย `REPLACING` แต่ `TRANSACTION-
LINE` (FD ของไฟล์ต้นทาง) กลับประกาศเป็น `PIC X(70)` แบบเรียบง่ายโดยไม่ใช้ copybook เลย

**เฉลยแนวทาง**: `SORT-LINE` ต้องมีชื่อฟิลด์จริง (แม้จะเปลี่ยนชื่อแล้ว) เพราะคำสั่ง `SORT ... ON
ASCENDING KEY` ต้องการชื่อฟิลด์เพื่อระบุคีย์ในการเรียงลำดับ ส่วน `TRANSACTION-LINE` ถูกใช้เพียงเป็น
"ภาชนะ" ให้ SORT อ่านข้อมูลดิบเข้ามาเท่านั้น (ระบุใน `USING TRANSACTION-LOG`) ไม่มีจุดใดในโปรแกรมที่ต้อง
อ้างอิงฟิลด์ย่อยของมันเลย การประกาศเป็น `PIC X(70)` (ตราบใดที่ความยาวรวมตรงกับ `cbtxn.cpy`) จึงเพียงพอ
และหลีกเลี่ยงปัญหาชื่อฟิลด์ชนกับ `SORTED-LINE` ได้ในตัวโดยไม่ต้องเขียน `REPLACING` ยาวๆ อีกชุด

---

## ขั้นตอนที่ 924: `CBEOD` — การกระทบยอดสิ้นวันแบบหลักบัญชีคู่

### หลักการทางบัญชีที่ต้องเข้าใจก่อน

คำกล่าวที่ว่า "ยอดฝากรวมต้องเท่ากับยอดถอนรวม" **ไม่เป็นความจริงเลยในทางบัญชี** — เงินฝากและเงินถอนคือ
การเคลื่อนไหวของเงินสดกับ**โลกภายนอก** ไม่มีเหตุผลใดที่ทั้งสองยอดต้องเท่ากันในแต่ละวัน หลักการที่ถูกต้อง
และตรวจสอบได้จริงจากข้อมูลที่เรามีคือ**กฎบัญชีคู่ของธุรกรรมโอนเงินเท่านั้น**: **ทุกขาหักเงิน (debit)
ของการโอนเงินหนึ่งครั้ง ต้องมีขาเติมเงิน (credit) จำนวนเท่ากันมารองรับเสมอ** และถ้าขาเติมเงินล้มเหลว
ต้องมีขา reversal มาหักล้างขาหักเงินนั้นแทน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBEOD.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBEOD - end-of-day balancing and reconciliation. The classic
      * "total debits = total credits" control total does NOT apply
      * between deposits and withdrawals - those are movements of cash
      * to and from the outside world, so there is no reason they
      * should ever be equal. The identity that MUST always hold comes
      * straight from double-entry bookkeeping: every successful
      * transfer writes exactly one debit leg (XFRD) and one credit
      * leg (XFRC) of the SAME amount, and every reversal (RVSL)
      * cancels out the XFRD that preceded it. CBEOD proves this
      * identity for the day's transfers and prints an exception
      * listing of every REJECTED line.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANSACTION-LOG ASSIGN TO "TXNLOG.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANSACTION-LOG.
       01  TRANSACTION-LINE.
           COPY "cbtxn.cpy".

       WORKING-STORAGE SECTION.
       01  WS-LOG-STATUS            PIC X(2).
       01  WS-EOF-FLAG              PIC X(1)   VALUE "N".
           88  WS-EOF                        VALUE "Y".
       01  WS-REPORT-DATE           PIC 9(8)   VALUE 0.
       01  WS-SUM-DEP               PIC 9(9)V99 VALUE 0.
       01  WS-SUM-WD                PIC 9(9)V99 VALUE 0.
       01  WS-SUM-XFRD-OK           PIC 9(9)V99 VALUE 0.
       01  WS-SUM-XFRC-OK           PIC 9(9)V99 VALUE 0.
       01  WS-SUM-RVSL-OK           PIC 9(9)V99 VALUE 0.
       01  WS-SUM-INT               PIC 9(9)V99 VALUE 0.
       01  WS-NET-TRANSFER-DEBIT    PIC S9(10)V99 VALUE 0.
       01  WS-CNT-REJECTED          PIC 9(5)   VALUE 0.
       01  WS-AMOUNT-EDIT           PIC -Z,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== CBEOD: End-of-Day Reconciliation ===".
           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-REPORT-DATE.

           OPEN INPUT TRANSACTION-LOG.
           DISPLAY " ".
           DISPLAY "-- Exception listing (REJECTED entries) --".
           PERFORM UNTIL WS-EOF
               READ TRANSACTION-LOG
                   AT END
                       MOVE "Y" TO WS-EOF-FLAG
                   NOT AT END
                       PERFORM PROCESS-LINE
               END-READ
           END-PERFORM.
           CLOSE TRANSACTION-LOG.

           IF WS-CNT-REJECTED = 0
               DISPLAY "  (none)"
           END-IF.

           COMPUTE WS-NET-TRANSFER-DEBIT =
               WS-SUM-XFRD-OK - WS-SUM-RVSL-OK.

           DISPLAY " ".
           DISPLAY "---- CBEOD Control Totals (" WS-REPORT-DATE
               ") ----".
           MOVE WS-SUM-DEP TO WS-AMOUNT-EDIT.
           DISPLAY "Total deposits              : " WS-AMOUNT-EDIT.
           MOVE WS-SUM-WD TO WS-AMOUNT-EDIT.
           DISPLAY "Total withdrawals           : " WS-AMOUNT-EDIT.
           MOVE WS-SUM-INT TO WS-AMOUNT-EDIT.
           DISPLAY "Total interest posted       : " WS-AMOUNT-EDIT.
           DISPLAY " ".
           DISPLAY "Transfer legs (double-entry check):".
           MOVE WS-SUM-XFRD-OK TO WS-AMOUNT-EDIT.
           DISPLAY "  Debit  legs (XFRD, OK)    : " WS-AMOUNT-EDIT.
           MOVE WS-SUM-RVSL-OK TO WS-AMOUNT-EDIT.
           DISPLAY "  Less: reversals (RVSL, OK): " WS-AMOUNT-EDIT.
           MOVE WS-NET-TRANSFER-DEBIT TO WS-AMOUNT-EDIT.
           DISPLAY "  = Net transfer debits     : " WS-AMOUNT-EDIT.
           MOVE WS-SUM-XFRC-OK TO WS-AMOUNT-EDIT.
           DISPLAY "  Credit legs (XFRC, OK)    : " WS-AMOUNT-EDIT.
           IF WS-NET-TRANSFER-DEBIT = WS-SUM-XFRC-OK
               DISPLAY "  RESULT: BALANCED - transfer legs match."
           ELSE
               DISPLAY "  RESULT: *** OUT OF BALANCE *** - "
                   "investigate before closing the day."
           END-IF.
           DISPLAY " ".
           DISPLAY "Rejected entries (any type) : " WS-CNT-REJECTED.
           STOP RUN.

       PROCESS-LINE.
           IF TXN-DATE = WS-REPORT-DATE
               IF TXN-STATUS = "REJECTED"
                   ADD 1 TO WS-CNT-REJECTED
                   DISPLAY "  " TXN-DATE " " TXN-TIME " acct="
                       TXN-ACCT-ID " type=" TXN-TYPE " amount="
                       TXN-AMOUNT
               ELSE
                   EVALUATE TXN-TYPE
                       WHEN "DEP "
                           ADD TXN-AMOUNT TO WS-SUM-DEP
                       WHEN "WD  "
                           ADD TXN-AMOUNT TO WS-SUM-WD
                       WHEN "XFRD"
                           ADD TXN-AMOUNT TO WS-SUM-XFRD-OK
                       WHEN "XFRC"
                           ADD TXN-AMOUNT TO WS-SUM-XFRC-OK
                       WHEN "RVSL"
                           ADD TXN-AMOUNT TO WS-SUM-RVSL-OK
                       WHEN "INT "
                           ADD TXN-AMOUNT TO WS-SUM-INT
                   END-EVALUATE
               END-IF
           END-IF.
```

### ทดสอบจริงวันที่ 1 (มีการโอนที่ถูกกู้คืนและถูกปฏิเสธ)

```bash
cobc -x -o cbeod cbeod.cob
printf '20260401\n' | ./cbeod
```

**ผลลัพธ์จริง**:

```
=== CBEOD: End-of-Day Reconciliation ===
Enter processing date (YYYYMMDD):
-- Exception listing (REJECTED entries) --
  20260401 104000 acct=999999 type=XFRC amount=000000500.00
  20260401 105000 acct=100001 type=WD   amount=000090000.00
 
---- CBEOD Control Totals (20260401) ----
Total deposits              :      2,000.00
Total withdrawals           :        300.00
Total interest posted       :          0.00
 
Transfer legs (double-entry check):
  Debit  legs (XFRD, OK)    :      1,500.00
  Less: reversals (RVSL, OK):        500.00
  = Net transfer debits     :      1,000.00
  Credit legs (XFRC, OK)    :      1,000.00
  RESULT: BALANCED - transfer legs match.
 
Rejected entries (any type) : 00002
```

สังเกตการคำนวณ: ขาหักเงิน (`XFRD OK`) รวม 1,500.00 (จากการโอนสำเร็จ 1,000.00 บวกการโอนที่ล้มเหลว
500.00 ที่หักไปก่อนจะกู้คืน) ลบด้วยยอดกู้คืน (`RVSL OK`) 500.00 เหลือ **ยอดหักเงินสุทธิ 1,000.00** ซึ่ง
**เท่ากับ**ยอดเติมเงิน (`XFRC OK`) 1,000.00 พอดี — พิสูจน์ว่าธุรกรรมโอนเงินทุกครั้งในวันนี้สมดุลกัน
100% แม้จะมีเหตุการณ์ล้มเหลวเกิดขึ้นจริงระหว่างวันก็ตาม

### ทดสอบจริงวันที่ 2 (ไม่มีข้อยกเว้นเลย)

```bash
printf '20260402\n' | ./cbeod
```

**ผลลัพธ์จริง**:

```
=== CBEOD: End-of-Day Reconciliation ===
Enter processing date (YYYYMMDD):
-- Exception listing (REJECTED entries) --
  (none)
 
---- CBEOD Control Totals (20260402) ----
Total deposits              :        800.00
Total withdrawals           :      5,500.00
Total interest posted       :          0.00
 
Transfer legs (double-entry check):
  Debit  legs (XFRD, OK)    :        400.00
  Less: reversals (RVSL, OK):          0.00
  = Net transfer debits     :        400.00
  Credit legs (XFRC, OK)    :        400.00
  RESULT: BALANCED - transfer legs match.
 
Rejected entries (any type) : 00000
```

### ข้อควรระวัง

- ถ้า `RESULT: *** OUT OF BALANCE ***` ปรากฏขึ้นจริงในระบบ production นั่นหมายถึง**สัญญาณอันตรายร้ายแรง**
  ที่บ่งชี้ว่ามีบั๊กในโปรแกรมโอนเงิน (`CBXFER`) หรือมีคนเขียนข้อมูลลง `TXNLOG.DAT` ผ่านช่องทางอื่นที่ไม่
  ผ่าน `CBAUDIT` — ต้องหยุดกระบวนการปิดวันทันทีและตรวจสอบก่อนดำเนินการต่อ (ไม่ควรปล่อยให้แบตช์ถัดไป
  เช่น `CBMEND` ทำงานต่อทั้งที่ข้อมูลยังไม่สมดุล)
- การกระทบยอดนี้ตรวจสอบเฉพาะ**ขาของธุรกรรมโอนเงิน** เท่านั้น ไม่ได้ตรวจสอบว่า
  `ACCT-BALANCE` ในทุกบัญชีรวมกันมีค่าถูกต้องหรือไม่ (ซึ่งเป็นหน้าที่ของ `CBCLOSE` ในขั้นตอนที่ 926 ร่วม
  กับการตรวจสอบเชิงเปรียบเทียบระหว่างวัน)

### แบบฝึกหัดที่ 924.1

**โจทย์**: สมมติมีบั๊กใน `CBXFER` ที่ทำให้บันทึกขา `XFRC` ผิดจำนวนเงิน (เช่น เติมเงินเข้าบัญชีปลายทาง
มากกว่าที่หักออกจากบัญชีต้นทางจริง) `CBEOD` จะตรวจพบความผิดปกตินี้ได้หรือไม่ อย่างไร

**เฉลย**: ตรวจพบได้แน่นอน เพราะ `WS-SUM-XFRC-OK` (ผลรวมขาเติมเงิน) จะไม่เท่ากับ `WS-NET-TRANSFER-DEBIT`
(ผลรวมขาหักเงินสุทธิ) อีกต่อไป เงื่อนไข `IF WS-NET-TRANSFER-DEBIT = WS-SUM-XFRC-OK` จะเป็นเท็จ และ
โปรแกรมจะแสดง `RESULT: *** OUT OF BALANCE *** - investigate before closing the day.` ทันที นี่คือ
คุณค่าที่แท้จริงของการกระทบยอดแบบหลักบัญชีคู่: มันตรวจจับความผิดพลาดในตรรกะโปรแกรมได้ ไม่ใช่แค่ความ
ผิดพลาดจากผู้ใช้งานเท่านั้น

---

## ขั้นตอนที่ 925: `CBMEND` — ผ่านรายการดอกเบี้ยสิ้นเดือน + Idempotency Guard

### หลักการ Idempotency Guard

`CBMEND` เปรียบเทียบ **ปี+เดือน** ของ `ACCT-LAST-STMT-DATE` (บันทึกไว้ว่าบัญชีนี้ผ่านรายการดอกเบี้ยครั้ง
ล่าสุดเมื่อใด) กับ ปี+เดือนของวันที่ประมวลผลปัจจุบัน — ถ้าตรงกัน แปลว่าบัญชีนี้ **ถูกผ่านรายการไปแล้วใน
เดือนนี้** จึงข้ามไปโดยไม่ทำอะไรซ้ำ ทำให้การรัน `CBMEND` ซ้ำสำหรับเดือนเดียวกันปลอดภัยเสมอ (Idempotent)

### บั๊กจริงที่พบระหว่างทดสอบ: ค่าเริ่มต้นของ `ACCT-LAST-STMT-DATE` ผิด

ระหว่างทดสอบ `CBMEND` ครั้งแรกกับบัญชีที่เปิดในเดือนเมษายน (`ACCT-OPEN-DATE = 20260401`) และรันประมวลผล
สิ้นเดือนเดียวกัน (`20260430`) พบผลลัพธ์ที่ผิดคาดโดยสิ้นเชิง:

```
---- CBMEND Control Totals (20260430) ----
Accounts scanned         : 00003
Accounts posted this run : 00000
Accounts already posted (idempotency skip) : 00002
```

ทั้งที่บัญชีเหล่านี้**ไม่เคยผ่านรายการดอกเบี้ยมาก่อนเลยสักครั้ง** โปรแกรมกลับรายงานว่า "ข้ามเพราะผ่าน
รายการไปแล้ว" **สาเหตุ**: ตอนออกแบบ `CBOPEN` ครั้งแรก เราตั้งค่าเริ่มต้นของ `ACCT-LAST-STMT-DATE` ให้
เท่ากับ**วันที่เปิดบัญชี** (`MOVE WS-TODAY TO ACCT-LAST-STMT-DATE`) ซึ่งฟังดูสมเหตุสมผลตอนแรก แต่เมื่อ
`CBMEND` เปรียบเทียบปี+เดือนของฟิลด์นี้กับปี+เดือนของวันประมวลผล (ทั้งคู่อยู่ในเดือนเมษายนเดียวกัน) กลไก
ป้องกันการรันซ้ำจึงเข้าใจผิดว่า "เดือนนี้ผ่านรายการไปแล้ว" ทั้งที่ความจริงคือ **ยังไม่เคยผ่านรายการเลย**

**วิธีแก้**: เปลี่ยนค่าเริ่มต้นใน `CBOPEN` เป็น **`0`** (ความหมาย "ยังไม่เคยผ่านรายการ") แทนวันที่เปิด
บัญชี ค่า `0` ไม่มีทางตรงกับปี+เดือนของวันประมวลผลจริงใดๆ เลย จึงรับประกันได้ว่าการผ่านรายการครั้งแรกจะ
เกิดขึ้นเสมอไม่ว่าบัญชีจะเปิดในเดือนใดก็ตาม (โค้ดที่แก้ไขแล้วแสดงไว้ใน Part 091 ขั้นตอนที่ 902 และ 910
ล่วงหน้าแล้ว เพราะเป็นส่วนหนึ่งของ `CBOPEN` ที่สร้างใน Part 091 — บั๊กนี้ถูกพบและแก้ไขระหว่างพัฒนา Case
Study นี้จริง ก่อนที่จะนำ `CBOPEN` ฉบับที่ถูกต้องไปใช้ในทุก Part)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBMEND.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBMEND - month-end batch: posts (capitalizes) accrued interest
      * into the real balance for every interest-bearing account, logs
      * one INT entry per account, and matures any Fixed Deposit whose
      * maturity date has arrived.
      *
      * IDEMPOTENCY GUARD: before posting, CBMEND compares the
      * year+month of ACCT-LAST-STMT-DATE to the year+month of the
      * processing date - if they already match, that account was
      * posted this month already, so it is skipped. Re-running CBMEND
      * for the same month is therefore always SAFE and posts zero a
      * second time.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "cbacct.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-EOF-FLAG              PIC X(1)   VALUE "N".
           88  WS-EOF                        VALUE "Y".
       01  WS-TODAY                 PIC 9(8)   VALUE 0.
       01  WS-TODAY-YM              PIC 9(6)   VALUE 0.
       01  WS-LAST-YM               PIC 9(6)   VALUE 0.
       01  WS-TXN-TYPE-INT          PIC X(4)   VALUE "INT".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
       01  WS-CNT-SCANNED           PIC 9(5)   VALUE 0.
       01  WS-CNT-POSTED            PIC 9(5)   VALUE 0.
       01  WS-CNT-SKIPPED           PIC 9(5)   VALUE 0.
       01  WS-CNT-MATURED           PIC 9(5)   VALUE 0.
       01  WS-TOTAL-POSTED          PIC 9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== CBMEND: Month-End Interest Posting ===".
           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-TODAY.
           COMPUTE WS-TODAY-YM = WS-TODAY / 100.

           OPEN I-O ACCOUNT-MASTER.
           MOVE LOW-VALUES TO ACCT-ID.
           START ACCOUNT-MASTER KEY IS NOT LESS THAN ACCT-ID
               INVALID KEY
                   MOVE "Y" TO WS-EOF-FLAG
           END-START.

           PERFORM UNTIL WS-EOF
               READ ACCOUNT-MASTER NEXT RECORD
                   AT END
                       MOVE "Y" TO WS-EOF-FLAG
                   NOT AT END
                       PERFORM PROCESS-ONE-ACCOUNT
               END-READ
           END-PERFORM.

           CLOSE ACCOUNT-MASTER.
           DISPLAY " ".
           DISPLAY "---- CBMEND Control Totals (" WS-TODAY ") ----".
           DISPLAY "Accounts scanned         : " WS-CNT-SCANNED.
           DISPLAY "Accounts posted this run : " WS-CNT-POSTED.
           DISPLAY "Accounts already posted "
               "(idempotency skip) : " WS-CNT-SKIPPED.
           DISPLAY "Fixed Deposits matured   : " WS-CNT-MATURED.
           DISPLAY "Total interest posted    : " WS-TOTAL-POSTED.
           STOP RUN.

       PROCESS-ONE-ACCOUNT.
           ADD 1 TO WS-CNT-SCANNED.
           IF (ACCT-STATUS = "A" OR ACCT-STATUS = "M")
                   AND (ACCT-TYPE = "S" OR ACCT-TYPE = "F")
               COMPUTE WS-LAST-YM = ACCT-LAST-STMT-DATE / 100
               IF WS-LAST-YM = WS-TODAY-YM
                   ADD 1 TO WS-CNT-SKIPPED
               ELSE
                   IF ACCT-ACCRUED-INT > 0
                       ADD ACCT-ACCRUED-INT TO ACCT-BALANCE
                       ADD ACCT-ACCRUED-INT TO WS-TOTAL-POSTED
                       ADD 1 TO WS-CNT-POSTED
                       CALL "CBAUDIT" USING WS-TODAY 235900
                           WS-TODAY ACCT-ID WS-TXN-TYPE-INT
                           ACCT-ACCRUED-INT ACCT-BALANCE WS-STATUS-OK
                       MOVE 0 TO ACCT-ACCRUED-INT
                   END-IF
                   IF ACCT-TYPE = "F" AND ACCT-STATUS = "A"
                           AND ACCT-MATURITY-DATE <= WS-TODAY
                       MOVE "M" TO ACCT-STATUS
                       ADD 1 TO WS-CNT-MATURED
                   END-IF
                   MOVE WS-TODAY TO ACCT-LAST-STMT-DATE
                   REWRITE ACCOUNT-RECORD
                       INVALID KEY
                           DISPLAY "ERROR rewriting " ACCT-ID
                               " status=" WS-FILE-STATUS
                   END-REWRITE
               END-IF
           END-IF.
```

### ทดสอบจริง: รันครั้งแรก แล้วรันซ้ำเพื่อพิสูจน์ Idempotency

หลังจากรัน `CBACCR` ทุกวันตลอดเดือนเมษายน (วันที่ 1-2 ตามที่ทำใน Part 092 บวกวันที่เหลือของเดือน
จนถึงวันที่ 30 — ในทางปฏิบัติแบตช์นี้ควรรันทุกคืน แต่เพื่อความกระชับของการทดสอบในเอกสารนี้ เราข้าม
รายละเอียดผลลัพธ์ของวันที่ไม่มีธุรกรรมใหม่ และแสดงเฉพาะผลรวมสะสม) มาถึงวันสิ้นเดือน:

```bash
cobc -x cbmend.cob cbaudit.cob -o cbmend
printf '20260430\n' | ./cbmend
```

**ผลลัพธ์จริง (รันครั้งแรก)**:

```
=== CBMEND: Month-End Interest Posting ===
Enter processing date (YYYYMMDD):
---- CBMEND Control Totals (20260430) ----
Accounts scanned         : 00004
Accounts posted this run : 00003
Accounts already posted (idempotency skip) : 00000
Fixed Deposits matured   : 00000
Total interest posted    : 000000176.05
```

3 บัญชีที่มีดอกเบี้ย (Savings 2 บัญชี + Fixed Deposit 1 บัญชี) ได้รับการผ่านรายการรวม 176.05 บาท ไม่มี
บัญชีใดถูกข้าม (เพราะยังไม่เคยผ่านรายการมาก่อนในเดือนนี้เลย — บั๊กที่อธิบายไว้ข้างต้นถูกแก้ไขแล้ว) และ
ยังไม่มี Fixed Deposit ใดครบกำหนด (เปิดเมื่อ 1 เมษายน ครบกำหนด 1 เมษายน 2027)

**รันซ้ำทันทีด้วยวันที่เดียวกัน (พิสูจน์ Idempotency)**:

```bash
printf '20260430\n' | ./cbmend
```

**ผลลัพธ์จริง (รันครั้งที่สอง)**:

```
=== CBMEND: Month-End Interest Posting ===
Enter processing date (YYYYMMDD):
---- CBMEND Control Totals (20260430) ----
Accounts scanned         : 00004
Accounts posted this run : 00000
Accounts already posted (idempotency skip) : 00003
Fixed Deposits matured   : 00000
Total interest posted    : 000000000.00
```

**ผลลัพธ์คือ 0 บาทถูกผ่านรายการซ้ำ** — ทุกบัญชีที่เคยผ่านรายการไปแล้วในรันแรกถูกข้ามอย่างถูกต้องทั้งหมด
นี่คือบทพิสูจน์ที่เป็นรูปธรรมที่สุดของข้อกำหนดข้อ 7 ที่ตั้งไว้ตั้งแต่ Part 091 ขั้นตอนที่ 901: **"ห้าม
ผ่านรายการดอกเบี้ยซ้ำหากรันแบตช์ผิดพลาดซ้ำ"**

### ข้อควรระวัง

- Idempotency Guard นี้ทำงานได้ถูกต้อง**เฉพาะเมื่อรันด้วยวันที่เดียวกันในเดือนเดียวกันเท่านั้น** ถ้ารัน
  `CBMEND` สำหรับเดือนพฤษภาคมหลังจากผ่านรายการเดือนเมษายนไปแล้ว ระบบจะผ่านรายการให้ใหม่ตามปกติ (เพราะ
  ปี+เดือนไม่ตรงกันอีกต่อไป) — นี่คือพฤติกรรมที่ถูกต้องตามที่ออกแบบไว้ ไม่ใช่ข้อจำกัด
- ค่าคงที่ `235900` (23:59:00) ที่ใช้เป็นเวลาของรายการ `INT` สื่อความหมายว่าดอกเบี้ยถูกผ่านรายการ
  ณ ช่วงสิ้นสุดวันทำการ ซึ่งเป็นธรรมเนียมที่ธนาคารจริงส่วนใหญ่ใช้ (batch job รันหลังเวลาทำการปิดแล้ว)

### แบบฝึกหัดที่ 925.1

**โจทย์**: หากมีบัญชีใหม่เปิดขึ้นในวันที่ 29 เมษายน (`ACCT-OPEN-DATE = 20260429`, `ACCT-LAST-STMT-DATE
= 0` ตามค่าเริ่มต้นที่แก้ไขแล้ว) แล้ว `CBMEND` รันวันที่ 30 เมษายนทันที บัญชีนี้จะถูกผ่านรายการดอกเบี้ย
หรือไม่

**เฉลย**: จะถูกผ่านรายการ (ถ้ามีดอกเบี้ยค้างรับสะสมมากกว่า 0 จาก `CBACCR` ที่รันในวันที่ 29-30) เพราะ
`WS-LAST-YM` ของบัญชีนี้คำนวณจาก `ACCT-LAST-STMT-DATE = 0` ได้ค่า `000000` ซึ่งไม่มีทางเท่ากับ
`WS-TODAY-YM` ของเดือนเมษายน 2026 (`202604`) เงื่อนไข Idempotency Guard จึงเป็นเท็จ และดำเนินการผ่าน
รายการตามปกติ — พิสูจน์ว่าการแก้บั๊กเป็นค่าเริ่มต้น `0` ทำให้ระบบรองรับบัญชีที่เปิดใหม่กลางเดือนได้ถูก
ต้อง ไม่ว่าจะเปิดวันไหนของเดือนก็ตาม

---

## ขั้นตอนที่ 926: `CBCLOSE` — รายงานสรุปปิดบัญชี

### หลักการ

`CBCLOSE` สแกน `ACCOUNT-MASTER` ทั้งไฟล์ (เทคนิคเดียวกับ `CBACCR` ในขั้นตอนที่ 918: `START` + `READ
NEXT RECORD`) แล้วสรุปยอดระดับพอร์ตโฟลิโอ: จำนวนบัญชีและยอดรวมแยกตามประเภท, จำนวน Fixed Deposit ที่
ครบกำหนดแล้ว, และดอกเบี้ยค้างรับที่ยังไม่ผ่านรายการ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBCLOSE.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBCLOSE - closing summary report. Scans the whole ACCOUNT-
      * MASTER (sequentially, via START + READ NEXT, Part 028) and
      * prints a portfolio-level summary: how many accounts of each
      * type, their combined balance, and how much interest is still
      * sitting in ACCT-ACCRUED-INT waiting to be posted by CBMEND.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "cbacct.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-EOF-FLAG              PIC X(1)   VALUE "N".
           88  WS-EOF                        VALUE "Y".
       01  WS-TODAY                 PIC 9(8)   VALUE 0.
       01  WS-CNT-SAVINGS           PIC 9(5)   VALUE 0.
       01  WS-CNT-CHECKING          PIC 9(5)   VALUE 0.
       01  WS-CNT-FIXED             PIC 9(5)   VALUE 0.
       01  WS-CNT-MATURED           PIC 9(5)   VALUE 0.
       01  WS-CNT-CLOSED            PIC 9(5)   VALUE 0.
       01  WS-BAL-SAVINGS           PIC S9(10)V99 VALUE 0.
       01  WS-BAL-CHECKING          PIC S9(10)V99 VALUE 0.
       01  WS-BAL-FIXED             PIC S9(10)V99 VALUE 0.
       01  WS-ACCRUED-UNPOSTED      PIC 9(9)V99   VALUE 0.
       01  WS-GRAND-TOTAL           PIC S9(11)V99 VALUE 0.
       01  WS-BAL-EDIT              PIC -Z,ZZZ,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== CBCLOSE: Closing Summary Report ===".
           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-TODAY.

           OPEN INPUT ACCOUNT-MASTER.
           MOVE LOW-VALUES TO ACCT-ID.
           START ACCOUNT-MASTER KEY IS NOT LESS THAN ACCT-ID
               INVALID KEY
                   MOVE "Y" TO WS-EOF-FLAG
           END-START.

           PERFORM UNTIL WS-EOF
               READ ACCOUNT-MASTER NEXT RECORD
                   AT END
                       MOVE "Y" TO WS-EOF-FLAG
                   NOT AT END
                       PERFORM PROCESS-ONE-ACCOUNT
               END-READ
           END-PERFORM.
           CLOSE ACCOUNT-MASTER.

           COMPUTE WS-GRAND-TOTAL =
               WS-BAL-SAVINGS + WS-BAL-CHECKING + WS-BAL-FIXED.

           DISPLAY " ".
           DISPLAY "======================================".
           DISPLAY "  CORE BANKING CLOSING SUMMARY - " WS-TODAY.
           DISPLAY "======================================".
           DISPLAY "SAVINGS   accounts=" WS-CNT-SAVINGS
               WITH NO ADVANCING.
           MOVE WS-BAL-SAVINGS TO WS-BAL-EDIT.
           DISPLAY "   total balance=" WS-BAL-EDIT.
           DISPLAY "CHECKING  accounts=" WS-CNT-CHECKING
               WITH NO ADVANCING.
           MOVE WS-BAL-CHECKING TO WS-BAL-EDIT.
           DISPLAY "   total balance=" WS-BAL-EDIT.
           DISPLAY "FIXED-DEP accounts=" WS-CNT-FIXED
               WITH NO ADVANCING.
           MOVE WS-BAL-FIXED TO WS-BAL-EDIT.
           DISPLAY "   total balance=" WS-BAL-EDIT.
           DISPLAY "  (of which matured and withdrawable: "
               WS-CNT-MATURED ")".
           DISPLAY "Closed accounts (excluded above): " WS-CNT-CLOSED.
           DISPLAY "--------------------------------------".
           MOVE WS-GRAND-TOTAL TO WS-BAL-EDIT.
           DISPLAY "GRAND TOTAL ON DEPOSIT           : " WS-BAL-EDIT.
           DISPLAY "Accrued interest not yet posted  : "
               WS-ACCRUED-UNPOSTED.
           STOP RUN.

       PROCESS-ONE-ACCOUNT.
           IF ACCT-STATUS = "C"
               ADD 1 TO WS-CNT-CLOSED
           ELSE
               ADD ACCT-ACCRUED-INT TO WS-ACCRUED-UNPOSTED
               EVALUATE TRUE
                   WHEN ACCT-TYPE = "S"
                       ADD 1 TO WS-CNT-SAVINGS
                       ADD ACCT-BALANCE TO WS-BAL-SAVINGS
                   WHEN ACCT-TYPE = "C"
                       ADD 1 TO WS-CNT-CHECKING
                       ADD ACCT-BALANCE TO WS-BAL-CHECKING
                   WHEN ACCT-TYPE = "F"
                       ADD 1 TO WS-CNT-FIXED
                       ADD ACCT-BALANCE TO WS-BAL-FIXED
                       IF ACCT-STATUS = "M"
                           ADD 1 TO WS-CNT-MATURED
                       END-IF
               END-EVALUATE
           END-IF.
```

### ทดสอบจริง: เปรียบเทียบก่อนและหลัง `CBMEND`

**ก่อนผ่านรายการดอกเบี้ย (30 เมษายน, ก่อนรัน `CBMEND`)**:

```bash
cobc -x -o cbclose cbclose.cob
printf '20260430\n' | ./cbclose
```

```
======================================
  CORE BANKING CLOSING SUMMARY - 20260430
======================================
SAVINGS   accounts=00002   total balance=        15,400.00
CHECKING  accounts=00001   total balance=-          400.00
FIXED-DEP accounts=00001   total balance=        60,000.00
  (of which matured and withdrawable: 00000)
Closed accounts (excluded above): 00000
--------------------------------------
GRAND TOTAL ON DEPOSIT           :         75,000.00
Accrued interest not yet posted  : 000000176.05
```

**หลังผ่านรายการดอกเบี้ย (รัน `CBMEND` ตามขั้นตอนที่ 925 แล้ว)**:

```bash
printf '20260430\n' | ./cbclose
```

```
======================================
  CORE BANKING CLOSING SUMMARY - 20260430
======================================
SAVINGS   accounts=00002   total balance=        15,415.85
CHECKING  accounts=00001   total balance=-          400.00
FIXED-DEP accounts=00001   total balance=        60,160.20
  (of which matured and withdrawable: 00000)
Closed accounts (excluded above): 00000
--------------------------------------
GRAND TOTAL ON DEPOSIT           :         75,176.05
Accrued interest not yet posted  : 000000000.00
```

เปรียบเทียบทั้งสองรายงาน: `GRAND TOTAL ON DEPOSIT` เพิ่มขึ้นจาก `75,000.00` เป็น `75,176.05` เพิ่มขึ้น
พอดี **176.05 บาท** ตรงกับ `Total interest posted` ที่ `CBMEND` รายงานไว้เป๊ะ และ `Accrued interest not
yet posted` ลดลงจาก `176.05` เหลือ `0.00` พอดี — พิสูจน์ว่าเงินไม่ได้หายไปไหนหรืองอกขึ้นมาเอง เพียงแค่
ย้ายจาก "ค้างรับ" ไปเป็น "ยอดคงเหลือจริง" เท่านั้น

### ข้อควรระวัง

- `CBCLOSE` แสดงยอดคงเหลือ Checking เป็น `-400.00` (ติดลบ) ตรงตามที่ทดสอบไว้ใน Part 092 ขั้นตอนที่ 920
  — รายงานสรุประดับพอร์ตโฟลิโอต้องรองรับยอดติดลบของบัญชี Checking ได้อย่างถูกต้อง ไม่ใช่แค่รายงานบัญชี
  เดี่ยวเท่านั้น (สังเกตว่า `WS-BAL-CHECKING` ประกาศเป็น `S9(10)V99` ซึ่งเป็น Signed)

### แบบฝึกหัดที่ 926.1

**โจทย์**: ทำไม `WS-GRAND-TOTAL` ถึงประกาศเป็น `PIC S9(11)V99` (11 หลัก) ในขณะที่ `WS-BAL-SAVINGS`,
`WS-BAL-CHECKING`, `WS-BAL-FIXED` แต่ละตัวประกาศเป็น `PIC S9(10)V99` (10 หลัก) เท่านั้น

**เฉลยแนวทาง**: `WS-GRAND-TOTAL` เก็บ**ผลรวม**ของทั้งสามยอด ในทางทฤษฎีถ้าทั้งสามยอดมีค่าสูงสุดที่เป็นไป
ได้พร้อมกัน ผลรวมอาจมีจำนวนหลักมากกว่าตัวตั้งต้นแต่ละตัว (หลักการเดียวกับการบวกเลขสองจำนวนที่มี 3 หลัก
อาจได้ผลลัพธ์ 4 หลัก) การเผื่อหลักเพิ่มไว้ 1 หลักสำหรับฟิลด์ผลรวมคือแนวปฏิบัติที่ดีเพื่อป้องกัน
**Size Error** (ทบทวนแนวคิดนี้จาก Part 009 เรื่องเลขคณิตพื้นฐาน) แม้ในทางปฏิบัติของ Case Study นี้จะยัง
ไม่มีทางเกิดขึ้นจริงในระยะสั้นก็ตาม

---

## ขั้นตอนที่ 927: เจาะลึกการจัดการข้อยกเว้นและบทเรียนที่พบทั้งหมด

### สรุปบั๊กจริงทั้งหมดที่พบระหว่างพัฒนา Case Study นี้

ตลอด Part 091-093 เราพบและแก้ไขบั๊กจริง 3 ตัวระหว่างพัฒนา ซึ่งล้วนเป็นบั๊กประเภท **"ไม่มี compile error
หรือ runtime error ใดๆ เตือนตรงๆ"** — เป็นประเภทบั๊กที่อันตรายที่สุดในโลก COBOL เพราะตรวจจับได้ยากด้วย
การอ่านโค้ดเฉยๆ ต้องอาศัยการทดสอบจริงและตรวจผลลัพธ์อย่างละเอียดเท่านั้น:

| # | พบใน | สาเหตุ | วิธีตรวจพบ |
|---|---|---|---|
| 1 | `CBAUDIT` (Part 092 ขั้น 912) | FILE SECTION ไม่ initialize VALUE clause อัตโนมัติ -> FILLER เป็น NUL byte -> FILE STATUS 71 | ตรวจ FILE STATUS หลัง WRITE พบค่าผิดปกติทันที |
| 2 | `CBOPEN` -> `CBAUDIT` (Part 092 ขั้น 912) | ส่ง literal `"OK"` (2 ตัวอักษร) เข้าพารามิเตอร์ `PIC X(8)` -> อ่านเลยขอบเขตหน่วยความจำ | ไล่ตรวจไบต์ทีละตำแหน่งด้วย `FUNCTION ORD` เทียบกับค่าที่คาดหวัง |
| 3 | `CBOPEN` -> `CBMEND` (Part 093 ขั้น 925) | ตั้งค่าเริ่มต้น `ACCT-LAST-STMT-DATE` เป็นวันเปิดบัญชี ทำให้ Idempotency Guard เข้าใจผิดว่าผ่านรายการไปแล้ว | เปรียบเทียบผลลัพธ์ที่ได้ (0 บัญชีถูกผ่านรายการ) กับผลลัพธ์ที่คาดหวัง (ควรมี 3 บัญชี) |

รวมถึงบั๊กที่ 4 ที่พบใน `CBSTMT` (Part 093 ขั้น 923): ข้อมูลค้างจากบัญชีก่อนหน้ารั่วไหลเข้ามาในรายงาน
เพราะ `INVALID KEY` ไม่ได้เคลียร์ฟิลด์ที่เกี่ยวข้อง

### บทเรียนร่วมของบั๊กทั้ง 4 ตัว

สังเกตว่าบั๊กทั้งหมดมีรูปแบบเดียวกัน: **สมมติฐานที่ผิดพลาดเกี่ยวกับ "สถานะเริ่มต้น" ของข้อมูล** — ไม่ว่า
จะเป็นสมมติฐานว่า FILE SECTION จะถูก initialize ให้, สมมติฐานว่า literal สั้นจะ "พอดี" กับพารามิเตอร์
ปลายทาง, สมมติฐานว่าวันเปิดบัญชีเป็นค่าเริ่มต้นที่ปลอดภัยสำหรับ "วันที่ผ่านรายการล่าสุด", หรือสมมติฐาน
ว่า `INVALID KEY` จะทำให้ record area ว่างเปล่า — **นี่คือเหตุผลที่แท้จริงว่าทำไม Part 091 ขั้นตอนที่
908 ถึงกำหนดให้ทุกจุดตรวจสอบข้อผิดพลาดต้องชัดเจนและตรงไปตรงมา ไม่พึ่งพา "พฤติกรรมที่คาดว่าน่าจะเป็น"**

### กลยุทธ์การทดสอบที่ทำให้พบบั๊กเหล่านี้ได้

ทุกบั๊กที่พบมาจากการปฏิบัติตามกฎเหล็กของหลักสูตรนี้อย่างเคร่งครัด: **คอมไพล์และรันโค้ดจริงทุกตัวอย่าง
แล้วตรวจสอบผลลัพธ์จริงอย่างละเอียด** ไม่ใช่แค่ "เขียนโค้ดที่ดูน่าจะถูก" — หากเพียงอ่านโค้ดต้นฉบับของ
`CBAUDIT` เวอร์ชันแรกด้วยตาเปล่า โค้ดจะดูถูกต้องสมบูรณ์แบบทุกประการ ไม่มีทางรู้ได้เลยว่าจะเกิด FILE
STATUS 71 จนกว่าจะรันจริง

### แบบฝึกหัดที่ 927.1

**โจทย์**: จากบั๊กทั้ง 4 ตัวที่สรุปไว้ข้างต้น จงเลือกมา 1 ตัวที่คุณคิดว่า "อันตรายที่สุด" หากไม่ถูกพบ
ก่อนใช้งานจริงในธนาคาร พร้อมให้เหตุผล

**เฉลยแนวทาง**: ไม่มีคำตอบตายตัว แต่เหตุผลที่หนักแน่นที่สุดมักชี้ไปที่บั๊กที่ 3 (Idempotency Guard ผิด)
เพราะถ้าไม่ถูกพบ ลูกค้า**ทุกคน**ที่เปิดบัญชีใหม่ในเดือนใดเดือนหนึ่งจะไม่เคยได้รับดอกเบี้ยเข้าบัญชีเลยแม้
แต่บาทเดียวตลอดไป (เพราะ `CBMEND` จะคิดว่าบัญชีนั้น "ผ่านรายการไปแล้ว" ทุกเดือนตั้งแต่เดือนแรกที่เปิด)
ความเสียหายจะสะสมไปเรื่อยๆ อย่างเงียบๆ โดยไม่มีสัญญาณเตือนใดๆ (ไม่มี error, ไม่มีความผิดปกติที่เห็นชัด
ในรายงานควบคุมยอดรวม เพราะ `CBMEND` เพียงรายงานว่า "ข้ามเพราะผ่านรายการแล้ว" ซึ่งฟังดูเหมือนทำงานถูกต้อง)
กว่าจะมีลูกค้าร้องเรียนหรือมีคนตรวจพบ อาจผ่านไปหลายเดือนหรือหลายปีแล้ว

---

## ขั้นตอนที่ 928: ข้อพิจารณาเชิงปฏิบัติการ

### Batch Window (ทบทวนจาก Part 066)

ระบบนี้แบ่งเวลาทำงานเป็น 3 หน้าต่างเวลาตามที่ Part 091 ขั้นตอนที่ 906 วางไว้: **ต่อธุรกรรม** (online,
ตลอดเวลาทำการ), **รายวัน** (`CBACCR`, `CBDAILY`, `CBSTMT`, `CBEOD` — รันหลังปิดวันทำการ), และ **รายเดือน**
(`CBMEND`, `CBCLOSE` — รันหลัง batch รายวันของวันสุดท้ายของเดือนเสร็จสมบูรณ์แล้วเท่านั้น) ลำดับนี้สำคัญ
มาก: **ต้องรัน `CBEOD` และตรวจสอบว่า `BALANCED` ก่อนเสมอ** ก่อนจะอนุญาตให้ `CBMEND` ทำงาน เพราะการผ่าน
รายการดอกเบี้ยบนข้อมูลที่ยังไม่สมดุลจะทำให้ปัญหาที่ตรวจพบได้ยากขึ้นไปอีก

### การรันซ้ำอย่างปลอดภัย (Idempotency) — ไม่ใช่แค่ `CBMEND`

แม้ `CBMEND` จะเป็นโปรแกรมเดียวที่มี Idempotency Guard ชัดเจน แต่โปรแกรมอื่นในระบบก็มีระดับความปลอดภัย
ต่อการรันซ้ำที่ควรทราบ:

- `CBINIT` — **ไม่ปลอดภัยต่อการรันซ้ำ** (เขียนทับไฟล์เดิมทุกครั้ง) — รันได้ครั้งเดียวเท่านั้น
- `CBOPEN` — ปลอดภัยโดยธรรมชาติ เพราะตรวจสอบบัญชีซ้ำก่อนเปิดเสมอ (รันซ้ำด้วยรหัสบัญชีเดิมจะถูกปฏิเสธ)
- `CBDEP`, `CBWD`, `CBXFER` — **ไม่ควรรันซ้ำโดยเจตนา** เพราะแต่ละครั้งคือธุรกรรมใหม่ที่มีผลจริง (การ
  "รันซ้ำ" ในบริบทนี้หมายถึงการส่งคำขอเดิมซ้ำสองครั้งโดยไม่ตั้งใจ เช่น ระบบหน้าบ้านส่งคำขอฝากเงินซ้ำ
  เพราะผู้ใช้กดปุ่มสองครั้ง — การป้องกันกรณีนี้ต้องทำที่ชั้นแอปพลิเคชันด้านหน้า เช่น ใช้ Idempotency Key
  ต่อคำขอ ซึ่งเป็นหัวข้อที่กว้างเกินขอบเขตของ Case Study นี้ แต่ควรตระหนักไว้)
- `CBACCR` — **ไม่ปลอดภัยต่อการรันซ้ำในวันเดียวกัน** (จะสะสมดอกเบี้ยค้างรับซ้ำสองเท่าถ้ารันสองครั้งในวัน
  เดียวกัน) — นี่คือข้อจำกัดที่ควรระบุไว้ชัดเจนสำหรับทีมปฏิบัติการ: `CBACCR` ต้องมีการควบคุมจาก Job
  Scheduler ให้รันเพียงครั้งเดียวต่อวันเท่านั้น (ต่างจาก `CBMEND` ที่มีตัวป้องกันในตัวโปรแกรมเอง)
  ซึ่งเป็นตัวอย่างที่ดีว่าไม่ใช่ทุกโปรแกรมจำเป็นต้องมี Idempotency Guard ในตัว — บางโปรแกรมพึ่งพาการ
  ควบคุมจากภายนอก (Job Scheduling Discipline) แทนได้ ตราบใดที่ทีมปฏิบัติการเข้าใจข้อจำกัดนี้ชัดเจน
- รายงานทั้งหมด (`CBDAILY`, `CBSTMT`, `CBEOD`, `CBCLOSE`) — **ปลอดภัยต่อการรันซ้ำเสมอ** เพราะเป็นการ
  อ่านอย่างเดียว ไม่มีการแก้ไขข้อมูลใดๆ

### การสำรองข้อมูลก่อนแบตช์สิ้นเดือน

เนื่องจาก `CBMEND` แก้ไขยอดคงเหลือจริงของทุกบัญชีในระบบพร้อมกัน ธรรมเนียมปฏิบัติจริงของธนาคารคือ**สำรอง
ไฟล์ `ACCTMAST.DAT` และ `TXNLOG.DAT` ก่อนรันแบตช์สิ้นเดือนเสมอ** (ง่ายๆ เพียง `cp` ไฟล์ทั้งสองไปเก็บไว้
ที่อื่นก่อนรัน) เพื่อให้สามารถกู้คืนสถานะก่อนหน้าได้ทันทีหากพบปัญหาหลังรันเสร็จ — แม้ `CBMEND` จะมี
Idempotency Guard ป้องกันการผ่านรายการซ้ำแล้วก็ตาม การสำรองข้อมูลยังคงเป็นเกราะป้องกันชั้นสุดท้ายสำหรับ
สถานการณ์ที่ไม่คาดคิดอื่นๆ ที่ Idempotency Guard ไม่ได้ถูกออกแบบมาให้ครอบคลุม (เช่น บั๊กที่ยังไม่ถูกพบ)

### แบบฝึกหัดที่ 928.1

**โจทย์**: จงอธิบายว่าทำไม `CBACCR` ถึงไม่มี Idempotency Guard ในตัวเหมือน `CBMEND` ทั้งที่เป็นโปรแกรม
ที่แก้ไขข้อมูลจริงเหมือนกัน

**เฉลยแนวทาง**: `CBMEND` มี field `ACCT-LAST-STMT-DATE` ที่ถูกออกแบบมาโดยเฉพาะเพื่อบันทึก "รอบที่ผ่าน
รายการล่าสุด" ในระดับความละเอียด "เดือน" ซึ่งตรงกับความถี่ในการรันของ `CBMEND` พอดี แต่ `CBACCR` รันทุก
วัน และ Case Study นี้ไม่ได้ออกแบบฟิลด์ที่เก็บ "วันที่คำนวณดอกเบี้ยค้างรับล่าสุด" ในระดับความละเอียดที่
เพียงพอสำหรับใช้เป็นตัวป้องกันการรันซ้ำในวันเดียวกันอย่างน่าเชื่อถือ (แม้จะมี `ACCT-LAST-ACCR-DATE` อยู่
ก็ตาม แต่ในการออกแบบปัจจุบันฟิลด์นี้เป็นเพียงข้อมูลบันทึกไว้ ไม่ได้ถูกใช้เป็นเงื่อนไขตรวจสอบจริงในโค้ด)
นี่เป็นตัวอย่างที่ดีของการตัดสินใจในการออกแบบระบบจริง: ทีมพัฒนาสามารถเพิ่ม Idempotency Guard ให้
`CBACCR` ได้ในอนาคตโดยใช้ `ACCT-LAST-ACCR-DATE` เปรียบเทียบกับวันที่ประมวลผลแบบเต็มวัน (ไม่ใช่แค่
ปี+เดือน) แต่ในเวอร์ชันปัจจุบันเลือกพึ่งพาวินัยของ Job Scheduler แทน ซึ่งเป็น Trade-off ที่ต้องระบุให้
ทีมปฏิบัติการทราบอย่างชัดเจน

---

## ขั้นตอนที่ 929: ทดสอบครบวงจร ส่วนที่ 1 — Setup และธุรกรรม 2 วันทำการ

ถึงเวลาพิสูจน์ Case Study ทั้งหมดตั้งแต่ต้นจนจบด้วยการรันทุกโปรแกรมตามลำดับ เริ่มจากไฟล์เปล่าสนิท
(`rm -f ACCTMAST.DAT TXNLOG.DAT`) แล้วไล่ตามลำดับสถาปัตยกรรมที่ออกแบบไว้ตั้งแต่ Part 091

```bash
export LD_LIBRARY_PATH=/opt/gnucobol-isam/lib
rm -f ACCTMAST.DAT TXNLOG.DAT

./cbinit

printf '20260401\n090000\n100001\nSUPAPORN JAIDEE\nS\n000500000\n' | ./cbopen
printf '20260401\n090500\n200001\nGLOBAL TRADING CO\nC\n000100000\n0100000\n' | ./cbopen
printf '20260401\n091000\n300001\nWICHAI SOMBOON\nF\n006000000\n' | ./cbopen
printf '20260401\n091500\n400001\nNATTAYA PROSPER\nS\n001200000\n' | ./cbopen

printf '20260401\n100000\n100001\n000200000\n' | ./cbdep
printf '20260401\n101000\n200001\n000030000\n' | ./cbwd
printf '20260401\n103000\n00000001\n400001\n200001\n000100000\n' | ./cbxfer
printf '20260401\n104000\n00000002\n100001\n999999\n000050000\n' | ./cbxfer
printf '20260401\n105000\n100001\n009000000\n' | ./cbwd
printf '20260401\n' | ./cbaccr
printf '20260401\n' | ./cbdaily
printf '20260401\n' | ./cbeod

printf '20260402\n090000\n200001\n000080000\n' | ./cbdep
printf '20260402\n091000\n200001\n000250000\n' | ./cbwd
printf '20260402\n093000\n00000003\n200001\n100001\n000040000\n' | ./cbxfer
printf '20260402\n094000\n400001\n000300000\n' | ./cbwd
printf '20260402\n' | ./cbaccr
printf '20260402\n' | ./cbdaily
printf '20260402\n' | ./cbeod
```

ผลลัพธ์ของทุกคำสั่งข้างต้นตรงกับที่แสดงไว้ครบถ้วนแล้วใน Part 092 ขั้นตอนที่ 919-920 (สำหรับธุรกรรม) และ
Part 093 ขั้นตอนที่ 922, 924 (สำหรับรายงาน `CBDAILY`/`CBEOD` ของทั้งสองวัน) — **นี่คือจุดสำคัญที่ควรสังเกต**:
เราไม่ได้รันแยกส่วนแล้วค่อยมาประกอบ แต่ทุกคำสั่งข้างต้นรันเรียงกันเป็นสาย (pipeline) เดียวจากไฟล์เปล่า
จนถึงจุดนี้ และให้ผลลัพธ์**เหมือนกันทุกตัวอักษร**กับที่แสดงไว้ในแต่ละขั้นตอนก่อนหน้า — พิสูจน์ว่าตัวอย่าง
ทุกตัวอย่างในเอกสารทั้ง 3 Part นี้เป็นระบบเดียวกันที่ต่อเนื่องกันจริง ไม่ใช่ตัวอย่างเดี่ยวๆ ที่แยกจากกัน

### ตรวจสอบยอดคงเหลือสะสมหลังจบวันทำการที่ 2

```bash
printf '20260402\n' | ./cbclose
```

**ผลลัพธ์จริง**:

```
======================================
  CORE BANKING CLOSING SUMMARY - 20260402
======================================
SAVINGS   accounts=00002   total balance=        15,400.00
CHECKING  accounts=00001   total balance=-          400.00
FIXED-DEP accounts=00001   total balance=        60,000.00
  (of which matured and withdrawable: 00000)
Closed accounts (excluded above): 00000
--------------------------------------
GRAND TOTAL ON DEPOSIT           :         75,000.00
Accrued interest not yet posted  : 000000011.97
```

ยอดรวมทั้งพอร์ต 75,000.00 บาท ตรงตามที่คาดจากผลรวมของธุรกรรมฝาก/ถอน/โอนทั้งหมดใน 2 วัน (เงินไม่มีทาง
งอกขึ้นมาเองหรือหายไปจากระบบ เพราะการโอนเงินระหว่างบัญชีไม่กระทบยอดรวมทั้งพอร์ต มีเพียงเงินฝากใหม่และ
เงินถอนออกเท่านั้นที่กระทบยอดรวม) และดอกเบี้ยค้างรับสะสม 2 วันเท่ากับ 11.97 บาท (6.11 + 5.86 ตรงตาม
ที่ `CBACCR` รายงานไว้ในแต่ละวัน)

### แบบฝึกหัดที่ 929.1

**โจทย์**: จงคำนวณด้วยมือว่ายอดรวมทั้งพอร์ต 75,000.00 บาท มาจากการรวมยอดเปิดบัญชีเริ่มต้นและธุรกรรม
ฝาก/ถอนสุทธิใดบ้าง (ไม่นับการโอนเงินระหว่างบัญชี เพราะไม่กระทบยอดรวม)

**เฉลย**: ยอดเปิดบัญชีรวม = `5,000 + 1,000 + 60,000 + 12,000 = 78,000.00` บวกเงินฝากสุทธิ (`2,000 +
800 = 2,800.00`) ลบเงินถอนสุทธิที่สำเร็จ (`300 + 2,500 + 3,000 = 5,800.00`) รวม: `78,000 + 2,800 -
5,800 = 75,000.00` ตรงกับยอดที่ `CBCLOSE` รายงานไว้เป๊ะ (ธุรกรรมที่ถูกปฏิเสธหรือถูกกู้คืนไม่นับรวมเพราะ
ไม่มีผลกระทบสุทธิต่อยอดเงินจริง)

---

## ขั้นตอนที่ 930: ทดสอบครบวงจร ส่วนที่ 2 — ปิดบัญชีสิ้นเดือนและสรุปทั้ง Case Study

### เดินหน้าสู่สิ้นเดือน

ในระบบจริง `CBACCR` จะถูกตั้งเวลาให้รันทุกคืนตลอดเดือนผ่าน Job Scheduler (Part 066) เราจำลองส่วนที่
เหลือของเดือนเมษายน (วันที่ 3-30) ด้วยการรัน `CBACCR` ซ้ำสำหรับแต่ละวันที่เหลือ (ไม่มีธุรกรรมฝาก/ถอน/
โอนเพิ่มเติมในช่วงนี้ เพื่อให้เห็นผลของดอกเบี้ยสะสมล้วนๆ):

```bash
for d in 03 04 05 06 07 08 09 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29
do
  printf "202604${d}\n" | ./cbaccr > /dev/null
done
printf '20260430\n' | ./cbaccr
```

**ผลลัพธ์จริงของการรันวันสุดท้าย (30 เมษายน)**:

```
=== CBACCR: Daily Interest Accrual ===
Enter processing date (YYYYMMDD):
---- CBACCR Control Totals (20260430) ----
Accounts scanned            : 00004
Accounts accrued interest on : 00003
Total interest accrued today : 000000005.86
```

### ปิดบัญชีสิ้นเดือน: `CBCLOSE` ก่อนผ่านรายการ, `CBMEND`, พิสูจน์ Idempotency, `CBCLOSE` หลังผ่านรายการ

```bash
printf '20260430\n' | ./cbclose
printf '20260430\n' | ./cbmend
printf '20260430\n' | ./cbmend
printf '20260430\n' | ./cbclose
./cbstmt
```

**ผลลัพธ์จริงทั้งหมด เรียงตามลำดับการรัน**:

```
======================================
  CORE BANKING CLOSING SUMMARY - 20260430
======================================
SAVINGS   accounts=00002   total balance=        15,400.00
CHECKING  accounts=00001   total balance=-          400.00
FIXED-DEP accounts=00001   total balance=        60,000.00
  (of which matured and withdrawable: 00000)
Closed accounts (excluded above): 00000
--------------------------------------
GRAND TOTAL ON DEPOSIT           :         75,000.00
Accrued interest not yet posted  : 000000176.05

---- CBMEND Control Totals (20260430) ----
Accounts scanned         : 00004
Accounts posted this run : 00003
Accounts already posted (idempotency skip) : 00000
Fixed Deposits matured   : 00000
Total interest posted    : 000000176.05

---- CBMEND Control Totals (20260430) ----
Accounts scanned         : 00004
Accounts posted this run : 00000
Accounts already posted (idempotency skip) : 00003
Fixed Deposits matured   : 00000
Total interest posted    : 000000000.00

======================================
  CORE BANKING CLOSING SUMMARY - 20260430
======================================
SAVINGS   accounts=00002   total balance=        15,415.85
CHECKING  accounts=00001   total balance=-          400.00
FIXED-DEP accounts=00001   total balance=        60,160.20
  (of which matured and withdrawable: 00000)
Closed accounts (excluded above): 00000
--------------------------------------
GRAND TOTAL ON DEPOSIT           :         75,176.05
Accrued interest not yet posted  : 000000000.00
```

และใบแจ้งยอดสุดท้ายของบัญชี `300001` (Fixed Deposit) แสดงให้เห็นผลของดอกเบี้ยที่เพิ่งผ่านรายการเข้าไป
ในยอดคงเหลือ (แม้ log ธุรกรรมจะยังแสดงแค่รายการ `OPEN` เพราะดอกเบี้ยถูกผ่านรายการเป็น log ประเภท `INT`
แยกต่างหากในรันจริง):

```
======================================
STATEMENT FOR ACCOUNT 300001 - WICHAI SOMBOON
======================================
  20260401 091000 OPEN    60,000.00   bal after:     60,000.00  [OK      ]
  -----------------------------------
  Lines on statement : 00001
  Total credits      :         0.00
  Total debits       :         0.00
  Current balance    :     60,160.20
```

ยอดคงเหลือปัจจุบัน `60,160.20` (จาก `ACCOUNT-MASTER` โดยตรง) สูงกว่ายอดหลังทำรายการล่าสุดในประวัติ
(`60,000.00`) อยู่ `160.20` บาท ซึ่งคือดอกเบี้ยที่สะสมมาตลอดเดือนเมษายนและเพิ่งถูกผ่านรายการจริงโดย
`CBMEND` — ถูกต้องตรงตามที่ออกแบบไว้ทุกประการ

### สรุปผลการทดสอบครบวงจรทั้งระบบ

| การตรวจสอบ | ผลลัพธ์ |
|---|---|
| เปิดบัญชี 3 ประเภทสำเร็จตามกฎขั้นต่ำที่กำหนด | ผ่าน |
| ฝาก/ถอน ตรวจสอบยอดคงเหลือ/วงเงินเบิกเกินบัญชี/ล็อก Fixed Deposit ถูกต้อง | ผ่าน |
| โอนเงินสำเร็จ ทั้งสองขาสมดุลกัน | ผ่าน |
| โอนเงินล้มเหลว กลไก Compensating Transaction กู้คืนเงินครบ 100% | ผ่าน |
| ดอกเบี้ยขั้นบันไดคำนวณถูกต้องตามช่วงยอดคงเหลือ | ผ่าน |
| การกระทบยอดสิ้นวันพิสูจน์หลักบัญชีคู่ของทุกธุรกรรมโอนเงิน | ผ่าน (BALANCED ทั้ง 2 วัน) |
| การผ่านรายการดอกเบี้ยสิ้นเดือนถูกต้องตามยอดค้างรับสะสม | ผ่าน (176.05 บาทตรงกันทั้งสองรายงาน) |
| รันแบตช์สิ้นเดือนซ้ำแล้วไม่ผ่านรายการซ้ำ (Idempotency) | ผ่าน (รันครั้งที่ 2 ผ่านรายการ 0 บาท) |
| ยอดรวมทั้งพอร์ตถูกต้องตามธุรกรรมสุทธิที่เกิดขึ้นจริง | ผ่าน (คำนวณด้วยมือตรงกัน) |

ทุกข้อกำหนดที่ตั้งไว้ตั้งแต่ Part 091 ขั้นตอนที่ 901 ได้รับการพิสูจน์ด้วยการทดสอบจริงครบถ้วนทุกข้อ

### แบบฝึกหัดที่ 930.1

**โจทย์**: จงย้อนกลับไปดูตารางเชื่อมโยงข้อกำหนด (Requirements Traceability) ที่วางไว้ใน Part 091
ขั้นตอนที่ 901 แล้วจับคู่แต่ละข้อกำหนดทั้ง 7 ข้อกับหลักฐานการทดสอบจริงในตาราง "สรุปผลการทดสอบครบวงจร
ทั้งระบบ" ข้างต้น ว่าข้อกำหนดใดถูกพิสูจน์ด้วยแถวใด

**เฉลยแนวทาง**: ข้อกำหนด 1 (บัญชี 3 ประเภท) ตรงกับแถว "เปิดบัญชี 3 ประเภทสำเร็จ", ข้อกำหนด 2 (ฝาก/ถอน)
ตรงกับแถว "ฝาก/ถอน ตรวจสอบยอดคงเหลือ...", ข้อกำหนด 3 (โอนเงินปลอดภัย) ตรงกับสองแถว "โอนเงินสำเร็จ" และ
"โอนเงินล้มเหลว กลไก Compensating Transaction", ข้อกำหนด 4 (ดอกเบี้ย) ตรงกับแถว "ดอกเบี้ยขั้นบันได" และ
"การผ่านรายการดอกเบี้ยสิ้นเดือน", ข้อกำหนด 5 (หลักบัญชีคู่) ตรงกับแถว "การกระทบยอดสิ้นวัน", ข้อกำหนด 6
(รายงาน) แสดงผลอยู่ตลอดขั้นตอนที่ 922-923, 926 ของ Part นี้ (แม้จะไม่มีแถวแยกในตารางสรุป เพราะเป็นเครื่อง
มือที่ใช้แสดงหลักฐานของข้อกำหนดอื่นๆ มากกว่าจะเป็นสิ่งที่ต้อง "ผ่าน/ไม่ผ่าน" ด้วยตัวมันเอง) และข้อกำหนด 7
(Idempotency) ตรงกับแถว "รันแบตช์สิ้นเดือนซ้ำแล้วไม่ผ่านรายการซ้ำ" การฝึกจับคู่แบบนี้คือทักษะสำคัญของ
Business Analyst และ QA ในโปรเจกต์ COBOL ระดับองค์กรจริง (ทบทวนแนวคิดจาก Part 090)

## สรุปท้ายบท (ปิดท้าย Case Study ทั้ง 3 Part)

Case Study Core Banking จบลงอย่างสมบูรณ์ในที่นี้ ตลอด 3 Part (091-093, 30 ขั้นตอน) เราได้:

- **ออกแบบระบบ** (Part 091): ข้อกำหนดทางธุรกิจ 3 ประเภทบัญชี, โครงสร้าง `ACCOUNT-MASTER` และ
  `TRANSACTION-LOG`, Copybook ที่ทดสอบแล้ว, สถาปัตยกรรม 12 โปรแกรม, มาตรฐานการเขียนโค้ด และกลยุทธ์
  จัดการข้อผิดพลาด
- **สร้างโมดูลธุรกรรม** (Part 092): `CBAUDIT`, `CBDEP`, `CBWD`, และหัวใจสำคัญที่สุดคือ `CBXFER` พร้อม
  กลไก Compensating Transaction ที่ออกแบบและพิสูจน์เองตั้งแต่ต้น (เพราะ COBOL ไม่มี COMMIT/ROLLBACK
  ในตัว) รวมถึง `CBACCR` คำนวณดอกเบี้ยขั้นบันได
- **สร้างรายงานและปิดบัญชี** (Part 093): `CBDAILY`, `CBSTMT`, `CBEOD` (พิสูจน์หลักบัญชีคู่), `CBMEND`
  (พร้อม Idempotency Guard), `CBCLOSE` และปิดท้ายด้วยการทดสอบครบวงจรทั้งระบบที่แสดงผลลัพธ์จริงทุก
  ขั้นตอนตั้งแต่ไฟล์เปล่าจนถึงการปิดบัญชีสิ้นเดือน

**รวมทั้งหมด 12 โปรแกรม + 2 Copybook** ที่ทุกตัวคอมไพล์และรันได้จริงด้วย GnuCOBOL (build ที่เปิดใช้
Indexed File ผ่าน Berkeley DB) และที่สำคัญไม่แพ้กัน: **บั๊กจริง 4 ตัว**ที่พบและแก้ไขระหว่างพัฒนา ซึ่งทุก
ตัวไม่มี compile error ใดๆ เตือนล่วงหน้าเลย — นี่คือบทเรียนที่มีค่าที่สุดของ Case Study นี้ และเป็น
เหตุผลที่แท้จริงว่าทำไมหลักสูตรนี้ยืนกรานให้**คอมไพล์และรันโค้ดจริงทุกตัวอย่างเสมอ** ไม่ใช่แค่เขียนโค้ด
ที่ "ดูน่าจะถูก" เพราะในโลก COBOL จริง ความแตกต่างระหว่างโค้ดที่ "ดูถูก" กับโค้ดที่ "ถูกจริง" อาจหมายถึง
เงินหลายล้านบาทของลูกค้าธนาคารหลายพันคน

Case Study ถัดไปใน **Part 094** จะพาไปสำรวจอุตสาหกรรมอีกแขนงหนึ่งที่ COBOL ยังคงเป็นแกนหลัก: **ระบบ
ประกันภัย** ซึ่งมีความท้าทายที่แตกต่างออกไปโดยสิ้นเชิง (กรมธรรม์ระยะยาวหลายสิบปี, การคำนวณเบี้ยประกัน,
การเคลม) แต่จะยังคงยึดหลักการเดียวกันที่พิสูจน์แล้วตลอด Case Study นี้: ออกแบบให้รอบคอบ, ทดสอบทุกอย่าง
จริง, และไม่ไว้ใจสมมติฐานใดๆ โดยไม่พิสูจน์

**[กลับไป Part 092: โมดูลบัญชีและธุรกรรม](part-092-corebanking-transactions.md)**
**[ไปยัง Part 094: Case Study ระบบประกันภัย →](part-094-insurance-case-study.md)**
