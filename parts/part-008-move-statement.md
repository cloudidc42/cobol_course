# Part 008: MOVE Statement และกฎการย้ายข้อมูล (MOVE Rules) (ขั้นตอนที่ 71–80)

## คำนำของ Part นี้

ใน Part 007 เราได้เรียนรู้วิธีนำข้อมูลเข้า-ออกจากโปรแกรมด้วย `DISPLAY` และ `ACCEPT` แล้ว
คำถามถัดไปที่ตามมาตามธรรมชาติคือ: แล้วเราจะ **ย้ายข้อมูลจากตัวแปรหนึ่งไปยังอีกตัวแปรหนึ่งภายในโปรแกรม**
ได้อย่างไร? นี่คือหน้าที่ของคำสั่ง **`MOVE`** ซึ่งเป็นหนึ่งในคำสั่งที่ถูกใช้บ่อยที่สุดในโปรแกรม COBOL ทุกโปรแกรม

แม้ `MOVE` จะดูเหมือนคำสั่งง่าย ๆ ที่ทำหน้าที่คล้าย `=` (assignment) ในภาษาอื่น แต่ความจริงแล้ว `MOVE`
มี **กฎเบื้องหลังที่ซับซ้อนกว่าที่คิด** เพราะ COBOL ต้องจัดการกับข้อมูลหลายชนิด (ตัวเลข ตัวอักษร
กลุ่มข้อมูล) ที่มีขนาดไม่เท่ากันเสมอ การไม่เข้าใจกฎเหล่านี้คือสาเหตุอันดับต้น ๆ ของบั๊กที่พบในโปรแกรม
COBOL มือใหม่ — ข้อมูลหายไปเงียบ ๆ โดยไม่มี error ใด ๆ แจ้งเตือน! Part นี้จะพาคุณเจาะลึกทุกกฎของ `MOVE`
ตั้งแต่พื้นฐานไปจนถึงกับดักที่มือโปรก็ยังเคยพลาด

---

## ขั้นตอนที่ 71: MOVE พื้นฐาน — การย้ายข้อมูลตัวอักษร (Alphanumeric MOVE)

### แนวคิด

รูปแบบพื้นฐานที่สุดของคำสั่ง `MOVE` คือ:

```
MOVE <source> TO <destination>
```

โดย `<source>` อาจเป็นค่าคงที่ (literal) หรือชื่อตัวแปรก็ได้ ส่วน `<destination>` ต้องเป็นชื่อตัวแปรเสมอ
ทิศทางของคำสั่งนี้สำคัญมาก: **ข้อมูลไหลจากขวาไปซ้ายเสมอ** (จาก TO ไปหาต้นประโยค) ต่างจาก `=` ในภาษาอื่น
ที่มักไหลจากขวาไปซ้ายเหมือนกันแต่เขียนสลับตำแหน่งกัน นักเรียนที่มาจากภาษาอื่นจึงต้องระวังเรื่องนี้เป็นพิเศษ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-BASIC.
       AUTHOR. STUDENT-NAME.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST-NAME       PIC X(10).
       01  WS-LAST-NAME        PIC X(10)   VALUE "SOMCHAI".
       01  WS-CITY             PIC X(15).

       PROCEDURE DIVISION.
           MOVE "JOHN" TO WS-FIRST-NAME
           MOVE WS-LAST-NAME TO WS-CITY
           DISPLAY "FIRST NAME : [" WS-FIRST-NAME "]"
           DISPLAY "LAST NAME  : [" WS-LAST-NAME "]"
           DISPLAY "CITY       : [" WS-CITY "]"
           STOP RUN.
```

### อธิบายโค้ด

- `MOVE "JOHN" TO WS-FIRST-NAME` — ย้ายค่าคงที่ (literal) `"JOHN"` เข้าไปในตัวแปร `WS-FIRST-NAME`
  ที่มีขนาด `PIC X(10)` เนื่องจาก "JOHN" มีเพียง 4 ตัวอักษร ส่วนที่เหลืออีก 6 ช่องจะถูกเติมด้วย
  **ช่องว่าง (space)** ทางด้านขวาโดยอัตโนมัติ (เรียกว่า left-justified, space-filled — พฤติกรรมมาตรฐาน
  ของฟิลด์ตัวอักษร)
- `MOVE WS-LAST-NAME TO WS-CITY` — ย้ายค่าจากตัวแปรหนึ่งไปยังอีกตัวแปรหนึ่ง สังเกตว่า `WS-CITY` มีขนาด
  ใหญ่กว่า `WS-LAST-NAME` (15 เทียบกับ 10) ผลลัพธ์คือค่าทั้งหมดของ `WS-LAST-NAME` ถูกคัดลอกมา
  และช่องที่เหลือถูกเติมด้วยช่องว่าง

### ผลลัพธ์ที่ได้จากการรันจริง

```
FIRST NAME : [JOHN      ]
LAST NAME  : [SOMCHAI   ]
CITY       : [SOMCHAI        ]
```

สังเกตวงเล็บ `[ ]` ที่เราใส่ล้อมตัวแปรใน `DISPLAY` — เทคนิคนี้ช่วยให้เห็นช่องว่างที่ถูกเติมเข้ามาชัดเจน
ซึ่งเป็นเทคนิคที่มีประโยชน์มากเวลา debug ปัญหาเรื่องความยาวของฟิลด์

### ข้อควรระวัง

- `MOVE` ของฟิลด์ตัวอักษร (alphanumeric) จะจัดข้อมูลชิดซ้ายเสมอ (left-justified) ต่างจากฟิลด์ตัวเลข
  ที่จัดชิดขวา (right-justified) ซึ่งเราจะเห็นในขั้นตอนถัดไป
- ทิศทางของ `MOVE` คือ **จากซ้ายไปขวาตามที่เขียน** (`MOVE A TO B` หมายถึง "เอาค่า A ไปใส่ B")
  อย่าสับสนกับภาษาที่ใช้ `=` ซึ่งมักเขียน `B = A`

### แบบฝึกหัดที่ 71.1

**โจทย์**: จงเขียนโปรแกรมที่ประกาศตัวแปร `WS-COUNTRY PIC X(20)` แล้ว `MOVE "THAILAND"` เข้าไป
จากนั้น `DISPLAY` ค่าด้วยวงเล็บล้อมรอบเพื่อดูว่ามีช่องว่างเติมเข้ามากี่ช่อง

**เฉลย**: "THAILAND" มี 8 ตัวอักษร เมื่อ `MOVE` เข้าฟิลด์ `PIC X(20)` จะได้ผลลัพธ์
`[THAILAND            ]` คือมีช่องว่างเติมเข้ามาอีก 12 ช่อง (20 - 8 = 12) ทางด้านขวาของคำ

---

## ขั้นตอนที่ 72: MOVE ข้อมูลตัวเลข — การจัดตำแหน่งชิดขวา (Right-Justified)

### แนวคิด

ฟิลด์ตัวเลข (numeric, `PIC 9...`) มีพฤติกรรมการ `MOVE` ที่ตรงข้ามกับฟิลด์ตัวอักษรโดยสิ้นเชิง:
**ข้อมูลจะถูกจัดชิดขวาเสมอ (right-justified)** และช่องที่เหลือทางซ้ายจะถูกเติมด้วย **เลข 0**
ไม่ใช่ช่องว่าง เหตุผลเบื้องหลังคือ COBOL ออกแบบมาให้ตัวเลขพร้อมสำหรับการคำนวณเสมอ
เลข 0 นำหน้าไม่กระทบค่าทางคณิตศาสตร์ ในขณะที่ช่องว่างนำหน้าจะทำให้ค่าไม่ใช่ตัวเลขที่ถูกต้อง

นอกจากนี้ เมื่อฟิลด์ปลายทางมีขนาด **เล็กกว่า** ฟิลด์ต้นทาง ตัวเลขที่เกินขอบเขตทาง **ซ้ายมือ (หลักสูง)**
จะถูกตัดทิ้งไปเงียบ ๆ โดยไม่มี error หรือ warning ใด ๆ — นี่คือกับดักสำคัญที่เราจะเจาะลึกใน ขั้นตอนที่ 79

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-NUMERIC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-QUANTITY         PIC 9(5)    VALUE 250.
       01  WS-BIG-FIELD        PIC 9(8).
       01  WS-SMALL-FIELD      PIC 9(3).

       PROCEDURE DIVISION.
           MOVE WS-QUANTITY TO WS-BIG-FIELD
           DISPLAY "QUANTITY (5 digits)   : " WS-QUANTITY
           DISPLAY "BIG-FIELD (8 digits)  : " WS-BIG-FIELD

           MOVE WS-QUANTITY TO WS-SMALL-FIELD
           DISPLAY "SMALL-FIELD (3 digits): " WS-SMALL-FIELD
           STOP RUN.
```

### อธิบายโค้ด

- `WS-QUANTITY` มีค่า 250 เก็บในฟิลด์ 5 หลัก จึงแสดงผลเป็น `00250` (เติม 0 นำหน้า 2 ตัว)
- เมื่อ `MOVE` ไปยัง `WS-BIG-FIELD` (8 หลัก) ค่าจะขยายและเติม 0 นำหน้าเพิ่มเป็น `00000250`
- เมื่อ `MOVE` ไปยัง `WS-SMALL-FIELD` (3 หลัก) ซึ่งพอดีกับค่า 250 จึงได้ผลลัพธ์ `250` ปกติ
  (กรณีนี้ไม่มีการตัดข้อมูลเพราะค่าพอดีกับขนาดฟิลด์)

### ผลลัพธ์ที่ได้จากการรันจริง

```
QUANTITY (5 digits)   : 00250
BIG-FIELD (8 digits)  : 00000250
SMALL-FIELD (3 digits): 250
```

### ข้อควรระวัง

- อย่าตกใจเมื่อเห็นเลข 0 นำหน้าตัวเลขใน COBOL — นี่คือพฤติกรรมปกติของฟิลด์ `PIC 9` แบบไม่มี edit
  characters (เราจะเรียนวิธีทำให้ตัวเลขแสดงผลสวยงามแบบไม่มี 0 นำหน้าด้วย PICTURE Edited ใน ขั้นตอนที่ 77)
- การจัดชิดขวาของตัวเลขเทียบกับการจัดชิดซ้ายของตัวอักษร เป็นกฎที่ **สลับกัน** และเป็นสิ่งที่ผู้เริ่มต้น
  จำสับสนกันบ่อยที่สุด — ให้จำหลักง่าย ๆ ว่า **"ตัวอักษรชิดซ้าย เติมว่างข้างขวา, ตัวเลขชิดขวา เติมศูนย์ข้างซ้าย"**

### แบบฝึกหัดที่ 72.1

**โจทย์**: หากมีตัวแปร `WS-SOURCE PIC 9(4) VALUE 99` และ `WS-TARGET PIC 9(6)` แล้วสั่ง
`MOVE WS-SOURCE TO WS-TARGET` ผลลัพธ์ของ `WS-TARGET` เมื่อ `DISPLAY` จะเป็นอะไร?

**เฉลย**: `000099` — ค่า 99 (เก็บเป็น 0099 ในฟิลด์ 4 หลัก) เมื่อขยายไปฟิลด์ 6 หลัก จะเติม 0 นำหน้า
เพิ่มอีก 2 ตัว รวมเป็น `000099`

---

## ขั้นตอนที่ 73: กฎการจัดตำแหน่งจุดทศนิยม (Decimal Point Alignment)

### แนวคิด

เมื่อฟิลด์ตัวเลขมีจุดทศนิยม (`V` ใน PICTURE clause) COBOL จะ **จัดตำแหน่งจุดทศนิยมให้ตรงกันเสมอ**
โดยอัตโนมัติ ไม่ว่าฟิลด์ต้นทางและปลายทางจะมีจำนวนหลักก่อน/หลังจุดทศนิยมต่างกันแค่ไหนก็ตาม
กฎนี้สำคัญมากเพราะการเงินและบัญชีต้องพึ่งพาความแม่นยำของทศนิยม 100%

หลักการคือ: COBOL จะ **"จัดแนวจุดทศนิยมในใจ" ก่อน** จากนั้นจึงคัดลอกตัวเลขแต่ละหลักไปตามตำแหน่งที่ตรงกัน
ถ้าฟิลด์ปลายทางมีทศนิยมน้อยกว่า ตัวเลขหลักท้าย ๆ จะถูก **ตัดทิ้ง (truncate) ไม่ใช่ปัดเศษ (round)**

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-DECIMAL.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE            PIC 9(3)V99  VALUE 125.5.
       01  WS-PRICE-4DEC       PIC 9(3)V9999.
       01  WS-PRICE-NODEC      PIC 9(3)V99  VALUE 7.
       01  WS-WHOLE-ONLY       PIC 9(5).

       PROCEDURE DIVISION.
           DISPLAY "ORIGINAL PRICE (3V99) : " WS-PRICE

           MOVE WS-PRICE TO WS-PRICE-4DEC
           DISPLAY "MOVED TO 3V9999       : " WS-PRICE-4DEC

           MOVE 7 TO WS-WHOLE-ONLY
           DISPLAY "WHOLE NUMBER 7 -> 5   : " WS-WHOLE-ONLY

           MOVE 42.999 TO WS-PRICE
           DISPLAY "42.999 MOVED TO 3V99  : " WS-PRICE
           STOP RUN.
```

### อธิบายโค้ด

- `WS-PRICE PIC 9(3)V99 VALUE 125.5` เก็บค่าเป็น `125.50` (V ไม่ใช่ตัวอักษรที่แสดงผล
  แต่เป็นตำแหน่งจุดทศนิยมโดยนัย — เรียนไปแล้วใน Part 006)
- `MOVE WS-PRICE TO WS-PRICE-4DEC` — ฟิลด์ปลายทางมีทศนิยม 4 หลัก มากกว่าต้นทาง (2 หลัก) COBOL
  จึงจัดจุดทศนิยมให้ตรงกันแล้วเติม 0 ในหลักทศนิยมที่เหลือ ได้ `125.5000`
- `MOVE 42.999 TO WS-PRICE` — ฟิลด์ปลายทางมีทศนิยมแค่ 2 หลัก แต่ค่าที่ MOVE เข้ามามี 3 หลักทศนิยม
  (.999) ผลลัพธ์คือ **ตัดหลักที่ 3 ทิ้งไปเลย (.9) ไม่ปัดเศษขึ้นเป็น 43.00** ได้ผลเป็น `042.99`

### ผลลัพธ์ที่ได้จากการรันจริง

```
ORIGINAL PRICE (3V99) : 125.50
MOVED TO 3V9999       : 125.5000
WHOLE NUMBER 7 -> 5   : 00007
42.999 MOVED TO 3V99  : 042.99
```

### ข้อควรระวัง

- **`MOVE` ไม่เคยปัดเศษ (round) เสมอตัดทิ้ง (truncate)** หากต้องการปัดเศษ ต้องใช้คำสั่งเลขคณิต
  พร้อม `ROUNDED` phrase ซึ่งจะสอนใน Part 009 (ขั้นตอนที่ 86)
- 42.999 ถูกตัดเป็น 42.99 ไม่ใช่ 43.00 — เป็นความผิดพลาดที่พบบ่อยมากเมื่อโปรแกรมเมอร์คาดหวังว่า
  ระบบจะปัดเศษให้อัตโนมัติ

### แบบฝึกหัดที่ 73.1

**โจทย์**: ถ้า `WS-A PIC 9(2)V9 VALUE 9.9` และ `MOVE 3.14159 TO WS-A` ผลลัพธ์จะเป็นเท่าไร?

**เฉลย**: `03.1` — เพราะฟิลด์ `WS-A` มีทศนิยมได้แค่ 1 หลัก ตัวเลข 3.14159 จะถูกตัดเหลือ 3.1
(ตัดหลักที่ 2 เป็นต้นไปทิ้งทั้งหมด ไม่มีการปัดเศษ) และเติม 0 นำหน้าให้ครบ 2 หลักก่อนจุดทศนิยม

---

## ขั้นตอนที่ 74: MOVE กับข้อมูลตัวอักษร — การตัดและเติมช่องว่าง (Truncation & Padding)

### แนวคิด

เมื่อ `MOVE` ข้อมูลตัวอักษร (alphanumeric) ระหว่างฟิลด์ที่มีขนาดต่างกัน กฎจะตรงไปตรงมากว่าตัวเลข:

- ถ้าฟิลด์ปลายทาง **เล็กกว่า** ต้นทาง: ตัวอักษรส่วนเกิน **ทางขวา** จะถูกตัดทิ้ง
- ถ้าฟิลด์ปลายทาง **ใหญ่กว่า** ต้นทาง: ช่องที่เหลือทางขวาจะถูกเติมด้วยช่องว่าง

หลักการคือ "ชิดซ้ายเสมอ" — ตัวอักษรตัวแรกของข้อมูลจะอยู่ที่ตำแหน่งซ้ายสุดของฟิลด์ปลายทางเสมอ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-ALPHA-TRUNC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LONG-TEXT        PIC X(20) VALUE "PROGRAMMING IN COBOL".
       01  WS-SHORT-FIELD      PIC X(10).
       01  WS-WIDE-FIELD       PIC X(20).
       01  WS-SHORT-SOURCE     PIC X(3)  VALUE "HI".

       PROCEDURE DIVISION.
           DISPLAY "SOURCE (20 chars) : [" WS-LONG-TEXT "]"

           MOVE WS-LONG-TEXT TO WS-SHORT-FIELD
           DISPLAY "TO 10-CHAR FIELD  : [" WS-SHORT-FIELD "]"

           MOVE WS-SHORT-SOURCE TO WS-WIDE-FIELD
           DISPLAY "TO 20-CHAR FIELD  : [" WS-WIDE-FIELD "]"
           STOP RUN.
```

### อธิบายโค้ด

- `WS-LONG-TEXT` เก็บข้อความ "PROGRAMMING IN COBOL" (20 ตัวอักษรพอดี)
- `MOVE WS-LONG-TEXT TO WS-SHORT-FIELD` — ฟิลด์ปลายทางมีแค่ 10 ช่อง ข้อความจึงถูกตัดเหลือ 10
  ตัวอักษรแรก คือ "PROGRAMMIN" (ตัด "G IN COBOL" ทิ้งไปเงียบ ๆ)
- `MOVE WS-SHORT-SOURCE TO WS-WIDE-FIELD` — "HI" (3 ช่อง มีค่าจริง 2 ตัวและ 1 ช่องว่าง) ถูกขยาย
  ไปฟิลด์ 20 ช่อง และเติมช่องว่างที่เหลือ

### ผลลัพธ์ที่ได้จากการรันจริง

```
SOURCE (20 chars) : [PROGRAMMING IN COBOL]
TO 10-CHAR FIELD  : [PROGRAMMIN]
TO 20-CHAR FIELD  : [HI                  ]
```

### ข้อควรระวัง

- **การตัดข้อความทิ้งไม่มี warning ใด ๆ ระหว่าง compile หรือ runtime** ต้องตรวจสอบขนาดฟิลด์
  ให้เหมาะสมเองเสมอ โดยเฉพาะเมื่อรับข้อมูลจากผู้ใช้หรือไฟล์ภายนอกที่ความยาวไม่แน่นอน
- ปัญหานี้เกิดขึ้นบ่อยมากในระบบจริง เช่น ชื่อลูกค้าที่ยาวเกินฟิลด์ที่ออกแบบไว้ ทำให้ข้อมูลบางส่วนหายไป
  โดยไม่มีใครสังเกตจนกว่าจะมีคนมาตรวจสอบรายงาน

### แบบฝึกหัดที่ 74.1

**โจทย์**: ถ้า `WS-CODE PIC X(6) VALUE "ABCDEFGH"` (ระบุ VALUE ยาวกว่าฟิลด์) จะเกิดอะไรขึ้น
เมื่อคอมไพล์?

**เฉลย**: คอมไพเลอร์ส่วนใหญ่ (รวมถึง GnuCOBOL) จะแจ้ง warning หรือ error ว่าค่า VALUE ยาวเกินขนาด
ฟิลด์ที่ประกาศไว้ (`value size exceeds data size`) เพราะการกำหนด VALUE ตอนประกาศตัวแปรต่างจาก
การ `MOVE` ตอน runtime — คอมไพเลอร์ตรวจสอบ VALUE ได้ตั้งแต่ compile time แต่ตรวจสอบ MOVE runtime
ไม่ได้เพราะค่าจริงยังไม่ทราบตอนคอมไพล์

---

## ขั้นตอนที่ 75: MOVE CORRESPONDING — ย้ายข้อมูลตามชื่อฟิลด์ที่ตรงกัน

### แนวคิด

เมื่อเรามีกลุ่มข้อมูล (group item) สองกลุ่มที่มีฟิลด์ย่อยชื่อ **ตรงกันบางส่วนหรือทั้งหมด**
คำสั่ง `MOVE CORRESPONDING` (หรือย่อว่า `MOVE CORR`) จะช่วยย้ายเฉพาะฟิลด์ที่ชื่อตรงกันให้อัตโนมัติ
โดยไม่ต้องเขียน `MOVE` ทีละฟิลด์ ซึ่งช่วยลดโค้ดได้มากเมื่อโครงสร้างข้อมูลมีฟิลด์จำนวนมาก

ข้อสำคัญ: ฟิลด์ที่ชื่อไม่ตรงกันจะถูกข้ามไปเฉย ๆ (ไม่ error) และการเทียบชื่อจะพิจารณาจาก **ชื่อฟิลด์ย่อย
ที่อยู่ในระดับเดียวกัน** ไม่ใช่ตำแหน่งลำดับ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-CORRESPONDING-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EMPLOYEE-INPUT.
           05  EMP-ID          PIC 9(5)  VALUE 10023.
           05  EMP-NAME        PIC X(15) VALUE "WICHAI SUKSAN".
           05  EMP-SALARY      PIC 9(7)V99 VALUE 35000.00.
           05  EMP-EXTRA-FLAG  PIC X      VALUE "Y".

       01  WS-EMPLOYEE-OUTPUT.
           05  EMP-ID          PIC 9(5).
           05  EMP-NAME        PIC X(15).
           05  EMP-SALARY      PIC 9(7)V99.

       PROCEDURE DIVISION.
           MOVE CORRESPONDING WS-EMPLOYEE-INPUT TO WS-EMPLOYEE-OUTPUT
           DISPLAY "OUTPUT ID     : " EMP-ID OF WS-EMPLOYEE-OUTPUT
           DISPLAY "OUTPUT NAME   : " EMP-NAME OF WS-EMPLOYEE-OUTPUT
           DISPLAY "OUTPUT SALARY : " EMP-SALARY OF WS-EMPLOYEE-OUTPUT
           STOP RUN.
```

### อธิบายโค้ด

- ทั้งสองกลุ่ม `WS-EMPLOYEE-INPUT` และ `WS-EMPLOYEE-OUTPUT` มีฟิลด์ย่อยชื่อ `EMP-ID`, `EMP-NAME`,
  `EMP-SALARY` ตรงกัน แต่ `WS-EMPLOYEE-INPUT` มีฟิลด์เพิ่มเติมคือ `EMP-EXTRA-FLAG` ที่ไม่มีในปลายทาง
- `MOVE CORRESPONDING WS-EMPLOYEE-INPUT TO WS-EMPLOYEE-OUTPUT` จะย้ายเฉพาะ 3 ฟิลด์ที่ชื่อตรงกัน
  โดยอัตโนมัติ เทียบเท่ากับการเขียน `MOVE EMP-ID OF WS-EMPLOYEE-INPUT TO EMP-ID OF
  WS-EMPLOYEE-OUTPUT` ทีละบรรทัดสามครั้ง แต่สั้นกว่ามาก ส่วน `EMP-EXTRA-FLAG` ถูกข้ามไปเพราะไม่มี
  ฟิลด์ชื่อเดียวกันในปลายทาง
- สังเกตการใช้ `EMP-ID OF WS-EMPLOYEE-OUTPUT` — เนื่องจากทั้งสองกลุ่มมีฟิลด์ย่อยชื่อซ้ำกัน
  (`EMP-ID` ปรากฏสองที่) เราต้องใช้ `OF` เพื่อระบุว่าหมายถึงฟิลด์ในกลุ่มไหน (คุณสมบัตินี้เรียกว่า
  "qualification")

### ผลลัพธ์ที่ได้จากการรันจริง

```
OUTPUT ID     : 10023
OUTPUT NAME   : WICHAI SUKSAN
OUTPUT SALARY : 0035000.00
```

### ข้อควรระวัง

- `MOVE CORRESPONDING` เทียบชื่อฟิลด์แบบตรงตัวทุกตัวอักษร (case-insensitive แต่ต้องสะกดตรงกัน)
  หากตั้งชื่อผิดเพียงตัวเดียว ฟิลด์นั้นจะถูกข้ามไปเงียบ ๆ โดยไม่มี error แจ้งเตือน
- ฟิลด์ที่ชื่อซ้ำกันในหลายกลุ่มต้องใช้ `OF` หรือ `IN` ระบุกลุ่มเสมอเมื่ออ้างอิงนอกบริบทที่ชัดเจน
  ไม่เช่นนั้นคอมไพเลอร์จะ error ว่า "ambiguous reference" (อ้างอิงกำกวม)

### แบบฝึกหัดที่ 75.1

**โจทย์**: หากปลายทางมีฟิลด์ชื่อ `EMP-SALARY` แต่ต้นทางสะกดว่า `EMP-SALARIES` (มี S ต่อท้าย)
`MOVE CORRESPONDING` จะย้ายค่าฟิลด์นี้หรือไม่?

**เฉลย**: ไม่ย้าย เพราะชื่อไม่ตรงกันทุกตัวอักษร (`EMP-SALARY` ≠ `EMP-SALARIES`) COBOL จะถือว่าเป็น
คนละฟิลด์กันโดยสิ้นเชิงและข้ามไปเงียบ ๆ นี่คือเหตุผลที่ทีมพัฒนาต้องตั้งชื่อฟิลด์ให้สอดคล้องกันอย่าง
เคร่งครัดเมื่อวางแผนจะใช้ `MOVE CORRESPONDING`

---

## ขั้นตอนที่ 76: Group MOVE เทียบกับ Elementary MOVE

### แนวคิด

ใน COBOL ตัวแปรแบ่งเป็น 2 แบบคือ **Elementary item** (ฟิลด์เดี่ยวที่มี PICTURE ของตัวเอง) และ
**Group item** (กลุ่มที่รวมฟิลด์ย่อยหลายตัวไว้ด้วยกัน ไม่มี PICTURE ของตัวเอง)

จุดที่น่าสนใจคือ เมื่อเรา `MOVE` ระดับ **group ทั้งกลุ่ม** ไปยังอีก group หนึ่ง COBOL จะ**ไม่สนใจ
PICTURE clause ของฟิลด์ย่อยภายในเลยแม้แต่น้อย** แต่จะปฏิบัติต่อข้อมูลทั้งกลุ่มเป็น **ก้อนตัวอักษร
(alphanumeric) ก้อนเดียว** และใช้กฎการ MOVE แบบตัวอักษร (ชิดซ้าย เติมช่องว่าง) เสมอ

ในทางกลับกัน ถ้า `MOVE` ทีละฟิลด์ย่อย (elementary) จะใช้กฎตามชนิดข้อมูลจริงของแต่ละฟิลด์
(ตัวเลขจัดชิดขวาเติมศูนย์ ตามที่เรียนในขั้นตอนที่ 72)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-GROUP-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE-IN.
           05  WS-YEAR-IN      PIC 9(4) VALUE 2026.
           05  WS-MONTH-IN     PIC 9(2) VALUE 9.
           05  WS-DAY-IN       PIC 9(2) VALUE 26.

       01  WS-DATE-OUT.
           05  WS-YEAR-OUT     PIC 9(4).
           05  WS-MONTH-OUT    PIC 9(2).
           05  WS-DAY-OUT      PIC 9(2).

       PROCEDURE DIVISION.
           DISPLAY "GROUP MOVE (treated as one big alphanumeric field):"
           MOVE WS-DATE-IN TO WS-DATE-OUT
           DISPLAY "YEAR  : " WS-YEAR-OUT
           DISPLAY "MONTH : " WS-MONTH-OUT
           DISPLAY "DAY   : " WS-DAY-OUT

           DISPLAY " "
           DISPLAY "ELEMENTARY MOVE (field by field, numeric rules):"
           MOVE WS-YEAR-IN  TO WS-YEAR-OUT
           MOVE WS-MONTH-IN TO WS-MONTH-OUT
           MOVE WS-DAY-IN   TO WS-DAY-OUT
           DISPLAY "YEAR  : " WS-YEAR-OUT
           DISPLAY "MONTH : " WS-MONTH-OUT
           DISPLAY "DAY   : " WS-DAY-OUT
           STOP RUN.
```

### อธิบายโค้ด

- `MOVE WS-DATE-IN TO WS-DATE-OUT` เป็น **group MOVE** เนื่องจากทั้งสองฝั่งมีขนาดรวมเท่ากันพอดี
  (4+2+2 = 8 ไบต์ทั้งคู่) COBOL จะคัดลอกไบต์ต่อไบต์แบบตัวอักษรล้วน ๆ โดยไม่สนใจว่าภายในแบ่งเป็น
  ฟิลด์ย่อยกี่ตัว ในกรณีนี้เนื่องจากขนาดพอดีกันทุกฟิลด์ย่อย ผลลัพธ์จึงออกมาถูกต้องเหมือนกับ elementary
  move (ค่าตรงกันทุกประการ)
- ส่วนที่สองใช้ elementary MOVE ทีละฟิลด์ ซึ่งได้ผลลัพธ์เดียวกันในตัวอย่างนี้เพราะขนาดฟิลด์ตรงกันพอดี

### ผลลัพธ์ที่ได้จากการรันจริง

```
GROUP MOVE (treated as one big alphanumeric field):
YEAR  : 2026
MONTH : 09
DAY   : 26

ELEMENTARY MOVE (field by field, numeric rules):
YEAR  : 2026
MONTH : 09
DAY   : 26
```

### ข้อควรระวัง

- **กับดักสำคัญที่สุด**: หาก group MOVE มีขนาดรวมของทั้งสองฝั่ง **ไม่เท่ากัน** (เช่น ปลายทางมีฟิลด์
  ย่อยเพิ่มมาอีกตัว หรือฟิลด์ย่อยบางตัวมีขนาดต่างกัน) ข้อมูลจะเลื่อนตำแหน่งผิดพลาดไปหมด เพราะ COBOL
  จัดชิดซ้ายแบบตัวอักษรกับก้อนข้อมูลทั้งก้อน ไม่ใช่จัดฟิลด์ย่อยแต่ละตัวให้ตรงกันเหมือน elementary
  MOVE — นี่คือบั๊กที่ทำให้ข้อมูลปีเดือนวันสลับตำแหน่งกันได้ในระบบจริงถ้าโครงสร้างสองฝั่งไม่ตรงกันเป๊ะ
- แนะนำให้ใช้ elementary MOVE (ทีละฟิลด์) หรือ `MOVE CORRESPONDING` เมื่อไม่มั่นใจว่าโครงสร้างตรงกัน
  100% เพื่อความปลอดภัย

### แบบฝึกหัดที่ 76.1

**โจทย์**: หาก `WS-DATE-IN` มีขนาดรวม 8 ไบต์ แต่ `WS-DATE-OUT` มีฟิลด์ย่อยเพิ่มมาอีกตัวทำให้ขนาดรวม
เป็น 10 ไบต์ การทำ group MOVE จะเกิดผลอย่างไร?

**เฉลยแนวทาง**: เนื่องจากปลายทางใหญ่กว่า ข้อมูล 8 ไบต์จากต้นทางจะถูกคัดลอกไปแบบชิดซ้าย
แล้วเติมช่องว่าง 2 ไบต์ท้ายสุด ถ้าฟิลด์ย่อยตัวสุดท้ายของปลายทางเป็นตัวเลข การเติมช่องว่าง
(แทนที่จะเติมศูนย์) อาจทำให้ฟิลด์นั้นมีค่าไม่ใช่ตัวเลขที่ถูกต้อง (invalid numeric data)
ซึ่งจะทำให้เกิดปัญหาเมื่อนำไปคำนวณต่อในภายหลัง

---

## ขั้นตอนที่ 77: MOVE ไปยัง PICTURE Edited Fields (ฟิลด์ตัวเลขแบบมีรูปแบบแสดงผล)

### แนวคิด

นอกจากฟิลด์ตัวเลขธรรมดา (`PIC 9...`) COBOL ยังมี **Numeric-Edited PICTURE** ที่ใช้สัญลักษณ์พิเศษ
เพื่อจัดรูปแบบการแสดงผลตัวเลขให้อ่านง่ายขึ้น เช่น:

| สัญลักษณ์ | ความหมาย |
|---|---|
| `Z` | แสดงตัวเลข แต่ถ้าเป็น 0 นำหน้าจะแสดงเป็นช่องว่างแทน (zero suppression) |
| `,` | ใส่เครื่องหมายจุลภาคคั่นหลักพัน |
| `.` | จุดทศนิยมที่แสดงผลจริง (ต่างจาก `V` ที่เป็นจุดทศนิยมโดยนัย ไม่แสดงผล) |
| `$` | ใส่เครื่องหมายสกุลเงิน |

การ `MOVE` ตัวเลขปกติเข้าไปยังฟิลด์ edited เหล่านี้จะทำการแปลงรูปแบบการแสดงผลให้อัตโนมัติ
แต่ **ทำได้ทิศทางเดียว**: MOVE จากฟิลด์ธรรมดาไปฟิลด์ edited ได้ แต่การนำฟิลด์ edited (ที่มี comma
หรือ $ ปนอยู่) กลับไปคำนวณต่อทำไม่ได้โดยตรง เพราะไม่ใช่ข้อมูลตัวเลขบริสุทธิ์อีกต่อไป

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-EDITED-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SALARY           PIC 9(6)V99  VALUE 48500.75.
       01  WS-SALARY-EDIT      PIC ZZZ,ZZ9.99.
       01  WS-SALARY-DOLLAR    PIC $$$,$$9.99.
       01  WS-ZERO-SALARY      PIC 9(6)V99  VALUE 0.
       01  WS-ZERO-EDIT        PIC ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
           MOVE WS-SALARY TO WS-SALARY-EDIT
           DISPLAY "SALARY EDITED (ZZZ,ZZ9.99) : " WS-SALARY-EDIT

           MOVE WS-SALARY TO WS-SALARY-DOLLAR
           DISPLAY "SALARY EDITED (WITH $)     : " WS-SALARY-DOLLAR

           MOVE WS-ZERO-SALARY TO WS-ZERO-EDIT
           DISPLAY "ZERO EDITED (BLANK ZEROES) : [" WS-ZERO-EDIT "]"
           STOP RUN.
```

### อธิบายโค้ด

- `WS-SALARY-EDIT PIC ZZZ,ZZ9.99` — ตัว `Z` ทำให้เลข 0 นำหน้าแสดงเป็นช่องว่างแทน (zero suppression)
  ส่วน `9` ตัวสุดท้ายก่อนจุดยังคงบังคับให้แสดงเลขเสมอแม้เป็น 0 (ป้องกันไม่ให้แสดงค่าว่างเปล่าทั้งหมด)
  และ `,` จะถูกแทรกเข้าไปในตำแหน่งที่กำหนดโดยอัตโนมัติเมื่อมีตัวเลขมากพอ
- `WS-SALARY-DOLLAR PIC $$$,$$9.99` — คล้ายกันแต่ใช้ `$` แทน `Z` ซึ่งนอกจากจะกดข่ม 0 นำหน้าแล้ว
  ยังแสดงเครื่องหมาย `$` ลอยตัว (floating) อยู่หน้าตัวเลขตัวแรกที่ไม่ใช่ 0 ด้วย
- กรณี `WS-ZERO-SALARY` มีค่าเป็น 0 ทั้งหมด แต่ `PIC ZZZ,ZZ9.99` ยังคงบังคับแสดง `9` ตำแหน่งสุดท้าย
  จึงได้ผลลัพธ์เป็น "0.00" ที่มีช่องว่างนำหน้าแทนที่จะเป็น "000000.00"

### ผลลัพธ์ที่ได้จากการรันจริง

```
SALARY EDITED (ZZZ,ZZ9.99) :  48,500.75
SALARY EDITED (WITH $)     : $48,500.75
ZERO EDITED (BLANK ZEROES) : [      0.00]
```

### ข้อควรระวัง

- ฟิลด์ Numeric-Edited **ใช้สำหรับแสดงผลเท่านั้น ไม่ควรใช้ในการคำนวณต่อ** เพราะมีอักขระที่ไม่ใช่
  ตัวเลขล้วนปะปนอยู่ (comma, จุด, เครื่องหมาย $) หากต้องคำนวณต่อ ให้เก็บค่าดิบไว้ในฟิลด์ `PIC 9...V99`
  แยกต่างหาก แล้วค่อย `MOVE` ไปยังฟิลด์ edited เฉพาะตอนจะแสดงผลให้ผู้ใช้เห็นเท่านั้น
- ระวังเรื่องขนาดฟิลด์ edited ที่มักจะใช้พื้นที่มากกว่าฟิลด์ตัวเลขปกติ (เพราะต้องเผื่อที่ให้ comma
  และเครื่องหมายพิเศษ) ให้คำนวณความยาวให้ครบถ้วนตอนออกแบบ PICTURE clause

### แบบฝึกหัดที่ 77.1

**โจทย์**: จงออกแบบ PICTURE clause แบบ edited สำหรับแสดงยอดเงิน 7 หลักก่อนจุดทศนิยม 2 หลักหลัง
จุดทศนิยม พร้อมเครื่องหมาย comma คั่นหลักพันและกดข่มเลข 0 นำหน้าทั้งหมด

**เฉลย**: `PIC Z,ZZZ,ZZ9.99` — 7 หลักก่อนจุดทศนิยมต้องใช้สัญลักษณ์ 7 ตัว (นับรวม comma 2 ตัวที่
คั่นหลักพันทุก ๆ 3 หลัก) โดยตัวสุดท้ายก่อนจุดต้องเป็น `9` เพื่อบังคับแสดงผลแม้เป็น 0

---

## ขั้นตอนที่ 78: MOVE กับค่าคงที่พิเศษ (Figurative Constants): ZERO, SPACES, ALL

### แนวคิด

COBOL มี **Figurative Constants** หรือค่าคงที่พิเศษที่ใช้แทนค่าที่ใช้บ่อยโดยไม่ต้องพิมพ์ตัวเลขหรือ
ตัวอักษรซ้ำ ๆ เอง ค่าที่ใช้บ่อยที่สุดคือ:

| Figurative Constant | ความหมาย | ใช้ได้กับ |
|---|---|---|
| `ZERO` / `ZEROS` / `ZEROES` | ค่า 0 (ทั้งฟิลด์) | ฟิลด์ตัวเลขและตัวอักษร |
| `SPACE` / `SPACES` | ช่องว่างเต็มฟิลด์ | ฟิลด์ตัวอักษรเป็นหลัก |
| `HIGH-VALUE` / `HIGH-VALUES` | ค่าไบต์สูงสุดเต็มฟิลด์ (มักใช้เทียบ/เรียงลำดับ) | ฟิลด์ตัวอักษร |
| `LOW-VALUE` / `LOW-VALUES` | ค่าไบต์ต่ำสุดเต็มฟิลด์ (มักใช้ล้างค่าเริ่มต้น) | ฟิลด์ตัวอักษร |
| `ALL "x"` | ทำซ้ำตัวอักษร/ข้อความที่ระบุจนเต็มฟิลด์ | ฟิลด์ตัวอักษร |

ค่าเหล่านี้มีประโยชน์มากในการ "ล้างค่า" (initialize/reset) ตัวแปรก่อนใช้งาน โดยไม่ต้องรู้ขนาด
ฟิลด์ที่แน่นอนล่วงหน้า

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-FIGURATIVE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER          PIC 9(5)  VALUE 999.
       01  WS-NAME-FIELD       PIC X(10) VALUE "TEMP".
       01  WS-FILLER-FIELD     PIC X(10).
       01  WS-AMOUNT           PIC 9(5)  VALUE 777.

       PROCEDURE DIVISION.
           DISPLAY "BEFORE: COUNTER=" WS-COUNTER
               " NAME=[" WS-NAME-FIELD "]"

           MOVE ZERO TO WS-COUNTER
           MOVE SPACES TO WS-NAME-FIELD
           MOVE ALL "*" TO WS-FILLER-FIELD
           MOVE ZEROES TO WS-AMOUNT

           DISPLAY "AFTER : COUNTER=" WS-COUNTER
               " NAME=[" WS-NAME-FIELD "]"
           DISPLAY "FILLER FIELD    : [" WS-FILLER-FIELD "]"
           DISPLAY "AMOUNT (ZEROES) : " WS-AMOUNT
           STOP RUN.
```

### อธิบายโค้ด

- `MOVE ZERO TO WS-COUNTER` — เติมค่า 0 เต็มฟิลด์ตัวเลข ไม่ว่าฟิลด์จะมีกี่หลักก็ตาม (ไม่ต้องรู้ขนาด
  ล่วงหน้า)
- `MOVE SPACES TO WS-NAME-FIELD` — ล้างฟิลด์ตัวอักษรให้เป็นช่องว่างทั้งหมด นิยมใช้ก่อนรับข้อมูลใหม่
  เข้าฟิลด์เพื่อป้องกันข้อมูลเก่าตกค้าง (โดยเฉพาะเมื่อข้อมูลใหม่สั้นกว่าข้อมูลเก่า)
- `MOVE ALL "*" TO WS-FILLER-FIELD` — ทำซ้ำตัวอักษร `*` จนเต็มฟิลด์ 10 ช่อง มักใช้ทำเส้นคั่นหรือ
  รูปแบบพิเศษในรายงาน
- `ZERO`, `ZEROS`, `ZEROES` เป็นคำเดียวกันเขียนได้หลายรูปแบบ (เพื่อความเป็นธรรมชาติของภาษาอังกฤษ)

### ผลลัพธ์ที่ได้จากการรันจริง

```
BEFORE: COUNTER=00999 NAME=[TEMP      ]
AFTER : COUNTER=00000 NAME=[          ]
FILLER FIELD    : [**********]
AMOUNT (ZEROES) : 00000
```

### ข้อควรระวัง

- `HIGH-VALUE` และ `LOW-VALUE` มักทำให้ผลลัพธ์ `DISPLAY` ออกมาเป็นอักขระประหลาดหรือ error
  encoding เพราะเป็นไบต์ที่ไม่ใช่ตัวอักษรที่พิมพ์ได้ (non-printable) — ใช้เพื่อวัตถุประสงค์เฉพาะ เช่น
  การเรียงลำดับไฟล์ (sorting) หรือกำหนดค่าเริ่มต้นพิเศษ ไม่ควรนำมา `DISPLAY` ตรง ๆ
- `MOVE ALL "AB"` กับฟิลด์ที่มีขนาดไม่ลงตัวกับความยาวของ "AB" (เช่นฟิลด์ 5 ช่อง) จะทำซ้ำเท่าที่พอดี
  แล้วตัดส่วนเกินทิ้ง เช่น ได้ "ABABA" ไม่ใช่ "ABABAB"

### แบบฝึกหัดที่ 78.1

**โจทย์**: จงเขียนคำสั่ง MOVE เพื่อล้างฟิลด์ `WS-DASH-LINE PIC X(20)` ให้เต็มไปด้วยเครื่องหมายขีด `-`
ทั้งหมด 20 ตัว

**เฉลย**: `MOVE ALL "-" TO WS-DASH-LINE` — จะได้ผลลัพธ์เป็นเส้นขีด 20 ตัวเต็มฟิลด์พอดี

---

## ขั้นตอนที่ 79: ข้อผิดพลาดที่พบบ่อยจาก MOVE — Deep Dive

### แนวคิด

ขั้นตอนนี้จะรวบรวม **3 กับดักที่อันตรายที่สุด** ของคำสั่ง `MOVE` ที่พบบ่อยในโปรแกรม COBOL จริง
ซึ่งล้วนมีจุดร่วมเดียวกันคือ **compiler ไม่แจ้ง error ใด ๆ แต่ผลลัพธ์ที่ได้อาจผิดพลาดอย่างร้ายแรง**
เพราะ MOVE ไม่มีการตรวจสอบความถูกต้องของข้อมูล (data validation) ในตัว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-PITFALLS-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ALPHA-DIGITS     PIC X(5)  VALUE "12A45".
       01  WS-NUMERIC-TARGET   PIC 9(5).

       01  WS-SIGNED-SOURCE    PIC S9(5)  VALUE -250.
       01  WS-UNSIGNED-TARGET  PIC 9(5).

       01  WS-OVERSIZE-SOURCE  PIC 9(9)   VALUE 123456789.
       01  WS-UNDERSIZE-TARGET PIC 9(4).

       PROCEDURE DIVISION.
           DISPLAY "PITFALL 1: MOVE non-numeric data into numeric field"
           DISPLAY "SOURCE (has letter A) : " WS-ALPHA-DIGITS
           MOVE WS-ALPHA-DIGITS TO WS-NUMERIC-TARGET
           DISPLAY "RESULT (undefined!)   : " WS-NUMERIC-TARGET

           DISPLAY " "
           DISPLAY "PITFALL 2: MOVE signed number to unsigned field"
           DISPLAY "SIGNED SOURCE  : " WS-SIGNED-SOURCE
           MOVE WS-SIGNED-SOURCE TO WS-UNSIGNED-TARGET
           DISPLAY "UNSIGNED TARGET (sign lost, absolute value kept) : "
               WS-UNSIGNED-TARGET

           DISPLAY " "
           DISPLAY "PITFALL 3: MOVE oversized number to a smaller field"
           DISPLAY "SOURCE (9 digits)     : " WS-OVERSIZE-SOURCE
           MOVE WS-OVERSIZE-SOURCE TO WS-UNDERSIZE-TARGET
           DISPLAY "TARGET (4 digits, high-order digits truncated!) : "
               WS-UNDERSIZE-TARGET
           STOP RUN.
```

### อธิบายกับดักทั้ง 3 ข้อ

1. **PITFALL 1 — MOVE ข้อมูลที่ไม่ใช่ตัวเลขล้วนเข้าฟิลด์ตัวเลข**: `WS-ALPHA-DIGITS` มีค่า "12A45"
   ซึ่งมีตัวอักษร "A" ปนอยู่ ไม่ใช่ตัวเลขบริสุทธิ์ เมื่อ `MOVE` เข้าฟิลด์ `PIC 9(5)` มาตรฐาน COBOL
   ถือว่านี่คือ **พฤติกรรมที่ไม่ได้นิยามไว้ชัดเจน (undefined behavior ตามมาตรฐาน)** — คอมไพเลอร์
   แต่ละตัวอาจได้ผลลัพธ์ต่างกัน ในการทดสอบจริงด้วย GnuCOBOL ได้ผลลัพธ์เป็น `00000` แต่ไม่ควรพึ่งพา
   พฤติกรรมนี้เลยในโค้ดจริง เพราะระบบอื่นอาจให้ผลต่างออกไป
2. **PITFALL 2 — MOVE เลขติดลบไปยังฟิลด์ที่ไม่มีเครื่องหมาย (unsigned)**: `WS-SIGNED-SOURCE`
   มีค่า -250 (`PIC S9(5)` มี `S` แสดงว่ามีเครื่องหมาย) แต่ `WS-UNSIGNED-TARGET` เป็น `PIC 9(5)`
   ไม่มี `S` เมื่อ MOVE เข้าไป **เครื่องหมายลบจะหายไปเงียบ ๆ เหลือแต่ค่าสัมบูรณ์ (absolute value)**
   กลายเป็น 250 แทนที่จะเป็น -250 — อันตรายมากถ้าเป็นยอดเงินติดลบในระบบบัญชี
3. **PITFALL 3 — MOVE ตัวเลขที่มีจำนวนหลักมากกว่าฟิลด์ปลายทาง**: 123456789 (9 หลัก) MOVE เข้าฟิลด์
   4 หลัก จะถูกตัดหลักสูง (ซ้ายมือ) ทิ้งไป เหลือเพียง 4 หลักขวาสุดคือ `6789` — ค่าที่ได้ผิดไปจากเดิม
   อย่างสิ้นเชิงโดยไม่มีการแจ้งเตือนใด ๆ

### ผลลัพธ์ที่ได้จากการรันจริง

```
PITFALL 1: MOVE non-numeric data into numeric field
SOURCE (has letter A) : 12A45
RESULT (undefined!)   : 00000

PITFALL 2: MOVE signed number to unsigned field
SIGNED SOURCE  : -00250
UNSIGNED TARGET (sign lost, absolute value kept) : 00250

PITFALL 3: MOVE oversized number to a smaller field
SOURCE (9 digits)     : 123456789
TARGET (4 digits, high-order digits truncated!) : 6789
```

### ข้อควรระวัง

- ทั้ง 3 กับดักนี้ **ไม่ทำให้โปรแกรม crash หรือแสดง error** จึงเป็นบั๊กที่ตรวจจับยากที่สุดประเภทหนึ่ง
  ในระบบ COBOL จริง มักถูกค้นพบตอนตรวจสอบรายงานทางบัญชีแล้วพบว่ายอดเงินไม่ตรงเท่านั้น
- แนวทางป้องกัน: (1) ตรวจสอบข้อมูลนำเข้าด้วย `IF ... IS NUMERIC` ก่อน MOVE เสมอเมื่อรับข้อมูลจาก
  ภายนอก (2) ออกแบบขนาดฟิลด์ให้เผื่อค่าที่เป็นไปได้จริงเสมอ ไม่ประมาณขนาดต่ำเกินไป
  (3) ใช้ `S` ใน PICTURE clause ทุกครั้งที่ค่าที่เป็นไปได้อาจติดลบ

### แบบฝึกหัดที่ 79.1

**โจทย์**: จงอธิบายว่าทำไมองค์กรจึงมักออกแบบฟิลด์ตัวเลขทางการเงินให้มีจำนวนหลักมากกว่าที่คาดว่า
จะใช้จริง เช่น ใช้ `PIC 9(9)V99` แทนที่จะเป็น `PIC 9(6)V99` สำหรับยอดเงินที่คาดว่าจะไม่เกินหลักล้าน

**เฉลยแนวทาง**: เพื่อป้องกัน PITFALL 3 (การตัดหลักสูงทิ้งแบบเงียบ ๆ) ในอนาคตหากธุรกิจเติบโตขึ้น
และยอดเงินที่ต้องประมวลผลเกินขอบเขตที่ออกแบบไว้ตอนแรก การเผื่อหลักไว้มากกว่าความจำเป็นปัจจุบัน
เป็นแนวทางป้องกันความเสี่ยงที่ต้นทุนต่ำมาก (แค่เพิ่มจำนวนหลักใน PICTURE) เทียบกับความเสียหายที่อาจ
เกิดขึ้นหากยอดเงินถูกตัดทอนอย่างเงียบ ๆ ในระบบการเงินจริง

---

## ขั้นตอนที่ 80: MOVE หลายเป้าหมายพร้อมกัน และแนวทางปฏิบัติที่ดี (Best Practices)

### แนวคิด

COBOL อนุญาตให้ `MOVE` ค่าเดียวไปยัง **หลายฟิลด์ปลายทางพร้อมกัน** ในคำสั่งเดียว โดยเขียนชื่อฟิลด์
ปลายทางเรียงต่อกันหลัง `TO` ซึ่งช่วยลดจำนวนบรรทัดโค้ดเมื่อต้องการตั้งค่าเริ่มต้นให้หลายตัวแปรพร้อมกัน

ปิดท้าย Part นี้ด้วยการสรุปแนวทางปฏิบัติที่ดีที่สุดสำหรับการใช้ `MOVE` ในการทำงานจริง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MOVE-MULTI-TARGET-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STATUS-CODE      PIC X(1).
       01  WS-STATUS-A         PIC X(1).
       01  WS-STATUS-B         PIC X(1).
       01  WS-STATUS-C         PIC X(1).

       01  WS-DEFAULT-QTY      PIC 9(3).
       01  WS-QTY-WAREHOUSE-1  PIC 9(3).
       01  WS-QTY-WAREHOUSE-2  PIC 9(3).

       PROCEDURE DIVISION.
           MOVE "N" TO WS-STATUS-A WS-STATUS-B WS-STATUS-C
           DISPLAY "A=" WS-STATUS-A " B=" WS-STATUS-B
               " C=" WS-STATUS-C

           MOVE 100 TO WS-QTY-WAREHOUSE-1 WS-QTY-WAREHOUSE-2
           DISPLAY "WAREHOUSE-1 QTY : " WS-QTY-WAREHOUSE-1
           DISPLAY "WAREHOUSE-2 QTY : " WS-QTY-WAREHOUSE-2

           MOVE "Y" TO WS-STATUS-CODE
           DISPLAY "STATUS-CODE     : " WS-STATUS-CODE
           STOP RUN.
```

### อธิบายโค้ด

- `MOVE "N" TO WS-STATUS-A WS-STATUS-B WS-STATUS-C` — ค่า "N" ถูกย้ายไปยังตัวแปรทั้งสามตัวพร้อมกัน
  เทียบเท่ากับการเขียน `MOVE "N" TO WS-STATUS-A. MOVE "N" TO WS-STATUS-B. MOVE "N" TO WS-STATUS-C.`
  แยกกันสามบรรทัด แต่กระชับกว่ามาก
- ทุกฟิลด์ปลายทางในคำสั่งเดียวกันจะได้รับ **ค่าเดียวกัน** ตามกฎ MOVE ปกติของแต่ละฟิลด์ (ปรับตามชนิด
  ข้อมูลของตัวเองแยกกัน)

### ผลลัพธ์ที่ได้จากการรันจริง

```
A=N B=N C=N
WAREHOUSE-1 QTY : 100
WAREHOUSE-2 QTY : 100
STATUS-CODE     : Y
```

### สรุปกฎ MOVE ทั้งหมดใน Part นี้ (Quick Reference)

| สถานการณ์ | กฎ |
|---|---|
| ตัวอักษร → ตัวอักษร (ปลายทางเล็กกว่า) | ตัดขวา ชิดซ้าย |
| ตัวอักษร → ตัวอักษร (ปลายทางใหญ่กว่า) | เติมช่องว่างขวา ชิดซ้าย |
| ตัวเลข → ตัวเลข (ปลายทางเล็กกว่า) | ตัดหลักสูง (ซ้าย) ทิ้ง จัดชิดขวา |
| ตัวเลข → ตัวเลข (ปลายทางใหญ่กว่า) | เติม 0 นำหน้า จัดชิดขวา |
| ตัวเลขมีทศนิยมต่างกัน | จัดจุดทศนิยมให้ตรงกัน ตัดทศนิยมส่วนเกินทิ้ง (ไม่ปัดเศษ) |
| เลขมีเครื่องหมาย → ฟิลด์ไม่มีเครื่องหมาย | เครื่องหมายหายไป เหลือค่าสัมบูรณ์ |
| Group MOVE | ปฏิบัติเหมือนก้อนตัวอักษรทั้งก้อน ไม่สนใจ PICTURE ย่อยภายใน |
| MOVE CORRESPONDING | ย้ายเฉพาะฟิลด์ย่อยที่ชื่อตรงกันทุกตัวอักษร |

### ข้อควรระวัง

- การ `MOVE` ค่าเดียวไปหลายเป้าหมายสะดวก แต่ต้องแน่ใจว่าทุกฟิลด์ปลายทางควรมีค่าเดียวกันจริง ๆ
  หากในอนาคตต้องแก้ให้ค่าต่างกัน จะต้องแยกกลับมาเขียนทีละบรรทัดใหม่
- แนวทางปฏิบัติที่ดี: (1) ใช้ `MOVE SPACES`/`MOVE ZERO` ล้างค่าตัวแปรก่อนใช้งานเสมอเมื่อไม่มั่นใจ
  ค่าเริ่มต้น (2) ตรวจสอบขนาดฟิลด์ต้นทาง-ปลายทางให้เหมาะสมก่อน MOVE เสมอ (3) ใช้ `MOVE
  CORRESPONDING` เมื่อโครงสร้างข้อมูลมีฟิลด์จำนวนมากและชื่อตรงกันเพื่อลดโค้ดซ้ำซ้อน (4) หลีกเลี่ยง
  group MOVE ระหว่างโครงสร้างที่ขนาดไม่แน่ใจว่าตรงกัน 100%

### แบบฝึกหัดที่ 80.1

**โจทย์**: จงเขียนคำสั่ง MOVE หนึ่งบรรทัดเพื่อกำหนดค่าเริ่มต้น 0 ให้กับตัวแปรตัวเลขสามตัวคือ
`WS-TOTAL-1`, `WS-TOTAL-2`, และ `WS-TOTAL-3` พร้อมกัน

**เฉลย**: `MOVE ZERO TO WS-TOTAL-1 WS-TOTAL-2 WS-TOTAL-3` — คำสั่งเดียวตั้งค่า 0 ให้ทั้งสามตัวแปร
พร้อมกัน แต่ละตัวจะถูกเติมด้วยเลข 0 เต็มฟิลด์ตามขนาดของตัวเอง

---

## สรุปท้ายบท

ใน Part นี้ เราได้เจาะลึกคำสั่ง `MOVE` ซึ่งเป็นหนึ่งในคำสั่งที่ใช้บ่อยที่สุดใน COBOL อย่างละเอียด:

- MOVE พื้นฐานสำหรับข้อมูลตัวอักษร (ชิดซ้าย เติมช่องว่าง) และข้อมูลตัวเลข (ชิดขวา เติมศูนย์)
- การจัดตำแหน่งจุดทศนิยมโดยอัตโนมัติ และกฎการตัดทิ้ง (truncate) แทนการปัดเศษ (round)
- การตัดและเติมช่องว่างของข้อมูลตัวอักษรเมื่อขนาดฟิลด์ไม่เท่ากัน
- `MOVE CORRESPONDING` สำหรับย้ายข้อมูลระหว่างกลุ่มที่มีฟิลด์ย่อยชื่อตรงกัน
- ความแตกต่างระหว่าง Group MOVE และ Elementary MOVE ที่อาจทำให้ข้อมูลผิดเพี้ยนถ้าไม่ระวัง
- MOVE ไปยัง PICTURE Edited Fields เพื่อจัดรูปแบบการแสดงผลตัวเลข
- Figurative Constants: ZERO, SPACES, ALL สำหรับการล้างค่าตัวแปร
- กับดักอันตราย 3 ประการที่ compiler ไม่แจ้งเตือน: ข้อมูลไม่ใช่ตัวเลขล้วน, เครื่องหมายลบหายไป,
  และการตัดหลักสูงทิ้งแบบเงียบ ๆ
- การ MOVE ค่าเดียวไปหลายเป้าหมายพร้อมกัน และแนวทางปฏิบัติที่ดี

ความเข้าใจกฎ MOVE อย่างละเอียดจะเป็นพื้นฐานสำคัญสำหรับ Part ถัดไป ซึ่งเราจะเรียนรู้คำสั่งเลขคณิต
(`ADD`, `SUBTRACT`, `MULTIPLY`, `DIVIDE`, `COMPUTE`) ที่ต้องอาศัยความเข้าใจเรื่องขนาดฟิลด์และ
การจัดตำแหน่งทศนิยมที่เราเพิ่งเรียนไปนี้เป็นพื้นฐานสำคัญเช่นกัน

**[← กลับไป Part 007](part-007-display-accept.md)** | **[ไปยัง Part 009: เลขคณิตพื้นฐาน →](part-009-arithmetic.md)**
