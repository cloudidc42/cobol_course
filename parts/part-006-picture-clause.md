# Part 006: PICTURE Clause และชนิดข้อมูลตัวเลข/ตัวอักษร (ขั้นตอนที่ 51–60)

## คำนำของ Part นี้

ใน Part 005 คุณได้เรียนรู้ WORKING-STORAGE SECTION แล้วว่าเป็นที่เก็บตัวแปรของโปรแกรม และได้เห็น
`PIC` ปรากฏอยู่ในเกือบทุกตัวอย่างที่ผ่านมา แต่เรายังไม่ได้เจาะลึกว่า `PIC X`, `PIC 9`, `PIC A`,
เครื่องหมาย `V`, เครื่องหมาย `S` และตัวเลขในวงเล็บมีความหมายและกฎเกณฑ์ที่แน่นอนอย่างไรบ้าง

**PICTURE clause** (เขียนย่อว่า `PIC`) คือหัวใจสำคัญที่สุดอย่างหนึ่งของ COBOL เพราะมันกำหนดพร้อมกันถึง
3 สิ่งในคำประกาศเดียว: **ชนิดของข้อมูล** (ตัวเลขล้วน, ตัวอักษรล้วน, หรือข้อความทั่วไป), **ขนาดที่แน่นอน**
เป็นไบต์ และ**รูปแบบการจัดเก็บ** (มีจุดทศนิยมโดยนัยหรือไม่ มีเครื่องหมายบวก/ลบหรือไม่) ทุกอย่างที่คุณ
ประกาศผิดพลาดใน PICTURE clause จะส่งผลกระทบต่อเนื่องไปยังการคำนวณ การเปรียบเทียบ และการแสดงผล
ทั้งหมดของโปรแกรม การเข้าใจ PICTURE clause อย่างละเอียดจึงเป็นการลงทุนที่คุ้มค่าที่สุดอย่างหนึ่งในการ
เรียน COBOL

Part นี้จะพาคุณไล่เรียงตั้งแต่โครงสร้างพื้นฐานของ PICTURE clause, ชนิดข้อมูลหลักทั้งสามแบบ (`9`, `X`, `A`),
เทคนิคการย่อด้วยตัวเลขซ้ำ, จุดทศนิยมโดยนัย (`V`), เครื่องหมายบวก/ลบ (`S`), ขีดจำกัดของขนาดข้อมูล,
ไปจนถึง Numeric-edited Picture ที่ใช้จัดรูปแบบตัวเลขให้อ่านง่ายสำหรับมนุษย์ ปิดท้ายด้วยโปรแกรมรวบยอด
ที่ผสมผสาน PICTURE ทุกแบบเข้าด้วยกันในสถานการณ์จริง

---

## ขั้นตอนที่ 51: PICTURE Clause คืออะไร — ภาพรวมและความสัมพันธ์กับขนาดข้อมูล

### แนวคิด

`PICTURE` (หรือเขียนย่อว่า `PIC`) เป็น clause ที่ตามหลังชื่อตัวแปรใน DATA DIVISION เพื่อบอกคอมไพเลอร์ว่า
ตัวแปรนี้ **เก็บข้อมูลชนิดอะไร** และ **มีความยาวกี่หลัก/กี่ตัวอักษร** สัญลักษณ์ที่ใช้บ่อยที่สุดมี 3 ตัวคือ:

| สัญลักษณ์ | ความหมาย | ตัวอย่าง |
|---|---|---|
| `9` | ตัวเลข (Numeric) หนึ่งหลัก | `PIC 9(4)` = ตัวเลข 4 หลัก |
| `X` | ตัวอักษรทั่วไป (Alphanumeric) หนึ่งตัว — รับได้ทั้งตัวอักษร ตัวเลข สัญลักษณ์ | `PIC X(10)` = ข้อความ 10 ตัวอักษร |
| `A` | ตัวอักษรล้วน (Alphabetic) หนึ่งตัว — รับได้เฉพาะ A-Z และช่องว่าง | `PIC A(5)` = ตัวอักษรล้วน 5 ตัว |

สิ่งสำคัญที่สุดที่ต้องจำคือ **ขนาดของ PICTURE คือจำนวนไบต์ที่ตัวแปรนั้นจะใช้ในหน่วยความจำจริง**
(สำหรับ `USAGE DISPLAY` ซึ่งเป็นค่าเริ่มต้นที่เราใช้กันมาตลอด แต่ละสัญลักษณ์ = 1 ไบต์เสมอ) ซึ่งเชื่อมโยง
โดยตรงกับกฎเหล็กเรื่องคอลัมน์ 72 ไบต์ที่กล่าวถึงในเอกสารหลักสูตร — ยิ่งเข้าใจ PICTURE ดีเท่าไร ยิ่งคาดการณ์
ขนาดของโปรแกรมและพฤติกรรมการ MOVE ได้แม่นยำมากขึ้นเท่านั้น

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PICTURE-OVERVIEW.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRODUCT-CODE        PIC X(6)  VALUE "AB1234".
       01  WS-QUANTITY            PIC 9(4)  VALUE 250.
       01  WS-UNIT-PRICE          PIC 9(5)V99 VALUE 199.50.

       PROCEDURE DIVISION.
           DISPLAY "PICTURE clause describes size and type:".
           DISPLAY "Product code (PIC X(6)): " WS-PRODUCT-CODE.
           DISPLAY "Quantity     (PIC 9(4)): " WS-QUANTITY.
           DISPLAY "Unit price (PIC 9(5)V99): " WS-UNIT-PRICE.
           STOP RUN.
```

### อธิบายโค้ด

- `PIC X(6)` — `WS-PRODUCT-CODE` เก็บข้อความ 6 ตัวอักษรพอดี ใช้พื้นที่ 6 ไบต์
- `PIC 9(4)` — `WS-QUANTITY` เก็บตัวเลข 4 หลัก ใช้พื้นที่ 4 ไบต์ ไม่มีจุดทศนิยม
- `PIC 9(5)V99` — `WS-UNIT-PRICE` เก็บตัวเลข 7 หลัก (5 หลักจำนวนเต็ม + 2 หลักทศนิยม) ใช้พื้นที่ 7 ไบต์
  แม้จะดูเหมือนมีจุดทศนิยม แต่ `V` ไม่ได้กินพื้นที่หน่วยความจำเลย (รายละเอียดในขั้นตอนที่ 56)

### ผลลัพธ์ที่ได้จากการรันจริง

```
PICTURE clause describes size and type:
Product code (PIC X(6)): AB1234
Quantity     (PIC 9(4)): 0250
Unit price (PIC 9(5)V99): 00199.50
```

สังเกตว่า `WS-QUANTITY` แสดงผลเป็น `0250` (เติมศูนย์ให้ครบ 4 หลัก) และ `WS-UNIT-PRICE` แสดงเป็น
`00199.50` (เติมศูนย์ให้ครบ 5 หลักก่อนจุด และ GnuCOBOL แสดงจุดทศนิยมให้เพื่อความอ่านง่ายแม้จะไม่ได้
เก็บจุดจริงในหน่วยความจำ) — พฤติกรรมทั้งสองนี้จะอธิบายละเอียดในขั้นตอนถัดไป

### ข้อควรระวัง

- PICTURE ต้องเขียนตามหลังชื่อตัวแปรเสมอ และเป็น clause ที่ **บังคับต้องมี** สำหรับ elementary item
  ทุกตัว (ยกเว้น group item ที่ไม่มี PIC ของตัวเองตามที่เรียนใน Part 005)
- อย่าสับสนระหว่าง "จำนวนหลักที่ PICTURE ระบุ" กับ "ค่าที่ตัวแปรเก็บได้จริง" เช่น `PIC 9(4)` เก็บได้
  ตั้งแต่ 0000 ถึง 9999 เท่านั้น ค่าที่เกินกว่านี้จะถูกตัดทิ้งอย่างเงียบ ๆ (จะสาธิตในขั้นตอนที่ 52)

### แบบฝึกหัดที่ 51.1

**โจทย์**: จงบอกว่าตัวแปร `PIC X(8)` และ `PIC 9(8)` ใช้พื้นที่หน่วยความจำกี่ไบต์เท่ากันหรือไม่
และเก็บข้อมูลชนิดเดียวกันหรือไม่

**เฉลย**: ทั้งสองตัวใช้พื้นที่ **8 ไบต์เท่ากัน** เพราะตัวเลขในวงเล็บบอกจำนวนตัวอักษร/หลักเท่ากัน
แต่เก็บ**ข้อมูลคนละชนิด**: `PIC X(8)` เก็บข้อความทั่วไปได้ (ตัวอักษร ตัวเลข สัญลักษณ์ปนกันได้)
ส่วน `PIC 9(8)` เก็บได้เฉพาะตัวเลข 0-9 เท่านั้น และมีพฤติกรรมการ MOVE ที่ต่างกัน (ชิดขวา เติมศูนย์
vs ชิดซ้าย เติมช่องว่าง)

---

## ขั้นตอนที่ 52: PIC 9 — ชนิดตัวเลข (Numeric) และการนับหลัก

### แนวคิด

`PIC 9` ประกาศฟิลด์ที่เก็บได้เฉพาะตัวเลข 0-9 เท่านั้น (ไม่มีเครื่องหมายบวก/ลบ ไม่มีจุดทศนิยม เว้นแต่จะ
เพิ่ม `S` หรือ `V` เข้าไปตามที่จะเรียนในขั้นตอนถัดไป) จุดเด่นสำคัญของฟิลด์ตัวเลขคือ:

1. **เติมศูนย์ทางซ้ายเสมอ (Zero-fill, ชิดขวา)** เมื่อค่าที่ MOVE เข้ามาสั้นกว่าขนาดที่ประกาศไว้
2. **ตัดหลักซ้ายสุดทิ้งอย่างเงียบ ๆ** เมื่อค่าที่ MOVE เข้ามายาวเกินขนาดที่ประกาศไว้ (high-order truncation)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-NUMERIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SMALL-NUM           PIC 9(2)  VALUE 7.
       01  WS-BIG-NUM             PIC 9(6)  VALUE 42.
       01  WS-OVERFLOW-TARGET     PIC 9(3).

       PROCEDURE DIVISION.
           DISPLAY "PIC 9(2) holding 7    : [" WS-SMALL-NUM "]".
           DISPLAY "PIC 9(6) holding 42   : [" WS-BIG-NUM "]".
           DISPLAY "Notice how PIC 9 always zero-fills on the".
           DISPLAY "left to occupy every digit position.".

      *> Moving a value too large for the target silently
      *> truncates the HIGH-ORDER (leftmost) digits.
           MOVE 123456 TO WS-OVERFLOW-TARGET.
           DISPLAY "MOVE 123456 TO PIC 9(3) gives: "
                   WS-OVERFLOW-TARGET.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-SMALL-NUM PIC 9(2) VALUE 7` — ค่า 7 มีแค่ 1 หลัก แต่ฟิลด์ต้องการ 2 หลัก จึงเติมศูนย์ด้านหน้า
  กลายเป็น `07`
- `WS-BIG-NUM PIC 9(6) VALUE 42` — ค่า 42 มี 2 หลัก แต่ฟิลด์ต้องการ 6 หลัก จึงกลายเป็น `000042`
- `MOVE 123456 TO WS-OVERFLOW-TARGET` — `WS-OVERFLOW-TARGET` เป็น `PIC 9(3)` เก็บได้แค่ 3 หลัก
  แต่ค่าต้นทางมี 6 หลัก COBOL จะเก็บเฉพาะ **3 หลักขวาสุด** คือ `456` และตัดหลัก `123` ทิ้งไปเลย
  โดยไม่มี error หรือ warning ใด ๆ ทั้งสิ้น

### ผลลัพธ์ที่ได้จากการรันจริง

```
PIC 9(2) holding 7    : [07]
PIC 9(6) holding 42   : [000042]
Notice how PIC 9 always zero-fills on the
left to occupy every digit position.
MOVE 123456 TO PIC 9(3) gives: 456
```

### ข้อควรระวัง

- **การตัดหลักซ้ายสุดทิ้งแบบเงียบ ๆ (silent truncation) คือหนึ่งในบั๊กที่พบบ่อยที่สุดในโปรแกรม COBOL
  มือใหม่** ควรออกแบบขนาดฟิลด์ให้ใหญ่เพียงพอสำหรับค่าสูงสุดที่ธุรกิจจะเจอจริง ไม่ใช่แค่ค่าที่ทดสอบผ่าน
  ในขณะพัฒนา
- `PIC 9` แบบไม่มี `S` นำหน้า จะไม่สามารถเก็บค่าติดลบได้อย่างถูกต้อง (ทดสอบเพิ่มเติมในขั้นตอนที่ 57)

### แบบฝึกหัดที่ 52.1

**โจทย์**: หากประกาศ `01 WS-YEAR PIC 9(4).` แล้ว `MOVE 99999 TO WS-YEAR` ผลลัพธ์ที่ได้คืออะไร

**เฉลย**: `9999` เพราะ `WS-YEAR` เก็บได้ 4 หลัก ค่าต้นทาง `99999` มี 5 หลัก COBOL จะตัดหลักซ้ายสุด
(เลข 9 ตัวแรก) ทิ้งไป เหลือ 4 หลักขวาสุดคือ `9999`

---

## ขั้นตอนที่ 53: PIC X — ชนิดตัวอักษรทั่วไป (Alphanumeric)

### แนวคิด

`PIC X` ประกาศฟิลด์ที่เก็บได้ **ทุกชนิดของอักขระ** ทั้งตัวอักษร ตัวเลข ช่องว่าง และสัญลักษณ์พิเศษ
เป็นชนิดข้อมูลที่ "ยืดหยุ่นที่สุด" ใน COBOL และมักถูกใช้เป็นฟิลด์รับข้อมูลนำเข้าจากผู้ใช้หรือไฟล์ภายนอก
ก่อนที่จะตรวจสอบและแปลงเป็นชนิดอื่นภายหลัง (ดังที่เห็นตัวอย่างการใช้ `IS NUMERIC` ตรวจสอบฟิลด์
`PIC X` ใน Part 010 ขั้นตอนที่ 93)

พฤติกรรมการ MOVE ของ `PIC X` **ตรงข้ามกับ** `PIC 9` โดยสิ้นเชิง:

1. **เติมช่องว่างทางขวาเสมอ (Space-fill, ชิดซ้าย)** เมื่อค่าสั้นกว่าขนาดที่ประกาศไว้
2. **ตัดอักขระขวาสุดทิ้ง** เมื่อค่ายาวเกินขนาดที่ประกาศไว้ (low-order truncation)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-ALPHANUMERIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SHORT-TEXT          PIC X(5)  VALUE "HI".
       01  WS-LONG-TARGET         PIC X(10).
       01  WS-SHORT-TARGET        PIC X(3).

       PROCEDURE DIVISION.
           DISPLAY "PIC X(5) holding 'HI' : [" WS-SHORT-TEXT "]".
           DISPLAY "Notice PIC X pads with SPACES on the RIGHT,".
           DISPLAY "the opposite side from PIC 9 zero-fill.".

      *> Moving a shorter value into a bigger alphanumeric field
           MOVE "CAT" TO WS-LONG-TARGET.
           DISPLAY "MOVE 'CAT' TO PIC X(10)  : [" WS-LONG-TARGET "]".

      *> Moving a longer value into a smaller field truncates the
      *> RIGHTMOST characters (opposite of numeric truncation).
           MOVE "ELEPHANT" TO WS-SHORT-TARGET.
           DISPLAY "MOVE 'ELEPHANT' TO PIC X(3): ["
                   WS-SHORT-TARGET "]".
           STOP RUN.
```

### อธิบายโค้ด

- `WS-SHORT-TEXT PIC X(5) VALUE "HI"` — คำว่า "HI" มี 2 ตัวอักษร ฟิลด์มี 5 ช่อง จึงเติมช่องว่าง 3 ช่อง
  ทางขวา กลายเป็น `HI   ` (สังเกตว่าเติมทางขวา ตรงข้ามกับ `PIC 9` ที่เติมศูนย์ทางซ้าย)
- `MOVE "CAT" TO WS-LONG-TARGET` — "CAT" (3 ตัวอักษร) เข้าไปในฟิลด์ 10 ช่อง เติมช่องว่าง 7 ช่อง
  ทางขวา
- `MOVE "ELEPHANT" TO WS-SHORT-TARGET` — "ELEPHANT" มี 8 ตัวอักษร แต่ปลายทางมีแค่ 3 ช่อง COBOL
  จะเก็บ **3 ตัวอักษรซ้ายสุด** คือ "ELE" และตัดตัวอักษรที่เหลือทิ้งไปทั้งหมด

### ผลลัพธ์ที่ได้จากการรันจริง

```
PIC X(5) holding 'HI' : [HI   ]
Notice PIC X pads with SPACES on the RIGHT,
the opposite side from PIC 9 zero-fill.
MOVE 'CAT' TO PIC X(10)  : [CAT       ]
MOVE 'ELEPHANT' TO PIC X(3): [ELE]
```

### ข้อควรระวัง

- **จำกฎการตัด/เติมของ `PIC X` และ `PIC 9` ให้แม่นเพราะตรงข้ามกันโดยสิ้นเชิง**: ตัวเลขเติม/ตัดทาง
  ซ้าย (ชิดขวา) ส่วนข้อความเติม/ตัดทางขวา (ชิดซ้าย) — เป็นกฎพื้นฐานที่สุดข้อหนึ่งที่ต้องจำให้ขึ้นใจ
  ก่อนไปเรียน MOVE Rules แบบละเอียดใน Part 008
- ฟิลด์ `PIC X` ที่ถูกตัดอักขระขวาสุดทิ้งอาจทำให้ข้อมูลสำคัญหายไปโดยไม่รู้ตัว เช่น รหัสสินค้าหรือ
  เลขบัญชีที่ยาวเกินขนาดฟิลด์ที่ออกแบบไว้

### แบบฝึกหัดที่ 53.1

**โจทย์**: หากประกาศ `01 WS-CODE PIC X(4).` แล้ว `MOVE "AB" TO WS-CODE` ผลลัพธ์ที่แสดงด้วย
`DISPLAY "[" WS-CODE "]"` จะเป็นอย่างไร

**เฉลย**: `[AB  ]` — "AB" มี 2 ตัวอักษร ฟิลด์มี 4 ช่อง จึงเติมช่องว่าง 2 ช่องทางขวา

---

## ขั้นตอนที่ 54: PIC A — ชนิดตัวอักษรล้วน (Alphabetic)

### แนวคิด

`PIC A` เป็นชนิดข้อมูลที่ **เข้มงวดกว่า** `PIC X` โดยตั้งใจให้เก็บได้เฉพาะตัวอักษร A-Z (ทั้งตัวใหญ่ตัวเล็ก
ขึ้นกับการตั้งค่า) และช่องว่างเท่านั้น ไม่รองรับตัวเลขหรือสัญลักษณ์ ใช้บ่อยเมื่อฟิลด์นั้นควรมีแต่ตัวอักษร
เชิงความหมายจริง ๆ เช่น ชื่อคน หรือรหัสจังหวัดแบบตัวอักษรล้วน

การตรวจสอบว่าฟิลด์เป็นตัวอักษรล้วนจริงหรือไม่ ทำได้ผ่าน **Class Condition** `IS ALPHABETIC`
ที่เรียนไปแล้วใน Part 010 ขั้นตอนที่ 93

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-ALPHABETIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST-NAME          PIC A(10) VALUE "JOHN".
       01  WS-MIXED-FIELD         PIC X(10) VALUE "JOHN99".

       PROCEDURE DIVISION.
           DISPLAY "PIC A(10) holding 'JOHN': ["
                   WS-FIRST-NAME "]".
           IF WS-FIRST-NAME IS ALPHABETIC
               DISPLAY "WS-FIRST-NAME passes the ALPHABETIC test."
           END-IF.

           IF WS-MIXED-FIELD IS NOT ALPHABETIC
               DISPLAY "WS-MIXED-FIELD ('JOHN99') is NOT purely"
               DISPLAY "alphabetic because it contains digits."
           END-IF.
           STOP RUN.
```

### ผลลัพธ์ที่ได้จากการรันจริง

```
PIC A(10) holding 'JOHN': [JOHN      ]
WS-FIRST-NAME passes the ALPHABETIC test.
WS-MIXED-FIELD ('JOHN99') is NOT purely
alphabetic because it contains digits.
```

### พิสูจน์กับดัก: GnuCOBOL ไม่ตรวจสอบ PIC A ตอน MOVE โดยอัตโนมัติ

หลายคนเข้าใจผิดว่าถ้าประกาศฟิลด์เป็น `PIC A` แล้ว COBOL จะปฏิเสธการ MOVE ตัวเลขเข้าไปทันที
ลองพิสูจน์ด้วยโค้ดจริง:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-ALPHABETIC-BAD.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME-FIELD          PIC A(5).

       PROCEDURE DIVISION.
           MOVE "AB12C" TO WS-NAME-FIELD.
           DISPLAY "Result: " WS-NAME-FIELD.
           STOP RUN.
```

**ผลลัพธ์จริงจากการรันบนเครื่องนี้ (GnuCOBOL 4.0-early)**:

```
Result: AB12C
```

โปรแกรมนี้ **คอมไพล์ผ่านและรันได้ตามปกติ** แม้จะ MOVE ตัวเลข "12" เข้าไปในฟิลด์ที่ประกาศเป็น
`PIC A(5)` ก็ตาม — GnuCOBOL (และคอมไพเลอร์ COBOL ส่วนใหญ่) **ไม่ตรวจสอบความถูกต้องของข้อมูล
ตอน MOVE โดยอัตโนมัติ** `PIC A` เป็นเพียง "คำอธิบายเจตนา" ของผู้เขียนโปรแกรมเท่านั้น ไม่ใช่กลไก
บังคับความถูกต้องของข้อมูล (data validation) การตรวจสอบจริงต้องทำด้วย `IS ALPHABETIC` อย่างชัดเจน
เหมือนในตัวอย่างแรก

### ข้อควรระวัง

- **`PIC A` ไม่ได้ป้องกันข้อมูลผิดชนิดเข้ามาโดยอัตโนมัติ** ต้องตรวจสอบด้วย `IS ALPHABETIC` เสมอ
  หากต้องการความถูกต้องของข้อมูลจริง ๆ อย่าไว้ใจแค่การประกาศ PICTURE ชนิดนี้เพียงอย่างเดียว
- ในทางปฏิบัติของอุตสาหกรรมปัจจุบัน `PIC A` ถูกใช้งาน**น้อยกว่า** `PIC X` มาก เพราะโปรแกรมเมอร์ส่วนใหญ่
  เลือกใช้ `PIC X` แล้วตรวจสอบเงื่อนไขด้วย IF/Class Condition แทน เนื่องจากยืดหยุ่นกว่าและพฤติกรรม
  คาดเดาได้ง่ายกว่า

### แบบฝึกหัดที่ 54.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม `PIC-ALPHABETIC-BAD` ข้างต้นจึงคอมไพล์ผ่านและรันได้ปกติ
ทั้งที่เนื้อหาข้อมูลขัดกับ "เจตนา" ของ `PIC A`

**เฉลย**: เพราะ PICTURE clause ใน COBOL เป็นเพียงตัวกำหนด**ขนาดและประเภทการจัดเก็บ**ทางกายภาพ
(กี่ไบต์ ปฏิบัติแบบตัวเลขหรือข้อความ) ไม่ใช่กลไกตรวจสอบข้อมูล (validation) ที่ทำงานอัตโนมัติทุกครั้งที่
มีการ MOVE ความถูกต้องเชิงความหมาย (semantic correctness) ต้องอาศัยผู้เขียนโปรแกรมตรวจสอบเองด้วย
คำสั่งอย่าง `IF ... IS ALPHABETIC` อย่างชัดเจนในโค้ด

---

## ขั้นตอนที่ 55: Repetition Factor — PIC 9(5) แทนการเขียน 99999

### แนวคิด

การเขียน `PIC 9(5)` เป็นเพียง **shorthand (ทางลัด)** ของการเขียนสัญลักษณ์ซ้ำ 5 ครั้งคือ `PIC 99999`
ทั้งสองแบบมีความหมายเหมือนกันทุกประการ ตัวเลขในวงเล็บเรียกว่า **Repetition Factor** ใช้ได้กับทุก
สัญลักษณ์ PICTURE รวมถึง `X` และ `A` ด้วย และสามารถผสมหลายกลุ่มในฟิลด์เดียวกันได้

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-REPETITION-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-USING-PAREN         PIC 9(5)  VALUE 123.
       01  WS-USING-SPELLED-OUT   PIC 99999 VALUE 123.
       01  WS-MIXED-STYLE         PIC X(3)9(2) VALUE "AB712".

       PROCEDURE DIVISION.
           DISPLAY "PIC 9(5)   : [" WS-USING-PAREN "]".
           DISPLAY "PIC 99999  : [" WS-USING-SPELLED-OUT "]".
           DISPLAY "Both lines above declare the exact same".
           DISPLAY "5-digit numeric field -- (5) is shorthand".
           DISPLAY "for repeating the symbol 5 times.".
           DISPLAY "PIC X(3)9(2) mixed: [" WS-MIXED-STYLE "]".
           STOP RUN.
```

### อธิบายโค้ด

- `PIC 9(5)` และ `PIC 99999` ให้ผลลัพธ์เหมือนกันทุกประการ ทั้งขนาด (5 ไบต์) และพฤติกรรม
- `PIC X(3)9(2)` เป็นตัวอย่างพิเศษที่ผสม `X` กับ `9` ในฟิลด์เดียวกัน — เมื่อผสมสัญลักษณ์ต่างชนิดกัน
  ในฟิลด์เดียว COBOL จะถือว่า**ทั้งฟิลด์เป็น alphanumeric** โดยอัตโนมัติ (เพราะ `X` "ครอบงำ" ชนิดของ
  ฟิลด์) แม้จะมี `9` ปนอยู่ก็ตาม ฟิลด์นี้จึงมีพฤติกรรมแบบ `PIC X` ทั้งหมด (ไม่สามารถใช้ในการคำนวณได้)

### ผลลัพธ์ที่ได้จากการรันจริง

```
PIC 9(5)   : [00123]
PIC 99999  : [00123]
Both lines above declare the exact same
5-digit numeric field -- (5) is shorthand
for repeating the symbol 5 times.
PIC X(3)9(2) mixed: [AB712]
```

### เมื่อไหร่ควรใช้แบบไหน

ในทางปฏิบัติ **แทบไม่มีใครเขียน `PIC 99999999` ยาว ๆ** เพราะอ่านยากและเสี่ยงนับจำนวนหลักผิด
รูปแบบ `PIC 9(n)` ที่มี repetition factor จึงเป็นมาตรฐานที่ใช้กันแทบทั้งหมดในโค้ด COBOL สมัยใหม่
และเป็นรูปแบบที่หลักสูตรนี้ใช้ตลอดทั้งเล่ม

### ข้อควรระวัง

- Repetition factor ต้องเป็นจำนวนเต็มบวกเท่านั้น และเขียนอยู่ในวงเล็บติดกับสัญลักษณ์ที่ต้องการซ้ำ
  ห้ามมีช่องว่างคั่นระหว่างสัญลักษณ์กับวงเล็บ (เช่น `PIC 9 (5)` ที่มีช่องว่างอาจทำให้บางคอมไพเลอร์
  ตีความผิดพลาดได้)
- การผสมสัญลักษณ์ต่างชนิดในฟิลด์เดียว (เช่น `X` ปนกับ `9`) ทำให้ฟิลด์กลายเป็น alphanumeric
  ทั้งหมด **ไม่สามารถนำไปใช้ในคำสั่งคำนวณอย่าง `COMPUTE` หรือ `ADD` ได้โดยตรง**

### แบบฝึกหัดที่ 55.1

**โจทย์**: จงเขียน `PIC A(8)` ในรูปแบบสัญลักษณ์ซ้ำแบบเต็ม (ไม่ใช้วงเล็บ)

**เฉลย**: `PIC AAAAAAAA` (ตัวอักษร A ซ้ำ 8 ตัว)

---

## ขั้นตอนที่ 56: V — จุดทศนิยมโดยนัย (Implied Decimal Point)

### แนวคิด

`V` คือสัญลักษณ์พิเศษที่บอกตำแหน่ง **จุดทศนิยมที่ถูกสมมติขึ้น (assumed/implied)** ในฟิลด์ตัวเลข
จุดสำคัญที่สุดที่ต้องเข้าใจคือ **`V` ไม่ได้กินพื้นที่หน่วยความจำแม้แต่ไบต์เดียว** มันเป็นเพียง "เครื่องหมาย
บอกตำแหน่ง" ให้คอมไพเลอร์รู้ว่าจะจัดวางจุดทศนิยมที่ไหนตอนคำนวณเท่านั้น ข้อมูลจริงที่เก็บในหน่วยความจำ
คือตัวเลขล้วน ๆ ไม่มีจุดทศนิยมอยู่เลย

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-V-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE               PIC 9(4)V99 VALUE 99.5.
       01  WS-TAX-RATE            PIC V9(3)   VALUE 0.070.
       01  WS-TOTAL               PIC 9(6)V99.

       PROCEDURE DIVISION.
           DISPLAY "WS-PRICE (PIC 9(4)V99) raw display: ["
                   WS-PRICE "]".
           DISPLAY "Notice: V does NOT store an actual decimal".
           DISPLAY "point character -- it only tells COBOL where".
           DISPLAY "the decimal point is ASSUMED to be.".
           DISPLAY "WS-TAX-RATE (PIC V9(3)) raw display: ["
                   WS-TAX-RATE "]".

           COMPUTE WS-TOTAL = WS-PRICE * (1 + WS-TAX-RATE).
           DISPLAY "PRICE * (1 + TAX RATE) = " WS-TOTAL.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-PRICE PIC 9(4)V99 VALUE 99.5` — ฟิลด์นี้มีขนาดจริง **6 ไบต์** (4 หลักจำนวนเต็ม + 2 หลักทศนิยม
  `V` ไม่นับเป็นไบต์) ค่า 99.5 ถูกเก็บเป็นตัวเลขดิบ `009950` ในหน่วยความจำ
- `WS-TAX-RATE PIC V9(3) VALUE 0.070` — ฟิลด์นี้มีแต่ทศนิยมล้วน ๆ ไม่มีจำนวนเต็มเลย ขนาดจริง 3 ไบต์
  เก็บค่าดิบเป็น `070`
- เมื่อ `DISPLAY` ฟิลด์เหล่านี้โดยตรง GnuCOBOL รุ่นนี้จะ**แสดงจุดทศนิยมให้เพื่อความอ่านง่าย**
  (ตามที่เคยพิสูจน์ไว้แล้วใน Part 005 ขั้นตอนที่ 44) แต่นั่นเป็นแค่การจัดรูปแบบตอนแสดงผลเท่านั้น
  ไม่ใช่สิ่งที่เก็บจริงในหน่วยความจำ (ยืนยันได้ด้วย `FUNCTION LENGTH(WS-PRICE)` ซึ่งคืนค่า 6
  ไม่ใช่ 7 — ถ้ามีจุดทศนิยมเก็บจริงขนาดจะต้องเป็น 7 ไบต์)
- `COMPUTE WS-TOTAL = WS-PRICE * (1 + WS-TAX-RATE)` — แม้ข้อมูลจะเก็บเป็นเลขดิบไม่มีจุด แต่ COBOL
  รู้ตำแหน่งจุดทศนิยมจาก `V` ในแต่ละฟิลด์ และคำนวณผลลัพธ์ทศนิยมได้ถูกต้องเสมอ

### ผลลัพธ์ที่ได้จากการรันจริง

```
WS-PRICE (PIC 9(4)V99) raw display: [0099.50]
Notice: V does NOT store an actual decimal
point character -- it only tells COBOL where
the decimal point is ASSUMED to be.
WS-TAX-RATE (PIC V9(3)) raw display: [.070]
PRICE * (1 + TAX RATE) = 000106.46
```

99.5 × 1.070 = 106.465 ปัดเศษ/ตัดเศษตามกฎการ COMPUTE (จะเรียนละเอียดใน Part 009) ได้ 106.46

### ข้อควรระวัง

- **ห้ามเข้าใจผิดว่า `V` คือจุดทศนิยมจริงที่ MOVE หรือ DISPLAY ตรง ๆ ได้เหมือนภาษาอื่น** ถ้าคุณลอง
  `ACCEPT` ค่าที่มีจุด `.` เข้าฟิลด์ที่มี `V` พฤติกรรมอาจไม่เป็นไปตามที่คาดหวัง (ขึ้นกับคอมไพเลอร์และ
  การตั้งค่า) — ต้องระมัดระวังเป็นพิเศษเมื่อรับข้อมูลจากภายนอกเข้าฟิลด์ที่มี `V`
- `V` ปรากฏได้เพียง**ตำแหน่งเดียว**ในหนึ่ง PICTURE เท่านั้น (ไม่มีทศนิยมสองจุด) และสามารถอยู่ตำแหน่ง
  เริ่มต้น (`PIC V999`), ตรงกลาง (`PIC 999V99`), หรือ implicit ที่ท้ายสุด (`PIC 999` ซึ่งเท่ากับมี `V`
  อยู่ท้ายสุดโดยไม่ต้องเขียน) ก็ได้

### แบบฝึกหัดที่ 56.1

**โจทย์**: ฟิลด์ `PIC 9(3)V9(2)` มีขนาดกี่ไบต์ และหากเก็บค่า 7.5 จะมีเลขดิบในหน่วยความจำเป็นอะไร

**เฉลย**: ขนาด 5 ไบต์ (3 + 2 หลัก) เก็บค่า 7.5 เป็นเลขดิบ `00750` (007 สำหรับจำนวนเต็ม, 50 สำหรับ
ทศนิยม 2 ตำแหน่ง)

---

## ขั้นตอนที่ 57: S — เครื่องหมายบวก/ลบ (Sign) และ SIGN Clause

### แนวคิด

โดยค่าเริ่มต้น ฟิลด์ตัวเลข `PIC 9` **ไม่มีเครื่องหมาย (unsigned)** และไม่สามารถเก็บค่าติดลบได้อย่าง
ถูกต้อง หากต้องการให้ฟิลด์รองรับค่าติดลบ ต้องเติม `S` ไว้หน้าสุดของ PICTURE เสมอ เช่น `PIC S9(4)`

เช่นเดียวกับ `V`, **`S` โดยปกติไม่กินพื้นที่หน่วยความจำเพิ่ม** (เครื่องหมายจะถูกฝังรวมไปกับหลักสุดท้าย
ในรูปแบบที่เรียกว่า "zoned decimal" หรือ "overpunch") แต่หากต้องการให้เครื่องหมายเป็น**ตัวอักษรแยก
ต่างหาก**ที่มองเห็นได้ชัดเจน (`+` หรือ `-`) ต้องเพิ่ม clause `SIGN LEADING SEPARATE` หรือ
`SIGN TRAILING SEPARATE` ซึ่งจะทำให้ฟิลด์ใหญ่ขึ้น 1 ไบต์

### พิสูจน์กับดักก่อน: ฟิลด์ไม่มี S แล้วใส่ VALUE ติดลบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-SIGN-BAD-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-UNSIGNED            PIC 9(4)      VALUE -50.

       PROCEDURE DIVISION.
           DISPLAY "This should fail to compile.".
           STOP RUN.
```

**ผลลัพธ์การคอมไพล์จริง (error)**:

```
s57b.cob:6: error: data item not signed
```

คอมไพเลอร์ปฏิเสธทันที เพราะเราพยายามใส่ `VALUE -50` (ค่าติดลบ) ให้กับฟิลด์ `PIC 9(4)` ที่ไม่มี `S`
นำหน้า COBOL รู้ตั้งแต่ตอนคอมไพล์ว่าฟิลด์นี้ไม่รองรับเครื่องหมาย จึงฟ้อง error ทันทีแทนที่จะปล่อยให้
เกิดพฤติกรรมที่ไม่คาดคิดตอนรันจริง

### ตัวอย่างโค้ดที่ถูกต้อง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-SIGN-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SIGNED              PIC S9(4)     VALUE -50.
       01  WS-SIGNED-LEADING      PIC S9(4) SIGN LEADING SEPARATE
                                             VALUE -50.
       01  WS-SIGNED-TRAILING     PIC S9(4) SIGN TRAILING SEPARATE
                                             VALUE -50.

       PROCEDURE DIVISION.
           DISPLAY "PIC S9(4) with -50 (sign in nibble): ["
                   WS-SIGNED "]".
           DISPLAY "SIGN LEADING SEPARATE  -50: ["
                   WS-SIGNED-LEADING "]".
           DISPLAY "SIGN TRAILING SEPARATE -50: ["
                   WS-SIGNED-TRAILING "]".
           IF WS-SIGNED IS NEGATIVE
               DISPLAY "WS-SIGNED correctly tests as NEGATIVE."
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `PIC S9(4) VALUE -50` — เก็บค่า -50 ได้อย่างถูกต้อง โดยเครื่องหมายลบถูก "ฝัง" รวมกับหลักสุดท้าย
  (ขนาดฟิลด์ยังคงเป็น 4 ไบต์เท่าเดิม ไม่เพิ่มขึ้น) เมื่อ `DISPLAY` ออกมา GnuCOBOL แสดงเป็น `-0050`
  เพื่อความอ่านง่าย
- `SIGN LEADING SEPARATE` — บังคับให้เครื่องหมายเป็นอักขระแยกต่างหากอยู่**หน้าสุด** ทำให้ฟิลด์มีขนาด
  5 ไบต์ (4 หลัก + 1 ไบต์เครื่องหมาย) ผลลัพธ์คือ `-0050`
- `SIGN TRAILING SEPARATE` — เหมือนกันแต่เครื่องหมายอยู่**ท้ายสุด** แทน ผลลัพธ์คือ `0050-`
- `IF WS-SIGNED IS NEGATIVE` — Sign Condition ที่เรียนไปแล้วใน Part 010 ขั้นตอนที่ 99 ทำงานถูกต้อง
  กับฟิลด์ที่มี `S` เท่านั้น

### ผลลัพธ์ที่ได้จากการรันจริง

```
PIC S9(4) with -50 (sign in nibble): [-0050]
SIGN LEADING SEPARATE  -50: [-0050]
SIGN TRAILING SEPARATE -50: [0050-]
WS-SIGNED correctly tests as NEGATIVE.
```

### ข้อควรระวัง

- **ฟิลด์การเงินเกือบทั้งหมดในระบบธุรกิจจริงควรพิจารณาใส่ `S`** แม้ในบางกรณีจะดูเหมือนไม่จำเป็น
  (เช่น ยอดคงเหลือบัญชี ที่ในทางทฤษฎีติดลบได้ถ้าเบิกเกิน) การลืมใส่ `S` ตั้งแต่แรกแล้วต้องมาแก้ทีหลัง
  มักสร้างผลกระทบเป็นวงกว้างต่อโปรแกรมที่เขียนไปแล้ว
- `SIGN LEADING/TRAILING SEPARATE` เพิ่มขนาดฟิลด์ 1 ไบต์เสมอ ต้องคำนึงถึงเมื่อออกแบบ record
  ที่มีขนาดคงที่ (fixed-length) สำหรับเชื่อมต่อกับระบบภายนอกหรือไฟล์ legacy
- ค่าเริ่มต้นของเครื่องหมาย (ไม่ใส่ `SIGN` clause) เป็นแบบ "ฝังในหลักสุดท้าย" (overpunch) ซึ่งเมื่อเปิด
  ไฟล์ด้วย text editor ธรรมดาอาจเห็นเป็นตัวอักษรแปลก ๆ แทนตัวเลข — เป็นเรื่องปกติของฟิลด์ signed
  แบบดั้งเดิม ไม่ใช่ข้อมูลเสียหาย

### แบบฝึกหัดที่ 57.1

**โจทย์**: จงอธิบายว่าทำไม `MOVE -25 TO WS-COUNT` (โดยที่ `WS-COUNT PIC 9(3)`) จึงไม่ทำให้เกิด
compile error เหมือนตัวอย่าง `VALUE -50` ข้างต้น แต่จะได้ผลลัพธ์อย่างไรแทน

**เฉลย**: `VALUE` เป็นการกำหนดค่าคงที่ตอนคอมไพล์ ทำให้คอมไพเลอร์ตรวจสอบและฟ้อง error ได้ทันที
แต่ `MOVE` เป็นคำสั่งที่ทำงานตอนรันโปรแกรม (runtime) คอมไพเลอร์จึงยอมให้ผ่านโดยไม่ error
แต่พฤติกรรมจริงคือ COBOL จะ**เก็บเฉพาะค่าสัมบูรณ์ (absolute value) ของตัวเลข** เพราะฟิลด์ไม่มี `S`
รองรับเครื่องหมาย ผลลัพธ์ที่ได้ใน `WS-COUNT` จะเป็น `025` (ไม่มีเครื่องหมายลบเลย) ซึ่งอาจนำไปสู่บั๊ก
เชิงตรรกะที่ตรวจจับยากมากในภายหลัง

---

## ขั้นตอนที่ 58: ขีดจำกัดของขนาดข้อมูล — สูงสุดกี่หลัก?

### แนวคิด

มาตรฐาน COBOL ดั้งเดิม (ตั้งแต่ COBOL-85) กำหนดว่าฟิลด์ตัวเลขมีจำนวนหลักได้สูงสุด **18 หลัก**
ซึ่งเพียงพอสำหรับงานธุรกิจเกือบทั้งหมด (18 หลักรองรับตัวเลขได้ถึงเกือบ 1 ล้านล้านล้าน) อย่างไรก็ตาม
คอมไพเลอร์สมัยใหม่อย่าง GnuCOBOL ได้ขยายขีดจำกัดนี้ออกไปตามที่เราจะพิสูจน์ด้วยการทดสอบจริง

### ตัวอย่างโค้ด: PIC 9(18) มาตรฐาน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-SIZE-LIMIT-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MAX-DIGITS          PIC 9(18) VALUE 123456789012345678.
       01  WS-TEXT-FIELD          PIC X(4)  VALUE "TEST".

       PROCEDURE DIVISION.
           DISPLAY "PIC 9(18), the maximum standard numeric".
           DISPLAY "size, holds: " WS-MAX-DIGITS.
           DISPLAY "PIC X(4) occupies exactly 4 bytes: ["
                   WS-TEXT-FIELD "]".
           STOP RUN.
```

**ผลลัพธ์**:

```
PIC 9(18), the maximum standard numeric
size, holds: 123456789012345678
PIC X(4) occupies exactly 4 bytes: [TEST]
```

### ทดสอบขยายขีดจำกัดจริงบน GnuCOBOL

หลักสูตรนี้ยืนยันด้วยการคอมไพล์จริงว่า GnuCOBOL 4.0-early ที่ใช้ในเครื่องนี้รองรับได้ถึง **38 หลัก**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-38-TEST.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HUGE PIC 9(38) VALUE 1.
       PROCEDURE DIVISION.
           DISPLAY "OK 38 DIGITS: " WS-HUGE.
           STOP RUN.
```

**ผลลัพธ์**: `OK 38 DIGITS: 00000000000000000000000000000000000001`

แต่เมื่อลองขยับไปที่ 39 หลัก:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-39-TEST.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HUGE PIC 9(39) VALUE 1.
       PROCEDURE DIVISION.
           DISPLAY "OK 39 DIGITS: " WS-HUGE.
           STOP RUN.
```

**ผลลัพธ์การคอมไพล์จริง (error)**:

```
s58d.cob:5: error: numeric field cannot be larger than 38 digits
```

### สรุปตัวเลขขีดจำกัดที่ต้องจำ

| แหล่งอ้างอิง | ขีดจำกัดจำนวนหลัก |
|---|---|
| มาตรฐาน COBOL-85 ดั้งเดิม (portable ข้ามคอมไพเลอร์ที่สุด) | 18 หลัก |
| GnuCOBOL 4.0-early (ที่ทดสอบจริงในหลักสูตรนี้) | 38 หลัก |

### ข้อควรระวัง

- **หากต้องการเขียนโค้ดที่พกพาได้ (portable) ข้ามคอมไพเลอร์และสภาพแวดล้อม Mainframe จริง
  ควรยึดขีดจำกัด 18 หลักเป็นหลัก** อย่าพึ่งพาส่วนขยาย 38 หลักของ GnuCOBOL เว้นแต่มั่นใจว่าโปรแกรม
  จะรันบน GnuCOBOL เท่านั้นตลอดไป
- ในทางปฏิบัติ ฟิลด์ตัวเลขที่เกิน 18 หลักแทบไม่เคยจำเป็นในงานธุรกิจทั่วไป (แม้แต่ตัวเลขบัญชีธนาคาร
  ระดับโลกก็ไม่เกินนี้) หากพบว่าต้องใช้ฟิลด์ใหญ่ขนาดนั้นจริง ควรทบทวนการออกแบบข้อมูลก่อน

### แบบฝึกหัดที่ 58.1

**โจทย์**: จงอธิบายว่าทำไมนักพัฒนาที่ต้องการให้โค้ดพกพาได้ (portable) ควรหลีกเลี่ยงการพึ่งพาขีดจำกัด
38 หลักของ GnuCOBOL แม้จะทดสอบผ่านในเครื่องพัฒนาของตนแล้วก็ตาม

**เฉลยแนวทาง**: เพราะขีดจำกัด 38 หลักเป็น**ส่วนขยายเฉพาะของ GnuCOBOL** ไม่ใช่ส่วนหนึ่งของ
มาตรฐาน COBOL-85 ที่ระบบ Mainframe และคอมไพเลอร์อื่น (เช่น IBM Enterprise COBOL) ใช้อ้างอิง
หากนำโค้ดที่มีฟิลด์เกิน 18 หลักไปคอมไพล์บนสภาพแวดล้อมอื่น อาจพบ compile error ทันที ทำให้โค้ด
ไม่สามารถย้าย (migrate) ไปรันที่อื่นได้โดยไม่ต้องแก้ไข ซึ่งขัดกับหลักการเขียนโค้ดที่ดีในโลก Enterprise
ที่มักต้องใช้งานข้ามหลายแพลตฟอร์ม

---

## ขั้นตอนที่ 59: Numeric-Edited PICTURE — จัดรูปแบบตัวเลขให้อ่านง่าย

### แนวคิด

จนถึงตอนนี้ตัวเลขที่เรา DISPLAY ออกมาจะมีศูนย์นำหน้าเต็มไปหมด (เช่น `004599.50`) ซึ่งดูไม่เป็นมิตร
กับผู้อ่านรายงานทั่วไป **Numeric-edited PICTURE** คือชุดสัญลักษณ์พิเศษที่ใช้จัดรูปแบบตัวเลขให้อ่านง่าย
ขึ้นสำหรับการแสดงผล (แต่**ห้ามใช้ในการคำนวณ**) สัญลักษณ์ที่ใช้บ่อยที่สุดมีดังนี้:

| สัญลักษณ์ | ความหมาย |
|---|---|
| `Z` | Zero suppression — แทนที่ศูนย์นำหน้าที่ไม่มีความหมายด้วยช่องว่าง |
| `,` | ใส่เครื่องหมายจุลภาคคั่นหลักพัน |
| `.` | ใส่จุดทศนิยมจริงที่มองเห็นได้ (ต่างจาก `V` ที่มองไม่เห็น) |
| `$` | ใส่เครื่องหมายสกุลเงิน (ลอยตัวไปอยู่ติดกับตัวเลขตัวแรกที่ไม่ใช่ศูนย์) |
| `-` | แสดงเครื่องหมายลบ (ลอยตัว) เฉพาะเมื่อค่าติดลบ; ถ้าเป็นบวกจะเป็นช่องว่าง |

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PIC-EDITED-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW-AMOUNT          PIC 9(6)V99   VALUE 4599.5.
       01  WS-ZERO-SUPPRESSED     PIC ZZZ,ZZ9.99.
       01  WS-WITH-DOLLAR         PIC $$$,$$9.99.
       01  WS-NEGATIVE-AMOUNT     PIC S9(5)V99  VALUE -320.75.
       01  WS-WITH-SIGN           PIC ---,---9.99.

       PROCEDURE DIVISION.
           MOVE WS-RAW-AMOUNT TO WS-ZERO-SUPPRESSED.
           DISPLAY "Raw            : " WS-RAW-AMOUNT.
           DISPLAY "Zero-suppressed: " WS-ZERO-SUPPRESSED.

           MOVE WS-RAW-AMOUNT TO WS-WITH-DOLLAR.
           DISPLAY "With $ sign    : " WS-WITH-DOLLAR.

           MOVE WS-NEGATIVE-AMOUNT TO WS-WITH-SIGN.
           DISPLAY "Negative shown : " WS-WITH-SIGN.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-ZERO-SUPPRESSED PIC ZZZ,ZZ9.99` — เมื่อ MOVE ค่า `4599.50` เข้ามา ศูนย์นำหน้าที่ไม่มี
  ความหมายจะถูกแทนที่ด้วยช่องว่าง เหลือเฉพาะตัวเลขที่มีความหมายจริง พร้อมจุลภาคคั่นหลักพันและจุด
  ทศนิยมที่มองเห็นได้จริง
- `WS-WITH-DOLLAR PIC $$$,$$9.99` — เครื่องหมาย `$` จะ "ลอยตัว" ไปติดกับตัวเลขตัวแรกที่ไม่ใช่ศูนย์
  เสมอ (เรียกว่า floating insertion) ทำให้ได้รูปแบบเงินที่คุ้นตา
- `WS-WITH-SIGN PIC ---,---9.99` — เครื่องหมาย `-` แบบลอยตัวจะแสดงก็ต่อเมื่อค่าเป็นลบเท่านั้น
  เมื่อ MOVE ค่า -320.75 เข้ามา จะเห็นเครื่องหมายลบลอยไปอยู่ติดกับตัวเลขตัวแรก

### ผลลัพธ์ที่ได้จากการรันจริง

```
Raw            : 004599.50
Zero-suppressed:   4,599.50
With $ sign    :  $4,599.50
Negative shown :     -320.75
```

### ข้อควรระวัง

- **ฟิลด์ Numeric-edited ใช้ได้เฉพาะ "ปลายทางของการแสดงผล" เท่านั้น ห้ามนำไปใช้ในการคำนวณต่อ**
  (เช่น `COMPUTE X = WS-WITH-DOLLAR + 1` จะ error หรือให้ผลลัพธ์ผิดพลาด) รูปแบบการทำงานที่ถูกต้อง
  คือคำนวณในฟิลด์ตัวเลขปกติก่อน แล้วค่อย `MOVE` ผลลัพธ์เข้าฟิลด์ edited เพื่อแสดงผลเป็นขั้นตอนสุดท้าย
  เท่านั้น (ดังตัวอย่างข้างต้นที่คำนวณใน `WS-RAW-AMOUNT` ก่อน)
- เมื่อ MOVE เข้าฟิลด์ edited แล้ว **ไม่สามารถ MOVE ย้อนกลับออกมาเป็นตัวเลขปกติได้ตรง ๆ** เพราะ
  ฟิลด์ edited กลายเป็น alphanumeric ที่มีเครื่องหมายพิเศษปะปนอยู่แล้ว

### แบบฝึกหัดที่ 59.1

**โจทย์**: จงเขียน PICTURE แบบ numeric-edited สำหรับแสดงยอดเงิน 8 หลักจำนวนเต็ม + 2 หลักทศนิยม
โดยไม่แสดงศูนย์นำหน้าที่ไม่มีความหมาย และมีจุลภาคคั่นหลักพัน

**เฉลย**: `PIC Z,ZZZ,ZZ9.99` (หรือปรับจำนวน Z ตามความยาวที่ต้องการ นับรวมให้ครบ 8 หลักจำนวนเต็ม)

---

## ขั้นตอนที่ 60: โปรแกรมรวบยอด — ระบบข้อมูลพนักงานที่ผสม PICTURE หลายชนิด

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่รวม PICTURE ทุกชนิดที่เรียนมาเข้าด้วยกันในสถานการณ์จริง: บันทึกข้อมูล
พนักงานหนึ่งคน ที่มีทั้งรหัสตัวเลข ชื่อข้อความ รหัสแผนกตัวอักษรล้วน เงินเดือนมีทศนิยม เปอร์เซ็นต์โบนัส
และยอดค้างจ่ายที่อาจติดลบได้ พร้อมจัดรูปแบบการแสดงผลด้วย numeric-edited picture

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EMPLOYEE-PICTURE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  EMPLOYEE-RECORD.
           05  EMP-ID             PIC 9(5)      VALUE 30045.
           05  EMP-NAME           PIC X(20)     VALUE "SIRIPORN JAIDEE".
           05  EMP-DEPT-CODE      PIC A(3)      VALUE "ACC".
           05  EMP-MONTHLY-SALARY PIC 9(7)V99   VALUE 45000.00.
           05  EMP-BONUS-PERCENT  PIC V999      VALUE 0.150.
           05  EMP-BALANCE-DUE    PIC S9(6)V99  VALUE -1250.50.
       01  WS-DISPLAY-SALARY      PIC $$$,$$$,$$9.99.
       01  WS-DISPLAY-BALANCE     PIC ---,---,--9.99.

       PROCEDURE DIVISION.
           DISPLAY "===== EMPLOYEE RECORD (PICTURE DEMO) =====".
           DISPLAY "ID           : " EMP-ID.
           DISPLAY "NAME         : " EMP-NAME.
           DISPLAY "DEPT CODE    : " EMP-DEPT-CODE.

           MOVE EMP-MONTHLY-SALARY TO WS-DISPLAY-SALARY.
           DISPLAY "SALARY       : " WS-DISPLAY-SALARY.

           DISPLAY "BONUS RATE   : " EMP-BONUS-PERCENT.

           MOVE EMP-BALANCE-DUE TO WS-DISPLAY-BALANCE.
           DISPLAY "BALANCE DUE  : " WS-DISPLAY-BALANCE.

           IF EMP-DEPT-CODE IS ALPHABETIC
               DISPLAY "DEPT CODE IS VALID (ALPHABETIC ONLY)."
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `EMPLOYEE-RECORD` เป็น group item ที่รวมฟิลด์ทุกชนิดที่เรียนมาในบทนี้ไว้ด้วยกัน: `PIC 9`
  (EMP-ID), `PIC X` (EMP-NAME), `PIC A` (EMP-DEPT-CODE), `PIC 9...V99` (EMP-MONTHLY-SALARY),
  `PIC V999` (EMP-BONUS-PERCENT), และ `PIC S9...V99` (EMP-BALANCE-DUE ที่ติดลบได้)
- `WS-DISPLAY-SALARY` และ `WS-DISPLAY-BALANCE` เป็นฟิลด์ numeric-edited แยกต่างหาก ใช้เฉพาะ
  สำหรับแสดงผลให้อ่านง่าย โดยคำนวณ/เก็บค่าจริงไว้ในฟิลด์ปกติก่อนเสมอ ตามหลักปฏิบัติที่ถูกต้องจาก
  ขั้นตอนที่ 59
- `IF EMP-DEPT-CODE IS ALPHABETIC` — สาธิตการตรวจสอบความถูกต้องของฟิลด์ `PIC A` อย่างชัดเจน
  ตามที่เรียนในขั้นตอนที่ 54 แทนที่จะไว้ใจ PICTURE เพียงอย่างเดียว

### ผลลัพธ์ที่ได้จากการรันจริง

```
===== EMPLOYEE RECORD (PICTURE DEMO) =====
ID           : 30045
NAME         : SIRIPORN JAIDEE     
DEPT CODE    : ACC
SALARY       :     $45,000.00
BONUS RATE   : .150
BALANCE DUE  :      -1,250.50
DEPT CODE IS VALID (ALPHABETIC ONLY).
```

### ข้อควรระวัง

- เมื่อออกแบบ record ที่มีหลายฟิลด์ผสมกันแบบนี้ ควรวางแผนขนาดของแต่ละฟิลด์ให้เพียงพอตั้งแต่แรก
  เพราะการแก้ไขขนาดฟิลด์ภายหลัง (โดยเฉพาะเมื่อมีการอ้างอิงจากหลายจุดในโปรแกรมขนาดใหญ่ หรือ
  เชื่อมกับไฟล์ record ความยาวคงที่) อาจส่งผลกระทบเป็นวงกว้าง
- สังเกตว่า `EMP-NAME` แสดงผลพร้อมช่องว่างต่อท้าย (เพราะเป็น `PIC X(20)` แต่ชื่อจริงสั้นกว่า) — ถ้า
  นำค่านี้ไปต่อกับข้อความอื่นโดยตรงโดยไม่ตัดช่องว่างท้ายออกก่อน อาจได้ผลลัพธ์ที่ดูแปลกตา (ช่องว่าง
  แทรกกลางประโยค) ซึ่งเป็นประเด็นที่จะพูดถึงเครื่องมือแก้ไขอย่าง `STRING`/`FUNCTION TRIM` ใน Part
  ถัดไปของหลักสูตร

### แบบฝึกหัดที่ 60.1

**โจทย์**: จงเพิ่มฟิลด์ใหม่ `EMP-YEARS-OF-SERVICE PIC 9(2)` เข้าไปใน `EMPLOYEE-RECORD` พร้อม
กำหนด `VALUE 5` และเขียนคำสั่ง `DISPLAY` เพื่อแสดงผล

**เฉลย**:
```cobol
       01  EMPLOYEE-RECORD.
           05  EMP-ID             PIC 9(5)      VALUE 30045.
           05  EMP-NAME           PIC X(20)     VALUE "SIRIPORN JAIDEE".
           05  EMP-DEPT-CODE      PIC A(3)      VALUE "ACC".
           05  EMP-MONTHLY-SALARY PIC 9(7)V99   VALUE 45000.00.
           05  EMP-BONUS-PERCENT  PIC V999      VALUE 0.150.
           05  EMP-BALANCE-DUE    PIC S9(6)V99  VALUE -1250.50.
           05  EMP-YEARS-OF-SERVICE PIC 9(2)    VALUE 5.
```
พร้อมเพิ่มบรรทัด `DISPLAY "YEARS OF SERVICE: " EMP-YEARS-OF-SERVICE.` ใน PROCEDURE DIVISION
ซึ่งจะแสดงผลเป็น `YEARS OF SERVICE: 05`

---

## สรุปท้ายบท

ใน Part นี้ เราได้เจาะลึก PICTURE Clause ซึ่งเป็นหัวใจของการประกาศข้อมูลใน COBOL อย่างครบถ้วน:

- PICTURE clause กำหนดทั้งชนิดข้อมูล ขนาด (เป็นไบต์) และรูปแบบการจัดเก็บในคำประกาศเดียว
- `PIC 9` (ตัวเลข) เติม/ตัดศูนย์ทางซ้าย (ชิดขวา) ส่วน `PIC X` (ข้อความ) เติม/ตัดช่องว่างทางขวา
  (ชิดซ้าย) — กฎที่ตรงข้ามกันโดยสิ้นเชิงที่ต้องจำให้ขึ้นใจ
- `PIC A` (ตัวอักษรล้วน) เป็นเพียง "เจตนา" ไม่ใช่กลไกตรวจสอบข้อมูลอัตโนมัติ ต้องใช้ `IS ALPHABETIC`
  ตรวจสอบเองเสมอ
- Repetition Factor `(n)` เป็นทางลัดแทนการเขียนสัญลักษณ์ซ้ำหลายตัว
- `V` คือจุดทศนิยมโดยนัยที่ไม่กินพื้นที่หน่วยความจำจริง ต่างจากจุดทศนิยมที่มองเห็นได้ใน numeric-edited
  picture
- `S` ทำให้ฟิลด์ตัวเลขรองรับเครื่องหมายลบได้อย่างถูกต้อง และ `SIGN LEADING/TRAILING SEPARATE`
  ทำให้เครื่องหมายเป็นอักขระแยกที่มองเห็นชัดเจน (แลกกับขนาดฟิลด์ที่เพิ่มขึ้น 1 ไบต์)
- ขีดจำกัดจำนวนหลัก: มาตรฐานดั้งเดิม 18 หลัก แต่ GnuCOBOL ขยายได้ถึง 38 หลัก (ทดสอบยืนยันจริง)
- Numeric-edited PICTURE (`Z`, `,`, `.`, `$`, `-`) ใช้จัดรูปแบบตัวเลขให้อ่านง่ายสำหรับการแสดงผล
  เท่านั้น ห้ามใช้ในการคำนวณ
- โปรแกรมรวบยอดที่ผสาน PICTURE ทุกชนิดเข้าด้วยกันในบันทึกข้อมูลพนักงานหนึ่งระเบียน

ตอนนี้คุณสามารถออกแบบ PICTURE clause ได้อย่างแม่นยำสำหรับทุกสถานการณ์ที่พบเจอ ใน Part ถัดไป
เราจะเรียนรู้วิธีการสื่อสารกับผู้ใช้งานโปรแกรมโดยตรงผ่านคำสั่ง **`DISPLAY`** และ **`ACCEPT`**
ซึ่งเป็นคำสั่ง Input/Output พื้นฐานที่สุดของ COBOL ก่อนที่เราจะไปเรียนรู้การย้ายข้อมูลอย่างละเอียด
ด้วย `MOVE` ใน Part 008

**[← กลับไป Part 005](part-005-working-storage-section.md)** | **[ไปยัง Part 007: DISPLAY และ ACCEPT →](part-007-display-accept.md)**
