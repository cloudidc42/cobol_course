# Part 024: FILE SECTION และ FD Entry แบบละเอียด (ขั้นตอนที่ 231–240)

## คำนำของ Part นี้

ใน Part 023 เราได้เรียนรู้วงจรชีวิตพื้นฐานของไฟล์ตามลำดับไปแล้ว: `SELECT`/`ASSIGN`, `FD`, `01` record,
`OPEN`/`WRITE`/`READ`/`CLOSE` แต่เรายังใช้ `FD` แบบง่ายที่สุดเท่านั้น (มักเป็น `PIC X(n)` ตัวเดียว หรือ
กลุ่ม field 2-3 ตัว) Part นี้จะพาคุณเจาะลึก **`FILE SECTION` และ `FD Entry`** อย่างละเอียดที่สุด
เพราะการออกแบบ record ที่ดีคือหัวใจของโปรแกรมประมวลผลไฟล์ทุกโปรแกรม

เนื้อหาที่คุณจะได้เรียนรู้ใน Part นี้ครอบคลุมตั้งแต่ clause ต่าง ๆ ที่ใช้ใน `FD`, การออกแบบ record
ด้วย group/elementary item, การใช้ `FILLER`, การมีหลาย record layout ใน `FD` เดียว, ไปจนถึง
**ข้อผิดพลาดที่พบบ่อยที่สุดข้อหนึ่งในโปรแกรมประมวลผลไฟล์จริง** นั่นคือปัญหาข้อมูลที่ยังไม่ได้กำหนดค่า
(Uninitialized Data) ในพื้นที่ record ซึ่งเราได้ทดสอบจริงจนพบพฤติกรรมที่น่าสนใจมากของ GnuCOBOL
ที่จะอธิบายละเอียดในขั้นตอนที่ 236–237

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน (คอลัมน์ 72 ของ
> Fixed-Format COBOL นับเป็นไบต์ไม่ใช่ตัวอักษร) โค้ดทุกตัวอย่างในนี้ผ่านการคอมไพล์และรันจริงด้วย
> GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง รวมถึงตัวอย่างที่ตั้งใจแสดง error จริง
> ก็ได้ทดสอบยืนยันข้อความ error และ error code จริงจากการรันด้วย

---

## ขั้นตอนที่ 231: ภาพรวม Clause ทั้งหมดของ FD Entry และ RECORD CONTAINS

### FD Entry คืออะไรกันแน่

`FD` (File Description) คือรายการที่อธิบาย**คุณลักษณะทางกายภาพ**ของไฟล์หนึ่งไฟล์ให้ compiler ทราบ
เช่น แต่ละ record มีขนาดเท่าไร มีป้ายกำกับ (label) แบบไหน ฯลฯ ตามด้วย `01` record description
ที่อธิบายว่า record นั้นมีโครงสร้างข้อมูลอย่างไร ไวยากรณ์เต็มรูปแบบของ `FD` มี clause ย่อยหลายตัว
ดังตารางต่อไปนี้:

| Clause | ความหมาย | สถานะในหลักสูตรนี้ |
|---|---|---|
| `RECORD CONTAINS n CHARACTERS` | กำหนดความยาว record คงที่เป็น n ไบต์ | ยังใช้บ้าง แต่ GnuCOBOL **เพิกเฉย (ignore)** clause นี้เมื่อ `ORGANIZATION IS LINE SEQUENTIAL` |
| `RECORD CONTAINS m TO n CHARACTERS` | ความยาว record แปรผันได้ระหว่าง m ถึง n ไบต์ | ใช้ร่วมกับ `RECORD IS VARYING` (ขั้นตอนที่ 235) |
| `RECORD IS VARYING IN SIZE ... DEPENDING ON` | ประกาศ record ความยาวแปรผันอย่างเป็นทางการ | สอนในขั้นตอนที่ 235 |
| `LABEL RECORDS ARE STANDARD/OMITTED` | ระบุว่าไฟล์มี label record กำกับหรือไม่ (แนวคิดจากยุคเทปแม่เหล็ก) | **Obsolete** ตั้งแต่ COBOL-2002 แต่ GnuCOBOL ยอมรับเพื่อ backward compatibility |
| `BLOCK CONTAINS n RECORDS` | บอกจำนวน record ต่อ physical block บนสื่อบันทึก (เทป/ดิสก์) | เป็นแนวคิดระดับ Mainframe ที่ GnuCOBOL ไม่บังคับใช้จริง (สอนใน 238) |
| `VALUE OF FILE-ID IS ...` | ผูกชื่อไฟล์แบบไดนามิกในยุค COBOL-74 | **Obsolete** สอนใน 238 |
| `DATA RECORDS ARE ...` | ระบุชื่อ record ทั้งหมดที่อยู่ใน FD นี้ | **Obsolete** สอนใน 238 |
| `CODE-SET IS ...` | กำหนด character encoding (เช่น EBCDIC) | ใช้เฉพาะงาน Mainframe/cross-platform ขั้นสูง (กล่าวถึงในเฟส 4) |

อย่ากังวลถ้าตารางนี้ดูเยอะ — ในทางปฏิบัติ 90% ของโปรแกรมสมัยใหม่ (รวมถึงตลอดหลักสูตรนี้) ใช้แค่
`RECORD CONTAINS` เป็นหลัก ส่วน clause ที่เหลือเป็นเรื่องประวัติศาสตร์ที่ควร**รู้จักไว้**เพื่ออ่านโค้ด
Legacy บน Mainframe ให้เข้าใจ (จะสอนละเอียดในขั้นตอนที่ 238)

### ตัวอย่าง: ประกาศ RECORD CONTAINS อย่างชัดเจน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP231-FD-BASICS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ITEM-FILE ASSIGN TO "ITEM.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ITEM-FILE
           RECORD CONTAINS 20 CHARACTERS.
       01  ITEM-RECORD                 PIC X(20).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT ITEM-FILE.
           MOVE "SUGAR 1KG" TO ITEM-RECORD.
           WRITE ITEM-RECORD.
           CLOSE ITEM-FILE.
           DISPLAY "ITEM.DAT created (record size 20 characters).".
           STOP RUN.
```

**ผลลัพธ์:**

```
ITEM.DAT created (record size 20 characters).
```

**เนื้อหาไฟล์ `ITEM.DAT`** (ตรวจสอบด้วย `cat -A ITEM.DAT` เพื่อดูอักขระท้ายบรรทัด):

```
SUGAR 1KG$
```

(สัญลักษณ์ `$` คือจุดสิ้นสุดบรรทัด `\n` ที่ `cat -A` แสดงให้เห็น)

### สิ่งที่น่าสนใจ: RECORD CONTAINS ถูกเพิกเฉยสำหรับ LINE SEQUENTIAL

ถ้าคอมไพล์โปรแกรมข้างต้นด้วยตัวเลือก `-Wall` (เปิดคำเตือนทั้งหมด) จะเห็นข้อความสำคัญ:

```bash
cobc -x -o step231 step231.cob -Wall
```

```
step231.cob:16: warning: RECORD clause ignored for LINE SEQUENTIAL
```

นี่คือข้อเท็จจริงสำคัญที่ต้องจำ: เมื่อใช้ `ORGANIZATION IS LINE SEQUENTIAL` (ซึ่งเป็นค่าเริ่มต้นของ
หลักสูตรนี้ตาม Part 023) **ความยาว record จริงถูกกำหนดโดยผลรวมของ field ทั้งหมดใน `01` เท่านั้น**
ไม่ใช่ตัวเลขที่เขียนไว้ใน `RECORD CONTAINS` เพราะไฟล์แบบบรรทัดข้อความไม่มีแนวคิด "ความยาว record คงที่"
จริง ๆ (แต่ละบรรทัดยาวเท่าที่มันเป็น) ตัวเลขใน `RECORD CONTAINS` จะมีผลจริงก็ต่อเมื่อใช้
`ORGANIZATION IS SEQUENTIAL` (แบบไบนารี ไม่ใช่ LINE) หรือ `INDEXED`/`RELATIVE` ที่จะเรียนใน Part 028–029

### ข้อควรระวัง

- อย่าพึ่งพา `RECORD CONTAINS` เพื่อ "บังคับ" ความยาว record เมื่อใช้ `LINE SEQUENTIAL` — ความยาวจริง
  มาจากผลรวม `PIC` clause ของ field ทั้งหมดใน `01` level เสมอ
- การเขียนตัวเลขใน `RECORD CONTAINS` ผิดจากความจริง (เช่นเขียน 20 ทั้งที่ field รวมกันได้ 15) จะไม่ทำให้
  compile error เมื่อใช้ LINE SEQUENTIAL แต่เป็นการเขียนโค้ดที่ทำให้คนอ่านสับสนภายหลัง ควรลบทิ้งหรือ
  เขียนให้ตรงกับความจริงเสมอ

### แบบฝึกหัดที่ 231.1

**โจทย์**: จงอธิบายว่าทำไม `RECORD CONTAINS n CHARACTERS` จึงยังมีประโยชน์ทางเอกสาร (documentation)
แม้ compiler จะเพิกเฉยต่อมันเมื่อใช้ `LINE SEQUENTIAL`

**เฉลย**: แม้ compiler จะไม่บังคับใช้ค่านี้จริงกับ `LINE SEQUENTIAL` แต่การเขียน `RECORD CONTAINS`
ไว้ชัดเจนช่วยให้**คนอ่านโค้ด** (รวมถึงตัวเราเองในอนาคต) ทราบทันทีว่า record ควรมีความยาวเท่าไรโดย
ไม่ต้องนับผลรวม `PIC` ของทุก field เอง เป็นเอกสารประกอบโค้ด (self-documentation) ที่มีประโยชน์
โดยเฉพาะเมื่อ record มี field จำนวนมาก อีกทั้งเมื่อย้ายโค้ดไปใช้กับ `ORGANIZATION IS SEQUENTIAL`
หรือ `INDEXED` ในอนาคต ค่านี้จะกลับมามีผลจริงทันที

---

## ขั้นตอนที่ 232: ออกแบบ Record ด้วย Group Item, Elementary Item และ Level Number

### ทบทวน: Group Item คืออะไร

จาก Part 005 เราทราบว่า `01` level เป็น record หรือกลุ่มข้อมูลระดับบนสุดเสมอ ส่วน level number
ที่มากกว่า (เช่น 05, 10, 15, ...) ใช้แบ่งย่อยข้อมูลภายใน record นั้น **Group Item** คือ item ที่มี
item ย่อยอยู่ภายใน (ไม่มี `PIC` ของตัวเอง) ในขณะที่ **Elementary Item** คือ item ที่มี `PIC` clause
ของตัวเองโดยตรง (ไม่มีลูกอยู่ข้างใน) หลักการนี้สำคัญมากเมื่อออกแบบ `FD` record ที่ซับซ้อน

### ตัวอย่าง: Record พนักงานที่มีกลุ่มข้อมูลซ้อนกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP232-GROUP-RECORD.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT EMP-FILE ASSIGN TO "EMP232.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  EMP-FILE.
       01  EMP-RECORD.
           05  EMP-ID                  PIC 9(5).
           05  EMP-NAME.
               10  EMP-FIRST-NAME      PIC X(10).
               10  EMP-LAST-NAME       PIC X(10).
           05  EMP-DEPT                PIC X(8).
           05  EMP-SALARY              PIC 9(6)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT EMP-FILE.
           MOVE 10023 TO EMP-ID.
           MOVE "SOMCHAI   " TO EMP-FIRST-NAME.
           MOVE "JAIDEE    " TO EMP-LAST-NAME.
           MOVE "SALES   " TO EMP-DEPT.
           MOVE 28500.50 TO EMP-SALARY.
           WRITE EMP-RECORD.
           CLOSE EMP-FILE.

           DISPLAY "Full name written as one group: " EMP-NAME.
           DISPLAY "Whole record as one group     : " EMP-RECORD.
           STOP RUN.
```

**ผลลัพธ์:**

```
Full name written as one group: SOMCHAI   JAIDEE
Whole record as one group     : 10023SOMCHAI   JAIDEE    SALES   02850050
```

**เนื้อหาไฟล์ `EMP232.DAT`:**

```
10023SOMCHAI   JAIDEE    SALES   02850050
```

### อธิบายโครงสร้างซ้อนกัน

- `EMP-RECORD` (level 01) คือ Group Item ระดับบนสุด — ไม่มี `PIC` ของตัวเอง
- `EMP-ID`, `EMP-DEPT`, `EMP-SALARY` (level 05) เป็น **Elementary Item** เพราะมี `PIC` โดยตรง
- `EMP-NAME` (level 05) เป็น **Group Item ซ้อนอยู่ภายใน Group Item อีกที** เพราะไม่มี `PIC` แต่มี
  `EMP-FIRST-NAME` และ `EMP-LAST-NAME` (level 10) เป็นลูกอยู่ข้างใน
- เมื่อ `DISPLAY EMP-NAME` เราได้ค่ารวมของทั้ง `EMP-FIRST-NAME` และ `EMP-LAST-NAME` ต่อกัน เพราะ
  Group Item เมื่อถูกอ้างอิงจะทำงานเป็น **Alphanumeric ตัวเดียวที่มีความยาวเท่ากับผลรวมของลูกทั้งหมด**
  เสมอ ไม่ว่าลูกภายในจะเป็นชนิดตัวเลขหรือตัวอักษรก็ตาม

### ข้อควรระวัง

- Level number ไม่จำเป็นต้องเรียงติดกันทีละ 1 (เช่น 05, 10, 15 ก็ได้ ไม่ต้องเป็น 01, 02, 03) นิยม
  เว้นช่วงทีละ 5 เพื่อเผื่อแทรก field ใหม่ภายหลังโดยไม่ต้องเปลี่ยนเลขทั้งหมด
- ห้ามใส่ `PIC` ให้กับ Group Item เด็ดขาด (เช่น `05 EMP-NAME PIC X(20).` แล้วมี `10` ซ้อนข้างใน)
  compiler จะรายงาน syntax error ทันที เพราะ Group Item ต้องไม่มี `PIC` ของตัวเอง

### แบบฝึกหัดที่ 232.1

**โจทย์**: จากตัวอย่างข้างต้น จงคำนวณด้วยมือว่า `EMP-RECORD` ทั้งก้อนมีความยาวรวมกี่ไบต์
(โดยไม่ต้องรันโปรแกรม) แล้วตรวจสอบคำตอบด้วย `DISPLAY LENGTH OF EMP-RECORD`

**เฉลย**: `EMP-ID` (5) + `EMP-FIRST-NAME` (10) + `EMP-LAST-NAME` (10) + `EMP-DEPT` (8) +
`EMP-SALARY` (8, เพราะ `9(6)V99` มี 6+2 = 8 หลัก) รวมทั้งหมด = 5+10+10+8+8 = **41 ไบต์**
ตรวจสอบได้ด้วยการเพิ่มบรรทัด `DISPLAY LENGTH OF EMP-RECORD.` ในโปรแกรม ซึ่งจะแสดงผลเป็น `000000041`

---

## ขั้นตอนที่ 233: FILLER Clause — จองพื้นที่โดยไม่ต้องตั้งชื่อ

### ทำไมต้องมี FILLER

บางครั้ง record ของเรามีช่องว่างที่**ไม่จำเป็นต้องอ้างอิงถึงโดยตรง**ในโปรแกรม เช่น พื้นที่เผื่อไว้
สำหรับอนาคต หรือช่องว่างคั่นระหว่าง field เพื่อให้อ่านง่ายเวลาเปิดไฟล์ดิบด้วยตา (fixed-width format)
COBOL ให้เราใช้คำสงวน `FILLER` แทนชื่อ field ในกรณีเหล่านี้ — field ที่ชื่อ `FILLER`
**ไม่สามารถอ้างอิงถึงได้โดยตรง**ในส่วนอื่นของโปรแกรม (ไม่มี "ชื่อ" ให้เรียกใช้)

### ตัวอย่าง: Record รายงานที่มีช่องว่างคั่น

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP233-FILLER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "REPORT233.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE.
       01  REPORT-RECORD.
           05  RPT-CODE                PIC X(6).
           05  FILLER                  PIC X(4).
           05  RPT-AMOUNT              PIC 9(7)V99.
           05  FILLER                  PIC X(3).
           05  RPT-STATUS              PIC X(8).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT REPORT-FILE.
      *> Clear the whole record (including FILLER bytes) BEFORE
      *> moving values into the named fields. The reason this is
      *> mandatory is explained in full in step 236.
           MOVE SPACES TO REPORT-RECORD.
           MOVE "AB1234" TO RPT-CODE.
           MOVE 1500.75 TO RPT-AMOUNT.
           MOVE "ACTIVE  " TO RPT-STATUS.
           WRITE REPORT-RECORD.
           CLOSE REPORT-FILE.

           DISPLAY "Record with gaps: [" REPORT-RECORD "]".
           STOP RUN.
```

**ผลลัพธ์:**

```
Record with gaps: [AB1234    001500.75   ACTIVE  ]
```

**เนื้อหาไฟล์ `REPORT233.DAT`:**

```
AB1234    001500.75   ACTIVE
```

### อธิบายจุดสำคัญ

- ใช้คำว่า `FILLER` ซ้ำได้หลายครั้งในหนึ่ง record (ในตัวอย่างนี้มี 2 ตัว) โดยไม่ถือว่าชื่อซ้ำกัน
  เพราะ `FILLER` ไม่ใช่ "ชื่อ" ในความหมายของ COBOL — มันคือคำสงวนพิเศษที่บอกว่า "ห้ามเรียกใช้ field นี้"
- ตั้งแต่ COBOL-2002 เป็นต้นไป อันที่จริงสามารถ**ละคำว่า `FILLER` ไปเลยได้** (เขียนแค่
  `05  PIC X(4).` เฉย ๆ ก็ได้ผลเหมือนกัน) แต่หลักสูตรนี้จะเขียน `FILLER` ให้ชัดเจนเสมอเพื่อให้อ่านง่าย
  และเข้ากันได้กับ Mainframe รุ่นเก่าที่ยังไม่รองรับการละคำนี้
- สังเกตว่าเราใส่ `MOVE SPACES TO REPORT-RECORD` **ก่อน** MOVE ค่าลงแต่ละ field เสมอ — นี่คือ
  แนวปฏิบัติที่ดีเพื่อให้แน่ใจว่าไบต์ของ `FILLER` มีค่าเป็นช่องว่าง ไม่ใช่ขยะจากหน่วยความจำ (รายละเอียด
  เต็มพร้อมหลักฐานการทดลองจริงอยู่ในขั้นตอนที่ 236 ถัดไป)

### ข้อควรระวัง

- **ห้ามพยายามอ้างอิง `FILLER` โดยตรง** เช่น `MOVE "XX" TO FILLER.` — compiler จะรายงาน error ทันที
  เพราะ `FILLER` ไม่ใช่ identifier ที่ใช้อ้างอิงได้
- อย่าลืมว่า `FILLER` field ยัง**กินพื้นที่จริงในไฟล์** แม้จะไม่มีชื่อให้เรียก การคำนวณ
  `LENGTH OF record` ยังคงนับความยาวของ `FILLER` รวมเข้าไปด้วยเสมอ

### แบบฝึกหัดที่ 233.1

**โจทย์**: จงคำนวณความยาวรวมของ `REPORT-RECORD` ในตัวอย่างข้างต้น (รวม `FILLER` ทั้งสองตัวด้วย)

**เฉลย**: `RPT-CODE` (6) + `FILLER` (4) + `RPT-AMOUNT` (9, เพราะ `9(7)V99` = 7+2 หลัก) +
`FILLER` (3) + `RPT-STATUS` (8) รวมทั้งหมด = 6+4+9+3+8 = **30 ไบต์**

---

## ขั้นตอนที่ 234: หลาย Record Layout ใน FD เดียว (Shared Storage)

### แนวคิด: FD หนึ่งตัวมีได้มากกว่าหนึ่ง 01 Level

COBOL อนุญาตให้ `FD` หนึ่งตัวมี **record description ระดับ `01` มากกว่าหนึ่งตัว** ได้ ซึ่งมีประโยชน์มาก
เมื่อไฟล์เดียวมี record หลายรูปแบบปะปนกัน (เช่น record ประเภท Header กับประเภท Detail อยู่ในไฟล์
เดียวกัน) จุดสำคัญที่ต้องเข้าใจคือ **record ทั้งหมดภายใต้ `FD` เดียวกันใช้พื้นที่หน่วยความจำร่วมกัน
(Shared Storage)** เหมือนกับการทำ `REDEFINES` โดยปริยาย (ทบทวน `REDEFINES` จาก Part 022)

### ตัวอย่าง: ไฟล์ที่มี Header Record และ Detail Record ปนกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP234-MULTI-LAYOUT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MIX-FILE ASSIGN TO "MIX234.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MIX-FILE.
       01  HEADER-RECORD.
           05  HDR-TAG                 PIC X(1) VALUE "H".
           05  HDR-COMPANY-NAME        PIC X(20).
           05  HDR-REPORT-DATE         PIC X(8).
       01  DETAIL-RECORD.
           05  DTL-TAG                 PIC X(1) VALUE "D".
           05  DTL-ITEM-CODE           PIC X(6).
           05  DTL-QTY                 PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-RECORD-SIZE-HEADER       PIC 9(3).
       01  WS-RECORD-SIZE-DETAIL       PIC 9(3).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> HEADER-RECORD and DETAIL-RECORD share the SAME physical
      *> storage area because they both belong to the same FD.
           MOVE LENGTH OF HEADER-RECORD TO WS-RECORD-SIZE-HEADER.
           MOVE LENGTH OF DETAIL-RECORD TO WS-RECORD-SIZE-DETAIL.
           DISPLAY "LENGTH OF HEADER-RECORD = "
               WS-RECORD-SIZE-HEADER.
           DISPLAY "LENGTH OF DETAIL-RECORD = "
               WS-RECORD-SIZE-DETAIL.

           OPEN OUTPUT MIX-FILE.
           MOVE "H" TO HDR-TAG.
           MOVE "ACME TRADING CO."  TO HDR-COMPANY-NAME.
           MOVE "20260101" TO HDR-REPORT-DATE.
           WRITE HEADER-RECORD.

           MOVE "D" TO DTL-TAG.
           MOVE "IT9001" TO DTL-ITEM-CODE.
           MOVE 150 TO DTL-QTY.
           WRITE DETAIL-RECORD.
           CLOSE MIX-FILE.

           DISPLAY "File MIX234.DAT written with 2 layouts.".
           STOP RUN.
```

**ผลลัพธ์:**

```
LENGTH OF HEADER-RECORD = 029
LENGTH OF DETAIL-RECORD = 012
File MIX234.DAT written with 2 layouts.
```

**เนื้อหาไฟล์ `MIX234.DAT`:**

```
HACME TRADING CO.    20260101
DIT900100150
```

### อธิบายจุดสำคัญ

- `HEADER-RECORD` และ `DETAIL-RECORD` เป็น**คนละชื่อ คนละความยาว** (29 กับ 12 ไบต์) แต่ทั้งคู่ผูกกับ
  `FD MIX-FILE` เดียวกัน เมื่อเรียก `WRITE HEADER-RECORD` COBOL จะเขียนตามความยาวและโครงสร้างของ
  `HEADER-RECORD` ณ ขณะนั้น และเมื่อเรียก `WRITE DETAIL-RECORD` ก็จะเขียนตามโครงสร้างของ
  `DETAIL-RECORD` แทน — พื้นที่หน่วยความจำใช้ร่วมกัน แต่**การตีความ**เปลี่ยนไปตาม record ที่ระบุใน
  คำสั่งขณะนั้น
- เทคนิคนี้เป็นที่นิยมมากในระบบ Batch จริง เช่น ไฟล์ Transaction ที่มี record บรรทัดแรกเป็น Header
  สรุปข้อมูล แล้วตามด้วย record รายละเอียดหลายพันบรรทัด — เมื่ออ่านกลับมา โปรแกรมจะเช็ค field แรก
  (`HDR-TAG`/`DTL-TAG`) เพื่อตัดสินใจว่า record นี้ควรตีความด้วย layout ไหน (เทคนิคนี้จะฝึกใช้จริง
  ใน Part 026 เรื่อง Master-Detail Processing)

### ข้อควรระวัง

- ห้ามสับสนว่า record ทั้งสองจะถูก "รวมกัน" เป็นก้อนเดียว — พวกมัน**ซ้อนทับ (overlay)** พื้นที่กัน
  ไม่ใช่ต่อกัน ถ้า `WRITE HEADER-RECORD` แล้วรีบไป `MOVE` ค่าใหม่ลง `DETAIL-RECORD` ทันที ค่าฝั่ง
  `HEADER-RECORD` เดิมอาจถูกทับไปแล้ว (แต่ในตัวอย่างนี้ปลอดภัยเพราะเรา `WRITE` ก่อนจะย้ายไปใช้อีกอัน)
- เมื่ออ่านไฟล์ประเภทนี้กลับมา ต้อง `READ` เข้า record ตัวใดตัวหนึ่งเท่านั้น (COBOL จะเลือกใช้ชื่อไฟล์
  `MIX-FILE` ใน `READ MIX-FILE` แล้วเราต้องตรวจ field แรกเองว่าควรตีความด้วย layout ไหน)

### แบบฝึกหัดที่ 234.1

**โจทย์**: จงอธิบายว่าทำไมการออกแบบ `HDR-TAG` และ `DTL-TAG` ให้อยู่ที่ตำแหน่งไบต์แรกสุดของทั้งสอง
record จึงเป็นการออกแบบที่ดี

**เฉลย**: เพราะเมื่ออ่าน record กลับมาโดยไม่รู้ล่วงหน้าว่าเป็น Header หรือ Detail โปรแกรมสามารถ
ตรวจสอบไบต์แรกสุดของ record (ซึ่งตำแหน่งเดียวกันในทั้งสอง layout) เพื่อตัดสินใจได้ทันทีว่าควร
ตีความ record ที่เหลือด้วย layout ใด หากตำแหน่ง tag ไม่ตรงกันระหว่างสอง layout จะทำให้ตรรกะ
การตรวจสอบซับซ้อนขึ้นมาก หรืออาจตรวจสอบผิดพลาดได้

---

## ขั้นตอนที่ 235: RECORD CONTAINS แบบช่วง และ RECORD IS VARYING

### ปัญหา: record ที่มีความยาวไม่คงที่

บางสถานการณ์ record แต่ละแถวมีความยาวไม่เท่ากันจริง ๆ เช่น บันทึกข้อความ (note) ที่สั้นบ้างยาวบ้าง
COBOL มี clause `RECORD IS VARYING IN SIZE ... DEPENDING ON` สำหรับประกาศ record ความยาวแปรผัน
อย่างเป็นทางการ โดยต้องมีตัวแปรตัวเลขใน `WORKING-STORAGE` คอยเก็บความยาวปัจจุบันก่อนเขียน

### ตัวอย่าง: บันทึกข้อความความยาวไม่คงที่

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP235-VARYING-RECORD.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT NOTE-FILE ASSIGN TO "NOTE235.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  NOTE-FILE
           RECORD IS VARYING IN SIZE FROM 5 TO 50 CHARACTERS
           DEPENDING ON WS-NOTE-LENGTH.
       01  NOTE-RECORD                 PIC X(50).

       WORKING-STORAGE SECTION.
       01  WS-NOTE-LENGTH               PIC 9(3).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT NOTE-FILE.

           MOVE "HI" TO NOTE-RECORD.
           MOVE 2 TO WS-NOTE-LENGTH.
           WRITE NOTE-RECORD.

           MOVE "THIS IS A LONGER NOTE TODAY" TO NOTE-RECORD.
           MOVE 27 TO WS-NOTE-LENGTH.
           WRITE NOTE-RECORD.

           CLOSE NOTE-FILE.
           DISPLAY "Wrote 2 records with varying lengths.".
           STOP RUN.
```

**ผลลัพธ์:**

```
Wrote 2 records with varying lengths.
```

**เนื้อหาไฟล์ `NOTE235.DAT`:**

```
HI
THIS IS A LONGER NOTE TODAY
```

### อธิบายจุดสำคัญ

- `WS-NOTE-LENGTH` **ต้องถูกกำหนดค่าก่อนทุกครั้งที่ `WRITE`** เพื่อบอก COBOL ว่า record รอบนี้ยาว
  เท่าไร (ในตัวอย่างนี้คือ 2 และ 27 ตามลำดับ)
- สำหรับ `LINE SEQUENTIAL` โดยเฉพาะ ความยาวจริงของแต่ละบรรทัดถูกกำหนดโดยการตัดช่องว่างท้ายอัตโนมัติ
  อยู่แล้ว (ตามที่เรียนใน Part 023 ขั้นตอน 222) ดังนั้น `RECORD IS VARYING` จะมีผลชัดเจนกว่ามาก
  เมื่อใช้กับ `ORGANIZATION IS SEQUENTIAL` (แบบไบนารี) หรือ `INDEXED`/`RELATIVE` ที่ไม่มีการตัด
  ช่องว่างอัตโนมัติ — แต่ก็ยังเป็น clause ที่ควรรู้จักไว้เพราะพบได้บ่อยในโค้ด Mainframe จริง
- `PIC X(50)` ใน `01 NOTE-RECORD` คือความยาว**สูงสุด**ที่ record จะมีได้ (ตรงกับตัวเลข `TO 50` ใน
  `RECORD IS VARYING`)

### ข้อควรระวัง

- ถ้า `WS-NOTE-LENGTH` มีค่ามากกว่าความยาวสูงสุดที่ประกาศไว้ (`TO 50`) หรือน้อยกว่าค่าต่ำสุด
  (`FROM 5`) พฤติกรรมจะขึ้นกับ compiler แต่ละตัว บางกรณีอาจตัดข้อมูลทิ้งเงียบ ๆ ควรตรวจสอบค่าก่อน
  กำหนดเสมอในโปรแกรมจริง
- ต้อง `MOVE` ความยาวเข้า `WS-NOTE-LENGTH` **ก่อน** `WRITE` เสมอ ไม่ใช่หลัง มิฉะนั้นจะใช้ค่าความยาว
  จากรอบก่อนหน้าโดยไม่ตั้งใจ

### แบบฝึกหัดที่ 235.1

**โจทย์**: จงอธิบายว่าทำไมตัวอย่างนี้จึงยังต้องกำหนดความยาวสูงสุดด้วย `PIC X(50)` ทั้งที่ประกาศ
`RECORD IS VARYING` ไปแล้ว

**เฉลย**: `RECORD IS VARYING` เป็นเพียงการบอก COBOL ว่า "ความยาว record จริงจะแปรผันได้ และให้ดูค่า
จากตัวแปร `DEPENDING ON`" แต่หน่วยความจำที่จัดสรรให้ `01 NOTE-RECORD` ยังคงเป็นพื้นที่คงที่ขนาดสูงสุด
ที่ระบุด้วย `PIC` เสมอ (ในที่นี้คือ 50 ไบต์) เพราะ COBOL ต้องจองพื้นที่ RAM ไว้ล่วงหน้าให้พอสำหรับ
กรณีที่ยาวที่สุดที่เป็นไปได้ ส่วนตัวเลขใน `DEPENDING ON` จะบอกแค่ว่า**จำนวนไบต์ที่จะเขียนจริง**ลงไฟล์
รอบนั้นมีกี่ไบต์เท่านั้น

---

## ขั้นตอนที่ 236: ข้อผิดพลาดสำคัญ — Uninitialized Data ใน FILLER และ Record Area

### การทดลองจริง: เมื่อลืม MOVE SPACES ก่อนเขียนไฟล์

นี่คือขั้นตอนที่สำคัญที่สุดขั้นตอนหนึ่งใน Part นี้ ลองสังเกตโปรแกรมต่อไปนี้ที่**ดูเหมือนถูกต้องทุก
ประการ** (มี `FD`, มี `FILLER`, มี `MOVE`/`WRITE` ตามปกติ) แต่**ไม่มี** `MOVE SPACES TO ...` ก่อน:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP236-NO-INIT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "REPORT236.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE.
       01  REPORT-RECORD.
           05  RPT-CODE                PIC X(6).
           05  FILLER                  PIC X(4).
           05  RPT-AMOUNT              PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT REPORT-FILE.
           MOVE "AB1234" TO RPT-CODE.
           MOVE 1500.75 TO RPT-AMOUNT.
      *> Notice: FILLER was never given a value here!
           WRITE REPORT-RECORD.
           CLOSE REPORT-FILE.
           DISPLAY "OK".
           STOP RUN.
```

เมื่อคอมไพล์และรันโปรแกรมนี้จริง (`cobc -x -o step236 step236.cob && ./step236`) ผลลัพธ์ที่ได้
**ไม่ใช่** `OK` แต่กลับเป็นข้อความ error และโปรแกรม**ล่มทันที**:

```
libcob: error: unknown file error (status = 71) for file REPORT-FILE ('REPORT236.DAT')
libcob: warning: implicit CLOSE of REPORT-FILE ('REPORT236.DAT')
```

### เกิดอะไรขึ้น — วิเคราะห์สาเหตุที่แท้จริง

เมื่อประกาศ `05 FILLER PIC X(4).` โดยไม่มี `VALUE` clause กำกับ COBOL **จะไม่กำหนดค่าเริ่มต้นใด ๆ**
ให้กับ field นี้ พื้นที่หน่วยความจำ 4 ไบต์นั้นจึงมีค่าเป็น**ขยะที่หลงเหลือจากการทำงานก่อนหน้าของ
โปรแกรม** (garbage/leftover bytes) ซึ่งอาจเป็นอักขระควบคุม (control character) ที่ไม่ใช่ตัวอักษร
ที่พิมพ์ได้ เมื่อ COBOL พยายาม `WRITE` record นี้ลงไฟล์แบบ `LINE SEQUENTIAL` (ซึ่งคาดหวังว่าเนื้อหา
เป็นข้อความล้วน) แล้วเจอไบต์ขยะที่ไม่ใช่ข้อความปกติปะปนอยู่ใน `FILLER` มันจะปฏิเสธการเขียนและคืนค่า
**File Status 71** (ข้อผิดพลาดเกี่ยวกับข้อมูลของไฟล์ที่ไม่ถูกต้อง — รายละเอียดเต็มของ File Status
Code ทั้งหมดจะสอนใน Part 030) นี่คือหลักฐานจริงที่ยืนยันว่า**ข้อมูลที่ยังไม่ได้กำหนดค่าใน record
area เป็นอันตรายจริง ไม่ใช่แค่ทฤษฎี**

### ทางแก้ที่ถูกต้อง: MOVE SPACES ก่อนเสมอ

```cobol
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT REPORT-FILE.
      *> Clear the ENTIRE record area (including FILLER bytes)
      *> to spaces BEFORE moving any real values into it.
           MOVE SPACES TO REPORT-RECORD.
           MOVE "AB1234" TO RPT-CODE.
           MOVE 1500.75 TO RPT-AMOUNT.
           WRITE REPORT-RECORD.
           CLOSE REPORT-FILE.
           DISPLAY "OK".
           STOP RUN.
```

เมื่อรันโปรแกรมที่แก้ไขแล้ว ผลลัพธ์จะเป็น `OK` และไฟล์ถูกสร้างขึ้นถูกต้อง เพราะ `MOVE SPACES TO
REPORT-RECORD` เป็นการ MOVE ระดับ**กลุ่ม (Group MOVE)** ที่เติมช่องว่างให้กับ**ทุกไบต์**ใน record
รวมถึงไบต์ของ `FILLER` ด้วย ก่อนที่เราจะ `MOVE` ค่าจริงทับลงไปเฉพาะ field ที่ต้องการ

### ข้อควรระวัง (สำคัญที่สุดของ Part นี้)

- **กฎทอง**: ทุกครั้งที่ `01` record มี `FILLER` (หรือ field ใดก็ตามที่อาจไม่ได้ถูก `MOVE` ค่าในทุก
  รอบ) ให้ `MOVE SPACES TO record-name` (หรือ `MOVE LOW-VALUES` แล้วแต่กรณี) **ก่อน** MOVE ค่าจริง
  เข้า field ต่าง ๆ เสมอ โดยเฉพาะก่อน `WRITE` ไปยังไฟล์ `LINE SEQUENTIAL`
- ปัญหานี้จะรุนแรงขึ้นเมื่อ `WRITE` record เดียวกัน**หลายรอบในลูป** โดยไม่ initialize ใหม่ทุกรอบ —
  รอบแรกอาจไม่มีปัญหาเพราะข้อมูลเก่าบังเอิญเป็นค่าที่ใช้ได้ แต่พอรันบนเครื่องอื่นหรือสถานการณ์อื่น
  หน่วยความจำเริ่มต้นอาจต่างกัน ทำให้บั๊กแสดงตัวแบบไม่แน่นอน (Intermittent Bug) ซึ่งเป็นบั๊กที่ตาม
  แก้ยากที่สุดประเภทหนึ่ง
- ในขั้นตอนถัดไป (237) เราจะเรียนรู้คำสั่ง `INITIALIZE` ซึ่ง**ดูเหมือน**จะแก้ปัญหานี้ได้เช่นกัน แต่
  มีข้อยกเว้นสำคัญที่ต้องระวังอย่างยิ่งเกี่ยวกับ `FILLER` โดยเฉพาะ

### แบบฝึกหัดที่ 236.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม `STEP236-NO-INIT` อาจ**ทำงานได้ปกติในบางครั้ง**และ**ล่มในบางครั้ง**
เมื่อรันซ้ำ ๆ กันหลายรอบบนเครื่องเดียวกัน (โดยไม่แก้โค้ดเลย)

**เฉลย**: เพราะค่าที่หลงเหลืออยู่ใน `FILLER` (ซึ่งไม่ได้ถูกกำหนดค่า) มาจากหน่วยความจำที่ระบบปฏิบัติการ
จัดสรรให้ process ณ ขณะนั้น ค่าที่หลงเหลือนี้**ไม่แน่นอน** ขึ้นกับสถานะหน่วยความจำของเครื่องในขณะรัน
แต่ละครั้ง (เช่น โปรแกรมอื่นที่เพิ่งใช้หน่วยความจำตำแหน่งเดียวกันก่อนหน้านี้ทิ้งค่าอะไรไว้) บางรอบอาจ
บังเอิญได้ค่าที่เป็นอักขระพิมพ์ได้ (จึงผ่านการ `WRITE` ไปได้) แต่บางรอบอาจได้อักขระควบคุมที่ทำให้เกิด
File Status 71 นี่คือเหตุผลที่ไม่ควรพึ่งพา "มันเคยรันผ่านมาก่อน" เป็นหลักฐานว่าโค้ดถูกต้อง —
ต้อง `MOVE SPACES`/`INITIALIZE` ให้ชัดเจนเสมอ ไม่ปล่อยให้หน่วยความจำอยู่ในสถานะไม่แน่นอน

---

## ขั้นตอนที่ 237: คำสั่ง INITIALIZE โดยละเอียด และข้อยกเว้นเรื่อง FILLER

### ไวยากรณ์พื้นฐานของ INITIALIZE

`INITIALIZE` เป็นคำสั่งที่ออกแบบมาเพื่อกำหนดค่าเริ่มต้นให้ทุก elementary item ภายใน group item
กลับไปเป็นค่า**เริ่มต้นมาตรฐาน**ในครั้งเดียว โดยไม่ต้อง `MOVE` ทีละ field ค่าเริ่มต้นมาตรฐานคือ
**ช่องว่าง (SPACES)** สำหรับข้อมูลตัวอักษร และ **ศูนย์ (ZERO)** สำหรับข้อมูลตัวเลข

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP237-INITIALIZE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-RECORD.
           05  WS-CUST-ID              PIC 9(5) VALUE 99.
           05  WS-CUST-NAME             PIC X(15) VALUE "OLD NAME".
           05  WS-CUST-BALANCE          PIC 9(7)V99 VALUE 500.25.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Before INITIALIZE:".
           DISPLAY "  ID      = " WS-CUST-ID.
           DISPLAY "  NAME    = [" WS-CUST-NAME "]".
           DISPLAY "  BALANCE = " WS-CUST-BALANCE.

           INITIALIZE WS-CUSTOMER-RECORD.

           DISPLAY "After INITIALIZE (numeric->0, text->spaces):".
           DISPLAY "  ID      = " WS-CUST-ID.
           DISPLAY "  NAME    = [" WS-CUST-NAME "]".
           DISPLAY "  BALANCE = " WS-CUST-BALANCE.

           INITIALIZE WS-CUSTOMER-RECORD
               REPLACING ALPHANUMERIC DATA BY "N/A"
                         NUMERIC DATA BY 999.

           DISPLAY "After INITIALIZE ... REPLACING:".
           DISPLAY "  ID      = " WS-CUST-ID.
           DISPLAY "  NAME    = [" WS-CUST-NAME "]".
           DISPLAY "  BALANCE = " WS-CUST-BALANCE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Before INITIALIZE:
  ID      = 00099
  NAME    = [OLD NAME       ]
  BALANCE = 0000500.25
After INITIALIZE (numeric->0, text->spaces):
  ID      = 00000
  NAME    = [               ]
  BALANCE = 0000000.00
After INITIALIZE ... REPLACING:
  ID      = 00999
  NAME    = [N/A            ]
  BALANCE = 0000999.00
```

### อธิบาย REPLACING Option

`INITIALIZE ... REPLACING` ให้เรากำหนดค่าเริ่มต้น**แบบกำหนดเองแยกตามชนิดข้อมูล**แทนค่ามาตรฐาน
(spaces/zero) ได้ โดยระบุประเภทข้อมูล (`ALPHANUMERIC DATA`, `NUMERIC DATA` เป็นต้น) ตามด้วยค่าที่
ต้องการ COBOL จะไล่ตรวจทุก elementary item ภายใน group แล้วเลือกใช้ค่าที่ตรงกับชนิดข้อมูลของ
field นั้น ๆ โดยอัตโนมัติ — มีประโยชน์มากเมื่อต้องการรีเซ็ต record ให้เป็นค่า "ยังไม่มีข้อมูล" ที่
ไม่ใช่ spaces/zero ธรรมดา เช่น `"N/A"` หรือ `-1`

### ข้อยกเว้นสำคัญที่สุด: INITIALIZE ไม่แตะต้อง FILLER

นี่คือจุดที่อันตรายที่สุดของ `INITIALIZE` และเป็นสิ่งที่พิสูจน์แล้วจากการทดลองจริง ลองดูโปรแกรม
ทดสอบสั้น ๆ ต่อไปนี้:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FILLERTEST.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-GROUP.
           05  WS-A       PIC X(5) VALUE "AAAAA".
           05  FILLER     PIC X(5) VALUE "BBBBB".
       PROCEDURE DIVISION.
       MAIN-PARA.
           INITIALIZE WS-GROUP.
           DISPLAY "[" WS-GROUP "]".
           STOP RUN.
```

**ผลลัพธ์ที่ทดสอบจริงด้วย GnuCOBOL:**

```
[     BBBBB]
```

สังเกตให้ดี: `WS-A` ถูกเปลี่ยนเป็นช่องว่างตามที่คาดหวัง (`     `) แต่ **`FILLER` ยังคงมีค่า
`"BBBBB"` เดิมอยู่ ไม่ถูก `INITIALIZE` แตะต้องเลย!** นี่ไม่ใช่บั๊กของ GnuCOBOL แต่เป็น**พฤติกรรมตาม
มาตรฐาน COBOL** ที่กำหนดไว้อย่างชัดเจนว่า **`INITIALIZE` จะข้าม (skip) field ที่ชื่อ `FILLER` โดย
ปริยายเสมอ** เหตุผลเชิงประวัติศาสตร์คือ `FILLER` ถูกออกแบบมาให้เป็น "พื้นที่ที่โปรแกรมไม่สนใจ" จึง
ไม่จำเป็นต้องเสียเวลาไป initialize มันทุกครั้ง

### สรุปเปรียบเทียบ: MOVE SPACES กับ INITIALIZE เมื่อมี FILLER

| คำสั่ง | ผลต่อ field ที่มีชื่อ | ผลต่อ FILLER |
|---|---|---|
| `MOVE SPACES TO record-name` | เซ็ตเป็นช่องว่างทั้งหมด (Group MOVE ไม่สนใจชนิดข้อมูลย่อย) | **เซ็ตเป็นช่องว่างด้วย** เพราะ MOVE ทำงานระดับไบต์ดิบทั้งก้อน |
| `INITIALIZE record-name` | เซ็ตเป็นค่ามาตรฐานตามชนิดข้อมูล (spaces/zero) | **ไม่ถูกแตะต้อง** ยังคงเป็นขยะเดิมจากหน่วยความจำ |

**ข้อสรุปเชิงปฏิบัติที่สำคัญที่สุด**: ถ้า record ของคุณมี `FILLER` ที่ยังไม่เคยมี `VALUE` มาก่อน
และคุณจะ `WRITE` record นั้นไปยังไฟล์ `LINE SEQUENTIAL` **ให้ใช้ `MOVE SPACES TO record-name`
แทน `INITIALIZE`** เพื่อความปลอดภัย (ตามที่ใช้แก้ปัญหาจริงในขั้นตอนที่ 236) หรือถ้าจำเป็นต้องใช้
`INITIALIZE` จริง ๆ ให้ตามด้วย `MOVE SPACES TO record-name` เพิ่มอีกบรรทัดเผื่อไว้เสมอ

### ข้อควรระวัง

- ข้อยกเว้นนี้ใช้กับคำว่า `FILLER` **ที่สะกดตรงตัว**เท่านั้น หากตั้งชื่อ field แทน `FILLER` (เช่น
  `05 RESERVED-AREA PIC X(4).`) `INITIALIZE` จะจัดการ field นั้นตามปกติเหมือน field อื่น ๆ ทุกประการ
- อย่าลืมว่าปัญหานี้จะเกิด**เฉพาะกรณีที่ FILLER ไม่มี `VALUE` clause กำกับไว้ตั้งแต่แรก** ถ้าประกาศ
  `05 FILLER PIC X(4) VALUE SPACES.` ไว้ตั้งแต่ต้น ค่าเริ่มต้นจะถูกกำหนดตอนโปรแกรมเริ่มทำงานอยู่แล้ว
  (เฉพาะ `WORKING-STORAGE`; record ใน `FILE SECTION` ไม่รับประกันค่าเริ่มต้นจาก `VALUE` เสมอไปใน
  ทุก compiler จึงยังควร `MOVE SPACES` ก่อน `WRITE` อยู่ดีเพื่อความปลอดภัยสูงสุด)

### แบบฝึกหัดที่ 237.1

**โจทย์**: จงเดาผลลัพธ์ของโปรแกรมต่อไปนี้ก่อนรันจริง แล้วอธิบายเหตุผล

```cobol
       01  WS-DATA.
           05  FILLER      PIC 9(3) VALUE 111.
           05  WS-NUM      PIC 9(3) VALUE 222.
       ...
           INITIALIZE WS-DATA.
           DISPLAY WS-DATA.
```

**เฉลย**: ผลลัพธ์คือ `111000` เพราะ `INITIALIZE` จะข้าม `FILLER` (ยังคงเป็น `111` ตามค่า `VALUE`
เดิมที่กำหนดไว้ตอนแรก ไม่ได้ถูกรีเซ็ตเป็น `000`) แต่จะเซ็ต `WS-NUM` (ซึ่งเป็น field ที่มีชื่อจริง)
ให้กลับเป็นค่าเริ่มต้นมาตรฐานของข้อมูลตัวเลขคือ `000` จุดนี้ยืนยันอีกครั้งว่า `INITIALIZE` ไม่แตะ
`FILLER` ไม่ว่า `FILLER` นั้นจะมี `VALUE` เริ่มต้นหรือไม่ก็ตาม

---

## ขั้นตอนที่ 238: Clause ประวัติศาสตร์ — LABEL RECORDS, BLOCK CONTAINS, VALUE OF

### ทำไมต้องรู้จัก Clause ที่ "ตายแล้ว"

เมื่อคุณต้องทำงานกับโค้ด COBOL ที่เขียนขึ้นในยุค 1970-1990 (ซึ่งยังมีอยู่จำนวนมากในระบบ Mainframe
จริงตามที่กล่าวถึงใน Part 001) คุณจะพบ clause เหล่านี้ปะปนอยู่ใน `FD` เป็นประจำ แม้จะไม่มีผลจริง
กับ compiler สมัยใหม่แล้วก็ตาม การเข้าใจความหมายของมันจึงจำเป็นสำหรับการอ่าน (ไม่ใช่การเขียนใหม่)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP238-LEGACY-CLAUSES.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OLD-FILE ASSIGN TO "OLD238.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  OLD-FILE
           LABEL RECORDS ARE STANDARD
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 10 CHARACTERS.
       01  OLD-RECORD                  PIC X(10).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT OLD-FILE.
           MOVE "MAINFRAME " TO OLD-RECORD.
           WRITE OLD-RECORD.
           CLOSE OLD-FILE.
           DISPLAY "Old-style FD clauses compiled fine.".
           STOP RUN.
```

**ผลลัพธ์:**

```
Old-style FD clauses compiled fine.
```

**เนื้อหาไฟล์ `OLD238.DAT`:**

```
MAINFRAME
```

### สิ่งที่เห็นเมื่อเปิดคำเตือนทั้งหมด (-Wall)

```bash
cobc -x -o step238 step238.cob -Wall
```

```
step238.cob:3: warning: AUTHOR is obsolete in GnuCOBOL
step238.cob:14: warning: LABEL RECORDS is obsolete in GnuCOBOL
step238.cob:16: warning: RECORD clause ignored for LINE SEQUENTIAL
```

จะเห็นว่า GnuCOBOL รายงานเตือน (ไม่ใช่ error) ว่า `LABEL RECORDS` ล้าสมัยแล้ว แต่ยังคงยอมให้
คอมไพล์ผ่านได้เพื่อความเข้ากันได้กับโค้ดเก่า (แม้แต่ `AUTHOR` ใน `IDENTIFICATION DIVISION` ที่เรา
ใช้มาตลอดหลักสูตรก็ถูกรายงานว่า obsolete เช่นกัน — นี่เป็นเรื่องปกติในหลาย compiler และไม่กระทบ
การทำงานของโปรแกรมแต่อย่างใด)

### ความหมายของแต่ละ Clause ประวัติศาสตร์

| Clause | ความหมายในยุคเทปแม่เหล็ก | สถานะปัจจุบัน |
|---|---|---|
| `LABEL RECORDS ARE STANDARD` | บอกว่าไฟล์มี "ป้ายกำกับ" มาตรฐานของระบบปฏิบัติการคั่นต้น-ท้ายไฟล์บนเทป | ระบบไฟล์สมัยใหม่ (Disk-based) จัดการเรื่องนี้อัตโนมัติอยู่แล้ว ไม่จำเป็นต้องเขียน |
| `LABEL RECORDS ARE OMITTED` | บอกว่าไฟล์ไม่มี label เลย (เช่น เทปข้อมูลดิบจากเครื่องอื่น) | เช่นเดียวกับข้างต้น |
| `BLOCK CONTAINS n RECORDS` | จัดกลุ่ม n record เข้าด้วยกันเป็นก้อนเดียวก่อนเขียนลงเทป/ดิสก์จริง เพื่อลดจำนวนรอบการเข้าถึงสื่อบันทึกทางกายภาพ (I/O overhead) | ระบบไฟล์และฮาร์ดแวร์สมัยใหม่จัดการ buffering ระดับต่ำให้อัตโนมัติ |
| `VALUE OF FILE-ID IS data-name` | ผูกชื่อไฟล์จริงแบบไดนามิกผ่านตัวแปรใน COBOL-74 (ก่อนที่ `ASSIGN TO` จะยืดหยุ่นแบบปัจจุบัน) | ปัจจุบันใช้ `ASSIGN TO` ร่วมกับตัวแปรได้โดยตรงแทน |
| `DATA RECORDS ARE record-name-1, record-name-2` | ประกาศรายชื่อ record ทั้งหมดที่อยู่ใน FD นี้อย่างชัดเจน | ปัจจุบันการมีหลาย `01` level ใต้ `FD` (ขั้นตอนที่ 234) ก็เพียงพอโดยไม่ต้องประกาศซ้ำ |

### ข้อควรระวัง

- **อย่าคัดลอก clause เหล่านี้เข้าไปในโค้ดใหม่ที่คุณเขียนเอง** เพราะเป็นภาระเพิ่มโดยไม่มีประโยชน์และ
  ทำให้โค้ดดูรกกว่าที่ควร ตลอดหลักสูตรนี้เราจะไม่ใช้ clause กลุ่มนี้ต่อไป (ยกเว้นเพื่อจุดประสงค์
  การสอนใน Part นี้)
- เมื่อพบ clause เหล่านี้ในโค้ด Mainframe จริงระหว่างทำงาน **อย่าตกใจและอย่าพยายามลบทิ้ง** โดยไม่
  เข้าใจ เพราะบางระบบ COBOL รุ่นเก่ามาก ๆ (โดยเฉพาะที่ยังรันบน Mainframe จริงกับเทปข้อมูล) อาจยัง
  ต้องพึ่งพา clause เหล่านี้จริง ๆ ให้ปรึกษาทีมที่ดูแลระบบก่อนแก้ไขเสมอ

### แบบฝึกหัดที่ 238.1

**โจทย์**: จงอธิบายว่าทำไม `BLOCK CONTAINS` จึงมีความสำคัญมากในยุคเทปแม่เหล็ก แต่แทบไม่มีความหมาย
กับไฟล์บนดิสก์ SSD ในปี 2026

**เฉลย**: เทปแม่เหล็กมีต้นทุนการ "เริ่มต้น/หยุด" การอ่านเขียนแต่ละครั้งสูงมาก (ต้องหมุนเทปและรอ
หัวอ่านเข้าตำแหน่ง) การจัดกลุ่มหลาย record เข้าด้วยกันเป็น block เดียวก่อนเขียนจึงช่วยลดจำนวนรอบ
การเข้าถึงเทปลงอย่างมาก ทำให้ประสิทธิภาพโดยรวมดีขึ้นมาก ในทางกลับกัน SSD สมัยใหม่มีความเร็วในการ
เข้าถึงข้อมูลแบบสุ่ม (Random Access) สูงมากและมี Buffer/Cache ของระบบปฏิบัติการจัดการให้อัตโนมัติ
อยู่แล้ว การกำหนด `BLOCK CONTAINS` ด้วยมือจึงแทบไม่ส่งผลต่อประสิทธิภาพเมื่อเทียบกับยุคเทปแม่เหล็ก

---

## ขั้นตอนที่ 239: แนวปฏิบัติที่ดี — คัดลอก FD Record ไปยัง WORKING-STORAGE ทันที

### ทำไมโปรแกรมมืออาชีพมักไม่ทำงานกับ FD Record โดยตรง

ตลอด Part 023 เราทำงานกับ field ใน `FD` โดยตรง (เช่น `DISPLAY CUSTOMER-RECORD`) ซึ่งใช้ได้ดีสำหรับ
โปรแกรมง่าย ๆ แต่ในโปรแกรมระดับองค์กรจริง แนวปฏิบัติที่แนะนำคือ **คัดลอกข้อมูลจาก FD record ไปยัง
record ใน `WORKING-STORAGE` ทันทีหลัง `READ` สำเร็จ** แล้วให้ paragraph ที่เหลือทำงานกับ
`WORKING-STORAGE` record เท่านั้น ไม่แตะ FD record อีก เหตุผลหลักมี 3 ข้อ:

1. **ความปลอดภัยของข้อมูล**: พื้นที่ FD record จะถูกเขียนทับทันทีที่มีการ `READ` ครั้งถัดไป ถ้าโปรแกรม
   เก็บ "ตัวชี้" ไปที่ field ใน FD ไว้ใช้ภายหลัง (เช่นเปรียบเทียบ record ปัจจุบันกับ record ก่อนหน้า)
   ข้อมูลจะเพี้ยนไปทันทีที่ `READ` รอบใหม่มาถึง
2. **ความชัดเจนของโค้ด**: การแยกชื่อ FD record (เช่น `FD-CUSTOMER-RECORD`) กับ WORKING-STORAGE
   record (เช่น `WS-CUSTOMER-RECORD`) ทำให้อ่านโค้ดแล้วรู้ทันทีว่าตัวแปรไหนมาจากไฟล์โดยตรง
   ตัวแปรไหนคือสำเนาที่โปรแกรมประมวลผลต่อได้อย่างอิสระ
3. **ความยืดหยุ่นเมื่อไฟล์เปลี่ยนแปลง**: ถ้าวันหนึ่งต้องเปลี่ยนแหล่งข้อมูลจากไฟล์เป็นฐานข้อมูล
   (Part 057-060 จะสอน DB2) โค้ดส่วน "ประมวลผล" ที่ทำงานกับ `WORKING-STORAGE` record จะไม่ต้อง
   แก้ไขอะไรเลย เปลี่ยนแค่ส่วน "อ่านข้อมูล" เท่านั้น

### ตัวอย่าง: แยก FD Record กับ WORKING-STORAGE Record อย่างชัดเจน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP239-COPY-TO-WS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODUCT239.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  FD-PRODUCT-RECORD.
           05  FD-PRODUCT-CODE          PIC X(6).
           05  FD-PRODUCT-QTY           PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-PRODUCT-RECORD.
           05  WS-PRODUCT-CODE          PIC X(6).
           05  WS-PRODUCT-QTY           PIC 9(5).
       01  WS-EOF-FLAG                  PIC X VALUE "N".
           88  END-OF-FILE              VALUE "Y".
       01  WS-TOTAL-QTY                 PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT PRODUCT-FILE.
           MOVE "IT9001" TO FD-PRODUCT-CODE.
           MOVE 100 TO FD-PRODUCT-QTY.
           WRITE FD-PRODUCT-RECORD.
           MOVE "IT9002" TO FD-PRODUCT-CODE.
           MOVE 50 TO FD-PRODUCT-QTY.
           WRITE FD-PRODUCT-RECORD.
           CLOSE PRODUCT-FILE.

           OPEN INPUT PRODUCT-FILE.
           PERFORM UNTIL END-OF-FILE
               READ PRODUCT-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
      *> Copy the FD record into a WORKING-STORAGE record right
      *> away. From this point on, the paragraph works only with
      *> WS-PRODUCT-RECORD, never touching FD-PRODUCT-RECORD again.
                       MOVE FD-PRODUCT-RECORD TO WS-PRODUCT-RECORD
                       ADD WS-PRODUCT-QTY TO WS-TOTAL-QTY
                       DISPLAY "Read: " WS-PRODUCT-CODE
                           " qty " WS-PRODUCT-QTY
               END-READ
           END-PERFORM.
           CLOSE PRODUCT-FILE.

           DISPLAY "Total quantity: " WS-TOTAL-QTY.
           STOP RUN.
```

**ผลลัพธ์:**

```
Read: IT9001 qty 00100
Read: IT9002 qty 00050
Total quantity: 0000150
```

### อธิบายจุดสำคัญ

- ตั้งชื่อ field ใน `FD` ด้วย prefix `FD-` และ field ใน `WORKING-STORAGE` ด้วย prefix `WS-` เป็น
  ธรรมเนียมที่นิยมมากในวงการ COBOL อาชีพ (Naming Convention) ช่วยให้อ่านโค้ดแล้วแยกแยะที่มาของ
  ตัวแปรได้ทันทีโดยไม่ต้องไปดู `DATA DIVISION`
- คำสั่ง `MOVE FD-PRODUCT-RECORD TO WS-PRODUCT-RECORD` เป็น **Group MOVE** ที่คัดลอกทุกไบต์
  จาก record หนึ่งไปยังอีก record หนึ่งในคำสั่งเดียว (ทั้งสอง record ต้องมีโครงสร้าง/ความยาว
  สอดคล้องกันเพื่อให้การคัดลอกถูกต้องตามตำแหน่ง)
- หลังจากบรรทัด `MOVE FD-PRODUCT-RECORD TO WS-PRODUCT-RECORD` โค้ดที่เหลือทั้งหมดในตัวอย่างนี้
  อ้างอิงเฉพาะ `WS-PRODUCT-CODE`/`WS-PRODUCT-QTY` เท่านั้น ไม่แตะ `FD-` อีกเลย ตามหลักปฏิบัติที่ดี

### ข้อควรระวัง

- Field ทั้งสองฝั่ง (FD และ WORKING-STORAGE) ต้องมี**โครงสร้างตรงกันทุกประการ** (ชนิดข้อมูลและ
  ลำดับ field เหมือนกัน) มิฉะนั้นการ `MOVE` ทั้งก้อนจะคัดลอกข้อมูลผิดตำแหน่งโดยไม่มี error เตือน
  (ทบทวนกฎ MOVE ระดับ Group จาก Part 008)
- แนวปฏิบัตินี้เพิ่มโค้ดอีกเล็กน้อย (ต้องประกาศตัวแปรซ้ำสองชุด) แต่ความชัดเจนและความปลอดภัยที่ได้
  คุ้มค่ามากในโปรแกรมขนาดใหญ่ สำหรับโปรแกรมสั้น ๆ ทดลองส่วนตัว การทำงานกับ FD record โดยตรง
  (แบบ Part 023) ก็ยังเป็นที่ยอมรับได้

### แบบฝึกหัดที่ 239.1

**โจทย์**: จงอธิบายว่าถ้าโปรแกรมต้องการเปรียบเทียบ record ปัจจุบันกับ record ก่อนหน้า (เช่น
ตรวจสอบว่าราคาสินค้าตัวเดียวกันเปลี่ยนแปลงหรือไม่ระหว่าง 2 record ที่ติดกัน) เพราะเหตุใดการทำงาน
กับ WORKING-STORAGE record จึงจำเป็น ไม่ใช่ทางเลือก

**เฉลย**: เพราะพื้นที่ของ FD record จะถูก COBOL เขียนทับด้วยข้อมูลใหม่ทุกครั้งที่ `READ` สำเร็จ
ถ้าต้องการเก็บ "ค่าของ record ก่อนหน้า" ไว้เปรียบเทียบกับ "ค่าของ record ปัจจุบัน" จำเป็นต้องมี
ตัวแปรอย่างน้อย 2 ชุดที่แยกพื้นที่ความจำออกจากกันจริง ๆ (เช่น `WS-CURRENT-RECORD` กับ
`WS-PREVIOUS-RECORD`) การพึ่งพา FD record ตัวเดียวจะทำให้ข้อมูลของ "record ก่อนหน้า" หายไปทันที
ที่ `READ` รอบใหม่มาถึง ทำให้ไม่สามารถเปรียบเทียบได้เลย (เทคนิคการเปรียบเทียบ record ติดกันแบบนี้
จะใช้จริงในเรื่อง Control Break ที่ Part 039)

---

## ขั้นตอนที่ 240: ตัวอย่างรวม — ออกแบบ FD สำหรับ Product Master File แบบสมบูรณ์

### โจทย์: ระบบสินค้าคงคลังอย่างง่าย

ขั้นตอนสุดท้ายของ Part นี้จะรวมทุกเทคนิคที่เรียนมาเข้าไว้ในโปรแกรมเดียว: `RECORD CONTAINS`,
Group/Elementary item, `FILLER` (พร้อมการป้องกันปัญหาข้อมูลขยะด้วย `MOVE SPACES`), Condition Name
(`88` level จาก Part 010), และการคำนวณสรุปผลจากไฟล์

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP240-FULL-FD-DESIGN.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-MASTER ASSIGN TO "PRODMAST240.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-MASTER
           RECORD CONTAINS 50 CHARACTERS.
       01  PM-RECORD.
           05  PM-PRODUCT-CODE          PIC X(8).
           05  FILLER                   PIC X(2).
           05  PM-PRODUCT-NAME          PIC X(20).
           05  PM-UNIT-PRICE            PIC 9(6)V99.
           05  FILLER                   PIC X(2).
           05  PM-QTY-ON-HAND           PIC 9(5).
           05  PM-STATUS                PIC X(1).
               88  PM-ACTIVE            VALUE "A".
               88  PM-DISCONTINUED      VALUE "D".

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                  PIC X VALUE "N".
           88  END-OF-FILE              VALUE "Y".
       01  WS-DISPLAY-PRICE             PIC ZZZ,ZZ9.99.
       01  WS-STOCK-VALUE               PIC 9(9)V99 VALUE 0.
       01  WS-DISPLAY-STOCK-VALUE       PIC ZZZ,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-PRODUCT-MASTER.
           PERFORM REPORT-PRODUCT-MASTER.
           STOP RUN.

       CREATE-PRODUCT-MASTER.
           OPEN OUTPUT PRODUCT-MASTER.

      *> Always clear the whole record first so FILLER bytes and
      *> any field we forget to MOVE never contain leftover garbage.
      *> Note: INITIALIZE would NOT be safe here because it skips
      *> FILLER items by design (see step 237) - MOVE SPACES does
      *> clear every byte, including FILLER, so we use it instead.
           MOVE SPACES TO PM-RECORD.
           MOVE "PRD00001" TO PM-PRODUCT-CODE.
           MOVE "RICE BAG 5KG        " TO PM-PRODUCT-NAME.
           MOVE 350.00 TO PM-UNIT-PRICE.
           MOVE 120 TO PM-QTY-ON-HAND.
           SET PM-ACTIVE TO TRUE.
           WRITE PM-RECORD.

           MOVE SPACES TO PM-RECORD.
           MOVE "PRD00002" TO PM-PRODUCT-CODE.
           MOVE "COOKING OIL 1L      " TO PM-PRODUCT-NAME.
           MOVE 95.50 TO PM-UNIT-PRICE.
           MOVE 60 TO PM-QTY-ON-HAND.
           SET PM-ACTIVE TO TRUE.
           WRITE PM-RECORD.

           MOVE SPACES TO PM-RECORD.
           MOVE "PRD00003" TO PM-PRODUCT-CODE.
           MOVE "OLD STYLE CANDLE    " TO PM-PRODUCT-NAME.
           MOVE 15.00 TO PM-UNIT-PRICE.
           MOVE 0 TO PM-QTY-ON-HAND.
           SET PM-DISCONTINUED TO TRUE.
           WRITE PM-RECORD.

           CLOSE PRODUCT-MASTER.
           DISPLAY "PRODMAST240.DAT created with 3 records.".

       REPORT-PRODUCT-MASTER.
           OPEN INPUT PRODUCT-MASTER.
           PERFORM UNTIL END-OF-FILE
               READ PRODUCT-MASTER
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       MOVE PM-UNIT-PRICE TO WS-DISPLAY-PRICE
                       DISPLAY PM-PRODUCT-CODE " " PM-PRODUCT-NAME
                           " " WS-DISPLAY-PRICE " qty="
                           PM-QTY-ON-HAND " status=" PM-STATUS
                       IF PM-ACTIVE
                           COMPUTE WS-STOCK-VALUE =
                               WS-STOCK-VALUE +
                               (PM-UNIT-PRICE * PM-QTY-ON-HAND)
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE PRODUCT-MASTER.

           MOVE WS-STOCK-VALUE TO WS-DISPLAY-STOCK-VALUE.
           DISPLAY "Total stock value (active only): "
               WS-DISPLAY-STOCK-VALUE.
```

**ผลลัพธ์:**

```
PRODMAST240.DAT created with 3 records.
PRD00001 RICE BAG 5KG             350.00 qty=00120 status=A
PRD00002 COOKING OIL 1L            95.50 qty=00060 status=A
PRD00003 OLD STYLE CANDLE          15.00 qty=00000 status=D
Total stock value (active only):      47,730.00
```

### อธิบายจุดสำคัญของโปรแกรมรวม

- ตรวจสอบผลรวม: `350.00 x 120 = 42,000.00` บวก `95.50 x 60 = 5,730.00` รวมเป็น
  `47,730.00` พอดี — สังเกตว่าสินค้ารายการที่ 3 (`OLD STYLE CANDLE`) มีสถานะ `PM-DISCONTINUED`
  จึง**ไม่ถูกนับรวม**ในมูลค่าสต๊อก แม้จะมี `PM-UNIT-PRICE` อยู่ก็ตาม เพราะเราใช้ `IF PM-ACTIVE`
  (Condition Name จาก Part 010) เป็นเงื่อนไขกรองข้อมูล
- `PM-STATUS` มี 2 ค่าที่เป็นไปได้ควบคุมด้วย `88` level สองตัว (`PM-ACTIVE`, `PM-DISCONTINUED`)
  ทำให้โค้ดใน `PROCEDURE DIVISION` อ่านง่ายเหมือนภาษาอังกฤษ (`IF PM-ACTIVE` แทนที่จะเป็น
  `IF PM-STATUS = "A"`) ตามหลักการที่เรียนมาตั้งแต่ Part 010
- `MOVE SPACES TO PM-RECORD` ถูกเรียก**ก่อนทุกครั้ง**ที่จะสร้าง record ใหม่ในลูป (จริง ๆ ในตัวอย่าง
  นี้เขียนแบบ Sequence ธรรมดาไม่ใช่ลูป แต่หลักการเดียวกันนี้สำคัญมากยิ่งขึ้นเมื่อเขียนในลูปจริง
  เพราะไม่เช่นนั้นค่าจาก record ก่อนหน้าอาจตกค้างบางส่วนถ้าลืม MOVE field ใดไป)

### ข้อควรระวัง

- ทดลองเปลี่ยน `MOVE SPACES TO PM-RECORD` เป็น `INITIALIZE PM-RECORD` แล้วรันดูจะพบว่าโปรแกรม
  **ล่มทันทีด้วย status 71 เหมือนขั้นตอนที่ 236** เพราะ `INITIALIZE` ไม่แตะ `FILLER` ทั้งสองตัวใน
  `PM-RECORD` ปล่อยให้มันมีค่าขยะจากหน่วยความจำเหมือนเดิม — นี่คือการยืนยันซ้ำอีกครั้งด้วยตัวอย่าง
  ที่สมจริงกว่าเดิมว่าทำไมกฎในขั้นตอนที่ 236–237 จึงสำคัญมาก
- ระวังการจัดวางตำแหน่ง `FILLER` ให้เหมาะสมกับการใช้งานจริง เช่น ในตัวอย่างนี้ `FILLER` ถูกใช้เป็น
  ช่องว่างคั่นระหว่าง field เพื่อให้ไฟล์ดิบอ่านง่ายขึ้นเมื่อเปิดด้วยตาเปล่า

### แบบฝึกหัดที่ 240.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงเพิ่มสถานะใหม่ `PM-OUT-OF-STOCK` (ใช้ตัวอักษร `"O"`) ให้กับ `PM-STATUS` แล้วปรับ
`REPORT-PRODUCT-MASTER` ให้แสดงข้อความ `"** LOW STOCK WARNING **"` ต่อท้ายบรรทัดรายงาน เมื่อสินค้า
มีสถานะนี้

**เฉลย**: เพิ่ม `88` level ใหม่ใน `FD`:

```cobol
               88  PM-OUT-OF-STOCK      VALUE "O".
```

แล้วเพิ่มเงื่อนไขใน `REPORT-PRODUCT-MASTER` หลังบรรทัด `DISPLAY` เดิม:

```cobol
                       IF PM-OUT-OF-STOCK
                           DISPLAY "  ** LOW STOCK WARNING **"
                       END-IF
```

จากนั้นเมื่อสร้างสินค้าตัวใหม่ที่มี `SET PM-OUT-OF-STOCK TO TRUE` แทน `SET PM-ACTIVE TO TRUE`
โปรแกรมจะแสดงคำเตือนต่อท้ายบรรทัดของสินค้าตัวนั้นโดยอัตโนมัติ โดยไม่กระทบ logic ของสินค้าอื่นเลย

---

## สรุปท้ายบท

ใน Part นี้เราได้เจาะลึก `FILE SECTION` และ `FD Entry` อย่างละเอียดที่สุด:

- ภาพรวม clause ทั้งหมดของ `FD` และรู้ว่า `RECORD CONTAINS` ถูก compiler เพิกเฉยเมื่อใช้
  `LINE SEQUENTIAL`
- การออกแบบ record ด้วย Group Item, Elementary Item และ Level Number ที่ซ้อนกันได้หลายชั้น
- การใช้ `FILLER` เพื่อจองพื้นที่โดยไม่ต้องตั้งชื่อ และเหตุผลที่ห้ามอ้างอิงถึงมันโดยตรง
- การมีหลาย record layout ใน `FD` เดียวกันแบบ Shared Storage สำหรับไฟล์ที่มี record หลายประเภท
- `RECORD IS VARYING IN SIZE ... DEPENDING ON` สำหรับ record ความยาวไม่คงที่
- **ปัญหาสำคัญที่พิสูจน์จริงด้วยการทดลอง**: ข้อมูลขยะใน `FILLER` ที่ไม่ได้กำหนดค่าทำให้ `WRITE`
  ไปยัง `LINE SEQUENTIAL` ล้มเหลวด้วย File Status 71 และวิธีแก้ด้วย `MOVE SPACES`
- คำสั่ง `INITIALIZE` โดยละเอียด รวมถึง **ข้อยกเว้นสำคัญที่สุด**: `INITIALIZE` จะข้าม `FILLER`
  เสมอตามมาตรฐาน COBOL ต่างจาก `MOVE SPACES` ที่ครอบคลุมทุกไบต์
- Clause ประวัติศาสตร์ (`LABEL RECORDS`, `BLOCK CONTAINS`, `VALUE OF`, `DATA RECORDS`) ที่ยังพบได้
  ในโค้ด Mainframe เก่า
- แนวปฏิบัติที่ดี: คัดลอก FD record ไปยัง WORKING-STORAGE record ทันทีหลัง `READ`
- ตัวอย่างรวมการออกแบบ FD สำหรับ Product Master File ที่สมบูรณ์

Part ถัดไป (**Part 025**) จะพาคุณกลับไปที่ `PROCEDURE DIVISION` เพื่อเจาะลึกคำสั่งจัดการไฟล์ทั้งหมด
อย่างครบถ้วน: `OPEN` ทุกโหมด, `CLOSE`, `READ`, `WRITE`, และคำสั่งใหม่ที่ยังไม่เคยเรียนคือ `REWRITE`
(แก้ไข record ที่มีอยู่แล้ว) และ `DELETE` (ลบ record) ซึ่งจำเป็นต้องใช้โหมด `OPEN I-O` ที่เพิ่งกล่าว
ถึงเป็นครั้งแรกใน Part 023

**[← กลับไป Part 023](part-023-sequential-files.md)** | **[ไปยัง Part 025: OPEN, CLOSE, READ, WRITE, REWRITE, DELETE →](part-025-open-close-read-write.md)**
