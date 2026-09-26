# Part 022: REDEFINES Clause และการใช้หน่วยความจำซ้ำ (ขั้นตอนที่ 211–220)

## คำนำของ Part นี้

ใน Part 021 เราเรียนรู้ `INSPECT` สำหรับตรวจสอบและแก้ไขเนื้อหาภายในฟิลด์ ทำให้ตอนนี้เรามีเครื่องมือ
จัดการข้อความครบทั้งสามตัว (`STRING`, `UNSTRING`, `INSPECT`) แล้ว Part นี้จะเปลี่ยนโฟกัสไปที่แนวคิด
สำคัญอีกเรื่องหนึ่งของ COBOL คือ **`REDEFINES` Clause** ซึ่งเป็นกลไกที่ทำให้เรา**มองข้อมูลชุดเดียวกัน
ในหน่วยความจำเป็นได้หลายรูปแบบพร้อมกัน** โดยไม่ต้องจองพื้นที่หน่วยความจำเพิ่มเลยแม้แต่ไบต์เดียว

แนวคิดนี้อาจฟังดูแปลกใหม่สำหรับผู้ที่คุ้นเคยกับภาษาสมัยใหม่ที่เน้นความปลอดภัยของชนิดข้อมูล (type safety)
แต่ `REDEFINES` เป็นเทคนิคที่**พบได้บ่อยมาก**ในระบบ COBOL ยุค Legacy โดยเฉพาะเมื่อพื้นที่หน่วยความจำ
และพื้นที่จัดเก็บไฟล์มีราคาแพงในอดีต และยังคงสำคัญมากในปัจจุบันสำหรับการประมวลผล **record ที่มีหลาย
รูปแบบปนกันในไฟล์เดียว** (variant records) เช่น ไฟล์ธุรกรรมที่มีทั้งรายการฝาก ถอน และโอนเงิน ซึ่งแต่ละ
ประเภทมีโครงสร้างข้อมูลต่างกัน แต่ถูกจัดเก็บในพื้นที่ขนาดเท่ากัน Part นี้จะพาคุณเรียนรู้ตั้งแต่แนวคิด
พื้นฐานที่สุด ไปจนถึงกฎเกณฑ์ ข้อจำกัด และการประยุกต์ใช้จริงของ `REDEFINES`

---

## ขั้นตอนที่ 211: แนะนำ REDEFINES - มองข้อมูลเดียวกันเป็นหลายมุม

### แนวคิด

**`REDEFINES`** คือ clause ที่ใช้ประกาศว่า **ฟิลด์ใหม่จะใช้พื้นที่หน่วยความจำ "ที่เดียวกัน" กับฟิลด์เดิม**
ที่ประกาศไว้ก่อนหน้า แทนที่จะจองพื้นที่ใหม่ ผลลัพธ์คือเรามี "สองชื่อ" (หรือมากกว่า) ที่ชี้ไปยังไบต์เดียวกัน
ในหน่วยความจำ แต่ตีความไบต์เหล่านั้นด้วย `PICTURE` clause คนละแบบกันได้ ตัวอย่างคลาสสิกที่สุดคือการมอง
วันที่ที่เก็บเป็นตัวเลข 8 หลักรวด (`YYYYMMDD`) เป็นได้ทั้ง "ตัวเลขก้อนเดียว" และ "ปี-เดือน-วัน แยกกัน"
พร้อมกัน โดยไม่ต้องมีการคัดลอกข้อมูลใด ๆ เกิดขึ้นเลย

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s1.cob
      *> Purpose : Introduce REDEFINES - viewing the SAME memory
      *>           through two different data descriptions
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-INTRO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE-NUMERIC      PIC 9(8) VALUE 20260926.
       01  WS-DATE-PARTS REDEFINES WS-DATE-NUMERIC.
           05  WS-YEAR          PIC 9(4).
           05  WS-MONTH         PIC 9(2).
           05  WS-DAY           PIC 9(2).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> WS-DATE-NUMERIC and WS-DATE-PARTS occupy the EXACT SAME
      *> 8 bytes of memory. They are two different "views" onto
      *> one physical storage area, not two separate copies.
           DISPLAY "As one number : " WS-DATE-NUMERIC
           DISPLAY "As year       : " WS-YEAR
           DISPLAY "As month      : " WS-MONTH
           DISPLAY "As day        : " WS-DAY

      *> Changing the numeric view changes what the redefined
      *> view sees too, because they share the same bytes.
           MOVE 20271231 TO WS-DATE-NUMERIC
           DISPLAY " "
           DISPLAY "After changing WS-DATE-NUMERIC to 20271231:"
           DISPLAY "As year       : " WS-YEAR
           DISPLAY "As month      : " WS-MONTH
           DISPLAY "As day        : " WS-DAY

           STOP RUN.
```

### คำอธิบายโค้ด

- `01 WS-DATE-PARTS REDEFINES WS-DATE-NUMERIC.` บอก COBOL ว่า `WS-DATE-PARTS` **ไม่ใช่ตัวแปรใหม่**
  ที่มีพื้นที่ของตัวเอง แต่เป็น "มุมมองใหม่" ที่ซ้อนทับอยู่บนพื้นที่ 8 ไบต์เดียวกับ `WS-DATE-NUMERIC`
- `WS-YEAR PIC 9(4)`, `WS-MONTH PIC 9(2)`, `WS-DAY PIC 9(2)` รวมกันมีขนาด 4+2+2 = 8 ไบต์ พอดีกับ
  `WS-DATE-NUMERIC PIC 9(8)` — นี่คือกฎสำคัญที่เราจะเรียนในรายละเอียดในขั้นตอนถัดไป
- เมื่อ `MOVE` ค่าใหม่ไปยัง `WS-DATE-NUMERIC` ค่าที่มองผ่าน `WS-YEAR`, `WS-MONTH`, `WS-DAY` ก็เปลี่ยน
  ตามไปด้วยทันที เพราะทั้งหมดคือไบต์ชุดเดียวกันในหน่วยความจำ

### ผลลัพธ์ที่ได้จากการรันจริง

```
As one number : 20260926
As year       : 2026
As month      : 09
As day        : 26

After changing WS-DATE-NUMERIC to 20271231:
As year       : 2027
As month      : 12
As day        : 31
```

### ข้อควรระวัง

- `REDEFINES` **ไม่ใช่การคัดลอกข้อมูล** และไม่ใช่การจองหน่วยความจำเพิ่ม มันคือการสร้าง "ชื่อเรียก"
  ใหม่ให้กับพื้นที่เดิม ผู้เริ่มต้นมักเข้าใจผิดว่า `REDEFINES` จะสร้างตัวแปรใหม่ที่มีค่าเริ่มต้นเป็นอิสระ
  จากกัน ซึ่งไม่ถูกต้อง
- เพราะทั้งสองมุมมองใช้พื้นที่เดียวกัน การเปลี่ยนแปลงผ่านมุมมองใดมุมมองหนึ่งจะกระทบอีกมุมมองเสมอ
  ทันที ไม่มีการหน่วงเวลาหรือ "sync" ใด ๆ ให้ต้องทำเอง

### แบบฝึกหัดที่ 211.1

**โจทย์**: จงอธิบายว่าทำไม `WS-DATE-PARTS` ในตัวอย่างข้างต้นจึงต้องมีฟิลด์ย่อยรวมกันขนาด 8 ไบต์พอดี
ไม่ใช่ 7 หรือ 9 ไบต์

**เฉลยแนวทาง**: เพราะ `REDEFINES` ต้องมองพื้นที่หน่วยความจำเดียวกันกับ `WS-DATE-NUMERIC PIC 9(8)`
ซึ่งมีขนาด 8 ไบต์พอดี ถ้า `WS-DATE-PARTS` มีขนาดน้อยกว่า จะเห็นข้อมูลไม่ครบ และถ้ามีขนาดมากกว่า
พื้นที่ส่วนเกินจะไม่ทับซ้อนกับข้อมูลเดิมอย่างที่ตั้งใจ (รายละเอียดเรื่องขนาดต่างกันจะอธิบายในขั้นตอนที่ 215)

---

## ขั้นตอนที่ 212: กฎพื้นฐานของ REDEFINES

### แนวคิด

`REDEFINES` มีกฎเกณฑ์บังคับที่สำคัญ 3 ข้อที่ต้องปฏิบัติตามเสมอ:

1. **ต้องมี level number เดียวกันกับรายการที่ถูก redefine** (เช่น ถ้า `WS-ORIGINAL` เป็นระดับ 01
   ตัว `REDEFINES` ก็ต้องเป็นระดับ 01 เช่นกัน)
2. **ต้องเขียนตามหลังรายการที่ถูก redefine ทันที** ห้ามมีรายการอื่นคั่นกลางแม้แต่ตัวเดียว
3. **ต้องมีชื่อที่ต่างจากรายการเดิม** (เป็นชื่อใหม่ที่ไม่ซ้ำกับชื่ออื่นในโปรแกรม)

ตัวอย่างนี้จะสาธิตข้อ (2) ผ่านการทดสอบจริงว่าเกิด compile error อย่างไรเมื่อฝ่าฝืนกฎ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s2.cob
      *> Purpose : Demonstrate the basic RULES of REDEFINES -
      *>           it shares storage rather than allocating more
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-RULES.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Rule: REDEFINES must come IMMEDIATELY after the item it
      *> redefines, and both must share the SAME level number.
       01  WS-ORIGINAL          PIC X(6) VALUE "ABCDEF".
       01  WS-REDEFINED-VIEW REDEFINES WS-ORIGINAL.
           05  WS-FIRST-HALF    PIC X(3).
           05  WS-SECOND-HALF   PIC X(3).

      *> A marker placed right after, to prove REDEFINES adds NO
      *> extra bytes: if it did, this marker would start later.
       01  WS-MARKER            PIC X(5) VALUE "-END-".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "Original       : " WS-ORIGINAL
           DISPLAY "First half     : " WS-FIRST-HALF
           DISPLAY "Second half    : " WS-SECOND-HALF
           DISPLAY "Marker (proof) : " WS-MARKER

      *> FUNCTION LENGTH confirms WS-REDEFINED-VIEW takes exactly
      *> as many bytes as WS-ORIGINAL: 3 + 3 = 6, no more, no less.
           DISPLAY "Length of WS-ORIGINAL       : "
               FUNCTION LENGTH(WS-ORIGINAL)
           DISPLAY "Length of WS-REDEFINED-VIEW : "
               FUNCTION LENGTH(WS-REDEFINED-VIEW)

           STOP RUN.
```

### คำอธิบายโค้ด

- `FUNCTION LENGTH` ยืนยันว่า `WS-REDEFINED-VIEW` มีขนาดเท่ากับ `WS-ORIGINAL` เป๊ะ (6 ไบต์ทั้งคู่)
  พิสูจน์ว่า `REDEFINES` ไม่ได้เพิ่มพื้นที่หน่วยความจำใด ๆ เลย
- `WS-MARKER` ที่ประกาศต่อจากกลุ่ม `REDEFINES` ทำหน้าที่เป็น "เครื่องพิสูจน์" ว่าไม่มีช่องว่างแปลกปลอม
  เกิดขึ้นในหน่วยความจำจากการใช้ `REDEFINES`

### ผลลัพธ์ที่ได้จากการรันจริง

```
Original       : ABCDEF
First half     : ABC
Second half    : DEF
Marker (proof) : -END-
Length of WS-ORIGINAL       : 6
Length of WS-REDEFINED-VIEW : 6
```

### ข้อควรระวัง

- **การฝ่าฝืนกฎข้อ (2)** (ต้องเขียนตามหลังรายการเดิมทันที) จะทำให้เกิด compile error ทันที
  ทดสอบจริงด้วยโค้ดต่อไปนี้:

```cobol
       01  WS-ORIGINAL          PIC X(6) VALUE "ABCDEF".
       01  WS-UNRELATED         PIC X(4) VALUE "GAP1".
       01  WS-BAD-VIEW REDEFINES WS-ORIGINAL PIC X(6).
```

  เมื่อ compile ด้วย `cobc` จะได้ error จริงดังนี้:

```
error: REDEFINES must follow the original definition
```

  ข้อความนี้ยืนยันชัดเจนว่า COBOL บังคับกฎเรื่องลำดับการเขียนอย่างเข้มงวด ไม่ยอมให้มีรายการอื่นคั่นกลาง
  ระหว่างรายการต้นฉบับกับรายการที่ `REDEFINES` มันแม้แต่รายการเดียว

### แบบฝึกหัดที่ 212.1

**โจทย์**: จงอธิบายว่าทำไมกฎข้อ (2) "ต้องเขียนตามหลังทันที" จึงสมเหตุสมผลในเชิงการออกแบบภาษา

**เฉลยแนวทาง**: เพราะ compiler ต้องรู้อย่างชัดเจนว่า `REDEFINES` กำลังอ้างอิงพื้นที่หน่วยความจำ
ณ ตำแหน่งใดในลำดับการจัดสรร ถ้าอนุญาตให้มีรายการอื่นคั่นกลางได้ จะทำให้เกิดความกำกวมว่าตำแหน่งเริ่มต้น
ของการซ้อนทับควรอยู่ตรงไหนกันแน่ กฎนี้จึงทำให้โครงสร้างข้อมูลอ่านง่ายและตรวจสอบได้ชัดเจนขึ้นสำหรับทั้ง
compiler และมนุษย์ที่อ่านโค้ด

---

## ขั้นตอนที่ 213: REDEFINES มุมมองตัวเลขกับตัวอักษร (Numeric vs Alphanumeric View)

### แนวคิด

การใช้งานที่พบบ่อยมากของ `REDEFINES` คือการมองฟิลด์เดียวกันเป็นทั้ง**ตัวอักษรดิบ (`PIC X`)** และ
**ตัวเลข (`PIC 9`)** สลับกันไปตามสถานการณ์ ประโยชน์สำคัญคือช่วยให้เรา**ตรวจสอบความถูกต้องของข้อมูล**
ก่อนนำไปใช้งานเป็นตัวเลขจริง เช่น ข้อมูลจากไฟล์ภายนอกที่ควรจะเป็นตัวเลขแต่บางครั้งกลับเป็นช่องว่างล้วน
(หมายถึง "ยังไม่มีข้อมูล") ถ้าพยายามอ่านเป็นตัวเลขตรง ๆ ทันทีอาจได้ผลลัพธ์ที่ไม่คาดคิด การมองผ่าน
`PIC X` ก่อนเพื่อตรวจสอบ (เช่น เทียบกับ `SPACES`) แล้วจึงตัดสินใจว่าจะใช้มุมมองตัวเลขหรือไม่ ปลอดภัยกว่ามาก

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s3.cob
      *> Purpose : REDEFINES a numeric field as alphanumeric, so
      *>           we can safely inspect raw bytes that might not
      *>           actually be valid digits (e.g. blank input)
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-NUMERIC-CHECK.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AMOUNT-RAW        PIC X(6) VALUE "      ".
       01  WS-AMOUNT-NUM REDEFINES WS-AMOUNT-RAW PIC 9(6).

       01  WS-AMOUNT-RAW-2      PIC X(6) VALUE "001250".
       01  WS-AMOUNT-NUM-2 REDEFINES WS-AMOUNT-RAW-2 PIC 9(6).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> This record's amount field arrived as all spaces, which
      *> is NOT a valid PIC 9 value. We declare the RAW bytes as
      *> PIC X first, and REDEFINE a numeric view for later use,
      *> so we can safely test for blanks BEFORE trusting it as
      *> a number.
           DISPLAY "Field 1 raw bytes : [" WS-AMOUNT-RAW "]"
           IF WS-AMOUNT-RAW = SPACES
               DISPLAY "Field 1: no amount supplied (blank)."
           ELSE
               DISPLAY "Field 1 as number : " WS-AMOUNT-NUM
           END-IF

           DISPLAY " "
           DISPLAY "Field 2 raw bytes : [" WS-AMOUNT-RAW-2 "]"
           IF WS-AMOUNT-RAW-2 = SPACES
               DISPLAY "Field 2: no amount supplied (blank)."
           ELSE
               DISPLAY "Field 2 as number : " WS-AMOUNT-NUM-2
           END-IF

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-AMOUNT-RAW PIC X(6)` คือฟิลด์หลักที่เก็บข้อมูล**ดิบ**ตามที่ได้รับมาจริง (อาจเป็นตัวเลขหรือ
  ช่องว่างก็ได้) ส่วน `WS-AMOUNT-NUM REDEFINES WS-AMOUNT-RAW PIC 9(6)` คือมุมมองตัวเลขที่ซ้อนทับอยู่
- โปรแกรมตรวจสอบ `WS-AMOUNT-RAW = SPACES` **ก่อน** ที่จะใช้งานมุมมองตัวเลข `WS-AMOUNT-NUM`
  เพื่อป้องกันการนำข้อมูลที่ไม่ใช่ตัวเลขจริงไปใช้ในการคำนวณโดยไม่ตั้งใจ

### ผลลัพธ์ที่ได้จากการรันจริง

```
Field 1 raw bytes : [      ]
Field 1: no amount supplied (blank).

Field 2 raw bytes : [001250]
Field 2 as number : 001250
```

### ข้อควรระวัง

- ถ้าพยายามใช้ `WS-AMOUNT-NUM` (มุมมองตัวเลข) กับข้อมูลที่เป็นช่องว่างล้วนโดยไม่ตรวจสอบก่อน
  พฤติกรรมของโปรแกรมอาจไม่แน่นอน (เช่น แสดงค่าขยะ หรือเกิดข้อผิดพลาดขณะคำนวณในบางกรณี) ขึ้นอยู่กับ
  การตั้งค่าคอมไพเลอร์ ควรตรวจสอบผ่านมุมมองตัวอักษรก่อนใช้งานมุมมองตัวเลขเสมอเมื่อข้อมูลมาจากภายนอก
- เทคนิคนี้เป็นที่นิยมมากในระบบ Mainframe รุ่นเก่าที่ข้อมูลตัวเลขบางครั้งถูกปล่อยว่างไว้แทนการใส่ 0
  ผู้ดูแลระบบ Legacy ควรคุ้นเคยกับรูปแบบนี้เป็นอย่างดี

### แบบฝึกหัดที่ 213.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบ `WS-AMOUNT-RAW = SPACES` จึงต้องทำผ่านมุมมอง `PIC X`
ไม่ใช่ผ่านมุมมอง `PIC 9` โดยตรง

**เฉลยแนวทาง**: เพราะ `SPACES` เป็นค่าตัวอักษร (alphanumeric literal) การเปรียบเทียบกับฟิลด์ที่มี
`PICTURE` เป็น `PIC X` จึงเป็นการเปรียบเทียบที่ตรงชนิดข้อมูลและมีความหมายชัดเจนที่สุด ในขณะที่ฟิลด์
`PIC 9` ถูกออกแบบมาให้เก็บเฉพาะตัวเลข 0-9 เท่านั้น การเปรียบเทียบกับ `SPACES` ผ่านมุมมองนี้อาจได้รับ
การตีความที่ไม่แน่นอนหรือไม่สอดคล้องกับเจตนาที่ต้องการตรวจสอบ

---

## ขั้นตอนที่ 214: REDEFINES กับ Group Items - มองโครงสร้างเดียวกันเป็นคนละแบบ

### แนวคิด

`REDEFINES` ไม่ได้จำกัดอยู่แค่ฟิลด์เดี่ยว (elementary item) เท่านั้น แต่ยังใช้กับ **group item**
(กลุ่มของฟิลด์ย่อยหลายตัว) ได้เช่นกัน ทำให้เราสามารถตีความ record ก้อนเดียวกันเป็น**โครงสร้างที่ต่างกัน
โดยสิ้นเชิง** ได้ ตัวอย่างนี้แสดงให้เห็นทั้งพลังและอันตรายของเทคนิคนี้: ถ้าโครงสร้างทั้งสองแบบไม่ได้
ออกแบบมาให้สอดคล้องกันอย่างตั้งใจ ข้อมูลที่มองผ่านมุมมองที่สองอาจดู "เพี้ยน" ไปจากที่คาดหวัง เพราะขอบเขต
ของฟิลด์ในแต่ละมุมมองไม่ตรงกัน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s4.cob
      *> Purpose : REDEFINES on GROUP items - view the same
      *>           record through two different structures
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-GROUP-ITEMS.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-VIEW.
           05  WS-CUST-FULL-NAME    PIC X(20).
           05  WS-CUST-PHONE        PIC X(10).

       01  WS-SUPPLIER-VIEW REDEFINES WS-CUSTOMER-VIEW.
           05  WS-SUPP-COMPANY      PIC X(15).
           05  WS-SUPP-CONTACT      PIC X(15).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Fill the record through the CUSTOMER structure.
           MOVE "JOHN SMITH"        TO WS-CUST-FULL-NAME
           MOVE "0812345678"        TO WS-CUST-PHONE

           DISPLAY "----- Viewed as CUSTOMER -----"
           DISPLAY "Full name : " WS-CUST-FULL-NAME
           DISPLAY "Phone     : " WS-CUST-PHONE

      *> The SAME 30 bytes, now viewed through a totally
      *> different structure with different field boundaries.
           DISPLAY " "
           DISPLAY "----- Same bytes viewed as SUPPLIER -----"
           DISPLAY "Company   : [" WS-SUPP-COMPANY "]"
           DISPLAY "Contact   : [" WS-SUPP-CONTACT "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-CUSTOMER-VIEW` มีโครงสร้าง 20+10 = 30 ไบต์ (ชื่อเต็ม + เบอร์โทร) ส่วน `WS-SUPPLIER-VIEW`
  มีโครงสร้าง 15+15 = 30 ไบต์ (บริษัท + ผู้ติดต่อ) ทั้งสองมีขนาดรวมเท่ากันพอดี (30 ไบต์) แต่แบ่งขอบเขต
  ฟิลด์ย่อยต่างกันโดยสิ้นเชิง
- เมื่อกรอกข้อมูลผ่านมุมมอง CUSTOMER แล้วอ่านผ่านมุมมอง SUPPLIER จะเห็นว่าขอบเขตของฟิลด์ไม่ตรงกับ
  ความหมายเดิมเลย ข้อมูล "JOHN SMITH" (20 ไบต์) ถูกตัดแบ่งใหม่เป็น 15+5 ไบต์ ทำให้ "WS-SUPP-CONTACT"
  มีข้อมูลบางส่วนของชื่อปนกับเบอร์โทรอย่างไม่มีความหมาย

### ผลลัพธ์ที่ได้จากการรันจริง

```
----- Viewed as CUSTOMER -----
Full name : JOHN SMITH
Phone     : 0812345678

----- Same bytes viewed as SUPPLIER -----
Company   : [JOHN SMITH     ]
Contact   : [     0812345678]
```

### ข้อควรระวัง

- **นี่คือตัวอย่างที่จงใจแสดง "การใช้ REDEFINES ผิดวิธี"** เพื่อให้เห็นอันตรายชัดเจน: การนำ `REDEFINES`
  ไปใช้กับโครงสร้างข้อมูลที่**ไม่มีความสัมพันธ์เชิงความหมายกัน**เลย (เช่น ลูกค้ากับซัพพลายเออร์ในตัวอย่างนี้)
  จะทำให้ได้ข้อมูลที่ไม่มีความหมายเมื่อมองผ่านอีกมุมหนึ่ง
- การใช้งาน `REDEFINES` ที่ถูกต้องในทางปฏิบัติ (ซึ่งเราจะเห็นในขั้นตอนที่ 217) คือการออกแบบให้ทั้งสอง
  (หรือมากกว่า) โครงสร้างมีความสัมพันธ์กันอย่างมีเหตุผล เช่น เป็น record คนละประเภทของธุรกรรมเดียวกัน
  ที่ระบุประเภทไว้อย่างชัดเจนในฟิลด์แรกเสมอ

### แบบฝึกหัดที่ 214.1

**โจทย์**: จงอธิบายว่าทำไมตัวอย่างข้างต้นจึงไม่ใช่การใช้งาน `REDEFINES` ที่เหมาะสมในระบบธุรกิจจริง

**เฉลยแนวทาง**: เพราะ "ลูกค้า" และ "ซัพพลายเออร์" เป็นแนวคิดทางธุรกิจที่แตกต่างกันโดยสิ้นเชิง ไม่มี
เหตุผลใดที่ record เดียวกันควรถูกตีความเป็นทั้งสองอย่างพร้อมกัน การใช้ `REDEFINES` ในลักษณะนี้มีแต่จะ
สร้างความสับสนและบั๊กที่ตรวจจับยาก การใช้งานที่เหมาะสมควรเป็นกรณีที่ข้อมูลมีความสัมพันธ์กันจริง เช่น
เป็นธุรกรรมประเภทต่าง ๆ ของระบบเดียวกัน ที่มีตัวบ่งชี้ (indicator field) ชัดเจนว่าควรใช้มุมมองใด

---

## ขั้นตอนที่ 215: REDEFINES กับขนาดที่แตกต่างกัน

### แนวคิด

โดยหลักการแล้ว `REDEFINES` ควรมีขนาดเท่ากับรายการต้นฉบับเสมอเพื่อความชัดเจน แต่ COBOL ก็อนุญาตให้
ขนาดต่างกันได้เช่นกัน โดยมีกฎว่า **ขนาดของพื้นที่หน่วยความจำทั้งหมดที่จองไว้จะเท่ากับขนาดที่ใหญ่ที่สุด**
ในบรรดารายการต้นฉบับและรายการที่ `REDEFINES` ทั้งหมด ถ้ามุมมองใหม่**เล็กกว่า**เดิม มันจะมองเห็นแค่
บางส่วนของพื้นที่เดิม (ส่วนที่เหลือยังคงมีอยู่จริงแต่ไม่ถูกเปิดเผยผ่านมุมมองนั้น) ถ้ามุมมองใหม่**ใหญ่กว่า**
เดิม COBOL (อย่างน้อยใน GnuCOBOL) จะขยายพื้นที่ที่จองไว้ให้พอดีกับมุมมองที่ใหญ่ที่สุด โดยไม่ไปทับซ้อน
กับตัวแปรถัดไปที่ประกาศหลังจากนั้น

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s5.cob
      *> Purpose : REDEFINES where the new view is a DIFFERENT
      *>           size than the original - the LARGER size wins
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-DIFFERENT-SIZE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> The original item is 10 bytes.
       01  WS-ORIGINAL-10       PIC X(10) VALUE "1234567890".

      *> A SMALLER redefinition only "sees" the first 4 bytes;
      *> the remaining 6 bytes still physically exist but are
      *> simply not covered by this view.
       01  WS-SMALL-VIEW REDEFINES WS-ORIGINAL-10 PIC X(4).

       01  WS-ORIGINAL-4        PIC X(4) VALUE "AB".

      *> A LARGER redefinition makes the group's total size grow
      *> to match the LARGEST redefining item, which can silently
      *> reach into whatever memory follows if not sized with care.
       01  WS-LARGE-VIEW REDEFINES WS-ORIGINAL-4 PIC X(8).

       01  WS-NEXT-FIELD        PIC X(6) VALUE "NEXT!!".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "10-byte original      : [" WS-ORIGINAL-10 "]"
           DISPLAY "Small view (4 bytes)  : [" WS-SMALL-VIEW "]"

           DISPLAY " "
           DISPLAY "4-byte original       : [" WS-ORIGINAL-4 "]"
           DISPLAY "Large view (8 bytes)  : [" WS-LARGE-VIEW "]"
           DISPLAY "Next field, unrelated : [" WS-NEXT-FIELD "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-SMALL-VIEW REDEFINES WS-ORIGINAL-10 PIC X(4)` มองเห็นแค่ 4 ไบต์แรกของ `WS-ORIGINAL-10`
  (ซึ่งมี 10 ไบต์) ไบต์ที่ 5 ถึง 10 ยังคงมีข้อมูลอยู่จริงในหน่วยความจำ แต่ไม่มีชื่อใดอ้างถึงมันในมุมมองนี้
- `WS-LARGE-VIEW REDEFINES WS-ORIGINAL-4 PIC X(8)` ใหญ่กว่า `WS-ORIGINAL-4` (4 ไบต์) ถึง 2 เท่า
  ผลลัพธ์จากการรันจริงแสดงให้เห็นว่า GnuCOBOL **ขยายพื้นที่ที่จองให้กับกลุ่มนี้เป็น 8 ไบต์อัตโนมัติ**
  และ `WS-NEXT-FIELD` ที่ประกาศถัดไปยังคงแสดงค่าของตัวเองอย่างถูกต้อง ไม่ได้ถูกอ่านทับหรือเสียหายแต่อย่างใด

### ผลลัพธ์ที่ได้จากการรันจริง

```
10-byte original      : [1234567890]
Small view (4 bytes)  : [1234]

4-byte original       : [AB  ]
Large view (8 bytes)  : [AB      ]
Next field, unrelated : [NEXT!!]
```

### ข้อควรระวัง

- แม้ผลการทดสอบจริงจะแสดงว่า GnuCOBOL จัดการเรื่องขนาดที่ต่างกันได้อย่างปลอดภัย (ไม่รั่วไปยังตัวแปร
  ถัดไป) แต่ **การพึ่งพาพฤติกรรมนี้เป็นแนวทางการออกแบบที่ไม่ควรทำ** เพราะมาตรฐาน COBOL ไม่ได้รับประกัน
  พฤติกรรมนี้อย่างเป็นทางการเหมือนกันในทุกคอมไพเลอร์ ควรออกแบบให้มุมมอง `REDEFINES` มีขนาด**เท่ากับ**
  รายการต้นฉบับเสมอเป็นแนวปฏิบัติที่ดีที่สุด
- ถ้าจำเป็นต้องมีขนาดต่างกันจริง ๆ (เช่น กรณี variant record ที่แต่ละประเภทมีขนาดต่างกัน) ควรออกแบบ
  ให้มุมมองที่ใหญ่ที่สุดเป็นตัวกำหนดขนาดของ record หลักตั้งแต่แรก เพื่อความชัดเจนและป้องกันความเข้าใจผิด

### แบบฝึกหัดที่ 215.1

**โจทย์**: จงอธิบายว่าทำไมการออกแบบที่ดีควรทำให้ `REDEFINES` มีขนาดเท่ากับรายการต้นฉบับเสมอ
แม้ COBOL จะอนุญาตให้ต่างกันได้

**เฉลยแนวทาง**: เพราะขนาดที่เท่ากันทำให้ผู้อ่านโค้ดเข้าใจได้ทันทีว่าทั้งสองมุมมองครอบคลุมพื้นที่เดียวกัน
พอดีโดยไม่มีส่วนเกินหรือส่วนขาด ลดความเสี่ยงที่จะเข้าใจผิดเกี่ยวกับขอบเขตของข้อมูล และทำให้พฤติกรรม
ของโปรแกรมสอดคล้องกันในทุกคอมไพเลอร์ COBOL โดยไม่ต้องพึ่งพากลไกเสริมเฉพาะของคอมไพเลอร์ตัวใดตัวหนึ่ง

---

## ขั้นตอนที่ 216: เปรียบเทียบ 66-level RENAMES กับ REDEFINES

### แนวคิด

COBOL มีอีกกลไกหนึ่งที่คล้ายกับ `REDEFINES` แต่ทำงานต่างกันคือ **`RENAMES`** (ประกาศด้วย level number
พิเศษคือ **66**) ความแตกต่างสำคัญคือ: **`REDEFINES` ให้คำอธิบายข้อมูลใหม่ทั้งหมด** (สามารถกำหนด
`PICTURE` ใหม่ แบ่งฟิลด์ย่อยใหม่ได้อย่างอิสระ) ในขณะที่ **`RENAMES` เพียงแค่ "จัดกลุ่มใหม่"** ให้กับ
ฟิลด์ที่**มีอยู่แล้ว** โดยไม่มี `PICTURE` เป็นของตัวเอง มันแค่รวมฟิลด์ที่ต่อเนื่องกันหลายตัวให้มีชื่อ
เรียกรวมชื่อเดียว ผ่านวลี **`RENAMES ฟิลด์แรก THRU ฟิลด์สุดท้าย`**

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s6.cob
      *> Purpose : Compare 66-level RENAMES with REDEFINES -
      *>           RENAMES only re-groups EXISTING fields, it
      *>           cannot change their PICTURE or add new ones
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RENAMES-VS-REDEFINES.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FULL-NAME.
           05  WS-FIRST-NAME    PIC X(10) VALUE "JOHN".
           05  WS-MIDDLE-NAME   PIC X(10) VALUE "ROBERT".
           05  WS-LAST-NAME     PIC X(10) VALUE "SMITH".

      *> RENAMES (level 66) groups two or more EXISTING elementary
      *> items under one new name, WITHOUT any new PICTURE clause.
      *> It must be written right after the group it renames from.
       66  WS-GIVEN-NAMES RENAMES WS-FIRST-NAME THRU WS-MIDDLE-NAME.

      *> REDEFINES, by contrast, gives a COMPLETELY new data
      *> description (new PICTURE, new sub-fields) to the same
      *> bytes - here re-slicing all 30 bytes into two 15-byte
      *> halves instead of the original three 10-byte names.
       01  WS-FULL-NAME-AS-HALVES REDEFINES WS-FULL-NAME.
           05  WS-LEFT-HALF     PIC X(15).
           05  WS-RIGHT-HALF    PIC X(15).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "Original 3 fields  : ["
               WS-FIRST-NAME "][" WS-MIDDLE-NAME "]["
               WS-LAST-NAME "]"

      *> WS-GIVEN-NAMES is simply FIRST-NAME + MIDDLE-NAME viewed
      *> as one combined field; no new PICTURE was ever declared.
           DISPLAY "RENAMES view (given names): [" WS-GIVEN-NAMES "]"

      *> WS-FULL-NAME-AS-HALVES re-slices the SAME 30 bytes using
      *> a brand-new PICTURE layout that ignores the original
      *> 10-10-10 field boundaries entirely.
           DISPLAY "REDEFINES view (15-15 halves):"
           DISPLAY "  Left  : [" WS-LEFT-HALF "]"
           DISPLAY "  Right : [" WS-RIGHT-HALF "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `66 WS-GIVEN-NAMES RENAMES WS-FIRST-NAME THRU WS-MIDDLE-NAME.` สร้างชื่อ `WS-GIVEN-NAMES` ที่หมายถึง
  "`WS-FIRST-NAME` ต่อด้วย `WS-MIDDLE-NAME`" (รวม 20 ไบต์) โดย**ไม่มี** `PICTURE` เป็นของตัวเอง
  มันแค่อ้างอิงถึงฟิลด์ที่มีอยู่แล้วสองตัวติดกัน
- `WS-FULL-NAME-AS-HALVES REDEFINES WS-FULL-NAME` สร้างมุมมองใหม่ทั้งหมด (30 ไบต์แบ่งเป็น 15+15)
  ซึ่งไม่สนใจขอบเขตเดิม (10+10+10) เลย นี่คือความแตกต่างสำคัญ: `RENAMES` เคารพขอบเขตฟิลด์เดิม
  ในขณะที่ `REDEFINES` เขียนกฎขอบเขตใหม่ทั้งหมด

### ผลลัพธ์ที่ได้จากการรันจริง

```
Original 3 fields  : [JOHN      ][ROBERT    ][SMITH     ]
RENAMES view (given names): [JOHN      ROBERT    ]
REDEFINES view (15-15 halves):
  Left  : [JOHN      ROBER]
  Right : [T    SMITH     ]
```

### ข้อควรระวัง

- สังเกตว่า `WS-LEFT-HALF` ตัดขาดคำว่า "ROBERT" ไปครึ่งหนึ่ง ("ROBER") เพราะขอบเขต 15 ไบต์ไม่ได้
  ตรงกับขอบเขตฟิลด์เดิมเลย นี่คือตัวอย่างที่ชัดเจนว่า `REDEFINES` ไม่สนใจโครงสร้างเดิมใด ๆ ทั้งสิ้น
  ในขณะที่ `RENAMES` จะไม่มีปัญหานี้เพราะมันแค่รวมฟิลด์ที่มีอยู่ทั้งก้อน ไม่ตัดแบ่งกลางฟิลด์
- `RENAMES` (level 66) ต้องเขียนอยู่**ท้ายกลุ่ม**ที่มันอ้างอิงถึง (หลังฟิลด์ย่อยทั้งหมดของกลุ่มนั้น)
  และใช้ได้เฉพาะกับฟิลด์ที่อยู่ในระดับกลุ่มเดียวกันเท่านั้น มีข้อจำกัดเรื่องตำแหน่งที่เข้มงวดกว่า
  `REDEFINES` พอสมควร

### แบบฝึกหัดที่ 216.1

**โจทย์**: จงสรุปเป็นกฎ 1 ประโยคว่าเมื่อไรควรใช้ `RENAMES` และเมื่อไรควรใช้ `REDEFINES`

**เฉลยแนวทาง**: ควรใช้ `RENAMES` เมื่อต้องการแค่ "เรียกกลุ่มฟิลด์ที่มีอยู่แล้วด้วยชื่อรวมใหม่" โดยไม่
เปลี่ยนแปลงขอบเขตหรือชนิดข้อมูลใด ๆ และควรใช้ `REDEFINES` เมื่อต้องการ "ตีความข้อมูลเดียวกันด้วย
โครงสร้างหรือชนิดข้อมูลที่ต่างไปจากเดิมโดยสิ้นเชิง"

---

## ขั้นตอนที่ 217: ใช้ REDEFINES จัดการ Record หลายประเภท (Variant Records)

### แนวคิด

นี่คือการใช้งาน `REDEFINES` ที่**พบบ่อยที่สุดและมีประโยชน์ที่สุด**ในระบบธุรกิจจริง: การประมวลผล
**record ที่มีหลายรูปแบบปนกันในไฟล์เดียว** โดยมีฟิลด์แรกสุดเป็น**ตัวบ่งชี้ประเภท (type indicator)**
ที่บอกว่า record นี้ควรตีความด้วยโครงสร้างใด เทคนิคนี้พบได้บ่อยมากในไฟล์ธุรกรรมทางการเงิน ที่แต่ละ
แถวอาจเป็นรายการฝากเงิน ถอนเงิน หรือโอนเงิน ซึ่งแต่ละประเภทมีฟิลด์ที่ต้องใช้ต่างกัน แต่ถูกจัดเก็บใน
พื้นที่ที่มีขนาดเท่ากันเพื่อความสม่ำเสมอของไฟล์

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s7.cob
      *> Purpose : Classic REDEFINES use case - a variant record
      *>           whose layout depends on a transaction-type code
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-VARIANT-RECORD.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  TRANSACTION-RECORD.
           05  TXN-TYPE             PIC X(1).
           05  TXN-DEPOSIT-DATA.
               10  TXN-DEP-ACCOUNT  PIC X(8).
               10  TXN-DEP-AMOUNT   PIC 9(7)V99.

       01  TRANSFER-VIEW REDEFINES TRANSACTION-RECORD.
           05  TXN2-TYPE                PIC X(1).
           05  TXN-TRANSFER-DATA.
               10  TXN-FROM-ACCOUNT     PIC X(8).
               10  TXN-TO-ACCOUNT       PIC X(6).
               10  TXN-TRANSFER-AMOUNT  PIC 9(5)V99.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Build a "D" (deposit) type transaction using the first
      *> layout - account number plus a 7.2-digit amount.
           MOVE "D"          TO TXN-TYPE
           MOVE "ACC00012"   TO TXN-DEP-ACCOUNT
           MOVE 1500.50      TO TXN-DEP-AMOUNT

           DISPLAY "Record type    : " TXN-TYPE
           DISPLAY "Deposit account: " TXN-DEP-ACCOUNT
           DISPLAY "Deposit amount : " TXN-DEP-AMOUNT

      *> Build a "T" (transfer) type transaction instead, which
      *> uses a COMPLETELY different field layout for the SAME
      *> physical record, chosen based on what TXN-TYPE holds.
           MOVE "T"          TO TXN2-TYPE
           MOVE "ACC00099"   TO TXN-FROM-ACCOUNT
           MOVE "ACC777"     TO TXN-TO-ACCOUNT
           MOVE 250.00       TO TXN-TRANSFER-AMOUNT

           DISPLAY " "
           DISPLAY "Record type    : " TXN2-TYPE
           DISPLAY "From account   : " TXN-FROM-ACCOUNT
           DISPLAY "To account     : " TXN-TO-ACCOUNT
           DISPLAY "Transfer amount: " TXN-TRANSFER-AMOUNT

      *> A real program would branch on TXN-TYPE (read from a
      *> file) to decide WHICH view is the correct one to trust.
           DISPLAY " "
           EVALUATE TXN2-TYPE
               WHEN "D"
                   DISPLAY "Interpreting record as a DEPOSIT."
               WHEN "T"
                   DISPLAY "Interpreting record as a TRANSFER."
               WHEN OTHER
                   DISPLAY "Unknown transaction type."
           END-EVALUATE

           STOP RUN.
```

### คำอธิบายโค้ด

- `TXN-TYPE` และ `TXN2-TYPE` เป็นฟิลด์เดียวกัน (ไบต์แรกสุด) ที่มองผ่านทั้งสองมุมมอง มันทำหน้าที่เป็น
  **ตัวบ่งชี้ประเภท** ที่โปรแกรมต้องอ่านก่อนเสมอ เพื่อตัดสินใจว่าจะใช้มุมมอง `TXN-DEPOSIT-DATA` หรือ
  `TXN-TRANSFER-DATA` ในการตีความไบต์ที่เหลือ
- `EVALUATE TXN2-TYPE` คือรูปแบบมาตรฐานของการเขียนโปรแกรมประมวลผล variant record: อ่านค่าตัวบ่งชี้
  ก่อน แล้วจึงแตกกิ่งไปยังโค้ดที่ใช้มุมมองที่ถูกต้องสำหรับประเภทนั้น ๆ

### ผลลัพธ์ที่ได้จากการรันจริง

```
Record type    : D
Deposit account: ACC00012
Deposit amount : 0001500.50

Record type    : T
From account   : ACC00099
To account     : ACC777
Transfer amount: 00250.00

Interpreting record as a TRANSFER.
```

### ข้อควรระวัง

- **ตัวบ่งชี้ประเภท (type indicator) ต้องอยู่ตำแหน่งเดียวกันในทุกมุมมองเสมอ** (ในตัวอย่างนี้คือไบต์แรก
  สุด) เพราะโปรแกรมต้องอ่านมันได้อย่างถูกต้องไม่ว่า record จะเป็นประเภทใดก็ตาม ถ้าตัวบ่งชี้ไม่อยู่
  ตำแหน่งเดียวกันในทุกมุมมอง ระบบทั้งหมดจะพังทันที
- โปรแกรมที่ประมวลผล variant record **ต้องตรวจสอบตัวบ่งชี้ก่อนเสมอ** ก่อนเข้าถึงฟิลด์ใด ๆ ในมุมมองที่
  ต้องการ ห้ามสมมติว่า record ที่อ่านเข้ามาจะตรงกับประเภทที่คาดหวังโดยไม่ตรวจสอบ

### แบบฝึกหัดที่ 217.1

**โจทย์**: จงเพิ่มมุมมองที่สามชื่อ `WITHDRAWAL-VIEW` สำหรับรายการถอนเงิน (type = "W") ที่มีโครงสร้าง
เหมือนกับ `TRANSACTION-RECORD` เดิม (account + amount) แต่ใช้ชื่อฟิลด์ต่างออกไป

**เฉลยแนวทาง**:
```cobol
       01  WITHDRAWAL-VIEW REDEFINES TRANSACTION-RECORD.
           05  TXN3-TYPE                PIC X(1).
           05  TXN-WITHDRAWAL-DATA.
               10  TXN-WD-ACCOUNT       PIC X(8).
               10  TXN-WD-AMOUNT        PIC 9(7)V99.
```
เพิ่มการประกาศนี้ต่อจาก `TRANSFER-VIEW` แล้วเพิ่ม `WHEN "W"` ในคำสั่ง `EVALUATE` เพื่อจัดการกรณี
ถอนเงินด้วยเช่นกัน ข้อสำคัญคือ `TXN3-TYPE` ต้องยังคงอยู่ตำแหน่งไบต์แรกสุดเหมือนมุมมองอื่น ๆ ทั้งหมด

---

## ขั้นตอนที่ 218: REDEFINES กับ FILLER - เปิดเผยเฉพาะไบต์ที่ต้องการ

### แนวคิด

เมื่อทำงานกับ record แบบ fixed-width จากระบบ Legacy บางครั้งเราสนใจแค่บางส่วนของ record เท่านั้น
ไม่จำเป็นต้องตั้งชื่อให้ครบทุกไบต์ COBOL อนุญาตให้ใช้คำว่า **`FILLER`** (หรือละเว้นชื่อไปเลยตั้งแต่
COBOL-2002) แทนชื่อของฟิลด์ที่เรา**ไม่สนใจ**เนื้อหา ทำให้โครงสร้างที่ redefine ดูสะอาดขึ้นมาก
เพราะแสดงเฉพาะฟิลด์ที่มีความหมายต่อโปรแกรมจริง ๆ เท่านั้น

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s8.cob
      *> Purpose : REDEFINES combined with FILLER to expose only
      *>           the byte ranges of a fixed record we care about
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-WITH-FILLER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> A 20-byte legacy record: only bytes 6-10 (the account
      *> code) and bytes 16-20 (the status flag) are useful here.
       01  LEGACY-RECORD        PIC X(20)
           VALUE "HDR01ACC123XXXXXAKTV".

       01  LEGACY-FIELDS REDEFINES LEGACY-RECORD.
           05  FILLER           PIC X(5).
           05  LR-ACCOUNT-CODE  PIC X(6).
           05  FILLER           PIC X(5).
           05  LR-STATUS-FLAG   PIC X(4).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> FILLER marks the byte ranges we deliberately ignore.
      *> Only LR-ACCOUNT-CODE and LR-STATUS-FLAG are given real
      *> names, because those are the only parts this program
      *> actually needs to read.
           DISPLAY "Full raw record : [" LEGACY-RECORD "]"
           DISPLAY "Account code    : [" LR-ACCOUNT-CODE "]"
           DISPLAY "Status flag     : [" LR-STATUS-FLAG "]"

           IF LR-STATUS-FLAG = "AKTV"
               DISPLAY "This account is ACTIVE."
           ELSE
               DISPLAY "This account is NOT active."
           END-IF

           STOP RUN.
```

### คำอธิบายโค้ด

- `LEGACY-RECORD` มีข้อมูลดิบ 20 ไบต์ที่มาจากระบบเก่า แต่โปรแกรมนี้สนใจแค่ 2 ส่วนเท่านั้นคือรหัส
  บัญชี (ไบต์ 6-11) และสถานะ (ไบต์ 17-20)
- `FILLER` ใช้แทนที่ส่วนที่ไม่สนใจ (ไบต์ 1-5 และ 12-16) สามารถใช้ชื่อ `FILLER` ซ้ำกันได้หลายครั้งใน
  โปรแกรมเดียวกัน (ต่างจากชื่อฟิลด์ปกติที่ต้องไม่ซ้ำกัน) เพราะ COBOL รู้ว่าไม่มีใครจะอ้างอิงถึง `FILLER`
  โดยตรงอยู่แล้ว

### ผลลัพธ์ที่ได้จากการรันจริง

```
Full raw record : [HDR01ACC123XXXXXAKTV]
Account code    : [ACC123]
Status flag     : [AKTV]
This account is ACTIVE.
```

### ข้อควรระวัง

- ต้องนับความยาวของแต่ละ `FILLER` และฟิลด์ที่มีชื่อให้ถูกต้องแม่นยำ เพราะถ้าความยาวรวมไม่ตรงกับ
  ตำแหน่งจริงของข้อมูลใน record ต้นฉบับ ฟิลด์ที่มีชื่อจะอ่านไบต์ผิดตำแหน่งไปโดยไม่มี error ใด ๆ เตือน
  (compile ผ่านและรันได้ปกติ แต่ข้อมูลผิด) ควรตรวจสอบผลรวมความยาวของทุกฟิลด์ (รวม FILLER) ให้เท่ากับ
  ขนาดของ record ต้นฉบับเสมอ
- ตั้งแต่ COBOL-2002 เป็นต้นมา สามารถละคำว่า `FILLER` ไปเลยได้ (เขียนแค่ `05 PIC X(5).`) แต่การเขียน
  `FILLER` ให้ชัดเจนยังคงเป็นธรรมเนียมที่นิยมในองค์กร เพราะช่วยให้อ่านโค้ดเข้าใจง่ายกว่า

### แบบฝึกหัดที่ 218.1

**โจทย์**: จงคำนวณว่าฟิลด์ `FILLER` ตัวที่สอง (ไบต์ 12-16) ในตัวอย่างข้างต้นมีความยาวเท่าไร
และครอบคลุมข้อความส่วนใดของ `LEGACY-RECORD`

**เฉลย**: `FILLER` ตัวที่สองมีความยาว 5 ไบต์ (`PIC X(5)`) ครอบคลุมไบต์ที่ 12 ถึง 16 ของ
`LEGACY-RECORD` ซึ่งตรงกับข้อความ "XXXXX" ในค่าตัวอย่าง "HDR01ACC123XXXXXAKTV"
(นับจาก HDR01=1-5, ACC123=6-11, XXXXX=12-16, AKTV=17-20)

---

## ขั้นตอนที่ 219: ข้อควรระวังสำคัญ - VALUE Clause และ INITIALIZE กับ REDEFINES

### แนวคิด

ขั้นตอนนี้จะเจาะลึกสองกับดักที่อันตรายที่สุดของ `REDEFINES` ที่ผู้เริ่มต้นมักไม่ทันสังเกต:

1. **`VALUE` clause บนรายการที่ `REDEFINES` จะถูก**เพิกเฉย (ignored)**โดยมาตรฐาน COBOL เสมอ**
   มีเพียง `VALUE` ของรายการต้นฉบับเท่านั้นที่มีผลตอนโปรแกรมเริ่มทำงาน
2. **เนื่องจากทั้งสองมุมมองใช้พื้นที่เดียวกัน การ `INITIALIZE` (หรือ `MOVE`) ผ่านมุมมองใดมุมมองหนึ่ง
   จะกระทบข้อมูลที่มองผ่านอีกมุมมองเสมอทันที** ไม่มีข้อมูลใดเป็น "อิสระ" จากกันเลย

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s9.cob
      *> Purpose : Two common REDEFINES pitfalls - a VALUE clause
      *>           on the redefining item is silently ignored, and
      *>           INITIALIZE on either view affects the SAME bytes
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REDEFINES-PITFALLS.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Pitfall 1: a VALUE clause on a REDEFINES item is ignored
      *> by the COBOL standard. Only the ORIGINAL item's VALUE
      *> takes effect when the program starts.
       01  WS-ORIGINAL          PIC X(6) VALUE "ABCDEF".
       01  WS-BAD-VIEW REDEFINES WS-ORIGINAL PIC X(6)
           VALUE "ZZZZZZ".

      *> Pitfall 2: because both views share the same storage,
      *> INITIALIZE (or MOVE) through EITHER name changes what
      *> the OTHER name sees too - there is only one set of bytes.
       01  WS-RECORD.
           05  WS-REC-CODE      PIC X(4) VALUE "CODE".
           05  WS-REC-AMOUNT    PIC 9(5) VALUE 100.

       01  WS-RECORD-AS-TEXT REDEFINES WS-RECORD PIC X(9).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "Pitfall 1: VALUE on REDEFINES is ignored"
           DISPLAY "  Expected 'ZZZZZZ', but WS-BAD-VIEW = ["
               WS-BAD-VIEW "]"
           DISPLAY "  (It shows the ORIGINAL item's value instead.)"

           DISPLAY " "
           DISPLAY "Pitfall 2: both views share the same bytes"
           DISPLAY "  Before INITIALIZE: WS-REC-CODE=["
               WS-REC-CODE "] WS-REC-AMOUNT=" WS-REC-AMOUNT
           DISPLAY "  As text view      : [" WS-RECORD-AS-TEXT "]"

      *> INITIALIZE through the ORIGINAL structure resets each
      *> elementary item to ITS OWN default (spaces or zero).
           INITIALIZE WS-RECORD
           DISPLAY "  After INITIALIZE WS-RECORD:"
           DISPLAY "  WS-REC-CODE=[" WS-REC-CODE
               "] WS-REC-AMOUNT=" WS-REC-AMOUNT
           DISPLAY "  As text view      : [" WS-RECORD-AS-TEXT "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-BAD-VIEW REDEFINES WS-ORIGINAL PIC X(6) VALUE "ZZZZZZ"` เขียน `VALUE "ZZZZZZ"` ไว้ แต่เมื่อ
  compile จริงด้วย flag `-Wall` จะเห็นคำเตือนทันที และค่าที่แสดงผลจริงคือ "ABCDEF" (ค่าของ
  `WS-ORIGINAL`) ไม่ใช่ "ZZZZZZ" ตามที่เขียนไว้เลย
- `INITIALIZE WS-RECORD` รีเซ็ต `WS-REC-CODE` กลับเป็นช่องว่าง (default ของ `PIC X`) และ
  `WS-REC-AMOUNT` กลับเป็น 0 (default ของ `PIC 9`) ตามการประกาศของรายการต้นฉบับ และเนื่องจาก
  `WS-RECORD-AS-TEXT` มองพื้นที่เดียวกัน มันก็เห็นการเปลี่ยนแปลงนี้ทันทีเช่นกัน

### ผลลัพธ์ที่ได้จากการรันจริง

เมื่อ compile ด้วย `cobc -x -Wall -o s9 s9.cob` จะเห็นคำเตือนจริงจากคอมไพเลอร์ก่อนรัน:

```
s9.cob:16: warning: initial VALUE clause ignored for REDEFINES item 'WS-BAD-VIEW'
```

และผลลัพธ์จากการรันโปรแกรมคือ:

```
Pitfall 1: VALUE on REDEFINES is ignored
  Expected 'ZZZZZZ', but WS-BAD-VIEW = [ABCDEF]
  (It shows the ORIGINAL item's value instead.)

Pitfall 2: both views share the same bytes
  Before INITIALIZE: WS-REC-CODE=[CODE] WS-REC-AMOUNT=00100
  As text view      : [CODE00100]
  After INITIALIZE WS-RECORD:
  WS-REC-CODE=[    ] WS-REC-AMOUNT=00000
  As text view      : [    00000]
```

### ข้อควรระวัง

- **ห้ามพึ่งพา `VALUE` clause บนรายการที่ `REDEFINES` โดยเด็ดขาด** เพราะมันจะถูกเพิกเฉยเสมอตาม
  มาตรฐาน COBOL แม้บางคอมไพเลอร์อาจไม่แจ้งเตือนถ้าไม่เปิด flag พิเศษ (เช่น `-Wall`) ก็ตาม ถ้าต้องการ
  ค่าเริ่มต้นที่แน่นอน ให้กำหนด `VALUE` ไว้ที่รายการ**ต้นฉบับ**เท่านั้น
- ก่อนเรียก `INITIALIZE` หรือ `MOVE` ผ่านมุมมองใดมุมมองหนึ่งของ `REDEFINES` ต้องตระหนักเสมอว่าการ
  เปลี่ยนแปลงจะกระทบทุกมุมมองที่แชร์พื้นที่เดียวกันทันที ไม่มีทาง "แก้ไขแค่มุมมองเดียว" ได้เลย
  เพราะโดยพื้นฐานแล้วมันคือข้อมูลชุดเดียวกัน

### แบบฝึกหัดที่ 219.1

**โจทย์**: จงอธิบายว่าทำไม COBOL ถึงออกแบบให้ `VALUE` บนรายการ `REDEFINES` ถูกเพิกเฉย แทนที่จะ
อนุญาตให้ใช้งานได้ตามปกติ

**เฉลยแนวทาง**: เพราะพื้นที่หน่วยความจำหนึ่งก้อนสามารถมีค่าเริ่มต้นได้เพียงค่าเดียวเท่านั้นเมื่อโปรแกรม
เริ่มทำงาน ถ้าทั้งรายการต้นฉบับและรายการ `REDEFINES` (รวมถึงมุมมองอื่นที่อาจมีอีกหลายตัว) ต่างก็มี
`VALUE` ของตัวเอง จะเกิดความขัดแย้งทันทีว่าค่าใดควรมีผลจริง มาตรฐาน COBOL จึงเลือกให้ความสำคัญกับ
รายการต้นฉบับเพียงตัวเดียวเสมอ เพื่อไม่ให้เกิดความกำกวมในการกำหนดค่าเริ่มต้น

---

## ขั้นตอนที่ 220: โปรแกรมสรุป - ระบบอ่าน Record หลายประเภทด้วย REDEFINES

### แนวคิด

ขั้นตอนสุดท้ายของ Part นี้จะรวบยอดทุกอย่างที่เรียนมาเป็นโปรแกรมประมวลผล**ชุดข้อมูลธุรกรรม (batch)**
ที่มีทั้งรายการฝากและถอนปนกัน โดยจำลองข้อมูลแบบ fixed-width ที่ได้รับมา (เหมือนกับที่จะอ่านจากไฟล์จริง
ใน Part 023 ถัดไป) แล้วใช้ `REDEFINES` ร่วมกับ `EVALUATE` เพื่อตีความและสรุปยอดรวมแต่ละประเภท

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s10.cob
      *> Purpose : Final summary - read a batch of fixed-width
      *>           variant records and interpret each one using
      *>           the REDEFINES view that matches its type code
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. VARIANT-RECORD-READER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Simulates 3 fixed-width, 21-byte "records" that would
      *> normally come from a file (Part 023 will read real files).
       01  RAW-BATCH.
           05  RAW-LINE PIC X(21) OCCURS 3 TIMES.

       01  WS-IDX               PIC 9 VALUE 1.
       01  WS-DEPOSIT-TOTAL     PIC 9(7)V99 VALUE 0.
       01  WS-WITHDRAW-TOTAL    PIC 9(7)V99 VALUE 0.

      *> One working record, filled fresh from RAW-LINE each loop,
      *> then interpreted through whichever REDEFINES view matches
      *> its type code.
       01  WORK-RECORD.
           05  WK-TYPE              PIC X(1).
           05  WK-DEPOSIT-DATA.
               10  WK-DEP-ACCOUNT   PIC X(8).
               10  WK-DEP-AMOUNT    PIC 9(7)V99.
               10  FILLER           PIC X(3).

       01  WORK-WITHDRAW-VIEW REDEFINES WORK-RECORD.
           05  WK2-TYPE                PIC X(1).
           05  WK-WITHDRAW-DATA.
               10  WK-WD-ACCOUNT       PIC X(8).
               10  WK-WD-AMOUNT        PIC 9(7)V99.
               10  FILLER              PIC X(3).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> "D" record: type + 8-char account + 9-digit amount.
           MOVE "DACC00012000150050" TO RAW-LINE(1)
      *> "W" record: same shape, reused for withdrawals.
           MOVE "WACC00099000050000" TO RAW-LINE(2)
           MOVE "DACC00034000320075" TO RAW-LINE(3)

           DISPLAY "===== Transaction Batch Report ====="
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               MOVE RAW-LINE(WS-IDX) TO WORK-RECORD

               EVALUATE WK-TYPE
                   WHEN "D"
                       DISPLAY "Record " WS-IDX
                           ": DEPOSIT  acct=" WK-DEP-ACCOUNT
                           " amount=" WK-DEP-AMOUNT
                       ADD WK-DEP-AMOUNT TO WS-DEPOSIT-TOTAL
                   WHEN "W"
                       DISPLAY "Record " WS-IDX
                           ": WITHDRAW acct=" WK-WD-ACCOUNT
                           " amount=" WK-WD-AMOUNT
                       ADD WK-WD-AMOUNT TO WS-WITHDRAW-TOTAL
                   WHEN OTHER
                       DISPLAY "Record " WS-IDX
                           ": UNKNOWN type [" WK-TYPE "]"
               END-EVALUATE
           END-PERFORM

           DISPLAY " "
           DISPLAY "Total deposits    : " WS-DEPOSIT-TOTAL
           DISPLAY "Total withdrawals : " WS-WITHDRAW-TOTAL

           STOP RUN.
```

### คำอธิบายโค้ด

- `RAW-BATCH` จำลองข้อมูล 3 แถวแบบ fixed-width ขนาด 21 ไบต์ต่อแถว (ประเภท 1 ไบต์ + บัญชี 8 ไบต์ +
  จำนวนเงิน 9 หลัก + ช่องว่างสำรอง 3 ไบต์) เหมือนกับข้อมูลที่มักพบในไฟล์ Mainframe จริง
- ทุกรอบของลูป `WORK-RECORD` จะถูก `MOVE` ค่าจาก `RAW-LINE(WS-IDX)` ใหม่ทุกครั้ง แล้วตรวจสอบ
  `WK-TYPE` (ตัวบ่งชี้ประเภท) ก่อนตัดสินใจว่าจะใช้มุมมองฝากเงิน (`WK-DEP-...`) หรือถอนเงิน
  (`WK-WD-...`) ผ่าน `WORK-WITHDRAW-VIEW REDEFINES WORK-RECORD`
- ยอดรวมของแต่ละประเภท (`WS-DEPOSIT-TOTAL`, `WS-WITHDRAW-TOTAL`) ถูกสะสมแยกกันตามประเภทของแต่ละ
  record ที่อ่านเข้ามา แสดงให้เห็นการประยุกต์ใช้ `REDEFINES` ร่วมกับ `EVALUATE` และการสะสมยอดในตัว
  โปรแกรมประมวลผลชุดข้อมูลจริง

### ผลลัพธ์ที่ได้จากการรันจริง

```
===== Transaction Batch Report =====
Record 1: DEPOSIT  acct=ACC00012 amount=0001500.50
Record 2: WITHDRAW acct=ACC00099 amount=0000500.00
Record 3: DEPOSIT  acct=ACC00034 amount=0003200.75

Total deposits    : 0004701.25
Total withdrawals : 0000500.00
```

### ข้อควรระวัง

- ในตัวอย่างนี้ `WORK-RECORD` ถูก `MOVE` ค่าใหม่ทุกรอบของลูปก่อนตรวจสอบประเภท ซึ่งถูกต้อง เพราะถ้า
  ลืม `MOVE` ค่าปัจจุบันเข้ามาก่อน ข้อมูลของรอบก่อนหน้าจะยังค้างอยู่และถูกตีความผิดประเภทได้
- ในระบบจริงที่อ่านข้อมูลจากไฟล์ (ซึ่งเราจะเรียนใน Part 023) รูปแบบการเขียนโปรแกรมจะคล้ายกันมาก:
  อ่าน record เข้ามาที่ตัวแปรกลาง ตรวจสอบตัวบ่งชี้ประเภท แล้วตีความผ่านมุมมอง `REDEFINES` ที่เหมาะสม
  ก่อนประมวลผลต่อ — นี่คือเหตุผลที่ `REDEFINES` เป็นทักษะสำคัญที่ต้องเข้าใจให้แม่นก่อนเข้าสู่เนื้อหา
  เรื่องไฟล์ในบทถัดไป

### แบบฝึกหัดสรุปรวมที่ 220.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น โดยเพิ่มการนับจำนวนรายการแต่ละประเภท (จำนวนรายการฝาก และจำนวน
รายการถอน) แล้วแสดงผลสรุปท้ายรายงานร่วมกับยอดรวมเดิม

**เฉลยแนวทาง**: ประกาศตัวแปรใหม่ `01 WS-DEPOSIT-COUNT PIC 9 VALUE 0.` และ
`01 WS-WITHDRAW-COUNT PIC 9 VALUE 0.` แล้วเพิ่ม `ADD 1 TO WS-DEPOSIT-COUNT` ในกิ่ง `WHEN "D"`
และ `ADD 1 TO WS-WITHDRAW-COUNT` ในกิ่ง `WHEN "W"` ของคำสั่ง `EVALUATE` จากนั้นหลังจบ
`PERFORM VARYING` ให้เพิ่ม `DISPLAY "Deposit count: " WS-DEPOSIT-COUNT` และ
`DISPLAY "Withdraw count: " WS-WITHDRAW-COUNT` ผลลัพธ์จากข้อมูลตัวอย่างนี้จะแสดง
"Deposit count: 2" และ "Withdraw count: 1" ตามลำดับ

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้ `REDEFINES` Clause และการใช้หน่วยความจำซ้ำอย่างครบถ้วน ได้แก่:

- แนวคิดพื้นฐานของ `REDEFINES` ในการมองข้อมูลชุดเดียวกันในหน่วยความจำเป็นหลายมุมมอง
- กฎพื้นฐานสามข้อของ `REDEFINES` (level เดียวกัน, เขียนตามหลังทันที, ชื่อไม่ซ้ำ) พร้อม error จริง
  เมื่อฝ่าฝืนกฎเรื่องลำดับการเขียน
- การใช้ `REDEFINES` มองฟิลด์เดียวกันเป็นตัวเลขและตัวอักษรสลับกัน เพื่อตรวจสอบข้อมูลก่อนใช้งานจริง
- การใช้ `REDEFINES` กับ group items และอันตรายเมื่อโครงสร้างสองแบบไม่มีความสัมพันธ์กันจริง
- ผลกระทบเมื่อ `REDEFINES` มีขนาดต่างจากรายการต้นฉบับ (เล็กกว่า/ใหญ่กว่า)
- ความแตกต่างระหว่าง 66-level `RENAMES` กับ `REDEFINES`
- การใช้ `REDEFINES` จัดการ variant record ตามตัวบ่งชี้ประเภท (type indicator) ซึ่งเป็นการใช้งานจริง
  ที่พบบ่อยที่สุด
- การใช้ `FILLER` ร่วมกับ `REDEFINES` เพื่อเปิดเผยเฉพาะไบต์ที่ต้องการจาก record แบบ fixed-width
- ข้อควรระวังสำคัญ: `VALUE` บน `REDEFINES` ถูกเพิกเฉยเสมอ และการแก้ไขผ่านมุมมองใดกระทบทุกมุมมอง
- โปรแกรมสรุประบบอ่าน record หลายประเภทแบบ batch พร้อมการสะสมยอดรวมแยกตามประเภท

ทักษะเรื่อง `REDEFINES` นี้เป็นพื้นฐานสำคัญมากที่จะถูกใช้ต่อยอดตลอดเฟส 2 ของหลักสูตร โดยเฉพาะใน
**Part 023** ที่เราจะเริ่มเรียนรู้เรื่อง **ไฟล์ตามลำดับ (Sequential Files)** ซึ่งการอ่าน record
จากไฟล์จริงมักต้องใช้ `REDEFINES` ร่วมกับ `FD` Entry เพื่อตีความข้อมูลที่มีหลายรูปแบบปนกันในไฟล์เดียว
ตามหลักการเดียวกับที่เราเพิ่งเรียนรู้ไปในบทนี้

**[← กลับไป Part 021](part-021-inspect-statement.md)** |
**[ไปยัง Part 023: ไฟล์ตามลำดับ (Sequential Files) เบื้องต้น →](part-023-sequential-files.md)**
