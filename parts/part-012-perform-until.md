# Part 012: PERFORM พื้นฐานและ PERFORM UNTIL (ลูป) (ขั้นตอนที่ 111–120)

## คำนำของ Part นี้

ใน Part 011 เราได้เรียนรู้ `EVALUATE` ซึ่งช่วยให้เราตัดสินใจเลือกทำสิ่งต่าง ๆ ตามเงื่อนไขได้อย่างเป็นระเบียบ
แต่ทุกโปรแกรมที่เราเขียนมาจนถึงตอนนี้ยังทำงานแบบ "เส้นตรง" คือรันคำสั่งจากบนลงล่างเพียงรอบเดียวแล้วจบ
ในโลกความจริง งานส่วนใหญ่ที่คอมพิวเตอร์ทำนั้นเป็นงาน**ที่ต้องทำซ้ำ ๆ** เช่น อ่านข้อมูลลูกค้าทีละคนจน
หมดไฟล์ พิมพ์ใบเสร็จทีละบรรทัดจนครบรายการสินค้า หรือคำนวณดอกเบี้ยทบต้นทีละเดือนจนครบสัญญา — งานเหล่านี้
ต้องอาศัย **การวนลูป (Looping)** ซึ่งเป็นหนึ่งในสามโครงสร้างพื้นฐานของการเขียนโปรแกรม (Sequence, Selection,
Iteration) ที่เรายังไม่เคยเรียนมาก่อนในหลักสูตรนี้

COBOL ใช้คำสั่งเดียวสำหรับการวนลูปทุกรูปแบบ นั่นคือ `PERFORM` แต่ `PERFORM` มีหน้าตาและวิธีใช้ได้หลาย
รูปแบบมาก ตั้งแต่การเรียก paragraph ธรรมดาเพียงครั้งเดียว ไปจนถึงการวนซ้ำแบบนับจำนวนรอบตายตัว และการวน
ซ้ำแบบตรวจสอบเงื่อนไข Part นี้จะพาคุณทำความรู้จักกับ `PERFORM` ในภาพรวมทั้งหมดก่อน แล้วเจาะลึกเฉพาะรูปแบบ
`PERFORM UNTIL` ซึ่งเป็นรูปแบบการวนลูปที่ยืดหยุ่นที่สุดและใช้บ่อยที่สุดในโปรแกรม COBOL จริง ส่วนรูปแบบ
`PERFORM VARYING` ที่ทรงพลังกว่าสำหรับการวนลูปแบบนับค่า (คล้าย `for` ในภาษาอื่น) จะแยกไปเรียนอย่างละเอียด
เต็มรูปแบบใน Part 013 ถัดไป

---

## ขั้นตอนที่ 111: ภาพรวมของ PERFORM — 4 รูปแบบหลักที่ต้องรู้จัก

### แนวคิด

`PERFORM` คือคำสั่งเดียวใน COBOL ที่ใช้ทำทั้ง "การเรียกทำงานซ้ำ" (loop) และ "การเรียก paragraph ไปทำงาน
แล้วกลับมา" (คล้าย function call) ทำให้ผู้เริ่มต้นมักสับสนว่าทำไมคำสั่งเดียวมีหน้าตาต่างกันได้หลายแบบ
สรุปรูปแบบหลักที่ COBOL รองรับมีดังนี้:

| รูปแบบ | ตัวอย่าง | ความหมาย |
|---|---|---|
| **PERFORM แบบเปล่า** | `PERFORM SHOW-HELLO.` | เรียก paragraph `SHOW-HELLO` ไปทำงาน**หนึ่งครั้ง** แล้วกลับมาทำคำสั่งถัดไป |
| **PERFORM ... TIMES** | `PERFORM SHOW-HELLO 3 TIMES.` | เรียกทำงานซ้ำ**ตามจำนวนรอบที่กำหนดตายตัว** |
| **PERFORM ... UNTIL** | `PERFORM UNTIL cond ... END-PERFORM` | เรียกทำงานซ้ำ**จนกว่าเงื่อนไขจะเป็นจริง** (เนื้อหาหลักของ Part นี้) |
| **PERFORM ... VARYING** | `PERFORM VARYING i FROM 1 BY 1 UNTIL i > 10` | ผสาน FROM/BY/UNTIL ไว้ในคำสั่งเดียว (เรียนละเอียดใน Part 013) |

นอกจากนี้ `PERFORM` ยังแบ่งได้เป็น 2 สไตล์การเขียนโครงสร้าง:

- **Out-of-line PERFORM**: เรียกชื่อ paragraph ที่ประกาศแยกไว้ต่างหาก (เช่น `PERFORM SHOW-HELLO.`)
  รูปแบบนี้จะเรียนอย่างละเอียดเต็มรูปแบบใน Part 014 ร่วมกับ `PERFORM ... THRU`
- **Inline PERFORM**: เขียนชุดคำสั่งไว้ระหว่าง `PERFORM` กับ `END-PERFORM` โดยตรง ไม่ต้องแยก paragraph
  (เช่นตัวอย่าง `PERFORM UNTIL ... END-PERFORM` ด้านบน) นิยมใช้มากในโค้ดสมัยใหม่เพราะอ่านง่าย เห็นเนื้อลูป
  อยู่ในที่เดียวกัน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP111.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER          PIC 9(02) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Overview of PERFORM forms ===".

           DISPLAY "-- 1. Out-of-line PERFORM (call paragraph once) --".
           PERFORM SHOW-HELLO.

           DISPLAY "-- 2. PERFORM ... TIMES (fixed count) --".
           PERFORM SHOW-HELLO 3 TIMES.

           DISPLAY "-- 3. PERFORM ... UNTIL (condition-based loop) --".
           MOVE 1 TO WS-COUNTER.
           PERFORM UNTIL WS-COUNTER > 3
               DISPLAY "  Loop tick " WS-COUNTER
               ADD 1 TO WS-COUNTER
           END-PERFORM.

           DISPLAY "-- 4. PERFORM VARYING (full detail in Part 013) --".
           DISPLAY "  (combines FROM/BY/UNTIL into a single statement)".

           STOP RUN.

       SHOW-HELLO.
           DISPLAY "  Hello from the SHOW-HELLO paragraph.".
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step111 step111.cob
./step111
```

### ผลลัพธ์จริงที่ได้

```
=== Overview of PERFORM forms ===
-- 1. Out-of-line PERFORM (call paragraph once) --
  Hello from the SHOW-HELLO paragraph.
-- 2. PERFORM ... TIMES (fixed count) --
  Hello from the SHOW-HELLO paragraph.
  Hello from the SHOW-HELLO paragraph.
  Hello from the SHOW-HELLO paragraph.
-- 3. PERFORM ... UNTIL (condition-based loop) --
  Loop tick 01
  Loop tick 02
  Loop tick 03
-- 4. PERFORM VARYING (full detail in Part 013) --
  (combines FROM/BY/UNTIL into a single statement)
```

### อธิบายโค้ดทีละส่วน

- `PERFORM SHOW-HELLO.` ตัวแรกสุด เรียก paragraph ไปทำงานแค่ **1 ครั้ง** แล้วควบคุมกลับมาที่บรรทัดถัดไป
  ใน `MAIN-LOGIC` ทันที — สังเกตว่าผลลัพธ์มีข้อความจาก `SHOW-HELLO` ปรากฏเพียงบรรทัดเดียวหลังหัวข้อที่ 1
- `PERFORM SHOW-HELLO 3 TIMES.` เรียก paragraph เดียวกันแต่ทำซ้ำ 3 รอบติดกัน โดยไม่ต้องเขียน `PERFORM`
  สามครั้งแยกกัน
- `PERFORM UNTIL WS-COUNTER > 3 ... END-PERFORM` คือ Inline PERFORM ที่มีเนื้อลูปอยู่ระหว่าง `PERFORM`
  และ `END-PERFORM` โดยตรง ไม่ต้องแยกเป็น paragraph ต่างหาก — นี่คือรูปแบบหลักที่ Part นี้จะเจาะลึกต่อไป
- `SHOW-HELLO` ท้ายโปรแกรมคือ paragraph ธรรมดา ไม่มีอะไรพิเศษ เพียงแค่มีชื่อให้ `PERFORM` เรียกใช้ได้

### ข้อควรระวัง

- อย่าสับสนระหว่าง "PERFORM แบบเปล่า" (เรียกครั้งเดียว) กับ "PERFORM ... UNTIL" (เรียกซ้ำจนกว่าเงื่อนไข
  จะจริง) — ผู้เริ่มต้นบางคนเข้าใจผิดว่า `PERFORM paragraph-name.` จะวนซ้ำเองโดยอัตโนมัติ ซึ่งไม่จริง
  มันจะเรียกแค่ครั้งเดียวเสมอ เว้นแต่จะมีคำว่า `TIMES`, `UNTIL`, หรือ `VARYING` กำกับอยู่ด้วย
- Inline PERFORM ทุกตัว **ต้องปิดด้วย `END-PERFORM`** เสมอ (Scope Terminator จาก COBOL-85) มิฉะนั้น
  COBOL อาจตีความขอบเขตของลูปผิดไปรวมกับคำสั่งถัดไปโดยไม่ตั้งใจ

### แบบฝึกหัดที่ 111.1

**โจทย์**: จงเพิ่ม paragraph ใหม่ชื่อ `SHOW-BYE` ที่แสดงข้อความ "Goodbye!" แล้วเรียกใช้ด้วย
`PERFORM SHOW-BYE 2 TIMES` ก่อน `STOP RUN`

**เฉลย**:

```cobol
           PERFORM SHOW-BYE 2 TIMES.
           STOP RUN.

       SHOW-BYE.
           DISPLAY "  Goodbye!".
```

ผลลัพธ์ที่ได้จะมีบรรทัด "  Goodbye!" ปรากฏสองครั้งก่อนโปรแกรมจบ

---

## ขั้นตอนที่ 112: PERFORM ... TIMES — วนซ้ำแบบนับจำนวนรอบตายตัว

### แนวคิด

`PERFORM ... TIMES` เหมาะสำหรับกรณีที่เรา**รู้จำนวนรอบที่แน่นอนล่วงหน้า** และไม่จำเป็นต้องใช้ตัวแปรตัวนับ
เลย (ต่างจาก `PERFORM VARYING` ที่ต้องมีตัวแปรนับเสมอ) รูปแบบคือ:

```
PERFORM identifier-1 OR literal-1 TIMES
    <statements>
END-PERFORM
```

หรือใช้เรียก paragraph:

```
PERFORM paragraph-name literal-1 TIMES.
```

จำนวนรอบสามารถเป็นค่าคงที่ (literal) หรือค่าจากตัวแปรก็ได้ ถ้าค่าที่ระบุเป็น 0 หรือติดลบ ลูปจะไม่ทำงาน
เลยแม้แต่รอบเดียว (ไม่ error แต่อย่างใด)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP112.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STAR-LINE        PIC X(20) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== PERFORM ... TIMES basic demo ===".

           DISPLAY "-- 1. Inline PERFORM 4 TIMES --".
           PERFORM 4 TIMES
               DISPLAY "  Beep!"
           END-PERFORM.

           DISPLAY "-- 2. Out-of-line PERFORM paragraph 3 TIMES --".
           PERFORM PRINT-STAR-LINE 3 TIMES.

           DISPLAY "Done.".
           STOP RUN.

       PRINT-STAR-LINE.
           DISPLAY "  **********".
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM ... TIMES basic demo ===
-- 1. Inline PERFORM 4 TIMES --
  Beep!
  Beep!
  Beep!
  Beep!
-- 2. Out-of-line PERFORM paragraph 3 TIMES --
  **********
  **********
  **********
Done.
```

### อธิบายโค้ดทีละส่วน

- `PERFORM 4 TIMES ... END-PERFORM` คือ Inline PERFORM ที่ไม่มีตัวแปรตัวนับให้เห็นเลย เห็นแค่ตัวเลข 4
  บอกจำนวนรอบตรง ๆ — เหมาะมากสำหรับกรณีที่เนื้อลูปไม่ได้ใช้ค่าลำดับรอบเลย เช่นการพิมพ์เสียงบี๊บซ้ำ ๆ
- `PERFORM PRINT-STAR-LINE 3 TIMES.` คือ Out-of-line PERFORM ที่เรียก paragraph `PRINT-STAR-LINE`
  ให้ทำงานซ้ำ 3 รอบติดกัน โดยไม่ต้องมีตัวแปรตัวนับเช่นกัน
- สังเกตว่า `WS-STAR-LINE` ที่ประกาศไว้ไม่ได้ถูกใช้งานจริงในตัวอย่างนี้ — เป็นเพียงตัวอย่างให้เห็นว่า
  การประกาศตัวแปรที่ไม่จำเป็นสำหรับ `PERFORM TIMES` ก็ไม่ได้ทำให้เกิด error แต่อย่างใด (แม้จะไม่ใช่
  ธรรมเนียมที่ดีในโค้ดจริง ควรลบตัวแปรที่ไม่ได้ใช้ทิ้งเสมอ)

### ข้อควรระวัง

- `PERFORM ... TIMES` ไม่ให้ค่าตัวนับรอบมาใช้งานในตัวเอง ถ้าต้องการรู้ว่ากำลังอยู่รอบที่เท่าไหร่ระหว่าง
  การวนลูป ต้องประกาศตัวแปรเองแล้วเพิ่มค่าด้วยตนเอง หรือใช้ `PERFORM VARYING` แทน (Part 013) ซึ่งให้
  ค่าตัวนับมาโดยอัตโนมัติ
- ถ้าตัวเลขจำนวนรอบมาจากตัวแปร และตัวแปรนั้นมีค่าติดลบหรือศูนย์โดยไม่ตั้งใจ (เช่น อ่านมาจากไฟล์ที่ข้อมูล
  ผิดพลาด) ลูปจะไม่ทำงานเลยโดยไม่มี error แจ้งเตือน ซึ่งอาจทำให้โปรแกรม "เงียบ ๆ ไม่ทำงานตามที่คาดหวัง"
  โดยหาสาเหตุยาก

### แบบฝึกหัดที่ 112.1

**โจทย์**: แก้ไขให้ `PRINT-STAR-LINE` แสดงเส้นดาว 5 ครั้งแทนที่จะเป็น 3 ครั้ง โดยใช้ตัวแปรเก็บจำนวนรอบ
แทนการเขียนตัวเลขคงที่

**เฉลย**:

```cobol
       01  WS-REPEAT-COUNT     PIC 9(02) VALUE 5.
       ...
           MOVE 5 TO WS-REPEAT-COUNT.
           PERFORM PRINT-STAR-LINE WS-REPEAT-COUNT TIMES.
```

`PERFORM ... TIMES` รับได้ทั้งตัวเลขคงที่และตัวแปร ทำให้ยืดหยุ่นตามค่าที่กำหนดในรันไทม์ได้

---

## ขั้นตอนที่ 113: PERFORM UNTIL — วนลูปแบบตรวจสอบเงื่อนไข (รูปแบบพื้นฐาน)

### แนวคิด

`PERFORM UNTIL` คือหัวใจของ Part นี้ ใช้เมื่อเราต้องการวนลูป**จนกว่าเงื่อนไขจะเป็นจริง** โดยไม่จำเป็นต้อง
รู้จำนวนรอบล่วงหน้าเป๊ะ ๆ (ต่างจาก `TIMES`) รูปแบบพื้นฐาน:

```
PERFORM UNTIL condition
    <statements>
END-PERFORM
```

จุดสำคัญที่สุดที่ต้องจำ: `PERFORM UNTIL` **ไม่มีกลไกเพิ่มค่าตัวแปรให้อัตโนมัติ** เหมือน `PERFORM VARYING`
ผู้เขียนโปรแกรมต้อง**รับผิดชอบเพิ่มค่าตัวแปรควบคุมเงื่อนไขเอง**ด้วยคำสั่งเช่น `ADD` ภายในเนื้อลูป
มิฉะนั้นเงื่อนไขจะไม่มีวันเป็นจริง และเกิด **infinite loop (ลูปไม่รู้จบ)**

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP113.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNT            PIC 9(02) VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== PERFORM UNTIL basic demo ===".
           DISPLAY "Before loop, WS-COUNT = " WS-COUNT.

           PERFORM UNTIL WS-COUNT > 5
               DISPLAY "Now processing item " WS-COUNT
               ADD 1 TO WS-COUNT
           END-PERFORM.

           DISPLAY "After loop, WS-COUNT = " WS-COUNT.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM UNTIL basic demo ===
Before loop, WS-COUNT = 01
Now processing item 01
Now processing item 02
Now processing item 03
Now processing item 04
Now processing item 05
After loop, WS-COUNT = 06
```

### อธิบายโค้ดทีละส่วน

- `WS-COUNT` ประกาศพร้อมค่าเริ่มต้น `VALUE 1` แทนที่จะกำหนดค่าใน `PROCEDURE DIVISION` แยก ก็ทำได้เช่นกัน
  แต่ในกรณีที่ต้องรีเซ็ตค่าซ้ำหลายรอบ (เช่นเรียก paragraph นี้หลายครั้ง) ควรมี `MOVE` กำหนดค่าเริ่มต้น
  อย่างชัดเจนใน `PROCEDURE DIVISION` ด้วยเสมอ
- `PERFORM UNTIL WS-COUNT > 5` จะตรวจสอบเงื่อนไข**ก่อน**ทำงานทุกรอบ (ค่าเริ่มต้นคือ `TEST BEFORE`
  เหมือนกับ `PERFORM VARYING` ที่เรียนใน Part 013) — เมื่อ `WS-COUNT` เป็น 1 (ไม่มากกว่า 5) จึงรันเนื้อลูป
- `ADD 1 TO WS-COUNT` คือ**หัวใจสำคัญที่สุด**ของลูปนี้ — ถ้าลืมบรรทัดนี้ไป `WS-COUNT` จะค้างที่ 1 ตลอดไป
  และเงื่อนไข `WS-COUNT > 5` จะไม่มีวันเป็นจริง ทำให้เกิด infinite loop ทันที
- ผลลัพธ์สุดท้ายหลังลูปจบ `WS-COUNT = 06` (ไม่ใช่ 05) ด้วยเหตุผลเดียวกับที่อธิบายใน Part 013:
  COBOL เพิ่มค่าอีกหนึ่งครั้งก่อนตรวจพบว่าเงื่อนไขเป็นจริงแล้วจึงออกจากลูป

### ข้อควรระวัง

- **นี่คือข้อผิดพลาดอันดับหนึ่งของผู้เริ่มต้นเขียน `PERFORM UNTIL`**: ลืมเพิ่มค่าตัวแปรควบคุมเงื่อนไข
  ภายในเนื้อลูป ทำให้เกิดลูปไม่รู้จบ ต่างจาก `PERFORM VARYING` ที่ COBOL จัดการเพิ่มค่าให้อัตโนมัติผ่าน
  วลี `BY` — `PERFORM UNTIL` ไม่มีกลไกแบบนั้น ทุกอย่างต้องเขียนเองทั้งหมด
- ควรวางคำสั่งที่เปลี่ยนแปลงค่าตัวแปรเงื่อนไข (`ADD`, `MOVE` ฯลฯ) ไว้ใน**ตำแหน่งที่ชัดเจน**ภายในเนื้อลูป
  เสมอ (นิยมวางไว้ท้ายสุดของเนื้อลูปเพื่อให้อ่านง่ายว่า "ทำงานเสร็จแล้วค่อยขยับตัวนับ")

### แบบฝึกหัดที่ 113.1

**โจทย์**: แก้ไขโค้ดให้วนลูปแสดงข้อความตั้งแต่ item 1 ถึง item 8 แทนที่จะเป็น 1 ถึง 5

**เฉลย**: เปลี่ยนเงื่อนไขในบรรทัด `PERFORM UNTIL WS-COUNT > 5` เป็น `PERFORM UNTIL WS-COUNT > 8`
เท่านั้น ส่วนที่เหลือไม่ต้องแก้ไขอะไรเพิ่มเติม

---

## ขั้นตอนที่ 114: TEST BEFORE กับ TEST AFTER ใน PERFORM UNTIL

### แนวคิด

เช่นเดียวกับ `PERFORM VARYING` (Part 013) คำสั่ง `PERFORM UNTIL` ก็รองรับการกำหนดจังหวะการตรวจสอบเงื่อนไข
ได้สองแบบ:

- `WITH TEST BEFORE` (ค่าเริ่มต้น แม้ไม่เขียนก็ได้): ตรวจสอบเงื่อนไข**ก่อน**รันเนื้อลูปทุกรอบ ถ้าเงื่อนไข
  เป็นจริงตั้งแต่แรก เนื้อลูปจะ**ไม่ทำงานเลยแม้แต่ครั้งเดียว**
- `WITH TEST AFTER`: รันเนื้อลูป**ก่อนอย่างน้อย 1 ครั้งเสมอ** แล้วจึงตรวจสอบเงื่อนไขทีหลัง (เทียบเท่ากับ
  `do...while` ในภาษาอื่น) — มีประโยชน์มากเมื่อต้องการให้ทำงานอย่างน้อยหนึ่งรอบเสมอ เช่น แสดงเมนูให้ผู้ใช้
  เลือกก่อนถามว่าจะออกหรือไม่

รูปแบบ:

```
PERFORM WITH TEST BEFORE UNTIL condition ... END-PERFORM
PERFORM WITH TEST AFTER  UNTIL condition ... END-PERFORM
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP114.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNT-A          PIC 9(02).
       01  WS-COUNT-B          PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== TEST BEFORE (default) ===".
           MOVE 10 TO WS-COUNT-A.
           PERFORM WITH TEST BEFORE UNTIL WS-COUNT-A > 5
               DISPLAY "  TEST BEFORE body ran, value=" WS-COUNT-A
               ADD 1 TO WS-COUNT-A
           END-PERFORM.
           DISPLAY "TEST BEFORE: body never ran (10 > 5 already).".

           DISPLAY "=== TEST AFTER ===".
           MOVE 10 TO WS-COUNT-B.
           PERFORM WITH TEST AFTER UNTIL WS-COUNT-B > 5
               DISPLAY "  TEST AFTER body ran, value=" WS-COUNT-B
               ADD 1 TO WS-COUNT-B
           END-PERFORM.
           DISPLAY "TEST AFTER: body ran once before the check.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== TEST BEFORE (default) ===
TEST BEFORE: body never ran (10 > 5 already).
=== TEST AFTER ===
  TEST AFTER body ran, value=10
TEST AFTER: body ran once before the check.
```

### อธิบายโค้ดทีละส่วน

- ทั้งสองบล็อกเริ่มต้นด้วยค่า 10 ซึ่งทำให้เงื่อนไข `> 5` เป็นจริงตั้งแต่แรกเหมือนกัน
- บล็อก `TEST BEFORE` ตรวจเงื่อนไขก่อนแล้วพบว่าจริงทันที จึง**ข้ามเนื้อลูปไปทั้งหมด** — จะเห็นว่าไม่มี
  บรรทัด "TEST BEFORE body ran" ปรากฏในผลลัพธ์เลยแม้แต่บรรทัดเดียว
- บล็อก `TEST AFTER` รันเนื้อลูปก่อนเสมออย่างน้อย 1 ครั้ง แล้วจึงตรวจสอบเงื่อนไขหลังจากเพิ่มค่า จึงเห็น
  บรรทัด "TEST AFTER body ran, value=10" ปรากฏอยู่ 1 ครั้งพอดี ก่อนที่ลูปจะหยุด

### ข้อควรระวัง

- คีย์เวิร์ด `WITH TEST BEFORE`/`WITH TEST AFTER` ต้องอยู่**ก่อน** `UNTIL` เสมอในไวยากรณ์ (เขียนว่า
  `PERFORM WITH TEST AFTER UNTIL ...` ไม่ใช่ `PERFORM UNTIL ... WITH TEST AFTER`) มิฉะนั้นจะเกิด
  syntax error
- `TEST AFTER` มีประโยชน์มากในโปรแกรมที่มีเมนูโต้ตอบกับผู้ใช้ (เช่นโปรเจกต์เครื่องคิดเลขใน Part 015)
  เพราะเมนูควรแสดงอย่างน้อยหนึ่งครั้งเสมอ ก่อนที่จะถามว่าผู้ใช้ต้องการออกหรือไม่

### แบบฝึกหัดที่ 114.1

**โจทย์**: จงอธิบายว่าถ้าเปลี่ยนค่าเริ่มต้นจาก `MOVE 10` เป็น `MOVE 1` ในทั้งสองบล็อก ผลลัพธ์ของ
`TEST BEFORE` และ `TEST AFTER` จะเหมือนกันหรือต่างกัน

**เฉลย**: จะ**เหมือนกันทุกประการ** เพราะเมื่อค่าเริ่มต้น (1) ยังไม่ทำให้เงื่อนไขหยุดเป็นจริงตั้งแต่แรก
ทั้งสองรูปแบบจะรันเนื้อลูปตั้งแต่ 1 ถึง 5 เหมือนกันทุกรอบ ความแตกต่างระหว่าง `TEST BEFORE` กับ
`TEST AFTER` จะปรากฏชัดเจน **เฉพาะกรณีที่เงื่อนไขเป็นจริงตั้งแต่ค่าเริ่มต้น** เท่านั้น (จุดนี้สอดคล้องกับ
พฤติกรรมของ `PERFORM VARYING WITH TEST AFTER` ที่เรียนใน Part 013 ทุกประการ)

---

## ขั้นตอนที่ 115: รูปแบบลูปยอดนิยม — วนอ่านข้อมูลจนกว่าจะเจอ Sentinel/Flag

### แนวคิด

หนึ่งในการใช้งาน `PERFORM UNTIL` ที่พบบ่อยที่สุดในโปรแกรมประมวลผลข้อมูลจริง คือการวนอ่านข้อมูลไปเรื่อย ๆ
**จนกว่าจะเจอสัญญาณบอกจบ (Sentinel Value)** หรือจนกว่า **Flag/Switch** จะถูกตั้งค่าเป็น "จบแล้ว"
รูปแบบนี้เรียกว่า **Sentinel-controlled loop** และเป็นต้นแบบของการอ่านไฟล์ในภายหลัง (Part 025 เป็นต้นไป)
ที่ใช้ `PERFORM UNTIL WS-EOF-FLAG = "Y"` เพื่ออ่านไฟล์ไปเรื่อย ๆ จนสุดไฟล์

ค่า Sentinel คือค่าพิเศษที่ตกลงกันไว้ล่วงหน้าว่า "เมื่อเจอค่านี้ ให้หยุดอ่านทันที" (ในตัวอย่างนี้ใช้ 9999
เป็นค่าที่ไม่ใช่ยอดสั่งซื้อจริง)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP115.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-TABLE.
           05  WS-ORDER-AMT    PIC 9(05) OCCURS 6 TIMES
                               VALUE ZERO.
       01  WS-IDX              PIC 9(02) VALUE 1.
       01  WS-EOF-FLAG         PIC X(01) VALUE "N".
           88  WS-END-OF-DATA          VALUE "Y".
       01  WS-TOTAL            PIC 9(06) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Sentinel-controlled loop demo ===".
           MOVE 1500 TO WS-ORDER-AMT(1).
           MOVE 2200 TO WS-ORDER-AMT(2).
           MOVE 9999 TO WS-ORDER-AMT(3).
           MOVE 3000 TO WS-ORDER-AMT(4).
           MOVE 1800 TO WS-ORDER-AMT(5).
           MOVE 0 TO WS-ORDER-AMT(6).

           PERFORM UNTIL WS-END-OF-DATA
               IF WS-ORDER-AMT(WS-IDX) = 9999
                   SET WS-END-OF-DATA TO TRUE
                   DISPLAY "  Sentinel 9999 found, stop reading."
               ELSE
                   DISPLAY "  Order amount: " WS-ORDER-AMT(WS-IDX)
                   ADD WS-ORDER-AMT(WS-IDX) TO WS-TOTAL
                   ADD 1 TO WS-IDX
               END-IF
           END-PERFORM.

           DISPLAY "Total (before sentinel) = " WS-TOTAL.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Sentinel-controlled loop demo ===
  Order amount: 01500
  Order amount: 02200
  Sentinel 9999 found, stop reading.
Total (before sentinel) = 003700
```

### อธิบายโค้ดทีละส่วน

- `01 WS-EOF-FLAG PIC X(01) VALUE "N". 88 WS-END-OF-DATA VALUE "Y".` คือ Flag/Switch พร้อม
  Condition Name (level-88) ที่เราเรียนไปแล้วใน Part 010 — ใช้เพื่อให้โค้ดอ่านว่า
  `PERFORM UNTIL WS-END-OF-DATA` สื่อความหมายชัดเจนว่า "ทำจนกว่าจะถึงจุดสิ้นสุดข้อมูล"
- ตารางข้อมูล (`WS-ORDER-AMT`) มี 6 ช่อง แต่โปรแกรมประมวลผลจริงแค่ 2 ช่องแรกเท่านั้น เพราะช่องที่ 3 คือ
  ค่า Sentinel (9999) ที่บอกให้หยุด — ข้อมูลหลังจากนั้น (ช่องที่ 4, 5, 6) จึงไม่ถูกประมวลผลเลย
- `SET WS-END-OF-DATA TO TRUE` เป็นวิธีที่สะอาดกว่าการเขียน `MOVE "Y" TO WS-EOF-FLAG` ตรง ๆ เพราะสื่อ
  ความหมายชัดเจนกว่า (ทบทวนได้จาก Part 010)
- สังเกตว่า `ADD 1 TO WS-IDX` อยู่ใน `ELSE` เท่านั้น ไม่ใช่นอก `IF` — เพราะถ้าเจอ Sentinel แล้ว เราไม่
  ต้องการขยับ index ต่อ (จะออกจากลูปอยู่แล้ว)

### ข้อควรระวัง

- รูปแบบนี้เป็น**ต้นแบบสำคัญ**ที่จะกลับมาใช้ซ้ำตลอดทั้งหลักสูตร โดยเฉพาะเมื่อเราเรียนการอ่านไฟล์แบบ
  ลำดับ (Sequential File) ใน Part 023-025 ที่ใช้ `PERFORM UNTIL WS-EOF-FLAG = "Y"` ควบคู่กับคำสั่ง
  `READ ... AT END SET WS-EOF-FLAG TO "Y"` เป๊ะ ๆ ตามแบบแผนนี้
- ต้องระวังไม่ให้ค่า Sentinel ชนกับข้อมูลจริงที่เป็นไปได้ (เช่น ถ้ายอดสั่งซื้อจริงมีโอกาสเป็น 9999 ได้
  จริง จะต้องเลือกค่า Sentinel อื่นที่ไม่มีทางเกิดขึ้นจริง หรือเปลี่ยนไปใช้ Flag แยกต่างหากแทน)

### แบบฝึกหัดที่ 115.1

**โจทย์**: แก้โค้ดให้ย้ายค่า Sentinel 9999 ไปไว้ที่ช่องที่ 5 แทนช่องที่ 3 แล้วดูว่าผลรวมที่ได้เปลี่ยนไป
อย่างไร

**เฉลย**: สลับค่าระหว่างช่อง 3 กับช่อง 5:

```cobol
           MOVE 1500 TO WS-ORDER-AMT(1).
           MOVE 2200 TO WS-ORDER-AMT(2).
           MOVE 3000 TO WS-ORDER-AMT(3).
           MOVE 1800 TO WS-ORDER-AMT(4).
           MOVE 9999 TO WS-ORDER-AMT(5).
           MOVE 0 TO WS-ORDER-AMT(6).
```

ผลลัพธ์: โปรแกรมจะประมวลผลช่องที่ 1-4 ทั้งหมด (1500+2200+3000+1800 = 8500) ก่อนพบ Sentinel ที่ช่อง 5
แล้วหยุด `Total (before sentinel)` ที่ได้จะเป็น `008500`

---

## ขั้นตอนที่ 116: EXIT PERFORM และ EXIT PERFORM CYCLE

### แนวคิด

ใน Inline PERFORM COBOL มีคำสั่งพิเศษสองตัวสำหรับควบคุมการไหลของลูปโดยไม่ต้องรอเงื่อนไข `UNTIL`:

- **`EXIT PERFORM`**: ออกจากลูปทั้งหมด**ทันที** ไม่สนใจว่าเงื่อนไข `UNTIL` จะจริงหรือยัง (เทียบเท่ากับ
  `break` ในภาษาอื่น)
- **`EXIT PERFORM CYCLE`**: ข้ามคำสั่งที่เหลือใน**รอบปัจจุบัน** แล้วกระโดดไปตรวจสอบเงื่อนไข `UNTIL`
  ทันทีเพื่อเริ่มรอบถัดไป (เทียบเท่ากับ `continue` ในภาษาอื่น — ต่างจาก `CONTINUE` ของ COBOL เองที่แปลว่า
  "ไม่ทำอะไรเลย" ไม่ใช่ "ข้ามไปรอบถัดไป")

ทั้งสองคำสั่งใช้ได้เฉพาะภายใน Inline PERFORM (`PERFORM ... END-PERFORM`) เท่านั้น

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP116.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NUM              PIC 9(02) VALUE ZERO.
       01  WS-REMAINDER        PIC 9(02) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== EXIT PERFORM and EXIT PERFORM CYCLE demo ===".

           DISPLAY "-- 1. EXIT PERFORM: stop the loop completely --".
           PERFORM UNTIL WS-NUM > 10
               ADD 1 TO WS-NUM
               IF WS-NUM = 4
                   DISPLAY "  Reached 4, exiting the loop now."
                   EXIT PERFORM
               END-IF
               DISPLAY "  Processing number " WS-NUM
           END-PERFORM.
           DISPLAY "  Loop stopped at WS-NUM = " WS-NUM.

           DISPLAY "-- 2. EXIT PERFORM CYCLE: skip to next round --".
           MOVE 0 TO WS-NUM.
           PERFORM UNTIL WS-NUM > 6
               ADD 1 TO WS-NUM
               DIVIDE WS-NUM BY 2 GIVING WS-REMAINDER
                   REMAINDER WS-REMAINDER
               IF WS-REMAINDER NOT = 0
                   EXIT PERFORM CYCLE
               END-IF
               DISPLAY "  Even number: " WS-NUM
           END-PERFORM.
           DISPLAY "Done.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== EXIT PERFORM and EXIT PERFORM CYCLE demo ===
-- 1. EXIT PERFORM: stop the loop completely --
  Processing number 01
  Processing number 02
  Processing number 03
  Reached 4, exiting the loop now.
  Loop stopped at WS-NUM = 04
-- 2. EXIT PERFORM CYCLE: skip to next round --
  Even number: 02
  Even number: 04
  Even number: 06
Done.
```

### อธิบายโค้ดทีละส่วน

- ในบล็อกที่ 1 ลูปควรจะวนได้ถึง 10 รอบตามเงื่อนไข `UNTIL WS-NUM > 10` แต่เมื่อ `WS-NUM` มีค่าเป็น 4
  โปรแกรมสั่ง `EXIT PERFORM` ทำให้ออกจากลูปทันที**ก่อน**ที่จะไปถึงคำสั่ง
  `DISPLAY "  Processing number " WS-NUM` ในรอบนั้น — จึงเห็นว่ามีแค่รอบ 1, 2, 3 ที่แสดง
  "Processing number" ส่วนรอบ 4 แสดงแค่ข้อความแจ้งเตือนแล้วหยุดเลย
- ในบล็อกที่ 2 ทุกรอบจะคำนวณเศษจากการหาร 2 ก่อน ถ้าเป็นเลขคี่ (`WS-REMAINDER NOT = 0`) จะสั่ง
  `EXIT PERFORM CYCLE` ทันที ทำให้ข้ามคำสั่ง `DISPLAY "  Even number: "` ไปเลย แล้วกระโดดกลับไปตรวจสอบ
  `UNTIL WS-NUM > 6` เพื่อเริ่มรอบถัดไป — ผลลัพธ์จึงมีแค่เลขคู่ (2, 4, 6) เท่านั้นที่แสดงออกมา
- ความแตกต่างสำคัญ: `EXIT PERFORM` หยุด**ทั้งลูป** ส่วน `EXIT PERFORM CYCLE` หยุดแค่**รอบปัจจุบัน**
  แล้ววนต่อ

### ข้อควรระวัง

- `EXIT PERFORM CYCLE` เป็นฟีเจอร์ที่มาใน COBOL รุ่นใหม่กว่า `EXIT PERFORM` เล็กน้อย คอมไพเลอร์บาง
  ตัวที่เก่ามากอาจไม่รองรับ — ถ้าใช้กับระบบ Mainframe เก่ามาก ควรตรวจสอบก่อนว่า Dialect รองรับหรือไม่
  (GnuCOBOL ที่เราใช้ในหลักสูตรนี้รองรับเต็มรูปแบบ)
- ทั้ง `EXIT PERFORM` และ `EXIT PERFORM CYCLE` ใช้ได้เฉพาะกับ Inline PERFORM เท่านั้น หากใช้ในบริบทอื่น
  (เช่นนอก `PERFORM ... END-PERFORM` ใด ๆ) จะเป็น syntax error

### แบบฝึกหัดที่ 116.1

**โจทย์**: แก้บล็อกที่ 1 ให้หยุดลูปเมื่อ `WS-NUM` เท่ากับ 7 แทนที่จะเป็น 4

**เฉลย**: เปลี่ยนเงื่อนไขใน `IF` เท่านั้น ส่วนที่เหลือเหมือนเดิม:

```cobol
               IF WS-NUM = 7
                   DISPLAY "  Reached 7, exiting the loop now."
                   EXIT PERFORM
               END-IF
```

ผลลัพธ์: จะเห็นข้อความ "Processing number" ตั้งแต่ 01 ถึง 06 แล้วหยุดที่ `WS-NUM = 07`

---

## ขั้นตอนที่ 117: PERFORM paragraph-name ทบทวนอีกครั้ง — เตรียมพื้นฐานก่อน Part 014

### แนวคิด

ก่อนจบ Part นี้ ขอย้อนกลับมาทบทวน Out-of-line PERFORM (การเรียก paragraph) อีกครั้งให้ชัดเจนยิ่งขึ้น
เพราะจะเป็นพื้นฐานสำคัญของ Part 014 ที่จะสอนการจัดโครงสร้างโปรแกรมด้วยหลาย paragraph/section และ
`PERFORM ... THRU` อย่างเต็มรูปแบบ สิ่งสำคัญที่ต้องเข้าใจตอนนี้คือ:

- เมื่อเรียก `PERFORM paragraph-name.` การควบคุมจะกระโดดไปทำงานที่ paragraph นั้นจนครบทุกคำสั่ง
  แล้ว**กลับมาทำงานต่อที่บรรทัดถัดจาก `PERFORM`** โดยอัตโนมัติ (ไม่ใช่จบโปรแกรมไปเลย)
- paragraph เดียวกันสามารถถูกเรียกซ้ำได้หลายครั้งจากหลายจุดในโปรแกรม และตัวแปรที่มันแก้ไข (เช่นตัวนับ)
  จะยังคงค่าที่สะสมไว้ข้ามการเรียกแต่ละครั้ง เพราะ `WORKING-STORAGE SECTION` มีอายุอยู่ตลอดการรันโปรแกรม

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP117.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CHOICE           PIC 9(01) VALUE ZERO.
       01  WS-VISIT-COUNT      PIC 9(01) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== PERFORM paragraph-name, revisited ===".
           PERFORM SHOW-MENU.
           PERFORM SHOW-MENU.
           DISPLAY "SHOW-MENU was called " WS-VISIT-COUNT " times.".
           DISPLAY "(Full multi-paragraph programs and PERFORM".
           DISPLAY " ... THRU are covered in detail in Part 014.)".
           STOP RUN.

       SHOW-MENU.
           ADD 1 TO WS-VISIT-COUNT.
           DISPLAY "  [Menu] 1=Add  2=List  3=Exit".
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM paragraph-name, revisited ===
  [Menu] 1=Add  2=List  3=Exit
  [Menu] 1=Add  2=List  3=Exit
SHOW-MENU was called 2 times.
(Full multi-paragraph programs and PERFORM
 ... THRU are covered in detail in Part 014.)
```

### อธิบายโค้ดทีละส่วน

- `WS-VISIT-COUNT` ถูกเพิ่มค่าอยู่ภายใน `SHOW-MENU` เอง ไม่ใช่ใน `MAIN-LOGIC` — และเมื่อเรียก
  `PERFORM SHOW-MENU.` สองครั้งจาก `MAIN-LOGIC` ค่า `WS-VISIT-COUNT` จึงสะสมกลายเป็น 2 ได้อย่างถูกต้อง
  เพราะตัวแปรใน `WORKING-STORAGE` ไม่ได้ถูกรีเซ็ตระหว่างการเรียก `PERFORM` แต่ละครั้ง
- ลำดับการทำงาน: `MAIN-LOGIC` เรียก `SHOW-MENU` ครั้งที่ 1 → ควบคุมกระโดดไปที่ `SHOW-MENU`, เพิ่มค่า,
  แสดงเมนู, จบ paragraph, **กระโดดกลับมาที่ `MAIN-LOGIC`** → เรียก `SHOW-MENU` ครั้งที่ 2 อีกรอบด้วย
  ลำดับเดียวกัน แล้วจึงไปทำ `DISPLAY` บรรทัดถัดไปต่อ
- นี่คือกลไกพื้นฐานที่ทำให้เราสามารถเขียนโปรแกรมแบบแบ่งเป็นส่วนย่อย ๆ (modular) ได้ ซึ่งจะขยายผลเต็มที่
  ใน Part 014

### ข้อควรระวัง

- ต่างจากภาษาสมัยใหม่ที่มี "function" แยกขอบเขตตัวแปรชัดเจน (local variable) ตัวแปรใน paragraph ของ
  COBOL (เมื่อประกาศใน `WORKING-STORAGE`) เป็น **global ทั้งหมด** — paragraph ใดก็แก้ไขตัวแปรใดก็ได้
  ทั้งโปรแกรม ซึ่งสะดวกแต่ก็เสี่ยงต่อบั๊กถ้าหลาย paragraph แก้ไขตัวแปรเดียวกันโดยไม่ตั้งใจ
- ถ้า paragraph ที่ถูกเรียกไม่มีการปิดท้ายด้วย `PERFORM` อื่นหรือ `STOP RUN` และมี paragraph ถัดไปเขียน
  ต่อท้ายในซอร์สโค้ดทันที ระวังสับสนกับพฤติกรรมการ "ไหลลงไป" (fall-through) ของ COBOL เมื่อไม่ได้เรียก
  ผ่าน `PERFORM` แต่ปล่อยให้โปรแกรมทำงานไล่ตามลำดับ paragraph ในซอร์สโค้ดแทน (รายละเอียดเต็มใน Part 014)

### แบบฝึกหัดที่ 117.1

**โจทย์**: เพิ่มการเรียก `PERFORM SHOW-MENU.` อีกหนึ่งครั้งเป็นครั้งที่ 3 แล้วดูว่า
`WS-VISIT-COUNT` เปลี่ยนเป็นเท่าไหร่

**เฉลย**: เพิ่มบรรทัด `PERFORM SHOW-MENU.` เป็นครั้งที่สาม ผลลัพธ์ `WS-VISIT-COUNT` จะกลายเป็น 3
และเมนูจะแสดงซ้ำ 3 ครั้งตามจำนวนการเรียก

---

## ขั้นตอนที่ 118: ลูปซ้อนกันด้วย PERFORM UNTIL (ไม่ใช้ VARYING)

### แนวคิด

เราสามารถวนลูปซ้อนกัน (Nested Loop) ด้วย `PERFORM UNTIL` ได้เช่นเดียวกับ `PERFORM VARYING` เพียงแต่
ต้องจัดการตัวแปรควบคุมเงื่อนไข (ทั้งค่าเริ่มต้นและการเพิ่มค่า) **ด้วยตนเองทุกจุด** ซึ่งจะเห็นได้ชัดว่า
ทำไม `PERFORM VARYING ... AFTER` (Part 013) ถึงสะดวกกว่ามากสำหรับลูปซ้อนแบบนับจำนวนรอบตรง ๆ — แต่
`PERFORM UNTIL` ซ้อนกันก็ยังจำเป็นมากในกรณีที่เงื่อนไขการหยุดของแต่ละชั้นซับซ้อนกว่าการนับเลขธรรมดา
(เช่น "หยุดเมื่อพบ flag" ผสมกับ "หยุดเมื่อครบจำนวน")

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP118.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ROW              PIC 9(02) VALUE 1.
       01  WS-COL              PIC 9(02) VALUE 1.
       01  WS-ROW-DONE-FLAG    PIC X(01).
       01  WS-ALL-DONE-FLAG    PIC X(01) VALUE "N".
           88  WS-ALL-ROWS-DONE         VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Nested PERFORM UNTIL (no VARYING) ===".
           PERFORM UNTIL WS-ALL-ROWS-DONE
               DISPLAY "Row " WS-ROW " starts:"
               MOVE 1 TO WS-COL
               MOVE "N" TO WS-ROW-DONE-FLAG
               PERFORM UNTIL WS-ROW-DONE-FLAG = "Y"
                   DISPLAY "  Cell (" WS-ROW ", " WS-COL ")"
                   ADD 1 TO WS-COL
                   IF WS-COL > 3
                       MOVE "Y" TO WS-ROW-DONE-FLAG
                   END-IF
               END-PERFORM
               ADD 1 TO WS-ROW
               IF WS-ROW > 2
                   SET WS-ALL-ROWS-DONE TO TRUE
               END-IF
           END-PERFORM.
           DISPLAY "Finished all rows.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Nested PERFORM UNTIL (no VARYING) ===
Row 01 starts:
  Cell (01, 01)
  Cell (01, 02)
  Cell (01, 03)
Row 02 starts:
  Cell (02, 01)
  Cell (02, 02)
  Cell (02, 03)
Finished all rows.
```

### อธิบายโค้ดทีละส่วน

- ลูปนอกควบคุมด้วย `WS-ALL-DONE-FLAG` (ผ่าน Condition Name `WS-ALL-ROWS-DONE`) ส่วนลูปในควบคุมด้วย
  `WS-ROW-DONE-FLAG` — ทั้งสองเป็นตัวแปร Flag คนละตัวกัน ไม่ชนกัน (คล้ายหลักการตั้งชื่อตัวนับต่างกัน
  ใน `PERFORM VARYING` ที่เรียนใน Part 013 ขั้นตอนที่ 124)
- ก่อนเริ่มลูปในทุกรอบของลูปนอก จะต้อง **รีเซ็ตค่าเริ่มต้น** ของตัวแปรลูปใน (`MOVE 1 TO WS-COL` และ
  `MOVE "N" TO WS-ROW-DONE-FLAG`) ใหม่ทุกครั้ง — นี่คือจุดที่แตกต่างจาก `PERFORM VARYING ... AFTER`
  ซึ่ง COBOL จะรีเซ็ตค่า `FROM` ให้อัตโนมัติทุกรอบของลูปนอก แต่ `PERFORM UNTIL` ต้องรีเซ็ตเอง
- ผลลัพธ์แสดงให้เห็นว่าลูปในวนครบ 3 คอลัมน์ทุกครั้งก่อนที่ลูปนอกจะขยับไปแถวถัดไป เหมือนกับพฤติกรรม
  ของลูปซ้อนแบบ `PERFORM VARYING` ทุกประการ เพียงแต่เขียนควบคุมด้วยมือทั้งหมด

### ข้อควรระวัง

- **ข้อผิดพลาดที่พบบ่อยที่สุด**: ลืมรีเซ็ตตัวแปรควบคุมลูปใน (`WS-COL`, `WS-ROW-DONE-FLAG`) ก่อนเริ่ม
  ลูปในรอบใหม่ของลูปนอก — ถ้าลืม `MOVE 1 TO WS-COL` ตัวแปรจะค้างค่าจากรอบก่อนหน้า ทำให้ลูปในรอบถัดไป
  ไม่ทำงานเลย (เพราะเงื่อนไขเป็นจริงตั้งแต่แรก) นี่คือสาเหตุที่ `PERFORM VARYING ... AFTER` ได้รับความ
  นิยมมากกว่าสำหรับลูปซ้อนแบบนับจำนวนรอบตรง ๆ เพราะ COBOL จัดการเรื่องนี้ให้อัตโนมัติ
- เมื่อเปรียบเทียบโค้ดนี้กับ Part 013 ขั้นตอนที่ 123 (ตารางสูตรคูณด้วย `PERFORM VARYING ... AFTER`)
  จะเห็นชัดเจนว่าโค้ดสั้นและอ่านง่ายกว่ามาก — ควรเลือกใช้ `PERFORM VARYING` เมื่อเงื่อนไขเป็นการนับ
  จำนวนรอบตรง ๆ และเก็บ `PERFORM UNTIL` ซ้อนกันไว้สำหรับกรณีที่ต้องมีเงื่อนไขซับซ้อนกว่านั้นจริง ๆ

### แบบฝึกหัดที่ 118.1

**โจทย์**: แก้ไขให้ลูปนอกวนทั้งหมด 3 แถว (แทน 2 แถว) และลูปในวน 4 คอลัมน์ (แทน 3 คอลัมน์)

**เฉลย**: แก้ไข 2 จุด คือเงื่อนไข `IF WS-ROW > 2` เป็น `IF WS-ROW > 3` และ `IF WS-COL > 3` เป็น
`IF WS-COL > 4`:

```cobol
                   IF WS-COL > 4
                       MOVE "Y" TO WS-ROW-DONE-FLAG
                   END-IF
               END-PERFORM
               ADD 1 TO WS-ROW
               IF WS-ROW > 3
                   SET WS-ALL-ROWS-DONE TO TRUE
               END-IF
```

ผลลัพธ์จะได้ 3 แถว แถวละ 4 เซลล์ รวม 12 บรรทัดข้อมูล

---

## ขั้นตอนที่ 119: ข้อผิดพลาดที่พบบ่อยของ PERFORM UNTIL (Infinite Loop)

### แนวคิด

เนื่องจาก `PERFORM UNTIL` ไม่มีกลไกเพิ่มค่าตัวแปรอัตโนมัติเหมือน `PERFORM VARYING` จึงเป็นจุดที่มือใหม่
COBOL สร้างบั๊ก **infinite loop (ลูปไม่รู้จบ)** บ่อยที่สุด บทนี้จะรวบรวมสาเหตุที่พบบ่อยที่สุด

### ข้อผิดพลาดที่ 1: ลืมเพิ่มค่าตัวแปรควบคุมเงื่อนไข

```cobol
      *> WRONG: forgot to ADD 1 TO WS-N inside the loop body
           MOVE 1 TO WS-N.
           PERFORM UNTIL WS-N > 5
               DISPLAY "N = " WS-N
           END-PERFORM.
```

ปัญหา: `WS-N` จะค้างค่าเป็น 1 ตลอดไป เพราะไม่มีคำสั่งใดในเนื้อลูปเปลี่ยนแปลงค่าของมันเลย เงื่อนไข
`WS-N > 5` จะไม่มีวันเป็นจริง โปรแกรมจะพิมพ์ "N = 1" ไม่หยุดจนกว่าจะถูกบังคับปิด (`Ctrl+C`)

### ข้อผิดพลาดที่ 2: เพิ่มค่าตัวแปรผิดทิศทางเทียบกับเงื่อนไข

```cobol
      *> WRONG: counting down, but the condition checks for "greater"
           MOVE 10 TO WS-N.
           PERFORM UNTIL WS-N > 5
               DISPLAY "N = " WS-N
               SUBTRACT 1 FROM WS-N
           END-PERFORM.
```

ปัญหา: `WS-N` เริ่มที่ 10 แล้วลดลงเรื่อย ๆ (10, 9, 8, ...) แต่เงื่อนไขหยุดคือ "มากกว่า 5" ซึ่งเป็นจริง
อยู่แล้วตั้งแต่ต้น (10 > 5) ดังนั้น**เนื้อลูปจะไม่ทำงานแม้แต่ครั้งเดียว** (กรณีนี้ตรงข้ามกับ Infinite
Loop คือ "ลูปไม่ทำงานเลย" ซึ่งเป็นอีกหนึ่งอาการของการตั้งเงื่อนไขผิดทิศทาง) และถ้าเงื่อนไขกลับด้าน
(เช่นเขียนเป็น `UNTIL WS-N < 5` แทน) ค่าจะไหลลงจนติดลบและ overflow เพราะ `WS-N` เป็น `PIC 9(02)`
แบบไม่มีเครื่องหมาย

### เวอร์ชันที่แก้ไขแล้ว (compile และรันจริง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP119.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-N                PIC 9(02) VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Correct PERFORM UNTIL pattern ===".
           PERFORM UNTIL WS-N > 5
               DISPLAY "  N = " WS-N
               ADD 1 TO WS-N
           END-PERFORM.
           DISPLAY "Finished safely, WS-N = " WS-N.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== Correct PERFORM UNTIL pattern ===
  N = 01
  N = 02
  N = 03
  N = 04
  N = 05
Finished safely, WS-N = 06
```

### สรุปรายการข้อผิดพลาดที่พบบ่อยของ PERFORM UNTIL

| ข้อผิดพลาด | อาการ | วิธีป้องกัน |
|---|---|---|
| ลืมเพิ่ม/ลดค่าตัวแปรควบคุมเงื่อนไข | Infinite Loop (โปรแกรมค้าง) | ตรวจสอบทุกครั้งว่ามีคำสั่งเปลี่ยนค่าตัวแปรอยู่ในเนื้อลูปจริง |
| ทิศทางการเพิ่ม/ลดค่าสวนทางกับเงื่อนไข | ลูปไม่ทำงานเลย หรือค่าล้น (overflow) | ตรวจว่านับขึ้นคู่กับ `UNTIL ... >` และนับลงคู่กับ `UNTIL ... <` |
| ลืมรีเซ็ตตัวแปรก่อนเริ่มลูปซ้อนรอบใหม่ | ลูปในไม่ทำงานตั้งแต่รอบที่ 2 เป็นต้นไป | `MOVE` ค่าเริ่มต้นให้ตัวแปรลูปในทุกครั้งก่อนเข้าลูปใน (ดูขั้นตอนที่ 118) |
| ใช้ตัวแปรเดียวกันในลูปซ้อนสองชั้น | ลูปนอกทำงานผิดจำนวนรอบ | ตั้งชื่อตัวแปรควบคุมให้ต่างกันเสมอในแต่ละชั้น |

### ข้อควรระวัง

- ควรทดสอบ `PERFORM UNTIL` ทุกครั้งด้วยขอบเขตเล็ก ๆ ก่อนเสมอ (เช่น 2-3 รอบ) เพื่อยืนยันว่าเงื่อนไข
  และการเพิ่มค่าถูกทิศทาง ก่อนขยายไปใช้กับข้อมูลจริงจำนวนมาก
- หากสงสัยว่าโปรแกรมค้างเพราะ Infinite Loop ให้กด `Ctrl+C` เพื่อหยุดโปรแกรมทันที แล้วกลับไปตรวจสอบว่า
  มีคำสั่งเปลี่ยนแปลงค่าตัวแปรควบคุมเงื่อนไขอยู่ในเนื้อลูปจริงหรือไม่เป็นอันดับแรก

### แบบฝึกหัดที่ 119.1

**โจทย์**: จงระบุว่าโค้ดต่อไปนี้จะเกิดปัญหาอะไร และแก้ไขให้ถูกต้อง

```cobol
       01  WS-M   PIC 9(02) VALUE 1.
       ...
           PERFORM UNTIL WS-M > 20
               DISPLAY WS-M
               ADD 2 TO WS-M
           END-PERFORM.
```

**เฉลย**: โค้ดนี้จริง ๆ แล้ว**ไม่มีปัญหา infinite loop** เพราะ `WS-M` เพิ่มค่าทีละ 2 อย่างถูกทิศทาง
ตรงกับเงื่อนไข `UNTIL WS-M > 20` (1, 3, 5, 7, ..., 21) แต่มีจุดที่ควรระวัง: `WS-M` เป็น `PIC 9(02)`
ซึ่งรองรับค่าได้สูงสุด 99 หากในอนาคตมีการแก้ไขเงื่อนไขเป็นค่าที่สูงกว่านั้นโดยไม่ขยาย `PICTURE` ตามไปด้วย
จะเกิด overflow แบบเงียบได้ ดังนั้นแม้โค้ดนี้จะรันได้ถูกต้อง ก็ควรตรวจสอบว่า `PICTURE` มีขนาดใหญ่พอเสมอ
สำหรับค่าสูงสุดที่ตัวแปรจะมีโอกาสไปถึง

---

## ขั้นตอนที่ 120: สรุปเทคนิค — เมนูแบบวนซ้ำด้วย PERFORM WITH TEST AFTER

### แนวคิด

ปิดท้าย Part นี้ด้วยการนำเทคนิคทั้งหมดที่เรียนมา (`PERFORM UNTIL`, `WITH TEST AFTER`, ตาราง `OCCURS`
เบื้องต้น, การเรียก paragraph ย่อย) มาผสมกันเป็นโปรแกรมรูปแบบ **"วนรับรายการจนกว่าจะหมดสต็อกหรือผู้ใช้
สั่งหยุด"** ซึ่งเป็นรูปแบบที่ใกล้เคียงกับโปรแกรมป้อนข้อมูล (Data Entry) ในงานจริงมาก และยังปูทางไปสู่
โปรเจกต์เครื่องคิดเลขแบบมีเมนูใน Part 015

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP120.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ITEM-TABLE.
           05  WS-ITEM-AMT     PIC 9(05) OCCURS 5 TIMES.
       01  WS-IDX              PIC 9(02) VALUE 1.
       01  WS-ANSWER           PIC X(01) VALUE "Y".
       01  WS-GRAND-TOTAL      PIC 9(06) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "=== Mini order-entry loop (menu style) ===".
           MOVE 1200 TO WS-ITEM-AMT(1).
           MOVE 3400 TO WS-ITEM-AMT(2).
           MOVE 2200 TO WS-ITEM-AMT(3).
           MOVE 1800 TO WS-ITEM-AMT(4).
           MOVE 2600 TO WS-ITEM-AMT(5).

           PERFORM WITH TEST AFTER UNTIL WS-ANSWER NOT = "Y"
               PERFORM ADD-ONE-ITEM
               IF WS-IDX > 5
                   MOVE "N" TO WS-ANSWER
                   DISPLAY "  No more items in stock, stopping."
               END-IF
           END-PERFORM.

           DISPLAY "Grand total entered: " WS-GRAND-TOTAL.
           STOP RUN.

       ADD-ONE-ITEM.
           IF WS-IDX <= 5
               DISPLAY "  Adding item " WS-IDX ": "
                       WS-ITEM-AMT(WS-IDX)
               ADD WS-ITEM-AMT(WS-IDX) TO WS-GRAND-TOTAL
               ADD 1 TO WS-IDX
           END-IF.
```

### ผลลัพธ์จริงที่ได้

```
=== Mini order-entry loop (menu style) ===
  Adding item 01: 01200
  Adding item 02: 03400
  Adding item 03: 02200
  Adding item 04: 01800
  Adding item 05: 02600
  No more items in stock, stopping.
Grand total entered: 011200
```

### อธิบายโค้ดทีละส่วน

- `PERFORM WITH TEST AFTER UNTIL WS-ANSWER NOT = "Y"` ใช้ `TEST AFTER` เพื่อให้แน่ใจว่าลูปจะรัน**อย่าง
  น้อย 1 รอบเสมอ** ก่อนที่จะตรวจสอบว่าควรหยุดหรือไม่ — เหมาะกับสถานการณ์ "รับรายการอย่างน้อยหนึ่งอย่าง
  ก่อนถามว่าจะทำต่อหรือไม่" ซึ่งพบบ่อยมากในโปรแกรมป้อนข้อมูลจริง (ในโปรแกรมจริงที่รับข้อมูลจากผู้ใช้ผ่าน
  `ACCEPT` ค่า `WS-ANSWER` มักจะถูกถามจากผู้ใช้ตรง ๆ ว่า "ต้องการป้อนรายการต่อหรือไม่ (Y/N)")
- `PERFORM ADD-ONE-ITEM` ในลูปคือ Out-of-line PERFORM ที่เรียกซ้ำทุกรอบ ทำหน้าที่หยิบสินค้าจากตาราง
  มาบวกเข้ายอดรวม — สังเกตว่า paragraph `ADD-ONE-ITEM` มีการตรวจสอบ `IF WS-IDX <= 5` ของตัวเองด้วย
  เพื่อป้องกันไม่ให้เข้าถึงตารางเกินขอบเขต (จะเรียนเรื่องนี้ลึกขึ้นใน Part 016)
- เมื่อ `WS-IDX` เกิน 5 (สินค้าหมดสต็อก) โปรแกรมจะตั้ง `WS-ANSWER` เป็น "N" ทำให้เงื่อนไข
  `WS-ANSWER NOT = "Y"` เป็นจริง และลูปจะหยุดในรอบถัดไปที่ตรวจสอบเงื่อนไข

### ข้อควรระวัง

- โค้ดตัวอย่างนี้จำลองสถานการณ์ "หมดสต็อก" แทนการถามผู้ใช้จริง (เพื่อให้โปรแกรมรันได้ครบอัตโนมัติโดยไม่
  ต้องพิมพ์โต้ตอบ) ในโปรแกรมจริงที่ใช้ `ACCEPT` รับคำตอบจากผู้ใช้ ต้องระวังเรื่องการรับค่าตัวพิมพ์เล็ก/
  ใหญ่ปนกัน (เช่นผู้ใช้พิมพ์ "y" แทน "Y") ซึ่งควรใช้ `INSPECT` หรือฟังก์ชันแปลงตัวพิมพ์แปลงค่าก่อน
  เปรียบเทียบเสมอ (จะเรียนเรื่องนี้ใน Part 021)
- เมื่อผสม `PERFORM UNTIL` (ลูปหลัก) กับ `PERFORM paragraph-name` (เรียก paragraph ย่อยในลูป) แบบนี้
  ควรตรวจสอบให้แน่ใจว่า paragraph ย่อยไม่ได้แก้ไขตัวแปรเงื่อนไขของลูปหลักโดยไม่ตั้งใจ เพราะจะทำให้
  พฤติกรรมของลูปเปลี่ยนไปอย่างไม่คาดคิด (ในตัวอย่างนี้ `ADD-ONE-ITEM` แก้ไข `WS-IDX` ซึ่งถูกใช้ตรวจสอบ
  ใน `MAIN-LOGIC` ด้วย — เป็นการออกแบบที่ตั้งใจ แต่ต้องระวังไม่ให้เกิดผลข้างเคียงที่ไม่ตั้งใจในโค้ดจริง)

### แบบฝึกหัดที่ 120.1

**โจทย์**: ขยายตารางสินค้าเป็น 7 ช่อง (เพิ่ม 2 รายการใหม่) แล้วดูว่ายอดรวมสุดท้ายเปลี่ยนไปเท่าไหร่

**เฉลย**: ขยาย `OCCURS 5 TIMES` เป็น `OCCURS 7 TIMES` เพิ่มค่าอีก 2 รายการ และแก้เงื่อนไขใน
`MAIN-LOGIC` และ `ADD-ONE-ITEM` จาก `> 5`/`<= 5` เป็น `> 7`/`<= 7`:

```cobol
       01  WS-ITEM-TABLE.
           05  WS-ITEM-AMT     PIC 9(05) OCCURS 7 TIMES.
       ...
           MOVE 1200 TO WS-ITEM-AMT(1).
           MOVE 3400 TO WS-ITEM-AMT(2).
           MOVE 2200 TO WS-ITEM-AMT(3).
           MOVE 1800 TO WS-ITEM-AMT(4).
           MOVE 2600 TO WS-ITEM-AMT(5).
           MOVE 1500 TO WS-ITEM-AMT(6).
           MOVE 2000 TO WS-ITEM-AMT(7).
           ...
               IF WS-IDX > 7
       ...
       ADD-ONE-ITEM.
           IF WS-IDX <= 7
```

ยอดรวมใหม่จะเป็น 1200+3400+2200+1800+2600+1500+2000 = 14700 ดังนั้น
`Grand total entered: 014700`

---

## สรุปท้ายบท

ใน Part นี้ คุณได้เรียนรู้:

- ภาพรวมของ `PERFORM` ทั้ง 4 รูปแบบหลัก (เปล่า, `TIMES`, `UNTIL`, `VARYING`) และความแตกต่างระหว่าง
  Out-of-line PERFORM กับ Inline PERFORM
- `PERFORM ... TIMES` สำหรับวนลูปแบบนับจำนวนรอบตายตัวโดยไม่ต้องใช้ตัวแปรตัวนับ
- `PERFORM UNTIL` รูปแบบพื้นฐาน และความรับผิดชอบของผู้เขียนโปรแกรมในการเพิ่มค่าตัวแปรควบคุมเงื่อนไขเอง
- ความแตกต่างระหว่าง `WITH TEST BEFORE` (ค่าเริ่มต้น) กับ `WITH TEST AFTER` ใน `PERFORM UNTIL`
- รูปแบบ Sentinel-controlled loop ที่จะกลับมาใช้ซ้ำเมื่อเรียนเรื่องการอ่านไฟล์ใน Part 023-025
- `EXIT PERFORM` (ออกจากลูปทันที) และ `EXIT PERFORM CYCLE` (ข้ามไปรอบถัดไป)
- การทบทวน Out-of-line PERFORM เพื่อเตรียมความพร้อมสำหรับ Part 014
- การวนลูปซ้อนกันด้วย `PERFORM UNTIL` โดยไม่ใช้ `VARYING` และความสำคัญของการรีเซ็ตตัวแปรลูปในทุกรอบ
- ข้อผิดพลาดที่พบบ่อยที่สุดของ `PERFORM UNTIL` ที่นำไปสู่ Infinite Loop และวิธีป้องกัน
- การผสาน `PERFORM WITH TEST AFTER` กับตาราง `OCCURS` และ paragraph ย่อยเพื่อสร้างลูปแบบเมนู/รับข้อมูล

ทักษะการวนลูปด้วย `PERFORM UNTIL` ที่คุณเพิ่งฝึกมาทั้งหมดนี้เป็นพื้นฐานสำคัญที่จะขยายผลต่อใน Part ถัดไป
เมื่อเราเรียนรู้ `PERFORM VARYING` อย่างละเอียดเต็มรูปแบบ — ตั้งแต่การกำหนด `FROM`/`BY` แบบกำหนดเอง
ไปจนถึงการวนลูปซ้อนกันหลายชั้นด้วยวลี `AFTER` ซึ่งจะทำให้โค้ดที่ต้องนับจำนวนรอบกระชับและอ่านง่ายกว่า
`PERFORM UNTIL` ที่ต้องควบคุมทุกอย่างด้วยมือแบบที่เราเพิ่งฝึกมา

**[← กลับไป Part 011: EVALUATE Statement](part-011-evaluate.md)** | **[ไปยัง Part 013: PERFORM VARYING และการวนลูปหลายชั้น →](part-013-perform-varying.md)**
