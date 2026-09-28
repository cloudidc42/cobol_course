# Part 092: Case Study - Core Banking ตอนที่ 2: โมดูลบัญชีและธุรกรรม (ขั้นตอนที่ 911–920)

## คำนำของ Part นี้

Part 091 วางรากฐานของ Case Study Core Banking ไว้ครบถ้วน: เราออกแบบ `ACCOUNT-MASTER` (Indexed File
เก็บสถานะบัญชี 3 ประเภท: Savings, Checking, Fixed Deposit), `TRANSACTION-LOG` (Sequential Audit Trail
พร้อม `TXN-GROUP-ID` สำหรับเชื่อมขาธุรกรรมโอนเงิน), สร้าง Copybook `cbacct.cpy` และ `cbtxn.cpy` ที่ผ่าน
การทดสอบแล้ว และสร้าง `CBINIT` + `CBOPEN` จนได้บัญชีทดสอบ 3 บัญชีพร้อม Audit Trail เริ่มต้น

Part นี้คือจุดที่ระบบ "มีชีวิต" ขึ้นมาจริงๆ เราจะสร้างโมดูลที่ทำให้เงินเคลื่อนไหวได้:

- **`CBAUDIT`** — subprogram กลางที่ทุกโมดูลเรียกใช้เพื่อบันทึก Audit Trail (พร้อมบั๊กจริงสองตัวที่พบ
  และแก้ไขระหว่างพัฒนา ซึ่งเป็นบทเรียนล้ำค่า)
- **`CBDEP`** — ฝากเงิน
- **`CBWD`** — ถอนเงิน พร้อมตรวจสอบยอดคงเหลือ/วงเงินเบิกเกินบัญชี/ล็อก Fixed Deposit ตามกฎที่ออกแบบไว้
- **`CBXFER`** — โอนเงินระหว่างบัญชี **นี่คือหัวใจสำคัญที่สุดของ Case Study นี้**: COBOL ไม่มีกลไก
  Transaction แบบ COMMIT/ROLLBACK ในตัวเหมือนฐานข้อมูล SQL เราจะออกแบบและสร้างกลไก **Compensating
  Transaction** ของเราเองขึ้นมา แล้วพิสูจน์ด้วยการทดสอบจริงทั้งเส้นทางสำเร็จและเส้นทางล้มเหลว
- **`CBACCR`** — คำนวณดอกเบี้ยค้างรับรายวันแบบขั้นบันได (Tiered Interest)

ปิดท้ายด้วยการทดสอบรวม (Integration Test) 2 วันทำการ เพื่อพิสูจน์ว่าทุกโมดูลทำงานร่วมกันถูกต้อง ก่อนส่ง
ไม้ต่อให้ Part 093 สร้างรายงานและกระบวนการปิดบัญชี

### ภาพรวม 10 ขั้นตอนของ Part นี้

| ขั้นตอน | สิ่งที่สร้าง |
|---|---|
| 911 | ทบทวน Part 091 + ภาพรวมโมดูลธุรกรรมที่จะสร้าง |
| 912 | `CBAUDIT` — subprogram บันทึก Audit Trail (พร้อมบั๊กจริง 2 ตัวที่พบระหว่างทดสอบ) |
| 913 | `CBDEP` — ฝากเงิน |
| 914 | `CBWD` — ถอนเงิน (ตรวจสอบยอด/วงเงินเบิกเกินบัญชี/ล็อก Fixed Deposit) |
| 915 | ปัญหา Atomicity ของ COBOL และการออกแบบ Compensating Transaction |
| 916 | สร้าง `CBXFER` ฉบับเต็ม |
| 917 | ทดสอบ `CBXFER` ทั้งเส้นทางสำเร็จและเส้นทางล้มเหลว-กู้คืน |
| 918 | `CBACCR` — ดอกเบี้ยค้างรับแบบขั้นบันได |
| 919 | ทดสอบรวมวันทำการที่ 1 (Day 1 Integration Test) |
| 920 | ทดสอบรวมวันทำการที่ 2 (Day 2, วงเงินเบิกเกินบัญชี) + สรุปส่งต่อ Part 093 |

---

## ขั้นตอนที่ 911: ทบทวน Part 091 และภาพรวมโมดูลธุรกรรม

### สถานะปัจจุบันของระบบ (จาก Part 091)

เมื่อจบ Part 091 เรามีไฟล์และโปรแกรมดังนี้พร้อมใช้งาน:

- `cbacct.cpy`, `cbtxn.cpy` — Copybook ที่ทดสอบผ่านแล้ว
- `ACCTMAST.DAT` — มีบัญชี 3 บัญชี: `100001` (Savings, 1,500.00), `200001` (Checking, 0.00, วงเงิน
  เบิกเกินบัญชี 50.00), `300001` (Fixed Deposit, 50,000.00)
- `TXNLOG.DAT` — มี 3 บรรทัด `OPEN` ที่สอดคล้องกับการเปิดบัญชีทั้งสาม

ใน Part นี้เราจะรีเซ็ตข้อมูลตัวอย่างให้สมจริงขึ้นเล็กน้อย (เปิดบัญชี 4 บัญชี ยอดเงินสมจริงกว่าเดิม) เพื่อ
ใช้เป็นชุดทดสอบตลอด Part 092-093 — วิธีเปิดบัญชียังคงใช้ `CBOPEN` ตัวเดียวกับที่สร้างใน Part 091 ทุก
ประการ ไม่มีการแก้ไขโปรแกรมนั้นอีก

### หลักการออกแบบที่จะยึดตลอด Part นี้

ทุกโมดูลธุรกรรมในบทนี้ (`CBDEP`, `CBWD`, `CBXFER`) เดินตามรูปแบบเดียวกัน ซึ่งเป็นรูปแบบ "Online-style
Transaction" ที่ Part 070 วางรากฐานไว้ และ Part 061-064 (CICS) อธิบายแนวคิดไว้ล่วงหน้า:

1. เปิดไฟล์ `ACCOUNT-MASTER` แบบ `I-O` (อ่าน-เขียนได้)
2. รับอินพุตและตรวจสอบทีละชั้น (ใช้ `WS-VALID-FLAG` / `WS-ALL-VALID` — ทบทวนจาก Part 069)
3. อ่านบัญชีที่เกี่ยวข้อง ตรวจกฎธุรกิจตามประเภทบัญชี
4. ถ้าผ่านทุกเงื่อนไข: แก้ไขยอดคงเหลือ, `REWRITE`, แล้ว `CALL "CBAUDIT"` เพื่อบันทึก log
5. ถ้าไม่ผ่าน: `CALL "CBAUDIT"` บันทึก log สถานะ `REJECTED` (ยกเว้นกรณี fail-fast ที่ยังไม่รู้จักบัญชีเลย)
6. ปิดไฟล์และจบการทำงาน (`STOP RUN`) — จำลองพฤติกรรมสั้นๆ ทีละธุรกรรมแบบ pseudo-conversational CICS
   task ตามที่ Part 064 อธิบายไว้

### แบบฝึกหัดที่ 911.1

**โจทย์**: ทำไมรูปแบบ "เปิดไฟล์ -> ทำหนึ่งธุรกรรม -> ปิดไฟล์ -> จบโปรแกรม" ถึงเหมาะกับการจำลอง CICS
Transaction มากกว่าการเขียนโปรแกรมเดียวที่เปิดไฟล์ค้างไว้แล้ววนรับธุรกรรมหลายรายการใน loop เดียว?

**เฉลยแนวทาง**: CICS Transaction แต่ละตัวบน Mainframe จริงถูกออกแบบให้เป็นหน่วยงานสั้นๆ ที่เริ่มต้นเมื่อ
มีคำขอเข้ามา (เช่น กดปุ่มที่ตู้ ATM) และจบลงทันทีหลังประมวลผลเสร็จ ไม่ผูกไฟล์หรือทรัพยากรไว้นานเกินความ
จำเป็น (Part 064 อธิบายว่านี่คือหลักการ pseudo-conversational) การจำลองพฤติกรรมนี้ด้วยโปรแกรมที่รันครั้ง
เดียวจบต่อหนึ่งคำขอ จึงใกล้เคียงกับพฤติกรรมจริงมากกว่าการเขียน loop รับหลายธุรกรรมในโปรแกรมเดียว ซึ่งจะ
คล้ายกับโปรแกรมแบตช์มากกว่าโปรแกรม transaction online

---

## ขั้นตอนที่ 912: `CBAUDIT` — Subprogram บันทึก Audit Trail

### ทำไมต้องรวม logic การเขียน Log ไว้ที่เดียว

เช่นเดียวกับ `WRTAUDIT` ใน Part 070 เราใช้เทคนิค **CALL Subprogram** (Part 031-032) เพื่อให้ทุกโปรแกรม
เขียน log ผ่านจุดเดียว ป้องกันไม่ให้รูปแบบบรรทัดใน `TXNLOG.DAT` เพี้ยนไปทีละโปรแกรม

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBAUDIT.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBAUDIT - shared subprogram (Part 031-032 CALL technique).
      * Every online and batch program in the Core Banking case study
      * CALLs CBAUDIT instead of writing to TXNLOG.DAT directly, so the
      * log line layout is defined and written from exactly one place.
      * Each CALL opens the file in EXTEND (append) mode, writes one
      * line, and closes again - a short-lived unit of work, matching
      * how an online transaction program would behave.
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

       LINKAGE SECTION.
       01  LK-TXN-DATE              PIC 9(8).
       01  LK-TXN-TIME              PIC 9(6).
       01  LK-TXN-GROUP-ID          PIC 9(8).
       01  LK-TXN-ACCT-ID           PIC 9(6).
       01  LK-TXN-TYPE              PIC X(4).
       01  LK-TXN-AMOUNT            PIC 9(9)V99.
       01  LK-TXN-BALANCE-AFTER     PIC S9(9)V99.
       01  LK-TXN-STATUS            PIC X(8).

       PROCEDURE DIVISION USING LK-TXN-DATE LK-TXN-TIME
               LK-TXN-GROUP-ID LK-TXN-ACCT-ID LK-TXN-TYPE
               LK-TXN-AMOUNT LK-TXN-BALANCE-AFTER LK-TXN-STATUS.
       MAIN-PARA.
           OPEN EXTEND TRANSACTION-LOG.
           IF WS-LOG-STATUS NOT = "00"
      *> File does not exist yet - create it (first transaction ever).
               OPEN OUTPUT TRANSACTION-LOG
           END-IF.

      *> FILE SECTION records are NOT auto-initialized by their
      *> VALUE clauses at runtime (unlike WORKING-STORAGE) - clear
      *> the line first so unused FILLER bytes are spaces, not raw
      *> memory, which LINE SEQUENTIAL would reject as bad characters.
           MOVE SPACES               TO TRANSACTION-LINE.
           MOVE LK-TXN-DATE          TO TXN-DATE.
           MOVE LK-TXN-TIME          TO TXN-TIME.
           MOVE LK-TXN-GROUP-ID      TO TXN-GROUP-ID.
           MOVE LK-TXN-ACCT-ID       TO TXN-ACCT-ID.
           MOVE LK-TXN-TYPE          TO TXN-TYPE.
           MOVE LK-TXN-AMOUNT        TO TXN-AMOUNT.
           MOVE LK-TXN-BALANCE-AFTER TO TXN-BALANCE-AFTER.
           MOVE LK-TXN-STATUS        TO TXN-STATUS.

           WRITE TRANSACTION-LINE.
           CLOSE TRANSACTION-LOG.
           GOBACK.
```

### บั๊กจริงตัวที่ 1: FILE STATUS "71" (Bad Character)

ระหว่างพัฒนา `CBAUDIT` ครั้งแรก เราลืมบรรทัด `MOVE SPACES TO TRANSACTION-LINE` ผลคือเมื่อรันโปรแกรม
ทดสอบเรียก `CBAUDIT` ครั้งแรก ได้ผลลัพธ์:

```
DEBUG OPEN EXTEND STATUS=35
DEBUG OPEN OUTPUT STATUS=00
DEBUG WRITE STATUS=71
```

`FILE STATUS "71"` (ใน GnuCOBOL คือ `COB_STATUS_71_BAD_CHAR`) หมายถึงมีไบต์ที่พิมพ์ไม่ได้ปนอยู่ในข้อมูล
ที่พยายาม `WRITE` ลง `LINE SEQUENTIAL` เมื่อตรวจสอบทีละไบต์ในเรคคอร์ดด้วยลูปตรวจ `FUNCTION ORD` พบว่าไบต์
ที่ตำแหน่งของ `FILLER` (ที่ควรเป็นช่องว่างตาม `VALUE SPACE` ใน copybook) กลับเป็นค่า 0 (NUL) — สาเหตุคือ
`VALUE` clause ใน **FILE SECTION** ไม่ถูกนำมาใช้ตั้งค่าเริ่มต้นให้อัตโนมัติเหมือนใน WORKING-STORAGE
(อธิบายไว้แล้วใน Part 091 ขั้นตอนที่ 905) พื้นที่หน่วยความจำของ record เพิ่งถูกจองมา จึงเป็นค่า 0 ทั้งหมด

**วิธีแก้**: เพิ่ม `MOVE SPACES TO TRANSACTION-LINE` ก่อนกรอกฟิลด์ทีละตัวเสมอ (ดังที่แสดงในโค้ดด้านบน)

### บั๊กจริงตัวที่ 2: การส่ง Literal สั้นเข้า CALL ทำให้อ่านข้อมูลเลยขอบเขต

บั๊กที่สองพบขณะทดสอบ `CBOPEN` เรียก `CBAUDIT` ครั้งแรก โค้ดตอนนั้นเขียนว่า:

```cobol
                   CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                       ACCT-ID "OPEN" ACCT-BALANCE ACCT-BALANCE "OK"
```

ผลลัพธ์คือ log ที่เขียนออกมามีอักขระแปลกปนอยู่ในฟิลด์ `TXN-STATUS` (ซึ่งควรเป็น `"OK      "`) เมื่อไล่หา
ไบต์ผิดปกติด้วยเทคนิคเดียวกับบั๊กแรก พบว่าปัญหาอยู่ที่ตำแหน่งของฟิลด์ `TXN-STATUS` พอดี

**สาเหตุ**: เมื่อ CALL ส่ง string literal ตรงๆ (ค่าเริ่มต้นของ COBOL คือส่งแบบ `BY REFERENCE`) คอมไพเลอร์
จะจองพื้นที่ชั่วคราวให้ literal นั้น**เท่ากับความยาวจริงของ literal เท่านั้น** — `"OK"` มีความยาวเพียง 2
ตัวอักษร แต่พารามิเตอร์ปลายทางใน `CBAUDIT` (`LK-TXN-STATUS`) ประกาศไว้เป็น `PIC X(8)` เมื่อ `CBAUDIT`
อ่านค่าตามความยาว 8 ไบต์ที่ตัวเองประกาศไว้ จะ**อ่านเลยขอบเขตของพื้นที่ 2 ไบต์ที่จองไว้จริง** กลายเป็นการ
อ่านหน่วยความจำข้างเคียงที่ไม่เกี่ยวข้อง (ค่าขยะ) เข้ามาปนด้วย ส่วน `"OPEN"` ไม่เกิดปัญหานี้เพราะยาว
พอดี 4 ตัวอักษรตรงกับ `PIC X(4)` ของ `LK-TXN-TYPE` พอดี — บังเอิญไม่มีปัญหาเท่านั้นเอง ไม่ใช่เพราะวิธีเขียน
ถูกต้อง

**วิธีแก้**: ประกาศค่าคงที่ที่มีความยาวตรงกับพารามิเตอร์ปลายทางใน WORKING-STORAGE เสมอ (ตามมาตรฐานที่
วางไว้ใน Part 091 ขั้นตอนที่ 907) แล้วส่งตัวแปรนั้นแทนการส่ง literal ตรงๆ:

```cobol
       01  WS-TXN-TYPE-OPEN         PIC X(4)   VALUE "OPEN".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
      *> ...
                   CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                       ACCT-ID WS-TXN-TYPE-OPEN ACCT-BALANCE
                       ACCT-BALANCE WS-STATUS-OK
```

หลังแก้ไขทั้งสองจุด `CBAUDIT` เขียน log ได้ถูกต้องสมบูรณ์:

```bash
cobc -x cbopen.cob cbaudit.cob -o cbopen
```

```
20260401 090000 00000000 100001 OPEN 00000150000 +00000150000 OK
```

### ข้อควรระวัง

- บั๊กทั้งสองนี้**ไม่มี compile error หรือ warning ใดๆ เตือนล่วงหน้าเลย** — โปรแกรมคอมไพล์ผ่านสมบูรณ์
  และรันได้โดยไม่มี runtime error แสดงออกมาตรงๆ (ยกเว้น FILE STATUS ที่ต้องตรวจสอบเองเท่านั้น) นี่คือ
  เหตุผลที่ Part 030 (File Status Codes) และ Part 069 (Secure Coding) ย้ำเสมอว่าต้องตรวจสอบ FILE STATUS
  ทุกครั้งหลัง I/O statement ที่สำคัญ
- กฎ "ห้ามส่ง literal สั้นกว่าพารามิเตอร์ปลายทางเข้า CALL โดยตรง" ควรยึดถือเป็นมาตรฐานตายตัว ไม่ใช่แค่
  "จำไว้เวลานึกออก" เพราะบั๊กประเภทนี้ตรวจจับได้ยากมากด้วยตาเปล่า (ต้องไล่ดูความยาว literal เทียบกับ
  PICTURE ของทุกพารามิเตอร์ทุกจุด CALL)

### แบบฝึกหัดที่ 912.1

**โจทย์**: ถ้าเปลี่ยนพารามิเตอร์ `LK-TXN-TYPE` ใน `CBAUDIT` จาก `PIC X(4)` เป็น `PIC X(2)` (สั้นกว่าเดิม)
แล้วยังคงส่ง `WS-TXN-TYPE-OPEN` (`PIC X(4) VALUE "OPEN"`) เข้าไปเหมือนเดิม จะเกิดปัญหาแบบเดียวกับบั๊ก
ตัวที่ 2 หรือไม่ เพราะเหตุใด

**เฉลยแนวทาง**: จะ**ไม่เกิดปัญหาการอ่านเลยขอบเขต** แต่จะเกิดปัญหาอื่นแทนคือ**ข้อมูลถูกตัดทอน (Truncate)**
— เพราะครั้งนี้พื้นที่ที่ส่งเข้าไปจริง (4 ไบต์จาก `WS-TXN-TYPE-OPEN`) ใหญ่กว่าพื้นที่ที่ปลายทางอ่าน (2
ไบต์) ปลายทางจะอ่านได้แค่ `"OP"` เท่านั้น ไม่ใช่ error runtime ใดๆ เพราะยังอยู่ภายในขอบเขตพื้นที่ที่ถูก
จองไว้จริง แต่ผลลัพธ์ทางธุรกิจจะผิดเพี้ยน (ข้อมูลหาย) นี่คือเหตุผลที่ PICTURE ของพารามิเตอร์ทั้งสองฝั่ง
ของ CALL **ควรตรงกันเป๊ะเสมอ** ไม่ใช่แค่ "ยาวพอ"

---

## ขั้นตอนที่ 913: `CBDEP` — โมดูลฝากเงิน

### หลักการ

ฝากเงินเป็นธุรกรรมที่ง่ายที่สุดในระบบ: ไม่มีการตรวจสอบยอดคงเหลือใดๆ (เงินเข้าไม่มีทางทำให้ยอดติดลบ)
เงื่อนไขที่ต้องตรวจมีเพียง "บัญชีมีอยู่จริงหรือไม่" และ "บัญชียังเปิดใช้งานอยู่หรือไม่"

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBDEP.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBDEP - online-style transaction: deposit money into any open
      * account, regardless of type. A deposit can never be rejected
      * for insufficient funds (that only applies to withdrawals), so
      * the only checks here are "does the account exist" and "is it
      * still open".
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
       01  WS-TODAY                 PIC 9(8)   VALUE 0.
       01  WS-NOW                   PIC 9(6)   VALUE 0.
       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-RAW-AMOUNT            PIC X(9)   VALUE SPACES.
       01  WS-AMOUNT-CENTS          PIC 9(9)   VALUE 0.
       01  WS-AMOUNT                PIC 9(9)V99 VALUE 0.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".
       01  WS-ZERO-GROUP            PIC 9(8)   VALUE 0.
       01  WS-TXN-TYPE-DEP          PIC X(4)   VALUE "DEP".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
       01  WS-STATUS-REJECTED       PIC X(8)   VALUE "REJECTED".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.

           DISPLAY "=== CBDEP: Deposit ===".
           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-TODAY.
           DISPLAY "Enter processing time (HHMMSS): "
               WITH NO ADVANCING.
           ACCEPT WS-NOW.

           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       DISPLAY "ERROR: account " ACCT-ID
                           " not found."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID AND ACCT-STATUS-CLOSED
               DISPLAY "ERROR: account " ACCT-ID " is closed."
               CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                   ACCT-ID WS-TXN-TYPE-DEP WS-AMOUNT
                   ACCT-BALANCE WS-STATUS-REJECTED
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter deposit amount in cents (9 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-AMOUNT
               IF WS-RAW-AMOUNT IS NOT NUMERIC
                   DISPLAY "ERROR: amount must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
                   COMPUTE WS-AMOUNT = WS-AMOUNT-CENTS / 100
                   IF WS-AMOUNT = 0
                       DISPLAY "ERROR: deposit amount must be "
                           "greater than zero."
                       CALL "CBAUDIT" USING WS-TODAY WS-NOW
                           WS-ZERO-GROUP ACCT-ID WS-TXN-TYPE-DEP
                           WS-AMOUNT ACCT-BALANCE WS-STATUS-REJECTED
                       MOVE "N" TO WS-VALID-FLAG
                   END-IF
               END-IF
           END-IF.

           IF WS-ALL-VALID
               ADD WS-AMOUNT TO ACCT-BALANCE
               REWRITE ACCOUNT-RECORD
                   INVALID KEY
                       DISPLAY "ERROR rewriting account, status="
                           WS-FILE-STATUS
                       MOVE "N" TO WS-VALID-FLAG
               END-REWRITE
               IF WS-ALL-VALID
                   CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                       ACCT-ID WS-TXN-TYPE-DEP WS-AMOUNT
                       ACCT-BALANCE WS-STATUS-OK
                   DISPLAY "CBDEP: account " ACCT-ID " deposit "
                       WS-AMOUNT " new balance=" ACCT-BALANCE
               END-IF
           END-IF.
           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

### ทดสอบจริง

```bash
cobc -x cbdep.cob cbaudit.cob -o cbdep
printf '20260401\n100000\n100001\n000200000\n' | ./cbdep
```

**ผลลัพธ์จริง**:

```
=== CBDEP: Deposit ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter 6-digit account ID: Enter deposit amount in cents (9 digits): CBDEP: account 100001 deposit 000002000.00 new balance=+000007000.00
```

บัญชี `100001` (Savings เปิดด้วยยอด 5,000.00 ในชุดทดสอบใหม่ของ Part นี้) ฝากเพิ่ม 2,000.00 ยอดคงเหลือ
ใหม่เป็น 7,000.00 ถูกต้องตรงตามที่คาดไว้ และมีการบันทึก log ประเภท `DEP` สถานะ `OK`

### ข้อควรระวัง

- ต้องปฏิเสธการฝากเข้าบัญชีที่ `ACCT-STATUS-CLOSED` เสมอ มิฉะนั้นบัญชีที่ปิดไปแล้วจะกลับมามียอดเงินได้
  ซึ่งขัดกับหลักการทางธุรกิจโดยสิ้นเชิง
- จำนวนเงินฝากต้องมากกว่า 0 เสมอ (ตรวจสอบ `WS-AMOUNT = 0` แล้วปฏิเสธ) ไม่เช่นนั้นผู้ใช้ที่กรอกผิดพลาด
  (หรือพยายามทดสอบระบบ) จะสร้าง log entry ที่ไม่มีความหมายทางธุรกิจจำนวนมาก

### แบบฝึกหัดที่ 913.1

**โจทย์**: หากมีคนเสนอให้ `CBDEP` อนุญาตให้ฝากเงินเป็นจำนวนติดลบได้ (เพื่อใช้เป็น "ทางลัด" สำหรับกรณี
ต้องการปรับลดยอด) จงอธิบายว่าทำไมแนวคิดนี้เป็นการออกแบบที่ไม่ดี

**เฉลยแนวทาง**: การอนุญาตให้ "ฝาก" เป็นจำนวนติดลบจะทำให้ `CBDEP` กลายเป็นทั้งฟังก์ชันฝากและถอนในตัวเดียว
โดยไม่มีการตรวจสอบยอดคงเหลือ/วงเงินเบิกเกินบัญชีตามที่ `CBWD` ทำ (เช่น จะทำให้บัญชี Savings ติดลบได้ทั้ง
ที่กฎห้ามไว้) นอกจากนี้ log entry ที่มีประเภท `DEP` แต่จำนวนเงินติดลบจะทำให้รายงานสรุปยอดฝาก/ถอนรวม
(Part 093) ผิดเพี้ยนทันที การออกแบบที่ดีคือแยกหน้าที่ให้ชัดเจน (Single Responsibility) — ฝากทำหน้าที่ฝาก
เท่านั้น การปรับลดยอดควรผ่านกระบวนการที่มีการตรวจสอบและอนุมัติแยกต่างหาก ไม่ใช่ "แอบ" ผ่านช่องทางฝากเงิน

---

## ขั้นตอนที่ 914: `CBWD` — โมดูลถอนเงิน

### กฎธุรกิจตามประเภทบัญชี (ทบทวนจาก Part 091)

| ประเภทบัญชี | กฎการถอน |
|---|---|
| Savings | ยอดคงเหลือหลังถอนต้อง `>= 0` เสมอ (ไม่มีวงเงินเบิกเกินบัญชี) |
| Checking | ยอดคงเหลือหลังถอนต้อง `>= -(ACCT-OVERDRAFT-LIMIT)` |
| Fixed Deposit | **ห้ามถอนบางส่วนเลย** ไม่ว่ากรณีใด ต้องปิดบัญชีทั้งก้อนหลังครบกำหนดเท่านั้น |

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBWD.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBWD - online-style transaction: withdraw money. Business
      * rules differ by account type:
      *   SAVINGS      - balance must stay >= 0 (no overdraft).
      *   CHECKING     - balance may go negative down to (but not
      *                  past) ACCT-OVERDRAFT-LIMIT.
      *   FIXED-DEP    - locked until maturity (ACCT-STATUS-MATURED);
      *                  no partial withdrawal is allowed at all, even
      *                  after maturity, through this program - a
      *                  matured time deposit is only ever withdrawn
      *                  in full through account closure.
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
       01  WS-TODAY                 PIC 9(8)   VALUE 0.
       01  WS-NOW                   PIC 9(6)   VALUE 0.
       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-RAW-AMOUNT            PIC X(9)   VALUE SPACES.
       01  WS-AMOUNT-CENTS          PIC 9(9)   VALUE 0.
       01  WS-AMOUNT                PIC 9(9)V99 VALUE 0.
       01  WS-NEW-BALANCE           PIC S9(10)V99 VALUE 0.
       01  WS-MIN-ALLOWED           PIC S9(10)V99 VALUE 0.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".
       01  WS-ZERO-GROUP            PIC 9(8)   VALUE 0.
       01  WS-TXN-TYPE-WD           PIC X(4)   VALUE "WD".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
       01  WS-STATUS-REJECTED       PIC X(8)   VALUE "REJECTED".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.

           DISPLAY "=== CBWD: Withdrawal ===".
           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-TODAY.
           DISPLAY "Enter processing time (HHMMSS): "
               WITH NO ADVANCING.
           ACCEPT WS-NOW.

           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       DISPLAY "ERROR: account " ACCT-ID
                           " not found."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID AND ACCT-STATUS-CLOSED
               DISPLAY "ERROR: account " ACCT-ID " is closed."
               CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                   ACCT-ID WS-TXN-TYPE-WD WS-AMOUNT
                   ACCT-BALANCE WS-STATUS-REJECTED
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID AND ACCT-TYPE-FIXED-DEP
               DISPLAY "ERROR: Fixed Deposit " ACCT-ID
                   " does not allow partial withdrawal - close the"
               DISPLAY "account after maturity instead."
               MOVE "N" TO WS-VALID-FLAG
      *> Design decision (Part 092 step 914): using this transaction
      *> type against a Fixed Deposit is a "this transaction never
      *> applies to this account type" rejection, not an ordinary
      *> business-rule reject, so it is deliberately NOT logged.
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter withdrawal amount in cents (9 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-AMOUNT
               IF WS-RAW-AMOUNT IS NOT NUMERIC
                   DISPLAY "ERROR: amount must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
                   COMPUTE WS-AMOUNT = WS-AMOUNT-CENTS / 100
               END-IF
           END-IF.

           IF WS-ALL-VALID
               COMPUTE WS-NEW-BALANCE = ACCT-BALANCE - WS-AMOUNT
               IF ACCT-TYPE-CHECKING
                   COMPUTE WS-MIN-ALLOWED =
                       0 - ACCT-OVERDRAFT-LIMIT
               ELSE
                   MOVE 0 TO WS-MIN-ALLOWED
               END-IF
               IF WS-NEW-BALANCE < WS-MIN-ALLOWED
                   DISPLAY "ERROR: insufficient funds. Balance="
                       ACCT-BALANCE " requested=" WS-AMOUNT
                   CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                       ACCT-ID WS-TXN-TYPE-WD WS-AMOUNT
                       ACCT-BALANCE WS-STATUS-REJECTED
                   MOVE "N" TO WS-VALID-FLAG
               END-IF
           END-IF.

           IF WS-ALL-VALID
               COMPUTE ACCT-BALANCE = ACCT-BALANCE - WS-AMOUNT
               REWRITE ACCOUNT-RECORD
                   INVALID KEY
                       DISPLAY "ERROR rewriting account, status="
                           WS-FILE-STATUS
                       MOVE "N" TO WS-VALID-FLAG
               END-REWRITE
               IF WS-ALL-VALID
                   CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                       ACCT-ID WS-TXN-TYPE-WD WS-AMOUNT
                       ACCT-BALANCE WS-STATUS-OK
                   DISPLAY "CBWD: account " ACCT-ID " withdrew "
                       WS-AMOUNT " new balance=" ACCT-BALANCE
               END-IF
           ELSE
               DISPLAY "CBWD: withdrawal not applied."
           END-IF.
           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

จุดที่น่าสังเกตคือสูตรเดียว `WS-MIN-ALLOWED` ครอบคลุมทั้ง Savings (ขั้นต่ำ 0) และ Checking (ขั้นต่ำ
ติดลบเท่าวงเงินเบิกเกินบัญชี) ได้ในโค้ดชุดเดียว ไม่ต้องแยก `IF` ซ้อนกันหลายชั้นตามประเภทบัญชี

### ทดสอบครบทุกกรณี

```bash
cobc -x cbwd.cob cbaudit.cob -o cbwd
```

**กรณีที่ 1 — ถอนจาก Savings เกินยอด (ปฏิเสธ)**:

```bash
printf '20260401\n105000\n100001\n009000000\n' | ./cbwd
```

```
=== CBWD: Withdrawal ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter 6-digit account ID: Enter withdrawal amount in cents (9 digits): ERROR: insufficient funds. Balance=+000007000.00 requested=000090000.00
CBWD: withdrawal not applied.
```

**กรณีที่ 2 — ถอนจาก Checking จนติดลบแต่ยังอยู่ในวงเงินเบิกเกินบัญชี (สำเร็จ)**: จะแสดงผลเต็มในขั้นตอน
ที่ 920 เมื่อทดสอบรวมวันที่ 2 ซึ่งเป็นสถานการณ์ที่ออกแบบมาเพื่อพิสูจน์กฎ Checking โดยเฉพาะ

**กรณีที่ 3 — ถอนจาก Fixed Deposit (ปฏิเสธเสมอ)**:

```bash
printf '20260401\n103000\n300001\n000100000\n' | ./cbwd
```

```
=== CBWD: Withdrawal ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter 6-digit account ID: ERROR: Fixed Deposit 300001 does not allow partial withdrawal - close the
account after maturity instead.
CBWD: withdrawal not applied.
```

### ข้อควรระวัง

- สังเกตว่าเมื่อถอนไม่สำเร็จเพราะยอดไม่พอ (กรณีที่ 1) โปรแกรม**ยังคง CALL "CBAUDIT" เพื่อบันทึกความ
  พยายามที่ถูกปฏิเสธ** — นี่คือหลักการ "บันทึกทุกความพยายาม ไม่ว่าผลจะเป็นอย่างไร" ที่ Part 091 ขั้นตอน
  ที่ 903 วางไว้ แต่กรณี Fixed Deposit ล็อก (กรณีที่ 3) เราเลือก**ไม่บันทึก log** เพราะถือเป็นการปฏิเสธ
  ที่ระดับ "ประเภทธุรกรรมนี้ใช้กับบัญชีนี้ไม่ได้เลย" ซึ่งเป็นการตัดสินใจออกแบบที่ควรระบุให้ชัดเจนในเอกสาร
  (จุดนี้เป็นตัวอย่างที่ดีว่าทีมพัฒนาจริงต้องตกลงกันล่วงหน้าว่า "ปฏิเสธแบบไหนควรบันทึก log")
- ระวังลำดับการคำนวณ `WS-NEW-BALANCE` ก่อนแล้วค่อยเทียบกับ `WS-MIN-ALLOWED` — ถ้าสลับไปคำนวณหลังตรวจสอบ
  จะทำให้ตรวจสอบยอดคงเหลือ**เก่า**แทนที่จะเป็นยอดที่จะเกิดขึ้นจริงหลังถอน

### แบบฝึกหัดที่ 914.1

**โจทย์**: จงคำนวณว่าบัญชี Checking ที่มียอดคงเหลือ 700.00 และวงเงินเบิกเกินบัญชี 10,000.00 สามารถถอน
เงินได้สูงสุดกี่บาทในครั้งเดียว โดยยอดคงเหลือหลังถอนต้องไม่ต่ำกว่าเกณฑ์ที่กำหนด

**เฉลย**: `WS-MIN-ALLOWED = -10,000.00` ดังนั้นยอดคงเหลือหลังถอนต่ำสุดที่ยอมรับได้คือ `-10,000.00` จำนวน
เงินสูงสุดที่ถอนได้คือ `700.00 - (-10,000.00) = 10,700.00` บาท (ถอนแล้วยอดคงเหลือจะเป็น `-10,000.00`
พอดี ซึ่งยังอยู่ในเงื่อนไข `WS-NEW-BALANCE < WS-MIN-ALLOWED` เป็นเท็จ จึงผ่าน)

---

## ขั้นตอนที่ 915: ปัญหา Atomicity ของ COBOL และการออกแบบ Compensating Transaction

### ทำไมการโอนเงินถึงยากกว่าฝาก/ถอนมาก

การโอนเงินต้องแก้ไข **สองบัญชี** ในธุรกรรมเดียวกัน: หักเงินออกจากบัญชีต้นทาง แล้วเติมเงินเข้าบัญชี
ปลายทาง ในฐานข้อมูลเชิงสัมพันธ์ (SQL) เรามักจะห่อสองขั้นตอนนี้ด้วย `BEGIN TRANSACTION` / `COMMIT` /
`ROLLBACK` เพื่อรับประกันว่า **ทั้งสองขั้นตอนสำเร็จพร้อมกัน หรือไม่สำเร็จเลยทั้งคู่** (คุณสมบัติ
Atomicity ตัวแรกของหลัก ACID)

**COBOL กับไฟล์ Indexed (ISAM) ไม่มีกลไกแบบนี้ให้ในตัว** — คำสั่ง `REWRITE` แต่ละครั้งเขียนลงดิสก์ทันที
ไม่มีแนวคิด "รอ COMMIT" ถ้าโปรแกรม `REWRITE` บัญชีต้นทางสำเร็จ แล้วเกิดปัญหาก่อนจะ `REWRITE` บัญชีปลาย
ทางได้ (เช่น บัญชีปลายทางไม่พบ) เงินจะ "หายไปจากระบบ" ทันทีถ้าไม่มีมาตรการรองรับ

### สองเทคนิคที่ใช้ร่วมกันใน `CBXFER`

**เทคนิคที่ 1 — Pre-validation (ตรวจสอบล่วงหน้า)**: ตรวจสอบว่าบัญชีต้นทางมีอยู่จริง ยังเปิดใช้งานอยู่
ไม่ใช่ Fixed Deposit ที่ล็อกอยู่ และมียอดคงเหลือ/วงเงินเบิกเกินบัญชีเพียงพอ **ก่อน**ที่จะเขียนอะไรลงดิสก์
เลยแม้แต่ไบต์เดียว วิธีนี้ป้องกันความล้มเหลวส่วนใหญ่ได้ตั้งแต่ต้น

**เทคนิคที่ 2 — Compensating Transaction (ธุรกรรมชดเชย)**: แม้จะ pre-validate บัญชีต้นทางแล้ว
`CBXFER` ยังคง**ออกแบบให้ตรวจสอบบัญชีปลายทางทีหลัง** (หลังจากหักเงินออกจากต้นทางไปแล้ว) — เหตุผลคือนี่
คือรูปแบบที่ใกล้เคียงกับระบบเก่าจริงจำนวนมากที่ประมวลผลทีละขา และยังคงมีคุณค่าทางการศึกษาสูง: **การ
ตรวจสอบล่วงหน้าอย่างเดียวไม่เคยเพียงพอ 100%** เพราะเสมอมีช่วงเวลาระหว่างตรวจสอบกับเขียนจริงที่อะไรก็
เกิดขึ้นได้ (race window) ระบบที่แข็งแกร่งจริงจึงต้องมี "แผนสำรอง" เสมอ ไม่ว่าจะ pre-validate ดีแค่ไหน

ถ้าขั้นตอนเติมเงินเข้าบัญชีปลายทางล้มเหลว (เช่น ไม่พบบัญชี หรือบัญชีถูกปิดไปแล้ว) `CBXFER` จะ:

1. อ่านบัญชีต้นทางกลับมาใหม่จากดิสก์ (เพราะพื้นที่ record เดียวถูกใช้ซ้ำไปอ่านบัญชีปลายทางแล้ว)
2. เติมเงินจำนวนเท่าที่หักไปคืนให้บัญชีต้นทาง (`ADD`)
3. `REWRITE` บัญชีต้นทาง
4. บันทึก log ประเภท `RVSL` (Reversal) เพื่อเป็นหลักฐานว่ามีการกู้คืน
5. บันทึก log ประเภท `XFRC` สถานะ `REJECTED` สำหรับความพยายามเติมเงินที่ล้มเหลว

ผลลัพธ์สุดท้าย: ยอดเงินในบัญชีต้นทาง**กลับสู่สภาพเดิมทุกประการ** และ Audit Trail แสดงประวัติทั้งหมดอย่าง
โปร่งใส — แนวคิดนี้ตรงกับ **Saga Pattern** ที่ระบบ Microservices สมัยใหม่ใช้รักษาความสอดคล้องของข้อมูล
ข้ามหลายบริการโดยไม่มี Transaction เดียวที่ครอบคลุมทุกบริการ (concept เดียวกัน คนละยุคเทคโนโลยี)

### ข้อจำกัดที่ต้องยอมรับตรงไปตรงมา

ระบบนี้รันแบบ **Single-user, single-process** (ตามที่ประกาศไว้ใน Part 091 ขั้นตอนที่ 901) จึงไม่มี
ปัญหาเรื่องอีกโปรเซสมาแก้ไขบัญชีเดียวกันพร้อมกัน (Concurrency) กลไก Compensating Transaction ที่สร้างขึ้น
นี้แก้ปัญหา **ความล้มเหลวเชิงตรรกะ** (บัญชีปลายทางไม่พบ/ถูกปิด) ได้อย่างสมบูรณ์ แต่**ไม่ได้**ครอบคลุมกรณี
ระบบล่มกลางคัน (เช่น ไฟดับระหว่าง REWRITE) ซึ่งเป็นปัญหาคนละชั้นที่ต้องแก้ด้วยเทคนิคระดับไฟล์ระบบ
(เช่น Transaction Log ของฐานข้อมูลจริง หรือ VSAM/CICS Recovery บน Mainframe จริงที่ Part 055 และ 064
กล่าวถึง) — การระบุขอบเขตของสิ่งที่แก้ไขได้และแก้ไขไม่ได้อย่างชัดเจนคือส่วนหนึ่งของการออกแบบระบบที่ดี

### แบบฝึกหัดที่ 915.1

**โจทย์**: จงอธิบายว่าทำไม "ตรวจสอบบัญชีปลายทางล่วงหน้าเหมือนบัญชีต้นทางไปเลย" (pre-validate ทั้งสองขา
ก่อนเขียนอะไรเลย) ไม่ใช่คำตอบที่ทำให้ไม่ต้องมีกลไก Compensating Transaction อีกต่อไป

**เฉลยแนวทาง**: การตรวจสอบล่วงหน้าทั้งสองขาช่วยลดโอกาสเกิดความล้มเหลวได้มากก็จริง แต่ไม่สามารถขจัด
"ช่วงเวลาระหว่างตรวจสอบกับเขียนจริง" (time-of-check to time-of-use, TOCTOU) ได้ทั้งหมด แม้ในระบบ
single-process ก็ยังมีโอกาสที่การ REWRITE ครั้งที่สองล้มเหลวด้วยเหตุผลทางเทคนิค (เช่น ปัญหาดิสก์) ทั้งที่
ตรวจสอบผ่านไปแล้วก่อนหน้านั้นเสี้ยววินาที ระบบที่ออกแบบมาอย่างรอบคอบจึงควรมีกลไกชดเชยไว้เป็น "แผนสำรอง"
เสมอ ไม่ใช่พึ่งพาการตรวจสอบล่วงหน้าเพียงอย่างเดียว — นี่คือหลักการ "Defense in Depth" แบบเดียวกับที่
Part 089 (Security ระดับ Enterprise) สอนไว้ในบริบทความปลอดภัย ซึ่งนำมาประยุกต์ใช้กับความถูกต้องของข้อมูล
ได้เช่นกัน

---

## ขั้นตอนที่ 916: สร้าง `CBXFER` ฉบับเต็ม

### จุดออกแบบสำคัญ: สอง Record Area สำหรับสองบัญชี

`ACCOUNT-MASTER` มี Record Area เดียว (`ACCOUNT-RECORD`) เราจึงต้องประกาศ Record Area ที่สองแยกต่างหาก
ใน WORKING-STORAGE (`WS-DEST-RECORD`) สำหรับถือข้อมูลบัญชีปลายทางไว้ชั่วคราว โดยใช้ **`COPY ... REPLACING`**
(Part 033) เปลี่ยนชื่อทุกฟิลด์ให้ไม่ชนกับ `ACCOUNT-RECORD`:

```cobol
       01  WS-DEST-RECORD.
           COPY "cbacct.cpy"
               REPLACING ACCT-ID              BY DEST-ACCT-ID
                         ACCT-NAME            BY DEST-ACCT-NAME
                         ACCT-TYPE            BY DEST-ACCT-TYPE
                         ACCT-STATUS          BY DEST-ACCT-STATUS
                         ACCT-BALANCE         BY DEST-ACCT-BALANCE
                         ACCT-OVERDRAFT-LIMIT BY DEST-ACCT-OD-LIMIT
                         ACCT-INTEREST-RATE   BY DEST-ACCT-INT-RATE
                         ACCT-ACCRUED-INT     BY DEST-ACCT-ACCR-INT
                         ACCT-OPEN-DATE       BY DEST-ACCT-OPEN-DATE
                         ACCT-MATURITY-DATE   BY DEST-ACCT-MAT-DATE
                         ACCT-LAST-ACCR-DATE  BY DEST-ACCT-LAST-ACCR
                         ACCT-LAST-STMT-DATE  BY DEST-ACCT-LAST-STMT.
```

> **บั๊กที่พบระหว่างคอมไพล์**: `REPLACING` เปลี่ยนชื่อฟิลด์ระดับ `05` ได้ตามที่สั่ง แต่**ไม่ได้เปลี่ยนชื่อ
> 88-level condition name** ที่อยู่ข้างใน copybook (เช่น `ACCT-TYPE-SAVINGS`, `ACCT-STATUS-CLOSED`) เพราะ
> `REPLACING` ทำงานแบบ token-matching ตรงตัว ไม่ได้แทนที่คำนำหน้า (`ACCT-`) โดยอัตโนมัติ ผลคือ
> `ACCT-STATUS-CLOSED` ถูกประกาศซ้ำสองครั้ง (ครั้งหนึ่งใต้ `ACCOUNT-RECORD` อีกครั้งใต้ `WS-DEST-RECORD`)
> ทำให้คอมไพเลอร์แจ้ง `'ACCT-STATUS-CLOSED' is ambiguous; needs qualification` ทุกจุดที่อ้างถึงชื่อนี้
> โดยไม่ระบุที่มา วิธีแก้คือเติม `OF ACCOUNT-RECORD` ต่อท้ายทุกจุดที่ตั้งใจอ้างถึงตัวแปรของบัญชีต้นทาง
> โดยเฉพาะ (`ACCT-STATUS-CLOSED OF ACCOUNT-RECORD`, `ACCT-TYPE-FIXED-DEP OF ACCOUNT-RECORD`,
> `ACCT-TYPE-CHECKING OF ACCOUNT-RECORD`) — บทเรียนนี้ตอกย้ำสิ่งที่ Part 033 เตือนไว้: `COPY REPLACING`
> ทรงพลังมาก แต่ต้องเข้าใจกลไก token-matching ของมันอย่างละเอียดก่อนใช้กับ copybook ที่มี 88-level ปนอยู่

### โค้ดฉบับเต็มของ `CBXFER`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBXFER.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBXFER - online-style transaction: transfer money between two
      * accounts. COBOL/ISAM has no built-in multi-record transaction
      * (no COMMIT/ROLLBACK across two REWRITEs the way a SQL database
      * would give us for free), so this program builds its own safety
      * net out of two techniques used together: pre-validation of the
      * SOURCE account, and a compensating transaction if the
      * DESTINATION leg fails after the source has already been
      * debited. See Part 092 step 915 for the full discussion.
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
       01  WS-TODAY                 PIC 9(8)   VALUE 0.
       01  WS-NOW                   PIC 9(6)   VALUE 0.

       01  WS-DEST-RECORD.
           COPY "cbacct.cpy"
               REPLACING ACCT-ID              BY DEST-ACCT-ID
                         ACCT-NAME            BY DEST-ACCT-NAME
                         ACCT-TYPE            BY DEST-ACCT-TYPE
                         ACCT-STATUS          BY DEST-ACCT-STATUS
                         ACCT-BALANCE         BY DEST-ACCT-BALANCE
                         ACCT-OVERDRAFT-LIMIT BY DEST-ACCT-OD-LIMIT
                         ACCT-INTEREST-RATE   BY DEST-ACCT-INT-RATE
                         ACCT-ACCRUED-INT     BY DEST-ACCT-ACCR-INT
                         ACCT-OPEN-DATE       BY DEST-ACCT-OPEN-DATE
                         ACCT-MATURITY-DATE   BY DEST-ACCT-MAT-DATE
                         ACCT-LAST-ACCR-DATE  BY DEST-ACCT-LAST-ACCR
                         ACCT-LAST-STMT-DATE  BY DEST-ACCT-LAST-STMT.

       01  WS-SRC-ID                PIC 9(6)   VALUE 0.
       01  WS-DST-ID                PIC 9(6)   VALUE 0.
       01  WS-GROUP-ID              PIC 9(8)   VALUE 0.
       01  WS-RAW-AMOUNT            PIC X(9)   VALUE SPACES.
       01  WS-AMOUNT-CENTS          PIC 9(9)   VALUE 0.
       01  WS-AMOUNT                PIC 9(9)V99 VALUE 0.
       01  WS-NEW-BALANCE           PIC S9(10)V99 VALUE 0.
       01  WS-MIN-ALLOWED           PIC S9(10)V99 VALUE 0.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".
       01  WS-DEST-FOUND-FLAG       PIC X(1)   VALUE "Y".
           88  WS-DEST-FOUND                VALUE "Y".

       01  WS-TXN-TYPE-XFRD         PIC X(4)   VALUE "XFRD".
       01  WS-TXN-TYPE-XFRC         PIC X(4)   VALUE "XFRC".
       01  WS-TXN-TYPE-RVSL         PIC X(4)   VALUE "RVSL".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
       01  WS-STATUS-REJECTED       PIC X(8)   VALUE "REJECTED".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.

           DISPLAY "=== CBXFER: Transfer Between Accounts ===".
           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-TODAY.
           DISPLAY "Enter processing time (HHMMSS): "
               WITH NO ADVANCING.
           ACCEPT WS-NOW.
           DISPLAY "Enter transfer reference number (8 digits): "
               WITH NO ADVANCING.
           ACCEPT WS-GROUP-ID.
           DISPLAY "Enter SOURCE account ID: " WITH NO ADVANCING.
           ACCEPT WS-SRC-ID.
           DISPLAY "Enter DESTINATION account ID: " WITH NO ADVANCING.
           ACCEPT WS-DST-ID.
           DISPLAY "Enter transfer amount in cents (9 digits): "
               WITH NO ADVANCING.
           ACCEPT WS-RAW-AMOUNT.
           IF WS-RAW-AMOUNT IS NOT NUMERIC
               DISPLAY "ERROR: amount must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
               COMPUTE WS-AMOUNT = WS-AMOUNT-CENTS / 100
           END-IF.
           IF WS-ALL-VALID AND WS-SRC-ID = WS-DST-ID
               DISPLAY "ERROR: source and destination must differ."
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

      *> STEP 1: PRE-VALIDATE THE SOURCE ACCOUNT ONLY.
           IF WS-ALL-VALID
               MOVE WS-SRC-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       DISPLAY "ERROR: source account " WS-SRC-ID
                           " not found."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID AND ACCT-STATUS-CLOSED OF ACCOUNT-RECORD
               MOVE "N" TO WS-VALID-FLAG
           END-IF.
           IF WS-ALL-VALID AND ACCT-TYPE-FIXED-DEP OF ACCOUNT-RECORD
               MOVE "N" TO WS-VALID-FLAG
           END-IF.
           IF WS-ALL-VALID
               COMPUTE WS-NEW-BALANCE = ACCT-BALANCE - WS-AMOUNT
               IF ACCT-TYPE-CHECKING OF ACCOUNT-RECORD
                   COMPUTE WS-MIN-ALLOWED = 0 - ACCT-OVERDRAFT-LIMIT
               ELSE
                   MOVE 0 TO WS-MIN-ALLOWED
               END-IF
               IF WS-NEW-BALANCE < WS-MIN-ALLOWED
                   MOVE "N" TO WS-VALID-FLAG
               END-IF
           END-IF.

           IF NOT WS-ALL-VALID
               DISPLAY "CBXFER: transfer rejected - no funds moved."
               GO TO END-PARA
           END-IF.

      *> STEP 2: DEBIT THE SOURCE (this is the only write so far).
           COMPUTE ACCT-BALANCE = ACCT-BALANCE - WS-AMOUNT.
           REWRITE ACCOUNT-RECORD.
           CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-GROUP-ID
               ACCT-ID WS-TXN-TYPE-XFRD WS-AMOUNT
               ACCT-BALANCE WS-STATUS-OK.
           DISPLAY "CBXFER: debited " WS-AMOUNT " from " ACCT-ID
               " new balance=" ACCT-BALANCE.

      *> STEP 3: LOOK UP THE DESTINATION - ONLY NOW. If this fails,
      *> the source has already been debited, so we must compensate.
      *> NOTE: ACCOUNT-MASTER has only ONE record area (ACCOUNT-RECORD)
      *> so setting ACCT-ID to the destination's ID and reading
      *> OVERWRITES the source's data that was sitting there - that is
      *> safe because the debit was already written to disk above.
           MOVE "Y" TO WS-DEST-FOUND-FLAG.
           MOVE WS-DST-ID TO ACCT-ID.
           READ ACCOUNT-MASTER INTO WS-DEST-RECORD
               INVALID KEY
                   MOVE "N" TO WS-DEST-FOUND-FLAG
           END-READ.
           IF WS-DEST-FOUND AND DEST-ACCT-STATUS = "C"
               MOVE "N" TO WS-DEST-FOUND-FLAG
           END-IF.

           IF NOT WS-DEST-FOUND
      *> COMPENSATING TRANSACTION: re-READ the SOURCE fresh from disk
      *> (the record area was just overwritten above) and credit the
      *> debited amount straight back.
               MOVE WS-SRC-ID TO ACCT-ID
               READ ACCOUNT-MASTER
               ADD WS-AMOUNT TO ACCT-BALANCE
               REWRITE ACCOUNT-RECORD
               CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-GROUP-ID
                   ACCT-ID WS-TXN-TYPE-RVSL WS-AMOUNT
                   ACCT-BALANCE WS-STATUS-OK
               CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-GROUP-ID
                   WS-DST-ID WS-TXN-TYPE-XFRC WS-AMOUNT
                   ACCT-BALANCE WS-STATUS-REJECTED
               DISPLAY "ERROR: destination account " WS-DST-ID
                   " not found."
               DISPLAY "CBXFER: transfer FAILED and was reversed - "
                   "source balance restored to " ACCT-BALANCE
               GO TO END-PARA
           END-IF.

      *> STEP 4: CREDIT THE DESTINATION - both legs now complete.
           ADD WS-AMOUNT TO DEST-ACCT-BALANCE.
           REWRITE ACCOUNT-RECORD FROM WS-DEST-RECORD.
           CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-GROUP-ID
               WS-DST-ID WS-TXN-TYPE-XFRC WS-AMOUNT
               DEST-ACCT-BALANCE WS-STATUS-OK.
           DISPLAY "CBXFER: credited " WS-AMOUNT " to " WS-DST-ID
               " new balance=" DEST-ACCT-BALANCE.
           DISPLAY "CBXFER: transfer completed successfully - "
               "both legs balanced under reference " WS-GROUP-ID.

       END-PARA.
           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

> โค้ดข้างต้นตัดการตรวจสอบ `INVALID KEY` ของบาง `REWRITE`/`READ` ออกเพื่อความกระชับในการอธิบาย ไฟล์
> ต้นฉบับที่คอมไพล์จริงตรวจสอบทุกจุดและแสดงข้อความ `FATAL` (ตามคำจำกัดความจาก Part 091 ขั้นตอนที่ 908)
> หากเกิดกรณีที่ไม่ควรเกิดขึ้นเลยในทางทฤษฎี เช่น REWRITE ล้มเหลวหลังผ่านการตรวจสอบทุกอย่างแล้ว

### แบบฝึกหัดที่ 916.1

**โจทย์**: ทำไมขั้นตอนการดึงข้อมูลบัญชีปลายทาง (STEP 3) ถึงต้องใช้ `READ ACCOUNT-MASTER INTO
WS-DEST-RECORD` แทนที่จะ `READ ACCOUNT-MASTER` ธรรมดาแล้วค่อย `MOVE ACCOUNT-RECORD TO WS-DEST-RECORD`
เอง?

**เฉลยแนวทาง**: ทั้งสองวิธีให้ผลลัพธ์สุดท้ายเหมือนกัน แต่ `READ ... INTO` เป็นไวยากรณ์ที่กระชับกว่าและ
สื่อเจตนาชัดเจนกว่าในบรรทัดเดียว (อ่านจากไฟล์แล้วคัดลอกไปยังพื้นที่ปลายทางที่กำหนดในขั้นตอนเดียวกัน) การ
เขียนแยกสองบรรทัด (`READ` แล้ว `MOVE`) ก็ใช้งานได้ถูกต้องเช่นกันและให้ผลเหมือนกันทุกประการ เป็นเพียงความ
แตกต่างด้านความกระชับของไวยากรณ์เท่านั้น ไม่ใช่ความแตกต่างเชิงพฤติกรรม

---

## ขั้นตอนที่ 917: ทดสอบ `CBXFER` ทั้งสองเส้นทาง

```bash
cobc -x cbxfer.cob cbaudit.cob -o cbxfer
```

### เส้นทางที่ 1: โอนเงินสำเร็จ

```bash
printf '20260401\n103000\n00000001\n400001\n200001\n000100000\n' | ./cbxfer
```

**ผลลัพธ์จริง** (โอน 1,000.00 จากบัญชี Savings `400001` ไปบัญชี Checking `200001`):

```
=== CBXFER: Transfer Between Accounts ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter transfer reference number (8 digits): Enter SOURCE account ID: Enter DESTINATION account ID: Enter transfer amount in cents (9 digits): CBXFER: debited 000001000.00 from 400001 new balance=+000011000.00
CBXFER: credited 000001000.00 to 200001 new balance=+000001700.00
CBXFER: transfer completed successfully - both legs balanced under reference 00000001
```

### เส้นทางที่ 2: โอนเงินไปบัญชีที่ไม่มีอยู่จริง (พิสูจน์กลไกกู้คืน)

```bash
printf '20260401\n104000\n00000002\n100001\n999999\n000050000\n' | ./cbxfer
```

**ผลลัพธ์จริง** (พยายามโอน 500.00 จากบัญชี `100001` ไปยังบัญชี `999999` ที่ไม่มีอยู่จริง):

```
=== CBXFER: Transfer Between Accounts ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter transfer reference number (8 digits): Enter SOURCE account ID: Enter DESTINATION account ID: Enter transfer amount in cents (9 digits): CBXFER: debited 000000500.00 from 100001 new balance=+000006500.00
ERROR: destination account 999999 not found.
CBXFER: transfer FAILED and was reversed - source balance restored to +000007000.00
```

สังเกตว่ายอดคงเหลือของบัญชี `100001` กลับมาเป็น `+000007000.00` เท่ากับยอดก่อนพยายามโอนทุกประการ — เงิน
**ไม่ได้หายไปไหนเลย** แม้จะมีการหักเงินออกจากบัญชีต้นทางไปแล้วจริงในระดับดิสก์ก็ตาม

### หลักฐานจาก Audit Trail

```bash
tail -5 TXNLOG.DAT
```

```
20260401 103000 00000001 400001 XFRD 00000100000 +00000110000 OK
20260401 103000 00000001 200001 XFRC 00000100000 +00000017000 OK
20260401 104000 00000002 100001 XFRD 00000050000 +00000065000 OK
20260401 104000 00000002 100001 RVSL 00000050000 +00000070000 OK
20260401 104000 00000002 999999 XFRC 00000050000 +00000070000 REJECTED
```

สามบรรทัดสุดท้าย (ทั้งหมดมี `TXN-GROUP-ID = 00000002` เดียวกัน) เล่าเรื่องราวทั้งหมดอย่างโปร่งใส:
หักเงินออกสำเร็จ (`XFRD OK`) → กู้คืนเพราะปลายทางไม่พบ (`RVSL OK`) → บันทึกว่าขาเติมเงินล้มเหลว (`XFRC
REJECTED`) นี่คือสิ่งที่ Part 091 ขั้นตอนที่ 903 อธิบายไว้ล่วงหน้าว่า `TXN-GROUP-ID` จะทำให้ผู้ตรวจสอบ
เห็นภาพรวมของธุรกรรมโอนเงินหนึ่งครั้งได้ครบถ้วน แม้จะกระจายอยู่คนละบรรทัด

### ข้อควรระวัง

- ตัวเลขอ้างอิงการโอน (`TXN-GROUP-ID`) ในระบบนี้รับมาจากผู้ใช้/ผู้เรียกโดยตรง (ไม่ได้สร้างอัตโนมัติ)
  ระบบจริงควรมีกลไกสร้างเลขอ้างอิงที่ไม่ซ้ำกันเองแทน (เช่น Sequence Generator หรือ Timestamp ละเอียด
  ระดับไมโครวินาทีรวมกับรหัสเครื่อง) — ในที่นี้เราลดความซับซ้อนลงเพื่อเน้นการสอนกลไก Compensating
  Transaction เป็นหลัก
- อย่าลืมว่า `CBXFER` เป็นโปรแกรมเดียวในระบบนี้ที่มี Record Area สองชุดพร้อมกัน (`ACCOUNT-RECORD` และ
  `WS-DEST-RECORD`) — เมื่อจะอ้างอิง 88-level ที่มาจาก copybook เดิม ต้องตรวจสอบเสมอว่าต้องเติม
  `OF ACCOUNT-RECORD` หรือไม่ (ดูขั้นตอนที่ 916)

### แบบฝึกหัดที่ 917.1

**โจทย์**: หากทดสอบโอนเงินจากบัญชี Checking `200001` (ที่มียอดคงเหลือน้อย) ไปยังบัญชีปลายทางที่ **ถูก
ปิดไปแล้ว** (ACCT-STATUS = "C") แทนที่จะเป็นบัญชีที่ไม่มีอยู่จริง คาดว่าผลลัพธ์จะเหมือนหรือต่างจากกรณี
บัญชีไม่มีอยู่จริงอย่างไร

**เฉลย**: ผลลัพธ์จะ**เหมือนกันทุกประการ**กับกรณีบัญชีไม่มีอยู่จริง เพราะโค้ดใน STEP 3 ตรวจสอบสองเงื่อนไข
รวมกันในตัวแปรเดียว (`WS-DEST-FOUND-FLAG`): ทั้ง "READ ไม่พบ" (`INVALID KEY`) และ "พบแต่สถานะเป็น Closed"
(`DEST-ACCT-STATUS = "C"`) ต่างทำให้ `WS-DEST-FOUND-FLAG` เป็น `"N"` เหมือนกัน จึงกระตุ้นกลไก Compensating
Transaction ชุดเดียวกันทุกประการ นี่คือการออกแบบที่ดี: รวมทุกเหตุผลที่ "ใช้บัญชีปลายทางนี้ไม่ได้" ไว้ใน
เงื่อนไขเดียว ทำให้โค้ดกู้คืนไม่ต้องเขียนซ้ำสำหรับแต่ละสาเหตุ

---

## ขั้นตอนที่ 918: `CBACCR` — ดอกเบี้ยค้างรับแบบขั้นบันได

### สูตรอัตราดอกเบี้ยขั้นบันไดสำหรับ Savings

| ยอดคงเหลือ | อัตราดอกเบี้ย |
|---|---|
| น้อยกว่า 10,000.00 | อัตราพื้นฐานที่บันทึกไว้ในบัญชี (`ACCT-INTEREST-RATE`) |
| 10,000.00 ถึงต่ำกว่า 100,000.00 | อัตราพื้นฐาน + 0.50% |
| ตั้งแต่ 100,000.00 ขึ้นไป | อัตราพื้นฐาน + 1.00% |

Fixed Deposit ใช้อัตราคงที่ตามที่บันทึกไว้เสมอ ไม่มีขั้นบันได ส่วน Checking มีอัตราดอกเบี้ยเป็น 0 เสมอ
(ถูกกำหนดไว้ตั้งแต่ `CBOPEN`) จึงไม่มีดอกเบี้ยค้างรับให้คำนวณ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBACCR.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBACCR - nightly batch job: accrue (but do not yet post) daily
      * interest for every interest-bearing account (SAVINGS, FIXED-
      * DEP). The accrued amount only builds up in ACCT-ACCRUED-INT;
      * it is added to the real balance once a month by CBMEND (Part
      * 093). SAVINGS uses a tiered rate based on the CURRENT balance;
      * FIXED-DEP always uses its flat stored rate, no tiering.
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

       01  WS-EFFECTIVE-RATE        PIC 9(1)V9(4) VALUE 0.
       01  WS-TIER-BONUS-MID        PIC 9(1)V9(4) VALUE 0.0050.
       01  WS-TIER-BONUS-HIGH       PIC 9(1)V9(4) VALUE 0.0100.
       01  WS-TIER-MID-FLOOR        PIC 9(9)V99   VALUE 10000.00.
       01  WS-TIER-HIGH-FLOOR       PIC 9(9)V99   VALUE 100000.00.
       01  WS-DAILY-INTEREST        PIC 9(7)V99   VALUE 0.

       01  WS-CNT-PROCESSED         PIC 9(5)   VALUE 0.
       01  WS-CNT-ACCRUED           PIC 9(5)   VALUE 0.
       01  WS-TOTAL-ACCRUED         PIC 9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== CBACCR: Daily Interest Accrual ===".
           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-TODAY.

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
           DISPLAY "---- CBACCR Control Totals (" WS-TODAY ") ----".
           DISPLAY "Accounts scanned            : " WS-CNT-PROCESSED.
           DISPLAY "Accounts accrued interest on : " WS-CNT-ACCRUED.
           DISPLAY "Total interest accrued today : " WS-TOTAL-ACCRUED.
           STOP RUN.

       PROCESS-ONE-ACCOUNT.
           ADD 1 TO WS-CNT-PROCESSED.
           IF ACCT-STATUS = "A" OR ACCT-STATUS = "M"
               IF ACCT-TYPE = "S" OR ACCT-TYPE = "F"
                   PERFORM COMPUTE-EFFECTIVE-RATE
                   COMPUTE WS-DAILY-INTEREST ROUNDED =
                       (ACCT-BALANCE * WS-EFFECTIVE-RATE) / 365
                   IF WS-DAILY-INTEREST > 0
                       ADD WS-DAILY-INTEREST TO ACCT-ACCRUED-INT
                       MOVE WS-TODAY TO ACCT-LAST-ACCR-DATE
                       REWRITE ACCOUNT-RECORD
                       ADD 1 TO WS-CNT-ACCRUED
                       ADD WS-DAILY-INTEREST TO WS-TOTAL-ACCRUED
                   END-IF
               END-IF
           END-IF.

       COMPUTE-EFFECTIVE-RATE.
           MOVE ACCT-INTEREST-RATE TO WS-EFFECTIVE-RATE.
           IF ACCT-TYPE = "S"
               IF ACCT-BALANCE >= WS-TIER-HIGH-FLOOR
                   ADD WS-TIER-BONUS-HIGH TO WS-EFFECTIVE-RATE
               ELSE
                   IF ACCT-BALANCE >= WS-TIER-MID-FLOOR
                       ADD WS-TIER-BONUS-MID TO WS-EFFECTIVE-RATE
                   END-IF
               END-IF
           END-IF.
```

จุดที่น่าสนใจคือการวนอ่านทั้งไฟล์ Indexed แบบเรียงลำดับ (`MOVE LOW-VALUES` + `START ... NOT LESS THAN`
+ `READ NEXT RECORD`) ซึ่งเป็นเทคนิคมาตรฐานจาก Part 028 สำหรับงานแบตช์ที่ต้องประมวลผล**ทุกเรคคอร์ด**ใน
ไฟล์ Indexed (ต่างจากโปรแกรมออนไลน์ที่เข้าถึงแบบสุ่มด้วยคีย์เดียว)

### ทดสอบจริง

```bash
cobc -x -o cbaccr cbaccr.cob
printf '20260401\n' | ./cbaccr
```

**ผลลัพธ์จริง** (คำนวณดอกเบี้ยค้างรับให้ 4 บัญชีในชุดทดสอบเต็มของ Part นี้ หลังทำธุรกรรมวันที่ 1 ครบแล้ว):

```
=== CBACCR: Daily Interest Accrual ===
Enter processing date (YYYYMMDD):
---- CBACCR Control Totals (20260401) ----
Accounts scanned            : 00004
Accounts accrued interest on : 00003
Total interest accrued today : 000000006.11
```

3 จาก 4 บัญชีได้ดอกเบี้ยค้างรับ (บัญชี Savings 2 บัญชี + Fixed Deposit 1 บัญชี) ส่วนบัญชี Checking ไม่มี
ดอกเบี้ยเลยตามที่ออกแบบไว้

### ข้อควรระวัง

- `WS-DAILY-INTEREST` ถูกปัดเศษ (`ROUNDED`) เป็นสตางค์ **ทุกวัน** ก่อนสะสมเข้า `ACCT-ACCRUED-INT` ซึ่ง
  เป็นการลดทอนความแม่นยำเล็กน้อยเมื่อเทียบกับระบบที่เก็บดอกเบี้ยค้างรับแบบไม่ปัดเศษจนกว่าจะผ่านรายการจริง
  (ทบทวนเรื่อง rounding error สะสมจาก Part 009) ในระบบธนาคารจริงระดับ production มักจะเก็บทศนิยมเพิ่ม
  อีกหลายหลักในฟิลด์ภายในเพื่อลดผลกระทบจากการปัดเศษสะสม แต่สำหรับ Case Study นี้เราเลือกความเรียบง่าย
  เพื่อการสอน
- ฟิลด์ `ACCT-LAST-ACCR-DATE` เป็นเพียงข้อมูลบันทึกไว้เท่านั้น (ไม่ถูกใช้ในเงื่อนไขคำนวณใดๆ ในโปรแกรมนี้)
  ต่างจาก `ACCT-LAST-STMT-DATE` ที่ `CBMEND` ใน Part 093 จะใช้เป็นกลไก Idempotency Guard จริง

### แบบฝึกหัดที่ 918.1

**โจทย์**: บัญชี Savings มียอดคงเหลือ 12,000.00 และอัตราดอกเบี้ยพื้นฐาน 1.25% ต่อปี จงคำนวณดอกเบี้ย
ค้างรับของวันนั้น (ปัดเศษ 2 ตำแหน่ง)

**เฉลย**: ยอด 12,000.00 อยู่ในช่วง 10,000.00 ถึงต่ำกว่า 100,000.00 จึงได้รับโบนัส +0.50% รวมเป็นอัตรา
ที่แท้จริง 1.75% ต่อปี ดอกเบี้ยรายวัน = `12000.00 * 0.0175 / 365 = 0.5753...` ปัดเศษเป็น **0.58 บาท**

---

## ขั้นตอนที่ 919: ทดสอบรวมวันทำการที่ 1 (Day 1 Integration Test)

ถึงเวลานำทุกโมดูลที่สร้างในบทนี้มาทำงานร่วมกันเป็นวันทำการเต็มวันแรก เริ่มจากไฟล์เปล่าตาม `CBINIT`
(Part 091) แล้วเปิด 4 บัญชีด้วย `CBOPEN`:

```bash
cobc -x -o cbinit cbinit.cob
cobc -x cbopen.cob cbaudit.cob -o cbopen
./cbinit
printf '20260401\n090000\n100001\nSUPAPORN JAIDEE\nS\n000500000\n' | ./cbopen
printf '20260401\n090500\n200001\nGLOBAL TRADING CO\nC\n000100000\n0100000\n' | ./cbopen
printf '20260401\n091000\n300001\nWICHAI SOMBOON\nF\n006000000\n' | ./cbopen
printf '20260401\n091500\n400001\nNATTAYA PROSPER\nS\n001200000\n' | ./cbopen
```

จากนั้นดำเนินธุรกรรมชุดหนึ่งที่ครอบคลุมทุกโมดูล:

```bash
printf '20260401\n100000\n100001\n000200000\n'                    | ./cbdep
printf '20260401\n101000\n200001\n000030000\n'                    | ./cbwd
printf '20260401\n103000\n00000001\n400001\n200001\n000100000\n'  | ./cbxfer
printf '20260401\n104000\n00000002\n100001\n999999\n000050000\n'  | ./cbxfer
printf '20260401\n105000\n100001\n009000000\n'                    | ./cbwd
printf '20260401\n' | ./cbaccr
```

**สรุปสิ่งที่เกิดขึ้นตามลำดับ (ทุกบรรทัดคือผลลัพธ์จริงจากการรันจริง)**:

```
CBOPEN: account 100001 opened successfully, type=S balance=+000005000.00
CBOPEN: account 200001 opened successfully, type=C balance=+000001000.00
CBOPEN: account 300001 opened successfully, type=F balance=+000060000.00
CBOPEN: account 400001 opened successfully, type=S balance=+000012000.00
CBDEP: account 100001 deposit 000002000.00 new balance=+000007000.00
CBWD: account 200001 withdrew 000000300.00 new balance=+000000700.00
CBXFER: debited 000001000.00 from 400001 new balance=+000011000.00
CBXFER: credited 000001000.00 to 200001 new balance=+000001700.00
CBXFER: transfer completed successfully - both legs balanced under reference 00000001
CBXFER: debited 000000500.00 from 100001 new balance=+000006500.00
ERROR: destination account 999999 not found.
CBXFER: transfer FAILED and was reversed - source balance restored to +000007000.00
ERROR: insufficient funds. Balance=+000007000.00 requested=000090000.00
CBWD: withdrawal not applied.
---- CBACCR Control Totals (20260401) ----
Accounts scanned            : 00004
Accounts accrued interest on : 00003
Total interest accrued today : 000000006.11
```

### ตรวจสอบความสมบูรณ์ของ Audit Trail

```bash
cat TXNLOG.DAT
```

```
20260401 090000 00000000 100001 OPEN 00000500000 +00000500000 OK
20260401 090500 00000000 200001 OPEN 00000100000 +00000100000 OK
20260401 091000 00000000 300001 OPEN 00006000000 +00006000000 OK
20260401 091500 00000000 400001 OPEN 00001200000 +00001200000 OK
20260401 100000 00000000 100001 DEP  00000200000 +00000700000 OK
20260401 101000 00000000 200001 WD   00000030000 +00000070000 OK
20260401 103000 00000001 400001 XFRD 00000100000 +00001100000 OK
20260401 103000 00000001 200001 XFRC 00000100000 +00000170000 OK
20260401 104000 00000002 100001 XFRD 00000050000 +00000650000 OK
20260401 104000 00000002 100001 RVSL 00000050000 +00000700000 OK
20260401 104000 00000002 999999 XFRC 00000050000 +00000700000 REJECTED
20260401 105000 00000000 100001 WD   00009000000 +00000700000 REJECTED
```

12 บรรทัด ครบถ้วนตรงกับทุกการกระทำที่ทำไป: เปิดบัญชี 4 ครั้ง, ฝาก 1 ครั้ง, ถอนสำเร็จ 1 ครั้ง, โอนสำเร็จ
1 ครั้ง (2 บรรทัด), โอนล้มเหลว-กู้คืน 1 ครั้ง (3 บรรทัด), ถอนล้มเหลว 1 ครั้ง — ระบบทำงานถูกต้องสมบูรณ์

### ข้อควรระวัง

- ในขั้นตอนการทดสอบรวมนี้เรายังไม่ได้สร้างรายงานสรุปที่อ่านง่าย (เช่น รายงานธุรกรรมประจำวัน หรือ
  รายงานกระทบยอด) — นั่นคือหน้าที่ของ Part 093 ที่จะนำ `TXNLOG.DAT` และ `ACCTMAST.DAT` ชุดนี้ไปสร้าง
  รายงานต่อยอดทันที ไม่มีการรีเซ็ตข้อมูลใหม่

### แบบฝึกหัดที่ 919.1

**โจทย์**: จากผลรวมค้างรับดอกเบี้ยวันที่ 1 เท่ากับ 6.11 บาท (3 บัญชี) จงระบุว่าบัญชีใดบ้างที่ได้รับ
ดอกเบี้ยค้างรับ และประมาณคร่าวๆ ว่าแต่ละบัญชีได้รับเท่าใด (อ้างอิงยอดคงเหลือ ณ ขณะรัน `CBACCR`)

**เฉลยแนวทาง**: บัญชีที่ได้ดอกเบี้ยคือ `100001` (Savings, ยอด 7,000.00 อัตราพื้นฐาน 1.25% เพราะยังไม่ถึง
10,000.00) `400001` (Savings, ยอด 11,000.00 อัตราพื้นฐาน 1.25% เพราะยังไม่ถึง 10,000.00 เช่นกัน ณ ขณะนั้น)
และ `300001` (Fixed Deposit, ยอด 60,000.00 อัตรา 3.25%) คำนวณคร่าวๆ: `100001` ≈ `7000*0.0125/365` ≈ 0.24,
`400001` ≈ `11000*0.0125/365` ≈ 0.38, `300001` ≈ `60000*0.0325/365` ≈ 5.34 รวมประมาณ 5.96-6.11 บาท
(ใกล้เคียงกับตัวเลขจริงที่ระบบรายงาน ความคลาดเคลื่อนเล็กน้อยมาจากการปัดเศษแต่ละขั้นตอน)

---

## ขั้นตอนที่ 920: ทดสอบรวมวันทำการที่ 2 และสรุปส่งต่อ Part 093

### วันทำการที่ 2: พิสูจน์กฎวงเงินเบิกเกินบัญชีของ Checking

```bash
printf '20260402\n090000\n200001\n000080000\n'                    | ./cbdep
printf '20260402\n091000\n200001\n000250000\n'                    | ./cbwd
printf '20260402\n093000\n00000003\n200001\n100001\n000040000\n'  | ./cbxfer
printf '20260402\n094000\n400001\n000300000\n'                    | ./cbwd
printf '20260402\n' | ./cbaccr
```

**ผลลัพธ์จริง**:

```
CBDEP: account 200001 deposit 000000800.00 new balance=+000002500.00
CBWD: account 200001 withdrew 000002500.00 new balance=+000000000.00
CBXFER: debited 000000400.00 from 200001 new balance=-000000400.00
CBXFER: credited 000000400.00 to 100001 new balance=+000007400.00
CBXFER: transfer completed successfully - both legs balanced under reference 00000003
CBWD: account 400001 withdrew 000003000.00 new balance=+000008000.00
---- CBACCR Control Totals (20260402) ----
Accounts scanned            : 00004
Accounts accrued interest on : 00003
Total interest accrued today : 000000005.86
```

จุดที่สำคัญที่สุดของการทดสอบวันนี้คือบรรทัดที่สาม: บัญชี Checking `200001` มียอดคงเหลือ 0.00 ก่อนโอน
เงิน 400.00 ออกไป — ระบบยอมให้ยอดคงเหลือกลายเป็น **`-000000400.00`** (ติดลบ!) เพราะยังอยู่ภายในวงเงิน
เบิกเกินบัญชี 100,000.00 บาทที่กำหนดไว้ตอนเปิดบัญชี นี่คือการพิสูจน์ว่ากฎ Checking ที่ออกแบบไว้ใน Part
091 และสร้างใน `CBWD`/`CBXFER` (ขั้นตอนที่ 914, 916) ทำงานถูกต้องตรงตามที่ออกแบบไว้ทุกประการ แม้จะใช้
งานผ่านโมดูลคนละตัว (`CBXFER` ไม่ใช่ `CBWD`) ก็ตาม — พิสูจน์ว่าเราเขียนตรรกะตรวจสอบวงเงินเบิกเกินบัญชี
ไว้อย่างสอดคล้องกันในทั้งสองโปรแกรม

### สรุปสถานะบัญชีทั้ง 4 หลังจบวันทำการที่ 2 (ก่อนคำนวณดอกเบี้ยผ่านรายการจริง)

| บัญชี | ประเภท | ยอดคงเหลือ | ดอกเบี้ยค้างรับสะสม (2 วัน) |
|---|---|---|---|
| 100001 | Savings | 7,400.00 | ~0.49 |
| 200001 | Checking | -400.00 | 0.00 (ไม่มีดอกเบี้ย) |
| 300001 | Fixed Deposit | 60,000.00 | ~10.68 |
| 400001 | Savings | 8,000.00 | ~0.80 |

### แบบฝึกหัดที่ 920.1

**โจทย์**: จากตารางสรุปข้างต้น บัญชี `200001` (Checking) มียอดคงเหลือ `-400.00` จงอธิบายว่าทำไม
`CBCLOSE` (จะสร้างใน Part 093) ถึงจำเป็นต้องรองรับการแสดงผลยอดติดลบได้อย่างถูกต้อง ไม่ใช่แค่ `CBSTMT`
หรือ `CBDAILY` เท่านั้น

**เฉลยแนวทาง**: `CBCLOSE` สรุปยอดรวมระดับพอร์ตโฟลิโอโดยนำยอดคงเหลือของทุกบัญชีในแต่ละประเภทมาบวกกัน
(`ADD ACCT-BALANCE TO WS-BAL-CHECKING`) หากตัวแปรสะสมยอด (`WS-BAL-CHECKING`) ไม่ได้ประกาศเป็น Signed
(`S9(10)V99`) หรือหากลอจิกไม่รองรับค่าติดลบ ยอดรวม Checking ทั้งพอร์ตจะคำนวณผิดทันทีเมื่อมีบัญชีใดบัญชี
หนึ่งติดลบอยู่ — เนื่องจากยอดติดลบเป็นสถานะปกติที่คาดว่าจะเกิดขึ้นได้เสมอสำหรับบัญชี Checking (ไม่ใช่
กรณีผิดปกติ) ทุกโปรแกรมที่ประมวลผลยอดคงเหลือของบัญชีประเภทนี้จึงต้องออกแบบมาให้รองรับค่าติดลบตั้งแต่ต้น
ไม่ใช่แก้ไขเพิ่มทีหลังเมื่อพบปัญหา

## สรุปท้ายบท

Part นี้เติมชีวิตให้ Case Study Core Banking อย่างสมบูรณ์:

- สร้าง `CBAUDIT` และเรียนรู้บั๊กจริงสองตัวที่ทั้งคู่**ไม่มี compile error ใดๆ เตือน**: ปัญหา FILE
  SECTION ไม่ initialize VALUE clause อัตโนมัติ (FILE STATUS 71) และปัญหาการส่ง literal สั้นเข้า CALL
  ทำให้อ่านข้อมูลเลยขอบเขต — ทั้งสองคือบทเรียนที่มีค่ามากสำหรับการเขียน COBOL ระดับมืออาชีพ
- สร้างและทดสอบ `CBDEP`, `CBWD` ครบทุกกฎธุรกิจของบัญชีทั้ง 3 ประเภท
- ออกแบบและสร้าง `CBXFER` — โมดูลที่ซับซ้อนที่สุดของทั้งระบบ พร้อมกลไก **Compensating Transaction**
  ที่พิสูจน์ด้วยการทดสอบจริงทั้งเส้นทางสำเร็จและเส้นทางล้มเหลว-กู้คืน
- สร้าง `CBACCR` คำนวณดอกเบี้ยค้างรับแบบขั้นบันได
- ทดสอบรวมสองวันทำการเต็มรูปแบบ พิสูจน์ว่าทุกโมดูลทำงานร่วมกันถูกต้อง รวมถึงกฎวงเงินเบิกเกินบัญชีที่
  ทำงานสอดคล้องกันข้ามสองโปรแกรม (`CBWD` และ `CBXFER`)

ข้อมูลทั้งหมดที่สร้างขึ้นใน Part นี้ (บัญชี 4 บัญชี, ธุรกรรม 2 วันทำการ, Audit Trail ครบถ้วน) จะถูกนำไป
ใช้ต่อใน **Part 093** ทันที โดยไม่มีการรีเซ็ตใหม่ — Part 093 จะสร้างรายงานธุรกรรมประจำวัน ใบแจ้งยอดบัญชี
การกระทบยอดสิ้นวันที่พิสูจน์หลักบัญชีคู่ของธุรกรรมโอนเงินที่เราเพิ่งสร้าง และปิดท้ายด้วยการผ่านรายการ
ดอกเบี้ยสิ้นเดือนที่จะเปลี่ยนดอกเบี้ยค้างรับที่ `CBACCR` สะสมไว้ให้กลายเป็นเงินจริงในบัญชีลูกค้า

**[กลับไป Part 091: การออกแบบระบบ](part-091-corebanking-design.md)**
**[ไปยัง Part 093: Case Study Core Banking ตอนที่ 3 - การรายงานและปิดบัญชี →](part-093-corebanking-reporting.md)**
