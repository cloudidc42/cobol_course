# Part 017: Subscript กับ Index และ SEARCH / SEARCH ALL (ขั้นตอนที่ 161–170)

## คำนำของ Part นี้

ใน Part 016 เราได้เรียนรู้การประกาศตาราง (Table) ด้วย `OCCURS` clause และการเข้าถึงสมาชิกแต่ละตัวด้วย
**subscript** ซึ่งเป็นตัวเลขระบุตำแหน่ง เช่น `SALES-AMT(3)` หมายถึงสมาชิกตัวที่ 3 ของตาราง `SALES-AMT`

Part นี้จะพาคุณเจาะลึกเรื่อง subscript ให้มากขึ้น พร้อมแนะนำแนวคิดที่สำคัญมากอีกตัวหนึ่งคือ **index**
(ผ่านวลี `INDEXED BY`) ซึ่งเป็นกลไกที่ COBOL ออกแบบมาโดยเฉพาะเพื่อให้การเข้าถึงตารางมีประสิทธิภาพสูงขึ้น
และปลอดภัยขึ้นในหลายกรณี นอกจากนี้เราจะเรียนรู้คำสั่งค้นหาข้อมูลในตารางสองแบบคือ **`SEARCH`**
(การค้นหาแบบเรียงลำดับทีละตัว หรือ Serial/Linear Search) และ **`SEARCH ALL`** (การค้นหาแบบทวิภาค
หรือ Binary Search ที่เร็วกว่ามากแต่ต้องการให้ตารางเรียงลำดับไว้ก่อน)

ทักษะทั้งสามเรื่องนี้ — subscript, index และ SEARCH/SEARCH ALL — เป็นพื้นฐานสำคัญที่จะถูกใช้ซ้ำตลอด
หลักสูตรที่เหลือ โดยเฉพาะเมื่อเราต้องจัดการข้อมูลปริมาณมากในตารางหลายมิติใน Part 018 ถัดไป

---

## ขั้นตอนที่ 161: ทบทวน Subscript และการใช้ตัวแปรเป็น Subscript

### แนวคิด

**Subscript** คือตัวเลข (หรือตัวแปรที่เก็บตัวเลข) ที่ใช้ระบุว่าเราต้องการเข้าถึงสมาชิกตัวที่เท่าไรของตาราง
โดย COBOL จะเริ่มนับจาก **1 เสมอ** (ไม่ใช่ 0 เหมือนหลายภาษาสมัยใหม่) เราสามารถใช้ subscript ได้ทั้งแบบ
**literal คงที่** (เช่น `PRODUCT-NAME(3)`) และแบบ **ตัวแปร** (เช่น `PRODUCT-NAME(WS-SUB)`) ซึ่งแบบหลังนี้
คือกุญแจสำคัญที่ทำให้เราเขียนลูปเดินผ่านทุกสมาชิกของตารางได้โดยไม่ต้องเขียนคำสั่งซ้ำ ๆ ทีละบรรทัด

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s1.cob
      *> Purpose : Recap OCCURS and subscript access using a
      *>           variable subscript instead of a literal
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUBSCRIPT-RECAP.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-NAME  PIC X(12) OCCURS 5 TIMES.

       01  WS-SUB               PIC 9 VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)

      *> A literal subscript accesses one fixed occurrence.
           DISPLAY "Literal subscript PRODUCT-NAME(3): "
               PRODUCT-NAME(3)

      *> A variable subscript lets the SAME statement reach a
      *> DIFFERENT occurrence each time it runs, driven by a loop.
           DISPLAY "Walking the table with a variable subscript:"
           PERFORM VARYING WS-SUB FROM 1 BY 1 UNTIL WS-SUB > 5
               DISPLAY "  Item " WS-SUB ": " PRODUCT-NAME(WS-SUB)
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- `PRODUCT-NAME(3)` คือ subscript แบบ literal คงที่ ใช้เมื่อรู้ตำแหน่งที่ต้องการแน่นอนตั้งแต่ตอนเขียนโค้ด
- `PRODUCT-NAME(WS-SUB)` คือ subscript แบบตัวแปร ค่าของ `WS-SUB` เปลี่ยนไปทุกรอบของ `PERFORM VARYING`
  ทำให้คำสั่งเดียวใน `DISPLAY` เข้าถึงสมาชิกคนละตัวในแต่ละรอบ

### ผลลัพธ์ที่ได้จากการรันจริง

```
Literal subscript PRODUCT-NAME(3): MONITOR
Walking the table with a variable subscript:
  Item 1: KEYBOARD
  Item 2: MOUSE
  Item 3: MONITOR
  Item 4: HEADSET
  Item 5: WEBCAM
```

### ข้อควรระวัง

- อย่าลืมว่า COBOL เริ่มนับ subscript จาก **1** เสมอ ผู้ที่เคยเขียนภาษาอื่นที่เริ่มนับจาก 0 (เช่น C, Java,
  Python) มักทำผิดพลาดตรงนี้บ่อยในช่วงแรก
- ตัวแปรที่ใช้เป็น subscript ต้องเป็นชนิดตัวเลขจำนวนเต็ม (ไม่มีทศนิยม) เท่านั้น

### แบบฝึกหัดที่ 161.1

**โจทย์**: จงแก้โค้ดข้างต้นให้แสดงผลตารางแบบย้อนกลับ (จากสมาชิกตัวที่ 5 ไปตัวที่ 1)

**เฉลย**:
```cobol
           PERFORM VARYING WS-SUB FROM 5 BY -1 UNTIL WS-SUB < 1
               DISPLAY "  Item " WS-SUB ": " PRODUCT-NAME(WS-SUB)
           END-PERFORM
```
ผลลัพธ์จะแสดง WEBCAM, HEADSET, MONITOR, MOUSE, KEYBOARD ตามลำดับ

---

## ขั้นตอนที่ 162: ข้อจำกัดของ Subscript ธรรมดา และความเสี่ยงเมื่อค่าเกินขอบเขต

### แนวคิด

Subscript ธรรมดา (plain numeric subscript) มีข้อจำกัดสำคัญสองข้อ: (1) **ประสิทธิภาพ** — ทุกครั้งที่ COBOL
เข้าถึงสมาชิกตาราง มันต้อง**คำนวณตำแหน่งในหน่วยความจำใหม่**จากค่า subscript คูณด้วยขนาดของสมาชิกแต่ละตัว
ซึ่งเป็นการคำนวณคณิตศาสตร์ทุกครั้งที่ใช้งาน และ (2) **ความปลอดภัย** — GnuCOBOL แบบมาตรฐาน (ไม่เปิด flag
ตรวจสอบขอบเขตเป็นพิเศษ) **จะไม่หยุดโปรแกรม**หากเราใช้ค่า subscript ที่เกินขอบเขตของตาราง (เช่น ตารางมี
5 ช่อง แต่ subscript มีค่า 7) พฤติกรรมที่ได้จะไม่แน่นอน (undefined) และเป็นสาเหตุของบั๊กที่ตรวจจับยากมาก
วิธีป้องกันที่ถูกต้องคือ **ตรวจสอบค่า subscript เองด้วย `IF`ก่อนใช้งานเสมอ**

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s2.cob
      *> Purpose : Show why an unchecked subscript is dangerous,
      *>           and the defensive-IF pattern to guard it
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUBSCRIPT-BOUNDS-RISK.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-NAME  PIC X(12) OCCURS 5 TIMES.

       01  WS-SUB               PIC 9 VALUE 1.
       01  WS-REQUESTED-POS     PIC 9 VALUE 7.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)

      *> GnuCOBOL, by default, does NOT stop you from using a
      *> subscript value outside 1 THRU 5 at compile time. Whether
      *> it is caught at run time depends on compiler flags that
      *> are not on by default, so the SAFE habit is to validate
      *> the subscript value yourself BEFORE using it.
           DISPLAY "Requested position: " WS-REQUESTED-POS

           IF WS-REQUESTED-POS >= 1 AND WS-REQUESTED-POS <= 5
               DISPLAY "Product: " PRODUCT-NAME(WS-REQUESTED-POS)
           ELSE
               DISPLAY "ERROR: position " WS-REQUESTED-POS
                   " is outside the table (1 to 5)."
           END-IF

      *> Now try a position that IS inside range, to show the
      *> normal, safe path through the very same guard.
           MOVE 3 TO WS-REQUESTED-POS
           DISPLAY " "
           DISPLAY "Requested position: " WS-REQUESTED-POS
           IF WS-REQUESTED-POS >= 1 AND WS-REQUESTED-POS <= 5
               DISPLAY "Product: " PRODUCT-NAME(WS-REQUESTED-POS)
           ELSE
               DISPLAY "ERROR: position " WS-REQUESTED-POS
                   " is outside the table (1 to 5)."
           END-IF

           STOP RUN.
```

### คำอธิบายโค้ด

- `IF WS-REQUESTED-POS >= 1 AND WS-REQUESTED-POS <= 5` คือ**การ์ด (guard)** ที่ตรวจสอบค่า subscript
  ก่อนนำไปใช้งานจริงเสมอ เป็นแบบแผน (pattern) ที่ควรทำเป็นนิสัยทุกครั้งที่ค่า subscript มาจากแหล่งภายนอก
  (ผู้ใช้ป้อน, ไฟล์, การคำนวณ) ที่ไม่สามารถรับประกันขอบเขตได้ล่วงหน้า
- โปรแกรมนี้จงใจ**ไม่**เข้าถึง `PRODUCT-NAME(7)` โดยตรง เพราะพฤติกรรมเมื่อ subscript เกินขอบเขตไม่แน่นอน
  (อาจอ่านขยะจากหน่วยความจำ หรือในบางกรณี crash) การสาธิตด้วยการ์ดที่ปลอดภัยจึงเป็นทางเลือกที่ถูกต้องกว่า

### ผลลัพธ์ที่ได้จากการรันจริง

```
Requested position: 7
ERROR: position 7 is outside the table (1 to 5).

Requested position: 3
Product: MONITOR
```

### ข้อควรระวัง

- **ห้ามสมมติว่า subscript ที่มาจากภายนอก (input ผู้ใช้, ค่าที่คำนวณจากไฟล์) จะอยู่ในขอบเขตเสมอ**
  ต้องตรวจสอบทุกครั้งก่อนใช้งาน โดยเฉพาะในระบบการเงินที่ความผิดพลาดเงียบ ๆ อาจสร้างความเสียหายมหาศาล
- การคำนวณตำแหน่งใหม่ทุกครั้งของ subscript ธรรมดาอาจทำให้ประสิทธิภาพลดลงเมื่อเข้าถึงตารางบ่อยมาก
  ในลูปขนาดใหญ่ นี่คือเหตุผลที่ COBOL มีกลไก `INDEXED BY` ซึ่งเราจะเรียนรู้ในขั้นตอนถัดไป

### แบบฝึกหัดที่ 162.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบขอบเขต subscript จึงสำคัญเป็นพิเศษในระบบที่รับค่าจากผู้ใช้โดยตรง
(เช่น ระบบค้นหาสินค้าตามหมายเลขที่ผู้ใช้พิมพ์เข้ามา)

**เฉลยแนวทาง**: เพราะผู้ใช้อาจพิมพ์ค่าที่ไม่ถูกต้องหรือเกินขอบเขตได้เสมอ (ตั้งใจหรือไม่ตั้งใจก็ตาม)
หากโปรแกรมไม่ตรวจสอบก่อน อาจเกิดการอ่านข้อมูลที่ไม่เกี่ยวข้อง แสดงผลผิดพลาด หรือโปรแกรม crash กลางคัน
ซึ่งในระบบธุรกิจจริงอาจหมายถึงการหยุดชะงักของบริการที่กระทบผู้ใช้จำนวนมาก

---

## ขั้นตอนที่ 163: การประกาศ INDEXED BY และการใช้ SET

### แนวคิด

COBOL มีกลไกพิเศษเรียกว่า **index** ซึ่งประกาศผ่านวลี **`INDEXED BY ชื่อ-index`** ต่อท้าย `OCCURS`
index แตกต่างจาก subscript ธรรมดาตรงที่มันไม่ใช่ตัวแปรตัวเลขปกติ (`PIC 9`) แต่เป็นชนิดข้อมูลพิเศษภายใน
(USAGE INDEX) ที่**เก็บค่า offset ในหน่วยความจำโดยตรง** แทนที่จะเก็บแค่ "หมายเลขลำดับ" ทำให้การเข้าถึง
ตารางเร็วขึ้นเพราะไม่ต้องคำนวณตำแหน่งใหม่ทุกครั้ง ข้อแตกต่างสำคัญคือ **ต้องใช้คำสั่ง `SET` เพื่อกำหนดค่า
ให้ index เท่านั้น** ไม่สามารถใช้ `MOVE` ตรง ๆ ได้เหมือน subscript ธรรมดา

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s3.cob
      *> Purpose : Declaring INDEXED BY and using SET to
      *>           position an index
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INDEX-BASICS.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-NAME  PIC X(12)
               OCCURS 5 TIMES INDEXED BY PROD-IDX.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)

      *> PROD-IDX is not an ordinary numeric variable: it must be
      *> positioned with SET, not with MOVE or COMPUTE directly.
           SET PROD-IDX TO 1
           DISPLAY "PROD-IDX = 1 -> " PRODUCT-NAME(PROD-IDX)

           SET PROD-IDX TO 4
           DISPLAY "PROD-IDX = 4 -> " PRODUCT-NAME(PROD-IDX)

      *> SET can also copy the position of one index into another.
           SET PROD-IDX TO 2
           DISPLAY "PROD-IDX = 2 -> " PRODUCT-NAME(PROD-IDX)

           STOP RUN.
```

### คำอธิบายโค้ด

- `OCCURS 5 TIMES INDEXED BY PROD-IDX` ประกาศ `PROD-IDX` เป็น index ที่ผูกติดกับตาราง `PRODUCT-NAME`
  โดยเฉพาะ ไม่ต้องประกาศตัวแปรแยกต่างหากในลักษณะ `01 PROD-IDX PIC 9.` เหมือน subscript ธรรมดา
- `SET PROD-IDX TO 1` คือวิธีที่ถูกต้องเดียวที่ใช้กำหนดค่าให้ index (ไม่ใช่ `MOVE 1 TO PROD-IDX`)

### ผลลัพธ์ที่ได้จากการรันจริง

```
PROD-IDX = 1 -> KEYBOARD
PROD-IDX = 4 -> HEADSET
PROD-IDX = 2 -> MOUSE
```

### ข้อควรระวัง

- ถ้าเผลอใช้ `MOVE 1 TO PROD-IDX` แทน `SET PROD-IDX TO 1` โปรแกรมส่วนใหญ่จะไม่ compile ผ่าน หรือถ้า
  compile ผ่าน (บางคอมไพเลอร์ผ่อนปรน) พฤติกรรมจะไม่ตรงตามที่คาดหวัง เพราะ index ไม่ใช่ฟิลด์ตัวเลขปกติ
  ควรใช้ `SET` เสมอเมื่อทำงานกับตัวแปรที่ประกาศผ่าน `INDEXED BY`
- index หนึ่งตัวผูกติดกับตารางที่มันถูกประกาศไว้เท่านั้น จะนำ `PROD-IDX` ไปใช้กับตารางอื่นที่ไม่ได้ประกาศ
  `INDEXED BY PROD-IDX` ไว้ไม่ได้

### แบบฝึกหัดที่ 163.1

**โจทย์**: จงเพิ่มตาราง `01 CATEGORY-TABLE. 05 CATEGORY-NAME PIC X(10) OCCURS 3 TIMES INDEXED BY CAT-IDX.`
และแสดงผลว่า `PROD-IDX` กับ `CAT-IDX` เป็นตัวแปรคนละตัวกัน แม้จะมีค่าตัวเลขเดียวกันได้

**เฉลยแนวทาง**: เพิ่มการประกาศตารางใหม่ตามโจทย์ แล้ว `SET CAT-IDX TO 2` และ `SET PROD-IDX TO 2`
พร้อมกัน จะเห็นว่าทั้งสองตัวมีค่าเป็น 2 เหมือนกันได้ในเวลาเดียวกัน โดยไม่กระทบกัน เพราะแต่ละตัวผูกกับ
ตารางของตัวเองแยกกันอย่างสมบูรณ์

---

## ขั้นตอนที่ 164: PERFORM VARYING กับ INDEX และ SET ... UP BY / DOWN BY

### แนวคิด

`PERFORM VARYING` สามารถใช้ index แทนตัวแปรตัวเลขธรรมดาได้โดยตรง โดย COBOL จะจัดการเรียก `SET`
ให้อัตโนมัติในทุกรอบของลูป นอกจากนี้ เรายังสามารถขยับตำแหน่งของ index ด้วยมือได้ผ่านวลี
**`SET ... UP BY จำนวน`** (เลื่อนไปข้างหน้า) และ **`SET ... DOWN BY จำนวน`** (เลื่อนถอยหลัง)
ซึ่งมีประโยชน์เมื่อต้องขยับ index นอกเหนือจากบริบทของ `PERFORM VARYING`

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s4.cob
      *> Purpose : PERFORM VARYING driven by an INDEX, and
      *>           SET ... UP BY / DOWN BY for manual stepping
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INDEX-PERFORM-VARYING.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-NAME  PIC X(12)
               OCCURS 5 TIMES INDEXED BY PROD-IDX.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)

      *> PERFORM VARYING accepts an index name directly. COBOL
      *> uses SET internally to move it, which is faster than
      *> recomputing an offset from a plain numeric subscript.
           DISPLAY "Forward walk with PERFORM VARYING (index):"
           PERFORM VARYING PROD-IDX FROM 1 BY 1
                   UNTIL PROD-IDX > 5
               DISPLAY "  " PRODUCT-NAME(PROD-IDX)
           END-PERFORM

      *> SET ... UP BY / DOWN BY moves an index manually by a
      *> given amount, useful outside a PERFORM VARYING header.
           DISPLAY " "
           DISPLAY "Manual backward walk with SET DOWN BY:"
           SET PROD-IDX TO 5
           PERFORM 5 TIMES
               DISPLAY "  " PRODUCT-NAME(PROD-IDX)
               SET PROD-IDX DOWN BY 1
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- `PERFORM VARYING PROD-IDX FROM 1 BY 1 UNTIL PROD-IDX > 5` ใช้ index ตรง ๆ แทนตัวแปร `PIC 9` ธรรมดา
  ไวยากรณ์เหมือนกันทุกประการ ต่างกันแค่ชนิดของตัวแปรที่ควบคุมลูปเบื้องหลัง
- `SET PROD-IDX DOWN BY 1` ลดค่า index ลงทีละ 1 ในแต่ละรอบของ `PERFORM 5 TIMES` ทำให้เดินตารางย้อนกลับ
  ได้โดยไม่ต้องใช้ `PERFORM VARYING ... BY -1` (ซึ่งใช้ได้กับ subscript ธรรมดา แต่กับ index นิยมใช้
  `SET ... DOWN BY` แทน)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Forward walk with PERFORM VARYING (index):
  KEYBOARD
  MOUSE
  MONITOR
  HEADSET
  WEBCAM

Manual backward walk with SET DOWN BY:
  WEBCAM
  HEADSET
  MONITOR
  MOUSE
  KEYBOARD
```

### ข้อควรระวัง

- ในตัวอย่างนี้ หลัง `PERFORM 5 TIMES` จบรอบสุดท้าย `PROD-IDX` จะถูก `SET DOWN BY 1` จนมีค่าเป็น 0
  ซึ่งอยู่นอกขอบเขตตาราง (1 ถึง 5) แต่เนื่องจากลูปหยุดพอดี**ก่อน**ที่จะนำค่า 0 นี้ไปเข้าถึงตาราง
  จึงไม่มีปัญหา ข้อคิดคือควรตรวจสอบเสมอว่าค่า index หลังจบลูปจะไม่ถูกนำไปใช้งานต่อโดยไม่ได้ตั้งใจ
- `PERFORM VARYING` กับ index ไม่รองรับ `BY -1` ตรง ๆ ในทุกคอมไพเลอร์เท่ากับ subscript ธรรมดา
  ควรทดสอบให้แน่ใจ หรือใช้แบบแผน `SET ... DOWN BY` ที่แสดงในตัวอย่างนี้แทนเพื่อความชัดเจน

### แบบฝึกหัดที่ 164.1

**โจทย์**: จงแก้โค้ดส่วนที่สองให้ใช้ `SET PROD-IDX UP BY 1` แทน แล้วเริ่มจาก `SET PROD-IDX TO 1`
เพื่อให้ได้ผลลัพธ์เดินหน้าเหมือนส่วนแรก

**เฉลย**: เปลี่ยน `SET PROD-IDX TO 5` เป็น `SET PROD-IDX TO 1` และเปลี่ยน `SET PROD-IDX DOWN BY 1`
เป็น `SET PROD-IDX UP BY 1` ผลลัพธ์ที่ได้จะเป็น KEYBOARD, MOUSE, MONITOR, HEADSET, WEBCAM เรียงลำดับปกติ

---

## ขั้นตอนที่ 165: เปรียบเทียบ Subscript กับ Index อย่างละเอียด

### แนวคิด

ขั้นตอนนี้จะสรุปเปรียบเทียบ subscript กับ index แบบเคียงข้างกัน เพื่อให้เห็นความแตกต่างชัดเจนในทุกมิติ:
ชนิดข้อมูล, วิธีกำหนดค่า, การแสดงผล, และการคำนวณ

| หัวข้อ | Subscript ธรรมดา | Index (`INDEXED BY`) |
|---|---|---|
| ชนิดข้อมูล | ตัวแปรตัวเลขปกติ (`PIC 9...`) | ชนิดพิเศษภายใน (USAGE INDEX) |
| วิธีกำหนดค่า | `MOVE`, `COMPUTE`, `ADD` ได้ตามปกติ | ต้องใช้ `SET` เท่านั้น |
| ความเร็วในการเข้าถึงตาราง | คำนวณ offset ใหม่ทุกครั้ง | เก็บ offset ไว้แล้ว เร็วกว่า |
| การแสดงผลตรง ๆ ด้วย DISPLAY | แสดงค่าตัวเลขปกติ | แสดงค่าภายในที่ไม่เป็นมิตร (เช่น `+000000003`) |
| ใช้กับ SEARCH ได้หรือไม่ | ใช้กับ `SEARCH` ธรรมดาได้ | จำเป็นสำหรับ `SEARCH` และ `SEARCH ALL` |

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s5.cob
      *> Purpose : Compare a plain subscript variable with an
      *>           INDEXED BY index side by side
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUBSCRIPT-VS-INDEX.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-NAME  PIC X(12)
               OCCURS 5 TIMES INDEXED BY PROD-IDX.

       01  WS-SUB               PIC 9 VALUE 1.
       01  WS-DISPLAY-IDX        PIC 9.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)

      *> WS-SUB is an ordinary PIC 9 numeric field: MOVE, COMPUTE
      *> and ADD all work on it directly, just like any variable.
           MOVE 2 TO WS-SUB
           COMPUTE WS-SUB = WS-SUB + 1
           DISPLAY "Plain subscript WS-SUB = " WS-SUB
           DISPLAY "  -> " PRODUCT-NAME(WS-SUB)

      *> PROD-IDX is an internal INDEX data item (USAGE INDEX).
      *> DISPLAYing it directly shows a raw internal value, NOT
      *> the friendly occurrence number a beginner expects.
           SET PROD-IDX TO 3
           DISPLAY "Raw DISPLAY of PROD-IDX: " PROD-IDX

      *> To show it as a normal number, MOVE it into a PIC 9 item.
           MOVE PROD-IDX TO WS-DISPLAY-IDX
           DISPLAY "PROD-IDX shown via WS-DISPLAY-IDX: "
               WS-DISPLAY-IDX
           DISPLAY "  -> " PRODUCT-NAME(PROD-IDX)

      *> Since COBOL-85, SET also allows simple arithmetic
      *> directly on an index, without a helper numeric field.
           SET PROD-IDX UP BY 1
           MOVE PROD-IDX TO WS-DISPLAY-IDX
           DISPLAY "After SET PROD-IDX UP BY 1: " WS-DISPLAY-IDX
           DISPLAY "  -> " PRODUCT-NAME(PROD-IDX)

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-SUB` ทำงานเหมือนตัวแปรตัวเลขทั่วไปทุกประการ ใช้ `MOVE` และ `COMPUTE` ได้ตรง ๆ
- `DISPLAY PROD-IDX` แสดงค่าดิบภายในเป็น `+000000003` ไม่ใช่ `3` ธรรมดา นี่คือจุดที่มือใหม่งงบ่อยที่สุด
  ต้อง `MOVE` ไปยังฟิลด์ `PIC 9` ก่อนเสมอถ้าต้องการแสดงผลที่อ่านง่าย
- `SET PROD-IDX UP BY 1` แสดงว่า index รองรับการคำนวณบวก/ลบง่าย ๆ ผ่าน `SET` ได้โดยตรง โดยไม่ต้องพึ่ง
  ตัวแปรตัวเลขช่วย

### ผลลัพธ์ที่ได้จากการรันจริง

```
Plain subscript WS-SUB = 3
  -> MONITOR
Raw DISPLAY of PROD-IDX: +000000003
PROD-IDX shown via WS-DISPLAY-IDX: 3
  -> MONITOR
After SET PROD-IDX UP BY 1: 4
  -> HEADSET
```

### ข้อควรระวัง

- ค่า `+000000003` ที่เห็นเมื่อ `DISPLAY` index ตรง ๆ ไม่ใช่ error แต่เป็นพฤติกรรมปกติของชนิดข้อมูล
  USAGE INDEX ภายใน ไม่ควรตกใจหรือพยายามแก้ไขมันเป็นอย่างอื่น เพียงจำไว้ว่าต้อง `MOVE` ไปยังฟิลด์
  `PIC 9` ก่อนแสดงผลให้ผู้ใช้เห็นเสมอ
- อย่าพยายามใช้ index ในสูตรคำนวณที่ซับซ้อน (เช่น `COMPUTE` กับตัวคูณหลายตัว) เพราะ index ออกแบบมา
  สำหรับการเข้าถึงตารางโดยเฉพาะ ถ้าต้องคำนวณซับซ้อน ควร `MOVE` ไปตัวแปรตัวเลขปกติก่อน

### แบบฝึกหัดที่ 165.1

**โจทย์**: จงสรุปเป็นกฎ 1 ประโยคว่า "เมื่อไรควรใช้ subscript ธรรมดา และเมื่อไรควรใช้ index"

**เฉลยแนวทาง**: ควรใช้ subscript ธรรมดาเมื่อค่าต้องผ่านการคำนวณซับซ้อนหรือแสดงผลบ่อย ๆ
และควรใช้ index (`INDEXED BY`) เมื่อเน้นประสิทธิภาพการเข้าถึงตารางในลูปขนาดใหญ่ หรือเมื่อจำเป็นต้องใช้
`SEARCH`/`SEARCH ALL` เพราะทั้งสองคำสั่งนี้ต้องการ index เสมอ

---

## ขั้นตอนที่ 166: ค้นหาข้อมูลในตารางด้วย SEARCH (Serial Search) พื้นฐาน

### แนวคิด

**`SEARCH`** คือคำสั่งค้นหาที่ COBOL มีให้ในตัว โดยจะ**เดินไล่ตรวจสอบสมาชิกทีละตัวตามลำดับ**
(เรียกว่า Serial Search หรือ Linear Search) เริ่มจากตำแหน่งที่ index ชี้อยู่ในขณะนั้น จนกว่าจะเจอเงื่อนไข
ที่ตรงกับ `WHEN` หรือจนกว่าจะครบทุกตัวแล้วไม่เจอ (จะเข้า `AT END`) ข้อดีของ `SEARCH` คือใช้ได้กับตารางที่
**ไม่ได้เรียงลำดับ** และไม่จำกัดชนิดของเงื่อนไข (ใช้ `<`, `>`, `NOT =` ได้ตามต้องการ)

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s6.cob
      *> Purpose : Basic SEARCH (serial search) with AT END
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SEARCH-BASIC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ROW OCCURS 5 TIMES INDEXED BY PROD-IDX.
               10  PRODUCT-CODE  PIC X(4).
               10  PRODUCT-NAME  PIC X(12).

       01  WS-TARGET-CODE       PIC X(4) VALUE "P003".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "P001" TO PRODUCT-CODE(1)
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE "P002" TO PRODUCT-CODE(2)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE "P003" TO PRODUCT-CODE(3)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE "P004" TO PRODUCT-CODE(4)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE "P005" TO PRODUCT-CODE(5)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)

      *> SEARCH walks the table entry by entry, starting from
      *> wherever PROD-IDX currently points. We must SET it first.
           SET PROD-IDX TO 1
           SEARCH PRODUCT-ROW
               AT END
                   DISPLAY "Code " WS-TARGET-CODE " was not found."
               WHEN PRODUCT-CODE(PROD-IDX) = WS-TARGET-CODE
                   DISPLAY "Found: " PRODUCT-NAME(PROD-IDX)
                       " at position " PROD-IDX
           END-SEARCH

      *> Search for a code that does not exist, to show AT END.
           MOVE "P999" TO WS-TARGET-CODE
           SET PROD-IDX TO 1
           SEARCH PRODUCT-ROW
               AT END
                   DISPLAY "Code " WS-TARGET-CODE " was not found."
               WHEN PRODUCT-CODE(PROD-IDX) = WS-TARGET-CODE
                   DISPLAY "Found: " PRODUCT-NAME(PROD-IDX)
                       " at position " PROD-IDX
           END-SEARCH

           STOP RUN.
```

### คำอธิบายโค้ด

- `SEARCH PRODUCT-ROW` บอก COBOL ให้ค้นหาในตารางที่ `PRODUCT-ROW` เป็นระดับที่ `OCCURS` (คือระดับที่
  ประกาศ `INDEXED BY` ไว้) ไม่ใช่ `PRODUCT-CODE` หรือ `PRODUCT-NAME` โดยตรง
- **ต้อง `SET PROD-IDX TO 1` ก่อนเรียก `SEARCH` เสมอ** เพื่อบอกจุดเริ่มต้นค้นหา (ต่างจาก `SEARCH ALL`
  ที่ไม่ต้องตั้งค่าก่อน ซึ่งจะเรียนในขั้นตอนที่ 168)
- `AT END` ทำงานเมื่อค้นหาจนครบทุกสมาชิกแล้วไม่พบเงื่อนไขใด ๆ ที่ตรงกับ `WHEN`
- เมื่อพบข้อมูลที่ตรงเงื่อนไข `PROD-IDX` จะหยุดอยู่ที่ตำแหน่งนั้นโดยอัตโนมัติ ทำให้ใช้ `PROD-IDX`
  อ้างอิงข้อมูลแถวเดียวกันในส่วน `WHEN` ได้ทันที

### ผลลัพธ์ที่ได้จากการรันจริง

```
Found: MONITOR      at position +000000003
Code P999 was not found.
```

### ข้อควรระวัง

- **ลืม `SET PROD-IDX TO 1` ก่อน `SEARCH`** เป็นข้อผิดพลาดที่พบบ่อยที่สุด ถ้า index ค้างอยู่ที่ตำแหน่ง
  กลางตาราง (เช่น 4) จากการค้นหาครั้งก่อน `SEARCH` ครั้งใหม่จะเริ่มค้นจากตำแหน่ง 4 แทนที่จะเริ่มจากตำแหน่ง
  1 ทำให้พลาดข้อมูลในตำแหน่งก่อนหน้านั้นไปโดยไม่รู้ตัว
- สังเกตว่า `DISPLAY ... PROD-IDX` แสดงค่าดิบ `+000000003` ตามที่อธิบายในขั้นตอนที่ 165 — ในระบบจริง
  ควร `MOVE` ไปฟิลด์ `PIC 9` ก่อนแสดงผลให้ผู้ใช้เห็น

### แบบฝึกหัดที่ 166.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมข้างต้นต้องเรียก `SET PROD-IDX TO 1` ซ้ำสองครั้ง (ก่อน `SEARCH` แต่ละครั้ง)

**เฉลยแนวทาง**: เพราะหลังจาก `SEARCH` ครั้งแรกทำงานเสร็จ (ไม่ว่าจะเจอหรือไม่เจอ) `PROD-IDX` จะค้างอยู่ที่
ตำแหน่งสุดท้ายที่มันไปถึง (ตำแหน่ง 3 ในกรณีเจอ) ถ้าไม่รีเซ็ตกลับเป็น 1 ก่อนเรียก `SEARCH` ครั้งที่สอง
มันจะเริ่มค้นจากตำแหน่ง 3 แทนที่จะเป็นตำแหน่ง 1 ซึ่งอาจทำให้ผลการค้นหาผิดพลาดได้

---

## ขั้นตอนที่ 167: SEARCH กับหลาย WHEN และ SEARCH ... VARYING

### แนวคิด

`SEARCH` หนึ่งคำสั่งสามารถมี **`WHEN` ได้หลายเงื่อนไข** โดย COBOL จะตรวจสอบทุกเงื่อนไขที่ตำแหน่งปัจจุบัน
ก่อนขยับไปตำแหน่งถัดไป เงื่อนไขแรกที่ตรงจะถูกเลือกใช้งาน (คล้ายกับ `IF...ELSE IF` แต่ตรวจในตำแหน่งเดียวกัน
ก่อนขยับ) นอกจากนี้ยังมีวลี **`SEARCH ... VARYING ตัวแปร-อื่น`** ที่ทำให้ index หรือตัวแปรตัวที่สอง
**ขยับตามไปพร้อมกัน** ในทุกจังหวะที่ index หลักขยับ มีประโยชน์เมื่อต้องติดตามตำแหน่งคู่ขนานในโครงสร้าง
ข้อมูลอื่นที่ไม่ใช่ตารางเดียวกัน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s7.cob
      *> Purpose : SEARCH with multiple WHEN conditions, and
      *>           VARYING to step a second index in lock-step
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SEARCH-MULTI-WHEN.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ROW OCCURS 5 TIMES INDEXED BY PROD-IDX.
               10  PRODUCT-CODE  PIC X(4).
               10  PRODUCT-NAME  PIC X(12).
               10  PRODUCT-STOCK PIC 9(4).

       01  WS-RANK-TABLE.
           05  WS-RANK PIC 9 OCCURS 5 TIMES INDEXED BY RANK-IDX.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "P001" TO PRODUCT-CODE(1)
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE 0 TO PRODUCT-STOCK(1)
           MOVE "P002" TO PRODUCT-CODE(2)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE 50 TO PRODUCT-STOCK(2)
           MOVE "P003" TO PRODUCT-CODE(3)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE 0 TO PRODUCT-STOCK(3)
           MOVE "P004" TO PRODUCT-CODE(4)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE 12 TO PRODUCT-STOCK(4)
           MOVE "P005" TO PRODUCT-CODE(5)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)
           MOVE 0 TO PRODUCT-STOCK(5)

      *> A single SEARCH can test several WHEN conditions. The
      *> first WHEN that matches wins; the others are skipped.
           SET PROD-IDX TO 1
           SEARCH PRODUCT-ROW
               AT END
                   DISPLAY "No out-of-stock product remains."
               WHEN PRODUCT-STOCK(PROD-IDX) = 0
                   DISPLAY "First out-of-stock item: "
                       PRODUCT-NAME(PROD-IDX)
               WHEN PRODUCT-STOCK(PROD-IDX) < 20
                   DISPLAY "First low-stock item: "
                       PRODUCT-NAME(PROD-IDX)
           END-SEARCH

      *> SEARCH ... VARYING advances a SECOND index in step with
      *> the table's own index, useful for tracking a rank or a
      *> position in a separate, parallel structure.
           SET PROD-IDX TO 1
           SET RANK-IDX TO 1
           SEARCH PRODUCT-ROW VARYING RANK-IDX
               AT END
                   DISPLAY "Reached end while ranking entries."
               WHEN PRODUCT-CODE(PROD-IDX) = "P004"
                   DISPLAY "P004 found; parallel RANK-IDX moved to "
                       RANK-IDX
           END-SEARCH

           STOP RUN.
```

### คำอธิบายโค้ด

- ในการ `SEARCH` ครั้งแรก มี 2 เงื่อนไข `WHEN` เรียงกัน: COBOL จะตรวจสอบ `WHEN` แรกก่อนที่ตำแหน่งปัจจุบัน
  เสมอ ถ้าไม่ตรงจึงตรวจ `WHEN` ถัดไปที่ตำแหน่งเดียวกัน ถ้ายังไม่ตรงอีกจึงขยับ index ไปตำแหน่งถัดไป
  แล้ววนตรวจใหม่ทั้งหมด — ผลคือรายการแรกที่ stock เท่ากับ 0 (ตำแหน่ง 1) จะถูกจับได้ก่อนเสมอ
- `SEARCH PRODUCT-ROW VARYING RANK-IDX` ทำให้ `RANK-IDX` ขยับตาม `PROD-IDX` ไปทีละ 1 ทุกครั้งที่
  `PROD-IDX` ขยับ แม้ `RANK-IDX` จะผูกกับตารางคนละตัว (`WS-RANK-TABLE`) ก็ตาม

### ผลลัพธ์ที่ได้จากการรันจริง

```
First out-of-stock item: KEYBOARD
P004 found; parallel RANK-IDX moved to +000000004
```

### ข้อควรระวัง

- ลำดับของ `WHEN` มีผลต่อผลลัพธ์เสมอ ถ้าสลับลำดับ `WHEN PRODUCT-STOCK(PROD-IDX) < 20` ไว้ก่อน
  `WHEN PRODUCT-STOCK(PROD-IDX) = 0` ผลลัพธ์แรกที่ได้จะยังคงเป็นตำแหน่ง 1 เหมือนเดิม (เพราะ stock=0
  ก็ < 20 ด้วย) แต่ในกรณีอื่นที่เงื่อนไขซับซ้อนกว่านี้ ลำดับอาจเปลี่ยนผลลัพธ์ได้ ควรเรียงจากเงื่อนไข
  ที่เฉพาะเจาะจงที่สุดไปหากว้างที่สุดเสมอ
- `SEARCH ... VARYING` ไม่ได้ทำให้ตัวแปรที่สองมีความสัมพันธ์เชิงตรรกะกับข้อมูลในตารางหลักโดยอัตโนมัติ
  มันแค่ขยับค่าตามจำนวนรอบเท่านั้น ผู้เขียนโปรแกรมต้องรับผิดชอบให้แน่ใจว่าความสัมพันธ์ระหว่างสองตาราง
  ถูกต้องตามที่ออกแบบไว้

### แบบฝึกหัดที่ 167.1

**โจทย์**: จงเพิ่ม `WHEN` ที่สามเพื่อตรวจสอบ `PRODUCT-STOCK(PROD-IDX) > 30` และแสดงข้อความ
"High-stock item" หากพบ

**เฉลย**:
```cobol
               WHEN PRODUCT-STOCK(PROD-IDX) > 30
                   DISPLAY "High-stock item: " PRODUCT-NAME(PROD-IDX)
```
เพิ่มต่อจาก `WHEN` เดิมก่อน `END-SEARCH` แต่เนื่องจากตำแหน่ง 1 มี stock=0 ตรงกับเงื่อนไขแรกอยู่แล้ว
`WHEN` ใหม่นี้จะไม่มีผลต่อผลลัพธ์ของการค้นหาครั้งแรกนี้ (มันจะทำงานก็ต่อเมื่อ `WHEN` ก่อนหน้าไม่ตรง)

---

## ขั้นตอนที่ 168: SEARCH ALL (Binary Search) และข้อกำหนดเรื่องการเรียงลำดับ

### แนวคิด

**`SEARCH ALL`** ใช้เทคนิคการค้นหาแบบ **Binary Search (ทวิภาค)** ซึ่งเร็วกว่า `SEARCH` ธรรมดามาก
โดยเฉพาะกับตารางขนาดใหญ่ เพราะแทนที่จะไล่ตรวจทีละตัว มันจะกระโดดไปตรวจที่ **จุดกึ่งกลาง** ของตารางก่อน
แล้วตัดครึ่งที่ไม่เกี่ยวข้องทิ้งไปทุกครั้ง ทำให้จำนวนรอบการค้นหาน้อยลงมากเมื่อตารางมีขนาดใหญ่
(ตัวอย่างเช่น ตาราง 1,000,000 แถว `SEARCH ALL` ใช้การเปรียบเทียบเพียงประมาณ 20 ครั้งเท่านั้น
เทียบกับ `SEARCH` ที่อาจต้องเทียบสูงสุดถึง 1,000,000 ครั้ง) **แต่มีเงื่อนไขสำคัญ**: ตารางต้องถูก
**เรียงลำดับไว้ก่อนแล้ว** ตามฟิลด์ที่ระบุใน **`ASCENDING KEY IS`** (หรือ `DESCENDING KEY IS`)
ที่ประกาศไว้ตอน `OCCURS` มิฉะนั้นผลการค้นหาจะผิดพลาดโดยไม่มีการเตือนใด ๆ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s8.cob
      *> Purpose : SEARCH ALL (binary search) - requires the
      *>           table to be sorted and an ASCENDING KEY
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SEARCH-ALL-BASIC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ROW OCCURS 5 TIMES
               ASCENDING KEY IS PRODUCT-CODE
               INDEXED BY PROD-IDX.
               10  PRODUCT-CODE  PIC X(4).
               10  PRODUCT-NAME  PIC X(12).

       01  WS-TARGET-CODE       PIC X(4) VALUE "P004".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> The table MUST already be in ascending order by
      *> PRODUCT-CODE for SEARCH ALL to work correctly.
           MOVE "P001" TO PRODUCT-CODE(1)
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE "P002" TO PRODUCT-CODE(2)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE "P003" TO PRODUCT-CODE(3)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE "P004" TO PRODUCT-CODE(4)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE "P005" TO PRODUCT-CODE(5)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)

      *> SEARCH ALL does NOT need PROD-IDX set beforehand: it
      *> jumps straight to the middle of the table and halves
      *> the remaining range each step (binary search).
           SEARCH ALL PRODUCT-ROW
               AT END
                   DISPLAY "Code " WS-TARGET-CODE " not found."
               WHEN PRODUCT-CODE(PROD-IDX) = WS-TARGET-CODE
                   DISPLAY "Found: " PRODUCT-NAME(PROD-IDX)
                       " at position " PROD-IDX
           END-SEARCH

           MOVE "P999" TO WS-TARGET-CODE
           SEARCH ALL PRODUCT-ROW
               AT END
                   DISPLAY "Code " WS-TARGET-CODE " not found."
               WHEN PRODUCT-CODE(PROD-IDX) = WS-TARGET-CODE
                   DISPLAY "Found: " PRODUCT-NAME(PROD-IDX)
                       " at position " PROD-IDX
           END-SEARCH

           STOP RUN.
```

### คำอธิบายโค้ด

- `ASCENDING KEY IS PRODUCT-CODE` ต่อท้าย `OCCURS` บอก COBOL ว่าตารางนี้เรียงจากน้อยไปมากตาม
  `PRODUCT-CODE` — เป็นเงื่อนไขบังคับสำหรับ `SEARCH ALL` (ไม่จำเป็นสำหรับ `SEARCH` ธรรมดา)
- `SEARCH ALL PRODUCT-ROW` **ไม่ต้อง** `SET PROD-IDX TO 1` ก่อนเรียก ต่างจาก `SEARCH` ธรรมดาโดยสิ้นเชิง
  เพราะการค้นหาแบบทวิภาคจะกำหนดจุดเริ่มต้น (กึ่งกลางตาราง) ให้เองโดยอัตโนมัติทุกครั้งที่เรียก
- `WHEN PRODUCT-CODE(PROD-IDX) = WS-TARGET-CODE` ใน `SEARCH ALL` รองรับ**เฉพาะเงื่อนไขความเท่ากัน
  (equality) เท่านั้น** และมีได้เพียง `WHEN` เดียวต่อคำสั่ง (ต่างจาก `SEARCH` ที่มีหลาย `WHEN` ได้)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Found: HEADSET      at position +000000004
Code P999 not found.
```

### ข้อควรระวัง

- **ถ้าตารางไม่ได้เรียงลำดับจริงตามที่ `ASCENDING KEY IS` ระบุไว้ `SEARCH ALL` จะให้ผลลัพธ์ที่ผิดพลาด
  โดยไม่มี error หรือคำเตือนใด ๆ** เพราะอัลกอริทึมทวิภาคอาศัยสมมติฐานเรื่องการเรียงลำดับเป็นพื้นฐาน
  ทั้งหมด นี่คือกับดักที่อันตรายที่สุดของ `SEARCH ALL`
- `SEARCH ALL` มี **เพียงหนึ่ง `WHEN`** เท่านั้นต่อคำสั่ง (จะเห็นวิธีรวมหลายเงื่อนไขด้วย `AND` ใน
  ขั้นตอนถัดไป) ถ้าต้องการหลายเงื่อนไขแบบ `SEARCH` ธรรมดา ต้องใช้ `SEARCH` แทน

### แบบฝึกหัดที่ 168.1

**โจทย์**: จงอธิบายว่าจะเกิดอะไรขึ้นถ้าข้อมูลใน `PRODUCT-CODE` ถูกใส่แบบไม่เรียงลำดับ (เช่น P003, P001,
P005, P002, P004) แต่ยังคงประกาศ `ASCENDING KEY IS PRODUCT-CODE` ไว้

**เฉลยแนวทาง**: `SEARCH ALL` จะยังคง compile และรันได้ตามปกติโดยไม่มี error แต่ผลการค้นหาจะ**ไม่น่าเชื่อถือ**
เพราะอัลกอริทึมทวิภาคจะตัดสินใจว่าจะค้นซีกซ้ายหรือขวาของตารางโดยอ้างอิงสมมติฐานว่าข้อมูลเรียงลำดับแล้ว
ถ้าข้อมูลจริงไม่เรียงลำดับ มันอาจตัดทิ้งซีกที่มีคำตอบที่ถูกต้องไปโดยไม่รู้ตัว ทำให้ได้ผลลัพธ์ "ไม่พบ"
ทั้งที่ข้อมูลมีอยู่จริงในตาราง

---

## ขั้นตอนที่ 169: SEARCH ALL กับเงื่อนไขผสมด้วย AND และข้อจำกัดที่สำคัญ

### แนวคิด

`SEARCH ALL` รองรับการรวมหลายเงื่อนไขด้วย **`AND`** ได้ (เช่น ค้นหาจากคีย์สองฟิลด์พร้อมกัน) โดยฟิลด์
ที่ใช้เปรียบเทียบทั้งหมดต้องเป็นส่วนหนึ่งของ **`ASCENDING KEY IS`** ที่ประกาศไว้ (สามารถระบุคีย์ได้
หลายฟิลด์เรียงเป็นลำดับความสำคัญ เช่น แผนกก่อน แล้วจึงรหัสพนักงาน) **ข้อจำกัดสำคัญที่ต้องจำ**:
`SEARCH ALL` **ไม่รองรับ `OR`, `<`, `>`, หรือเงื่อนไขช่วง (range)** เลย รองรับเฉพาะความเท่ากัน
(`=`) ที่เชื่อมด้วย `AND` เท่านั้น ถ้าต้องการเงื่อนไขแบบอื่น ต้องใช้ `SEARCH` ธรรมดาหรือ `PERFORM` แทน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s9.cob
      *> Purpose : SEARCH ALL with a compound AND condition,
      *>           and why SEARCH ALL cannot use OR or ranges
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SEARCH-ALL-AND-COND.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  EMPLOYEE-TABLE.
           05  EMP-ROW OCCURS 5 TIMES
               ASCENDING KEY IS EMP-DEPT EMP-ID
               INDEXED BY EMP-IDX.
               10  EMP-DEPT      PIC X(6).
               10  EMP-ID        PIC 9(3).
               10  EMP-NAME      PIC X(12).

       01  WS-TARGET-DEPT       PIC X(6) VALUE "SALES".
       01  WS-TARGET-ID         PIC 9(3) VALUE 102.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> The table must be sorted by the FULL key combination
      *> (EMP-DEPT major, EMP-ID minor) for SEARCH ALL to work.
           MOVE "IT"     TO EMP-DEPT(1)
           MOVE 201      TO EMP-ID(1)
           MOVE "MARY"   TO EMP-NAME(1)
           MOVE "IT"     TO EMP-DEPT(2)
           MOVE 205      TO EMP-ID(2)
           MOVE "DAVID"  TO EMP-NAME(2)
           MOVE "SALES"  TO EMP-DEPT(3)
           MOVE 101      TO EMP-ID(3)
           MOVE "JOHN"   TO EMP-NAME(3)
           MOVE "SALES"  TO EMP-DEPT(4)
           MOVE 102      TO EMP-ID(4)
           MOVE "ANNA"   TO EMP-NAME(4)
           MOVE "SALES"  TO EMP-DEPT(5)
           MOVE 103      TO EMP-ID(5)
           MOVE "PAUL"   TO EMP-NAME(5)

      *> SEARCH ALL allows only equality tests joined by AND.
      *> It CANNOT express OR, <, > or ranges: those need a
      *> plain serial SEARCH (or a PERFORM loop) instead.
           SEARCH ALL EMP-ROW
               AT END
                   DISPLAY "No employee matches dept/id combo."
               WHEN EMP-DEPT(EMP-IDX) = WS-TARGET-DEPT
                    AND EMP-ID(EMP-IDX) = WS-TARGET-ID
                   DISPLAY "Found: " EMP-NAME(EMP-IDX)
                       " in " EMP-DEPT(EMP-IDX)
           END-SEARCH

           STOP RUN.
```

### คำอธิบายโค้ด

- `ASCENDING KEY IS EMP-DEPT EMP-ID` ระบุคีย์สองระดับ: เรียงตาม `EMP-DEPT` เป็นหลักก่อน แล้วจึงเรียง
  ตาม `EMP-ID` ภายในแผนกเดียวกัน ข้อมูลตัวอย่างจึงต้องเรียงตามลำดับนี้เป๊ะ (IT ก่อน SALES, และภายใน
  แต่ละแผนกเรียง ID จากน้อยไปมาก)
- `WHEN EMP-DEPT(EMP-IDX) = WS-TARGET-DEPT AND EMP-ID(EMP-IDX) = WS-TARGET-ID` คือการรวมสองเงื่อนไข
  ความเท่ากันด้วย `AND` ซึ่ง `SEARCH ALL` รองรับ เพราะทั้งสองฟิลด์เป็นส่วนหนึ่งของคีย์ที่ประกาศไว้

### ผลลัพธ์ที่ได้จากการรันจริง

```
Found: ANNA         in SALES
```

### ข้อควรระวัง

- **ห้ามเขียน `WHEN ... OR ...` หรือ `WHEN ... < ...` ใน `SEARCH ALL` โดยเด็ดขาด** แม้บางคอมไพเลอร์
  อาจยอม compile ผ่านแต่ผลลัพธ์จะไม่ถูกต้องตามหลักการของ binary search เพราะอัลกอริทึมนี้ออกแบบมา
  สำหรับการเปรียบเทียบความเท่ากันเพื่อตัดสินใจว่าจะค้นซีกซ้ายหรือขวาเท่านั้น
- ลำดับของฟิลด์ใน `ASCENDING KEY IS` (เช่น `EMP-DEPT EMP-ID`) ต้องตรงกับลำดับการเรียงข้อมูลจริงเสมอ
  ถ้าข้อมูลเรียงตาม `EMP-ID` ก่อนแล้วจึง `EMP-DEPT` แต่ประกาศคีย์สลับกัน ผลการค้นหาจะผิดพลาดทันที

### แบบฝึกหัดที่ 169.1

**โจทย์**: จงอธิบายว่าทำไมถ้าต้องการค้นหาพนักงานที่มี `EMP-ID` มากกว่า 102 (ใช้เงื่อนไข `>`)
จึงไม่สามารถใช้ `SEARCH ALL` ได้ และควรใช้อะไรแทน

**เฉลยแนวทาง**: เพราะ `SEARCH ALL` รองรับเฉพาะเงื่อนไขความเท่ากัน (`=`) เท่านั้น ไม่รองรับ `>`, `<`
หรือเงื่อนไขช่วงใด ๆ เนื่องจากอัลกอริทึมทวิภาคใช้ความเท่ากันเป็นเกณฑ์ตัดสินใจตัดครึ่งตาราง
ถ้าต้องการค้นหาแบบช่วง ควรใช้ `SEARCH` ธรรมดา (ซึ่งรองรับ `>` ได้) หรือเขียน `PERFORM VARYING`
ร่วมกับ `IF` ตรวจสอบเงื่อนไขเองโดยตรง

---

## ขั้นตอนที่ 170: โปรแกรมสรุป - ระบบค้นหาสินค้า และการเลือกใช้เครื่องมือให้เหมาะสม

### แนวคิด

ขั้นตอนสุดท้ายของ Part นี้จะรวบยอดทุกอย่างที่เรียนมาเป็นโปรแกรมค้นหาสินค้าขนาดย่อมที่ใช้ `SEARCH ALL`
เพื่อค้นหาสินค้าหลายรายการติดต่อกัน พร้อมทั้งสรุปหลักการเลือกใช้เครื่องมือที่เหมาะสมระหว่าง subscript
ธรรมดา, `SEARCH`, และ `SEARCH ALL`

**หลักการเลือกใช้โดยสรุป**:

| สถานการณ์ | เครื่องมือที่เหมาะสม |
|---|---|
| รู้ตำแหน่งแน่นอนอยู่แล้ว | Subscript หรือ Index ตรง ๆ ไม่ต้องค้นหา |
| ตารางเล็ก (ไม่กี่สิบรายการ) ไม่ได้เรียงลำดับ | `SEARCH` (Serial Search) |
| ตารางใหญ่ที่เรียงลำดับแล้ว ต้องการความเร็ว | `SEARCH ALL` (Binary Search) |
| เงื่อนไขซับซ้อน (`OR`, ช่วง, หลายเงื่อนไขไม่เท่ากัน) | `SEARCH` หรือ `PERFORM VARYING` + `IF` |

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s10.cob
      *> Purpose : Final summary - a small product lookup system
      *>           choosing SEARCH ALL for a sorted table, with
      *>           a fallback message pattern for "not found"
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PRODUCT-LOOKUP-SYSTEM.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ROW OCCURS 8 TIMES
               ASCENDING KEY IS PRODUCT-CODE
               INDEXED BY PROD-IDX.
               10  PRODUCT-CODE  PIC X(4).
               10  PRODUCT-NAME  PIC X(12).
               10  PRODUCT-PRICE PIC 9(5)V99.

       01  WS-LOOKUP-TABLE.
           05  WS-LOOKUP-CODE PIC X(4) OCCURS 3 TIMES.
       01  WS-LOOKUP-IDX        PIC 9 VALUE 1.
       01  WS-PRICE-EDIT        PIC $$,$$9.99.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Data already sorted ascending by PRODUCT-CODE, a
      *> requirement for SEARCH ALL to give correct results.
           MOVE "P001" TO PRODUCT-CODE(1)
           MOVE "KEYBOARD"   TO PRODUCT-NAME(1)
           MOVE 590.00 TO PRODUCT-PRICE(1)
           MOVE "P002" TO PRODUCT-CODE(2)
           MOVE "MOUSE"      TO PRODUCT-NAME(2)
           MOVE 250.00 TO PRODUCT-PRICE(2)
           MOVE "P003" TO PRODUCT-CODE(3)
           MOVE "MONITOR"    TO PRODUCT-NAME(3)
           MOVE 4500.00 TO PRODUCT-PRICE(3)
           MOVE "P004" TO PRODUCT-CODE(4)
           MOVE "HEADSET"    TO PRODUCT-NAME(4)
           MOVE 890.00 TO PRODUCT-PRICE(4)
           MOVE "P005" TO PRODUCT-CODE(5)
           MOVE "WEBCAM"     TO PRODUCT-NAME(5)
           MOVE 1200.00 TO PRODUCT-PRICE(5)
           MOVE "P006" TO PRODUCT-CODE(6)
           MOVE "SPEAKER"    TO PRODUCT-NAME(6)
           MOVE 990.00 TO PRODUCT-PRICE(6)
           MOVE "P007" TO PRODUCT-CODE(7)
           MOVE "USB-HUB"    TO PRODUCT-NAME(7)
           MOVE 350.00 TO PRODUCT-PRICE(7)
           MOVE "P008" TO PRODUCT-CODE(8)
           MOVE "MOUSEPAD"   TO PRODUCT-NAME(8)
           MOVE 120.00 TO PRODUCT-PRICE(8)

           MOVE "P005" TO WS-LOOKUP-CODE(1)
           MOVE "P999" TO WS-LOOKUP-CODE(2)
           MOVE "P001" TO WS-LOOKUP-CODE(3)

           DISPLAY "===== Product Lookup System ====="
           PERFORM VARYING WS-LOOKUP-IDX FROM 1 BY 1
                   UNTIL WS-LOOKUP-IDX > 3
               SEARCH ALL PRODUCT-ROW
                   AT END
                       DISPLAY "  " WS-LOOKUP-CODE(WS-LOOKUP-IDX)
                           " -> NOT FOUND"
                   WHEN PRODUCT-CODE(PROD-IDX)
                        = WS-LOOKUP-CODE(WS-LOOKUP-IDX)
                       MOVE PRODUCT-PRICE(PROD-IDX)
                           TO WS-PRICE-EDIT
                       DISPLAY "  " WS-LOOKUP-CODE(WS-LOOKUP-IDX)
                           " -> " PRODUCT-NAME(PROD-IDX)
                           " (" WS-PRICE-EDIT ")"
               END-SEARCH
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- โปรแกรมนี้มีรายการรหัสสินค้าที่ต้องการค้นหา 3 รายการเก็บไว้ใน `WS-LOOKUP-TABLE` แล้วใช้
  `PERFORM VARYING` วนเรียก `SEARCH ALL` ทีละรายการ
- สังเกตว่าไม่ต้อง `SET PROD-IDX` ก่อนเรียก `SEARCH ALL` ในแต่ละรอบของลูปเลย เพราะ `SEARCH ALL`
  เริ่มค้นหาใหม่จากกึ่งกลางตารางเสมอในทุกครั้งที่ถูกเรียก ไม่ขึ้นกับค่า index ก่อนหน้า
- แต่ละรายการที่พบจะถูกจัดรูปแบบราคาด้วยฟิลด์ numeric-edited (`WS-PRICE-EDIT`) ก่อนแสดงผล ตามเทคนิค
  ที่เรียนไปแล้วใน Part 019

### ผลลัพธ์ที่ได้จากการรันจริง

```
===== Product Lookup System =====
  P005 -> WEBCAM       ($1,200.00)
  P999 -> NOT FOUND
  P001 -> KEYBOARD     (  $590.00)
```

### ข้อควรระวัง

- ถ้าตารางสินค้าในระบบจริงมีการเพิ่ม/ลบ/แก้ไขรายการบ่อย ๆ ต้อง**เรียงลำดับใหม่ทุกครั้ง**ก่อนเรียก
  `SEARCH ALL` (เช่นด้วย `SORT` ซึ่งจะเรียนใน Part 027) มิฉะนั้นผลการค้นหาจะผิดพลาดแบบเงียบ ๆ
- การเลือกระหว่าง `SEARCH` กับ `SEARCH ALL` ควรพิจารณาทั้งขนาดของตารางและความถี่ในการค้นหา
  ถ้าตารางมีขนาดเล็กมาก (ไม่กี่รายการ) ความแตกต่างด้านประสิทธิภาพแทบไม่มีผล การเลือกใช้ `SEARCH`
  ธรรมดาที่ยืดหยุ่นกว่าอาจเหมาะสมกว่าในกรณีนั้น

### แบบฝึกหัดสรุปรวมที่ 170.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น โดยเพิ่มการนับจำนวนรายการที่ค้นหา "ไม่พบ" ทั้งหมด แล้วแสดงสรุปท้ายรายงาน

**เฉลยแนวทาง**: ประกาศตัวแปรใหม่ `01 WS-NOT-FOUND-COUNT PIC 9 VALUE 0.` แล้วในส่วน `AT END` ของ
`SEARCH ALL` เพิ่มคำสั่ง `ADD 1 TO WS-NOT-FOUND-COUNT` ต่อจาก `DISPLAY` เดิม จากนั้นหลังจบ
`PERFORM VARYING` ทั้งหมด ให้เพิ่ม `DISPLAY "Total not found: " WS-NOT-FOUND-COUNT` เพื่อสรุปผลรวม
ผลลัพธ์ที่ได้จากข้อมูลตัวอย่างนี้จะแสดง "Total not found: 1" เนื่องจากมีเพียง P999 เท่านั้นที่ไม่พบ

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้เรื่อง Subscript, Index และ SEARCH/SEARCH ALL อย่างครบถ้วน ได้แก่:

- ทบทวนการใช้ subscript แบบ literal และแบบตัวแปรในการเข้าถึงตาราง
- ความเสี่ยงของ subscript ที่ไม่ตรวจสอบขอบเขต และแบบแผนการ์ด (guard) ที่ปลอดภัย
- การประกาศ index ด้วย `INDEXED BY` และการกำหนดค่าด้วย `SET`
- การใช้ index ร่วมกับ `PERFORM VARYING` และ `SET ... UP BY / DOWN BY`
- การเปรียบเทียบ subscript กับ index อย่างละเอียดในทุกมิติ
- `SEARCH` (Serial Search) พื้นฐาน พร้อม `AT END` และ `WHEN`
- `SEARCH` กับหลายเงื่อนไข `WHEN` และวลี `SEARCH ... VARYING`
- `SEARCH ALL` (Binary Search) และเงื่อนไขบังคับเรื่องการเรียงลำดับด้วย `ASCENDING KEY IS`
- `SEARCH ALL` กับเงื่อนไขผสมด้วย `AND` และข้อจำกัดสำคัญ (ไม่รองรับ `OR`/ช่วง)
- โปรแกรมสรุประบบค้นหาสินค้า พร้อมหลักการเลือกใช้เครื่องมือค้นหาที่เหมาะสมกับสถานการณ์

ทักษะเรื่อง index และ `SEARCH`/`SEARCH ALL` นี้เป็นพื้นฐานสำคัญมากที่จะถูกนำไปใช้ต่อยอดใน **Part 018**
ที่เราจะขยายแนวคิดตารางไปสู่ **ตารางหลายมิติ (Multi-dimensional Tables)** ซึ่งต้องใช้ index หลายตัว
พร้อมกันในการเข้าถึงข้อมูลแต่ละแกน

**[← กลับไป Part 016](part-016-tables-occurs.md)** |
**[ไปยัง Part 018: ตารางหลายมิติ (Multi-dimensional Tables) →](part-018-multidim-tables.md)**
