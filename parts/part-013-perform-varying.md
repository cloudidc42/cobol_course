# Part 013: PERFORM VARYING และการวนลูปหลายชั้น (Nested Loops) (ขั้นตอนที่ 121–130)

## คำนำของ Part นี้

ใน Part 012 เราได้เรียนรู้ `PERFORM` พื้นฐานและ `PERFORM UNTIL` ซึ่งเหมาะกับการวนลูปที่ตรวจสอบเงื่อนไข
แบบอิสระ (เช่น "วนไปเรื่อย ๆ จนกว่าผู้ใช้จะพิมพ์ END") แต่เมื่อใดก็ตามที่เราต้องการ **นับจำนวนรอบที่แน่นอน**
เช่น "ทำซ้ำ 10 ครั้ง" หรือ "ไล่ตั้งแต่ 1 ถึง 100" `PERFORM VARYING` คือเครื่องมือที่ COBOL ออกแบบมาสำหรับงานนี้
โดยเฉพาะ — มันรวมเอาทั้งการกำหนดค่าเริ่มต้น (initialize), เงื่อนไขหยุด (until), และการเพิ่ม/ลดค่า (increment)
ไว้ในคำสั่งเดียว คล้ายกับ `for` loop ในภาษาโปรแกรมสมัยใหม่

Part นี้จะพาคุณเจาะลึก `PERFORM VARYING` ตั้งแต่รูปแบบพื้นฐานที่สุด ไปจนถึงการวนลูปซ้อนกันหลายชั้น
(Nested Loops) ซึ่งเป็นทักษะสำคัญมากที่จะใช้ตลอดทั้งหลักสูตร โดยเฉพาะเมื่อเราเริ่มทำงานกับตาราง (Tables)
ใน Part 016 เป็นต้นไป

---

## ขั้นตอนที่ 121: รูปแบบพื้นฐานของ PERFORM VARYING

### แนวคิด

`PERFORM VARYING` คือคำสั่งวนลูปแบบนับจำนวนรอบ (Counted Loop) ที่มีโครงสร้างดังนี้:

```
PERFORM VARYING identifier-1 FROM value-1 BY value-2
        UNTIL condition
    <statements>
END-PERFORM
```

ส่วนประกอบสำคัญ:

- **identifier-1**: ตัวแปรที่ใช้เป็นตัวนับ (Loop Control Variable) ต้องเป็นตัวเลข
- **FROM value-1**: ค่าเริ่มต้นของตัวนับ
- **BY value-2**: ค่าที่จะบวกเพิ่ม (หรือลบ ถ้าใส่ค่าติดลบ) ในแต่ละรอบ
- **UNTIL condition**: เงื่อนไขที่เมื่อเป็นจริงแล้วจะ **หยุด** ลูปทันที (สังเกตว่าเป็น "จนกว่า" ไม่ใช่ "ในขณะที่")

จุดสำคัญที่ต้องจำ: COBOL ตรวจสอบเงื่อนไข `UNTIL` **ก่อน** ที่จะรันเนื้อลูปในแต่ละรอบ (ค่าเริ่มต้นคือ
`TEST BEFORE` ซึ่งเราจะพูดถึงอย่างละเอียดในขั้นตอนที่ 125) ดังนั้นถ้าเงื่อนไขเป็นจริงตั้งแต่รอบแรก
เนื้อลูปจะไม่ถูกรันเลยแม้แต่ครั้งเดียว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP121.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER          PIC 9(03).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== PERFORM VARYING basic demo ===".
           PERFORM VARYING WS-COUNTER FROM 1 BY 1
                   UNTIL WS-COUNTER > 5
               DISPLAY "Counter value: " WS-COUNTER
           END-PERFORM.
           DISPLAY "Loop finished. Final counter = " WS-COUNTER.
           STOP RUN.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step121 step121.cob
./step121
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM VARYING basic demo ===
Counter value: 001
Counter value: 002
Counter value: 003
Counter value: 004
Counter value: 005
Loop finished. Final counter = 006
```

### อธิบายโค้ดทีละส่วน

- `PERFORM VARYING WS-COUNTER FROM 1 BY 1 UNTIL WS-COUNTER > 5` — เริ่มจาก 1, เพิ่มทีละ 1, หยุดเมื่อ
  ค่ามากกว่า 5 (คือรันตั้งแต่ 1 ถึง 5 รวม 5 รอบ)
- สังเกตว่า **หลังจบลูป** ค่า `WS-COUNTER` จะเป็น `006` ไม่ใช่ `005` เพราะ COBOL จะเพิ่มค่าอีกหนึ่งครั้ง
  แล้วจึงตรวจพบว่าเงื่อนไข `UNTIL` เป็นจริง แล้วค่อยออกจากลูป — นี่คือจุดที่มือใหม่มักงงและนำไปสู่บั๊ก
  off-by-one ถ้านำ `WS-COUNTER` ไปใช้ต่อหลังลูปโดยไม่ระวัง
- `END-PERFORM` คือ Scope Terminator (จาก COBOL-85) ที่ปิดขอบเขตของ `PERFORM` แบบชัดเจน

### ข้อควรระวัง

- ตัวแปรตัวนับต้องมี `PICTURE` เป็นตัวเลขเท่านั้น และต้องมีขนาดใหญ่พอที่จะรองรับค่าสูงสุดที่จะวนถึง
  มิฉะนั้นจะเกิด Overflow แบบเงียบ (silent truncation) โดยไม่มี error แจ้งเตือน
- อย่าลืมว่า `UNTIL` คือเงื่อนไข "หยุดเมื่อจริง" ไม่ใช่ "ทำต่อเมื่อจริง" เหมือน `while` ในบางภาษา
  หากเขียนเงื่อนไขผิดทิศทาง (เช่น `UNTIL WS-COUNTER < 5` ทั้งที่ตั้งใจจะนับขึ้น) จะกลายเป็นลูปไม่ทำงานเลย
  หรือวนไม่รู้จบ

### แบบฝึกหัดที่ 121.1

**โจทย์**: แก้ไข `step121.cob` ให้แสดงเลข 1 ถึง 10 แทนที่จะเป็น 1 ถึง 5

**เฉลย**: เปลี่ยนเงื่อนไขเป็น `UNTIL WS-COUNTER > 10` เท่านั้น ส่วนอื่นเหมือนเดิมทั้งหมด:

```cobol
           PERFORM VARYING WS-COUNTER FROM 1 BY 1
                   UNTIL WS-COUNTER > 10
               DISPLAY "Counter value: " WS-COUNTER
           END-PERFORM.
```

---

## ขั้นตอนที่ 122: ปรับ FROM และ BY เพื่อวนลูปแบบกำหนดเอง (ก้าวกระโดด, นับถอยหลัง)

### แนวคิด

`FROM` และ `BY` ไม่จำเป็นต้องเป็น `1` และ `1` เสมอไป เราสามารถ:

- กำหนด `BY` เป็นค่าอื่น เช่น `BY 2` เพื่อวนแบบข้ามทีละ 2 (เลขคู่, เลขคี่ ฯลฯ)
- กำหนด `BY` เป็น**ค่าติดลบ** เช่น `BY -1` เพื่อ**นับถอยหลัง** — แต่ต้องระวังว่าตัวแปรตัวนับต้องเป็น
  ตัวเลข **มีเครื่องหมาย (signed)** คือใช้ `S` นำหน้าใน PICTURE เช่น `PIC S9(03)` มิฉะนั้นค่าติดลบ
  จะถูกตัดทิ้งเครื่องหมายและพฤติกรรมจะผิดเพี้ยน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP122.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EVEN-NUM         PIC 9(03).
       01  WS-COUNTDOWN        PIC S9(03).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Custom FROM and BY demo ===".

           DISPLAY "-- Even numbers from 2 to 10 (step +2) --".
           PERFORM VARYING WS-EVEN-NUM FROM 2 BY 2
                   UNTIL WS-EVEN-NUM > 10
               DISPLAY "Even number: " WS-EVEN-NUM
           END-PERFORM.

           DISPLAY "-- Countdown from 5 to 1 (step -1) --".
           PERFORM VARYING WS-COUNTDOWN FROM 5 BY -1
                   UNTIL WS-COUNTDOWN < 1
               DISPLAY "Countdown: " WS-COUNTDOWN
           END-PERFORM.

           DISPLAY "All loops finished.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Custom FROM and BY demo ===
-- Even numbers from 2 to 10 (step +2) --
Even number: 002
Even number: 004
Even number: 006
Even number: 008
Even number: 010
-- Countdown from 5 to 1 (step -1) --
Countdown: +005
Countdown: +004
Countdown: +003
Countdown: +002
Countdown: +001
All loops finished.
```

### อธิบายโค้ดทีละส่วน

- `WS-COUNTDOWN` ประกาศเป็น `PIC S9(03)` (มีเครื่องหมาย) เพื่อรองรับการเปรียบเทียบ `< 1` ได้ถูกต้อง
  แม้ในตัวอย่างนี้ค่าจะไม่ติดลบจริง แต่การประกาศแบบมีเครื่องหมายเป็นนิสัยที่ดีเมื่อใช้ `BY` เป็นค่าลบ
- สังเกตผลลัพธ์การแสดง `WS-COUNTDOWN` จะมีเครื่องหมาย `+` นำหน้าเสมอ (เช่น `+005`) เพราะ `DISPLAY`
  ตัวแปรที่มีเครื่องหมายจะแสดงเครื่องหมายกำกับด้วยตามค่าเริ่มต้น

### ข้อควรระวัง

- ถ้าลืมใส่ `S` ใน PICTURE ตอนใช้ `BY` เป็นค่าลบ ตัวแปรจะไม่สามารถเก็บค่าติดลบได้จริง และเงื่อนไข
  `UNTIL ... < 1` อาจไม่มีวันเป็นจริง ทำให้เกิดลูปไม่รู้จบ (infinite loop)
- `FROM` และ `BY` รับได้ทั้งตัวเลขคงที่ (literal) และตัวแปร/นิพจน์ ทำให้ยืดหยุ่นมาก แต่ก็หมายความว่า
  ถ้าตัวแปรที่ใช้ใน `BY` มีค่าเป็น 0 โดยไม่ตั้งใจ จะกลายเป็นลูปไม่รู้จบทันที (ตัวนับไม่ขยับเลย)

### แบบฝึกหัดที่ 122.1

**โจทย์**: เขียนลูปแสดงเลขคี่ตั้งแต่ 1 ถึง 9

**เฉลย**:

```cobol
           PERFORM VARYING WS-EVEN-NUM FROM 1 BY 2
                   UNTIL WS-EVEN-NUM > 9
               DISPLAY "Odd number: " WS-EVEN-NUM
           END-PERFORM.
```

---

## ขั้นตอนที่ 123: ลูปซ้อน 2 ชั้นด้วย PERFORM VARYING ... AFTER

### แนวคิด

เมื่อต้องการวนลูปสองมิติ (เช่น แถวและคอลัมน์) COBOL มีวลี `AFTER` ที่ให้เรากำหนดลูปชั้นในได้ **ในคำสั่ง
เดียวกัน** โดยไม่ต้องเขียน `PERFORM VARYING` ซ้อนกันสองคำสั่งแยกกัน:

```
PERFORM VARYING outer-var FROM ... BY ... UNTIL ...
        AFTER inner-var FROM ... BY ... UNTIL ...
    <statements>
END-PERFORM
```

กลไกการทำงาน: สำหรับทุกค่าของ `outer-var` หนึ่งค่า COBOL จะวน `inner-var` ให้ครบทุกค่าก่อน แล้วจึง
ขยับ `outer-var` ไปค่าถัดไปและเริ่มวน `inner-var` ใหม่ตั้งแต่ค่า `FROM` อีกครั้ง — พฤติกรรมนี้เหมือนกับ
การเขียน `PERFORM VARYING` สองอันซ้อนกัน แต่กระชับกว่าและเป็นที่นิยมมากในโค้ด COBOL มาตรฐาน

### ตัวอย่างโค้ด: ตารางสูตรคูณ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP123.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ROW              PIC 9(02).
       01  WS-COL              PIC 9(02).
       01  WS-PRODUCT          PIC 9(04).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Multiplication table 1-5 (nested PERFORM) ===".
           PERFORM VARYING WS-ROW FROM 1 BY 1 UNTIL WS-ROW > 5
               PERFORM VARYING WS-COL FROM 1 BY 1 UNTIL WS-COL > 5
                   COMPUTE WS-PRODUCT = WS-ROW * WS-COL
                   DISPLAY WS-ROW " x " WS-COL " = " WS-PRODUCT
               END-PERFORM
               DISPLAY "----------------------"
           END-PERFORM.
           STOP RUN.
```

> **หมายเหตุ**: ตัวอย่างนี้เขียนแบบ `PERFORM VARYING` ซ้อนกันสองคำสั่ง (nested statement) เพื่อให้เห็น
> โครงสร้างชัดเจนก่อน โดยสามารถเขียนแบบใช้ `AFTER` รวมเป็นคำสั่งเดียวได้ผลลัพธ์เดียวกันทุกประการ ดังนี้:

```cobol
           PERFORM VARYING WS-ROW FROM 1 BY 1 UNTIL WS-ROW > 5
                   AFTER WS-COL FROM 1 BY 1 UNTIL WS-COL > 5
               COMPUTE WS-PRODUCT = WS-ROW * WS-COL
               DISPLAY WS-ROW " x " WS-COL " = " WS-PRODUCT
           END-PERFORM.
```

### ผลลัพธ์จริงที่ได้ (บางส่วน)

```
=== Multiplication table 1-5 (nested PERFORM) ===
01 x 01 = 0001
01 x 02 = 0002
01 x 03 = 0003
01 x 04 = 0004
01 x 05 = 0005
----------------------
02 x 01 = 0002
02 x 02 = 0004
02 x 03 = 0006
02 x 04 = 0008
02 x 05 = 0010
----------------------
...
05 x 01 = 0005
05 x 02 = 0010
05 x 03 = 0015
05 x 04 = 0020
05 x 05 = 0025
----------------------
```

(ผลลัพธ์เต็มมีทั้งหมด 5 บล็อก บล็อกละ 5 บรรทัด รวม 25 บรรทัดข้อมูล ทดสอบและยืนยันแล้วว่าคอมไพล์และ
รันได้จริงด้วย GnuCOBOL)

### อธิบายโค้ดทีละส่วน

- ลูปนอก (`WS-ROW`) ควบคุม "แถว" ของตารางสูตรคูณ ตั้งแต่ 1 ถึง 5
- ลูปใน (`WS-COL`) ควบคุม "คอลัมน์" และจะวนครบทั้ง 5 รอบทุกครั้งที่ลูปนอกขยับไปหนึ่งค่า
- `DISPLAY "----------------------"` อยู่นอกลูปใน แต่ในลูปนอก จึงพิมพ์เส้นคั่นครั้งเดียวต่อหนึ่งแถว

### ข้อควรระวัง

- รูปแบบ `AFTER` สะดวก แต่ทำให้โค้ดอ่านยากขึ้นเล็กน้อยสำหรับผู้เริ่มต้น เพราะเนื้อลูปทั้งสองชั้นถูกยุบ
  รวมกันในบล็อกเดียว หากมีคำสั่งที่ต้องทำ "ต่อรอบของลูปนอกเท่านั้น" (เช่นเส้นคั่นในตัวอย่างนี้) จะทำ
  ไม่ได้ถ้าใช้ `AFTER` — ต้องแยกเป็น `PERFORM VARYING` ซ้อนกันแบบข้างต้นแทน
- `AFTER` รองรับได้มากกว่า 1 ชั้น (ดูขั้นตอนที่ 124) แต่ยิ่งซ้อนมากยิ่งอ่านยาก ควรพิจารณาแยกเป็น
  Paragraph ย่อยเมื่อซับซ้อนเกินไป (เนื้อหานี้จะสอนใน Part 014)

### แบบฝึกหัดที่ 123.1

**โจทย์**: ปรับ `step123.cob` ให้แสดงตารางสูตรคูณเฉพาะแม่ 7 เท่านั้น (ไม่ต้องมีลูปนอก)

**เฉลย**:

```cobol
       PROCEDURE DIVISION.
       MAIN-LOGIC.
           PERFORM VARYING WS-COL FROM 1 BY 1 UNTIL WS-COL > 12
               COMPUTE WS-PRODUCT = 7 * WS-COL
               DISPLAY "7 x " WS-COL " = " WS-PRODUCT
           END-PERFORM.
           STOP RUN.
```

---

## ขั้นตอนที่ 124: ลูปซ้อน 3 ชั้น และการติดตามค่าตัวนับหลายตัวพร้อมกัน

### แนวคิด

ในงานจริง เช่น การประมวลผลข้อมูล 3 มิติ (ปี → เดือน → วัน) หรือ Matrix 3 มิติ เราอาจต้องซ้อนลูปถึง 3 ชั้น
หลักการเหมือนกับ 2 ชั้นทุกประการ เพียงแค่เพิ่มระดับการซ้อนเข้าไปอีก สิ่งสำคัญคือ **ต้องตั้งชื่อตัวแปร
ตัวนับให้แตกต่างกันในทุกชั้น** (เช่น `WS-LEVEL`, `WS-ROW`, `WS-COL`) เพื่อไม่ให้สับสนหรือชนกัน (ปัญหานี้
จะพูดถึงอย่างละเอียดในขั้นตอนที่ 129)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP124.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LEVEL            PIC 9(02).
       01  WS-ROW              PIC 9(02).
       01  WS-COL              PIC 9(02).
       01  WS-CELL-COUNT       PIC 9(05) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== 3-level nested PERFORM VARYING demo ===".
           PERFORM VARYING WS-LEVEL FROM 1 BY 1 UNTIL WS-LEVEL > 2
               DISPLAY "Level " WS-LEVEL " begins:"
               PERFORM VARYING WS-ROW FROM 1 BY 1 UNTIL WS-ROW > 2
                   PERFORM VARYING WS-COL FROM 1 BY 1 UNTIL WS-COL > 3
                       ADD 1 TO WS-CELL-COUNT
                       DISPLAY "  L=" WS-LEVEL " R=" WS-ROW
                               " C=" WS-COL
                               " (cell #" WS-CELL-COUNT ")"
                   END-PERFORM
               END-PERFORM
           END-PERFORM.
           DISPLAY "Total cells visited: " WS-CELL-COUNT.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== 3-level nested PERFORM VARYING demo ===
Level 01 begins:
  L=01 R=01 C=01 (cell #00001)
  L=01 R=01 C=02 (cell #00002)
  L=01 R=01 C=03 (cell #00003)
  L=01 R=02 C=01 (cell #00004)
  L=01 R=02 C=02 (cell #00005)
  L=01 R=02 C=03 (cell #00006)
Level 02 begins:
  L=02 R=01 C=01 (cell #00007)
  L=02 R=01 C=02 (cell #00008)
  L=02 R=01 C=03 (cell #00009)
  L=02 R=02 C=01 (cell #00010)
  L=02 R=02 C=02 (cell #00011)
  L=02 R=02 C=03 (cell #00012)
Total cells visited: 00012
```

### อธิบายโค้ดทีละส่วน

- จำนวนรอบทั้งหมด = 2 (Level) × 2 (Row) × 3 (Col) = 12 รอบ ตรงกับ `WS-CELL-COUNT` สุดท้ายที่ได้คือ 12
  พอดี ซึ่งเป็นวิธีที่ดีในการ "ตรวจสอบตัวเอง" (self-check) ว่าลูปซ้อนทำงานถูกต้องครบทุกรอบ
- ลูปที่ซ้อนลึกที่สุด (`WS-COL`) จะวนครบก่อนเสมอ ก่อนที่ลูปชั้นกลาง (`WS-ROW`) จะขยับ และลูปชั้นกลาง
  จะวนครบก่อนที่ลูปนอกสุด (`WS-LEVEL`) จะขยับ — หลักการนี้เรียกว่า "ลูปในสุดหมุนเร็วที่สุด"

### ข้อควรระวัง

- ยิ่งซ้อนลูปมาก จำนวนรอบทั้งหมดจะเพิ่มแบบทวีคูณ (multiplicative) ไม่ใช่บวกกัน ตัวอย่างนี้ดูเล็ก
  (12 รอบ) แต่ถ้าแต่ละชั้นมี 100 รอบ ลูป 3 ชั้นจะกลายเป็น 1,000,000 รอบทันที ควรระวังเรื่อง
  ประสิทธิภาพ (performance) เสมอเมื่อซ้อนลูปหลายชั้นกับข้อมูลขนาดใหญ่
- การใช้ตัวนับที่มีความหมายชัดเจน (`WS-LEVEL`, `WS-ROW`, `WS-COL`) แทนชื่อสั้น ๆ อย่าง `I`, `J`, `K`
  (ที่นิยมในภาษาอื่น) เป็นธรรมเนียมที่ดีของ COBOL เพราะช่วยให้อ่านโค้ดเข้าใจได้ทันทีโดยไม่ต้องดูบริบท

### แบบฝึกหัดที่ 124.1

**โจทย์**: จงคำนวณด้วยมือว่าถ้าเปลี่ยนขอบเขตเป็น `WS-LEVEL > 3`, `WS-ROW > 4`, `WS-COL > 2` จะได้
`WS-CELL-COUNT` สุดท้ายเท่าไหร่ แล้วแก้โค้ดทดสอบว่าตรงกับที่คำนวณไว้หรือไม่

**เฉลย**: 3 × 4 × 2 = 24 รอบ ดังนั้น `WS-CELL-COUNT` สุดท้ายควรเป็น `00024`

---

## ขั้นตอนที่ 125: TEST BEFORE กับ TEST AFTER — ลำดับการตรวจสอบเงื่อนไข

### แนวคิด

ตามค่าเริ่มต้น `PERFORM VARYING` จะตรวจสอบเงื่อนไข `UNTIL` **ก่อน** รันเนื้อลูปเสมอ (เรียกว่า
`WITH TEST BEFORE` แม้จะไม่เขียนก็ได้เพราะเป็นค่า default) แต่ COBOL-85 เป็นต้นมาก็อนุญาตให้เราสลับ
พฤติกรรมเป็น `WITH TEST AFTER` ได้ ซึ่งจะ **รันเนื้อลูปก่อนอย่างน้อย 1 ครั้งเสมอ** แล้วจึงค่อยตรวจสอบ
เงื่อนไข — พฤติกรรมนี้เทียบเท่ากับ `do...while` ในภาษาอื่น

รูปแบบ:

```
PERFORM WITH TEST BEFORE VARYING ... FROM ... BY ... UNTIL ...
PERFORM WITH TEST AFTER  VARYING ... FROM ... BY ... UNTIL ...
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP125.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER-A        PIC 9(03).
       01  WS-COUNTER-B        PIC 9(03).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== TEST BEFORE (default) ===".
           MOVE 10 TO WS-COUNTER-A.
           PERFORM WITH TEST BEFORE
                   VARYING WS-COUNTER-A FROM 10 BY 1
                   UNTIL WS-COUNTER-A > 5
               DISPLAY "TEST BEFORE body ran, value=" WS-COUNTER-A
           END-PERFORM.
           DISPLAY "TEST BEFORE: body never ran, since 10 > 5 already.".

           DISPLAY "=== TEST AFTER ===".
           MOVE 10 TO WS-COUNTER-B.
           PERFORM WITH TEST AFTER
                   VARYING WS-COUNTER-B FROM 10 BY 1
                   UNTIL WS-COUNTER-B > 5
               DISPLAY "TEST AFTER body ran, value=" WS-COUNTER-B
           END-PERFORM.
           DISPLAY "TEST AFTER: ran once before condition checked.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== TEST BEFORE (default) ===
TEST BEFORE: body never ran, since 10 > 5 already.
=== TEST AFTER ===
TEST AFTER body ran, value=010
TEST AFTER: ran once before condition checked.
```

### อธิบายโค้ดทีละส่วน

- ในบล็อก `TEST BEFORE` ค่าเริ่มต้น (`FROM 10`) ทำให้เงื่อนไข `UNTIL ... > 5` เป็นจริงตั้งแต่แรก
  COBOL จึงตรวจสอบก่อนแล้ว**ไม่รันเนื้อลูปเลยแม้แต่ครั้งเดียว** — จะเห็นว่าไม่มีบรรทัด
  "TEST BEFORE body ran" ปรากฏออกมาเลยในผลลัพธ์จริง
- ในบล็อก `TEST AFTER` แม้เงื่อนไขจะเป็นจริงตั้งแต่แรกเหมือนกัน แต่ COBOL จะ**รันเนื้อลูปก่อน 1 ครั้ง
  เสมอ** แล้วจึงตรวจสอบเงื่อนไขหลังจากเพิ่มค่า จึงเห็นบรรทัด "TEST AFTER body ran, value=010" ปรากฏ
  อยู่ 1 ครั้งพอดี

### ข้อควรระวัง

- `TEST AFTER` ใช้น้อยกว่า `TEST BEFORE` มากในทางปฏิบัติ แต่มีประโยชน์มากในสถานการณ์ "ต้องทำอย่างน้อย
  1 ครั้งเสมอ" เช่น การแสดงเมนูให้ผู้ใช้เลือกอย่างน้อยหนึ่งรอบก่อนถามว่าจะออกหรือไม่ (จะใช้จริงใน
  โปรเจกต์เครื่องคิดเลขที่ Part 015)
- อย่าลืมว่าคีย์เวิร์ด `WITH TEST BEFORE`/`WITH TEST AFTER` ต้องอยู่**ก่อน** `VARYING` เสมอ ไม่ใช่
  หลัง มิฉะนั้นจะเป็น syntax error

### แบบฝึกหัดที่ 125.1

**โจทย์**: จงอธิบายว่าถ้าเปลี่ยน `FROM 10` เป็น `FROM 1` ในทั้งสองบล็อก ผลลัพธ์ของ `TEST BEFORE`
และ `TEST AFTER` จะเหมือนกันหรือต่างกัน เพราะเหตุใด

**เฉลย**: จะ**เหมือนกันทุกประการ** เพราะเมื่อค่าเริ่มต้น (1) ยังไม่ทำให้เงื่อนไขหยุดเป็นจริง ทั้งสอง
รูปแบบจะรันเนื้อลูปตั้งแต่ 1 ถึง 5 เหมือนกัน ความแตกต่างระหว่าง `TEST BEFORE` กับ `TEST AFTER` จะ
ปรากฏชัดเจน **เฉพาะกรณีที่เงื่อนไขหยุดเป็นจริงตั้งแต่ค่าเริ่มต้น** เท่านั้น

---

## ขั้นตอนที่ 126: ใช้ PERFORM VARYING ประมวลผลตารางข้อมูลอย่างง่าย

### แนวคิด

หนึ่งในประโยชน์ที่สำคัญที่สุดของ `PERFORM VARYING` คือการใช้เป็นตัวควบคุมการไล่เข้าถึงสมาชิกในตาราง
(Table/Array) ซึ่ง COBOL ประกาศด้วย clause `OCCURS` (เราจะเรียนรู้ `OCCURS` แบบละเอียดเต็มรูปแบบใน
Part 016 — ที่นี่ขอแนะนำแค่พอให้เห็นภาพการใช้งานร่วมกับ `PERFORM VARYING`)

รูปแบบการเข้าถึงสมาชิกตารางคือ `table-name(index)` โดย index เริ่มต้นที่ 1 เสมอ (ไม่ใช่ 0 เหมือนหลาย
ภาษา) และ `PERFORM VARYING` เหมาะมากสำหรับการไล่ index ตั้งแต่ 1 ถึงจำนวนสมาชิกทั้งหมด

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP126.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SCORE-TABLE.
           05  WS-SCORE        PIC 9(03) OCCURS 5 TIMES.
       01  WS-IDX              PIC 9(02).
       01  WS-TOTAL            PIC 9(05) VALUE ZERO.
       01  WS-AVERAGE          PIC 9(03)V99 VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== PERFORM VARYING over a simple table ===".
           MOVE 80 TO WS-SCORE(1).
           MOVE 95 TO WS-SCORE(2).
           MOVE 70 TO WS-SCORE(3).
           MOVE 88 TO WS-SCORE(4).
           MOVE 60 TO WS-SCORE(5).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               DISPLAY "Score(" WS-IDX ") = " WS-SCORE(WS-IDX)
               ADD WS-SCORE(WS-IDX) TO WS-TOTAL
           END-PERFORM.

           COMPUTE WS-AVERAGE = WS-TOTAL / 5.
           DISPLAY "Total = " WS-TOTAL.
           DISPLAY "Average = " WS-AVERAGE.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM VARYING over a simple table ===
Score(01) = 080
Score(02) = 095
Score(03) = 070
Score(04) = 088
Score(05) = 060
Total = 00393
Average = 078.60
```

### อธิบายโค้ดทีละส่วน

- `05 WS-SCORE PIC 9(03) OCCURS 5 TIMES.` ประกาศตารางชื่อ `WS-SCORE` ที่มี 5 ช่อง แต่ละช่องเก็บเลข
  3 หลัก — รายละเอียดเต็มรูปแบบของ `OCCURS` (เช่น `INDEXED BY`, ตารางหลายมิติ) จะสอนใน Part 016-018
- `WS-SCORE(WS-IDX)` คือการเข้าถึงสมาชิกตารางโดยใช้ **ตัวแปร** เป็น subscript ซึ่งเป็นรูปแบบเดียวกับ
  ที่เราจะใช้ตลอดเมื่อไล่ประมวลผลตาราง
- ลูปนี้ทำสองอย่างพร้อมกัน: แสดงค่าแต่ละช่อง และสะสมผลรวมไปพร้อมกันใน `WS-TOTAL` — เป็นรูปแบบมาตรฐาน
  ที่จะพบบ่อยมากตลอดหลักสูตร (accumulate-while-iterate pattern)

### ข้อควรระวัง

- Subscript ใน COBOL เริ่มต้นที่ 1 เสมอ ต่างจากภาษาอย่าง C/Java/Python ที่เริ่มที่ 0 — นี่คือจุดที่
  โปรแกรมเมอร์ที่มาจากภาษาอื่นมักเขียนลูปผิดพลาด (off-by-one) เมื่อมาเขียน COBOL
- ถ้า index เกินขอบเขตที่ประกาศไว้ (เช่น `OCCURS 5 TIMES` แต่พยายามเข้าถึง `WS-SCORE(6)`) บางครั้ง
  GnuCOBOL อาจไม่ error ให้ทันทีในโหมด compile ปกติ (ต้องเปิด flag ตรวจสอบขอบเขตเพิ่มเติม) แต่จะทำให้
  โปรแกรมอ่าน/เขียนหน่วยความจำผิดตำแหน่งซึ่งอันตรายมาก ต้องระวังขอบเขตของลูปให้ตรงกับ `OCCURS` เสมอ

### แบบฝึกหัดที่ 126.1

**โจทย์**: แก้โค้ดให้หาคะแนน**สูงสุด**ในตาราง แทนการหาผลรวม

**เฉลย**: เพิ่มตัวแปร `WS-MAX` ตั้งต้นด้วยค่าน้อยที่สุดที่เป็นไปได้ แล้วเทียบทีละรอบ:

```cobol
       01  WS-MAX              PIC 9(03) VALUE ZERO.
       ...
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               IF WS-SCORE(WS-IDX) > WS-MAX
                   MOVE WS-SCORE(WS-IDX) TO WS-MAX
               END-IF
           END-PERFORM.
           DISPLAY "Max score = " WS-MAX.
```

---

## ขั้นตอนที่ 127: ค้นหาข้อมูลในตารางด้วยลูปแบบ Manual และ EXIT PERFORM

### แนวคิด

ก่อนที่เราจะเรียนคำสั่ง `SEARCH` ที่ COBOL สร้างมาให้เฉพาะทาง (Part 017) เราสามารถเขียนการค้นหาแบบ
manual ได้ด้วย `PERFORM VARYING` ร่วมกับ `IF` และคำสั่ง `EXIT PERFORM` ซึ่งใช้ **ออกจากลูปทันที**
เมื่อเจอสิ่งที่ต้องการ (เทียบเท่ากับ `break` ในภาษาอื่น) โดยไม่ต้องรอให้ลูปวนจนครบ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP127.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SCORE-TABLE.
           05  WS-SCORE        PIC 9(03) OCCURS 5 TIMES.
       01  WS-IDX              PIC 9(02).
       01  WS-TARGET           PIC 9(03) VALUE 70.
       01  WS-FOUND-POS        PIC 9(02) VALUE ZERO.
       01  WS-FOUND-SWITCH     PIC X(01) VALUE "N".
           88  WS-FOUND                  VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Manual search with EXIT PERFORM ===".
           MOVE 80 TO WS-SCORE(1).
           MOVE 95 TO WS-SCORE(2).
           MOVE 70 TO WS-SCORE(3).
           MOVE 88 TO WS-SCORE(4).
           MOVE 60 TO WS-SCORE(5).

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               IF WS-SCORE(WS-IDX) = WS-TARGET
                   MOVE WS-IDX TO WS-FOUND-POS
                   SET WS-FOUND TO TRUE
                   DISPLAY "Found " WS-TARGET " at position " WS-IDX
                   EXIT PERFORM
               END-IF
           END-PERFORM.

           IF WS-FOUND
               DISPLAY "Search result: found at " WS-FOUND-POS
           ELSE
               DISPLAY "Search result: not found"
           END-IF.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Manual search with EXIT PERFORM ===
Found 070 at position 03
Search result: found at 03
```

### อธิบายโค้ดทีละส่วน

- `88 WS-FOUND VALUE "Y".` คือ Condition Name (level-88) ที่เราเรียนไปแล้วใน Part 010 — ใช้ทำให้โค้ด
  อ่านง่ายขึ้นเป็น `IF WS-FOUND` แทนการเทียบค่าตรง ๆ
- `EXIT PERFORM` จะหยุดลูปทันทีโดยไม่สนใจว่า `WS-IDX` จะถึงเงื่อนไข `UNTIL` หรือยัง เมื่อพบคำตอบที่
  ตำแหน่ง 3 (`WS-SCORE(3) = 70`) ลูปจะหยุดทันทีแทนที่จะไล่ต่อจนถึงตำแหน่ง 5 โดยเปล่าประโยชน์
- นี่คือ **หัวใจสำคัญของการค้นหาแบบมีประสิทธิภาพ**: หยุดทันทีที่เจอ ไม่ต้องเสียเวลาไล่ต่อ

### ข้อควรระวัง

- `EXIT PERFORM` ใช้ได้เฉพาะภายใน `PERFORM` ที่มี body เป็น inline (คือ `PERFORM ... END-PERFORM`
  แบบในตัวอย่างนี้) ถ้าเป็นการ `PERFORM paragraph-name` (ตามที่จะสอนใน Part 014) ต้องใช้เทคนิคอื่น
  ในการควบคุมการหยุดกลางคัน
- ต้องตั้งค่า switch/flag (`WS-FOUND-SWITCH`) ให้ครบทุกกรณี ทั้งกรณีเจอและไม่เจอ มิฉะนั้นค่าจากการ
  รันครั้งก่อนอาจตกค้างอยู่ (โดยเฉพาะถ้านำโค้ดนี้ไปใส่ในลูปใหญ่ที่ค้นหาซ้ำหลายรอบ)

### แบบฝึกหัดที่ 127.1

**โจทย์**: แก้โค้ดให้ค้นหาค่า 100 (ซึ่งไม่มีในตาราง) แล้วดูว่าข้อความ "not found" แสดงถูกต้องหรือไม่

**เฉลย**: เปลี่ยน `01 WS-TARGET PIC 9(03) VALUE 100.` แล้วรันใหม่ ผลลัพธ์ที่ได้จะเป็น:

```
=== Manual search with EXIT PERFORM ===
Search result: not found
```

เพราะลูปวนครบทั้ง 5 รอบโดยไม่มีรอบใดตรงเงื่อนไข `IF WS-SCORE(WS-IDX) = WS-TARGET` เลย จึงไม่มีการ
`SET WS-FOUND TO TRUE` เกิดขึ้น ค่า switch จึงยังเป็น "N" (ค่าเริ่มต้น) และเข้าเงื่อนไข `ELSE`

---

## ขั้นตอนที่ 128: ลูปซ้อนกับเงื่อนไขข้าม (Conditional Skip) ด้วย CONTINUE

### แนวคิด

บ่อยครั้งเราต้องการให้ลูปทำงานเฉพาะบางรอบตามเงื่อนไข (เช่น เฉพาะเลขคู่, เฉพาะวันที่ตรงเงื่อนไข) COBOL
ไม่มีคำสั่ง `continue` แบบภาษาสมัยใหม่ที่ "ข้ามไปรอบถัดไปทันที" แต่เราใช้ `IF...ELSE` ร่วมกับคำสั่ง
`CONTINUE` (ซึ่งแปลว่า "ไม่ต้องทำอะไร แล้วไปต่อ" — คล้าย `pass` ใน Python) เพื่อจำลองพฤติกรรมนี้ได้

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP128.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ROW              PIC 9(02).
       01  WS-COL              PIC 9(02).
       01  WS-REMAINDER        PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Nested loop with conditional skip ===".
           PERFORM VARYING WS-ROW FROM 1 BY 1 UNTIL WS-ROW > 3
               PERFORM VARYING WS-COL FROM 1 BY 1 UNTIL WS-COL > 6
                   DIVIDE WS-COL BY 2 GIVING WS-REMAINDER
                       REMAINDER WS-REMAINDER
                   IF WS-REMAINDER = 0
                       DISPLAY "  Skip odd-check, printing R"
                               WS-ROW "C" WS-COL " (even column)"
                   ELSE
                       CONTINUE
                   END-IF
               END-PERFORM
           END-PERFORM.
           DISPLAY "Done: only even columns were printed.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Nested loop with conditional skip ===
  Skip odd-check, printing R01C02 (even column)
  Skip odd-check, printing R01C04 (even column)
  Skip odd-check, printing R01C06 (even column)
  Skip odd-check, printing R02C02 (even column)
  Skip odd-check, printing R02C04 (even column)
  Skip odd-check, printing R02C06 (even column)
  Skip odd-check, printing R03C02 (even column)
  Skip odd-check, printing R03C04 (even column)
  Skip odd-check, printing R03C06 (even column)
Done: only even columns were printed.
```

### อธิบายโค้ดทีละส่วน

- `DIVIDE WS-COL BY 2 GIVING WS-REMAINDER REMAINDER WS-REMAINDER` คือการหาเศษจากการหาร (modulo)
  ที่เราเรียนไปแล้วใน Part 009 — ใช้ตรวจสอบว่า `WS-COL` เป็นเลขคู่หรือไม่ (เศษ = 0 คือเลขคู่)
- แต่ละคอลัมน์จะถูกตรวจสอบทุกรอบ (ลูปยังคงวนครบ 6 รอบต่อแถวเหมือนเดิม) เพียงแต่รอบที่เป็นเลขคี่จะ
  ไม่ทำอะไร (`CONTINUE`) แล้วปล่อยให้ลูปขยับไปรอบถัดไปตามปกติ
- ข้อแตกต่างสำคัญระหว่าง `CONTINUE` กับ `EXIT PERFORM` (จากขั้นตอนที่ 127): `CONTINUE` แค่ข้ามการ
  ทำงาน **ในรอบนั้น** แล้วไปต่อรอบถัดไป ส่วน `EXIT PERFORM` คือ**ออกจากลูปทั้งหมด**ทันที

### ข้อควรระวัง

- `CONTINUE` ในที่นี้ทำหน้าที่เป็น "no-operation statement" เฉย ๆ ในสาขา `ELSE` — บางคนอาจคิดว่า
  ตัดสาขา `ELSE` ทิ้งไปเลยได้เพราะไม่ได้ทำอะไร แต่การใส่ `CONTINUE` ไว้อย่างชัดเจนช่วยให้โค้ดสื่อ
  ความหมายว่า "ตั้งใจข้าม" ไม่ใช่ "ลืมเขียน" ซึ่งเป็นนิสัยการเขียนโค้ดที่ดี
- ระวังสับสนระหว่าง `CONTINUE` ของ COBOL กับ `continue` ของภาษา C/Java/Python — พฤติกรรมไม่เหมือนกัน
  100% เพราะ `CONTINUE` ของ COBOL ไม่ได้ "กระโดดข้ามไปยังจุดเริ่มลูปรอบถัดไป" แต่หมายถึง "ไม่มีคำสั่ง
  ให้ทำ" เฉย ๆ ซึ่งบังเอิญให้ผลลัพธ์คล้ายกันเมื่อใช้ในบริบทแบบนี้

### แบบฝึกหัดที่ 128.1

**โจทย์**: ปรับโค้ดให้พิมพ์เฉพาะคอลัมน์ที่เป็นเลข**คี่**แทน

**เฉลย**: สลับเงื่อนไขใน `IF` กับ `ELSE`:

```cobol
                   IF WS-REMAINDER NOT = 0
                       DISPLAY "  Printing R" WS-ROW "C" WS-COL
                               " (odd column)"
                   ELSE
                       CONTINUE
                   END-IF
```

---

## ขั้นตอนที่ 129: ข้อผิดพลาดที่พบบ่อยในลูปซ้อน (Infinite Loop, ตัวนับชนกัน)

### แนวคิด

ลูปซ้อนเป็นจุดที่มือใหม่ COBOL เจอบั๊กบ่อยที่สุด บทนี้จะรวบรวมข้อผิดพลาดที่พบบ่อยที่สุด พร้อมตัวอย่าง
"ก่อนแก้" และ "หลังแก้" เพื่อให้เห็นภาพชัดเจน

### ข้อผิดพลาดที่ 1: ใช้ตัวแปรตัวนับตัวเดียวกันในลูปซ้อน

```cobol
      *> WRONG: reusing WS-COUNTER for both outer and inner loop
           PERFORM VARYING WS-COUNTER FROM 1 BY 1
                   UNTIL WS-COUNTER > 3
               PERFORM VARYING WS-COUNTER FROM 1 BY 1
                       UNTIL WS-COUNTER > 3
                   DISPLAY "Value: " WS-COUNTER
               END-PERFORM
           END-PERFORM.
```

ปัญหา: ลูปในจะเปลี่ยนค่า `WS-COUNTER` จนถึง 4 แล้วออกจากลูปใน แต่ลูปนอกก็ใช้ตัวแปรเดียวกัน ทำให้
เงื่อนไข `UNTIL WS-COUNTER > 3` ของลูปนอกเป็นจริงทันทีในรอบแรก **ลูปนอกจะทำงานแค่รอบเดียว** ทั้งที่
ตั้งใจจะให้ทำ 3 รอบ

### ข้อผิดพลาดที่ 2: ลืมกำหนดค่า BY ให้ถูกทิศทาง (Infinite Loop)

```cobol
      *> WRONG: BY counts up, but UNTIL checks for a smaller value
           PERFORM VARYING WS-COUNTER FROM 10 BY 1
                   UNTIL WS-COUNTER < 1
               DISPLAY "Value: " WS-COUNTER
           END-PERFORM.
```

ปัญหา: `WS-COUNTER` เริ่มที่ 10 แล้วเพิ่มขึ้นเรื่อย ๆ (`BY 1`) แต่เงื่อนไขหยุดคือ "น้อยกว่า 1" ซึ่งจะ
ไม่มีวันเป็นจริงเลย ค่าจะวิ่งขึ้นไปเรื่อย ๆ จนกระทั่ง overflow ของ `PICTURE` (เช่น `PIC 9(03)` รับได้
สูงสุด 999) แล้วเกิดพฤติกรรมที่คาดเดาไม่ได้ ถือเป็น **infinite loop** ในทางปฏิบัติ

### เวอร์ชันที่แก้ไขแล้ว (compile และรันจริง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP129.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-OUTER            PIC 9(02).
       01  WS-INNER            PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Correct nested loop (fixed version) ===".
           PERFORM VARYING WS-OUTER FROM 1 BY 1 UNTIL WS-OUTER > 3
               PERFORM VARYING WS-INNER FROM 1 BY 1 UNTIL WS-INNER > 3
                   DISPLAY "Outer=" WS-OUTER " Inner=" WS-INNER
               END-PERFORM
           END-PERFORM.
           DISPLAY "Finished without infinite loop or index clash.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Correct nested loop (fixed version) ===
Outer=01 Inner=01
Outer=01 Inner=02
Outer=01 Inner=03
Outer=02 Inner=01
Outer=02 Inner=02
Outer=02 Inner=03
Outer=03 Inner=01
Outer=03 Inner=02
Outer=03 Inner=03
Finished without infinite loop or index clash.
```

### สรุปรายการข้อผิดพลาดที่พบบ่อยของ PERFORM VARYING

| ข้อผิดพลาด | อาการ | วิธีป้องกัน |
|---|---|---|
| ใช้ตัวแปรตัวนับซ้ำกันในลูปซ้อน | ลูปนอกทำงานผิดจำนวนรอบ (มักจะน้อยกว่าที่ควร) | ตั้งชื่อตัวนับให้ต่างกันเสมอในแต่ละชั้น |
| `BY` กับ `UNTIL` ผิดทิศทาง | ลูปไม่รู้จบ หรือ ไม่ทำงานเลยสักรอบ | ตรวจสอบว่า `BY` บวกคู่กับ `UNTIL ... >` และ `BY` ลบคู่กับ `UNTIL ... <` |
| PICTURE ของตัวนับเล็กเกินไป | ค่าล้น (overflow) แบบเงียบ ไม่มี error | คำนวณค่าสูงสุดที่จะวนถึงก่อนกำหนดขนาด PICTURE |
| นำค่าตัวนับไปใช้หลังลูปโดยไม่ระวัง | ค่าที่ได้เกินขอบเขตที่คาดไว้ 1 หน่วยเสมอ (off-by-one) | จำไว้ว่าหลังลูปจบ ตัวนับจะมีค่าที่ "เกินเงื่อนไข" แล้วเสมอ |

### ข้อควรระวัง

- ควรทดสอบลูปซ้อนด้วยขอบเขตเล็ก ๆ ก่อนเสมอ (เช่น 2-3 รอบ) เพื่อตรวจสอบว่าตรรกะถูกต้อง ก่อนขยายไปใช้
  กับข้อมูลจริงจำนวนมาก
- หากสงสัยว่าโปรแกรมค้างเพราะ infinite loop ให้กด `Ctrl+C` เพื่อหยุดโปรแกรมทันที แล้วกลับไปตรวจสอบ
  ทิศทางของ `BY` และ `UNTIL` เป็นอันดับแรก

### แบบฝึกหัดที่ 129.1

**โจทย์**: จงระบุว่าโค้ดต่อไปนี้จะเกิดปัญหาอะไร และแก้ไขให้ถูกต้อง

```cobol
       01  WS-N   PIC 9(02).
       ...
           PERFORM VARYING WS-N FROM 5 BY -1 UNTIL WS-N > 0
               DISPLAY WS-N
           END-PERFORM.
```

**เฉลย**: `WS-N` ประกาศเป็น `PIC 9(02)` (ไม่มีเครื่องหมาย `S`) แต่ใช้ `BY -1` ซึ่งพยายามลดค่าลงเรื่อย ๆ
เมื่อค่าลดต่ำกว่า 0 ตัวแปรแบบไม่มีเครื่องหมายจะไม่สามารถเก็บค่าติดลบได้ อีกทั้งเงื่อนไข
`UNTIL WS-N > 0` ก็เขียนผิดทิศทางด้วย (ควรเป็น `UNTIL WS-N < 1` สำหรับการนับถอยหลัง) วิธีแก้คือ
เปลี่ยนเป็น `01 WS-N PIC S9(02).` และแก้เงื่อนไขเป็น `UNTIL WS-N < 1`

---

## ขั้นตอนที่ 130: สรุปเทคนิค — ผสาน PERFORM VARYING กับ EVALUATE สร้างรายงานสรุป

### แนวคิด

ปิดท้าย Part นี้ด้วยการนำสิ่งที่เรียนมาทั้งหมด (PERFORM VARYING, EVALUATE จาก Part 011, ตัวแปรสะสมค่า)
มาผสมกันเป็นโปรแกรมที่ใกล้เคียงงานจริงมากขึ้น: วนลูปประมวลผลข้อมูล 7 วัน แล้วใช้ `EVALUATE` จัดหมวดหมู่
แต่ละวันเป็น "วันธรรมดา" หรือ "วันหยุดสุดสัปดาห์" พร้อมสะสมจำนวนแต่ละประเภท

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP130.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DAY              PIC 9(02).
       01  WS-DAY-TYPE         PIC X(09).
       01  WS-WEEKEND-COUNT    PIC 9(02) VALUE ZERO.
       01  WS-WEEKDAY-COUNT    PIC 9(02) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== PERFORM VARYING + EVALUATE report demo ===".
           PERFORM VARYING WS-DAY FROM 1 BY 1 UNTIL WS-DAY > 7
               EVALUATE WS-DAY
                   WHEN 1 THRU 5
                       MOVE "Weekday" TO WS-DAY-TYPE
                       ADD 1 TO WS-WEEKDAY-COUNT
                   WHEN 6
                   WHEN 7
                       MOVE "Weekend" TO WS-DAY-TYPE
                       ADD 1 TO WS-WEEKEND-COUNT
                   WHEN OTHER
                       MOVE "Invalid" TO WS-DAY-TYPE
               END-EVALUATE
               DISPLAY "Day " WS-DAY ": " WS-DAY-TYPE
           END-PERFORM.
           DISPLAY "Weekdays: " WS-WEEKDAY-COUNT
                   " Weekends: " WS-WEEKEND-COUNT.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM VARYING + EVALUATE report demo ===
Day 01: Weekday  
Day 02: Weekday  
Day 03: Weekday  
Day 04: Weekday  
Day 05: Weekday  
Day 06: Weekend  
Day 07: Weekend  
Weekdays: 05 Weekends: 02
```

(สังเกตว่ามีช่องว่างตามหลังคำว่า "Weekday"/"Weekend" ในผลลัพธ์ เพราะ `WS-DAY-TYPE` ประกาศเป็น
`PIC X(09)` แต่คำว่า "Weekday" มีแค่ 7 ตัวอักษร ส่วนที่เหลือจะถูกเติมด้วยช่องว่างขวามืออัตโนมัติ)

### อธิบายโค้ดทีละส่วน

- `PERFORM VARYING WS-DAY FROM 1 BY 1 UNTIL WS-DAY > 7` คือโครงลูปหลักที่ไล่ตั้งแต่วันที่ 1 ถึง 7
- `EVALUATE WS-DAY` ข้างในลูป ทำหน้าที่ตัดสินใจแยกประเภทของแต่ละวันที่กำลังประมวลผลอยู่ ณ ขณะนั้น
  — สังเกตว่า `WHEN 6` กับ `WHEN 7` เขียนต่อกันโดยไม่มีคำสั่งคั่นกลาง หมายถึง "ถ้าตรงกับ 6 **หรือ** 7"
  ให้ทำชุดคำสั่งเดียวกัน (เทคนิคเดียวกับที่เรียนใน Part 011)
- ลูปกับเงื่อนไขทำงานร่วมกันอย่างเป็นธรรมชาติ: ลูปควบคุม "ทำกี่รอบ" ส่วน `EVALUATE` ควบคุม "แต่ละรอบ
  ทำอะไรบ้าง" — รูปแบบนี้คือโครงสร้างพื้นฐานของโปรแกรมประมวลผลข้อมูลแบบ Batch เกือบทุกโปรแกรมในโลก
  ธุรกิจจริง

### ข้อควรระวัง

- ระวังอย่าสับสนระหว่าง `EVALUATE` ที่ใช้ตรวจสอบ "ค่าของตัวแปรลูป" (เหมือนตัวอย่างนี้) กับการ
  ใช้ `EVALUATE TRUE` เพื่อตรวจสอบหลายเงื่อนไขที่ไม่เกี่ยวกับตัวแปรเดียวกัน (ทบทวนได้ใน Part 011)
- เมื่อโปรแกรมมีทั้งลูปและเงื่อนไขซับซ้อนปนกันมาก ควรพิจารณาแยกส่วน `EVALUATE` ออกเป็น Paragraph ย่อย
  เพื่อให้ `PROCEDURE DIVISION` อ่านง่ายขึ้น (เนื้อหานี้จะสอนอย่างละเอียดใน Part 014 ถัดไป)

### แบบฝึกหัดที่ 130.1

**โจทย์**: ขยายโปรแกรมให้ประมวลผล 10 วันแทน 7 วัน (สมมติว่าวันที่ 8-10 เป็นวันหยุดพิเศษ ให้จัดเป็น
"Weekend" เหมือนกัน)

**เฉลย**:

```cobol
           PERFORM VARYING WS-DAY FROM 1 BY 1 UNTIL WS-DAY > 10
               EVALUATE WS-DAY
                   WHEN 1 THRU 5
                       MOVE "Weekday" TO WS-DAY-TYPE
                       ADD 1 TO WS-WEEKDAY-COUNT
                   WHEN 6 THRU 10
                       MOVE "Weekend" TO WS-DAY-TYPE
                       ADD 1 TO WS-WEEKEND-COUNT
                   WHEN OTHER
                       MOVE "Invalid" TO WS-DAY-TYPE
               END-EVALUATE
               DISPLAY "Day " WS-DAY ": " WS-DAY-TYPE
           END-PERFORM.
```

โดยเปลี่ยนเงื่อนไขจาก `WHEN 6` / `WHEN 7` แยกกัน มาเป็น `WHEN 6 THRU 10` เพื่อครอบคลุมช่วงวันที่ 6-10
ในคำสั่งเดียว

---

## สรุปท้ายบท

ใน Part นี้ คุณได้เรียนรู้:

- รูปแบบพื้นฐานของ `PERFORM VARYING ... FROM ... BY ... UNTIL` และกลไกการทำงานของแต่ละส่วน
- การปรับ `FROM`/`BY` เพื่อวนลูปแบบก้าวกระโดดหรือแบบนับถอยหลัง และความสำคัญของ `PICTURE` แบบมีเครื่องหมาย
- การวนลูปซ้อน 2 ชั้นด้วย `AFTER` และการวนลูปซ้อน 2-3 ชั้นด้วยการเขียน `PERFORM VARYING` ซ้อนกันโดยตรง
- ความแตกต่างระหว่าง `WITH TEST BEFORE` (ค่าเริ่มต้น) กับ `WITH TEST AFTER`
- การใช้ `PERFORM VARYING` ไล่ประมวลผลตาราง (`OCCURS`) เบื้องต้น เตรียมพื้นฐานสำหรับ Part 016
- การค้นหาแบบ manual ด้วย `IF` + `EXIT PERFORM` และการข้ามรอบด้วย `CONTINUE`
- ข้อผิดพลาดที่พบบ่อยที่สุดของลูปซ้อน (ตัวนับชนกัน, ทิศทาง BY/UNTIL ผิด, PICTURE เล็กเกินไป)
- การผสาน `PERFORM VARYING` กับ `EVALUATE` เพื่อสร้างโปรแกรมประมวลผลข้อมูลแบบวนลูปพร้อมจัดหมวดหมู่

ทักษะการวนลูปที่คุณเพิ่งฝึกมาทั้งหมดนี้จะถูกนำไปใช้อย่างเข้มข้นใน Part ถัดไป เมื่อเราเรียนรู้วิธี
**จัดโครงสร้างโปรแกรมให้เป็นระเบียบ** ด้วย Paragraph, Section และ `PERFORM...THRU` — ซึ่งจะทำให้โปรแกรม
ที่มีลูปซับซ้อนหลายจุดอ่านและดูแลรักษาง่ายขึ้นมาก ก่อนที่เราจะนำทุกอย่างมารวมกันในโปรเจกต์รวบยอด
เฟส 1 ที่ Part 015

**[← กลับไป Part 012: PERFORM พื้นฐานและ PERFORM UNTIL](part-012-perform-until.md)** | **[ไปยัง Part 014: โครงสร้างโปรแกรมแบบมีโมดูล →](part-014-paragraphs-sections.md)**
