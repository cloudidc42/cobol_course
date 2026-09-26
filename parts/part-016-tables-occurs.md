# Part 016: ตาราง (Tables) และ OCCURS Clause (ขั้นตอนที่ 151–160)

## คำนำของ Part นี้

Part 015 ที่ผ่านมาเป็นโปรเจกต์รวบยอดของเฟส 1 ที่นำ IF-ELSE, EVALUATE, PERFORM UNTIL และ PERFORM VARYING
มาประกอบกันเป็นโปรแกรมเครื่องคิดเลขและระบบคำนวณเกรดที่ใช้งานได้จริง ระหว่างทางเราได้แตะ `OCCURS` แบบ
ผิวเผินไปบ้างแล้ว (ใน Part 013 ขั้นตอนที่ 126-127 ตอนสอน `PERFORM VARYING` ควบคู่กับตาราง) แต่ยังไม่เคย
เจาะลึกว่า `OCCURS` มีความสามารถอะไรบ้าง ประกาศแบบไหนได้บ้าง และมีข้อควรระวังอะไรบ้าง

Part นี้คือจุดเริ่มต้นของ **เฟส 2: ระดับกลาง (Intermediate)** ซึ่งจะเน้นการจัดการข้อมูลจำนวนมากอย่างเป็น
ระบบ — และเครื่องมือแรกที่สำคัญที่สุดคือ **ตาราง (Table)** ที่ COBOL สร้างขึ้นด้วย `OCCURS` Clause
ตารางคือโครงสร้างข้อมูลที่ให้เราเก็บค่าจำนวนมากภายใต้ **ชื่อเดียว** แทนที่จะต้องประกาศตัวแปรแยกกัน
ทีละตัว ซึ่งเป็นทักษะที่จำเป็นอย่างยิ่งสำหรับงานประมวลผลข้อมูลระดับองค์กร ไม่ว่าจะเป็นยอดขายรายเดือน
คะแนนสอบของนักเรียนทั้งชั้น หรือรายการสินค้าในคลัง

Part นี้จะพาคุณตั้งแต่แนวคิดพื้นฐานว่าทำไมต้องมีตาราง ไปจนถึงการประกาศตารางหลายรูปแบบ (ตัวเลข ตัวอักษร
กลุ่มฟิลด์) การเข้าถึงข้อมูลด้วย Subscript การกำหนดค่าเริ่มต้น การประมวลผลสถิติพื้นฐาน (ผลรวม ค่าเฉลี่ย
ค่าสูงสุด-ต่ำสุด) ไปจนถึง `OCCURS ... DEPENDING ON` สำหรับตารางที่มีขนาดไม่คงที่ ส่วนเทคนิคขั้นสูงกว่านี้
อย่างการใช้ `INDEXED BY` กับ `SEARCH`/`SEARCH ALL` จะแยกไปเรียนอย่างละเอียดใน Part 017 และตารางหลายมิติ
(Multi-dimensional Tables) จะเรียนใน Part 018 ต่อไป

---

## ขั้นตอนที่ 151: ทำไมต้องมีตาราง — ปัญหาของการประกาศตัวแปรแยกกันหลายตัว

### แนวคิด

ลองจินตนาการว่าต้องเก็บคะแนนสอบของนักเรียน 5 คน วิธีที่เราเคยใช้มาตลอด (Part 001-015) คือประกาศตัวแปร
แยกกัน 5 ตัว เช่น `WS-SCORE-1`, `WS-SCORE-2`, ..., `WS-SCORE-5` ซึ่งพอมีแค่ 5 ตัวก็ยังพอไหว แต่ถ้าเป็น
50 คน หรือ 500 คน วิธีนี้จะกลายเป็นฝันร้ายทันที ทั้งตอนประกาศตัวแปรและตอนเขียนโค้ดประมวลผล (จะต้องเขียน
`IF`/`DISPLAY`/`ADD` ซ้ำแยกกันทีละตัวแปรทั้งหมด)

**ตาราง (Table)** คือคำตอบของปัญหานี้ — COBOL ให้เราประกาศ**ชื่อเดียว**ที่มีหลาย "ช่อง" (Occurrence)
ภายในตัว ด้วย Clause ที่ชื่อว่า `OCCURS` แล้วเข้าถึงแต่ละช่องด้วยหมายเลขลำดับ (Subscript) เช่น
`WS-SCORE(1)`, `WS-SCORE(2)` และที่สำคัญที่สุดคือสามารถใช้ **ตัวแปร** เป็น Subscript ร่วมกับ `PERFORM`
เพื่อวนประมวลผลทุกช่องด้วยโค้ดเพียงไม่กี่บรรทัด ไม่ว่าตารางจะมี 5 ช่องหรือ 5,000 ช่องก็ตาม

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP151.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> --- Without a table: one variable per student (painful) ---
       01  WS-SCORE-1          PIC 9(03) VALUE 80.
       01  WS-SCORE-2          PIC 9(03) VALUE 95.
       01  WS-SCORE-3          PIC 9(03) VALUE 70.
       01  WS-SCORE-4          PIC 9(03) VALUE 88.
       01  WS-SCORE-5          PIC 9(03) VALUE 60.

      *> --- With a table: one name, many occurrences ---
       01  WS-SCORE-TABLE.
           05  WS-SCORE        PIC 9(03) OCCURS 5 TIMES.
       01  WS-IDX              PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Without OCCURS: 5 separate variables ===".
           DISPLAY "Score 1: " WS-SCORE-1.
           DISPLAY "Score 2: " WS-SCORE-2.
           DISPLAY "Score 3: " WS-SCORE-3.
           DISPLAY "Score 4: " WS-SCORE-4.
           DISPLAY "Score 5: " WS-SCORE-5.
           DISPLAY " ".

           DISPLAY "=== With OCCURS: one table, five slots ===".
           MOVE 80 TO WS-SCORE(1).
           MOVE 95 TO WS-SCORE(2).
           MOVE 70 TO WS-SCORE(3).
           MOVE 88 TO WS-SCORE(4).
           MOVE 60 TO WS-SCORE(5).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               DISPLAY "Score(" WS-IDX ") = " WS-SCORE(WS-IDX)
           END-PERFORM.

           DISPLAY " ".
           DISPLAY "Imagine 500 students: the table code above".
           DISPLAY "stays exactly the same size. The separate".
           DISPLAY "variable version would need 500 lines.".
           STOP RUN.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step151 step151.cob
./step151
```

### ผลลัพธ์จริงที่ได้

```
=== Without OCCURS: 5 separate variables ===
Score 1: 080
Score 2: 095
Score 3: 070
Score 4: 088
Score 5: 060
 
=== With OCCURS: one table, five slots ===
Score(01) = 080
Score(02) = 095
Score(03) = 070
Score(04) = 088
Score(05) = 060
 
Imagine 500 students: the table code above
stays exactly the same size. The separate
variable version would need 500 lines.
```

### อธิบายโค้ดทีละส่วน

- `05 WS-SCORE PIC 9(03) OCCURS 5 TIMES.` คือการประกาศตาราง — `OCCURS 5 TIMES` บอกว่าฟิลด์นี้มี 5 ช่อง
  ซ้ำกัน แต่ละช่องมีชนิดข้อมูลเป็น `PIC 9(03)` เหมือนกันทุกช่อง
- สังเกตว่า `WS-SCORE` อยู่ภายใต้ `WS-SCORE-TABLE` (ระดับ 01) ซึ่งเป็น **Group Item** — ตัวตารางเองต้อง
  อยู่ที่ระดับย่อยกว่าเสมอ (ในที่นี้คือระดับ 05) ไม่สามารถประกาศ `OCCURS` ตรงที่ระดับ 01 ได้ถ้าต้องการ
  ให้มีชื่อกลุ่มครอบตารางไว้ (แม้ในทางเทคนิค COBOL อนุญาตให้ `OCCURS` อยู่ที่ระดับ 01 ได้เช่นกัน
  แต่ไม่นิยมเพราะจะไม่มีชื่อกลุ่มให้ `MOVE`/`INITIALIZE` ทั้งตารางในคำสั่งเดียวได้)
- `PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5` คือรูปแบบมาตรฐานที่จะใช้ไล่ประมวลผลตารางตลอด
  ทั้งหลักสูตร — ทบทวนจาก Part 013
- จุดสำคัญที่สุดของตัวอย่างนี้: โค้ดส่วนที่ใช้ตาราง (การประกาศ + ลูป) จะมีขนาด**เท่าเดิม**ไม่ว่าจะมี
  นักเรียนกี่คน เพียงแค่เปลี่ยนตัวเลขใน `OCCURS n TIMES` และ `UNTIL WS-IDX > n` เท่านั้น

### ข้อควรระวัง

- Subscript ใน COBOL **เริ่มต้นที่ 1 เสมอ** ไม่ใช่ 0 เหมือนภาษา C/Java/Python — เป็นจุดที่ผู้มาจาก
  ภาษาอื่นมักเขียนผิดเมื่อเริ่มเรียน COBOL (จะเจาะลึกปัญหานี้อีกครั้งในขั้นตอนที่ 159)
- อย่าพยายามแก้ปัญหาข้อมูลจำนวนมากด้วยการสร้างตัวแปรแยกกันหลายสิบ/หลายร้อยตัว เพราะนอกจากจะเขียนโค้ด
  ยาวแล้ว การแก้ไข/บำรุงรักษาก็ยากขึ้นแบบทวีคูณ ควรมองหาสัญญาณว่า "ข้อมูลชุดนี้มีลักษณะซ้ำกันเป็นชุด"
  ตั้งแต่ตอนออกแบบ `DATA DIVISION`

### แบบฝึกหัดที่ 151.1

**โจทย์**: จงขยายตารางคะแนนให้รองรับนักเรียน 8 คนแทนที่จะเป็น 5 คน (ไม่ต้องแก้เวอร์ชันตัวแปรแยก)

**เฉลย**: แก้ `OCCURS 5 TIMES` เป็น `OCCURS 8 TIMES` แล้วเพิ่มค่าและแก้เงื่อนไขลูป:

```cobol
       01  WS-SCORE-TABLE.
           05  WS-SCORE        PIC 9(03) OCCURS 8 TIMES.
       ...
           MOVE 80 TO WS-SCORE(1).
           MOVE 95 TO WS-SCORE(2).
           MOVE 70 TO WS-SCORE(3).
           MOVE 88 TO WS-SCORE(4).
           MOVE 60 TO WS-SCORE(5).
           MOVE 75 TO WS-SCORE(6).
           MOVE 90 TO WS-SCORE(7).
           MOVE 65 TO WS-SCORE(8).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 8
               DISPLAY "Score(" WS-IDX ") = " WS-SCORE(WS-IDX)
           END-PERFORM.
```

สังเกตว่าโครงสร้างโค้ดไม่ได้เปลี่ยนไปเลย เปลี่ยนแค่ตัวเลขสองจุด (`OCCURS` และเงื่อนไข `UNTIL`)

---

## ขั้นตอนที่ 152: การประกาศตารางพื้นฐานด้วย OCCURS กับชนิดข้อมูลต่าง ๆ

### แนวคิด

`OCCURS` ใช้ได้กับชนิดข้อมูลใดก็ได้ที่ COBOL รองรับ ไม่ว่าจะเป็นตัวเลข ตัวอักษร หรือแม้แต่ 1 ตัวอักษร
(flag) รูปแบบพื้นฐานคือ:

```
level-number  item-name  PICTURE clause  OCCURS integer TIMES.
```

โดย `integer` คือจำนวนช่องทั้งหมดที่ต้องการ (ต้องเป็นค่าคงที่ตอน compile ในกรณีพื้นฐานนี้ — ส่วนตาราง
ที่ขนาดยืดหยุ่นได้จะเรียนใน `OCCURS ... DEPENDING ON` ที่ขั้นตอนที่ 157)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP152.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DAY-TABLE.
           05  WS-DAY-NAME     PIC X(09) OCCURS 7 TIMES.

       01  WS-PRICE-TABLE.
           05  WS-PRICE        PIC 9(05)V99 OCCURS 4 TIMES.

       01  WS-FLAG-TABLE.
           05  WS-ACTIVE-FLAG  PIC X(01) OCCURS 3 TIMES.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Declaring OCCURS tables of different types ===".

           MOVE "Monday" TO WS-DAY-NAME(1).
           MOVE "Tuesday" TO WS-DAY-NAME(2).
           MOVE "Wednesday" TO WS-DAY-NAME(3).
           DISPLAY "Day 1: " WS-DAY-NAME(1).
           DISPLAY "Day 2: " WS-DAY-NAME(2).
           DISPLAY "Day 3: " WS-DAY-NAME(3).

           MOVE 199.50 TO WS-PRICE(1).
           MOVE 2500.00 TO WS-PRICE(2).
           DISPLAY "Price 1: " WS-PRICE(1).
           DISPLAY "Price 2: " WS-PRICE(2).

           MOVE "Y" TO WS-ACTIVE-FLAG(1).
           MOVE "N" TO WS-ACTIVE-FLAG(2).
           DISPLAY "Flag 1: " WS-ACTIVE-FLAG(1).
           DISPLAY "Flag 2: " WS-ACTIVE-FLAG(2).

           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Declaring OCCURS tables of different types ===
Day 1: Monday   
Day 2: Tuesday  
Day 3: Wednesday
Price 1: 00199.50
Price 2: 02500.00
Flag 1: Y
Flag 2: N
```

### อธิบายโค้ดทีละส่วน

- `WS-DAY-NAME PIC X(09) OCCURS 7 TIMES` คือตารางตัวอักษร 7 ช่อง ช่องละ 9 ตัวอักษร — เหมาะสำหรับเก็บ
  ชื่อวันในสัปดาห์ สังเกตผลลัพธ์ว่าคำว่า "Monday" (6 ตัวอักษร) และ "Tuesday" (7 ตัวอักษร) ถูกเติมด้วย
  ช่องว่างทางขวาจนครบ 9 ตัวอักษรอัตโนมัติ (พฤติกรรม MOVE ตัวอักษรที่เรียนใน Part 008)
- `WS-PRICE PIC 9(05)V99 OCCURS 4 TIMES` คือตารางตัวเลขทศนิยม 4 ช่อง — `V99` หมายถึงจุดทศนิยมโดยนัย
  2 ตำแหน่ง (ทบทวนจาก Part 006) ผลลัพธ์แสดง `00199.50` และ `02500.00` ตามรูปแบบ `PICTURE` ที่กำหนด
- `WS-ACTIVE-FLAG PIC X(01) OCCURS 3 TIMES` คือตารางตัวอักษรตัวเดียว มักใช้เป็น Flag/Switch สำหรับ
  แต่ละรายการ เช่น "รายการนี้ Active หรือไม่"

### ข้อควรระวัง

- จำนวน `OCCURS n TIMES` แบบพื้นฐานต้องเป็น**ค่าคงที่**ที่กำหนดตอน compile เท่านั้น ไม่สามารถใช้ตัวแปร
  มากำหนดจำนวนช่องได้โดยตรง (ถ้าต้องการขนาดยืดหยุ่นได้ ต้องใช้ `OCCURS ... DEPENDING ON` ในขั้นตอนที่ 157)
- ต้องคำนวณขนาดหน่วยความจำที่ใช้ล่วงหน้าเสมอ เช่น `WS-DAY-NAME PIC X(09) OCCURS 7 TIMES` จะใช้พื้นที่
  ทั้งหมด 9 × 7 = 63 ไบต์ ยิ่งตารางมีขนาดใหญ่และ `OCCURS` มาก ยิ่งต้องระวังเรื่องหน่วยความจำที่ใช้ทั้งหมด
  ของโปรแกรม โดยเฉพาะระบบ Mainframe รุ่นเก่าที่มีข้อจำกัดหน่วยความจำเข้มงวด

### แบบฝึกหัดที่ 152.1

**โจทย์**: จงประกาศตารางใหม่ชื่อ `WS-MONTH-NUM` เก็บหมายเลขเดือน (1-12) จำนวน 12 ช่อง ชนิดข้อมูล
`PIC 9(02)` แล้วกำหนดค่าช่องแรกและช่องสุดท้ายเป็น 1 และ 12

**เฉลย**:

```cobol
       01  WS-MONTH-TABLE.
           05  WS-MONTH-NUM    PIC 9(02) OCCURS 12 TIMES.
       ...
           MOVE 1 TO WS-MONTH-NUM(1).
           MOVE 12 TO WS-MONTH-NUM(12).
           DISPLAY "First month: " WS-MONTH-NUM(1).
           DISPLAY "Last month: " WS-MONTH-NUM(12).
```

---

## ขั้นตอนที่ 153: การเข้าถึงสมาชิกตารางด้วย Subscript (ค่าคงที่, ตัวแปร, นิพจน์)

### แนวคิด

**Subscript** คือหมายเลขที่ใส่ในวงเล็บต่อท้ายชื่อตารางเพื่อระบุว่าต้องการเข้าถึงช่องไหน (เช่น
`WS-TEMP-C(3)`) COBOL รองรับ Subscript ได้ 3 รูปแบบ:

1. **ค่าคงที่ (Literal)**: เช่น `WS-TEMP-C(3)` เข้าถึงช่องที่ 3 ตรง ๆ
2. **ตัวแปร (Identifier)**: เช่น `WS-TEMP-C(WS-IDX)` เข้าถึงช่องตามค่าที่เก็บอยู่ใน `WS-IDX` ขณะนั้น
   — รูปแบบนี้คือหัวใจของการวนลูปประมวลผลตาราง
3. **นิพจน์ (Expression)**: เช่น `WS-TEMP-C(WS-IDX - 1)` หรือ `WS-TEMP-C(WS-IDX + 1)` ใช้บวก/ลบเลขจำนวน
   เต็มกับตัวแปรก่อนนำไปเป็น Subscript ได้โดยตรง มีประโยชน์มากเมื่อต้องการเปรียบเทียบ "ช่องปัจจุบัน"
   กับ "ช่องก่อนหน้า/ถัดไป"

> **หมายเหตุ**: Part นี้ยังใช้ Subscript แบบตัวแปรธรรมดา (data item ทั่วไป) ส่วนการใช้ **Index** ที่ประกาศ
> ด้วย `INDEXED BY` (ซึ่งเร็วกว่าและใช้คู่กับ `SEARCH`) จะแยกไปเรียนอย่างละเอียดใน Part 017

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP153.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEMP-TABLE.
           05  WS-TEMP-C       PIC S9(03) OCCURS 6 TIMES.
       01  WS-IDX              PIC 9(02).
       01  WS-NEXT-IDX         PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Subscripts: literal, variable, expression ===".
           MOVE 25 TO WS-TEMP-C(1).
           MOVE 27 TO WS-TEMP-C(2).
           MOVE 30 TO WS-TEMP-C(3).
           MOVE 28 TO WS-TEMP-C(4).
           MOVE 22 TO WS-TEMP-C(5).
           MOVE 19 TO WS-TEMP-C(6).

           DISPLAY "-- 1. Literal subscript --".
           DISPLAY "WS-TEMP-C(3) = " WS-TEMP-C(3).

           DISPLAY "-- 2. Variable subscript --".
           MOVE 5 TO WS-IDX.
           DISPLAY "WS-TEMP-C(WS-IDX) where WS-IDX=5 : "
                   WS-TEMP-C(WS-IDX).

           DISPLAY "-- 3. Expression subscript (WS-IDX - 1) --".
           DISPLAY "WS-TEMP-C(WS-IDX - 1) : " WS-TEMP-C(WS-IDX - 1).

           DISPLAY "-- 4. Comparing today vs tomorrow's slot --".
           MOVE 2 TO WS-IDX.
           MOVE WS-IDX TO WS-NEXT-IDX.
           ADD 1 TO WS-NEXT-IDX.
           DISPLAY "Day " WS-IDX ": " WS-TEMP-C(WS-IDX).
           DISPLAY "Day " WS-NEXT-IDX ": " WS-TEMP-C(WS-NEXT-IDX).
           IF WS-TEMP-C(WS-NEXT-IDX) > WS-TEMP-C(WS-IDX)
               DISPLAY "Getting warmer tomorrow."
           ELSE
               DISPLAY "Not warmer tomorrow."
           END-IF.

           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Subscripts: literal, variable, expression ===
-- 1. Literal subscript --
WS-TEMP-C(3) = +030
-- 2. Variable subscript --
WS-TEMP-C(WS-IDX) where WS-IDX=5 : +022
-- 3. Expression subscript (WS-IDX - 1) --
WS-TEMP-C(WS-IDX - 1) : +028
-- 4. Comparing today vs tomorrow's slot --
Day 02: +027
Day 03: +030
Getting warmer tomorrow.
```

### อธิบายโค้ดทีละส่วน

- `WS-TEMP-C` ประกาศเป็น `PIC S9(03)` (มีเครื่องหมาย) เพื่อรองรับอุณหภูมิติดลบได้ในสถานการณ์จริง — จึง
  เห็นเครื่องหมาย `+` นำหน้าทุกค่าที่แสดงผล (ทบทวนพฤติกรรมนี้ได้จาก Part 013 ขั้นตอนที่ 122)
- ตัวอย่างที่ 3 ใช้ `WS-TEMP-C(WS-IDX - 1)` โดยที่ `WS-IDX` ยังคงมีค่า 5 จากขั้นตอนก่อนหน้า ดังนั้น
  `WS-IDX - 1` จึงเท่ากับ 4 และ `WS-TEMP-C(4)` มีค่า 28 ตรงกับผลลัพธ์ที่ได้ — สังเกตว่า**นิพจน์ในวงเล็บ
  ไม่ได้เปลี่ยนแปลงค่าจริงของ `WS-IDX`** เป็นเพียงการคำนวณชั่วคราวเพื่อหาตำแหน่งเท่านั้น
- ตัวอย่างที่ 4 แสดงรูปแบบการใช้งานจริงที่พบบ่อย: เปรียบเทียบ "ค่าวันนี้" กับ "ค่าพรุ่งนี้" โดยใช้ตัวแปร
  สองตัว (`WS-IDX` และ `WS-NEXT-IDX`) แทนการคำนวณนิพจน์ในวงเล็บโดยตรง ซึ่งอ่านง่ายกว่าและปลอดภัยกว่า
  เมื่อโค้ดมีความซับซ้อนมากขึ้น

### ข้อควรระวัง

- นิพจน์ Subscript อย่าง `WS-TEMP-C(WS-IDX + 1)` มีความเสี่ยงสูงที่จะทำให้เข้าถึงตำแหน่งเกินขอบเขตตาราง
  ถ้า `WS-IDX` เป็นค่าสุดท้ายอยู่แล้ว (เช่น `WS-IDX = 6` ในตารางนี้ `WS-TEMP-C(WS-IDX + 1)` จะกลายเป็น
  `WS-TEMP-C(7)` ซึ่งไม่มีอยู่จริง) ต้องตรวจสอบขอบเขตก่อนใช้นิพจน์แบบนี้เสมอ (ดูขั้นตอนที่ 159)
  สำหรับรายละเอียดปัญหาที่เกิดขึ้นเมื่อเข้าถึงเกินขอบเขต
- การใช้ Subscript แบบตัวแปรธรรมดา (ไม่ใช่ `INDEXED BY`) จะมีการคำนวณตำแหน่งหน่วยความจำ (offset)
  ทุกครั้งที่เข้าถึง ซึ่งช้ากว่า Index เล็กน้อยในทางเทคนิค แต่สำหรับตารางขนาดเล็กถึงปานกลางความแตกต่างนี้
  ไม่มีนัยสำคัญ (จะเปรียบเทียบให้เห็นชัดเจนใน Part 017)

### แบบฝึกหัดที่ 153.1

**โจทย์**: จงเขียนโค้ดเปรียบเทียบว่าอุณหภูมิช่องที่ 1 กับช่องที่ 6 (ช่องแรกกับช่องสุดท้าย) ช่องไหนสูงกว่า

**เฉลย**:

```cobol
           IF WS-TEMP-C(1) > WS-TEMP-C(6)
               DISPLAY "Slot 1 is warmer than slot 6."
           ELSE
               DISPLAY "Slot 6 is warmer than or equal to slot 1."
           END-IF.
```

จากข้อมูลในตัวอย่าง (`WS-TEMP-C(1) = 25`, `WS-TEMP-C(6) = 19`) ผลลัพธ์จะเป็น
"Slot 1 is warmer than slot 6."

---

## ขั้นตอนที่ 154: VALUE Clause และ INITIALIZE กับตาราง

### แนวคิด

เราสามารถกำหนดค่าเริ่มต้นให้**ทุกช่อง**ของตารางพร้อมกันด้วย `VALUE` Clause ต่อท้าย `OCCURS` ได้ในคำสั่ง
เดียว โดยทุกช่องจะได้รับค่าเดียวกันหมด (ต่างจากการ `MOVE` ทีละช่องที่ต้องเขียนแยกกัน) นอกจากนี้ COBOL
ยังมีคำสั่ง `INITIALIZE` ที่ใช้**รีเซ็ตค่ากลับไปเป็นค่าเริ่มต้นตาม VALUE ที่ประกาศไว้**ได้ทั้งตารางในคำสั่ง
เดียว มีประโยชน์มากเมื่อต้องการล้างค่าตารางก่อนเริ่มประมวลผลรอบใหม่

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP154.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER-TABLE.
           05  WS-HIT-COUNT    PIC 9(03) OCCURS 5 TIMES VALUE ZERO.

       01  WS-GRADE-TABLE.
           05  WS-GRADE        PIC X(01) OCCURS 4 TIMES VALUE "F".

       01  WS-IDX              PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== VALUE clause and INITIALIZE with tables ===".

           DISPLAY "-- 1. All slots start at the same VALUE --".
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               DISPLAY "WS-HIT-COUNT(" WS-IDX ") = "
                       WS-HIT-COUNT(WS-IDX)
           END-PERFORM.

           DISPLAY "-- 2. Same idea with an alphanumeric table --".
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 4
               DISPLAY "WS-GRADE(" WS-IDX ") = " WS-GRADE(WS-IDX)
           END-PERFORM.

           DISPLAY "-- 3. Change some values, then INITIALIZE back --".
           MOVE 10 TO WS-HIT-COUNT(1).
           MOVE 20 TO WS-HIT-COUNT(2).
           DISPLAY "After MOVE: WS-HIT-COUNT(1)=" WS-HIT-COUNT(1)
                   " WS-HIT-COUNT(2)=" WS-HIT-COUNT(2).

           INITIALIZE WS-COUNTER-TABLE.
           DISPLAY "After INITIALIZE: WS-HIT-COUNT(1)="
                   WS-HIT-COUNT(1) " WS-HIT-COUNT(2)="
                   WS-HIT-COUNT(2).

           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== VALUE clause and INITIALIZE with tables ===
-- 1. All slots start at the same VALUE --
WS-HIT-COUNT(01) = 000
WS-HIT-COUNT(02) = 000
WS-HIT-COUNT(03) = 000
WS-HIT-COUNT(04) = 000
WS-HIT-COUNT(05) = 000
-- 2. Same idea with an alphanumeric table --
WS-GRADE(01) = F
WS-GRADE(02) = F
WS-GRADE(03) = F
WS-GRADE(04) = F
-- 3. Change some values, then INITIALIZE back --
After MOVE: WS-HIT-COUNT(1)=010 WS-HIT-COUNT(2)=020
After INITIALIZE: WS-HIT-COUNT(1)=000 WS-HIT-COUNT(2)=000
```

### อธิบายโค้ดทีละส่วน

- `05 WS-HIT-COUNT PIC 9(03) OCCURS 5 TIMES VALUE ZERO.` ทำให้ทั้ง 5 ช่องมีค่าเริ่มต้นเป็น 0 ทันทีที่
  โปรแกรมเริ่มทำงาน โดยไม่ต้องเขียน `MOVE ZERO TO WS-HIT-COUNT(1)` ซ้ำห้าครั้ง
- `05 WS-GRADE PIC X(01) OCCURS 4 TIMES VALUE "F".` เช่นเดียวกัน แต่เป็นตัวอักษร — ทุกช่องเริ่มต้นเป็น
  "F" พร้อมกันหมด เหมาะกับสถานการณ์ เช่น "ตั้งเกรดเริ่มต้นของทุกวิชาเป็น F ก่อนที่จะคำนวณเกรดจริง"
- `INITIALIZE WS-COUNTER-TABLE.` คือคำสั่งที่ทรงพลังมาก — มันจะรีเซ็ตทุกฟิลด์ภายใต้ `WS-COUNTER-TABLE`
  (รวมถึงทุกช่องของตาราง `WS-HIT-COUNT`) ให้กลับไปเป็นค่าตาม `VALUE` ที่ประกาศไว้ (คือ ZERO) โดยอัตโนมัติ
  — จากผลลัพธ์จะเห็นว่าหลัง `MOVE 10`/`MOVE 20` แล้วค่อย `INITIALIZE` ค่าทั้งสองช่องกลับไปเป็น 000
  เหมือนเดิม

### ข้อควรระวัง

- `VALUE` ที่ต่อท้าย `OCCURS` จะกำหนดค่าเดียวกันให้**ทุกช่อง**เท่านั้น ไม่สามารถกำหนดค่าเริ่มต้นที่
  แตกต่างกันในแต่ละช่องผ่าน `VALUE` ธรรมดาได้ (ถ้าต้องการค่าเริ่มต้นต่างกันในแต่ละช่อง ต้องใช้ `MOVE`
  แยกทีละช่อง หรือใช้เทคนิค `REDEFINES` ร่วมกับค่าคงที่ที่จะเรียนใน Part 022)
- `INITIALIZE` จะรีเซ็ตกลับไปตาม `VALUE` ที่ประกาศไว้ **หรือค่าเริ่มต้นมาตรฐานของชนิดข้อมูล** (ตัวเลข
  เป็น 0, ตัวอักษรเป็นช่องว่าง) หากฟิลด์ใดไม่มี `VALUE` กำหนดไว้เลย — ต้องระวังว่า `INITIALIZE` จะรีเซ็ต
  **ทุกฟิลด์ย่อย**ภายใต้ Group Item นั้น ถ้ามีฟิลด์อื่นที่ไม่ต้องการให้ถูกรีเซ็ตด้วย ต้องแยกออกไปอยู่
  Group Item คนละกลุ่ม

### แบบฝึกหัดที่ 154.1

**โจทย์**: จงเพิ่มตารางใหม่ `WS-STATUS-TABLE` ที่มี `WS-STATUS-FLAG PIC X(01) OCCURS 3 TIMES` โดยกำหนด
ค่าเริ่มต้นทุกช่องเป็น "N" แล้วทดสอบด้วย `INITIALIZE`

**เฉลย**:

```cobol
       01  WS-STATUS-TABLE.
           05  WS-STATUS-FLAG  PIC X(01) OCCURS 3 TIMES VALUE "N".
       ...
           MOVE "Y" TO WS-STATUS-FLAG(1).
           DISPLAY "Before INITIALIZE: " WS-STATUS-FLAG(1).
           INITIALIZE WS-STATUS-TABLE.
           DISPLAY "After INITIALIZE: " WS-STATUS-FLAG(1).
```

ผลลัพธ์: "Before INITIALIZE: Y" ตามด้วย "After INITIALIZE: N" (กลับไปเป็นค่าเริ่มต้นตามที่ `VALUE`
กำหนดไว้)

---

## ขั้นตอนที่ 155: OCCURS บน Group Item — ตารางของระเบียนข้อมูล (Table of Records)

### แนวคิด

จนถึงตอนนี้ตารางทุกตัวที่เราเห็นเก็บข้อมูลแค่ **1 ฟิลด์ต่อช่อง** แต่ในงานจริง ข้อมูลหนึ่งรายการมักมี
หลายฟิลด์ประกอบกัน เช่น พนักงานหนึ่งคนมีทั้งชื่อ เงินเดือน และแผนก COBOL รองรับสิ่งนี้ได้ด้วยการใส่
`OCCURS` ไว้ที่ **Group Item** (ไม่ใช่ Elementary Item เดี่ยว ๆ) แล้วให้ฟิลด์ย่อยหลายตัวอยู่ภายใต้กลุ่ม
นั้น — ผลลัพธ์คือ "ตารางของระเบียนข้อมูล" (Table of Records) ที่แต่ละช่องมีหลายฟิลด์ในตัว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP155.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EMPLOYEE-TABLE.
           05  WS-EMPLOYEE     OCCURS 3 TIMES.
               10  WS-EMP-NAME     PIC X(10).
               10  WS-EMP-SALARY   PIC 9(06)V99.
               10  WS-EMP-DEPT     PIC X(04).

       01  WS-IDX              PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== OCCURS on a group item (table of records) ===".

           MOVE "Somchai"  TO WS-EMP-NAME(1).
           MOVE 35000.00   TO WS-EMP-SALARY(1).
           MOVE "SALE"     TO WS-EMP-DEPT(1).

           MOVE "Suda"     TO WS-EMP-NAME(2).
           MOVE 42000.50   TO WS-EMP-SALARY(2).
           MOVE "ACCT"     TO WS-EMP-DEPT(2).

           MOVE "Anan"     TO WS-EMP-NAME(3).
           MOVE 28000.00   TO WS-EMP-SALARY(3).
           MOVE "IT  "     TO WS-EMP-DEPT(3).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               DISPLAY "Employee " WS-IDX ": "
                       WS-EMP-NAME(WS-IDX) " | Dept="
                       WS-EMP-DEPT(WS-IDX) " | Salary="
                       WS-EMP-SALARY(WS-IDX)
           END-PERFORM.

           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== OCCURS on a group item (table of records) ===
Employee 01: Somchai    | Dept=SALE | Salary=035000.00
Employee 02: Suda       | Dept=ACCT | Salary=042000.50
Employee 03: Anan       | Dept=IT   | Salary=028000.00
```

### อธิบายโค้ดทีละส่วน

- `05 WS-EMPLOYEE OCCURS 3 TIMES.` คือ Group Item ที่ทำซ้ำ 3 ครั้ง — สังเกตว่า `WS-EMPLOYEE` เองไม่มี
  `PICTURE` เพราะมันเป็นกลุ่ม ไม่ใช่ฟิลด์เดี่ยว
- ฟิลด์ย่อยทั้งสาม (`WS-EMP-NAME`, `WS-EMP-SALARY`, `WS-EMP-DEPT`) ที่ระดับ 10 คือข้อมูลของ "พนักงาน
  หนึ่งคน" — เมื่อ `WS-EMPLOYEE` ทำซ้ำ 3 ครั้ง ฟิลด์ย่อยทั้งสามนี้ก็จะถูกจำลองซ้ำไปด้วย 3 ชุดโดยอัตโนมัติ
- การเข้าถึงข้อมูลทำได้โดยใส่ Subscript ต่อท้าย**ฟิลด์ย่อย**โดยตรง เช่น `WS-EMP-NAME(1)`,
  `WS-EMP-SALARY(2)` — ไม่ใช่ต่อท้ายชื่อกลุ่ม `WS-EMPLOYEE` ซึ่งเป็นรูปแบบมาตรฐานที่ใช้กันทั่วไปเมื่อ
  ต้องการอ้างอิงฟิลด์ใดฟิลด์หนึ่งของระเบียนที่ตำแหน่งใดตำแหน่งหนึ่ง
- รูปแบบนี้คือพื้นฐานสำคัญที่จะใช้ตลอดในโลกธุรกิจจริง เช่น ตารางสินค้าคงคลัง (รหัส, ชื่อ, จำนวน, ราคา)
  หรือตารางรายการสั่งซื้อ (รหัสสินค้า, จำนวน, ราคารวม) ซึ่งจะกลับมาใช้ในโปรเจกต์ระบบจัดการสินค้าคงคลัง
  ที่ Part 035

### ข้อควรระวัง

- ฟิลด์ย่อยภายใน Group Item ที่มี `OCCURS` (เช่น `WS-EMP-NAME`) **ทุกตัวจะถูกทำซ้ำไปพร้อมกัน** ไม่สามารถ
  ทำให้ฟิลด์ย่อยบางตัว "มีแค่ช่องเดียว" ในขณะที่ฟิลด์อื่นมีหลายช่องได้ ถ้าต้องการข้อมูลบางอย่างที่ไม่
  ซ้ำตามจำนวน occurrence (เช่น "จำนวนพนักงานทั้งหมด") ต้องแยกไปประกาศเป็นฟิลด์ต่างหากนอก Group Item
  ที่มี `OCCURS` (ดังตัวอย่างจริงในขั้นตอนที่ 160)
- ต้องระวังการตั้งชื่อฟิลด์ย่อยให้สื่อความหมายชัดเจน เพราะเมื่อโปรแกรมมีหลายตารางของระเบียนพร้อมกัน
  การเข้าถึงฟิลด์ผิดตารางโดยไม่ตั้งใจ (เช่นสับสนระหว่าง `WS-EMP-NAME` กับตารางลูกค้าอีกตารางที่มีฟิลด์
  `NAME` เหมือนกัน) เป็นเรื่องที่เกิดขึ้นได้ง่ายถ้าตั้งชื่อไม่ระวัง

### แบบฝึกหัดที่ 155.1

**โจทย์**: เพิ่มพนักงานคนที่ 4 ชื่อ "Piti" แผนก "HR" เงินเดือน 31000.00 (ต้องขยาย `OCCURS` เป็น 4 ด้วย)

**เฉลย**:

```cobol
       01  WS-EMPLOYEE-TABLE.
           05  WS-EMPLOYEE     OCCURS 4 TIMES.
               10  WS-EMP-NAME     PIC X(10).
               10  WS-EMP-SALARY   PIC 9(06)V99.
               10  WS-EMP-DEPT     PIC X(04).
       ...
           MOVE "Piti"     TO WS-EMP-NAME(4).
           MOVE 31000.00   TO WS-EMP-SALARY(4).
           MOVE "HR  "     TO WS-EMP-DEPT(4).
       ...
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 4
```

---

## ขั้นตอนที่ 156: การประมวลผลสถิติของตาราง — ผลรวม ค่าเฉลี่ย ค่าสูงสุด-ต่ำสุด

### แนวคิด

หนึ่งในงานที่พบบ่อยที่สุดเมื่อมีตาราง คือการคำนวณ**สถิติสรุป** จากข้อมูลทั้งหมดในตาราง เช่น ผลรวม
(Total), ค่าเฉลี่ย (Average), ค่าสูงสุด (Maximum) และค่าต่ำสุด (Minimum) — ทั้งหมดนี้ทำได้ด้วยการวนลูป
**เพียงรอบเดียว** ผ่านตาราง โดยสะสมผลลัพธ์ไปพร้อมกันในตัวแปรหลายตัว (Accumulate-while-iterate pattern
ที่แนะนำไว้ใน Part 013) แทนที่จะต้องวนลูปแยกกันสำหรับแต่ละสถิติ

เทคนิคการหาค่าสูงสุด/ต่ำสุด: ตั้งตัวแปรเก็บค่าสูงสุดเริ่มต้นที่ **น้อยที่สุดเท่าที่เป็นไปได้** (เช่น 0)
และตัวแปรเก็บค่าต่ำสุดเริ่มต้นที่ **มากที่สุดเท่าที่เป็นไปได้** (เช่น 999999) แล้วเทียบค่าทุกช่องกับค่า
เหล่านี้ทีละรอบ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP156.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SALES-TABLE.
           05  WS-SALES-AMT    PIC 9(06) OCCURS 6 TIMES.
       01  WS-IDX              PIC 9(02).
       01  WS-TOTAL            PIC 9(07) VALUE ZERO.
       01  WS-AVERAGE          PIC 9(06)V99 VALUE ZERO.
       01  WS-MAX-VAL          PIC 9(06) VALUE ZERO.
       01  WS-MIN-VAL          PIC 9(06) VALUE 999999.
       01  WS-MAX-IDX          PIC 9(02) VALUE ZERO.
       01  WS-MIN-IDX          PIC 9(02) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Table statistics: sum/avg/max/min ===".
           MOVE 12000 TO WS-SALES-AMT(1).
           MOVE 18500 TO WS-SALES-AMT(2).
           MOVE 9000  TO WS-SALES-AMT(3).
           MOVE 22000 TO WS-SALES-AMT(4).
           MOVE 15500 TO WS-SALES-AMT(5).
           MOVE 8000  TO WS-SALES-AMT(6).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 6
               DISPLAY "Month " WS-IDX ": " WS-SALES-AMT(WS-IDX)
               ADD WS-SALES-AMT(WS-IDX) TO WS-TOTAL
               IF WS-SALES-AMT(WS-IDX) > WS-MAX-VAL
                   MOVE WS-SALES-AMT(WS-IDX) TO WS-MAX-VAL
                   MOVE WS-IDX TO WS-MAX-IDX
               END-IF
               IF WS-SALES-AMT(WS-IDX) < WS-MIN-VAL
                   MOVE WS-SALES-AMT(WS-IDX) TO WS-MIN-VAL
                   MOVE WS-IDX TO WS-MIN-IDX
               END-IF
           END-PERFORM.

           COMPUTE WS-AVERAGE = WS-TOTAL / 6.

           DISPLAY "Total sales   : " WS-TOTAL.
           DISPLAY "Average sales : " WS-AVERAGE.
           DISPLAY "Best month    : " WS-MAX-IDX
                   " (" WS-MAX-VAL ")".
           DISPLAY "Worst month   : " WS-MIN-IDX
                   " (" WS-MIN-VAL ")".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Table statistics: sum/avg/max/min ===
Month 01: 012000
Month 02: 018500
Month 03: 009000
Month 04: 022000
Month 05: 015500
Month 06: 008000
Total sales   : 0085000
Average sales : 014166.66
Best month    : 04 (022000)
Worst month   : 06 (008000)
```

### อธิบายโค้ดทีละส่วน

- `WS-MAX-VAL VALUE ZERO` เริ่มต้นที่ 0 (น้อยที่สุดเท่าที่ยอดขายจะเป็นไปได้) ทำให้ค่าจริงตัวแรกที่เจอ
  (12000) จะมากกว่าเสมอและถูกบันทึกเป็นค่าสูงสุดชั่วคราวทันที
- `WS-MIN-VAL VALUE 999999` เริ่มต้นที่ค่าสูงสุดเท่าที่ `PIC 9(06)` จะเก็บได้ ทำให้ค่าจริงตัวแรกที่เจอ
  จะน้อยกว่าเสมอและถูกบันทึกเป็นค่าต่ำสุดชั่วคราวทันที — เทคนิคนี้เรียกว่า "sentinel initial value"
  และเป็นวิธีมาตรฐานในการหาค่าสูงสุด/ต่ำสุดโดยไม่ต้องมีเงื่อนไขพิเศษสำหรับรอบแรก
- `WS-MAX-IDX`/`WS-MIN-IDX` เก็บ**ตำแหน่ง**ของเดือนที่ดีที่สุด/แย่ที่สุดไว้ด้วย ไม่ใช่แค่ค่าตัวเลข
  เพราะในรายงานจริงเรามักต้องการรู้ว่า "เดือนไหน" ไม่ใช่แค่ "ยอดเท่าไหร่"
- ทั้งหมดนี้ทำในลูป**เดียว** — ผลรวม ค่าสูงสุด และค่าต่ำสุด ถูกคำนวณไปพร้อมกันในการวนลูปเพียงรอบเดียว
  ซึ่งมีประสิทธิภาพกว่าการวนลูปแยกกัน 3 รอบสำหรับแต่ละสถิติมาก

### ข้อควรระวัง

- ถ้าลืมกำหนดค่าเริ่มต้นของ `WS-MIN-VAL` ให้สูงพอ (เช่นลืมใส่ `VALUE 999999` ปล่อยเป็น 0 ตามค่าเริ่มต้น
  ของตัวเลข) การหาค่าต่ำสุดจะผิดพลาดทันที เพราะเงื่อนไข `WS-SALES-AMT(WS-IDX) < WS-MIN-VAL` จะไม่มีวัน
  เป็นจริงเลย (เนื่องจากยอดขายจริงย่อมมากกว่า 0 เสมอ) นี่คือข้อผิดพลาดที่พบบ่อยมากสำหรับมือใหม่
- `COMPUTE WS-AVERAGE = WS-TOTAL / 6` ใช้เลข 6 คงที่ตรง ๆ ในตัวอย่างนี้เพื่อความง่าย แต่ในโค้ดจริงควร
  ใช้ตัวแปรแทน (เช่น จำนวนเดือนทั้งหมดที่มีข้อมูลจริง) เพื่อไม่ให้ต้องแก้โค้ดทุกครั้งที่ขนาดตารางเปลี่ยน
  (จะเห็นแนวทางนี้ชัดเจนขึ้นเมื่อผสานกับ `OCCURS ... DEPENDING ON` ในขั้นตอนที่ 160)

### แบบฝึกหัดที่ 156.1

**โจทย์**: จงเพิ่มการนับ**จำนวนเดือนที่ยอดขายต่ำกว่า 10000** ลงในโปรแกรมนี้ด้วย

**เฉลย**: เพิ่มตัวแปรนับและเงื่อนไขในลูปเดิม:

```cobol
       01  WS-LOW-MONTH-COUNT  PIC 9(02) VALUE ZERO.
       ...
               IF WS-SALES-AMT(WS-IDX) < 10000
                   ADD 1 TO WS-LOW-MONTH-COUNT
               END-IF
           END-PERFORM.
           ...
           DISPLAY "Months below 10000: " WS-LOW-MONTH-COUNT.
```

จากข้อมูลตัวอย่าง (9000 และ 8000 ต่ำกว่า 10000) ผลลัพธ์ที่ได้จะเป็น "Months below 10000: 02"

---

## ขั้นตอนที่ 157: OCCURS ... DEPENDING ON — ตารางที่มีขนาดยืดหยุ่นได้ (Variable-length Table)

### แนวคิด

ตารางที่เราประกาศมาจนถึงตอนนี้ทั้งหมดมีขนาด**คงที่**ตาม `OCCURS n TIMES` แต่ในงานจริงจำนวนรายการมักไม่
แน่นอน เช่น ใบสั่งซื้อหนึ่งใบอาจมี 1 รายการหรือ 20 รายการก็ได้ COBOL แก้ปัญหานี้ด้วย
**`OCCURS ... DEPENDING ON`** (มักย่อว่า **ODO** — Occurs Depending On) ซึ่งให้เราประกาศตารางแบบ
"ขนาดต่ำสุดถึงสูงสุด" พร้อมตัวแปรตัวหนึ่งที่บอกว่า **ตอนนี้มีข้อมูลจริงกี่ช่อง**:

```
OCCURS min-value TO max-value TIMES
       DEPENDING ON counter-field
```

ตัวแปร `counter-field` ต้องประกาศไว้**ก่อน**ตารางในลำดับ `DATA DIVISION` และเป็นตัวกำหนดว่า ณ ขณะนั้น
ตารางมีข้อมูลจริงอยู่กี่ช่อง — โปรแกรมต้องคอยอัปเดตค่าตัวแปรนี้เองให้ตรงกับข้อมูลจริงเสมอ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP157.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ITEM-COUNT       PIC 9(02) VALUE ZERO.
       01  WS-ORDER-TABLE.
           05  WS-ORDER-ITEM   PIC X(10)
                               OCCURS 1 TO 10 TIMES
                               DEPENDING ON WS-ITEM-COUNT.
       01  WS-IDX              PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== OCCURS ... DEPENDING ON (variable table) ===".

           MOVE 3 TO WS-ITEM-COUNT.
           MOVE "Rice"   TO WS-ORDER-ITEM(1).
           MOVE "Sugar"  TO WS-ORDER-ITEM(2).
           MOVE "Salt"   TO WS-ORDER-ITEM(3).

           DISPLAY "Order has " WS-ITEM-COUNT " item(s):".
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-ITEM-COUNT
               DISPLAY "  Item " WS-IDX ": " WS-ORDER-ITEM(WS-IDX)
           END-PERFORM.

           DISPLAY "-- Now the order grows to 5 items --".
           MOVE 5 TO WS-ITEM-COUNT.
           MOVE "Oil"    TO WS-ORDER-ITEM(4).
           MOVE "Eggs"   TO WS-ORDER-ITEM(5).

           DISPLAY "Order has " WS-ITEM-COUNT " item(s):".
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-ITEM-COUNT
               DISPLAY "  Item " WS-IDX ": " WS-ORDER-ITEM(WS-IDX)
           END-PERFORM.

           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== OCCURS ... DEPENDING ON (variable table) ===
Order has 03 item(s):
  Item 01: Rice      
  Item 02: Sugar     
  Item 03: Salt      
-- Now the order grows to 5 items --
Order has 05 item(s):
  Item 01: Rice      
  Item 02: Sugar     
  Item 03: Salt      
  Item 04: Oil       
  Item 05: Eggs      
```

### อธิบายโค้ดทีละส่วน

- `01 WS-ITEM-COUNT PIC 9(02) VALUE ZERO.` ต้องประกาศ**ก่อน**ตาราง `WS-ORDER-TABLE` ในลำดับ
  `WORKING-STORAGE SECTION` เสมอ — นี่คือกฎบังคับของ ODO ไม่ใช่แค่ธรรมเนียม
- `OCCURS 1 TO 10 TIMES DEPENDING ON WS-ITEM-COUNT` บอกว่าตารางนี้มีอย่างน้อย 1 ช่อง อย่างมาก 10 ช่อง
  และจำนวนที่ "ใช้งานจริง ณ ขณะนั้น" ขึ้นอยู่กับค่าของ `WS-ITEM-COUNT`
- เมื่อ `WS-ITEM-COUNT` เป็น 3 ลูป `PERFORM VARYING ... UNTIL WS-IDX > WS-ITEM-COUNT` จะประมวลผลแค่ 3
  ช่องแรกเท่านั้น แม้ว่าตารางจะมีพื้นที่จองไว้สูงสุดถึง 10 ช่องก็ตาม
- เมื่อโปรแกรมอัปเดต `WS-ITEM-COUNT` เป็น 5 และเพิ่มข้อมูลในช่องที่ 4-5 ลูปเดิม (ที่ใช้เงื่อนไข
  `UNTIL WS-IDX > WS-ITEM-COUNT` เหมือนเดิมทุกตัวอักษร) จะประมวลผลครบ 5 ช่องโดยอัตโนมัติ**โดยไม่ต้อง
  แก้โค้ดลูปเลย** — นี่คือประโยชน์หลักของ ODO เมื่อเทียบกับ `OCCURS n TIMES` แบบคงที่

### ข้อควรระวัง

- **กฎที่พลาดไม่ได้**: ตัวแปรที่ใช้ใน `DEPENDING ON` ต้องมีค่า**ตรงกับจำนวนข้อมูลจริง**อยู่เสมอ ถ้า
  โปรแกรมเพิ่มข้อมูลลงตารางแต่ลืมอัปเดตตัวแปรนี้ (เช่น ลืม `MOVE 5 TO WS-ITEM-COUNT` ในตัวอย่างข้างต้น)
  ลูปที่ใช้ `UNTIL WS-IDX > WS-ITEM-COUNT` จะยังคงอ่านแค่ข้อมูลเก่า ทำให้รายการที่เพิ่มใหม่ถูก "มองข้าม"
  ไปโดยไม่มี error แจ้งเตือนใด ๆ
- ค่า `min-value` และ `max-value` ต้องเป็นค่าคงที่ตอน compile และตัวแปรใน `DEPENDING ON` ต้องมีขนาด
  ใหญ่พอที่จะเก็บค่าได้ถึง `max-value` เสมอ (ในตัวอย่างนี้ `WS-ITEM-COUNT PIC 9(02)` รองรับได้ถึง 99
  ซึ่งเกินพอสำหรับ `max` ที่กำหนดไว้คือ 10)
- ODO มักใช้คู่กับ `FILE SECTION` ในสถานการณ์จริง (เช่น เรกคอร์ดที่มีจำนวนรายการย่อยไม่แน่นอนต่อหนึ่ง
  ใบสั่งซื้อ) ซึ่งจะกลับมาเจออีกครั้งเมื่อเรียนเรื่องไฟล์ใน Part 023 เป็นต้นไป

### แบบฝึกหัดที่ 157.1

**โจทย์**: จงเพิ่มรายการที่ 6 ชื่อ "Flour" เข้าไปในออร์เดอร์ (อย่าลืมอัปเดตตัวนับให้ถูกต้อง)

**เฉลย**:

```cobol
           MOVE 6 TO WS-ITEM-COUNT.
           MOVE "Flour"  TO WS-ORDER-ITEM(6).
           DISPLAY "Order has " WS-ITEM-COUNT " item(s):".
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-ITEM-COUNT
               DISPLAY "  Item " WS-IDX ": " WS-ORDER-ITEM(WS-IDX)
           END-PERFORM.
```

จุดสำคัญคือต้อง `MOVE 6 TO WS-ITEM-COUNT` **ก่อน** เพื่อให้ลูปมองเห็นรายการที่ 6 ด้วย มิฉะนั้นแม้จะ
`MOVE "Flour" TO WS-ORDER-ITEM(6)` ไปแล้ว ลูปก็ยังจะแสดงแค่ 5 รายการเดิมเท่านั้น

---

## ขั้นตอนที่ 158: การคัดลอกและเปรียบเทียบตารางทั้งชุด

### แนวคิด

เมื่อสองตารางมีโครงสร้าง (PICTURE และ OCCURS) เหมือนกันทุกประการ COBOL อนุญาตให้ `MOVE`
**ทั้งตาราง**ในคำสั่งเดียว โดยอ้างอิงชื่อ Group Item ที่ครอบตารางไว้ (ไม่ใช่ชื่อฟิลด์ระดับล่างที่มี
`OCCURS`) วิธีนี้สะดวกกว่าการเขียนลูปคัดลอกทีละช่องมาก แต่การ**เปรียบเทียบ**ว่าสองตารางเหมือนกันหรือไม่
ยังคงต้องวนลูปเปรียบเทียบทีละช่อง เพราะ COBOL ไม่มีคำสั่งเปรียบเทียบทั้งตารางในคำสั่งเดียวแบบ `MOVE`

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP158.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TABLE-A.
           05  WS-A-VAL        PIC 9(04) OCCURS 4 TIMES.
       01  WS-TABLE-B.
           05  WS-B-VAL        PIC 9(04) OCCURS 4 TIMES.
       01  WS-IDX              PIC 9(02).
       01  WS-DIFF-COUNT       PIC 9(02) VALUE ZERO.
       01  WS-TABLES-EQUAL     PIC X(01) VALUE "Y".
           88  WS-SAME-TABLES          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Copying and comparing whole tables ===".
           MOVE 100 TO WS-A-VAL(1).
           MOVE 200 TO WS-A-VAL(2).
           MOVE 300 TO WS-A-VAL(3).
           MOVE 400 TO WS-A-VAL(4).

           DISPLAY "-- 1. Copy the entire table with one MOVE --".
           MOVE WS-TABLE-A TO WS-TABLE-B.
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 4
               DISPLAY "B(" WS-IDX ") = " WS-B-VAL(WS-IDX)
           END-PERFORM.

           DISPLAY "-- 2. Change one slot in B, then compare --".
           MOVE 999 TO WS-B-VAL(3).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 4
               IF WS-A-VAL(WS-IDX) NOT = WS-B-VAL(WS-IDX)
                   ADD 1 TO WS-DIFF-COUNT
                   MOVE "N" TO WS-TABLES-EQUAL
                   DISPLAY "  Difference at slot " WS-IDX
                           ": A=" WS-A-VAL(WS-IDX)
                           " B=" WS-B-VAL(WS-IDX)
               END-IF
           END-PERFORM.

           IF WS-SAME-TABLES
               DISPLAY "Tables are identical."
           ELSE
               DISPLAY "Tables differ in " WS-DIFF-COUNT " slot(s)."
           END-IF.

           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Copying and comparing whole tables ===
-- 1. Copy the entire table with one MOVE --
B(01) = 0100
B(02) = 0200
B(03) = 0300
B(04) = 0400
-- 2. Change one slot in B, then compare --
  Difference at slot 03: A=0300 B=0999
Tables differ in 01 slot(s).
```

### อธิบายโค้ดทีละส่วน

- `MOVE WS-TABLE-A TO WS-TABLE-B.` คัดลอกข้อมูลทั้งหมด 4 ช่องจากตาราง A ไปตาราง B ในคำสั่งเดียว
  เพราะทั้งสองประกาศด้วยโครงสร้างเดียวกันทุกประการ (`PIC 9(04) OCCURS 4 TIMES`) — COBOL จะคัดลอกเป็น
  ไบต์ต่อไบต์ตามลำดับหน่วยความจำ ผลลัพธ์ที่ได้ตรงกับค่าในตาราง A ทุกประการ
- หลังจากแก้ค่าช่องที่ 3 ของ B เป็น 999 โปรแกรมวนลูปเปรียบเทียบทีละช่องด้วย
  `IF WS-A-VAL(WS-IDX) NOT = WS-B-VAL(WS-IDX)` — นี่คือวิธีเดียวที่ทำได้ใน COBOL มาตรฐาน เพราะไม่มี
  คำสั่ง `IF WS-TABLE-A = WS-TABLE-B` ที่เปรียบเทียบทั้งกลุ่มแบบมีความหมายทาง logic ตรง ๆ (ในทางเทคนิค
  COBOL อนุญาตให้เปรียบเทียบ Group Item แบบ byte-by-byte ได้ในบาง Dialect แต่ไม่ใช่แนวทางที่แนะนำ
  เพราะอ่านยากและพึ่งพา encoding ของข้อมูลมากเกินไป ควรวนลูปเปรียบเทียบทีละฟิลด์เสมอเพื่อความชัดเจน)
- `WS-DIFF-COUNT` และ `WS-TABLES-EQUAL` (Flag พร้อม Condition Name) ทำงานร่วมกันเพื่อสรุปผลลัพธ์
  สุดท้ายว่าตารางเหมือนกันหรือต่างกันกี่จุด

### ข้อควรระวัง

- `MOVE table-a TO table-b` ทำงานถูกต้องก็ต่อเมื่อทั้งสองตารางมี**โครงสร้างเหมือนกันทุกประการ**
  (จำนวนช่อง, PICTURE ของแต่ละฟิลด์) หากขนาดหรือโครงสร้างไม่ตรงกัน ข้อมูลอาจถูกคัดลอกผิดตำแหน่งโดยไม่มี
  error เตือน (COBOL จะคัดลอกไปตามขนาดไบต์ที่ระบุ ไม่ตรวจสอบความหมายเชิงตรรกะ)
- การเปรียบเทียบตารางที่มีขนาดใหญ่มากด้วยการวนลูปทีละช่องอาจใช้เวลานานถ้าข้อมูลมีจำนวนมาก ในระบบจริง
  ที่ต้องเทียบข้อมูลปริมาณมาก มักมีการออกแบบให้เรียงลำดับข้อมูลก่อน หรือใช้เทคนิคการค้นหาที่มี
  ประสิทธิภาพกว่า (เช่น `SEARCH ALL` ที่จะเรียนใน Part 017) แทนการเทียบแบบ Brute-force ทุกช่อง

### แบบฝึกหัดที่ 158.1

**โจทย์**: จงแก้โค้ดให้ตรวจสอบกรณีตารางเหมือนกันทุกช่อง (ไม่แก้ไขค่าใด ๆ ใน B เลย) แล้วดูข้อความสรุป
ที่ได้

**เฉลย**: ลบหรือคอมเมนต์บรรทัด `MOVE 999 TO WS-B-VAL(3).` ทิ้ง แล้วรันใหม่ ผลลัพธ์ในส่วนที่ 2 จะไม่มี
บรรทัด "Difference at slot" ปรากฏเลย และข้อความสุดท้ายจะเป็น "Tables are identical." แทน เพราะ
`WS-TABLES-EQUAL` ยังคงเป็น "Y" (ค่าเริ่มต้น) ตลอดการวนลูป

---

## ขั้นตอนที่ 159: ข้อผิดพลาดที่พบบ่อยเกี่ยวกับ OCCURS และ Subscript

### แนวคิด

ตารางเป็นหนึ่งในจุดที่มือใหม่ COBOL สร้างบั๊กที่**อันตรายที่สุด**ได้บ่อยที่สุด เพราะการเข้าถึง Subscript
เกินขอบเขตมักไม่ทำให้เกิด compile error หรือแม้แต่ runtime error ในโหมดมาตรฐาน — โปรแกรมจะยังคง "รันได้"
แต่ให้ผลลัพธ์ที่ผิดเพี้ยนโดยไม่มีการเตือนใด ๆ บทนี้จะสาธิตปัญหานี้ให้เห็นชัดเจนด้วยการรันจริง

### ข้อผิดพลาดที่ 1: Subscript เกินขอบเขตของ OCCURS (สาธิตจริง)

```cobol
      *> WRONG: table has 5 slots, but the loop goes up to slot 6
       01  WS-SAFE-TABLE.
           05  WS-SAFE-VAL     PIC 9(03) OCCURS 5 TIMES.
       01  WS-GUARD            PIC 9(03) VALUE 777.
       01  WS-IDX              PIC 9(02).
       ...
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 6
               DISPLAY "WS-SAFE-VAL(" WS-IDX ") = "
                       WS-SAFE-VAL(WS-IDX)
           END-PERFORM.
```

เมื่อคอมไพล์และรันโค้ดนี้จริงด้วย GnuCOBOL (ตั้งค่าตารางแค่ 5 ช่องแต่ลูปวนถึง 6) ผลลัพธ์ที่ได้คือ:

```
=== Demo: reading past the end of a table ===
WS-SAFE-VAL(01) = 010
WS-SAFE-VAL(02) = 020
WS-SAFE-VAL(03) = 030
WS-SAFE-VAL(04) = 040
WS-SAFE-VAL(05) = 050
WS-SAFE-VAL(06) = 06
WS-GUARD (declared right after the table) = 777
```

**สังเกตบรรทัด `WS-SAFE-VAL(06) = 06`** — โปรแกรม**ไม่ error เลย**แต่แสดงค่าที่ไม่มีความหมายอะไรออกมา
(ไม่ใช่ 060 ตามรูปแบบปกติของฟิลด์อื่น ๆ) เพราะช่องที่ 6 ไม่มีอยู่จริงในตารางที่ประกาศไว้เพียง 5 ช่อง
GnuCOBOL ในโหมดมาตรฐาน (ไม่เปิด flag ตรวจสอบขอบเขตเพิ่มเติม) จะอ่านหน่วยความจำที่อยู่ถัดจากตารางไป
ตรง ๆ โดยไม่ตรวจสอบว่าถูกต้องหรือไม่ ผลลัพธ์ที่ได้จึงเป็น**ค่าที่คาดเดาไม่ได้**และขึ้นอยู่กับการจัด
เรียงหน่วยความจำภายในของคอมไพเลอร์ ซึ่งอาจต่างกันไปในแต่ละเครื่อง/แต่ละเวอร์ชันของคอมไพเลอร์ — นี่คือ
เหตุผลที่การเข้าถึง Subscript เกินขอบเขตถูกจัดว่าเป็น **undefined behavior** ที่อันตรายมาก

### เวอร์ชันที่ถูกต้อง (compile และรันจริง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP159.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SAFE-TABLE.
           05  WS-SAFE-VAL     PIC 9(03) OCCURS 5 TIMES.
       01  WS-IDX              PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Correct pattern: loop bound matches OCCURS ===".
           MOVE 10 TO WS-SAFE-VAL(1).
           MOVE 20 TO WS-SAFE-VAL(2).
           MOVE 30 TO WS-SAFE-VAL(3).
           MOVE 40 TO WS-SAFE-VAL(4).
           MOVE 50 TO WS-SAFE-VAL(5).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               DISPLAY "WS-SAFE-VAL(" WS-IDX ") = "
                       WS-SAFE-VAL(WS-IDX)
           END-PERFORM.

           DISPLAY "Finished safely: never touched slot 6.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้ (เวอร์ชันที่ถูกต้อง)

```
=== Correct pattern: loop bound matches OCCURS ===
WS-SAFE-VAL(01) = 010
WS-SAFE-VAL(02) = 020
WS-SAFE-VAL(03) = 030
WS-SAFE-VAL(04) = 040
WS-SAFE-VAL(05) = 050
Finished safely: never touched slot 6.
```

### สรุปรายการข้อผิดพลาดที่พบบ่อยเกี่ยวกับ OCCURS/Subscript

| ข้อผิดพลาด | อาการ | วิธีป้องกัน |
|---|---|---|
| ลูปวนเกินจำนวนช่องที่ `OCCURS` ประกาศไว้ | ค่าที่ไม่มีความหมาย (garbage) ปรากฏออกมาโดยไม่มี error | ตรวจสอบเสมอว่าเงื่อนไข `UNTIL`/`TIMES` ตรงกับจำนวนช่องจริงของตาราง |
| ใช้ subscript เริ่มที่ 0 (ตามความเคยชินจากภาษาอื่น) | ผลลัพธ์เพี้ยนตั้งแต่ช่องแรก หรือ error/undefined behavior | จำไว้เสมอว่า COBOL เริ่ม subscript ที่ 1 |
| นิพจน์ subscript (`idx + 1`) เกินขอบเขตในรอบสุดท้ายของลูป | เข้าถึงช่องที่ไม่มีอยู่จริงในรอบท้าย ๆ | ตรวจสอบขอบเขตด้วย `IF` ก่อนใช้นิพจน์ subscript ที่ขยับตำแหน่ง |
| ลืมอัปเดตตัวแปรใน `DEPENDING ON` ให้ตรงกับข้อมูลจริง | รายการที่เพิ่มใหม่ถูกมองข้ามไปเงียบ ๆ | อัปเดตตัวนับทันทีทุกครั้งที่เพิ่ม/ลบข้อมูลในตาราง (ดูขั้นตอนที่ 157) |

### ข้อควรระวัง

- เนื่องจาก GnuCOBOL ในโหมดมาตรฐานไม่ตรวจสอบขอบเขต Subscript ให้อัตโนมัติ **ความรับผิดชอบทั้งหมดตกอยู่
  ที่ผู้เขียนโปรแกรม** ต้องเขียนเงื่อนไขลูปให้ตรงกับขนาดตารางเสมอ และควรทดสอบด้วยขอบเขตที่ถูกต้องก่อน
  ทุกครั้งที่แก้ไขขนาด `OCCURS`
- ในระบบ Mainframe จริง การเข้าถึง Subscript เกินขอบเขตอาจร้ายแรงถึงขั้นทำให้โปรแกรมเขียนทับข้อมูลของ
  ตัวแปรอื่นในหน่วยความจำโดยไม่รู้ตัว (memory corruption) ซึ่งอาจทำให้เกิดบั๊กที่ตามหาสาเหตุได้ยากมาก
  เพราะอาการจะไปปรากฏที่ตัวแปรอื่นที่ไม่เกี่ยวข้องโดยตรงกับตารางที่มีปัญหา

### แบบฝึกหัดที่ 159.1

**โจทย์**: จงอธิบายว่าทำไมค่าที่ได้จากการเข้าถึง `WS-SAFE-VAL(6)` ในตัวอย่างข้างต้นจึงแสดงเป็น "06"
แทนที่จะเป็นค่าอื่น

**เฉลยแนวทาง**: คำตอบที่ถูกต้องคือ **ไม่มีใครรับประกันได้แน่นอน** ว่าทำไมถึงได้ค่านั้น เพราะนี่คือ
undefined behavior — ค่าที่ได้ขึ้นอยู่กับว่าหน่วยความจำที่อยู่ถัดจากขอบเขตตารางในขณะนั้นมีข้อมูลอะไร
อยู่ ซึ่งอาจเป็นผลจากการจัดวางตัวแปรของคอมไพเลอร์ ค่าที่หลงเหลือจากการคำนวณก่อนหน้า หรือค่าขยะในหน่วย
ความจำ ประเด็นสำคัญที่ต้องเรียนรู้จากแบบฝึกหัดนี้ไม่ใช่ "ทำไมได้ค่านี้" แต่คือ **ห้ามเข้าถึง Subscript
เกินขอบเขตโดยเด็ดขาด** เพราะผลลัพธ์คาดเดาไม่ได้และอาจต่างกันไปในแต่ละครั้งที่รัน หรือแต่ละเครื่องที่ใช้

---

## ขั้นตอนที่ 160: มินิโปรเจกต์ — รายงานยอดขายรายเดือนด้วยตาราง (Table-driven Report)

### แนวคิด

ปิดท้าย Part นี้ด้วยการนำเทคนิคทั้งหมดที่เรียนมารวมกันเป็นโปรแกรมเดียว: ตารางของระเบียนข้อมูล (ขั้นตอน
ที่ 155) ผสานกับ `OCCURS ... DEPENDING ON` (ขั้นตอนที่ 157) และการคำนวณสถิติ (ขั้นตอนที่ 156) เพื่อสร้าง
**รายงานยอดขายรายเดือนที่ขนาดยืดหยุ่นได้** — รองรับได้ตั้งแต่ 1 ถึง 12 เดือน โดยไม่ต้องแก้โครงสร้างโค้ด
เมื่อจำนวนเดือนเปลี่ยนไป นี่คือรูปแบบการเขียนโปรแกรมที่ใกล้เคียงกับงานประมวลผลรายงานในโลกธุรกิจจริงมาก
ที่สุดในบรรดาตัวอย่างทั้งหมดของ Part นี้

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP160.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MONTH-COUNT      PIC 9(02) VALUE 6.
       01  WS-SALES-TABLE.
           05  WS-MONTH-REC    OCCURS 1 TO 12 TIMES
                               DEPENDING ON WS-MONTH-COUNT.
               10  WS-MONTH-NAME   PIC X(09).
               10  WS-MONTH-AMT    PIC 9(06).

       01  WS-IDX              PIC 9(02).
       01  WS-TOTAL            PIC 9(07) VALUE ZERO.
       01  WS-AVERAGE          PIC 9(06)V99 VALUE ZERO.
       01  WS-BEST-IDX         PIC 9(02) VALUE 1.
       01  WS-BEST-AMT         PIC 9(06) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Mini project: table-driven sales report ===".

           MOVE "January"  TO WS-MONTH-NAME(1).
           MOVE 12000      TO WS-MONTH-AMT(1).
           MOVE "February" TO WS-MONTH-NAME(2).
           MOVE 15500      TO WS-MONTH-AMT(2).
           MOVE "March"    TO WS-MONTH-NAME(3).
           MOVE 9800       TO WS-MONTH-AMT(3).
           MOVE "April"    TO WS-MONTH-NAME(4).
           MOVE 21000      TO WS-MONTH-AMT(4).
           MOVE "May"      TO WS-MONTH-NAME(5).
           MOVE 17600      TO WS-MONTH-AMT(5).
           MOVE "June"     TO WS-MONTH-NAME(6).
           MOVE 13200      TO WS-MONTH-AMT(6).

           DISPLAY "Report for " WS-MONTH-COUNT " month(s):".
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-MONTH-COUNT
               DISPLAY "  " WS-MONTH-NAME(WS-IDX) ": "
                       WS-MONTH-AMT(WS-IDX)
               ADD WS-MONTH-AMT(WS-IDX) TO WS-TOTAL
               IF WS-MONTH-AMT(WS-IDX) > WS-BEST-AMT
                   MOVE WS-MONTH-AMT(WS-IDX) TO WS-BEST-AMT
                   MOVE WS-IDX TO WS-BEST-IDX
               END-IF
           END-PERFORM.

           COMPUTE WS-AVERAGE = WS-TOTAL / WS-MONTH-COUNT.

           DISPLAY "-------------------------------".
           DISPLAY "Total sales   : " WS-TOTAL.
           DISPLAY "Average sales : " WS-AVERAGE.
           DISPLAY "Best month    : "
                   WS-MONTH-NAME(WS-BEST-IDX)
                   " (" WS-BEST-AMT ")".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Mini project: table-driven sales report ===
Report for 06 month(s):
  January  : 012000
  February : 015500
  March    : 009800
  April    : 021000
  May      : 017600
  June     : 013200
-------------------------------
Total sales   : 0089100
Average sales : 014850.00
Best month    : April     (021000)
```

### อธิบายโค้ดทีละส่วน

- `01 WS-MONTH-COUNT PIC 9(02) VALUE 6.` ประกาศก่อนตาราง `WS-SALES-TABLE` ตามกฎของ ODO และมีค่าเริ่มต้น
  เป็น 6 (ประมวลผลแค่ครึ่งปีแรกในตัวอย่างนี้) — ถ้าต้องการรายงานทั้งปี เพียงแค่เปลี่ยนค่าเริ่มต้นและ
  เพิ่มข้อมูลอีก 6 เดือน โดยไม่ต้องแก้โครงสร้างลูปหรือการคำนวณใด ๆ เลย
- `05 WS-MONTH-REC OCCURS 1 TO 12 TIMES DEPENDING ON WS-MONTH-COUNT.` คือ Group Item ที่รวม ODO
  (ขนาดยืดหยุ่นได้) เข้ากับตารางของระเบียนข้อมูล (มีทั้งชื่อเดือนและยอดขาย) ในคำประกาศเดียว — ผสาน
  เทคนิคจากขั้นตอนที่ 155 และ 157 เข้าด้วยกัน
- `COMPUTE WS-AVERAGE = WS-TOTAL / WS-MONTH-COUNT.` ใช้ `WS-MONTH-COUNT` (ตัวแปร) แทนเลขคงที่ เพื่อให้
  สูตรคำนวณค่าเฉลี่ยถูกต้องเสมอไม่ว่าจำนวนเดือนจะเป็นเท่าไหร่ — นี่คือแนวทางที่ควรใช้ในโค้ดจริงเสมอ
  (ตอบข้อควรระวังที่ทิ้งไว้จากขั้นตอนที่ 156)
- `WS-MONTH-NAME(WS-BEST-IDX)` ในบรรทัดสุดท้ายแสดงให้เห็นพลังของการเก็บ**ตำแหน่ง**ของค่าที่ดีที่สุดไว้
  (`WS-BEST-IDX`) แทนที่จะเก็บแค่ค่าตัวเลข — ทำให้สามารถย้อนกลับไปดึงข้อมูลฟิลด์อื่นของระเบียนเดียวกัน
  (ในที่นี้คือชื่อเดือน) มาแสดงในรายงานได้อย่างสมบูรณ์

### ข้อควรระวัง

- โปรแกรมนี้เป็นตัวอย่างที่ผสานหลายเทคนิคเข้าด้วยกัน (Group Item + OCCURS + ODO + Accumulate pattern)
  ซึ่งเป็นรูปแบบที่**พบบ่อยมากในระบบธุรกิจจริง** ควรฝึกอ่านโค้ดแบบนี้ให้คล่องเพราะจะเจอรูปแบบใกล้เคียงกัน
  นี้อีกหลายครั้งตลอดหลักสูตร โดยเฉพาะในโปรเจกต์ระบบจัดการสินค้าคงคลังที่ Part 035
- ยังไม่ได้ใส่การตรวจสอบขอบเขต (bound check) ก่อนเข้าถึง `WS-MONTH-NAME(WS-IDX)`/`WS-MONTH-AMT(WS-IDX)`
  ในลูป — ในโค้ดระดับ production จริง ควรเพิ่มการตรวจสอบว่า `WS-MONTH-COUNT` ไม่เกิน 12 (ค่าสูงสุดที่
  `OCCURS` กำหนดไว้) ก่อนเริ่มประมวลผลเสมอ เพื่อป้องกันปัญหาแบบที่สาธิตไว้ในขั้นตอนที่ 159

### แบบฝึกหัดที่ 160.1

**โจทย์**: จงขยายรายงานให้ครบทั้งปี (12 เดือน) โดยเพิ่มข้อมูลเดือนกรกฎาคมถึงธันวาคม (ตัวเลขสมมติเอง
ได้ตามต้องการ) แล้วอัปเดต `WS-MONTH-COUNT`

**เฉลยแนวทาง**: เพิ่ม `MOVE` อีก 6 คู่สำหรับเดือน July ถึง December และเปลี่ยน
`01 WS-MONTH-COUNT PIC 9(02) VALUE 6.` เป็น `VALUE 12.`:

```cobol
       01  WS-MONTH-COUNT      PIC 9(02) VALUE 12.
       ...
           MOVE "July"      TO WS-MONTH-NAME(7).
           MOVE 19500       TO WS-MONTH-AMT(7).
           MOVE "August"    TO WS-MONTH-NAME(8).
           MOVE 18200       TO WS-MONTH-AMT(8).
           MOVE "September" TO WS-MONTH-NAME(9).
           MOVE 16700       TO WS-MONTH-AMT(9).
           MOVE "October"   TO WS-MONTH-NAME(10).
           MOVE 20300       TO WS-MONTH-AMT(10).
           MOVE "November"  TO WS-MONTH-NAME(11).
           MOVE 22800       TO WS-MONTH-AMT(11).
           MOVE "December"  TO WS-MONTH-NAME(12).
           MOVE 25000       TO WS-MONTH-AMT(12).
```

ส่วนลูปประมวลผล การคำนวณผลรวม/ค่าเฉลี่ย/ค่าสูงสุดที่เหลือทั้งหมด **ไม่ต้องแก้ไขแม้แต่บรรทัดเดียว**
เพราะทุกอย่างอ้างอิงผ่าน `WS-MONTH-COUNT` ทั้งหมด — นี่คือข้อพิสูจน์ที่ชัดเจนที่สุดว่าทำไม
`OCCURS ... DEPENDING ON` ถึงมีประโยชน์มากเมื่อออกแบบโปรแกรมที่ต้องรองรับขนาดข้อมูลที่เปลี่ยนแปลงได้

---

## สรุปท้ายบท

ใน Part นี้ คุณได้เรียนรู้:

- เหตุผลที่ต้องมีตาราง (`OCCURS`) แทนการประกาศตัวแปรแยกกันหลายตัว และประโยชน์ของการที่โค้ดมีขนาดคงที่
  ไม่ว่าข้อมูลจะมีกี่รายการ
- การประกาศตาราง `OCCURS` พื้นฐานกับชนิดข้อมูลต่าง ๆ (ตัวเลข ตัวเลขทศนิยม ตัวอักษร ตัวอักษรเดี่ยว)
- การเข้าถึงสมาชิกตารางด้วย Subscript ทั้ง 3 รูปแบบ: ค่าคงที่ ตัวแปร และนิพจน์
- การกำหนดค่าเริ่มต้นให้ทุกช่องของตารางด้วย `VALUE` และการรีเซ็ตค่ากลับด้วย `INITIALIZE`
- การประกาศ `OCCURS` บน Group Item เพื่อสร้างตารางของระเบียนข้อมูลที่มีหลายฟิลด์ต่อช่อง
- การคำนวณสถิติพื้นฐานของตาราง (ผลรวม ค่าเฉลี่ย ค่าสูงสุด-ต่ำสุด) ด้วยการวนลูปเพียงรอบเดียว
- `OCCURS ... DEPENDING ON` สำหรับสร้างตารางที่มีขนาดยืดหยุ่นได้ตามข้อมูลจริง
- การคัดลอกตารางทั้งชุดด้วย `MOVE` และการเปรียบเทียบตารางทีละช่องด้วยลูป
- ข้อผิดพลาดที่อันตรายที่สุดของตาราง คือการเข้าถึง Subscript เกินขอบเขต ซึ่งไม่ error แต่ให้ผลลัพธ์ที่
  คาดเดาไม่ได้ (undefined behavior) พร้อมสาธิตให้เห็นจริงด้วยการคอมไพล์และรัน
- มินิโปรเจกต์ที่ผสานทุกเทคนิคเข้าด้วยกันเป็นรายงานยอดขายรายเดือนแบบยืดหยุ่นขนาดได้

ทักษะเรื่องตารางที่คุณเพิ่งฝึกมาทั้งหมดนี้เป็นพื้นฐานสำคัญที่สุดอย่างหนึ่งของ COBOL ระดับกลาง Part ถัดไป
จะพาคุณไปรู้จักกับ **Index** ที่ประกาศด้วย `INDEXED BY` ซึ่งมีประสิทธิภาพดีกว่า Subscript ธรรมดา และ
คำสั่ง `SEARCH`/`SEARCH ALL` ที่ COBOL ออกแบบมาเฉพาะสำหรับการค้นหาข้อมูลในตารางอย่างรวดเร็ว ก่อนที่จะ
ขยายไปสู่ตารางหลายมิติ (Multi-dimensional Tables) ใน Part 018 ซึ่งจะนำทุกอย่างที่เรียนใน Part นี้ไปต่อ
ยอดสร้างโครงสร้างข้อมูลที่ซับซ้อนยิ่งขึ้นสำหรับงานประมวลผลข้อมูลระดับองค์กรจริง

**[← กลับไป Part 015: โปรเจกต์เฟส 1 — เครื่องคิดเลขและระบบคำนวณเกรด](part-015-phase1-project.md)** | **[ไปยัง Part 017: Subscript กับ Index และ SEARCH / SEARCH ALL →](part-017-subscript-search.md)**
