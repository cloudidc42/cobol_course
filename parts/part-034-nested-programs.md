# Part 034: Nested Programs และ END PROGRAM (ขั้นตอนที่ 331–340)

## คำนำของ Part นี้

Part 033 สอนวิธีแบ่งปัน**โครงสร้างข้อมูล**ระหว่างโปรแกรมหลายไฟล์ผ่าน `COPY` ไปแล้ว ส่วน Part
031-032 สอนวิธีแบ่งปัน**ตรรกะการทำงาน**ระหว่างไฟล์ผ่าน `CALL` Subprogram ที่แยกไฟล์กัน Part นี้
จะแนะนำอีกวิธีหนึ่งในการจัดโครงสร้างโปรแกรมขนาดใหญ่ที่ COBOL รองรับมาตั้งแต่มาตรฐาน COBOL-74:
**Nested Programs** หรือ "โปรแกรมซ้อนโปรแกรม" — การเขียนหลายโปรแกรมไว้ใน**ไฟล์ซอร์สเดียว**
โดยโปรแกรมหนึ่งเป็น "โปรแกรมนอก" (Containing Program) และมีโปรแกรมอื่นซ้อนอยู่ข้างในเป็น
"โปรแกรมใน" (Contained/Nested Program)

Nested Programs มีกฎเฉพาะตัวที่ต่างจาก Subprogram แบบแยกไฟล์อย่างชัดเจน โดยเฉพาะเรื่อง
**ขอบเขตของข้อมูล** (ใครมองเห็นตัวแปรของใคร) และ **ขอบเขตของการเรียกใช้** (ใครเรียกใครได้บ้าง)
เราจะเรียนรู้ตั้งแต่ไวยากรณ์พื้นฐานและกฎการจับคู่ `END PROGRAM`, การแชร์ข้อมูลด้วย `GLOBAL`, การ
เปิดให้โปรแกรมพี่น้องเรียกกันได้ด้วย `COMMON`, การรีเซ็ตค่าทุกครั้งที่ถูกเรียกด้วย `INITIAL`, ไปจนถึง
การเปรียบเทียบว่าเมื่อไรควรใช้ Nested Program เมื่อไรควรแยกเป็น Subprogram คนละไฟล์ ปิดท้ายด้วย
ระบบเมนูเล็ก ๆ ที่ประกอบขึ้นจากโปรแกรมซ้อนกันหลายตัวทำงานร่วมกันจริง

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และทุกตัวอย่าง
> ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 331: แนวคิด Nested Program และไวยากรณ์พื้นฐาน

### Nested Program คืออะไร

Nested Program คือการเขียน `IDENTIFICATION DIVISION. PROGRAM-ID. ...` ของโปรแกรมที่สอง
ต่อท้าย `PROCEDURE DIVISION` ของโปรแกรมแรก **ในไฟล์ `.cob` เดียวกัน** โดยโปรแกรมที่สองนี้จะถูก
"ห่อหุ้ม" อยู่ภายในโปรแกรมแรก และปิดท้ายด้วย `END PROGRAM ชื่อโปรแกรม.` ก่อนที่โปรแกรมแรกจะปิด
ด้วย `END PROGRAM` ของตัวเองเช่นกัน

จุดที่ต่างจาก Subprogram ที่แยกไฟล์ (ที่เรียนใน Part 031) คือ **โปรแกรมที่ถูกซ้อนอยู่ (Nested/
Contained Program) โดยปกติจะถูกเรียกได้เฉพาะจากโปรแกรมที่ห่อหุ้มมันอยู่เท่านั้น** ไม่ใช่จากที่ไหน
ก็ได้ในระบบเหมือน Subprogram ทั่วไป (รายละเอียดกฎขอบเขตนี้จะขยายความในขั้นตอนถัดไป)

### ตัวอย่าง: Nested Program พื้นฐานที่สุด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP331-OUTER.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-OUTER-MSG        PIC X(20) VALUE "OUTER PROGRAM".

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "1) OUTER STARTS: " WS-OUTER-MSG.
           CALL "STEP331-INNER".
           DISPLAY "3) OUTER RESUMES AFTER CALL".
           STOP RUN.

      *> A program nested inside STEP331-OUTER. It can ONLY be
      *> called from code inside STEP331-OUTER, never from outside.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP331-INNER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-INNER-MSG        PIC X(20) VALUE "INNER PROGRAM".

       PROCEDURE DIVISION.
       INNER-MAIN.
           DISPLAY "2) INNER RUNS: " WS-INNER-MSG.
           GOBACK.
       END PROGRAM STEP331-INNER.

       END PROGRAM STEP331-OUTER.
```

**ผลลัพธ์:**

```
1) OUTER STARTS: OUTER PROGRAM       
2) INNER RUNS: INNER PROGRAM       
3) OUTER RESUMES AFTER CALL
```

### อธิบายจุดสำคัญ

- `STEP331-OUTER` และ `STEP331-INNER` อยู่ในไฟล์ `.cob` เดียวกัน คอมไพล์ด้วยคำสั่งปกติ
  `cobc -x -o step331_basic_nesting step331_basic_nesting.cob` เพียงคำสั่งเดียว ไม่ต้อง
  ระบุไฟล์สองไฟล์เหมือนตอน CALL Subprogram แยกไฟล์
- `CALL "STEP331-INNER".` เรียกโปรแกรมในโดยใช้ชื่อ (literal) เหมือนการ `CALL` Subprogram ทั่วไป
  ทุกประการ — วิธีเรียกไม่ต่างกัน สิ่งที่ต่างคือ**ขอบเขต**ว่าใครเรียกได้
- `GOBACK.` ในโปรแกรมใน ทำหน้าที่เหมือน `RETURN` คือส่งการควบคุมกลับไปยังผู้เรียก (คล้ายกับที่ใช้
  ใน Subprogram แยกไฟล์จาก Part 031) โปรแกรมนอกจะทำงานต่อจากบรรทัดหลัง `CALL` ทันที
- `END PROGRAM STEP331-INNER.` ปิดโปรแกรมใน ก่อนที่ `END PROGRAM STEP331-OUTER.` จะปิดโปรแกรม
  นอกในบรรทัดสุดท้ายของไฟล์ — ลำดับการปิดนี้สำคัญมาก (รายละเอียดกฎการจับคู่ END PROGRAM อยู่ใน
  ขั้นตอนที่ 332)

### ข้อควรระวัง: STOP RUN ในโปรแกรมที่ถูกซ้อนอยู่ อันตรายกว่าที่คิด

จุดที่ผู้เริ่มต้นพลาดบ่อยที่สุดคือการใช้ `STOP RUN` แทน `GOBACK` ในโปรแกรมที่ถูกเรียก ลองดูโค้ด
ต่อไปนี้ที่เหมือนตัวอย่างข้างบนทุกอย่าง ยกเว้นเปลี่ยน `GOBACK` เป็น `STOP RUN` ในโปรแกรมใน:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP331-OUTER-BAD.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "1) OUTER STARTS".
           CALL "STEP331-INNER-BAD".
           DISPLAY "3) THIS LINE NEVER RUNS".
           STOP RUN.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP331-INNER-BAD.

       PROCEDURE DIVISION.
       INNER-MAIN.
           DISPLAY "2) INNER RUNS, THEN STOP RUN BY MISTAKE".
           STOP RUN.
       END PROGRAM STEP331-INNER-BAD.

       END PROGRAM STEP331-OUTER-BAD.
```

**ผลลัพธ์ (สังเกตว่าบรรทัดที่ 3 หายไป):**

```
1) OUTER STARTS
2) INNER RUNS, THEN STOP RUN BY MISTAKE
```

`STOP RUN` มีความหมายว่า **"จบการทำงานของทั้งโปรแกรมทั้งหมด (Run Unit) ทันที"** ไม่ว่าจะถูกเขียน
ไว้ในโปรแกรมนอกหรือโปรแกรมในก็ตาม เมื่อ `STEP331-INNER-BAD` สั่ง `STOP RUN` ทั้งโปรแกรมทั้งชุด
(รวมถึง `STEP331-OUTER-BAD` ที่กำลังรออยู่) จะจบทันที บรรทัด `DISPLAY "3) THIS LINE NEVER
RUNS".` ในโปรแกรมนอกจึงไม่มีทางถูกรันเลย

**กฎจำง่าย ๆ**: โปรแกรมที่ถูก `CALL` (ไม่ว่าจะเป็น Nested Program หรือ Subprogram แยกไฟล์)
ควรจบด้วย `GOBACK` เสมอ ส่วน `STOP RUN` ควรใช้เฉพาะในโปรแกรมหลักที่สุด (Main Program) ที่เป็น
จุดเริ่มต้นการทำงานของทั้งระบบเท่านั้น

### แบบฝึกหัดที่ 331.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `GOBACK` กับ `STOP RUN` เมื่อใช้ในโปรแกรมที่ถูกซ้อนอยู่
(Nested Program)

**เฉลย**: `GOBACK` ส่งการควบคุมกลับไปยังจุดที่เรียกโปรแกรมนี้เท่านั้น (คล้ายการ `return` จาก
ฟังก์ชันในภาษาอื่น) โปรแกรมที่เรียกจะทำงานต่อจากบรรทัดถัดจาก `CALL` ทันที ส่วน `STOP RUN` จะ
สั่งจบการทำงานของ**ทั้ง Run Unit** คือทั้งโปรแกรมนอกและโปรแกรมในทั้งหมดในคราวเดียว ไม่ว่า `STOP
RUN` จะถูกเรียกจากจุดไหนในลำดับการซ้อนก็ตาม ดังนั้นโปรแกรมที่ถูกเรียกควรใช้ `GOBACK` เพื่อให้ผู้เรียก
ทำงานต่อได้ตามปกติ

---

## ขั้นตอนที่ 332: กฎการจับคู่ END PROGRAM และโปรแกรมพี่น้องหลายตัว

### กฎของ END PROGRAM

ทุกโปรแกรมที่เขียนแบบ Nested (ยกเว้นโปรแกรมสุดท้ายในไฟล์ ซึ่งอนุญาตให้ละเว้น `END PROGRAM`
ได้ แต่หลักสูตรนี้แนะนำให้ใส่เสมอเพื่อความชัดเจน) ต้องปิดท้ายด้วย `END PROGRAM ชื่อโปรแกรม.` โดย
**ชื่อที่ระบุต้องตรงกับ `PROGRAM-ID` ของโปรแกรมนั้นเป๊ะ** และโปรแกรมที่ซ้อนอยู่ชั้นในสุดต้องถูกปิด
ก่อนโปรแกรมชั้นนอกเสมอ (ปิดจากในสุดออกไปนอกสุด เหมือนวงเล็บที่ต้องปิดให้ครบตามลำดับ)

### ตัวอย่าง: หลายโปรแกรมพี่น้อง (Sibling) ซ้อนอยู่ในโปรแกรมนอกเดียวกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP332-OUTER.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "OUTER CALLS FIRST-CHILD".
           CALL "STEP332-FIRST-CHILD".
           DISPLAY "OUTER CALLS SECOND-CHILD".
           CALL "STEP332-SECOND-CHILD".
           STOP RUN.

      *> Two sibling programs nested one after another, each
      *> closed off with its own matching END PROGRAM.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP332-FIRST-CHILD.
       PROCEDURE DIVISION.
       FC-MAIN.
           DISPLAY "  FIRST-CHILD RUNNING".
           GOBACK.
       END PROGRAM STEP332-FIRST-CHILD.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP332-SECOND-CHILD.
       PROCEDURE DIVISION.
       SC-MAIN.
           DISPLAY "  SECOND-CHILD RUNNING".
           GOBACK.
       END PROGRAM STEP332-SECOND-CHILD.

       END PROGRAM STEP332-OUTER.
```

**ผลลัพธ์:**

```
OUTER CALLS FIRST-CHILD
  FIRST-CHILD RUNNING
OUTER CALLS SECOND-CHILD
  SECOND-CHILD RUNNING
```

### อธิบายจุดสำคัญ

- `STEP332-FIRST-CHILD` และ `STEP332-SECOND-CHILD` ทั้งคู่ซ้อนอยู่ภายใน `STEP332-OUTER` ใน
  ระดับเดียวกัน เรียกว่าเป็น **โปรแกรมพี่น้อง (Sibling Programs)** — ทั้งคู่มองไม่เห็นกันโดยตรง
  (ยกเว้นจะประกาศ `COMMON` ซึ่งจะเรียนในขั้นตอนที่ 335) แต่ทั้งคู่ถูกเรียกได้จาก `STEP332-OUTER`
  ซึ่งเป็นผู้ห่อหุ้มร่วมกัน
- สังเกตลำดับ `END PROGRAM`: `STEP332-FIRST-CHILD` ปิดตัวเองก่อน จากนั้น `STEP332-SECOND-CHILD`
  จึงเริ่มและปิดตัวเอง สุดท้าย `STEP332-OUTER` จึงปิดเป็นลำดับสุดท้ายสุด — เหมือนวงเล็บที่ซ้อนกัน
  ต้องปิดให้ครบและถูกลำดับ

### ข้อควรระวัง: ชื่อ END PROGRAM ต้องตรงกับ PROGRAM-ID เป๊ะ (พิสูจน์ด้วย Error จริง)

ลองดูโค้ดที่พิมพ์ชื่อใน `END PROGRAM` ผิดจาก `PROGRAM-ID`:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP332-OUTER2.
       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "STEP332-INNER2".
           STOP RUN.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP332-INNER2.
       PROCEDURE DIVISION.
       INNER-MAIN.
           DISPLAY "INNER2".
           GOBACK.
       END PROGRAM STEP332-WRONG-NAME.
       END PROGRAM STEP332-OUTER2.
```

**ผลลัพธ์การคอมไพล์ (Error จริงจาก GnuCOBOL):**

```
step332_mismatch.cob: in paragraph 'INNER-MAIN':
step332_mismatch.cob:14: error: END PROGRAM 'STEP332-WRONG-NAME' is different from PROGRAM-ID 'STEP332-INNER2'
```

คอมไพเลอร์ตรวจจับความไม่ตรงกันนี้ได้ทันที และปฏิเสธที่จะคอมไพล์ต่อ นี่เป็นเรื่องดีเพราะทำให้เรารู้
ปัญหาตั้งแต่ตอนคอมไพล์ ไม่ใช่ไปเจอพฤติกรรมประหลาดตอนรันโปรแกรมจริง

### แบบฝึกหัดที่ 332.1

**โจทย์**: จงเพิ่มโปรแกรมพี่น้องตัวที่สามชื่อ `STEP332-THIRD-CHILD` เข้าไปในตัวอย่างข้างบน โดยให้
`STEP332-OUTER` เรียกมันต่อจาก `STEP332-SECOND-CHILD`

**เฉลย**: เพิ่มโค้ดต่อไปนี้ก่อนบรรทัด `END PROGRAM STEP332-OUTER.`:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP332-THIRD-CHILD.
       PROCEDURE DIVISION.
       TC-MAIN.
           DISPLAY "  THIRD-CHILD RUNNING".
           GOBACK.
       END PROGRAM STEP332-THIRD-CHILD.
```

และเพิ่มบรรทัดนี้ใน `MAIN-PARA` ของ `STEP332-OUTER` ก่อน `STOP RUN.`:

```cobol
           DISPLAY "OUTER CALLS THIRD-CHILD".
           CALL "STEP332-THIRD-CHILD".
```

---

## ขั้นตอนที่ 333: ขอบเขตข้อมูล (Data Scope) — ค่าเริ่มต้นคือ "มองไม่เห็นกัน"

### กฎสำคัญที่สุดของ Nested Programs

หลายคนเข้าใจผิดว่าเมื่อโปรแกรมซ้อนกันแล้ว โปรแกรมในจะมองเห็นตัวแปรใน `WORKING-STORAGE
SECTION` ของโปรแกรมนอกได้อัตโนมัติ **ความเข้าใจนี้ผิด** — ตามค่าเริ่มต้น โปรแกรมแต่ละตัว (ไม่ว่า
จะซ้อนกันหรือไม่) มี `WORKING-STORAGE SECTION` เป็นของตัวเองโดยสมบูรณ์ ไม่แชร์กัน

### ตัวอย่าง: พิสูจน์ว่าข้อมูลไม่ถูกแชร์กันโดยปริยาย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP333-OUTER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-OUTER-COUNT      PIC 9(4) VALUE 100.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "OUTER BEFORE CALL: " WS-OUTER-COUNT.
           CALL "STEP333-INNER".
           DISPLAY "OUTER AFTER CALL:  " WS-OUTER-COUNT.
           STOP RUN.

      *> By default, WS-OUTER-COUNT above is NOT visible in here.
      *> This program has its own, completely separate WS-COUNT.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP333-INNER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNT            PIC 9(4) VALUE 999.

       PROCEDURE DIVISION.
       INNER-MAIN.
           ADD 1 TO WS-COUNT.
           DISPLAY "INNER OWN WS-COUNT: " WS-COUNT.
           GOBACK.
       END PROGRAM STEP333-INNER.

       END PROGRAM STEP333-OUTER.
```

**ผลลัพธ์:**

```
OUTER BEFORE CALL: 0100
INNER OWN WS-COUNT: 1000
OUTER AFTER CALL:  0100
```

### อธิบายจุดสำคัญ

- `WS-OUTER-COUNT` ในโปรแกรมนอกเริ่มต้นที่ `100` และ**ยังคงเป็น 100** หลังเรียกโปรแกรมใน — เพราะ
  `STEP333-INNER` ไม่เคยแตะต้องมันเลย มันไม่รู้จักด้วยซ้ำว่ามีตัวแปรชื่อนี้อยู่
- `STEP333-INNER` มีตัวแปรของตัวเองชื่อ `WS-COUNT` เริ่มต้นที่ `999` เมื่อ `ADD 1` จะได้ `1000`
  นี่เป็นตัวแปรคนละตัวกับ `WS-OUTER-COUNT` ในโปรแกรมนอกโดยสิ้นเชิง แม้จะซ้อนอยู่ในกันก็ตาม
- พฤติกรรมนี้เหมือนกับที่เราเห็นใน ขั้นตอนที่ 333 ของ Subprogram แยกไฟล์ (Part 031-032): แต่ละ
  โปรแกรมมีพื้นที่หน่วยความจำของตัวเอง การส่งข้อมูลระหว่างกันต้องทำผ่าน `CALL ... USING`
  (Parameter Passing ที่เรียนมาแล้ว) หรือผ่าน `GLOBAL` (ที่จะเรียนในขั้นตอนถัดไป)

### ข้อควรระวัง

- อย่าคาดหวังว่าโปรแกรมในจะ "เห็น" ตัวแปรของโปรแกรมนอกโดยอัตโนมัติ นี่เป็นความเข้าใจผิดที่พบบ่อย
  ที่สุดสำหรับผู้เริ่มต้นเรียน Nested Programs
- หากต้องการส่งข้อมูลระหว่างโปรแกรมนอกกับโปรแกรมใน มีสองทางเลือกหลัก: (1) ส่งผ่าน `CALL ...
  USING` เหมือน Subprogram ทั่วไป หรือ (2) ประกาศตัวแปรที่ต้องการแชร์ด้วย `IS GLOBAL` ใน
  โปรแกรมนอก (เรียนในขั้นตอนที่ 334)

### แบบฝึกหัดที่ 333.1

**โจทย์**: จงอธิบายว่าทำไม `WS-OUTER-COUNT` และ `WS-COUNT` ในตัวอย่างข้างบนจึงเป็นคนละตัวแปร
กันโดยสิ้นเชิง แม้ `STEP333-INNER` จะถูกซ้อนอยู่ภายใน `STEP333-OUTER`

**เฉลย**: เพราะ `WORKING-STORAGE SECTION` ของแต่ละโปรแกรมใน COBOL (ไม่ว่าจะเป็นโปรแกรมระดับ
บนสุดหรือโปรแกรมที่ถูกซ้อนอยู่) เป็นพื้นที่หน่วยความจำที่แยกจากกันโดยสมบูรณ์ตามค่าเริ่มต้น การซ้อน
โปรแกรม (Nesting) มีผลแค่กับ**ขอบเขตของชื่อโปรแกรม** (ว่าใครเรียกใครได้บ้าง — จะอธิบายเพิ่มเติมใน
ขั้นตอนที่ 337) ไม่ได้มีผลกับขอบเขตของ**ข้อมูล**โดยอัตโนมัติ การจะให้ข้อมูลถูกแชร์กันได้ต้องประกาศ
ไว้อย่างชัดเจนด้วยคีย์เวิร์ด `GLOBAL` เท่านั้น มิฉะนั้นแต่ละโปรแกรมจะทำงานราวกับเป็นเอกเทศต่อกัน
ในเรื่องข้อมูล

---

## ขั้นตอนที่ 334: GLOBAL Clause — การแชร์ข้อมูลระหว่างโปรแกรมซ้อนกันอย่างตั้งใจ

### GLOBAL คืออะไร

เมื่อต้องการให้ตัวแปรใน `WORKING-STORAGE SECTION` ของโปรแกรมนอก ถูกมองเห็นได้จากโปรแกรมที่
ซ้อนอยู่ภายใน (ทุกระดับ ไม่ใช่แค่ชั้นแรก) เราประกาศตัวแปรนั้นด้วยวลี **`IS GLOBAL`** ต่อท้าย
Data Description Entry ตัวแปรที่ประกาศแบบนี้จะถูกมองเห็นได้จากทุกโปรแกรมที่ซ้อนอยู่ภายในโปรแกรม
ที่ประกาศมัน โดยไม่ต้องส่งผ่านพารามิเตอร์เลย

### ตัวอย่าง: แชร์ตัวนับรวมด้วย GLOBAL

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP334-OUTER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LOCAL-VAR        PIC 9(4) VALUE 100.
      *> IS GLOBAL makes this item visible from EVERY program
      *> nested inside STEP334-OUTER, at any depth.
       01  WS-SHARED-TOTAL     PIC 9(6) VALUE 500 IS GLOBAL.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "OUTER LOCAL-VAR:    " WS-LOCAL-VAR.
           DISPLAY "OUTER SHARED-TOTAL: " WS-SHARED-TOTAL.
           CALL "STEP334-INNER".
           DISPLAY "AFTER CALL, SHARED-TOTAL: " WS-SHARED-TOTAL.
           STOP RUN.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP334-INNER.

       PROCEDURE DIVISION.
       INNER-MAIN.
      *> WS-SHARED-TOTAL is visible here because it was declared
      *> GLOBAL in the containing program.
           ADD 1 TO WS-SHARED-TOTAL.
           DISPLAY "INNER SEES SHARED-TOTAL: " WS-SHARED-TOTAL.
           GOBACK.
       END PROGRAM STEP334-INNER.

       END PROGRAM STEP334-OUTER.
```

**ผลลัพธ์:**

```
OUTER LOCAL-VAR:    0100
OUTER SHARED-TOTAL: 000500
INNER SEES SHARED-TOTAL: 000501
AFTER CALL, SHARED-TOTAL: 000501
```

### อธิบายจุดสำคัญ

- `01 WS-SHARED-TOTAL PIC 9(6) VALUE 500 IS GLOBAL.` — สังเกตว่า `STEP334-INNER` **ไม่ต้อง
  ประกาศ** `WS-SHARED-TOTAL` ซ้ำเลย มันไม่มี `WORKING-STORAGE SECTION` ของตัวเองด้วยซ้ำใน
  ตัวอย่างนี้ แต่ยังคงเข้าถึงและแก้ไข `WS-SHARED-TOTAL` ได้โดยตรง
- `WS-LOCAL-VAR` (ไม่มี `IS GLOBAL`) ยังคงเป็นของ `STEP334-OUTER` เท่านั้น ตามกฎเดิมจาก
  ขั้นตอนที่ 333 — `GLOBAL` ต้องระบุไว้อย่างชัดเจนเป็นรายตัวแปร ไม่ใช่ทุกตัวแปรในโปรแกรมนอกจะ
  กลายเป็น Global โดยอัตโนมัติ
- ค่า `WS-SHARED-TOTAL` หลังจากโปรแกรมในแก้ไข (`501`) ยังคงอยู่เมื่อกลับมาที่โปรแกรมนอก เพราะ
  มันคือหน่วยความจำก้อนเดียวกันจริง ๆ ไม่ใช่การคัดลอกค่าไปมาเหมือนพารามิเตอร์แบบ `BY CONTENT`

### ข้อควรระวัง

- `GLOBAL` ทำงานได้เฉพาะ **จากโปรแกรมนอกไปยังโปรแกรมในเท่านั้น** (Top-down) ตัวแปรที่ประกาศ
  `GLOBAL` ในโปรแกรมในจะไม่ถูกมองเห็นจากโปรแกรมนอก — ทิศทางของ `GLOBAL` มีทางเดียวเสมอ
- การใช้ `GLOBAL` มากเกินไปทำให้โปรแกรมยากต่อการอ่านและดูแลรักษา เพราะข้อมูลถูกแก้ไขจากหลายจุด
  โดยไม่ผ่านพารามิเตอร์ที่ชัดเจน ควรใช้เฉพาะกรณีที่จำเป็นจริง ๆ เช่น ตัวนับรวมหรือค่าที่ต้องใช้ร่วมกัน
  อย่างแพร่หลายในระบบย่อยเดียวกัน

### แบบฝึกหัดที่ 334.1

**โจทย์**: จงเพิ่มโปรแกรมในตัวที่สองชื่อ `STEP334-INNER2` (เป็นพี่น้องกับ `STEP334-INNER`) ที่
`DISPLAY` ค่า `WS-SHARED-TOTAL` เพื่อพิสูจน์ว่าโปรแกรมพี่น้องก็มองเห็นตัวแปร Global ตัวเดียวกันได้

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP334-INNER2.
       PROCEDURE DIVISION.
       INNER2-MAIN.
           DISPLAY "INNER2 ALSO SEES: " WS-SHARED-TOTAL.
           GOBACK.
       END PROGRAM STEP334-INNER2.
```

วางไว้ก่อน `END PROGRAM STEP334-OUTER.` แล้วเพิ่ม `CALL "STEP334-INNER2".` ใน `MAIN-PARA`
ของโปรแกรมนอก ผลลัพธ์จะแสดงค่า `WS-SHARED-TOTAL` เดียวกันที่ `STEP334-INNER` เพิ่งแก้ไขไป

---

## ขั้นตอนที่ 335: COMMON Clause — เปิดให้โปรแกรมพี่น้องเรียกกันได้

### ปัญหา: โปรแกรมพี่น้องปกติเรียกกันไม่ได้

จากที่เรียนใน ขั้นตอนที่ 332 โปรแกรมพี่น้อง (Sibling) ที่ซ้อนอยู่ในโปรแกรมนอกเดียวกัน โดยปกติจะ
เรียกได้เฉพาะจากโปรแกรมนอกเท่านั้น พี่น้องด้วยกันเรียกกันเองไม่ได้ แต่ในทางปฏิบัติ เรามักต้องการ
เขียน Utility Program เล็ก ๆ (เช่น โปรแกรมตรวจสอบข้อมูล) ที่ทั้งโปรแกรมนอกและโปรแกรมพี่น้องอื่น ๆ
ควรเรียกใช้ร่วมกันได้ นี่คือหน้าที่ของคีย์เวิร์ด **`COMMON`**

### ตัวอย่าง: HELPER-A ประกาศเป็น COMMON เพื่อให้พี่น้องเรียกได้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP335-OUTER.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "OUTER CALLS HELPER-A".
           CALL "STEP335-HELPER-A".
           DISPLAY "OUTER CALLS HELPER-B".
           CALL "STEP335-HELPER-B".
           STOP RUN.

      *> COMMON makes this program callable not only from the
      *> container, but also from ANY of its sibling programs.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP335-HELPER-A COMMON.

       PROCEDURE DIVISION.
       HA-MAIN.
           DISPLAY "  HELPER-A RUNNING".
           GOBACK.
       END PROGRAM STEP335-HELPER-A.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP335-HELPER-B.

       PROCEDURE DIVISION.
       HB-MAIN.
           DISPLAY "  HELPER-B RUNNING, NOW CALLS HELPER-A TOO".
           CALL "STEP335-HELPER-A".
           GOBACK.
       END PROGRAM STEP335-HELPER-B.

       END PROGRAM STEP335-OUTER.
```

**ผลลัพธ์:**

```
OUTER CALLS HELPER-A
  HELPER-A RUNNING
OUTER CALLS HELPER-B
  HELPER-B RUNNING, NOW CALLS HELPER-A TOO
  HELPER-A RUNNING
```

### อธิบายจุดสำคัญ

- `PROGRAM-ID. STEP335-HELPER-A COMMON.` — คำว่า `COMMON` ต่อท้ายชื่อโปรแกรม บอกคอมไพเลอร์
  ว่าโปรแกรมนี้ต้องเปิดให้เรียกได้จาก**ทุกโปรแกรมที่อยู่ในระดับเดียวกัน** (Sibling) ภายใต้ผู้ห่อหุ้ม
  เดียวกัน ไม่ใช่แค่จากผู้ห่อหุ้มเท่านั้น
- `STEP335-HELPER-B` (ซึ่งไม่ได้ประกาศ `COMMON`) สามารถ `CALL "STEP335-HELPER-A"` ได้สำเร็จ
  เพราะ `HELPER-A` เป็นฝ่ายเปิดตัวเองให้พี่น้องเรียกได้ ไม่ใช่เพราะ `HELPER-B` มีสิทธิพิเศษอะไร
- ถ้า `STEP335-HELPER-B` พยายาม `CALL "STEP335-HELPER-A"` โดยที่ `HELPER-A` **ไม่ได้** ประกาศ
  `COMMON` การ `CALL` นั้นจะล้มเหลวตอนรัน (module not found) เหมือนกับกรณีการเรียกข้ามขอบเขต
  ที่ไม่ได้รับอนุญาตอื่น ๆ ที่เราจะเห็นใน ขั้นตอนที่ 337

### ข้อควรระวัง

- `COMMON` เหมาะสำหรับโปรแกรม Utility ขนาดเล็กที่หลายโปรแกรมพี่น้องต้องใช้ร่วมกัน (เช่น ฟังก์ชัน
  ตรวจสอบรูปแบบข้อมูล, ฟังก์ชันแปลงหน่วย) แต่ไม่ควรใช้พร่ำเพรื่อ เพราะยิ่งมีโปรแกรม `COMMON`
  มากเท่าไร โครงสร้างการพึ่งพากันของระบบก็ยิ่งซับซ้อนขึ้นเท่านั้น
- โปรแกรม `COMMON` เอง **ไม่สามารถ** เรียกโปรแกรมพี่น้องอื่นที่ไม่ใช่ `COMMON` ได้ (กฎขอบเขต
  พื้นฐานยังคงใช้อยู่) `COMMON` เปิดทางให้ถูกเรียก ไม่ได้เปิดทางให้เรียกออกไปหาใครก็ได้

### แบบฝึกหัดที่ 335.1

**โจทย์**: จงอธิบายว่าทำไมในตัวอย่างข้างบน `STEP335-OUTER` (โปรแกรมนอกสุด) จึงสามารถเรียก
`STEP335-HELPER-A` ได้อยู่แล้ว แม้จะไม่มี `COMMON`

**เฉลย**: เพราะกฎพื้นฐานของ Nested Programs (จาก ขั้นตอนที่ 331) คือ**โปรแกรมที่ห่อหุ้ม (Container)
สามารถเรียกโปรแกรมที่ซ้อนอยู่ในตัวเองได้อยู่แล้วเสมอ** โดยไม่ต้องมี `COMMON` — `COMMON` มีไว้
เพื่อขยายสิทธิ์การเรียกให้ครอบคลุมถึง**โปรแกรมพี่น้อง** (Sibling) ด้วยเท่านั้น ในตัวอย่างนี้
`STEP335-OUTER` คือผู้ห่อหุ้มของทั้ง `HELPER-A` และ `HELPER-B` จึงเรียกทั้งสองได้อยู่แล้วโดยไม่ต้อง
พึ่ง `COMMON` เลย ส่วน `COMMON` ที่ใส่ไว้ที่ `HELPER-A` มีผลเฉพาะกับการที่ `HELPER-B` (ซึ่งเป็น
พี่น้อง ไม่ใช่ผู้ห่อหุ้ม) จะเรียก `HELPER-A` ได้หรือไม่เท่านั้น

---

## ขั้นตอนที่ 336: INITIAL Clause — รีเซ็ตข้อมูลทุกครั้งที่ถูกเรียก

### ปัญหา: ค่าตัวแปรที่ "จำ" ค่าเดิมข้ามการเรียก

ตามค่าเริ่มต้น เมื่อโปรแกรม (ไม่ว่าจะเป็น Nested Program หรือ Subprogram ทั่วไป) ถูกเรียกซ้ำหลาย
ครั้งในรอบการทำงานเดียวกัน (Run Unit เดียวกัน) ค่าตัวแปรใน `WORKING-STORAGE SECTION` ของมัน
จะ**คงค่าจากการเรียกครั้งก่อนหน้าไว้** ไม่ถูกรีเซ็ตกลับไปเป็นค่า `VALUE` ที่ประกาศไว้ พฤติกรรมนี้
บางครั้งเป็นสิ่งที่ต้องการ (เช่น ตัวนับสะสม) แต่บางครั้งก็ไม่ใช่ (เช่น โปรแกรมที่ควรเริ่มต้นใหม่ทุกครั้ง
ที่ถูกเรียก) COBOL มีวลี **`IS INITIAL PROGRAM`** สำหรับกรณีหลัง

### ตัวอย่าง: เปรียบเทียบโปรแกรมปกติกับโปรแกรม INITIAL

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP336-OUTER.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "-- NORMAL PROGRAM, CALLED TWICE --".
           CALL "STEP336-NORMAL-COUNTER".
           CALL "STEP336-NORMAL-COUNTER".
           DISPLAY "-- INITIAL PROGRAM, CALLED TWICE --".
           CALL "STEP336-INITIAL-COUNTER".
           CALL "STEP336-INITIAL-COUNTER".
           STOP RUN.

      *> A normal nested program KEEPS its data between calls.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP336-NORMAL-COUNTER.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNT            PIC 9(4) VALUE 0.
       PROCEDURE DIVISION.
       NC-MAIN.
           ADD 1 TO WS-COUNT.
           DISPLAY "  NORMAL COUNT=" WS-COUNT.
           GOBACK.
       END PROGRAM STEP336-NORMAL-COUNTER.

      *> IS INITIAL PROGRAM resets ALL its data to VALUE clauses
      *> every single time it is called.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP336-INITIAL-COUNTER IS INITIAL PROGRAM.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNT            PIC 9(4) VALUE 0.
       PROCEDURE DIVISION.
       IC-MAIN.
           ADD 1 TO WS-COUNT.
           DISPLAY "  INITIAL COUNT=" WS-COUNT.
           GOBACK.
       END PROGRAM STEP336-INITIAL-COUNTER.

       END PROGRAM STEP336-OUTER.
```

**ผลลัพธ์:**

```
-- NORMAL PROGRAM, CALLED TWICE --
  NORMAL COUNT=0001
  NORMAL COUNT=0002
-- INITIAL PROGRAM, CALLED TWICE --
  INITIAL COUNT=0001
  INITIAL COUNT=0001
```

### อธิบายจุดสำคัญ

- `STEP336-NORMAL-COUNTER` ถูกเรียกสองครั้ง ค่า `WS-COUNT` สะสมจาก `1` เป็น `2` ตามที่คาดหวัง
  จากพฤติกรรมปกติของตัวแปรใน `WORKING-STORAGE SECTION`
- `STEP336-INITIAL-COUNTER` (ที่มี `IS INITIAL PROGRAM` ต่อท้าย `PROGRAM-ID`) แม้จะถูกเรียกสอง
  ครั้งเช่นกัน แต่ค่า `WS-COUNT` กลับเป็น `1` ทั้งสองครั้ง เพราะ COBOL รีเซ็ตข้อมูลทั้งหมดของโปรแกรม
  นี้กลับไปเป็นค่า `VALUE` ที่ประกาศไว้ (คือ `0`) ทุกครั้งที่ถูกเรียกใหม่ ก่อนที่ `PROCEDURE DIVISION`
  จะเริ่มทำงาน

### ข้อควรระวัง

- `IS INITIAL PROGRAM` มีประโยชน์มากสำหรับโปรแกรม Utility ที่ต้องการให้แน่ใจว่าจะไม่มีข้อมูลตกค้าง
  จากการเรียกครั้งก่อน (เช่น โปรแกรมตรวจสอบข้อมูลที่ต้องเริ่มสถานะใหม่ทุกครั้ง) แต่ถ้าโปรแกรมนั้นต้อง
  พึ่งพาการ "จำค่า" ระหว่างการเรียกครั้งก่อนหน้า (เช่น ตัวนับสะสม) ห้ามใส่ `IS INITIAL PROGRAM`
  เด็ดขาด มิฉะนั้นตัวนับจะไม่มีวันสะสมค่าได้เลย
- การรีเซ็ตนี้มีค่าใช้จ่ายด้านประสิทธิภาพเล็กน้อย (ต้องเขียนค่าเริ่มต้นทับข้อมูลเดิมทุกครั้ง) ในระบบที่
  เรียกโปรแกรมนี้บ่อยมาก ๆ ควรพิจารณาผลกระทบด้านประสิทธิภาพประกอบด้วย

### แบบฝึกหัดที่ 336.1

**โจทย์**: จงอธิบายสถานการณ์ในโลกจริงที่ `IS INITIAL PROGRAM` เหมาะสมกว่าโปรแกรมปกติ

**เฉลยแนวทาง**: ตัวอย่างที่ดี เช่น โปรแกรมตรวจสอบความถูกต้องของข้อมูลนำเข้า (Input Validation
Utility) ที่ใช้ตัวแปร Flag หลายตัวระหว่างการตรวจสอบ หากไม่รีเซ็ตค่า Flag เหล่านี้ทุกครั้งที่ถูกเรียก
อาจเกิดกรณีที่ Flag จากการตรวจสอบข้อมูลชุดก่อนหน้า "รั่วไหล" มาปะปนกับผลการตรวจสอบข้อมูลชุดใหม่
ทำให้ผลลัพธ์การตรวจสอบผิดพลาดโดยที่ผู้เขียนโปรแกรมมองไม่เห็นสาเหตุชัดเจน การใช้ `IS INITIAL
PROGRAM` รับประกันว่าทุกการเรียกจะเริ่มต้นด้วยสถานะที่สะอาดเสมอ

---

## ขั้นตอนที่ 337: การซ้อนหลายชั้น (Multi-level Nesting) และกฎขอบเขตการเรียก

### Nested Programs ซ้อนได้มากกว่า 2 ชั้น

COBOL ไม่ได้จำกัดว่า Nested Program จะซ้อนกันได้แค่ 2 ชั้น เราสามารถมีโปรแกรม A ห่อหุ้ม B และ
B ห่อหุ้ม C อีกทีได้ (ซ้อนกันกี่ชั้นก็ได้ตามต้องการ) โดยกฎขอบเขตพื้นฐานยังคงเดิม: **แต่ละโปรแกรม
เรียกได้เฉพาะโปรแกรมที่ตัวเองห่อหุ้มโดยตรงเท่านั้น** (บวกกับโปรแกรมพี่น้องที่ประกาศ `COMMON`)

### ตัวอย่าง: ซ้อน 3 ชั้น

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-LEVEL-A.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "LEVEL-A START".
           CALL "STEP337-LEVEL-B".
           DISPLAY "LEVEL-A END".
           STOP RUN.

      *> LEVEL-B is nested inside LEVEL-A.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-LEVEL-B.

       PROCEDURE DIVISION.
       LEVEL-B-MAIN.
           DISPLAY "  LEVEL-B START".
           CALL "STEP337-LEVEL-C".
           DISPLAY "  LEVEL-B END".
           GOBACK.

      *> LEVEL-C is nested inside LEVEL-B (3 levels deep in total).
      *> Only LEVEL-B may CALL "STEP337-LEVEL-C" directly.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-LEVEL-C.

       PROCEDURE DIVISION.
       LEVEL-C-MAIN.
           DISPLAY "    LEVEL-C RUNNING (innermost)".
           GOBACK.
       END PROGRAM STEP337-LEVEL-C.

       END PROGRAM STEP337-LEVEL-B.
       END PROGRAM STEP337-LEVEL-A.
```

**ผลลัพธ์:**

```
LEVEL-A START
  LEVEL-B START
    LEVEL-C RUNNING (innermost)
  LEVEL-B END
LEVEL-A END
```

### อธิบายจุดสำคัญ

- `STEP337-LEVEL-C` ถูกซ้อนอยู่**ภายใน** `STEP337-LEVEL-B` (ไม่ใช่ภายใน `STEP337-LEVEL-A`
  โดยตรง) สังเกตจากตำแหน่งของ `END PROGRAM STEP337-LEVEL-C.` ที่อยู่ก่อน `END PROGRAM
  STEP337-LEVEL-B.` — นี่คือสิ่งที่ทำให้ `LEVEL-C` เป็น "หลาน" ของ `LEVEL-A` ในเชิงโครงสร้าง
- มีเพียง `STEP337-LEVEL-B` เท่านั้นที่ `CALL "STEP337-LEVEL-C"` ได้โดยตรง เพราะมันเป็นผู้ห่อหุ้ม
  ของ `LEVEL-C` โดยตรง

### ข้อควรระวัง: "ปู่" เรียก "หลาน" โดยตรงไม่ได้ (พิสูจน์ด้วย Error จริง)

ข้อผิดพลาดที่พบบ่อยคือคิดว่าโปรแกรมนอกสุด (ปู่) สามารถเรียกโปรแกรมที่ซ้อนอยู่ลึกที่สุด (หลาน) ได้
โดยตรง ทั้งที่ความจริงแล้วทำไม่ได้ ลองดูตัวอย่าง:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-OUTER-SV.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "TRYING TO CALL GRANDCHILD DIRECTLY".
           CALL "STEP337-GRANDCHILD-SV".
           STOP RUN.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-CHILD-SV.
       PROCEDURE DIVISION.
       CHILD-MAIN.
           DISPLAY "CHILD RUNNING".
           GOBACK.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-GRANDCHILD-SV.
       PROCEDURE DIVISION.
       GC-MAIN.
           DISPLAY "GRANDCHILD RUNNING".
           GOBACK.
       END PROGRAM STEP337-GRANDCHILD-SV.
       END PROGRAM STEP337-CHILD-SV.
       END PROGRAM STEP337-OUTER-SV.
```

โปรแกรมนี้ **คอมไพล์ผ่านโดยไม่มี Error ใด ๆ เลย** (คอมไพเลอร์ไม่ได้ตรวจสอบขอบเขตการเรียกแบบนี้
ตอนคอมไพล์) แต่เมื่อรันจริงจะพัง:

```
TRYING TO CALL GRANDCHILD DIRECTLY
libcob: error: module 'STEP337-GRANDCHILD-SV' not found
```

### อธิบายจุดสำคัญ (ต่อ)

- `STEP337-GRANDCHILD-SV` ถูกซ้อนอยู่ภายใน `STEP337-CHILD-SV` ไม่ใช่ภายใน `STEP337-OUTER-SV`
  โดยตรง ดังนั้น `STEP337-OUTER-SV` (ซึ่งเป็นแค่ "ปู่" ไม่ใช่ "พ่อ" ของ GRANDCHILD) จึงไม่มีสิทธิ์
  เรียกมันได้โดยตรง
- ที่น่าสังเกตคือ **คอมไพเลอร์ไม่ได้ตรวจจับปัญหานี้ตอนคอมไพล์** — โปรแกรมคอมไพล์ผ่านสมบูรณ์ แต่
  ไปพังตอนรันจริงด้วย Runtime Error `module not found` ซึ่งอันตรายกว่า Compile Error มาก เพราะ
  อาจไม่ถูกตรวจพบจนกว่าจะรันโค้ดเส้นทางนั้นจริงในการทดสอบหรือในการใช้งานจริง

### ข้อควรระวังสรุป

- ก่อนออกแบบโครงสร้าง Nested Programs หลายชั้น ควรวาดแผนผัง (Diagram) ว่าใครห่อหุ้มใครให้ชัดเจน
  ก่อนเขียนโค้ด เพื่อป้องกันการเรียกข้ามขอบเขตที่คอมไพเลอร์ตรวจไม่พบ
- ควรทดสอบ (Test) ทุกเส้นทางการเรียกใช้จริงเสมอ อย่าพึ่งพาการคอมไพล์ผ่านเป็นเครื่องยืนยันว่าโค้ด
  ถูกต้อง เพราะ Error ประเภทขอบเขตการเรียกใน Nested Programs จะปรากฏเฉพาะตอนรันเท่านั้น

### แบบฝึกหัดที่ 337.1

**โจทย์**: จากตัวอย่างที่ผิดพลาดข้างบน จงแก้ไขให้ `STEP337-OUTER-SV` เรียก `STEP337-GRANDCHILD-SV`
ได้สำเร็จ โดยแก้ไขโครงสร้างการซ้อนให้ถูกต้อง (ไม่ใช้ `COMMON`)

**เฉลย**: ย้าย `STEP337-GRANDCHILD-SV` ให้ไปซ้อนอยู่ภายใน `STEP337-OUTER-SV` โดยตรง (เป็น
พี่น้องกับ `STEP337-CHILD-SV` แทนที่จะเป็นลูกของมัน):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-OUTER-SV.
       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "STEP337-GRANDCHILD-SV".
           STOP RUN.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-CHILD-SV.
       PROCEDURE DIVISION.
       CHILD-MAIN.
           DISPLAY "CHILD RUNNING".
           GOBACK.
       END PROGRAM STEP337-CHILD-SV.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP337-GRANDCHILD-SV.
       PROCEDURE DIVISION.
       GC-MAIN.
           DISPLAY "GRANDCHILD RUNNING".
           GOBACK.
       END PROGRAM STEP337-GRANDCHILD-SV.

       END PROGRAM STEP337-OUTER-SV.
```

ตอนนี้ `STEP337-GRANDCHILD-SV` เป็นลูกโดยตรงของ `STEP337-OUTER-SV` จึงเรียกได้สำเร็จ

---

## ขั้นตอนที่ 338: Nested Programs เทียบกับ Subprogram แยกไฟล์ — เมื่อไรควรใช้แบบไหน

### สองวิธีที่ทำสิ่งเดียวกันได้ แต่มีข้อแลกเปลี่ยนต่างกัน

เราได้เรียนรู้มาแล้วว่ามีสองวิธีในการแบ่งโปรแกรมออกเป็นส่วนย่อย: **Subprogram แยกไฟล์** (Part
031-032, คอมไพล์แยกกันแล้วนำมาลิงก์รวมกัน) และ **Nested Program** (Part นี้, อยู่ในไฟล์ซอร์ส
เดียวกัน) มาดูตัวอย่างที่ทำงานเหมือนกันทุกประการทั้งสองแบบ เพื่อเปรียบเทียบให้เห็นชัดเจน

### เวอร์ชัน Nested Program

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP338-MAIN-NESTED.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE            PIC 9(6)V99 VALUE 250.00.
       01  WS-TAX              PIC 9(6)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "STEP338-CALC-TAX" USING WS-PRICE WS-TAX.
           DISPLAY "NESTED VERSION - TAX: " WS-TAX.
           STOP RUN.

      *> Only STEP338-MAIN-NESTED (and anything nested inside IT)
      *> can ever CALL this program. It lives and dies with one
      *> source file - it cannot be reused by an unrelated program.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP338-CALC-TAX.
       DATA DIVISION.
       LINKAGE SECTION.
       01  TAX-INPUT           PIC 9(6)V99.
       01  TAX-OUTPUT          PIC 9(6)V99.
       PROCEDURE DIVISION USING TAX-INPUT TAX-OUTPUT.
       CT-MAIN.
           COMPUTE TAX-OUTPUT = TAX-INPUT * 0.07.
           GOBACK.
       END PROGRAM STEP338-CALC-TAX.

       END PROGRAM STEP338-MAIN-NESTED.
```

คอมไพล์ด้วยคำสั่งเดียว (ไฟล์เดียว): `cobc -x -o step338_nested_version
step338_nested_version.cob`

**ผลลัพธ์:**

```
NESTED VERSION - TAX: 000017.50
```

### เวอร์ชัน Subprogram แยกไฟล์

โปรแกรมหลัก (`step338_main_separate.cob`):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP338-MAIN-SEPARATE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE            PIC 9(6)V99 VALUE 250.00.
       01  WS-TAX              PIC 9(6)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "STEP338CALCTAX" USING WS-PRICE WS-TAX.
           DISPLAY "SEPARATE VERSION - TAX: " WS-TAX.
           STOP RUN.
```

Subprogram แยกไฟล์ (`step338calctax.cob`):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP338CALCTAX.

      *> This subprogram is a SEPARATE .cob file. Any OTHER
      *> program in the whole application can CALL it too - it
      *> is not locked inside one containing program.
       DATA DIVISION.
       LINKAGE SECTION.
       01  TAX-INPUT           PIC 9(6)V99.
       01  TAX-OUTPUT          PIC 9(6)V99.
       PROCEDURE DIVISION USING TAX-INPUT TAX-OUTPUT.
       CT-MAIN.
           COMPUTE TAX-OUTPUT = TAX-INPUT * 0.07.
           GOBACK.
```

คอมไพล์ทั้งสองไฟล์เข้าด้วยกัน:

```
cobc -x step338_main_separate.cob step338calctax.cob -o step338_separate_version
```

**ผลลัพธ์:**

```
SEPARATE VERSION - TAX: 000017.50
```

### ตารางเปรียบเทียบ

| ประเด็น | Nested Program | Subprogram แยกไฟล์ |
|---|---|---|
| จำนวนไฟล์ซอร์ส | ไฟล์เดียว | หลายไฟล์ |
| คอมไพล์ | คำสั่งเดียว | ต้องคอมไพล์/ลิงก์หลายไฟล์เข้าด้วยกัน |
| การใช้ซ้ำข้ามระบบ | ทำไม่ได้ (ผูกกับโปรแกรมที่ห่อหุ้ม) | ทำได้ (โปรแกรมใดก็ตามที่รู้ชื่อก็เรียกได้) |
| การแชร์ข้อมูล | ทำได้ง่ายผ่าน `GLOBAL` | ต้องผ่านพารามิเตอร์ (`USING`) เท่านั้น |
| เหมาะกับ | โค้ดช่วยเหลือเฉพาะโปรแกรมนั้น ไม่มีใครอื่นใช้ | Utility ที่หลายโปรแกรม/หลายทีมใช้ร่วมกัน |
| การดูแลรักษาระยะยาว | ไฟล์ใหญ่ขึ้นเรื่อย ๆ ถ้าซ้อนมาก | แยกไฟล์ชัดเจน จัดการ Version แยกกันได้ |

### อธิบายแนวทางการเลือกใช้

- ใช้ **Nested Program** เมื่อโค้ดส่วนนั้นมีประโยชน์เฉพาะกับโปรแกรมหลักตัวนี้เท่านั้น ไม่มีแผนจะให้
  โปรแกรมอื่นเรียกใช้ และต้องการแชร์ข้อมูลกันง่าย ๆ ผ่าน `GLOBAL` โดยไม่ต้องส่งพารามิเตอร์ยาว ๆ
- ใช้ **Subprogram แยกไฟล์** เมื่อโค้ดส่วนนั้นเป็น Utility ที่มีคุณค่าใช้ซ้ำได้กว้างกว่าโปรแกรมเดียว
  เช่น ฟังก์ชันคำนวณภาษีที่หลายระบบในองค์กรต้องใช้ หรือเมื่อทีมงานต้องการแยกกันดูแล Version ของ
  แต่ละโปรแกรมอย่างอิสระ

### ข้อควรระวัง

- ในองค์กรจริง Subprogram แยกไฟล์เป็นที่นิยมมากกว่า Nested Programs อย่างชัดเจน เพราะยืดหยุ่น
  กว่าในการดูแลรักษาระยะยาวและการทำงานเป็นทีมขนาดใหญ่ Nested Programs มักถูกใช้ในกรณีเฉพาะ
  ที่ต้องการความกระชับของไฟล์เดียว หรือมีเหตุผลด้าน Legacy Code ที่เขียนมาแบบนี้ตั้งแต่ต้น

### แบบฝึกหัดที่ 338.1

**โจทย์**: หากในอนาคตมีความต้องการให้โปรแกรมอื่นในระบบ (ที่ไม่ใช่ `STEP338-MAIN-NESTED`) เรียก
ใช้ตรรกะคำนวณภาษี 7% นี้ด้วย ควรเลือกใช้ Nested Program หรือ Subprogram แยกไฟล์ เพราะเหตุใด

**เฉลย**: ควรเลือก **Subprogram แยกไฟล์** เพราะ Nested Program (`STEP338-CALC-TAX`) ถูกผูกไว้
กับ `STEP338-MAIN-NESTED` เท่านั้น โปรแกรมอื่นที่ไม่ใช่ผู้ห่อหุ้มมันโดยตรงจะไม่สามารถ `CALL` มันได้
เลย (ตามกฎขอบเขตจาก ขั้นตอนที่ 331 และ 337) ในขณะที่ Subprogram แยกไฟล์อย่าง `STEP338CALCTAX`
สามารถถูกเรียกจากโปรแกรมใดก็ได้ในระบบที่รู้จักชื่อของมัน โดยไม่มีข้อจำกัดเรื่องการห่อหุ้ม จึงเหมาะกับ
การใช้ซ้ำข้ามหลายโปรแกรมมากกว่ามาก

---

## ขั้นตอนที่ 339: ข้อผิดพลาดที่พบบ่อยกับ Nested Programs

### สรุปข้อผิดพลาดที่ควรระวังเป็นพิเศษ

Part นี้ได้แสดงข้อผิดพลาดจริงหลายแบบมาแล้วในแต่ละขั้นตอน ขั้นตอนนี้จะรวบรวมและเพิ่มเติมกรณีที่
ยังไม่ได้กล่าวถึง เพื่อให้เห็นภาพครบถ้วนก่อนไปสู่ตัวอย่างสรุปรวมใน ขั้นตอนที่ 340

### ลำดับการประกาศโปรแกรมในไฟล์ไม่กระทบการ CALL (Forward Reference ใช้ได้)

ข้อสงสัยที่พบบ่อยคือ "โปรแกรมในต้องถูกประกาศ**ก่อน**จุดที่มันถูกเรียกในซอร์สโค้ดหรือไม่" คำตอบคือ
**ไม่จำเป็น** เพราะ `CALL` ใช้ชื่อ (literal) อ้างอิงและถูก Resolve ตอนรันจริง ไม่ใช่ตามลำดับการ
ประกาศในไฟล์:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP339-OUTER.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "OUTER CALLS A PROGRAM DEFINED *BELOW* IT".
           CALL "STEP339-LATER-DEFINED".
           STOP RUN.

      *> Even though this program appears AFTER MAIN-PARA calls it,
      *> compilation still succeeds - CALL is resolved at run time,
      *> not by the order the nested programs appear in the file.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP339-LATER-DEFINED.
       PROCEDURE DIVISION.
       LD-MAIN.
           DISPLAY "LATER-DEFINED PROGRAM RUNS FINE".
           GOBACK.
       END PROGRAM STEP339-LATER-DEFINED.
       END PROGRAM STEP339-OUTER.
```

**ผลลัพธ์:**

```
OUTER CALLS A PROGRAM DEFINED *BELOW* IT
LATER-DEFINED PROGRAM RUNS FINE
```

โปรแกรมนี้คอมไพล์และรันได้อย่างราบรื่น แม้ `MAIN-PARA` จะเรียก `STEP339-LATER-DEFINED` ก่อนที่
มันจะถูกประกาศในไฟล์ก็ตาม — สิ่งที่สำคัญคือ**ขอบเขตการซ้อน** (ว่า `LATER-DEFINED` ถูกซ้อนอยู่ใน
`OUTER` หรือไม่) ไม่ใช่ลำดับการเขียนก่อนหลังในไฟล์

### ข้อผิดพลาด: ชื่อ PROGRAM-ID ซ้ำกันในระดับเดียวกัน

หากมีโปรแกรมสองตัวที่ซ้อนอยู่ในระดับเดียวกันแต่ตั้งชื่อ `PROGRAM-ID` เหมือนกัน (ในตัวอย่างนี้คือ
`STEP339-DUP-CHILD` ประกาศซ้ำสองครั้ง) คอมไพเลอร์จะไม่รายงาน Error ในระดับ COBOL โดยตรง แต่
กระบวนการแปลงเป็นภาษา C ภายใน (ซึ่ง GnuCOBOL ใช้ C เป็นตัวกลาง) จะพบการนิยามซ้ำและรายงาน
Error ระดับ C ที่มีรูปแบบซับซ้อนกว่าปกติ:

```
error: redefinition of 'STEP339__DUP__CHILD_0__'
note: previous definition of 'STEP339__DUP__CHILD_0__' with type 'int(void)'
```

### อธิบายจุดสำคัญ

- ข้อความ Error ประเภทนี้ดูน่ากลัวเพราะมันมาจากตัวแปลภาษา C ที่อยู่เบื้องหลัง GnuCOBOL ไม่ใช่
  ข้อความจาก COBOL โดยตรง แต่ต้นตอที่แท้จริงนั้นเรียบง่ายมาก: **มีการประกาศ `PROGRAM-ID` ชื่อ
  เดียวกันซ้ำสองครั้งในระดับการซ้อนเดียวกัน** วิธีแก้คือเปลี่ยนชื่อโปรแกรมใดโปรแกรมหนึ่งให้ไม่ซ้ำกัน
- บทเรียนสำคัญคือ เมื่อเจอ Error ที่ดูเหมือนมาจากภาษา C หรือมีคำที่ไม่คุ้นเคยปรากฏขึ้นระหว่างการ
  คอมไพล์ COBOL ด้วย GnuCOBOL ให้สงสัยไว้ก่อนว่าอาจเป็นปัญหาจากโครงสร้าง COBOL ระดับสูง (เช่น
  ชื่อโปรแกรมซ้ำ) ที่ถูกส่งต่อไปเป็น Error ระดับล่างของตัวแปลภาษา C

### ข้อควรระวังสรุป

- ตั้งชื่อ `PROGRAM-ID` ให้ไม่ซ้ำกันในระดับการซ้อนเดียวกันเสมอ (แม้แต่ในระบบใหญ่ที่มีโปรแกรมนับ
  ร้อยตัว ควรมีธรรมเนียมการตั้งชื่อที่รับประกันความไม่ซ้ำ เช่น ใส่ Prefix ของโมดูล)
- อย่าพึ่งพาการคอมไพล์ผ่านเป็นเครื่องยืนยันความถูกต้องของ "ขอบเขตการเรียก" เสมอไป (จาก
  ขั้นตอนที่ 337) ต้องทดสอบรันจริงประกอบด้วย
- Forward Reference (เรียกโปรแกรมที่ประกาศทีหลังในไฟล์) ใช้งานได้ปกติ ไม่ต้องกังวลเรื่องลำดับการ
  เขียนโปรแกรมในไฟล์ ตราบใดที่ขอบเขตการซ้อนถูกต้อง

### แบบฝึกหัดที่ 339.1

**โจทย์**: จงอธิบายว่าทำไมการที่ COBOL อนุญาตให้ Forward Reference (เรียกโปรแกรมที่ประกาศทีหลัง
ในไฟล์) ทำได้ จึงไม่ขัดแย้งกับกฎที่ว่า "ปู่เรียกหลานโดยตรงไม่ได้" จาก ขั้นตอนที่ 337

**เฉลย**: ทั้งสองกฎพิจารณาคนละมิติกัน Forward Reference เกี่ยวกับ**ลำดับการเขียนโปรแกรมในไฟล์**
(บนลงล่าง) ซึ่ง COBOL ไม่สนใจเลยเพราะ `CALL` ถูก Resolve ด้วยชื่อตอนรันจริง ส่วนกฎ "ปู่เรียกหลาน
ไม่ได้" เกี่ยวกับ**โครงสร้างการซ้อน** (ใครห่อหุ้มใคร) ซึ่งเป็นคนละเรื่องกับลำดับการเขียนในไฟล์โดย
สิ้นเชิง โปรแกรมสองตัวอาจถูกเขียนเรียงติดกันในไฟล์ (ลำดับใกล้กันมาก) แต่ถ้าตัวหนึ่งซ้อนอยู่ในอีกตัว
ที่ไม่ใช่ผู้เรียก การเรียกก็ยังคงล้มเหลวอยู่ดี เพราะสิ่งที่ COBOL ตรวจสอบ (ที่ Runtime) คือโครงสร้าง
การห่อหุ้มจริง ไม่ใช่ตำแหน่งบรรทัดในไฟล์

---

## ขั้นตอนที่ 340: สรุปรวม — ระบบเมนูเล็ก ๆ ที่ประกอบจาก Nested Programs หลายตัว

### นำทุกแนวคิดมาประกอบกันเป็นระบบเดียว

ขั้นตอนสุดท้ายของ Part นี้จะรวมแนวคิดที่เรียนมาทั้งหมด — Nested Program พื้นฐาน, `GLOBAL`,
`COMMON` — เข้าด้วยกันเป็นระบบเมนูจำลองขนาดเล็ก โปรแกรมนี้จำลองการเลือกเมนู "เพิ่มสินค้า" (A)
หรือ "แสดงรายการสินค้า" (L) หรือ "ออกจากระบบ" (X) โดยใช้ตารางค่าที่เตรียมไว้ล่วงหน้าแทนการรับ
ค่าจากผู้ใช้จริง (`ACCEPT`) เพื่อให้ผลลัพธ์คงที่และทดสอบซ้ำได้เสมอ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP340-MENU-SYSTEM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ITEM-COUNT       PIC 9(3) VALUE 0 IS GLOBAL.
       01  WS-ITEM-TABLE IS GLOBAL.
           05  WS-ITEM-ENTRY OCCURS 5 TIMES.
               10  WS-ITEM-NAME    PIC X(15).
       01  WS-CHOICES.
           05  FILLER          PIC X VALUE "A".
           05  FILLER          PIC X VALUE "L".
           05  FILLER          PIC X VALUE "A".
           05  FILLER          PIC X VALUE "L".
           05  FILLER          PIC X VALUE "X".
       01  WS-CHOICE-TABLE REDEFINES WS-CHOICES.
           05  WS-CHOICE       PIC X OCCURS 5 TIMES.
       01  WS-IDX              PIC 9 VALUE 1.
       01  WS-VALID            PIC X VALUE "N".

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               CALL "STEP340-VALIDATE-CHOICE"
                   USING WS-CHOICE(WS-IDX) WS-VALID
               IF WS-VALID = "Y"
                   EVALUATE WS-CHOICE(WS-IDX)
                       WHEN "A"
                           CALL "STEP340-ADD-ITEM"
                       WHEN "L"
                           CALL "STEP340-LIST-ITEMS"
                       WHEN "X"
                           DISPLAY "MENU: EXIT REQUESTED"
                   END-EVALUATE
               ELSE
                   DISPLAY "MENU: INVALID CHOICE IGNORED"
               END-IF
           END-PERFORM.
           STOP RUN.

      *> COMMON so it could also be reached from a sibling that
      *> is not the direct container (not used that way here, but
      *> that is exactly what COMMON is for).
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP340-VALIDATE-CHOICE COMMON.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-CHOICE           PIC X.
       01  LS-VALID            PIC X.
       PROCEDURE DIVISION USING LS-CHOICE LS-VALID.
       VC-MAIN.
           IF LS-CHOICE = "A" OR "L" OR "X"
               MOVE "Y" TO LS-VALID
           ELSE
               MOVE "N" TO LS-VALID
           END-IF.
           GOBACK.
       END PROGRAM STEP340-VALIDATE-CHOICE.

      *> Uses the GLOBAL items declared in the container - no
      *> LINKAGE SECTION needed for them.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP340-ADD-ITEM.
       PROCEDURE DIVISION.
       AI-MAIN.
           ADD 1 TO WS-ITEM-COUNT.
           MOVE "ITEM-" TO WS-ITEM-NAME(WS-ITEM-COUNT).
           DISPLAY "ADD-ITEM: ADDED ITEM #" WS-ITEM-COUNT.
           GOBACK.
       END PROGRAM STEP340-ADD-ITEM.

       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP340-LIST-ITEMS.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LIST-IDX         PIC 9.
       PROCEDURE DIVISION.
       LI-MAIN.
           DISPLAY "LIST-ITEMS: TOTAL=" WS-ITEM-COUNT.
           PERFORM VARYING WS-LIST-IDX FROM 1 BY 1
                   UNTIL WS-LIST-IDX > WS-ITEM-COUNT
               DISPLAY "  " WS-ITEM-NAME(WS-LIST-IDX)
                   " #" WS-LIST-IDX
           END-PERFORM.
           GOBACK.
       END PROGRAM STEP340-LIST-ITEMS.

       END PROGRAM STEP340-MENU-SYSTEM.
```

**ผลลัพธ์:**

```
ADD-ITEM: ADDED ITEM #001
LIST-ITEMS: TOTAL=001
  ITEM-           #1
ADD-ITEM: ADDED ITEM #002
LIST-ITEMS: TOTAL=002
  ITEM-           #1
  ITEM-           #2
MENU: EXIT REQUESTED
```

### อธิบายจุดสำคัญ

- `WS-ITEM-COUNT` และ `WS-ITEM-TABLE` ประกาศเป็น `IS GLOBAL` ใน `STEP340-MENU-SYSTEM`
  (โปรแกรมนอกสุด) ทำให้ทั้ง `STEP340-ADD-ITEM` และ `STEP340-LIST-ITEMS` (โปรแกรมพี่น้องที่
  ซ้อนอยู่ในระดับเดียวกัน) เข้าถึงและแก้ไขข้อมูลชุดเดียวกันได้โดยไม่ต้องส่งผ่านพารามิเตอร์เลย ตรงตาม
  แนวคิดจาก ขั้นตอนที่ 334
- `STEP340-VALIDATE-CHOICE` ประกาศเป็น `COMMON` (แม้ในตัวอย่างนี้จะถูกเรียกจากโปรแกรมนอกเท่านั้น
  ก็ยังคงตั้งใจประกาศไว้เพื่อแสดงให้เห็นว่า ถ้าในอนาคตมีโปรแกรมพี่น้องอื่นต้องการ Validate เงื่อนไข
  เดียวกัน ก็สามารถเรียกมันได้โดยตรงเช่นกัน) ตรงตามแนวคิดจาก ขั้นตอนที่ 335
- โครงสร้างทั้งหมดนี้ใช้ `EVALUATE` (Part 011) ร่วมกับ `PERFORM VARYING` (Part 013) ที่เรียนมา
  ก่อนหน้านี้ในหลักสูตร แสดงให้เห็นว่า Nested Programs ทำงานร่วมกับแนวคิดพื้นฐานอื่น ๆ ของ COBOL
  ได้อย่างกลมกลืน ไม่ใช่แนวคิดที่แยกขาดจากสิ่งที่เรียนมาก่อนหน้า

### ข้อควรระวังสรุปรวมทั้ง Part

- `STOP RUN` ในโปรแกรมที่ถูกเรียก จะจบทั้ง Run Unit ทันที ใช้ `GOBACK` แทนเสมอ (ขั้นตอนที่ 331)
- `END PROGRAM` ต้องมีชื่อตรงกับ `PROGRAM-ID` เป๊ะ และต้องปิดจากในสุดออกไปนอกสุดตามลำดับ
  (ขั้นตอนที่ 332)
- ข้อมูลไม่ถูกแชร์กันโดยปริยาย ต้องใช้ `GLOBAL` อย่างตั้งใจ (ขั้นตอนที่ 333-334)
- `COMMON` เปิดให้โปรแกรมพี่น้องเรียกกันได้ แต่ไม่ได้ขยายสิทธิ์ในการเรียกออกไปหาที่อื่น
  (ขั้นตอนที่ 335)
- `IS INITIAL PROGRAM` รีเซ็ตข้อมูลทุกครั้งที่ถูกเรียก เหมาะกับ Utility ที่ต้องเริ่มสถานะใหม่เสมอ
  (ขั้นตอนที่ 336)
- การซ้อนหลายชั้นต้องระวังเรื่อง "ปู่เรียกหลานไม่ได้" ซึ่งเป็น Runtime Error ที่คอมไพเลอร์ตรวจไม่พบ
  (ขั้นตอนที่ 337)
- เลือกใช้ Nested Program เมื่อโค้ดใช้เฉพาะโปรแกรมเดียว เลือก Subprogram แยกไฟล์เมื่อต้องใช้ซ้ำ
  ข้ามระบบ (ขั้นตอนที่ 338)
- ชื่อ `PROGRAM-ID` ต้องไม่ซ้ำกันในระดับการซ้อนเดียวกัน มิฉะนั้นจะได้ Error ระดับ C ที่อ่านยาก
  (ขั้นตอนที่ 339)

### แบบฝึกหัดที่ 340.1

**โจทย์**: จงเพิ่มโปรแกรมพี่น้องใหม่ชื่อ `STEP340-CLEAR-ITEMS` ที่รีเซ็ต `WS-ITEM-COUNT` กลับเป็น
`0` และเพิ่มตัวเลือกเมนู `"C"` (Clear) ในตาราง `WS-CHOICES` เพื่อเรียกใช้มัน

**เฉลยแนวทาง**:

เพิ่มโปรแกรมใหม่ (ก่อน `END PROGRAM STEP340-MENU-SYSTEM.`):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP340-CLEAR-ITEMS.
       PROCEDURE DIVISION.
       CI-MAIN.
           MOVE 0 TO WS-ITEM-COUNT.
           DISPLAY "CLEAR-ITEMS: ITEM COUNT RESET TO ZERO".
           GOBACK.
       END PROGRAM STEP340-CLEAR-ITEMS.
```

แก้ `VALIDATE-CHOICE` ให้ยอมรับ `"C"` ด้วย:

```cobol
           IF LS-CHOICE = "A" OR "L" OR "X" OR "C"
```

และเพิ่ม `WHEN "C" CALL "STEP340-CLEAR-ITEMS"` ใน `EVALUATE` ของ `MAIN-PARA` พร้อมปรับตาราง
`WS-CHOICES` ให้มีค่า `"C"` แทรกอยู่ในลำดับที่ต้องการทดสอบ

---

## สรุปท้ายบท

Part นี้พาเราเจาะลึกการเขียนโปรแกรมแบบซ้อนกัน (Nested Programs) อย่างครบวงจร ได้แก่:

- แนวคิดพื้นฐานของ Nested Programs และความอันตรายของการใช้ `STOP RUN` แทน `GOBACK` ใน
  โปรแกรมที่ถูกเรียก
- กฎการจับคู่ `END PROGRAM` กับ `PROGRAM-ID` และการจัดการโปรแกรมพี่น้องหลายตัวในระดับเดียวกัน
- ขอบเขตของข้อมูล (Data Scope) ที่แยกจากกันโดยปริยาย และวิธีแชร์ข้อมูลอย่างตั้งใจด้วย `GLOBAL`
- การเปิดให้โปรแกรมพี่น้องเรียกกันได้ด้วย `COMMON`
- การรีเซ็ตข้อมูลทุกครั้งที่ถูกเรียกด้วย `IS INITIAL PROGRAM`
- การซ้อนหลายชั้น (Multi-level Nesting) และกฎขอบเขตการเรียกที่คอมไพเลอร์ตรวจไม่พบตอนคอมไพล์
  แต่จะพังตอนรัน (`module not found`)
- การเปรียบเทียบ Nested Programs กับ Subprogram แยกไฟล์ และแนวทางเลือกใช้ให้เหมาะกับงาน
- ข้อผิดพลาดที่พบบ่อย ทั้ง Forward Reference (ใช้ได้ปกติ) และชื่อ `PROGRAM-ID` ซ้ำกัน (Error
  ระดับ C ที่อ่านยาก)
- ระบบเมนูเล็ก ๆ ที่ผสมผสานทุกแนวคิดของ Part นี้เข้าด้วยกันเป็นตัวอย่างที่ทำงานได้จริง

ตอนนี้เรามีเครื่องมือครบมือแล้วสำหรับการจัดโครงสร้างโปรแกรม COBOL ขนาดใหญ่ ทั้งการแบ่งปัน
โครงสร้างข้อมูลผ่าน `COPY` (Part 033), การแบ่งปันตรรกะผ่าน Subprogram แยกไฟล์ (Part 031-032),
และการซ้อนโปรแกรมในไฟล์เดียวกัน (Part 034 นี้) **Part 035** จะเป็น Part สำคัญที่รวบยอดความรู้ทั้งหมด
จากเฟส 2 (Parts 016-034) เข้าด้วยกันในโปรเจกต์รวบยอด: **ระบบจัดการสินค้าคงคลัง (Inventory
Management System)** ที่ใช้ทั้งตาราง, การจัดการไฟล์, Subprogram, Copybook, และเทคนิคอื่น ๆ ที่
เรียนมาตลอดเฟสนี้ประกอบกันเป็นระบบที่ทำงานได้จริงสมบูรณ์

**[กลับไปยัง Part 033: COPY Statement และการใช้ Copybooks →](part-033-copy-copybooks.md)**

**[ไปยัง Part 035: โปรเจกต์เฟส 2 - ระบบจัดการสินค้าคงคลัง →](part-035-phase2-project.md)**
