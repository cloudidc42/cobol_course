# Part 018: ตารางหลายมิติ (Multi-dimensional Tables) (ขั้นตอนที่ 171–180)

## คำนำของ Part นี้

ใน Part 016 เราได้เรียนรู้การประกาศตารางด้วย `OCCURS` และการเข้าถึงสมาชิกด้วย Subscript
ส่วนใน Part 017 เราได้เรียนรู้การใช้ `INDEXED BY` และคำสั่ง `SEARCH`/`SEARCH ALL` เพื่อค้นหาข้อมูลในตาราง
อย่างมีประสิทธิภาพ ตารางที่เราเรียนมาทั้งหมดนั้นเป็น **ตารางมิติเดียว (One-dimensional Table)**
คือมีเพียงแกนเดียว เช่น ยอดขายรายเดือน 12 เดือน หรือคะแนนสอบของนักเรียน 30 คน

แต่ในโลกธุรกิจจริง ข้อมูลมักมีความสัมพันธ์หลายมิติพร้อมกัน เช่น "ยอดขายของแต่ละ**ภาค**ในแต่ละ**ไตรมาส**"
ซึ่งมีสองแกน (ภาค x ไตรมาส) หรือ "จำนวนสินค้าคงคลังของแต่ละ**คลังสินค้า** แต่ละ**สินค้า** ในแต่ละ**เดือน**"
ซึ่งมีสามแกน (คลัง x สินค้า x เดือน) Part นี้จะพาคุณไปรู้จักกับ **ตารางหลายมิติ (Multi-dimensional Tables)**
ตั้งแต่การประกาศด้วย `OCCURS` ซ้อนกัน การเติมข้อมูลด้วย `PERFORM VARYING` ซ้อนกันหลายชั้น
ไปจนถึงการค้นหาและคำนวณผลรวมในตารางหลายมิติ ซึ่งเป็นทักษะสำคัญมากสำหรับการเขียนรายงานทางธุรกิจ
และระบบประมวลผลข้อมูลขนาดใหญ่ที่เราจะเจอตลอดหลักสูตรนี้

---

## ขั้นตอนที่ 171: ทบทวนตารางมิติเดียว และข้อจำกัดที่นำไปสู่ตารางหลายมิติ

### ทบทวนสั้น ๆ

ตารางมิติเดียวที่เราเรียนใน Part 016-017 ใช้ `OCCURS` หนึ่งชั้น เช่น ยอดขาย 4 ไตรมาสของภาคเดียว
สามารถเข้าถึงแต่ละสมาชิกได้ด้วย subscript หรือ index เดียว เช่น `SALES-AMT(2)`

### ปัญหาที่เกิดขึ้นเมื่อข้อมูลมีมากกว่าหนึ่งมิติ

ลองนึกภาพว่าบริษัทมี 3 ภาค (North, Central, South) และแต่ละภาคมียอดขาย 4 ไตรมาส
ถ้าใช้ตารางมิติเดียว เราจะต้องสร้างตัวแปรแยกกันสามตัว (`NORTH-SALES`, `CENTRAL-SALES`, `SOUTH-SALES`)
ซึ่งทำให้เขียนโค้ดซ้ำซ้อนมาก และยิ่งแย่ลงไปอีกถ้ามี 50 ภาค! นี่คือเหตุผลที่ COBOL รองรับการซ้อน `OCCURS`
หลายชั้นเพื่อสร้าง **ตารางของตาราง** (a table of tables) ซึ่งเราเรียกว่าตารางหลายมิติ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s1.cob
      *> Purpose : Recap one-dimensional OCCURS and show its limit
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ONE-DIM-RECAP.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  MONTHLY-SALES.
           05  SALES-AMT       PIC 9(6) OCCURS 4 TIMES.

       01  WS-SUB              PIC 9  VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE 100000 TO SALES-AMT(1)
           MOVE 120000 TO SALES-AMT(2)
           MOVE 135000 TO SALES-AMT(3)
           MOVE 141000 TO SALES-AMT(4)

           DISPLAY "One region, 4 quarters of sales:"
           PERFORM VARYING WS-SUB FROM 1 BY 1 UNTIL WS-SUB > 4
               DISPLAY "  Quarter " WS-SUB ": " SALES-AMT(WS-SUB)
           END-PERFORM

           DISPLAY " "
           DISPLAY "Problem: 3 regions, each with 4 quarters?"
           DISPLAY "A single one-dimensional table cannot express"
           DISPLAY "that relationship. We need a table of tables:"
           DISPLAY "a two-dimensional (multi-dimensional) table."

           STOP RUN.
```

### คำอธิบายโค้ด

- `SALES-AMT PIC 9(6) OCCURS 4 TIMES` คือตารางมิติเดียวมาตรฐานที่เราคุ้นเคยแล้ว
- โปรแกรมนี้แสดงให้เห็นว่าตารางแบบนี้จัดการได้แค่ "แกนเดียว" (ไตรมาส) แต่ไม่มีที่ทางสำหรับ "แกนที่สอง" (ภาค)

### ผลลัพธ์ที่ได้จากการรันจริง

```
One region, 4 quarters of sales:
  Quarter 1: 100000
  Quarter 2: 120000
  Quarter 3: 135000
  Quarter 4: 141000
 
Problem: 3 regions, each with 4 quarters?
A single one-dimensional table cannot express
that relationship. We need a table of tables:
a two-dimensional (multi-dimensional) table.
```

### ข้อควรระวัง

- อย่าพยายามแก้ปัญหาหลายมิติด้วยการสร้างตัวแปรแยกกันหลายตัว (เช่น `NORTH-SALES`, `CENTRAL-SALES`)
  เพราะจะทำให้โค้ดยาวและบำรุงรักษายากขึ้นแบบทวีคูณเมื่อจำนวนภาคเพิ่มขึ้น
- ควรมองหาสัญญาณว่าข้อมูลของคุณ "มีสองแกนขึ้นไป" ตั้งแต่ตอนออกแบบ Data Division ไม่ใช่ตอนเขียน Logic แล้ว

### แบบฝึกหัดที่ 171.1

**โจทย์**: จงยกตัวอย่างข้อมูลทางธุรกิจอีก 2 กรณีที่มีลักษณะ "สองมิติ" ขึ้นไป
(นอกเหนือจากยอดขายภาค x ไตรมาสที่ยกตัวอย่างไปแล้ว)

**เฉลยแนวทาง**: (1) คะแนนสอบของนักเรียนแต่ละคนในแต่ละวิชา (นักเรียน x วิชา)
(2) จำนวนที่นั่งว่างของเที่ยวบินแต่ละเที่ยว ในแต่ละวันของสัปดาห์ (เที่ยวบิน x วัน)
ทั้งสองกรณีนี้ต้องใช้ตารางสองมิติจึงจะแทนข้อมูลได้ครบถ้วนและเป็นระเบียบ

---

## ขั้นตอนที่ 172: ประกาศตารางสองมิติด้วย OCCURS ซ้อนกัน

### แนวคิด

การสร้างตารางสองมิติใน COBOL ทำได้โดยการซ้อน `OCCURS` สองชั้น: ชั้นนอกสุดคือ "แถว" (เช่น ภาค)
และชั้นในคือ "คอลัมน์" (เช่น ไตรมาส) การเข้าถึงข้อมูลต้องระบุ subscript ทั้งสองแกน โดยคั่นด้วยจุลภาค (comma)
เรียงจาก **แกนนอกสุดไปแกนในสุด** เช่น `QTR-SALES(1, 1)` หมายถึงภาคที่ 1 ไตรมาสที่ 1

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s2.cob
      *> Purpose : Declare a two-dimensional table with nested OCCURS
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TWO-DIM-DECLARE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  SALES-TABLE.
           05  REGION-ROW OCCURS 3 TIMES.
               10  QTR-SALES  PIC 9(6) OCCURS 4 TIMES.

       01  WS-REGION            PIC 9  VALUE 1.
       01  WS-QTR                PIC 9  VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> REGION-ROW(1) is Region 1's row of 4 quarters.
      *> QTR-SALES(1, 1) is Region 1, Quarter 1.
           MOVE 100000 TO QTR-SALES(1, 1)
           MOVE 110000 TO QTR-SALES(1, 2)
           MOVE 120000 TO QTR-SALES(1, 3)
           MOVE 130000 TO QTR-SALES(1, 4)

           MOVE 200000 TO QTR-SALES(2, 1)
           MOVE 210000 TO QTR-SALES(2, 2)
           MOVE 220000 TO QTR-SALES(2, 3)
           MOVE 230000 TO QTR-SALES(2, 4)

           MOVE 300000 TO QTR-SALES(3, 1)
           MOVE 305000 TO QTR-SALES(3, 2)
           MOVE 310000 TO QTR-SALES(3, 3)
           MOVE 315000 TO QTR-SALES(3, 4)

           DISPLAY "Two-dimensional sales table (Region x Quarter):"
           PERFORM VARYING WS-REGION FROM 1 BY 1 UNTIL WS-REGION > 3
               PERFORM VARYING WS-QTR FROM 1 BY 1 UNTIL WS-QTR > 4
                   DISPLAY "  Region " WS-REGION
                       " Quarter " WS-QTR
                       " = " QTR-SALES(WS-REGION, WS-QTR)
               END-PERFORM
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- `05 REGION-ROW OCCURS 3 TIMES` คือแกนที่ 1 (มี 3 แถว หมายถึง 3 ภาค)
- `10 QTR-SALES PIC 9(6) OCCURS 4 TIMES` ซ้อนอยู่ **ภายใน** `REGION-ROW` คือแกนที่ 2 (มี 4 ค่าต่อแถว)
- โครงสร้างนี้จองพื้นที่หน่วยความจำทั้งหมด 3 x 4 x 6 ไบต์ = 72 ไบต์ ต่อเนื่องกัน
- การอ้างอิงสมาชิกต้องระบุ subscript ทั้งสองตัวเสมอ: `QTR-SALES(ภาค, ไตรมาส)`

### ผลลัพธ์ที่ได้จากการรันจริง

```
Two-dimensional sales table (Region x Quarter):
  Region 1 Quarter 1 = 100000
  Region 1 Quarter 2 = 110000
  Region 1 Quarter 3 = 120000
  Region 1 Quarter 4 = 130000
  Region 2 Quarter 1 = 200000
  Region 2 Quarter 2 = 210000
  Region 2 Quarter 3 = 220000
  Region 2 Quarter 4 = 230000
  Region 3 Quarter 1 = 300000
  Region 3 Quarter 2 = 305000
  Region 3 Quarter 3 = 310000
  Region 3 Quarter 4 = 315000
```

### ข้อควรระวัง

- ลำดับของ subscript สำคัญมาก: `QTR-SALES(1, 2)` กับ `QTR-SALES(2, 1)` เป็นคนละตัวกัน
  การสลับลำดับโดยไม่ตั้งใจเป็นบั๊กที่พบบ่อยที่สุดของตารางหลายมิติ
- ระดับ (level number) ของ `REGION-ROW` และ `QTR-SALES` ต้องมีความสัมพันธ์แบบ "แม่-ลูก" ที่ถูกต้อง
  (05 คลุม 10 คลุม 15 ฯลฯ) มิฉะนั้น compiler จะตีความโครงสร้างผิด

### แบบฝึกหัดที่ 172.1

**โจทย์**: จากโค้ดตัวอย่างข้างต้น จงเขียนคำสั่งที่แสดงเฉพาะยอดขายของ "ภาค 2 ไตรมาส 3" อย่างเดียว

**เฉลย**:
```cobol
           DISPLAY "Region 2 Quarter 3 = " QTR-SALES(2, 3)
```
ผลลัพธ์ที่ได้คือ `Region 2 Quarter 3 = 220000`

---

## ขั้นตอนที่ 173: การเติมข้อมูลตารางสองมิติด้วย PERFORM VARYING ซ้อนกัน

### แนวคิด

การ `MOVE` ค่าทีละตัวแบบขั้นตอนที่แล้วใช้ได้กับข้อมูลจำนวนน้อย แต่ถ้าตารางมีขนาดใหญ่ (เช่น 100x100)
เราต้องใช้ **ลูปซ้อนกัน (Nested Loop)** ด้วย `PERFORM VARYING ... AFTER` หรือ `PERFORM VARYING` สองอันซ้อนกัน
เพื่อเดินผ่านทุกช่องของตารางอย่างเป็นระบบ หลักการคือ: ลูปนอกควบคุมแกนแรก (แถว) ลูปในควบคุมแกนที่สอง (คอลัมน์)

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s3.cob
      *> Purpose : Fill a 2D table using nested PERFORM VARYING
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TWO-DIM-FILL.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  MULT-TABLE.
           05  MT-ROW OCCURS 5 TIMES.
               10  MT-CELL PIC 9(3) OCCURS 5 TIMES.

       01  WS-I                 PIC 9  VALUE 1.
       01  WS-J                 PIC 9  VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Build a 5x5 multiplication table using nested loops.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 5
               PERFORM VARYING WS-J FROM 1 BY 1 UNTIL WS-J > 5
                   COMPUTE MT-CELL(WS-I, WS-J) = WS-I * WS-J
               END-PERFORM
           END-PERFORM

           DISPLAY "5x5 multiplication table:"
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 5
               PERFORM VARYING WS-J FROM 1 BY 1 UNTIL WS-J > 5
                   DISPLAY MT-CELL(WS-I, WS-J) " " WITH NO ADVANCING
               END-PERFORM
               DISPLAY " "
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- ลูปแรก (`WS-I`) วนจาก 1 ถึง 5 คือแกนแถว ลูปที่สอง (`WS-J`) ซ้อนอยู่ข้างในวนจาก 1 ถึง 5 คือแกนคอลัมน์
  รวมแล้วมีการวนทั้งหมด 5 x 5 = 25 รอบ เพื่อเติมค่าให้ครบทุกช่อง
- `WITH NO ADVANCING` ทำให้ `DISPLAY` ไม่ขึ้นบรรทัดใหม่ เราจึงพิมพ์ตัวเลขเรียงกันในแนวนอนได้
  ก่อนจะสั่ง `DISPLAY " "` เปล่า ๆ เพื่อขึ้นบรรทัดใหม่หลังจบแต่ละแถว

### ผลลัพธ์ที่ได้จากการรันจริง

```
5x5 multiplication table:
001 002 003 004 005  
002 004 006 008 010  
003 006 009 012 015  
004 008 012 016 020  
005 010 015 020 025  
```

### ข้อควรระวัง

- ต้องระวังการสลับลำดับลูปนอก-ลูปใน ให้ตรงกับลำดับ subscript ที่ประกาศไว้ใน `DATA DIVISION`
  ถ้าสลับกัน โปรแกรมจะยัง compile ผ่านแต่ผลลัพธ์จะผิดเพี้ยน (เติมค่าซ้ำหรือข้ามช่องบางช่อง)
- ระวัง `PIC 9` ของตัวแปรควบคุมลูป (`WS-I`, `WS-J`) ให้มีจำนวนหลักเพียงพอ ถ้าตารางมีขนาดเกิน 9 แถว/คอลัมน์
  แต่ประกาศเป็น `PIC 9` (1 หลัก) ค่าจะ wrap-around หรือ error ได้

### แบบฝึกหัดที่ 173.1

**โจทย์**: จงแก้โค้ดข้างต้นให้สร้างตารางสูตรคูณขนาด 3x3 แทนที่จะเป็น 5x5

**เฉลย**: เปลี่ยน `OCCURS 5 TIMES` ทั้งสองตำแหน่งเป็น `OCCURS 3 TIMES` และเปลี่ยนเงื่อนไข `UNTIL WS-I > 5`
กับ `UNTIL WS-J > 5` เป็น `UNTIL WS-I > 3` และ `UNTIL WS-J > 3` ตามลำดับ ผลลัพธ์ที่ได้จะเป็นตาราง 3x3
คือ `1 2 3 / 2 4 6 / 3 6 9`

---

## ขั้นตอนที่ 174: ตารางสามมิติ (Three-dimensional Tables)

### แนวคิด

หลักการเดียวกับตารางสองมิติสามารถขยายต่อไปเป็นสามมิติ (หรือมากกว่า) ได้โดยการซ้อน `OCCURS` เพิ่มอีกชั้น
ตัวอย่างคลาสสิกคือ "คลังสินค้า x สินค้า x เดือน" ซึ่งต้องใช้ subscript สามตัวในการอ้างอิงแต่ละช่อง

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s4.cob
      *> Purpose : Three-dimensional table (Warehouse x Product x Month)
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. THREE-DIM-TABLE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  STOCK-TABLE.
           05  WAREHOUSE-ROW OCCURS 2 TIMES.
               10  PRODUCT-ROW OCCURS 3 TIMES.
                   15  MONTH-QTY PIC 9(5) OCCURS 3 TIMES.

       01  WS-WH                PIC 9  VALUE 1.
       01  WS-PROD              PIC 9  VALUE 1.
       01  WS-MON               PIC 9  VALUE 1.
       01  WS-COUNTER           PIC 9(5) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Fill: quantity = warehouse*10000 + product*100 + month
           PERFORM VARYING WS-WH FROM 1 BY 1 UNTIL WS-WH > 2
               PERFORM VARYING WS-PROD FROM 1 BY 1 UNTIL WS-PROD > 3
                   PERFORM VARYING WS-MON FROM 1 BY 1 UNTIL WS-MON > 3
                       COMPUTE MONTH-QTY(WS-WH, WS-PROD, WS-MON) =
                           WS-WH * 10000 + WS-PROD * 100 + WS-MON
                   END-PERFORM
               END-PERFORM
           END-PERFORM

           DISPLAY "3D table: Warehouse x Product x Month"
           PERFORM VARYING WS-WH FROM 1 BY 1 UNTIL WS-WH > 2
               PERFORM VARYING WS-PROD FROM 1 BY 1 UNTIL WS-PROD > 3
                   PERFORM VARYING WS-MON FROM 1 BY 1 UNTIL WS-MON > 3
                       DISPLAY "  WH=" WS-WH
                           " PRODUCT=" WS-PROD
                           " MONTH=" WS-MON
                           " QTY=" MONTH-QTY(WS-WH, WS-PROD, WS-MON)
                   END-PERFORM
               END-PERFORM
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- โครงสร้างนี้มี `OCCURS` ซ้อนกัน 3 ชั้น (level 05, 10, 15) ทำให้ `MONTH-QTY` ต้องใช้ subscript 3 ตัว
  เรียงตามลำดับจากแกนนอกสุด (คลังสินค้า) ไปแกนในสุด (เดือน): `MONTH-QTY(คลัง, สินค้า, เดือน)`
- ขนาดรวมของตารางนี้คือ 2 x 3 x 3 = 18 ช่อง ต้องใช้ลูปซ้อนกัน 3 ชั้นในการเติมและแสดงข้อมูลให้ครบทุกช่อง

### ผลลัพธ์ที่ได้จากการรันจริง (แสดงบางส่วน)

```
3D table: Warehouse x Product x Month
  WH=1 PRODUCT=1 MONTH=1 QTY=10101
  WH=1 PRODUCT=1 MONTH=2 QTY=10102
  WH=1 PRODUCT=1 MONTH=3 QTY=10103
  WH=1 PRODUCT=2 MONTH=1 QTY=10201
  ...
  WH=2 PRODUCT=3 MONTH=3 QTY=20303
```

### ข้อควรระวัง

- ยิ่งมิติเพิ่มขึ้น จำนวนรอบการวนลูปจะเพิ่มแบบทวีคูณ (ตัวอย่างนี้คือ 2x3x3 = 18 รอบ)
  ถ้าตารางมีขนาดใหญ่ (เช่น 100x100x12) จำนวนรอบอาจสูงถึงหลักแสนหรือหลักล้าน ควรคำนึงถึงประสิทธิภาพเสมอ
- COBOL มาตรฐานอนุญาตให้ซ้อน `OCCURS` ได้สูงสุด 7 มิติ แต่ในทางปฏิบัติ การออกแบบเกิน 3 มิติ
  มักทำให้โค้ดอ่านยากมาก ควรพิจารณาการออกแบบใหม่ (เช่น แยกเป็นหลายตารางย่อย หรือใช้ไฟล์/ฐานข้อมูลแทน)

### แบบฝึกหัดที่ 174.1

**โจทย์**: จากตารางสามมิติข้างต้น จงเขียนคำสั่งแสดงค่าของ "คลังสินค้า 2, สินค้า 3, เดือน 1" อย่างเดียว

**เฉลย**:
```cobol
           DISPLAY "WH2-PROD3-MONTH1 = " MONTH-QTY(2, 3, 1)
```
ผลลัพธ์ที่ได้คือ `WH2-PROD3-MONTH1 = 20301`

---

## ขั้นตอนที่ 175: การใช้ INDEXED BY กับตารางหลายมิติ

### แนวคิด

จาก Part 017 เราเรียนรู้ว่า `INDEXED BY` ทำให้การเข้าถึงตารางเร็วขึ้น (เพราะ index ถูกเก็บเป็น
ตำแหน่งหน่วยความจำโดยตรง ไม่ต้องคำนวณ offset ใหม่ทุกครั้งเหมือน subscript ธรรมดา) หลักการเดียวกันนี้ใช้ได้กับ
ตารางหลายมิติ โดยประกาศ `INDEXED BY` ในทุกชั้นของ `OCCURS` ที่ต้องการสร้าง index ให้มัน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s5.cob
      *> Purpose : Using INDEXED BY with multi-dimensional tables
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TWO-DIM-INDEXED.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  SCORE-TABLE.
           05  STUDENT-ROW OCCURS 3 TIMES INDEXED BY STU-IDX.
               10  SUBJECT-SCORE PIC 9(3)
                   OCCURS 4 TIMES INDEXED BY SUB-IDX.

       01  WS-TOTAL             PIC 9(4) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           SET STU-IDX SUB-IDX TO 1

      *> Fill scores: student index * 20 + subject index * 5
           PERFORM VARYING STU-IDX FROM 1 BY 1 UNTIL STU-IDX > 3
               PERFORM VARYING SUB-IDX FROM 1 BY 1 UNTIL SUB-IDX > 4
                   COMPUTE SUBJECT-SCORE(STU-IDX, SUB-IDX) =
                       (STU-IDX * 20) + (SUB-IDX * 5)
               END-PERFORM
           END-PERFORM

           DISPLAY "Scores addressed through INDEXED BY:"
           PERFORM VARYING STU-IDX FROM 1 BY 1 UNTIL STU-IDX > 3
               MOVE 0 TO WS-TOTAL
               PERFORM VARYING SUB-IDX FROM 1 BY 1 UNTIL SUB-IDX > 4
                   DISPLAY "  Student " STU-IDX
                       " Subject " SUB-IDX
                       " Score " SUBJECT-SCORE(STU-IDX, SUB-IDX)
                   ADD SUBJECT-SCORE(STU-IDX, SUB-IDX) TO WS-TOTAL
               END-PERFORM
               DISPLAY "  --> Student " STU-IDX " total = " WS-TOTAL
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- `STUDENT-ROW OCCURS 3 TIMES INDEXED BY STU-IDX` สร้าง index ชื่อ `STU-IDX` สำหรับแกนที่ 1 (นักเรียน)
- `SUBJECT-SCORE ... OCCURS 4 TIMES INDEXED BY SUB-IDX` สร้าง index ชื่อ `SUB-IDX` สำหรับแกนที่ 2 (วิชา)
- `SET STU-IDX SUB-IDX TO 1` คือการตั้งค่าเริ่มต้นให้ index ทั้งสองตัวพร้อมกันในคำสั่งเดียว
- สังเกตว่า `PERFORM VARYING` สามารถใช้ตัวแปร index (`STU-IDX`, `SUB-IDX`) แทน subscript ธรรมดาได้เลย

### ผลลัพธ์ที่ได้จากการรันจริง (แสดงบางส่วน)

```
Scores addressed through INDEXED BY:
  Student +000000001 Subject +000000001 Score 025
  Student +000000001 Subject +000000002 Score 030
  ...
  --> Student +000000001 total = 0130
```

### ข้อควรระวัง

- **จุดที่มือใหม่งงบ่อยที่สุด**: เมื่อ `DISPLAY` ตัวแปร index (เช่น `STU-IDX`) โดยตรง จะเห็นค่าประหลาด
  เช่น `+000000001` แทนที่จะเป็น `1` ธรรมดา เพราะ index เป็นชนิดข้อมูลไบนารีภายใน (USAGE INDEX)
  ไม่ใช่ `PIC 9` ปกติ ถ้าต้องการแสดงผลตัวเลขให้สวยงาม ควร `MOVE` ค่า index ไปเก็บใน `PIC 9` ก่อนแสดงผล
- index ที่ถูกประกาศไว้ในระดับหนึ่งของ `OCCURS` จะผูกติดกับแกนนั้นเท่านั้น ไม่สามารถใช้ `STU-IDX`
  ไปควบคุมแกนวิชาได้ (และในทางกลับกัน)

### แบบฝึกหัดที่ 175.1

**โจทย์**: จงแก้โค้ดข้างต้นเพื่อแสดงค่า `STU-IDX` ให้เป็นตัวเลขปกติ (ไม่มีเครื่องหมาย `+` และเลขศูนย์นำหน้า)

**เฉลย**: ประกาศตัวแปรใหม่ `01 WS-DISPLAY-IDX PIC 9.` แล้วก่อน `DISPLAY` ให้ `MOVE STU-IDX TO WS-DISPLAY-IDX`
จากนั้นใช้ `WS-DISPLAY-IDX` แทน `STU-IDX` ในคำสั่ง `DISPLAY` จะได้ผลลัพธ์เป็น `1`, `2`, `3` ตามปกติ

---

## ขั้นตอนที่ 176: ตารางของกลุ่มข้อมูล (Table of Group Items)

### แนวคิด

ตารางหลายมิติไม่จำเป็นต้องเป็นตัวเลขล้วน ๆ เสมอไป เราสามารถสร้าง **ตารางของกลุ่มข้อมูล (group item)**
ที่แต่ละแถวมีทั้งฟิลด์ข้อความ (เช่น ชื่อพนักงาน) และตารางย่อยซ้อนอยู่ข้างในอีกที (เช่น ชั่วโมงทำงานรายเดือน)
รูปแบบนี้พบบ่อยมากในระบบธุรกิจจริง เพราะข้อมูลแต่ละ "รายการหลัก" (master record) มักมีทั้งข้อมูลเดี่ยว
และข้อมูลที่ซ้ำกันเป็นชุด (repeating group) ปนกันอยู่

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s6.cob
      *> Purpose : Table of group items, with a nested table inside
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TABLE-OF-GROUPS.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  EMPLOYEE-TABLE.
           05  EMPLOYEE-ROW OCCURS 3 TIMES INDEXED BY EMP-IDX.
               10  EMP-NAME        PIC X(15).
               10  EMP-DEPT        PIC X(10).
               10  EMP-MONTH-HOURS PIC 9(3)
                   OCCURS 3 TIMES INDEXED BY MON-IDX.

       01  WS-TOTAL-HOURS       PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           SET EMP-IDX TO 1
           MOVE "JOHN SMITH"    TO EMP-NAME(EMP-IDX)
           MOVE "SALES"         TO EMP-DEPT(EMP-IDX)
           MOVE 160 TO EMP-MONTH-HOURS(EMP-IDX, 1)
           MOVE 168 TO EMP-MONTH-HOURS(EMP-IDX, 2)
           MOVE 172 TO EMP-MONTH-HOURS(EMP-IDX, 3)

           SET EMP-IDX TO 2
           MOVE "MARY JONES"    TO EMP-NAME(EMP-IDX)
           MOVE "IT"            TO EMP-DEPT(EMP-IDX)
           MOVE 176 TO EMP-MONTH-HOURS(EMP-IDX, 1)
           MOVE 160 TO EMP-MONTH-HOURS(EMP-IDX, 2)
           MOVE 164 TO EMP-MONTH-HOURS(EMP-IDX, 3)

           SET EMP-IDX TO 3
           MOVE "DAVID LEE"     TO EMP-NAME(EMP-IDX)
           MOVE "FINANCE"       TO EMP-DEPT(EMP-IDX)
           MOVE 180 TO EMP-MONTH-HOURS(EMP-IDX, 1)
           MOVE 175 TO EMP-MONTH-HOURS(EMP-IDX, 2)
           MOVE 168 TO EMP-MONTH-HOURS(EMP-IDX, 3)

           DISPLAY "Employee monthly-hours report:"
           PERFORM VARYING EMP-IDX FROM 1 BY 1 UNTIL EMP-IDX > 3
               MOVE 0 TO WS-TOTAL-HOURS
               PERFORM VARYING MON-IDX FROM 1 BY 1 UNTIL MON-IDX > 3
                   ADD EMP-MONTH-HOURS(EMP-IDX, MON-IDX)
                       TO WS-TOTAL-HOURS
               END-PERFORM
               DISPLAY "  " EMP-NAME(EMP-IDX)
                   " (" EMP-DEPT(EMP-IDX) ") total hours = "
                   WS-TOTAL-HOURS
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- `EMPLOYEE-ROW OCCURS 3 TIMES` คือตารางหลัก (พนักงาน 3 คน) แต่ละแถวมีฟิลด์เดี่ยว `EMP-NAME`, `EMP-DEPT`
  และตารางย่อยซ้อนอยู่ข้างใน `EMP-MONTH-HOURS OCCURS 3 TIMES`
- การอ้างอิง `EMP-NAME(EMP-IDX)` ใช้ subscript ตัวเดียว (เพราะเป็นฟิลด์เดี่ยวในแต่ละแถว)
  แต่ `EMP-MONTH-HOURS(EMP-IDX, MON-IDX)` ต้องใช้ subscript สองตัว (เพราะมันเป็นตารางซ้อนในตาราง)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Employee monthly-hours report:
  JOHN SMITH      (SALES     ) total hours = 0500
  MARY JONES      (IT        ) total hours = 0500
  DAVID LEE       (FINANCE   ) total hours = 0523
```

### ข้อควรระวัง

- อย่าลืมว่าฟิลด์ที่อยู่ "ระดับเดียวกับ" ตารางย่อย (เช่น `EMP-NAME`, `EMP-DEPT`) ต้องการ subscript
  เท่ากับจำนวนแกนของตารางที่มันสังกัดเท่านั้น (1 ตัว) ในขณะที่ตารางย่อยเองต้องการ subscript รวมของทุกแกน (2 ตัว)
  การนับจำนวน subscript ผิดเป็นสาเหตุของ compile error ที่พบบ่อยมาก

### แบบฝึกหัดที่ 176.1

**โจทย์**: จงเพิ่มการแสดงผลชั่วโมงทำงานรายเดือนของพนักงานแต่ละคน (ไม่ใช่แค่ยอดรวม)

**เฉลย**: เพิ่มบรรทัด `DISPLAY "    Month " MON-IDX ": " EMP-MONTH-HOURS(EMP-IDX, MON-IDX)`
ไว้ข้างในลูป `PERFORM VARYING MON-IDX` ก่อนหรือหลังคำสั่ง `ADD` ก็ได้ จะทำให้เห็นรายละเอียดชั่วโมงแต่ละเดือน
ของพนักงานแต่ละคนเพิ่มเติมจากยอดรวม

---

## ขั้นตอนที่ 177: การกำหนดค่าเริ่มต้นให้ตารางด้วย VALUE และ INITIALIZE

### แนวคิด

เราสามารถกำหนดค่าเริ่มต้น (`VALUE`) ให้กับฟิลด์ที่อยู่ใต้ `OCCURS` ได้ ซึ่งจะทำให้ **ทุกสมาชิกของตาราง**
เริ่มต้นด้วยค่าเดียวกันตั้งแต่โปรแกรมเริ่มทำงาน (มักใช้ `VALUE ZERO` หรือ `VALUE SPACES`)
และเมื่อต้องการ "ล้างค่า" ตารางกลับไปเป็นค่าเริ่มต้นในระหว่างการทำงานของโปรแกรม (ไม่ใช่แค่ตอนเริ่มโปรแกรม)
เราใช้คำสั่ง `INITIALIZE` ซึ่งรีเซ็ตทุกช่องของตาราง (ไม่ว่าจะกี่มิติ) กลับไปเป็นค่าเริ่มต้นในคำสั่งเดียว

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s7.cob
      *> Purpose : Initial VALUE on OCCURS items and the
      *>           INITIALIZE statement for resetting a table
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TABLE-INIT-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  COUNTER-TABLE.
           05  COUNTER-ROW OCCURS 3 TIMES.
               10  COUNTER-CELL PIC 9(3) OCCURS 4 TIMES VALUE ZERO.

       01  WS-I                 PIC 9 VALUE 1.
       01  WS-J                 PIC 9 VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "Table right after start (VALUE ZERO applied):"
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
               PERFORM VARYING WS-J FROM 1 BY 1 UNTIL WS-J > 4
                   DISPLAY "  (" WS-I "," WS-J ") = "
                       COUNTER-CELL(WS-I, WS-J)
               END-PERFORM
           END-PERFORM

      *> Now put some values in, to prove they can change.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
               PERFORM VARYING WS-J FROM 1 BY 1 UNTIL WS-J > 4
                   COMPUTE COUNTER-CELL(WS-I, WS-J) = WS-I + WS-J
               END-PERFORM
           END-PERFORM

           DISPLAY " "
           DISPLAY "After filling with I+J:"
           DISPLAY "  (2,3) = " COUNTER-CELL(2, 3)

      *> INITIALIZE resets every occurrence back to its
      *> defined starting value (ZERO for numeric fields here).
           INITIALIZE COUNTER-TABLE

           DISPLAY " "
           DISPLAY "After INITIALIZE COUNTER-TABLE:"
           DISPLAY "  (2,3) = " COUNTER-CELL(2, 3)
           DISPLAY "  (1,1) = " COUNTER-CELL(1, 1)

           STOP RUN.
```

### คำอธิบายโค้ด

- `COUNTER-CELL PIC 9(3) OCCURS 4 TIMES VALUE ZERO` กำหนดให้ทุกช่องของตาราง (ทั้ง 3x4=12 ช่อง)
  เริ่มต้นเป็น 0 ตั้งแต่โปรแกรมเริ่มทำงาน โดยไม่ต้องเขียนลูปมา `MOVE 0` เอง
- `INITIALIZE COUNTER-TABLE` คือคำสั่งที่รีเซ็ตทุกช่องของ `COUNTER-TABLE` (รวมทุกแถวทุกคอลัมน์)
  กลับไปเป็นค่าเริ่มต้นตามที่ประกาศไว้ (ในที่นี้คือ 0) ในคำสั่งเดียว โดยไม่ต้องเขียนลูป

### ผลลัพธ์ที่ได้จากการรันจริง

```
Table right after start (VALUE ZERO applied):
  (1,1) = 000
  ...
  (3,4) = 000
 
After filling with I+J:
  (2,3) = 005
 
After INITIALIZE COUNTER-TABLE:
  (2,3) = 000
  (1,1) = 000
```

### ข้อควรระวัง

- `VALUE` บนฟิลด์ที่อยู่ใต้ `OCCURS` จะถูกนำไปใช้กับ**ทุกสมาชิก**เหมือนกันหมด ไม่สามารถกำหนดให้แต่ละช่อง
  มีค่าเริ่มต้นต่างกันได้ด้วย `VALUE` ตัวเดียว (ถ้าต้องการค่าต่างกัน ต้องใช้ `MOVE` ทีละช่องในภายหลัง)
- `INITIALIZE` จะรีเซ็ตกลับไปที่ค่าตาม `VALUE` ที่ประกาศไว้ใน `DATA DIVISION` เท่านั้น
  ถ้าไม่มี `VALUE` ระบุไว้ ค่าเริ่มต้นของฟิลด์ตัวเลขจะเป็น 0 และฟิลด์ตัวอักษรจะเป็น SPACE โดยปริยาย

### แบบฝึกหัดที่ 177.1

**โจทย์**: จงอธิบายว่าทำไม `INITIALIZE` ถึงสำคัญมากเมื่อโปรแกรมต้องประมวลผลข้อมูลหลายชุด (batch) ในลูปใหญ่

**เฉลยแนวทาง**: เมื่อประมวลผลข้อมูลชุดที่สอง สาม สี่ ฯลฯ ในลูปเดียวกัน ถ้าไม่ล้างค่าตารางสะสม (accumulator table)
ก่อนเริ่มรอบใหม่ ค่าที่ค้างจากรอบก่อนหน้าจะปนกับค่าของรอบใหม่ ทำให้ยอดรวมผิดพลาด `INITIALIZE`
ช่วยให้มั่นใจได้ว่าทุกรอบเริ่มต้นจากศูนย์อย่างถูกต้อง โดยไม่ต้องเขียนลูป `MOVE ZERO` เองทุกครั้ง

---

## ขั้นตอนที่ 178: โปรแกรมตัวอย่าง - ตารางยอดขายรายภาค-รายเดือน และการรวมยอดแถว/คอลัมน์

### แนวคิด

การประยุกต์ใช้ตารางสองมิติที่พบบ่อยที่สุดในงานธุรกิจคือ **รายงานตารางไขว้ (cross-tabulation report)**
เช่น ยอดขายแยกตามภาคและเดือน พร้อมผลรวมท้ายแถว (row total) ผลรวมท้ายคอลัมน์ (column total)
และผลรวมใหญ่ (grand total) ขั้นตอนนี้จะแสดงเทคนิคการคำนวณทั้งสามแบบไปพร้อมกันในตารางเดียว

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s8.cob
      *> Purpose : Sales matrix, with row totals, column totals,
      *>           and grand total
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SALES-MATRIX-TOTALS.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  SALES-TABLE.
           05  REGION-ROW OCCURS 3 TIMES.
               10  MONTH-SALES PIC 9(6) OCCURS 4 TIMES.

       01  ROW-TOTAL-TABLE.
           05  ROW-TOTAL        PIC 9(7) OCCURS 3 TIMES.

       01  COL-TOTAL-TABLE.
           05  COL-TOTAL        PIC 9(7) OCCURS 4 TIMES.

       01  GRAND-TOTAL          PIC 9(8) VALUE 0.
       01  WS-R                 PIC 9 VALUE 1.
       01  WS-C                 PIC 9 VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE 120000 TO MONTH-SALES(1, 1)
           MOVE 135000 TO MONTH-SALES(1, 2)
           MOVE 128000 TO MONTH-SALES(1, 3)
           MOVE 141000 TO MONTH-SALES(1, 4)

           MOVE 98000  TO MONTH-SALES(2, 1)
           MOVE 102000 TO MONTH-SALES(2, 2)
           MOVE 115000 TO MONTH-SALES(2, 3)
           MOVE 119500 TO MONTH-SALES(2, 4)

           MOVE 156000 TO MONTH-SALES(3, 1)
           MOVE 149000 TO MONTH-SALES(3, 2)
           MOVE 163000 TO MONTH-SALES(3, 3)
           MOVE 171000 TO MONTH-SALES(3, 4)

           PERFORM VARYING WS-R FROM 1 BY 1 UNTIL WS-R > 3
               MOVE 0 TO ROW-TOTAL(WS-R)
               PERFORM VARYING WS-C FROM 1 BY 1 UNTIL WS-C > 4
                   ADD MONTH-SALES(WS-R, WS-C) TO ROW-TOTAL(WS-R)
                   ADD MONTH-SALES(WS-R, WS-C) TO COL-TOTAL(WS-C)
                   ADD MONTH-SALES(WS-R, WS-C) TO GRAND-TOTAL
               END-PERFORM
           END-PERFORM

           DISPLAY "Sales matrix (Region x Month):"
           PERFORM VARYING WS-R FROM 1 BY 1 UNTIL WS-R > 3
               DISPLAY "  Region " WS-R ": "
                   MONTH-SALES(WS-R, 1) " "
                   MONTH-SALES(WS-R, 2) " "
                   MONTH-SALES(WS-R, 3) " "
                   MONTH-SALES(WS-R, 4)
                   " | Row Total = " ROW-TOTAL(WS-R)
           END-PERFORM

           DISPLAY "  Column totals: "
               COL-TOTAL(1) " " COL-TOTAL(2) " "
               COL-TOTAL(3) " " COL-TOTAL(4)

           DISPLAY "  Grand total = " GRAND-TOTAL

           STOP RUN.
```

### คำอธิบายโค้ด

- ตาราง `ROW-TOTAL-TABLE` และ `COL-TOTAL-TABLE` เป็นตารางมิติเดียวแยกต่างหาก ใช้เก็บผลรวมของแต่ละแถว/คอลัมน์
- ในลูปซ้อนเดียว (single nested loop pass) เราสามารถสะสมค่าลงใน **สามที่พร้อมกัน**:
  ผลรวมแถว (`ROW-TOTAL(WS-R)`), ผลรวมคอลัมน์ (`COL-TOTAL(WS-C)`), และผลรวมใหญ่ (`GRAND-TOTAL`)
  นี่คือเทคนิคสำคัญที่ทำให้ไม่ต้องวนลูปซ้ำหลายรอบเพื่อคำนวณผลรวมแต่ละแบบแยกกัน

### ผลลัพธ์ที่ได้จากการรันจริง

```
Sales matrix (Region x Month):
  Region 1: 120000 135000 128000 141000 | Row Total = 0524000
  Region 2: 098000 102000 115000 119500 | Row Total = 0434500
  Region 3: 156000 149000 163000 171000 | Row Total = 0639000
  Column totals: 0374000 0386000 0406000 0431500
  Grand total = 01597500
```

### ข้อควรระวัง

- ต้อง `MOVE 0` ให้ `ROW-TOTAL(WS-R)` ก่อนเริ่มลูปคอลัมน์ในแต่ละแถวเสมอ (ไม่เช่นนั้นค่าจะสะสมผิดจากรอบก่อน)
  แต่ `COL-TOTAL` และ `GRAND-TOTAL` ต้องเคลียร์เพียงครั้งเดียวก่อนเริ่มลูปทั้งหมด (ใช้ `VALUE 0` ตอนประกาศได้)
- ควรตรวจสอบว่าขนาด `PIC` ของตัวแปรผลรวม (เช่น `GRAND-TOTAL PIC 9(8)`) ใหญ่พอที่จะรองรับผลรวมสูงสุดที่เป็นไปได้
  ถ้าเล็กเกินไป ค่าจะ overflow และถูกตัดหลักซ้าย (truncate) โดยไม่มี error เตือน

### แบบฝึกหัดที่ 178.1

**โจทย์**: จงเพิ่มการหาว่า "เดือนไหน" มียอดขายรวม (column total) สูงที่สุด แล้วแสดงผลออกมา

**เฉลยแนวทาง**: ประกาศตัวแปร `WS-BEST-MONTH PIC 9` และ `WS-BEST-COL-TOTAL PIC 9(7) VALUE 0`
จากนั้นเพิ่มลูป `PERFORM VARYING WS-C FROM 1 BY 1 UNTIL WS-C > 4` แยกต่างหากหลังคำนวณ `COL-TOTAL` เสร็จ
เพื่อเปรียบเทียบ `COL-TOTAL(WS-C)` กับ `WS-BEST-COL-TOTAL` ทีละตัว หากมากกว่าให้บันทึกทั้งค่าและหมายเลขเดือนไว้

---

## ขั้นตอนที่ 179: การค้นหาในตารางหลายมิติด้วย SEARCH และ SEARCH VARYING

### แนวคิด

จาก Part 017 เรารู้จัก `SEARCH` สำหรับค้นหาในตารางมิติเดียว สำหรับตารางหลายมิติ เราใช้เทคนิคสองแบบ:
(1) **`SEARCH ... VARYING`** ค้นหาในแกนหนึ่ง (แกนใน) ขณะที่ "ตรึง" ตำแหน่งของอีกแกนไว้คงที่
(2) **`SEARCH` ซ้อนอยู่ใน `PERFORM`** คือให้ลูปภายนอกเดินผ่านแกนแรกทีละตำแหน่ง แล้วใช้ `SEARCH` ค้นหาในแกนที่สอง
   ของแต่ละตำแหน่งนั้น จนกว่าจะเจอ หรือจนครบทุกตำแหน่งของแกนแรก

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s9.cob
      *> Purpose : SEARCH and SEARCH VARYING in a 2D table
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TWO-DIM-SEARCH.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  SALES-TABLE.
           05  REGION-ROW OCCURS 3 TIMES INDEXED BY REG-IDX.
               10  MONTH-SALES PIC 9(6)
                   OCCURS 4 TIMES INDEXED BY MON-IDX.

       01  WS-TARGET-REGION     PIC 9 VALUE 2.
       01  WS-TARGET-AMOUNT     PIC 9(6) VALUE 115000.
       01  WS-FOUND-FLAG        PIC X VALUE "N".
           88  FOUND-IT               VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE 120000 TO MONTH-SALES(1, 1)
           MOVE 135000 TO MONTH-SALES(1, 2)
           MOVE 128000 TO MONTH-SALES(1, 3)
           MOVE 141000 TO MONTH-SALES(1, 4)

           MOVE 98000  TO MONTH-SALES(2, 1)
           MOVE 102000 TO MONTH-SALES(2, 2)
           MOVE 115000 TO MONTH-SALES(2, 3)
           MOVE 119500 TO MONTH-SALES(2, 4)

           MOVE 156000 TO MONTH-SALES(3, 1)
           MOVE 149000 TO MONTH-SALES(3, 2)
           MOVE 163000 TO MONTH-SALES(3, 3)
           MOVE 171000 TO MONTH-SALES(3, 4)

      *> SEARCH VARYING: search inside one fixed region's row,
      *> while a second index variable is tracked in step.
           SET REG-IDX TO WS-TARGET-REGION
           SET MON-IDX TO 1
           SEARCH MONTH-SALES VARYING MON-IDX
               AT END
                   DISPLAY "Amount " WS-TARGET-AMOUNT
                       " not found in region " WS-TARGET-REGION
               WHEN MONTH-SALES(REG-IDX, MON-IDX) = WS-TARGET-AMOUNT
                   DISPLAY "Found " WS-TARGET-AMOUNT
                       " in region " REG-IDX
                       " month " MON-IDX
           END-SEARCH

      *> Nested SEARCH: outer PERFORM over regions, inner SEARCH
      *> over months, to find ANY region/month matching a value.
           MOVE "N" TO WS-FOUND-FLAG
           PERFORM VARYING REG-IDX FROM 1 BY 1
                   UNTIL REG-IDX > 3 OR FOUND-IT
               SET MON-IDX TO 1
               SEARCH MONTH-SALES
                   AT END
                       CONTINUE
                   WHEN MONTH-SALES(REG-IDX, MON-IDX) = 163000
                       SET FOUND-IT TO TRUE
                       DISPLAY "Nested search found 163000 at region "
                           REG-IDX " month " MON-IDX
               END-SEARCH
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- `SEARCH MONTH-SALES VARYING MON-IDX` ค้นหาในแกนเดือนของภาคที่ `REG-IDX` ชี้อยู่ (ซึ่งถูกตั้งค่าคงที่ไว้ก่อนแล้ว)
  คำว่า `VARYING MON-IDX` บอกว่า index ตัวไหนที่จะถูกขยับเดินหน้าทีละ 1 ในระหว่างการค้นหา
- ในลูปที่สอง เราใช้ `PERFORM VARYING REG-IDX` เป็นตัวควบคุมแกนภาค แล้วซ้อน `SEARCH` ธรรมดา (ไม่ระบุ `VARYING`)
  ไว้ข้างในเพื่อค้นหาในแกนเดือนของแต่ละภาคที่กำลังตรวจสอบอยู่ พร้อมใช้ 88-level (`FOUND-IT`)
  เป็นธงบอกให้ลูปภายนอกหยุดทันทีเมื่อเจอ

### ผลลัพธ์ที่ได้จากการรันจริง

```
Found 115000 in region +000000002 month +000000003
Nested search found 163000 at region +000000003 month +000000003
```

### ข้อควรระวัง

- ก่อนเรียก `SEARCH VARYING` ต้อง `SET` ทั้ง index ที่ตรึงไว้ (`REG-IDX`) และ index ที่จะให้ `SEARCH`
  ขยับ (`MON-IDX`) ให้เรียบร้อยก่อนเสมอ มิฉะนั้นอาจเริ่มค้นหาจากตำแหน่งที่ไม่ได้ตั้งใจ
- `SEARCH` (ไม่ใช่ `SEARCH ALL`) เป็นการค้นหาแบบเรียงลำดับ (linear search) ทีละตัว จึงใช้ได้กับข้อมูลที่
  ไม่ได้เรียงลำดับ แต่ถ้าข้อมูลถูกเรียงลำดับไว้แล้วและตารางมีขนาดใหญ่ ควรพิจารณา `SEARCH ALL` เพื่อประสิทธิภาพที่ดีกว่า
  (การใช้ `SEARCH ALL` กับตารางหลายมิติซับซ้อนกว่านี้ และมักใช้กับแกนที่เรียงลำดับเพียงแกนเดียว)

### แบบฝึกหัดที่ 179.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `SEARCH ... VARYING` กับการซ้อน `SEARCH` ไว้ใน `PERFORM VARYING`

**เฉลยแนวทาง**: `SEARCH ... VARYING` ใช้เมื่อเราต้องการค้นหาในแกนเดียว (แกนใน) โดยตรึงตำแหน่งของแกนนอกไว้คงที่
ค่าเดียว เหมาะกับกรณีที่รู้ตำแหน่งแกนนอกอยู่แล้ว ส่วนการซ้อน `SEARCH` ไว้ใน `PERFORM VARYING` ใช้เมื่อต้องการ
ค้นหา "ทุกตำแหน่งของแกนนอก" เพื่อหาคำตอบที่อาจอยู่ในแถวไหนก็ได้ ซึ่งครอบคลุมพื้นที่ค้นหาทั้งสองมิติอย่างสมบูรณ์

---

## ขั้นตอนที่ 180: ข้อควรระวังและเทคนิคด้านประสิทธิภาพ พร้อมโปรแกรมสรุปรวม

### สรุปข้อควรระวังสำคัญของตารางหลายมิติ

1. **ลำดับ subscript/index ต้องตรงกับลำดับการประกาศ `OCCURS` เสมอ** เรียงจากแกนนอกสุดไปแกนในสุด
2. **จำนวนรอบการวนลูปเพิ่มแบบทวีคูณ** เมื่อมิติเพิ่มขึ้น ควรระวังประสิทธิภาพเมื่อตารางมีขนาดใหญ่
3. **`INITIALIZE` และการเคลียร์ค่าตัวสะสม (accumulator)** ต้องทำในตำแหน่งลูปที่ถูกต้อง (นอกลูปที่ไม่ต้องการรีเซ็ตซ้ำ)
4. **การแสดงผลตัวแปร index โดยตรงจะได้รูปแบบแปลก ๆ** ควร `MOVE` ไปฟิลด์ `PIC 9` ก่อนแสดงผลเสมอ
5. **อย่าออกแบบเกิน 3 มิติโดยไม่จำเป็น** เพราะจะทำให้โค้ดอ่านยากและดูแลรักษายาก ควรพิจารณาแยกเป็นตารางย่อย
   หรือย้ายไปเก็บในไฟล์/ฐานข้อมูลถ้าข้อมูลมีความซับซ้อนมากขึ้น

### โปรแกรมสรุปรวม: รายงานยอดขายรายภาคแบบรวบยอด

โปรแกรมนี้นำทุกเทคนิคที่เรียนมาในบทนี้มารวมกัน: การประกาศตารางสองมิติ, การเติมข้อมูล, การคำนวณผลรวมแถว,
และการค้นหาภาคที่มียอดขายสูงสุด

```cobol
      *> ===================================================
      *> Program : s10.cob
      *> Purpose : Mini project - Quarterly regional sales report
      *>           combining declaration, fill, totals and search
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. QUARTERLY-REPORT.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  REGION-NAME-TABLE.
           05  REGION-NAME PIC X(10) OCCURS 3 TIMES.

       01  SALES-TABLE.
           05  REGION-ROW OCCURS 3 TIMES INDEXED BY REG-IDX.
               10  QTR-SALES PIC 9(6)
                   OCCURS 4 TIMES INDEXED BY QTR-IDX.

       01  ROW-TOTAL-TABLE.
           05  ROW-TOTAL PIC 9(7) OCCURS 3 TIMES.

       01  GRAND-TOTAL          PIC 9(8) VALUE 0.
       01  WS-BEST-REGION       PIC 9 VALUE 1.
       01  WS-BEST-TOTAL        PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "NORTH"   TO REGION-NAME(1)
           MOVE "CENTRAL" TO REGION-NAME(2)
           MOVE "SOUTH"   TO REGION-NAME(3)

           MOVE 120000 TO QTR-SALES(1, 1)
           MOVE 135000 TO QTR-SALES(1, 2)
           MOVE 128000 TO QTR-SALES(1, 3)
           MOVE 141000 TO QTR-SALES(1, 4)

           MOVE 98000  TO QTR-SALES(2, 1)
           MOVE 102000 TO QTR-SALES(2, 2)
           MOVE 115000 TO QTR-SALES(2, 3)
           MOVE 119500 TO QTR-SALES(2, 4)

           MOVE 156000 TO QTR-SALES(3, 1)
           MOVE 149000 TO QTR-SALES(3, 2)
           MOVE 163000 TO QTR-SALES(3, 3)
           MOVE 171000 TO QTR-SALES(3, 4)

           DISPLAY "===== Quarterly Regional Sales Report ====="
           PERFORM VARYING REG-IDX FROM 1 BY 1 UNTIL REG-IDX > 3
               MOVE 0 TO ROW-TOTAL(REG-IDX)
               PERFORM VARYING QTR-IDX FROM 1 BY 1 UNTIL QTR-IDX > 4
                   ADD QTR-SALES(REG-IDX, QTR-IDX)
                       TO ROW-TOTAL(REG-IDX)
               END-PERFORM
               ADD ROW-TOTAL(REG-IDX) TO GRAND-TOTAL

               DISPLAY REGION-NAME(REG-IDX) ": "
                   QTR-SALES(REG-IDX, 1) " "
                   QTR-SALES(REG-IDX, 2) " "
                   QTR-SALES(REG-IDX, 3) " "
                   QTR-SALES(REG-IDX, 4)
                   " | TOTAL = " ROW-TOTAL(REG-IDX)

               IF ROW-TOTAL(REG-IDX) > WS-BEST-TOTAL
                   MOVE ROW-TOTAL(REG-IDX) TO WS-BEST-TOTAL
                   SET WS-BEST-REGION TO REG-IDX
               END-IF
           END-PERFORM

           DISPLAY "---------------------------------------------"
           DISPLAY "Grand total (all regions, all quarters) = "
               GRAND-TOTAL
           DISPLAY "Best performing region = "
               REGION-NAME(WS-BEST-REGION)
               " with total " WS-BEST-TOTAL

           STOP RUN.
```

### คำอธิบายโค้ด

- โปรแกรมนี้รวม 4 เทคนิคจากขั้นตอนก่อนหน้า: ตารางสองมิติ (172), ลูปซ้อน (173), `INDEXED BY` (175),
  และการคำนวณผลรวมแถว (178) เข้าไว้ในโปรแกรมเดียวที่ทำงานได้จริงและมีประโยชน์ทางธุรกิจ
- `WS-BEST-REGION` และ `WS-BEST-TOTAL` ใช้เทคนิค "ติดตามค่าที่ดีที่สุดระหว่างวนลูป" (running maximum)
  ซึ่งเป็นรูปแบบที่พบบ่อยมากในการเขียนรายงานสรุป

### ผลลัพธ์ที่ได้จากการรันจริง

```
===== Quarterly Regional Sales Report =====
NORTH     : 120000 135000 128000 141000 | TOTAL = 0524000
CENTRAL   : 098000 102000 115000 119500 | TOTAL = 0434500
SOUTH     : 156000 149000 163000 171000 | TOTAL = 0639000
---------------------------------------------
Grand total (all regions, all quarters) = 01597500
Best performing region = SOUTH      with total 0639000
```

### ข้อควรระวัง

- สังเกตว่า `REGION-NAME` เป็นตารางแยกต่างหากจาก `SALES-TABLE` แต่ทั้งสองตารางมี "ความสัมพันธ์กันโดยนัย"
  ผ่านตำแหน่ง subscript ที่ตรงกัน (`REGION-NAME(1)` คู่กับ `QTR-SALES(1, ...)`) การออกแบบแบบนี้ต้องระวัง
  ให้ลำดับการเติมข้อมูลของทั้งสองตารางตรงกันเสมอ มิฉะนั้นชื่อภาคกับยอดขายจะไม่ตรงกัน
- ในระบบจริงที่ซับซ้อนกว่านี้ มักจะรวมชื่อภาคเข้าไปเป็นส่วนหนึ่งของ `SALES-TABLE` เอง (ในรูปแบบ "ตารางของกลุ่มข้อมูล"
  แบบที่เรียนในขั้นตอนที่ 176) เพื่อลดความเสี่ยงเรื่องข้อมูลไม่ตรงกันระหว่างสองตาราง

### แบบฝึกหัดที่ 180.1

**โจทย์**: จงปรับโปรแกรมสรุปรวมข้างต้น ให้รวม `REGION-NAME` เข้าไปเป็นส่วนหนึ่งของ `SALES-TABLE`
โดยใช้เทคนิค "ตารางของกลุ่มข้อมูล" จากขั้นตอนที่ 176 แทนการมีสองตารางแยกกัน

**เฉลยแนวทาง**: เปลี่ยนโครงสร้างเป็น:
```cobol
       01  SALES-TABLE.
           05  REGION-ROW OCCURS 3 TIMES INDEXED BY REG-IDX.
               10  REGION-NAME  PIC X(10).
               10  QTR-SALES    PIC 9(6)
                   OCCURS 4 TIMES INDEXED BY QTR-IDX.
```
วิธีนี้ทำให้ชื่อภาคและยอดขายอยู่ใน "แถว" เดียวกันของโครงสร้างเดียวกันเสมอ ไม่มีความเสี่ยงที่ข้อมูล
จะไม่ตรงกันระหว่างสองตารางอีกต่อไป การอ้างอิงชื่อภาคจะเปลี่ยนจาก `REGION-NAME(REG-IDX)` เหมือนเดิม
(เพราะยังคงใช้ subscript ตัวเดียว เนื่องจากอยู่ในระดับเดียวกับ `REGION-ROW`)

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้เรื่องตารางหลายมิติอย่างครบถ้วน ได้แก่:

- ข้อจำกัดของตารางมิติเดียว และเหตุผลที่ต้องมีตารางหลายมิติ
- การประกาศตารางสองมิติและสามมิติด้วยการซ้อน `OCCURS` หลายชั้น
- การเติมข้อมูลตารางหลายมิติด้วย `PERFORM VARYING` ซ้อนกันหลายชั้น
- การใช้ `INDEXED BY` กับตารางหลายมิติเพื่อประสิทธิภาพที่ดีขึ้น
- การสร้างตารางของกลุ่มข้อมูล (table of group items) ที่มีทั้งฟิลด์เดี่ยวและตารางย่อย
- การกำหนดค่าเริ่มต้นด้วย `VALUE` และการรีเซ็ตตารางด้วย `INITIALIZE`
- การคำนวณผลรวมแถว ผลรวมคอลัมน์ และผลรวมใหญ่ในตารางเดียว (cross-tabulation)
- การค้นหาในตารางหลายมิติด้วย `SEARCH VARYING` และการซ้อน `SEARCH` ใน `PERFORM`
- ข้อควรระวังและแนวทางออกแบบตารางหลายมิติที่ดีในโปรเจกต์จริง

ทักษะเรื่องตารางหลายมิตินี้เป็นพื้นฐานสำคัญที่จะถูกนำไปใช้ซ้ำ ๆ ตลอดหลักสูตร โดยเฉพาะเมื่อเราต้องประมวลผล
รายงานทางธุรกิจที่ซับซ้อนขึ้นในเฟสถัด ๆ ไป ใน **Part 019** เราจะเปลี่ยนไปเรียนรู้อีกทักษะสำคัญของการเขียนโปรแกรม
COBOL คือ **การจัดการข้อความ (Text/String Manipulation)** โดยเริ่มจากคำสั่ง `STRING`
ซึ่งใช้สำหรับรวมข้อความจากหลายฟิลด์เข้าเป็นฟิลด์เดียว

**[← กลับไป Part 017](part-017-subscript-search.md)** |
**[ไปยัง Part 019: การจัดการข้อความ - STRING Statement →](part-019-string-statement.md)**
