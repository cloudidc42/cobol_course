# Part 067: DFSORT และ Sort Utilities ขั้นสูง (ขั้นตอนที่ 661–670)

## คำนำของ Part นี้

Part 027 สอนให้เราเรียงลำดับและรวมข้อมูลด้วยคำสั่ง `SORT`/`MERGE` ของภาษา COBOL เอง ซึ่งเป็นเครื่องมือ
ที่ทรงพลังและทดสอบได้จริงบน GnuCOBOL Part 066 ก็เพิ่งแสดงให้เห็นว่าเราสามารถนำ `SORT` มาผสมกับ
Control-Break Reporting เพื่อสร้าง Batch Pipeline ที่สมบูรณ์ได้ แต่ในโลก Mainframe จริง งานเรียง
ลำดับ/กรอง/สรุปข้อมูลปริมาณมหาศาล (หลักร้อยล้านถึงพันล้านระเบียนต่อคืน) มักไม่ได้ทำผ่านโปรแกรม COBOL
ที่มีคำสั่ง `SORT` ฝังอยู่ในตัว แต่ทำผ่าน**ยูทิลิตี้ระดับ JCL** ที่ทรงพลังและปรับแต่งได้ละเอียดกว่ามาก
นั่นคือ **DFSORT** (Data Facility SORT) ของ IBM

Part นี้จะพาคุณเรียนรู้ไวยากรณ์ควบคุมของ DFSORT (`SORT FIELDS=`, `INCLUDE`/`OMIT COND=`, `OUTFIL`,
`ICETOOL`) ในฐานะ**ไวยากรณ์อ้างอิง**ที่ต้องรู้จักเพื่อทำงานกับ JCL จริงบน Mainframe จากนั้นในทุก
หัวข้อจะแสดง**ตัวอย่าง COBOL SORT ที่คอมไพล์และรันได้จริง**ที่ทำหน้าที่แบบเดียวกัน เพื่อให้เห็นภาพว่า
สิ่งที่ DFSORT ทำในระดับ JCL/Utility นั้นคือแนวคิดเดียวกันกับสิ่งที่ COBOL `SORT` ทำในระดับโปรแกรม
เพียงแค่คนละเครื่องมือ คนละระดับของ stack เท่านั้น

> **หลักการของ Part นี้**: ไวยากรณ์ DFSORT (control statements) และ JCL ทุกตัวอย่างเป็น **⚠️
> REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้** เพราะ DFSORT เป็นยูทิลิตี้เชิงพาณิชย์ของ IBM
> ที่ทำงานบน z/OS เท่านั้น ไม่มีอยู่ใน GnuCOBOL หรือ Linux ทั่วไป ในทางตรงข้าม **ทุกโปรแกรม COBOL
> SORT ในเอกสารนี้คอมไพล์และรันจริงแล้วด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`)** และแสดง
> ผลลัพธ์จริงจากการรัน — ทั้งสองส่วนจะมีป้ายกำกับชัดเจนตลอดทั้ง Part นี้

---

## ขั้นตอนที่ 661: DFSORT คืออะไร — ยูทิลิตี้ระดับ JCL เทียบกับ COBOL SORT Statement

### DFSORT ทำงานที่ "ระดับ JCL" ไม่ใช่ "ระดับโปรแกรม"

ความแตกต่างพื้นฐานที่สุดระหว่าง COBOL `SORT` (Part 027) กับ DFSORT คือ **DFSORT ไม่ใช่ส่วนหนึ่งของ
โปรแกรม COBOL เลย** — มันคือ**โปรแกรมยูทิลิตี้สำเร็จรูป**ที่ IBM เตรียมไว้ให้ เรียกใช้งานโดยตรงจาก
JCL ผ่าน `EXEC PGM=SORT` (ทบทวนแนวคิด Job Step จาก Part 052) โดยไม่ต้องเขียนโปรแกรม COBOL แม้แต่
บรรทัดเดียว เพียงแค่เขียน "control statement" (ภาษาควบคุมเฉพาะของ DFSORT) ใน DD statement ชื่อ
`SYSIN`

### ตารางเปรียบเทียบภาพรวม

| แง่มุม | COBOL SORT (Part 027) | DFSORT |
|---|---|---|
| ระดับการทำงาน | ส่วนหนึ่งของโปรแกรม COBOL | ยูทิลิตี้แยกต่างหาก เรียกจาก JCL โดยตรง |
| ต้องเขียนโปรแกรมไหม | ต้อง (แม้จะสั้นก็ตาม) | ไม่ต้อง — เขียนแค่ control statement ใน JCL |
| ความเร็ว/ประสิทธิภาพ | ดี สำหรับข้อมูลขนาดปานกลาง | ปรับแต่งพิเศษสำหรับข้อมูลขนาดมหาศาล (หลักร้อยล้าน-พันล้านระเบียน) |
| ความสามารถกรอง/สรุปข้อมูล | ทำได้ผ่าน `INPUT`/`OUTPUT PROCEDURE` (ต้องเขียนโค้ดเอง) | มีในตัว (`INCLUDE`/`OMIT`, `SUM`, `OUTFIL`) ไม่ต้องเขียนโค้ด |
| นิยมใช้เมื่อ | ต้องการตรรกะซับซ้อนปนกับ business logic อื่นในโปรแกรมเดียว | ต้องการเรียง/กรอง/สรุปข้อมูลล้วน ๆ โดยไม่มี business logic ซับซ้อน |
| ใช้งานได้บน GnuCOBOL ไหม | ได้ (ทดสอบแล้วตลอด Part 027, 066) | **ไม่ได้** — เป็นยูทิลิตี้ z/OS โดยเฉพาะ |

### ทำไมองค์กรถึงนิยมใช้ DFSORT แทนที่จะเขียน COBOL SORT

เหตุผลหลักคือ **ประสิทธิภาพและความสะดวก** — DFSORT ถูกออกแบบและปรับแต่ง (optimize) โดย IBM มาเป็น
เวลาหลายสิบปีโดยเฉพาะสำหรับงานเรียงลำดับข้อมูลปริมาณมหาศาลบน Mainframe มันใช้เทคนิคขั้นสูงด้าน
I/O และหน่วยความจำที่การเขียนโปรแกรม COBOL ทั่วไปเข้าไม่ถึง (จะกล่าวถึงในขั้นตอนที่ 668) นอกจากนี้
งาน "เรียง กรอง สรุปยอด" จำนวนมากในระบบ batch **ไม่มี business logic ที่ซับซ้อนเลย** (แค่ต้องการ
เรียงไฟล์ หรือกรองข้อมูลบางเงื่อนไขทิ้ง) การเขียน control statement 2-3 บรรทัดใน JCL จึงเร็วและง่าย
กว่าการเขียน-compile-ทดสอบโปรแกรม COBOL ทั้งโปรแกรมมาก

### ข้อควรระวัง

- อย่าเข้าใจผิดว่า DFSORT "ดีกว่า" COBOL SORT เสมอไป — เมื่องานต้องการตรรกะทางธุรกิจที่ซับซ้อนปนอยู่
  ด้วย (เช่น คำนวณดอกเบี้ยระหว่างการประมวลผลแต่ละ record) การเขียนโปรแกรม COBOL ที่มี `INPUT
  PROCEDURE`/`OUTPUT PROCEDURE` (Part 027 ขั้นตอนที่ 263-265) มักเหมาะสมกว่า เพราะ DFSORT ไม่ได้
  ออกแบบมาสำหรับตรรกะทางธุรกิจที่ซับซ้อน
- DFSORT เป็นผลิตภัณฑ์ของ IBM แต่ยังมียูทิลิตี้เรียงลำดับคู่แข่งที่ทำงานคล้ายกันจากผู้ผลิตอื่น เช่น
  **SyncSort** (ปัจจุบันคือ Precisely) ซึ่งมีไวยากรณ์ control statement ที่**เกือบเหมือนกันทุกประการ**
  กับ DFSORT (เพื่อให้ลูกค้าย้ายมาใช้ได้ง่าย) เนื้อหาใน Part นี้อ้างอิงไวยากรณ์ DFSORT เป็นหลักเพราะ
  เป็นมาตรฐานที่ใช้แพร่หลายที่สุด

### แบบฝึกหัดที่ 661.1

**โจทย์**: จงอธิบายว่าทำไมงาน "เรียงลำดับไฟล์ธุรกรรม 50 ล้านรายการตาม Account Number" ล้วน ๆ (ไม่มี
การคำนวณอะไรเพิ่มเติม) จึงเหมาะกับ DFSORT มากกว่าการเขียนโปรแกรม COBOL ที่มีคำสั่ง SORT

**เฉลยแนวทาง**: เพราะงานนี้ไม่มี business logic ที่ซับซ้อนเลย เป็นแค่การเรียงลำดับข้อมูลตาม key
เพียงอย่างเดียว การเขียน control statement DFSORT เพียง 1-2 บรรทัด (`SORT FIELDS=(1,10,CH,A)`) ใน
JCL ทำงานได้ทันทีโดยไม่ต้องเขียน-คอมไพล์-ทดสอบโปรแกรม COBOL เลย นอกจากนี้ DFSORT ยังถูกปรับแต่งมา
เฉพาะสำหรับงานเรียงลำดับข้อมูลปริมาณมหาศาลแบบนี้โดยเฉพาะ (จะกล่าวถึงเทคนิคด้านประสิทธิภาพในขั้นตอนที่
668) ทำให้มีแนวโน้มจะเร็วกว่าการเขียนโปรแกรม COBOL เองสำหรับงานขนาด 50 ล้านระเบียนอย่างมีนัยสำคัญ

---

## ขั้นตอนที่ 662: SORT FIELDS= — ไวยากรณ์ควบคุมการเรียงลำดับพื้นฐานของ DFSORT

### รูปแบบพื้นฐานของ Control Statement

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้ (ต้องมี DFSORT บน z/OS จริง)**

```jcl
//SORTJOB  JOB (ACCTNO),'SORT CUSTOMER FILE',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=SORT
//SYSOUT   DD SYSOUT=*
//SORTIN   DD DSN=PROD.CUSTOMER.UNSORTED,DISP=SHR
//SORTOUT  DD DSN=PROD.CUSTOMER.SORTED,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(50,10)),
//            DCB=(LRECL=80,RECFM=FB)
//SYSIN    DD *
    SORT FIELDS=(1,5,CH,A,20,9,PD,D)
/*
```

### อธิบายทีละส่วน

- **`SORTIN`/`SORTOUT`**: DD statement มาตรฐาน (ทบทวนจาก Part 052) ระบุไฟล์ต้นทาง/ปลายทาง — เทียบ
  เคียงได้กับ `SELECT ... ASSIGN TO` ของ COBOL แต่ DFSORT ใช้ชื่อ DD ที่ตายตัวเสมอ (`SORTIN`,
  `SORTOUT`) ไม่สามารถตั้งชื่อเองได้เหมือน COBOL
- **`SYSIN`**: DD statement ที่บรรจุ control statement ของ DFSORT (เทียบเคียงได้กับซอร์สโค้ดส่วน
  `SORT` ในโปรแกรม COBOL — แต่แยกออกมาเป็น "ข้อมูล" ใน JCL แทนที่จะเป็น "โค้ด" ในโปรแกรม)
- **`SORT FIELDS=(start,length,format,order, ...)`**: รูปแบบเป็นชุดตัวเลข 4 ค่าต่อ 1 Key ซ้ำกันได้
  หลาย Key
  - `start`: ตำแหน่งไบต์เริ่มต้นของ field นี้ในแต่ละ record (**นับจาก 1 เสมอ** ไม่ใช่นับจาก 0)
  - `length`: ความยาวของ field เป็นไบต์
  - `format`: ชนิดข้อมูล — `CH` (Character/ตัวอักษร), `ZD` (Zoned Decimal), `PD` (Packed Decimal),
    `BI` (Binary) ฯลฯ
  - `order`: `A` (Ascending) หรือ `D` (Descending)

ตัวอย่างในโค้ดข้างต้น `SORT FIELDS=(1,5,CH,A,20,9,PD,D)` หมายถึง: **เรียงตามไบต์ที่ 1-5 (ตัวอักษร)
จากน้อยไปมากก่อน แล้วถ้าเท่ากันให้เรียงตามไบต์ที่ 20-28 (Packed Decimal ความยาว 9 ไบต์) จากมากไปน้อย**
— นี่คือ Multi-Key Sort แบบเดียวกับที่เรียนใน Part 027 ขั้นตอนที่ 262 (`ON ASCENDING KEY ... ON
DESCENDING KEY ...`) เพียงแต่ DFSORT ระบุตำแหน่งด้วยตัวเลขไบต์ดิบ แทนที่จะอ้างอิงชื่อ field ใน
`DATA DIVISION` แบบ COBOL

### เปรียบเทียบโดยตรงกับ COBOL SORT ที่ทดสอบได้ (ทบทวนจาก Part 027 ขั้นตอนที่ 262)

```cobol
      *> COBOL SORT equivalent (ALREADY tested in Part 027, step
      *> 262) - the SAME multi-key sort, but referencing fields
      *> by NAME (declared in the SD entry) instead of raw byte
      *> positions. This is genuinely runnable with GnuCOBOL.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-DEPT
               ON DESCENDING KEY SORT-SALARY
               USING UNSORTED-FILE
               GIVING SORTED-FILE.
```

### ข้อควรระวัง

- **ตำแหน่งไบต์ใน DFSORT นับจาก 1 เสมอ** (ไม่ใช่ 0) และต้องคำนวณเองจากโครงสร้าง record ว่า field
  ที่ต้องการอยู่ที่ไบต์ไหนถึงไบต์ไหน — ความผิดพลาดเรื่องตำแหน่งไบต์เป็นข้อผิดพลาดที่พบบ่อยที่สุดของ
  ผู้เริ่มต้นเขียน DFSORT control statement เพราะไม่มี compiler ตรวจสอบให้เหมือน COBOL ที่อ้างอิง
  ด้วยชื่อ field
- ชนิดข้อมูล `PD` (Packed Decimal) เป็นรูปแบบการเก็บตัวเลขแบบบีบอัด (คล้ายกับ `COMP-3` ใน COBOL ที่
  จะกล่าวถึงในบริบทของ Performance Tuning ใน Part 068) การระบุ format ผิดพลาด (เช่น ระบุ `CH` ทั้งที่
  ข้อมูลจริงเป็น `PD`) จะทำให้ผลการเรียงลำดับผิดพลาดอย่างสิ้นเชิงโดยไม่มี error แจ้งเตือนใด ๆ

### แบบฝึกหัดที่ 662.1

**โจทย์**: ถ้า record มีโครงสร้างดังนี้: ไบต์ 1-4 = รหัสสาขา (ตัวอักษร), ไบต์ 5-14 = ยอดขาย (Packed
Decimal ความยาว 10 ไบต์) จงเขียน `SORT FIELDS=` เพื่อเรียงตามรหัสสาขาก่อน (น้อยไปมาก) แล้วเรียงตาม
ยอดขายภายในสาขาเดียวกัน (มากไปน้อย)

**เฉลย**:

```jcl
    SORT FIELDS=(1,4,CH,A,5,10,PD,D)
```

รหัสสาขาอยู่ที่ไบต์ 1 ยาว 4 ไบต์ เรียงจากน้อยไปมาก (`A`) และยอดขายอยู่ที่ไบต์ 5 (ต่อจากรหัสสาขาที่จบ
ที่ไบต์ 4 พอดี) ยาว 10 ไบต์ เป็น Packed Decimal เรียงจากมากไปน้อย (`D`)

---

## ขั้นตอนที่ 663: INCLUDE/OMIT COND= — กรองข้อมูลก่อนเรียงลำดับ

### แนวคิด

**`INCLUDE COND=`** และ **`OMIT COND=`** คือกลไกของ DFSORT ที่ใช้กรอง record **ก่อน**ที่จะเข้าสู่
กระบวนการเรียงลำดับ — `INCLUDE` หมายถึง "เก็บเฉพาะ record ที่ตรงเงื่อนไข" ส่วน `OMIT` หมายถึง "ทิ้ง
record ที่ตรงเงื่อนไข" (เก็บที่เหลือทั้งหมด) ทั้งสองใช้ไวยากรณ์เงื่อนไขแบบเดียวกัน

### ไวยากรณ์อ้างอิง

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้**

```jcl
//SYSIN    DD *
    SORT FIELDS=(1,5,CH,A)
    OMIT COND=(6,1,CH,EQ,C'X')
/*
```

ตัวอย่างนี้หมายถึง: **เรียงตามไบต์ 1-5 จากน้อยไปมาก แต่ทิ้ง record ที่ไบต์ที่ 6 (ยาว 1 ไบต์ ชนิด
ตัวอักษร) มีค่าเท่ากับ `'X'` ก่อน** (สมมติว่าไบต์ 6 คือรหัสสถานะ และ `'X'` หมายถึง "ยกเลิกแล้ว")

### ตัวอย่างเปรียบเทียบ COBOL SORT (ทดสอบจริงแล้ว)

DFSORT ไม่มีในสภาพแวดล้อมนี้ แต่ผลลัพธ์แบบเดียวกันทำได้จริงด้วย COBOL `SORT` + `INPUT PROCEDURE`
(ทบทวนจาก Part 027 ขั้นตอนที่ 263) — โปรแกรมต่อไปนี้กรองบัญชีที่มีสถานะ `X` (ยกเลิกแล้ว) ทิ้งก่อน
เรียงลำดับ:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP663-INCLUDE-OMIT-EQUIV.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED663.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORTED-FILE ASSIGN TO "SORTED663.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK663.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ID               PIC X(5).
           05  UNS-STATUS           PIC X(1).
           05  UNS-AMOUNT           PIC 9(7)V99.

       FD  SORTED-FILE.
       01  SRT-RECORD.
           05  SRT-ID               PIC X(5).
           05  SRT-STATUS           PIC X(1).
           05  SRT-AMOUNT           PIC 9(7)V99.

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ID              PIC X(5).
           05  SORT-STATUS          PIC X(1).
           05  SORT-AMOUNT          PIC 9(7)V99.

       WORKING-STORAGE SECTION.
       01  WS-U-EOF                 PIC X VALUE "N".
           88  UNSORTED-EOF         VALUE "Y".
       01  WS-S-EOF                 PIC X VALUE "N".
           88  END-OF-SORT          VALUE "Y".
       01  WS-OMITTED-COUNT         PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.

      *> DFSORT reference syntax for the SAME filtering job:
      *>
      *>   //SYSIN DD *
      *>     SORT FIELDS=(1,5,CH,A)
      *>     OMIT COND=(6,1,CH,EQ,C'X')
      *>   /*
      *>
      *> This drops every record whose status byte (column 6) is
      *> 'X' (cancelled) BEFORE sorting by the 5-byte ID key.
      *> GnuCOBOL has no DFSORT, so we reproduce the same result
      *> with the COBOL SORT statement's INPUT PROCEDURE, using
      *> RELEASE to decide record-by-record what enters the sort -
      *> exactly the pattern from Part 027, step 263.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ID
               INPUT PROCEDURE IS FILTER-CANCELLED
               GIVING SORTED-FILE.

           DISPLAY "Omitted (status = X): " WS-OMITTED-COUNT.
           PERFORM SHOW-RESULT.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "C0003" TO UNS-ID. MOVE "A" TO UNS-STATUS.
           MOVE 15000.00 TO UNS-AMOUNT. WRITE UNS-RECORD.
           MOVE "C0001" TO UNS-ID. MOVE "X" TO UNS-STATUS.
           MOVE 5000.00  TO UNS-AMOUNT. WRITE UNS-RECORD.
           MOVE "C0002" TO UNS-ID. MOVE "A" TO UNS-STATUS.
           MOVE 27500.00 TO UNS-AMOUNT. WRITE UNS-RECORD.
           MOVE "C0004" TO UNS-ID. MOVE "X" TO UNS-STATUS.
           MOVE 8000.00  TO UNS-AMOUNT. WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

       FILTER-CANCELLED.
           OPEN INPUT UNSORTED-FILE.
           PERFORM UNTIL UNSORTED-EOF
               READ UNSORTED-FILE
                   AT END
                       SET UNSORTED-EOF TO TRUE
                   NOT AT END
                       IF UNS-STATUS = "X"
                           ADD 1 TO WS-OMITTED-COUNT
                       ELSE
                           MOVE UNS-ID TO SORT-ID
                           MOVE UNS-STATUS TO SORT-STATUS
                           MOVE UNS-AMOUNT TO SORT-AMOUNT
                           RELEASE SORT-RECORD
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE UNSORTED-FILE.

       SHOW-RESULT.
           MOVE "N" TO WS-S-EOF.
           OPEN INPUT SORTED-FILE.
           DISPLAY "Result (active records, sorted by ID):".
           PERFORM UNTIL END-OF-SORT
               READ SORTED-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       DISPLAY "  " SRT-ID " " SRT-STATUS " "
                           SRT-AMOUNT
               END-READ
           END-PERFORM.
           CLOSE SORTED-FILE.
```

คอมไพล์และรัน:

```bash
cobc -x -o step663 step663.cob
./step663
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
Omitted (status = X): 002
Result (active records, sorted by ID):
  C0002 A 0027500.00
  C0003 A 0015000.00
```

### อธิบายจุดสำคัญ

- Record `C0001` และ `C0004` (สถานะ `X`) ถูกกรองทิ้งใน `FILTER-CANCELLED` **ก่อน** `RELEASE` เข้าสู่
  กระบวนการเรียงลำดับ — เทียบเท่ากับ `OMIT COND=` ของ DFSORT ทุกประการในเชิงผลลัพธ์
- ผลลัพธ์สุดท้ายมีแค่ 2 record ที่เหลือ (`C0002`, `C0003`) เรียงลำดับตาม ID ถูกต้อง — พิสูจน์ว่าการ
  กรองเกิดขึ้นก่อนการเรียงลำดับจริง (สอดคล้องกับที่ DFSORT ประมวลผล `OMIT`/`INCLUDE` ก่อน `SORT
  FIELDS=` เสมอ)

### ข้อควรระวัง

- ตัวดำเนินการเปรียบเทียบของ DFSORT (`EQ`, `NE`, `GT`, `LT`, `GE`, `LE`) คือคำย่อของภาษาอังกฤษ
  ต่างจาก COBOL ที่ใช้ทั้งคำเต็ม (`EQUAL TO`) และสัญลักษณ์ (`=`) ปนกันได้ (ทบทวนจาก Part 010) ผู้ที่
  คุ้นเคยกับ COBOL อาจสับสนกับรูปแบบย่อนี้ในตอนแรก
- `INCLUDE` และ `OMIT` **ใช้พร้อมกันไม่ได้ในคำสั่งเดียวกัน** — ต้องเลือกอย่างใดอย่างหนึ่งเท่านั้น
  (เพราะทั้งสองเป็นตรรกะตรงข้ามกัน ใช้ร่วมกันจะขัดแย้งกันเอง)

### แบบฝึกหัดที่ 663.1

**โจทย์**: จงเขียน DFSORT control statement (ไวยากรณ์อ้างอิง) ที่เก็บเฉพาะ record ที่ยอดเงิน (ไบต์
10-18, Packed Decimal) มากกว่า 100,000 เท่านั้น โดยใช้ `INCLUDE` แทน `OMIT`

**เฉลย**:

```jcl
    SORT FIELDS=(1,5,CH,A)
    INCLUDE COND=(10,9,PD,GT,+100000)
```

---

## ขั้นตอนที่ 664: OUTFIL — แยกผลลัพธ์ออกหลายไฟล์ในการเรียงลำดับครั้งเดียว

### แนวคิด

**`OUTFIL`** คือความสามารถของ DFSORT ที่ให้เรา**ส่งผลลัพธ์ที่เรียงลำดับแล้วออกไปหลายไฟล์พร้อมกัน**
โดยแต่ละไฟล์มีเงื่อนไขการกรองของตัวเอง ทำได้ในการรัน DFSORT เพียงครั้งเดียว (pass เดียว) แทนที่จะต้อง
รัน SORT หลายรอบแยกกันสำหรับแต่ละเงื่อนไข

### ไวยากรณ์อ้างอิง

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้**

```jcl
//SORTOF1  DD DSN=PROD.HIGHVALUE.OUT,DISP=(NEW,CATLG,DELETE)
//SORTOF2  DD DSN=PROD.LOWVALUE.OUT,DISP=(NEW,CATLG,DELETE)
//SYSIN    DD *
    SORT FIELDS=(1,5,CH,A)
    OUTFIL FNAMES=SORTOF1,INCLUDE=(6,9,PD,GE,+100000)
    OUTFIL FNAMES=SORTOF2,INCLUDE=(6,9,PD,LT,+100000)
/*
```

ตัวอย่างนี้เรียงข้อมูลตามไบต์ 1-5 เพียงครั้งเดียว แล้วแยกผลลัพธ์ที่เรียงแล้วออกเป็นสองไฟล์: ไฟล์แรก
(`SORTOF1`) เก็บ record ที่ยอดเงิน (ไบต์ 6-14) มากกว่าหรือเท่ากับ 100,000 และไฟล์ที่สอง (`SORTOF2`)
เก็บที่เหลือ

### ตัวอย่างเปรียบเทียบ COBOL SORT (ทดสอบจริงแล้ว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP664-OUTFIL-EQUIV.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED664.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT HIGH-VALUE-FILE ASSIGN TO "HIGHVAL664.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT LOW-VALUE-FILE ASSIGN TO "LOWVAL664.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK664.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ID               PIC X(5).
           05  UNS-AMOUNT           PIC 9(7)V99.

       FD  HIGH-VALUE-FILE.
       01  HIGH-RECORD.
           05  HIGH-ID              PIC X(5).
           05  HIGH-AMOUNT          PIC 9(7)V99.

       FD  LOW-VALUE-FILE.
       01  LOW-RECORD.
           05  LOW-ID               PIC X(5).
           05  LOW-AMOUNT           PIC 9(7)V99.

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ID              PIC X(5).
           05  SORT-AMOUNT          PIC 9(7)V99.

       WORKING-STORAGE SECTION.
       01  WS-S-EOF                 PIC X VALUE "N".
           88  END-OF-SORT          VALUE "Y".
       01  WS-HIGH-COUNT            PIC 9(3) VALUE 0.
       01  WS-LOW-COUNT             PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.

      *> DFSORT reference syntax for the SAME split-by-value job:
      *>
      *>   //SYSIN DD *
      *>     SORT FIELDS=(1,5,CH,A)
      *>     OUTFIL FNAMES=HIGHOUT,INCLUDE=(6,9,PD,GE,+100000)
      *>     OUTFIL FNAMES=LOWOUT, INCLUDE=(6,9,PD,LT,+100000)
      *>   /*
      *>
      *> A single DFSORT SORT+OUTFIL step can fan a sorted stream
      *> out to MULTIPLE output datasets in one pass based on a
      *> condition. GnuCOBOL's SORT has no OUTFIL, so we reproduce
      *> the fan-out with an OUTPUT PROCEDURE that RETURNs the
      *> already-sorted records and WRITEs each one to whichever of
      *> two files matches its amount - the pattern from Part 027,
      *> step 264, extended to write to more than one file.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ID
               USING UNSORTED-FILE
               OUTPUT PROCEDURE IS SPLIT-BY-AMOUNT.

           DISPLAY "High-value (>= 1000.00): " WS-HIGH-COUNT.
           DISPLAY "Low-value  (<  1000.00): " WS-LOW-COUNT.
           PERFORM SHOW-HIGH.
           PERFORM SHOW-LOW.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "C0003" TO UNS-ID. MOVE 500.00   TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "C0001" TO UNS-ID. MOVE 2500.00  TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "C0002" TO UNS-ID. MOVE 750.00   TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "C0004" TO UNS-ID. MOVE 12000.00 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

       SPLIT-BY-AMOUNT.
           OPEN OUTPUT HIGH-VALUE-FILE.
           OPEN OUTPUT LOW-VALUE-FILE.
           PERFORM UNTIL END-OF-SORT
               RETURN SORT-WORK-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       IF SORT-AMOUNT >= 1000.00
                           MOVE SORT-ID TO HIGH-ID
                           MOVE SORT-AMOUNT TO HIGH-AMOUNT
                           WRITE HIGH-RECORD
                           ADD 1 TO WS-HIGH-COUNT
                       ELSE
                           MOVE SORT-ID TO LOW-ID
                           MOVE SORT-AMOUNT TO LOW-AMOUNT
                           WRITE LOW-RECORD
                           ADD 1 TO WS-LOW-COUNT
                       END-IF
               END-RETURN
           END-PERFORM.
           CLOSE HIGH-VALUE-FILE.
           CLOSE LOW-VALUE-FILE.

       SHOW-HIGH.
           MOVE "N" TO WS-S-EOF.
           OPEN INPUT HIGH-VALUE-FILE.
           DISPLAY "---- HIGHVAL664.DAT ----".
           PERFORM UNTIL END-OF-SORT
               READ HIGH-VALUE-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       DISPLAY "  " HIGH-ID " " HIGH-AMOUNT
               END-READ
           END-PERFORM.
           CLOSE HIGH-VALUE-FILE.

       SHOW-LOW.
           MOVE "N" TO WS-S-EOF.
           OPEN INPUT LOW-VALUE-FILE.
           DISPLAY "---- LOWVAL664.DAT ----".
           PERFORM UNTIL END-OF-SORT
               READ LOW-VALUE-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       DISPLAY "  " LOW-ID " " LOW-AMOUNT
               END-READ
           END-PERFORM.
           CLOSE LOW-VALUE-FILE.
```

คอมไพล์และรัน:

```bash
cobc -x -o step664 step664.cob
./step664
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
High-value (>= 1000.00): 002
Low-value  (<  1000.00): 002
---- HIGHVAL664.DAT ----
  C0001 0002500.00
  C0004 0012000.00
---- LOWVAL664.DAT ----
  C0002 0000750.00
  C0003 0000500.00
```

### อธิบายจุดสำคัญ

- ทั้งสองไฟล์ผลลัพธ์เรียงลำดับตาม ID ถูกต้องภายในตัวเอง (`C0001` ก่อน `C0004` ในไฟล์ high-value,
  `C0002` ก่อน `C0003` ในไฟล์ low-value) เพราะข้อมูลผ่านการเรียงลำดับมาก่อนตั้งแต่ `SORT` แล้ว
  `OUTPUT PROCEDURE` แค่แยกไปเขียนคนละไฟล์เท่านั้น ไม่ได้เรียงใหม่
- `OUTPUT PROCEDURE` หนึ่งตัวสามารถเปิดและเขียนได้**หลายไฟล์พร้อมกัน**ตามต้องการ ทำให้เลียนแบบ
  ความสามารถ "fan-out" ของ `OUTFIL` ได้อย่างสมบูรณ์

### ข้อควรระวัง

- DFSORT `OUTFIL` ยังมีความสามารถอื่นอีกมากที่ตัวอย่างนี้ไม่ได้ครอบคลุม เช่น การเปลี่ยนรูปแบบข้อมูล
  ระหว่างทาง (`OUTREC`), การเรียงลำดับ field ใหม่ในผลลัพธ์, หรือการใส่ header/trailer record พิเศษ
  — ความสามารถเหล่านี้ต้องเขียนโค้ด COBOL เพิ่มเติมเองถ้าต้องการเลียนแบบ (ไม่ได้ "ฟรี" เหมือน DFSORT)
- ยิ่งจำนวนไฟล์ผลลัพธ์ที่ต้องการมากขึ้น (`OUTFIL` หลายตัวใน DFSORT) โค้ด COBOL ที่ต้องเขียนใน
  `OUTPUT PROCEDURE` จะซับซ้อนขึ้นตามไปด้วย (ต้อง `OPEN`/`CLOSE`/`IF` เพิ่มสำหรับแต่ละไฟล์)

### แบบฝึกหัดที่ 664.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `OUTFIL` หลายตัวในคำสั่ง DFSORT เดียว จึงมีประสิทธิภาพดีกว่าการรัน
DFSORT แยกกัน 2 รอบ (รอบแรกกรอง high-value รอบสองกรอง low-value)

**เฉลย**: เพราะการรัน DFSORT แยกกัน 2 รอบหมายความว่าต้อง**อ่านไฟล์ต้นทางและเรียงลำดับข้อมูลซ้ำสองครั้ง
**ซึ่งสิ้นเปลืองทั้งเวลาและทรัพยากร I/O อย่างมากเมื่อไฟล์ต้นทางมีขนาดใหญ่ (เรียงลำดับเป็นการดำเนินการ
ที่ต้นทุนสูงที่สุดในกระบวนการนี้) การใช้ `OUTFIL` หลายตัวใน DFSORT เดียวทำให้ **เรียงลำดับข้อมูลแค่
ครั้งเดียว** แล้วแยกผลลัพธ์ที่เรียงแล้วออกไปยังหลายไฟล์พร้อมกันในขณะที่กำลังเขียนผลลัพธ์ ซึ่งประหยัด
เวลาได้มากเมื่อเทียบกับการรันซ้ำหลายรอบ โดยเฉพาะกับข้อมูลขนาดหลักร้อยล้านระเบียนที่ DFSORT มักถูกใช้
งานจริง

---

## ขั้นตอนที่ 665: SUM FIELDS= — รวมยอด record ที่มี Key ซ้ำกันเป็นแถวเดียว

### แนวคิด

**`SUM FIELDS=`** คือความสามารถของ DFSORT ที่**รวม record ที่มี Key เหมือนกันให้เหลือแถวเดียว**
โดยบวกค่าตัวเลขของ field ที่ระบุเข้าด้วยกัน — เหมาะสำหรับงาน "ยุบรวมรายการธุรกรรมย่อยให้เหลือยอด
สุทธิต่อบัญชี" ที่พบบ่อยมากในงาน batch ทางการเงิน

### ไวยากรณ์อ้างอิง

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้**

```jcl
//SYSIN    DD *
    SORT FIELDS=(1,5,CH,A)
    SUM FIELDS=(6,9,PD)
/*
```

ตัวอย่างนี้เรียงข้อมูลตามไบต์ 1-5 (รหัสบัญชี) ก่อน แล้ว**ยุบรวม record ที่มีรหัสบัญชีเดียวกันให้เหลือ
แถวเดียว** โดยบวกค่าไบต์ 6-14 (ยอดเงิน, Packed Decimal) เข้าด้วยกัน — ถ้าบัญชี `A0001` มี 3 รายการ
ธุรกรรมย่อย ผลลัพธ์จะเหลือแค่ 1 แถวที่มียอดรวมของทั้ง 3 รายการ

### ตัวอย่างเปรียบเทียบ COBOL SORT (ทดสอบจริงแล้ว)

DFSORT ทำสิ่งนี้ในคำสั่งเดียว แต่ COBOL `SORT` ไม่มี clause แบบนี้ในตัว ต้องอาศัย**สองขั้นตอนรวมกัน**:
(1) `SORT` ธรรมดาตาม Key ก่อน (2) แล้วรันตรรกะ **Control-Break** (ทบทวนจาก Part 066 ขั้นตอนที่ 653)
เพื่อรวมยอดของแต่ละกลุ่ม Key และเขียนออกเป็น 1 record ต่อกลุ่ม:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP665-SUM-FIELDS-EQUIV.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED665.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORTED-FILE ASSIGN TO "SORTED665.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK665.DAT".
           SELECT SUMMARY-FILE ASSIGN TO "SUMMARY665.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ACCT             PIC X(5).
           05  UNS-AMOUNT           PIC 9(7)V99.

       FD  SORTED-FILE.
       01  SRT-RECORD.
           05  SRT-ACCT             PIC X(5).
           05  SRT-AMOUNT           PIC 9(7)V99.

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ACCT            PIC X(5).
           05  SORT-AMOUNT          PIC 9(7)V99.

       FD  SUMMARY-FILE.
       01  SUM-RECORD.
           05  SUM-ACCT             PIC X(5).
           05  SUM-TOTAL            PIC 9(8)V99.

       WORKING-STORAGE SECTION.
       01  WS-S-EOF                 PIC X VALUE "N".
           88  END-OF-SORT          VALUE "Y".
       01  WS-M-EOF                 PIC X VALUE "N".
           88  END-OF-SUMMARY       VALUE "Y".
       01  WS-PREV-ACCT             PIC X(5) VALUE SPACES.
       01  WS-FIRST-RECORD          PIC X VALUE "Y".
           88  IS-FIRST-RECORD      VALUE "Y".
       01  WS-ACCT-TOTAL            PIC 9(8)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.

      *> DFSORT reference syntax for the SAME dedup-and-sum job:
      *>
      *>   //SYSIN DD *
      *>     SORT FIELDS=(1,5,CH,A)
      *>     SUM FIELDS=(6,9,PD)
      *>   /*
      *>
      *> DFSORT's SUM FIELDS collapses records with matching keys
      *> into ONE output record per key, adding the numeric field.
      *> Plain COBOL SORT has no SUM clause, so we get the same
      *> result in two stages: SORT the raw feed by key first
      *> (Part 027), then run a single-level control-break total
      *> (step 653 of Part 066) that writes ONE summary record per
      *> account instead of merely displaying the totals.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ACCT
               USING UNSORTED-FILE
               GIVING SORTED-FILE.

           PERFORM SUMMARIZE-BY-ACCOUNT.
           PERFORM SHOW-SUMMARY.
           STOP RUN.

       BUILD-TEST-DATA.
      *> Same account can appear several times - typical for a
      *> feed of individual transaction lines that must be
      *> collapsed into one balance per account.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "A0001" TO UNS-ACCT. MOVE 100.00 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A0002" TO UNS-ACCT. MOVE 250.00 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A0001" TO UNS-ACCT. MOVE 300.00 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A0001" TO UNS-ACCT. MOVE  50.00 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A0002" TO UNS-ACCT. MOVE 400.00 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

       SUMMARIZE-BY-ACCOUNT.
           MOVE "N" TO WS-S-EOF.
           MOVE "Y" TO WS-FIRST-RECORD.
           MOVE 0 TO WS-ACCT-TOTAL.
           OPEN INPUT SORTED-FILE.
           OPEN OUTPUT SUMMARY-FILE.
           PERFORM READ-SORTED.
           PERFORM UNTIL END-OF-SORT
               IF NOT IS-FIRST-RECORD
                   IF SRT-ACCT NOT = WS-PREV-ACCT
                       PERFORM WRITE-SUMMARY-RECORD
                   END-IF
               END-IF
               ADD SRT-AMOUNT TO WS-ACCT-TOTAL
               MOVE SRT-ACCT TO WS-PREV-ACCT
               MOVE "N" TO WS-FIRST-RECORD
               PERFORM READ-SORTED
           END-PERFORM.
           PERFORM WRITE-SUMMARY-RECORD.
           CLOSE SORTED-FILE.
           CLOSE SUMMARY-FILE.

       READ-SORTED.
           READ SORTED-FILE
               AT END
                   SET END-OF-SORT TO TRUE
           END-READ.

       WRITE-SUMMARY-RECORD.
           MOVE WS-PREV-ACCT TO SUM-ACCT.
           MOVE WS-ACCT-TOTAL TO SUM-TOTAL.
           WRITE SUM-RECORD.
           MOVE 0 TO WS-ACCT-TOTAL.

       SHOW-SUMMARY.
           MOVE "N" TO WS-M-EOF.
           OPEN INPUT SUMMARY-FILE.
           DISPLAY "---- SUMMARY665.DAT (one record per account) ----".
           PERFORM UNTIL END-OF-SUMMARY
               READ SUMMARY-FILE
                   AT END
                       SET END-OF-SUMMARY TO TRUE
                   NOT AT END
                       DISPLAY "  " SUM-ACCT " " SUM-TOTAL
               END-READ
           END-PERFORM.
           CLOSE SUMMARY-FILE.
```

คอมไพล์และรัน:

```bash
cobc -x -o step665 step665.cob
./step665
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
---- SUMMARY665.DAT (one record per account) ----
  A0001 00000450.00
  A0002 00000650.00
```

### อธิบายจุดสำคัญ

- บัญชี `A0001` มี 3 รายการ (100.00, 300.00, 50.00) รวมเป็น **450.00** ตรงตามที่คำนวณ ✓
- บัญชี `A0002` มี 2 รายการ (250.00, 400.00) รวมเป็น **650.00** ตรงตามที่คำนวณ ✓
- ไฟล์ผลลัพธ์ `SUMMARY665.DAT` มีแค่ 2 record (หนึ่งต่อบัญชี) ทั้งที่ไฟล์ต้นทางมี 5 record — พิสูจน์
  ว่าการ "ยุบรวม" (collapse) ทำงานถูกต้องเหมือนกับที่ `SUM FIELDS=` ของ DFSORT จะทำ

### ข้อควรระวัง

- **DFSORT `SUM FIELDS=` ทำงานใน pass เดียวระหว่างการเรียงลำดับ** (มีประสิทธิภาพสูงมาก) ในขณะที่
  วิธีเทียบเท่าด้วย COBOL ต้องใช้ **สองขั้นตอนแยกกัน** (SORT ก่อน แล้วจึง control-break ทีหลัง) ทำให้
  ต้องอ่าน/เขียนข้อมูลผ่านดิสก์เพิ่มอีกหนึ่งรอบ (ไฟล์ `SORTED665.DAT` เป็น intermediate file) — นี่
  คือตัวอย่างที่ชัดเจนของความแตกต่างด้านประสิทธิภาพระหว่างยูทิลิตี้ระดับ JCL กับการเขียนโปรแกรมเอง
- ถ้าต้องการ `SUM FIELDS=` แบบมีเงื่อนไขกรองร่วมด้วย (เช่น สรุปยอดเฉพาะสถานะ active) ต้องรวม
  เทคนิคจากขั้นตอนที่ 663 (`INPUT PROCEDURE` กรอง) เข้ากับขั้นตอนนี้ (control-break สรุปยอด) ด้วย
  กัน — ตัวอย่างแบบบูรณาการนี้จะแสดงในขั้นตอนที่ 669

### แบบฝึกหัดที่ 665.1

**โจทย์**: จงอธิบายว่าทำไมการยุบรวม record ด้วย control-break (แบบ COBOL) จึง**ต้อง**เรียงลำดับข้อมูล
ตาม Key มาก่อนเสมอ ทั้งที่ในทางทฤษฎีเราสามารถรวมยอดของ Key เดียวกันได้โดยไม่ต้องเรียงลำดับ (เช่น ใช้
ตารางในหน่วยความจำสะสมยอด)

**เฉลยแนวทาง**: การเรียงลำดับก่อนทำให้ record ที่มี Key เดียวกันมาอยู่ติดกันเสมอ ทำให้ตรวจจับ "จุดที่
Key เปลี่ยน" ได้ง่ายด้วยการเปรียบเทียบกับ record ก่อนหน้าเพียงตัวเดียว (ใช้หน่วยความจำคงที่ไม่ว่าจะมี
กี่ Key ที่แตกต่างกัน) ในขณะที่วิธีใช้ตารางสะสมยอดในหน่วยความจำ (ไม่ต้องเรียงลำดับ) ต้องการหน่วยความ
จำที่แปรผันตามจำนวน Key ที่ไม่ซ้ำกันทั้งหมด ซึ่งอาจมีขนาดใหญ่เกินกว่าจะเก็บในหน่วยความจำได้เมื่อมี
บัญชีนับล้านบัญชี (ปัญหาแบบเดียวกับที่ Part 066 ขั้นตอนที่ 658 เตือนไว้เรื่องตาราง `OCCURS` ที่มี
ขนาดจำกัด) การเรียงลำดับก่อนจึงเป็นแนวทางที่ปรับขนาด (scale) ได้ดีกว่าสำหรับข้อมูลปริมาณมหาศาล แม้จะ
มีต้นทุนของการเรียงลำดับเพิ่มเข้ามาก็ตาม

---

## ขั้นตอนที่ 666: ICETOOL เบื้องต้น — ยูทิลิตี้เสริมสำหรับงานที่ซับซ้อนกว่า SORT เดียว

> ⚠️ **หมายเหตุ**: ขั้นตอนนี้เป็นเนื้อหาเชิงแนวคิดล้วน ๆ ไม่มีโค้ด COBOL เทียบเท่าให้ทดสอบ เพราะ
> ความสามารถบางอย่างของ ICETOOL (โดยเฉพาะสถิติหลายมิติ) ซับซ้อนเกินกว่าจะเทียบเท่าด้วยโค้ดสั้น ๆ

### ICETOOL คืออะไร

**ICETOOL** คือยูทิลิตี้ที่มาพร้อมกับ DFSORT ทำหน้าที่เป็น "เปลือกห่อ" (wrapper) รอบ DFSORT อีกที
เพื่อให้เขียนงานที่ต้องการ**หลายขั้นตอนของ DFSORT ต่อเนื่องกัน**หรือ**สร้างรายงานสถิติสรุป**ได้ง่ายขึ้น
โดยไม่ต้องเขียน JCL Step แยกหลาย Step

### ไวยากรณ์อ้างอิงเบื้องต้น

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้**

```jcl
//TOOLSTEP EXEC PGM=ICETOOL
//TOOLMSG  DD SYSOUT=*
//IN1      DD DSN=PROD.TRANS.DAILY,DISP=SHR
//SELOUT   DD DSN=PROD.TRANS.SELECTED,DISP=(NEW,CATLG,DELETE)
//STATRPT  DD SYSOUT=*
//TOOLIN   DD *
    SELECT FROM(IN1) TO(SELOUT) ON(1,5,CH) FIRST
    STATS FROM(IN1) TOTAL(6,9,PD)
/*
```

- **`SELECT ... FIRST`**: เลือก record **แรก**ของแต่ละกลุ่ม Key เท่านั้น (เช่น "เอาแค่ธุรกรรมแรกสุด
  ของแต่ละบัญชี") — operator ที่ใช้บ่อยอื่น ๆ ได้แก่ `LAST` (record สุดท้ายของกลุ่ม), `ALLDUPS`
  (record ที่มี Key ซ้ำทั้งหมด), `NODUPS` (record ที่มี Key ไม่ซ้ำใครเลย)
- **`STATS ... TOTAL(...)`**: สร้างรายงานสถิติสรุป (ผลรวม, ค่าเฉลี่ย, ค่าสูงสุด/ต่ำสุด) ของ field
  ที่ระบุ โดยไม่ต้องเขียน record ผลลัพธ์ใหม่เลย เพียงแค่สร้างรายงานสรุปเท่านั้น

### เปรียบเทียบเชิงแนวคิดกับสิ่งที่ COBOL ทำได้

| ความสามารถ ICETOOL | เทียบเท่า COBOL |
|---|---|
| `SELECT ... FIRST` | Control-Break ที่เขียนแค่ record แรกของแต่ละกลุ่ม (ตรงข้ามกับ `SUM` ในขั้นตอนที่ 665 ที่เขียนยอดรวม) |
| `SELECT ... NODUPS` | ตรวจสอบด้วย `SEARCH`/ตารางแบบ Part 066 ขั้นตอนที่ 658 (Idempotent check) ปรับใช้เพื่อหา Key ที่ไม่ซ้ำ |
| `STATS ... TOTAL` | สะสมยอดด้วยตัวแปร `WS-GRAND-TOTAL` (แบบเดียวกับ Control-Break ทุก Part ที่ผ่านมา) แล้ว `DISPLAY` แทนที่จะเขียนไฟล์ผลลัพธ์ใหม่ |

### ข้อควรระวัง

- ICETOOL เป็นเครื่องมือที่มีความสามารถมากกว่าที่แสดงในขั้นตอนนี้อย่างมาก (มี operator อีกหลายสิบตัว)
  หลักสูตรนี้ครอบคลุมแค่แนวคิดพื้นฐานเพื่อให้รู้จักชื่อและจุดประสงค์ของมันเท่านั้น การเรียนรู้ ICETOOL
  อย่างละเอียดควรทำในสภาพแวดล้อม Mainframe จริงหรือผ่านเอกสาร IBM Redbook เฉพาะทาง
- อย่าสับสน ICETOOL กับ DFSORT — ICETOOL **เรียกใช้ DFSORT เป็นเครื่องมือภายใน**ของมันเองอีกที ไม่ใช่
  คนละผลิตภัณฑ์ที่แยกจากกันโดยสิ้นเชิง

### แบบฝึกหัดที่ 666.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ ICETOOL เพื่อรัน "เลือก record แรกของแต่ละกลุ่ม" และ "สร้างรายงาน
สถิติสรุป" พร้อมกันในขั้นตอนเดียว จึงมีประโยชน์มากกว่าการเขียน JCL Step แยกกันสองงาน

**เฉลยแนวทาง**: เพราะ ICETOOL สามารถประมวลผลหลายคำสั่ง (`SELECT`, `STATS` ฯลฯ) จากข้อมูลต้นทางเดียวกัน
ภายใน JCL Step เดียว ทำให้อ่านไฟล์ต้นทางแค่ครั้งเดียว (หรือน้อยครั้งกว่าการแยก Step) ลดภาระ I/O และ
เวลารวมของ batch window ลงได้ นอกจากนี้ยังลดความซับซ้อนในการจัดการ JCL (Job Step น้อยลง จัดการ
Dependency ระหว่าง Step น้อยลง ทบทวนแนวคิดจาก Part 066 ขั้นตอนที่ 656) ทำให้ดูแลรักษาง่ายกว่าการมี
หลาย Job Step แยกกันสำหรับแต่ละความต้องการ

---

## ขั้นตอนที่ 667: JCL สำหรับรัน DFSORT — ภาพรวมสมบูรณ์และการเชื่อมโยงกับ Part 052/053

### ทบทวนโครงสร้าง JCL

Part 052-053 สอนโครงสร้าง JCL พื้นฐาน (`JOB`, `EXEC`, `DD`) และแนวคิด Job Step/Condition Code ไว้
แล้ว ขั้นตอนนี้จะแสดง JCL ที่สมบูรณ์สำหรับรันงาน DFSORT จริง พร้อมเชื่อมโยงทุกแนวคิดที่เรียนมาใน Part
นี้เข้าด้วยกัน

### ตัวอย่าง JCL สมบูรณ์

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้**

```jcl
//SORTJOB  JOB (ACCT123),'MONTHLY SUMMARY',CLASS=A,MSGCLASS=X,
//         NOTIFY=&SYSUID
//*
//* STEP 1: Sort the raw daily transaction feed by account,
//*         dropping cancelled transactions, and summarizing
//*         duplicate accounts into one balance each.
//*
//STEP010  EXEC PGM=SORT,REGION=4M
//SYSOUT   DD SYSOUT=*
//SORTIN   DD DSN=PROD.TRANS.DAILY.RAW,DISP=SHR
//SORTOUT  DD DSN=PROD.TRANS.DAILY.SUMMARY,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(20,5),RLSE),
//            DCB=(LRECL=20,RECFM=FB)
//SYSIN    DD *
    SORT FIELDS=(1,5,CH,A)
    OMIT COND=(6,1,CH,EQ,C'X')
    SUM FIELDS=(7,9,PD)
/*
//*
//* STEP 2: Only run if STEP010 completed with return code 0 -
//* this dependency is exactly the kind of thing a job scheduler
//* like CA-7/Control-M (Part 066, step 656) also enforces across
//* MULTIPLE jobs, not just steps within one job.
//*
//STEP020  EXEC PGM=IEFBR14,COND=(0,NE,STEP010)
//DUMMY    DD DSN=PROD.TEMP.PLACEHOLDER,
//            DISP=(NEW,DELETE,DELETE),
//            SPACE=(TRK,(1,1))
```

### อธิบายจุดสำคัญ

- **`SORTIN`/`SORTOUT` DD statement** ระบุไฟล์จริงบน z/OS (`DSN=` คือ Data Set Name) — สังเกต
  `DISP=(NEW,CATLG,DELETE)` ที่หมายถึง "สร้างไฟล์ใหม่ (NEW), ถ้าสำเร็จให้บันทึกลง catalog (CATLG),
  ถ้าล้มเหลวให้ลบทิ้ง (DELETE)" ทบทวนแนวคิด disposition จาก Part 052
- **`SYSIN` DD \* ... /\*** คือรูปแบบ "in-stream data" มาตรฐานของ JCL ที่ฝัง control statement ไว้
  ในตัว JCL เอง (ไม่ต้องแยกไฟล์ต่างหาก) — ทบทวนจาก Part 052
- **`COND=(0,NE,STEP010)`** บน `STEP020` หมายถึง "ข้าม step นี้ถ้า return code ของ STEP010 ไม่เท่ากับ
  0" — นี่คือกลไกเดียวกับที่ Part 066 ขั้นตอนที่ 657 พิสูจน์แล้วว่า `RETURN-CODE` ของโปรแกรม COBOL
  กลายเป็นค่าที่ JCL ตรวจสอบได้โดยตรง เพียงแต่ในกรณีนี้เป็น return code ของโปรแกรม `SORT` (DFSORT)
  แทนที่จะเป็นโปรแกรม COBOL ของเราเอง

### ข้อควรระวัง

- **`REGION=4M`** บน `EXEC PGM=SORT` กำหนดขนาดหน่วยความจำสูงสุดที่ DFSORT ใช้ได้ — ถ้าตั้งค่าน้อย
  เกินไปสำหรับปริมาณข้อมูลจริง DFSORT อาจทำงานช้าลงมาก (ต้องใช้ work dataset บนดิสก์แทนหน่วยความจำ
  มากขึ้น รายละเอียดจะกล่าวถึงในขั้นตอนที่ 668) หรือล้มเหลวไปเลยถ้าน้อยเกินไปจริง ๆ
- ผลลัพธ์ของ `STEP010` (`PROD.TRANS.DAILY.SUMMARY`) คือไฟล์ที่มีโครงสร้างตรงกับผลลัพธ์ของขั้นตอนที่
  665 ในเอกสารนี้ทุกประการในเชิงแนวคิด (สรุปยอดต่อบัญชี กรองสถานะยกเลิกทิ้ง) — แสดงให้เห็นว่า JCL
  หนึ่ง Step สามารถทำสิ่งที่โปรแกรม COBOL ในขั้นตอนที่ 663+665 ต้องเขียนรวมกันหลายสิบบรรทัดได้ด้วย
  control statement เพียง 3 บรรทัด

### แบบฝึกหัดที่ 667.1

**โจทย์**: จงอธิบายว่าทำไม `STEP020` ในตัวอย่างข้างต้นจึงมีเงื่อนไข `COND=(0,NE,STEP010)` แทนที่จะ
รันเสมอโดยไม่มีเงื่อนไข

**เฉลย**: เพราะ `STEP020` (ในตัวอย่างนี้สมมติเป็นงานที่ต้องพึ่งพาผลลัพธ์ของ `STEP010` เช่น การส่ง
ไฟล์สรุปไปยังระบบถัดไป) ไม่ควรทำงานเลยถ้า `STEP010` ล้มเหลว เพราะผลลัพธ์ `PROD.TRANS.DAILY.SUMMARY`
อาจไม่สมบูรณ์หรือไม่มีอยู่จริง การรัน `STEP020` ต่อไปทั้งที่ข้อมูลต้นทางมีปัญหาอาจทำให้เกิดข้อผิดพลาด
ลุกลามต่อไปยังระบบอื่น `COND=(0,NE,STEP010)` จึงทำหน้าที่เป็นเงื่อนไขป้องกัน (guard condition) ที่
บังคับให้ `STEP020` ถูกข้าม (bypass) โดยอัตโนมัติถ้า Return Code ของ `STEP010` ไม่เท่ากับ 0 (คือไม่
สำเร็จสมบูรณ์) หลักการเดียวกับ Dependency Graph ที่ Part 066 ขั้นตอนที่ 656 อธิบายไว้ในระดับของ Job
Scheduler เพียงแต่ในที่นี้เป็น Dependency ระหว่าง Step ภายใน Job เดียวกัน

---

## ขั้นตอนที่ 668: ประสิทธิภาพของ DFSORT — Work Dataset และการจัดการหน่วยความจำ

> ⚠️ **หมายเหตุ**: ขั้นตอนนี้เป็นเนื้อหาเชิงแนวคิดเกี่ยวกับการปรับแต่งประสิทธิภาพของยูทิลิตี้ z/OS
> ล้วน ๆ ไม่มีโค้ดให้ทดสอบ เพราะเป็นพฤติกรรมภายในของ DFSORT ที่ผู้ใช้ทั่วไปมองไม่เห็นโดยตรง

### ทำไม DFSORT ถึงเร็วกว่าการเขียนโปรแกรมเรียงลำดับเอง

DFSORT ใช้เทคนิคขั้นสูงหลายอย่างที่สั่งสมมาจากประสบการณ์หลายสิบปีของ IBM ในการปรับแต่งการเรียงลำดับ
ข้อมูลปริมาณมหาศาลให้เร็วที่สุด:

1. **Sort Work Datasets (SORTWKxx)**: เมื่อข้อมูลมีขนาดใหญ่เกินกว่าจะเรียงในหน่วยความจำได้ทั้งหมด
   DFSORT จะแบ่งข้อมูลเป็นส่วนย่อย ๆ เรียงแต่ละส่วนในหน่วยความจำ แล้วเขียนลง **Work Dataset**
   ชั่วคราว (DD ชื่อ `SORTWK01`, `SORTWK02`, ... ) จากนั้นจึงทำ **Merge Pass** รวมส่วนย่อยที่เรียง
   แล้วทั้งหมดเข้าด้วยกัน — แนวคิดพื้นฐานเดียวกับอัลกอริทึม **External Merge Sort** ที่สอนในวิชา
   Algorithm ทั่วไป เพียงแต่ DFSORT ปรับแต่งอย่างละเอียดสำหรับสถาปัตยกรรม Mainframe โดยเฉพาะ
2. **Dynamic Memory Allocation**: DFSORT รุ่นใหม่สามารถปรับขนาดหน่วยความจำที่ใช้แบบไดนามิกตาม
   ปริมาณข้อมูลจริง แทนที่จะต้องกำหนดค่าคงที่ล่วงหน้าเสมอ
3. **Hardware-assisted Sorting**: บน Mainframe รุ่นใหม่ DFSORT สามารถใช้ประโยชน์จากชุดคำสั่งพิเศษของ
   ฮาร์ดแวร์ IBM Z (เช่น z/Architecture Sort Assist Instructions) เพื่อเร่งความเร็วการเปรียบเทียบและ
   ย้ายข้อมูลระดับต่ำ ซึ่งเป็นสิ่งที่โปรแกรม COBOL ทั่วไปเข้าไม่ถึงโดยตรง

### ผลกระทบเชิงปฏิบัติสำหรับนักพัฒนา

แม้นักพัฒนา COBOL ทั่วไปจะไม่ได้เขียนโค้ดเพื่อควบคุมกลไกเหล่านี้โดยตรง แต่การเข้าใจว่ามันมีอยู่ช่วยให้
ตัดสินใจได้ดีขึ้นว่าเมื่อไรควรใช้ DFSORT แทนโปรแกรม COBOL: **ยิ่งข้อมูลมีขนาดใหญ่มากเท่าไร ช่องว่าง
ด้านประสิทธิภาพระหว่าง DFSORT กับ COBOL SORT ก็ยิ่งกว้างขึ้นเท่านั้น** สำหรับข้อมูลขนาดเล็ก-กลาง (เช่น
ที่ทดสอบในหลักสูตรนี้) ความแตกต่างแทบไม่มีนัยสำคัญ แต่สำหรับข้อมูลระดับร้อยล้านระเบียนขึ้นไป การเลือก
ใช้ DFSORT แทนการเขียนโปรแกรม COBOL เองอาจลดเวลาการประมวลผลได้อย่างมีนัยสำคัญ

### ตารางสรุปปัจจัยที่ส่งผลต่อประสิทธิภาพ

| ปัจจัย | ผลกระทบ |
|---|---|
| จำนวน `SORTWKxx` datasets ที่กำหนด | ยิ่งมากยิ่งรองรับข้อมูลขนาดใหญ่ได้ดีขึ้น แต่ใช้พื้นที่ดิสก์มากขึ้น |
| ขนาด `REGION=` ที่กำหนดใน JCL | ยิ่งมากยิ่งลดการพึ่งพา work dataset (เรียงในหน่วยความจำได้มากขึ้น) |
| จำนวน Key และความซับซ้อนของ `INCLUDE`/`OMIT` | ยิ่งซับซ้อนยิ่งใช้เวลาประมวลผลต่อ record มากขึ้น |
| การใช้ `OUTFIL`/`SUM` รวมในคำสั่งเดียว | ลดจำนวนรอบการอ่าน/เขียนข้อมูล เพิ่มประสิทธิภาพโดยรวม (ทบทวนขั้นตอนที่ 664-665) |

### ข้อควรระวัง

- การปรับแต่งค่าพารามิเตอร์เหล่านี้ (`REGION=`, จำนวน `SORTWKxx`) เป็นงานของทีม **Systems
  Programmer** หรือ **Performance Tuning Specialist** ไม่ใช่งานประจำวันของโปรแกรมเมอร์แอปพลิเคชัน
  ทั่วไป — แต่การเข้าใจแนวคิดพื้นฐานช่วยให้สื่อสารกับทีมเหล่านี้ได้อย่างมีประสิทธิภาพเมื่อพบปัญหาเรื่อง
  ประสิทธิภาพของงาน batch
- เนื้อหาการปรับแต่งประสิทธิภาพ COBOL ในระดับที่ลึกกว่านี้ (รวมถึงประเด็นที่เกี่ยวกับโปรแกรม COBOL
  โดยตรง ไม่ใช่แค่ DFSORT) จะเรียนอย่างละเอียดใน **Part 068 (Performance Tuning สำหรับ Mainframe
  COBOL)** ซึ่งเป็น Part ถัดจากนี้ในหลักสูตร

### แบบฝึกหัดที่ 668.1

**โจทย์**: จงอธิบายว่าทำไมช่องว่างด้านประสิทธิภาพระหว่าง DFSORT กับ COBOL SORT statement จึง "กว้างขึ้น
เมื่อข้อมูลมีขนาดใหญ่ขึ้น" แทนที่จะคงที่ไม่ว่าข้อมูลจะมีขนาดเท่าไร

**เฉลยแนวทาง**: เพราะเทคนิคขั้นสูงที่ DFSORT ใช้ (Sort Work Dataset สำหรับ External Merge Sort,
Hardware-assisted Sorting) ให้ประโยชน์ชัดเจนก็ต่อเมื่อข้อมูลมีขนาดใหญ่จนไม่สามารถเรียงในหน่วยความจำ
เดียวได้หมด สำหรับข้อมูลขนาดเล็กที่เรียงในหน่วยความจำได้สบาย ๆ (เช่น ตัวอย่างในหลักสูตรนี้ที่มีแค่
ไม่กี่ระเบียน) ทั้ง DFSORT และ COBOL SORT statement ต่างก็ใช้อัลกอริทึมการเรียงลำดับพื้นฐานที่ให้
ผลลัพธ์เร็วพอ ๆ กันในทางปฏิบัติ แต่เมื่อข้อมูลใหญ่ขึ้นเรื่อย ๆ จนต้องพึ่งพา external sort (เรียงลำดับ
โดยใช้ดิสก์ช่วย) ความแตกต่างของเทคนิคการจัดการ I/O และหน่วยความจำระหว่างสองเครื่องมือจะเริ่มส่งผล
ชัดเจนมากขึ้นเรื่อย ๆ ตามขนาดข้อมูลที่เพิ่มขึ้น

---

## ขั้นตอนที่ 669: ตัวอย่างบูรณาการ — จำลองงาน DFSORT+ICETOOL แบบเต็มรูปแบบด้วย COBOL SORT

### รวมทุกเทคนิคเข้าด้วยกันในโปรแกรมเดียว

ปิดท้ายด้วยตัวอย่างที่ครอบคลุมที่สุดของ Part นี้: จำลองงาน DFSORT ที่ผสมผสาน**ทั้ง OMIT, SUM, และ
OUTFIL INCLUDE** เข้าด้วยกันในคำสั่งเดียว (แบบที่แสดงไว้ในขั้นตอนที่ 667) ด้วย COBOL `SORT` ที่
คอมไพล์และรันได้จริง

### JCL/DFSORT ต้นแบบที่กำลังจำลอง (อ้างอิง)

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถรันได้ในสภาพแวดล้อมนี้**

```jcl
//STEP1 EXEC PGM=SORT
//SYSOUT DD SYSOUT=*
//SORTIN  DD DSN=PROD.TRANS.UNSORTED,DISP=SHR
//SORTOUT DD DSN=PROD.TRANS.SUMMARY,DISP=(NEW,CATLG)
//SYSIN DD *
    OPTION COPY
    SORT FIELDS=(1,5,CH,A)
    OMIT COND=(6,1,CH,EQ,C'X')
    SUM FIELDS=(7,9,PD)
    OUTFIL INCLUDE=(7,9,PD,GE,+50000)
/*
```

งานนี้ทำ 4 อย่างในคำสั่งเดียว: (1) เรียงตามบัญชี (2) ทิ้งธุรกรรมที่ยกเลิกแล้ว (สถานะ `X`) (3) รวมยอด
ธุรกรรมย่อยของบัญชีเดียวกันเป็นยอดเดียว (4) เก็บเฉพาะบัญชีที่ยอดรวมสุดท้าย **มากกว่าหรือเท่ากับ
500.00** เท่านั้น (ตัวอย่างนี้ปรับ threshold ให้เล็กกว่า JCL อ้างอิงเพื่อให้เห็นผลกับข้อมูลทดสอบขนาด
เล็ก)

### โค้ดตัวอย่างสมบูรณ์ (ทดสอบจริงแล้ว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP669-FULL-DFSORT-EQUIV.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED669.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORTED-FILE ASSIGN TO "SORTED669.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK669.DAT".
           SELECT FINAL-REPORT ASSIGN TO "FINAL669.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ACCT             PIC X(5).
           05  UNS-STATUS           PIC X(1).
           05  UNS-AMOUNT           PIC 9(7)V99.

       FD  SORTED-FILE.
       01  SRT-RECORD.
           05  SRT-ACCT             PIC X(5).
           05  SRT-STATUS           PIC X(1).
           05  SRT-AMOUNT           PIC 9(7)V99.

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ACCT            PIC X(5).
           05  SORT-STATUS          PIC X(1).
           05  SORT-AMOUNT          PIC 9(7)V99.

       FD  FINAL-REPORT.
       01  RPT-RECORD               PIC X(50).

       WORKING-STORAGE SECTION.
       01  WS-U-EOF                 PIC X VALUE "N".
           88  UNSORTED-EOF         VALUE "Y".
       01  WS-S-EOF                 PIC X VALUE "N".
           88  END-OF-SORT          VALUE "Y".
       01  WS-PREV-ACCT             PIC X(5) VALUE SPACES.
       01  WS-FIRST-RECORD          PIC X VALUE "Y".
           88  IS-FIRST-RECORD      VALUE "Y".
       01  WS-ACCT-TOTAL            PIC 9(8)V99 VALUE 0.
       01  WS-OMITTED-COUNT         PIC 9(3) VALUE 0.
       01  WS-REPORTED-COUNT        PIC 9(3) VALUE 0.
       01  WS-DISPLAY-AMT           PIC Z,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-TEST-DATA.

      *> ==================================================
      *> REFERENCE-ONLY: the DFSORT/ICETOOL job this program
      *> replicates step-by-step, exactly as it would be typed
      *> into a real z/OS JCL SYSIN dataset. This text is NOT
      *> COBOL and does NOT run in this sandbox - see Part 052/
      *> 053 for full JCL syntax and Part 067 steps 661-668 for
      *> a field-by-field explanation of every clause below.
      *>
      *>   //STEP1 EXEC PGM=SORT
      *>   //SYSOUT DD SYSOUT=*
      *>   //SORTIN  DD DSN=PROD.TRANS.UNSORTED,DISP=SHR
      *>   //SORTOUT DD DSN=PROD.TRANS.SUMMARY,DISP=(NEW,CATLG)
      *>   //SYSIN DD *
      *>     OPTION COPY
      *>     SORT FIELDS=(1,5,CH,A)
      *>     OMIT COND=(6,1,CH,EQ,C'X')
      *>     SUM FIELDS=(7,9,PD)
      *>     OUTFIL INCLUDE=(7,9,PD,GE,+50000)
      *>   /*
      *> ==================================================
      *>
      *> The COBOL equivalent below does the SAME four things in
      *> one pass: (1) SORT by account, (2) OMIT cancelled (status
      *> X) records via INPUT PROCEDURE, (3) SUM duplicate accounts
      *> via a control-break OUTPUT PROCEDURE, (4) INCLUDE only
      *> accounts whose final total is >= 500.00.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ACCT
               INPUT PROCEDURE IS OMIT-CANCELLED
               OUTPUT PROCEDURE IS SUM-AND-FILTER.

           DISPLAY "Omitted (status=X)      : " WS-OMITTED-COUNT.
           DISPLAY "Accounts in final report: " WS-REPORTED-COUNT.
           PERFORM SHOW-FINAL-REPORT.
           STOP RUN.

       BUILD-TEST-DATA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "A0003" TO UNS-ACCT. MOVE "A" TO UNS-STATUS.
           MOVE 200.00 TO UNS-AMOUNT. WRITE UNS-RECORD.
           MOVE "A0001" TO UNS-ACCT. MOVE "A" TO UNS-STATUS.
           MOVE 300.00 TO UNS-AMOUNT. WRITE UNS-RECORD.
           MOVE "A0002" TO UNS-ACCT. MOVE "X" TO UNS-STATUS.
           MOVE 999.00 TO UNS-AMOUNT. WRITE UNS-RECORD.
           MOVE "A0001" TO UNS-ACCT. MOVE "A" TO UNS-STATUS.
           MOVE 250.00 TO UNS-AMOUNT. WRITE UNS-RECORD.
           MOVE "A0003" TO UNS-ACCT. MOVE "A" TO UNS-STATUS.
           MOVE 100.00 TO UNS-AMOUNT. WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

       OMIT-CANCELLED.
           OPEN INPUT UNSORTED-FILE.
           PERFORM UNTIL UNSORTED-EOF
               READ UNSORTED-FILE
                   AT END
                       SET UNSORTED-EOF TO TRUE
                   NOT AT END
                       IF UNS-STATUS = "X"
                           ADD 1 TO WS-OMITTED-COUNT
                       ELSE
                           MOVE UNS-ACCT TO SORT-ACCT
                           MOVE UNS-STATUS TO SORT-STATUS
                           MOVE UNS-AMOUNT TO SORT-AMOUNT
                           RELEASE SORT-RECORD
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE UNSORTED-FILE.

       SUM-AND-FILTER.
           MOVE "Y" TO WS-FIRST-RECORD.
           MOVE 0 TO WS-ACCT-TOTAL.
           OPEN OUTPUT FINAL-REPORT.
           PERFORM UNTIL END-OF-SORT
               RETURN SORT-WORK-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                       PERFORM FLUSH-IF-QUALIFIES
                   NOT AT END
                       IF NOT IS-FIRST-RECORD
                           IF SORT-ACCT NOT = WS-PREV-ACCT
                               PERFORM FLUSH-IF-QUALIFIES
                           END-IF
                       END-IF
                       ADD SORT-AMOUNT TO WS-ACCT-TOTAL
                       MOVE SORT-ACCT TO WS-PREV-ACCT
                       MOVE "N" TO WS-FIRST-RECORD
               END-RETURN
           END-PERFORM.
           CLOSE FINAL-REPORT.

       FLUSH-IF-QUALIFIES.
      *> This paragraph plays the role of OUTFIL INCLUDE= in the
      *> reference JCL: only accounts whose SUMMED total clears
      *> the 500.00 threshold make it into the final report.
           IF WS-ACCT-TOTAL >= 500.00
               MOVE SPACES TO RPT-RECORD
               MOVE WS-ACCT-TOTAL TO WS-DISPLAY-AMT
               STRING WS-PREV-ACCT " TOTAL: " WS-DISPLAY-AMT
                   DELIMITED BY SIZE INTO RPT-RECORD
               WRITE RPT-RECORD
               ADD 1 TO WS-REPORTED-COUNT
           END-IF.
           MOVE 0 TO WS-ACCT-TOTAL.

       SHOW-FINAL-REPORT.
           MOVE "N" TO WS-S-EOF.
           OPEN INPUT FINAL-REPORT.
           DISPLAY "---- FINAL669.DAT ----".
           PERFORM UNTIL END-OF-SORT
               READ FINAL-REPORT
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       DISPLAY "  " RPT-RECORD
               END-READ
           END-PERFORM.
           CLOSE FINAL-REPORT.
```

คอมไพล์และรัน:

```bash
cobc -x -o step669 step669.cob
./step669
```

**ผลลัพธ์จริง (ทดสอบแล้วด้วย GnuCOBOL):**

```
Omitted (status=X)      : 001
Accounts in final report: 001
---- FINAL669.DAT ----
  A0001 TOTAL:       550.00
```

### วิเคราะห์ผลลัพธ์ทีละขั้นตอน

ตรวจสอบผลลัพธ์ตามตรรกะทั้ง 4 ขั้นตอนที่จำลองมาจาก DFSORT:

1. **OMIT (สถานะ X)**: บัญชี `A0002` (999.00, สถานะ `X`) ถูกทิ้งไป → `Omitted = 1` ✓
2. **SORT**: บัญชีที่เหลือ (`A0001` x2, `A0003` x2) ถูกเรียงลำดับตาม Key
3. **SUM**: `A0001` = 300.00 + 250.00 = **550.00**, `A0003` = 200.00 + 100.00 = **300.00**
4. **OUTFIL INCLUDE (>= 500.00)**: `A0001` (550.00) ผ่านเงื่อนไข ✓ แต่ `A0003` (300.00) **ไม่ผ่าน**
   จึงไม่ปรากฏในรายงานสุดท้าย → `Accounts in final report = 1` ✓

ผลลัพธ์ตรงกับที่คาดหวังทุกประการ พิสูจน์ว่าตรรกะทั้ง 4 ขั้นตอนที่ DFSORT ทำในคำสั่งเดียว สามารถทำได้
ด้วย COBOL `SORT` ที่มีทั้ง `INPUT PROCEDURE` และ `OUTPUT PROCEDURE` ผสมผสานกันอย่างถูกต้อง

### ข้อควรระวัง

- สังเกตว่า `FLUSH-IF-QUALIFIES` ถูกเรียกทั้งตอน**เปลี่ยนกลุ่ม Key** (ภายใน `NOT AT END`) และตอน
  **จบไฟล์** (ภายใน `AT END`) — รูปแบบเดียวกับ "flush กลุ่มสุดท้าย" ที่ Part 066 ขั้นตอนที่ 653 เตือน
  ไว้ ถ้าลืมส่วนใดส่วนหนึ่ง บัญชีกลุ่มสุดท้าย (`A0003` ในตัวอย่างนี้) จะไม่ถูกตรวจสอบเงื่อนไข INCLUDE
  เลย
- โค้ดนี้ซับซ้อนกว่าตัวอย่างก่อนหน้าทั้งหมดใน Part นี้ เพราะรวม 3 เทคนิค (`INPUT PROCEDURE` กรอง,
  `OUTPUT PROCEDURE` ควบคู่กับ control-break, และเงื่อนไขกรองหลัง sum) เข้าด้วยกันในโปรแกรมเดียว
  ในขณะที่ DFSORT ทำสิ่งเดียวกันด้วย control statement เพียง 4 บรรทัด — นี่คือข้อแตกต่างที่ชัดเจน
  ที่สุดที่แสดงให้เห็นว่าทำไมองค์กรจึงยังคงเลือกใช้ DFSORT สำหรับงานประเภทนี้ (ทบทวนเหตุผลจากขั้นตอน
  ที่ 661)

### แบบฝึกหัดที่ 669.1

**โจทย์**: จงคำนวณด้วยมือว่าถ้าเปลี่ยน threshold ใน `FLUSH-IF-QUALIFIES` จาก `500.00` เป็น `250.00`
ผลลัพธ์สุดท้ายจะมีกี่บัญชี และบัญชีอะไรบ้าง

**เฉลย**: จะมี **2 บัญชี**: `A0001` (550.00 ผ่านเงื่อนไข >= 250.00) และ `A0003` (300.00 ก็ผ่านเงื่อนไข
>= 250.00 เช่นกัน เพราะ 300.00 มากกว่า 250.00) ทั้งสองบัญชีจะปรากฏในรายงานสุดท้าย มีเพียงบัญชี
`A0002` เท่านั้นที่ยังคงถูกกรองทิ้งตั้งแต่ขั้นตอน OMIT (เพราะสถานะ `X`) ไม่เกี่ยวข้องกับ threshold นี้
เลย

---

## ขั้นตอนที่ 670: สรุปเปรียบเทียบ DFSORT, ICETOOL, และ COBOL SORT — เมื่อไรควรใช้อะไร

### ตารางสรุปสมบูรณ์

| แง่มุม | COBOL SORT (Part 027, 066) | DFSORT | ICETOOL |
|---|---|---|---|
| ระดับการทำงาน | ภายในโปรแกรม | ยูทิลิตี้ระดับ JCL | ยูทิลิตี้ห่อรอบ DFSORT |
| ต้องเขียนโปรแกรมไหม | ต้อง | ไม่ต้อง | ไม่ต้อง |
| ความสามารถกรอง/สรุปยอด | ต้องเขียนเอง (`INPUT`/`OUTPUT PROCEDURE`) | มีในตัว (`INCLUDE`/`OMIT`, `SUM`) | มีในตัว (`SELECT`, `STATS`) มากกว่า DFSORT อีก |
| ความยืดหยุ่นด้าน business logic | สูงสุด (เขียนตรรกะอะไรก็ได้) | จำกัด (เฉพาะเงื่อนไขเชิงเปรียบเทียบ) | จำกัดคล้าย DFSORT |
| ประสิทธิภาพกับข้อมูลขนาดใหญ่มาก | ดี | ดีที่สุด (ปรับแต่งเฉพาะทาง) | ดีที่สุด (ใช้ DFSORT เป็นเครื่องยนต์) |
| ทดสอบบน GnuCOBOL ได้ไหม | **ได้** | ไม่ได้ | ไม่ได้ |

### หลักการเลือกใช้เครื่องมือที่เหมาะสม

1. **ถ้างานมีแค่เรียงลำดับ/กรอง/สรุปยอดล้วน ๆ โดยไม่มี business logic ซับซ้อน** → เลือก **DFSORT**
   (หรือ ICETOOL ถ้าต้องการความสามารถขั้นสูงกว่า เช่น `SELECT FIRST/LAST`, สถิติหลายมิติ)
2. **ถ้างานต้องมีการคำนวณทางธุรกิจที่ซับซ้อนปนอยู่ระหว่างการประมวลผล** (เช่น ต้องเรียกใช้ตาราง lookup,
   คำนวณดอกเบี้ยแบบมีเงื่อนไขหลายชั้น) → เลือกเขียนโปรแกรม COBOL ที่มี `SORT`
3. **ในทางปฏิบัติ ระบบจริงมักผสมทั้งสองแนวทางเข้าด้วยกัน**: ใช้ DFSORT สำหรับขั้นตอนเตรียมข้อมูล
   (เรียง/กรอง/สรุปเบื้องต้น) ก่อนส่งต่อให้โปรแกรม COBOL ที่มี business logic ซับซ้อนประมวลผลต่อ —
   แบ่งงานตามจุดแข็งของแต่ละเครื่องมือ (เชื่อมโยงกับแนวคิด Job Step Dependency จาก Part 066 ขั้นตอนที่
   656)

### บทเรียนสำคัญที่สุดของ Part นี้

แม้ DFSORT จะเป็นเครื่องมือที่ทรงพลังและใช้กันแพร่หลายในโลก Mainframe จริง แต่**แนวคิดพื้นฐานเบื้องหลัง
มันไม่ต่างจาก COBOL SORT ที่เรียนมาตั้งแต่ Part 027 เลย**: เรียงลำดับตาม Key, กรองข้อมูลก่อน/หลังเรียง,
และสรุปยอดของกลุ่มข้อมูล การเข้าใจแนวคิดเหล่านี้อย่างลึกซึ้งผ่าน COBOL ที่ทดสอบได้จริงตลอด Part 027
และ 066-067 ทำให้เมื่อต้องทำงานกับ DFSORT จริงบน Mainframe ในอนาคต จะสามารถอ่านและเขียน control
statement ได้อย่างเข้าใจ ไม่ใช่แค่ท่องจำไวยากรณ์โดยไม่รู้ว่าทำไมมันถึงทำงานแบบนั้น

### ข้อควรระวัง

- อย่าลืมว่า DFSORT/ICETOOL เป็นเทคโนโลยีเฉพาะของ z/OS Mainframe เท่านั้น หากในอนาคตต้องทำงาน
  Modernization ย้ายระบบไปยัง Cloud/Linux (เนื้อหาที่จะเรียนใน Part 076-077) ตรรกะที่เคยเขียนด้วย
  DFSORT control statement จะต้องถูกแปลงกลับมาเป็นโค้ดโปรแกรม (เช่น Python, Java, หรือ COBOL ที่มี
  `SORT`) เพราะ Cloud/Linux ไม่มี DFSORT ให้ใช้ — ความเข้าใจ COBOL SORT ที่ทดสอบได้จริงใน Part นี้
  จึงมีค่ามากสำหรับงาน Modernization ในอนาคตด้วย

### แบบฝึกหัดที่ 670.1

**โจทย์**: จงสรุปเป็นย่อหน้าสั้น ๆ ว่าทำไมการเรียนรู้ COBOL SORT statement อย่างลึกซึ้ง (Part 027,
066-067) จึงมีคุณค่าแม้ในอนาคตองค์กรจะเลือกใช้ DFSORT เป็นเครื่องมือหลักสำหรับงานเรียงลำดับข้อมูลจริง

**เฉลยแนวทาง**: เพราะแนวคิดพื้นฐานของการเรียงลำดับ กรอง และสรุปยอดข้อมูลเป็นแนวคิดสากลที่ใช้ได้กับทุก
เครื่องมือ ไม่ว่าจะเป็น COBOL SORT, DFSORT, หรือแม้แต่ฟังก์ชัน `sort()`/`GROUP BY` ในภาษาโปรแกรมสมัย
ใหม่ก็ตาม การเข้าใจตรรกะเหล่านี้อย่างลึกซึ้งผ่านการเขียนโค้ดที่ทดสอบได้จริง (ซึ่ง DFSORT ทำไม่ได้ใน
สภาพแวดล้อมการเรียนรู้ทั่วไป) ช่วยให้อ่านและเขียน control statement ของ DFSORT ได้อย่างเข้าใจความ
หมายที่แท้จริง ไม่ใช่แค่ท่องจำรูปแบบตัวเลข นอกจากนี้ ทักษะนี้ยังมีประโยชน์โดยตรงเมื่อองค์กรต้องทำ
Modernization ย้ายระบบออกจาก Mainframe ในอนาคต (Part 076-077) เพราะตรรกะที่เคยเขียนด้วย DFSORT
control statement จะต้องถูกแปลงกลับมาเป็นโค้ดโปรแกรมเสมอ และผู้ที่เข้าใจ COBOL SORT อย่างลึกซึ้งจะ
สามารถทำงานแปลงนี้ได้อย่างถูกต้องแม่นยำกว่าผู้ที่รู้จักแค่ไวยากรณ์ DFSORT เพียงอย่างเดียว

---

## สรุปท้ายบท

Part นี้พาคุณเจาะลึกเทคโนโลยียูทิลิตี้เรียงลำดับข้อมูลระดับองค์กรของ IBM ควบคู่ไปกับการนำความรู้
COBOL SORT จาก Part 027 และ 066 มาสร้างผลลัพธ์แบบเดียวกันเพื่อการทดสอบจริง:

- ความแตกต่างพื้นฐานระหว่าง **DFSORT** (ยูทิลิตี้ระดับ JCL) กับ **COBOL SORT** (คำสั่งภายในโปรแกรม)
- ไวยากรณ์ **`SORT FIELDS=`** สำหรับเรียงลำดับพื้นฐานและหลาย Key
- **`INCLUDE`/`OMIT COND=`** สำหรับกรองข้อมูล — พร้อมโปรแกรม COBOL เทียบเท่าที่ทดสอบจริง
- **`OUTFIL`** สำหรับแยกผลลัพธ์ออกหลายไฟล์ในการรันครั้งเดียว — พร้อมโปรแกรม COBOL เทียบเท่าที่ทดสอบ
  จริง
- **`SUM FIELDS=`** สำหรับยุบรวม record ที่มี Key ซ้ำกัน — พร้อมโปรแกรม COBOL เทียบเท่าที่ทดสอบจริง
- **ICETOOL** เบื้องต้น เครื่องมือเสริมที่ห่อรอบ DFSORT
- JCL สมบูรณ์สำหรับรันงาน DFSORT พร้อมเชื่อมโยงกับแนวคิด Condition Code จาก Part 052-053 และ 066
- ประเด็นด้านประสิทธิภาพ (Sort Work Dataset, Hardware-assisted Sorting)
- ตัวอย่างบูรณาการที่รวมทั้ง OMIT, SUM, และ INCLUDE เข้าด้วยกันในโปรแกรม COBOL เดียว พร้อมพิสูจน์
  ผลลัพธ์ตรงกับที่คำนวณด้วยมือทุกประการ
- ตารางสรุปเปรียบเทียบและหลักการเลือกใช้เครื่องมือที่เหมาะสมกับสถานการณ์

Part 067 นี้ปิดท้ายชุดเนื้อหาเรื่องเทคนิคการประมวลผล Batch ของหลักสูตร (Part 066-067) ต่อจากนี้
Part 068 จะพาเราไปสำรวจหัวข้อที่เชื่อมโยงกับทุก Part ที่ผ่านมาในเฟส 4: **Performance Tuning สำหรับ
Mainframe COBOL** ซึ่งจะรวมเทคนิคการปรับแต่งประสิทธิภาพทั้งในระดับโค้ด COBOL เอง (เช่น การเลือกใช้
`COMP-3`/Packed Decimal ที่ถูกกล่าวถึงในขั้นตอนที่ 662 ของ Part นี้) และระดับระบบ (เช่น การจัด buffer
ของไฟล์ VSAM ที่เรียนใน Part 055-056) เข้าด้วยกันเป็นภาพรวมที่สมบูรณ์

**[ไปยัง Part 068: Performance Tuning สำหรับ Mainframe COBOL →](part-068-performance-tuning.md)**
