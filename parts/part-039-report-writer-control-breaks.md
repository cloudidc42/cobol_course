# Part 039: Report Writer ขั้นสูง: Control Breaks (ขั้นตอนที่ 381–390)

## คำนำของ Part นี้

Part 038 พาเราไปรู้จัก Report Writer ในระดับพื้นฐาน — พิมพ์หัวกระดาษ พิมพ์รายละเอียดทีละแถว
พิมพ์ท้ายกระดาษ และขึ้นหน้าใหม่อัตโนมัติ แต่รายงานธุรกิจจริงแทบทุกฉบับต้องการมากกว่านั้น: เมื่อข้อมูล
เปลี่ยนกลุ่ม (เช่น เปลี่ยนภูมิภาค เปลี่ยนแผนก) ต้องขึ้นหัวข้อกลุ่มใหม่ พิมพ์ยอดรวมของกลุ่มก่อนหน้า
แล้วจึงเริ่มกลุ่มถัดไป และเมื่อจบรายงานทั้งฉบับต้องมียอดรวมทั้งหมด (grand total) ปิดท้าย

ความสามารถนี้เรียกว่า **Control Break** และ Report Writer มีกลไกเฉพาะทางรองรับมันโดยตรงผ่าน
**`CONTROLS ARE`**, **`TYPE CONTROL HEADING`**, **`TYPE CONTROL FOOTING`** และ **`SUM`** clause
Part นี้ได้ทดสอบคอมไพล์และรันจริงทุกตัวอย่างด้วย GnuCOBOL 4.0 (early-dev) เช่นเดียวกับ Part 038
และพบทั้งความสามารถที่ทำงานได้ดีมาก (single-level และ multi-level control break, `SUM`,
`CONTROL FOOTING FINAL`, `NEXT GROUP`) และ **ข้อจำกัด/พฤติกรรมที่ควรระวังจริง** ในบิลด์ที่ทดสอบนี้
(ลำดับการพิมพ์ `TYPE CONTROL HEADING` หลายระดับ และ `SUM ... UPON`) ซึ่งจะอธิบายอย่างตรงไปตรงมา
พร้อมหลักฐานการทดสอบจริงในขั้นตอนที่เกี่ยวข้อง

---

## ขั้นตอนที่ 381: CONTROLS ARE และ TYPE CONTROL HEADING — แนวคิด Control Break

### แนวคิด

**Control Break** คือเทคนิคที่ Report Writer ตรวจสอบว่าค่าของฟิลด์ที่กำหนด (เรียกว่า **control
field**) เปลี่ยนไปจาก record ก่อนหน้าหรือไม่ ทุกครั้งที่ `GENERATE` ถูกเรียก — ถ้าค่าเปลี่ยน (เช่น
จาก "NORTH" เป็น "SOUTH") จะถือว่าเกิด **"break"** และ Report Writer จะพิมพ์กลุ่มรายการพิเศษ
ที่ผูกกับฟิลด์นั้นให้อัตโนมัติ

การเปิดใช้งาน control break ทำผ่าน clause **`CONTROLS ARE field-name`** ใน `RD` และประกาศ
กลุ่มรายการ **`TYPE CONTROL HEADING field-name`** เพื่อกำหนดว่าจะพิมพ์อะไรเมื่อ**เริ่มกลุ่มใหม่**
ของฟิลด์นั้น (ตรงข้ามกับ `TYPE CONTROL FOOTING` ที่พิมพ์เมื่อ**จบกลุ่ม** ซึ่งจะเรียนในขั้นตอนถัดไป)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-CONTROL-HEADING-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "ch_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-REGION     PIC X(10).
       01  WS-NAME       PIC X(10).
       01  WS-AMOUNT     PIC 9(6)V99.

       REPORT SECTION.
       RD  SALES-REPORT
           CONTROLS ARE WS-REGION
           PAGE LIMIT 30 LINES.

       01  TYPE CONTROL HEADING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) VALUE "REGION:".
               05  COLUMN 12 PIC X(10) SOURCE WS-REGION.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 3   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 22  PIC ZZZ,ZZ9.99 SOURCE WS-AMOUNT.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.

           MOVE "NORTH"  TO WS-REGION.
           MOVE "APPLE"  TO WS-NAME.
           MOVE 100.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "BANANA" TO WS-NAME.
           MOVE 250.50   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "SOUTH"  TO WS-REGION.
           MOVE "CHERRY" TO WS-NAME.
           MOVE 300.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `RD SALES-REPORT CONTROLS ARE WS-REGION ...` — บอก Report Writer ว่าให้ตรวจสอบการเปลี่ยนแปลง
  ค่าของ `WS-REGION` ทุกครั้งที่ `GENERATE`
- `01 TYPE CONTROL HEADING WS-REGION.` — พิมพ์ทุกครั้งที่ `WS-REGION` เปลี่ยนค่า (รวมถึงครั้งแรกสุด
  ของรายงานเสมอ เพราะถือว่าเป็นการ "เข้าสู่กลุ่มใหม่" ตั้งแต่ record แรก)
- `GENERATE` ครั้งที่ 1 และ 2: `WS-REGION` เป็น "NORTH" เหมือนกัน — Report Writer เปรียบเทียบกับ
  ค่าก่อนหน้า พบว่าไม่เปลี่ยน จึงไม่พิมพ์ `TYPE CONTROL HEADING` ซ้ำ (ยกเว้นครั้งแรกสุด)
- `GENERATE` ครั้งที่ 3: `WS-REGION` เปลี่ยนจาก "NORTH" เป็น "SOUTH" — Report Writer ตรวจพบ
  การเปลี่ยนแปลง จึงพิมพ์ `TYPE CONTROL HEADING WS-REGION` ("REGION: SOUTH") ก่อนพิมพ์รายละเอียด

### ผลลัพธ์ที่ได้จากการรันจริง

```
REGION:    NORTH
   APPLE                100.00
   BANANA               250.50
REGION:    SOUTH
   CHERRY               300.00
```

### ข้อควรระวัง

- **Control Break ทำงานถูกต้องก็ต่อเมื่อข้อมูลถูกเรียง (sort) ตามฟิลด์ควบคุมมาก่อนแล้วเท่านั้น**
  Report Writer เปรียบเทียบเฉพาะกับ record ก่อนหน้าทันที ไม่ได้ตรวจสอบทั้งไฟล์ (จะสาธิตปัญหาที่
  เกิดขึ้นจริงเมื่อข้อมูลไม่ได้เรียงลำดับในขั้นตอนที่ 389)
- ฟิลด์ที่ระบุใน `CONTROLS ARE` ต้องเป็นตัวแปรที่มีอยู่จริงและถูก `MOVE`/`SOURCE` ค่าก่อน `GENERATE`
  แต่ละครั้งเสมอ

### แบบฝึกหัดที่ 381.1

**โจทย์**: จงอธิบายว่าทำไม `TYPE CONTROL HEADING WS-REGION` จึงพิมพ์ในการ `GENERATE` ครั้งแรกสุด
ของรายงาน ทั้งที่ไม่มี record ก่อนหน้าให้เปรียบเทียบเลย

**เฉลยแนวทาง**: เพราะ Report Writer ถือว่า "การเริ่มต้นรายงาน" คือการเข้าสู่กลุ่มควบคุมกลุ่มแรกเสมอ
(เปรียบเสมือนค่าก่อนหน้าเป็นค่าว่างที่ไม่มีทางตรงกับค่าจริงใด ๆ) จึงทริกเกอร์ `TYPE CONTROL HEADING`
ทุกครั้งที่ `GENERATE` ครั้งแรกของรายงานทำงาน โดยไม่ต้องมี record ก่อนหน้าจริง ๆ

---

## ขั้นตอนที่ 382: TYPE CONTROL FOOTING และ SUM Clause

### แนวคิด

**`TYPE CONTROL FOOTING field-name`** คือกลุ่มรายการที่พิมพ์**เมื่อกลุ่มควบคุมกำลังจะจบลง** —
กล่าวคือ พิมพ์ก่อนที่ `TYPE CONTROL HEADING` ของกลุ่มถัดไปจะเริ่ม (หรือก่อน `TERMINATE` ถ้าเป็น
กลุ่มสุดท้ายของรายงาน) ใช้บ่อยที่สุดสำหรับพิมพ์ **ยอดรวมของกลุ่ม (subtotal)**

**`SUM identifier`** เป็น clause พิเศษที่ใส่ในฟิลด์ของ `TYPE CONTROL FOOTING` เพื่อบอกให้ Report
Writer สะสมผลรวมของตัวแปรที่ระบุโดยอัตโนมัติทุกครั้งที่มีการ `GENERATE` บรรทัดรายละเอียดในกลุ่มนั้น
แล้วรีเซ็ตตัวสะสมเป็นศูนย์ใหม่หลังพิมพ์ท้ายกลุ่มเสร็จ — เราไม่ต้องเขียน `ADD` เองแม้แต่บรรทัดเดียว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-CONTROL-FOOTING-SUM-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "cf_sum_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-REGION     PIC X(10).
       01  WS-NAME       PIC X(10).
       01  WS-AMOUNT     PIC 9(6)V99.

       REPORT SECTION.
       RD  SALES-REPORT
           CONTROLS ARE WS-REGION
           PAGE LIMIT 30 LINES.

       01  TYPE CONTROL HEADING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) VALUE "REGION:".
               05  COLUMN 12 PIC X(10) SOURCE WS-REGION.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 3   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 22  PIC ZZZ,ZZ9.99 SOURCE WS-AMOUNT.

       01  TYPE CONTROL FOOTING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(15) VALUE "REGION TOTAL:".
      *> SUM automatically accumulates WS-AMOUNT every time a
      *> detail line is generated inside this region's group.
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.

           MOVE "NORTH"  TO WS-REGION.
           MOVE "APPLE"  TO WS-NAME.
           MOVE 100.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "BANANA" TO WS-NAME.
           MOVE 250.50   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "SOUTH"  TO WS-REGION.
           MOVE "CHERRY" TO WS-NAME.
           MOVE 300.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `05 COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.` — ทุกครั้งที่ `GENERATE DETAIL-LINE` ทำงานภายใน
  กลุ่ม "NORTH" ค่า `WS-AMOUNT` ปัจจุบันจะถูกบวกสะสมเข้าตัวนับภายในโดยอัตโนมัติ
- เมื่อ `WS-REGION` เปลี่ยนจาก "NORTH" เป็น "SOUTH" (เกิด control break) Report Writer จะพิมพ์
  `TYPE CONTROL FOOTING WS-REGION` ของกลุ่ม NORTH ก่อน (แสดงยอดรวม 100.00 + 250.50 = 350.50)
  แล้วจึง**รีเซ็ตตัวสะสมเป็นศูนย์** ก่อนเริ่มกลุ่ม SOUTH ใหม่
- กลุ่ม SOUTH มีเพียง record เดียว (300.00) แต่เนื่องจากไม่มี record ถัดไปให้ตรวจพบการเปลี่ยนแปลง
  ยอดรวมของกลุ่ม SOUTH จะถูกพิมพ์ก็ต่อเมื่อ `TERMINATE` ถูกเรียก (เพราะถือเป็นการ "จบกลุ่มสุดท้าย"
  โดยปริยาย)

### ผลลัพธ์ที่ได้จากการรันจริง

```
REGION:    NORTH
   APPLE                100.00
   BANANA               250.50
REGION TOTAL:            350.50
REGION:    SOUTH
   CHERRY               300.00
REGION TOTAL:            300.00
```

### ข้อควรระวัง

- **`SUM` ต้องอยู่ในกลุ่ม `TYPE CONTROL FOOTING` เท่านั้น** ใส่ใน `TYPE DETAIL` หรือ
  `TYPE PAGE HEADING` จะไม่มีความหมาย (หรือ compile error)
- ตัวสะสมของ `SUM` จะ**รีเซ็ตเป็นศูนย์อัตโนมัติทุกครั้งหลังพิมพ์ `TYPE CONTROL FOOTING` เสร็จ**
  โปรแกรมเมอร์ไม่ต้องเขียนโค้ดรีเซ็ตเอง (ต่างจากตัวแปรสะสมปกติที่ต้องคอยรีเซ็ตด้วยมือ)
- ความกว้างของ `PIC` ที่ใช้กับ `SUM` ต้องเผื่อพื้นที่ให้พอกับยอดรวมสูงสุดที่จะเกิดขึ้นจริง (ซึ่งอาจ
  ใหญ่กว่าค่าตัวเดียวมาก) มิเช่นนั้นจะเกิด overflow แบบเงียบเหมือนที่เรียนใน Part 038 ขั้นตอนที่ 377

### แบบฝึกหัดที่ 382.1

**โจทย์**: จงอธิบายว่าทำไมยอดรวมของกลุ่ม SOUTH ในตัวอย่างข้างต้นจึงถูกพิมพ์ออกมาได้ ทั้งที่ไม่มี
record ถัดไปที่ทำให้ `WS-REGION` เปลี่ยนค่าอีกเลย

**เฉลยแนวทาง**: เพราะคำสั่ง `TERMINATE` ถือเป็นจุดสิ้นสุดของกลุ่มควบคุมทุกกลุ่มที่ยังเปิดค้างอยู่โดย
ปริยาย Report Writer จะพิมพ์ `TYPE CONTROL FOOTING` ของกลุ่มปัจจุบัน (SOUTH) ก่อนจะปิดรายงาน
อย่างสมบูรณ์เสมอ แม้จะไม่มี record อื่นตามมาให้ตรวจพบการเปลี่ยนแปลงค่าก็ตาม

---

## ขั้นตอนที่ 383: TYPE CONTROL FOOTING FINAL — ยอดรวมทั้งฉบับ

### แนวคิด

นอกจากยอดรวมของแต่ละกลุ่มแล้ว รายงานส่วนใหญ่ต้องการ **ยอดรวมทั้งหมด (grand total)** ที่ท้าย
รายงานด้วย COBOL มีคำสงวนพิเศษคือ **`FINAL`** ที่ใช้แทนชื่อฟิลด์ควบคุมใน `TYPE CONTROL FOOTING`
กลุ่มที่ประกาศเป็น `TYPE CONTROL FOOTING FINAL` จะพิมพ์**เพียงครั้งเดียว** ตอน `TERMINATE`
เท่านั้น (หลังจากพิมพ์ `TYPE CONTROL FOOTING` ของกลุ่มควบคุมปกติกลุ่มสุดท้ายเสร็จแล้ว) และ `SUM`
ใน `TYPE CONTROL FOOTING FINAL` จะสะสมค่ารวมจาก**ทุก record ตลอดทั้งรายงาน** ไม่ใช่แค่กลุ่มเดียว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-FINAL-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "final_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-REGION     PIC X(10).
       01  WS-NAME       PIC X(10).
       01  WS-AMOUNT     PIC 9(6)V99.

       REPORT SECTION.
       RD  SALES-REPORT
           CONTROLS ARE WS-REGION
           PAGE LIMIT 30 LINES.

       01  TYPE CONTROL HEADING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) VALUE "REGION:".
               05  COLUMN 12 PIC X(10) SOURCE WS-REGION.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 3   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 22  PIC ZZZ,ZZ9.99 SOURCE WS-AMOUNT.

       01  TYPE CONTROL FOOTING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(15) VALUE "REGION TOTAL:".
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

      *> FINAL is a reserved control level: prints once at
      *> TERMINATE, summing across the WHOLE report.
       01  TYPE CONTROL FOOTING FINAL.
           03  LINE PLUS 2.
               05  COLUMN 1  PIC X(15) VALUE "GRAND TOTAL:".
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.

           MOVE "NORTH"  TO WS-REGION.
           MOVE "APPLE"  TO WS-NAME.
           MOVE 100.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "BANANA" TO WS-NAME.
           MOVE 250.50   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "SOUTH"  TO WS-REGION.
           MOVE "CHERRY" TO WS-NAME.
           MOVE 300.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `01 TYPE CONTROL FOOTING FINAL.` — ใช้คำสงวน `FINAL` แทนชื่อฟิลด์ ทำให้กลุ่มนี้พิมพ์เพียงครั้งเดียว
  ตอนจบรายงานทั้งฉบับเท่านั้น
- `SUM WS-AMOUNT` ใน `CONTROL FOOTING FINAL` เป็นตัวสะสม**คนละตัว**กับ `SUM WS-AMOUNT` ใน
  `CONTROL FOOTING WS-REGION` — ตัวแรกสะสมยอดของทุก record ตลอดทั้งรายงาน (100+250.50+300
  = 650.50) ในขณะที่ตัวหลังรีเซ็ตทุกครั้งที่ขึ้นกลุ่มใหม่
- `LINE PLUS 2` เว้นบรรทัดว่างก่อน "GRAND TOTAL:" เพื่อแยกให้เห็นชัดว่าเป็นยอดรวมระดับรายงาน
  ไม่ใช่ยอดรวมระดับกลุ่ม

### ผลลัพธ์ที่ได้จากการรันจริง

```
REGION:    NORTH
   APPLE                100.00
   BANANA               250.50
REGION TOTAL:            350.50
REGION:    SOUTH
   CHERRY               300.00
REGION TOTAL:            300.00

GRAND TOTAL:            650.50
```

### ข้อควรระวัง

- **`TYPE CONTROL FOOTING FINAL` ต้องพิมพ์คำว่า `FINAL` ตรง ๆ** ไม่ใช่ชื่อฟิลด์ของโปรแกรม —
  เป็นคำสงวนพิเศษของ COBOL ในบริบทนี้
- ถ้าลืม `TERMINATE` (ตามที่เรียนใน Part 038 ขั้นตอนที่ 375) `TYPE CONTROL FOOTING FINAL` จะไม่ถูก
  พิมพ์เลย เพราะมันผูกกับจังหวะ `TERMINATE` โดยตรง
- ควรมี `TYPE CONTROL FOOTING FINAL` ในทุกรายงานที่มีการคำนวณยอดรวม เพื่อให้ผู้อ่านรายงานเห็น
  ภาพรวมทั้งหมดโดยไม่ต้องบวกยอดรวมย่อยเองด้วยมือ

### แบบฝึกหัดที่ 383.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `SUM WS-AMOUNT` ที่อยู่ใน `TYPE CONTROL FOOTING WS-REGION`
กับ `SUM WS-AMOUNT` ที่อยู่ใน `TYPE CONTROL FOOTING FINAL`

**เฉลยแนวทาง**: แม้จะเขียนเหมือนกันทุกตัวอักษร แต่ทั้งสองเป็นตัวสะสมคนละตัวที่ COBOL runtime
จัดการแยกกันภายใน ตัวแรกสะสมเฉพาะ record ที่อยู่ในกลุ่มควบคุมปัจจุบันแล้วรีเซ็ตทุกครั้งที่ขึ้นกลุ่มใหม่
(ให้ยอดรวมระดับกลุ่ม/subtotal) ส่วนตัวที่สองสะสมจากทุก record ตลอดทั้งรายงานโดยไม่มีการรีเซ็ตเลย
จนกว่าจะถึง `TERMINATE` (ให้ยอดรวมทั้งฉบับ/grand total)

---

## ขั้นตอนที่ 384: Multi-Level Control Break — ควบคุมหลายระดับพร้อมกัน

### แนวคิด

`CONTROLS ARE` รับฟิลด์ได้มากกว่า 1 ตัว เพื่อสร้าง**ลำดับชั้นของกลุ่ม** (hierarchy) เช่น
"ภูมิภาค > จังหวัด" โดยเขียนฟิลด์ระดับ**ใหญ่สุดไปหาเล็กสุด**เรียงกัน:

```
RD report-name
    CONTROLS ARE major-field minor-field
    ...
```

จากนั้นประกาศ `TYPE CONTROL HEADING`/`TYPE CONTROL FOOTING` แยกกันสำหรับแต่ละระดับ
ตามหลักการมาตรฐานของ COBOL: **`TYPE CONTROL HEADING` ควรพิมพ์จากระดับใหญ่ไปเล็ก (major → minor)
ส่วน `TYPE CONTROL FOOTING` ควรพิมพ์จากระดับเล็กไปใหญ่ (minor → major)** เพื่อให้อ่านเป็นลำดับชั้น
ที่สมเหตุสมผล (เปิดกลุ่มใหญ่ก่อนแล้วค่อยเปิดกลุ่มย่อย, ปิดกลุ่มย่อยก่อนแล้วค่อยปิดกลุ่มใหญ่)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-MULTILEVEL-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "multilevel_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-REGION     PIC X(10).
       01  WS-CITY       PIC X(10).
       01  WS-NAME       PIC X(10).
       01  WS-AMOUNT     PIC 9(6)V99.

       REPORT SECTION.
       RD  SALES-REPORT
           CONTROLS ARE WS-REGION WS-CITY
           PAGE LIMIT 40 LINES.

       01  TYPE CONTROL HEADING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) VALUE "REGION:".
               05  COLUMN 12 PIC X(10) SOURCE WS-REGION.

       01  TYPE CONTROL HEADING WS-CITY.
           03  LINE PLUS 1.
               05  COLUMN 3  PIC X(8)  VALUE "CITY:".
               05  COLUMN 12 PIC X(10) SOURCE WS-CITY.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 5   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 22  PIC ZZZ,ZZ9.99 SOURCE WS-AMOUNT.

       01  TYPE CONTROL FOOTING WS-CITY.
           03  LINE PLUS 1.
               05  COLUMN 3  PIC X(15) VALUE "CITY TOTAL:".
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

       01  TYPE CONTROL FOOTING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(15) VALUE "REGION TOTAL:".
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

       01  TYPE CONTROL FOOTING FINAL.
           03  LINE PLUS 2.
               05  COLUMN 1  PIC X(15) VALUE "GRAND TOTAL:".
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.

           MOVE "NORTH"     TO WS-REGION.
           MOVE "CHIANGMAI" TO WS-CITY.
           MOVE "APPLE"     TO WS-NAME.
           MOVE 100.00      TO WS-AMOUNT.
           GENERATE DETAIL-LINE.
           MOVE "BANANA"    TO WS-NAME.
           MOVE 250.50      TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "LAMPANG"   TO WS-CITY.
           MOVE "CHERRY"    TO WS-NAME.
           MOVE 300.00      TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "SOUTH"     TO WS-REGION.
           MOVE "PHUKET"    TO WS-CITY.
           MOVE "DURIAN"    TO WS-NAME.
           MOVE 500.00      TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `CONTROLS ARE WS-REGION WS-CITY` — REGION เป็นระดับใหญ่ (major), CITY เป็นระดับย่อย (minor)
- เมื่อ `WS-REGION` เปลี่ยน ถือว่า `WS-CITY` เปลี่ยนไปด้วยโดยปริยาย (การเปลี่ยนระดับใหญ่ทำให้
  ระดับย่อยทั้งหมดต้อง "เปิดกลุ่มใหม่" ตามไปด้วยเสมอ) ดังนั้นเมื่อ REGION เปลี่ยนจาก NORTH เป็น
  SOUTH ทั้ง `TYPE CONTROL FOOTING WS-CITY`, `TYPE CONTROL FOOTING WS-REGION` ของกลุ่มเดิม
  จะพิมพ์ก่อน แล้วจึงเปิด `TYPE CONTROL HEADING WS-REGION`, `TYPE CONTROL HEADING WS-CITY`
  ของกลุ่มใหม่

### ผลลัพธ์ที่ได้จากการรันจริง

```
  CITY:    CHIANGMAI
REGION:    NORTH
    APPLE                100.00
    BANANA               250.50
  CITY TOTAL:            350.50
  CITY:    LAMPANG
    CHERRY               300.00
  CITY TOTAL:            300.00
REGION TOTAL:            650.50
  CITY:    PHUKET
REGION:    SOUTH
    DURIAN               500.00
  CITY TOTAL:            500.00
REGION TOTAL:            500.00

GRAND TOTAL:           1,150.50
```

### ข้อค้นพบจริงจากการทดสอบ — ลำดับ CONTROL HEADING สลับกันในบิลด์นี้

สังเกตผลลัพธ์ด้านบนอย่างละเอียด: บรรทัดแรกคือ **"CITY: CHIANGMAI" ตามด้วย "REGION: NORTH"**
และก่อนเข้ากลุ่ม SOUTH ก็เช่นกัน คือ **"CITY: PHUKET" ตามด้วย "REGION: SOUTH"** — นี่คือลำดับ
**สลับกัน** จากที่มาตรฐาน COBOL คาดหวัง (ควรพิมพ์ระดับใหญ่ REGION ก่อน แล้วจึงพิมพ์ระดับย่อย CITY)
ทั้งที่ในซอร์สโค้ดเราประกาศ `01 TYPE CONTROL HEADING WS-REGION` **ไว้ก่อน**
`01 TYPE CONTROL HEADING WS-CITY` แล้วก็ตาม

นี่คือพฤติกรรมจริงที่ทดสอบพบใน **GnuCOBOL 4.0-early-dev** เมื่อเกิด control break พร้อมกันทั้ง
สองระดับ (คือครั้งแรกสุดของรายงาน และตอนเปลี่ยนระดับ major) — ในทางกลับกัน ลำดับของ
`TYPE CONTROL FOOTING` (CITY TOTAL ก่อน REGION TOTAL) เป็นไปตามมาตรฐานถูกต้อง (minor ก่อน major)

**ข้อสรุปเชิงปฏิบัติ**: หากคุณใช้ multi-level control break ใน GnuCOBOL บิลด์นี้ ให้**ทดสอบผลลัพธ์
จริงเสมอ** อย่าเชื่อแค่ทฤษฎีมาตรฐาน โดยเฉพาะช่วงที่เกิด control break พร้อมกันหลายระดับ
(record แรกของรายงาน หรือตอนที่ major field เปลี่ยนพร้อมกับ minor field) หากพบว่าลำดับหัวข้อ
สลับกันแบบนี้ในระบบของคุณ วิธีแก้ไขที่ปลอดภัยที่สุดคือรวมข้อความหัวข้อทั้งสองระดับไว้ใน
`TYPE CONTROL HEADING` ของระดับ major เพียงกลุ่มเดียว (พิมพ์ทั้ง "REGION: NORTH" และบรรทัด
"CITY: ..." เริ่มต้นในกลุ่มเดียวกัน) แล้วปล่อยให้เฉพาะ `TYPE CONTROL HEADING` ของ minor
ทำงานเฉพาะตอนเปลี่ยนแค่ระดับย่อยจริง ๆ

### ข้อควรระวัง

- ทดสอบ multi-level control break ด้วยข้อมูลจริงของคุณเสมอ อย่าสมมติว่าลำดับการพิมพ์จะตรงตาม
  มาตรฐานเป๊ะในทุกคอมไพเลอร์/ทุกเวอร์ชัน
- `SUM WS-AMOUNT` ในแต่ละระดับยังคงทำงานถูกต้อง (350.50, 300.00, 650.50, 500.00, 1,150.50
  ล้วนคำนวณถูกต้องตามที่คาดหวัง) แม้ลำดับการพิมพ์ควบคุมหัวข้อ (heading) จะมีข้อค้นพบข้างต้นก็ตาม
- ควรเรียงลำดับฟิลด์ใน `CONTROLS ARE` จากใหญ่ไปเล็กเสมอ (major ก่อน minor) เพราะเป็นไวยากรณ์
  บังคับ ไม่ใช่แค่ธรรมเนียม

### แบบฝึกหัดที่ 384.1

**โจทย์**: จากข้อค้นพบข้างต้น จงเสนอวิธีแก้ไขปัญหาลำดับ CONTROL HEADING สลับกัน โดยไม่ต้องเปลี่ยน
โครงสร้างข้อมูล

**เฉลยแนวทาง**: แนวทางหนึ่งคือย้ายเนื้อหาที่ต้องการให้แสดงก่อน (REGION) เข้าไปรวมไว้ใน
`TYPE CONTROL HEADING WS-REGION` และปล่อยให้กลุ่มนี้พิมพ์ทั้งบรรทัด "REGION: ..." เอง โดยไม่ต้อง
พึ่งลำดับสัมพัทธ์ระหว่างสองกลุ่ม control heading แยกกัน แล้วให้ `TYPE CONTROL HEADING WS-CITY`
พิมพ์เฉพาะบรรทัด "CITY: ..." ต่อท้ายเท่านั้น วิธีนี้ยังคงพิมพ์ครบทุกข้อมูล เพียงแต่ลดการพึ่งพา
ลำดับการทำงานภายในของคอมไพเลอร์ที่อาจไม่แน่นอน

---

## ขั้นตอนที่ 385: NEXT GROUP Clause — ควบคุมระยะห่างระหว่างกลุ่ม

### แนวคิด

**`NEXT GROUP`** เป็น clause เสริมที่ใส่ต่อท้าย `TYPE CONTROL HEADING` (หรือกลุ่มรายการอื่น) เพื่อ
กำหนดว่า**หลังจากพิมพ์กลุ่มนี้เสร็จ ให้เลื่อนตำแหน่งบรรทัดเพิ่มเติมก่อนพิมพ์รายการถัดไป** มีประโยชน์
มากสำหรับเพิ่มช่องว่างให้รายงานอ่านง่ายขึ้น โดยไม่ต้องเพิ่ม `LINE PLUS` ในทุกฟิลด์ย่อยเอง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-NEXT-GROUP-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "next_group_demo.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-REGION     PIC X(10).
       01  WS-NAME       PIC X(10).
       01  WS-AMOUNT     PIC 9(6)V99.

       REPORT SECTION.
       RD  SALES-REPORT
           CONTROLS ARE WS-REGION
           PAGE LIMIT 40 LINES.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(20) VALUE "NEXT GROUP DEMO".

      *> After printing this control heading, skip one extra
      *> blank line before the next report group prints.
       01  TYPE CONTROL HEADING WS-REGION
           NEXT GROUP PLUS 1.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) VALUE "REGION:".
               05  COLUMN 12 PIC X(10) SOURCE WS-REGION.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 3   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 22  PIC ZZZ,ZZ9.99 SOURCE WS-AMOUNT.

       01  TYPE CONTROL FOOTING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(15) VALUE "REGION TOTAL:".
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.
           MOVE "NORTH"  TO WS-REGION.
           MOVE "APPLE"  TO WS-NAME.
           MOVE 100.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.
           MOVE "SOUTH"  TO WS-REGION.
           MOVE "CHERRY" TO WS-NAME.
           MOVE 300.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.
           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `01 TYPE CONTROL HEADING WS-REGION NEXT GROUP PLUS 1.` — หลังจากพิมพ์บรรทัด "REGION: ..."
  เสร็จ ให้เว้นบรรทัดว่างเพิ่มอีก 1 บรรทัดก่อนพิมพ์บรรทัดรายละเอียดถัดไป (`DETAIL-LINE`)
- ผลคือทุกครั้งที่ขึ้นกลุ่มภูมิภาคใหม่ จะมีบรรทัดว่างคั่นระหว่างหัวข้อกลุ่มกับรายการแรกของกลุ่มนั้น
  โดยอัตโนมัติ ไม่ต้องเขียน `LINE PLUS 2` ในฟิลด์แรกของ `DETAIL-LINE` เอง (ซึ่งจะกระทบทุกบรรทัด
  รายละเอียด ไม่ใช่แค่บรรทัดแรกของกลุ่ม)

### ผลลัพธ์ที่ได้จากการรันจริง

```
NEXT GROUP DEMO

REGION:    NORTH

  APPLE                  100.00
REGION TOTAL:            100.00
REGION:    SOUTH

  CHERRY                 300.00
REGION TOTAL:            300.00
```

### ข้อควรระวัง

- `NEXT GROUP` ส่งผลต่อ**ตำแหน่งของรายการถัดไปที่จะพิมพ์** ไม่ใช่ตำแหน่งของกลุ่มปัจจุบันเอง
  จึงอาจทำให้สับสนได้ในช่วงแรกที่ใช้งาน ควรทดสอบผลลัพธ์จริงเสมอเพื่อยืนยันตำแหน่งที่ต้องการ
- ใช้ `NEXT GROUP PLUS n` เพื่อเว้น n บรรทัด หรือ `NEXT GROUP PAGE` เพื่อบังคับขึ้นหน้าใหม่ทันที
  หลังกลุ่มนี้ (มีประโยชน์เมื่อต้องการให้แต่ละกลุ่มควบคุมเริ่มต้นหน้าใหม่เสมอ)

### แบบฝึกหัดที่ 385.1

**โจทย์**: จงอธิบายว่า `NEXT GROUP PAGE` ต่างจาก `NEXT GROUP PLUS 1` อย่างไร และควรใช้ `NEXT
GROUP PAGE` ในสถานการณ์แบบใด

**เฉลยแนวทาง**: `NEXT GROUP PLUS n` เว้นบรรทัดว่าง n บรรทัดในหน้าเดิม ในขณะที่ `NEXT GROUP PAGE`
บังคับขึ้นหน้ากระดาษใหม่ทันทีหลังพิมพ์กลุ่มนั้นเสร็จ เหมาะกับสถานการณ์ที่ต้องการให้แต่ละกลุ่มควบคุม
(เช่น แต่ละแผนกหรือแต่ละสาขา) เริ่มต้นที่หน้าใหม่เสมอ เพื่อความสะดวกเวลาแยกพิมพ์หรือแจกจ่ายรายงาน
เป็นรายกลุ่ม (เช่น ส่งรายงานยอดขายแยกแต่ละสาขาให้ผู้จัดการสาขานั้น ๆ)

---

## ขั้นตอนที่ 386: SUM ... UPON — และข้อจำกัดที่พบจากการทดสอบจริง

### แนวคิด

ตามมาตรฐาน COBOL, clause **`SUM identifier UPON detail-name`** ควรจำกัดให้ตัวสะสมนับรวมค่า
**เฉพาะตอนที่ `GENERATE detail-name` ที่ระบุไว้ทำงานเท่านั้น** มีประโยชน์เมื่อรายงานมีหลายชนิด
`TYPE DETAIL` ปนกัน (เช่น จากที่เรียนใน Part 038 ขั้นตอนที่ 374) แต่ต้องการสะสมยอดรวมแยกเฉพาะ
บางชนิดเท่านั้น

### ตัวอย่างโค้ดที่ทดสอบจริง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-SUM-UPON-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "sum_upon_demo.txt"
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

       01  SALE-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) SOURCE WS-ITEM-NAME.
               05  COLUMN 12 PIC ZZZZ9 SOURCE WS-QTY.

       01  RETURN-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) SOURCE WS-ITEM-NAME.
               05  COLUMN 12 PIC ZZZZ9 SOURCE WS-QTY.

      *> Per the COBOL standard, "SUM ... UPON SALE-DETAIL" should
      *> add WS-QTY ONLY when SALE-DETAIL (not RETURN-DETAIL) is
      *> generated.
       01  TYPE CONTROL FOOTING FINAL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(15) VALUE "NET SOLD TOTAL:".
               05  COLUMN 18 PIC ZZZZZ9 SUM WS-QTY UPON SALE-DETAIL.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE ITEM-REPORT.
           MOVE "APPLE" TO WS-ITEM-NAME.
           MOVE 100 TO WS-QTY.
           GENERATE SALE-DETAIL.
           MOVE "APPLE-RTN" TO WS-ITEM-NAME.
           MOVE 1000 TO WS-QTY.
           GENERATE RETURN-DETAIL.
           MOVE "BANANA" TO WS-ITEM-NAME.
           MOVE 50 TO WS-QTY.
           GENERATE SALE-DETAIL.
           TERMINATE ITEM-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### ผลลัพธ์ที่ได้จากการรันจริง

```
ITEM      QTY
APPLE        100
APPLE-RTN   1000
BANANA        50
NET SOLD TOTAL:    1150
```

### ข้อค้นพบจริงจากการทดสอบ — SUM UPON ไม่กรองตามที่มาตรฐานกำหนด

ตามทฤษฎี ยอดรวม "NET SOLD TOTAL" ควรนับเฉพาะ `SALE-DETAIL` เท่านั้น คือ **100 + 50 = 150**
(ไม่รวม 1000 ของ `RETURN-DETAIL`) แต่ผลลัพธ์จริงที่ทดสอบได้คือ **1150** ซึ่งเท่ากับ
100 + 1000 + 50 — แสดงว่า **`UPON` ไม่ได้กรองให้สะสมเฉพาะ `SALE-DETAIL` ตามที่ระบุ** ใน
GnuCOBOL 4.0-early-dev บิลด์ที่ทดสอบนี้ แต่กลับสะสมค่าจากทุก `GENERATE` ของทุกชนิด `TYPE DETAIL`
ในรายงานเหมือนไม่มี `UPON` เลย

**ข้อสรุปเชิงปฏิบัติ**: อย่าพึ่งพา `SUM ... UPON` เพื่อกรองผลรวมตามชนิด detail ในบิลด์นี้
ให้ใช้ทางเลือกที่ทดสอบแล้วว่าเชื่อถือได้แทน คือสะสมยอดด้วยตัวแปรใน WORKING-STORAGE เอง
แล้วนำมาแสดงด้วย `SOURCE` (ไม่ใช่ `SUM`) ดังตัวอย่างด้านล่าง:

```cobol
       WORKING-STORAGE SECTION.
       01  WS-NET-SOLD-TOTAL  PIC 9(6) VALUE 0.
       ...
       01  TYPE CONTROL FOOTING FINAL.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(15) VALUE "NET SOLD TOTAL:".
               05  COLUMN 18 PIC ZZZZZ9 SOURCE WS-NET-SOLD-TOTAL.
       ...
       PROCEDURE DIVISION.
           ...
           MOVE "APPLE" TO WS-ITEM-NAME.
           MOVE 100 TO WS-QTY.
           ADD WS-QTY TO WS-NET-SOLD-TOTAL.
           GENERATE SALE-DETAIL.
```

เมื่อทดสอบด้วยวิธีนี้ (`ADD WS-QTY TO WS-NET-SOLD-TOTAL` เฉพาะตอนเป็นการขาย ไม่ใช่การคืนสินค้า)
`WS-NET-SOLD-TOTAL` จะเป็น **150** ตรงตามที่ต้องการอย่างแม่นยำ เพราะเราควบคุมการบวกเองใน
PROCEDURE DIVISION โดยตรง ไม่ต้องพึ่งพา `UPON` ที่มีพฤติกรรมไม่ตรงตามมาตรฐานในบิลด์นี้

### ข้อควรระวัง

- นี่คือตัวอย่างสำคัญว่าทำไมกฎของหลักสูตรนี้จึงกำชับให้ **ทดสอบคอมไพล์และรันจริงทุกตัวอย่างก่อนเชื่อ**
  แม้แต่ฟีเจอร์ที่มีอยู่ในมาตรฐาน COBOL ก็อาจมีพฤติกรรมไม่ตรงตามเอกสารในบางคอมไพเลอร์/บางเวอร์ชัน
- หากคุณต้องทำงานกับคอมไพเลอร์ COBOL ตัวอื่น (เช่น IBM Enterprise COBOL บน Mainframe หรือ
  Micro Focus) ควรทดสอบ `SUM ... UPON` ในสภาพแวดล้อมนั้นแยกต่างหาก เพราะพฤติกรรมอาจแตกต่างจาก
  ที่พบใน GnuCOBOL นี้
- `SUM` แบบไม่มี `UPON` (ที่เรียนในขั้นตอนที่ 382-383) ทำงานถูกต้องสมบูรณ์ตามที่ทดสอบ ปัญหาพบเฉพาะ
  เมื่อใช้ `UPON` เพื่อกรองชนิด detail เท่านั้น

### แบบฝึกหัดที่ 386.1

**โจทย์**: จงอธิบายว่าทำไมแนวทาง "ADD ด้วยตัวเองใน PROCEDURE DIVISION แล้วแสดงด้วย SOURCE"
จึงเป็นทางเลือกที่ปลอดภัยกว่า `SUM ... UPON` ในกรณีนี้

**เฉลยแนวทาง**: เพราะการ `ADD` ด้วยตัวเองทำให้โปรแกรมเมอร์ควบคุมตรรกะการสะสมยอดได้อย่างชัดเจน
และตรวจสอบได้ตรง ๆ ในโค้ด PROCEDURE DIVISION ไม่ต้องพึ่งพาพฤติกรรมภายในของ Report Writer
ที่อาจแตกต่างกันไปในแต่ละคอมไพเลอร์ ทำให้ผลลัพธ์คาดเดาได้แน่นอนกว่า และเมื่อทดสอบแล้วพบว่าทำงาน
ถูกต้อง ก็มั่นใจได้ว่าจะทำงานถูกต้องเช่นเดียวกันในสภาพแวดล้อมอื่นด้วย (เพราะไม่ได้พึ่งพาฟีเจอร์เฉพาะทาง
ที่มีความเสี่ยงเรื่องความเข้ากันได้)

---

## ขั้นตอนที่ 387: Control Break ร่วมกับการขึ้นหน้าใหม่ (PAGE FOOTING)

### แนวคิด

เมื่อรายงานมีทั้ง control break และการขึ้นหน้าใหม่ (`PAGE LIMIT`) พร้อมกัน อาจเกิดกรณีที่หน้าเต็ม
**ระหว่างกลางกลุ่มควบคุม** (เช่น กำลังพิมพ์รายการของภูมิภาค NORTH อยู่ แต่หน้าเต็มพอดี) Report
Writer จะขึ้นหน้าใหม่ พิมพ์ `TYPE PAGE HEADING` ซ้ำ แล้วพิมพ์รายการที่เหลือของกลุ่มเดิมต่อในหน้าใหม่
**โดยไม่พิมพ์ `TYPE CONTROL HEADING` ซ้ำ** (เพราะยังอยู่ในกลุ่มเดิม ไม่ได้เกิดการเปลี่ยนค่า)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-PAGE-BREAK-MID-GROUP.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "page_break_mid.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-REGION     PIC X(10).
       01  WS-NAME       PIC X(10).
       01  WS-AMOUNT     PIC 9(6)V99.
       01  WS-COUNTER    PIC 9(2).

       REPORT SECTION.
       RD  SALES-REPORT
           CONTROLS ARE WS-REGION
           PAGE LIMIT 8 LINES
           HEADING 1
           FIRST DETAIL 3
           FOOTING 7.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(20) VALUE "MID-GROUP PAGE BREAK".

       01  TYPE CONTROL HEADING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) VALUE "REGION:".
               05  COLUMN 12 PIC X(10) SOURCE WS-REGION.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 3   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 22  PIC ZZZ9  SOURCE WS-AMOUNT.

       01  TYPE PAGE FOOTING.
           03  LINE PLUS 1.
               05  COLUMN 1 PIC X(15) VALUE "-- PAGE END --".

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.
           MOVE "NORTH" TO WS-REGION.
           PERFORM VARYING WS-COUNTER FROM 1 BY 1 UNTIL WS-COUNTER > 6
               MOVE "ITEM" TO WS-NAME
               MOVE WS-COUNTER TO WS-AMOUNT
               GENERATE DETAIL-LINE
           END-PERFORM.
           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

- `FIRST DETAIL 3`, `FOOTING 7` — แต่ละหน้าพิมพ์รายละเอียดได้ที่บรรทัด 3, 4, 5, 6, 7 (5 บรรทัด)
  แต่บรรทัดแรกของแต่ละหน้าถูกใช้ไปกับ `TYPE CONTROL HEADING` ในหน้าแรก ทำให้เหลือที่จริงสำหรับ
  `DETAIL-LINE` น้อยกว่านั้นในหน้าแรก
- เมื่อ NORTH มี 6 รายการ แต่ที่ว่างในหน้าแรกไม่พอ (หน้าแรกใช้บรรทัด 2 ไปกับ `TYPE CONTROL
  HEADING` แล้ว เหลือที่จริงสำหรับ `DETAIL-LINE` เพียง 4 บรรทัดคือบรรทัด 3-6 ก่อนถึง `FOOTING 7`)
  Report Writer จะขึ้นหน้าใหม่กลางกลุ่ม NORTH หลังพิมพ์ครบ 4 รายการแรก โดยพิมพ์
  `TYPE PAGE FOOTING`, `TYPE PAGE HEADING` ใหม่ แต่**ไม่พิมพ์ "REGION: NORTH" ซ้ำ**
  เพราะยังเป็นกลุ่มเดิม

### ผลลัพธ์ที่ได้จากการรันจริง

```
MID-GROUP PAGE BREAK

REGION:    NORTH
  ITEM                  1
  ITEM                  2
  ITEM                  3
  ITEM                  4
-- PAGE END --
MID-GROUP PAGE BREAK

  ITEM                  5
  ITEM                  6
-- PAGE END --
```

สังเกตว่าหน้าที่สองมีเพียงรายการ "ITEM 5" และ "ITEM 6" โดยไม่มี "REGION: NORTH" ซ้ำ เพราะยังอยู่ใน
กลุ่มควบคุมเดียวกันกับหน้าแรก

### ข้อควรระวัง

- ถ้าต้องการให้ทุกหน้าที่มีข้อมูลของกลุ่มใดกลุ่มหนึ่งแสดงชื่อกลุ่มกำกับไว้เสมอ (เช่น เพื่อความชัดเจน
  เวลาพิมพ์หลายหน้าแยกกัน) จำเป็นต้องเขียนโค้ดเสริมเอง เพราะ Report Writer มาตรฐานจะไม่พิมพ์
  `TYPE CONTROL HEADING` ซ้ำในหน้าใหม่ที่ยังอยู่กลุ่มเดิม
- ควรออกแบบ `PAGE LIMIT` ให้มีขนาดใหญ่พอสำหรับข้อมูลจริง เพื่อลดโอกาสที่กลุ่มควบคุมจะถูกตัดข้าม
  หน้ากลางกลุ่มบ่อยเกินไปจนอ่านยาก

### แบบฝึกหัดที่ 387.1

**โจทย์**: จงอธิบายว่าทำไมหน้าที่สองในตัวอย่างข้างต้นจึงไม่มีข้อความ "REGION: NORTH" ปรากฏซ้ำ

**เฉลยแนวทาง**: เพราะ `TYPE CONTROL HEADING` ถูกทริกเกอร์โดยการ**เปลี่ยนค่า**ของฟิลด์ควบคุม
(`WS-REGION`) เท่านั้น ไม่ใช่โดยการขึ้นหน้าใหม่ ในกรณีนี้ `WS-REGION` ยังคงเป็น "NORTH" เหมือนเดิม
ตลอดทั้ง 6 รายการ การขึ้นหน้าใหม่ (page break) เป็นกลไกที่แยกอิสระจาก control break — ทั้งสอง
เกิดพร้อมกันได้ แต่ไม่ได้ทริกเกอร์ซึ่งกันและกัน

---

## ขั้นตอนที่ 388: อ่านข้อมูลจากไฟล์จริงเพื่อสร้าง Control Break Report

### แนวคิด

ในทางปฏิบัติ ข้อมูลสำหรับ control break report มาจากไฟล์ที่**เรียงลำดับ (sort) ตามฟิลด์ควบคุมมาแล้ว**
เสมอ (มักจะผ่านขั้นตอน `SORT` ที่เรียนใน Part 027 ก่อนนำมาอ่านสร้างรายงาน) ขั้นตอนนี้จะสาธิตการอ่าน
ไฟล์พนักงานที่เรียงตามแผนกมาแล้ว มาสร้างรายงานเงินเดือนแยกตามแผนกพร้อมยอดรวม

### ตัวอย่างโค้ด

ไฟล์ข้อมูล `employees_sorted.txt` (เรียงตามแผนกมาแล้ว):

```
SIRIPORN  HR        002800000
NAPAPORN  HR        003100000
WICHAI    IT        004200000
SOMSAK    IT        003900000
PIMCHANOK SALES     003500000
```

โปรแกรม COBOL:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-FILE-DRIVEN-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT EMPLOYEE-FILE ASSIGN TO "employees_sorted.txt"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT REPORT-FILE ASSIGN TO "dept_payroll_report.txt"
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
           CONTROLS ARE EMP-DEPT
           PAGE LIMIT 30 LINES.

       01  TYPE PAGE HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(26) VALUE "DEPARTMENT PAYROLL REPORT".

       01  TYPE CONTROL HEADING EMP-DEPT.
           03  LINE PLUS 2.
               05  COLUMN 1  PIC X(12) VALUE "DEPARTMENT:".
               05  COLUMN 14 PIC X(10) SOURCE EMP-DEPT.

       01  PAYROLL-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 3  PIC X(10)      SOURCE EMP-NAME.
               05  COLUMN 20 PIC ZZ,ZZ9.99  SOURCE EMP-SALARY.

       01  TYPE CONTROL FOOTING EMP-DEPT.
           03  LINE PLUS 1.
               05  COLUMN 3  PIC X(16) VALUE "DEPT TOTAL:".
               05  COLUMN 20 PIC ZZ,ZZ9.99 SUM EMP-SALARY.

       01  TYPE CONTROL FOOTING FINAL.
           03  LINE PLUS 2.
               05  COLUMN 1  PIC X(16) VALUE "COMPANY TOTAL:".
               05  COLUMN 20 PIC ZZZ,ZZ9.99 SUM EMP-SALARY.

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

- `CONTROLS ARE EMP-DEPT` — ฟิลด์ควบคุมคือ `EMP-DEPT` ซึ่งอยู่ใน **FILE SECTION** ไม่ใช่
  WORKING-STORAGE เหมือนตัวอย่างก่อนหน้า Report Writer อ่านค่าปัจจุบันของฟิลด์นี้ได้โดยตรงทุกครั้ง
  ที่ `GENERATE`
- เนื่องจากไฟล์เรียงตามแผนกมาแล้ว (HR, HR, IT, IT, SALES) การเปลี่ยนแผนกจะตรงกับตำแหน่งที่ควร
  เกิด control break พอดี ทำให้ยอดรวมแต่ละแผนกถูกต้อง

### ผลลัพธ์ที่ได้จากการรันจริง

```
DEPARTMENT PAYROLL REPORT

DEPARTMENT:  HR
  SIRIPORN         28,000.00
  NAPAPORN         31,000.00
  DEPT TOTAL:      59,000.00

DEPARTMENT:  IT
  WICHAI           42,000.00
  SOMSAK           39,000.00
  DEPT TOTAL:      81,000.00

DEPARTMENT:  SALES
  PIMCHANOK        35,000.00
  DEPT TOTAL:      35,000.00

COMPANY TOTAL:     175,000.00
```

### ข้อควรระวัง

- **ข้อมูลนำเข้าต้องเรียงตามฟิลด์ควบคุมมาก่อนเสมอ** ในโลกจริงมักทำผ่านขั้นตอน `SORT` (Part 027)
  ก่อนนำมาป้อนให้โปรแกรมสร้างรายงาน หรืออ่านจาก Indexed File ที่เรียงลำดับคีย์อยู่แล้ว (Part 028)
- การใช้ฟิลด์จาก FILE SECTION เป็น `CONTROLS ARE` โดยตรง (ไม่ผ่าน WORKING-STORAGE) ทำงานได้ดี
  ตราบใดที่ record ปัจจุบันยังไม่ถูกอ่านทับด้วย record ถัดไปก่อน `GENERATE`

### แบบฝึกหัดที่ 388.1

**โจทย์**: จงอธิบายว่าจะเกิดอะไรขึ้นกับยอดรวมแต่ละแผนก หากไฟล์ `employees_sorted.txt` ไม่ได้เรียง
ตามแผนกมาก่อน (เช่น สลับให้พนักงาน IT คนหนึ่งอยู่ปนกับกลุ่ม HR)

**เฉลยแนวทาง**: จะเกิดปัญหาเดียวกับที่จะสาธิตในขั้นตอนที่ 389 คือ Report Writer จะตรวจพบว่า
`EMP-DEPT` เปลี่ยนค่าทุกครั้งที่พบแผนกที่ต่างจาก record ก่อนหน้า ทำให้เกิด "กลุ่ม HR" หลายกลุ่มแยกกัน
พร้อมยอดรวมแยกกันหลายก้อน แทนที่จะเป็นยอดรวมแผนก HR เพียงก้อนเดียวตามที่ควรจะเป็น

---

## ขั้นตอนที่ 389: อันตรายของข้อมูลที่ไม่เรียงลำดับ — พิสูจน์ด้วยการทดสอบจริง

### แนวคิด

ขั้นตอนนี้จะพิสูจน์ให้เห็น**ด้วยผลการทดสอบจริง**ว่าเกิดอะไรขึ้นเมื่อป้อนข้อมูลที่ไม่ได้เรียงตามฟิลด์
ควบคุมให้กับ Report Writer — เป็นข้อควรระวังที่สำคัญที่สุดข้อหนึ่งของ control break ทั้งหมด

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-UNSORTED-DANGER-DEMO.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT REPORT-FILE ASSIGN TO "unsorted_danger.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  REPORT-FILE
           REPORT IS SALES-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-REGION     PIC X(10).
       01  WS-NAME       PIC X(10).
       01  WS-AMOUNT     PIC 9(6)V99.

       REPORT SECTION.
       RD  SALES-REPORT
           CONTROLS ARE WS-REGION
           PAGE LIMIT 30 LINES.

       01  TYPE CONTROL HEADING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(10) VALUE "REGION:".
               05  COLUMN 12 PIC X(10) SOURCE WS-REGION.

       01  DETAIL-LINE TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 3   PIC X(10) SOURCE WS-NAME.
               05  COLUMN 22  PIC ZZZ,ZZ9.99 SOURCE WS-AMOUNT.

       01  TYPE CONTROL FOOTING WS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(15) VALUE "REGION TOTAL:".
               05  COLUMN 22 PIC ZZZ,ZZ9.99 SUM WS-AMOUNT.

       PROCEDURE DIVISION.
           OPEN OUTPUT REPORT-FILE.
           INITIATE SALES-REPORT.

      *> WARNING: data is deliberately NOT sorted by region --
      *> NORTH appears again AFTER a SOUTH record in between.
           MOVE "NORTH"  TO WS-REGION.
           MOVE "APPLE"  TO WS-NAME.
           MOVE 100.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "SOUTH"  TO WS-REGION.
           MOVE "CHERRY" TO WS-NAME.
           MOVE 300.00   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           MOVE "NORTH"  TO WS-REGION.
           MOVE "BANANA" TO WS-NAME.
           MOVE 250.50   TO WS-AMOUNT.
           GENERATE DETAIL-LINE.

           TERMINATE SALES-REPORT.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### ผลลัพธ์ที่ได้จากการรันจริง

```
REGION:    NORTH
   APPLE                100.00
REGION TOTAL:            100.00
REGION:    SOUTH
   CHERRY               300.00
REGION TOTAL:            300.00
REGION:    NORTH
   BANANA               250.50
REGION TOTAL:            250.50
```

### วิเคราะห์ผลลัพธ์ — บั๊กทางธุรกิจที่ร้ายแรงแต่คอมไพล์ผ่านปกติ

สังเกตว่า **"REGION: NORTH" ปรากฏถึง 2 ครั้งแยกกัน** พร้อมยอดรวมแยกกันคนละก้อน (100.00 และ
250.50) แทนที่จะเป็นยอดรวม NORTH เพียงก้อนเดียว **350.50** ตามที่ควรจะเป็นจริง — Report Writer
ไม่ได้ error หรือเตือนอะไรเลย เพราะมันแค่ทำงานตามกลไกที่ถูกต้องของมัน (เปรียบเทียบกับ record
ก่อนหน้าทันทีเท่านั้น) โปรแกรมนี้**คอมไพล์ผ่านและรันได้ปกติ แต่ผลลัพธ์ทางธุรกิจผิดพลาดอย่างร้ายแรง**
หากนำยอดรวม "REGION TOTAL" ของ NORTH ทั้งสองก้อนไปใช้งานต่อโดยไม่ทันสังเกตว่ามันคือภูมิภาคเดียวกัน
ที่ถูกแยกออกเป็นสองกลุ่มโดยไม่ได้ตั้งใจ

### ข้อควรระวัง

- **นี่คือเหตุผลสำคัญที่สุดที่ทุกโปรแกรม control break ต้องรับข้อมูลที่เรียงลำดับตามฟิลด์ควบคุมมาแล้ว
  เท่านั้น** ไม่มีข้อยกเว้น หากไม่แน่ใจว่าข้อมูลเรียงลำดับมาแล้วหรือไม่ ให้ใช้ `SORT` (Part 027)
  ก่อนเสมอ หรืออ่านจาก Indexed File ที่เรียงคีย์อยู่แล้ว (Part 028)
- ควรเพิ่มการตรวจสอบเชิงป้องกัน (defensive check) ในโค้ดจริง เช่น เปรียบเทียบค่าฟิลด์ควบคุมของ
  record ปัจจุบันกับ record ก่อนหน้าที่เก็บไว้ต่างหาก หากพบว่าค่าเรียงลำดับผิดปกติ (เช่น กลับมาเป็น
  ค่าเดิมที่เคยผ่านไปแล้ว) ให้แจ้งเตือนหรือหยุดโปรแกรมทันที แทนที่จะปล่อยให้สร้างรายงานที่ผิดเงียบ ๆ

### แบบฝึกหัดที่ 389.1

**โจทย์**: จงอธิบายว่าทำไมข้อผิดพลาดประเภทนี้ถึง "อันตราย" มากกว่า compile error หรือ runtime error
ทั่วไป

**เฉลยแนวทาง**: เพราะ compile error หรือ runtime error (เช่น การลืม `INITIATE` ที่เรียนใน Part 038)
จะหยุดโปรแกรมทันทีและแจ้งปัญหาให้เห็นชัดเจน ทำให้แก้ไขได้เร็ว แต่ข้อผิดพลาดจากข้อมูลไม่เรียงลำดับนี้
**ไม่มี error ใด ๆ เลย** โปรแกรมทำงานสำเร็จ รายงานถูกสร้างออกมาสมบูรณ์ ตัวเลขในรายงานดูสมเหตุสมผล
ในตัวเอง (ยอดรวมแต่ละกลุ่มถูกต้องตามข้อมูลที่มันเห็น) แต่**ความหมายทางธุรกิจผิดพลาด** — ผู้ใช้รายงาน
อาจไม่ทันสังเกตว่ามีภูมิภาคเดียวกันปรากฏซ้ำสองครั้ง จนกว่าจะมีคนตรวจสอบยอดรวมทั้งหมดอย่างละเอียด
ซึ่งอาจสายเกินไปหากนำตัวเลขไปใช้ตัดสินใจทางธุรกิจไปแล้ว

---

## ขั้นตอนที่ 390: โปรแกรมรวบยอด — รายงานยอดขายแยกภูมิภาค/จังหวัดพร้อมยอดรวมทุกระดับ

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่รวมทุกเทคนิค control break เข้าด้วยกัน: multi-level `CONTROLS ARE`,
`TYPE CONTROL HEADING`/`FOOTING` ทั้งสองระดับ, `SUM`, `TYPE CONTROL FOOTING FINAL`,
`TYPE PAGE HEADING`/`FOOTING` และการอ่านข้อมูลจากไฟล์จริงที่เรียงลำดับมาแล้ว

### ตัวอย่างโค้ด

ไฟล์ข้อมูล `regional_sales_sorted.txt` (เรียงตาม ภูมิภาค แล้วตาม จังหวัด มาแล้ว, fixed-width
ไม่มีตัวคั่นระหว่างฟิลด์):

```
NORTHCHIANGMAIAPPLE     0000100000
NORTHCHIANGMAIBANANA    0000250500
NORTHLAMPANG  CHERRY    0000300000
SOUTHPHUKET   DURIAN    0000500000
SOUTHSURATTHANMANGO     0000180000
```

(รูปแบบ: ภูมิภาค 5 ตัวอักษร + จังหวัด 9 ตัวอักษร + ชื่อสินค้า 10 ตัวอักษร + ยอดขาย 10 หลัก
มีทศนิยม 2 ตำแหน่งโดยนัย รวมความยาวบรรทัดละ 34 ตัวอักษรพอดี)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CB-CAPSTONE-REGIONAL-REPORT.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SALES-FILE ASSIGN TO "regional_sales_sorted.txt"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT REPORT-FILE ASSIGN TO "regional_final_report.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  SALES-FILE.
       01  SALES-RECORD.
           05  SLS-REGION    PIC X(5).
           05  SLS-CITY      PIC X(9).
           05  SLS-ITEM      PIC X(10).
           05  SLS-AMOUNT    PIC 9(8)V99.

       FD  REPORT-FILE
           REPORT IS REGIONAL-REPORT.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG   PIC X VALUE "N".

       REPORT SECTION.
       RD  REGIONAL-REPORT
           CONTROLS ARE SLS-REGION SLS-CITY
           PAGE LIMIT 40 LINES.

       01  TYPE REPORT HEADING.
           03  LINE 1.
               05  COLUMN 1 PIC X(22) VALUE "REGIONAL SALES REPORT".

       01  TYPE PAGE HEADING.
           03  LINE 2.
               05  COLUMN 1  PIC X(10) VALUE "ITEM".
               05  COLUMN 20 PIC X(10) VALUE "AMOUNT".

       01  TYPE CONTROL HEADING SLS-REGION.
           03  LINE PLUS 2.
               05  COLUMN 1  PIC X(9)  VALUE "REGION:".
               05  COLUMN 10 PIC X(5)  SOURCE SLS-REGION.

       01  TYPE CONTROL HEADING SLS-CITY.
           03  LINE PLUS 1.
               05  COLUMN 3  PIC X(7)  VALUE "CITY:".
               05  COLUMN 10 PIC X(9)  SOURCE SLS-CITY.

       01  SALES-DETAIL TYPE DETAIL.
           03  LINE PLUS 1.
               05  COLUMN 5  PIC X(10)       SOURCE SLS-ITEM.
               05  COLUMN 18 PIC ZZ,ZZZ,ZZ9.99 SOURCE SLS-AMOUNT.

       01  TYPE CONTROL FOOTING SLS-CITY.
           03  LINE PLUS 1.
               05  COLUMN 3  PIC X(14) VALUE "CITY TOTAL:".
               05  COLUMN 18 PIC ZZ,ZZZ,ZZ9.99 SUM SLS-AMOUNT.

       01  TYPE CONTROL FOOTING SLS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(16) VALUE "REGION TOTAL:".
               05  COLUMN 18 PIC ZZ,ZZZ,ZZ9.99 SUM SLS-AMOUNT.

       01  TYPE CONTROL FOOTING FINAL.
           03  LINE PLUS 2.
               05  COLUMN 1  PIC X(16) VALUE "COMPANY TOTAL:".
               05  COLUMN 18 PIC ZZ,ZZZ,ZZ9.99 SUM SLS-AMOUNT.

       01  TYPE PAGE FOOTING.
           03  LINE PLUS 1.
               05  COLUMN 1 PIC X(20) VALUE "-- END OF PAGE --".

       PROCEDURE DIVISION.
           OPEN INPUT SALES-FILE.
           OPEN OUTPUT REPORT-FILE.
           INITIATE REGIONAL-REPORT.

           READ SALES-FILE
               AT END MOVE "Y" TO WS-EOF-FLAG
           END-READ.
           PERFORM UNTIL WS-EOF-FLAG = "Y"
               GENERATE SALES-DETAIL
               READ SALES-FILE
                   AT END MOVE "Y" TO WS-EOF-FLAG
               END-READ
           END-PERFORM.

           TERMINATE REGIONAL-REPORT.
           CLOSE SALES-FILE.
           CLOSE REPORT-FILE.
           STOP RUN.
```

### อธิบายโค้ด

โปรแกรมนี้ผสานทุกเทคนิคของ Part 039: `CONTROLS ARE` สองระดับ, `TYPE CONTROL HEADING`/
`FOOTING` ของทั้งสองระดับ, `SUM` สำหรับยอดรวมย่อยและยอดรวมทั้งหมด (`TYPE CONTROL FOOTING
FINAL`), `TYPE REPORT HEADING`/`PAGE HEADING`/`PAGE FOOTING` จาก Part 038 และการอ่านไฟล์จริง
ที่**เรียงลำดับมาแล้ว**ตามที่ย้ำในขั้นตอนที่ 389 — โปรดสังเกต (ตามข้อค้นพบในขั้นตอนที่ 384) ว่า
ลำดับการพิมพ์ "REGION:" กับ "CITY:" ตอนเกิด control break พร้อมกันทั้งสองระดับ อาจสลับกันใน
บิลด์ GnuCOBOL นี้ ผู้เรียนควรรันโปรแกรมนี้ด้วยตนเองและสังเกตผลลัพธ์จริงเทียบกับที่คาดหวัง

### ผลลัพธ์ที่ได้จากการรันจริง

```
REGIONAL SALES REPORT
ITEM               AMOUNT
  CITY:  CHIANGMAI

REGION:  NORTH
    APPLE             1,000.00
    BANANA            2,505.00
  CITY TOTAL:         3,505.00
  CITY:  LAMPANG
    CHERRY            3,000.00
  CITY TOTAL:         3,000.00
REGION TOTAL:         6,505.00
  CITY:  PHUKET

REGION:  SOUTH
    DURIAN            5,000.00
  CITY TOTAL:         5,000.00
  CITY:  SURATTHAN
    MANGO             1,800.00
  CITY TOTAL:         1,800.00
REGION TOTAL:         6,800.00

COMPANY TOTAL:       13,305.00
-- END OF PAGE --
```

(ผลลัพธ์นี้ยืนยันข้อค้นพบจากขั้นตอนที่ 384 อีกครั้ง: ลำดับ "CITY:" มาก่อน "REGION:" ตอนเกิด
การเปลี่ยนกลุ่มพร้อมกันทั้งสองระดับ แต่ยอดรวมทุกตัวคำนวณถูกต้องสมบูรณ์: 1,000+2,505=3,505 สำหรับ
เชียงใหม่, รวมกับลำปาง 3,000 ได้ยอดภาคเหนือ 6,505 และยอดรวมทั้งบริษัท 13,305 ตรงตามการคำนวณ
ด้วยมือทุกประการ)

### ข้อควรระวัง

- ยอดรวมทุกระดับ (CITY TOTAL, REGION TOTAL, COMPANY TOTAL) ถูกต้องแม่นยำแม้ลำดับการพิมพ์หัวข้อ
  จะมีข้อค้นพบตามขั้นตอนที่ 384 — ควรตรวจสอบตัวเลขยอดรวมด้วยการคำนวณมือเทียบกับผลลัพธ์เสมอเมื่อ
  นำไปใช้งานจริงครั้งแรก
- ไฟล์ข้อมูลในตัวอย่างนี้ถูกเรียงลำดับด้วยมือไว้ล่วงหน้า ในระบบจริงควรใช้ `SORT` (Part 027) เป็น
  ขั้นตอนก่อนหน้าเสมอ โดยเฉพาะเมื่อข้อมูลมาจากหลายแหล่งหรือถูกเพิ่มเข้ามาไม่เรียงลำดับ

### แบบฝึกหัดที่ 390.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น โดยเพิ่มการนับจำนวนรายการสินค้า (record count) ในแต่ละภูมิภาค
แสดงควบคู่กับยอดขายรวมของภูมิภาคนั้น

**เฉลยแนวทาง**: ทางเลือกแรกที่คิดได้ตามทฤษฎีมาตรฐานคือ `SUM 1` (บวกค่าคงที่ 1 ทุกครั้งที่
`GENERATE`) แต่เมื่อทดสอบจริงกับ GnuCOBOL 4.0-early-dev บิลด์นี้ **`SUM` ตามด้วยตัวเลข literal
ตรง ๆ (`SUM 1`) ทำให้ตัวคอมไพเลอร์ `cobc` ล่มทันทีระหว่าง codegen** (`attempt to reference
unallocated memory (signal SIGSEGV)`) — เป็นบั๊กของคอมไพเลอร์เอง ไม่ใช่ข้อผิดพลาดทางไวยากรณ์
ของเรา จึง**ห้ามใช้ `SUM` กับตัวเลขคงที่ตรง ๆ ในบิลด์นี้เด็ดขาด**

ทางแก้ที่ทดสอบแล้วว่าใช้งานได้จริงคือประกาศตัวแปรเก็บค่า 1 แยกต่างหาก แล้ว `SUM` จากตัวแปรนั้นแทน:
```cobol
       WORKING-STORAGE SECTION.
       01  WS-ONE            PIC 9 VALUE 1.
       ...
       01  TYPE CONTROL FOOTING SLS-REGION.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(16) VALUE "REGION TOTAL:".
               05  COLUMN 18 PIC ZZ,ZZZ,ZZ9.99 SUM SLS-AMOUNT.
           03  LINE PLUS 1.
               05  COLUMN 1  PIC X(16) VALUE "ITEM COUNT:".
               05  COLUMN 18 PIC ZZ9           SUM WS-ONE.
```
ทดสอบแล้วว่า `SUM WS-ONE` (ตัวแปรที่มีค่าคงที่ 1 เสมอ) คอมไพล์ผ่านและให้ผลลัพธ์ถูกต้อง คือนับ
จำนวนครั้งที่ `GENERATE` เกิดขึ้นในกลุ่มนั้นได้อย่างแม่นยำ — นี่คืออีกตัวอย่างสำคัญที่ย้ำหลักการของ
สอง Part นี้: **ทดสอบคอมไพล์และรันจริงทุกครั้งก่อนเชื่อ แม้แต่รูปแบบที่ดูสมเหตุสมผลตามทฤษฎีที่สุดก็อาจ
ทำให้คอมไพเลอร์ล่มได้ในทางปฏิบัติ**

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้ **Control Break** ซึ่งเป็นความสามารถขั้นสูงของ Report Writer อย่างครบถ้วน
โดยทดสอบคอมไพล์และรันจริงทุกตัวอย่างด้วย GnuCOBOL 4.0 (early-dev):

- `CONTROLS ARE` และ `TYPE CONTROL HEADING` — แนวคิดพื้นฐานของการตรวจจับการเปลี่ยนกลุ่ม
- `TYPE CONTROL FOOTING` และ `SUM` clause — ยอดรวมของแต่ละกลุ่มที่คำนวณอัตโนมัติ
- `TYPE CONTROL FOOTING FINAL` — ยอดรวมทั้งฉบับที่พิมพ์ครั้งเดียวตอน `TERMINATE`
- Multi-level control break พร้อม**ข้อค้นพบจริง**เรื่องลำดับการพิมพ์ `TYPE CONTROL HEADING`
  หลายระดับที่สลับกันในบิลด์ที่ทดสอบ
- `NEXT GROUP` สำหรับควบคุมระยะห่างระหว่างกลุ่ม
- `SUM ... UPON` พร้อม**ข้อค้นพบจริง**ว่าไม่กรองตามชนิด detail ตามมาตรฐาน และทางเลือกที่ปลอดภัย
  กว่าด้วยตัวแปรสะสมเอง
- Control break ร่วมกับการขึ้นหน้าใหม่กลางกลุ่ม
- การอ่านข้อมูลจากไฟล์จริงที่เรียงลำดับมาแล้วเพื่อสร้างรายงาน
- **อันตรายของข้อมูลที่ไม่เรียงลำดับ** พิสูจน์ด้วยผลการทดสอบจริงที่แสดงกลุ่มซ้ำซ้อนที่ไม่ควรเกิดขึ้น
- โปรแกรมรวบยอดรายงานยอดขายแยกภูมิภาค/จังหวัดที่ผสานทุกเทคนิคเข้าด้วยกัน

Part นี้ปิดท้ายเนื้อหาเรื่อง Report Writer ของหลักสูตร (Part 038-039) ด้วยข้อคิดสำคัญที่สุด:
**Report Writer เป็นเครื่องมือที่ทรงพลังมากเมื่อใช้ถูกวิธี แต่ต้องทดสอบผลลัพธ์จริงเสมอ ไม่ว่าจะเป็น
เรื่องลำดับการพิมพ์หรือพฤติกรรมของ clause เฉพาะทางอย่าง `UPON`** — หลักการ "ทดสอบก่อนเชื่อ"
นี้จะติดตัวคุณไปตลอดการเขียนโปรแกรม COBOL ระดับมืออาชีพ

ใน **Part 040** เราจะเปลี่ยนโหมดจากการพิมพ์รายงานแบบ batch (เขียนลงไฟล์/กระดาษ) ไปสู่การสร้าง
**หน้าจอโต้ตอบกับผู้ใช้แบบ Text-mode** ผ่าน **SCREEN SECTION** ซึ่งเป็นอีกหนึ่งฟีเจอร์ที่ทำให้ COBOL
สามารถสร้างโปรแกรมแบบ Interactive ได้โดยไม่ต้องพึ่งพา `ACCEPT`/`DISPLAY` แบบพื้นฐานที่เรียนใน
Part 007 เท่านั้น

**[← กลับไป Part 038](part-038-report-writer-basics.md)** | **[ไปยัง Part 040: Screen Section — สร้าง UI แบบ Text-mode →](part-040-screen-section.md)**
