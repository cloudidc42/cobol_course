# Part 009: เลขคณิตพื้นฐาน: ADD, SUBTRACT, MULTIPLY, DIVIDE, COMPUTE (ขั้นตอนที่ 81–90)

## คำนำของ Part นี้

ใน Part 008 เราเรียนรู้วิธีย้ายข้อมูลด้วย `MOVE` และเข้าใจกฎการจัดตำแหน่งตัวเลขและทศนิยมอย่างละเอียด
แล้ว ความรู้นั้นคือรากฐานสำคัญที่เราจะนำมาใช้ต่อใน Part นี้ เพราะตอนนี้ถึงเวลาที่จะทำสิ่งที่ COBOL
ถูกออกแบบมาให้ทำได้ดีที่สุดในโลก: **การคำนวณตัวเลขทางธุรกิจอย่างแม่นยำ**

COBOL มีคำสั่งเลขคณิตเฉพาะทางถึง 4 คำสั่ง (`ADD`, `SUBTRACT`, `MULTIPLY`, `DIVIDE`) ที่อ่านออกเสียง
เหมือนภาษาอังกฤษปกติ บวกกับคำสั่งอเนกประสงค์ `COMPUTE` ที่ให้เราเขียนสมการคณิตศาสตร์แบบทั่วไปได้
Part นี้จะพาคุณเรียนรู้ทุกรูปแบบของคำสั่งเหล่านี้ พร้อมทั้ง `ROUNDED` phrase สำหรับการปัดเศษ
`ON SIZE ERROR` สำหรับดักจับข้อผิดพลาด และฟังก์ชันในตัว (Intrinsic Functions) ที่ช่วยงานคำนวณให้
ง่ายขึ้น ปิดท้ายด้วยโปรแกรมเครื่องคิดเลขขนาดย่อมที่รวมทุกอย่างเข้าด้วยกัน

---

## ขั้นตอนที่ 81: คำสั่ง ADD — การบวกเลขทั้ง 3 รูปแบบ

### แนวคิด

คำสั่ง `ADD` มีอยู่ 3 รูปแบบหลักที่ให้ผลลัพธ์ต่างกัน:

1. **`ADD A TO B`** — นำค่า A บวกเข้าไปใน B แล้วเก็บผลรวมกลับไว้ที่ B (B ถูกเขียนทับด้วยผลลัพธ์ใหม่)
2. **`ADD A B GIVING C`** — นำ A บวกกับ B แล้วเก็บผลลัพธ์ไว้ที่ C โดย **A และ B ไม่ถูกแก้ไข**
3. **`ADD A B C TO D`** — นำหลายฟิลด์มาบวกรวมกันแล้วบวกเข้า D ทั้งหมดในคำสั่งเดียว

การเลือกใช้รูปแบบไหนขึ้นอยู่กับว่าต้องการให้ค่าตั้งต้นถูกแก้ไขหรือไม่ ซึ่งเป็นการตัดสินใจสำคัญเวลา
ออกแบบโปรแกรมจริง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ADD-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SALES-MON        PIC 9(6)  VALUE 15000.
       01  WS-SALES-TUE        PIC 9(6)  VALUE 18000.
       01  WS-SALES-WED        PIC 9(6)  VALUE 12000.
       01  WS-WEEKLY-TOTAL     PIC 9(6)  VALUE ZERO.
       01  WS-RUNNING-TOTAL    PIC 9(6)  VALUE 0.

       PROCEDURE DIVISION.
      *> Form 1: ADD ... TO ... (accumulates into existing field)
           MOVE WS-SALES-MON TO WS-RUNNING-TOTAL
           ADD WS-SALES-TUE TO WS-RUNNING-TOTAL
           DISPLAY "AFTER ADD TUE : " WS-RUNNING-TOTAL

      *> Form 2: ADD ... GIVING ... (does not touch the addends)
           ADD WS-SALES-MON WS-SALES-TUE WS-SALES-WED
               GIVING WS-WEEKLY-TOTAL
           DISPLAY "WEEKLY TOTAL  : " WS-WEEKLY-TOTAL

      *> Form 3: ADD several fields TO one target at once
           ADD WS-SALES-MON WS-SALES-TUE WS-SALES-WED
               TO WS-RUNNING-TOTAL
           DISPLAY "RUNNING TOTAL : " WS-RUNNING-TOTAL
           STOP RUN.
```

### อธิบายโค้ด

- **Form 1**: `ADD WS-SALES-TUE TO WS-RUNNING-TOTAL` — เริ่มจาก `WS-RUNNING-TOTAL` มีค่า 15000
  (มาจากยอดวันจันทร์) แล้วบวกยอดวันอังคาร (18000) เข้าไป ผลลัพธ์ใหม่ (33000) ถูกเขียนทับกลับไปที่
  `WS-RUNNING-TOTAL` ทันที
- **Form 2**: `ADD WS-SALES-MON WS-SALES-TUE WS-SALES-WED GIVING WS-WEEKLY-TOTAL` — บวกยอดขาย
  ทั้งสามวันเข้าด้วยกัน (15000+18000+12000 = 45000) แล้วเก็บผลลัพธ์ไว้ที่ `WS-WEEKLY-TOTAL`
  โดยที่ `WS-SALES-MON`, `WS-SALES-TUE`, `WS-SALES-WED` **ยังคงมีค่าเดิมไม่เปลี่ยนแปลง**
- **Form 3**: `ADD WS-SALES-MON WS-SALES-TUE WS-SALES-WED TO WS-RUNNING-TOTAL` — บวกยอดขายทั้งสาม
  วันเข้าไปที่ `WS-RUNNING-TOTAL` ที่มีค่าอยู่แล้ว (33000 จาก Form 1) รวมเป็น 33000+45000 = 78000

### ผลลัพธ์ที่ได้จากการรันจริง

```
AFTER ADD TUE : 033000
WEEKLY TOTAL  : 045000
RUNNING TOTAL : 078000
```

### ข้อควรระวัง

- ความแตกต่างระหว่าง `TO` และ `GIVING` เป็นจุดที่ผู้เริ่มต้นสับสนบ่อยที่สุด: `TO` **แก้ไขฟิลด์
  ปลายทางที่มีอยู่แล้ว** ส่วน `GIVING` **สร้างผลลัพธ์ใหม่โดยไม่แตะต้องฟิลด์ตั้งต้น**
- หากฟิลด์ผลลัพธ์มีขนาดเล็กเกินไป ผลรวมอาจเกินขอบเขตและถูกตัดหลักสูงทิ้งแบบเงียบ ๆ เหมือนที่เรียน
  ใน Part 008 (เว้นแต่จะใช้ `ON SIZE ERROR` ซึ่งจะสอนใน ขั้นตอนที่ 87)

### แบบฝึกหัดที่ 81.1

**โจทย์**: หากต้องการบวก `WS-A`, `WS-B` เข้าด้วยกันแล้วเก็บผลลัพธ์ไว้ในตัวแปรใหม่ `WS-SUM`
โดยไม่ต้องการให้ `WS-A` และ `WS-B` เปลี่ยนแปลงค่า ควรใช้รูปแบบคำสั่งใด?

**เฉลย**: `ADD WS-A WS-B GIVING WS-SUM` — ใช้ฟอร์ม GIVING เพราะต้องการเก็บผลลัพธ์แยกไปยังตัวแปรใหม่
โดยไม่แก้ไขค่าตั้งต้นของ WS-A และ WS-B

---

## ขั้นตอนที่ 82: คำสั่ง SUBTRACT — การลบเลขทั้ง 3 รูปแบบ

### แนวคิด

`SUBTRACT` มีโครงสร้างคล้าย `ADD` แต่ทิศทางของคำบุพบทต่างออกไปเล็กน้อย ต้องจำให้แม่นเพราะสลับ
ทิศทางกับที่คนส่วนใหญ่คาดเดา:

1. **`SUBTRACT A FROM B`** — นำค่า A ไปลบออกจาก B แล้วเก็บผลลัพธ์กลับไว้ที่ B (คือ B = B - A)
2. **`SUBTRACT A FROM B GIVING C`** — นำ A ลบออกจาก B แล้วเก็บผลลัพธ์ไว้ที่ C โดย B ไม่ถูกแก้ไข
3. **`SUBTRACT A1 A2 FROM B`** — ลบหลายค่าออกจาก B พร้อมกันในคำสั่งเดียว

จุดสำคัญคือคำว่า `FROM` แปลว่า "ลบออกจาก" ดังนั้น `SUBTRACT A FROM B` จึงหมายถึง B ลบด้วย A
ไม่ใช่ A ลบด้วย B — ตรงข้ามกับสิ่งที่นักเรียนบางคนเข้าใจผิดตอนแรก

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUBTRACT-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STOCK-QTY        PIC 9(6)  VALUE 500.
       01  WS-SOLD-QTY         PIC 9(6)  VALUE 120.
       01  WS-DAMAGED-QTY      PIC 9(6)  VALUE 15.
       01  WS-REMAINING-QTY    PIC 9(6).

       PROCEDURE DIVISION.
      *> Form 1: SUBTRACT ... FROM ... (changes the FROM field)
           SUBTRACT WS-SOLD-QTY FROM WS-STOCK-QTY
           DISPLAY "STOCK AFTER SALE      : " WS-STOCK-QTY

      *> Form 2: SUBTRACT ... FROM ... GIVING ... (keeps FROM field)
           SUBTRACT WS-DAMAGED-QTY FROM WS-STOCK-QTY
               GIVING WS-REMAINING-QTY
           DISPLAY "STOCK UNCHANGED        : " WS-STOCK-QTY
           DISPLAY "REMAINING AFTER DAMAGE : " WS-REMAINING-QTY

      *> Form 3: subtract more than one field
           SUBTRACT WS-SOLD-QTY WS-DAMAGED-QTY FROM WS-STOCK-QTY
           DISPLAY "STOCK AFTER BOTH       : " WS-STOCK-QTY
           STOP RUN.
```

### อธิบายโค้ด

- **Form 1**: `SUBTRACT WS-SOLD-QTY FROM WS-STOCK-QTY` คือ 500 - 120 = 380 ผลลัพธ์เขียนทับกลับไป
  ที่ `WS-STOCK-QTY` ทันที
- **Form 2**: `SUBTRACT WS-DAMAGED-QTY FROM WS-STOCK-QTY GIVING WS-REMAINING-QTY` คือ 380 - 15 = 365
  แต่ผลลัพธ์เก็บไว้ที่ `WS-REMAINING-QTY` แยกต่างหาก ดังนั้น `WS-STOCK-QTY` **ยังคงเป็น 380 เหมือนเดิม**
- **Form 3**: `SUBTRACT WS-SOLD-QTY WS-DAMAGED-QTY FROM WS-STOCK-QTY` คือลบทั้งสองค่าออกจาก
  `WS-STOCK-QTY` ที่ยังเป็น 380 อยู่: 380 - 120 - 15 = 245

### ผลลัพธ์ที่ได้จากการรันจริง

```
STOCK AFTER SALE      : 000380
STOCK UNCHANGED        : 000380
REMAINING AFTER DAMAGE : 000365
STOCK AFTER BOTH       : 000245
```

### ข้อควรระวัง

- ถ้าผลลัพธ์ของการลบติดลบ แต่ฟิลด์ปลายทางไม่มี `S` (unsigned) ค่าที่ได้จะกลายเป็นค่าสัมบูรณ์
  (เหมือนที่เรียนใน Part 008 ขั้นตอนที่ 79) เช่น ถ้ายอดขายมากกว่าสต๊อกที่มี ผลลัพธ์ที่ควรติดลบจะ
  กลายเป็นค่าบวกผิด ๆ แทน — ให้ใช้ `PIC S9...` เมื่อผลลัพธ์อาจติดลบได้ในทางธุรกิจจริง
- `FROM` คือ "ลบออกจาก" เสมอ — จำสูตร **"SUBTRACT [สิ่งที่จะลบ] FROM [ตัวที่ถูกลบ]"**

### แบบฝึกหัดที่ 82.1

**โจทย์**: หากต้องการคำนวณ `WS-BALANCE = WS-BALANCE - WS-WITHDRAWAL` ควรเขียนคำสั่ง SUBTRACT
อย่างไร?

**เฉลย**: `SUBTRACT WS-WITHDRAWAL FROM WS-BALANCE` — ลบยอดถอนเงิน (WS-WITHDRAWAL) ออกจากยอดคงเหลือ
(WS-BALANCE) แล้วเก็บผลลัพธ์กลับไว้ที่ WS-BALANCE เดิม

---

## ขั้นตอนที่ 83: คำสั่ง MULTIPLY — การคูณเลข

### แนวคิด

`MULTIPLY` มี 2 รูปแบบหลัก:

1. **`MULTIPLY A BY B`** — นำ A คูณกับ B แล้วเก็บผลลัพธ์กลับไว้ที่ B (คือ B = A × B)
2. **`MULTIPLY A BY B GIVING C`** — นำ A คูณกับ B แล้วเก็บผลลัพธ์ไว้ที่ C โดย A และ B ไม่ถูกแก้ไข

สังเกตว่า `BY` ทำหน้าที่คล้ายกับ `TO` ใน `ADD` คือฟิลด์ที่ตามหลัง `BY` (ในฟอร์มไม่มี GIVING) คือ
ฟิลด์ที่จะถูกเขียนทับด้วยผลลัพธ์ ซึ่งมักใช้ในการคำนวณราคารวม, ภาษี, หรือดอกเบี้ยในธุรกิจจริง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MULTIPLY-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-UNIT-PRICE       PIC 9(5)V99  VALUE 250.00.
       01  WS-QUANTITY         PIC 9(5)     VALUE 12.
       01  WS-TOTAL-PRICE      PIC 9(7)V99.
       01  WS-TAX-RATE         PIC 9V99     VALUE 0.07.
       01  WS-TAX-AMOUNT       PIC 9(7)V99.

       PROCEDURE DIVISION.
      *> Form 1: MULTIPLY ... BY ... GIVING ... (keeps both operands)
           MULTIPLY WS-UNIT-PRICE BY WS-QUANTITY
               GIVING WS-TOTAL-PRICE
           DISPLAY "UNIT PRICE  : " WS-UNIT-PRICE
           DISPLAY "QUANTITY    : " WS-QUANTITY
           DISPLAY "TOTAL PRICE : " WS-TOTAL-PRICE

      *> Form 2: MULTIPLY ... BY ... (overwrites the BY field)
           MULTIPLY WS-TAX-RATE BY WS-TOTAL-PRICE
               GIVING WS-TAX-AMOUNT
           DISPLAY "TAX AMOUNT  : " WS-TAX-AMOUNT

           MULTIPLY 2 BY WS-QUANTITY
           DISPLAY "QUANTITY X2 : " WS-QUANTITY
           STOP RUN.
```

### อธิบายโค้ด

- `MULTIPLY WS-UNIT-PRICE BY WS-QUANTITY GIVING WS-TOTAL-PRICE` — คำนวณ 250.00 × 12 = 3000.00
  เก็บไว้ที่ `WS-TOTAL-PRICE` โดยทั้ง `WS-UNIT-PRICE` และ `WS-QUANTITY` ไม่เปลี่ยนแปลง
- `MULTIPLY WS-TAX-RATE BY WS-TOTAL-PRICE GIVING WS-TAX-AMOUNT` — คำนวณภาษี 7% จากยอดรวม:
  0.07 × 3000.00 = 210.00
- `MULTIPLY 2 BY WS-QUANTITY` — ตัวอย่างการใช้ค่าคงที่ (literal) เป็นตัวคูณ และไม่มี GIVING จึง
  เขียนทับค่าเดิมของ `WS-QUANTITY` ทันที (12 × 2 = 24)

### ผลลัพธ์ที่ได้จากการรันจริง

```
UNIT PRICE  : 00250.00
QUANTITY    : 00012
TOTAL PRICE : 0003000.00
TAX AMOUNT  : 0000210.00
QUANTITY X2 : 00024
```

### ข้อควรระวัง

- ฟิลด์ผลลัพธ์ของการคูณต้องมีขนาดใหญ่พอ เพราะการคูณมีแนวโน้มทำให้ตัวเลขมีจำนวนหลักเพิ่มขึ้นอย่าง
  รวดเร็ว (เช่น 5 หลัก × 5 หลัก อาจได้ผลลัพธ์ถึง 10 หลัก) หากฟิลด์เล็กเกินไปจะถูกตัดหลักสูงทิ้ง
- การไม่ใช้ GIVING ในการคูณ (`MULTIPLY 2 BY WS-QUANTITY`) จะเขียนทับค่าตั้งต้นทันที ควรระวังไม่ให้
  ค่าตั้งต้นที่จำเป็นต้องใช้ต่อถูกเขียนทับโดยไม่ตั้งใจ

### แบบฝึกหัดที่ 83.1

**โจทย์**: จงเขียนคำสั่งคำนวณค่าคอมมิชชั่น 5% จากยอดขาย `WS-SALES-AMOUNT` แล้วเก็บผลลัพธ์ไว้ที่
`WS-COMMISSION` โดยไม่แก้ไขค่า `WS-SALES-AMOUNT`

**เฉลย**: `MULTIPLY WS-SALES-AMOUNT BY 0.05 GIVING WS-COMMISSION` — ใช้ GIVING เพื่อไม่ให้
`WS-SALES-AMOUNT` ถูกแก้ไข

---

## ขั้นตอนที่ 84: คำสั่ง DIVIDE — การหารเลขและเศษที่เหลือ (Remainder)

### แนวคิด

`DIVIDE` เป็นคำสั่งเลขคณิตที่มีรูปแบบมากที่สุด เนื่องจากการหารมีความซับซ้อนกว่าคำสั่งอื่น
(ต้องจัดการทั้งผลหารและเศษ):

1. **`DIVIDE A INTO B`** — นำ B หารด้วย A แล้วเก็บผลหารกลับไว้ที่ B (สังเกตทิศทาง: INTO สลับกับที่
   คนทั่วไปคาดคิด — คือ "เอา A ไปหาร (เข้าไปใน) B" หมายถึง B ÷ A)
2. **`DIVIDE A BY B GIVING C`** — นำ A หารด้วย B แล้วเก็บผลหารไว้ที่ C (ทิศทางตรงไปตรงมาตามที่อ่าน)
3. **`DIVIDE A BY B GIVING C REMAINDER D`** — เหมือนข้อ 2 แต่เก็บเศษที่เหลือจากการหารไว้ที่ D ด้วย

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DIVIDE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TOTAL-STUDENTS   PIC 9(4)    VALUE 37.
       01  WS-BUS-CAPACITY     PIC 9(4)    VALUE 8.
       01  WS-BUSES-NEEDED     PIC 9(4).
       01  WS-SEATS-LEFT       PIC 9(4).

       01  WS-TOTAL-AMOUNT     PIC 9(7)V99 VALUE 1000.00.
       01  WS-PEOPLE-COUNT     PIC 9(3)    VALUE 3.
       01  WS-SHARE-EACH       PIC 9(7)V99.

       PROCEDURE DIVISION.
      *> Form 1: DIVIDE ... INTO ... (result replaces the INTO field)
           DIVIDE WS-BUS-CAPACITY INTO WS-TOTAL-STUDENTS
           DISPLAY "STUDENTS FIELD BECOMES QUOTIENT: "
               WS-TOTAL-STUDENTS

      *> Form 2: DIVIDE ... BY ... GIVING ... REMAINDER ...
           MOVE 37 TO WS-TOTAL-STUDENTS
           DIVIDE WS-TOTAL-STUDENTS BY WS-BUS-CAPACITY
               GIVING WS-BUSES-NEEDED
               REMAINDER WS-SEATS-LEFT
           DISPLAY "FULL BUSES NEEDED : " WS-BUSES-NEEDED
           DISPLAY "STUDENTS LEFT OVER: " WS-SEATS-LEFT

      *> Form 3: dividing money evenly
           DIVIDE WS-TOTAL-AMOUNT BY WS-PEOPLE-COUNT
               GIVING WS-SHARE-EACH
           DISPLAY "SHARE EACH PERSON : " WS-SHARE-EACH
           STOP RUN.
```

### อธิบายโค้ด

- `DIVIDE WS-BUS-CAPACITY INTO WS-TOTAL-STUDENTS` — คือ 37 ÷ 8 = 4 (ปัดเศษทิ้งแบบ integer
  division) ผลลัพธ์เขียนทับกลับไปที่ `WS-TOTAL-STUDENTS` (ฟิลด์ตัวเลขจำนวนเต็ม ไม่มีทศนิยม
  จึงตัดเศษทศนิยมทิ้งอัตโนมัติ)
- `DIVIDE WS-TOTAL-STUDENTS BY WS-BUS-CAPACITY GIVING WS-BUSES-NEEDED REMAINDER WS-SEATS-LEFT`
  — คำนวณว่าต้องใช้รถบัสกี่คัน (37 ÷ 8 = 4 คันเต็ม) และเหลือนักเรียนกี่คนที่นั่งไม่พอดี (37 - 4×8 = 5)
  `REMAINDER` ช่วยให้เราไม่ต้องคำนวณเศษเองด้วยมือ
- `DIVIDE WS-TOTAL-AMOUNT BY WS-PEOPLE-COUNT GIVING WS-SHARE-EACH` — หารเงิน 1000.00 บาท
  ให้ 3 คนเท่า ๆ กัน ได้ 333.33 (ตัดทศนิยมส่วนเกินทิ้งเพราะฟิลด์มีแค่ 2 ตำแหน่งทศนิยม)

### ผลลัพธ์ที่ได้จากการรันจริง

```
STUDENTS FIELD BECOMES QUOTIENT: 0004
FULL BUSES NEEDED : 0004
STUDENTS LEFT OVER: 0005
SHARE EACH PERSON : 0000333.33
```

### ข้อควรระวัง

- **`DIVIDE ... INTO ...` มีทิศทางที่สับสนบ่อยที่สุดในบรรดาคำสั่งเลขคณิตทั้งหมด** ให้จำว่า
  `DIVIDE A INTO B` หมายถึง "เอา A ไปหารเข้า B" คือ B ÷ A ไม่ใช่ A ÷ B
- **การหารด้วยศูนย์ (division by zero)** ทำให้เกิด runtime error ร้ายแรง ต้องตรวจสอบตัวหารก่อนเสมอ
  หรือใช้ `ON SIZE ERROR` ดักจับ (สอนใน ขั้นตอนที่ 87)
- 1000.00 ÷ 3 = 333.333... ถูกตัดทศนิยมเหลือ 333.33 (เงินหายไป 0.01 บาทต่อคน คูณ 3 คน = 0.01 บาท
  หายไปจากระบบ) — ปัญหานี้เรียกว่า "penny rounding problem" ที่ระบบบัญชีจริงต้องจัดการอย่างระมัดระวัง

### แบบฝึกหัดที่ 84.1

**โจทย์**: จงเขียนคำสั่ง DIVIDE เพื่อหาว่า 100 หารด้วย 7 ได้ผลหารเท่าไรและเหลือเศษเท่าไร
โดยเก็บผลหารไว้ที่ `WS-Q` และเศษไว้ที่ `WS-R`

**เฉลย**: `DIVIDE 100 BY 7 GIVING WS-Q REMAINDER WS-R` — จะได้ผลหาร (WS-Q) เท่ากับ 14
และเศษ (WS-R) เท่ากับ 2 (เพราะ 14×7=98, เหลือเศษ 2)

---

## ขั้นตอนที่ 85: คำสั่ง COMPUTE — เขียนสมการคณิตศาสตร์แบบทั่วไป

### แนวคิด

`COMPUTE` เป็นคำสั่งที่ทรงพลังที่สุดในกลุ่มเลขคณิต เพราะให้เราเขียน **สมการคณิตศาสตร์แบบเต็มรูปแบบ**
โดยใช้เครื่องหมายที่คุ้นเคย (`+`, `-`, `*`, `/`, `**` สำหรับยกกำลัง) แทนที่จะต้องแยกเป็นคำสั่ง ADD,
SUBTRACT, MULTIPLY ทีละคำสั่ง

ลำดับการคำนวณ (operator precedence) ของ COMPUTE เป็นไปตามหลักคณิตศาสตร์มาตรฐาน: วงเล็บก่อน
ตามด้วยยกกำลัง คูณ/หาร แล้วจึงบวก/ลบ (เหมือนที่เรียนในวิชาคณิตศาสตร์)

รูปแบบพื้นฐาน:

```
COMPUTE <result> = <expression>
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. COMPUTE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A                PIC 9(4)    VALUE 10.
       01  WS-B                PIC 9(4)    VALUE 4.
       01  WS-C                PIC 9(4)    VALUE 2.
       01  WS-RESULT-1         PIC 9(6).
       01  WS-RESULT-2         PIC 9(6).
       01  WS-RESULT-3         PIC S9(6).

       01  WS-BASE-SALARY      PIC 9(7)V99 VALUE 25000.00.
       01  WS-BONUS-PERCENT    PIC 9V99    VALUE 0.10.
       01  WS-FINAL-SALARY     PIC 9(7)V99.

       PROCEDURE DIVISION.
      *> COMPUTE lets us write a normal math expression directly
           COMPUTE WS-RESULT-1 = WS-A + WS-B * WS-C
           DISPLAY "A + B * C           = " WS-RESULT-1

           COMPUTE WS-RESULT-2 = (WS-A + WS-B) * WS-C
           DISPLAY "(A + B) * C         = " WS-RESULT-2

           COMPUTE WS-RESULT-3 = WS-A - WS-B - WS-C
           DISPLAY "A - B - C           = " WS-RESULT-3

           COMPUTE WS-FINAL-SALARY =
               WS-BASE-SALARY + (WS-BASE-SALARY * WS-BONUS-PERCENT)
           DISPLAY "SALARY WITH BONUS   = " WS-FINAL-SALARY
           STOP RUN.
```

### อธิบายโค้ด

- `COMPUTE WS-RESULT-1 = WS-A + WS-B * WS-C` — ตามลำดับการคำนวณมาตรฐาน คูณก่อนบวก:
  4 × 2 = 8, จากนั้น 10 + 8 = 18
- `COMPUTE WS-RESULT-2 = (WS-A + WS-B) * WS-C` — วงเล็บบังคับให้บวกก่อน: (10+4) × 2 = 28
  ผลลัพธ์ต่างจากสมการแรกโดยสิ้นเชิงแม้จะใช้ตัวเลขชุดเดียวกัน แสดงให้เห็นความสำคัญของวงเล็บ
- `COMPUTE WS-FINAL-SALARY = WS-BASE-SALARY + (WS-BASE-SALARY * WS-BONUS-PERCENT)` — คำนวณ
  เงินเดือนพร้อมโบนัส 10% ในสมการเดียว: 25000.00 + (25000.00 × 0.10) = 25000.00 + 2500.00 = 27500.00
  แสดงให้เห็นพลังของ `COMPUTE` ที่รวมการคำนวณหลายขั้นตอนไว้ในบรรทัดเดียว แทนที่จะต้องเขียน MULTIPLY
  แล้วตามด้วย ADD แยกกัน

### ผลลัพธ์ที่ได้จากการรันจริง

```
A + B * C           = 000018
(A + B) * C         = 000028
A - B - C           = +000004
SALARY WITH BONUS   = 0027500.00
```

### ข้อควรระวัง

- **ลืมใส่วงเล็บ** เป็นสาเหตุอันดับหนึ่งของผลลัพธ์การคำนวณที่ผิดพลาดเมื่อใช้ `COMPUTE` เพราะลำดับ
  การคำนวณอาจไม่ตรงกับที่ผู้เขียนโค้ดตั้งใจ ให้ใส่วงเล็บชัดเจนเสมอเมื่อสมการซับซ้อน แม้บางครั้งจะ
  ไม่จำเป็นตามหลักลำดับการคำนวณ เพื่อความชัดเจนในการอ่านโค้ด
- ฟิลด์ผลลัพธ์ของ `COMPUTE` ต้องเป็นฟิลด์ตัวเลข ไม่สามารถ COMPUTE ผลลัพธ์ไปยังฟิลด์ตัวอักษรได้
- หากผลลัพธ์อาจติดลบ (เช่น `WS-RESULT-3` ในตัวอย่าง) ต้องใช้ `PIC S9...` มิเช่นนั้นเครื่องหมายลบ
  จะหายไปเหมือนที่เรียนใน Part 008

### แบบฝึกหัดที่ 85.1

**โจทย์**: จงเขียนคำสั่ง `COMPUTE` เพื่อคำนวณพื้นที่วงกลม (Area = 3.14159 × r × r) โดยกำหนดให้ r
เก็บอยู่ในตัวแปร `WS-RADIUS` และเก็บผลลัพธ์ไว้ที่ `WS-AREA`

**เฉลย**: `COMPUTE WS-AREA = 3.14159 * WS-RADIUS * WS-RADIUS`

---

## ขั้นตอนที่ 86: ROUNDED Phrase — การปัดเศษอย่างถูกต้อง

### แนวคิด

ตามที่เรียนมาตลอด Part นี้ คำสั่งเลขคณิตของ COBOL จะ **ตัดทศนิยมส่วนเกินทิ้ง (truncate) เป็นค่า
เริ่มต้นเสมอ ไม่ปัดเศษ** หากต้องการให้ผลลัพธ์ถูกปัดเศษแบบคณิตศาสตร์ (ปัดขึ้นถ้าตัวเลขถัดไป ≥ 5
ปัดลงถ้า < 5) ต้องเพิ่ม phrase `ROUNDED` ต่อท้ายฟิลด์ผลลัพธ์อย่างชัดเจน

`ROUNDED` ใช้ได้กับคำสั่งเลขคณิตทุกตัว (`ADD`, `SUBTRACT`, `MULTIPLY`, `DIVIDE`, `COMPUTE`)
โดยวางไว้ทันทีหลังชื่อฟิลด์ผลลัพธ์

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ROUNDED-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE            PIC 9(5)V99  VALUE 100.00.
       01  WS-QUANTITY         PIC 9(3)     VALUE 6.
       01  WS-SHARE-TRUNC      PIC 9(5)V99.
       01  WS-SHARE-ROUNDED    PIC 9(5)V99.

       PROCEDURE DIVISION.
           DIVIDE WS-PRICE BY WS-QUANTITY GIVING WS-SHARE-TRUNC
           DISPLAY "WITHOUT ROUNDED (truncated) : " WS-SHARE-TRUNC

           DIVIDE WS-PRICE BY WS-QUANTITY
               GIVING WS-SHARE-ROUNDED ROUNDED
           DISPLAY "WITH ROUNDED               : " WS-SHARE-ROUNDED

           COMPUTE WS-SHARE-TRUNC = WS-PRICE / 6
           DISPLAY "COMPUTE (truncated)        : " WS-SHARE-TRUNC

           COMPUTE WS-SHARE-ROUNDED ROUNDED = WS-PRICE / 6
           DISPLAY "COMPUTE ROUNDED            : " WS-SHARE-ROUNDED
           STOP RUN.
```

### อธิบายโค้ด

- 100.00 ÷ 6 = 16.666666... โดยไม่ใช้ `ROUNDED` ผลลัพธ์ถูกตัดทศนิยมเหลือ 2 ตำแหน่งแบบตัดทิ้งตรง ๆ
  ได้ `16.66`
- เมื่อเพิ่ม `ROUNDED` หลังชื่อฟิลด์ผลลัพธ์ (`GIVING WS-SHARE-ROUNDED ROUNDED`) COBOL จะพิจารณา
  หลักทศนิยมถัดไป (หลักที่ 3 คือ 6) ซึ่ง ≥ 5 จึงปัดขึ้นเป็น `16.67`
- สังเกตตำแหน่งของ `ROUNDED` ใน `COMPUTE`: เขียนต่อจากชื่อฟิลด์ผลลัพธ์ก่อนเครื่องหมาย `=` เช่น
  `COMPUTE WS-SHARE-ROUNDED ROUNDED = WS-PRICE / 6`

### ผลลัพธ์ที่ได้จากการรันจริง

```
WITHOUT ROUNDED (truncated) : 00016.66
WITH ROUNDED               : 00016.67
COMPUTE (truncated)        : 00016.66
COMPUTE ROUNDED            : 00016.67
```

### ข้อควรระวัง

- **`ROUNDED` เป็น opt-in เสมอ** — ถ้าลืมใส่ ผลลัพธ์จะถูกตัดทิ้งแบบเงียบ ๆ โดยไม่มีการแจ้งเตือนใด ๆ
  ในระบบการเงินที่ต้องการความแม่นยำ (เช่น ดอกเบี้ย, ภาษี, ค่าคอมมิชชั่น) ควรพิจารณาใช้ `ROUNDED`
  เกือบทุกครั้งที่มีการหารหรือคูณที่อาจเกิดทศนิยมยาว
- แม้ COBOL มาตรฐานจะมี `ROUNDED MODE` แบบต่าง ๆ (เช่น NEAREST-AWAY-FROM-ZERO, NEAREST-EVEN)
  ในระดับที่สูงขึ้น แต่พฤติกรรมเริ่มต้นของ `ROUNDED` ใน GnuCOBOL คือปัดแบบ "round half up"
  (ปัดขึ้นเมื่อหลักถัดไป ≥ 5) ซึ่งตรงกับที่ใช้ในชีวิตประจำวันทั่วไป

### แบบฝึกหัดที่ 86.1

**โจทย์**: หาก `WS-VALUE PIC 9(3)V99` และเราคำนวณ `COMPUTE WS-VALUE ROUNDED = 10 / 3` ผลลัพธ์
ที่ได้คือเท่าไร?

**เฉลย**: `003.33` — เพราะ 10 ÷ 3 = 3.3333... หลักทศนิยมตำแหน่งที่ 3 คือ 3 ซึ่งน้อยกว่า 5
จึงปัดลง ผลลัพธ์คือ 3.33 (ในกรณีนี้ค่าที่ได้จาก ROUNDED และไม่ ROUNDED จะเหมือนกัน
เพราะหลักถัดไปน้อยกว่า 5 อยู่แล้ว)

---

## ขั้นตอนที่ 87: ON SIZE ERROR — การดักจับข้อผิดพลาดจากการคำนวณ

### แนวคิด

เมื่อผลลัพธ์ของการคำนวณ **ใหญ่เกินกว่าที่ฟิลด์ปลายทางจะเก็บได้** หรือเกิด **การหารด้วยศูนย์**
COBOL จะไม่หยุดโปรแกรมโดยอัตโนมัติเสมอไป (พฤติกรรมขึ้นกับคอมไพเลอร์) แต่เราสามารถใช้ phrase
**`ON SIZE ERROR`** เพื่อดักจับสถานการณ์เหล่านี้และสั่งให้โปรแกรมทำงานอย่างปลอดภัยแทนที่จะปล่อยให้
ข้อมูลผิดพลาดแบบเงียบ ๆ หรือโปรแกรม crash

รูปแบบ:

```
<arithmetic-statement>
    ON SIZE ERROR
        <imperative-statement>
    [NOT ON SIZE ERROR
        <imperative-statement>]
END-<verb>
```

`ON SIZE ERROR` ใช้ได้กับคำสั่งเลขคณิตทุกตัว รวมถึง `COMPUTE` และครอบคลุมทั้งกรณีค่าล้นฟิลด์
(overflow) และการหารด้วยศูนย์

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SIZE-ERROR-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SMALL-TOTAL      PIC 9(3)  VALUE 0.
       01  WS-BIG-NUMBER       PIC 9(3)  VALUE 950.
       01  WS-ADD-AMOUNT       PIC 9(3)  VALUE 200.
       01  WS-DIVISOR          PIC 9(3)  VALUE 0.
       01  WS-SAFE-RESULT      PIC 9(5).

       PROCEDURE DIVISION.
           MOVE WS-BIG-NUMBER TO WS-SMALL-TOTAL
           ADD WS-ADD-AMOUNT TO WS-SMALL-TOTAL
               ON SIZE ERROR
                   DISPLAY "ERROR: RESULT TOO BIG FOR THE FIELD!"
               NOT ON SIZE ERROR
                   DISPLAY "ADD OK: " WS-SMALL-TOTAL
           END-ADD

           DISPLAY "FIELD VALUE AFTER OVERFLOW ATTEMPT: "
               WS-SMALL-TOTAL

           DIVIDE WS-BIG-NUMBER BY WS-DIVISOR
               GIVING WS-SAFE-RESULT
               ON SIZE ERROR
                   DISPLAY "ERROR: DIVIDE BY ZERO DETECTED!"
           END-DIVIDE

           DISPLAY "PROGRAM CONTINUES SAFELY AFTER THE ERROR."
           STOP RUN.
```

### อธิบายโค้ด

- `WS-SMALL-TOTAL` เป็น `PIC 9(3)` เก็บได้สูงสุด 999 เริ่มต้นมีค่า 950 แล้วบวก 200 เข้าไปจะได้ 1150
  ซึ่งเกินขอบเขตของฟิลด์ 3 หลัก — `ON SIZE ERROR` จึงถูกกระตุ้นและแสดงข้อความแจ้งเตือนแทนที่จะปล่อย
  ให้ค่าที่ผิดถูกเก็บแบบเงียบ ๆ
- สังเกตว่าเมื่อเกิด `SIZE ERROR` **ฟิลด์ปลายทางจะไม่ถูกแก้ไขเลย** (ยังคงค่าเดิมคือ 950)
  เพราะ COBOL จะไม่ทำการ assign ค่าที่ผิดพลาดเข้าไป
- `NOT ON SIZE ERROR` เป็น phrase เสริมที่ทำงานเมื่อการคำนวณสำเร็จปกติ (ตรงข้ามกับ ON SIZE ERROR)
- `DIVIDE WS-BIG-NUMBER BY WS-DIVISOR` — `WS-DIVISOR` มีค่า 0 การหารด้วยศูนย์จะถูกจัดเป็น SIZE
  ERROR เช่นกันตามมาตรฐาน COBOL ทำให้โปรแกรมไม่ crash แต่แสดงข้อความแจ้งเตือนแทน

### ผลลัพธ์ที่ได้จากการรันจริง

```
ERROR: RESULT TOO BIG FOR THE FIELD!
FIELD VALUE AFTER OVERFLOW ATTEMPT: 950
ERROR: DIVIDE BY ZERO DETECTED!
PROGRAM CONTINUES SAFELY AFTER THE ERROR.
```

### ข้อควรระวัง

- **หากไม่ใช้ `ON SIZE ERROR` การหารด้วยศูนย์อาจทำให้โปรแกรม crash (runtime abend)** ทันที
  ในระบบที่ประมวลผลข้อมูลจำนวนมาก (batch processing) นี่คือสาเหตุที่ทำให้งานทั้ง batch ล้มเหลว
  หากไม่มีการป้องกันไว้ล่วงหน้า — ควรใช้ `ON SIZE ERROR` ทุกครั้งที่ตัวหารมาจากข้อมูลภายนอก
  ที่ไม่สามารถควบคุมได้ 100%
- ต้องปิดท้ายด้วย scope terminator ที่ตรงกับคำสั่ง (`END-ADD`, `END-DIVIDE`, `END-COMPUTE` ฯลฯ)
  เมื่อใช้ `ON SIZE ERROR` ร่วมกับคำสั่งเลขคณิตแบบหลายบรรทัด เพื่อไม่ให้ scope ของเงื่อนไขคลุมเครือ

### แบบฝึกหัดที่ 87.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบ `IF WS-DIVISOR NOT = ZERO` ก่อนหารด้วยตนเอง กับการใช้
`ON SIZE ERROR` มีข้อดีข้อเสียต่างกันอย่างไร

**เฉลยแนวทาง**: การตรวจสอบด้วย `IF` ก่อนหารช่วยป้องกันปัญหาได้เชิงรุก (proactive) และอ่านโค้ด
ได้ตรงไปตรงมา แต่ต้องเขียนเงื่อนไขแยกทุกจุดที่มีการหาร ส่วน `ON SIZE ERROR` เป็นกลไกในตัวคำสั่ง
เดียวกัน (in-line) ทำให้จัดการ error ได้ทันทีในจุดที่เกิดปัญหาโดยไม่ต้องเขียน IF แยก และยังครอบคลุม
กรณี overflow อื่น ๆ ที่ IF ตรวจสอบล่วงหน้าไม่ได้ (เช่นผลคูณที่ใหญ่เกินฟิลด์) ในทางปฏิบัติมักใช้
ทั้งสองแนวทางร่วมกันเพื่อความปลอดภัยสูงสุด

---

## ขั้นตอนที่ 88: ความแม่นยำของทศนิยมและผลลัพธ์กลาง (Intermediate Results)

### แนวคิด

เมื่อสมการมีการคำนวณหลายขั้นตอนต่อเนื่องกัน (เช่น หารสองครั้งติดกัน) COBOL จะเก็บ **ผลลัพธ์กลาง
(intermediate result)** ไว้ในหน่วยความจำชั่วคราวระหว่างการคำนวณ ก่อนที่จะนำผลลัพธ์สุดท้ายไปเก็บ
ในฟิลด์ปลายทางตามที่เราประกาศไว้

ประเด็นสำคัญคือ **ขนาด PICTURE ของฟิลด์ผลลัพธ์สุดท้ายเป็นตัวกำหนดว่าความแม่นยำจะเหลือเท่าไร**
หากฟิลด์ปลายทางมีทศนิยมน้อย ความแม่นยำที่สะสมมาจากการคำนวณหลายขั้นตอนจะถูกตัดทิ้งไปในตอนสุดท้าย
ซึ่งอาจทำให้ผลลัพธ์คลาดเคลื่อนสะสม (rounding/truncation error) มากกว่าที่คาดไว้โดยเฉพาะเมื่อมี
การคำนวณต่อเนื่องหลายขั้น

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DECIMAL-PRECISION-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A                PIC 9(3)V99  VALUE 10.00.
       01  WS-B                PIC 9(3)V99  VALUE 3.00.
       01  WS-C                PIC 9(3)V99  VALUE 3.00.
       01  WS-RESULT-2DEC      PIC 9(5)V99.
       01  WS-RESULT-6DEC      PIC 9(5)V9(6).

       PROCEDURE DIVISION.
      *> Result field only has 2 decimal places: extra
      *> precision from intermediate division is lost.
           COMPUTE WS-RESULT-2DEC = WS-A / WS-B / WS-C
           DISPLAY "A/B/C INTO 2 DECIMALS : " WS-RESULT-2DEC

      *> Give the result field more decimal places to keep
      *> more precision from the intermediate calculation.
           COMPUTE WS-RESULT-6DEC = WS-A / WS-B / WS-C
           DISPLAY "A/B/C INTO 6 DECIMALS : " WS-RESULT-6DEC

           COMPUTE WS-RESULT-2DEC = WS-A * WS-B
           DISPLAY "A * B (exact)         : " WS-RESULT-2DEC
           STOP RUN.
```

### อธิบายโค้ด

- `WS-A / WS-B / WS-C` คือ 10.00 ÷ 3.00 ÷ 3.00 = 1.11111... (ทศนิยมไม่รู้จบ)
- เมื่อเก็บผลลัพธ์ไปยัง `WS-RESULT-2DEC` (ทศนิยม 2 หลัก) ได้ `1.11` — ความแม่นยำที่เหลือจากทศนิยม
  หลักที่ 3 เป็นต้นไปถูกตัดทิ้งทั้งหมด
- เมื่อเก็บผลลัพธ์ไปยัง `WS-RESULT-6DEC` (ทศนิยม 6 หลัก) ได้ `1.111111` — เห็นความแม่นยำที่มากขึ้น
  อย่างชัดเจนจากฟิลด์เดียวกัน สมการเดียวกัน เพียงแค่เปลี่ยนขนาด PICTURE ของฟิลด์ปลายทาง
- `WS-A * WS-B` คือ 10.00 × 3.00 = 30.00 พอดี ไม่มีทศนิยมส่วนเกิน จึงไม่มีปัญหาความแม่นยำในกรณีนี้

### ผลลัพธ์ที่ได้จากการรันจริง

```
A/B/C INTO 2 DECIMALS : 00001.11
A/B/C INTO 6 DECIMALS : 00001.111111
A * B (exact)         : 00030.00
```

### ข้อควรระวัง

- ในการคำนวณทางการเงินที่มีหลายขั้นตอนต่อเนื่องกัน (เช่น คำนวณดอกเบี้ยทบต้นหลายงวด) ควรพิจารณา
  ใช้ฟิลด์กลาง (intermediate working field) ที่มีทศนิยมมากกว่าฟิลด์ผลลัพธ์สุดท้าย เพื่อลดการสะสม
  ความคลาดเคลื่อนจากการตัดทศนิยมซ้ำ ๆ หลายครั้ง แล้วค่อยปัดเศษ (ROUNDED) เฉพาะตอนเก็บผลลัพธ์
  สุดท้ายเพียงครั้งเดียว
- อย่าลืมว่าการคำนวณที่ต่อเนื่องกันหลายขั้นตอนใน `COMPUTE` เดียว (เช่น `A / B / C`) COBOL จัดการ
  ความแม่นยำภายในให้ในระดับหนึ่งตามมาตรฐาน แต่ผลลัพธ์สุดท้ายที่เรา `DISPLAY` ออกมายังคงถูกจำกัด
  ด้วยขนาด PICTURE ของฟิลด์ปลายทางเสมอ

### แบบฝึกหัดที่ 88.1

**โจทย์**: เพราะเหตุใดโปรแกรมเมอร์ COBOL ที่คำนวณดอกเบี้ยธนาคารจึงมักประกาศฟิลด์คำนวณกลาง
(intermediate field) ด้วยทศนิยม 4-6 หลัก แม้ว่าฟิลด์แสดงผลสุดท้ายที่ลูกค้าเห็นจะมีแค่ 2 หลัก
(หน่วยสตางค์)?

**เฉลยแนวทาง**: เพื่อรักษาความแม่นยำระหว่างการคำนวณหลายขั้นตอน (เช่น คำนวณดอกเบี้ยรายวันแล้ว
รวมหลายวันเป็นรายเดือน) หากตัดทศนิยมเหลือ 2 หลักตั้งแต่ขั้นตอนกลาง ความคลาดเคลื่อนเล็กน้อยจะถูก
สะสมและขยายใหญ่ขึ้นเมื่อคำนวณซ้ำหลายรอบ ทำให้ผลลัพธ์สุดท้ายคลาดเคลื่อนมากกว่าที่ควรจะเป็น
การเก็บทศนิยมมากไว้ในฟิลด์กลางแล้วปัดเศษเฉพาะตอนแสดงผลครั้งสุดท้ายจึงแม่นยำกว่า

---

## ขั้นตอนที่ 89: Intrinsic Functions ที่ใช้บ่อยในงานคำนวณ

### แนวคิด

นอกจาก `ROUNDED` แล้ว COBOL ยังมี **Intrinsic Functions** (ฟังก์ชันในตัวที่ compiler เตรียมไว้ให้)
ที่ช่วยงานคำนวณทั่วไปโดยไม่ต้องเขียนตรรกะเองตั้งแต่ต้น เรียกใช้ผ่านคำว่า `FUNCTION` ตามด้วยชื่อ
ฟังก์ชันและพารามิเตอร์ในวงเล็บ ฟังก์ชันที่ใช้บ่อยในงานคำนวณมีดังนี้:

| ฟังก์ชัน | หน้าที่ | ตัวอย่าง |
|---|---|---|
| `FUNCTION ABS(x)` | หาค่าสัมบูรณ์ (absolute value) | `FUNCTION ABS(-42)` = 42 |
| `FUNCTION INTEGER(x)` | ปัดเศษลงเป็นจำนวนเต็ม (floor) | `FUNCTION INTEGER(123.45)` = 123 |
| `FUNCTION MOD(x,y)` | หาเศษจากการหาร (modulo) | `FUNCTION MOD(17,5)` = 2 |
| `FUNCTION MAX(...)` | หาค่ามากที่สุดจากรายการ | `FUNCTION MAX(12,87,45,3)` = 87 |
| `FUNCTION SQRT(x)` | หารากที่สอง | `FUNCTION SQRT(16)` = 4 |

หมายเหตุ: GnuCOBOL **ไม่มี** `FUNCTION ROUND` (การปัดเศษทำผ่าน `ROUNDED` phrase ที่เรียนใน
ขั้นตอนที่ 86 แทน ไม่ใช่ intrinsic function)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INTRINSIC-FUNC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-VALUE            PIC 9(5)V99  VALUE 123.45.
       01  WS-INTEGER-VALUE    PIC 9(5).
       01  WS-NEGATIVE-NUM     PIC S9(5)    VALUE -42.
       01  WS-ABS-RESULT       PIC 9(5).
       01  WS-DIVIDEND         PIC 9(5)     VALUE 17.
       01  WS-DIVISOR          PIC 9(5)     VALUE 5.
       01  WS-MOD-RESULT       PIC 9(5).
       01  WS-BIGGEST          PIC 9(5).

       PROCEDURE DIVISION.
           COMPUTE WS-INTEGER-VALUE = FUNCTION INTEGER(WS-VALUE)
           DISPLAY "FUNCTION INTEGER(123.45): " WS-INTEGER-VALUE

           COMPUTE WS-ABS-RESULT = FUNCTION ABS(WS-NEGATIVE-NUM)
           DISPLAY "FUNCTION ABS(-42)       : " WS-ABS-RESULT

           COMPUTE WS-MOD-RESULT =
               FUNCTION MOD(WS-DIVIDEND, WS-DIVISOR)
           DISPLAY "FUNCTION MOD(17, 5)     : " WS-MOD-RESULT

           COMPUTE WS-BIGGEST = FUNCTION MAX(12, 87, 45, 3)
           DISPLAY "FUNCTION MAX(12,87,45,3): " WS-BIGGEST
           STOP RUN.
```

### อธิบายโค้ด

- `FUNCTION INTEGER(WS-VALUE)` — 123.45 ถูกปัดเศษลงเป็นจำนวนเต็ม 123 (ตัดทศนิยมทิ้งแบบ floor
  ไม่ใช่การปัดเศษแบบคณิตศาสตร์)
- `FUNCTION ABS(WS-NEGATIVE-NUM)` — -42 กลายเป็นค่าสัมบูรณ์ 42 มีประโยชน์มากเมื่อต้องการเปรียบเทียบ
  ขนาดของค่าโดยไม่สนใจเครื่องหมาย
- `FUNCTION MOD(WS-DIVIDEND, WS-DIVISOR)` — หาเศษจาก 17 ÷ 5 (17 = 3×5 + 2) ได้เศษ 2
  เทียบเท่ากับการใช้ `DIVIDE ... REMAINDER` แต่กระชับกว่าเมื่อต้องการแค่เศษอย่างเดียว
- `FUNCTION MAX(12, 87, 45, 3)` — หาค่ามากที่สุดจากรายการตัวเลขที่ระบุ ได้ 87 โดยไม่ต้องเขียน
  IF เปรียบเทียบทีละคู่เอง

### ผลลัพธ์ที่ได้จากการรันจริง

```
FUNCTION INTEGER(123.45): 00123
FUNCTION ABS(-42)       : 00042
FUNCTION MOD(17, 5)     : 00002
FUNCTION MAX(12,87,45,3): 00087
```

### ข้อควรระวัง

- ชื่อฟังก์ชันและจำนวนพารามิเตอร์ต้องตรงตามที่มาตรฐานกำหนดเป๊ะ หากพิมพ์ผิดหรือใส่พารามิเตอร์ผิด
  จำนวน คอมไพเลอร์จะแจ้ง error ทันทีตอน compile (ต่างจาก MOVE ที่ error อาจซ่อนอยู่จนถึง runtime)
- `FUNCTION INTEGER` ปัดเศษลงเสมอ (floor) ไม่ใช่การปัดแบบคณิตศาสตร์ทั่วไป (round) — สำหรับค่าลบ
  ต้องระวังเป็นพิเศษเพราะ floor ของค่าลบจะปัดออกห่างจากศูนย์มากขึ้น (เช่น floor ของ -2.5 คือ -3)
- GnuCOBOL แต่ละเวอร์ชันอาจรองรับฟังก์ชันไม่เท่ากันทั้งหมด ควรตรวจสอบด้วยคำสั่ง
  `cobc --list-intrinsics` ก่อนใช้งานฟังก์ชันที่ไม่คุ้นเคย

### แบบฝึกหัดที่ 89.1

**โจทย์**: จงเขียนคำสั่งตรวจสอบว่าตัวเลขใน `WS-NUM` เป็นเลขคู่หรือเลขคี่ โดยใช้ `FUNCTION MOD`

**เฉลย**:
```cobol
IF FUNCTION MOD(WS-NUM, 2) = 0
    DISPLAY "EVEN NUMBER"
ELSE
    DISPLAY "ODD NUMBER"
END-IF
```
หลักการคือถ้าเศษจากการหารด้วย 2 เป็น 0 แสดงว่าเป็นเลขคู่ มิฉะนั้นเป็นเลขคี่

---

## ขั้นตอนที่ 90: โปรแกรมรวบยอด — เครื่องคิดเลขขนาดย่อม (Mini Calculator)

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่รวมคำสั่งเลขคณิตทั้งหมดที่เรียนมา (`ADD`, `SUBTRACT`, `MULTIPLY`,
`DIVIDE`, `COMPUTE`) เข้าไว้ในโปรแกรมเดียว พร้อมใช้ `ROUNDED` อย่างเหมาะสม เพื่อแสดงให้เห็นภาพรวม
ว่าคำสั่งเหล่านี้ทำงานร่วมกันในสถานการณ์จริงได้อย่างไร

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MINI-CALCULATOR.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NUM-1            PIC S9(7)V99.
       01  WS-NUM-2            PIC S9(7)V99.
       01  WS-SUM              PIC S9(8)V99.
       01  WS-DIFF             PIC S9(8)V99.
       01  WS-PRODUCT          PIC S9(9)V99.
       01  WS-QUOTIENT         PIC S9(7)V9999.

       PROCEDURE DIVISION.
           MOVE 150.50 TO WS-NUM-1
           MOVE 40.25  TO WS-NUM-2

           DISPLAY "=== MINI CALCULATOR ==="
           DISPLAY "NUM-1 = " WS-NUM-1
           DISPLAY "NUM-2 = " WS-NUM-2
           DISPLAY " "

           ADD WS-NUM-1 WS-NUM-2 GIVING WS-SUM
           DISPLAY "SUM        : " WS-SUM

           SUBTRACT WS-NUM-2 FROM WS-NUM-1 GIVING WS-DIFF
           DISPLAY "DIFFERENCE : " WS-DIFF

           MULTIPLY WS-NUM-1 BY WS-NUM-2 GIVING WS-PRODUCT
           DISPLAY "PRODUCT    : " WS-PRODUCT

           DIVIDE WS-NUM-1 BY WS-NUM-2
               GIVING WS-QUOTIENT ROUNDED
           DISPLAY "QUOTIENT   : " WS-QUOTIENT

           COMPUTE WS-SUM ROUNDED =
               (WS-NUM-1 + WS-NUM-2) * 2
           DISPLAY "(N1+N2)*2  : " WS-SUM
           STOP RUN.
```

### อธิบายโค้ด

- ตัวแปรทุกตัวใช้ `PIC S9...` (มี S) เพื่อรองรับผลลัพธ์ที่อาจติดลบได้ในอนาคต แม้ตัวอย่างนี้จะใช้
  เฉพาะเลขบวกก็ตาม เป็นแนวปฏิบัติที่ดีสำหรับฟิลด์คำนวณทั่วไป
- ขนาดฟิลด์ผลลัพธ์ (`WS-SUM`, `WS-PRODUCT`) ถูกออกแบบให้ใหญ่กว่าฟิลด์ตั้งต้นเพื่อรองรับผลลัพธ์
  ที่มีจำนวนหลักเพิ่มขึ้นจากการคำนวณ (โดยเฉพาะการคูณที่ `WS-PRODUCT` มี 9 หลักเทียบกับตั้งต้น 7 หลัก)
- `DIVIDE WS-NUM-1 BY WS-NUM-2 GIVING WS-QUOTIENT ROUNDED` ใช้ `ROUNDED` เพื่อความแม่นยำของผลหาร
  ที่มีทศนิยม 4 ตำแหน่ง
- `COMPUTE WS-SUM ROUNDED = (WS-NUM-1 + WS-NUM-2) * 2` แสดงให้เห็นว่า `COMPUTE` สามารถรวม
  การคำนวณหลายขั้นตอน (บวกแล้วคูณ) พร้อมการปัดเศษไว้ในบรรทัดเดียว และนำผลลัพธ์กลับมาเก็บใน
  ตัวแปรเดิม (`WS-SUM`) ซ้ำได้

### ผลลัพธ์ที่ได้จากการรันจริง

```
=== MINI CALCULATOR ===
NUM-1 = +0000150.50
NUM-2 = +0000040.25

SUM        : +00000190.75
DIFFERENCE : +00000110.25
PRODUCT    : +000006057.62
QUOTIENT   : +0000003.7391
(N1+N2)*2  : +00000381.50
```

### ข้อควรระวัง

- สังเกตว่าเครื่องหมาย `+` แสดงนำหน้าตัวเลขเสมอเมื่อฟิลด์เป็น `PIC S9...` และแสดงผลผ่าน `DISPLAY`
  ตรง ๆ (ไม่ผ่าน PICTURE Edited) นี่คือพฤติกรรมมาตรฐานที่ต้องคุ้นเคยไว้ หากต้องการซ่อนเครื่องหมาย
  `+` เมื่อเป็นค่าบวก ต้องใช้ PICTURE Edited ที่เรียนใน Part 008 ขั้นตอนที่ 77
- การออกแบบขนาดฟิลด์ผลลัพธ์ให้เหมาะสมกับแต่ละคำสั่ง (บวก/ลบ/คูณ/หาร) เป็นทักษะสำคัญที่ต้องฝึกฝน
  เพราะแต่ละการคำนวณมีแนวโน้มทำให้จำนวนหลักเพิ่มขึ้นต่างกัน (การคูณเพิ่มหลักเร็วที่สุด)

### แบบฝึกหัดที่ 90.1

**โจทย์**: จงต่อยอดโปรแกรมเครื่องคิดเลขข้างต้น โดยเพิ่มการคำนวณเปอร์เซ็นต์ที่ NUM-1 คิดเป็นกี่
เปอร์เซ็นต์ของผลรวม (SUM) แล้วเก็บผลลัพธ์ไว้ในตัวแปรใหม่ `WS-PERCENT`

**เฉลยแนวทาง**:
```cobol
01  WS-PERCENT PIC 9(3)V99.
...
COMPUTE WS-PERCENT ROUNDED = (WS-NUM-1 / WS-SUM) * 100
DISPLAY "NUM-1 PERCENT OF SUM: " WS-PERCENT
```
การคำนวณคือนำ NUM-1 หารด้วยผลรวมทั้งหมดแล้วคูณ 100 เพื่อแปลงเป็นเปอร์เซ็นต์ พร้อมใช้ `ROUNDED`
เพื่อความแม่นยำของผลลัพธ์ทศนิยม

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้คำสั่งเลขคณิตพื้นฐานทั้งหมดของ COBOL อย่างครบถ้วน:

- คำสั่ง `ADD` ทั้ง 3 รูปแบบ: `TO`, `GIVING`, และการบวกหลายฟิลด์พร้อมกัน
- คำสั่ง `SUBTRACT` และทิศทางของคำว่า `FROM` ที่ต้องจำให้แม่น
- คำสั่ง `MULTIPLY` และการออกแบบขนาดฟิลด์ผลลัพธ์ให้รองรับจำนวนหลักที่เพิ่มขึ้น
- คำสั่ง `DIVIDE` ทั้งรูปแบบ `INTO`, `BY...GIVING`, และ `REMAINDER`
- คำสั่ง `COMPUTE` สำหรับเขียนสมการคณิตศาสตร์แบบเต็มรูปแบบพร้อมลำดับการคำนวณมาตรฐาน
- `ROUNDED` phrase สำหรับการปัดเศษที่ถูกต้อง แทนการตัดทิ้งแบบค่าเริ่มต้น
- `ON SIZE ERROR` สำหรับดักจับข้อผิดพลาดจากค่าล้นฟิลด์และการหารด้วยศูนย์
- ความแม่นยำของทศนิยมและผลลัพธ์กลางในการคำนวณหลายขั้นตอน
- Intrinsic Functions ที่ใช้บ่อย: `ABS`, `INTEGER`, `MOD`, `MAX`, `SQRT`
- โปรแกรมรวบยอดเครื่องคิดเลขที่ผสานทุกคำสั่งเข้าด้วยกัน

ความเข้าใจเรื่องเลขคณิตนี้จะเป็นพื้นฐานสำคัญสำหรับ Part ถัดไป ซึ่งเราจะเรียนรู้การตัดสินใจ
เชิงตรรกะด้วย `IF-ELSE` และ Condition Names (88-level) — ทักษะที่จำเป็นสำหรับการเขียนโปรแกรมที่
ต้องตัดสินใจแตกต่างกันตามเงื่อนไขต่าง ๆ เช่น การตรวจสอบว่ายอดเงินเพียงพอหรือไม่ก่อนคำนวณดอกเบี้ย

**[← กลับไป Part 008](part-008-move-statement.md)** | **[ไปยัง Part 010: เงื่อนไข IF-ELSE และ Condition Names →](part-010-if-else-condition-names.md)**
