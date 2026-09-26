# Part 042: Class Condition และ Sign Condition (ขั้นตอนที่ 411–420)

## คำนำของ Part นี้

ใน Part 010 เราแนะนำ **Class Condition** (`IS NUMERIC`, `IS ALPHABETIC`) และ **Sign Condition**
(`IS POSITIVE`, `IS NEGATIVE`, `IS ZERO`) เพียงผิวเผินในฐานะเงื่อนไขเสริมของ `IF` เท่านั้น
Part นี้จะพาทั้งสองเรื่องกลับมาเจาะลึกอย่างเต็มรูปแบบ เพราะทั้งคู่เป็นเครื่องมือสำคัญที่สุดสำหรับ
**การตรวจสอบความถูกต้องของข้อมูล (data validation)** ก่อนนำไปประมวลผล ซึ่งเป็นงานที่พบในโปรแกรม
ธุรกิจแทบทุกโปรแกรม

สิ่งที่ทำให้ Part นี้พิเศษคือ ระหว่างการทดสอบคอมไพล์และรันจริงทุกตัวอย่างด้วย GnuCOBOL เราค้นพบ
**พฤติกรรมจริงที่คาดไม่ถึง** ของ `IS NUMERIC` กับข้อมูลที่มีช่องว่างนำหน้า/ตามหลัง ซึ่งเป็นกับดักที่พบ
บ่อยมากในโค้ดจริง โดยเฉพาะเมื่อรับข้อมูลจาก `ACCEPT` — Part นี้จะแสดงหลักฐานการทดสอบจริงและวิธี
แก้ไขที่ยืนยันแล้วว่าใช้งานได้ พร้อมปิดท้ายด้วยโปรแกรมตรวจสอบข้อมูล (validation module) ที่ผสาน
Class Condition และ Sign Condition เข้าด้วยกันอย่างครบถ้วน

---

## ขั้นตอนที่ 411: IS NUMERIC แบบเจาะลึก — ทดสอบกับข้อมูลหลากหลายรูปแบบ

### แนวคิด

`IS NUMERIC` ตรวจสอบว่าเนื้อหาของฟิลด์เป็น**ตัวเลขที่ถูกต้องตามหลักไวยากรณ์**หรือไม่ ใช้ได้ทั้งกับ
ฟิลด์ตัวเลข (`PIC 9`, `PIC S9`) และฟิลด์ตัวอักษร (`PIC X`) ที่บังเอิญเก็บตัวเลขไว้ ขั้นตอนนี้จะทดสอบ
กับข้อมูล 5 รูปแบบที่แตกต่างกันเพื่อดูขอบเขตที่แท้จริงของมัน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IS-NUMERIC-DEEP-DIVE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIELD-1   PIC X(5) VALUE "12345".
       01  WS-FIELD-2   PIC X(5) VALUE "12A45".
       01  WS-FIELD-3   PIC X(5) VALUE "  123".
       01  WS-FIELD-4   PIC S9(5) VALUE -123.
       01  WS-FIELD-5   PIC X(5) VALUE "-1234".

       PROCEDURE DIVISION.
           IF WS-FIELD-1 IS NUMERIC
               DISPLAY "FIELD-1 (12345)      : NUMERIC"
           ELSE
               DISPLAY "FIELD-1 (12345)      : NOT NUMERIC"
           END-IF.

           IF WS-FIELD-2 IS NUMERIC
               DISPLAY "FIELD-2 (12A45)      : NUMERIC"
           ELSE
               DISPLAY "FIELD-2 (12A45)      : NOT NUMERIC"
           END-IF.

           IF WS-FIELD-3 IS NUMERIC
               DISPLAY "FIELD-3 (SP SP 123)  : NUMERIC"
           ELSE
               DISPLAY "FIELD-3 (SP SP 123)  : NOT NUMERIC"
           END-IF.

           IF WS-FIELD-4 IS NUMERIC
               DISPLAY "FIELD-4 (S9 -123)    : NUMERIC"
           ELSE
               DISPLAY "FIELD-4 (S9 -123)    : NOT NUMERIC"
           END-IF.

           IF WS-FIELD-5 IS NUMERIC
               DISPLAY "FIELD-5 (X '-1234')  : NUMERIC"
           ELSE
               DISPLAY "FIELD-5 (X '-1234')  : NOT NUMERIC"
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-FIELD-1` ("12345") — ตัวเลขล้วน 5 หลัก คาดว่าเป็น NUMERIC
- `WS-FIELD-2` ("12A45") — มีตัวอักษร "A" ปน คาดว่าไม่ใช่ NUMERIC
- `WS-FIELD-3` ("  123", มีช่องว่างนำหน้า 2 ตัว) — ทดสอบว่า `IS NUMERIC` ยอมรับช่องว่างนำหน้า
  หรือไม่
- `WS-FIELD-4` เป็น `PIC S9(5)` (signed numeric แท้ ๆ) ค่า -123 — คาดว่าเป็น NUMERIC เพราะเป็น
  ฟิลด์ตัวเลขที่ถูกต้องตามหลักไวยากรณ์ตั้งแต่การประกาศ
- `WS-FIELD-5` เป็น `PIC X(5)` (ฟิลด์ตัวอักษร) ที่เก็บข้อความ "-1234" — ทดสอบว่าเครื่องหมายลบที่
  เป็น**ตัวอักษรจริง** (ไม่ใช่ sign nibble ของฟิลด์ signed) ทำให้ `IS NUMERIC` เป็นจริงหรือไม่

### ผลลัพธ์ที่ได้จากการรันจริง

```
FIELD-1 (12345)      : NUMERIC
FIELD-2 (12A45)      : NOT NUMERIC
FIELD-3 (SP SP 123)  : NOT NUMERIC
FIELD-4 (S9 -123)    : NUMERIC
FIELD-5 (X '-1234')  : NOT NUMERIC
```

### ข้อค้นพบสำคัญจากการทดสอบจริง

ผลลัพธ์ของ `WS-FIELD-3` (ช่องว่างนำหน้า + ตัวเลข) เป็น **NOT NUMERIC** ซึ่งอาจขัดกับสัญชาตญาณของ
หลายคน (บางคนอาจคิดว่า "ก็มีแต่ตัวเลขกับช่องว่างนี่ ทำไมไม่ NUMERIC?") แต่ผลการทดสอบจริงกับ
GnuCOBOL ยืนยันว่า **สำหรับฟิลด์ `PIC X` ตัวอักษรทุกตัวในฟิลด์ต้องเป็นเลข 0-9 ล้วนเท่านั้น (ไม่รวม
ช่องว่างเลย) จึงจะถือว่า `IS NUMERIC` เป็นจริง** ต่างจากฟิลด์ตัวเลขแท้ (`PIC 9`/`PIC S9`) ที่การ
`MOVE` ค่าเข้าไปแล้วจะจัดรูปแบบ (เติมศูนย์นำหน้า) ให้เป็นตัวเลขล้วนเสมอโดยอัตโนมัติ ไม่มีโอกาสมี
ช่องว่างปนอยู่ภายในได้เลย — ปัญหานี้จึงเกิดเฉพาะกับฟิลด์ `PIC X` ที่รับข้อมูลดิบมาโดยตรง (เช่นจาก
`ACCEPT` หรือไฟล์ภายนอก) ซึ่งจะเจาะลึกพร้อมวิธีแก้ไขในขั้นตอนที่ 415

### ข้อควรระวัง

- **`IS NUMERIC` กับฟิลด์ `PIC X` ที่มีช่องว่างปนอยู่ (แม้จะเป็นแค่ช่องว่างนำหน้าหรือตามหลัง) จะได้
  ผลลัพธ์เป็นเท็จเสมอ** นี่คือกับดักที่พบบ่อยมากเมื่อรับข้อมูลจาก `ACCEPT` (ตามที่จะพิสูจน์ในขั้นตอนที่
  414-415)
- เครื่องหมายลบที่เป็นตัวอักษรธรรมดา ("-1234" ในฟิลด์ `PIC X`) **ไม่ถือเป็นตัวเลขที่ถูกต้อง**
  ต้องเป็นฟิลด์ signed numeric แท้ (`PIC S9`) เท่านั้นจึงจะรองรับเครื่องหมายลบได้อย่างถูกต้องตามหลัก
  ไวยากรณ์ COBOL

### แบบฝึกหัดที่ 411.1

**โจทย์**: จงทำนายผลลัพธ์ของ `IF WS-CODE IS NUMERIC` เมื่อ `WS-CODE PIC X(6)` มีค่า `"001234"`
(ไม่มีช่องว่างเลย) เปรียบเทียบกับ `"1234  "` (มีช่องว่างต่อท้าย 2 ตัว)

**เฉลย**: `"001234"` จะเป็น **NUMERIC** (ตัวเลขล้วน 6 หลัก ไม่มีช่องว่างปนเลย) ในขณะที่
`"1234  "` จะเป็น **NOT NUMERIC** (มีช่องว่าง 2 ตัวต่อท้ายปนอยู่ในฟิลด์) แม้ตัวเลข "1234" เองจะ
ถูกต้องสมบูรณ์ก็ตาม

---

## ขั้นตอนที่ 412: IS ALPHABETIC, ALPHABETIC-UPPER, ALPHABETIC-LOWER

### แนวคิด

COBOL มี Class Condition สำหรับตรวจสอบตัวอักษร 3 ระดับ:

- **`IS ALPHABETIC`** — ตรวจว่าเนื้อหาเป็นตัวอักษร A-Z, a-z หรือช่องว่างเท่านั้น (**ไม่สนใจว่าเป็น
  ตัวพิมพ์เล็กหรือใหญ่ ผสมกันได้**)
- **`IS ALPHABETIC-UPPER`** — ตรวจว่าเป็นตัวอักษร**ตัวพิมพ์ใหญ่** A-Z หรือช่องว่างเท่านั้น
- **`IS ALPHABETIC-LOWER`** — ตรวจว่าเป็นตัวอักษร**ตัวพิมพ์เล็ก** a-z หรือช่องว่างเท่านั้น

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ALPHABETIC-DEEP-DIVE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-F1   PIC X(8) VALUE "HELLO   ".
       01  WS-F2   PIC X(8) VALUE "hello   ".
       01  WS-F3   PIC X(8) VALUE "Hello   ".
       01  WS-F4   PIC X(8) VALUE "HELLO123".

       PROCEDURE DIVISION.
           IF WS-F1 IS ALPHABETIC
               DISPLAY "F1 (HELLO)  : ALPHABETIC"
           END-IF.
           IF WS-F1 IS ALPHABETIC-UPPER
               DISPLAY "F1 (HELLO)  : ALPHABETIC-UPPER"
           END-IF.
           IF WS-F2 IS ALPHABETIC-LOWER
               DISPLAY "F2 (hello)  : ALPHABETIC-LOWER"
           END-IF.
           IF WS-F3 IS ALPHABETIC
               DISPLAY "F3 (Hello)  : ALPHABETIC"
           ELSE
               DISPLAY "F3 (Hello)  : NOT PURE UPPER OR LOWER"
           END-IF.
           IF WS-F3 IS ALPHABETIC-UPPER
               DISPLAY "F3 (Hello)  : ALPHABETIC-UPPER"
           ELSE
               DISPLAY "F3 (Hello)  : NOT ALPHABETIC-UPPER"
           END-IF.
           IF WS-F4 IS ALPHABETIC
               DISPLAY "F4 (HELLO123): ALPHABETIC"
           ELSE
               DISPLAY "F4 (HELLO123): NOT ALPHABETIC"
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-F1` ("HELLO   ", มีช่องว่างต่อท้ายจาก `PIC X(8)`) — เป็นทั้ง `ALPHABETIC` และ
  `ALPHABETIC-UPPER` เพราะ**ช่องว่างถือว่าถูกต้องเสมอสำหรับ Class Condition ทั้งสามแบบนี้**
  (ต่างจาก `IS NUMERIC` ในขั้นตอนที่แล้วที่ช่องว่างทำให้เป็นเท็จ!)
- `WS-F3` ("Hello   ", ตัวพิมพ์ผสม) — เป็น `ALPHABETIC` (เพราะมีแต่ตัวอักษรกับช่องว่าง ไม่สนใจ
  case) **แต่ไม่เป็น** `ALPHABETIC-UPPER` (เพราะมีตัวพิมพ์เล็กปนอยู่)
- `WS-F4` ("HELLO123") — มีตัวเลขปนอยู่ จึง**ไม่เป็น** `ALPHABETIC` เลย

### ผลลัพธ์ที่ได้จากการรันจริง

```
F1 (HELLO)  : ALPHABETIC
F1 (HELLO)  : ALPHABETIC-UPPER
F2 (hello)  : ALPHABETIC-LOWER
F3 (Hello)  : ALPHABETIC
F3 (Hello)  : NOT ALPHABETIC-UPPER
F4 (HELLO123): NOT ALPHABETIC
```

### ข้อค้นพบสำคัญ — เปรียบเทียบพฤติกรรมช่องว่างกับ IS NUMERIC

สังเกตความแตกต่างที่สำคัญมากระหว่างขั้นตอนนี้กับขั้นตอนที่ 411: **`IS ALPHABETIC` (และ
`-UPPER`/`-LOWER`) ยอมรับช่องว่างในฟิลด์เป็นค่าที่ถูกต้องเสมอ** แต่ **`IS NUMERIC` ไม่ยอมรับ
ช่องว่างเลยแม้แต่ตัวเดียว** นี่เป็นความไม่สมมาตร (asymmetry) ที่สำคัญมากที่ต้องจำให้แม่นเมื่อเขียน
โค้ดตรวจสอบข้อมูล เพราะฟิลด์ตัวอักษรที่ยังกรอกไม่เต็ม (มีช่องว่างเหลือ) จะยังคง `IS ALPHABETIC`
เป็นจริง แต่ฟิลด์ตัวเลขที่ยังกรอกไม่เต็มจะ `IS NUMERIC` เป็นเท็จทันที

### ข้อควรระวัง

- อย่าสับสนระหว่าง `IS ALPHABETIC` (ผสมตัวพิมพ์เล็ก-ใหญ่ได้) กับการที่หลายคนเข้าใจผิดว่า
  "ALPHABETIC แปลว่าต้องเป็น case เดียวกันทั้งหมด" — ต้องใช้ `ALPHABETIC-UPPER`/`-LOWER`
  โดยเฉพาะหากต้องการบังคับ case ที่แน่นอน
- Class Condition เหล่านี้ตรวจสอบ**รูปแบบ**เท่านั้น ไม่ได้ตรวจสอบว่าเป็นชื่อที่มีความหมายจริงหรือไม่
  (เช่น "XXXXX" ก็ถือว่า `IS ALPHABETIC` เป็นจริงเช่นกัน)

### แบบฝึกหัดที่ 412.1

**โจทย์**: จงเขียนเงื่อนไขตรวจสอบว่า `WS-COUNTRY-CODE PIC X(2)` เป็นรหัสประเทศที่ถูกต้องตาม
รูปแบบหรือไม่ (ต้องเป็นตัวอักษรพิมพ์ใหญ่ล้วน 2 ตัว เช่น "TH", "US")

**เฉลย**:
```cobol
IF WS-COUNTRY-CODE IS ALPHABETIC-UPPER
    DISPLAY "VALID COUNTRY CODE FORMAT"
ELSE
    DISPLAY "INVALID: MUST BE 2 UPPERCASE LETTERS"
END-IF
```

---

## ขั้นตอนที่ 413: Sign Condition แบบเจาะลึก — IS POSITIVE, IS NEGATIVE, IS ZERO

### แนวคิด

**Sign Condition** ตรวจสอบเครื่องหมายของค่าตัวเลขโดยตรง โดยไม่ต้องเขียนเปรียบเทียบกับ 0 เอง:

| เงื่อนไข | เทียบเท่ากับ |
|---|---|
| `IS POSITIVE` | `> 0` |
| `IS NEGATIVE` | `< 0` |
| `IS ZERO` | `= 0` |

Sign Condition ใช้ได้กับทั้งตัวแปรเดี่ยว ๆ และ**ผลลัพธ์ของนิพจน์ทางคณิตศาสตร์**โดยตรง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SIGN-CONDITION-DEEP-DIVE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-BALANCE      PIC S9(7)V99 VALUE 0.
       01  WS-INCOME       PIC 9(7)V99 VALUE 45000.00.
       01  WS-EXPENSE      PIC 9(7)V99 VALUE 45000.00.
       01  WS-NET          PIC S9(7)V99.

       PROCEDURE DIVISION.
           IF WS-BALANCE IS ZERO
               DISPLAY "BALANCE IS EXACTLY ZERO"
           END-IF.

           COMPUTE WS-NET = WS-INCOME - WS-EXPENSE.
           IF WS-NET IS ZERO
               DISPLAY "BREAK-EVEN: INCOME EQUALS EXPENSE"
           ELSE
               IF WS-NET IS POSITIVE
                   DISPLAY "PROFIT THIS MONTH"
               ELSE
                   DISPLAY "LOSS THIS MONTH"
               END-IF
           END-IF.

           SUBTRACT 100.00 FROM WS-EXPENSE.
           COMPUTE WS-NET = WS-INCOME - WS-EXPENSE.
           IF WS-NET IS POSITIVE
               DISPLAY "PROFIT: " WS-NET
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `IF WS-BALANCE IS ZERO` — ตรวจสอบตัวแปรเดี่ยว ๆ ตรง ๆ
- `COMPUTE WS-NET = WS-INCOME - WS-EXPENSE.` แล้วตรวจสอบ `WS-NET IS ZERO`/`IS POSITIVE` —
  เมื่อรายรับเท่ากับรายจ่ายพอดี (45000.00 ทั้งคู่) ผลต่างเป็น 0 พอดี จึงเข้าเงื่อนไข "BREAK-EVEN"
- หลัง `SUBTRACT 100.00 FROM WS-EXPENSE.` รายจ่ายลดลงเหลือ 44900.00 ทำให้ผลต่างกลายเป็นบวก
  (100.00) จึงแสดงข้อความกำไร

### ผลลัพธ์ที่ได้จากการรันจริง

```
BALANCE IS EXACTLY ZERO
BREAK-EVEN: INCOME EQUALS EXPENSE
PROFIT: +0000100.00
```

### ข้อควรระวัง

- **Sign Condition ใช้ได้เฉพาะกับข้อมูลชนิดตัวเลขเท่านั้น** (`PIC 9`, `PIC S9`, `COMP` ฯลฯ)
  ไม่สามารถใช้กับฟิลด์ `PIC X` ได้โดยตรงในความหมายเดียวกับ `IS NUMERIC` ที่ตรวจสอบรูปแบบตัวอักษร
- `WS-NET` ต้องประกาศเป็น **signed** (`PIC S9...`) เสมอ หากต้องการให้ `IS NEGATIVE` มีโอกาส
  เป็นจริงได้จริง เพราะฟิลด์ unsigned (`PIC 9...` ธรรมดา) ไม่มีทางเก็บค่าติดลบได้ตั้งแต่แรก
  (ทบทวนได้จาก Part 006 เรื่อง PICTURE Clause)

### แบบฝึกหัดที่ 413.1

**โจทย์**: จงเขียนเงื่อนไขตรวจสอบว่าคะแนนสอบที่หักลบแล้ว (`WS-ADJUSTED-SCORE PIC S9(3)`)
ติดลบหรือไม่ ถ้าติดลบให้ปรับเป็น 0 อัตโนมัติ (ป้องกันคะแนนติดลบ)

**เฉลย**:
```cobol
IF WS-ADJUSTED-SCORE IS NEGATIVE
    MOVE 0 TO WS-ADJUSTED-SCORE
END-IF
```

---

## ขั้นตอนที่ 414: NOT กับ Class/Sign Condition และรูปแบบ Validation Loop

### แนวคิด

เช่นเดียวกับเงื่อนไขทั่วไป Class Condition และ Sign Condition สามารถใช้ **`NOT`** นำหน้าเพื่อกลับค่า
ความจริงได้ (`IS NOT NUMERIC`, `IS NOT ALPHABETIC`, `IS NOT POSITIVE` เป็นต้น) ซึ่งมักใช้บ่อย
กว่า `IS ... ` เปล่า ๆ ด้วยซ้ำในการเขียนโค้ดตรวจสอบ**ข้อผิดพลาด** (เพราะเราสนใจ "กรณีที่ผิด" มากกว่า
"กรณีที่ถูก") ขั้นตอนนี้จะแสดงรูปแบบที่พบบ่อยที่สุดคือ **Validation Loop**: วนรับข้อมูลซ้ำจนกว่าจะ
ถูกต้อง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. NOT-NUMERIC-VALIDATION-LOOP.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-INPUT-QTY     PIC X(5).
       01  WS-VALID-FLAG    PIC X VALUE "N".
       01  WS-ATTEMPT-COUNT PIC 9(1) VALUE 0.

       PROCEDURE DIVISION.
           PERFORM UNTIL WS-VALID-FLAG = "Y" OR WS-ATTEMPT-COUNT >= 3
               ADD 1 TO WS-ATTEMPT-COUNT
               DISPLAY "ENTER QUANTITY (DIGITS ONLY): "
                   WITH NO ADVANCING
               ACCEPT WS-INPUT-QTY
      *> NOTE: WS-INPUT-QTY IS NOT NUMERIC would fail here even for
      *> good input, because ACCEPT pads the rest of the field with
      *> trailing spaces. FUNCTION TRIM removes them before testing.
               IF FUNCTION TRIM(WS-INPUT-QTY) IS NUMERIC
                   MOVE "Y" TO WS-VALID-FLAG
                   DISPLAY "ACCEPTED QUANTITY: "
                       FUNCTION TRIM(WS-INPUT-QTY)
               ELSE
                   DISPLAY "INVALID INPUT, PLEASE TRY AGAIN."
               END-IF
           END-PERFORM.

           IF WS-VALID-FLAG NOT = "Y"
               DISPLAY "TOO MANY INVALID ATTEMPTS. ABORTING."
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `PERFORM UNTIL WS-VALID-FLAG = "Y" OR WS-ATTEMPT-COUNT >= 3` — วนรับข้อมูลสูงสุด 3 ครั้ง
  หรือจนกว่าจะได้ข้อมูลที่ถูกต้อง (เทคนิคจาก Part 012)
- `IF FUNCTION TRIM(WS-INPUT-QTY) IS NUMERIC` — ใช้ `FUNCTION TRIM` (Part 037) ตัดช่องว่าง
  ที่ `ACCEPT` เติมเข้ามาออกก่อนตรวจสอบ ตรงตามข้อค้นพบสำคัญจากขั้นตอนที่ 411 ที่ว่าช่องว่างทำให้
  `IS NUMERIC` เป็นเท็จเสมอ

### ผลการทดสอบจริง (ป้อนข้อมูลผ่าน pipe เพื่อทดสอบอัตโนมัติ)

ทดสอบด้วยคำสั่ง `printf "AB12\nXY99\n150\n" | ./not-numeric-validation-loop` ได้ผลลัพธ์จริง:

```
ENTER QUANTITY (DIGITS ONLY): INVALID INPUT, PLEASE TRY AGAIN.
ENTER QUANTITY (DIGITS ONLY): INVALID INPUT, PLEASE TRY AGAIN.
ENTER QUANTITY (DIGITS ONLY): ACCEPTED QUANTITY: 150
```

ป้อน "AB12" (ไม่ใช่ตัวเลข) → ปฏิเสธ, ป้อน "XY99" (ไม่ใช่ตัวเลข) → ปฏิเสธ, ป้อน "150" (ตัวเลขล้วน)
→ ยอมรับที่ครั้งที่ 3 พอดี

### ข้อควรระวัง

- **หากลบ `FUNCTION TRIM` ออกจากเงื่อนไข** (`IF WS-INPUT-QTY IS NUMERIC` ตรง ๆ) ผลลัพธ์จะ
  แย่กว่ามาก: แม้ผู้ใช้จะป้อน "150" (ตัวเลขล้วนที่ถูกต้อง) ก็ยังคงถูกปฏิเสธว่า "INVALID INPUT" ทุกครั้ง!
  เพราะ `WS-INPUT-QTY PIC X(5)` จะเก็บค่าเป็น `"150  "` (มีช่องว่างต่อท้าย 2 ตัวจาก `ACCEPT`)
  ซึ่งตามข้อค้นพบในขั้นตอนที่ 411 ทำให้ `IS NUMERIC` เป็นเท็จเสมอ — นี่คือบั๊กจริงที่พบบ่อยมากใน
  โค้ด COBOL ของมือใหม่ที่ตรวจสอบข้อมูลจาก `ACCEPT` โดยไม่ใช้ `FUNCTION TRIM` ก่อน
- การใช้ `WITH NO ADVANCING` ใน `DISPLAY` ทำให้ข้อความคำถามกับผลลัพธ์อยู่บรรทัดเดียวกัน (ทบทวน
  ได้จาก Part 007)

### แบบฝึกหัดที่ 414.1

**โจทย์**: จงอธิบายว่าทำไม `FUNCTION TRIM(WS-INPUT-QTY) IS NUMERIC` จึงทำงานถูกต้อง ในขณะที่
`MOVE FUNCTION TRIM(WS-INPUT-QTY) TO WS-SOME-X-FIELD` แล้วเช็ค `WS-SOME-X-FIELD IS NUMERIC`
(โดยที่ `WS-SOME-X-FIELD` มีขนาดเท่ากับ `WS-INPUT-QTY`) กลับไม่ช่วยแก้ปัญหา

**เฉลยแนวทาง**: เพราะ `IF FUNCTION TRIM(...) IS NUMERIC` ตรวจสอบผลลัพธ์ของฟังก์ชัน**ในทันที**
โดยไม่ผ่านการ `MOVE` เข้าฟิลด์ `PIC X` อื่นเลย ในขณะที่การ `MOVE FUNCTION TRIM(...) TO
WS-SOME-X-FIELD` เป็นการ MOVE แบบ alphanumeric ปกติ (Part 008) ซึ่งจะ**เติมช่องว่างด้านขวา
กลับเข้าไปใหม่**หากฟิลด์ปลายทางมีขนาดใหญ่กว่าข้อความที่ตัดช่องว่างแล้ว (เช่น "150" ถูกตัดเหลือ
3 ตัวอักษร แต่ MOVE เข้าฟิลด์ขนาด 5 ตัวอักษรจะได้ "150  " กลับมาเหมือนเดิม) ทำให้ปัญหาเดิมกลับมา
อีกครั้ง วิธีแก้ที่ถูกต้องคือตรวจสอบผลลัพธ์ของ `FUNCTION TRIM` โดยตรงในเงื่อนไข หรือ MOVE เข้า
ฟิลด์ตัวเลขแท้ (`PIC 9`) แทน (ซึ่งจะจัดรูปแบบให้ถูกต้องเสมอ ตามที่จะสาธิตในขั้นตอนที่ 415)

---

## ขั้นตอนที่ 415: กับดักช่องว่างของ IS NUMERIC — วิเคราะห์เจาะลึกและวิธีแก้ที่ทดสอบแล้ว

### แนวคิด

ขั้นตอนนี้ขยายความข้อค้นพบสำคัญจากขั้นตอนที่ 411 และ 414 ให้ครบถ้วนที่สุด ด้วยการทดสอบวิธีแก้ไข
ที่เป็นไปได้หลายแบบเทียบกัน เพื่อพิสูจน์ว่าวิธีไหนใช้ได้จริงและวิธีไหนใช้ไม่ได้

### ตัวอย่างโค้ด — ทดสอบวิธีแก้ 3 แบบเทียบกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. NUMERIC-TRAILING-SPACE-FIX.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW-FIELD    PIC X(5) VALUE "150  ".
       01  WS-MOVED-FIELD  PIC X(5).
       01  WS-NUM-FIELD    PIC 9(5).

       PROCEDURE DIVISION.
      *> Attempt 1: check the raw field directly (BROKEN).
           IF WS-RAW-FIELD IS NUMERIC
               DISPLAY "ATTEMPT 1 (RAW FIELD)      : NUMERIC"
           ELSE
               DISPLAY "ATTEMPT 1 (RAW FIELD)      : NOT NUMERIC"
           END-IF.

      *> Attempt 2: MOVE FUNCTION TRIM into a same-size PIC X
      *> field first, THEN check (STILL BROKEN).
           MOVE FUNCTION TRIM(WS-RAW-FIELD) TO WS-MOVED-FIELD.
           IF WS-MOVED-FIELD IS NUMERIC
               DISPLAY "ATTEMPT 2 (TRIM THEN MOVE) : NUMERIC"
           ELSE
               DISPLAY "ATTEMPT 2 (TRIM THEN MOVE) : NOT NUMERIC"
           END-IF.

      *> Attempt 3: check FUNCTION TRIM's result directly,
      *> with no intermediate MOVE into a PIC X field (WORKS).
           IF FUNCTION TRIM(WS-RAW-FIELD) IS NUMERIC
               DISPLAY "ATTEMPT 3 (TRIM IN IF)     : NUMERIC"
           ELSE
               DISPLAY "ATTEMPT 3 (TRIM IN IF)     : NOT NUMERIC"
           END-IF.

      *> Attempt 4: MOVE the raw field into a real numeric PIC 9
      *> field first, then check that field instead (ALSO WORKS).
           MOVE WS-RAW-FIELD TO WS-NUM-FIELD.
           IF WS-NUM-FIELD IS NUMERIC
               DISPLAY "ATTEMPT 4 (MOVE TO PIC 9)  : NUMERIC"
           ELSE
               DISPLAY "ATTEMPT 4 (MOVE TO PIC 9)  : NOT NUMERIC"
           END-IF.
           DISPLAY "ATTEMPT 4 VALUE: " WS-NUM-FIELD.
           STOP RUN.
```

### ผลลัพธ์ที่ได้จากการรันจริง

```
ATTEMPT 1 (RAW FIELD)      : NOT NUMERIC
ATTEMPT 2 (TRIM THEN MOVE) : NOT NUMERIC
ATTEMPT 3 (TRIM IN IF)     : NUMERIC
ATTEMPT 4 (MOVE TO PIC 9)  : NUMERIC
ATTEMPT 4 VALUE: 00150
```

### วิเคราะห์ผลลัพธ์

- **Attempt 1 ล้มเหลว** ตามที่คาดไว้ (ช่องว่างต่อท้ายทำให้ไม่ NUMERIC)
- **Attempt 2 ล้มเหลวเช่นกัน แม้จะใช้ `FUNCTION TRIM` แล้วก็ตาม!** เพราะ MOVE ผลลัพธ์ที่ตัด
  ช่องว่างแล้ว ("150") เข้าฟิลด์ `PIC X(5)` ที่มีขนาดใหญ่กว่า ทำให้ COBOL เติมช่องว่างด้านขวากลับเข้า
  มาใหม่ตามกฎ MOVE ปกติ (Part 008) กลายเป็น "150  " เหมือนเดิมทุกประการ — วิธีนี้จึง**ใช้ไม่ได้จริง**
  แม้จะดูเหมือนใช้ `FUNCTION TRIM` ถูกต้องแล้วก็ตาม
- **Attempt 3 สำเร็จ** เพราะตรวจสอบผลลัพธ์ของ `FUNCTION TRIM` **โดยตรงในเงื่อนไข** โดยไม่มีการ
  `MOVE` เข้าฟิลด์ `PIC X` คั่นกลางเลย
- **Attempt 4 สำเร็จเช่นกัน** ด้วยวิธีที่ต่างออกไป: MOVE ฟิลด์ดิบเข้า `WS-NUM-FIELD` ซึ่งเป็น
  **ฟิลด์ตัวเลขแท้** (`PIC 9(5)`) โดยตรง — MOVE แบบตัวเลข (numeric MOVE) จะจัดตำแหน่งเลข
  ให้ชิดขวาและเติมศูนย์นำหน้าให้อัตโนมัติเสมอ (Part 008) ทำให้ไม่มีช่องว่างเหลืออยู่ได้เลย ผลลัพธ์คือ
  "00150" ซึ่งเป็น NUMERIC แน่นอน

### ข้อควรระวัง

- **จำสองวิธีแก้ที่ทดสอบแล้วว่าใช้งานได้จริงนี้ไว้ให้แม่น**: (1) ตรวจสอบ `FUNCTION TRIM(...)
  IS NUMERIC` โดยตรงในเงื่อนไข ไม่ผ่านตัวแปรคั่นกลาง หรือ (2) `MOVE` เข้าฟิลด์ตัวเลขแท้ (`PIC 9`)
  ก่อนแล้วค่อยตรวจสอบ
- **ห้ามเข้าใจผิดว่าการ `MOVE FUNCTION TRIM(...)` เข้าฟิลด์ `PIC X` ขนาดเท่าเดิมแล้วช่วยแก้ปัญหา
  ได้** เพราะได้พิสูจน์แล้วว่าไม่ช่วยอะไรเลย (Attempt 2)
- วิธีที่ (2) มีข้อดีเพิ่มเติมคือได้ค่าตัวเลขที่พร้อมนำไปคำนวณต่อทันที (จัดรูปแบบเรียบร้อยแล้ว)
  ในขณะที่วิธีที่ (1) เหมาะกับการตรวจสอบอย่างเดียวโดยยังไม่ต้องการแปลงชนิดข้อมูล

### แบบฝึกหัดที่ 415.1

**โจทย์**: จงอธิบายว่าทำไม Attempt 4 (MOVE เข้า `PIC 9`) จึงมีความเสี่ยงที่ต้องระวังเพิ่มเติม
หากค่าดิบใน `WS-RAW-FIELD` **ไม่ใช่ตัวเลขเลย** (เช่น "AB12 ")

**เฉลยแนวทาง**: การ `MOVE` ค่าที่ไม่ใช่ตัวเลข (เช่น "AB12 ") เข้าฟิลด์ `PIC 9` โดยตรงเป็นพฤติกรรม
ที่ไม่ได้นิยามผลลัพธ์ไว้ชัดเจนตามมาตรฐาน (undefined/implementation-defined behavior) และอาจ
ทำให้ได้ค่าขยะที่ไม่คาดคิดในฟิลด์ปลายทาง (ทบทวนกับดักนี้ได้จาก Part 008 ขั้นตอนที่ 79) ดังนั้น
แนวทางที่ปลอดภัยกว่าคือ**ตรวจสอบด้วย Attempt 3 (`FUNCTION TRIM(...) IS NUMERIC`) ก่อนเสมอ**
แล้วจึงค่อย `MOVE` เข้าฟิลด์ตัวเลขในภายหลัง เมื่อมั่นใจแล้วว่าข้อมูลถูกต้องจริง — ไม่ควร `MOVE` เข้า
ฟิลด์ตัวเลขก่อนแล้วค่อยตรวจสอบทีหลังแบบ Attempt 4 หากยังไม่เคยตรวจสอบความถูกต้องมาก่อนเลย

---

## ขั้นตอนที่ 416: Class Condition ที่กำหนดเอง — CLASS Clause ใน SPECIAL-NAMES

### แนวคิด

นอกจาก Class Condition มาตรฐาน (`IS NUMERIC`, `IS ALPHABETIC`) COBOL ยังให้เรา**กำหนด
กลุ่มตัวอักษรของตัวเอง**ผ่าน `CLASS` clause ใน `SPECIAL-NAMES` paragraph (ส่วนหนึ่งของ
ENVIRONMENT DIVISION ที่เรียนใน Part 004) ทำให้สร้างเงื่อนไขตรวจสอบรูปแบบเฉพาะทางธุรกิจได้
โดยไม่ต้องเขียน `INSPECT`/loop ตรวจสอบตัวอักษรทีละตัวเอง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTOM-CLASS-DEMO.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
       SPECIAL-NAMES.
      *> Define a custom class: valid hexadecimal digit characters.
           CLASS HEX-DIGIT IS "0" THRU "9", "A" THRU "F".

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CODE   PIC X(4) VALUE "1AF3".
       01  WS-BAD    PIC X(4) VALUE "1AG3".

       PROCEDURE DIVISION.
           IF WS-CODE IS HEX-DIGIT
               DISPLAY "WS-CODE (1AF3) IS ALL HEX DIGITS"
           ELSE
               DISPLAY "WS-CODE (1AF3) IS NOT ALL HEX DIGITS"
           END-IF.

           IF WS-BAD IS HEX-DIGIT
               DISPLAY "WS-BAD (1AG3) IS ALL HEX DIGITS"
           ELSE
               DISPLAY "WS-BAD (1AG3) IS NOT ALL HEX DIGITS"
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `CLASS HEX-DIGIT IS "0" THRU "9", "A" THRU "F".` — ประกาศกลุ่มตัวอักษรใหม่ชื่อ `HEX-DIGIT`
  ที่ประกอบด้วยตัวเลข 0-9 และตัวอักษร A-F (ตัวเลขฐาน 16 ที่ถูกต้อง)
- `IF WS-CODE IS HEX-DIGIT` — ใช้ชื่อกลุ่มที่กำหนดเองเป็นเงื่อนไขได้เหมือน `IS NUMERIC`/
  `IS ALPHABETIC` มาตรฐานทุกประการ
- `WS-CODE` ("1AF3") ทุกตัวอักษรอยู่ในกลุ่ม HEX-DIGIT จึงเป็นจริง แต่ `WS-BAD` ("1AG3") มี "G"
  ซึ่งไม่ใช่เลขฐาน 16 ที่ถูกต้อง (ไม่อยู่ในช่วง A-F) จึงเป็นเท็จ

### ผลลัพธ์ที่ได้จากการรันจริง

```
WS-CODE (1AF3) IS ALL HEX DIGITS
WS-BAD (1AG3) IS NOT ALL HEX DIGITS
```

### ข้อควรระวัง

- `CLASS` clause ต้องประกาศใน `SPECIAL-NAMES` paragraph ของ `CONFIGURATION SECTION`
  ใน ENVIRONMENT DIVISION เท่านั้น ไม่สามารถประกาศใน DATA DIVISION ได้
- ชื่อ class ที่กำหนดเอง (เช่น `HEX-DIGIT`) กลายเป็นคำสงวนเฉพาะในโปรแกรมนั้น ต้องเลือกชื่อที่ไม่ชน
  กับตัวแปรหรือคำสงวนอื่น ๆ
- ฟีเจอร์นี้มีประโยชน์มากสำหรับตรวจสอบรูปแบบรหัสเฉพาะทางธุรกิจ (เช่น รหัสไปรษณีย์, รหัสสินค้าที่มี
  ชุดตัวอักษรจำกัด) แต่ควรใช้ชื่ออย่างระมัดระวัง เพราะเป็นการประกาศระดับโปรแกรม (ไม่ใช่แค่ตัวแปร)

### แบบฝึกหัดที่ 416.1

**โจทย์**: จงประกาศ `CLASS` ที่กำหนดเองชื่อ `THAI-DIGIT` ที่ตรวจสอบว่าเป็นตัวเลข 0-9 หรือ
เครื่องหมายจุด (`.`) เท่านั้น (สำหรับตรวจสอบรูปแบบตัวเลขทศนิยมอย่างง่าย)

**เฉลย**:
```cobol
       SPECIAL-NAMES.
           CLASS NUMERIC-OR-DOT IS "0" THRU "9", ".".
```
(หมายเหตุ: ชื่อ class ต้องเป็นภาษาอังกฤษ/ASCII ตามกฎเหล็กของหลักสูตรนี้ จึงตั้งชื่อว่า
`NUMERIC-OR-DOT` แทน `THAI-DIGIT` ที่เป็นชื่อภาษาอังกฤษแต่สื่อความหมายผิดเพราะเนื้อหาจริงเป็น
ตัวเลขอารบิกและจุดทศนิยม ไม่เกี่ยวกับเลขไทย)

---

## ขั้นตอนที่ 417: Sign Condition กับ DIVIDE REMAINDER — เครื่องหมายที่อาจไม่คาดคิด

### แนวคิด

ขั้นตอนนี้ทดสอบพฤติกรรมของ Sign Condition ร่วมกับ `DIVIDE ... REMAINDER` (Part 009) ในกรณีที่
ตัวตั้งหาร (dividend) เป็นค่าลบ — พฤติกรรมนี้อาจขัดกับความเข้าใจของโปรแกรมเมอร์ที่มาจากภาษาอื่น
บางภาษา (เช่น Python) ที่ผลลัพธ์ modulo มักเป็นบวกเสมอ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DIVIDE-REMAINDER-SIGN-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DIVIDEND  PIC S9(5) VALUE -17.
       01  WS-DIVISOR   PIC S9(5) VALUE 5.
       01  WS-QUOTIENT  PIC S9(5).
       01  WS-REMAINDER PIC S9(5).

       PROCEDURE DIVISION.
           DIVIDE WS-DIVIDEND BY WS-DIVISOR
               GIVING WS-QUOTIENT
               REMAINDER WS-REMAINDER.

           DISPLAY "QUOTIENT: " WS-QUOTIENT.
           DISPLAY "REMAINDER: " WS-REMAINDER.

           IF WS-REMAINDER IS NEGATIVE
               DISPLAY "REMAINDER IS NEGATIVE -- WATCH OUT"
           ELSE
               IF WS-REMAINDER IS ZERO
                   DISPLAY "DIVIDES EVENLY"
               ELSE
                   DISPLAY "REMAINDER IS POSITIVE"
               END-IF
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `-17 / 5` — COBOL คำนวณผลหารแบบ**ตัดเศษเข้าหาศูนย์ (truncation toward zero)** ได้ผลหาร -3
  (ไม่ใช่ -4 แบบการปัดลง/floor division ที่บางภาษาใช้) และเศษที่เหลือคือ -17 - (-3 × 5) = -2
- **เศษที่เหลือ (remainder) มีเครื่องหมายตามตัวตั้งหาร (dividend) เสมอ** ในที่นี้ตัวตั้งหารเป็นลบ
  เศษที่เหลือจึงเป็นลบด้วย (-2) ไม่ใช่บวก

### ผลลัพธ์ที่ได้จากการรันจริง

```
QUOTIENT: -00003
REMAINDER: -00002
REMAINDER IS NEGATIVE -- WATCH OUT
```

### ทำไมเรื่องนี้ถึงสำคัญ

โปรแกรมเมอร์ที่คุ้นเคยกับภาษาอย่าง Python (ที่ `-17 % 5` ให้ผลลัพธ์เป็น `3`, เป็นบวกเสมอตามกฎ
floor division) อาจคาดหวังผิดว่า COBOL จะให้ผลลัพธ์แบบเดียวกัน แต่ COBOL ใช้กฎการตัดเศษเข้าหา
ศูนย์ ทำให้ผลลัพธ์ต่างกันโดยสิ้นเชิงเมื่อตัวตั้งหารติดลบ หากนำ `WS-REMAINDER` ไปใช้เป็น index
ของ array หรือใช้ในการคำนวณต่อโดยไม่ตรวจสอบเครื่องหมายก่อน อาจเกิดข้อผิดพลาดที่ตรวจจับได้ยากมาก

### ข้อควรระวัง

- **ควรตรวจสอบ `IS NEGATIVE` กับผลลัพธ์ของ `REMAINDER` เสมอ** เมื่อมีโอกาสที่ตัวตั้งหารจะติดลบ
  ก่อนนำไปใช้งานต่อ โดยเฉพาะเมื่อคาดหวังผลลัพธ์แบบไม่ติดลบ (เช่น ใช้เป็น index หรือใช้แสดงผลกับ
  ผู้ใช้)
- หากต้องการพฤติกรรมแบบ "เศษเป็นบวกเสมอ" (คล้าย floor division) ต้องเขียนตรรกะเพิ่มเติมเอง เช่น
  `IF WS-REMAINDER IS NEGATIVE ADD WS-DIVISOR TO WS-REMAINDER END-IF` (บวกตัวหารกลับเข้าไป
  เมื่อเศษติดลบ)

### แบบฝึกหัดที่ 417.1

**โจทย์**: จงเขียนโค้ดปรับ `WS-REMAINDER` จากตัวอย่างข้างต้นให้เป็นค่าบวกเสมอ (แบบ floor
division ของ Python) โดยใช้ Sign Condition

**เฉลย**:
```cobol
           DIVIDE WS-DIVIDEND BY WS-DIVISOR
               GIVING WS-QUOTIENT
               REMAINDER WS-REMAINDER.

           IF WS-REMAINDER IS NEGATIVE
               ADD WS-DIVISOR TO WS-REMAINDER
               SUBTRACT 1 FROM WS-QUOTIENT
           END-IF.
```
(การ `SUBTRACT 1 FROM WS-QUOTIENT` เพิ่มเติมทำให้ผลหารสอดคล้องกับเศษที่ปรับใหม่ ตรงตามหลัก
คณิตศาสตร์ที่ว่า `dividend = quotient * divisor + remainder` ต้องเป็นจริงเสมอ)

---

## ขั้นตอนที่ 418: ผสาน Class Condition และ Sign Condition ในโปรแกรมเดียว

### แนวคิด

ขั้นตอนนี้แสดงการผสาน Class Condition และ Sign Condition เข้าด้วยกันในสถานการณ์ตรวจสอบข้อมูล
ลูกค้าจริง ที่ต้องตรวจทั้งรูปแบบตัวอักษร (ชื่อ), รูปแบบตัวเลข (รหัสบัญชี), และเครื่องหมายของยอดเงิน
พร้อมกันในโปรแกรมเดียว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CLASS-AND-SIGN-COMBINED-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUST-NAME     PIC X(15) VALUE "Somchai".
       01  WS-CUST-AGE      PIC S9(3) VALUE 25.
       01  WS-ACCOUNT-CODE  PIC X(6)  VALUE "AC12F9".
       01  WS-BALANCE       PIC S9(7)V99 VALUE -1500.00.
       01  WS-ERROR-COUNT   PIC 9(1) VALUE 0.

       PROCEDURE DIVISION.
      *> Rule 1: name must be alphabetic (letters and spaces only)
           IF WS-CUST-NAME IS NOT ALPHABETIC
               DISPLAY "ERROR: NAME MUST CONTAIN LETTERS ONLY"
               ADD 1 TO WS-ERROR-COUNT
           END-IF.

      *> Rule 2: age must be numeric and positive
           IF WS-CUST-AGE IS NOT NUMERIC OR WS-CUST-AGE IS NOT POSITIVE
               DISPLAY "ERROR: AGE MUST BE A POSITIVE NUMBER"
               ADD 1 TO WS-ERROR-COUNT
           END-IF.

      *> Rule 3: account code must start with 2 alphabetic letters
           IF WS-ACCOUNT-CODE(1:2) IS NOT ALPHABETIC
               DISPLAY "ERROR: ACCOUNT CODE PREFIX MUST BE LETTERS"
               ADD 1 TO WS-ERROR-COUNT
           END-IF.

      *> Rule 4: report account status by balance sign
           EVALUATE TRUE
               WHEN WS-BALANCE IS NEGATIVE
                   DISPLAY "ACCOUNT STATUS: OVERDRAWN"
               WHEN WS-BALANCE IS ZERO
                   DISPLAY "ACCOUNT STATUS: EMPTY"
               WHEN OTHER
                   DISPLAY "ACCOUNT STATUS: HAS FUNDS"
           END-EVALUATE.

           DISPLAY "TOTAL VALIDATION ERRORS: " WS-ERROR-COUNT.
           STOP RUN.
```

### อธิบายโค้ด

- Rule 1 ใช้ `IS NOT ALPHABETIC` ตรวจสอบชื่อ (Part นี้ ขั้นตอนที่ 412)
- Rule 2 ผสาน `IS NOT NUMERIC` กับ `IS NOT POSITIVE` ด้วย `OR` — ทั้งสองเป็น Class/Sign
  Condition คนละประเภทที่ทำงานร่วมกันได้อย่างเป็นธรรมชาติในเงื่อนไขผสมเดียว (Part 010 ขั้นตอนที่ 94)
- Rule 3 ใช้ **Reference Modification** `WS-ACCOUNT-CODE(1:2)` (Part 019/020 บริบทการตัด
  ข้อความ) ร่วมกับ `IS NOT ALPHABETIC` เพื่อตรวจสอบเฉพาะ 2 ตัวอักษรแรกของรหัสบัญชี
- Rule 4 ใช้ `EVALUATE TRUE` ร่วมกับ Sign Condition (Part 011 + ขั้นตอนที่ 413 ของ Part นี้)

### ผลลัพธ์ที่ได้จากการรันจริง

```
ACCOUNT STATUS: OVERDRAWN
TOTAL VALIDATION ERRORS: 0
```

ด้วยข้อมูลตัวอย่างนี้ทุกกฎผ่านหมด (ชื่อเป็นตัวอักษรล้วน, อายุเป็นบวก, รหัสบัญชีขึ้นต้นด้วยตัวอักษร)
มีเพียงยอดเงินที่ติดลบซึ่งรายงานเป็นสถานะ "OVERDRAWN" ตามที่ออกแบบไว้ (ไม่ใช่ข้อผิดพลาด แต่เป็น
สถานะทางธุรกิจปกติของบัญชีที่เบิกเกินบัญชี)

### ข้อควรระวัง

- สังเกตว่า "ยอดเงินติดลบ" **ไม่ใช่ข้อผิดพลาดในการตรวจสอบข้อมูล** (validation error) แต่เป็น
  **สถานะทางธุรกิจ** (business status) — ต้องแยกความแตกต่างระหว่างสองสิ่งนี้ให้ชัดเจนเสมอ: การ
  ตรวจสอบข้อมูลตรวจว่า "รูปแบบ/ชนิดข้อมูลถูกต้องหรือไม่" ในขณะที่สถานะทางธุรกิจสะท้อน "ความหมาย
  ของค่านั้นในบริบทที่ใช้งาน ซึ่งบางครั้งค่าที่ถูกต้องตามรูปแบบ (ยอดเงินติดลบเป็นตัวเลขที่ถูกต้อง) ก็ยังคง
  มีความหมายทางธุรกิจที่ต้องจัดการแยกต่างหาก
- Reference Modification (`field(start:length)`) ต้องระวังไม่ให้ `start + length - 1` เกิน
  ความยาวจริงของฟิลด์ มิเช่นนั้นจะเกิด runtime error หรือพฤติกรรมไม่คาดคิด

### แบบฝึกหัดที่ 418.1

**โจทย์**: จงอธิบายว่าทำไม Rule 2 จึงเขียนเป็น `IS NOT NUMERIC OR IS NOT POSITIVE` แทนที่จะ
เขียนแยกเป็นสอง `IF` เดี่ยว ๆ

**เฉลยแนวทาง**: ทั้งสองแบบให้ผลลัพธ์การตรวจับข้อผิดพลาดเหมือนกัน แต่การรวมเป็นเงื่อนไขเดียวด้วย
`OR` ทำให้เพิ่ม `WS-ERROR-COUNT` เพียงครั้งเดียวต่อ 1 ข้อผิดพลาดด้านอายุ (แม้จะผิดทั้งสองแบบพร้อม
กัน เช่น ค่าไม่ใช่ตัวเลขและติดลบพร้อมกันในบางกรณี) ในขณะที่การแยกเป็นสอง `IF` เดี่ยวอาจทำให้
`WS-ERROR-COUNT` เพิ่มขึ้น 2 ครั้งสำหรับข้อผิดพลาดที่ถือว่าเป็นเรื่องเดียวกันทางธุรกิจ ("อายุไม่ถูกต้อง")
การออกแบบเงื่อนไขผสมจึงขึ้นอยู่กับว่าต้องการนับข้อผิดพลาดแบบละเอียด (แยกทุกกฎย่อย) หรือแบบรวม
(นับเป็นภาพรวมต่อฟิลด์) ตามความต้องการของระบบจริง

---

## ขั้นตอนที่ 419: ตารางสรุปกับดักที่พบบ่อยของ Class/Sign Condition

### แนวคิด

ขั้นตอนนี้สรุปกับดักทั้งหมดที่ทดสอบพบตลอด Part นี้ไว้ในที่เดียว เพื่อให้ทบทวนได้ง่ายก่อนไปสอบหรือ
ก่อนเขียนโค้ดตรวจสอบข้อมูลในโปรเจกต์จริง

### ตารางสรุปกับดักที่ทดสอบยืนยันแล้วจริง

| กับดัก | ตัวอย่าง | ผลลัพธ์ที่พบจริง | วิธีแก้ที่ทดสอบแล้ว |
|---|---|---|---|
| ช่องว่างในฟิลด์ `PIC X` ทำให้ `IS NUMERIC` เป็นเท็จ | `"  123"`, `"150  "` | `NOT NUMERIC` | ใช้ `FUNCTION TRIM(...)` ตรวจสอบโดยตรงในเงื่อนไข หรือ MOVE เข้าฟิลด์ `PIC 9` ก่อน |
| MOVE ผลลัพธ์ TRIM เข้าฟิลด์ `PIC X` ขนาดเท่าเดิม แล้วเช็คทีหลัง | `MOVE FUNCTION TRIM(F) TO SAME-SIZE-X-FIELD` | ช่องว่างถูกเติมกลับมาใหม่ ยัง `NOT NUMERIC` | ตรวจสอบ `FUNCTION TRIM(...)` **ในเงื่อนไขทันที** ไม่ผ่านตัวแปรคั่นกลาง |
| เครื่องหมายลบตัวอักษรใน `PIC X` | `"-1234"` (PIC X) | `NOT NUMERIC` | ต้องเป็นฟิลด์ signed แท้ (`PIC S9`) จึงรองรับเครื่องหมายลบได้ |
| `IS ALPHABETIC` ยอมรับตัวพิมพ์ผสม | `"Hello"` | `ALPHABETIC` = จริง, `ALPHABETIC-UPPER` = เท็จ | ใช้ `ALPHABETIC-UPPER`/`-LOWER` หากต้องการบังคับ case ที่แน่นอน |
| `IS NUMERIC` ไม่ตรวจสอบช่วงค่าทางธุรกิจ | `-5` เป็น `PIC S9` | `IS NUMERIC` = จริง แม้เป็นค่าติดลบที่ไม่สมเหตุสมผลทางธุรกิจ (เช่น อายุ) | ต้องใช้ Sign Condition (`IS POSITIVE`) หรือเงื่อนไขช่วง (`THRU`, 88-level) ร่วมด้วยเสมอ |
| Remainder ของ `DIVIDE` ติดลบตามตัวตั้งหาร | `-17 / 5` | เศษ = -2 (ไม่ใช่ +3 แบบ floor division) | ตรวจสอบ `IS NEGATIVE` แล้วปรับค่าด้วยมือหากต้องการเศษเป็นบวกเสมอ |

### หลักการสำคัญที่ควรจำ

1. **Class Condition ตรวจสอบ "รูปแบบ" (format) เท่านั้น ไม่ตรวจสอบ "ความสมเหตุสมผลทางธุรกิจ"
   (business validity)** — ค่าที่ผ่าน `IS NUMERIC` อาจยังคงเป็นค่าที่ไม่สมเหตุสมผล (เช่น อายุ
   -5 ปี, เลขบัตรประชาชนที่มีจำนวนหลักถูกต้องแต่ตรวจ checksum ไม่ผ่าน) ต้องใช้ Sign Condition,
   88-level, หรือตรรกะทางธุรกิจเพิ่มเติมเสมอ
2. **ช่องว่างเป็นศัตรูตัวฉกาจของ `IS NUMERIC`** แต่กลับเป็นมิตรของ `IS ALPHABETIC` — ต้องจำความ
   ไม่สมมาตรนี้ให้แม่น
3. **ทุกครั้งที่รับข้อมูลจาก `ACCEPT` หรือไฟล์ภายนอกเข้าฟิลด์ `PIC X` แล้วต้องการตรวจสอบว่าเป็น
   ตัวเลขหรือไม่ ให้ใช้ `FUNCTION TRIM` ควบคู่เสมอ** เพื่อป้องกันกับดักช่องว่างที่พบบ่อยที่สุดใน Part นี้

### ข้อควรระวัง

- ตารางนี้สรุปเฉพาะกับดักที่**ทดสอบยืนยันแล้วจริง**ด้วย GnuCOBOL 4.0 (early-dev) ในหลักสูตรนี้
  คอมไพเลอร์ COBOL ตัวอื่นอาจมีรายละเอียดปลีกย่อยต่างกันบ้าง (โดยเฉพาะเรื่อง sign condition กับ
  ฟิลด์ signed ที่ overpunch) ควรทดสอบยืนยันอีกครั้งหากย้ายไปใช้คอมไพเลอร์อื่นในงานจริง

### แบบฝึกหัดที่ 419.1

**โจทย์**: จงเลือกกับดัก 1 ข้อจากตารางข้างต้น แล้วอธิบายด้วยคำพูดของตัวเองว่าทำไมมันถึงเป็นอันตราย
ต่อโปรแกรมมือใหม่เป็นพิเศษ

**เฉลยแนวทาง**: (ตัวอย่างคำตอบ) กับดักเรื่องช่องว่างใน `IS NUMERIC` อันตรายเป็นพิเศษเพราะ**ไม่มี
compile error หรือ warning ใด ๆ เลย** โปรแกรมคอมไพล์ผ่านและรันได้ปกติ แต่ตรรกะการตรวจสอบข้อมูล
กลับปฏิเสธข้อมูลที่ถูกต้อง 100% ทุกครั้งที่รับผ่าน `ACCEPT` ทำให้ผู้ใช้โปรแกรมสับสนว่าทำไมพิมพ์ตัวเลข
ถูกต้องแล้วยังถูกปฏิเสธอยู่เรื่อย ๆ และมือใหม่มักจะไปหาสาเหตุผิดจุด (เช่น สงสัยว่า `ACCEPT` มีปัญหา
หรือคิดว่า `IS NUMERIC` เขียนผิด) แทนที่จะรู้ทันทีว่าต้นเหตุคือช่องว่างที่ `ACCEPT` เติมเข้ามาในฟิลด์

---

## ขั้นตอนที่ 420: โปรแกรมรวบยอด — โมดูลตรวจสอบข้อมูลลูกค้าแบบครบวงจร

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่จำลองการตรวจสอบข้อมูลลูกค้าหลายรายการพร้อมกัน (ผ่านตารางข้อมูล
ทดสอบใน WORKING-STORAGE แทนการ `ACCEPT` จริง เพื่อให้ผลลัพธ์สามารถตรวจสอบซ้ำได้แน่นอน)
ผสานทุกเทคนิคของ Part นี้: `IS ALPHABETIC`, `IS NOT NUMERIC` (พร้อม `FUNCTION TRIM` แก้กับดัก
ช่องว่าง), `IS POSITIVE`, `IS NOT ALPHABETIC-UPPER`, `EVALUATE TRUE` กับ Sign Condition,
และการนับจำนวนข้อผิดพลาดต่อ record

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. VALIDATION-CAPSTONE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUST-NAME     PIC X(15).
       01  WS-CUST-AGE-RAW  PIC X(5).
       01  WS-CUST-AGE      PIC 9(3).
       01  WS-ACCOUNT-CODE  PIC X(6).
       01  WS-BALANCE       PIC S9(7)V99.
       01  WS-ERROR-COUNT   PIC 9(2).
       01  WS-VALID-FLAG    PIC X(1).

       01  TEST-CASE-TABLE.
           05  TEST-CASE OCCURS 3 TIMES.
               10  TC-NAME     PIC X(15).
               10  TC-AGE-RAW  PIC X(5).
               10  TC-CODE     PIC X(6).
               10  TC-BALANCE  PIC S9(7)V99.
       01  WS-IDX  PIC 9(1).

       PROCEDURE DIVISION.
           MOVE "Somchai        " TO TC-NAME(1).
           MOVE "30   "           TO TC-AGE-RAW(1).
           MOVE "AC1234"          TO TC-CODE(1).
           MOVE -500.00           TO TC-BALANCE(1).

           MOVE "John99         " TO TC-NAME(2).
           MOVE "-5   "           TO TC-AGE-RAW(2).
           MOVE "12FGXY"          TO TC-CODE(2).
           MOVE 1000.00           TO TC-BALANCE(2).

           MOVE "Mary           " TO TC-NAME(3).
           MOVE "abc  "           TO TC-AGE-RAW(3).
           MOVE "AB5678"          TO TC-CODE(3).
           MOVE 0.00              TO TC-BALANCE(3).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               MOVE 0 TO WS-ERROR-COUNT
               MOVE "Y" TO WS-VALID-FLAG
               MOVE TC-NAME(WS-IDX)    TO WS-CUST-NAME
               MOVE TC-AGE-RAW(WS-IDX) TO WS-CUST-AGE-RAW
               MOVE TC-CODE(WS-IDX)    TO WS-ACCOUNT-CODE
               MOVE TC-BALANCE(WS-IDX) TO WS-BALANCE

               DISPLAY "=== RECORD " WS-IDX " ==="

               IF WS-CUST-NAME IS NOT ALPHABETIC
                   DISPLAY "  ERROR: NAME MUST BE LETTERS ONLY"
                   ADD 1 TO WS-ERROR-COUNT
                   MOVE "N" TO WS-VALID-FLAG
               END-IF

               IF FUNCTION TRIM(WS-CUST-AGE-RAW) IS NOT NUMERIC
                   DISPLAY "  ERROR: AGE MUST BE NUMERIC"
                   ADD 1 TO WS-ERROR-COUNT
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-CUST-AGE-RAW TO WS-CUST-AGE
                   IF WS-CUST-AGE IS NOT POSITIVE
                       DISPLAY "  ERROR: AGE MUST BE POSITIVE"
                       ADD 1 TO WS-ERROR-COUNT
                       MOVE "N" TO WS-VALID-FLAG
                   END-IF
               END-IF

               IF WS-ACCOUNT-CODE(1:2) IS NOT ALPHABETIC-UPPER
                   DISPLAY "  ERROR: CODE PREFIX MUST BE UPPERCASE"
                   ADD 1 TO WS-ERROR-COUNT
                   MOVE "N" TO WS-VALID-FLAG
               END-IF

               EVALUATE TRUE
                   WHEN WS-BALANCE IS NEGATIVE
                       DISPLAY "  STATUS: ACCOUNT OVERDRAWN"
                   WHEN WS-BALANCE IS ZERO
                       DISPLAY "  STATUS: ACCOUNT EMPTY"
                   WHEN OTHER
                       DISPLAY "  STATUS: ACCOUNT HAS FUNDS"
               END-EVALUATE

               IF WS-VALID-FLAG = "Y"
                   DISPLAY "  RESULT: RECORD IS VALID"
               ELSE
                   DISPLAY "  RESULT: RECORD HAS " WS-ERROR-COUNT
                           " ERROR(S)"
               END-IF
           END-PERFORM.

           STOP RUN.
```

### อธิบายโค้ด

- ใช้ตาราง `TEST-CASE-TABLE` (Part 016) เก็บ 3 record ทดสอบที่ออกแบบให้ครอบคลุมทุกกรณี:
  record 1 ถูกต้องทั้งหมด, record 2 ผิดหลายกฎพร้อมกัน, record 3 ผิดเฉพาะอายุ
- แต่ละรอบของลูป `PERFORM VARYING` ตรวจสอบครบ 3 กฎ (`IS NOT ALPHABETIC`, `FUNCTION
  TRIM(...) IS NOT NUMERIC` ร่วมกับ `IS NOT POSITIVE`, `IS NOT ALPHABETIC-UPPER`) แล้วรายงาน
  สถานะบัญชีด้วย `EVALUATE TRUE` กับ Sign Condition
- สังเกตว่า record 2 มี `TC-AGE-RAW` เป็น `"-5   "` — เมื่อ `FUNCTION TRIM` ตัดช่องว่างเหลือ
  `"-5"` แล้วตรวจสอบ `IS NUMERIC` จะได้ **NOT NUMERIC** (ตามข้อค้นพบขั้นตอนที่ 411 ว่าเครื่องหมาย
  ลบตัวอักษรใน `PIC X` ไม่นับเป็นตัวเลขที่ถูกต้อง) ไม่ใช่เพราะ "เป็นตัวเลขติดลบที่ไม่ควรยอมรับ" —
  เป็นตัวอย่างที่ดีว่าทำไมต้องเข้าใจพฤติกรรมจริงของ `IS NUMERIC` อย่างละเอียดตามที่เรียนในขั้นตอนที่
  411 และ 415

### ผลลัพธ์ที่ได้จากการรันจริง

```
=== RECORD 1 ===
  STATUS: ACCOUNT OVERDRAWN
  RESULT: RECORD IS VALID
=== RECORD 2 ===
  ERROR: NAME MUST BE LETTERS ONLY
  ERROR: AGE MUST BE NUMERIC
  ERROR: CODE PREFIX MUST BE UPPERCASE
  STATUS: ACCOUNT HAS FUNDS
  RESULT: RECORD HAS 03 ERROR(S)
=== RECORD 3 ===
  ERROR: AGE MUST BE NUMERIC
  STATUS: ACCOUNT EMPTY
  RESULT: RECORD HAS 01 ERROR(S)
```

ผลลัพธ์ตรงตามที่ออกแบบไว้ทุกประการ: Record 1 ผ่านทุกกฎ (แม้ยอดเงินติดลบ แต่นั่นเป็นสถานะธุรกิจ
ไม่ใช่ข้อผิดพลาด), Record 2 ผิด 3 กฎพร้อมกัน (ชื่อมีตัวเลขปน "John99", อายุ "-5" ไม่ใช่ NUMERIC
ตามกฎ PIC X, รหัสบัญชีขึ้นต้นด้วยตัวเลข "12"), Record 3 ผิดเฉพาะอายุ ("abc" ไม่ใช่ตัวเลข)

### ข้อควรระวัง

- โปรแกรมนี้เป็นตัวอย่าง**โครงสร้างมาตรฐาน**ของโมดูลตรวจสอบข้อมูล (validation module) ที่จะพบ
  บ่อยมากในโปรแกรมธุรกิจจริง: ตรวจครบทุกกฎ, นับจำนวนข้อผิดพลาด, และรายงานผลสรุปท้ายสุด — ควร
  นำรูปแบบนี้ไปประยุกต์ใช้กับการตรวจสอบข้อมูลจริงในโปรเจกต์ของคุณเอง
- ในระบบจริง ควรบันทึกรายละเอียดข้อผิดพลาดแต่ละอย่างไว้ (เช่น ในไฟล์ error log ตามที่จะเรียนใน
  Part 048 เรื่อง Error Handling ขั้นสูง) แทนที่จะแค่ `DISPLAY` ออกหน้าจอเฉย ๆ เหมือนตัวอย่างนี้

### แบบฝึกหัดที่ 420.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น โดยเพิ่มตัวนับ `WS-VALID-RECORD-COUNT` และ
`WS-INVALID-RECORD-COUNT` ที่นับจำนวน record ที่ผ่าน/ไม่ผ่านการตรวจสอบทั้งหมด แล้วแสดงสรุป
ท้ายโปรแกรมหลังลูปจบ

**เฉลยแนวทาง**:
```cobol
       01  WS-VALID-RECORD-COUNT   PIC 9(2) VALUE 0.
       01  WS-INVALID-RECORD-COUNT PIC 9(2) VALUE 0.
       ...
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               ...
               IF WS-VALID-FLAG = "Y"
                   DISPLAY "  RESULT: RECORD IS VALID"
                   ADD 1 TO WS-VALID-RECORD-COUNT
               ELSE
                   DISPLAY "  RESULT: RECORD HAS " WS-ERROR-COUNT
                           " ERROR(S)"
                   ADD 1 TO WS-INVALID-RECORD-COUNT
               END-IF
           END-PERFORM.

           DISPLAY "=== SUMMARY ===".
           DISPLAY "VALID RECORDS  : " WS-VALID-RECORD-COUNT.
           DISPLAY "INVALID RECORDS: " WS-INVALID-RECORD-COUNT.
```
ด้วยข้อมูลทดสอบ 3 record ในตัวอย่างนี้ ผลลัพธ์ที่ควรได้คือ VALID RECORDS = 1 และ INVALID
RECORDS = 2 ตรงตามผลลัพธ์ที่ทดสอบไว้ในขั้นตอนหลัก

---

## สรุปท้ายบท

ใน Part นี้ เราได้เจาะลึก **Class Condition** และ **Sign Condition** อย่างครบถ้วน โดยทดสอบ
คอมไพล์และรันจริงทุกตัวอย่างด้วย GnuCOBOL 4.0 (early-dev) รวมถึงค้นพบพฤติกรรมจริงที่สำคัญ
หลายอย่างที่ไม่ได้กล่าวถึงชัดเจนในเอกสารทั่วไป:

- `IS NUMERIC` แบบเจาะลึก และ**ข้อค้นพบสำคัญ**ว่าช่องว่างในฟิลด์ `PIC X` ทำให้เป็นเท็จเสมอ
- `IS ALPHABETIC`/`ALPHABETIC-UPPER`/`ALPHABETIC-LOWER` และความไม่สมมาตรกับช่องว่างเทียบ
  กับ `IS NUMERIC`
- Sign Condition (`IS POSITIVE`/`NEGATIVE`/`ZERO`) กับตัวแปรและนิพจน์ทางคณิตศาสตร์
- `NOT` กับ Class/Sign Condition และรูปแบบ Validation Loop ที่ใช้บ่อยในโปรแกรมจริง
- การวิเคราะห์เจาะลึกกับดักช่องว่าง พร้อมทดสอบวิธีแก้ 4 แบบเทียบกัน (2 แบบใช้ได้จริง, 2 แบบใช้ไม่ได้)
- Class Condition ที่กำหนดเองผ่าน `CLASS` clause ใน `SPECIAL-NAMES`
- Sign Condition กับ `DIVIDE REMAINDER` และพฤติกรรมเครื่องหมายที่อาจไม่คาดคิด
- การผสาน Class Condition และ Sign Condition ในโปรแกรมตรวจสอบข้อมูลลูกค้าเดียว
- ตารางสรุปกับดักทั้งหมดที่ทดสอบยืนยันแล้วจริงตลอด Part นี้
- โปรแกรมรวบยอดโมดูลตรวจสอบข้อมูลลูกค้าแบบครบวงจรที่ทดสอบยืนยันผลลัพธ์ครบทั้ง 3 record

หลักการสำคัญที่สุดที่ควรนำติดตัวไปจาก Part นี้คือ **Class Condition ตรวจสอบรูปแบบเท่านั้น ไม่ตรวจสอบ
ความสมเหตุสมผลทางธุรกิจ** และ **ต้องใช้ `FUNCTION TRIM` ควบคู่กับ `IS NUMERIC` เสมอเมื่อตรวจสอบ
ข้อมูลจาก `ACCEPT`** — ทั้งสองหลักการนี้จะช่วยป้องกันบั๊กที่พบบ่อยที่สุดในโค้ด COBOL ของทั้งมือใหม่
และมืออาชีพ

Part นี้ปิดท้ายเฟส 3 ส่วนที่เกี่ยวกับเงื่อนไขและการควบคุมโครงสร้างขั้นสูงของหลักสูตร ใน **Part 043**
เราจะเปลี่ยนไปสำรวจหัวข้อใหม่ที่เชื่อมโยงกับ Subprogram ที่เรียนไปแล้วใน Part 031-032 คือ
**Dynamic CALL และ Program Pointers** ซึ่งเปิดโอกาสให้โปรแกรม COBOL เรียกใช้ subprogram ที่
กำหนดชื่อได้แบบไดนามิกตอนรันไทม์ แทนที่จะต้องระบุชื่อโปรแกรมตายตัวตอน compile-time เหมือนที่
เรียนมาก่อนหน้า

**[← กลับไป Part 041](part-041-advanced-evaluate.md)** | **[ไปยัง Part 043: Dynamic CALL และ Program Pointers →](part-043-dynamic-call.md)**
