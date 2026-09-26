# Part 036: Intrinsic Functions: ตัวเลขและวันที่ (FUNCTION) (ขั้นตอนที่ 351–360)

## คำนำของ Part นี้

ยินดีต้อนรับสู่ **เฟส 3: ขั้นสูง (Advanced)** ของหลักสูตร! เราปิดเฟส 2 ไปแล้วด้วยโปรเจกต์ระบบ
จัดการสินค้าคงคลังใน Part 035 ที่ผสมผสานตาราง, การจัดการไฟล์, Subprogram, และ Copybook เข้า
ด้วยกัน ตอนนี้เราจะเริ่มเฟสใหม่ด้วยเครื่องมือที่ทรงพลังและใช้บ่อยที่สุดตัวหนึ่งใน COBOL สมัยใหม่:
**Intrinsic Functions** หรือ "ฟังก์ชันในตัว" ที่คอมไพเลอร์เตรียมไว้ให้ใช้งานได้ทันทีโดยไม่ต้องเขียน
โค้ดคำนวณเอง

ก่อนที่ COBOL-85 จะเพิ่ม Intrinsic Functions เข้ามา หากเราต้องการหาค่ารากที่สอง หรือหาผลต่าง
ระหว่างวันที่สองวัน เราต้องเขียนตรรกะการคำนวณเองทั้งหมด (ซึ่งเสี่ยงต่อบั๊กและกินเวลามาก) การมี
`FUNCTION` ในตัวช่วยให้โค้ดสั้นลง อ่านง่ายขึ้น และลดโอกาสเกิดบั๊กจากการคำนวณเอง Part นี้จะเน้นไปที่
กลุ่มฟังก์ชันด้าน **ตัวเลข** (MOD, REM, SQRT, MAX, MIN, ABS, FACTORIAL, INTEGER) และ
**วันที่** (CURRENT-DATE, INTEGER-OF-DATE, DATE-OF-INTEGER) พร้อมเทคนิคการทำเลขคณิตวันที่
(Date Arithmetic) ที่ใช้บ่อยมากในงานธุรกิจจริง เช่น การหาอายุ หรือการคำนวณวันครบกำหนดชำระเงิน
ส่วนฟังก์ชันด้านข้อความและสถิติจะแยกไปสอนใน Part 037

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และทุกตัวอย่าง
> ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง
>
> **หมายเหตุพิเศษสำหรับ Part นี้**: ชื่อฟังก์ชัน Intrinsic Function หลายตัวมีความยาวมาก (เช่น
> `INTEGER-OF-DATE`, `STANDARD-DEVIATION`) เมื่อรวมกับชื่อตัวแปรที่มีความหมายชัดเจน บรรทัด
> `COMPUTE` มักยาวเกิน**คอลัมน์ 72** ได้ง่ายกว่าโค้ดทั่วไปมาก ตัวอย่างในเอกสารนี้จึงมักตัดบรรทัด
> `COMPUTE`/`MOVE` ออกเป็นสองบรรทัดโดยตั้งใจ — เป็นแบบแผนที่ควรฝึกให้ชินตั้งแต่ต้น

---

## ขั้นตอนที่ 351: ไวยากรณ์ FUNCTION ทั่วไป และ FUNCTION MOD, FUNCTION REM

### ไวยากรณ์พื้นฐานของการเรียกใช้ FUNCTION

ทุก Intrinsic Function ใน COBOL เริ่มต้นด้วยคำสงวน **`FUNCTION`** ตามด้วยชื่อฟังก์ชัน และวงเล็บ
ที่ครอบอาร์กิวเมนต์ (argument) ที่ส่งเข้าไป:

```
FUNCTION ชื่อฟังก์ชัน(อาร์กิวเมนต์1 อาร์กิวเมนต์2 ...)
```

ผลลัพธ์ของ `FUNCTION` สามารถนำไปใช้ได้ทุกที่ที่ต้องการ "ค่า" เช่น ฝั่งขวาของ `COMPUTE`, ใน
`MOVE`, หรือแม้แต่ใน `DISPLAY` โดยตรง

### FUNCTION MOD และ FUNCTION REM — สองฟังก์ชันหาเศษที่ดูคล้ายกันแต่ต่างกัน

`FUNCTION MOD(a b)` และ `FUNCTION REM(a b)` ต่างก็หา "เศษ" จากการหาร `a / b` แต่ต่างกันตรงที่
**เครื่องหมาย (sign)** ของผลลัพธ์เมื่อตัวเลขติดลบ: `MOD` จะให้ผลลัพธ์ที่มีเครื่องหมายตามตัวหาร (b)
เสมอ ในขณะที่ `REM` ให้เศษที่แท้จริงตามการหารปกติ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP351-MOD-REM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A                PIC 9(4) VALUE 17.
       01  WS-B                PIC 9(4) VALUE 5.
       01  WS-RESULT           PIC S9(6)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> FUNCTION MOD returns the same sign as the divisor.
           COMPUTE WS-RESULT = FUNCTION MOD(WS-A WS-B).
           DISPLAY "MOD(17, 5)  = " WS-RESULT.

      *> FUNCTION REM returns the true division remainder.
           COMPUTE WS-RESULT = FUNCTION REM(WS-A WS-B).
           DISPLAY "REM(17, 5)  = " WS-RESULT.

           COMPUTE WS-RESULT = FUNCTION MOD(-17 5).
           DISPLAY "MOD(-17, 5) = " WS-RESULT.
           COMPUTE WS-RESULT = FUNCTION REM(-17 5).
           DISPLAY "REM(-17, 5) = " WS-RESULT.
           STOP RUN.
```

**ผลลัพธ์:**

```
MOD(17, 5)  = +000002.00
REM(17, 5)  = +000002.00
MOD(-17, 5) = +000003.00
REM(-17, 5) = -000002.00
```

### อธิบายจุดสำคัญ

- เมื่อทั้งสองค่าเป็นบวก (`17` และ `5`) `MOD` กับ `REM` ให้ผลลัพธ์เหมือนกันคือ `2` (เพราะ 17 = 3×5
  + 2)
- เมื่อตัวตั้งติดลบ (`-17` และ `5`) จะเห็นความต่างชัดเจน: `MOD(-17, 5) = 3` (เครื่องหมายตามตัวหาร
  ที่เป็นบวก) ในขณะที่ `REM(-17, 5) = -2` (เศษจริงจากการหารที่ -17 = (-4)×5 + 3... ในทางคณิตศาสตร์
  `REM` คำนวณจากสูตร `a - (b * FUNCTION INTEGER-PART(a / b))` ในขณะที่ `MOD` ปรับผลลัพธ์ให้อยู่
  ในช่วงเดียวกับเครื่องหมายของตัวหารเสมอ)
- ทั้งสองฟังก์ชันรับอาร์กิวเมนต์ 2 ตัวเสมอ คั่นด้วยช่องว่าง (ไม่ใช่จุลภาค) ภายในวงเล็บเดียวกัน

### ข้อควรระวัง

- อย่าใช้ `MOD` และ `REM` สลับกันโดยไม่ได้ตั้งใจ โดยเฉพาะเมื่อทำงานกับตัวเลขที่อาจติดลบ (เช่น
  การคำนวณที่เกี่ยวกับผลต่างวันที่หรือยอดเงินคงเหลือที่อาจติดลบ) เพราะผลลัพธ์อาจต่างกันและนำไปสู่
  บั๊กทางธุรกิจที่ตรวจจับยาก
- `PIC S9(6)V99` ในตัวอย่างนี้ต้องมีเครื่องหมาย `S` เพื่อรองรับผลลัพธ์ติดลบจาก `REM` มิฉะนั้นค่าที่
  ควรติดลบจะถูกเก็บเป็นค่าสัมบูรณ์ (Absolute Value) แทน ทำให้ผลลัพธ์ผิดโดยไม่มี Error เตือน

### แบบฝึกหัดที่ 351.1

**โจทย์**: จงคำนวณ `FUNCTION MOD(20 -3)` และ `FUNCTION REM(20 -3)` ด้วยมือ แล้วเขียนโค้ด
ทดสอบเพื่อตรวจคำตอบ

**เฉลย**: ตามกฎ `MOD` จะให้เครื่องหมายตามตัวหาร (`-3` เป็นลบ) ดังนั้น `MOD(20, -3) = -1` ส่วน
`REM(20, -3) = 2` (เพราะ 20 = (-6)×(-3) + 2) โค้ดทดสอบ:

```cobol
           COMPUTE WS-RESULT = FUNCTION MOD(20 -3).
           DISPLAY "MOD(20,-3) = " WS-RESULT.
           COMPUTE WS-RESULT = FUNCTION REM(20 -3).
           DISPLAY "REM(20,-3) = " WS-RESULT.
```

---

## ขั้นตอนที่ 352: FUNCTION SQRT, FUNCTION ABS, FUNCTION FACTORIAL

### สามฟังก์ชันคณิตศาสตร์พื้นฐานที่ใช้บ่อย

- **`FUNCTION SQRT(x)`** หารากที่สองของ `x`
- **`FUNCTION ABS(x)`** หาค่าสัมบูรณ์ (ตัดเครื่องหมายลบทิ้ง)
- **`FUNCTION FACTORIAL(x)`** หาค่าแฟกทอเรียล (x!) ของจำนวนเต็มไม่ติดลบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP352-SQRT-ABS-FACT.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RESULT           PIC S9(6)V9999.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-RESULT = FUNCTION SQRT(144).
           DISPLAY "SQRT(144) = " WS-RESULT.
           COMPUTE WS-RESULT = FUNCTION SQRT(2).
           DISPLAY "SQRT(2)   = " WS-RESULT.

           COMPUTE WS-RESULT = FUNCTION ABS(-42.5).
           DISPLAY "ABS(-42.5) = " WS-RESULT.

           COMPUTE WS-RESULT = FUNCTION FACTORIAL(6).
           DISPLAY "FACTORIAL(6) = " WS-RESULT.

      *> Pitfall: SQRT of a negative number does NOT raise an
      *> error here - it silently returns zero.
           DISPLAY "SQRT(-4), NO ERROR RAISED: "
               FUNCTION SQRT(-4).
           STOP RUN.
```

**ผลลัพธ์:**

```
SQRT(144) = +000012.0000
SQRT(2)   = +000001.4142
ABS(-42.5) = +000042.5000
FACTORIAL(6) = +000720.0000
SQRT(-4), NO ERROR RAISED: 0000000000
```

### อธิบายจุดสำคัญ

- `SQRT(144) = 12` และ `SQRT(2) = 1.4142` (ปัดตามจำนวนตำแหน่งทศนิยมที่ `PIC` ของตัวแปรปลายทาง
  กำหนดไว้ คือ `V9999` สี่ตำแหน่ง)
- `ABS(-42.5) = 42.5` — ตัดเครื่องหมายลบออกอย่างง่ายดาย มีประโยชน์มากเมื่อต้องเปรียบเทียบ "ขนาด"
  ของผลต่างโดยไม่สนใจทิศทาง เช่น ผลต่างระหว่างยอดที่คาดหวังกับยอดจริงในงาน Reconciliation
- `FACTORIAL(6) = 720` (6×5×4×3×2×1) มีประโยชน์ในการคำนวณเชิงสถิติและความน่าจะเป็น
- `DISPLAY FUNCTION SQRT(-4)` เรียกใช้ `FUNCTION` ได้โดยตรงใน `DISPLAY` โดยไม่ต้องผ่าน `COMPUTE`
  หรือ `MOVE` ก่อน — เป็นรูปแบบที่กระชับและใช้บ่อยเมื่อไม่ต้องการเก็บผลลัพธ์ไว้ใช้ต่อ

### ข้อควรระวัง: SQRT ของค่าติดลบไม่ Error แต่คืนค่าศูนย์เงียบ ๆ

จากตัวอย่างข้างบน `FUNCTION SQRT(-4)` ควรจะเป็นข้อผิดพลาดทางคณิตศาสตร์ (รากที่สองของจำนวนลบ
ไม่มีค่าจริง) แต่ GnuCOBOL **ไม่แจ้ง Error ใด ๆ** ทั้งตอนคอมไพล์และตอนรัน มันเพียงคืนค่า `0`
เงียบ ๆ นี่เป็นพฤติกรรมที่อันตรายมาก เพราะหากมีข้อมูลผิดพลาด (เช่น ค่าที่ควรเป็นบวกกลับติดลบเพราะ
บั๊กจุดอื่น) แล้วถูกส่งเข้า `FUNCTION SQRT` โปรแกรมจะไม่แจ้งเตือนเราเลยว่ามีบางอย่างผิดปกติ ผลลัพธ์
`0` ที่ได้อาจถูกนำไปคำนวณต่อและกลายเป็นข้อมูลผิดพลาดที่ตรวจจับได้ยากในภายหลัง

**แนวทางป้องกัน**: ควรตรวจสอบค่าก่อนส่งเข้า `FUNCTION SQRT` เสมอด้วย `IF x < 0` หากมีโอกาสที่
ค่านั้นจะติดลบจากข้อมูลนำเข้าที่ไม่น่าเชื่อถือ

### แบบฝึกหัดที่ 352.1

**โจทย์**: จงเขียนโค้ดที่ตรวจสอบก่อนเรียก `FUNCTION SQRT` ว่าค่าที่จะส่งเข้าไปต้องไม่ติดลบ หากติดลบ
ให้แสดงข้อความเตือนแทนที่จะคำนวณต่อ

**เฉลย**:

```cobol
       01  WS-INPUT-VALUE      PIC S9(6)V99 VALUE -9.
       01  WS-SQRT-RESULT      PIC S9(6)V9999.
       ...
           IF WS-INPUT-VALUE < 0
               DISPLAY "ERROR: CANNOT TAKE SQRT OF A NEGATIVE VALUE"
           ELSE
               COMPUTE WS-SQRT-RESULT = FUNCTION SQRT(WS-INPUT-VALUE)
               DISPLAY "SQRT RESULT: " WS-SQRT-RESULT
           END-IF.
```

---

## ขั้นตอนที่ 353: FUNCTION MAX และ FUNCTION MIN

### หาค่ามากที่สุดและน้อยที่สุดโดยไม่ต้องเขียน IF ซ้อนกันเอง

ก่อนมี Intrinsic Function การหาค่ามากที่สุดจากตัวแปรหลายตัวต้องเขียน `IF` เปรียบเทียบทีละคู่ ซึ่ง
ยิ่งมีตัวแปรมากยิ่งเขียนยุ่งยากและเสี่ยงผิดพลาด `FUNCTION MAX` และ `FUNCTION MIN` รับอาร์กิวเมนต์
ได้ไม่จำกัดจำนวน ทำให้โค้ดสั้นและอ่านง่ายขึ้นมาก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP353-MAX-MIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SCORE-1          PIC 9(3) VALUE 78.
       01  WS-SCORE-2          PIC 9(3) VALUE 95.
       01  WS-SCORE-3          PIC 9(3) VALUE 62.
       01  WS-RESULT           PIC 9(3).

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-RESULT =
               FUNCTION MAX(WS-SCORE-1 WS-SCORE-2 WS-SCORE-3).
           DISPLAY "HIGHEST SCORE: " WS-RESULT.

           COMPUTE WS-RESULT =
               FUNCTION MIN(WS-SCORE-1 WS-SCORE-2 WS-SCORE-3).
           DISPLAY "LOWEST SCORE:  " WS-RESULT.

           DISPLAY "MAX OF LITERALS: " FUNCTION MAX(3 88 21 6).
           STOP RUN.
```

**ผลลัพธ์:**

```
HIGHEST SCORE: 095
LOWEST SCORE:  062
MAX OF LITERALS: 88
```

### อธิบายจุดสำคัญ

- `FUNCTION MAX(WS-SCORE-1 WS-SCORE-2 WS-SCORE-3)` เปรียบเทียบทั้ง 3 ตัวแปรพร้อมกันในคำสั่ง
  เดียว ไม่ต้องเขียน `IF WS-SCORE-1 > WS-SCORE-2 ... ` ซ้อนกันหลายชั้นเหมือนวิธีดั้งเดิม
- `FUNCTION MAX` และ `MIN` รับได้ทั้งตัวแปรและค่า Literal ปนกัน และรับจำนวนอาร์กิวเมนต์ได้ไม่จำกัด
  (ในทางปฏิบัติจำกัดตามขีดความสามารถของคอมไพเลอร์ แต่เพียงพอสำหรับงานทั่วไป)
- ฟังก์ชันนี้เปรียบเทียบได้ทั้งข้อมูลตัวเลขและข้อมูลตัวอักษร (Alphanumeric) ตามกฎการเรียงลำดับ
  (Collating Sequence) ของระบบ แต่ในหลักสูตรนี้เราเน้นการใช้กับตัวเลขเป็นหลัก

### ข้อควรระวัง

- เมื่อเปรียบเทียบฟิลด์ที่มีทศนิยมต่างจำนวนตำแหน่งกัน (เช่น `PIC 9V9` กับ `PIC 9V99`) ควรตรวจสอบ
  ว่าตัวแปรปลายทางที่เก็บผลลัพธ์มีความละเอียดเพียงพอ มิฉะนั้นอาจเกิดการปัดเศษที่ไม่ตั้งใจ
- `FUNCTION MAX`/`MIN` เปรียบเทียบเฉพาะ**ค่า**ที่ส่งเข้าไปในขณะนั้น มันไม่ใช่ฟังก์ชันที่ค้นหาค่ามากสุด
  ในตาราง (Table) ทั้งตารางโดยอัตโนมัติ — หากต้องการหาค่ามากสุดในตาราง `OCCURS` ที่มีขนาดใหญ่
  ยังคงต้องใช้ `PERFORM VARYING` วนลูปเปรียบเทียบเอง (หรือส่งอาร์กิวเมนต์ทุกตัวเข้า `FUNCTION MAX`
  โดยตรงหากตารางมีขนาดคงที่และไม่ใหญ่เกินไป)

### แบบฝึกหัดที่ 353.1

**โจทย์**: จงเขียนโค้ดหาคะแนน "ช่วงห่าง" (Range) ระหว่างคะแนนสูงสุดกับต่ำสุดจากตัวแปร
`WS-SCORE-1`, `WS-SCORE-2`, `WS-SCORE-3` โดยใช้ `FUNCTION MAX` และ `FUNCTION MIN` ร่วมกัน

**เฉลย**:

```cobol
       01  WS-RANGE            PIC 9(3).
       ...
           COMPUTE WS-RANGE =
               FUNCTION MAX(WS-SCORE-1 WS-SCORE-2 WS-SCORE-3) -
               FUNCTION MIN(WS-SCORE-1 WS-SCORE-2 WS-SCORE-3).
           DISPLAY "SCORE RANGE: " WS-RANGE.
```

ผลลัพธ์ที่คาดไว้: `SCORE RANGE: 033` (95 - 62)

---

## ขั้นตอนที่ 354: การปัดเศษ — ROUNDED Phrase, FUNCTION INTEGER, FUNCTION INTEGER-PART

### ความแตกต่างระหว่างการตัดทอนกับการปัดเศษ

ตามค่าเริ่มต้น เมื่อ `COMPUTE` ได้ผลลัพธ์ที่มีทศนิยมมากกว่าที่ตัวแปรปลายทางเก็บได้ COBOL จะ
**ตัดทอน (Truncate)** ตัวเลขส่วนเกินทิ้งไปเฉย ๆ ไม่ใช่ปัดเศษ หากต้องการให้ปัดเศษ (Round) ต้องเติม
วลี **`ROUNDED`** ต่อท้ายชื่อตัวแปรปลายทางใน `COMPUTE` อย่างชัดเจน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP354-ROUNDING.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NO-ROUND         PIC S9(4)V99.
       01  WS-ROUNDED          PIC S9(4)V99.
       01  WS-INT-RESULT       PIC S9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Without ROUNDED, COMPUTE simply truncates extra digits.
           COMPUTE WS-NO-ROUND = 20 / 3.
           DISPLAY "20/3 (TRUNCATED): " WS-NO-ROUND.

      *> ROUNDED asks COBOL to round to the nearest storable value.
           COMPUTE WS-ROUNDED ROUNDED = 20 / 3.
           DISPLAY "20/3 ROUNDED:     " WS-ROUNDED.

      *> FUNCTION INTEGER always rounds DOWN (toward negative
      *> infinity), unlike simple truncation.
           COMPUTE WS-INT-RESULT = FUNCTION INTEGER(7.8).
           DISPLAY "INTEGER(7.8)  = " WS-INT-RESULT.
           COMPUTE WS-INT-RESULT = FUNCTION INTEGER(-7.8).
           DISPLAY "INTEGER(-7.8) = " WS-INT-RESULT.

      *> FUNCTION INTEGER-PART truncates toward zero instead.
           COMPUTE WS-INT-RESULT = FUNCTION INTEGER-PART(7.8).
           DISPLAY "INTEGER-PART(7.8)  = " WS-INT-RESULT.
           COMPUTE WS-INT-RESULT = FUNCTION INTEGER-PART(-7.8).
           DISPLAY "INTEGER-PART(-7.8) = " WS-INT-RESULT.
           STOP RUN.
```

**ผลลัพธ์:**

```
20/3 (TRUNCATED): +0006.66
20/3 ROUNDED:     +0006.67
INTEGER(7.8)  = +0007
INTEGER(-7.8) = -0008
INTEGER-PART(7.8)  = +0007
INTEGER-PART(-7.8) = -0007
```

### อธิบายจุดสำคัญ

- `20 / 3 = 6.6666...` — เมื่อไม่ใส่ `ROUNDED` ได้ผล `6.66` (ตัดทอนตำแหน่งที่ 3 ทิ้งไปเฉย ๆ)
  เมื่อใส่ `ROUNDED` ได้ผล `6.67` (ปัดขึ้นเพราะตำแหน่งที่ 3 คือ `6` ซึ่ง ≥ 5)
- `FUNCTION INTEGER(x)` ปัดลงเสมอไปทาง**ค่าลบอนันต์** (Round toward negative infinity) สังเกตว่า
  `INTEGER(7.8) = 7` (ปัดลงธรรมดา) แต่ `INTEGER(-7.8) = -8` (ปัดลงต่อไปอีก ไม่ใช่ -7) เพราะ -8
  น้อยกว่า -7.8 ในทิศทางลบอนันต์
- `FUNCTION INTEGER-PART(x)` ตัดทอนไปทาง**ศูนย์**เสมอ (Truncate toward zero) — `INTEGER-PART
  (-7.8) = -7` (ตัดทศนิยมทิ้งตรง ๆ โดยไม่สนทิศทาง) ต่างจาก `FUNCTION INTEGER` ที่ปัดลงเสมอ
- ความแตกต่างระหว่าง `INTEGER` กับ `INTEGER-PART` จะเห็นชัดเฉพาะกับค่าติดลบเท่านั้น — สำหรับค่า
  บวก ทั้งสองฟังก์ชันให้ผลลัพธ์เหมือนกัน

### ข้อควรระวัง

- ต้องแยกให้ชัดว่า `ROUNDED` เป็น**วลี** (phrase) ที่ใช้ร่วมกับ `COMPUTE`/`ADD`/`SUBTRACT` ฯลฯ
  ไม่ใช่ Intrinsic Function จึงเขียนว่า `COMPUTE x ROUNDED = ...` ไม่ใช่ `FUNCTION ROUNDED(...)`
- งานที่เกี่ยวกับเงิน (Currency) มักต้องการ `ROUNDED` เสมอ เพราะการตัดทอนธรรมดาจะทำให้ยอดเงิน
  ขาดหายไปทีละเล็กละน้อยสะสมเป็นความคลาดเคลื่อนจำนวนมากเมื่อมีธุรกรรมจำนวนมาก (ปัญหานี้เคยเกิด
  ขึ้นจริงในประวัติศาสตร์ที่เรียกว่า "Salami Slicing Fraud" ซึ่งอาศัยเศษเงินที่ถูกตัดทอนทิ้งไป)
- การเลือกระหว่าง `FUNCTION INTEGER` กับ `FUNCTION INTEGER-PART` สำคัญมากเมื่อทำงานกับค่าติดลบ
  เลือกผิดอาจทำให้ผลลัพธ์คลาดเคลื่อนไป 1 หน่วยโดยไม่รู้ตัว

### แบบฝึกหัดที่ 354.1

**โจทย์**: จงอธิบายว่าทำไม `FUNCTION INTEGER(-7.8)` จึงได้ `-8` ในขณะที่ `FUNCTION INTEGER-PART
(-7.8)` ได้ `-7`

**เฉลย**: `FUNCTION INTEGER` ปัดลงไปทาง**ค่าลบอนันต์เสมอ** ไม่ว่าค่าจะเป็นบวกหรือลบ เมื่อ
`-7.8` ถูกปัดลง (เคลื่อนไปทางซ้ายบนเส้นจำนวน) จะได้ `-8` เพราะ `-8 < -7.8 < -7` ส่วน
`FUNCTION INTEGER-PART` เพียงแค่ตัดส่วนทศนิยมทิ้งไปตรง ๆ (เคลื่อนเข้าหาศูนย์) จาก `-7.8` เมื่อตัด
ทศนิยม `.8` ทิ้งจะเหลือ `-7` เท่านั้น ไม่ได้เคลื่อนต่อไปยัง `-8` ความแตกต่างนี้คือ "ทิศทางการปัด" ที่
ต่างกัน: หนึ่งปัดลงเสมอ อีกหนึ่งปัดเข้าหาศูนย์เสมอ

---

## ขั้นตอนที่ 355: FUNCTION CURRENT-DATE และ FUNCTION WHEN-COMPILED

### การอ่านวันที่และเวลาปัจจุบันจากระบบ

`FUNCTION CURRENT-DATE` คืนค่าข้อความความยาว 21 ตัวอักษร ที่ประกอบด้วยวันที่ เวลา และ Time
Zone Offset ของเครื่องที่รันโปรแกรมอยู่ ในรูปแบบ `YYYYMMDDHHMMSSssZZZZZ` (ปี-เดือน-วัน,
ชั่วโมง-นาที-วินาที, เสี้ยววินาที, และผลต่างจาก UTC)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP355-CURRENT-DATE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FULL-STAMP       PIC X(21).
       01  WS-TODAY            PIC X(8).
       01  WS-TIME-PART        PIC X(8).
       01  WS-COMPILE-STAMP    PIC X(21).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> FUNCTION CURRENT-DATE returns 21 characters:
      *> YYYYMMDDHHMMSSss+HHMM (date, time, hundredths, UTC offset).
           MOVE FUNCTION CURRENT-DATE TO WS-FULL-STAMP.
           DISPLAY "FULL STAMP (21 CHARS): " WS-FULL-STAMP.

           MOVE FUNCTION CURRENT-DATE(1:8) TO WS-TODAY.
           DISPLAY "DATE PART (YYYYMMDD):  " WS-TODAY.

           MOVE FUNCTION CURRENT-DATE(9:8) TO WS-TIME-PART.
           DISPLAY "TIME PART (HHMMSSss):  " WS-TIME-PART.

      *> FUNCTION WHEN-COMPILED - same 21-char format, but the
      *> moment this program was COMPILED, not run.
           MOVE FUNCTION WHEN-COMPILED TO WS-COMPILE-STAMP.
           DISPLAY "COMPILED AT:           " WS-COMPILE-STAMP.
           STOP RUN.
```

**ผลลัพธ์ (ตัวเลขจริงจะเปลี่ยนไปตามวันเวลาที่รัน):**

```
FULL STAMP (21 CHARS): 2026092621151521+0000
DATE PART (YYYYMMDD):  20260926
TIME PART (HHMMSSss):  21151521
COMPILED AT:           2026092621151514+0000
```

### อธิบายจุดสำคัญ

- `FUNCTION CURRENT-DATE(1:8)` ใช้ **Reference Modification** (ที่เรียนใน Part 019-020 เรื่อง
  STRING/UNSTRING) เพื่อตัดเอาเฉพาะ 8 ตัวอักษรแรก (ส่วนวันที่ `YYYYMMDD`) ออกมาจากผลลัพธ์เต็ม
  21 ตัวอักษร ไวยากรณ์ `(ตำแหน่งเริ่มต้น:ความยาว)` ใช้ได้กับผลลัพธ์ของ `FUNCTION` เช่นเดียวกับที่ใช้
  กับตัวแปรทั่วไป
- `FUNCTION CURRENT-DATE(9:8)` ตัดเอา 8 ตัวอักษรถัดไป (เริ่มที่ตำแหน่งที่ 9) ซึ่งคือส่วนเวลา
  `HHMMSSss` (ชั่วโมง-นาที-วินาที-เสี้ยววินาที)
- `FUNCTION WHEN-COMPILED` มีรูปแบบเดียวกับ `CURRENT-DATE` เป๊ะ (21 ตัวอักษร) แต่คืนค่า ณ
  **เวลาที่คอมไพล์โปรแกรมนี้** ไม่ใช่เวลาที่รัน มีประโยชน์มากสำหรับการติดตาม Version ของโปรแกรมที่
  Deploy อยู่ในระบบจริง (เช่น แสดงในหน้าจอ About หรือ Log เพื่อยืนยันว่าเซิร์ฟเวอร์กำลังรันโค้ดเวอร์ชัน
  ล่าสุดที่คอมไพล์ไว้จริง)

### ข้อควรระวัง

- `FUNCTION CURRENT-DATE` ขึ้นอยู่กับนาฬิกาของเครื่องที่รันโปรแกรม หากเครื่อง Server ตั้งเวลาผิด
  หรือ Time Zone ไม่ตรงกับที่คาดหวัง ค่าที่ได้จะผิดไปด้วย ในระบบที่ต้องการความแม่นยำสูง (เช่น
  Timestamp ของธุรกรรมทางการเงิน) ควรตรวจสอบการตั้งค่า Time Zone ของเครื่อง Production ให้ถูกต้อง
  เสมอ
- อย่าสับสนระหว่าง `CURRENT-DATE` (เวลารัน) กับ `WHEN-COMPILED` (เวลาคอมไพล์) — ใช้ผิด
  วัตถุประสงค์จะทำให้ Log หรือ Timestamp ที่บันทึกไว้สื่อความหมายผิด

### แบบฝึกหัดที่ 355.1

**โจทย์**: จงเขียนโค้ดที่ดึงเฉพาะ "ปี" (4 หลักแรก) จาก `FUNCTION CURRENT-DATE` ออกมาเก็บใน
ตัวแปรแยกต่างหาก

**เฉลย**:

```cobol
       01  WS-CURRENT-YEAR     PIC X(4).
       ...
           MOVE FUNCTION CURRENT-DATE(1:4) TO WS-CURRENT-YEAR.
           DISPLAY "CURRENT YEAR: " WS-CURRENT-YEAR.
```

---

## ขั้นตอนที่ 356: FUNCTION INTEGER-OF-DATE — แปลงวันที่เป็นตัวเลขวัน

### ปัญหาของการคำนวณวันที่แบบดั้งเดิม

การหาผลต่างระหว่างวันที่สองวัน (เช่น "วันนี้ห่างจากวันเกิดกี่วัน") เป็นเรื่องยุ่งยากมากถ้าคำนวณด้วยมือ
เพราะต้องคำนึงถึงจำนวนวันในแต่ละเดือนที่ไม่เท่ากัน และปีอธิกสุรทิน (Leap Year) `FUNCTION
INTEGER-OF-DATE` แก้ปัญหานี้โดยแปลงวันที่ในรูปแบบ `YYYYMMDD` ให้เป็น **ตัวเลขจำนวนเต็มตัวเดียว**
ที่นับวันต่อเนื่องจากจุดอ้างอิงคงที่จุดหนึ่ง ทำให้การหาผลต่างวันที่กลายเป็นแค่การลบเลขธรรมดา

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP356-INTEGER-OF-DATE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE-INT         PIC 9(7).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> FUNCTION INTEGER-OF-DATE turns a YYYYMMDD date into a
      *> single integer "day number", counting days since a fixed
      *> reference point. This makes date subtraction trivial.
           COMPUTE WS-DATE-INT = FUNCTION INTEGER-OF-DATE(19601231).
           DISPLAY "DAY NUMBER OF 1960-12-31: " WS-DATE-INT.

           COMPUTE WS-DATE-INT = FUNCTION INTEGER-OF-DATE(19610101).
           DISPLAY "DAY NUMBER OF 1961-01-01: " WS-DATE-INT.

           COMPUTE WS-DATE-INT = FUNCTION INTEGER-OF-DATE(20260926).
           DISPLAY "DAY NUMBER OF 2026-09-26: " WS-DATE-INT.
           STOP RUN.
```

**ผลลัพธ์:**

```
DAY NUMBER OF 1960-12-31: 0131487
DAY NUMBER OF 1961-01-01: 0131488
DAY NUMBER OF 2026-09-26: 0155497
```

### อธิบายจุดสำคัญ

- วันที่ `1960-12-31` และ `1961-01-01` เป็นวันที่ติดกัน (วันถัดไป) และตัวเลขวันที่ได้จาก
  `INTEGER-OF-DATE` ก็ต่างกันพอดี `1` หน่วย (`131488 - 131487 = 1`) นี่คือหัวใจสำคัญของฟังก์ชัน
  นี้: **วันที่ติดกันเสมอ จะได้ตัวเลขต่างกันพอดี 1 เสมอ ไม่ว่าจะข้ามเดือนหรือข้ามปีก็ตาม**
- จุดอ้างอิง (Day 0) ของ GnuCOBOL คือวันที่ไกลย้อนไปในอดีตมาก (ในทางปฏิบัติเราไม่จำเป็นต้องรู้ว่า
  Day 0 ตรงกับวันที่เท่าไรจริง ๆ เพราะเราสนใจแค่ "ผลต่าง" ระหว่างตัวเลขสองตัวเท่านั้น)
- ด้วยหลักการนี้ การหาว่าสองวันที่ห่างกันกี่วันจึงเหลือเพียงแค่: แปลงทั้งสองวันที่เป็นตัวเลขด้วย
  `INTEGER-OF-DATE` แล้วลบกันตรง ๆ (จะสาธิตแบบเต็มรูปแบบใน ขั้นตอนที่ 358)

### ข้อควรระวัง

- อาร์กิวเมนต์ของ `FUNCTION INTEGER-OF-DATE` ต้องอยู่ในรูปแบบ `YYYYMMDD` (ตัวเลข 8 หลัก) เท่านั้น
  หากส่งวันที่ผิดรูปแบบ (เช่น `DDMMYYYY`) ผลลัพธ์จะผิดพลาดโดยไม่มี Error เตือน เพราะฟังก์ชันไม่รู้ว่า
  เราตั้งใจส่งรูปแบบไหนมา มันตีความตามตำแหน่งเสมอ
- ควรตรวจสอบว่าค่าเดือน (`MM`) อยู่ในช่วง 01-12 และวัน (`DD`) สมเหตุสมผลกับเดือนนั้นก่อนส่งเข้า
  ฟังก์ชัน มิฉะนั้นอาจได้ผลลัพธ์ที่ดูสมเหตุสมผล (ไม่ Error) แต่ผิดจากความเป็นจริงโดยสิ้นเชิง

### แบบฝึกหัดที่ 356.1

**โจทย์**: จงพิสูจน์ว่าปี 2024 เป็นปีอธิกสุรทิน (มี 366 วัน) โดยใช้ `FUNCTION INTEGER-OF-DATE`
เปรียบเทียบวันที่ 2024-01-01 กับ 2025-01-01

**เฉลย**:

```cobol
       01  WS-START            PIC 9(7).
       01  WS-END              PIC 9(7).
       01  WS-DAYS-IN-YEAR     PIC 9(3).
       ...
           COMPUTE WS-START = FUNCTION INTEGER-OF-DATE(20240101).
           COMPUTE WS-END   = FUNCTION INTEGER-OF-DATE(20250101).
           COMPUTE WS-DAYS-IN-YEAR = WS-END - WS-START.
           DISPLAY "DAYS IN 2024: " WS-DAYS-IN-YEAR.
```

ผลลัพธ์ที่คาดไว้: `DAYS IN 2024: 366` เพราะ 2024 หารด้วย 4 ลงตัวและไม่ใช่ปีร้อยที่ไม่หารด้วย 400
ลงตัว จึงเป็นปีอธิกสุรทิน

---

## ขั้นตอนที่ 357: FUNCTION DATE-OF-INTEGER — แปลงตัวเลขวันกลับเป็นวันที่

### ฟังก์ชันย้อนกลับของ INTEGER-OF-DATE

`FUNCTION DATE-OF-INTEGER` ทำงานตรงข้ามกับ `FUNCTION INTEGER-OF-DATE` จาก ขั้นตอนที่ 356:
รับตัวเลขวัน (Day Number) แล้วแปลงกลับเป็นวันที่ในรูปแบบ `YYYYMMDD` ทั้งสองฟังก์ชันนี้ใช้คู่กันเสมอ
สำหรับงานเลขคณิตวันที่

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP357-DATE-OF-INTEGER.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE-INT         PIC 9(7) VALUE 155497.
       01  WS-DATE-OUT         PIC 9(8).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> FUNCTION DATE-OF-INTEGER is the reverse of INTEGER-OF-DATE:
      *> day number back to a YYYYMMDD date.
           MOVE FUNCTION DATE-OF-INTEGER(WS-DATE-INT) TO WS-DATE-OUT.
           DISPLAY "DAY NUMBER 155497 = " WS-DATE-OUT.

           MOVE FUNCTION DATE-OF-INTEGER(WS-DATE-INT + 1)
               TO WS-DATE-OUT.
           DISPLAY "DAY NUMBER 155498 = " WS-DATE-OUT.

      *> Round-trip check: date -> integer -> date gives back the
      *> exact original date.
           COMPUTE WS-DATE-INT =
               FUNCTION INTEGER-OF-DATE(20260101).
           MOVE FUNCTION DATE-OF-INTEGER(WS-DATE-INT) TO WS-DATE-OUT.
           DISPLAY "ROUND-TRIP OF 20260101 = " WS-DATE-OUT.
           STOP RUN.
```

**ผลลัพธ์:**

```
DAY NUMBER 155497 = 20260926
DAY NUMBER 155498 = 20260927
ROUND-TRIP OF 20260101 = 20260101
```

### อธิบายจุดสำคัญ

- `FUNCTION DATE-OF-INTEGER(WS-DATE-INT)` แปลงตัวเลข `155497` กลับเป็นวันที่ `20260926` ตรงกับ
  ค่าที่เราคำนวณได้ใน ขั้นตอนที่ 356 พอดี — ยืนยันว่าทั้งสองฟังก์ชันเป็นคู่กลับกันของกันและกันจริง
- การทดสอบ "Round-Trip" (แปลงวันที่เป็นตัวเลขแล้วแปลงกลับเป็นวันที่) เป็นเทคนิคที่ดีมากในการยืนยัน
  ว่าตรรกะการคำนวณวันที่ของเราถูกต้อง ก่อนนำไปใช้ในโค้ดจริงที่ซับซ้อนกว่านี้
- สามารถบวกเลขเข้ากับผลลัพธ์ของ `INTEGER-OF-DATE` ได้โดยตรงก่อนแปลงกลับ (`WS-DATE-INT + 1`)
  ซึ่งเป็นวิธีที่ใช้ "บวกวัน" เข้ากับวันที่ได้อย่างถูกต้องแม่นยำ 100% (จะสาธิตแบบเต็มใน ขั้นตอนที่ 358)

### ข้อควรระวัง

- อาร์กิวเมนต์ของ `FUNCTION DATE-OF-INTEGER` ต้องเป็นตัวเลขที่ได้มาจาก `FUNCTION
  INTEGER-OF-DATE` เท่านั้น (หรือค่าที่บวก/ลบจากมัน) ห้ามส่งตัวเลขวันที่แบบ `YYYYMMDD` เข้าไปตรง ๆ
  โดยเข้าใจผิดว่าทั้งสองฟังก์ชันรับอาร์กิวเมนต์แบบเดียวกัน เพราะจะได้ผลลัพธ์ที่ผิดพลาดอย่างมาก
  (ตัวเลข `YYYYMMDD` มีค่าหลายล้าน ในขณะที่ Day Number ของ `INTEGER-OF-DATE` มีค่าราวหลักแสน
  เท่านั้นสำหรับวันที่ในยุคปัจจุบัน)

### แบบฝึกหัดที่ 357.1

**โจทย์**: จงเขียนโค้ดหาวันที่ **ก่อนหน้า** วันที่ `20260301` ไป 1 วัน (ควรได้ `20260228` เพราะ
2026 ไม่ใช่ปีอธิกสุรทิน)

**เฉลย**:

```cobol
       01  WS-BASE-INT         PIC 9(7).
       01  WS-RESULT-DATE      PIC 9(8).
       ...
           COMPUTE WS-BASE-INT = FUNCTION INTEGER-OF-DATE(20260301).
           MOVE FUNCTION DATE-OF-INTEGER(WS-BASE-INT - 1)
               TO WS-RESULT-DATE.
           DISPLAY "ONE DAY BEFORE 2026-03-01: " WS-RESULT-DATE.
```

---

## ขั้นตอนที่ 358: เลขคณิตวันที่ (Date Arithmetic) แบบเต็มรูปแบบ

### รวมสองฟังก์ชันเข้าด้วยกันเพื่อแก้ปัญหาจริง

ขั้นตอนนี้จะนำ `FUNCTION INTEGER-OF-DATE` และ `FUNCTION DATE-OF-INTEGER` มาใช้แก้ปัญหาทาง
ธุรกิจจริงสองแบบที่พบบ่อยที่สุด: **(1) หาอายุ/ระยะเวลาเป็นวัน** ระหว่างสองวันที่ และ **(2) บวกจำนวน
วันเข้ากับวันที่** เพื่อหาวันครบกำหนด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP358-DATE-ARITHMETIC.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-BIRTH-DT         PIC 9(8) VALUE 19900615.
       01  WS-TODAY-DT         PIC 9(8) VALUE 20260926.
       01  WS-BIRTH-INT        PIC 9(7).
       01  WS-TODAY-INT        PIC 9(7).
       01  WS-AGE-DAYS         PIC 9(6).
       01  WS-AGE-YEARS        PIC 9(3).
       01  WS-DUE-DT           PIC 9(8) VALUE 20260901.
       01  WS-DUE-INT          PIC 9(7).
       01  WS-NEW-DUE-INT      PIC 9(7).
       01  WS-NEW-DUE-DT       PIC 9(8).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> How many days old is someone born on 1990-06-15?
           COMPUTE WS-BIRTH-INT =
               FUNCTION INTEGER-OF-DATE(WS-BIRTH-DT).
           COMPUTE WS-TODAY-INT =
               FUNCTION INTEGER-OF-DATE(WS-TODAY-DT).
           COMPUTE WS-AGE-DAYS = WS-TODAY-INT - WS-BIRTH-INT.
           COMPUTE WS-AGE-YEARS = WS-AGE-DAYS / 365.
           DISPLAY "AGE IN DAYS: " WS-AGE-DAYS.
           DISPLAY "AGE IN YEARS (APPROX): " WS-AGE-YEARS.

      *> Add 30 days to an invoice due date - convert to integer,
      *> add, then convert back to a real calendar date.
           COMPUTE WS-DUE-INT = FUNCTION INTEGER-OF-DATE(WS-DUE-DT).
           COMPUTE WS-NEW-DUE-INT = WS-DUE-INT + 30.
           MOVE FUNCTION DATE-OF-INTEGER(WS-NEW-DUE-INT)
               TO WS-NEW-DUE-DT.
           DISPLAY "DUE DATE + 30 DAYS: " WS-NEW-DUE-DT.
           STOP RUN.
```

**ผลลัพธ์:**

```
AGE IN DAYS: 013252
AGE IN YEARS (APPROX): 036
DUE DATE + 30 DAYS: 20261001
```

### อธิบายจุดสำคัญ

- ขั้นตอนการหาอายุเป็นวัน: (1) แปลงวันเกิดและวันนี้เป็น Day Number ด้วย `INTEGER-OF-DATE`
  (2) ลบกันตรง ๆ ได้จำนวนวันที่ผ่านมา (3) หารด้วย 365 (โดยประมาณ) เพื่อประมาณเป็นปี — วิธีนี้
  แม่นยำกว่าและง่ายกว่าการพยายามคำนวณปี-เดือน-วันด้วยตรรกะ `IF` ซับซ้อนมาก
- ขั้นตอนการบวกวัน: (1) แปลงวันครบกำหนดเดิมเป็น Day Number (2) บวกจำนวนวันที่ต้องการ (30 วัน)
  เข้ากับ Day Number โดยตรง (เป็นแค่เลขคณิตธรรมดา) (3) แปลง Day Number ใหม่กลับเป็นวันที่ด้วย
  `DATE-OF-INTEGER` — สังเกตว่าผลลัพธ์ `20260901 + 30 วัน = 20261001` ข้ามเดือนกันยายน (30 วัน)
  ไปเป็นเดือนตุลาคมได้อย่างถูกต้องโดยอัตโนมัติ โดยที่เราไม่ต้องเขียนตรรกะตรวจสอบจำนวนวันในแต่ละ
  เดือนเองเลย
- นี่คือประโยชน์สูงสุดของคู่ฟังก์ชันนี้: **การนับวันข้ามเดือน ข้ามปี หรือข้ามปีอธิกสุรทิน ถูกต้องเสมอ
  โดยอัตโนมัติ** เพราะฟังก์ชันจัดการความซับซ้อนของปฏิทินให้เราทั้งหมด

### ข้อควรระวัง

- การหารด้วย `365` เพื่อประมาณจำนวนปีเป็นเพียงค่าประมาณเท่านั้น (ไม่ได้คำนึงถึงปีอธิกสุรทิน) หาก
  ต้องการความแม่นยำระดับปี-เดือน-วันแบบสมบูรณ์ (เช่น ระบบคำนวณอายุสำหรับใบสมัครงานราชการ) ควร
  ใช้เทคนิคที่ซับซ้อนกว่านี้ หรือใช้ Library เฉพาะทาง แต่สำหรับงานทั่วไปที่ต้องการ "อายุโดยประมาณ"
  วิธีนี้เพียงพอและรวดเร็ว
- ต้องระวังเรื่องคอลัมน์ 72 เป็นพิเศษในโค้ดประเภทนี้ เพราะชื่อตัวแปรที่สื่อความหมาย (เช่น
  `WS-NEW-DUE-INT`) รวมกับชื่อฟังก์ชันที่ยาว (`FUNCTION INTEGER-OF-DATE`) ทำให้บรรทัดยาวเกิน
  ขีดจำกัดได้ง่ายมาก ดังที่เห็นในตัวอย่างนี้ที่ต้องตัดบรรทัด `COMPUTE`/`MOVE` หลายจุด

### แบบฝึกหัดที่ 358.1

**โจทย์**: จงเขียนโค้ดหาว่า "อีก 90 วันข้างหน้าจากวันนี้ (สมมติวันนี้คือ 2026-09-26) จะตรงกับวันที่
เท่าไร"

**เฉลย**:

```cobol
       01  WS-TODAY-DT2        PIC 9(8) VALUE 20260926.
       01  WS-TODAY-INT2       PIC 9(7).
       01  WS-FUTURE-INT       PIC 9(7).
       01  WS-FUTURE-DT        PIC 9(8).
       ...
           COMPUTE WS-TODAY-INT2 =
               FUNCTION INTEGER-OF-DATE(WS-TODAY-DT2).
           COMPUTE WS-FUTURE-INT = WS-TODAY-INT2 + 90.
           MOVE FUNCTION DATE-OF-INTEGER(WS-FUTURE-INT)
               TO WS-FUTURE-DT.
           DISPLAY "90 DAYS FROM TODAY: " WS-FUTURE-DT.
```

ผลลัพธ์ที่คาดไว้: `90 DAYS FROM TODAY: 20261225`

---

## ขั้นตอนที่ 359: การซ้อนฟังก์ชัน (Nested Functions)

### ผลลัพธ์ของฟังก์ชันหนึ่ง เป็นอาร์กิวเมนต์ของอีกฟังก์ชันหนึ่งได้

Intrinsic Functions สามารถ **ซ้อนกัน** ได้ — ผลลัพธ์ของฟังก์ชันชั้นในสุดจะถูกคำนวณก่อน แล้วส่งต่อ
เป็นอาร์กิวเมนต์ให้ฟังก์ชันชั้นถัดไปตามลำดับ เหมือนกับการซ้อนฟังก์ชันทางคณิตศาสตร์ทั่วไป

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP359-NESTED-FUNCTIONS.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RESULT           PIC S9(6)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Functions can be nested inside one another: the innermost
      *> one is evaluated first, and its result feeds the next.
           COMPUTE WS-RESULT =
               FUNCTION INTEGER(FUNCTION SQRT(FUNCTION MAX(50 200 90))).
           DISPLAY "INTEGER(SQRT(MAX(50,200,90))) = " WS-RESULT.

           COMPUTE WS-RESULT =
               FUNCTION ABS(FUNCTION MOD(-17 5) - 10).
           DISPLAY "ABS(MOD(-17,5) - 10) = " WS-RESULT.
           STOP RUN.
```

**ผลลัพธ์:**

```
INTEGER(SQRT(MAX(50,200,90))) = +000014.00
ABS(MOD(-17,5) - 10) = +000007.00
```

### อธิบายจุดสำคัญ

- `FUNCTION INTEGER(FUNCTION SQRT(FUNCTION MAX(50 200 90)))` คำนวณจากในสุดออกมา: (1)
  `MAX(50, 200, 90) = 200` (2) `SQRT(200) ≈ 14.142` (3) `INTEGER(14.142) = 14` — ทั้งหมดนี้
  เกิดขึ้นในนิพจน์เดียว ไม่ต้องประกาศตัวแปรชั่วคราวมาเก็บผลลัพธ์ระหว่างทาง
- `FUNCTION ABS(FUNCTION MOD(-17 5) - 10)`: (1) `MOD(-17, 5) = 3` (ตามกฎจาก ขั้นตอนที่ 351)
  (2) `3 - 10 = -7` (3) `ABS(-7) = 7`
- การซ้อนฟังก์ชันช่วยลดจำนวนตัวแปรชั่วคราวที่ต้องประกาศ ทำให้โค้ดกระชับขึ้น แต่ก็ต้องแลกกับความ
  ยากในการอ่านหากซ้อนลึกเกินไป

### ข้อควรระวัง

- ยิ่งซ้อนฟังก์ชันลึกเท่าไร โอกาสที่บรรทัดจะยาวเกินคอลัมน์ 72 ก็ยิ่งสูงขึ้นเท่านั้น (ดังตัวอย่างที่ต้อง
  ตัดบรรทัดในโค้ดข้างบน) ควรจำกัดความลึกของการซ้อนไม่ให้เกิน 2-3 ชั้น เพื่อรักษาความอ่านง่าย
  หากซับซ้อนกว่านั้น ควรแบ่งเป็นหลายบรรทัดโดยใช้ตัวแปรชั่วคราวเก็บผลลัพธ์ระหว่างทางแทน
- การซ้อนฟังก์ชันมากเกินไปทำให้การ Debug ยากขึ้นมาก เพราะไม่สามารถ `DISPLAY` ค่ากลางระหว่างการ
  คำนวณแต่ละชั้นได้โดยตรง หากโค้ดมีปัญหา อาจต้องแยกนิพจน์ออกเป็นหลายบรรทัดชั่วคราวเพื่อ Debug
  ก่อนนำกลับมารวมเป็นบรรทัดเดียวอีกครั้ง

### แบบฝึกหัดที่ 359.1

**โจทย์**: จงเขียนนิพจน์ที่ซ้อนกันเพื่อหาค่ารากที่สองของผลต่างสัมบูรณ์ระหว่าง 100 กับ 36 (คำตอบ
ควรได้ `SQRT(ABS(36-100)) = SQRT(64) = 8`)

**เฉลย**:

```cobol
       01  WS-ANSWER           PIC S9(4)V99.
       ...
           COMPUTE WS-ANSWER = FUNCTION SQRT(FUNCTION ABS(36 - 100)).
           DISPLAY "SQRT(ABS(36-100)) = " WS-ANSWER.
```

ผลลัพธ์ที่คาดไว้: `SQRT(ABS(36-100)) = +0008.00`

---

## ขั้นตอนที่ 360: สรุปรวม — โปรแกรมคำนวณใบแจ้งหนี้ (Invoice Calculator)

### รวมทุกฟังก์ชันของ Part นี้เข้าด้วยกันในโปรแกรมเดียว

ขั้นตอนสุดท้ายของ Part นี้จะนำ `ROUNDED`, `FUNCTION MAX`, และคู่ฟังก์ชันวันที่ (`INTEGER-OF-DATE`
/ `DATE-OF-INTEGER`) มารวมกันเป็นโปรแกรมคำนวณใบแจ้งหนี้ (Invoice) ขนาดเล็กที่ใกล้เคียงกับงาน
จริงในระบบธุรกิจ: คำนวณราคารวม, ภาษี, ยอดสุทธิ, และวันครบกำหนดชำระเงิน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP360-INVOICE-CALC.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-UNIT-PRICE       PIC 9(5)V99 VALUE 149.99.
       01  WS-QTY              PIC 9(3)    VALUE 7.
       01  WS-SUBTOTAL         PIC 9(7)V99.
       01  WS-TAX-RATE         PIC 9V99    VALUE 0.07.
       01  WS-TAX-AMT          PIC 9(7)V99.
       01  WS-GRAND-TOTAL      PIC 9(7)V99.
       01  WS-INVOICE-DT       PIC 9(8)    VALUE 20260926.
       01  WS-INVOICE-INT      PIC 9(7).
       01  WS-DUE-INT          PIC 9(7).
       01  WS-DUE-DT           PIC 9(8).
       01  WS-DAYS-TO-PAY      PIC 9(3)    VALUE 30.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "==== INVOICE CALCULATOR (STEP 360) ====".

           COMPUTE WS-SUBTOTAL ROUNDED = WS-UNIT-PRICE * WS-QTY.
           DISPLAY "SUBTOTAL: " WS-SUBTOTAL.

           COMPUTE WS-TAX-AMT ROUNDED = WS-SUBTOTAL * WS-TAX-RATE.
           DISPLAY "TAX (7%): " WS-TAX-AMT.

           COMPUTE WS-GRAND-TOTAL = WS-SUBTOTAL + WS-TAX-AMT.
           DISPLAY "GRAND TOTAL: " WS-GRAND-TOTAL.

           COMPUTE WS-INVOICE-INT =
               FUNCTION INTEGER-OF-DATE(WS-INVOICE-DT).
           COMPUTE WS-DUE-INT = WS-INVOICE-INT + WS-DAYS-TO-PAY.
           MOVE FUNCTION DATE-OF-INTEGER(WS-DUE-INT) TO WS-DUE-DT.
           DISPLAY "INVOICE DATE: " WS-INVOICE-DT.
           DISPLAY "DUE DATE (+30 DAYS): " WS-DUE-DT.

           DISPLAY "HIGHEST OF SUBTOTAL/TAX/TOTAL: "
               FUNCTION MAX(WS-SUBTOTAL WS-TAX-AMT WS-GRAND-TOTAL).
           STOP RUN.
```

**ผลลัพธ์:**

```
==== INVOICE CALCULATOR (STEP 360) ====
SUBTOTAL: 0001049.93
TAX (7%): 0000073.50
GRAND TOTAL: 0001123.43
INVOICE DATE: 20260926
DUE DATE (+30 DAYS): 20261026
HIGHEST OF SUBTOTAL/TAX/TOTAL: 0001123.43
```

### อธิบายจุดสำคัญ

- `COMPUTE WS-SUBTOTAL ROUNDED = WS-UNIT-PRICE * WS-QTY.` — `149.99 × 7 = 1049.93` พอดี
  (ไม่มีเศษให้ปัด) แสดงให้เห็นว่าการใส่ `ROUNDED` ไว้เสมอในงานคำนวณเงินเป็นนิสัยที่ดี แม้ในกรณีนี้
  จะไม่มีผลต่างจากการไม่ใส่ก็ตาม เพราะเราไม่มีทางรู้ล่วงหน้าว่าตัวเลขไหนจะลงตัวพอดีหรือไม่
- `COMPUTE WS-TAX-AMT ROUNDED = WS-SUBTOTAL * WS-TAX-RATE.` — `1049.93 × 0.07 = 73.4951`
  ปัดเป็น `73.50` ด้วย `ROUNDED` (หากไม่ใส่ `ROUNDED` จะได้ `73.49` ซึ่งขาดไป 1 สตางค์ — ตัวอย่าง
  ที่ดีของผลกระทบจากการลืมใส่ `ROUNDED` ในงานการเงิน)
- การคำนวณวันครบกำหนดใช้เทคนิคเดียวกับ ขั้นตอนที่ 358 ทุกประการ: แปลงเป็น Day Number, บวกจำนวน
  วัน, แปลงกลับเป็นวันที่
- `FUNCTION MAX` ในบรรทัดสุดท้ายแสดงให้เห็นว่าแม้แต่ยอดเงินที่คำนวณมาแล้ว ก็ยังนำไปใช้กับ
  `Intrinsic Function` อื่นต่อได้อย่างอิสระ (ในที่นี้คือ `GRAND-TOTAL` ย่อมมากที่สุดเสมอเพราะเป็นผลรวม
  ของอีกสองค่า แต่การสาธิตนี้แสดงหลักการที่นำไปประยุกต์กับกรณีอื่นได้)

### ข้อควรระวังสรุปรวมทั้ง Part

- `FUNCTION SQRT` ของค่าติดลบ และ `FUNCTION NUMVAL`/`NUMVAL-C` ของข้อความที่แปลงไม่ได้ (จะ
  เรียนใน Part 037) มักคืนค่า `0` แบบเงียบ ๆ โดยไม่แจ้ง Error — ต้องตรวจสอบข้อมูลนำเข้าเองก่อน
  เรียกใช้ฟังก์ชันเหล่านี้เสมอหากมีความเสี่ยงที่ข้อมูลจะไม่ถูกต้อง
- งานคำนวณเงินควรใช้ `ROUNDED` เสมอ ไม่ควรปล่อยให้ COBOL ตัดทอนทศนิยมทิ้งไปเงียบ ๆ
- ชื่อฟังก์ชันวันที่ (`INTEGER-OF-DATE`, `DATE-OF-INTEGER`) ยาวมาก ต้องระวังคอลัมน์ 72 เป็นพิเศษ
  เมื่อรวมกับชื่อตัวแปรที่มีความหมาย
- ฟังก์ชันซ้อนกันได้ (Nested) แต่ควรจำกัดความลึกไม่เกิน 2-3 ชั้นเพื่อรักษาความอ่านง่ายและ Debug ได้

### แบบฝึกหัดที่ 360.1

**โจทย์**: จงขยายโปรแกรม `STEP360-INVOICE-CALC` ให้เพิ่มส่วนลด (Discount) 10% ก่อนคำนวณภาษี
โดยส่วนลดคำนวณจาก `WS-SUBTOTAL` และใช้ `ROUNDED` ในทุกการคำนวณที่เกี่ยวกับเงิน

**เฉลย**:

```cobol
       01  WS-DISCOUNT-RATE    PIC 9V99 VALUE 0.10.
       01  WS-DISCOUNT-AMT     PIC 9(7)V99.
       01  WS-NET-SUBTOTAL     PIC 9(7)V99.
       ...
           COMPUTE WS-DISCOUNT-AMT ROUNDED =
               WS-SUBTOTAL * WS-DISCOUNT-RATE.
           COMPUTE WS-NET-SUBTOTAL =
               WS-SUBTOTAL - WS-DISCOUNT-AMT.
           DISPLAY "DISCOUNT (10%): " WS-DISCOUNT-AMT.
           DISPLAY "NET SUBTOTAL: " WS-NET-SUBTOTAL.
      *> Then compute tax on WS-NET-SUBTOTAL instead of WS-SUBTOTAL:
           COMPUTE WS-TAX-AMT ROUNDED =
               WS-NET-SUBTOTAL * WS-TAX-RATE.
```

ผลลัพธ์ที่คาดไว้ (โดยประมาณ): ส่วนลด `104.99`, ยอดหลังหักส่วนลด `944.94`, ภาษีคำนวณจากยอดใหม่นี้
แทนที่จะเป็นยอดเต็มก่อนหักส่วนลด

---

## สรุปท้ายบท

Part นี้พาเราทำความรู้จักกับ Intrinsic Functions กลุ่มตัวเลขและวันที่อย่างละเอียด ได้แก่:

- ไวยากรณ์พื้นฐานของ `FUNCTION` และความแตกต่างระหว่าง `FUNCTION MOD` กับ `FUNCTION REM`
- `FUNCTION SQRT`, `FUNCTION ABS`, `FUNCTION FACTORIAL` และข้อควรระวังเรื่อง SQRT ของค่าติดลบ
- `FUNCTION MAX` และ `FUNCTION MIN` ที่ลดความยุ่งยากของการเปรียบเทียบค่าหลายตัว
- ความแตกต่างระหว่างการตัดทอนกับการปัดเศษ (`ROUNDED`), และความต่างระหว่าง `FUNCTION INTEGER`
  กับ `FUNCTION INTEGER-PART` เมื่อทำงานกับค่าติดลบ
- `FUNCTION CURRENT-DATE` และ `FUNCTION WHEN-COMPILED` สำหรับอ่านวันเวลาปัจจุบันและเวลาคอมไพล์
- `FUNCTION INTEGER-OF-DATE` และ `FUNCTION DATE-OF-INTEGER` ที่เป็นหัวใจของการทำเลขคณิตวันที่
  อย่างแม่นยำ โดยไม่ต้องกังวลเรื่องจำนวนวันในแต่ละเดือนหรือปีอธิกสุรทิน
- การซ้อนฟังก์ชัน (Nested Functions) และข้อจำกัดด้านความอ่านง่าย
- โปรแกรมคำนวณใบแจ้งหนี้ที่รวมทุกเทคนิคของ Part นี้เข้าด้วยกันเป็นตัวอย่างใกล้เคียงงานจริง

Intrinsic Functions ช่วยให้โค้ด COBOL กระชับและแม่นยำขึ้นอย่างมาก ลดความจำเป็นในการเขียนตรรกะ
คำนวณที่ซับซ้อนด้วยตัวเอง ใน **Part 037** เราจะเรียนรู้ Intrinsic Functions อีกกลุ่มหนึ่งที่สำคัญไม่แพ้
กัน: ฟังก์ชันด้าน**ข้อความ** (UPPER-CASE, LOWER-CASE, TRIM, LENGTH, NUMVAL) และ **สถิติ**
(SUM, MEAN, MEDIAN, VARIANCE, STANDARD-DEVIATION) พร้อมทดสอบจริงว่า GnuCOBOL รองรับ
ฟังก์ชันใดบ้างในทางปฏิบัติ

**[กลับไปยัง Part 035: โปรเจกต์เฟส 2 - ระบบจัดการสินค้าคงคลัง →](part-035-phase2-project.md)**

**[ไปยัง Part 037: Intrinsic Functions ข้อความและสถิติ →](part-037-intrinsic-functions-string-stats.md)**
