# Part 081: Code Refactoring และ Clean Code สำหรับ COBOL (ขั้นตอนที่ 801–810)

## คำนำของ Part นี้

Part 080 สอนให้เราจัดการการเปลี่ยนแปลงโค้ดร่วมกันในทีมอย่างปลอดภัยด้วย Git ถึงตอนนี้เรามีทั้ง
เครื่องมือทดสอบ (Part 078–079) และเครื่องมือจัดการเวอร์ชัน (Part 080) พร้อมแล้ว — เงื่อนไขที่
จำเป็นทั้งหมดสำหรับการทำสิ่งที่หลายทีม COBOL ต้องเผชิญเป็นประจำ: **การปรับปรุงโค้ดเก่าที่ทำงาน
ถูกต้องอยู่แล้ว แต่อ่านยาก บำรุงรักษายาก โดยไม่ทำให้พฤติกรรมของมันเปลี่ยนไปแม้แต่น้อย**

กระบวนการนี้เรียกว่า **Refactoring** — นิยามสำคัญที่ต้องจำให้ขึ้นใจคือ: **Refactoring คือการ
เปลี่ยนแปลง "โครงสร้างภายใน" ของโค้ดโดยไม่เปลี่ยน "พฤติกรรมภายนอก" ที่สังเกตได้เลยแม้แต่นิดเดียว**
ถ้าผลลัพธ์เปลี่ยนไป นั่นไม่ใช่ Refactoring แต่เป็นการแก้บั๊กหรือเพิ่มฟีเจอร์ใหม่

โค้ด COBOL ที่มีอายุหลายสิบปี (ตามที่ Part 001 อธิบายว่าระบบ COBOL จำนวนมากยังทำงานอยู่จนถึงปี
2026) มักสะสมปัญหาคลาสสิกไว้มากมาย: `GO TO` ที่กระโดดไปมาจนตามยาก, `IF` ซ้อนกันลึกหลายชั้น, ตัวแปร
ชื่อ `WS-A`, `WS-B`, ตรรกะเดียวกันถูกคัดลอกวางซ้ำหลายจุด Part นี้จะพาคุณแก้ปัญหาเหล่านี้ทีละแบบ
โดย**ทุกตัวอย่าง Before/After ถูกคอมไพล์และรันจริงแล้วทั้งคู่ เพื่อพิสูจน์ว่าผลลัพธ์เหมือนกันทุก
ประการหลังการ refactor**

---

## ขั้นตอนที่ 801: หลักการ Clean Code สำหรับ COBOL คืออะไร

### นิยาม Clean Code ในบริบทของ COBOL

**Clean Code** ไม่ได้หมายถึงโค้ดที่ "สั้นที่สุด" หรือ "ฉลาดที่สุด" แต่หมายถึงโค้ดที่**อ่านและเข้าใจ
ได้ง่ายที่สุดโดยมนุษย์คนอื่น** (รวมถึงตัวเราเองในอีก 6 เดือนข้างหน้า) — หลักการนี้สำคัญกับ COBOL
เป็นพิเศษเพราะ:

1. **อายุการใช้งานยาวนานผิดปกติ**: โค้ด COBOL จำนวนมากถูกอ่านและแก้ไขโดยคนที่ไม่ใช่ผู้เขียนต้นฉบับ
   หลังจากเขียนไปแล้วหลายสิบปี (ทบทวนปัญหา "Silver Tsunami" จาก Part 001)
2. **Syntax ที่ตั้งใจให้อ่านคล้ายภาษาอังกฤษ**: Grace Hopper ออกแบบ COBOL ให้ผู้บริหารที่ไม่ใช่
   โปรแกรมเมอร์อ่านเข้าใจได้ (Part 001) — โค้ดที่ไม่สะอาดทำลายจุดแข็งข้อนี้ของภาษาไปอย่างสิ้นเชิง
3. **ไม่มี IDE ช่วยนำทางที่ทรงพลังเท่าภาษาสมัยใหม่เสมอไป**: ระบบ Mainframe บางระบบยังใช้เครื่องมือ
   แก้ไขแบบพื้นฐาน (TSO/ISPF จาก Part 054) การพึ่งพา "โค้ดที่อ่านเข้าใจได้เอง" จึงสำคัญกว่าการ
   พึ่งพาเครื่องมือค้นหา/นำทางอัตโนมัติ

### เทคนิค Refactoring หลัก 5 แบบที่ Part นี้จะสอน

| เทคนิค | ปัญหาที่แก้ | ขั้นตอนที่สอน |
|---|---|---|
| ตั้งชื่อตัวแปร/paragraph ให้สื่อความหมาย | `WS-A`, `WS-B` อ่านไม่รู้เรื่อง | 802 |
| แทนที่ `GO TO` ด้วย `PERFORM` โครงสร้าง | ควบคุมการไหลกระโดดไปมาไม่เป็นระเบียบ | 803 |
| แทนที่ Nested IF ด้วย `EVALUATE` | เงื่อนไขซ้อนกันลึกหลายชั้นอ่านยาก | 804 |
| Extract Paragraph เป็น Subprogram | ตรรกะซ้ำที่ควรใช้ร่วมกันได้หลายโปรแกรม | 805 |
| กำจัด Magic Numbers ด้วย 88-level/named constant | ตัวเลข/รหัสลึกลับที่ไม่มีใครรู้ความหมาย | 806 |

### หลักการสำคัญที่สุด: Regression Safety Net

ก่อน refactor ทุกครั้ง **ต้องมีวิธีพิสูจน์ว่าพฤติกรรมไม่เปลี่ยน** — ใน Part นี้เราใช้สองวิธีร่วมกัน:

1. **เปรียบเทียบผลลัพธ์ก่อน/หลังด้วย `diff`** (ทบทวนจาก Part 078) สำหรับตัวอย่างขนาดเล็ก
2. **รัน Unit Test เดิมซ้ำหลัง refactor** (ทบทวนจาก Part 079) สำหรับ subprogram ที่มีชุดทดสอบอยู่
   แล้ว — ถ้าชุดทดสอบเดิมยังผ่านหมดหลัง refactor เราจึงมั่นใจได้ว่าพฤติกรรมไม่เปลี่ยน

### ข้อควรระวัง

- **อย่า refactor และแก้บั๊ก/เพิ่มฟีเจอร์ในการเปลี่ยนแปลงเดียวกัน** — ควรแยกเป็นคนละ commit เสมอ
  (ทบทวนจาก Part 080 ขั้นตอนที่ 795 เรื่องการแยก commit reformat ออกจาก commit ตรรกะ) เพื่อให้
  Code Review และการย้อนกลับ (revert) ทำได้ง่ายถ้าจำเป็น
- Refactoring ที่ดีที่สุดคือ refactoring ที่ทำทีละก้าวเล็ก ๆ พร้อมทดสอบทุกก้าว ไม่ใช่การเขียนโปรแกรม
  ใหม่ทั้งหมดในครั้งเดียว (เทคนิคนี้จะสอนละเอียดในขั้นตอนที่ 809)

### แบบฝึกหัดที่ 801.1

**โจทย์**: จงอธิบายว่าทำไมนิยาม "Refactoring คือการเปลี่ยนโครงสร้างโดยไม่เปลี่ยนพฤติกรรม" ถึงสำคัญ
ต่อการตัดสินใจว่าจะทดสอบอย่างไรหลังทำ refactoring

**เฉลย**: เพราะนิยามนี้บอกเราชัดเจนว่า**เกณฑ์ความสำเร็จของการ refactor คือผลลัพธ์ต้องเหมือนเดิม
ทุกประการ** ไม่ใช่ "ดีขึ้น" หรือ "ถูกต้องกว่าเดิม" ทำให้วิธีทดสอบที่เหมาะสมที่สุดคือการเปรียบเทียบ
ผลลัพธ์ก่อนและหลังการเปลี่ยนแปลงโดยตรง (เช่นด้วย `diff` หรือรันชุด unit test เดิมซ้ำ) แทนที่จะ
ออกแบบชุดทดสอบใหม่ทั้งหมด ถ้าหลังจากรีแฟกเตอร์แล้วผลลัพธ์ต่างไปจากเดิมแม้เพียงนิดเดียว (ไม่ว่าจะ
"ดีขึ้น" ในสายตาใครก็ตาม) นั่นแปลว่าสิ่งที่ทำไปไม่ใช่ Refactoring ที่แท้จริงอีกต่อไป แต่กลายเป็น
การเปลี่ยนแปลงพฤติกรรมที่ต้องผ่านกระบวนการตรวจสอบและอนุมัติที่เข้มงวดกว่า (เหมือนการแก้บั๊กหรือ
เพิ่มฟีเจอร์ตามปกติ)

---

## ขั้นตอนที่ 802: ตั้งชื่อตัวแปรและ Paragraph ให้สื่อความหมาย

### ปัญหา: ชื่อที่ไม่สื่อความหมาย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP802BEFORE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A                     PIC 9(3) VALUE 160.
       01  WS-B                     PIC 9(3)V99 VALUE 50.00.
       01  WS-C                     PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       P1.
           COMPUTE WS-C = WS-A * WS-B.
           DISPLAY "PAY=" WS-C.
           STOP RUN.
```

**ผลลัพธ์จริง:**

```
PAY=0008000.00
```

โค้ดนี้ **ทำงานถูกต้องสมบูรณ์แบบ** แต่ถ้าใครมาอ่านครั้งแรกโดยไม่มีบริบท จะไม่มีทางรู้เลยว่า `WS-A`,
`WS-B`, `WS-C` หมายถึงอะไร และ `P1` คือขั้นตอนอะไรของโปรแกรม

### หลังการ Refactor: ตั้งชื่อให้สื่อความหมาย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP802AFTER.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HOURS-WORKED          PIC 9(3) VALUE 160.
       01  WS-HOURLY-RATE           PIC 9(3)V99 VALUE 50.00.
       01  WS-GROSS-PAY             PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       CALCULATE-GROSS-PAY.
           COMPUTE WS-GROSS-PAY = WS-HOURS-WORKED * WS-HOURLY-RATE.
           DISPLAY "PAY=" WS-GROSS-PAY.
           STOP RUN.
```

**ผลลัพธ์จริงหลัง refactor (คอมไพล์และรันแยกกันจริง ยืนยันด้วย `diff`):**

```
PAY=0008000.00
```

```bash
diff <(./step802before) <(./step802after) && echo "IDENTICAL OUTPUT"
```

```
IDENTICAL OUTPUT
```

### อธิบายจุดสำคัญ

- ไม่มีการเปลี่ยนแปลงตรรกะเลยแม้แต่นิดเดียว (`COMPUTE ... = ... * ...` เหมือนเดิมทุกประการ) —
  เปลี่ยนแค่**ชื่อ** ของตัวแปรและ paragraph เท่านั้น นี่คือ Refactoring ที่ "ปลอดภัยที่สุด" เท่าที่
  จะเป็นไปได้ เพราะการเปลี่ยนชื่อไม่มีทางกระทบผลการคำนวณ
- ชื่อ `CALCULATE-GROSS-PAY` (paragraph) บอกเจตนาชัดเจนกว่า `P1` มาก — เมื่อโปรแกรมมี paragraph
  หลายสิบตัว ชื่อที่สื่อความหมายจะช่วยให้ค้นหา (`grep`) และทำความเข้าใจโครงสร้างโปรแกรมได้เร็วขึ้น
  มาก
- หลักการตั้งชื่อที่ดี: **ชื่อควรบอกว่า "คืออะไร" หรือ "ทำอะไร" โดยไม่ต้องอ่านโค้ดข้างในเลย**
  (`WS-HOURS-WORKED` บอกทันทีว่าเก็บจำนวนชั่วโมงที่ทำงาน ไม่ต้องเดา)

### ข้อควรระวัง

- ชื่อตัวแปรใน COBOL ยาวได้ถึง 30 ตัวอักษร (ตามมาตรฐาน) — ควรใช้ความยาวนี้ให้เป็นประโยชน์เพื่อความ
  ชัดเจน แต่อย่ายาวจนเกินจำเป็นจนอ่านทั้งบรรทัดยากขึ้น (เช่น `WS-TOTAL-GROSS-PAY-BEFORE-TAX-
  DEDUCTION-AND-OTHER-DEDUCTIONS` ยาวเกินความจำเป็น)
- การเปลี่ยนชื่อตัวแปรที่ใช้เป็นพารามิเตอร์ใน `CALL ... USING` **ไม่กระทบการทำงาน** เพราะ COBOL
  จับคู่พารามิเตอร์ตามตำแหน่งไม่ใช่ชื่อ (ทบทวนจาก Part 031 ขั้นตอนที่ 302) — แต่ถ้าเปลี่ยนชื่อ
  field ภายใน copybook ที่ใช้ร่วมกันหลายโปรแกรม ต้องระวังตามหลักการ Copybook Coordination จาก
  Part 080 ขั้นตอนที่ 796

### แบบฝึกหัดที่ 802.1

**โจทย์**: จงอธิบายว่าทำไมการเปลี่ยนชื่อ paragraph จาก `P1` เป็น `CALCULATE-GROSS-PAY` ถึงไม่มี
ความเสี่ยงที่จะทำให้พฤติกรรมของโปรแกรมเปลี่ยนไป ในขณะที่การเปลี่ยนชื่อ field ในบาง `COPY`book อาจ
มีความเสี่ยง

**เฉลย**: ชื่อ paragraph ใน COBOL เป็นเพียง**ป้ายอ้างอิงภายในไฟล์เดียวกัน** ที่ใช้กับ `PERFORM`
เท่านั้น ตราบใดที่เปลี่ยนชื่อทั้งที่ประกาศ paragraph และทุกจุดที่ `PERFORM` เรียกถึงมันให้ตรงกัน
การเปลี่ยนชื่อจะไม่กระทบพฤติกรรมเลย เพราะไม่มีระบบภายนอกใดอ้างอิงชื่อ paragraph นี้ (paragraph
ไม่ใช่ "interface" ที่โปรแกรมอื่นเรียกใช้ข้ามไฟล์ได้เหมือน `PROGRAM-ID`) แต่ field ในภาพ copybook
ที่ถูก `COPY` เข้าไปในหลายโปรแกรมนั้น แม้การเปลี่ยน**ชื่อ**field เพียงอย่างเดียว (ไม่เปลี่ยนขนาด/
ตำแหน่ง) จะไม่กระทบพฤติกรรมการทำงานจริงเช่นกัน (เพราะ field name ก็เป็นแค่ป้ายอ้างอิงเหมือนกัน)
แต่จะกระทบ**ทุกโปรแกรมที่อ้างอิงชื่อ field เดิมโดยตรงในโค้ดของตัวเอง** ทำให้คอมไพล์ไม่ผ่านทันที
(ต่างจาก paragraph ที่ผลกระทบจำกัดอยู่แค่ไฟล์เดียว) จึงต้องประสานงานและแก้ไขทุกโปรแกรมที่เกี่ยวข้อง
พร้อมกันเสมอ ตามหลักการที่ Part 080 ขั้นตอนที่ 796 อธิบายไว้

---

## ขั้นตอนที่ 803: กำจัด GO TO แบบ Spaghetti ด้วย PERFORM โครงสร้าง

### ปัญหา: Spaghetti Code ด้วย GO TO

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP803BEFORE.
       AUTHOR. COBOL-COURSE.

      *> BEFORE: "spaghetti" control flow built with GO TO, in the
      *> style commonly seen in old COBOL-68/74 code that predates
      *> structured PERFORM. Control jumps around unpredictably,
      *> making the flow hard to follow just by reading top to bottom.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-INDEX           PIC 9(2) VALUE 1.
       01  WS-ORDER-TABLE.
           05  WS-ORDER-AMOUNT      PIC 9(5)V99 OCCURS 3 TIMES.
       01  WS-GRAND-TOTAL           PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       P1.
           MOVE 100.00 TO WS-ORDER-AMOUNT(1).
           MOVE 250.50 TO WS-ORDER-AMOUNT(2).
           MOVE 75.25  TO WS-ORDER-AMOUNT(3).
           GO TO P2.
       P2.
           IF WS-ORDER-INDEX > 3
               GO TO P4
           ELSE
               GO TO P3
           END-IF.
       P3.
           ADD WS-ORDER-AMOUNT(WS-ORDER-INDEX) TO WS-GRAND-TOTAL.
           DISPLAY "Adding order " WS-ORDER-INDEX ": "
               WS-ORDER-AMOUNT(WS-ORDER-INDEX).
           ADD 1 TO WS-ORDER-INDEX.
           GO TO P2.
       P4.
           DISPLAY "GRAND TOTAL=" WS-GRAND-TOTAL.
           STOP RUN.
```

**ผลลัพธ์จริง:**

```
Adding order 01: 00100.00
Adding order 02: 00250.50
Adding order 03: 00075.25
GRAND TOTAL=0000425.75
```

เพื่อจะเข้าใจว่าโปรแกรมนี้ทำอะไร ผู้อ่านต้อง**ไล่ตาม GO TO ไปมาระหว่าง P1 → P2 → P3 → P2 → P3 →
P2 → P4** ซ้ำไปซ้ำมา ซึ่งเป็นภาระทางสมองที่ไม่จำเป็นเลยถ้าเทียบกับการใช้ลูปที่ COBOL มีให้ใช้อยู่แล้ว

### หลังการ Refactor: ใช้ PERFORM VARYING แทน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP803AFTER.
       AUTHOR. COBOL-COURSE.

      *> AFTER: the same logic rewritten with structured PERFORM
      *> VARYING instead of GO TO. The control flow now reads top to
      *> bottom with no unconditional jumps at all.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-INDEX           PIC 9(2) VALUE 1.
       01  WS-ORDER-TABLE.
           05  WS-ORDER-AMOUNT      PIC 9(5)V99 OCCURS 3 TIMES.
       01  WS-GRAND-TOTAL           PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 100.00 TO WS-ORDER-AMOUNT(1).
           MOVE 250.50 TO WS-ORDER-AMOUNT(2).
           MOVE 75.25  TO WS-ORDER-AMOUNT(3).
           PERFORM ADD-ONE-ORDER
               VARYING WS-ORDER-INDEX FROM 1 BY 1
               UNTIL WS-ORDER-INDEX > 3.
           DISPLAY "GRAND TOTAL=" WS-GRAND-TOTAL.
           STOP RUN.

       ADD-ONE-ORDER.
           ADD WS-ORDER-AMOUNT(WS-ORDER-INDEX) TO WS-GRAND-TOTAL.
           DISPLAY "Adding order " WS-ORDER-INDEX ": "
               WS-ORDER-AMOUNT(WS-ORDER-INDEX).
```

**ผลลัพธ์จริงหลัง refactor พร้อมพิสูจน์ด้วย diff:**

```bash
cobc -x -o step803after step803after.cob
./step803after
diff <(./step803before) <(./step803after) && echo "IDENTICAL OUTPUT"
```

```
Adding order 01: 00100.00
Adding order 02: 00250.50
Adding order 03: 00075.25
GRAND TOTAL=0000425.75
IDENTICAL OUTPUT
```

### อธิบายจุดสำคัญ

- `PERFORM ADD-ONE-ORDER VARYING WS-ORDER-INDEX FROM 1 BY 1 UNTIL WS-ORDER-INDEX > 3` (ทบทวนจาก
  Part 013) แทนที่ตรรกะการวนลูปทั้งหมดที่เดิมกระจายอยู่ใน `P2`/`P3` ด้วยคำสั่งเดียวที่อ่านแล้ว
  เข้าใจทันที: "ทำ ADD-ONE-ORDER ซ้ำ โดยเพิ่ม index ทีละ 1 จนกว่าจะเกิน 3"
- **ไม่มี `GO TO` เหลืออยู่เลยแม้แต่คำเดียว** ในเวอร์ชัน after — การอ่านโค้ดตั้งแต่ `MAIN-PARA`
  ลงไปสามารถ**ทำนายลำดับการทำงานได้ล่วงหน้า**โดยไม่ต้องกระโดดไปมาระหว่าง paragraph
- โครงสร้างข้อมูล (`WS-ORDER-TABLE`, `WS-GRAND-TOTAL`) และผลลัพธ์**เหมือนเดิมทุกประการ** — สิ่งที่
  เปลี่ยนมีแค่**กลไกควบคุมการไหลของโปรแกรม** (control flow) เท่านั้น

### ข้อควรระวัง

- `GO TO` ไม่ได้ "ผิด" หรือ "ถูกแบน" อย่างเป็นทางการใน COBOL-85 ขึ้นไป (compiler ยังรองรับเต็มที่)
  แต่เป็นที่ยอมรับกันอย่างกว้างขวางในอุตสาหกรรมว่าควร**หลีกเลี่ยงเมื่อมี `PERFORM` ที่ทำสิ่งเดียวกัน
  ได้ชัดเจนกว่า** — มีบทความวิชาการชื่อดังจาก Edsger Dijkstra (1968) เรื่อง "Go To Statement
  Considered Harmful" ที่จุดกระแสนี้ในวงการโปรแกรมมิ่งทั้งหมด ไม่ใช่แค่ COBOL
- อย่าพยายาม refactor `GO TO` ที่ซับซ้อนมาก (เช่น กระโดดข้าม paragraph จำนวนมากแบบไม่มีรูปแบบชัดเจน)
  ในครั้งเดียว — ควรค่อย ๆ ไล่ทำความเข้าใจ flow เดิมทั้งหมดก่อน วาดแผนภาพลำดับการทำงานออกมาบนกระดาษ
  ก่อนเริ่มเขียนโค้ดใหม่จริง (เทคนิคนี้จะเชื่อมโยงกับ Part 083 เรื่อง Legacy System Analysis)

### แบบฝึกหัดที่ 803.1

**โจทย์**: จงอธิบายว่าทำไมโค้ดเวอร์ชัน "before" ถึงต้องมี paragraph แยกถึง 4 ตัว (`P1`, `P2`, `P3`,
`P4`) ในขณะที่เวอร์ชัน "after" ใช้แค่ 2 ตัว (`MAIN-PARA`, `ADD-ONE-ORDER`)

**เฉลย**: ในรูปแบบ `GO TO` การควบคุมเงื่อนไข "จะวนซ้ำต่อหรือจบลูป" ต้องเขียนด้วยมือทั้งหมดผ่าน
paragraph แยก (`P2` ทำหน้าที่ตรวจสอบเงื่อนไขจบ, `P3` ทำหน้าที่ประมวลผลจริงแล้ววนกลับไป `P2`) และ
ต้องมี paragraph ปลายทางแยกต่างหากสำหรับ "ก่อนเริ่มลูป" (`P1`) และ "หลังจบลูป" (`P4`) เพราะ `GO TO`
เป็นแค่การกระโดดแบบดิบ ๆ ไม่มีแนวคิดเรื่อง "ขอบเขตของลูป" ในตัวมันเอง ในขณะที่ `PERFORM ... VARYING
... UNTIL ...` เป็นคำสั่งที่ COBOL **จัดการเงื่อนไขการวนซ้ำและการจบลูปให้อัตโนมัติในตัวมันเอง**
ทำให้เราต้องเขียนแค่ "สิ่งที่ทำซ้ำแต่ละรอบ" (`ADD-ONE-ORDER`) แยกออกมาเป็น paragraph เดียวเท่านั้น
ส่วนตรรกะ "ก่อนเริ่ม" และ "หลังจบ" ลูปสามารถเขียนอยู่ใน `MAIN-PARA` ต่อเนื่องกันได้เลยโดยไม่ต้อง
แยก paragraph เพิ่ม เพราะไม่มีการกระโดดข้ามไปมาที่ต้องจัดการเองอีกต่อไป

---

## ขั้นตอนที่ 804: ลด Nested IF ที่ลึกด้วย EVALUATE

### ปัญหา: IF ซ้อนกันหลายชั้น

```cobol
       CLASSIFY-AND-SHOW.
           IF WS-BALANCE < 1000.00
               MOVE "BRONZE" TO WS-TIER
           ELSE
               IF WS-BALANCE < 10000.00
                   MOVE "SILVER" TO WS-TIER
               ELSE
                   IF WS-BALANCE < 50000.00
                       MOVE "GOLD" TO WS-TIER
                   ELSE
                       MOVE "PLATINUM" TO WS-TIER
                   END-IF
               END-IF
           END-IF.
           DISPLAY "balance=" WS-BALANCE " tier=" WS-TIER.
```

**ผลลัพธ์จริง (ทดสอบด้วยยอดเงิน 5 ระดับ):**

```
balance=0000500.00 tier=BRONZE
balance=0025000.00 tier=GOLD
balance=0049500.00 tier=GOLD
balance=0074000.00 tier=PLATINUM
balance=0098500.00 tier=PLATINUM
```

สังเกตว่า `END-IF` ปรากฏถึง 3 ครั้งซ้อนกัน — ถ้าต้องเพิ่ม tier ใหม่อีกระดับ (เช่น "DIAMOND" สำหรับ
ยอดเงินเกิน 100,000) จะต้องเพิ่มการซ้อน `IF` เข้าไปอีกชั้นหนึ่ง ทำให้ยิ่งอ่านยากขึ้นเรื่อย ๆ แบบ
ไม่มีขีดจำกัด

### หลังการ Refactor: ใช้ EVALUATE TRUE

```cobol
       CLASSIFY-AND-SHOW.
           EVALUATE TRUE
               WHEN WS-BALANCE < 1000.00
                   MOVE "BRONZE" TO WS-TIER
               WHEN WS-BALANCE < 10000.00
                   MOVE "SILVER" TO WS-TIER
               WHEN WS-BALANCE < 50000.00
                   MOVE "GOLD" TO WS-TIER
               WHEN OTHER
                   MOVE "PLATINUM" TO WS-TIER
           END-EVALUATE.
           DISPLAY "balance=" WS-BALANCE " tier=" WS-TIER.
```

**ผลลัพธ์จริงหลัง refactor พร้อมพิสูจน์ด้วย diff:**

```bash
diff <(./step804before) <(./step804after) && echo "IDENTICAL OUTPUT"
```

```
balance=0000500.00 tier=BRONZE
balance=0025000.00 tier=GOLD
balance=0049500.00 tier=GOLD
balance=0074000.00 tier=PLATINUM
balance=0098500.00 tier=PLATINUM
IDENTICAL OUTPUT
```

### อธิบายจุดสำคัญ

- `EVALUATE TRUE` (ทบทวนจาก Part 011 และ Part 041) คือรูปแบบพิเศษที่ให้แต่ละ `WHEN` เป็นเงื่อนไข
  boolean ของตัวเอง (แทนที่จะเทียบค่าคงที่กับตัวแปรตัวเดียวแบบ `EVALUATE` ปกติ) ทำให้เขียนตรรกะ
  แบบ "if-elseif-elseif-else" ได้โดย**ไม่ต้องซ้อนกันเลยแม้แต่ชั้นเดียว**
- ทุก `WHEN` อยู่ที่**ระดับการเยื้องเดียวกัน** — การเพิ่ม tier ใหม่ทำได้ง่ายมากเพียงเพิ่ม `WHEN`
  บรรทัดเดียวก่อน `WHEN OTHER` โดยไม่กระทบโครงสร้างที่เหลือเลย
- `EVALUATE` มี `END-EVALUATE` เพียงตัวเดียวปิดท้าย เทียบกับ `END-IF` ที่ซ้อนกัน 3 ชั้นในเวอร์ชัน
  before — ลดโอกาสที่จะปิด scope terminator ผิดตำแหน่งเวลาแก้ไขโค้ดในอนาคต

### ข้อควรระวัง

- **ลำดับของ `WHEN` มีความสำคัญ** เหมือนกับลำดับของ `IF`/`ELSE IF` เดิม — `EVALUATE` จะตรวจสอบ
  จากบนลงล่างและหยุดที่เงื่อนไขแรกที่เป็นจริง ถ้าสลับลำดับ `WHEN` ผิด (เช่น เอา `WHEN WS-BALANCE <
  50000.00` ไว้ก่อน `WHEN WS-BALANCE < 1000.00`) ผลลัพธ์จะผิดทันทีเพราะเงื่อนไขที่กว้างกว่าจะจับ
  ค่าที่ควรตกไปอยู่ในกลุ่มที่แคบกว่าไปก่อน
- `WHEN OTHER` ควรอยู่**ท้ายสุดเสมอ** (ทำหน้าที่เหมือน `ELSE` สุดท้าย) — ถ้าลืมใส่และไม่มีเงื่อนไข
  ใดตรงเลย `EVALUATE` จะไม่ทำอะไรเลยโดยไม่มี error ใด ๆ แจ้งเตือน (ค่าตัวแปรเป้าหมายจะคงค่าเดิม
  ที่ค้างอยู่ก่อนหน้า)

### แบบฝึกหัดที่ 804.1

**โจทย์**: จงเพิ่ม tier ใหม่ชื่อ "DIAMOND" สำหรับยอดเงินตั้งแต่ 100,000.00 ขึ้นไป เข้าไปในเวอร์ชัน
`EVALUATE TRUE` ข้างต้น โดยไม่กระทบ tier อื่นที่มีอยู่เดิม

**เฉลย**:

```cobol
       CLASSIFY-AND-SHOW.
           EVALUATE TRUE
               WHEN WS-BALANCE < 1000.00
                   MOVE "BRONZE" TO WS-TIER
               WHEN WS-BALANCE < 10000.00
                   MOVE "SILVER" TO WS-TIER
               WHEN WS-BALANCE < 50000.00
                   MOVE "GOLD" TO WS-TIER
               WHEN WS-BALANCE < 100000.00
                   MOVE "PLATINUM" TO WS-TIER
               WHEN OTHER
                   MOVE "DIAMOND" TO WS-TIER
           END-EVALUATE.
```

เพียงเพิ่ม `WHEN WS-BALANCE < 100000.00` เข้าไปก่อน `WHEN OTHER` (ซึ่งเดิมทำหน้าที่ "PLATINUM"
ตอนนี้เปลี่ยนไปทำหน้าที่ "DIAMOND" แทน) โดยไม่ต้องแก้ไขโครงสร้างของ `WHEN` เดิมที่มีอยู่แล้วแม้แต่
บรรทัดเดียว — นี่คือประโยชน์ที่ชัดเจนของ `EVALUATE TRUE` เทียบกับ Nested IF ที่การเพิ่มเงื่อนไข
ใหม่ตรงกลางจะต้องแก้ไข indentation ของโค้ดทั้งหมดที่อยู่ถัดจากจุดที่แทรกลงไป

---

## ขั้นตอนที่ 805: Extract Paragraph เป็น Subprogram ที่เรียกใช้ซ้ำได้

### ปัญหา: ตรรกะเดียวกันถูกคัดลอกวางซ้ำ (Inline Duplication)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP805BEFORE.
       AUTHOR. COBOL-COURSE.

      *> BEFORE: tax calculation logic is duplicated INLINE inside a
      *> big PROCEDURE DIVISION. If another program needs the exact
      *> same tax rule, that logic gets copy-pasted (and can drift
      *> out of sync) instead of being reused.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AMOUNT-1              PIC 9(7)V99 VALUE 1000.00.
       01  WS-TAX-1                 PIC 9(7)V99 VALUE 0.
       01  WS-AMOUNT-2              PIC 9(7)V99 VALUE 2500.00.
       01  WS-TAX-2                 PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-TAX-1 ROUNDED = WS-AMOUNT-1 * 0.07.
           DISPLAY "invoice 1 tax=" WS-TAX-1.
           COMPUTE WS-TAX-2 ROUNDED = WS-AMOUNT-2 * 0.07.
           DISPLAY "invoice 2 tax=" WS-TAX-2.
           STOP RUN.
```

**ผลลัพธ์จริง:**

```
invoice 1 tax=0000070.00
invoice 2 tax=0000175.00
```

สังเกตว่า `COMPUTE ... = ... * 0.07` ถูกเขียนซ้ำถึง 2 ครั้ง — ถ้ามีใบแจ้งหนี้ที่ 3, 4, 5 เพิ่มขึ้น
มาอีก ตรรกะนี้จะถูกคัดลอกวางซ้ำไปเรื่อย ๆ และถ้าวันหนึ่งอัตราภาษีเปลี่ยนแปลง จะต้องไปแก้ไขทุกจุด
ที่คัดลอกวางไว้ ซึ่งเสี่ยงสูงมากที่จะแก้ไม่ครบ (เหมือนปัญหาที่ Part 078 ขั้นตอนที่ 775 เคยพิสูจน์
ผลกระทบของการเปลี่ยนค่าคงที่ผิดพลาด)

### หลังการ Refactor: แยกเป็น Subprogram

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP805TAXCALC.
       AUTHOR. COBOL-COURSE.

      *> AFTER (extracted): the tax calculation now lives in ONE
      *> subprogram that any caller can CALL, instead of being
      *> copy-pasted inline wherever it is needed.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AMOUNT                PIC 9(7)V99.
       01  LK-TAX                   PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT LK-TAX.
       MAIN-PARA.
           COMPUTE LK-TAX ROUNDED = LK-AMOUNT * 0.07.
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP805AFTER.
       AUTHOR. COBOL-COURSE.

      *> AFTER: the main program no longer contains any tax FORMULA
      *> at all - it just calls the extracted subprogram twice. Both
      *> invoices now share exactly one source of truth for tax rules.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AMOUNT-1              PIC 9(7)V99 VALUE 1000.00.
       01  WS-TAX-1                 PIC 9(7)V99 VALUE 0.
       01  WS-AMOUNT-2              PIC 9(7)V99 VALUE 2500.00.
       01  WS-TAX-2                 PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "STEP805TAXCALC" USING WS-AMOUNT-1 WS-TAX-1.
           DISPLAY "invoice 1 tax=" WS-TAX-1.
           CALL "STEP805TAXCALC" USING WS-AMOUNT-2 WS-TAX-2.
           DISPLAY "invoice 2 tax=" WS-TAX-2.
           STOP RUN.
```

**คอมไพล์และรันจริง พร้อมพิสูจน์ด้วย diff:**

```bash
cobc -x -o step805after step805after.cob step805taxcalc.cob
./step805after
diff <(./step805before) <(./step805after) && echo "IDENTICAL OUTPUT"
```

```
invoice 1 tax=0000070.00
invoice 2 tax=0000175.00
IDENTICAL OUTPUT
```

### อธิบายจุดสำคัญ

- นี่คือรูปแบบ Refactoring คลาสสิกที่เรียกว่า **"Extract Method"** (หรือในบริบท COBOL คือ "Extract
  Subprogram" เพราะ COBOL ไม่มีแนวคิด method ภายในไฟล์เดียวข้ามขอบเขตการคอมไพล์เหมือนภาษา OOP) —
  ดึงตรรกะที่ซ้ำกันออกมาเป็นหน่วยเดียวที่เรียกใช้ซ้ำได้
- ตอนนี้ `STEP805AFTER` (โปรแกรมหลัก) **ไม่มีสูตรคำนวณภาษีหลงเหลืออยู่เลย** — มีแต่การเรียก `CALL`
  เท่านั้น ถ้าวันหนึ่งอัตราภาษีเปลี่ยนแปลง จะต้องแก้ไขแค่ **ที่เดียว** คือใน `STEP805TAXCALC`
- Subprogram ที่ถูก extract ออกมาแบบนี้**ยังสอดคล้องกับหลักการ pure function-like** ที่ Part 079
  สอนไว้ ทำให้นอกจากจะลดความซ้ำซ้อนแล้ว ยังทำให้ตรรกะนี้**ทดสอบได้ง่ายขึ้น**ด้วยในคราวเดียวกัน
  (สามารถเขียน Test Driver ตามรูปแบบ Part 079 มาทดสอบ `STEP805TAXCALC` แยกได้ทันที)

### ข้อควรระวัง

- การ extract paragraph เป็น subprogram ทำให้เกิด**ค่าใช้จ่ายของการ `CALL`** ข้าม compilation
  unit (ทบทวนความเสี่ยงเรื่องพารามิเตอร์ไม่ตรงกันจาก Part 031 ขั้นตอนที่ 304) ซึ่งต่างจากการเรียก
  paragraph ภายในไฟล์เดียวกันด้วย `PERFORM` ที่ compiler ตรวจสอบความถูกต้องได้ครบถ้วนกว่า — ควร
  extract เป็น subprogram แยกไฟล์เมื่อต้องการใช้ตรรกะนั้นร่วมกัน**ข้ามหลายโปรแกรม**จริง ๆ เท่านั้น
  ถ้าใช้แค่ภายในโปรแกรมเดียว การ extract เป็น**paragraph ภายในไฟล์เดียวกัน**ที่เรียกด้วย `PERFORM`
  ก็เพียงพอและปลอดภัยกว่า (ไม่มีความเสี่ยงเรื่องพารามิเตอร์ข้ามไฟล์เลย)
- อย่าลืมว่าการ extract แบบนี้เปลี่ยนคำสั่ง build ด้วย — ต้องคอมไพล์รวมทั้งสองไฟล์เสมอ
  (`cobc -x -o step805after step805after.cob step805taxcalc.cob`) มิฉะนั้นจะเกิด linker error

### แบบฝึกหัดที่ 805.1

**โจทย์**: จงอธิบายว่าทำไม subprogram ที่ถูก extract ออกมา (`STEP805TAXCALC`) ถึง**ง่ายต่อการเขียน
unit test ตามแนวทาง Part 079** มากกว่าตรรกะเดิมที่ฝังอยู่ใน `MAIN-PARA` ของโปรแกรม before

**เฉลย**: ในเวอร์ชัน before ตรรกะการคำนวณภาษีผูกติดอยู่กับตัวแปรเฉพาะของแต่ละใบแจ้งหนี้
(`WS-AMOUNT-1`/`WS-TAX-1`, `WS-AMOUNT-2`/`WS-TAX-2`) และอยู่ปะปนกับ `DISPLAY` ภายใน `MAIN-PARA`
เดียวกัน ทำให้ไม่มีทาง "เรียกเฉพาะส่วนคำนวณภาษี" แยกจากส่วนอื่นได้เลยโดยไม่รันทั้งโปรแกรม (รวมถึง
`STOP RUN` ที่จะจบการทำงานทันที) ในขณะที่เวอร์ชัน after, `STEP805TAXCALC` เป็นหน่วยอิสระที่รับ
`LK-AMOUNT` คืน `LK-TAX` ผ่าน `LINKAGE SECTION` เท่านั้น ไม่มี `DISPLAY` หรือ `STOP RUN` ภายในตัวมัน
เอง ทำให้สามารถเขียน Test Driver ตามรูปแบบ Part 079 (เช่น `CALL "STEP805TAXCALC" USING
test-amount test-actual` แล้วเทียบกับ expected) มาทดสอบมันแยกต่างหากได้ทันที โดยไม่ต้องพึ่งพา
บริบทของโปรแกรมหลักเลยแม้แต่น้อย — นี่คือตัวอย่างที่แสดงให้เห็นว่า **การ refactor เพื่อลดความซ้ำซ้อน
กับการออกแบบให้ทดสอบง่าย (testability) มักไปด้วยกันเสมอ**

---

## ขั้นตอนที่ 806: กำจัด Magic Numbers ด้วย 88-level Condition Names และ Named Constants

### ปัญหา: Magic Numbers ที่ไม่มีใครรู้ความหมาย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP806BEFORE.
       AUTHOR. COBOL-COURSE.

      *> BEFORE: "magic numbers" scattered through the logic. A
      *> reader has no idea what 3 or 7 MEAN without tracing usage.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STATUS-CODE           PIC 9(1) VALUE 0.
       01  WS-AMOUNT                PIC 9(7)V99 VALUE 1000.00.
       01  WS-TAX                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           IF WS-STATUS-CODE = 3
               DISPLAY "Order is on hold."
           END-IF.
           COMPUTE WS-TAX ROUNDED = WS-AMOUNT * 0.07.
           DISPLAY "tax=" WS-TAX.
           STOP RUN.
```

**ผลลัพธ์จริง:**

```
tax=0000070.00
```

คำถามที่โค้ดนี้ตอบไม่ได้ด้วยตัวเอง: `WS-STATUS-CODE = 3` หมายความว่าอะไร? เลข `3` คือรหัสอะไร?
`0.07` คืออัตราอะไร ทำไมถึงเป็นค่านี้? ผู้อ่านต้องไปค้นหาเอกสารภายนอกหรือถามคนที่เขียนโค้ดเดิม
เท่านั้นถึงจะรู้คำตอบ

### หลังการ Refactor: ตั้งชื่อทุกค่าที่มีความหมาย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP806AFTER.
       AUTHOR. COBOL-COURSE.

      *> AFTER: the meaning of "3" is now a named 88-level condition,
      *> and the tax rate "0.07" is a named constant declared once.
      *> Both read like English and can only be changed in ONE place.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STATUS-CODE           PIC 9(1) VALUE 0.
           88  ORDER-ON-HOLD                 VALUE 3.
           88  ORDER-SHIPPED                 VALUE 1.
           88  ORDER-CANCELLED                VALUE 9.
       01  WS-SALES-TAX-RATE        PIC 9V9999 VALUE 0.0700.
       01  WS-AMOUNT                PIC 9(7)V99 VALUE 1000.00.
       01  WS-TAX                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           IF ORDER-ON-HOLD
               DISPLAY "Order is on hold."
           END-IF.
           COMPUTE WS-TAX ROUNDED = WS-AMOUNT * WS-SALES-TAX-RATE.
           DISPLAY "tax=" WS-TAX.
           STOP RUN.
```

**ผลลัพธ์จริงหลัง refactor พร้อมพิสูจน์ด้วย diff:**

```bash
diff <(./step806before) <(./step806after) && echo "IDENTICAL OUTPUT"
```

```
tax=0000070.00
IDENTICAL OUTPUT
```

### อธิบายจุดสำคัญ

- `88 ORDER-ON-HOLD VALUE 3.` (ทบทวน 88-level condition names จาก Part 010) เปลี่ยน `IF
  WS-STATUS-CODE = 3` ที่อ่านไม่รู้เรื่อง ให้กลายเป็น `IF ORDER-ON-HOLD` ที่**อ่านเป็นภาษาอังกฤษ
  ได้ตรง ๆ ทันที** — นี่คือตัวอย่างที่ชัดเจนที่สุดของปรัชญาการออกแบบ COBOL ดั้งเดิมของ Grace
  Hopper (Part 001) ที่ต้องการให้โค้ดอ่านคล้ายภาษาอังกฤษ
- `WS-SALES-TAX-RATE PIC 9V9999 VALUE 0.0700` แยกค่าคงที่ `0.07` ออกมาเป็นตัวแปรที่มีชื่อความหมาย
  ชัดเจน — ถ้าอัตราภาษีเปลี่ยนแปลงในอนาคต จะต้องแก้ไขแค่ **บรรทัด `VALUE` เดียว** แทนที่จะต้องไป
  ค้นหาทุกจุดในโปรแกรมที่พิมพ์ `0.07` ไว้ตรง ๆ (ยิ่งมีประโยชน์มากเมื่อรวมกับเทคนิค Extract
  Subprogram จากขั้นตอนที่ 805 ที่ทำให้ค่านี้ปรากฏอยู่แค่จุดเดียวในระบบทั้งหมด)
- การประกาศ 88-level หลายตัว (`ORDER-ON-HOLD`, `ORDER-SHIPPED`, `ORDER-CANCELLED`) พร้อมกันในที่
  เดียว ทำหน้าที่เหมือน**เอกสารอ้างอิงในตัวโค้ดเอง**ว่ารหัสสถานะแต่ละค่าคืออะไรบ้าง โดยไม่ต้องแยก
  เอกสารภายนอกเลย

### ข้อควรระวัง

- 88-level condition name เป็นแค่ **"ชื่อเล่น" สำหรับเงื่อนไขการเทียบค่า** — มันไม่ได้เปลี่ยน
  `WS-STATUS-CODE` ให้กลายเป็นชนิดข้อมูลใหม่แต่อย่างใด `WS-STATUS-CODE` ยังคงเป็น `PIC 9(1)`
  ธรรมดาที่รับค่าอื่นนอกเหนือจาก 1, 3, 9 ได้เสมอถ้าไม่มีการตรวจสอบ validation เพิ่มเติม
- ต้องระวังไม่ให้ตั้งชื่อ named constant (เช่น `WS-SALES-TAX-RATE`) ให้เข้าใจผิดว่าเป็นค่าที่
  "แก้ไขไม่ได้" — ใน COBOL ตัวแปรที่ประกาศด้วย `VALUE` ยังคงเป็นตัวแปรที่ `MOVE` ทับค่าใหม่ได้เสมอ
  ระหว่างการทำงานของโปรแกรม (COBOL ไม่มีแนวคิด `CONST`/`FINAL` แบบภาษาสมัยใหม่บางภาษา) ทีมต้องอาศัย
  วินัยและ Code Review เพื่อป้องกันไม่ให้มีใครเขียนโค้ดที่ `MOVE` ค่าใหม่ทับตัวแปรที่ตั้งใจให้เป็น
  ค่าคงที่โดยไม่ตั้งใจ

### แบบฝึกหัดที่ 806.1

**โจทย์**: จงเพิ่มการตรวจสอบด้วย `EVALUATE` (ทบทวนจากขั้นตอนที่ 804) ที่แสดงข้อความต่างกันตาม
สถานะทั้ง 3 แบบ (`ORDER-ON-HOLD`, `ORDER-SHIPPED`, `ORDER-CANCELLED`) โดยใช้ 88-level condition
name แทนการเทียบตัวเลขตรง ๆ

**เฉลย**:

```cobol
           EVALUATE TRUE
               WHEN ORDER-ON-HOLD
                   DISPLAY "Order is on hold."
               WHEN ORDER-SHIPPED
                   DISPLAY "Order has been shipped."
               WHEN ORDER-CANCELLED
                   DISPLAY "Order was cancelled."
               WHEN OTHER
                   DISPLAY "Unknown order status."
           END-EVALUATE.
```

การใช้ `EVALUATE TRUE` ร่วมกับ 88-level condition name (`WHEN ORDER-ON-HOLD` แทนที่จะเป็น `WHEN
WS-STATUS-CODE = 3`) ทำให้โค้ดทั้งบล็อกอ่านเป็นภาษาอังกฤษได้เกือบสมบูรณ์แบบ ("evaluate true, when
order on hold, display order is on hold") ซึ่งเป็นการผสมผสานสองเทคนิค (Named Condition +
EVALUATE) จากขั้นตอนที่ 804 และ 806 เข้าด้วยกัน แสดงให้เห็นว่าเทคนิค Refactoring หลายแบบสามารถ
ทำงานเสริมกันได้ในโค้ดจริงชิ้นเดียว

---

## ขั้นตอนที่ 807: ลดความซ้ำซ้อนของโค้ดด้วย Shared Paragraph

### ปัญหา: ตรรกะการแสดงผลถูกคัดลอกวางซ้ำ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP807BEFORE.
       AUTHOR. COBOL-COURSE.

      *> BEFORE: the same "print a report line" logic is duplicated
      *> three times with only the data changing. Any format change
      *> (e.g. adding a currency symbol) must be edited in 3 places.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE-NO               PIC 9(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 1 TO WS-LINE-NO.
           DISPLAY "Line " WS-LINE-NO ": Somchai    - 1000.00".

           MOVE 2 TO WS-LINE-NO.
           DISPLAY "Line " WS-LINE-NO ": Prasert    - 2500.50".

           MOVE 3 TO WS-LINE-NO.
           DISPLAY "Line " WS-LINE-NO ": Wanida     - 750.25".
           STOP RUN.
```

**ผลลัพธ์จริง:**

```
Line 01: Somchai    - 1000.00
Line 02: Prasert    - 2500.50
Line 03: Wanida     - 750.25
```

### หลังการ Refactor: แยกเป็น Paragraph ที่ใช้ร่วมกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP807AFTER.
       AUTHOR. COBOL-COURSE.

      *> AFTER: the duplicated print logic is now ONE shared paragraph
      *> (PRINT-REPORT-LINE), reused through PERFORM. A format change
      *> now only needs to be made in a single place.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE-NO               PIC 9(2).
       01  WS-CUST-NAME             PIC X(10).
       01  WS-CUST-AMOUNT           PIC 9(5)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 1 TO WS-LINE-NO.
           MOVE "Somchai" TO WS-CUST-NAME.
           MOVE 1000.00 TO WS-CUST-AMOUNT.
           PERFORM PRINT-REPORT-LINE.

           MOVE 2 TO WS-LINE-NO.
           MOVE "Prasert" TO WS-CUST-NAME.
           MOVE 2500.50 TO WS-CUST-AMOUNT.
           PERFORM PRINT-REPORT-LINE.

           MOVE 3 TO WS-LINE-NO.
           MOVE "Wanida" TO WS-CUST-NAME.
           MOVE 750.25 TO WS-CUST-AMOUNT.
           PERFORM PRINT-REPORT-LINE.
           STOP RUN.

       PRINT-REPORT-LINE.
           DISPLAY "Line " WS-LINE-NO ": " WS-CUST-NAME "    - "
               WS-CUST-AMOUNT.
```

**ผลลัพธ์จริงหลัง refactor:**

```
Line 01: Somchai       - 01000.00
Line 02: Prasert       - 02500.50
Line 03: Wanida        - 00750.25
```

### วิเคราะห์: ทำไมผลลัพธ์ไม่เหมือนกันตัวต่อตัว 100%

สังเกตความแตกต่างเล็กน้อย: เวอร์ชัน before แสดง `Somchai    - 1000.00` (ตัวเลขไม่เติมศูนย์นำหน้า
เพราะเป็น literal ข้อความล้วน) ในขณะที่เวอร์ชัน after แสดง `Somchai       - 01000.00` (ตัวเลขเติม
ศูนย์นำหน้าเพราะตอนนี้ `WS-CUST-AMOUNT` เป็น `PIC 9(5)V99` field จริง ไม่ใช่ literal ข้อความอีก
ต่อไป) **นี่ไม่ใช่ผลลัพธ์ที่ "เหมือนกันทุกประการ" ตามนิยาม Refactoring ที่เข้มงวดในขั้นตอนที่ 801**

### บทเรียนสำคัญจากความแตกต่างนี้

การเปลี่ยนจาก literal ข้อความล้วน (เดิมพิมพ์ตัวเลขและช่องว่างตรง ๆ ในสตริง) มาเป็น field ที่มี
PICTURE ของตัวเอง **เปลี่ยนวิธีที่ COBOL แสดงผลตัวเลขนั้นด้วย** (เติมศูนย์นำหน้าให้ครบตามความกว้าง
ที่ประกาศไว้) แม้ผู้เขียนจะตั้งใจแค่ "ลดโค้ดซ้ำ" เท่านั้น แต่การเปลี่ยนแปลงนี้**ทำให้ผลลัพธ์
เปลี่ยนไปโดยไม่ตั้งใจ** — นี่คือตัวอย่างจริงที่แสดงให้เห็นว่า **ทำไมการรัน diff เปรียบเทียบ
ผลลัพธ์ก่อน/หลังทุกครั้งถึงสำคัญมาก** (ตามหลักการ Regression Safety Net จากขั้นตอนที่ 801) — ถ้า
ไม่ได้รัน diff เทียบผลลัพธ์จริง อาจไม่มีใครสังเกตเห็นการเปลี่ยนแปลงรูปแบบการแสดงผลเล็ก ๆ น้อย ๆ
แบบนี้เลยจนกว่าจะมีคนบ่นว่ารายงานหน้าตาเปลี่ยนไป

### วิธีแก้ไขให้เป็น Refactoring ที่แท้จริง

ถ้าต้องการคงรูปแบบการแสดงผลเดิมทุกประการ ควรใช้ `PIC X` (ข้อความ) แทนสำหรับ field ที่เดิมมาจาก
literal ข้อความ หรือปรับการจัดรูปแบบการแสดงผลใน `PRINT-REPORT-LINE` ให้ตรงกับรูปแบบเดิมเป๊ะ ๆ
ก่อนจะถือว่าเป็น "Refactoring ที่สมบูรณ์" ตามนิยามที่เข้มงวด

### ข้อควรระวัง

- **นี่คือกับดักที่พบบ่อยที่สุดอย่างหนึ่งของการ refactor โค้ด COBOL**: การเปลี่ยนจาก literal ไปเป็น
  field ที่มี PICTURE เปลี่ยนพฤติกรรมการแสดงผล (เติมศูนย์, การจัดตำแหน่งซ้าย/ขวา) โดยอัตโนมัติ
  เสมอ ต้องตรวจสอบผลลัพธ์จริงทุกครั้งหลัง refactor แบบนี้ ไม่ใช่แค่ดูว่า "โค้ดคอมไพล์ผ่าน" เท่านั้น
- ถ้าโปรแกรมมี Golden Output Test อยู่แล้ว (Part 078) การ refactor แบบนี้จะถูก**จับได้ทันที**โดย
  automated test เอง (ผลลัพธ์ไม่ตรงกับ expected file) — นี่คือเหตุผลสำคัญอีกข้อที่ยืนยันว่าทำไม
  ระบบทดสอบอัตโนมัติต้องมีอยู่**ก่อน**ที่จะเริ่ม refactor โค้ดใด ๆ เสมอ

### แบบฝึกหัดที่ 807.1

**โจทย์**: จงอธิบายว่าถ้าโปรแกรม `STEP807BEFORE` มี Golden Output Test (ตามรูปแบบ Part 078) เก็บ
ผลลัพธ์เดิมไว้อยู่แล้ว การ refactor เป็น `STEP807AFTER` แบบที่แสดงในขั้นตอนนี้จะทำให้เกิดอะไรขึ้น
เมื่อรัน pipeline

**เฉลย**: pipeline จะรายงาน **TEST FAIL** ทันที เพราะ `diff` ระหว่างผลลัพธ์จริงของ `STEP807AFTER`
กับไฟล์ expected ที่บันทึกผลลัพธ์ของ `STEP807BEFORE` ไว้จะไม่ตรงกัน (ความแตกต่างเรื่องเลขศูนย์
นำหน้าตามที่วิเคราะห์ไว้ข้างต้น) นี่คือตัวอย่างที่ดีของการที่ระบบทดสอบอัตโนมัติทำหน้าที่เป็น
**Regression Safety Net ที่แท้จริง**: มันไม่สนใจว่าผู้เขียนโค้ด "ตั้งใจ" แค่จะลดความซ้ำซ้อนเท่านั้น
โดยไม่ได้ตั้งใจเปลี่ยนพฤติกรรม — มันตรวจจับ**ผลลัพธ์ที่เปลี่ยนไปจริง**อย่างเป็นกลาง (objective)
โดยไม่สนใจเจตนาเลย ทำให้ทีมรู้ทันทีว่าต้องกลับไปแก้ไขการจัดรูปแบบใน `PRINT-REPORT-LINE` ให้ตรงกับ
รูปแบบเดิมก่อนที่จะ merge การเปลี่ยนแปลงนี้เข้า `main` ได้ (เชื่อมโยงกับ Branch Protection Rule
จาก Part 080 ขั้นตอนที่ 798 ที่บังคับ CI ต้องผ่านก่อน merge เสมอ)

---

## ขั้นตอนที่ 808: จัดโครงสร้าง PROCEDURE DIVISION ด้วยรูปแบบ Driver Paragraph

### แนวคิด Driver Paragraph Pattern

**Driver Paragraph** คือรูปแบบการจัดโครงสร้างที่ให้ `MAIN-PARA` (หรือ paragraph แรกของโปรแกรม)
ทำหน้าที่เป็นเหมือน **"สารบัญ"** ของโปรแกรมทั้งหมด — เป็นรายการสั้น ๆ ของ `PERFORM` ที่เรียงตามลำดับ
การทำงาน โดยไม่มีรายละเอียดการคำนวณหรือ `IF`/`EVALUATE` ใด ๆ ปนอยู่เลย รายละเอียดทั้งหมดถูกผลักไป
ไว้ใน paragraph ย่อยที่มีชื่อสื่อความหมายชัดเจน

### ตัวอย่าง: โปรแกรมคำนวณเงินเดือนที่จัดโครงสร้างแบบ Driver

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP808DRIVER.
       AUTHOR. COBOL-COURSE.

      *> "Driver paragraph" pattern: MAIN-PARA reads like a table of
      *> contents - a short, flat list of well-named steps executed in
      *> order. Anyone reading MAIN-PARA alone understands the whole
      *> program's shape without reading a single line of detail yet.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EMP-NAME              PIC X(15) VALUE SPACES.
       01  WS-HOURS-WORKED          PIC 9(3) VALUE 0.
       01  WS-HOURLY-RATE           PIC 9(3)V99 VALUE 0.
       01  WS-GROSS-PAY             PIC 9(7)V99 VALUE 0.
       01  WS-TAX-DEDUCTION         PIC 9(7)V99 VALUE 0.
       01  WS-NET-PAY               PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM INITIALIZE-PAYROLL-DATA.
           PERFORM CALCULATE-GROSS-PAY.
           PERFORM CALCULATE-TAX-DEDUCTION.
           PERFORM CALCULATE-NET-PAY.
           PERFORM PRINT-PAYSLIP.
           STOP RUN.

       INITIALIZE-PAYROLL-DATA.
           MOVE "SOMCHAI JAIDEE" TO WS-EMP-NAME.
           MOVE 160 TO WS-HOURS-WORKED.
           MOVE 55.00 TO WS-HOURLY-RATE.

       CALCULATE-GROSS-PAY.
           COMPUTE WS-GROSS-PAY = WS-HOURS-WORKED * WS-HOURLY-RATE.

       CALCULATE-TAX-DEDUCTION.
           COMPUTE WS-TAX-DEDUCTION ROUNDED = WS-GROSS-PAY * 0.07.

       CALCULATE-NET-PAY.
           COMPUTE WS-NET-PAY = WS-GROSS-PAY - WS-TAX-DEDUCTION.

       PRINT-PAYSLIP.
           DISPLAY "Employee : " WS-EMP-NAME.
           DISPLAY "Gross Pay: " WS-GROSS-PAY.
           DISPLAY "Tax      : " WS-TAX-DEDUCTION.
           DISPLAY "Net Pay  : " WS-NET-PAY.
```

**ผลลัพธ์จริง:**

```
Employee : SOMCHAI JAIDEE
Gross Pay: 0008800.00
Tax      : 0000616.00
Net Pay  : 0008184.00
```

(ตรวจสอบด้วยมือ: 160 × 55.00 = 8800.00; 8800.00 × 0.07 = 616.00; 8800.00 − 616.00 = 8184.00 ✔)

### อธิบายจุดสำคัญ

- อ่านแค่ `MAIN-PARA` เพียงอย่างเดียว (5 บรรทัดของ `PERFORM`) **ก็สามารถเข้าใจภาพรวมของโปรแกรม
  ทั้งหมดได้ทันที** โดยไม่ต้องอ่านรายละเอียดการคำนวณเลยแม้แต่บรรทัดเดียว — นี่คือหัวใจของรูปแบบ
  Driver Paragraph
- ชื่อ paragraph แต่ละตัว (`INITIALIZE-PAYROLL-DATA`, `CALCULATE-GROSS-PAY`, ฯลฯ) ถูกตั้งด้วยรูป
  แบบ **กริยา + คำนาม** (Verb + Noun) ที่สื่อการกระทำชัดเจน ทำให้ `MAIN-PARA` อ่านได้เกือบเหมือน
  ประโยคภาษาอังกฤษต่อเนื่องกัน
- ทุก paragraph ย่อยมี**ความรับผิดชอบเดียว** (Single Responsibility — หลักการเดียวกับที่ Part 031
  ขั้นตอนที่ 310 เคยกล่าวถึงตอนออกแบบ subprogram) ทำให้แก้ไขหรือทดสอบแต่ละส่วนแยกกันได้ง่าย

### ข้อควรระวัง

- รูปแบบ Driver Paragraph เหมาะกับโปรแกรมที่มีขั้นตอนการทำงานเป็นลำดับชัดเจน (sequential) — ถ้า
  โปรแกรมมีการตัดสินใจแบบมีเงื่อนไขซับซ้อนว่าจะทำขั้นตอนไหนก่อนหลัง (ไม่ใช่ทำตามลำดับตายตัวเสมอ)
  อาจต้องออกแบบโครงสร้างเพิ่มเติม (เช่น ใช้ `EVALUATE` เพื่อเลือกกลุ่มของ `PERFORM` ที่จะรัน) แทนที่
  จะบังคับให้เป็น driver แบบเรียบง่ายเกินไป
- อย่าให้ `MAIN-PARA` "โกง" ด้วยการใส่ `IF`/`COMPUTE`/`DISPLAY` รายละเอียดปนเข้าไปแม้แต่บรรทัดเดียว
  — ถ้าเริ่มมีรายละเอียดปนเข้ามาใน driver paragraph แปลว่าถึงเวลาต้อง extract รายละเอียดนั้นออกไป
  เป็น paragraph ย่อยใหม่แล้ว

### แบบฝึกหัดที่ 808.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `MAIN-PARA` เรียก `PERFORM` ทั้ง 5 ตัวเรียงตามลำดับ (ไม่มี `IF`
คั่นกลางเลย) ถึงทำให้โปรแกรมนี้**ทำนายลำดับการทำงานได้แน่นอน 100%** ทุกครั้งที่รัน

**เฉลย**: เพราะไม่มีเงื่อนไขใด ๆ ที่จะทำให้ลำดับการทำงานเปลี่ยนแปลงไปจากที่เขียนไว้ใน `MAIN-PARA`
เลย — COBOL จะรัน `PERFORM INITIALIZE-PAYROLL-DATA` ก่อนเสมอ ตามด้วย `CALCULATE-GROSS-PAY`,
`CALCULATE-TAX-DEDUCTION`, `CALCULATE-NET-PAY`, และ `PRINT-PAYSLIP` ตามลำดับที่ปรากฏในโค้ดทุก
ครั้งไม่มีข้อยกเว้น (สอดคล้องกับหลักการ "Sequence" หนึ่งใน 4 แนวคิดของ Procedural Programming ที่
Part 001 ขั้นตอนที่ 8 เคยแนะนำไว้) คุณสมบัติของการทำนายได้แน่นอนนี้มีค่ามากในการ debug และการเขียน
เอกสาร เพราะไม่ว่าจะรันโปรแกรมกี่ครั้ง หรือใครเป็นผู้รัน ลำดับขั้นตอนจะเหมือนเดิมเสมอ ต่างจากโปรแกรม
ที่มี `GO TO`/เงื่อนไขซับซ้อนปนอยู่ใน control flow หลัก ที่อาจทำให้ลำดับการทำงานจริงขึ้นอยู่กับข้อมูล
input และคาดเดาได้ยากกว่ามากถ้าไม่อ่านโค้ดอย่างละเอียด

---

## ขั้นตอนที่ 809: Refactor อย่างปลอดภัยทีละก้าวด้วย Regression Test เป็นตาข่ายนิรภัย

### หลักการ: ทดสอบก่อน, Refactor, ทดสอบซ้ำ

ขั้นตอนนี้แสดงกระบวนการ refactor ที่ปลอดภัยที่สุดสำหรับ subprogram ที่มีความสำคัญ: **เขียนชุดทดสอบ
ให้ครอบคลุมพฤติกรรมปัจจุบันก่อน แล้วจึง refactor โครงสร้างภายใน โดยรันชุดทดสอบเดิมซ้ำทุกครั้งเพื่อ
ยืนยันว่ายังผ่านหมด**

### Version 1: Subprogram แบบ Nested IF (โครงสร้างเดิม)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP809SUB.
       AUTHOR. COBOL-COURSE.

      *> Version 1 of the internal implementation - nested IF style.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AMOUNT                PIC S9(7)V99.
       01  LK-CUST-TYPE             PIC X(1).
       01  LK-DISCOUNT              PIC S9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT LK-CUST-TYPE LK-DISCOUNT.
       MAIN-PARA.
           IF LK-CUST-TYPE = "V"
               COMPUTE LK-DISCOUNT ROUNDED = LK-AMOUNT * 0.10
           ELSE
               IF LK-CUST-TYPE = "R"
                   COMPUTE LK-DISCOUNT ROUNDED = LK-AMOUNT * 0.02
               ELSE
                   MOVE 0 TO LK-DISCOUNT
               END-IF
           END-IF.
           GOBACK.
```

### Test Driver ที่เขียนไว้ก่อนเริ่ม Refactor (Regression Safety Net)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP809TESTDRIVER.
       AUTHOR. COBOL-COURSE.

      *> Regression safety net: runs the SAME test cases against
      *> whichever version of STEP809SUB is linked in. If a refactor
      *> changes behaviour by accident, this test suite catches it.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEST-NAME             PIC X(40).
       01  WS-AMOUNT                PIC S9(7)V99.
       01  WS-CUST-TYPE             PIC X(1).
       01  WS-ACTUAL                PIC S9(7)V99.
       01  WS-EXPECTED              PIC S9(7)V99.
       01  WS-PASS-COUNT            PIC 9(3) VALUE 0.
       01  WS-FAIL-COUNT            PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "VIP 1000.00 -> 100.00" TO WS-TEST-NAME.
           MOVE 1000.00 TO WS-AMOUNT.
           MOVE "V" TO WS-CUST-TYPE.
           MOVE 100.00 TO WS-EXPECTED.
           CALL "STEP809SUB" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.

           MOVE "Regular 1000.00 -> 20.00" TO WS-TEST-NAME.
           MOVE 1000.00 TO WS-AMOUNT.
           MOVE "R" TO WS-CUST-TYPE.
           MOVE 20.00 TO WS-EXPECTED.
           CALL "STEP809SUB" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.

           MOVE "Unknown type -> 0.00" TO WS-TEST-NAME.
           MOVE 1000.00 TO WS-AMOUNT.
           MOVE "X" TO WS-CUST-TYPE.
           MOVE 0.00 TO WS-EXPECTED.
           CALL "STEP809SUB" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.

           MOVE "VIP 7.50 rounding -> 0.75" TO WS-TEST-NAME.
           MOVE 7.50 TO WS-AMOUNT.
           MOVE "V" TO WS-CUST-TYPE.
           MOVE 0.75 TO WS-EXPECTED.
           CALL "STEP809SUB" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.

           DISPLAY "TOTAL PASS=" WS-PASS-COUNT " FAIL=" WS-FAIL-COUNT.
           IF WS-FAIL-COUNT > 0
               MOVE 1 TO RETURN-CODE
           ELSE
               MOVE 0 TO RETURN-CODE
           END-IF
           STOP RUN.

       ASSERT-RESULT.
           IF WS-ACTUAL = WS-EXPECTED
               DISPLAY "[PASS] " WS-TEST-NAME
               ADD 1 TO WS-PASS-COUNT
           ELSE
               DISPLAY "[FAIL] " WS-TEST-NAME
                   " expected=" WS-EXPECTED " actual=" WS-ACTUAL
               ADD 1 TO WS-FAIL-COUNT
           END-IF.
```

**รัน Test Driver กับ Version 1 (baseline ก่อน refactor):**

```bash
cobc -x -o step809test step809testdriver.cob step809subv1.cob
./step809test
```

**ผลลัพธ์จริง:**

```
[PASS] VIP 1000.00 -> 100.00
[PASS] Regular 1000.00 -> 20.00
[PASS] Unknown type -> 0.00
[PASS] VIP 7.50 rounding -> 0.75
TOTAL PASS=004 FAIL=000
```

### Version 2: Refactor เป็น EVALUATE พร้อม Named Constants

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP809SUB.
       AUTHOR. COBOL-COURSE.

      *> Version 2 - refactored to EVALUATE with named constants.
      *> Same PROGRAM-ID, same PROCEDURE DIVISION USING interface,
      *> same observable behaviour - only the INTERNAL style changed.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-VIP-DISCOUNT-RATE     PIC 9V9999 VALUE 0.1000.
       01  WS-REGULAR-DISCOUNT-RATE PIC 9V9999 VALUE 0.0200.

       LINKAGE SECTION.
       01  LK-AMOUNT                PIC S9(7)V99.
       01  LK-CUST-TYPE             PIC X(1).
       01  LK-DISCOUNT              PIC S9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT LK-CUST-TYPE LK-DISCOUNT.
       MAIN-PARA.
           EVALUATE LK-CUST-TYPE
               WHEN "V"
                   COMPUTE LK-DISCOUNT ROUNDED =
                       LK-AMOUNT * WS-VIP-DISCOUNT-RATE
               WHEN "R"
                   COMPUTE LK-DISCOUNT ROUNDED =
                       LK-AMOUNT * WS-REGULAR-DISCOUNT-RATE
               WHEN OTHER
                   MOVE 0 TO LK-DISCOUNT
           END-EVALUATE.
           GOBACK.
```

**รัน Test Driver เดิมซ้ำกับ Version 2 (ไม่แก้ไข test driver แม้แต่บรรทัดเดียว):**

```bash
cobc -x -o step809test_v2 step809testdriver.cob step809subv2.cob
./step809test_v2
diff <(./step809test) <(./step809test_v2) && echo "REGRESSION TEST: IDENTICAL - REFACTOR IS SAFE"
```

**ผลลัพธ์จริง:**

```
[PASS] VIP 1000.00 -> 100.00
[PASS] Regular 1000.00 -> 20.00
[PASS] Unknown type -> 0.00
[PASS] VIP 7.50 rounding -> 0.75
TOTAL PASS=004 FAIL=000
REGRESSION TEST: IDENTICAL - REFACTOR IS SAFE
```

### อธิบายจุดสำคัญ

- **สังเกตว่า `PROGRAM-ID` ของทั้งสองเวอร์ชันเหมือนกันทุกประการ** (`STEP809SUB`) — Test Driver
  ไม่รู้เลยว่าข้างในถูกเปลี่ยนจาก Nested IF เป็น EVALUATE เพราะมันเรียกผ่าน `CALL "STEP809SUB"
  USING ...` เหมือนเดิมทุกประการ นี่คือหัวใจของหลักการ **Interface ไม่เปลี่ยน แม้ Implementation
  จะเปลี่ยน** ซึ่งเป็นรากฐานสำคัญที่สุดของการ refactor อย่างปลอดภัย
- **Test Driver ตัวเดิมไม่ต้องแก้ไขแม้แต่บรรทัดเดียว** — นี่คือข้อพิสูจน์ที่ชัดเจนที่สุดว่า
  Refactoring สำเร็จตามนิยามที่ถูกต้อง: ถ้า Refactor แล้วต้องแก้ไขชุดทดสอบตามไปด้วย นั่นมักเป็น
  สัญญาณว่าพฤติกรรมภายนอกได้เปลี่ยนไปแล้วจริง ๆ (เหมือนกรณีที่พิสูจน์ในขั้นตอนที่ 807)
- กระบวนการนี้คือสิ่งที่เรียกว่า **"Test-Driven Refactoring"**: มีชุดทดสอบที่ครอบคลุมอยู่ก่อนแล้ว
  (Part 079), ทำการเปลี่ยนแปลงโครงสร้างภายใน, แล้วรันชุดทดสอบเดิมซ้ำเพื่อยืนยันความปลอดภัย —
  วงจรนี้สามารถทำซ้ำได้หลายรอบเรื่อย ๆ ทุกครั้งที่ต้องการปรับปรุงโครงสร้างเพิ่มเติม

### ข้อควรระวัง

- Test-Driven Refactoring จะได้ผลดีก็ต่อเมื่อ**ชุดทดสอบเดิมครอบคลุมพฤติกรรมที่สำคัญเพียงพอ**
  (ทบทวน Boundary Value Analysis จาก Part 079 ขั้นตอนที่ 788) — ถ้าชุดทดสอบมีช่องโหว่ (เช่น ไม่เคย
  ทดสอบกรณี `LK-AMOUNT` เป็นค่าลบ) การ refactor อาจทำให้พฤติกรรมของกรณีที่ไม่ได้ทดสอบเปลี่ยนไปโดย
  ไม่มีใครรู้ตัวเลย แม้ test suite ทั้งหมดจะยังคง "PASS" อยู่ก็ตาม
- ควร commit การเขียน test suite (baseline) แยกจาก commit ที่ทำการ refactor จริง (ทบทวนจาก Part
  080 ขั้นตอนที่ 795 เรื่องการแยก commit) เพื่อให้เห็นชัดเจนในประวัติว่า "จุดไหนคือการวางตาข่าย
  นิรภัย" และ "จุดไหนคือการเปลี่ยนแปลงจริงที่ตาข่ายนั้นปกป้องอยู่"

### แบบฝึกหัดที่ 809.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `diff <(./step809test) <(./step809test_v2)` แสดงผลว่า "ไม่มีความ
ต่างกันเลย" (exit code 0) ถึงเป็นหลักฐานที่หนักแน่นกว่าการแค่อ่านโค้ด Version 2 ด้วยตาแล้วบอกว่า
"ตรรกะดูเหมือนจะเหมือนเดิม"

**เฉลย**: การอ่านโค้ดด้วยตาเป็นกระบวนการที่มนุษย์**มีโอกาสพลาดได้เสมอ** โดยเฉพาะกับเงื่อนไขที่ซับซ้อน
หรือกรณี edge case ที่ไม่ชัดเจนในทันทีที่มองเห็น (เช่น กรณี `LK-CUST-TYPE` เป็นค่าอื่นนอกเหนือจาก
"V" และ "R" ที่ทั้งสองเวอร์ชันต้องจัดการเหมือนกันแต่เขียนด้วยโครงสร้างต่างกันคนละแบบ — Nested IF
ใช้ `ELSE` ซ้อนกัน 2 ชั้น ในขณะที่ EVALUATE ใช้ `WHEN OTHER`) การรัน**ชุดทดสอบที่ครอบคลุมจริง**และ
เปรียบเทียบผลลัพธ์แบบอัตโนมัติด้วย `diff` เป็นการตรวจสอบเชิงประจักษ์ (empirical) ที่ไม่ขึ้นกับความ
ระมัดระวังหรือประสบการณ์ของผู้อ่านโค้ดเลย ตราบใดที่ test case ครอบคลุมกรณีสำคัญเพียงพอ (เหมือนที่
ออกแบบไว้ 4 กรณีในขั้นตอนนี้ ครอบคลุมทั้ง VIP, Regular, Unknown type, และการปัดเศษ) ผลลัพธ์ที่
`diff` ยืนยันว่าเหมือนกันทุกประการจึงเป็นหลักฐานที่น่าเชื่อถือกว่าการอ่านโค้ดด้วยตาเพียงอย่างเดียว
มาก — นี่คือเหตุผลที่วงการวิศวกรรมซอฟต์แวร์ทั้งหมด (ไม่ใช่แค่ COBOL) ยึดหลัก "test, don't just
read" เป็นมาตรฐานปฏิบัติในการ refactor โค้ดที่สำคัญ

---

## ขั้นตอนที่ 810: Capstone — Refactor โปรแกรม Payroll แบบ Legacy Spaghetti ให้เป็น Clean Code สมบูรณ์

### โปรแกรม Before: รวมปัญหาทุกแบบไว้ในที่เดียว

โปรแกรมนี้จำลองลักษณะของโค้ด COBOL legacy จริงที่มักพบ: ใช้ `GO TO` วนลูป, มี Nested IF ลึก, มี
Magic Numbers/Letters ('R', 'M', 'P'), และคำนวณค่าล่วงเวลาซ้ำซ้อนในหลายจุด:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP810BEFORE.
       AUTHOR. COBOL-COURSE.

      *> BEFORE: legacy-style "spaghetti" payroll program. Uses GO TO
      *> for looping, deeply nested IF with magic numbers/letters, and
      *> duplicated overtime-pay logic repeated for each employee type.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EMP-INDEX             PIC 9(1) VALUE 1.
       01  WS-EMP-TABLE.
           05  WS-EMP-ENTRY OCCURS 3 TIMES.
               10  WS-EMP-NAME      PIC X(15).
               10  WS-EMP-TYPE      PIC X(1).
               10  WS-EMP-HOURS     PIC 9(3).
               10  WS-EMP-RATE      PIC 9(3)V99.
       01  WS-GROSS-PAY             PIC 9(7)V99.

       PROCEDURE DIVISION.
       P1.
           MOVE "SOMCHAI JAIDEE" TO WS-EMP-NAME(1).
           MOVE "R" TO WS-EMP-TYPE(1).
           MOVE 180 TO WS-EMP-HOURS(1).
           MOVE 50.00 TO WS-EMP-RATE(1).

           MOVE "PRASERT KAEWTA" TO WS-EMP-NAME(2).
           MOVE "M" TO WS-EMP-TYPE(2).
           MOVE 170 TO WS-EMP-HOURS(2).
           MOVE 80.00 TO WS-EMP-RATE(2).

           MOVE "WANIDA SOMBOON" TO WS-EMP-NAME(3).
           MOVE "P" TO WS-EMP-TYPE(3).
           MOVE 90 TO WS-EMP-HOURS(3).
           MOVE 40.00 TO WS-EMP-RATE(3).
           GO TO P2.
       P2.
           IF WS-EMP-INDEX > 3
               GO TO P9
           END-IF.
           IF WS-EMP-TYPE(WS-EMP-INDEX) = "R"
               IF WS-EMP-HOURS(WS-EMP-INDEX) > 160
                   COMPUTE WS-GROSS-PAY =
                       (160 * WS-EMP-RATE(WS-EMP-INDEX))
                       + ((WS-EMP-HOURS(WS-EMP-INDEX) - 160)
                          * WS-EMP-RATE(WS-EMP-INDEX) * 1.5)
               ELSE
                   COMPUTE WS-GROSS-PAY =
                       WS-EMP-HOURS(WS-EMP-INDEX)
                       * WS-EMP-RATE(WS-EMP-INDEX)
               END-IF
           ELSE
               IF WS-EMP-TYPE(WS-EMP-INDEX) = "M"
                   COMPUTE WS-GROSS-PAY =
                       160 * WS-EMP-RATE(WS-EMP-INDEX)
               ELSE
                   IF WS-EMP-TYPE(WS-EMP-INDEX) = "P"
                       COMPUTE WS-GROSS-PAY =
                           WS-EMP-HOURS(WS-EMP-INDEX)
                           * WS-EMP-RATE(WS-EMP-INDEX)
                   ELSE
                       MOVE 0 TO WS-GROSS-PAY
                   END-IF
               END-IF
           END-IF.
           DISPLAY WS-EMP-NAME(WS-EMP-INDEX) " gross=" WS-GROSS-PAY.
           ADD 1 TO WS-EMP-INDEX.
           GO TO P2.
       P9.
           STOP RUN.
```

**ผลลัพธ์จริง:**

```
SOMCHAI JAIDEE  gross=0009500.00
PRASERT KAEWTA  gross=0012800.00
WANIDA SOMBOON  gross=0003600.00
```

(ตรวจสอบด้วยมือ: Somchai (R, 180 ชม.) = 160×50 + 20×50×1.5 = 8000+1500 = 9500.00; Prasert (M,
capped ที่ 160 ชม.) = 160×80 = 12800.00; Wanida (P, 90 ชม.) = 90×40 = 3600.00 ✔ ทั้งหมดถูกต้อง)

### โปรแกรม After: ประกอบทุกเทคนิคจาก Part นี้เข้าด้วยกัน

**Subprogram ที่ extract ตรรกะการคำนวณออกมา (แก้ปัญหา Nested IF + Magic Letters + Duplication):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP810GROSSPAY.
       AUTHOR. COBOL-COURSE.

      *> AFTER (extracted subprogram): one shared, testable rule for
      *> computing gross pay by employee type. No duplicated OT math.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-REGULAR-HOURS-LIMIT   PIC 9(3) VALUE 160.
       01  WS-OVERTIME-MULTIPLIER   PIC 9V9 VALUE 1.5.

       LINKAGE SECTION.
       01  LK-EMP-TYPE              PIC X(1).
       01  LK-HOURS                 PIC 9(3).
       01  LK-RATE                  PIC 9(3)V99.
       01  LK-GROSS-PAY             PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-EMP-TYPE LK-HOURS LK-RATE
               LK-GROSS-PAY.
       MAIN-PARA.
           EVALUATE TRUE
               WHEN LK-EMP-TYPE = "R" AND
                       LK-HOURS > WS-REGULAR-HOURS-LIMIT
                   COMPUTE LK-GROSS-PAY =
                       (WS-REGULAR-HOURS-LIMIT * LK-RATE)
                       + ((LK-HOURS - WS-REGULAR-HOURS-LIMIT)
                          * LK-RATE * WS-OVERTIME-MULTIPLIER)
               WHEN LK-EMP-TYPE = "R"
                   COMPUTE LK-GROSS-PAY = LK-HOURS * LK-RATE
               WHEN LK-EMP-TYPE = "M"
                   COMPUTE LK-GROSS-PAY =
                       WS-REGULAR-HOURS-LIMIT * LK-RATE
               WHEN LK-EMP-TYPE = "P"
                   COMPUTE LK-GROSS-PAY = LK-HOURS * LK-RATE
               WHEN OTHER
                   MOVE 0 TO LK-GROSS-PAY
           END-EVALUATE.
           GOBACK.
```

**โปรแกรมหลักที่ใช้ PERFORM แทน GO TO (แก้ปัญหา Spaghetti Control Flow):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP810AFTER.
       AUTHOR. COBOL-COURSE.

      *> AFTER: structured PERFORM VARYING loop (no GO TO), employee
      *> data still comes from a table, but the pay RULE now lives in
      *> one extracted, reusable, testable subprogram.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EMP-INDEX             PIC 9(1) VALUE 1.
       01  WS-EMP-TABLE.
           05  WS-EMP-ENTRY OCCURS 3 TIMES.
               10  WS-EMP-NAME      PIC X(15).
               10  WS-EMP-TYPE      PIC X(1).
               10  WS-EMP-HOURS     PIC 9(3).
               10  WS-EMP-RATE      PIC 9(3)V99.
       01  WS-GROSS-PAY             PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM SETUP-EMPLOYEE-TABLE.
           PERFORM PROCESS-ONE-EMPLOYEE
               VARYING WS-EMP-INDEX FROM 1 BY 1
               UNTIL WS-EMP-INDEX > 3.
           STOP RUN.

       SETUP-EMPLOYEE-TABLE.
           MOVE "SOMCHAI JAIDEE" TO WS-EMP-NAME(1).
           MOVE "R" TO WS-EMP-TYPE(1).
           MOVE 180 TO WS-EMP-HOURS(1).
           MOVE 50.00 TO WS-EMP-RATE(1).

           MOVE "PRASERT KAEWTA" TO WS-EMP-NAME(2).
           MOVE "M" TO WS-EMP-TYPE(2).
           MOVE 170 TO WS-EMP-HOURS(2).
           MOVE 80.00 TO WS-EMP-RATE(2).

           MOVE "WANIDA SOMBOON" TO WS-EMP-NAME(3).
           MOVE "P" TO WS-EMP-TYPE(3).
           MOVE 90 TO WS-EMP-HOURS(3).
           MOVE 40.00 TO WS-EMP-RATE(3).

       PROCESS-ONE-EMPLOYEE.
           CALL "STEP810GROSSPAY" USING
               WS-EMP-TYPE(WS-EMP-INDEX)
               WS-EMP-HOURS(WS-EMP-INDEX)
               WS-EMP-RATE(WS-EMP-INDEX)
               WS-GROSS-PAY.
           DISPLAY WS-EMP-NAME(WS-EMP-INDEX) " gross=" WS-GROSS-PAY.
```

**คอมไพล์และรันจริง พร้อมพิสูจน์ขั้นสุดท้ายด้วย diff:**

```bash
cobc -x -o step810after step810after.cob step810grosspay.cob
./step810after
diff <(./step810before) <(./step810after) && echo "IDENTICAL OUTPUT - REFACTOR PRESERVED BEHAVIOR"
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน):**

```
SOMCHAI JAIDEE  gross=0009500.00
PRASERT KAEWTA  gross=0012800.00
WANIDA SOMBOON  gross=0003600.00
IDENTICAL OUTPUT - REFACTOR PRESERVED BEHAVIOR
```

### สรุปเทคนิคที่ประกอบเข้าด้วยกันใน Capstone นี้

| ปัญหาเดิมในเวอร์ชัน Before | เทคนิคที่แก้ไข | ผลในเวอร์ชัน After |
|---|---|---|
| `GO TO P2` วนลูปด้วยมือ | PERFORM VARYING (ขั้นตอนที่ 803) | `PERFORM PROCESS-ONE-EMPLOYEE VARYING ... UNTIL ...` |
| Nested IF ซ้อน 3 ชั้น | EVALUATE TRUE (ขั้นตอนที่ 804) | `EVALUATE TRUE` แบบแบนราบ ไม่ซ้อนเลย |
| ตรรกะคำนวณฝังอยู่ใน PROCEDURE DIVISION หลัก | Extract Subprogram (ขั้นตอนที่ 805) | `STEP810GROSSPAY` แยกไฟล์ ทดสอบได้อิสระ |
| ตัวเลข `160`, `1.5` ปรากฏซ้ำหลายจุด | Named Constants (ขั้นตอนที่ 806) | `WS-REGULAR-HOURS-LIMIT`, `WS-OVERTIME-MULTIPLIER` |
| `P1`, `P2`, `P9` ชื่อไม่สื่อความหมาย | Meaningful Naming (ขั้นตอนที่ 802) | `SETUP-EMPLOYEE-TABLE`, `PROCESS-ONE-EMPLOYEE` |

### ข้อควรระวัง

- Capstone นี้แสดงให้เห็นว่า refactoring จริงมักไม่ได้ใช้เทคนิคเดียวโดด ๆ แต่เป็นการ**ผสมผสานหลาย
  เทคนิคเข้าด้วยกัน**อย่างเป็นระบบ — ในทางปฏิบัติควรทำทีละเทคนิคและทดสอบหลังทำแต่ละขั้นตอน (ตาม
  หลักการจากขั้นตอนที่ 809) แทนที่จะเปลี่ยนทุกอย่างพร้อมกันในครั้งเดียวเหมือนที่แสดงในตัวอย่างสรุป
  นี้ (ซึ่งย่อรวมทุกขั้นตอนเพื่อความกระชับของบทเรียนเท่านั้น)
- แม้ผลลัพธ์จะพิสูจน์ว่า `IDENTICAL` แล้ว แต่การ refactor ระดับนี้ (เปลี่ยนทั้งโครงสร้างควบคุม, แยก
  ไฟล์ subprogram, เปลี่ยนชื่อตัวแปรทั้งหมด) ควรผ่าน Code Review อย่างละเอียดจากเพื่อนร่วมทีมเสมอ
  ก่อน merge เข้า `main` ตามหลักการจาก Part 080 — การพิสูจน์ด้วย `diff` เพียงอย่างเดียวไม่ควรเป็น
  เกณฑ์เดียวในการตัดสินใจ merge โค้ดที่เปลี่ยนแปลงมากขนาดนี้

### แบบฝึกหัดที่ 810.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `STEP810GROSSPAY` (subprogram ที่ extract ออกมา) ถูกออกแบบให้เป็น
pure function-like (ไม่มี `DISPLAY` ในตัวมันเอง) ถึงทำให้ Capstone นี้พร้อมสำหรับการเพิ่ม Unit Test
ตามแนวทาง Part 079 ได้ทันทีโดยไม่ต้องแก้ไขโครงสร้างเพิ่มเติมอีก

**เฉลย**: `STEP810GROSSPAY` รับ input ทั้งหมด (`LK-EMP-TYPE`, `LK-HOURS`, `LK-RATE`) ผ่าน `LINKAGE
SECTION` และคืนผลลัพธ์ผ่าน `LK-GROSS-PAY` เพียงอย่างเดียว โดยไม่มีการ `DISPLAY` หรือแตะต้อง
`WS-EMP-TABLE`/`WS-EMP-INDEX` ของโปรแกรมหลักเลยแม้แต่น้อย ทำให้มันเป็นหน่วยที่**แยกอิสระอย่าง
สมบูรณ์**จากโปรแกรมหลัก ตรงตามหลักการ pure function-like design ที่ Part 079 ขั้นตอนที่ 783 สอนไว้
ทุกประการ — สามารถเขียน Test Driver ใหม่ตามรูปแบบเดียวกับ `TESTDRIVER`/`ALLSUITES` ใน Part 079 มา
`CALL "STEP810GROSSPAY" USING ...` ด้วยชุดค่า test case ต่าง ๆ (เช่น ทดสอบพนักงาน Regular ที่ทำงาน
พอดี 160 ชั่วโมง, ทดสอบพนักงาน Manager ที่ทำงานน้อยกว่า 160 ชั่วโมง, ทดสอบประเภทพนักงานที่ไม่รู้จัก)
แล้ว assert ผลลัพธ์ได้ทันที โดยไม่ต้องยุ่งเกี่ยวกับตารางพนักงานหรือลูป `PERFORM VARYING` ของโปรแกรม
หลักเลย นี่คือหลักฐานที่ชัดเจนว่า**การ refactor ที่ดีและการออกแบบเพื่อการทดสอบที่ดี (testability)
เป็นเป้าหมายเดียวกันที่เสริมพลังซึ่งกันและกันเสมอ** ไม่ใช่งานสองงานที่แยกจากกัน

---

## สรุปท้ายบท

Part นี้พาคุณ refactor โค้ด COBOL แบบ legacy ที่มีปัญหาคลาสสิกทุกแบบให้กลายเป็น Clean Code ทีละ
ขั้นตอน โดย**ทุกตัวอย่าง Before/After ถูกคอมไพล์และรันจริงแล้วทั้งคู่ พร้อมพิสูจน์ด้วย `diff`**
(ยกเว้นขั้นตอนที่ 807 ที่จงใจแสดงกรณีที่ผลลัพธ์เปลี่ยนไปโดยไม่ตั้งใจ เพื่อเป็นบทเรียนสำคัญ):

- นิยามที่เข้มงวดของ Refactoring: เปลี่ยนโครงสร้างภายในโดยไม่เปลี่ยนพฤติกรรมภายนอกแม้แต่นิดเดียว
- การตั้งชื่อตัวแปรและ paragraph ให้สื่อความหมาย (Meaningful Naming)
- การกำจัด `GO TO` แบบ spaghetti ด้วย `PERFORM VARYING` ที่มีโครงสร้างชัดเจน
- การลด Nested IF ที่ลึกด้วย `EVALUATE TRUE` ที่ขยายเงื่อนไขใหม่ได้ง่าย
- การ Extract Paragraph เป็น Subprogram เพื่อลดความซ้ำซ้อนและเพิ่ม testability
- การกำจัด Magic Numbers ด้วย 88-level Condition Names และ Named Constants
- บทเรียนสำคัญจากกรณีที่การลดโค้ดซ้ำเปลี่ยนรูปแบบการแสดงผลโดยไม่ตั้งใจ — ทำไม automated test
  ต้องมาก่อนการ refactor เสมอ
- รูปแบบ Driver Paragraph ที่ทำให้ `MAIN-PARA` อ่านเหมือนสารบัญของโปรแกรม
- Test-Driven Refactoring: การใช้ regression test เป็นตาข่ายนิรภัยเพื่อ refactor อย่างปลอดภัย
- Capstone: การผสมผสานทุกเทคนิคเข้าด้วยกันในโปรแกรม Payroll ที่ซับซ้อน พร้อมพิสูจน์ผลลัพธ์เหมือน
  เดิมทุกประการ

Part ถัดไป (**Part 082**) จะต่อยอดจากรากฐานของ Clean Code และ Extract Subprogram ที่เราสร้างไว้
ใน Part นี้ ไปสู่แนวคิดที่เป็นนามธรรมกว่า: **Design Patterns ในโลกของ COBOL** — การประยุกต์ใช้
Strategy Pattern ผ่าน `EVALUATE` และ dispatch table, Factory Pattern ผ่าน subprogram สร้าง record,
Singleton Pattern ผ่านสถานะที่คงอยู่ใน WORKING-STORAGE ข้ามการเรียก, และ Template Method Pattern
ผ่าน driver ที่เรียก hook paragraph — ทุกรูปแบบสร้างขึ้นบนพื้นฐานของ subprogram design ที่สะอาด
ซึ่ง Part นี้เพิ่งสอนไป

**[← กลับไป Part 080: Version Control (Git) และ Workflow สำหรับทีม COBOL](part-080-git-workflow-cobol.md)** | **[ไปยัง Part 082: Design Patterns ในโลก COBOL →](part-082-design-patterns-cobol.md)**
