# Part 066: Batch Processing Patterns และ Job Scheduling (ขั้นตอนที่ 651–660)

## คำนำของ Part นี้

Part 061-065 พาเราเจาะลึกโลกของระบบ **ออนไลน์** (CICS, IMS) ที่ตอบสนองผู้ใช้แบบทันทีทันใด แต่ในความ
เป็นจริง ระบบ Mainframe ขององค์กรขนาดใหญ่ใช้เวลาส่วนใหญ่ในแต่ละคืนไปกับงานอีกประเภทหนึ่งที่สำคัญไม่
แพ้กัน นั่นคือ **Batch Processing** — การประมวลผลข้อมูลจำนวนมหาศาลเป็นชุด (เช่น คำนวณดอกเบี้ยของ
บัญชีลูกค้าทุกบัญชีทุกคืน, สร้างรายงานยอดขายประจำวันของทุกสาขา, ปิดยอดบัญชีสิ้นเดือน)

Part นี้แตกต่างจาก Part 063-065 อย่างชัดเจนในแง่ที่**สามารถทดสอบได้จริง**เกือบทั้งหมด เพราะ Batch
Processing Pattern ที่จะเรียนล้วนเป็น**ตรรกะ COBOL บริสุทธิ์** ที่ต่อยอดจากไฟล์ Sequential/Indexed
(Part 023-029), Master-Detail Matching (Part 026), และ SORT (Part 027) ที่เรียนไปแล้วทั้งสิ้น —
ไม่ต้องพึ่งพา CICS, DB2, หรือ IMS แต่อย่างใด ส่วนที่เป็นแนวคิดเชิงทฤษฎีล้วน (Job Scheduling ด้วย
เครื่องมืออย่าง CA-7/Control-M) จะมีป้ายกำกับชัดเจนแยกจากส่วนที่ทดสอบได้จริง

> **หลักการของ Part นี้**: ทุกโปรแกรม COBOL ในเอกสารนี้ **คอมไพล์และรันจริงแล้วด้วย GnuCOBOL
> (`cobc (GnuCOBOL) 4.0-early-dev.0`)** และผลลัพธ์ที่แสดงไว้คือผลลัพธ์จริงจากการรัน ยกเว้นส่วนที่ระบุ
> ชัดเจนว่าเป็น **⚠️ REFERENCE (แนวคิด/เครื่องมือเชิงทฤษฎี)** ซึ่งเกี่ยวกับซอฟต์แวร์จัดตารางงาน
> (Job Scheduler) เชิงพาณิชย์ที่ไม่มีในสภาพแวดล้อมนี้

---

## ขั้นตอนที่ 651: ภาพรวม Batch Design Patterns คลาสสิกที่ใช้กันทั่วโลก

### ทำไม Batch Processing ยังสำคัญในปี 2026

แม้โลกจะหมุนไปสู่ระบบ Real-time มากขึ้นเรื่อย ๆ แต่งานหลายประเภทยัง**เหมาะกับการประมวลผลเป็นชุด
มากกว่า**เสมอ: การคำนวณที่ต้องพิจารณาข้อมูลทั้งหมดพร้อมกัน (เช่น จัดอันดับลูกค้า VIP จากยอดใช้จ่าย
สะสมทั้งเดือน), งานที่ควรทำตอนระบบมีภาระงานน้อย (กลางดึก), หรืองานที่ต้องประมวลผลปริมาณมหาศาลด้วย
ต้นทุนต่ำที่สุด (การรัน batch ครั้งเดียวประมวลผลล้านรายการมักถูกกว่าการยิง transaction ออนไลน์ล้าน
ครั้ง)

### แพทเทิร์นคลาสสิก 4 แบบที่ Part นี้จะครอบคลุม

| แพทเทิร์น | ปัญหาที่แก้ | ขั้นตอนที่สอน |
|---|---|---|
| **Sequential Update** | ปรับปรุงไฟล์ Master ด้วยไฟล์ Transaction (เพิ่ม/แก้ไข/ลบ) | 652 |
| **Control-Break Reporting** | สรุปยอดเป็นกลุ่มตามลำดับชั้นของข้อมูล | 653-654 |
| **Restart/Checkpoint** | กู้คืนงานที่ล้มเหลวกลางทางโดยไม่ต้องเริ่มใหม่ทั้งหมด | 655 |
| **Idempotent Batch Design** | ออกแบบให้รันซ้ำได้อย่างปลอดภัยโดยไม่เกิดผลข้างเคียงซ้ำซ้อน | 658 |

แพทเทิร์นทั้งหมดนี้ต่อยอดจากพื้นฐานที่เรียนมาแล้ว: **Sequential Update** คือการนำ Master-Detail
Matching (Part 026) มาประยุกต์ให้เขียนไฟล์ Master ใหม่ด้วย ไม่ใช่แค่อ่านอย่างเดียว **Control-Break**
คือการนำแนวคิดจาก Report Writer (Part 039) มาเขียนด้วยตรรกะ `PROCEDURE DIVISION` ล้วน ๆ ซึ่งเป็น
วิธีที่โปรแกรม Batch จริงในองค์กรส่วนใหญ่นิยมใช้มากกว่า Report Writer เสียด้วยซ้ำ (เพราะยืดหยุ่นกว่า
และพกพาข้ามคอมไพเลอร์ได้ง่ายกว่า)

### ข้อควรระวัง

- แพทเทิร์นเหล่านี้ **ไม่ใช่เทคนิคเฉพาะของ Mainframe** — เป็นหลักการออกแบบซอฟต์แวร์ที่ใช้ได้กับงาน
  ประมวลผลข้อมูลจำนวนมากในทุกแพลตฟอร์ม (แม้แต่ Python script ที่รันบน cron job ของ Linux servers
  ในบริษัทเทคโนโลยีสมัยใหม่ก็ยังใช้หลักการเดียวกันนี้) เพียงแต่ COBOL/Mainframe เป็นสภาพแวดล้อมที่
  แพทเทิร์นเหล่านี้ถูกพิสูจน์และขัดเกลามานานที่สุด

### แบบฝึกหัดที่ 651.1

**โจทย์**: จงยกตัวอย่างงานประมวลผลข้อมูล 2 ประเภทที่เหมาะกับ Batch Processing มากกว่า Real-time
Processing พร้อมเหตุผล

**เฉลยแนวทาง**: (1) การคำนวณดอกเบี้ยเงินฝากรายเดือนของธนาคาร — เหมาะกับ batch เพราะต้องคำนวณจาก
ยอดคงเหลือ ณ สิ้นวันที่แน่นอน ไม่จำเป็นต้องคำนวณทันทีที่มีการเปลี่ยนแปลง และควรทำตอนที่ระบบมีภาระงาน
น้อย (2) การสร้างรายงานสรุปยอดขายรายวันของทุกสาขาทั่วประเทศ — เหมาะกับ batch เพราะต้องรวบรวมข้อมูล
จากทุกสาขาให้ครบก่อนจึงจะสรุปยอดได้อย่างถูกต้อง การพยายามอัปเดตรายงานแบบ real-time ทุกครั้งที่มี
การขายจะสิ้นเปลืองทรัพยากรโดยไม่จำเป็น เพราะผู้บริหารต้องการดูรายงานแค่วันละครั้งเท่านั้น

---

## ขั้นตอนที่ 652: Sequential Update Pattern — ปรับปรุงไฟล์ Master ด้วยไฟล์ Transaction

### ต่อยอดจาก Master-Detail Matching (Part 026)

Part 026 สอนให้เรา**อ่าน**สองไฟล์ที่เรียงลำดับ Key เดียวกันไปพร้อมกันเพื่อจับคู่ข้อมูล แต่ Sequential
Update Pattern ไปไกลกว่านั้น: แทนที่จะแค่อ่านและแสดงผล เราจะ**เขียนไฟล์ Master ฉบับใหม่**ที่ผ่านการ
ปรับปรุงตามรายการ Transaction แล้ว โดย Transaction แต่ละรายการมี **รหัสประเภท (Transaction Code)**
บอกว่าต้องการทำอะไรกับ Master: `A` = เพิ่มลูกค้าใหม่ (Add), `C` = แก้ไขยอดเงิน (Change), `D` = ลบ
ลูกค้า (Delete)

### โค้ดตัวอย่างสมบูรณ์ (ทดสอบจริงแล้ว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP652-SEQUENTIAL-UPDATE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OLD-MASTER ASSIGN TO "OLDMST652.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT TRANS-FILE ASSIGN TO "TRANS652.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT NEW-MASTER ASSIGN TO "NEWMST652.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT EXCEPTION-FILE ASSIGN TO "EXCEPT652.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-EXC-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  OLD-MASTER.
       01  OLD-RECORD.
           05  OLD-ID              PIC X(4).
           05  OLD-NAME            PIC X(15).
           05  OLD-BALANCE         PIC 9(6)V99.

       FD  TRANS-FILE.
       01  TRANS-RECORD.
           05  TRANS-CODE          PIC X(1).
           05  TRANS-ID            PIC X(4).
           05  TRANS-NAME          PIC X(15).
           05  TRANS-AMOUNT        PIC 9(6)V99.

       FD  NEW-MASTER.
       01  NEW-RECORD.
           05  NEW-ID              PIC X(4).
           05  NEW-NAME            PIC X(15).
           05  NEW-BALANCE         PIC 9(6)V99.

       FD  EXCEPTION-FILE.
       01  EXCEPTION-RECORD        PIC X(60).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                PIC X VALUE "N".
           88  MASTER-EOF          VALUE "Y".
       01  WS-T-EOF                PIC X VALUE "N".
           88  TRANS-EOF           VALUE "Y".
       01  WS-ADD-COUNT            PIC 9(3) VALUE 0.
       01  WS-CHANGE-COUNT         PIC 9(3) VALUE 0.
       01  WS-DELETE-COUNT         PIC 9(3) VALUE 0.
       01  WS-REJECT-COUNT         PIC 9(3) VALUE 0.
       01  WS-EXC-STATUS           PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.

           OPEN INPUT OLD-MASTER.
           OPEN INPUT TRANS-FILE.
           OPEN OUTPUT NEW-MASTER.
           OPEN OUTPUT EXCEPTION-FILE.

           PERFORM READ-MASTER.
           PERFORM READ-TRANS.

      *> Classic sequential-update pattern: OLD-MASTER and
      *> TRANS-FILE are BOTH sorted by key. We walk them together
      *> exactly like the matching logic from Part 026, but now we
      *> also WRITE a NEW-MASTER as we go, instead of only reading.
           PERFORM UNTIL MASTER-EOF AND TRANS-EOF
               EVALUATE TRUE
                   WHEN MASTER-EOF
                       PERFORM APPLY-TRANS-NO-MASTER
                       PERFORM READ-TRANS
                   WHEN TRANS-EOF
                       PERFORM COPY-MASTER-UNCHANGED
                       PERFORM READ-MASTER
                   WHEN OLD-ID < TRANS-ID
                       PERFORM COPY-MASTER-UNCHANGED
                       PERFORM READ-MASTER
                   WHEN OLD-ID > TRANS-ID
                       PERFORM APPLY-TRANS-NO-MASTER
                       PERFORM READ-TRANS
                   WHEN OTHER
                       PERFORM APPLY-TRANS-TO-MASTER
               END-EVALUATE
           END-PERFORM.

           CLOSE OLD-MASTER.
           CLOSE TRANS-FILE.
           CLOSE NEW-MASTER.
           CLOSE EXCEPTION-FILE.

           DISPLAY "Added   : " WS-ADD-COUNT.
           DISPLAY "Changed : " WS-CHANGE-COUNT.
           DISPLAY "Deleted : " WS-DELETE-COUNT.
           DISPLAY "Rejected: " WS-REJECT-COUNT.

           PERFORM SHOW-NEW-MASTER.
           PERFORM SHOW-EXCEPTIONS.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT OLD-MASTER.
           MOVE "C001" TO OLD-ID.
           MOVE "SOMCHAI JAIDEE " TO OLD-NAME.
           MOVE 500.00 TO OLD-BALANCE.
           WRITE OLD-RECORD.
           MOVE "C002" TO OLD-ID.
           MOVE "SUDA MEECHAI   " TO OLD-NAME.
           MOVE 1200.00 TO OLD-BALANCE.
           WRITE OLD-RECORD.
           MOVE "C004" TO OLD-ID.
           MOVE "ANAN SUKJAI    " TO OLD-NAME.
           MOVE 300.00 TO OLD-BALANCE.
           WRITE OLD-RECORD.
           CLOSE OLD-MASTER.

      *> Transaction codes: A = Add new customer, C = Change
      *> balance (adds amount), D = Delete customer. This feed
      *> must already be sorted by TRANS-ID for this pattern.
           OPEN OUTPUT TRANS-FILE.
           MOVE "C" TO TRANS-CODE.
           MOVE "C001" TO TRANS-ID.
           MOVE SPACES TO TRANS-NAME.
           MOVE 150.00 TO TRANS-AMOUNT.
           WRITE TRANS-RECORD.

           MOVE "A" TO TRANS-CODE.
           MOVE "C003" TO TRANS-ID.
           MOVE "PRASERT KAEWTA " TO TRANS-NAME.
           MOVE 800.00 TO TRANS-AMOUNT.
           WRITE TRANS-RECORD.

           MOVE "D" TO TRANS-CODE.
           MOVE "C004" TO TRANS-ID.
           MOVE SPACES TO TRANS-NAME.
           MOVE 0 TO TRANS-AMOUNT.
           WRITE TRANS-RECORD.

           MOVE "C" TO TRANS-CODE.
           MOVE "C009" TO TRANS-ID.
           MOVE SPACES TO TRANS-NAME.
           MOVE 50.00 TO TRANS-AMOUNT.
           WRITE TRANS-RECORD.
           CLOSE TRANS-FILE.

       READ-MASTER.
           READ OLD-MASTER
               AT END
                   SET MASTER-EOF TO TRUE
           END-READ.

       READ-TRANS.
           READ TRANS-FILE
               AT END
                   SET TRANS-EOF TO TRUE
           END-READ.

       COPY-MASTER-UNCHANGED.
           MOVE OLD-ID TO NEW-ID.
           MOVE OLD-NAME TO NEW-NAME.
           MOVE OLD-BALANCE TO NEW-BALANCE.
           WRITE NEW-RECORD.

       APPLY-TRANS-TO-MASTER.
           EVALUATE TRANS-CODE
               WHEN "C"
                   MOVE OLD-ID TO NEW-ID
                   MOVE OLD-NAME TO NEW-NAME
                   COMPUTE NEW-BALANCE = OLD-BALANCE + TRANS-AMOUNT
                   WRITE NEW-RECORD
                   ADD 1 TO WS-CHANGE-COUNT
                   PERFORM READ-MASTER
               WHEN "D"
                   ADD 1 TO WS-DELETE-COUNT
                   PERFORM READ-MASTER
               WHEN "A"
                   MOVE SPACES TO EXCEPTION-RECORD
                   STRING "REJECTED: ADD for existing ID " TRANS-ID
                       DELIMITED BY SIZE INTO EXCEPTION-RECORD
                   WRITE EXCEPTION-RECORD
                   ADD 1 TO WS-REJECT-COUNT
                   PERFORM READ-MASTER
               WHEN OTHER
                   MOVE SPACES TO EXCEPTION-RECORD
                   STRING "REJECTED: unknown code for " TRANS-ID
                       DELIMITED BY SIZE INTO EXCEPTION-RECORD
                   WRITE EXCEPTION-RECORD
                   ADD 1 TO WS-REJECT-COUNT
                   PERFORM READ-MASTER
           END-EVALUATE.
           PERFORM READ-TRANS.

       APPLY-TRANS-NO-MASTER.
           EVALUATE TRANS-CODE
               WHEN "A"
                   MOVE TRANS-ID TO NEW-ID
                   MOVE TRANS-NAME TO NEW-NAME
                   MOVE TRANS-AMOUNT TO NEW-BALANCE
                   WRITE NEW-RECORD
                   ADD 1 TO WS-ADD-COUNT
               WHEN OTHER
                   MOVE SPACES TO EXCEPTION-RECORD
                   STRING "REJECTED: " TRANS-CODE
                       " for non-existent ID " TRANS-ID
                       DELIMITED BY SIZE INTO EXCEPTION-RECORD
                   WRITE EXCEPTION-RECORD
                   ADD 1 TO WS-REJECT-COUNT
           END-EVALUATE.

       SHOW-NEW-MASTER.
           MOVE "N" TO WS-M-EOF.
           OPEN INPUT NEW-MASTER.
           DISPLAY "---- NEW-MASTER content ----".
           PERFORM UNTIL MASTER-EOF
               READ NEW-MASTER
                   AT END
                       SET MASTER-EOF TO TRUE
                   NOT AT END
                       DISPLAY "  " NEW-ID " " NEW-NAME " " NEW-BALANCE
               END-READ
           END-PERFORM.
           CLOSE NEW-MASTER.

       SHOW-EXCEPTIONS.
           MOVE "N" TO WS-T-EOF.
           OPEN INPUT EXCEPTION-FILE.
           DISPLAY "---- EXCEPTION-FILE content ----".
           PERFORM UNTIL TRANS-EOF
               READ EXCEPTION-FILE
                   AT END
                       SET TRANS-EOF TO TRUE
                   NOT AT END
                       DISPLAY "  " EXCEPTION-RECORD
               END-READ
           END-PERFORM.
           CLOSE EXCEPTION-FILE.
```

คอมไพล์และรัน:

```bash
cobc -x -o step652 step652.cob
./step652
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
Added   : 001
Changed : 001
Deleted : 001
Rejected: 001
---- NEW-MASTER content ----
  C001 SOMCHAI JAIDEE  000650.00
  C002 SUDA MEECHAI    001200.00
  C003 PRASERT KAEWTA  000800.00
---- EXCEPTION-FILE content ----
  REJECTED: C for non-existent ID C009
```

### อธิบายจุดสำคัญ

- ลูกค้า `C001` ถูก `C` (เปลี่ยนยอด +150) จาก 500.00 กลายเป็น 650.00 ✓
- ลูกค้า `C003` ถูก `A` (เพิ่มใหม่) เข้ามาในตำแหน่งที่ถูกต้องตามลำดับ Key ✓
- ลูกค้า `C004` ถูก `D` (ลบ) จึงหายไปจาก `NEW-MASTER` โดยสิ้นเชิง ✓
- ลูกค้า `C002` ไม่มี Transaction ใดเกี่ยวข้อง จึงถูกคัดลอกไปยัง `NEW-MASTER` โดยไม่เปลี่ยนแปลง ✓
- Transaction `C009` (โค้ด `C` แต่ไม่มี Master ID นี้อยู่จริง) ถูกปฏิเสธและบันทึกลง
  `EXCEPTION-FILE` แทนที่จะทำให้โปรแกรม crash หรือสร้างข้อมูลผิดพลาดเงียบ ๆ

### ข้อควรระวัง (พบจริงระหว่างพัฒนาตัวอย่างนี้)

**ระหว่างพัฒนาโปรแกรมนี้ เราพบบั๊กจริงที่ควรบันทึกไว้เป็นบทเรียน**: การเขียน
`STRING ... DELIMITED BY SIZE INTO EXCEPTION-RECORD` แล้ว `WRITE EXCEPTION-RECORD` ทันทีโดย**ไม่ได้
`MOVE SPACES TO EXCEPTION-RECORD` ก่อน** ทำให้เกิด runtime error ทันที:

```
libcob: error: unknown file error (status = 71) for file EXCEPTION-FILE ('EXCEPT652.DAT')
```

**สาเหตุที่แท้จริง**: status `71` ใน GnuCOBOL คือ `COB_STATUS_71_BAD_CHAR` ซึ่งเกิดขึ้นเมื่อไฟล์
`LINE SEQUENTIAL` พยายามเขียนไบต์ที่ไม่ใช่ตัวอักษรที่แสดงผลได้ (non-printable character) ลงไฟล์
พื้นที่ของ record ใน `FD` ที่ยังไม่เคยถูกเขียนค่าเลย (เพราะเพิ่ง `OPEN OUTPUT` ใหม่) มีค่าเริ่มต้นเป็น
ไบต์ต่ำ (low-values) ไม่ใช่ช่องว่าง เมื่อ `STRING` เติมข้อมูลแค่บางส่วนของ `EXCEPTION-RECORD PIC
X(60)` (เช่น ยาวแค่ 36 ตัวอักษรจาก 60) **ไบต์ที่เหลือ 24 ไบต์ท้าย record ยังคงเป็นไบต์ต่ำที่ยังไม่ถูก
เขียนทับ** และ GnuCOBOL ปฏิเสธที่จะเขียนไบต์เหล่านี้ลงไฟล์ `LINE SEQUENTIAL` (ซึ่งเป็นไฟล์ข้อความ)
วิธีแก้คือ **`MOVE SPACES TO EXCEPTION-RECORD` ก่อน `STRING` เสมอ** (ตามที่โค้ดด้านบนทำไว้แล้ว)
เพื่อให้ทุกไบต์ของ record เป็นช่องว่าง (spaces) ก่อนที่ `STRING` จะเติมข้อความทับบางส่วน — หลักการ
เดียวกับที่ Part 027 ขั้นตอนที่ 264 เตือนไว้แล้วเรื่อง `MOVE SPACES TO RPT-RECORD` ก่อน `STRING`
แต่ครั้งนี้พิสูจน์ให้เห็นว่าการละเลยกฎนี้ไม่ได้แค่ทำให้ข้อมูลไม่สวยงาม แต่**ทำให้โปรแกรม crash ทันที**

### แบบฝึกหัดที่ 652.1

**โจทย์**: จงอธิบายว่าทำไม Transaction ที่มีโค้ด `A` (Add) แต่ ID ซ้ำกับที่มีอยู่ใน Master แล้ว
(เช่น พยายามเพิ่ม `C001` ทั้งที่มีอยู่แล้ว) จึงควรถูกปฏิเสธแทนที่จะเขียนทับข้อมูลเดิมไปเลย

**เฉลย**: เพราะ Transaction Code `A` มีความหมายเฉพาะเจาะจงว่า "นี่คือลูกค้าใหม่ที่ไม่เคยมีมาก่อน"
หากระบบยอมให้ Transaction `A` เขียนทับ Master ที่มี ID ซ้ำกันโดยไม่แจ้งเตือน อาจเป็นสัญญาณของ
ข้อผิดพลาดร้ายแรงในต้นทางของข้อมูล (เช่น ระบบต้นทางสร้าง ID ซ้ำโดยไม่ตั้งใจ หรือไฟล์ transaction
ถูกประมวลผลซ้ำสองรอบ) การปฏิเสธและบันทึกเป็น exception ทำให้ผู้ดูแลระบบตรวจสอบสาเหตุที่แท้จริงได้
แทนที่จะปล่อยให้ข้อมูลผิดพลาดเงียบ ๆ ซึ่งอาจนำไปสู่ปัญหาที่ตรวจจับยากกว่ามากในภายหลัง (หลักการเดียว
กับที่ Part 030 สอนเรื่องการตรวจสอบ File Status Code อย่างเข้มงวด)

---

## ขั้นตอนที่ 653: Control-Break Reporting แบบ Single-Level ด้วยตรรกะ PROCEDURE DIVISION ล้วน ๆ

### ทำไมต้องเขียนเองแทนที่จะใช้ Report Writer

Part 039 สอน Control Break ผ่าน Report Writer Feature (`CONTROLS ARE`, `TYPE CONTROL HEADING`) ซึ่ง
สะดวกมากแต่**ไม่ใช่ COBOL Compiler ทุกตัวจะรองรับ Report Writer อย่างสมบูรณ์** (Part 038-039 เองก็
เคยเจอพฤติกรรมไม่แน่นอนบางจุดใน GnuCOBOL build ที่ทดสอบ) โปรแกรม Batch จำนวนมากในองค์กรจริงจึงเขียน
ตรรกะ Control Break ด้วยมือผ่าน `PROCEDURE DIVISION` ธรรมดา — วิธีนี้ทำงานเหมือนกันทุกประการในทุก
COBOL Compiler ทุกแพลตฟอร์ม เพราะไม่พึ่งพา extension พิเศษใด ๆ เลย

### โค้ดตัวอย่างสมบูรณ์ (ทดสอบจริงแล้ว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP653-CONTROL-BREAK.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SALES-FILE ASSIGN TO "SALES653.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  SALES-FILE.
       01  SALES-RECORD.
           05  SR-REGION            PIC X(5).
           05  SR-AMOUNT            PIC 9(6)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-PREV-REGION           PIC X(5) VALUE SPACES.
       01  WS-FIRST-RECORD          PIC X VALUE "Y".
           88  IS-FIRST-RECORD      VALUE "Y".
       01  WS-REGION-TOTAL          PIC 9(8)V99 VALUE 0.
       01  WS-GRAND-TOTAL           PIC 9(9)V99 VALUE 0.
       01  WS-DISPLAY-AMT           PIC Z,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.
           OPEN INPUT SALES-FILE.
           PERFORM READ-SALES.

      *> Single-level control break, written by hand with plain
      *> PROCEDURE DIVISION logic (no Report Writer). This is the
      *> style most real mainframe batch programs actually use,
      *> because it is a subroutine, not a separate feature to
      *> learn, and works the same in every COBOL dialect.
           PERFORM UNTIL END-OF-FILE
               IF NOT IS-FIRST-RECORD
                   IF SR-REGION NOT = WS-PREV-REGION
                       PERFORM REGION-BREAK
                   END-IF
               END-IF
               ADD SR-AMOUNT TO WS-REGION-TOTAL
               MOVE SR-REGION TO WS-PREV-REGION
               MOVE "N" TO WS-FIRST-RECORD
               PERFORM READ-SALES
           END-PERFORM.

           PERFORM REGION-BREAK.
           MOVE WS-GRAND-TOTAL TO WS-DISPLAY-AMT.
           DISPLAY "GRAND TOTAL: " WS-DISPLAY-AMT.

           CLOSE SALES-FILE.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT SALES-FILE.
           MOVE "NORTH" TO SR-REGION. MOVE 1000.00 TO SR-AMOUNT.
           WRITE SALES-RECORD.
           MOVE "NORTH" TO SR-REGION. MOVE 500.00  TO SR-AMOUNT.
           WRITE SALES-RECORD.
           MOVE "SOUTH" TO SR-REGION. MOVE 750.00  TO SR-AMOUNT.
           WRITE SALES-RECORD.
           MOVE "WEST " TO SR-REGION. MOVE 2200.00 TO SR-AMOUNT.
           WRITE SALES-RECORD.
           CLOSE SALES-FILE.

       READ-SALES.
           READ SALES-FILE
               AT END
                   SET END-OF-FILE TO TRUE
           END-READ.

       REGION-BREAK.
           MOVE WS-REGION-TOTAL TO WS-DISPLAY-AMT.
           DISPLAY "REGION " WS-PREV-REGION " TOTAL: "
               WS-DISPLAY-AMT.
           ADD WS-REGION-TOTAL TO WS-GRAND-TOTAL.
           MOVE 0 TO WS-REGION-TOTAL.
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
REGION NORTH TOTAL:     1,500.00
REGION SOUTH TOTAL:       750.00
REGION WEST  TOTAL:     2,200.00
GRAND TOTAL:     4,450.00
```

### อธิบายจุดสำคัญ

- **`WS-FIRST-RECORD`** คือกลไกป้องกันไม่ให้เกิดการ "break" ที่ไม่มีความหมายในรอบแรกสุด (ยังไม่มี
  ข้อมูลก่อนหน้าให้เทียบ) — ถ้าลืมตรวจสอบเงื่อนไขนี้ รอบแรกจะพิมพ์ "REGION (ว่างเปล่า) TOTAL: 0.00"
  ออกมาก่อนข้อมูลจริงโดยไม่ตั้งใจ
- **`PERFORM REGION-BREAK` นอก loop หลังจบการวนซ้ำ** คือขั้นตอนสำคัญที่พลาดบ่อยที่สุด — ต้อง "flush"
  ยอดของกลุ่มสุดท้าย (WEST) ออกมาด้วย เพราะไม่มี record ถัดไปมากระตุ้นให้เกิด break ตามธรรมชาติ
- ข้อมูลทดสอบต้อง**เรียงลำดับตาม Region มาก่อนแล้ว**เสมอ (ในตัวอย่างนี้จัดกลุ่มไว้ให้แล้ว) — ในระบบ
  จริง ข้อมูลดิบมักไม่เรียงลำดับมาให้ จึงต้องผ่าน `SORT` (Part 027) ก่อนเสมอ ซึ่งจะสาธิตร่วมกับ
  ขั้นตอนนี้แบบเต็มรูปแบบในขั้นตอนที่ 660

### ข้อควรระวัง

- ตรรกะแบบนี้ (การเช็ค `NOT IS-FIRST-RECORD` ก่อนเทียบ) เป็นรูปแบบที่ต้องเขียนซ้ำทุกครั้งที่ทำ
  Control Break ด้วยมือ ต่างจาก Report Writer ที่จัดการเรื่องนี้ให้อัตโนมัติ (Part 038-039) —
  ข้อแลกเปลี่ยนคือความเข้ากันได้ข้ามแพลตฟอร์มที่สูงกว่า แลกกับโค้ดที่ต้องเขียนเองมากกว่า
- ถ้าลืม `PERFORM REGION-BREAK` ตัวสุดท้ายหลัง loop จบ ยอดของกลุ่มสุดท้ายจะหายไปจากรายงานโดยไม่มี
  error ใด ๆ แจ้งเตือน (ผลรวม Grand Total ก็จะผิดตามไปด้วยแบบเงียบ ๆ) นี่คือข้อผิดพลาดที่พบบ่อยที่สุด
  ในการเขียน Control Break ด้วยมือ

### แบบฝึกหัดที่ 653.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้ต้องมีคำสั่ง `PERFORM REGION-BREAK` อยู่ทั้งภายใน loop (เมื่อ
region เปลี่ยน) และหลัง loop จบ (สำหรับกลุ่มสุดท้าย) — ทำไมไม่ใช้แค่จุดเดียว

**เฉลย**: เพราะ break ภายใน loop จะทำงานก็ต่อเมื่อ**พบ record ของกลุ่มถัดไป**เข้ามาเปรียบเทียบ (ถ้า
region เปลี่ยนจาก NORTH เป็น SOUTH แสดงว่า NORTH จบกลุ่มแล้ว) แต่กลุ่มสุดท้าย (WEST ในตัวอย่างนี้) ไม่มี
record ถัดไปมากระตุ้นให้เกิดการเปรียบเทียบอีกเลย เพราะไฟล์จบแล้ว ถ้าไม่มีการ `PERFORM REGION-BREAK`
เพิ่มเติมหลัง loop จบ ยอดสะสมของกลุ่มสุดท้ายที่ค้างอยู่ใน `WS-REGION-TOTAL` จะไม่ถูกพิมพ์ออกมาและไม่
ถูกรวมเข้า Grand Total เลย ทำให้รายงานผิดพลาด

---

## ขั้นตอนที่ 654: Multi-Level Control Break — สรุปยอดหลายระดับชั้นพร้อมกัน

### เมื่อธุรกิจต้องการมากกว่าหนึ่งระดับของการสรุปยอด

รายงานธุรกิจจริงมักต้องการสรุปยอดซ้อนกันหลายชั้น เช่น "ยอดขายแยกตามแผนก **ภายใน**แต่ละภูมิภาค แล้ว
สรุปรวมทั้งภูมิภาค" — นี่คือ **Multi-Level Control Break** ที่มี Key รอง (Minor: แผนก) ซ้อนอยู่ภายใน
Key หลัก (Major: ภูมิภาค)

### โค้ดตัวอย่างสมบูรณ์ (ทดสอบจริงแล้ว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP654-MULTILEVEL-BREAK.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SALES-FILE ASSIGN TO "SALES654.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  SALES-FILE.
       01  SALES-RECORD.
           05  SR-REGION            PIC X(5).
           05  SR-DEPT              PIC X(5).
           05  SR-AMOUNT            PIC 9(6)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-PREV-REGION           PIC X(5) VALUE SPACES.
       01  WS-PREV-DEPT             PIC X(5) VALUE SPACES.
       01  WS-FIRST-RECORD          PIC X VALUE "Y".
           88  IS-FIRST-RECORD      VALUE "Y".
       01  WS-DEPT-TOTAL            PIC 9(8)V99 VALUE 0.
       01  WS-REGION-TOTAL          PIC 9(8)V99 VALUE 0.
       01  WS-GRAND-TOTAL           PIC 9(9)V99 VALUE 0.
       01  WS-DISPLAY-AMT           PIC Z,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.
           OPEN INPUT SALES-FILE.
           PERFORM READ-SALES.

      *> Two-level control break: REGION is the MAJOR key, DEPT is
      *> the MINOR key inside it. This is the classic pattern used
      *> in nightly batch sales reports (build on Part 026/039).
           PERFORM UNTIL END-OF-FILE
               IF NOT IS-FIRST-RECORD
                   IF SR-REGION NOT = WS-PREV-REGION
                       PERFORM DEPT-BREAK
                       PERFORM REGION-BREAK
                   ELSE
                       IF SR-DEPT NOT = WS-PREV-DEPT
                           PERFORM DEPT-BREAK
                       END-IF
                   END-IF
               END-IF
               DISPLAY "  DETAIL " SR-REGION " " SR-DEPT " "
                   SR-AMOUNT
               ADD SR-AMOUNT TO WS-DEPT-TOTAL
               MOVE SR-REGION TO WS-PREV-REGION
               MOVE SR-DEPT TO WS-PREV-DEPT
               MOVE "N" TO WS-FIRST-RECORD
               PERFORM READ-SALES
           END-PERFORM.

      *> Final breaks after the last record - a batch report must
      *> always flush the last group's totals before ending.
           PERFORM DEPT-BREAK.
           PERFORM REGION-BREAK.
           MOVE WS-GRAND-TOTAL TO WS-DISPLAY-AMT.
           DISPLAY "GRAND TOTAL:            " WS-DISPLAY-AMT.

           CLOSE SALES-FILE.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT SALES-FILE.
           MOVE "NORTH" TO SR-REGION. MOVE "SALES" TO SR-DEPT.
           MOVE 1000.00 TO SR-AMOUNT. WRITE SALES-RECORD.
           MOVE "NORTH" TO SR-REGION. MOVE "SALES" TO SR-DEPT.
           MOVE 500.00 TO SR-AMOUNT.  WRITE SALES-RECORD.
           MOVE "NORTH" TO SR-REGION. MOVE "IT   " TO SR-DEPT.
           MOVE 2000.00 TO SR-AMOUNT. WRITE SALES-RECORD.
           MOVE "SOUTH" TO SR-REGION. MOVE "IT   " TO SR-DEPT.
           MOVE 750.00 TO SR-AMOUNT.  WRITE SALES-RECORD.
           MOVE "SOUTH" TO SR-REGION. MOVE "SALES" TO SR-DEPT.
           MOVE 1250.00 TO SR-AMOUNT. WRITE SALES-RECORD.
           CLOSE SALES-FILE.

       READ-SALES.
           READ SALES-FILE
               AT END
                   SET END-OF-FILE TO TRUE
           END-READ.

       DEPT-BREAK.
           MOVE WS-DEPT-TOTAL TO WS-DISPLAY-AMT.
           DISPLAY "    -- DEPT " WS-PREV-DEPT " TOTAL: "
               WS-DISPLAY-AMT.
           ADD WS-DEPT-TOTAL TO WS-REGION-TOTAL.
           MOVE 0 TO WS-DEPT-TOTAL.

       REGION-BREAK.
           MOVE WS-REGION-TOTAL TO WS-DISPLAY-AMT.
           DISPLAY "  ---- REGION " WS-PREV-REGION " TOTAL: "
               WS-DISPLAY-AMT.
           ADD WS-REGION-TOTAL TO WS-GRAND-TOTAL.
           MOVE 0 TO WS-REGION-TOTAL.
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
  DETAIL NORTH SALES 001000.00
  DETAIL NORTH SALES 000500.00
    -- DEPT SALES TOTAL:     1,500.00
  DETAIL NORTH IT    002000.00
    -- DEPT IT    TOTAL:     2,000.00
  ---- REGION NORTH TOTAL:     3,500.00
  DETAIL SOUTH IT    000750.00
    -- DEPT IT    TOTAL:       750.00
  DETAIL SOUTH SALES 001250.00
    -- DEPT SALES TOTAL:     1,250.00
  ---- REGION SOUTH TOTAL:     2,000.00
GRAND TOTAL:                5,500.00
```

### อธิบายจุดสำคัญ

- **เมื่อ Key หลัก (Region) เปลี่ยน ต้อง break ทั้ง Key รองและ Key หลักตามลำดับจากในสู่นอก**
  (`PERFORM DEPT-BREAK` ก่อน แล้วค่อย `PERFORM REGION-BREAK`) — เพราะยอดของแผนกสุดท้ายในภูมิภาคเก่า
  ต้องถูกรวมเข้ายอดภูมิภาคก่อน ยอดภูมิภาคจึงจะถูกต้องสมบูรณ์
- **เมื่อ Key รอง (Dept) เปลี่ยนแต่ Key หลัก (Region) ยังเหมือนเดิม ให้ break แค่ Key รองเท่านั้น**
  (สังเกต `ELSE` block ที่ตรวจสอบ `SR-DEPT NOT = WS-PREV-DEPT` แยกต่างหาก)
- `DEPT-BREAK` มีบรรทัด `ADD WS-DEPT-TOTAL TO WS-REGION-TOTAL` — นี่คือกลไก "ไหลขึ้น" ของยอดจาก
  ระดับย่อยไปสู่ระดับใหญ่กว่า ซึ่งเป็นหัวใจของ Multi-Level Control Break

### ข้อควรระวัง

- ลำดับการ `PERFORM` ตอน break สำคัญมาก **ต้องเรียงจากระดับย่อยที่สุดไปหาระดับใหญ่สุดเสมอ** (Dept
  ก่อน Region) ถ้าสลับลำดับ ยอด Region จะถูกคำนวณโดยไม่รวมยอดของ Dept กลุ่มสุดท้ายที่เพิ่งจบไป
- ยิ่งมีจำนวนระดับมากขึ้น (เช่น ประเทศ → ภูมิภาค → แผนก → พนักงาน) จำนวนตัวแปรสะสมยอดและความซับซ้อน
  ของเงื่อนไข `IF` จะเพิ่มขึ้นตามไปด้วย ควรออกแบบให้ชัดเจนเป็นชั้น ๆ เสมอ ไม่ผสมปนกัน

### แบบฝึกหัดที่ 654.1

**โจทย์**: ถ้าต้องการเพิ่มระดับที่สาม (ประเทศ ครอบ Region ไว้อีกชั้น) จะต้องเพิ่มตัวแปรและ paragraph
อะไรบ้าง และลำดับการ `PERFORM` ตอน break จะเป็นอย่างไร

**เฉลยแนวทาง**: ต้องเพิ่ม `WS-PREV-COUNTRY`, `WS-COUNTRY-TOTAL` และ paragraph `COUNTRY-BREAK` ที่มี
`ADD WS-COUNTRY-TOTAL TO WS-GRAND-TOTAL` ลำดับการ `PERFORM` เมื่อ Country เปลี่ยนจะกลายเป็น
`PERFORM DEPT-BREAK` แล้ว `PERFORM REGION-BREAK` แล้วจึง `PERFORM COUNTRY-BREAK` (ไล่จากระดับย่อย
สุดไปหาระดับใหญ่สุดเสมอ ตามหลักการในขั้นตอนนี้) และต้องเพิ่มเงื่อนไขตรวจสอบ 3 ชั้นซ้อนกันแทนที่จะเป็น
2 ชั้น: ถ้า Country เปลี่ยน break ทั้งสาม, ถ้า Country เหมือนแต่ Region เปลี่ยน break สองระดับล่าง,
ถ้า Region เหมือนแต่ Dept เปลี่ยน break แค่ระดับเดียว

---

## ขั้นตอนที่ 655: Restart/Checkpoint Pattern — กู้คืนงาน Batch ที่ล้มเหลวกลางทาง

### ปัญหาที่ Batch งานใหญ่ต้องเผชิญ

ลองจินตนาการงาน batch ที่ประมวลผลธุรกรรม 10 ล้านรายการ ใช้เวลา 4 ชั่วโมง ถ้าระบบล่มกลางทาง (ไฟดับ,
ฮาร์ดแวร์ขัดข้อง) ที่รายการที่ 7 ล้าน **การเริ่มรันใหม่ตั้งแต่รายการที่ 1 จะเสียเวลาซ้ำซ้อนมหาศาล**
และในหลายกรณี (เช่น ถ้างานนั้นตัดเงินจากบัญชีลูกค้าไปแล้วบางส่วน) **การรันซ้ำตั้งแต่ต้นอาจทำให้เกิด
การประมวลผลซ้ำซ้อน (double processing)** ด้วย

**Checkpoint/Restart Pattern** แก้ปัญหานี้ด้วยการบันทึก **"ตำแหน่งล่าสุดที่ประมวลผลสำเร็จ"** ลงไฟล์
พิเศษเป็นระยะ ๆ ระหว่างการทำงาน เมื่อโปรแกรมต้องรันใหม่ (restart) มันจะอ่าน checkpoint นี้ก่อน แล้ว
**ข้ามรายการที่ประมวลผลไปแล้ว** ไปเริ่มทำงานต่อจากจุดที่ค้างไว้แทน

### โค้ดตัวอย่างสมบูรณ์ (ทดสอบจริงแล้ว — จำลองการ crash และ restart ในโปรแกรมเดียว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP655-CHECKPOINT-RESTART.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANS-FILE ASSIGN TO "BIGTRANS655.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT CHECKPOINT-FILE ASSIGN TO "CKPT655.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANS-FILE.
       01  TRANS-RECORD.
           05  TR-SEQ-NUM           PIC 9(4).
           05  TR-AMOUNT            PIC 9(6)V99.

       FD  CHECKPOINT-FILE.
       01  CKPT-RECORD.
           05  CKPT-LAST-SEQ        PIC 9(4).
           05  CKPT-RUNNING-TOTAL   PIC 9(9)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-LAST-CHECKPOINT       PIC 9(4) VALUE 0.
       01  WS-RECORDS-THIS-RUN      PIC 9(4) VALUE 0.
       01  WS-SIMULATED-CRASH-AT    PIC 9(4) VALUE 0.
       01  WS-RUNNING-TOTAL         PIC 9(9)V99 VALUE 0.
       01  WS-CRASHED-FLAG          PIC X VALUE "N".
           88  JOB-CRASHED          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.

           DISPLAY "===== RUN 1 (simulates crash at seq 0006) =====".
           MOVE 0 TO WS-LAST-CHECKPOINT.
           MOVE 6 TO WS-SIMULATED-CRASH-AT.
           PERFORM PROCESS-TRANSACTIONS.

           DISPLAY "===== RUN 2 (restart - reads checkpoint) =====".
      *> A real restart is a brand new JCL/job invocation that
      *> starts with EMPTY working storage - we simulate that by
      *> resetting the total here and rebuilding it ONLY from what
      *> the checkpoint file says was already accumulated.
           MOVE 0 TO WS-RUNNING-TOTAL.
           PERFORM READ-CHECKPOINT.
           DISPLAY "Resuming after seq: " WS-LAST-CHECKPOINT.
           DISPLAY "Restored running total: " WS-RUNNING-TOTAL.
           MOVE 9999 TO WS-SIMULATED-CRASH-AT.
           PERFORM PROCESS-TRANSACTIONS.

           STOP RUN.

       BUILD-TEST-DATA.
      *> 10 transactions simulate a large overnight batch job.
           OPEN OUTPUT TRANS-FILE.
           MOVE 1 TO TR-SEQ-NUM. MOVE 100.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 2 TO TR-SEQ-NUM. MOVE 200.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 3 TO TR-SEQ-NUM. MOVE 150.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 4 TO TR-SEQ-NUM. MOVE 300.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 5 TO TR-SEQ-NUM. MOVE 250.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 6 TO TR-SEQ-NUM. MOVE 400.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 7 TO TR-SEQ-NUM. MOVE 175.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 8 TO TR-SEQ-NUM. MOVE 225.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 9 TO TR-SEQ-NUM. MOVE 500.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE 10 TO TR-SEQ-NUM. MOVE 350.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           CLOSE TRANS-FILE.

       PROCESS-TRANSACTIONS.
           MOVE "N" TO WS-EOF-FLAG.
           MOVE "N" TO WS-CRASHED-FLAG.
           MOVE 0 TO WS-RECORDS-THIS-RUN.
           OPEN INPUT TRANS-FILE.
           PERFORM UNTIL END-OF-FILE OR JOB-CRASHED
               READ TRANS-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
      *> Skip anything already processed in a prior run. This is
      *> what makes a restart safe: we never reprocess a record
      *> whose sequence number is at or below the checkpoint.
                       IF TR-SEQ-NUM > WS-LAST-CHECKPOINT
                           IF TR-SEQ-NUM = WS-SIMULATED-CRASH-AT
                               DISPLAY "  *** SIMULATED CRASH before "
                                   "processing seq "
                                   TR-SEQ-NUM " ***"
                               PERFORM WRITE-CHECKPOINT
                               SET JOB-CRASHED TO TRUE
                           ELSE
                               ADD TR-AMOUNT TO WS-RUNNING-TOTAL
                               ADD 1 TO WS-RECORDS-THIS-RUN
                               DISPLAY "  processed seq " TR-SEQ-NUM
                                   " amount " TR-AMOUNT
                               MOVE TR-SEQ-NUM TO WS-LAST-CHECKPOINT
                               PERFORM WRITE-CHECKPOINT
                           END-IF
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE TRANS-FILE.
           DISPLAY "Records processed this run: "
               WS-RECORDS-THIS-RUN.
           DISPLAY "Running total after this run: " WS-RUNNING-TOTAL.

       WRITE-CHECKPOINT.
           OPEN OUTPUT CHECKPOINT-FILE.
           MOVE WS-LAST-CHECKPOINT TO CKPT-LAST-SEQ.
           MOVE WS-RUNNING-TOTAL TO CKPT-RUNNING-TOTAL.
           WRITE CKPT-RECORD.
           CLOSE CHECKPOINT-FILE.

       READ-CHECKPOINT.
           OPEN INPUT CHECKPOINT-FILE.
           READ CHECKPOINT-FILE
               AT END
                   MOVE 0 TO WS-LAST-CHECKPOINT
                   MOVE 0 TO WS-RUNNING-TOTAL
               NOT AT END
                   MOVE CKPT-LAST-SEQ TO WS-LAST-CHECKPOINT
                   MOVE CKPT-RUNNING-TOTAL TO WS-RUNNING-TOTAL
           END-READ.
           CLOSE CHECKPOINT-FILE.
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
===== RUN 1 (simulates crash at seq 0006) =====
  processed seq 0001 amount 000100.00
  processed seq 0002 amount 000200.00
  processed seq 0003 amount 000150.00
  processed seq 0004 amount 000300.00
  processed seq 0005 amount 000250.00
  *** SIMULATED CRASH before processing seq 0006 ***
Records processed this run: 0005
Running total after this run: 000001000.00
===== RUN 2 (restart - reads checkpoint) =====
Resuming after seq: 0005
Restored running total: 000001000.00
  processed seq 0006 amount 000400.00
  processed seq 0007 amount 000175.00
  processed seq 0008 amount 000225.00
  processed seq 0009 amount 000500.00
  processed seq 0010 amount 000350.00
Records processed this run: 0005
Running total after this run: 000002650.00
```

### อธิบายจุดสำคัญ

- **RUN 1** ประมวลผลได้แค่ 5 รายการ (seq 1-5) แล้ว "จำลองการ crash" ก่อนที่จะประมวลผล seq 6 — สังเกต
  ว่า checkpoint ถูกเขียนบันทึกค่า `CKPT-LAST-SEQ = 5` และ `CKPT-RUNNING-TOTAL = 1000.00` ไว้**ก่อน**
  ที่จะ "ล่ม"
- **RUN 2** จำลองการ restart จริง โดย**รีเซ็ต `WS-RUNNING-TOTAL` เป็น 0 ก่อน** (เพื่อจำลองว่า
  หน่วยความจำของ Task/Job ก่อนหน้าหายไปหมดแล้วจริง ๆ — เหมือนที่ Part 064 อธิบายไว้เรื่อง
  pseudo-conversational) แล้วจึงอ่านค่าที่บันทึกไว้กลับมาจาก `CKPT655.DAT`
- ผลลัพธ์สุดท้าย `2650.00` ถูกต้อง (100+200+150+300+250+400+175+225+500+350 = 2650) พิสูจน์ว่าไม่มี
  รายการใดถูกประมวลผลซ้ำหรือขาดหายไปเลย
- **การบันทึกทั้ง `CKPT-LAST-SEQ` และ `CKPT-RUNNING-TOTAL` ลง checkpoint พร้อมกัน** คือจุดออกแบบที่
  สำคัญมาก — ถ้าบันทึกแค่ตำแหน่งล่าสุดโดยไม่บันทึกยอดสะสม โปรแกรมที่ restart จะไม่มีทางรู้ยอดสะสม
  ก่อนหน้าได้เลย (นอกจากจะอ่านย้อนกลับไปคำนวณใหม่ทั้งหมด ซึ่งเสียเวลาพอ ๆ กับไม่มี checkpoint)

### ข้อควรระวัง

- **Checkpoint ควรเขียนบ่อยแค่ไหน** เป็นการตัดสินใจสำคัญที่ต้อง trade-off: เขียนทุก record (ปลอดภัย
  ที่สุด แต่ I/O overhead สูง) เทียบกับเขียนทุก N record (I/O น้อยกว่า แต่ถ้าล่มกลางช่วง N record
  จะต้องประมวลผลซ้ำ N-1 รายการสุดท้าย) — แนวคิดเดียวกับ **Commit Interval** ที่จะเรียนในขั้นตอนที่ 659
- ในระบบจริงบน Mainframe การ Restart มักควบคุมผ่าน JCL (`RESTART=` parameter บน `EXEC` statement
  ทบทวนจาก Part 053) ที่บอกให้ job เริ่มจาก step ใด ไม่ใช่แค่ตรรกะภายในโปรแกรมเพียงอย่างเดียว — ทั้ง
  สองกลไกทำงานร่วมกัน

### แบบฝึกหัดที่ 655.1

**โจทย์**: จงอธิบายว่าทำไมการเขียน checkpoint บ่อยเกินไป (เช่น ทุก 1 record) จึงไม่ใช่ทางเลือกที่ดี
ที่สุดเสมอไป ทั้งที่ดูปลอดภัยที่สุด

**เฉลย**: เพราะการเขียนไฟล์ (I/O operation) มีต้นทุนด้านเวลาและทรัพยากรสูงกว่าการประมวลผลในหน่วยความ
จำมาก การเขียน checkpoint ทุก 1 record หมายความว่าทุกรายการที่ประมวลผลต้องมีการเปิด/เขียน/ปิดไฟล์
checkpoint ตามไปด้วย ซึ่งจะทำให้เวลารวมของงาน batch ทั้งหมดช้าลงอย่างมีนัยสำคัญ โดยเฉพาะเมื่อมีข้อมูล
หลักล้านรายการ ในทางปฏิบัติจึงมักเลือกเขียน checkpoint เป็นช่วง ๆ (เช่น ทุก 1,000 หรือ 10,000
รายการ) เพื่อสร้างสมดุลระหว่างความเร็วของงานปกติ กับปริมาณงานที่ต้องทำซ้ำหากเกิดการล่มกลางทาง

---

## ขั้นตอนที่ 656: แนวคิด Job Scheduling — CA-7, Control-M และหลักการจัดตารางงาน

> ⚠️ **หมายเหตุ**: ขั้นตอนนี้เป็นเนื้อหาเชิงแนวคิดล้วน ๆ เกี่ยวกับซอฟต์แวร์เชิงพาณิชย์ที่ไม่มีการ
> ติดตั้งในสภาพแวดล้อมนี้ (และไม่มีทางติดตั้งได้ เพราะเป็นซอฟต์แวร์ลิขสิทธิ์ราคาสูงที่ใช้เฉพาะใน
> องค์กร) ไม่มีโค้ดให้ทดสอบในขั้นตอนนี้

### ปัญหาที่ Job Scheduler แก้ไข

องค์กรขนาดใหญ่มีงาน batch นับร้อยนับพันงานที่ต้องรันทุกคืน และงานเหล่านี้มัก**ขึ้นต่อกัน**
(dependency) เช่น "งานคำนวณดอกเบี้ยต้องรอให้งานปิดยอดธุรกรรมประจำวันเสร็จก่อน" หรือ "งานสร้างรายงาน
สรุปต้องรอให้ทุกสาขาส่งไฟล์ยอดขายมาครบก่อน" การจัดการ dependency ที่ซับซ้อนระดับนี้ด้วยมือ (หรือแค่
`cron` ธรรมดา) เป็นไปไม่ได้ในทางปฏิบัติ องค์กรจึงใช้ซอฟต์แวร์ **Job Scheduler** เชิงพาณิชย์ เช่น
**CA-7** (Broadcom) และ **Control-M** (BMC Software) เพื่อจัดการเรื่องนี้โดยเฉพาะ

### ความสามารถหลักของ Job Scheduler ระดับองค์กร

1. **Dependency Management**: กำหนดว่า Job B ต้องรอ Job A เสร็จสมบูรณ์ก่อนจึงจะเริ่มได้ (ต่างจาก
   `cron` ที่กำหนดแค่เวลา ไม่รู้จัก dependency ระหว่างงาน)
2. **Calendar-based Scheduling**: กำหนดตารางงานตามปฏิทินธุรกิจ (เช่น "รันเฉพาะวันทำการ ข้ามวันหยุด
   นักขัตฤกษ์" หรือ "รันเฉพาะวันสิ้นเดือน")
3. **Resource Management**: จำกัดจำนวนงานที่รันพร้อมกันได้ตามทรัพยากรที่มี (CPU, พื้นที่ดิสก์)
4. **Alerting และ Automatic Restart**: แจ้งเตือนทีมปฏิบัติการเมื่องานล้มเหลว และบางกรณีสามารถสั่ง
   restart งานที่ล้มเหลวโดยอัตโนมัติตามกฎที่ตั้งไว้
5. **Historical Reporting**: เก็บประวัติการรันงานทั้งหมด ใช้วิเคราะห์แนวโน้ม (เช่น งานไหนเริ่มใช้
   เวลานานขึ้นเรื่อย ๆ อาจเป็นสัญญาณของปัญหาประสิทธิภาพที่กำลังจะเกิดขึ้น)

### เปรียบเทียบแนวคิดกับ cron ของ Linux ที่นักเรียนอาจคุ้นเคย

| แง่มุม | `cron` (Linux) | CA-7/Control-M (Mainframe) |
|---|---|---|
| การกำหนดเวลา | Cron expression (`0 2 * * *`) | Calendar-based, รองรับปฏิทินธุรกิจซับซ้อน |
| Dependency ระหว่างงาน | ไม่มีในตัว (ต้องเขียน script เชื่อมเอง) | มีในตัว เป็นความสามารถหลัก |
| การแจ้งเตือนเมื่อล้มเหลว | ไม่มีในตัว (ต้องเขียนเพิ่มเอง) | มีในตัว พร้อม escalation rules |
| ขนาดองค์กรที่เหมาะสม | ระบบเดี่ยว/ทีมเล็ก | องค์กรขนาดใหญ่ที่มีงานนับพันงาน |
| ต้นทุน | ฟรี (มากับ Linux) | ซอฟต์แวร์ลิขสิทธิ์ราคาสูง |

### ตัวอย่างแนวคิด Dependency Graph (ไม่ใช่โค้ดที่รันได้ เป็นแผนภาพประกอบความเข้าใจ)

```
[JOB: DAILY-TRANS-CAPTURE] (รับไฟล์ธุรกรรมจากทุกสาขา 02:00)
            |
            v (ต้องเสร็จสมบูรณ์ก่อน)
[JOB: SORT-AND-VALIDATE]   (เรียงลำดับและตรวจสอบข้อมูล 02:30)
            |
            v
[JOB: SEQUENTIAL-UPDATE]   (ปรับปรุง Master ตาม Pattern ขั้นตอนที่ 652, 03:00)
            |
      +-----+-----+
      v           v  (สองงานนี้รันพร้อมกันได้ เพราะไม่ขึ้นต่อกัน)
[JOB: INTEREST-CALC]   [JOB: DAILY-REPORT]
      |                       |
      +-----+-----+
            v
[JOB: SEND-TO-REGULATORY-SYSTEM] (ต้องรอทั้งสองงานก่อนหน้าเสร็จ)
```

### ข้อควรระวัง

- Job Scheduler ระดับองค์กรเป็นการลงทุนที่มีค่าใช้จ่ายสูงมาก องค์กรขนาดเล็กหรือระบบที่มีงาน batch
  ไม่กี่งานอาจไม่คุ้มค่าที่จะใช้ และเลือกใช้ `cron` หรือเครื่องมือ open-source แทนได้
- อย่าสับสนระหว่าง "Job Scheduler" (ควบคุมว่า**เมื่อไร**และ**ตามลำดับไหน**ที่งานจะรัน) กับ "JCL"
  (Part 052-053, ควบคุมว่างานหนึ่งงาน**ทำอะไรบ้าง**ภายในตัวมันเอง) — ทั้งสองทำงานร่วมกัน: Job
  Scheduler เป็นผู้ "กด" ให้ JCL แต่ละงานเริ่มทำงานตามเงื่อนไขและเวลาที่กำหนดไว้

### แบบฝึกหัดที่ 656.1

**โจทย์**: จากแผนภาพ Dependency Graph ข้างต้น จงอธิบายว่าทำไม `JOB: INTEREST-CALC` และ
`JOB: DAILY-REPORT` จึงสามารถรันพร้อมกันได้ ในขณะที่ `JOB: SEND-TO-REGULATORY-SYSTEM` ต้องรอทั้งสอง
งานนั้นให้เสร็จก่อน

**เฉลย**: `JOB: INTEREST-CALC` และ `JOB: DAILY-REPORT` ไม่มีความสัมพันธ์แบบพึ่งพากันโดยตรง (งานหนึ่ง
ไม่ต้องใช้ผลลัพธ์ของอีกงานหนึ่งเป็น input) ทั้งสองต้องการแค่ผลลัพธ์จาก `JOB: SEQUENTIAL-UPDATE`
เท่านั้น จึงสามารถรันขนานกันได้เพื่อประหยัดเวลารวมของ batch window (ช่วงเวลาที่จัดสรรไว้สำหรับงาน
กลางคืน) ในขณะที่ `JOB: SEND-TO-REGULATORY-SYSTEM` ต้องการข้อมูลจาก**ทั้งสองงาน**ก่อนหน้า (เช่น
ต้องส่งทั้งยอดดอกเบี้ยที่คำนวณแล้วและรายงานสรุปยอดขายไปยังหน่วยงานกำกับดูแลพร้อมกัน) จึงต้องรอให้ทั้ง
สองงานเสร็จสมบูรณ์ก่อนจึงจะเริ่มทำงานได้ นี่คือตัวอย่างของ "AND dependency" ที่ Job Scheduler ต้อง
จัดการ

---

## ขั้นตอนที่ 657: RETURN-CODE และการเชื่อมโยงกับ JCL Condition Codes

### RETURN-CODE คือสะพานเชื่อมระหว่างโปรแกรมกับผู้ที่เรียกมัน

ทบทวนจาก Part 053 (JCL ขั้นสูง): แต่ละ Job Step ใน JCL จะจบด้วย **Condition Code** ที่ step ถัดไป
สามารถตรวจสอบผ่าน `COND=` หรือ `IF (stepname.RC ...)` เพื่อตัดสินใจว่าจะรันต่อหรือข้ามไป
**RETURN-CODE** คือ Special Register ของ COBOL ที่โปรแกรมใช้ **"ส่งค่า Condition Code" นี้กลับไปให้
JCL** เมื่อ `STOP RUN`

### ตัวอย่างที่ทดสอบได้จริง: กำหนด RETURN-CODE ตามผลการตรวจสอบข้อมูล

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP657-RETURN-CODE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANS-FILE ASSIGN TO "TRANS657.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANS-FILE.
       01  TRANS-RECORD.
           05  TR-ID                PIC X(4).
           05  TR-AMOUNT            PIC S9(6)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-WARNING-COUNT         PIC 9(3) VALUE 0.
       01  WS-ERROR-COUNT           PIC 9(3) VALUE 0.
       01  WS-RECORD-COUNT          PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.
           OPEN INPUT TRANS-FILE.
           PERFORM UNTIL END-OF-FILE
               READ TRANS-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-RECORD-COUNT
      *> A negative amount is unusual but not fatal - flag it as
      *> a WARNING. A zero amount is treated as a hard ERROR here.
                       EVALUATE TRUE
                           WHEN TR-AMOUNT < 0
                               DISPLAY "WARNING: negative amount on "
                                   TR-ID
                               ADD 1 TO WS-WARNING-COUNT
                           WHEN TR-AMOUNT = 0
                               DISPLAY "ERROR: zero amount on " TR-ID
                               ADD 1 TO WS-ERROR-COUNT
                           WHEN OTHER
                               DISPLAY "OK: " TR-ID " " TR-AMOUNT
                       END-EVALUATE
               END-READ
           END-PERFORM.
           CLOSE TRANS-FILE.

           DISPLAY "Records : " WS-RECORD-COUNT.
           DISPLAY "Warnings: " WS-WARNING-COUNT.
           DISPLAY "Errors  : " WS-ERROR-COUNT.

      *> RETURN-CODE is a COBOL special register. On z/OS, JCL
      *> reads the terminating program's return code as the job
      *> step's COMPLETION CODE, and a later step can test it with
      *> IF (stepname.RC ...) or COND=(n,GT,stepname) to decide
      *> whether to even run. On Linux, the same value becomes the
      *> process exit status ($? in the shell) - same idea, just a
      *> different name for the mechanism that reads it.
           EVALUATE TRUE
               WHEN WS-ERROR-COUNT > 0
                   MOVE 8 TO RETURN-CODE
               WHEN WS-WARNING-COUNT > 0
                   MOVE 4 TO RETURN-CODE
               WHEN OTHER
                   MOVE 0 TO RETURN-CODE
           END-EVALUATE.
           DISPLAY "RETURN-CODE set to: " RETURN-CODE.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT TRANS-FILE.
           MOVE "T001" TO TR-ID. MOVE 100.00  TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE "T002" TO TR-ID. MOVE -50.00  TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE "T003" TO TR-ID. MOVE 250.00  TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           CLOSE TRANS-FILE.
```

คอมไพล์ รัน และตรวจสอบ exit status ของเชลล์ (`$?` — เทียบเท่า Condition Code ของ JCL):

```bash
cobc -x -o step657 step657.cob
./step657
echo "shell exit status (\$?): $?"
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
OK: T001 +000100.00
WARNING: negative amount on T002
OK: T003 +000250.00
Records : 003
Warnings: 001
Errors  : 000
RETURN-CODE set to: +000000004
shell exit status ($?): 4
```

### อธิบายจุดสำคัญ

- **`RETURN-CODE` เป็น Special Register ที่ COBOL ประกาศไว้ให้อัตโนมัติ** ไม่ต้องประกาศเองใน
  `WORKING-STORAGE` (สังเกตว่าโค้ดข้างต้นไม่มีการประกาศ `01 RETURN-CODE` ที่ไหนเลย)
- **ค่าที่ `MOVE` เข้า `RETURN-CODE` ก่อน `STOP RUN` จะกลายเป็น process exit status ของ GnuCOBOL
  โดยตรง** — ผลลัพธ์ยืนยันชัดเจนว่า `RETURN-CODE = 4` แปลงเป็น `$? = 4` ในเชลล์ทันที นี่คือ**หลักฐาน
  จริง**ว่ากลไกเดียวกันนี้คือสิ่งที่ JCL ใช้ตรวจสอบผ่าน `COND=`/`IF (stepname.RC ...)` บน z/OS จริง
  (แนวคิดเดียวกัน เพียงแค่ระบบปฏิบัติการเรียกชื่อกลไกนี้ต่างกัน)
- ธรรมเนียมของ Condition Code ที่นิยมใช้กันในอุตสาหกรรม (ไม่ใช่กฎตายตัว แต่เป็นธรรมเนียมที่พบบ่อย
  มาก): `0` = สำเร็จสมบูรณ์, `4` = สำเร็จแต่มีคำเตือน, `8` = มีข้อผิดพลาดที่ควรหยุดกระบวนการ, `12`
  ขึ้นไป = ข้อผิดพลาดร้ายแรง

### ข้อควรระวัง

- ค่า `RETURN-CODE` ต้อง `MOVE` **ก่อน** `STOP RUN` เสมอ — ถ้า `STOP RUN` ทำงานไปแล้วโดยไม่ได้ตั้งค่า
  `RETURN-CODE` ไว้ล่วงหน้า ค่าเริ่มต้นจะเป็น 0 เสมอ (ซึ่งอาจทำให้ JCL step ถัดไปเข้าใจผิดว่าทุกอย่าง
  สำเร็จ ทั้งที่จริงมีปัญหาเกิดขึ้น)
- แต่ละองค์กรมักมีธรรมเนียมการกำหนดความหมายของ Condition Code ที่แตกต่างกันไปบ้าง ควรตรวจสอบมาตรฐาน
  ขององค์กรก่อนกำหนดค่าเอง เพื่อให้สอดคล้องกับระบบ monitoring และ Job Scheduler ที่มีอยู่แล้ว (ทบทวน
  จากขั้นตอนที่ 656)

### แบบฝึกหัดที่ 657.1

**โจทย์**: จงอธิบายว่าทำไม Job Scheduler (CA-7/Control-M จากขั้นตอนที่ 656) จึงสามารถใช้ค่า
`RETURN-CODE` ของโปรแกรมหนึ่งเป็นเงื่อนไขในการตัดสินใจว่าจะรัน Job ถัดไปหรือไม่

**เฉลยแนวทาง**: เพราะ `RETURN-CODE` เป็นค่าที่ระบบปฏิบัติการ (z/OS หรือ Linux) เก็บไว้เป็นส่วนหนึ่ง
ของผลลัพธ์การรันโปรแกรมเสมอ ไม่ว่าจะเรียกชื่อว่า "Condition Code" หรือ "Exit Status" ก็ตาม เมื่อ
โปรแกรมจบการทำงาน ค่านี้จะถูกส่งกลับไปยังกระบวนการที่เรียกมัน (ใน JCL คือ Job Step หรือ Job Scheduler
ที่ควบคุมอยู่) Job Scheduler จึงสามารถอ่านค่านี้และเปรียบเทียบกับเงื่อนไขที่ผู้ดูแลระบบกำหนดไว้ (เช่น
"ถ้า RETURN-CODE ของ Job A มากกว่า 4 ให้หยุดกระบวนการทั้งหมดและแจ้งเตือนทีมปฏิบัติการ แทนที่จะรัน
Job B ต่อ") ได้โดยไม่ต้องพึ่งพากลไกอื่นใดนอกเหนือจากสิ่งที่ระบบปฏิบัติการมอบให้อยู่แล้ว

---

## ขั้นตอนที่ 658: Idempotent Batch Design — ออกแบบให้รันซ้ำได้อย่างปลอดภัย

### ปัญหาของ "รันงานซ้ำโดยไม่ตั้งใจ"

ในโลกจริง งาน batch อาจถูกรันซ้ำโดยไม่ตั้งใจได้หลายสาเหตุ: operator กดส่งงานผิดพลาดสองครั้ง, Job
Scheduler ตั้งค่าผิดทำให้ trigger ซ้ำ, หรือแม้แต่การ restart จากขั้นตอนที่ 655 ที่ทำผิดจุด **หากงาน
batch ไม่ได้ถูกออกแบบมาให้รองรับการรันซ้ำ ผลลัพธ์ของการรันซ้ำจะสร้างความเสียหาย** เช่น ระบบตัดเงิน
จากบัญชีลูกค้าซ้ำสองครั้งสำหรับธุรกรรมเดียวกัน

**Idempotent** คือคุณสมบัติที่ทำให้ **"รันกี่ครั้งก็ได้ผลลัพธ์สุดท้ายเหมือนเดิมเสมอ"** — เป็นแนวคิด
สำคัญมากสำหรับงาน batch ระดับองค์กรทุกงาน

### โค้ดตัวอย่างสมบูรณ์ (ทดสอบจริงแล้ว): บันทึกรหัสธุรกรรมที่ประมวลผลแล้ว เพื่อป้องกันการทำซ้ำ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP658-IDEMPOTENT-BATCH.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANS-FILE ASSIGN TO "TRANS658.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT PROCESSED-LOG ASSIGN TO "PROCLOG658.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANS-FILE.
       01  TRANS-RECORD.
           05  TR-ID                PIC X(6).
           05  TR-AMOUNT            PIC 9(6)V99.

       FD  PROCESSED-LOG.
       01  LOG-RECORD               PIC X(6).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-ALREADY-DONE          PIC X VALUE "N".
           88  ALREADY-PROCESSED    VALUE "Y".
       01  WS-PROCESSED-TABLE.
           05  WS-PROCESSED-ENTRY PIC X(6)
                   OCCURS 20 TIMES INDEXED BY LOG-IDX.
       01  WS-PROCESSED-COUNT       PIC 9(3) VALUE 0.
       01  WS-APPLIED-COUNT         PIC 9(3) VALUE 0.
       01  WS-SKIPPED-COUNT         PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.

           DISPLAY "===== FIRST RUN =====".
           PERFORM RUN-BATCH.

           DISPLAY "===== SECOND RUN (operator resubmitted job) =====".
           PERFORM RUN-BATCH.

           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT TRANS-FILE.
           MOVE "TX0001" TO TR-ID. MOVE 100.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE "TX0002" TO TR-ID. MOVE 200.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           MOVE "TX0003" TO TR-ID. MOVE 300.00 TO TR-AMOUNT.
           WRITE TRANS-RECORD.
           CLOSE TRANS-FILE.

      *> The processed-ID log starts EMPTY only the very first
      *> time. On every later run we must load whatever IDs are
      *> already recorded - this file is what makes reruns safe.
           OPEN OUTPUT PROCESSED-LOG.
           CLOSE PROCESSED-LOG.

       RUN-BATCH.
           PERFORM LOAD-PROCESSED-LOG.
           MOVE 0 TO WS-APPLIED-COUNT.
           MOVE 0 TO WS-SKIPPED-COUNT.
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT TRANS-FILE.
           OPEN EXTEND PROCESSED-LOG.
           PERFORM UNTIL END-OF-FILE
               READ TRANS-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       PERFORM CHECK-ALREADY-PROCESSED
                       IF ALREADY-PROCESSED
                           DISPLAY "  SKIP (already applied): "
                               TR-ID
                           ADD 1 TO WS-SKIPPED-COUNT
                       ELSE
                           DISPLAY "  APPLY: " TR-ID " amount "
                               TR-AMOUNT
                           MOVE TR-ID TO LOG-RECORD
                           WRITE LOG-RECORD
                           ADD 1 TO WS-APPLIED-COUNT
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE TRANS-FILE.
           CLOSE PROCESSED-LOG.
           DISPLAY "Applied: " WS-APPLIED-COUNT
               " Skipped: " WS-SKIPPED-COUNT.

       LOAD-PROCESSED-LOG.
           MOVE 0 TO WS-PROCESSED-COUNT.
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT PROCESSED-LOG.
           PERFORM UNTIL END-OF-FILE
               READ PROCESSED-LOG
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-PROCESSED-COUNT
                       MOVE LOG-RECORD
                           TO WS-PROCESSED-ENTRY (WS-PROCESSED-COUNT)
               END-READ
           END-PERFORM.
           CLOSE PROCESSED-LOG.

       CHECK-ALREADY-PROCESSED.
           MOVE "N" TO WS-ALREADY-DONE.
           IF WS-PROCESSED-COUNT > 0
               SET LOG-IDX TO 1
               SEARCH WS-PROCESSED-ENTRY
                   AT END
                       MOVE "N" TO WS-ALREADY-DONE
                   WHEN WS-PROCESSED-ENTRY (LOG-IDX) = TR-ID
                       SET ALREADY-PROCESSED TO TRUE
               END-SEARCH
           END-IF.
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
===== FIRST RUN =====
  APPLY: TX0001 amount 000100.00
  APPLY: TX0002 amount 000200.00
  APPLY: TX0003 amount 000300.00
Applied: 003 Skipped: 000
===== SECOND RUN (operator resubmitted job) =====
  SKIP (already applied): TX0001
  SKIP (already applied): TX0002
  SKIP (already applied): TX0003
Applied: 000 Skipped: 003
```

### อธิบายจุดสำคัญ

- **`FIRST RUN`** ประมวลผลทั้ง 3 ธุรกรรมตามปกติ และบันทึกรหัสธุรกรรมแต่ละตัวลง `PROCLOG658.DAT`
- **`SECOND RUN`** (จำลองว่า operator รันงานเดิมซ้ำโดยไม่ตั้งใจ) — โปรแกรม `LOAD-PROCESSED-LOG`
  ก่อนเริ่มเสมอ แล้วใช้ `SEARCH` (ทบทวนจาก Part 017) ตรวจสอบว่ารหัสธุรกรรมนี้เคยถูกประมวลผลไปแล้ว
  หรือยัง ผลคือทั้ง 3 ธุรกรรมถูก **SKIP** ทั้งหมด ไม่มีการประมวลผลซ้ำเลย
- **`OPEN EXTEND PROCESSED-LOG`** (ไม่ใช่ `OPEN OUTPUT`) คือกุญแจสำคัญ — `EXTEND` เปิดไฟล์เพื่อ
  **เพิ่มข้อมูลต่อท้าย**ของที่มีอยู่แล้ว แทนที่จะเขียนทับไฟล์เดิมทั้งหมด (ถ้าใช้ `OPEN OUTPUT` แทน
  ทุกครั้งที่รันจะล้างประวัติเก่าทิ้งหมด ทำให้กลไก idempotent ใช้ไม่ได้เลย)

### ข้อควรระวัง

- ตัวอย่างนี้ใช้ตาราง `WS-PROCESSED-TABLE OCCURS 20 TIMES` เพื่อความง่ายในการสาธิต — ในระบบจริงที่มี
  ธุรกรรมนับล้านรายการ การโหลดประวัติทั้งหมดเข้าตารางในหน่วยความจำแบบนี้ไม่สามารถทำได้ (จะ Subscript
  เกินขอบเขตทันที) ควรใช้ไฟล์ **Indexed** (Part 028) แทน เพื่อค้นหาด้วย `READ ... INVALID KEY` แบบ
  `ACCESS MODE IS RANDOM` ซึ่งไม่จำกัดจำนวนรายการในหน่วยความจำ
- Idempotent Design ไม่ได้แปลว่า "ห้ามรันซ้ำ" แต่หมายถึง "รันซ้ำได้อย่างปลอดภัยโดยผลลัพธ์สุดท้ายไม่
  เปลี่ยนแปลง" — เป็นคนละแนวคิดกับการป้องกันไม่ให้รันซ้ำ (ซึ่งบางครั้งทำไม่ได้จริงในทางปฏิบัติ เพราะ
  ควบคุมพฤติกรรมของมนุษย์ไม่ได้ 100%)

### แบบฝึกหัดที่ 658.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `OPEN EXTEND` แทน `OPEN OUTPUT` สำหรับไฟล์ `PROCESSED-LOG` จึงเป็น
จุดสำคัญที่สุดที่ทำให้กลไก Idempotent ในตัวอย่างนี้ทำงานได้ถูกต้อง

**เฉลย**: เพราะ `OPEN OUTPUT` จะสร้างไฟล์ใหม่และล้างเนื้อหาเดิมทิ้งทุกครั้งที่เปิด (ทบทวนจาก Part 025)
ถ้าใช้ `OPEN OUTPUT` แทน `OPEN EXTEND` ทุกครั้งที่ `RUN-BATCH` ถูกเรียก ประวัติการประมวลผลที่บันทึกไว้
จากการรันครั้งก่อนหน้าจะถูกลบทิ้งไปทันที ทำให้ `LOAD-PROCESSED-LOG` ในรอบถัดไปโหลดได้แค่ตารางว่างเปล่า
และ `CHECK-ALREADY-PROCESSED` จะไม่พบว่าธุรกรรมใดเคยถูกประมวลผลมาก่อนเลย ส่งผลให้ทุกธุรกรรมถูกประมวลผล
ซ้ำอีกครั้งอย่างผิดพลาด `OPEN EXTEND` ทำให้ข้อมูลที่เขียนไปก่อนหน้ายังคงอยู่ และข้อมูลใหม่ถูกเพิ่มต่อ
ท้ายเท่านั้น ทำให้ประวัติสะสมของทุกการรันยังคงอยู่ครบถ้วน

---

## ขั้นตอนที่ 659: Commit Interval — สมดุลระหว่างความปลอดภัยและประสิทธิภาพ

### แนวคิดที่ต่อยอดจาก Checkpoint (ขั้นตอนที่ 655)

ในระบบที่เชื่อมต่อกับฐานข้อมูล DB2 (Part 057-060) หรือ IMS DB (Part 065) การเขียนข้อมูลแต่ละครั้ง
มักอยู่ภายใน **Unit of Work** ที่ต้อง `COMMIT` เพื่อให้การเปลี่ยนแปลงมีผลถาวร **Commit Interval**
คือจำนวน record ที่ประมวลผลระหว่าง `COMMIT` แต่ละครั้ง — ค่านี้มีผลกระทบสำคัญต่อทั้งประสิทธิภาพและ
ความเสี่ยงของงาน batch

### ทำไม Commit Interval ถึงสำคัญ

- **COMMIT บ่อยเกินไป** (interval เล็ก): overhead ของการ sync ข้อมูลลงดิสก์สูง ทำให้งานช้าลง แต่ถ้า
  ล้มเหลวกลางทาง จะสูญเสียงานที่ทำไปน้อย (คล้ายกับ checkpoint ถี่ในขั้นตอนที่ 655)
- **COMMIT ไม่บ่อยพอ** (interval ใหญ่): ประสิทธิภาพสูงกว่า (I/O น้อยกว่า) แต่ถือ lock บนข้อมูลไว้นาน
  กว่า (กีดกันธุรกรรมอื่นที่ต้องการเข้าถึงข้อมูลเดียวกัน — ทบทวนแนวคิด lock จาก Part 064 ขั้นตอนที่
  648 เรื่อง `GHU`) และถ้าล้มเหลวกลางทาง จะต้องประมวลผลซ้ำจำนวนมากกว่า

### โค้ดตัวอย่างที่ทดสอบได้จริง: จำลองการนับ Commit Point

แม้ GnuCOBOL จะไม่มีฐานข้อมูลจริงให้ `COMMIT` แต่ตรรกะการนับจำนวน record และตัดสินใจว่าถึงจุด commit
หรือยังนั้น**เป็น COBOL ล้วน ๆ ที่ทดสอบได้จริง**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP659-COMMIT-INTERVAL.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANS-FILE ASSIGN TO "TRANS659.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANS-FILE.
       01  TRANS-RECORD.
           05  TR-ID                PIC 9(5).
           05  TR-AMOUNT            PIC 9(6)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-COMMIT-INTERVAL       PIC 9(4) VALUE 25.
       01  WS-RECORDS-SINCE-COMMIT  PIC 9(4) VALUE 0.
       01  WS-TOTAL-RECORDS         PIC 9(6) VALUE 0.
       01  WS-COMMIT-COUNT          PIC 9(4) VALUE 0.
       01  WS-I                     PIC 9(6).

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.
           OPEN INPUT TRANS-FILE.

      *> In a real DB2/CICS batch job, issuing COMMIT too rarely
      *> holds locks too long and risks losing a huge amount of
      *> work on a failure; committing too often wastes CPU on
      *> sync-point overhead. The commit interval (a count of
      *> records between COMMITs) is a tuning knob batch designers
      *> set deliberately - here we simulate it by counting records
      *> and printing a "COMMIT POINT" marker every N records,
      *> which is the same idea GnuCOBOL cannot demonstrate against
      *> a real database, but the counting logic is 100% real.
           PERFORM UNTIL END-OF-FILE
               READ TRANS-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-TOTAL-RECORDS
                       ADD 1 TO WS-RECORDS-SINCE-COMMIT
                       IF WS-RECORDS-SINCE-COMMIT = WS-COMMIT-INTERVAL
                           ADD 1 TO WS-COMMIT-COUNT
                           DISPLAY "  COMMIT POINT #" WS-COMMIT-COUNT
                               " after record " TR-ID
                           MOVE 0 TO WS-RECORDS-SINCE-COMMIT
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE TRANS-FILE.

           IF WS-RECORDS-SINCE-COMMIT > 0
               ADD 1 TO WS-COMMIT-COUNT
               DISPLAY "  FINAL COMMIT #" WS-COMMIT-COUNT
                   " (" WS-RECORDS-SINCE-COMMIT " records)"
           END-IF.

           DISPLAY "Total records   : " WS-TOTAL-RECORDS.
           DISPLAY "Commit interval : " WS-COMMIT-INTERVAL.
           DISPLAY "Total commits   : " WS-COMMIT-COUNT.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT TRANS-FILE.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 60
               MOVE WS-I TO TR-ID
               MOVE 100.00 TO TR-AMOUNT
               WRITE TRANS-RECORD
           END-PERFORM.
           CLOSE TRANS-FILE.
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
  COMMIT POINT #0001 after record 00025
  COMMIT POINT #0002 after record 00050
  FINAL COMMIT #0003 (0010 records)
Total records   : 000060
Commit interval : 0025
Total commits   : 0003
```

### อธิบายจุดสำคัญ

- ด้วยข้อมูล 60 record และ Commit Interval = 25 จะได้ commit ที่ record 25, 50, และ final commit
  สำหรับ 10 record สุดท้ายที่เหลือ (60 - 50 = 10) — ตรงกับผลลัพธ์ที่แสดง
- **`IF WS-RECORDS-SINCE-COMMIT > 0` หลัง loop จบ** คือรูปแบบเดียวกับที่เรียนใน Control Break
  (ขั้นตอนที่ 653) — ต้อง "flush" ยอดสุดท้ายที่ยังไม่ถึงรอบ commit ปกติเสมอ มิฉะนั้นข้อมูล 10 รายการ
  สุดท้ายจะไม่ถูก commit เลย

### ข้อควรระวัง

- ค่า Commit Interval ที่เหมาะสมขึ้นอยู่กับลักษณะงานเฉพาะแต่ละระบบ ไม่มีค่าสากลที่ใช้ได้ทุกกรณี
  ต้องพิจารณาจาก: ขนาดของ transaction log ที่ระบบรองรับได้, ระดับความเสี่ยงที่ยอมรับได้หากล้มเหลว
  กลางทาง, และปริมาณข้อมูลที่ธุรกรรมอื่นต้องรอ (ถ้ามี lock เกี่ยวข้อง)
- ในระบบจริงที่ใช้ DB2 Embedded SQL (Part 058) คำสั่ง `EXEC SQL COMMIT END-EXEC` จะถูกเรียกจริง ณ
  จุดที่ตรรกะแบบนี้กำหนด — ตัวอย่างนี้แสดงแค่ตรรกะการนับและตัดสินใจ "เมื่อไรควร commit" ซึ่งเป็นส่วน
  ที่ทดสอบได้ด้วย GnuCOBOL แม้ตัว `COMMIT` จริงจะทำไม่ได้ในสภาพแวดล้อมนี้ก็ตาม

### แบบฝึกหัดที่ 659.1

**โจทย์**: ถ้าเปลี่ยน `WS-COMMIT-INTERVAL` จาก 25 เป็น 100 (มากกว่าจำนวน record ทั้งหมดที่มี 60
record) ผลลัพธ์จะเป็นอย่างไร และมีความหมายเชิงธุรกิจอย่างไร

**เฉลย**: จะไม่มี `COMMIT POINT` ปกติเกิดขึ้นเลยระหว่าง loop (เพราะ `WS-RECORDS-SINCE-COMMIT` ไม่
มีทางถึง 100 เนื่องจากมีแค่ 60 record) แต่จะมี `FINAL COMMIT #0001 (0060 records)` เกิดขึ้นครั้งเดียว
หลัง loop จบ ความหมายเชิงธุรกิจคือ **ทั้งงาน 60 record จะถูกมองเป็น Unit of Work เดียวทั้งหมด** —
ถ้าเกิดความล้มเหลวที่ record 59 (เกือบจะจบแล้ว) ระบบจะต้องย้อนกลับ (rollback) การเปลี่ยนแปลงทั้งหมด
59 record ที่ทำไปแล้ว และต้องเริ่มประมวลผลใหม่ตั้งแต่ record แรกทั้งหมด ซึ่งมีความเสี่ยงสูงกว่าการตั้ง
Commit Interval ให้เล็กกว่าจำนวนข้อมูลทั้งหมดมาก

---

## ขั้นตอนที่ 660: ตัวอย่างบูรณาการ — Batch Pipeline สมบูรณ์แบบ (SORT + Control Break + Return Code)

### รวมทุกแพทเทิร์นเข้าด้วยกันในโปรแกรมเดียว

ปิดท้าย Part นี้ด้วยการจำลอง **Batch Pipeline** ที่สมจริง: (1) เรียงลำดับข้อมูลดิบที่มาแบบไม่เรียง
(ทบทวน SORT จาก Part 027) (2) รันรายงาน Control-Break บนข้อมูลที่เรียงแล้ว (ขั้นตอนที่ 653) (3) ตั้ง
ค่า RETURN-CODE ให้ Job Scheduler ตรวจสอบ (ขั้นตอนที่ 657) — เหมือนกับที่ JCL จริงจะเชื่อมสาม Job
Step เหล่านี้เข้าด้วยกันด้วย Dependency (ทบทวนแผนภาพจากขั้นตอนที่ 656)

### โค้ดตัวอย่างสมบูรณ์ (ทดสอบจริงแล้ว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP660-FULL-BATCH-PIPELINE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT RAW-FEED ASSIGN TO "RAWFEED660.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORTED-FEED ASSIGN TO "SORTED660.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK660.DAT".
           SELECT REPORT-FILE ASSIGN TO "REPORT660.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  RAW-FEED.
       01  RAW-RECORD.
           05  RAW-REGION           PIC X(5).
           05  RAW-AMOUNT           PIC 9(6)V99.

       FD  SORTED-FEED.
       01  SORTED-RECORD.
           05  SRT-REGION           PIC X(5).
           05  SRT-AMOUNT           PIC 9(6)V99.

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-REGION          PIC X(5).
           05  SORT-AMOUNT          PIC 9(6)V99.

       FD  REPORT-FILE.
       01  RPT-RECORD               PIC X(60).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-PREV-REGION           PIC X(5) VALUE SPACES.
       01  WS-FIRST-RECORD          PIC X VALUE "Y".
           88  IS-FIRST-RECORD      VALUE "Y".
       01  WS-REGION-TOTAL          PIC 9(8)V99 VALUE 0.
       01  WS-GRAND-TOTAL           PIC 9(9)V99 VALUE 0.
       01  WS-DISPLAY-AMT           PIC Z,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-RAW-FEED.

      *> STAGE 1 (JCL step 1 equivalent): SORT the unsorted feed
      *> that arrived overnight from a branch office - reuses the
      *> exact SORT statement pattern from Part 027.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-REGION
               USING RAW-FEED
               GIVING SORTED-FEED.
           DISPLAY "STAGE 1 complete: feed sorted by REGION.".

      *> STAGE 2 (JCL step 2 equivalent): run the control-break
      *> report against the now-sorted feed, exactly like a real
      *> batch pipeline chains several job steps together via JCL,
      *> each one handing its output to the next (Part 052/053).
           PERFORM RUN-CONTROL-BREAK-REPORT.
           DISPLAY "STAGE 2 complete: report written.".

      *> STAGE 3: show the report and set a RETURN-CODE, exactly
      *> like a job scheduler (CA-7/Control-M) would check before
      *> deciding whether to trigger the next dependent job.
           PERFORM SHOW-REPORT.
           MOVE 0 TO RETURN-CODE.
           DISPLAY "Pipeline finished, RETURN-CODE=" RETURN-CODE.
           STOP RUN.

       BUILD-RAW-FEED.
      *> Deliberately OUT OF ORDER, like a real incoming feed.
           OPEN OUTPUT RAW-FEED.
           MOVE "SOUTH" TO RAW-REGION. MOVE 750.00  TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           MOVE "NORTH" TO RAW-REGION. MOVE 1000.00 TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           MOVE "WEST " TO RAW-REGION. MOVE 400.00  TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           MOVE "NORTH" TO RAW-REGION. MOVE 500.00  TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           MOVE "SOUTH" TO RAW-REGION. MOVE 300.00  TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           CLOSE RAW-FEED.

       RUN-CONTROL-BREAK-REPORT.
           MOVE "N" TO WS-EOF-FLAG.
           MOVE "Y" TO WS-FIRST-RECORD.
           MOVE 0 TO WS-REGION-TOTAL.
           MOVE 0 TO WS-GRAND-TOTAL.
           OPEN INPUT SORTED-FEED.
           OPEN OUTPUT REPORT-FILE.
           PERFORM READ-SORTED.
           PERFORM UNTIL END-OF-FILE
               IF NOT IS-FIRST-RECORD
                   IF SRT-REGION NOT = WS-PREV-REGION
                       PERFORM WRITE-REGION-BREAK
                   END-IF
               END-IF
               ADD SRT-AMOUNT TO WS-REGION-TOTAL
               MOVE SRT-REGION TO WS-PREV-REGION
               MOVE "N" TO WS-FIRST-RECORD
               PERFORM READ-SORTED
           END-PERFORM.
           PERFORM WRITE-REGION-BREAK.
           MOVE SPACES TO RPT-RECORD.
           MOVE WS-GRAND-TOTAL TO WS-DISPLAY-AMT.
           STRING "GRAND TOTAL: " WS-DISPLAY-AMT
               DELIMITED BY SIZE INTO RPT-RECORD.
           WRITE RPT-RECORD.
           CLOSE SORTED-FEED.
           CLOSE REPORT-FILE.

       READ-SORTED.
           READ SORTED-FEED
               AT END
                   SET END-OF-FILE TO TRUE
           END-READ.

       WRITE-REGION-BREAK.
           MOVE SPACES TO RPT-RECORD.
           MOVE WS-REGION-TOTAL TO WS-DISPLAY-AMT.
           STRING "REGION " WS-PREV-REGION " TOTAL: " WS-DISPLAY-AMT
               DELIMITED BY SIZE INTO RPT-RECORD.
           WRITE RPT-RECORD.
           ADD WS-REGION-TOTAL TO WS-GRAND-TOTAL.
           MOVE 0 TO WS-REGION-TOTAL.

       SHOW-REPORT.
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT REPORT-FILE.
           DISPLAY "---- REPORT660.DAT content ----".
           PERFORM UNTIL END-OF-FILE
               READ REPORT-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " RPT-RECORD
               END-READ
           END-PERFORM.
           CLOSE REPORT-FILE.
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
STAGE 1 complete: feed sorted by REGION.
STAGE 2 complete: report written.
---- REPORT660.DAT content ----
  REGION NORTH TOTAL:     1,500.00
  REGION SOUTH TOTAL:     1,050.00
  REGION WEST  TOTAL:       400.00
  GRAND TOTAL:     2,950.00
Pipeline finished, RETURN-CODE=+000000000
```

### อธิบายจุดสำคัญ

- ข้อมูลดิบ (`RAWFEED660.DAT`) ถูกเขียนแบบไม่เรียงลำดับโดยตั้งใจ (SOUTH, NORTH, WEST, NORTH, SOUTH)
  จำลองไฟล์ที่มาจากสาขาต่าง ๆ ในโลกจริงตามที่ Part 027 อธิบายไว้
- `SORT` (Stage 1) จัดเรียงให้เป็น NORTH, NORTH, SOUTH, SOUTH, WEST ก่อนที่จะเข้าสู่ Control Break
  (Stage 2) ซึ่งต้องการข้อมูลที่เรียงลำดับแล้วเสมอ (ทบทวนความจำเป็นนี้จากขั้นตอนที่ 653)
- ผลรวม `2,950.00` ถูกต้อง (1000+500+750+300+400 = 2950) และ `RETURN-CODE = 0` แสดงว่า pipeline
  ทำงานสำเร็จสมบูรณ์ ซึ่งในระบบจริง Job Scheduler จะใช้ค่านี้ (ตามหลักการขั้นตอนที่ 656-657) เพื่อ
  ตัดสินใจว่าจะ trigger job ถัดไปในสายงานหรือไม่
- โครงสร้างโปรแกรมนี้จำลอง **3 Job Steps ของ JCL หนึ่ง Job** ไว้ในโปรแกรม COBOL เดียว เพื่อความ
  สะดวกในการสาธิต — ในระบบจริงบน Mainframe แต่ละ Stage มักเป็นโปรแกรมแยกกันคนละ Step ใน JCL
  (ทบทวนแนวคิด Job Step จาก Part 052)

### ข้อควรระวัง

- การรวมหลาย Stage ไว้ในโปรแกรมเดียวแบบนี้เหมาะกับการสาธิตและงานขนาดเล็ก แต่ระบบจริงระดับองค์กร
  มักแยกแต่ละ Stage เป็นโปรแกรม/Job Step อิสระ เพื่อให้สามารถ restart เฉพาะ Stage ที่ล้มเหลวได้โดย
  ไม่ต้องรันใหม่ทั้งหมด (เชื่อมโยงกับแนวคิด Checkpoint/Restart จากขั้นตอนที่ 655)
- ควรตรวจสอบ Return Code ของแต่ละ Stage แยกกันในระบบจริง ไม่ใช่ตั้งค่า Return Code เดียวรวมท้ายสุด
  เหมือนตัวอย่างนี้ เพื่อให้รู้ได้ชัดเจนว่า Stage ไหนที่ล้มเหลว (ถ้ามี)

### แบบฝึกหัดที่ 660.1

**โจทย์**: จงอธิบายว่าทำไมการแยก 3 Stage นี้เป็น 3 Job Step แยกกันใน JCL จริง (แทนที่จะรวมในโปรแกรม
เดียวแบบตัวอย่างนี้) จึงมีข้อดีด้าน Restart/Checkpoint (ขั้นตอนที่ 655) มากกว่า

**เฉลยแนวทาง**: ถ้า Stage 2 (Control-Break Report) ล้มเหลวกลางทาง (เช่น ดิสก์เต็มระหว่างเขียนรายงาน)
ในกรณีที่แยกเป็น 3 Job Step อิสระ ผู้ดูแลระบบสามารถแก้ปัญหา (เช่น เคลียร์พื้นที่ดิสก์) แล้ว restart
เฉพาะ **Step ที่ 2** ได้ทันที โดยไม่ต้องรัน **Step ที่ 1** (SORT) ซ้ำอีกครั้ง เพราะผลลัพธ์ของ Step 1
(`SORTED660.DAT`) ยังคงอยู่ครบถ้วนสมบูรณ์แล้ว แต่ถ้ารวมทั้งหมดไว้ในโปรแกรมเดียวแบบตัวอย่างนี้ การ
ล้มเหลวที่ Stage 2 จะบังคับให้ต้องรันทั้งโปรแกรมใหม่ตั้งแต่ Stage 1 เสมอ แม้ว่า Stage 1 จะไม่มีปัญหา
อะไรเลยก็ตาม ทำให้เสียเวลาโดยไม่จำเป็นสำหรับงานที่มีปริมาณข้อมูลมากและใช้เวลานาน

---

## สรุปท้ายบท

Part นี้แตกต่างจาก Part 063-065 ที่เพิ่งผ่านมาอย่างชัดเจน เพราะเนื้อหาเกือบทั้งหมด**คอมไพล์และรันได้
จริงด้วย GnuCOBOL** โดยไม่ต้องพึ่งพา CICS, DB2, หรือ IMS แต่อย่างใด สิ่งที่เราได้เรียนรู้:

- **Sequential Update Pattern**: การปรับปรุงไฟล์ Master ด้วย Transaction (Add/Change/Delete) ต่อยอด
  จาก Master-Detail Matching (Part 026) — พร้อมการค้นพบบั๊กจริงเรื่อง `MOVE SPACES` ก่อน `STRING`
- **Control-Break Reporting** ทั้งแบบ Single-Level และ Multi-Level เขียนด้วยตรรกะ `PROCEDURE
  DIVISION` ล้วน ๆ โดยไม่พึ่งพา Report Writer
- **Checkpoint/Restart Pattern**: การบันทึกความคืบหน้าเพื่อกู้คืนงานที่ล้มเหลวโดยไม่ต้องเริ่มใหม่
  ทั้งหมด พิสูจน์ด้วยการจำลองการ crash และ restart จริง
- แนวคิด **Job Scheduling** (CA-7, Control-M) เชิงทฤษฎี และความแตกต่างจาก `cron`
- **RETURN-CODE** และความเชื่อมโยงกับ JCL Condition Code — พิสูจน์แล้วว่าค่าที่ตั้งใน COBOL กลายเป็น
  shell exit status จริง
- **Idempotent Batch Design**: การออกแบบให้รันซ้ำได้อย่างปลอดภัย ผ่านการบันทึกประวัติการประมวลผล
- **Commit Interval**: สมดุลระหว่างประสิทธิภาพและความเสี่ยงในการ commit ข้อมูล
- ตัวอย่างบูรณาการ **Batch Pipeline** ที่รวม SORT + Control Break + Return Code เข้าด้วยกัน

ความรู้เหล่านี้เป็นแพทเทิร์นการออกแบบที่ใช้งานจริงในองค์กรทุกวันนี้ ไม่ว่าจะเป็นระบบ Mainframe หรือ
ระบบสมัยใหม่ก็ตาม ใน Part 067 เราจะกลับไปเจาะลึกเรื่อง **DFSORT** ยูทิลิตี้เรียงลำดับข้อมูลระดับ
องค์กรของ IBM ที่ใช้กันอย่างแพร่หลายบน Mainframe พร้อมทั้งแสดงให้เห็นว่าตรรกะเดียวกันนี้สามารถทำได้
ด้วย COBOL SORT statement ที่เราคุ้นเคยจาก Part 027 อย่างไร

**[ไปยัง Part 067: DFSORT และ Sort Utilities ขั้นสูง →](part-067-dfsort-advanced.md)**
