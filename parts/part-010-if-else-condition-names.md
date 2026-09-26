# Part 010: เงื่อนไข IF-ELSE และ Condition Names (88-level) (ขั้นตอนที่ 91–100)

## คำนำของ Part นี้

จน Part 009 โปรแกรมของเราทำงานแบบ **ลำดับ (Sequence)** เท่านั้น คือทำคำสั่งตามลำดับจากบนลงล่างเสมอ
โดยไม่มีการ "ตัดสินใจ" ใด ๆ แต่โปรแกรมจริงในโลกธุรกิจแทบทุกโปรแกรมต้องมีการตัดสินใจ เช่น
"ถ้าลูกค้าอายุต่ำกว่า 18 ปี ห้ามซื้อ", "ถ้ายอดสั่งซื้อเกิน 1000 บาท ให้ส่วนลด 10%" เป็นต้น

Part นี้จะพาคุณเรียนรู้แนวคิด **Selection (การเลือกทำ)** ซึ่งเป็นหนึ่งใน 4 แนวคิดหลักของ Procedural
Programming ที่เรากล่าวถึงใน Part 001 ผ่านคำสั่ง `IF...ELSE` และแนะนำเทคนิคขั้นสูงที่ทำให้โค้ด
เงื่อนไขอ่านง่ายเหมือนภาษาอังกฤษยิ่งขึ้นไปอีก นั่นคือ **Condition Names หรือ 88-level** ซึ่งเป็น
เอกลักษณ์เฉพาะตัวของ COBOL ที่ภาษาโปรแกรมอื่นแทบไม่มี

---

## ขั้นตอนที่ 91: คำสั่ง IF พื้นฐาน

### แนวคิด

รูปแบบพื้นฐานที่สุดของ `IF` คือ:

```
IF <condition>
    <statement(s)>
END-IF
```

ถ้าเงื่อนไข (`condition`) เป็นจริง คำสั่งภายในจะถูกประมวลผล ถ้าเป็นเท็จ โปรแกรมจะข้ามไปทำงาน
หลัง `END-IF` ทันที `END-IF` คือ **scope terminator** ที่บอกขอบเขตชัดเจนว่า IF นี้จบตรงไหน
(เพิ่มเข้ามาตั้งแต่มาตรฐาน COBOL-85 ตามที่กล่าวถึงใน Part 001)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IF-BASIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AGE              PIC 9(3)  VALUE 20.
       01  WS-SCORE            PIC 9(3)  VALUE 45.

       PROCEDURE DIVISION.
           IF WS-AGE >= 18
               DISPLAY "YOU ARE AN ADULT."
           END-IF

           IF WS-SCORE < 50
               DISPLAY "YOU DID NOT PASS THE EXAM."
           END-IF

           DISPLAY "PROGRAM CONTINUES AFTER THE IF STATEMENTS."
           STOP RUN.
```

### อธิบายโค้ด

- `IF WS-AGE >= 18` — ตรวจสอบว่า `WS-AGE` (20) มากกว่าหรือเท่ากับ 18 หรือไม่ ในกรณีนี้เป็นจริง
  จึงแสดงข้อความ "YOU ARE AN ADULT."
- `IF WS-SCORE < 50` — ตรวจสอบว่า `WS-SCORE` (45) น้อยกว่า 50 หรือไม่ เป็นจริงเช่นกัน จึงแสดง
  ข้อความสอบไม่ผ่าน
- ทั้งสอง IF ไม่มี `ELSE` ดังนั้นถ้าเงื่อนไขเป็นเท็จ โปรแกรมจะข้ามไปโดยไม่ทำอะไรเลย แล้วไปทำงาน
  บรรทัดถัดไปต่อ (ในที่นี้คือ DISPLAY สุดท้ายที่ทำงานเสมอไม่ว่าเงื่อนไขก่อนหน้าจะเป็นอย่างไร)

### ผลลัพธ์ที่ได้จากการรันจริง

```
YOU ARE AN ADULT.
YOU DID NOT PASS THE EXAM.
PROGRAM CONTINUES AFTER THE IF STATEMENTS.
```

### ข้อควรระวัง

- **`END-IF` ไม่ใช่คำสั่งบังคับตามมาตรฐานเก่า** (สามารถใช้ period `.` แทนได้) แต่หลักสูตรนี้แนะนำ
  ให้ใช้ `END-IF` เสมอเพื่อความชัดเจน และเพื่อหลีกเลี่ยงกับดักที่เราจะเจาะลึกใน ขั้นตอนที่ 95
- ตัวดำเนินการเปรียบเทียบ (`>=`, `<`, `=` ฯลฯ) ต้องเขียนติดกับตัวแปรทั้งสองข้างด้วยช่องว่างเสมอ
  เพื่อความอ่านง่ายและป้องกัน syntax error

### แบบฝึกหัดที่ 91.1

**โจทย์**: จงเขียนคำสั่ง IF เพื่อแสดงข้อความ "TEMPERATURE IS HIGH." เมื่อตัวแปร `WS-TEMP` มีค่า
มากกว่า 35

**เฉลย**:
```cobol
IF WS-TEMP > 35
    DISPLAY "TEMPERATURE IS HIGH."
END-IF
```

---

## ขั้นตอนที่ 92: IF-ELSE และการซ้อนเงื่อนไข (Nested IF)

### แนวคิด

เมื่อต้องการให้โปรแกรมทำอย่างหนึ่งเมื่อเงื่อนไขเป็นจริง และทำอีกอย่างหนึ่งเมื่อเป็นเท็จ ให้ใช้
`ELSE`:

```
IF <condition>
    <statement(s)-if-true>
ELSE
    <statement(s)-if-false>
END-IF
```

และเมื่อมีหลายเงื่อนไขที่ต้องตรวจสอบต่อเนื่องกัน (เช่น ให้เกรดตามช่วงคะแนน) เราสามารถ **ซ้อน IF
ไว้ภายใน ELSE ของ IF อีกตัว** ได้ เรียกว่า **Nested IF** — เทคนิคนี้เป็นพื้นฐานสำคัญก่อนที่เราจะ
เรียนรู้ทางเลือกที่กระชับกว่าคือ `EVALUATE` ใน Part 011

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IF-ELSE-NESTED-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SCORE            PIC 9(3)  VALUE 78.
       01  WS-GRADE            PIC X(1).

       PROCEDURE DIVISION.
           IF WS-SCORE >= 50
               DISPLAY "RESULT: PASS"
           ELSE
               DISPLAY "RESULT: FAIL"
           END-IF

      *> Nested IF: an IF statement inside another IF branch
           IF WS-SCORE >= 80
               MOVE "A" TO WS-GRADE
           ELSE
               IF WS-SCORE >= 70
                   MOVE "B" TO WS-GRADE
               ELSE
                   IF WS-SCORE >= 60
                       MOVE "C" TO WS-GRADE
                   ELSE
                       MOVE "F" TO WS-GRADE
                   END-IF
               END-IF
           END-IF

           DISPLAY "SCORE " WS-SCORE " GETS GRADE " WS-GRADE
           STOP RUN.
```

### อธิบายโค้ด

- IF ตัวแรกตรวจสอบว่าสอบผ่านหรือไม่ (`WS-SCORE >= 50`) — 78 ผ่านเกณฑ์ จึงแสดง "RESULT: PASS"
- IF ตัวที่สองเป็น nested IF สำหรับหาเกรด: ตรวจสอบจากเงื่อนไขที่สูงสุดก่อน (`>= 80`) ถ้าไม่ผ่าน
  จึงลงไปตรวจเงื่อนไขถัดไปใน `ELSE` (`>= 70`) เรื่อย ๆ จนกว่าจะเจอเงื่อนไขที่เป็นจริง
- สังเกตว่าแต่ละ `IF` ที่ซ้อนกันต้องมี `END-IF` ของตัวเองครบทุกตัว การนับ `END-IF` ให้ครบเท่ากับ
  จำนวน `IF` ที่เปิดไว้เป็นทักษะสำคัญเมื่อโค้ดซ้อนกันหลายชั้น
- เนื่องจาก 78 ไม่ >= 80 แต่ >= 70 จึงได้เกรด "B"

### ผลลัพธ์ที่ได้จากการรันจริง

```
RESULT: PASS
SCORE 078 GETS GRADE B
```

### ข้อควรระวัง

- **Nested IF ที่ซ้อนกันลึกเกินไป (deep nesting) ทำให้โค้ดอ่านยากขึ้นเรื่อย ๆ** ยิ่งมีเงื่อนไข
  มากเท่าไร ยิ่งต้องนับ `END-IF` ให้ครบและเยื้อง (indent) ให้ถูกต้อง หากมีมากกว่า 3-4 ชั้น
  ควรพิจารณาใช้ `EVALUATE` แทน (จะสอนใน Part 011)
- ลืมใส่ `END-IF` แม้แต่ตัวเดียวจะทำให้คอมไพเลอร์ error หรือแย่กว่านั้นคือ ELSE ตัวถัดไปไปจับคู่กับ
  IF ผิดตัว (ปัญหาที่เรียกว่า "dangling else") ทำให้ตรรกะผิดพลาดโดยไม่มี error แจ้งเตือน

### แบบฝึกหัดที่ 92.1

**โจทย์**: จงเขียน Nested IF เพื่อจำแนกตัวเลขในตัวแปร `WS-NUM` ว่าเป็น "POSITIVE", "NEGATIVE"
หรือ "ZERO"

**เฉลย**:
```cobol
IF WS-NUM > 0
    DISPLAY "POSITIVE"
ELSE
    IF WS-NUM < 0
        DISPLAY "NEGATIVE"
    ELSE
        DISPLAY "ZERO"
    END-IF
END-IF
```

---

## ขั้นตอนที่ 93: ตัวดำเนินการเปรียบเทียบและ Class Condition

### แนวคิด

COBOL รองรับตัวดำเนินการเปรียบเทียบ (relational operators) ครบถ้วนทั้งแบบสัญลักษณ์และแบบคำ:

| สัญลักษณ์ | คำอ่าน | ความหมาย |
|---|---|---|
| `=` | IS EQUAL TO | เท่ากับ |
| `<` | IS LESS THAN | น้อยกว่า |
| `>` | IS GREATER THAN | มากกว่า |
| `<=` | IS LESS THAN OR EQUAL TO | น้อยกว่าหรือเท่ากับ |
| `>=` | IS GREATER THAN OR EQUAL TO | มากกว่าหรือเท่ากับ |
| `NOT =` | IS NOT EQUAL TO | ไม่เท่ากับ |

นอกจากนี้ยังมี **Class Condition** สำหรับตรวจสอบ "ชนิด" ของข้อมูลที่เก็บอยู่ในฟิลด์ว่าเป็นตัวเลข
ล้วน (`IS NUMERIC`) หรือตัวอักษรล้วน (`IS ALPHABETIC`) หรือไม่ ซึ่งมีประโยชน์มากในการตรวจสอบ
ความถูกต้องของข้อมูลก่อนนำไปประมวลผล (ย้อนกลับไปดูกับดักใน Part 008 ขั้นตอนที่ 79 ที่การ MOVE
ข้อมูลไม่ใช่ตัวเลขเข้าฟิลด์ตัวเลขทำให้เกิดปัญหา — Class Condition คือเครื่องมือที่ใช้ป้องกันปัญหานั้น)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IF-OPERATORS-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A                PIC 9(3)  VALUE 10.
       01  WS-B                PIC 9(3)  VALUE 20.
       01  WS-NAME              PIC X(5) VALUE "AAAAA".
       01  WS-VALUE             PIC X(5) VALUE "12345".

       PROCEDURE DIVISION.
           IF WS-A = 10
               DISPLAY "A EQUALS 10"
           END-IF

           IF WS-A NOT = WS-B
               DISPLAY "A IS NOT EQUAL TO B"
           END-IF

           IF WS-A < WS-B
               DISPLAY "A IS LESS THAN B"
           END-IF

           IF WS-B > WS-A
               DISPLAY "B IS GREATER THAN A"
           END-IF

           IF WS-A <= 10
               DISPLAY "A IS LESS THAN OR EQUAL TO 10"
           END-IF

      *> Class conditions: check the kind of data in a field
           IF WS-NAME IS ALPHABETIC
               DISPLAY "WS-NAME CONTAINS ONLY LETTERS"
           END-IF

           IF WS-VALUE IS NUMERIC
               DISPLAY "WS-VALUE CONTAINS ONLY DIGITS"
           END-IF
           STOP RUN.
```

### อธิบายโค้ด

- ตัวดำเนินการเปรียบเทียบทั้งหมดทำงานตามความหมายปกติทางคณิตศาสตร์
- `WS-NAME IS ALPHABETIC` — ตรวจสอบว่า `WS-NAME` ("AAAAA") มีแต่ตัวอักษร A-Z และช่องว่างเท่านั้น
  หรือไม่ ในที่นี้เป็นจริง
- `WS-VALUE IS NUMERIC` — ตรวจสอบว่า `WS-VALUE` ("12345") ประกอบด้วยตัวเลข 0-9 ล้วนหรือไม่
  แม้ `WS-VALUE` จะประกาศเป็น `PIC X(5)` (ฟิลด์ตัวอักษร) ก็ยังสามารถตรวจสอบได้ว่าเนื้อหาข้างในเป็น
  ตัวเลขล้วนหรือไม่ นี่คือประโยชน์สำคัญของ Class Condition สำหรับตรวจสอบข้อมูลนำเข้าจากผู้ใช้
  (ที่มักรับเป็นฟิลด์ตัวอักษรก่อนแล้วค่อยตรวจสอบ)

### ผลลัพธ์ที่ได้จากการรันจริง

```
A EQUALS 10
A IS NOT EQUAL TO B
A IS LESS THAN B
B IS GREATER THAN A
A IS LESS THAN OR EQUAL TO 10
WS-NAME CONTAINS ONLY LETTERS
WS-VALUE CONTAINS ONLY DIGITS
```

### ข้อควรระวัง

- `IS NUMERIC` ตรวจสอบเฉพาะว่าเนื้อหาเป็น**ตัวเลขที่ถูกต้อง**หรือไม่ (รวมเครื่องหมายบวก/ลบถ้าเป็น
  signed field) แต่ไม่ได้ตรวจสอบว่าค่านั้นอยู่ในช่วงที่ธุรกิจต้องการหรือไม่ (เช่น อายุ -5 ปี
  ก็ยังถือว่า `IS NUMERIC` เป็นจริง เพราะเป็นตัวเลขที่ถูกต้องทางไวยากรณ์)
- **แนวปฏิบัติที่ดี**: ควรตรวจสอบด้วย `IS NUMERIC` ก่อนทุกครั้งที่จะนำฟิลด์ตัวอักษร (ที่รับข้อมูล
  จากผู้ใช้หรือไฟล์ภายนอก) ไปใช้ในการคำนวณหรือ MOVE เข้าฟิลด์ตัวเลข เพื่อป้องกันปัญหาที่กล่าวถึง
  ใน Part 008 ขั้นตอนที่ 79

### แบบฝึกหัดที่ 93.1

**โจทย์**: จงเขียนคำสั่ง IF เพื่อแสดงข้อความ "INVALID INPUT" เมื่อตัวแปร `WS-INPUT PIC X(5)`
มีเนื้อหาที่ไม่ใช่ตัวเลขล้วน

**เฉลย**:
```cobol
IF WS-INPUT IS NOT NUMERIC
    DISPLAY "INVALID INPUT"
END-IF
```

---

## ขั้นตอนที่ 94: เงื่อนไขผสม — AND, OR, NOT

### แนวคิด

เมื่อการตัดสินใจต้องอาศัยหลายเงื่อนไขพร้อมกัน COBOL ให้เราเชื่อมเงื่อนไขด้วย:

- **`AND`** — เงื่อนไขทั้งสองฝั่งต้องเป็นจริงพร้อมกันทั้งคู่
- **`OR`** — เงื่อนไขฝั่งใดฝั่งหนึ่งเป็นจริงก็เพียงพอ
- **`NOT`** — กลับค่าความจริงของเงื่อนไข (จริงเป็นเท็จ, เท็จเป็นจริง)

ลำดับความสำคัญ (precedence) ของตัวเชื่อมเหล่านี้คือ `NOT` มาก่อน ตามด้วย `AND` แล้วจึง `OR`
เหมือนคณิตศาสตร์บูลีนทั่วไป และสามารถใช้ **วงเล็บ** เพื่อบังคับลำดับการประเมินผลได้เช่นเดียวกับ
สมการคณิตศาสตร์ใน `COMPUTE`

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IF-COMPOUND-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AGE              PIC 9(3)  VALUE 25.
       01  WS-HAS-LICENSE      PIC X(1)  VALUE "Y".
       01  WS-COUNTRY          PIC X(2)  VALUE "TH".

       PROCEDURE DIVISION.
           IF WS-AGE >= 18 AND WS-HAS-LICENSE = "Y"
               DISPLAY "ALLOWED TO DRIVE."
           ELSE
               DISPLAY "NOT ALLOWED TO DRIVE."
           END-IF

           IF WS-COUNTRY = "TH" OR WS-COUNTRY = "LA"
               DISPLAY "ELIGIBLE FOR REGIONAL PROMOTION."
           END-IF

           IF NOT (WS-AGE < 18)
               DISPLAY "CONFIRMED: NOT A MINOR."
           END-IF

      *> Combining AND, OR and NOT together (parentheses matter!)
           IF (WS-AGE >= 18 AND WS-HAS-LICENSE = "Y")
               OR WS-COUNTRY = "TH"
               DISPLAY "PASSES THE COMBINED CONDITION."
           END-IF
           STOP RUN.
```

### อธิบายโค้ด

- `WS-AGE >= 18 AND WS-HAS-LICENSE = "Y"` — ต้องอายุครบ 18 ปี**และ**มีใบขับขี่ ทั้งสองเงื่อนไข
  เป็นจริงพร้อมกัน (25 >= 18 และ "Y" = "Y") จึงแสดง "ALLOWED TO DRIVE."
- `WS-COUNTRY = "TH" OR WS-COUNTRY = "LA"` — เงื่อนไขใดเงื่อนไขหนึ่งเป็นจริงก็พอ ในที่นี้
  `WS-COUNTRY` เป็น "TH" ตรงกับเงื่อนไขแรก จึงเป็นจริง
- `NOT (WS-AGE < 18)` — กลับค่าความจริงของ "อายุน้อยกว่า 18" ซึ่งเป็นเท็จ (25 ไม่น้อยกว่า 18)
  เมื่อกลับค่าด้วย NOT จึงกลายเป็นจริง
- เงื่อนไขสุดท้ายรวม AND, OR เข้าด้วยกันโดยใช้วงเล็บระบุลำดับชัดเจนว่ากลุ่ม AND ต้องคำนวณก่อน
  แล้วจึงนำผลไป OR กับเงื่อนไขที่สาม

### ผลลัพธ์ที่ได้จากการรันจริง

```
ALLOWED TO DRIVE.
ELIGIBLE FOR REGIONAL PROMOTION.
CONFIRMED: NOT A MINOR.
PASSES THE COMBINED CONDITION.
```

### ข้อควรระวัง

- **การลืมใส่วงเล็บในเงื่อนไขผสมที่ซับซ้อน** เป็นสาเหตุอันดับต้น ๆ ของบั๊กเชิงตรรกะ เพราะแม้ผลลัพธ์
  จะดู "compile ผ่าน" แต่ผลการประเมินอาจไม่ตรงกับที่ผู้เขียนตั้งใจ ควรใส่วงเล็บชัดเจนเสมอเมื่อผสม
  AND กับ OR ในเงื่อนไขเดียวกัน แม้จะรู้ลำดับความสำคัญดีแล้วก็ตาม เพื่อให้คนอื่นอ่านโค้ดเข้าใจง่าย
- `NOT` วางไว้หน้าเงื่อนไขที่ต้องการกลับค่า และหากเงื่อนไขนั้นซับซ้อนต้องครอบด้วยวงเล็บเสมอ
  (`NOT (A AND B)` ต่างจาก `NOT A AND B` โดยสิ้นเชิง)

### แบบฝึกหัดที่ 94.1

**โจทย์**: จงเขียนเงื่อนไขตรวจสอบว่าลูกค้ามีสิทธิ์รับส่วนลด VIP หรือไม่ โดยเงื่อนไขคือ
"เป็นสมาชิกมาแล้วอย่างน้อย 3 ปี **และ** (ยอดซื้อสะสมเกิน 50000 บาท **หรือ** เป็นสมาชิกระดับ Gold)"

**เฉลย**:
```cobol
IF WS-MEMBER-YEARS >= 3
    AND (WS-TOTAL-PURCHASE > 50000 OR WS-MEMBER-LEVEL = "GOLD")
    DISPLAY "ELIGIBLE FOR VIP DISCOUNT."
END-IF
```

---

## ขั้นตอนที่ 95: กับดักของ Scope Terminator — END-IF กับ Period

### แนวคิด

ใน Part 001 เราเรียนรู้ว่า COBOL-85 เพิ่ม scope terminator (`END-IF`, `END-PERFORM` ฯลฯ) เข้ามา
เพื่อแก้ปัญหาความคลุมเครือของการใช้ period (`.`) ปิดท้ายประโยคแบบเดิม ขั้นตอนนี้จะสาธิตให้เห็น
**อันตรายจริง** ของการใช้ period แทน END-IF ในโค้ดที่มีหลายคำสั่งซ้อนกันภายใน IF

กฎสำคัญที่ต้องจำ: **period จะปิด IF (หรือ PERFORM ฯลฯ) ที่เปิดอยู่ทันทีที่เจอ** ไม่ว่าตำแหน่งที่
period ปรากฏจะอยู่กลางกลุ่มคำสั่งหรือไม่ก็ตาม ทำให้คำสั่งที่ตามหลัง period (แต่ยังอยู่ในบล็อกเดิม
ตามที่ผู้เขียนตั้งใจ) กลับกลายเป็นคำสั่งที่ **ทำงานแบบไม่มีเงื่อนไข (unconditional)**

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IF-SCOPE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SCORE            PIC 9(3)  VALUE 90.
       01  WS-BONUS            PIC 9(3)  VALUE 0.

       PROCEDURE DIVISION.
      *> Using END-IF (explicit scope terminator, recommended)
           IF WS-SCORE >= 90
               DISPLAY "EXCELLENT SCORE!"
               MOVE 10 TO WS-BONUS
           END-IF
           DISPLAY "BONUS AFTER CLEAR SCOPE: " WS-BONUS

      *> DANGER: a stray period ends the IF right there. Below,
      *> the period after DISPLAY closes the IF early, so the
      *> MOVE statement runs UNCONDITIONALLY -- even though the
      *> score is only 60 and should not count as excellent!
           MOVE 60 TO WS-SCORE
           MOVE 0 TO WS-BONUS
           IF WS-SCORE >= 90
               DISPLAY "EXCELLENT SCORE AGAIN!".
               MOVE 10 TO WS-BONUS
           DISPLAY "BUG: BONUS GIVEN EVEN THOUGH SCORE IS "
               WS-SCORE ": " WS-BONUS
           STOP RUN.
```

### อธิบายโค้ด

- ส่วนแรกใช้ `END-IF` อย่างถูกต้อง: `WS-SCORE` (90) >= 90 เป็นจริง ทั้ง DISPLAY และ MOVE จึงทำงาน
  ภายในขอบเขตของ IF ชัดเจน ผลลัพธ์คือ `WS-BONUS` = 10 ถูกต้องตามที่ตั้งใจ
- ส่วนที่สองคือ**กับดักจริง**: หลัง `MOVE 60 TO WS-SCORE` เงื่อนไข `WS-SCORE >= 90` เป็น **เท็จ**
  (60 ไม่ >= 90) ตามที่ควรจะเป็น การแสดงผล "EXCELLENT SCORE AGAIN!" จึงไม่ทำงาน (ถูกต้อง)
  **แต่**เนื่องจากมี period (`.`) ต่อท้าย `DISPLAY "EXCELLENT SCORE AGAIN!".` แทนที่จะเป็น END-IF
  period ตัวนี้ปิด IF ทั้งบล็อกทันที ทำให้บรรทัด `MOVE 10 TO WS-BONUS` **หลุดออกจากขอบเขตของ IF**
  กลายเป็นคำสั่งที่ทำงานเสมอไม่ว่าเงื่อนไขจะเป็นจริงหรือเท็จ
- ผลลัพธ์คือ `WS-BONUS` กลายเป็น 10 ทั้งที่คะแนนแค่ 60 ซึ่งไม่ควรได้โบนัสเลย — เป็นบั๊กที่ตรวจจับได้
  ยากมากเพราะคอมไพล์ผ่านปกติไม่มี error หรือ warning ใด ๆ

### ผลลัพธ์ที่ได้จากการรันจริง

```
EXCELLENT SCORE!
BONUS AFTER CLEAR SCOPE: 010
BUG: BONUS GIVEN EVEN THOUGH SCORE IS 060: 010
```

สังเกตบรรทัดสุดท้าย: โปรแกรมยืนยันชัดเจนว่า `WS-BONUS` เป็น 010 ทั้งที่คะแนนแค่ 060 — นี่คือบั๊ก
ที่เกิดจาก period ผิดตำแหน่งโดยตรง

### ข้อควรระวัง

- **นี่คือเหตุผลสำคัญที่สุดที่หลักสูตรนี้ (และมาตรฐานอุตสาหกรรมสมัยใหม่) แนะนำให้ใช้ `END-IF`
  เสมอ แทนที่จะพึ่งพา period ปิดท้ายบล็อกคำสั่งที่มีมากกว่า 1 บรรทัด** โดยเฉพาะเมื่อโค้ดมีการ
  ซ้อน IF หลายชั้นหรือมีหลายคำสั่งภายในบล็อกเดียวกัน
- ในโค้ด Legacy จริงที่เขียนก่อนมาตรฐาน COBOL-85 (ที่ยังไม่มี END-IF) บั๊กประเภทนี้พบได้บ่อยมาก
  และเป็นสาเหตุสำคัญที่การบำรุงรักษาโค้ดเก่า (legacy maintenance) ต้องใช้ความระมัดระวังสูง
  เมื่อคุณต้องทำงานกับโค้ด Legacy ในอนาคต ให้ตรวจสอบตำแหน่ง period ทุกจุดอย่างละเอียด

### แบบฝึกหัดที่ 95.1

**โจทย์**: จากโค้ดปัญหาข้างต้น จงแก้ไขให้ถูกต้องโดยใช้ `END-IF` แทน period หลัง DISPLAY

**เฉลย**:
```cobol
IF WS-SCORE >= 90
    DISPLAY "EXCELLENT SCORE AGAIN!"
    MOVE 10 TO WS-BONUS
END-IF
```
เมื่อแก้ไขแล้ว ถ้า `WS-SCORE` เป็น 60 ทั้งสองคำสั่งจะไม่ทำงานเลย เพราะอยู่ในขอบเขต IF เดียวกัน
อย่างถูกต้อง `WS-BONUS` จะยังคงเป็น 0 ตามที่ควรจะเป็น

---

## ขั้นตอนที่ 96: Condition Names (88-level) พื้นฐาน

### แนวคิด

**Condition Name** หรือที่เรียกกันทั่วไปว่า **88-level** เป็นเทคนิคเฉพาะของ COBOL ที่ให้เรา
**ตั้งชื่อที่มีความหมาย** ให้กับค่าที่เป็นไปได้ของตัวแปร แล้วนำชื่อนั้นไปใช้ในเงื่อนไข IF ได้โดยตรง
แทนที่จะต้องเขียนเปรียบเทียบค่าดิบ (raw value) ซ้ำ ๆ ทุกครั้ง

วิธีประกาศ: เขียน `88` ตามด้วยชื่อ condition-name และ `VALUE` ของค่าที่ตรงกับเงื่อนไขนั้น
ไว้ **ใต้** ตัวแปรหลักที่เกี่ยวข้อง (88-level ต้องประกาศตามหลังฟิลด์ระดับ 01-49 ที่เป็นเจ้าของเสมอ)

```
01  WS-STATUS           PIC X(1).
    88  CONDITION-NAME-1        VALUE "X".
    88  CONDITION-NAME-2        VALUE "Y".
```

จากนั้นใช้ `IF CONDITION-NAME-1` แทนการเขียน `IF WS-STATUS = "X"` — โค้ดจะอ่านง่ายขึ้นมากราวกับ
เป็นภาษาอังกฤษปกติ ตรงกับปรัชญาของ Grace Hopper ที่กล่าวถึงใน Part 001

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CONDITION-NAME-BASIC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-GENDER-CODE      PIC X(1)  VALUE "M".
           88  IS-MALE                   VALUE "M".
           88  IS-FEMALE                 VALUE "F".

       01  WS-ORDER-STATUS     PIC X(1)  VALUE "P".
           88  ORDER-IS-PENDING          VALUE "P".
           88  ORDER-IS-SHIPPED          VALUE "S".
           88  ORDER-IS-CANCELLED        VALUE "C".

       PROCEDURE DIVISION.
      *> Without condition-names, we would write:
      *>   IF WS-GENDER-CODE = "M"
      *> With condition-names, the code reads like English:
           IF IS-MALE
               DISPLAY "CUSTOMER IS MALE."
           END-IF

           IF ORDER-IS-PENDING
               DISPLAY "ORDER IS STILL PENDING."
           END-IF

           IF ORDER-IS-SHIPPED
               DISPLAY "THIS WILL NOT PRINT."
           END-IF
           STOP RUN.
```

### อธิบายโค้ด

- `88 IS-MALE VALUE "M"` — ประกาศว่า `IS-MALE` เป็นจริงเมื่อ `WS-GENDER-CODE` เท่ากับ "M"
  เท่านั้น 88-level ไม่ได้จองพื้นที่หน่วยความจำเพิ่มเติมแต่อย่างใด (ไม่เหมือนตัวแปรปกติ)
  มันเป็นเพียง "ชื่อเรียก" ของเงื่อนไขที่อ้างอิงถึงค่าของฟิลด์แม่ (`WS-GENDER-CODE`) เท่านั้น
- `IF IS-MALE` — อ่านง่ายกว่า `IF WS-GENDER-CODE = "M"` มาก และยังป้องกันการพิมพ์ผิดค่า "M" ซ้ำ ๆ
  หลายที่ในโปรแกรม (เขียนค่า "M" ไว้ที่เดียวตอนประกาศ 88-level เท่านั้น)
- `WS-ORDER-STATUS` มีค่าเป็น "P" (Pending) ดังนั้น `ORDER-IS-PENDING` เป็นจริง แต่
  `ORDER-IS-SHIPPED` เป็นเท็จ

### ผลลัพธ์ที่ได้จากการรันจริง

```
CUSTOMER IS MALE.
ORDER IS STILL PENDING.
```

### ข้อควรระวัง

- 88-level **ไม่ใช่ตัวแปร** ไม่สามารถ `MOVE` ค่าเข้าไปตรง ๆ ได้ (เช่น `MOVE IS-MALE TO ...`
  ทำไม่ได้) แต่สามารถใช้ `SET condition-name TO TRUE` เพื่อกำหนดค่าที่ถูกต้องให้ฟิลด์แม่แทนได้
  (จะสอนใน ขั้นตอนที่ 98)
- ชื่อ 88-level ต้องไม่ซ้ำกับชื่อตัวแปรอื่นในโปรแกรม และควรตั้งชื่อให้สื่อความหมายชัดเจนเพื่อ
  ประโยชน์สูงสุดของเทคนิคนี้ (เช่น `ORDER-IS-PENDING` ดีกว่า `FLAG-1`)

### แบบฝึกหัดที่ 96.1

**โจทย์**: จงประกาศ 88-level สำหรับตัวแปร `WS-PAYMENT-METHOD PIC X(1)` ที่มีค่าที่เป็นไปได้คือ
"C" (Cash), "K" (Credit Card), "T" (Transfer) พร้อมตั้งชื่อ condition-name ที่เหมาะสม

**เฉลย**:
```cobol
01  WS-PAYMENT-METHOD   PIC X(1).
    88  PAY-BY-CASH             VALUE "C".
    88  PAY-BY-CREDIT-CARD      VALUE "K".
    88  PAY-BY-TRANSFER         VALUE "T".
```

---

## ขั้นตอนที่ 97: Condition Names กับช่วงค่า (THRU) และหลายค่า

### แนวคิด

88-level ไม่ได้จำกัดแค่ค่าเดียว เราสามารถกำหนด **ช่วงของค่า** ด้วย `THRU` (หรือ `THROUGH`)
หรือกำหนด **หลายค่าที่ไม่ต่อเนื่องกัน** คั่นด้วยจุลภาคได้ ทำให้ 88-level มีประโยชน์มากในการจัดกลุ่ม
ค่าที่เกี่ยวข้องกันทางธุรกิจ เช่น ช่วงเกรด หรือกลุ่มวันในสัปดาห์

```
88  condition-name    VALUE low-value THRU high-value.
88  condition-name    VALUES value-1, value-2, value-3.
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CONDITION-NAME-THRU.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EXAM-SCORE       PIC 9(3)  VALUE 85.
           88  GRADE-A                   VALUE 90 THRU 100.
           88  GRADE-B                   VALUE 80 THRU 89.
           88  GRADE-C                   VALUE 70 THRU 79.
           88  GRADE-F                   VALUE 0 THRU 69.

       01  WS-DAY-NUMBER       PIC 9(1)  VALUE 6.
           88  IS-WEEKEND                VALUES 6, 7.
           88  IS-WEEKDAY                VALUES 1, 2, 3, 4, 5.

       PROCEDURE DIVISION.
           IF GRADE-A
               DISPLAY "GRADE: A"
           ELSE IF GRADE-B
               DISPLAY "GRADE: B"
           ELSE IF GRADE-C
               DISPLAY "GRADE: C"
           ELSE
               DISPLAY "GRADE: F"
           END-IF END-IF END-IF

           IF IS-WEEKEND
               DISPLAY "NO WORK TODAY, IT IS THE WEEKEND."
           ELSE
               DISPLAY "IT IS A WEEKDAY."
           END-IF
           STOP RUN.
```

### อธิบายโค้ด

- `88 GRADE-B VALUE 80 THRU 89` — `GRADE-B` เป็นจริงเมื่อ `WS-EXAM-SCORE` อยู่ในช่วง 80 ถึง 89
  (รวมทั้งสองค่าปลาย) ค่าปัจจุบันคือ 85 จึงทำให้ `GRADE-B` เป็นจริง
- `88 IS-WEEKEND VALUES 6, 7` — ใช้ `VALUES` (มี S) เมื่อระบุหลายค่าที่ไม่ต่อเนื่องกันคั่นด้วย
  จุลภาค `WS-DAY-NUMBER` เป็น 6 ตรงกับค่าแรกในรายการ จึงทำให้ `IS-WEEKEND` เป็นจริง
- โครงสร้าง `IF ... ELSE IF ... ELSE IF ... ELSE ... END-IF END-IF END-IF` เป็นรูปแบบการเขียน
  ที่กระชับกว่าการเยื้อง (indent) IF ซ้อนลึกลงไปเรื่อย ๆ ทุกตัว — สังเกตว่าต้องปิดด้วย `END-IF`
  ให้ครบเท่ากับจำนวน `IF` ที่เปิดไว้ทั้งหมด (ในที่นี้มี 3 ตัว)

### ผลลัพธ์ที่ได้จากการรันจริง

```
GRADE: B
NO WORK TODAY, IT IS THE WEEKEND.
```

### ข้อควรระวัง

- ช่วงค่าที่กำหนดด้วย `THRU` ในแต่ละ 88-level **ไม่ควรทับซ้อนกัน** มิเช่นนั้นค่าเดียวกันอาจทำให้
  หลาย condition-name เป็นจริงพร้อมกัน ซึ่งอาจสร้างความสับสนในการเขียนโปรแกรมภายหลัง (ในตัวอย่างนี้
  ช่วงคะแนนแต่ละเกรดถูกออกแบบให้ไม่ทับซ้อนกันอย่างรอบคอบแล้ว)
- รูปแบบ `ELSE IF` ต่อเนื่องกันหลายตัวพร้อมปิดด้วย `END-IF` รวมท้ายสุดนั้น แม้จะคอมไพล์ผ่านและ
  ทำงานถูกต้อง แต่ในทางปฏิบัติทีมพัฒนาหลายแห่งนิยมใช้ `EVALUATE` แทนเมื่อมีมากกว่า 3 เงื่อนไข
  เพื่อความชัดเจน (จะสอนใน Part 011)

### แบบฝึกหัดที่ 97.1

**โจทย์**: จงประกาศ 88-level ชื่อ `IS-VOWEL` สำหรับตัวแปร `WS-LETTER PIC X(1)` ที่เป็นจริงเมื่อ
ตัวอักษรเป็นสระ A, E, I, O หรือ U

**เฉลย**:
```cobol
01  WS-LETTER           PIC X(1).
    88  IS-VOWEL                 VALUES "A", "E", "I", "O", "U".
```

---

## ขั้นตอนที่ 98: SET condition-name TO TRUE — การกำหนดค่าผ่าน Condition Name

### แนวคิด

ในเมื่อ 88-level ไม่ใช่ตัวแปรที่ `MOVE` ค่าเข้าตรง ๆ ได้ COBOL จึงมีคำสั่งเฉพาะคือ
**`SET condition-name TO TRUE`** ซึ่งจะไปกำหนดค่า `VALUE` ที่ระบุไว้ตอนประกาศ 88-level นั้น
กลับเข้าไปยังฟิลด์แม่โดยอัตโนมัติ โดยที่เราไม่ต้องจำหรือพิมพ์ค่าดิบ (raw value) ซ้ำอีกเลย

ข้อดีสำคัญ: หากในอนาคตต้องเปลี่ยนรหัสสถานะ (เช่นเปลี่ยนจาก "P" เป็น "PD") เราแก้ไขแค่จุดเดียว
ตรงที่ประกาศ 88-level เท่านั้น โดยไม่ต้องไปค้นหาและแก้ `MOVE "P" TO ...` ทุกจุดในโปรแกรม

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CONDITION-NAME-SET.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ACCOUNT-STATUS   PIC X(1).
           88  ACCOUNT-ACTIVE             VALUE "A".
           88  ACCOUNT-SUSPENDED          VALUE "S".
           88  ACCOUNT-CLOSED             VALUE "C".

       PROCEDURE DIVISION.
      *> SET ... TO TRUE moves the correct VALUE for us
      *> so we never have to remember the raw code "A".
           SET ACCOUNT-ACTIVE TO TRUE
           DISPLAY "STATUS CODE STORED: " WS-ACCOUNT-STATUS
           IF ACCOUNT-ACTIVE
               DISPLAY "ACCOUNT IS ACTIVE."
           END-IF

           SET ACCOUNT-SUSPENDED TO TRUE
           DISPLAY "STATUS CODE STORED: " WS-ACCOUNT-STATUS
           IF ACCOUNT-SUSPENDED
               DISPLAY "ACCOUNT IS SUSPENDED, ACCESS LIMITED."
           END-IF
           STOP RUN.
```

### อธิบายโค้ด

- `SET ACCOUNT-ACTIVE TO TRUE` — เทียบเท่ากับ `MOVE "A" TO WS-ACCOUNT-STATUS` แต่เขียนในรูปแบบ
  ที่สื่อความหมายทางธุรกิจได้ตรงกว่ามาก (อ่านว่า "ตั้งค่าบัญชีให้เป็น Active")
- หลัง `SET ACCOUNT-ACTIVE TO TRUE` ตัวแปร `WS-ACCOUNT-STATUS` จะมีค่าเป็น "A" จริง ๆ ตามที่
  `DISPLAY` แสดงให้เห็น และ `IF ACCOUNT-ACTIVE` จึงเป็นจริงในบรรทัดถัดมา
- เมื่อเปลี่ยนไป `SET ACCOUNT-SUSPENDED TO TRUE` ค่าของ `WS-ACCOUNT-STATUS` จะเปลี่ยนเป็น "S"
  ทันที และตอนนี้ `ACCOUNT-ACTIVE` จะกลายเป็นเท็จโดยอัตโนมัติ (เพราะค่าฟิลด์แม่เปลี่ยนไปแล้ว)

### ผลลัพธ์ที่ได้จากการรันจริง

```
STATUS CODE STORED: A
ACCOUNT IS ACTIVE.
STATUS CODE STORED: S
ACCOUNT IS SUSPENDED, ACCESS LIMITED.
```

### ข้อควรระวัง

- **`SET condition-name TO TRUE` ใช้ได้เฉพาะกับ 88-level ที่มีค่าเดียว (single VALUE) เท่านั้น**
  หาก 88-level นั้นประกาศด้วยช่วง (`THRU`) หรือหลายค่า (`VALUES`) COBOL จะไม่ทราบว่าควรกำหนดค่า
  ไหนกลับไปยังฟิลด์แม่ (ตัวอย่างเช่น `GRADE-B VALUE 80 THRU 89` ในขั้นตอนก่อนหน้า จะไม่สามารถใช้
  `SET GRADE-B TO TRUE` ได้ เพราะไม่รู้ว่าจะตั้งเป็น 80, 85 หรือ 89)
- ไม่มี `SET condition-name TO FALSE` ในความหมายที่ตรงไปตรงมา (เว้นแต่จะประกาศ `VALUE ... WHEN
  SET TO FALSE IS ...` แบบขั้นสูงซึ่งไม่ค่อยพบในโค้ดทั่วไป) หากต้องการ "ปิด" เงื่อนไข ให้ SET
  เงื่อนไขอื่นที่ต้องการให้เป็นจริงแทน

### แบบฝึกหัดที่ 98.1

**โจทย์**: จากตัวแปร `WS-PAYMENT-METHOD` และ 88-level `PAY-BY-CASH`, `PAY-BY-CREDIT-CARD`,
`PAY-BY-TRANSFER` ในแบบฝึกหัด 96.1 จงเขียนคำสั่งเพื่อกำหนดว่าลูกค้าจ่ายด้วยบัตรเครดิต

**เฉลย**: `SET PAY-BY-CREDIT-CARD TO TRUE` — จะกำหนดค่า "K" ให้กับ `WS-PAYMENT-METHOD` โดยอัตโนมัติ

---

## ขั้นตอนที่ 99: Sign Condition และ Abbreviated Combined Relation Condition

### แนวคิด

นอกจาก Class Condition (`IS NUMERIC`, `IS ALPHABETIC`) ที่เรียนไปแล้ว COBOL ยังมี **Sign
Condition** สำหรับตรวจสอบเครื่องหมายของตัวเลขโดยตรง:

| เงื่อนไข | ความหมาย |
|---|---|
| `IS POSITIVE` | มากกว่า 0 |
| `IS NEGATIVE` | น้อยกว่า 0 |
| `IS ZERO` | เท่ากับ 0 พอดี |

และยังมีเทคนิคที่เรียกว่า **Abbreviated Combined Relation Condition** ที่ช่วยย่อเงื่อนไขที่เปรียบเทียบ
ตัวแปรตัวเดียวกันซ้ำ ๆ ให้สั้นลง เช่น แทนที่จะเขียน `A = B AND A = C` สามารถย่อเป็น
`A = B AND = C` ได้ (ละตัวแปร A ตัวที่สองไว้ในฐานที่เข้าใจ)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IF-ADVANCED-CONDITIONS.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-BALANCE          PIC S9(7)V99  VALUE -500.00.
       01  WS-CODE             PIC X(5)      VALUE "12345".
       01  WS-A                PIC 9(3)      VALUE 10.
       01  WS-B                PIC 9(3)      VALUE 10.
       01  WS-C                PIC 9(3)      VALUE 10.

       PROCEDURE DIVISION.
      *> Sign condition: checks POSITIVE, NEGATIVE or ZERO
           IF WS-BALANCE IS NEGATIVE
               DISPLAY "ACCOUNT IS OVERDRAWN."
           END-IF

           IF WS-CODE IS NUMERIC
               DISPLAY "CODE FIELD CONTAINS ONLY DIGITS."
           END-IF

      *> Abbreviated combined relation condition:
      *> "A = B AND A = C" can be shortened like this
           IF WS-A = WS-B AND = WS-C
               DISPLAY "A, B AND C ARE ALL EQUAL (ABBREVIATED)."
           END-IF

      *> The same idea for a range check
           IF WS-A > 0 AND < 100
               DISPLAY "A IS BETWEEN 0 AND 100 (ABBREVIATED)."
           END-IF
           STOP RUN.
```

### อธิบายโค้ด

- `WS-BALANCE IS NEGATIVE` — `WS-BALANCE` มีค่า -500.00 ซึ่งน้อยกว่า 0 เงื่อนไขจึงเป็นจริง
  แสดงข้อความแจ้งเตือนบัญชีติดลบ (เทียบเท่ากับ `IF WS-BALANCE < 0` แต่อ่านเป็นธรรมชาติกว่า)
- `IF WS-A = WS-B AND = WS-C` — รูปแบบย่อของ `IF WS-A = WS-B AND WS-A = WS-C` โดยละตัวแปร
  `WS-A` ตัวที่สองไว้ (COBOL เข้าใจว่ายังหมายถึงตัวแปรตัวแรกก่อนหน้า AND) ทั้งสามตัวแปรมีค่า 10
  เท่ากันหมด เงื่อนไขจึงเป็นจริง
- `IF WS-A > 0 AND < 100` — รูปแบบย่อของ `IF WS-A > 0 AND WS-A < 100` ใช้ตรวจสอบว่าค่าอยู่ในช่วง
  ที่กำหนดโดยไม่ต้องพิมพ์ชื่อตัวแปรซ้ำ

### ผลลัพธ์ที่ได้จากการรันจริง

```
ACCOUNT IS OVERDRAWN.
CODE FIELD CONTAINS ONLY DIGITS.
A, B AND C ARE ALL EQUAL (ABBREVIATED).
A IS BETWEEN 0 AND 100 (ABBREVIATED).
```

### ข้อควรระวัง

- แม้ Abbreviated Combined Relation Condition จะช่วยลดความยาวโค้ด แต่ **อ่านยากกว่าเงียนเต็มรูป
  สำหรับผู้ที่ไม่คุ้นเคย** หลายทีมในอุตสาหกรรมจึงเลือกเขียนเงื่อนไขแบบเต็ม (ไม่ย่อ) เพื่อความชัดเจน
  ในการบำรุงรักษาโค้ดระยะยาว โดยเฉพาะในทีมที่มีทั้งโปรแกรมเมอร์รุ่นใหม่และรุ่นเก่าทำงานร่วมกัน
- `IS ZERO` แตกต่างจาก `= 0` เล็กน้อยในบางบริบท (`IS ZERO` เป็น Sign Condition ที่ตรวจสอบทั้งค่า
  ตัวเลขและฟิลด์ตัวอักษรที่มีแต่ตัวอักษร "0" ได้ในบางคอมไพเลอร์) แต่ในทางปฏิบัติทั่วไปสามารถใช้
  แทนกันได้กับฟิลด์ตัวเลขปกติ

### แบบฝึกหัดที่ 99.1

**โจทย์**: จงเขียนเงื่อนไขแบบย่อ (abbreviated) เพื่อตรวจสอบว่า `WS-SCORE` อยู่ในช่วง 60 ถึง 100
(มากกว่าหรือเท่ากับ 60 และน้อยกว่าหรือเท่ากับ 100)

**เฉลย**: `IF WS-SCORE >= 60 AND <= 100` — เทียบเท่ากับ `IF WS-SCORE >= 60 AND WS-SCORE <= 100`

---

## ขั้นตอนที่ 100: โปรแกรมรวบยอด — ระบบพิจารณาสินเชื่อ (Loan Approval System)

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่รวม `IF-ELSE` แบบซ้อนกันหลายชั้น เข้ากับ **Condition Name (88-level)**
แบบช่วงค่า (THRU) เพื่อจำลองสถานการณ์ธุรกิจจริง: ระบบพิจารณาอนุมัติสินเชื่อที่ต้องตรวจสอบเงื่อนไข
หลายข้อร่วมกันตามลำดับความสำคัญ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LOAN-APPROVAL-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-APPLICANT-AGE    PIC 9(3)      VALUE 28.
       01  WS-MONTHLY-INCOME   PIC 9(7)V99   VALUE 35000.00.
       01  WS-CREDIT-SCORE     PIC 9(3)      VALUE 720.
           88  CREDIT-EXCELLENT           VALUE 750 THRU 850.
           88  CREDIT-GOOD                VALUE 650 THRU 749.
           88  CREDIT-POOR                VALUE 300 THRU 649.
       01  WS-APPROVAL-RESULT  PIC X(30).

       PROCEDURE DIVISION.
           IF WS-APPLICANT-AGE < 20 OR WS-APPLICANT-AGE > 65
               MOVE "REJECTED: AGE LIMIT" TO WS-APPROVAL-RESULT
           ELSE
               IF WS-MONTHLY-INCOME < 15000.00
                   MOVE "REJECTED: LOW INCOME"
                       TO WS-APPROVAL-RESULT
               ELSE
                   IF CREDIT-EXCELLENT
                       MOVE "APPROVED: BEST RATE"
                           TO WS-APPROVAL-RESULT
                   ELSE
                       IF CREDIT-GOOD
                           MOVE "APPROVED: STANDARD RATE"
                               TO WS-APPROVAL-RESULT
                       ELSE
                           MOVE "REJECTED: LOW CREDIT SCORE"
                               TO WS-APPROVAL-RESULT
                       END-IF
                   END-IF
               END-IF
           END-IF

           DISPLAY "=== LOAN APPLICATION RESULT ==="
           DISPLAY "AGE           : " WS-APPLICANT-AGE
           DISPLAY "MONTHLY INCOME: " WS-MONTHLY-INCOME
           DISPLAY "CREDIT SCORE  : " WS-CREDIT-SCORE
           DISPLAY "DECISION      : " WS-APPROVAL-RESULT
           STOP RUN.
```

### อธิบายโค้ด

- เงื่อนไขแรกตรวจสอบอายุก่อน: ถ้าอายุต่ำกว่า 20 หรือมากกว่า 65 ปี ปฏิเสธทันที (ไม่ต้องตรวจสอบ
  เงื่อนไขอื่นต่อ) — นี่คือการจัดลำดับเงื่อนไขที่สำคัญที่สุดไว้ก่อน (fail-fast pattern)
- เงื่อนไขที่สองตรวจสอบรายได้ขั้นต่ำ: ถ้ารายได้ต่ำกว่า 15000 บาทต่อเดือน ปฏิเสธ
- เงื่อนไขที่สามและสี่ใช้ 88-level `CREDIT-EXCELLENT` และ `CREDIT-GOOD` ที่ประกาศเป็นช่วงคะแนน
  เครดิต (THRU) ทำให้โค้ดอ่านง่ายกว่าการเขียน `IF WS-CREDIT-SCORE >= 750` ตรง ๆ มาก
- ด้วยคะแนนเครดิต 720 ซึ่งอยู่ในช่วง `CREDIT-GOOD` (650-749) ผลลัพธ์สุดท้ายคือ "APPROVED: STANDARD
  RATE"
- สังเกตว่า `WS-APPROVAL-RESULT` ถูกประกาศเป็น `PIC X(30)` เพื่อรองรับข้อความยาวที่สุดคือ
  "REJECTED: LOW CREDIT SCORE" (27 ตัวอักษร) — หากประกาศสั้นเกินไป ข้อความจะถูกตัดทิ้งตามกฎ MOVE
  ที่เรียนใน Part 008 ขั้นตอนที่ 74

### ผลลัพธ์ที่ได้จากการรันจริง

```
=== LOAN APPLICATION RESULT ===
AGE           : 028
MONTHLY INCOME: 0035000.00
CREDIT SCORE  : 720
DECISION      : APPROVED: STANDARD RATE
```

### ข้อควรระวัง

- **การจัดลำดับเงื่อนไขมีความสำคัญมาก** ในตัวอย่างนี้เราตรวจสอบอายุก่อนรายได้ก่อนเครดิต หากสลับ
  ลำดับ ผลลัพธ์อาจต่างกันในบางกรณี (เช่น ผู้สมัครอายุเกินและรายได้ต่ำพร้อมกัน ควรได้ข้อความปฏิเสธ
  ข้อไหนก่อน) การออกแบบลำดับเงื่อนไขต้องพิจารณาจากกฎธุรกิจจริงเสมอ
- ตัวอย่างนี้แสดงให้เห็นว่าเมื่อ nested IF ลึกมากขึ้น (4 ชั้นในที่นี้) การเยื้องโค้ด (indentation)
  และการนับ `END-IF` ให้ครบเริ่มซับซ้อนขึ้นเรื่อย ๆ — นี่คือแรงจูงใจสำคัญที่ทำให้ Part ถัดไปแนะนำ
  `EVALUATE` เป็นทางเลือกที่จัดการเงื่อนไขหลายระดับได้เรียบร้อยกว่า

### แบบฝึกหัดที่ 100.1

**โจทย์**: จงต่อยอดโปรแกรมระบบพิจารณาสินเชื่อข้างต้น โดยเพิ่มเงื่อนไข 88-level ใหม่ชื่อ
`CREDIT-VERY-POOR VALUE 0 THRU 299` และเพิ่มเงื่อนไขปฏิเสธพิเศษ "REJECTED: VERY POOR CREDIT,
CONTACT BRANCH" สำหรับกรณีนี้

**เฉลยแนวทาง**:
```cobol
01  WS-CREDIT-SCORE     PIC 9(3)      VALUE 720.
    88  CREDIT-EXCELLENT           VALUE 750 THRU 850.
    88  CREDIT-GOOD                VALUE 650 THRU 749.
    88  CREDIT-POOR                VALUE 300 THRU 649.
    88  CREDIT-VERY-POOR           VALUE 0 THRU 299.
```
จากนั้นเพิ่มเงื่อนไขตรวจสอบ `IF CREDIT-VERY-POOR` ก่อนเงื่อนไข `CREDIT-POOR` เดิม เพื่อแยกกรณี
คะแนนเครดิตต่ำมากออกจากกรณีคะแนนต่ำทั่วไป และแสดงข้อความแนะนำให้ติดต่อสาขาแทนการปฏิเสธเฉย ๆ

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้แนวคิด Selection (การเลือกทำ) ผ่านคำสั่ง `IF-ELSE` และเทคนิค Condition
Names อย่างครบถ้วน:

- คำสั่ง `IF` พื้นฐานและ `IF-ELSE` พร้อม scope terminator `END-IF`
- Nested IF สำหรับตรวจสอบเงื่อนไขหลายระดับต่อเนื่องกัน
- ตัวดำเนินการเปรียบเทียบครบชุด และ Class Condition (`IS NUMERIC`, `IS ALPHABETIC`)
- เงื่อนไขผสมด้วย `AND`, `OR`, `NOT` และความสำคัญของวงเล็บ
- กับดักอันตรายของการใช้ period แทน `END-IF` ที่ทำให้ตรรกะผิดพลาดแบบเงียบ ๆ
- Condition Names (88-level) พื้นฐาน สำหรับตั้งชื่อเงื่อนไขที่มีความหมาย
- Condition Names กับช่วงค่า (`THRU`) และหลายค่า (`VALUES`)
- `SET condition-name TO TRUE` สำหรับกำหนดค่าผ่านชื่อเงื่อนไข
- Sign Condition (`IS POSITIVE`, `IS NEGATIVE`, `IS ZERO`) และ Abbreviated Combined Relation
  Condition
- โปรแกรมรวบยอดระบบพิจารณาสินเชื่อที่ผสาน IF ซ้อนกันหลายชั้นกับ 88-level

การตัดสินใจแบบมีเงื่อนไขคือหัวใจของตรรกะโปรแกรมธุรกิจ แต่เมื่อเงื่อนไขมีมากขึ้นเรื่อย ๆ nested IF
จะเริ่มอ่านยากและดูแลรักษายาก ใน Part ถัดไปเราจะเรียนรู้ **`EVALUATE`** ซึ่งเป็นคำสั่งที่ออกแบบมา
เพื่อจัดการเงื่อนไขหลายทางเลือกได้อย่างเป็นระเบียบและอ่านง่ายกว่า เทียบเท่ากับ switch/case ใน
ภาษาโปรแกรมอื่น ๆ

**[← กลับไป Part 009](part-009-arithmetic.md)** | **[ไปยัง Part 011: EVALUATE Statement →](part-011-evaluate.md)**
