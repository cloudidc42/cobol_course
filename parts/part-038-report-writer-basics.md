# Part 038: Report Writer Feature เบื้องต้น (ขั้นตอนที่ 371–380)

## คำนำของ Part นี้

ตลอดเฟส 2 และเฟส 3 ที่ผ่านมา เมื่อเราต้องการพิมพ์รายงาน เราใช้วิธี `DISPLAY` หรือเขียน `WRITE`
เข้าไฟล์ตรง ๆ ทีละบรรทัด พร้อมคำนวณตำแหน่งคอลัมน์และหัวกระดาษ (heading) ด้วยมือทุกครั้ง ซึ่งใช้งานได้ดี
สำหรับรายงานง่าย ๆ แต่เมื่อรายงานซับซ้อนขึ้น — ต้องมีหัวกระดาษซ้ำทุกหน้า ต้องขึ้นหน้าใหม่อัตโนมัติเมื่อ
เนื้อหาเต็มหน้า ต้องคำนวณยอดรวม (subtotal/grand total) ตามกลุ่มข้อมูล — โค้ดแบบ `DISPLAY`/`WRITE`
ล้วนจะเริ่มยาวและซับซ้อนมาก

COBOL จึงมีฟีเจอร์เฉพาะทางที่ออกแบบมาเพื่องานพิมพ์รายงานโดยเฉพาะ เรียกว่า **Report Writer Feature**
เป็นกลุ่มคำสั่งใน DATA DIVISION (REPORT SECTION) ที่ให้เรา "ประกาศ" ว่ารายงานหน้าตาเป็นอย่างไร
(หัวกระดาษอยู่ตรงไหน รายละเอียดแต่ละบรรทัดพิมพ์อะไร ท้ายหน้าพิมพ์อะไร) แล้วปล่อยให้ COBOL runtime
จัดการเรื่องการขึ้นบรรทัดใหม่ การขึ้นหน้าใหม่ และการพิมพ์หัวกระดาษซ้ำให้เราโดยอัตโนมัติ

Part นี้ได้ทดสอบคอมไพล์และรันจริงทุกตัวอย่างด้วย **GnuCOBOL 4.0 (early-dev)** ก่อนนำมาเขียนบทเรียน
พบว่า GnuCOBOL รองรับ Report Writer ในระดับที่ใช้งานได้จริงค่อนข้างดี ครอบคลุม `RD`, `REPORT SECTION`,
`TYPE DETAIL`/`TYPE PAGE HEADING`/`TYPE PAGE FOOTING`/`TYPE REPORT HEADING`, `SOURCE`, `GENERATE`,
`INITIATE`/`TERMINATE` และการขึ้นหน้าอัตโนมัติตาม `PAGE LIMIT` เราจะพาคุณไล่เรียงตั้งแต่แนวคิดพื้นฐาน
ไปจนถึงการอ่านข้อมูลจากไฟล์จริงมาพิมพ์เป็นรายงาน ปิดท้ายด้วยการเปรียบเทียบกับแนวทาง "รายงานแบบมือ"
(Manual DISPLAY-based Report) ที่ยังคงมีที่ใช้งานในโลกจริงเช่นกัน

ส่วน Control Break แบบเต็มรูป (การขึ้นกลุ่มใหม่ตามคีย์ พร้อมยอดรวมแต่ละกลุ่ม) จะแยกไปสอนอย่างละเอียด
ใน **Part 039** ต่อจาก Part นี้

---

## ขั้นตอนที่ 371: Report Writer คืออะไร และ RD Entry แรกของคุณ

### แนวคิด

**Report Writer** เป็นชุดคำสั่งพิเศษของ COBOL ที่แยกกลไกการพิมพ์รายงาน (layout, การขึ้นหน้าใหม่,
การนับบรรทัด) ออกจากตรรกะทางธุรกิจใน PROCEDURE DIVISION ให้ชัดเจน หัวใจของฟีเจอร์นี้มี 3 ส่วน:

1. **`FD ... REPORT IS report-name`** ใน FILE SECTION — บอกว่าไฟล์นี้เป็นไฟล์รายงาน และผูกกับ
   รายงานชื่ออะไร
2. **`REPORT SECTION`** ใน DATA DIVISION — ส่วนใหม่ที่ประกาศ**โครงสร้าง**ของรายงานทั้งหมด เริ่มด้วย
   **`RD` (Report Description)** ตามด้วยกลุ่มรายการ (report group) ต่าง ๆ ที่มี level number
   (01, 03, 05 ...) เหมือน WORKING-STORAGE ปกติ แต่มี clause พิเศษเช่น `TYPE`, `LINE`, `COLUMN`
3. **คำสั่งควบคุมใน PROCEDURE DIVISION**: `INITIATE`, `GENERATE`, `TERMINATE` (จะเรียนละเอียดใน
   ขั้นตอนที่ 375)

ข้อสำคัญที่ต้องรู้ทันที: **ไฟล์รายงาน (report file) ไม่มี `01` record ปกติใน FILE SECTION** เหมือน
ไฟล์ทั่วไปที่เรียนใน Part 024 — เราแค่ประกาศ `FD` พร้อม clause `REPORT IS` แล้วปล่อยให้ REPORT SECTION
เป็นผู้กำหนดว่าแต่ละบรรทัดของไฟล์นี้หน้าตาเป็นอย่างไรแทน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-FIRST-REPORT.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "sales_report.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-NAME    PIC X(10).
       01  WS-AMOUNT  PIC 9(4).

       REPORT SECTION.
       RD  SALES-REPORT
           PAGE LIMIT 20 LINES.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1  PIC X(20) VALUE "SALES REPORT DEMO".

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 20  PIC ZZZ9  SOURCE WS-AMOUNT.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.
           MOVE "APPLE" TO WS-NAME.
           MOVE 100 TO WS-AMOUNT.
           GENERATE DETAIL-LINE.
           MOVE "BANANA" TO WS-NAME.
           MOVE 250 TO WS-AMOUNT.
           GENERATE DETAIL-LINE.
           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `FD REPORT-FILE REPORT IS SALES-REPORT.` — ผูกไฟล์กายภาพ `REPORT-FILE` เข้ากับรายงานชื่อ
  `SALES-REPORT` ที่จะถูกประกาศโครงสร้างใน REPORT SECTION ด้านล่าง สังเกตว่า**ไม่มี** `01` record
  ตามหลัง `FD` แบบไฟล์ปกติเลย
- `RD SALES-REPORT PAGE LIMIT 20 LINES.` — `RD` (Report Description) ประกาศว่ารายงานชื่อ
  `SALES-REPORT` แต่ละหน้ามีความยาวสูงสุด 20 บรรทัด
- `01 TYPE PAGE HEADING.` — กลุ่มรายการที่มี `TYPE PAGE HEADING` จะถูกพิมพ์อัตโนมัติที่ต้นทุกหน้า
  (จะลงรายละเอียดในขั้นตอนที่ 372)
- `01 DETAIL-LINE TYPE DETAIL.` — กลุ่มรายการที่มี `TYPE DETAIL` คือบรรทัดข้อมูลหลักของรายงาน
  `03 LINE PLUS 1` หมายถึง "เลื่อนลงมา 1 บรรทัดจากตำแหน่งปัจจุบัน" และ `05 COLUMN n PIC ... SOURCE
  field-name` หมายถึง "พิมพ์ค่าจากตัวแปร `field-name` เริ่มที่คอลัมน์ `n`" (จะลงรายละเอียดในขั้นตอนที่ 373)
- ใน PROCEDURE DIVISION เราเพียงแค่ `OPEN`, `INITIATE`, ตั้งค่าตัวแปร แล้วสั่ง `GENERATE` ซ้ำ ๆ
  โดยไม่ต้องเขียนโค้ดคำนวณตำแหน่งบรรทัดเองเลย

### ผลลัพธ์ที่ได้จากการรันจริง

ทดสอบคอมไพล์และรันจริงด้วย `cobc -x -o rw-first-report rw-first-report.cob && ./rw-first-report`
ไฟล์ `sales_report.txt` ที่ได้มีเนื้อหา (แสดง `$` แทนจุดจบบรรทัดเพื่อความชัดเจน):

```
SALES REPORT DEMO$
APPLE               100$
BANANA              250$
```

ตามด้วยบรรทัดว่างอีกหลายบรรทัดจนครบ `PAGE LIMIT 20 LINES` (พฤติกรรมนี้จะอธิบายในขั้นตอนที่ 376
เรื่อง PAGE FOOTING และการขึ้นหน้าใหม่)

### ข้อควรระวัง

- **REPORT SECTION ต้องมาหลัง WORKING-STORAGE SECTION** ในลำดับของ DATA DIVISION (ลำดับที่ถูกต้อง
  คือ FILE SECTION → WORKING-STORAGE SECTION → REPORT SECTION) หากสลับลำดับจะ compile error
- ชื่อรายงาน (report-name) หลัง `RD` ต้องตรงกับชื่อที่ระบุใน `REPORT IS` ของ `FD` ทุกตัวอักษร
- ต้องเปิดไฟล์ (`OPEN OUTPUT`) ก่อนเรียก `INITIATE` เสมอ เช่นเดียวกับไฟล์ปกติ

### แบบฝึกหัดที่ 371.1

**โจทย์**: จงอธิบายว่าทำไม `FD` ของไฟล์รายงานจึงไม่มี `01` record แบบไฟล์ปกติที่เรียนใน Part 024

**เฉลยแนวทาง**: เพราะโครงสร้างของแต่ละบรรทัดในไฟล์รายงานถูกกำหนดไว้ล่วงหน้าแล้วใน `REPORT SECTION`
ผ่านกลุ่มรายการต่าง ๆ (`TYPE PAGE HEADING`, `TYPE DETAIL` ฯลฯ) COBOL runtime จะเป็นผู้สร้างเนื้อหา
แต่ละบรรทัดให้เองตอนที่เราสั่ง `GENERATE` หรือเมื่อถึงจังหวะพิมพ์หัวกระดาษ/ท้ายกระดาษ เราจึงไม่ต้อง
ประกาศ record layout ซ้ำใน FILE SECTION อีก

---

## ขั้นตอนที่ 372: TYPE PAGE HEADING และ TYPE REPORT HEADING

### แนวคิด

Report Writer แยกความแตกต่างระหว่าง**หัวกระดาษที่พิมพ์ทุกหน้า**กับ**หัวกระดาษที่พิมพ์ครั้งเดียวตอน
เริ่มรายงาน**อย่างชัดเจนด้วย 2 ชนิดกลุ่มรายการ:

- **`TYPE PAGE HEADING`** — พิมพ์ซ้ำที่ต้นทุกหน้า (เหมาะกับหัวตาราง เช่น ชื่อคอลัมน์)
- **`TYPE REPORT HEADING`** — พิมพ์เพียงครั้งเดียวที่ต้นหน้าแรกของรายงานทั้งฉบับเท่านั้น (เหมาะกับ
  หน้าปกรายงาน เช่น ชื่อรายงานเต็ม, วันที่จัดทำ)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-PAGE-VS-REPORT-HEADING.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "stock_report.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS ITEM-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-ITEM-NAME  PIC X(10).
       01  WS-QTY        PIC 9(4).
       01  WS-COUNTER    PIC 9(2).

       REPORT SECTION.
       RD  ITEM-REPORT
           PAGE LIMIT 8 LINES
           HEADING 1
           FIRST DETAIL 4
           FOOTING 7.

       01  TYPE REPORT HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(30) VALUE "STOCK REPORT (COVER PAGE)".

       01  TYPE PAGE HEADING.
           03  LINE 2.
               05  COLUMN 1 PIC X(20) VALUE "ITEM      QTY".

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) SOURCE WS-ITEM-NAME.
               05  COLUMN 12 PIC ZZZ9  SOURCE WS-QTY.

       01  TYPE PAGE FOOTING.
           03  LINE PLUS 1.
               05  COLUMN 1 PIC X(15) VALUE "-- PAGE END --".

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE ITEM-REPORT.
           PERFORM VARYING WS-COUNTER FROM 1 BY 1 UNTIL WS-COUNTER > 5
               MOVE "ITEM" TO WS-ITEM-NAME
               MOVE WS-COUNTER TO WS-QTY
               GENERATE DETAIL-LINE
           END-PERFORM.
           TERMINATE ITEM-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `RD ITEM-REPORT PAGE LIMIT 8 LINES HEADING 1 FIRST DETAIL 4 FOOTING 7.` — กำหนดว่าแต่ละหน้ายาว
  8 บรรทัด, หัวกระดาษเริ่มที่บรรทัด 1, บรรทัดรายละเอียดแรกเริ่มที่บรรทัด 4, และท้ายกระดาษ (footing)
  ต้องเริ่มไม่เกินบรรทัด 7 — ตั้งค่าให้เล็กโดยตั้งใจเพื่อบังคับให้ขึ้นหน้าใหม่เร็ว ๆ จะได้เห็นพฤติกรรม
  ชัดเจนในตัวอย่างนี้
- `01 TYPE REPORT HEADING` ที่ `LINE 1` — พิมพ์ที่บรรทัด 1 ของ**หน้าแรกเท่านั้น**
- `01 TYPE PAGE HEADING` ที่ `LINE 2` — พิมพ์ที่บรรทัด 2 ของ**ทุกหน้า** (รวมหน้าแรกด้วย)
- ลูป `PERFORM VARYING` สร้างรายการสินค้า 5 รายการ ทำให้ต้องขึ้นหน้าใหม่เนื่องจาก `PAGE LIMIT`
  มีขนาดเล็ก

### ผลลัพธ์ที่ได้จากการรันจริง

```
STOCK REPORT (COVER PAGE)
ITEM      QTY

ITEM          1
ITEM          2
ITEM          3
ITEM          4
-- PAGE END --

ITEM      QTY

ITEM          5
-- PAGE END --
```

สังเกตว่า **"STOCK REPORT (COVER PAGE)" ปรากฏเพียงครั้งเดียว** ที่บรรทัดแรกสุดของทั้งรายงาน
ในขณะที่ **"ITEM      QTY" ปรากฏซ้ำทุกหน้า** (ทั้งหน้าแรกและหน้าที่สอง) ตรงตามความหมายของ
`TYPE REPORT HEADING` กับ `TYPE PAGE HEADING` ทุกประการ และหน้าที่ 1 พิมพ์ได้ 4 รายการพอดี
ก่อนขึ้นหน้าใหม่ไปพิมพ์รายการที่ 5 ในหน้าที่ 2

### ข้อควรระวัง

- ถ้าใช้ `TYPE REPORT HEADING` และ `TYPE PAGE HEADING` วาง `LINE` ทับกัน (เช่นทั้งคู่ `LINE 1`)
  ในหน้าแรกจะพิมพ์ทับกันหรือเกิดผลลัพธ์ไม่ตรงตามที่ตั้งใจ ควรจัดสรรเลขบรรทัดให้ไม่ชนกัน
- `RD` clause `HEADING`, `FIRST DETAIL`, `FOOTING`, `PAGE LIMIT` ต้องสอดคล้องกัน
  (`HEADING <= FIRST DETAIL <= FOOTING <= PAGE LIMIT`) มิเช่นนั้น layout จะผิดเพี้ยนหรือ
  compile error
- ถ้าไม่ระบุ `TYPE REPORT HEADING` เลย โปรแกรมจะไม่ error แต่จะไม่มีหน้าปกใด ๆ พิมพ์ออกมา
  (เป็น clause เสริม ไม่บังคับ)

### แบบฝึกหัดที่ 372.1

**โจทย์**: จงอธิบายว่าถ้าลบ `01 TYPE REPORT HEADING` ออกจากตัวอย่างข้างต้น (เหลือแต่
`TYPE PAGE HEADING`) ผลลัพธ์หน้าแรกของรายงานจะเปลี่ยนไปอย่างไร

**เฉลยแนวทาง**: บรรทัด "STOCK REPORT (COVER PAGE)" ที่เคยปรากฏเฉพาะหน้าแรกจะหายไปทั้งหมด
เหลือเพียง "ITEM      QTY" ที่พิมพ์ซ้ำทุกหน้ารวมถึงหน้าแรกเช่นเดิม เพราะไม่มีกลุ่มรายการชนิด
`TYPE REPORT HEADING` ให้ COBOL runtime พิมพ์แล้ว

---

## ขั้นตอนที่ 373: TYPE DETAIL, SOURCE Clause และการวางตำแหน่งด้วย LINE/COLUMN

### แนวคิด

**`TYPE DETAIL`** คือกลุ่มรายการที่เป็นหัวใจของรายงาน ใช้พิมพ์ข้อมูลทีละแถว (record) ทุกครั้งที่มี
การสั่ง `GENERATE` แต่ละฟิลด์ย่อยภายในกลุ่ม `TYPE DETAIL` มักประกอบด้วย:

- **`LINE`** — ตำแหน่งบรรทัด เขียนได้ทั้งเลขบรรทัดตายตัว (`LINE 5`) หรือแบบสัมพัทธ์
  (`LINE PLUS 1` = เลื่อนลง 1 บรรทัดจากตำแหน่งก่อนหน้า ซึ่งเป็นรูปแบบที่ใช้บ่อยที่สุดสำหรับ
  `TYPE DETAIL` เพราะแต่ละแถวไม่ได้อยู่ตำแหน่งตายตัว)
- **`COLUMN`** — ตำแหน่งคอลัมน์เริ่มพิมพ์ของแต่ละฟิลด์
- **`SOURCE`** — บอกว่าค่าที่จะพิมพ์มาจากตัวแปรใดใน WORKING-STORAGE (หรือ FILE SECTION)
  COBOL จะดึงค่าปัจจุบันของตัวแปรนั้นมาแปลงตาม `PIC` ที่ระบุใน report entry เองโดยอัตโนมัติ
  ทุกครั้งที่ `GENERATE`

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-DETAIL-SOURCE-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "detail_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS PRODUCT-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-PRODUCT-CODE  PIC X(6).
       01  WS-PRODUCT-NAME  PIC X(12).
       01  WS-UNIT-PRICE    PIC 9(5)V99.
       01  WS-QTY-ON-HAND   PIC 9(5).

       REPORT SECTION.
       RD  PRODUCT-REPORT
           PAGE LIMIT 30 LINES.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1  PIC X(6)  VALUE "CODE".
               05  COLUMN 10 PIC X(12) VALUE "NAME".
               05  COLUMN 25 PIC X(8)  VALUE "PRICE".
               05  COLUMN 36 PIC X(5)  VALUE "QTY".

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(6)      SOURCE WS-PRODUCT-CODE.
               05  COLUMN 10 PIC X(12)     SOURCE WS-PRODUCT-NAME.
               05  COLUMN 25 PIC ZZ,ZZ9.99 SOURCE WS-UNIT-PRICE.
               05  COLUMN 36 PIC ZZ,ZZ9    SOURCE WS-QTY-ON-HAND.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE PRODUCT-REPORT.

           MOVE "P-001"        TO WS-PRODUCT-CODE.
           MOVE "USB CABLE"    TO WS-PRODUCT-NAME.
           MOVE 129.50         TO WS-UNIT-PRICE.
           MOVE 350            TO WS-QTY-ON-HAND.
           GENERATE DETAIL-LINE.

           MOVE "P-002"        TO WS-PRODUCT-CODE.
           MOVE "KEYBOARD"     TO WS-PRODUCT-NAME.
           MOVE 890.00         TO WS-UNIT-PRICE.
           MOVE 42             TO WS-QTY-ON-HAND.
           GENERATE DETAIL-LINE.

           TERMINATE PRODUCT-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `05 COLUMN 25 PIC ZZ,ZZ9.99 SOURCE WS-UNIT-PRICE.` — พิมพ์ค่าจาก `WS-UNIT-PRICE`
  (ซึ่งเก็บเป็น `PIC 9(5)V99` ดิบ ๆ ในหน่วยความจำ) โดยแปลงรูปแบบให้มีจุลภาคคั่นหลักพันและจุดทศนิยม
  2 ตำแหน่งโดยอัตโนมัติ — นี่คือจุดแข็งสำคัญของ Report Writer: **PICTURE editing** เกิดขึ้นเองตอน
  `GENERATE` โดยไม่ต้องเขียนโค้ดแปลงรูปแบบเองเลย
- ทุกฟิลด์ที่มี `SOURCE` จะดึงค่า ณ **ขณะที่ `GENERATE` ทำงาน** เท่านั้น หากตัวแปรเปลี่ยนค่าหลังจาก
  `GENERATE` ไปแล้ว จะไม่กระทบกับบรรทัดที่พิมพ์ไปแล้ว
- `COLUMN` ของแต่ละฟิลด์ในตัวอย่างนี้เว้นระยะห่างให้พอดีกับความกว้างของฟิลด์ก่อนหน้า
  (`COLUMN 1` กว้าง 6, ฟิลด์ถัดไปเริ่ม `COLUMN 10` มีช่องว่างคั่น) ผู้เขียนต้องคำนวณตำแหน่งเอง
  ไม่มีการจัดเรียงอัตโนมัติ

### ผลลัพธ์ที่ได้จากการรันจริง

```
CODE     NAME           PRICE      QTY
P-001    USB CABLE         129.50     350
P-002    KEYBOARD          890.00      42
```

### ข้อควรระวัง

- **`SOURCE` field-name ต้องเป็นตัวแปรที่มีอยู่จริงใน DATA DIVISION** (WORKING-STORAGE หรือ
  FILE SECTION) และชนิดข้อมูลควรเข้ากันได้กับ `PIC` ของ report entry (ตัวเลขกับตัวเลข)
- ถ้า `COLUMN` ของสองฟิลด์วางซ้อนทับกัน (เช่นฟิลด์แรกกว้าง 12 ตัวอักษรเริ่มที่ COLUMN 1 แต่ฟิลด์ที่สอง
  เริ่มที่ COLUMN 10) ข้อความจะพิมพ์ทับกันโดยไม่มี error เตือน ต้องคำนวณความกว้างของแต่ละฟิลด์
  ให้รอบคอบเสมอ
- ระวังอย่าลืมว่าคอลัมน์เริ่มนับจาก 1 ไม่ใช่ 0

### แบบฝึกหัดที่ 373.1

**โจทย์**: จงเพิ่มฟิลด์ใหม่ `WS-TOTAL-VALUE PIC 9(7)V99` ที่คำนวณจาก `WS-UNIT-PRICE * WS-QTY-ON-HAND`
ในโปรแกรม แล้วเพิ่มคอลัมน์ใหม่ในรายงานที่ `COLUMN 46` เพื่อแสดงมูลค่ารวมของสินค้าแต่ละรายการ

**เฉลยแนวทาง**:
```cobol
       01  WS-TOTAL-VALUE   PIC 9(7)V99.
       ...
       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(6)      SOURCE WS-PRODUCT-CODE.
               05  COLUMN 10 PIC X(12)     SOURCE WS-PRODUCT-NAME.
               05  COLUMN 25 PIC ZZ,ZZ9.99 SOURCE WS-UNIT-PRICE.
               05  COLUMN 36 PIC ZZ,ZZ9    SOURCE WS-QTY-ON-HAND.
               05  COLUMN 46 PIC ZZZ,ZZ9.99 SOURCE WS-TOTAL-VALUE.
       ...
       PROCEDURE DIVISION.
           ...
           COMPUTE WS-TOTAL-VALUE = WS-UNIT-PRICE * WS-QTY-ON-HAND.
           GENERATE DETAIL-LINE.
```
ต้องคำนวณ `WS-TOTAL-VALUE` **ก่อน** `GENERATE DETAIL-LINE` ทุกครั้ง เพราะ `SOURCE` จะดึงค่า
ณ ขณะที่ `GENERATE` ทำงานเท่านั้น

---

## ขั้นตอนที่ 374: คำสั่ง GENERATE — กลไกเบื้องหลังการพิมพ์อัตโนมัติ

### แนวคิด

`GENERATE report-group-name` คือคำสั่งเดียวที่สั่งให้ Report Writer "พิมพ์" ข้อมูลออกมา แต่เบื้องหลัง
มันทำงานมากกว่าการพิมพ์บรรทัดเดียวธรรมดา ทุกครั้งที่ `GENERATE` ถูกเรียกกับกลุ่ม `TYPE DETAIL`
COBOL runtime จะตรวจสอบและทำงานตามลำดับต่อไปนี้ให้อัตโนมัติ:

1. ถ้าเป็นการ `GENERATE` ครั้งแรกของรายงาน จะพิมพ์ `TYPE REPORT HEADING` (ถ้ามี) และ
   `TYPE PAGE HEADING` ก่อน
2. ตรวจสอบว่าตำแหน่งบรรทัดปัจจุบันเลยขอบเขต `FOOTING`/`PAGE LIMIT` หรือยัง ถ้าเลยแล้วจะพิมพ์
   `TYPE PAGE FOOTING` (ถ้ามี) ปิดหน้าเดิม แล้วขึ้นหน้าใหม่พร้อมพิมพ์ `TYPE PAGE HEADING` ซ้ำ
3. พิมพ์เนื้อหาของกลุ่ม `TYPE DETAIL` ตามตำแหน่ง `LINE`/`COLUMN` ที่ประกาศไว้ โดยดึงค่าจาก
   `SOURCE` ทุกฟิลด์ ณ ขณะนั้น

นี่คือเหตุผลที่ Report Writer สะดวกกว่าการเขียน `WRITE` เอง: เราไม่ต้องคอยเช็คเองว่า "ตอนนี้ถึงบรรทัด
สุดท้ายของหน้าหรือยัง" — `GENERATE` จัดการให้ทั้งหมด

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-GENERATE-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "generate_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS ITEM-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-ITEM-NAME  PIC X(10).
       01  WS-QTY        PIC 9(4).

       REPORT SECTION.
       RD  ITEM-REPORT
           PAGE LIMIT 30 LINES.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(20) VALUE "ITEM      QTY".

       01  NORMAL-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) SOURCE WS-ITEM-NAME.
               05  COLUMN 12 PIC ZZZ9  SOURCE WS-QTY.

       01  WARNING-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) SOURCE WS-ITEM-NAME.
               05  COLUMN 12 PIC ZZZ9  SOURCE WS-QTY.
               05  COLUMN 18 PIC X(12) VALUE "** LOW! **".

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE ITEM-REPORT.

      *> A report can have MORE THAN ONE type-detail group. We
      *> choose which one to print by naming it in GENERATE.
           MOVE "APPLE" TO WS-ITEM-NAME.
           MOVE 100 TO WS-QTY.
           GENERATE NORMAL-DETAIL.

           MOVE "BANANA" TO WS-ITEM-NAME.
           MOVE 3 TO WS-QTY.
           GENERATE WARNING-DETAIL.

           TERMINATE ITEM-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- รายงานเดียวสามารถมี**กลุ่ม `TYPE DETAIL` มากกว่า 1 กลุ่ม** ได้ โดยแต่ละกลุ่มตั้งชื่อไม่ซ้ำกัน
  (`NORMAL-DETAIL`, `WARNING-DETAIL`) และเราเลือกว่าจะพิมพ์แบบไหนโดยระบุชื่อกลุ่มนั้นตรง ๆ ใน
  `GENERATE`
- `GENERATE NORMAL-DETAIL` ครั้งแรกทริกเกอร์การพิมพ์ `TYPE PAGE HEADING` โดยอัตโนมัติก่อน แล้วจึง
  พิมพ์เนื้อหาของ `NORMAL-DETAIL`
- `GENERATE WARNING-DETAIL` ครั้งที่สองไม่พิมพ์หัวกระดาษซ้ำอีก (เพราะยังอยู่หน้าเดิม) และพิมพ์เนื้อหา
  ของ `WARNING-DETAIL` ที่มีข้อความเตือน "** LOW! **" เพิ่มเข้ามา

### ผลลัพธ์ที่ได้จากการรันจริง

```
ITEM      QTY
APPLE       100
BANANA        3  ** LOW! **
```

### ข้อควรระวัง

- **`GENERATE` ต้องระบุชื่อกลุ่มรายงานที่มี `TYPE DETAIL` เท่านั้น** ไม่สามารถ `GENERATE` กลุ่ม
  `TYPE PAGE HEADING` หรือ `TYPE CONTROL FOOTING` ตรง ๆ ได้ (กลุ่มเหล่านั้นถูกพิมพ์โดยอัตโนมัติ
  ตามจังหวะของมันเอง)
- ต้องเรียก `INITIATE` ก่อน `GENERATE` เสมอ มิเช่นนั้นจะเกิด runtime error ทันที
  (จะสาธิตให้เห็นข้อผิดพลาดจริงในขั้นตอนที่ 375)
- การมีหลาย `TYPE DETAIL` มีประโยชน์มากในการแยก layout ของแถวปกติกับแถวพิเศษ (เช่น แถวเตือน,
  แถวผิดปกติ) แต่ต้องระวังอย่าตั้งชื่อกลุ่มซ้ำกับตัวแปรอื่นในโปรแกรม

### แบบฝึกหัดที่ 374.1

**โจทย์**: จงอธิบายว่าทำไมการมี `TYPE DETAIL` หลายกลุ่มในรายงานเดียวถึงมีประโยชน์ ยกตัวอย่างสถานการณ์
ทางธุรกิจ 1 กรณีที่ควรใช้เทคนิคนี้

**เฉลยแนวทาง**: มีประโยชน์เมื่อข้อมูลแต่ละแถวมีรูปแบบการแสดงผลต่างกันตามเงื่อนไข เช่น รายงานสต๊อกสินค้า
ที่ต้องการเน้นแถวที่สินค้าใกล้หมด (`WARNING-DETAIL` ที่มีข้อความเตือนเพิ่ม) แยกจากแถวสินค้าปกติ
(`NORMAL-DETAIL`) หรือรายงานผลการเรียนที่แสดงแถวนักเรียนที่สอบตกด้วยรูปแบบพิเศษ (ตัวหนา/มีเครื่องหมาย)
แยกจากแถวนักเรียนที่สอบผ่านปกติ โดยตรรกะ IF/EVALUATE ใน PROCEDURE DIVISION เป็นผู้เลือกว่าจะ
`GENERATE` กลุ่มไหนตามเงื่อนไขทางธุรกิจ

---

## ขั้นตอนที่ 375: INITIATE และ TERMINATE — วงจรชีวิตของรายงาน

### แนวคิด

รายงานทุกฉบับที่ใช้ Report Writer มีวงจรชีวิต 3 ช่วงเสมอ:

1. **`INITIATE report-name`** — เริ่มต้นรายงาน รีเซ็ตตัวนับบรรทัด/หน้า และตัวสะสมยอดรวม
   (accumulator) ทั้งหมดให้เป็นศูนย์ ต้องเรียกก่อน `GENERATE` ตัวแรกเสมอ
2. **`GENERATE`** (เรียกซ้ำได้หลายครั้ง) — พิมพ์แต่ละแถวข้อมูล ตามที่เรียนในขั้นตอนที่ 374
3. **`TERMINATE report-name`** — ปิดท้ายรายงาน พิมพ์ `TYPE CONTROL FOOTING FINAL` (ถ้ามี — จะสอน
   ใน Part 039) และ `TYPE PAGE FOOTING` ของหน้าสุดท้าย จากนั้นเคลียร์สถานะภายในของรายงานทั้งหมด

ทั้งสามคำสั่งนี้ต้องเรียกตามลำดับ `INITIATE` → `GENERATE` (0 หรือหลายครั้ง) → `TERMINATE` เสมอ
และไฟล์รายงานต้อง `OPEN` ไว้ก่อนแล้วตลอดช่วงเวลานี้

### ตัวอย่างโค้ด (สาธิตข้อผิดพลาดจริงเมื่อลืม INITIATE)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-NO-INITIATE-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "no_initiate_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS ITEM-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-ITEM-NAME  PIC X(10) VALUE "TEST".

       REPORT SECTION.
       RD  ITEM-REPORT.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) SOURCE WS-ITEM-NAME.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
      *> BUG: forgot to call INITIATE ITEM-REPORT before GENERATE.
           GENERATE DETAIL-LINE.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

โค้ดนี้ **คอมไพล์ผ่านโดยไม่มี error** เพราะไวยากรณ์ถูกต้องทุกประการ แต่เมื่อรันจริง COBOL runtime
จะตรวจพบว่ามีการ `GENERATE` ทั้งที่ยังไม่ได้ `INITIATE` รายงานนี้ และหยุดโปรแกรมทันทีพร้อม error
message ที่ชัดเจน — นี่เป็นตัวอย่างที่ดีว่า Report Writer มีการตรวจสอบวงจรชีวิตที่เข้มงวดกว่าการเขียน
`WRITE` ไฟล์ปกติมาก

### ผลลัพธ์ที่ได้จากการรันจริง

ทดสอบด้วย `cobc -x -o rw-no-initiate-demo rw-no-initiate-demo.cob && ./rw-no-initiate-demo`
ได้ผลลัพธ์จริงดังนี้ (exit code เป็น 1 คือรันไม่สำเร็จ):

```
libcob: error: GENERATE ITEM-REPORT but no INITIATE was done
libcob: warning: implicit CLOSE of REPORT-FILE ('no_initiate_demo.txt')
```

เมื่อแก้ไขโดยเพิ่ม `INITIATE ITEM-REPORT.` ก่อน `GENERATE DETAIL-LINE.` และเพิ่ม
`TERMINATE ITEM-REPORT.` ก่อน `CLOSE` โปรแกรมจะรันสำเร็จและได้ไฟล์ผลลัพธ์ที่มีข้อความ `TEST`
ตามปกติ

### ข้อควรระวัง

- **`INITIATE`/`TERMINATE` ต้องเรียกให้ครบคู่เสมอ** เรียก `INITIATE` ซ้ำสองครั้งติดกันโดยไม่มี
  `TERMINATE` คั่นกลางก็เป็นข้อผิดพลาดเชิงตรรกะเช่นกัน (แม้บางคอมไพเลอร์จะไม่ error ทันทีก็ตาม)
- ลืม `TERMINATE` จะทำให้ `TYPE CONTROL FOOTING FINAL` และ `TYPE PAGE FOOTING` ของหน้าสุดท้าย
  ไม่ถูกพิมพ์ออกมา ทำให้รายงานดูเหมือน "ขาดหาย" ตอนท้าย
- ควรจัดโครงสร้างโค้ดให้ `OPEN` → `INITIATE` → (ลูป `GENERATE`) → `TERMINATE` → `CLOSE` เรียงกัน
  ชัดเจนเสมอ เพื่อให้อ่านง่ายและไม่พลาดขั้นตอนใดขั้นตอนหนึ่ง

### แบบฝึกหัดที่ 375.1

**โจทย์**: จงเรียงลำดับคำสั่งต่อไปนี้ให้ถูกต้องตามวงจรชีวิตของ Report Writer:
`TERMINATE`, `CLOSE`, `GENERATE`, `OPEN`, `INITIATE`

**เฉลย**: `OPEN` → `INITIATE` → `GENERATE` → `TERMINATE` → `CLOSE`

---

## ขั้นตอนที่ 376: TYPE PAGE FOOTING และการขึ้นหน้าใหม่อัตโนมัติ

### แนวคิด

**`TYPE PAGE FOOTING`** คือกลุ่มรายการที่พิมพ์ที่ท้ายทุกหน้า (ตรงข้ามกับ `TYPE PAGE HEADING` ที่พิมพ์
ที่ต้นทุกหน้า) ตำแหน่งที่มันจะถูกพิมพ์กำหนดโดย clause **`FOOTING`** ใน `RD` — เมื่อบรรทัดรายละเอียด
เขียนไปจนถึงตำแหน่งที่กำหนดใน `FOOTING` แล้ว `GENERATE` ครั้งถัดไปจะสั่งพิมพ์ `TYPE PAGE FOOTING`
ปิดหน้าเดิม แล้วขึ้นหน้าใหม่พร้อมพิมพ์ `TYPE PAGE HEADING` โดยอัตโนมัติทันที

เราได้เห็นพฤติกรรมนี้แล้วบางส่วนในขั้นตอนที่ 372 คราวนี้จะโฟกัสที่ `TYPE PAGE FOOTING` โดยเฉพาะ
และดูว่าเมื่อ `TERMINATE` รายงานตอนที่ยังไม่ถึงท้ายหน้าพอดี ส่วนที่เหลือของหน้าจะเป็นบรรทัดว่างจนครบ
`PAGE LIMIT`

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-PAGE-FOOTING-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "footing_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS ITEM-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-ITEM-NAME  PIC X(10).
       01  WS-QTY        PIC 9(4).
       01  WS-COUNTER    PIC 9(2).

       REPORT SECTION.
       RD  ITEM-REPORT
           PAGE LIMIT 8 LINES
           HEADING 1
           FIRST DETAIL 4
           FOOTING 7.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(20) VALUE "PAGE FOOTING DEMO".

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) SOURCE WS-ITEM-NAME.
               05  COLUMN 12 PIC ZZZ9  SOURCE WS-QTY.

       01  TYPE PAGE FOOTING.
           03  LINE PLUS 1.
               05  COLUMN 1 PIC X(15) VALUE "-- PAGE END --".

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE ITEM-REPORT.
           PERFORM VARYING WS-COUNTER FROM 1 BY 1 UNTIL WS-COUNTER > 5
               MOVE "ITEM" TO WS-ITEM-NAME
               MOVE WS-COUNTER TO WS-QTY
               GENERATE DETAIL-LINE
           END-PERFORM.
           TERMINATE ITEM-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `RD ... PAGE LIMIT 8 LINES ... FOOTING 7.` — แต่ละหน้ายาว 8 บรรทัด และเนื้อหารายละเอียดต้อง
  ไม่เกินบรรทัดที่ 7 (บรรทัดที่ 7 หรือหลังจากนั้นสงวนไว้ให้ `TYPE PAGE FOOTING`)
- `FIRST DETAIL 4` — บรรทัดรายละเอียดแรกเริ่มที่บรรทัด 4 (เว้นที่บรรทัด 1-3 ให้หัวกระดาษและช่องว่าง)
  ดังนั้นแต่ละหน้าพิมพ์รายละเอียดได้ที่บรรทัด 4, 5, 6, 7 รวม **4 บรรทัดพอดี** ก่อนที่บรรทัดที่ 5
  (รายการที่ 5) จะต้องเริ่มหน้าใหม่
- เมื่อ `GENERATE` ครั้งที่ 5 มาถึง ตำแหน่งบรรทัดปัจจุบันจะเกินขอบเขต `FOOTING 7` ที่ตั้งไว้
  Report Writer จึงพิมพ์ `TYPE PAGE FOOTING` ("-- PAGE END --") ปิดหน้าที่ 1 แล้วขึ้นหน้าใหม่พร้อม
  พิมพ์ `TYPE PAGE HEADING` ซ้ำก่อนพิมพ์รายการที่ 5

### ผลลัพธ์ที่ได้จากการรันจริง

```
PAGE FOOTING DEMO


ITEM          1
ITEM          2
ITEM          3
ITEM          4
-- PAGE END --
PAGE FOOTING DEMO


ITEM          5
-- PAGE END --
```

ตามด้วยบรรทัดว่างจนครบ `PAGE LIMIT 8 LINES` ของหน้าสุดท้าย (เพราะ `TERMINATE` ถูกเรียกตอนที่
หน้ายังไม่เต็ม COBOL จะเติมช่องว่างที่เหลือให้ครบตามจำนวนบรรทัดที่กำหนด เพื่อให้ทุกหน้าของรายงานมี
ความยาวเท่ากันเสมอ ซึ่งสำคัญมากเมื่อพิมพ์ลงกระดาษจริงหรือ Mainframe printer)

### ข้อควรระวัง

- **ถ้าไม่ประกาศ `FOOTING` ใน `RD`** COBOL จะถือว่า `FOOTING` เท่ากับ `PAGE LIMIT` (คือไม่เว้นที่
  ให้ `TYPE PAGE FOOTING` โดยเฉพาะ) ซึ่งอาจทำให้ท้ายกระดาษพิมพ์ชนกับบรรทัดสุดท้ายของรายละเอียด
- ตัวเลข `HEADING`, `FIRST DETAIL`, `FOOTING`, `PAGE LIMIT` ต้องคำนวณให้พอดีกับจำนวนบรรทัดที่
  หัวกระดาษ/ท้ายกระดาษของคุณต้องใช้จริง มิเช่นนั้นอาจเหลือช่องว่างเกินความจำเป็นหรือพิมพ์ทับกัน
- การเติมบรรทัดว่างจนครบหน้าตอน `TERMINATE` เป็นพฤติกรรมปกติของ Report Writer ไม่ใช่บั๊ก
  แต่ผู้เรียนใหม่มักแปลกใจเมื่อเห็นบรรทัดว่างจำนวนมากในไฟล์ผลลัพธ์

### แบบฝึกหัดที่ 376.1

**โจทย์**: จากตัวอย่างข้างต้น หากเปลี่ยน `PAGE LIMIT` เป็น 10 บรรทัด และ `FOOTING` เป็น 9
(โดยค่าอื่นเท่าเดิม) แต่ละหน้าจะพิมพ์รายการได้กี่รายการก่อนขึ้นหน้าใหม่

**เฉลย**: บรรทัดรายละเอียดเริ่มที่บรรทัด 4 (`FIRST DETAIL 4`) และต้องไม่เกินบรรทัด 9
(`FOOTING 9`) ดังนั้นพิมพ์ได้ที่บรรทัด 4, 5, 6, 7, 8, 9 รวม **6 รายการ** ต่อหน้า ก่อนที่รายการ
ที่ 7 จะต้องขึ้นหน้าใหม่

---

## ขั้นตอนที่ 377: PICTURE Editing ในรายงาน — ตัวเลขที่อ่านง่ายโดยอัตโนมัติ

### แนวคิด

จุดแข็งสำคัญของ Report Writer ที่เราเห็นผ่าน ๆ มาแล้วคือความสามารถแปลงตัวเลขดิบให้อยู่ในรูปแบบที่
อ่านง่ายสำหรับมนุษย์โดยอัตโนมัติผ่าน **PICTURE editing symbols** — เทคนิคเดียวกับที่เรียนใน Part 006
แต่นำมาใช้ใน `PIC` ของ report entry ที่มี `SOURCE` โดยตรง สัญลักษณ์ที่ใช้บ่อยที่สุด ได้แก่:

| สัญลักษณ์ | ความหมาย |
|---|---|
| `Z` | ระงับเลขศูนย์นำหน้า (zero suppression) แสดงเป็นช่องว่างแทน |
| `,` | ใส่จุลภาคคั่นหลักพัน |
| `.` | จุดทศนิยม |
| `-` | เครื่องหมายลบ (แสดงเฉพาะเมื่อค่าติดลบ) |
| `$` | เครื่องหมายสกุลเงิน (ตัวอย่างนี้ใช้ตัวเลขไทยจึงเน้น `,` และ `.` เป็นหลัก) |

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-PIC-EDITING-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "editing_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS FINANCE-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-ACCOUNT-NAME  PIC X(12).
       01  WS-BALANCE       PIC S9(7)V99.

       REPORT SECTION.
       RD  FINANCE-REPORT
           PAGE LIMIT 30 LINES.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1  PIC X(12) VALUE "ACCOUNT".
               05  COLUMN 16 PIC X(15) VALUE "BALANCE".

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(12)       SOURCE WS-ACCOUNT-NAME.
               05  COLUMN 16 PIC -,ZZZ,ZZ9.99 SOURCE WS-BALANCE.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE FINANCE-REPORT.

           MOVE "SAVINGS"    TO WS-ACCOUNT-NAME.
           MOVE 125000.50    TO WS-BALANCE.
           GENERATE DETAIL-LINE.

           MOVE "CREDIT CARD" TO WS-ACCOUNT-NAME.
           MOVE -3250.75      TO WS-BALANCE.
           GENERATE DETAIL-LINE.

           TERMINATE FINANCE-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `PIC -,ZZZ,ZZ9.99` — ตัวเลขที่มีเครื่องหมายลบนำหน้า (เฉพาะเมื่อติดลบเท่านั้น), จุลภาคคั่นหลักพัน,
  ระงับเลขศูนย์นำหน้าด้วย `Z`, และจุดทศนิยม 2 ตำแหน่ง
- `WS-BALANCE` เป็น `PIC S9(7)V99` (signed) — เมื่อค่าเป็นบวก (125000.50) เครื่องหมาย `-`
  จะไม่แสดง (เป็นช่องว่างแทน) แต่เมื่อค่าเป็นลบ (-3250.75) เครื่องหมาย `-` จะปรากฏในตำแหน่งคงที่
  ของ `-` ตัวเดียวที่ระบุไว้ (**ตำแหน่งซ้ายสุดของ PIC เสมอ ไม่ใช่ "ลอย" ไปติดกับหลักแรกที่มีค่า**)
- สังเกตว่าตัวเลข 125000.50 ถูกจัดรูปแบบเป็น "125,000.50" และ -3250.75 เป็น "-   3,250.75"
  (มีช่องว่างคั่นระหว่างเครื่องหมายลบกับตัวเลขที่เหลือ) โดยที่โปรแกรมเมอร์ไม่ต้องเขียนโค้ดแปลงรูปแบบ
  เองแม้แต่บรรทัดเดียว — หากต้องการให้เครื่องหมายลบ "ลอย" ไปติดกับหลักแรกที่มีค่าจริง (floating
  sign) ต้องใช้เครื่องหมาย `-` ซ้ำแทนที่ตำแหน่ง `Z` ทุกตำแหน่งก่อนหลักสุดท้าย เช่น
  `PIC -,---,--9.99` (ทดสอบแล้วให้ผลลัพธ์ `-3,250.75` โดยเครื่องหมายลบอยู่ติดกับเลข 3 พอดี) —
  ข้อควรระวังคือ **ห้ามผสม `-` แบบลอยกับ `Z` ในตำแหน่งก่อนจุดทศนิยมเดียวกัน** เช่น
  `PIC ---,ZZZ,ZZ9.99` จะ compile error ทันที เพราะ COBOL ไม่ยอมให้ผสมสัญลักษณ์ทั้งสองแบบ
  ในกลุ่มเดียวกัน

### ผลลัพธ์ที่ได้จากการรันจริง

```
ACCOUNT        BALANCE
SAVINGS          125,000.50
CREDIT CARD    -   3,250.75
```

### ข้อควรระวัง

- ความกว้างของ `PIC` ที่ใช้ editing symbols ต้องเผื่อพื้นที่ให้พอกับตัวเลขที่ใหญ่ที่สุดที่จะเกิดขึ้นจริง
  รวมเครื่องหมายลบและจุลภาคด้วย (`-,ZZZ,ZZ9.99` รองรับได้ถึง -9,999,999.99) ถ้าค่าจริงเกินขนาดที่
  กำหนดไว้ ตัวเลขจะถูกตัดหลักสูงสุดทิ้งโดยไม่มี error เตือน (overflow แบบเงียบ)
- `SOURCE` field ที่เป็นตัวเลขต้องประกาศเป็น `PIC S9...` (signed) หากต้องการให้เครื่องหมายลบ
  ทำงานถูกต้อง หากเป็น `PIC 9...` ธรรมดา (unsigned) ค่าจะไม่มีทางติดลบได้ตั้งแต่แรก
- ผลลัพธ์การ editing นี้ใช้ได้เฉพาะตอนพิมพ์ในรายงานเท่านั้น ค่าดิบใน `WS-BALANCE` ที่เก็บใน
  หน่วยความจำยังคงเป็นตัวเลขไม่มีจุลภาคตามปกติ

### แบบฝึกหัดที่ 377.1

**โจทย์**: จงปรับ `PIC` ของฟิลด์ `WS-BALANCE` ในรายงานให้แสดงเครื่องหมายสกุลเงินบาท (ใช้ตัวอักษร
`B` แทนไม่ได้เพราะเป็นสัญลักษณ์ blank insertion — ให้ใช้ข้อความ "THB" คงที่แทน) นำหน้าตัวเลข
โดยไม่ต้องเปลี่ยนตัวแปรใน WORKING-STORAGE

**เฉลยแนวทาง**: เพิ่มฟิลด์ literal ข้อความคงที่แยกต่างหากไว้ก่อนคอลัมน์ของตัวเลข เนื่องจาก Report
Writer ไม่มีสัญลักษณ์ PIC สำหรับสกุลเงินไทยโดยตรง:
```cobol
       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(12)        SOURCE WS-ACCOUNT-NAME.
               05  COLUMN 16 PIC X(3)         VALUE "THB".
               05  COLUMN 20 PIC -,ZZZ,ZZ9.99 SOURCE WS-BALANCE.
```

---

## ขั้นตอนที่ 378: การอ่านข้อมูลจากไฟล์จริงมาสร้างรายงาน

### แนวคิด

ในโลกจริง รายงานแทบไม่เคยใช้ค่าที่ `MOVE` เข้าไปตรง ๆ ในโค้ด แต่จะอ่านข้อมูลจากไฟล์ (ที่เรียนใน
Part 023-026) แล้วนำแต่ละ record มา `GENERATE` เป็นบรรทัดรายงานทีละแถว ขั้นตอนนี้จะรวมความรู้เรื่อง
`OPEN INPUT`/`READ`/`AT END` จาก Part 025 เข้ากับ Report Writer เป็นครั้งแรก

### ตัวอย่างโค้ด

ไฟล์ข้อมูลนำเข้า `employees.txt` (LINE SEQUENTIAL, แต่ละบรรทัดคือชื่อ 10 ตัวอักษร + แผนก
10 ตัวอักษร + เงินเดือน 9 หลัก มีทศนิยม 2 ตำแหน่งโดยนัย):

```
SIRIPORN  SALES     003500000
WICHAI    IT        004200000
NAPAPORN  HR        002800000
```

โปรแกรม COBOL:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-FILE-TO-REPORT-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT EMPLOYEE-FILE ASSIGN TO "employees.txt"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT REPORT-FILE ASSIGN TO "payroll_report.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  EMPLOYEE-FILE.
       01  EMPLOYEE-RECORD.
           05  EMP-NAME    PIC X(10).
           05  EMP-DEPT    PIC X(10).
           05  EMP-SALARY  PIC 9(7)V99.

       FD  REPORT-FILE
           REPORT IS PAYROLL-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG   PIC X VALUE "N".

       REPORT SECTION.
       RD  PAYROLL-REPORT
           PAGE LIMIT 30 LINES.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1  PIC X(30) VALUE "EMPLOYEE PAYROLL REPORT".
           03  LINE 2.
               05  COLUMN 1  PIC X(10) VALUE "NAME".
               05  COLUMN 12 PIC X(10) VALUE "DEPT".
               05  COLUMN 24 PIC X(10) VALUE "SALARY".

       01  PAYROLL-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10)     SOURCE EMP-NAME.
               05  COLUMN 12 PIC X(10)     SOURCE EMP-DEPT.
               05  COLUMN 24 PIC ZZZ,ZZ9.99 SOURCE EMP-SALARY.

       PROCEDURE DIVISION.
           OPEN INPUT EMPLOYEE-FILE.
           OPEN OUTPUT REPORT-FILE.
           INITIATE PAYROLL-REPORT.

           READ EMPLOYEE-FILE
               AT END MOVE "Y" TO WS-EOF-FLAG
           END-READ.
           PERFORM UNTIL WS-EOF-FLAG = "Y"
               GENERATE PAYROLL-DETAIL
               READ EMPLOYEE-FILE
                   AT END MOVE "Y" TO WS-EOF-FLAG
               END-READ
           END-PERFORM.

           TERMINATE PAYROLL-REPORT.
           CLOSE EMPLOYEE-FILE.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- โครงสร้างลูป `READ ... AT END` ตามด้วย `PERFORM UNTIL WS-EOF-FLAG = "Y"` คือรูปแบบมาตรฐานการ
  อ่านไฟล์ทีละ record ที่เรียนมาแล้วใน Part 025 — Report Writer ไม่ได้เปลี่ยนวิธีอ่านไฟล์แต่อย่างใด
- จุดสำคัญคือ `SOURCE EMP-NAME`, `SOURCE EMP-DEPT`, `SOURCE EMP-SALARY` อ้างอิงตัวแปรใน
  **FILE SECTION** ของ `EMPLOYEE-FILE` โดยตรง ไม่จำเป็นต้อง `MOVE` เข้า WORKING-STORAGE ก่อน
  เพราะ `SOURCE` ดึงค่า ณ ขณะ `GENERATE` ซึ่งเป็นช่วงที่ `EMPLOYEE-RECORD` ยังมีข้อมูลของ record
  ปัจจุบันอยู่ (ยังไม่ถูก `READ` ทับด้วย record ถัดไป)
- แต่ละครั้งที่อ่าน record ใหม่สำเร็จ เราสั่ง `GENERATE PAYROLL-DETAIL` ทันทีก่อนจะไปอ่าน record
  ถัดไป — รูปแบบนี้ ("อ่านมา 1 แถว, GENERATE 1 แถว") คือรูปแบบที่พบบ่อยที่สุดในโปรแกรมรายงานจริง

### ผลลัพธ์ที่ได้จากการรันจริง

```
EMPLOYEE PAYROLL REPORT
NAME       DEPT        SALARY
SIRIPORN   SALES        35,000.00
WICHAI     IT           42,000.00
NAPAPORN   HR           28,000.00
```

### ข้อควรระวัง

- **ต้องระวังลำดับ `GENERATE` กับ `READ` ให้ถูกต้อง**: ในโค้ดข้างต้น เรา `GENERATE` ก่อน `READ`
  record ถัดไปเสมอ เพื่อให้แน่ใจว่าข้อมูลใน `EMPLOYEE-RECORD` ตอน `GENERATE` ยังเป็นของ record
  ที่เพิ่งอ่านมาจริง ๆ หากสลับลำดับผิดจะได้ผลลัพธ์ที่ "เลื่อน" ไปหนึ่งแถวหรือพิมพ์ record สุดท้ายซ้ำ
- ต้อง `OPEN INPUT` ไฟล์ข้อมูล และ `OPEN OUTPUT` ไฟล์รายงานแยกกันให้ถูกโหมด (สลับกันจะ error
  ทันทีตามที่เรียนใน Part 025)
- ถ้าไฟล์ข้อมูลว่างเปล่า (ไม่มี record เลย) ลูปจะไม่ `GENERATE` แม้แต่ครั้งเดียว แต่รายงานยังคงมี
  `TYPE PAGE HEADING` พิมพ์ออกมาหรือไม่ ขึ้นอยู่กับว่ามีการ `GENERATE` เกิดขึ้นจริงหรือไม่ — หาก
  ไม่มี `GENERATE` เลยแม้แต่ครั้งเดียว **จะไม่มีการพิมพ์หัวกระดาษใด ๆ ออกมาเลย** เพราะหัวกระดาษ
  ถูกทริกเกอร์โดย `GENERATE` ครั้งแรกเท่านั้น (ตามที่เรียนในขั้นตอนที่ 374)

### แบบฝึกหัดที่ 378.1

**โจทย์**: จงอธิบายว่าทำไมตัวอย่างข้างต้นจึงใช้ `SOURCE EMP-NAME` (อ้างอิงตัวแปรใน FILE SECTION
โดยตรง) แทนที่จะ `MOVE EMP-NAME TO WS-NAME` แล้วค่อย `SOURCE WS-NAME` เหมือนตัวอย่างก่อนหน้า
ในขั้นตอนนี้

**เฉลยแนวทาง**: เพราะ `SOURCE` สามารถอ้างอิงตัวแปรใด ๆ ก็ได้ใน DATA DIVISION ไม่จำเป็นต้องเป็น
WORKING-STORAGE เท่านั้น เมื่อ record ถูกอ่านเข้ามาด้วย `READ` แล้ว ข้อมูลจะอยู่ใน `EMPLOYEE-RECORD`
(FILE SECTION) พร้อมใช้งานทันที การ `MOVE` ไป WORKING-STORAGE ก่อนจะเป็นขั้นตอนที่ไม่จำเป็น
เพิ่มโค้ดโดยใช่เหตุ เว้นแต่จะต้องการเก็บค่าไว้ใช้ต่อหลังจากอ่าน record ถัดไปแล้ว (ซึ่งจะทำให้ค่าใน
FILE SECTION ถูกเขียนทับ)

---

## ขั้นตอนที่ 379: ทางเลือกแบบ Manual Report ด้วย DISPLAY/WRITE

### แนวคิด

แม้ Report Writer จะสะดวกมาก แต่ในทางปฏิบัติจริง (โดยเฉพาะในโค้ด Legacy จำนวนมาก หรือทีมที่ต้องการ
ควบคุม layout ทุกรายละเอียดด้วยตัวเอง) โปรแกรมเมอร์จำนวนไม่น้อยเลือกเขียนรายงานด้วยวิธี **"Manual
Report"** คือใช้ `WRITE` เข้าไฟล์ปกติ (หรือ `DISPLAY`) พร้อมคำนวณตำแหน่งบรรทัด/หน้าด้วยตัวแปรนับเอง
ทั้งหมด ขั้นตอนนี้จะเปรียบเทียบทั้งสองแนวทางให้เห็นภาพชัดเจน เพื่อให้คุณเลือกใช้ได้อย่างเหมาะสมเมื่อ
ต้องทำงานกับโค้ดจริงในอนาคต

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MANUAL-REPORT-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "manual_report.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE.
       01  REPORT-LINE   PIC X(40).

       WORKING-STORAGE SECTION.
       01  WS-ITEM-NAME       PIC X(10).
       01  WS-QTY             PIC 9(4).
       01  WS-EDITED-QTY      PIC ZZZ9.
       01  WS-LINE-COUNT      PIC 9(2) VALUE 0.
       01  WS-PAGE-COUNT      PIC 9(2) VALUE 1.
       01  WS-COUNTER         PIC 9(2).
       01  WS-DETAIL-LINE.
           05  FILLER         PIC X(10).
           05  FILLER         PIC X(2)  VALUE SPACES.
           05  DL-QTY         PIC ZZZ9.

      *> A hand-rolled "page heading" paragraph, written manually.
       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.

           MOVE "ITEM      QTY" TO REPORT-LINE.
           WRITE REPORT-LINE.
           MOVE 1 TO WS-LINE-COUNT.

           PERFORM VARYING WS-COUNTER FROM 1 BY 1 UNTIL WS-COUNTER > 6
               MOVE "ITEM" TO WS-ITEM-NAME
               MOVE WS-COUNTER TO WS-QTY

      *> We must manually check for page overflow ourselves --
      *> Report Writer would have done this step automatically.
               IF WS-LINE-COUNT >= 4
                   MOVE SPACES TO REPORT-LINE
                   WRITE REPORT-LINE
                   ADD 1 TO WS-PAGE-COUNT
                   MOVE "ITEM      QTY" TO REPORT-LINE
                   WRITE REPORT-LINE
                   MOVE 1 TO WS-LINE-COUNT
               END-IF

               MOVE WS-ITEM-NAME TO WS-DETAIL-LINE(1:10)
               MOVE WS-QTY       TO DL-QTY
               MOVE WS-DETAIL-LINE TO REPORT-LINE
               WRITE REPORT-LINE
               ADD 1 TO WS-LINE-COUNT
           END-PERFORM.

           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- ทุกอย่างที่ Report Writer เคยทำให้ฟรี (นับบรรทัด, ตรวจสอบเมื่อหน้าเต็ม, พิมพ์หัวกระดาษซ้ำ)
  ในโค้ดนี้ต้อง**เขียนด้วยตัวเองทั้งหมด**: `WS-LINE-COUNT` นับจำนวนบรรทัดรายละเอียดในหน้าปัจจุบัน
  และ `IF WS-LINE-COUNT >= 4` คือการจำลอง "page break" ด้วยมือ
- `WS-DETAIL-LINE` เป็น group item ที่ประกอบด้วยฟิลด์ย่อยเพื่อจัดตำแหน่งคอลัมน์เอง (แทนที่จะมี
  `COLUMN` clause ให้ใช้เหมือน Report Writer)
- โค้ดนี้ให้ผลลัพธ์คล้ายกับตัวอย่างในขั้นตอนที่ 372 แต่ต้องเขียนตรรกะการขึ้นหน้าใหม่เองทั้งหมด

### ผลลัพธ์ที่ได้จากการรันจริง

```
ITEM      QTY
ITEM          1
ITEM          2
ITEM          3
ITEM          4

ITEM      QTY
ITEM          5
ITEM          6
```

### เมื่อไหร่ควรใช้แบบไหน

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| รายงานมีโครงสร้างซับซ้อน มีหัวกระดาษ/ท้ายกระดาษ/ยอดรวมหลายระดับ | **Report Writer** — ลดโค้ดคำนวณตำแหน่งด้วยมือได้มาก |
| ต้องการควบคุม layout พิเศษที่ Report Writer ทำไม่ได้ (เช่น กราฟข้อความ, ตารางไม่สม่ำเสมอ) | **Manual (WRITE/DISPLAY)** |
| ทำงานกับโค้ด Legacy ที่เขียนแบบ Manual มาแต่เดิม | **Manual** (เพื่อความสอดคล้องกับโค้ดเดิม) |
| ต้องพอร์ตโค้ดไปคอมไพเลอร์ COBOL อื่นที่รองรับ Report Writer ไม่เต็มรูปแบบ | **Manual** (เพื่อความเข้ากันได้สูงสุด) |

### ข้อควรระวัง

- แนวทาง Manual Report เสี่ยงต่อบั๊กด้าน "นับบรรทัดผิด" มากกว่า Report Writer มาก เพราะทุกจุดที่
  เพิ่ม/ลด `WS-LINE-COUNT` ต้องแก้ไขด้วยมือหากมีการเปลี่ยนแปลง layout ในอนาคต
- ทีมพัฒนาโค้ด Legacy จำนวนมากใช้ Manual Report เพราะ Report Writer **ไม่ได้รับการรองรับเท่ากัน
  ในทุกคอมไพเลอร์ COBOL** (บางระบบ Mainframe รุ่นเก่ามีข้อจำกัดเรื่อง Report Writer มากกว่า
  GnuCOBOL ที่เราทดสอบใน Part นี้) จึงควรรู้ทั้งสองแนวทางไว้เสมอ

### แบบฝึกหัดที่ 379.1

**โจทย์**: จงเขียนย่อหน้าอธิบายเปรียบเทียบข้อดี-ข้อเสียของ Report Writer กับ Manual Report
คนละ 2 ข้อ

**เฉลยแนวทาง**:
- **Report Writer**: ข้อดี — ลดโค้ดคำนวณตำแหน่งบรรทัด/หน้าเอง, PICTURE editing อัตโนมัติ
  ข้อเสีย — ต้องเรียนรู้ syntax เฉพาะทาง (`RD`, `TYPE`, clauses ต่าง ๆ), การรองรับในบางคอมไพเลอร์
  อาจไม่สมบูรณ์
- **Manual Report**: ข้อดี — ควบคุมทุกรายละเอียดได้เต็มที่, ใช้ได้กับทุกคอมไพเลอร์ COBOL แน่นอน
  ข้อเสีย — ต้องเขียนโค้ดคำนวณตำแหน่งเองทั้งหมด, เสี่ยงบั๊กเรื่องนับบรรทัด/หน้าผิดพลาดมากกว่า

---

## ขั้นตอนที่ 380: โปรแกรมรวบยอด — รายงานยอดขายสินค้าแบบครบวงจร

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่รวมทุกความรู้จาก 9 ขั้นตอนที่ผ่านมาเข้าด้วยกัน: อ่านข้อมูลจากไฟล์จริง,
มี `TYPE REPORT HEADING` (หน้าปก), `TYPE PAGE HEADING` (หัวตาราง), `TYPE DETAIL` พร้อม
PICTURE editing, `TYPE PAGE FOOTING`, และวงจรชีวิต `INITIATE`/`GENERATE`/`TERMINATE` ที่ถูกต้อง

### ตัวอย่างโค้ด

ไฟล์ข้อมูล `stock_items.txt`:

```
LAPTOP    ELECTRONIC0025000000015
MOUSE     ELECTRONIC0000350000120
CHAIR     FURNITURE 0001200000045
DESK      FURNITURE 0003500000012
```

(รูปแบบ: ชื่อสินค้า 10 ตัวอักษร, หมวดหมู่ 10 ตัวอักษร, ราคา 9 หลักมีทศนิยม 2 ตำแหน่งโดยนัย,
จำนวนคงเหลือ 4 หลัก)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RW-CAPSTONE-STOCK-REPORT.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT STOCK-FILE ASSIGN TO "stock_items.txt"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT REPORT-FILE ASSIGN TO "stock_final_report.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  STOCK-FILE.
       01  STOCK-RECORD.
           05  STK-NAME      PIC X(10).
           05  STK-CATEGORY  PIC X(10).
           05  STK-PRICE     PIC 9(7)V99.
           05  STK-QTY       PIC 9(4).

       FD  REPORT-FILE
           REPORT IS STOCK-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG    PIC X VALUE "N".

       REPORT SECTION.
       RD  STOCK-REPORT
           PAGE LIMIT 15 LINES
           HEADING 1
           FIRST DETAIL 5
           FOOTING 13.

       01  TYPE REPORT HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(30) VALUE "WAREHOUSE STOCK REPORT".

       01  TYPE PAGE HEADING.
           03  LINE 3.
               05  COLUMN 1  PIC X(10) VALUE "ITEM".
               05  COLUMN 12 PIC X(10) VALUE "CATEGORY".
               05  COLUMN 24 PIC X(8)  VALUE "PRICE".
               05  COLUMN 35 PIC X(5)  VALUE "QTY".

       01  STOCK-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10)      SOURCE STK-NAME.
               05  COLUMN 12 PIC X(10)      SOURCE STK-CATEGORY.
               05  COLUMN 24 PIC ZZ,ZZ9.99  SOURCE STK-PRICE.
               05  COLUMN 35 PIC ZZZ9       SOURCE STK-QTY.

       01  TYPE PAGE FOOTING.
           03  LINE PLUS 1.
               05  COLUMN 1 PIC X(20) VALUE "-- END OF PAGE --".

       PROCEDURE DIVISION.
           OPEN INPUT STOCK-FILE.
           OPEN OUTPUT REPORT-FILE.
           INITIATE STOCK-REPORT.

           READ STOCK-FILE
               AT END MOVE "Y" TO WS-EOF-FLAG
           END-READ.
           PERFORM UNTIL WS-EOF-FLAG = "Y"
               GENERATE STOCK-DETAIL
               READ STOCK-FILE
                   AT END MOVE "Y" TO WS-EOF-FLAG
               END-READ
           END-PERFORM.

           TERMINATE STOCK-REPORT.
           CLOSE STOCK-FILE.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

โปรแกรมนี้ผสานทุกเทคนิคที่เรียนมาใน Part นี้: `FD ... REPORT IS`, `RD` พร้อม clause ควบคุม
การขึ้นหน้า, `TYPE REPORT HEADING` สำหรับหน้าปก, `TYPE PAGE HEADING`/`TYPE PAGE FOOTING`
สำหรับหัว-ท้ายกระดาษทุกหน้า, `TYPE DETAIL` พร้อม PICTURE editing, และการอ่านไฟล์จริงมา
`GENERATE` ทีละ record ตามรูปแบบมาตรฐานที่เรียนในขั้นตอนที่ 378

### ผลลัพธ์ที่ได้จากการรันจริง

```
WAREHOUSE STOCK REPORT

ITEM       CATEGORY    PRICE      QTY

LAPTOP     ELECTRONIC  25,000.00    15
MOUSE      ELECTRONIC     350.00   120
CHAIR      FURNITURE    1,200.00    45
DESK       FURNITURE    3,500.00    12
-- END OF PAGE --
```

(ด้วยข้อมูล 4 รายการและ `FIRST DETAIL 5`/`FOOTING 13` ทั้งหมดพอดีอยู่ในหน้าเดียว หากเพิ่มข้อมูล
มากกว่า 8-9 รายการ จะเห็นการขึ้นหน้าใหม่พร้อม `TYPE PAGE HEADING` พิมพ์ซ้ำ แต่ `TYPE REPORT
HEADING` จะไม่ปรากฏซ้ำอีกในหน้าถัดไป)

### ข้อควรระวัง

- โปรแกรมนี้ยังไม่มีการคำนวณยอดรวม (subtotal/grand total) ตามหมวดหมู่สินค้า — ความสามารถนี้
  ต้องใช้ **`CONTROLS ARE`** และ **`TYPE CONTROL FOOTING`** ซึ่งจะสอนอย่างละเอียดใน **Part 039**
  ต่อจากนี้
- ควรทดสอบด้วยจำนวนข้อมูลหลายขนาด (น้อยกว่า, เท่ากับ, มากกว่าความจุ 1 หน้า) เพื่อให้มั่นใจว่า
  `RD` clauses ตั้งค่าไว้ถูกต้อง ก่อนนำไปใช้กับข้อมูลจริงจำนวนมาก

### แบบฝึกหัดที่ 380.1

**โจทย์**: จงเพิ่มข้อมูลอีก 6 รายการเข้าไปในไฟล์ `stock_items.txt` (รวมเป็น 10 รายการ) แล้วรัน
โปรแกรมอีกครั้ง สังเกตว่ารายงานขึ้นหน้าใหม่กี่ครั้ง และ `TYPE PAGE HEADING` ปรากฏกี่ครั้ง

**เฉลยแนวทาง**: ด้วย `FIRST DETAIL 5` และ `FOOTING 13` แต่ละหน้าพิมพ์รายละเอียดได้ที่บรรทัด
5 ถึง 13 รวม 9 รายการต่อหน้า ดังนั้นข้อมูล 10 รายการจะทำให้ขึ้นหน้าใหม่ 1 ครั้ง (หน้าแรกมี 9 รายการ
หน้าที่สองมี 1 รายการ) และ `TYPE PAGE HEADING` จะปรากฏ **2 ครั้ง** (ครั้งละ 1 ครั้งต่อหน้า) ในขณะที่
`TYPE REPORT HEADING` ยังคงปรากฏเพียง **1 ครั้ง** ที่ต้นหน้าแรกเท่านั้น

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้ **Report Writer Feature** ของ COBOL อย่างครบถ้วน โดยทดสอบคอมไพล์และ
รันจริงด้วย GnuCOBOL 4.0 (early-dev) ทุกตัวอย่าง:

- `RD` Entry และการผูก `FD ... REPORT IS` เข้ากับ `REPORT SECTION`
- `TYPE PAGE HEADING` (พิมพ์ทุกหน้า) เทียบกับ `TYPE REPORT HEADING` (พิมพ์ครั้งเดียว)
- `TYPE DETAIL`, `SOURCE` clause และการวางตำแหน่งด้วย `LINE`/`COLUMN`
- กลไกเบื้องหลังคำสั่ง `GENERATE` ที่จัดการหัวกระดาษ/ท้ายกระดาษ/การขึ้นหน้าใหม่ให้อัตโนมัติ
- วงจรชีวิต `INITIATE`/`GENERATE`/`TERMINATE` และข้อผิดพลาดจริงเมื่อลืม `INITIATE`
- `TYPE PAGE FOOTING` และการขึ้นหน้าใหม่อัตโนมัติเมื่อถึง `FOOTING`/`PAGE LIMIT`
- PICTURE editing ในรายงาน (`Z`, `,`, `.`, `-`) สำหรับตัวเลขที่อ่านง่าย
- การอ่านข้อมูลจากไฟล์จริงมาสร้างรายงานด้วยรูปแบบ "อ่าน 1 แถว, GENERATE 1 แถว"
- ทางเลือก Manual Report ด้วย `WRITE`/`DISPLAY` และเมื่อไหร่ควรใช้แบบไหน
- โปรแกรมรวบยอดรายงานสต๊อกสินค้าที่ผสานทุกเทคนิคเข้าด้วยกัน

จากการทดสอบจริง GnuCOBOL รองรับ Report Writer ในระดับที่ใช้งานได้ดีมากสำหรับฟีเจอร์พื้นฐานที่สอน
ใน Part นี้ทั้งหมด แต่ Part นี้ยังไม่ได้แตะเรื่อง **Control Break** (การขึ้นกลุ่มใหม่ตามคีย์ข้อมูล
พร้อมยอดรวมแต่ละกลุ่ม) ซึ่งเป็นความสามารถขั้นสูงที่สำคัญมากในรายงานธุรกิจจริง เช่น รายงานยอดขาย
แยกตามภูมิภาค หรือรายงานเงินเดือนแยกตามแผนก **Part 040** จะไม่ได้สอนเรื่องนี้ต่อทันที แต่จะพา
คุณไปสำรวจ **Screen Section** สำหรับสร้าง UI แบบ Text-mode ก่อน ส่วน Control Break แบบเต็มรูป
จะอยู่ใน **Part 039** ที่คุณสามารถอ่านต่อได้ทันที

**[← กลับไป Part 037](part-037-intrinsic-functions-string-stats.md)** | **[ไปยัง Part 039: Report Writer ขั้นสูง — Control Breaks →](part-039-report-writer-control-breaks.md)**
