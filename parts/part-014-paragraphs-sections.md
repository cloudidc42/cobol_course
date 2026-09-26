# Part 014: โครงสร้างโปรแกรมแบบมีโมดูล: Paragraph, Section, PERFORM...THRU (ขั้นตอนที่ 131–140)

## คำนำของ Part นี้

ใน Part 013 เราเรียนรู้วิธีวนลูปอย่างละเอียดด้วย `PERFORM VARYING` รวมถึงลูปซ้อนหลายชั้น แต่ถ้าเรา
เขียนโปรแกรมทุกอย่างไว้ใน `PROCEDURE DIVISION` เดียวยาว ๆ โดยไม่แบ่งเป็นส่วนย่อย โปรแกรมจะอ่านยากขึ้น
เรื่อย ๆ เมื่อขนาดใหญ่ขึ้น (ลองนึกภาพโปรแกรมจริงในองค์กรที่มีความยาวหลายพันหรือหลายหมื่นบรรทัด)

Part นี้จะสอนวิธี **แบ่งโปรแกรมออกเป็นโมดูลย่อย** ด้วยแนวคิดพื้นฐานที่สุดของ COBOL คือ **Paragraph**
และ **Section** พร้อมคำสั่ง `PERFORM paragraph-name` และ `PERFORM ... THRU` ที่ใช้เรียกใช้งานโมดูล
เหล่านั้น ทักษะนี้คือกุญแจสำคัญที่จะทำให้คุณเขียนโปรแกรมขนาดใหญ่ได้อย่างเป็นระเบียบ และเป็นพื้นฐานที่
จำเป็นก่อนเข้าสู่โปรเจกต์รวบยอดเฟส 1 ใน Part 015

---

## ขั้นตอนที่ 131: Paragraph คืออะไร และการเรียกใช้ด้วย PERFORM

### แนวคิด

**Paragraph** คือส่วนย่อยของ `PROCEDURE DIVISION` ที่มีชื่อกำกับไว้ (paragraph-name) ตามด้วยเครื่องหมาย
จุด (`.`) แล้วตามด้วยชุดคำสั่งหนึ่งชุดขึ้นไป Paragraph จะสิ้นสุดโดยอัตโนมัติเมื่อพบชื่อ Paragraph ถัดไป
หรือ Section ถัดไป หรือจบไฟล์

ไวยากรณ์:

```
paragraph-name.
    statement-1.
    statement-2.
    ...
```

เราเรียกใช้ Paragraph ด้วยคำสั่ง `PERFORM paragraph-name.` ซึ่งจะ**ย้ายการควบคุมไปทำงานที่ Paragraph
นั้น จนจบ Paragraph แล้วย้อนกลับมาทำงานต่อที่จุดเดิม** (คล้ายการเรียกฟังก์ชันในภาษาสมัยใหม่ แต่ไม่มี
การรับพารามิเตอร์หรือค่าคืนกลับโดยตรง — Paragraph ทำงานผ่านตัวแปรร่วมใน `WORKING-STORAGE SECTION`)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP131.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== Basic paragraph PERFORM demo ===".
           DISPLAY "Main paragraph: before calling GREET-PARA".
           PERFORM GREET-PARA.
           DISPLAY "Main paragraph: after calling GREET-PARA".
           PERFORM FAREWELL-PARA.
           DISPLAY "Main paragraph: program finished normally.".
           STOP RUN.

       GREET-PARA.
           DISPLAY "  -> Hello from GREET-PARA!".

       FAREWELL-PARA.
           DISPLAY "  -> Goodbye from FAREWELL-PARA!".
```

### ผลลัพธ์จริงที่ได้

```
=== Basic paragraph PERFORM demo ===
Main paragraph: before calling GREET-PARA
  -> Hello from GREET-PARA!
Main paragraph: after calling GREET-PARA
  -> Goodbye from FAREWELL-PARA!
Main paragraph: program finished normally.
```

### อธิบายโค้ดทีละส่วน

- `MAIN-PARA.`, `GREET-PARA.`, `FAREWELL-PARA.` คือชื่อ Paragraph สามชื่อ ต้องเริ่มเขียนที่ Area A
  (คอลัมน์ 8-11) ตามกฎการเขียนคอลัมน์ที่เรียนไปใน Part 003
- เมื่อรันถึง `PERFORM GREET-PARA.` การควบคุมจะกระโดดไปทำงานที่ `GREET-PARA` จนจบ (ในที่นี้คือ
  `DISPLAY` หนึ่งบรรทัด) แล้วย้อนกลับมาทำงานบรรทัดถัดจาก `PERFORM GREET-PARA.` ทันที — สังเกตจาก
  ผลลัพธ์ว่า "Main paragraph: after calling GREET-PARA" แสดงตามหลัง "Hello from GREET-PARA!" ถูกต้อง
- `STOP RUN.` ยังคงอยู่ใน `MAIN-PARA` และเป็นจุดที่โปรแกรมจบการทำงานทั้งหมด แม้ว่าโค้ดจริงจะยังมี
  Paragraph อื่นตามหลังอยู่ในไฟล์ (`FAREWELL-PARA` ถูกเรียกไปแล้วก่อนหน้า `STOP RUN` จึงไม่มีปัญหา)

### ข้อควรระวัง

- ชื่อ Paragraph ต้องไม่ซ้ำกันภายในโปรแกรมเดียวกัน (ยกเว้นในกรณีพิเศษของ Nested Programs ที่จะเรียน
  ใน Part 034) และไม่ควรตั้งชื่อซ้ำกับคำสงวน (Reserved Word) ของ COBOL
- Paragraph ที่ไม่ได้ถูกเรียกด้วย `PERFORM` เลย แต่การไหลของโปรแกรมจากบนลงล่างเดินมาถึงเข้าโดยบังเอิญ
  ก็จะถูกรันด้วยเช่นกัน (เรียกว่า "fall-through") ซึ่งเป็นพฤติกรรมที่ต้องระวังมาก เราจะพูดถึงปัญหานี้
  อย่างละเอียดในขั้นตอนที่ 139

### แบบฝึกหัดที่ 131.1

**โจทย์**: เพิ่ม Paragraph ใหม่ชื่อ `SHOW-DATE-PARA` ที่แสดงข้อความ "Today's processing complete."
แล้วเรียกใช้จาก `MAIN-PARA` ก่อน `STOP RUN`

**เฉลย**:

```cobol
       MAIN-PARA.
           ...
           PERFORM FAREWELL-PARA.
           PERFORM SHOW-DATE-PARA.
           DISPLAY "Main paragraph: program finished normally.".
           STOP RUN.
       ...
       SHOW-DATE-PARA.
           DISPLAY "Today's processing complete.".
```

---

## ขั้นตอนที่ 132: เรียก Paragraph ซ้ำหลายครั้งด้วย TIMES และ UNTIL

### แนวคิด

`PERFORM paragraph-name` ไม่ได้จำกัดให้เรียกได้แค่ครั้งเดียว เราสามารถผสมกับวลีที่เรียนมาจาก Part 012
และ Part 013 ได้ทั้งหมด เช่น:

```
PERFORM paragraph-name N TIMES.
PERFORM paragraph-name UNTIL condition.
```

รูปแบบนี้ทำให้ Paragraph กลายเป็น "เนื้อลูป" ที่ถูกเรียกซ้ำตามจำนวนครั้งหรือเงื่อนไขที่กำหนด
เหมาะมากเมื่อเนื้อลูปมีความซับซ้อนและอยากแยกออกจากโครงลูปหลักเพื่อความอ่านง่าย

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP132.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER          PIC 9(02) VALUE ZERO.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== PERFORM paragraph-name N TIMES demo ===".
           PERFORM PRINT-STAR-LINE 3 TIMES.
           DISPLAY "Done printing with fixed TIMES.".

           DISPLAY "=== PERFORM paragraph-name UNTIL demo ===".
           PERFORM PRINT-COUNTER UNTIL WS-COUNTER >= 3.
           DISPLAY "Done printing with UNTIL.".
           STOP RUN.

       PRINT-STAR-LINE.
           DISPLAY "  *".

       PRINT-COUNTER.
           ADD 1 TO WS-COUNTER.
           DISPLAY "  Counter is now " WS-COUNTER.
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM paragraph-name N TIMES demo ===
  *
  *
  *
Done printing with fixed TIMES.
=== PERFORM paragraph-name UNTIL demo ===
  Counter is now 01
  Counter is now 02
  Counter is now 03
Done printing with UNTIL.
```

### อธิบายโค้ดทีละส่วน

- `PERFORM PRINT-STAR-LINE 3 TIMES.` เรียก Paragraph `PRINT-STAR-LINE` ซ้ำตรง ๆ 3 ครั้งโดยไม่ต้อง
  มีตัวแปรตัวนับใด ๆ เลย เหมาะกับกรณีที่รู้จำนวนรอบตายตัวอยู่แล้วและไม่สนใจค่าลำดับรอบ
- `PERFORM PRINT-COUNTER UNTIL WS-COUNTER >= 3.` ใช้ตัวแปร `WS-COUNTER` ที่ประกาศไว้นอก Paragraph
  ร่วมกัน (shared state) — Paragraph เพิ่มค่าตัวแปรเองทุกครั้งที่ถูกเรียก และลูปจะหยุดเมื่อค่าถึง 3
  รูปแบบนี้เทียบเท่ากับ `PERFORM UNTIL` ที่เรียนไปใน Part 012 เพียงแต่เนื้อลูปอยู่ใน Paragraph แยก

### ข้อควรระวัง

- เมื่อใช้ `PERFORM paragraph-name UNTIL`, Paragraph นั้น**ต้องมีคำสั่งที่ทำให้เงื่อนไขเปลี่ยนแปลง
  ไปในทางที่จะหยุดลูปได้จริง** (ในตัวอย่างนี้คือ `ADD 1 TO WS-COUNTER`) มิฉะนั้นจะเกิด infinite loop
  เหมือนที่เคยเตือนไว้ใน Part 012-013
- ตัวแปรที่ Paragraph ใช้ร่วมกันแบบนี้ (global-like state ผ่าน `WORKING-STORAGE`) เป็นได้ทั้งข้อดี
  (สื่อสารข้อมูลระหว่างส่วนต่าง ๆ ได้ง่าย) และข้อเสีย (Paragraph หนึ่งอาจไปแก้ค่าที่อีก Paragraph
  ไม่คาดคิด) ซึ่งเป็นเหตุผลสำคัญที่ต้องตั้งชื่อตัวแปรและ Paragraph ให้สื่อความหมายชัดเจนเสมอ

### แบบฝึกหัดที่ 132.1

**โจทย์**: เปลี่ยน `PERFORM PRINT-STAR-LINE 3 TIMES.` ให้พิมพ์ดาว 5 ครั้งแทน

**เฉลย**: แก้เป็น `PERFORM PRINT-STAR-LINE 5 TIMES.` เท่านั้น ไม่ต้องแก้ Paragraph `PRINT-STAR-LINE`
เลย เพราะตัวเลขจำนวนรอบเป็นส่วนหนึ่งของคำสั่ง `PERFORM` ไม่ใช่ของตัว Paragraph

---

## ขั้นตอนที่ 133: SECTION — การจัดกลุ่ม Paragraph หลายตัวเข้าด้วยกัน

### แนวคิด

**Section** คือหน่วยจัดกลุ่มที่ใหญ่กว่า Paragraph — Section หนึ่งตัวสามารถมี Paragraph ย่อยได้หลายตัว
ไวยากรณ์การประกาศ Section คือ `section-name SECTION.` ตามด้วย Paragraph ต่าง ๆ ที่เป็นสมาชิกของ Section
นั้น (จนกว่าจะเจอ Section ถัดไป)

เมื่อเราสั่ง `PERFORM section-name.` COBOL จะรัน **ทุก Paragraph ที่อยู่ภายใน Section นั้นตามลำดับ**
ตั้งแต่ Paragraph แรกไปจนถึง Paragraph สุดท้ายของ Section แล้วจึงย้อนกลับมาทำงานต่อ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP133.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== PERFORM SECTION demo ===".
           DISPLAY "Calling the whole REPORT-SECTION at once:".
           PERFORM REPORT-SECTION.
           DISPLAY "Back in MAIN-PARA after the section finished.".
           STOP RUN.

       REPORT-SECTION SECTION.
       PRINT-HEADER.
           DISPLAY "  ---- Report Header ----".
       PRINT-BODY.
           DISPLAY "  Report body line 1".
           DISPLAY "  Report body line 2".
       PRINT-FOOTER.
           DISPLAY "  ---- Report Footer ----".

       OTHER-SECTION SECTION.
       OTHER-PARA.
           DISPLAY "This paragraph is NOT part of REPORT-SECTION.".
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM SECTION demo ===
Calling the whole REPORT-SECTION at once:
  ---- Report Header ----
  Report body line 1
  Report body line 2
  ---- Report Footer ----
Back in MAIN-PARA after the section finished.
```

### อธิบายโค้ดทีละส่วน

- `REPORT-SECTION SECTION.` ประกาศ Section ชื่อ `REPORT-SECTION` ที่มี Paragraph สมาชิกอยู่ 3 ตัว คือ
  `PRINT-HEADER`, `PRINT-BODY`, `PRINT-FOOTER`
- `PERFORM REPORT-SECTION.` รันทั้ง 3 Paragraph เรียงตามลำดับในไฟล์ทันที โดยไม่ต้องเขียน
  `PERFORM PRINT-HEADER.` แยกทีละตัว
- สังเกตว่า `OTHER-SECTION` ที่ตามมาไม่ถูกรันเลย เพราะเราสั่ง `PERFORM REPORT-SECTION.` เท่านั้น
  ไม่ใช่ `PERFORM OTHER-SECTION.` — นี่ยืนยันว่า `PERFORM section-name` จะหยุดพอดีที่ขอบเขตของ
  Section นั้น ไม่ไหลต่อไปยัง Section ถัดไป

### ข้อควรระวัง

- Section เป็นหน่วยจัดกลุ่มที่มีประโยชน์มากในโปรแกรมขนาดใหญ่ (พบได้บ่อยมากในโค้ด Mainframe จริง)
  แต่ไม่ได้บังคับว่าโปรแกรม COBOL ทุกโปรแกรมต้องมี Section — โปรแกรมเล็ก ๆ ที่ใช้แค่ Paragraph
  ธรรมดา (แบบ Part 131-132) ก็ใช้งานได้สมบูรณ์เช่นกัน
- ถ้าโปรแกรมมีทั้ง Section และ Paragraph ที่ไม่ได้อยู่ใน Section ใดเลย (Paragraph ลอย) ต้องเขียน
  Paragraph ลอยเหล่านั้น**ก่อน**ที่จะประกาศ Section แรก มิฉะนั้นจะเกิด syntax error เพราะ COBOL
  จะตีความว่า Paragraph ลอยเป็นส่วนหนึ่งของ Section ก่อนหน้าโดยอัตโนมัติ

### แบบฝึกหัดที่ 133.1

**โจทย์**: เพิ่ม Paragraph ใหม่ชื่อ `PRINT-SIGNATURE` เข้าไปใน `REPORT-SECTION` (ต่อจาก `PRINT-FOOTER`)
ให้แสดงข้อความ "Signed by the system." แล้วทดสอบว่าเมื่อ `PERFORM REPORT-SECTION.` จะมีข้อความนี้
แสดงออกมาด้วยหรือไม่

**เฉลย**: เพิ่ม Paragraph ก่อน `OTHER-SECTION SECTION.`:

```cobol
       PRINT-FOOTER.
           DISPLAY "  ---- Report Footer ----".
       PRINT-SIGNATURE.
           DISPLAY "  Signed by the system.".

       OTHER-SECTION SECTION.
```

เมื่อรันใหม่ ข้อความ "Signed by the system." จะปรากฏต่อจาก "Report Footer" เพราะเป็น Paragraph
ที่อยู่ภายใน `REPORT-SECTION` เช่นกัน

---

## ขั้นตอนที่ 134: PERFORM ... THRU — รันช่วงของ Paragraph ตามลำดับ

### แนวคิด

บางครั้งเราต้องการรัน Paragraph ติดต่อกันหลายตัว แต่ **ไม่ต้องการจัดกลุ่มเป็น Section** (เช่น อาจ
ต้องการความยืดหยุ่นให้เรียกแค่บางส่วนได้ในบางสถานการณ์) COBOL มีคำสั่ง `PERFORM para-A THRU para-Z`
(หรือเขียนว่า `THROUGH` ก็ได้ ความหมายเหมือนกัน) ซึ่งจะรัน Paragraph ทุกตัวที่อยู่ **ระหว่าง** `para-A`
ถึง `para-Z` ตามลำดับที่ปรากฏในไฟล์ต้นฉบับ (physical sequence) ไม่ใช่ตามชื่อตามตัวอักษร

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP134.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== PERFORM ... THRU demo ===".
           PERFORM STEP-A THRU STEP-C.
           DISPLAY "Back in MAIN-PARA after PERFORM ... THRU.".
           STOP RUN.

       STEP-A.
           DISPLAY "  Running STEP-A".
       STEP-B.
           DISPLAY "  Running STEP-B".
       STEP-C.
           DISPLAY "  Running STEP-C".
       STEP-D.
           DISPLAY "  STEP-D is outside the THRU range, not run here.".
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM ... THRU demo ===
  Running STEP-A
  Running STEP-B
  Running STEP-C
Back in MAIN-PARA after PERFORM ... THRU.
```

### อธิบายโค้ดทีละส่วน

- `PERFORM STEP-A THRU STEP-C.` จะรัน `STEP-A`, `STEP-B`, `STEP-C` เรียงกันตามลำดับที่ปรากฏในไฟล์
  แม้ว่าเราจะไม่ได้เขียน `PERFORM STEP-B.` แยกไว้เลยก็ตาม — จุดสำคัญคือ COBOL ใช้ **ตำแหน่งจริงในไฟล์
  ต้นฉบับ** เป็นตัวกำหนดว่า Paragraph ใดอยู่ "ระหว่าง" จุดเริ่มกับจุดจบ ไม่ใช่ลำดับตัวอักษรของชื่อ
- `STEP-D` อยู่นอกขอบเขตที่ระบุ (`THRU STEP-C`) จึงไม่ถูกรันเลย ยืนยันได้จากผลลัพธ์ที่ไม่มีข้อความ
  "STEP-D" ปรากฏออกมา

### ข้อควรระวัง

- **ความเสี่ยงสำคัญที่สุดของ `PERFORM ... THRU`**: หากมีใครมาแทรก Paragraph ใหม่ระหว่าง `STEP-A`
  กับ `STEP-C` ในภายหลัง (เช่นเพิ่ม `STEP-B2` ไว้ระหว่าง `STEP-B` กับ `STEP-C`) Paragraph ใหม่นั้นจะ
  ถูกรันไปด้วยโดยอัตโนมัติ **แม้ผู้เขียนโค้ดใหม่จะไม่ได้ตั้งใจ** เพราะ `PERFORM ... THRU` อ้างอิงจาก
  ตำแหน่งในไฟล์ ไม่ใช่รายชื่อที่ระบุไว้ชัดเจน นี่คือเหตุผลที่โค้ด Mainframe จริงจำนวนมากนิยมกำหนด
  Paragraph พิเศษชื่อ `xxxx-EXIT` ไว้ท้ายช่วงเสมอ (จะสอนใน Part 136 และ 140) เพื่อทำเครื่องหมาย
  ขอบเขตให้ชัดเจน ป้องกันการแทรกโค้ดผิดที่โดยไม่รู้ตัว
- `PERFORM ... THRU` ใช้ได้กับทั้งชื่อ Paragraph และชื่อ Section ผสมกันได้ (เช่น
  `PERFORM 1000-INIT THRU 2000-PROCESS.`) แต่ควรใช้อย่างระมัดระวังเพื่อไม่ให้โค้ดอ่านยากเกินไป

### แบบฝึกหัดที่ 134.1

**โจทย์**: ถ้าเปลี่ยนคำสั่งเป็น `PERFORM STEP-B THRU STEP-D.` ผลลัพธ์จะเป็นอย่างไร

**เฉลย**: จะรัน `STEP-B`, `STEP-C`, `STEP-D` เรียงกัน ได้ผลลัพธ์:

```
  Running STEP-B
  Running STEP-C
  STEP-D is outside the THRU range, not run here.
```

(ข้อความใน `STEP-D` แม้จะเขียนไว้ว่า "is outside the THRU range" แต่นั่นเป็นเพียงข้อความอธิบายใน
โค้ดตัวอย่างเดิม ในกรณีนี้ `STEP-D` ถูกรวมอยู่ใน THRU range แล้วจริง ๆ เพราะเราระบุ `THRU STEP-D`
โดยตรง)

---

## ขั้นตอนที่ 135: การเรียก Paragraph ซ้อนกัน (Chained PERFORM) และข้อจำกัดเรื่อง Recursion

### แนวคิด

Paragraph หนึ่งสามารถ `PERFORM` Paragraph อื่นได้ และ Paragraph นั้นก็สามารถ `PERFORM` Paragraph อื่น
ต่อไปอีกได้เรื่อย ๆ สร้างเป็น "สายโซ่การเรียก" (call chain) หลายชั้น เหมือนการเรียกฟังก์ชันซ้อนกัน
ในภาษาโปรแกรมทั่วไป

**ข้อควรรู้ที่สำคัญมาก**: COBOL แบบดั้งเดิม **ไม่รองรับการเรียกตัวเองซ้ำ (Recursion) ผ่าน `PERFORM`**
กล่าวคือ ถ้า Paragraph A กำลังถูก `PERFORM` อยู่ และมีคำสั่งภายในที่ `PERFORM A` ซ้อนตัวเองเข้าไปอีก
พฤติกรรมจะไม่ได้ผลลัพธ์ตามที่ภาษาสมัยใหม่ (เช่น Python, Java) คาดหวังไว้ — ในหลายคอมไพเลอร์รวมถึง
GnuCOBOL แบบมาตรฐาน (ไม่เปิด `RECURSIVE` clause พิเศษ) จะถือว่าเป็น**พฤติกรรมที่ไม่ได้นิยาม**
(undefined behavior) จึงควรหลีกเลี่ยงการเขียน Paragraph ที่เรียกตัวเองเด็ดขาดในระดับนี้ของหลักสูตร

### ตัวอย่างโค้ด: สายโซ่การเรียกที่ปลอดภัย (ไม่มี Recursion)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP135.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== Chained (nested) PERFORM calls demo ===".
           PERFORM PROCESS-ORDER.
           DISPLAY "Back in MAIN-PARA. All processing done.".
           STOP RUN.

       PROCESS-ORDER.
           DISPLAY "PROCESS-ORDER: validating order...".
           PERFORM VALIDATE-CUSTOMER.
           DISPLAY "PROCESS-ORDER: validated, calculating total...".
           PERFORM CALCULATE-TOTAL.
           DISPLAY "PROCESS-ORDER: finished.".

       VALIDATE-CUSTOMER.
           DISPLAY "  VALIDATE-CUSTOMER: customer looks valid.".

       CALCULATE-TOTAL.
           DISPLAY "  CALCULATE-TOTAL: total calculated.".
```

### ผลลัพธ์จริงที่ได้

```
=== Chained (nested) PERFORM calls demo ===
PROCESS-ORDER: validating order...
  VALIDATE-CUSTOMER: customer looks valid.
PROCESS-ORDER: validated, calculating total...
  CALCULATE-TOTAL: total calculated.
PROCESS-ORDER: finished.
Back in MAIN-PARA. All processing done.
```

### อธิบายโค้ดทีละส่วน

- ลำดับการเรียกคือ `MAIN-PARA` → `PROCESS-ORDER` → `VALIDATE-CUSTOMER` (แล้วย้อนกลับมาที่
  `PROCESS-ORDER`) → `CALCULATE-TOTAL` (แล้วย้อนกลับมาที่ `PROCESS-ORDER` อีกครั้ง) → ย้อนกลับมาที่
  `MAIN-PARA` — เป็นสายโซ่การเรียก 3 ระดับที่ทำงานถูกต้องตามลำดับที่คาดไว้ทุกประการ
- สังเกตจากผลลัพธ์ว่าข้อความจาก `VALIDATE-CUSTOMER` แทรกอยู่**ระหว่าง**ข้อความสองบรรทัดของ
  `PROCESS-ORDER` พอดี ซึ่งพิสูจน์ว่าการควบคุมย้อนกลับมาทำงานต่อที่จุดเดิมได้ถูกต้องหลังจบ Paragraph
  ที่ถูกเรียก

### ข้อควรระวัง

- แม้ Paragraph จะเรียกซ้อนกันได้หลายชั้น (ในทางปฏิบัติมักไม่ควรเกิน 3-4 ชั้นเพื่อความอ่านง่าย)
  แต่**ห้ามให้เกิดวงจรการเรียกกลับมาหาตัวเอง** เช่น A เรียก B และ B เรียก A กลับไปอีก (circular call)
  เพราะจะทำให้โปรแกรม stack overflow หรือพฤติกรรมผิดเพี้ยนได้
- หากต้องการฟังก์ชันที่ทำงานแบบเรียกตัวเอง (เช่น คำนวณแฟกทอเรียลแบบ recursive) ใน COBOL ยุคใหม่
  มีการรองรับผ่าน `CALL ... RECURSIVE` กับ Subprogram (จะสอนใน Part 031-034) ไม่ใช่ผ่าน `PERFORM`
  ธรรมดาแบบที่เรียนในบทนี้

### แบบฝึกหัดที่ 135.1

**โจทย์**: เพิ่ม Paragraph ใหม่ชื่อ `SEND-CONFIRMATION` ที่แสดงข้อความ "Confirmation sent." แล้วให้
`PROCESS-ORDER` เรียกใช้เป็นขั้นตอนสุดท้ายก่อนจบ

**เฉลย**:

```cobol
       PROCESS-ORDER.
           DISPLAY "PROCESS-ORDER: validating order...".
           PERFORM VALIDATE-CUSTOMER.
           DISPLAY "PROCESS-ORDER: validated, calculating total...".
           PERFORM CALCULATE-TOTAL.
           PERFORM SEND-CONFIRMATION.
           DISPLAY "PROCESS-ORDER: finished.".
       ...
       SEND-CONFIRMATION.
           DISPLAY "  Confirmation sent.".
```

---

## ขั้นตอนที่ 136: แบบแผนการตั้งชื่อ Paragraph แบบมีโครงสร้าง (Numbered Paragraph Convention)

### แนวคิด

ในโค้ด COBOL ระดับองค์กรจริง (โดยเฉพาะบน Mainframe) มีธรรมเนียมการตั้งชื่อ Paragraph ที่นิยมใช้กัน
อย่างแพร่หลายเรียกว่า **Numbered Paragraph Convention** โดยแบ่งเลขนำหน้าเป็นช่วง ๆ ตามหน้าที่ของ
โปรแกรม เช่น:

| ช่วงเลข | ความหมาย |
|---|---|
| `0000-xxxx` | Paragraph หลัก (Main Control) ที่เรียกลำดับการทำงานทั้งหมด |
| `1000-xxxx` | ขั้นตอนเริ่มต้น (Initialization) |
| `2000-xxxx` ถึง `8000-xxxx` | ขั้นตอนประมวลผลหลัก (Processing) แบ่งเป็นกลุ่มตามหน้าที่ |
| `9000-xxxx` | ขั้นตอนปิดท้าย/สรุปผล (Termination) |

ข้อดีของธรรมเนียมนี้คือทำให้เห็นภาพรวมของโปรแกรมได้ทันทีจากชื่อ Paragraph โดยไม่ต้องอ่านโค้ดละเอียด
และช่วยให้การแทรก Paragraph ใหม่ในอนาคตทำได้ง่าย (เช่น แทรก `2500-xxxx` ระหว่าง `2000` กับ `3000`)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP136.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EMPLOYEE-COUNT   PIC 9(03) VALUE ZERO.
       01  WS-TOTAL-SALARY     PIC 9(07)V99 VALUE ZERO.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-INITIALIZE.
           PERFORM 2000-PROCESS-DATA.
           PERFORM 9000-TERMINATE.
           STOP RUN.

       1000-INITIALIZE.
           DISPLAY "1000-INITIALIZE: setting up starting values...".
           MOVE ZERO TO WS-EMPLOYEE-COUNT.
           MOVE ZERO TO WS-TOTAL-SALARY.

       2000-PROCESS-DATA.
           DISPLAY "2000-PROCESS-DATA: processing employee records...".
           ADD 1 TO WS-EMPLOYEE-COUNT.
           ADD 25000.50 TO WS-TOTAL-SALARY.
           ADD 1 TO WS-EMPLOYEE-COUNT.
           ADD 30000.00 TO WS-TOTAL-SALARY.
           DISPLAY "  Employees processed: " WS-EMPLOYEE-COUNT.
           DISPLAY "  Total salary so far: " WS-TOTAL-SALARY.

       9000-TERMINATE.
           DISPLAY "9000-TERMINATE: printing final summary...".
           DISPLAY "  Final employee count: " WS-EMPLOYEE-COUNT.
           DISPLAY "  Final total salary:   " WS-TOTAL-SALARY.
```

### ผลลัพธ์จริงที่ได้

```
1000-INITIALIZE: setting up starting values...
2000-PROCESS-DATA: processing employee records...
  Employees processed: 002
  Total salary so far: 0055000.50
9000-TERMINATE: printing final summary...
  Final employee count: 002
  Final total salary:   0055000.50
```

### อธิบายโค้ดทีละส่วน

- `0000-MAIN-PROCESS` ทำหน้าที่เป็น "สารบัญ" ของโปรแกรมทั้งหมด อ่านแค่ Paragraph นี้ก็รู้ทันทีว่า
  โปรแกรมทำอะไรบ้างตามลำดับ (initialize → process → terminate) โดยไม่ต้องลงรายละเอียด
- การเว้นช่วงตัวเลข (0000, 1000, 2000, ..., 9000) แทนที่จะเรียงต่อกัน (1, 2, 3, ...) เปิดโอกาสให้
  แทรก Paragraph ใหม่ในอนาคตได้โดยไม่ต้องเปลี่ยนเลขของ Paragraph ที่มีอยู่เดิมทั้งหมด เช่น ถ้าต้อง
  เพิ่มขั้นตอนตรวจสอบข้อมูลก่อนประมวลผล ก็สามารถตั้งชื่อ `1500-VALIDATE-DATA` แทรกได้ทันที

### ข้อควรระวัง

- ธรรมเนียมนี้เป็น "แนวปฏิบัติที่แนะนำ" ไม่ใช่กฎบังคับของภาษา COBOL คุณสามารถตั้งชื่อ Paragraph
  แบบใดก็ได้ที่ถูกไวยากรณ์ แต่การไม่ใช้ธรรมเนียมนี้ในโปรแกรมขนาดใหญ่มักทำให้โค้ดอ่านยากขึ้นมาก
  เมื่อทำงานร่วมกับทีมหรือดูแลโค้ด Legacy ที่มีคนอื่นเขียนไว้
- ตัวเลขที่ใช้ไม่จำเป็นต้องเป็นหลักพันเสมอไป บางองค์กรใช้ตัวเลขสองหลักหรือหกหลักตามขนาดของโปรแกรม
  สิ่งสำคัญคือ **ความสม่ำเสมอ (consistency) ภายในทีมหรือองค์กรเดียวกัน**

### แบบฝึกหัดที่ 136.1

**โจทย์**: เพิ่มขั้นตอนตรวจสอบข้อมูลก่อนประมวลผล โดยแทรก Paragraph ชื่อ `1500-VALIDATE-INPUT` ที่
แสดงข้อความ "1500-VALIDATE-INPUT: input looks OK." แล้วเรียกใช้ใน `0000-MAIN-PROCESS` ระหว่าง
`1000-INITIALIZE` กับ `2000-PROCESS-DATA`

**เฉลย**:

```cobol
       0000-MAIN-PROCESS.
           PERFORM 1000-INITIALIZE.
           PERFORM 1500-VALIDATE-INPUT.
           PERFORM 2000-PROCESS-DATA.
           PERFORM 9000-TERMINATE.
           STOP RUN.
       ...
       1500-VALIDATE-INPUT.
           DISPLAY "1500-VALIDATE-INPUT: input looks OK.".
```

---

## ขั้นตอนที่ 137: ผสาน PERFORM VARYING เข้ากับการเรียก Paragraph

### แนวคิด

เรารู้จัก `PERFORM VARYING` (Part 013) และการเรียก Paragraph (ขั้นตอนที่ 131-136) แยกกันมาแล้ว
ตอนนี้เราจะรวมทั้งสองอย่างเข้าด้วยกัน: ใช้ `PERFORM paragraph-name VARYING ... FROM ... BY ... UNTIL`
เพื่อเรียก Paragraph ซ้ำ ๆ พร้อมกับมีตัวแปรตัวนับควบคุมอัตโนมัติ — รูปแบบนี้เป็นการรวม "โครงลูป"
(loop control) กับ "เนื้อหาที่แยกเป็นโมดูล" (modular body) ไว้ในคำสั่งเดียว ทำให้โค้ดของลูปที่ซับซ้อน
อ่านง่ายขึ้นมาก เพราะรายละเอียดการทำงานถูกซ่อนไว้ใน Paragraph แยกต่างหาก

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP137.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE-NO          PIC 9(02).

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== PERFORM paragraph-name VARYING demo ===".
           PERFORM PRINT-NUMBERED-LINE
                   VARYING WS-LINE-NO FROM 1 BY 1
                   UNTIL WS-LINE-NO > 4.
           DISPLAY "Finished printing 4 numbered lines.".
           STOP RUN.

       PRINT-NUMBERED-LINE.
           DISPLAY "  Line number " WS-LINE-NO.
```

### ผลลัพธ์จริงที่ได้

```
=== PERFORM paragraph-name VARYING demo ===
  Line number 01
  Line number 02
  Line number 03
  Line number 04
Finished printing 4 numbered lines.
```

### อธิบายโค้ดทีละส่วน

- `PERFORM PRINT-NUMBERED-LINE VARYING WS-LINE-NO FROM 1 BY 1 UNTIL WS-LINE-NO > 4.` มีความหมาย
  เหมือนกับการเขียน `PERFORM VARYING ... END-PERFORM` ที่มี `PERFORM PRINT-NUMBERED-LINE.` อยู่ข้างใน
  ทุกประการ เพียงแต่กระชับกว่าเพราะไม่ต้องมี `END-PERFORM` และไม่ต้องเขียนเนื้อลูปแทรกอยู่ตรงกลาง
- `WS-LINE-NO` ถูกควบคุมโดยคำสั่ง `PERFORM` เอง (initialize, increment, check condition) ในขณะที่
  Paragraph `PRINT-NUMBERED-LINE` มีหน้าที่แค่ "ใช้" ค่าตัวแปรนั้นในการแสดงผลเท่านั้น — เป็นการแบ่ง
  ความรับผิดชอบ (separation of concerns) ที่ชัดเจน

### ข้อควรระวัง

- รูปแบบนี้เหมาะกับกรณีที่เนื้อลูปมีความซับซ้อนพอสมควรจนควรแยกเป็น Paragraph ต่างหาก แต่ถ้าเนื้อลูป
  สั้นมาก (แค่ 1-2 บรรทัด) การเขียนแบบ inline `PERFORM VARYING ... END-PERFORM` แบบ Part 013 อาจ
  อ่านง่ายกว่าเพราะเห็นทุกอย่างอยู่ในที่เดียว
- หาก Paragraph ที่ถูกเรียกด้วยวิธีนี้ไปแก้ไขค่าตัวแปรตัวนับเอง (เช่น เผลอ `ADD 1 TO WS-LINE-NO`
  ซ้ำเข้าไปอีกใน `PRINT-NUMBERED-LINE`) จะทำให้ค่าตัวนับกระโดดผิดจากที่ `PERFORM` คาดหวังไว้ ควรถือ
  เป็นกฎว่า **ตัวแปรตัวนับควรถูกแก้ไขโดยคำสั่ง `PERFORM VARYING` เท่านั้น** ไม่ควรมีใครมาแก้ไขซ้ำเอง
  ข้างในเนื้อลูป

### แบบฝึกหัดที่ 137.1

**โจทย์**: ปรับให้พิมพ์เลขคู่ตั้งแต่ 2 ถึง 8 แทน (ใช้ `PERFORM ... VARYING` ตามรูปแบบเดียวกัน)

**เฉลย**:

```cobol
           PERFORM PRINT-NUMBERED-LINE
                   VARYING WS-LINE-NO FROM 2 BY 2
                   UNTIL WS-LINE-NO > 8.
```

---

## ขั้นตอนที่ 138: GO TO เทียบกับ PERFORM — ทำไม COBOL สมัยใหม่เลี่ยงการใช้ GO TO

### แนวคิด

ก่อนที่ COBOL-85 จะนำแนวคิด Structured Programming (`PERFORM`, `IF...END-IF`, `EVALUATE`) เข้ามา
โปรแกรมเมอร์ยุคก่อนต้องพึ่งพาคำสั่ง **`GO TO`** เป็นหลักในการควบคุมทิศทางการทำงานของโปรแกรม ซึ่งจะ
"กระโดด" ไปยัง Paragraph เป้าหมายทันทีโดย**ไม่มีการย้อนกลับมาอัตโนมัติ**เหมือน `PERFORM`

รูปแบบหนึ่งที่ยังพบได้ในโค้ด Legacy คือ `GO TO ... DEPENDING ON` ซึ่งเลือกกระโดดไปยัง Paragraph ตาม
ค่าตัวแปร (คล้าย `switch` ที่ใช้ label กระโดดในภาษา C) แต่ในปัจจุบันแนวทางที่แนะนำคือใช้ `EVALUATE`
(Part 011) แทน เพราะอ่านง่ายกว่าและปลอดภัยกว่ามาก

### ตัวอย่างโค้ด: เปรียบเทียบสองรูปแบบในโปรแกรมเดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP138.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DAY-CODE         PIC 9(01) VALUE 2.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== GO TO DEPENDING ON demo (legacy style) ===".
           GO TO SHOW-MONDAY SHOW-TUESDAY SHOW-WEDNESDAY
               DEPENDING ON WS-DAY-CODE.
           DISPLAY "This line is skipped by GO TO.".

       SHOW-MONDAY.
           DISPLAY "  Today is Monday.".
           GO TO END-OF-PROGRAM.

       SHOW-TUESDAY.
           DISPLAY "  Today is Tuesday.".
           GO TO END-OF-PROGRAM.

       SHOW-WEDNESDAY.
           DISPLAY "  Today is Wednesday.".
           GO TO END-OF-PROGRAM.

       END-OF-PROGRAM.
           DISPLAY "=== Modern equivalent using EVALUATE ===".
           EVALUATE WS-DAY-CODE
               WHEN 1
                   DISPLAY "  Today is Monday."
               WHEN 2
                   DISPLAY "  Today is Tuesday."
               WHEN 3
                   DISPLAY "  Today is Wednesday."
               WHEN OTHER
                   DISPLAY "  Unknown day code."
           END-EVALUATE.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
=== GO TO DEPENDING ON demo (legacy style) ===
  Today is Tuesday.
=== Modern equivalent using EVALUATE ===
  Today is Tuesday.
```

### อธิบายโค้ดทีละส่วน

- `GO TO SHOW-MONDAY SHOW-TUESDAY SHOW-WEDNESDAY DEPENDING ON WS-DAY-CODE.` จะกระโดดไปยัง Paragraph
  ตัวที่ `WS-DAY-CODE` ระบุ (นับจาก 1) — เนื่องจาก `WS-DAY-CODE` มีค่า 2 จึงกระโดดไปที่ `SHOW-TUESDAY`
  (ตัวที่สองในรายการ) ตรงกับผลลัพธ์ที่ได้
- สังเกตว่าแต่ละ Paragraph ปลายทาง (`SHOW-MONDAY`, `SHOW-TUESDAY`, `SHOW-WEDNESDAY`) ต้องมี
  `GO TO END-OF-PROGRAM.` กำกับไว้เอง เพื่อป้องกัน "fall-through" ตกไปยัง Paragraph ถัดไปโดยไม่ตั้งใจ
  (ปัญหานี้จะอธิบายละเอียดในขั้นตอนที่ 139) — นี่คือภาระที่เพิ่มขึ้นมากเมื่อใช้ `GO TO`
- เวอร์ชัน `EVALUATE` ทำงานให้ผลลัพธ์เหมือนกันทุกประการ แต่ไม่ต้องกังวลเรื่อง fall-through เลย
  เพราะ `EVALUATE ... END-EVALUATE` มีขอบเขตปิดชัดเจนในตัวเอง

### ข้อควรระวัง

- หลักสูตรนี้และวงการ COBOL สมัยใหม่**ไม่แนะนำให้ใช้ `GO TO` ในการเขียนโปรแกรมใหม่** ควรใช้
  `PERFORM`, `IF`, `EVALUATE` แทนเสมอ เนื่องจากทำให้โค้ดคาดเดาทิศทางการทำงานได้ยาก (เรียกปัญหานี้ว่า
  "Spaghetti Code") และดูแลรักษายากมากเมื่อโปรแกรมมีขนาดใหญ่
- เหตุผลที่ยังต้องเรียนรู้ `GO TO` ไว้คือ **โค้ด Legacy จำนวนมหาศาลที่ยังทำงานอยู่จริงในองค์กร**
  (ย้อนกลับไปดู Part 001 เรื่องสถิติ COBOL ในปี 2026) เขียนด้วย `GO TO` เป็นหลัก นักพัฒนา COBOL
  มืออาชีพต้อง**อ่านและเข้าใจ**โค้ดแบบนี้ได้ แม้จะไม่เขียนโค้ดใหม่ในรูปแบบนี้ก็ตาม

### แบบฝึกหัดที่ 138.1

**โจทย์**: จงอธิบายว่าถ้าไม่มี `GO TO END-OF-PROGRAM.` ต่อท้าย `DISPLAY "Today is Monday."` ใน
`SHOW-MONDAY` จะเกิดอะไรขึ้นเมื่อ `WS-DAY-CODE` มีค่า 1

**เฉลย**: โปรแกรมจะแสดง "Today is Monday." แล้ว**ไหลต่อลงไปยัง `SHOW-TUESDAY` ทันที** (fall-through)
เพราะไม่มีคำสั่งใดบอกให้หยุดหรือกระโดดออกจาก Paragraph นี้ ผลลัพธ์ที่ได้จะกลายเป็นแสดงทั้ง
"Today is Monday." และ "Today is Tuesday." และ "Today is Wednesday." ต่อกันหมด ทั้งที่ตั้งใจให้แสดง
แค่ข้อความเดียว — นี่คือตัวอย่างคลาสสิกของปัญหา fall-through ที่เกิดจากการใช้ `GO TO` โดยไม่ระวัง

---

## ขั้นตอนที่ 139: กับดัก Fall-through — เมื่อโปรแกรมไหลเข้า Paragraph ถัดไปโดยไม่ตั้งใจ

### แนวคิด

นี่คือหนึ่งในกับดักที่อันตรายที่สุดสำหรับผู้เริ่มต้นเขียน COBOL: **Paragraph ไม่มีขอบเขตปิดในตัวเอง
เหมือนฟังก์ชันในภาษาสมัยใหม่** เมื่อโปรแกรมทำงานมาถึง Paragraph หนึ่งแล้วทำจนจบคำสั่งสุดท้าย
**การควบคุมจะไหลต่อไปยัง Paragraph ถัดไปในไฟล์โดยอัตโนมัติทันที** แม้ว่าเราจะไม่ได้ตั้งใจให้เกิดขึ้น
ก็ตาม (พฤติกรรมนี้แตกต่างจากการเรียกด้วย `PERFORM` ที่จะย้อนกลับมาที่จุดเดิมพอดี)

กับดักนี้มักเกิดเมื่อ **ลืมใส่ `STOP RUN`** ในจุดที่ควรจะจบการทำงานของโปรแกรม

### ตัวอย่างโค้ด: เวอร์ชันที่มีบั๊ก (compile และรันจริงเพื่อพิสูจน์ปัญหา)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP139.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== Fall-through pitfall demo ===".
           DISPLAY "Main start.".
           PERFORM SUB-PARA.
           DISPLAY "Main end.".
      *> BUG: no STOP RUN here! Execution keeps falling through
      *> into SUB-PARA below, even though it already ran once.

       SUB-PARA.
           DISPLAY "Sub-para running.".

       LAST-PARA.
           DISPLAY "Last-para running.".
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้ (บั๊กที่เกิดขึ้นจริง)

```
=== Fall-through pitfall demo ===
Main start.
Sub-para running.
Main end.
Sub-para running.
Last-para running.
```

สังเกตว่า **"Sub-para running." ปรากฏถึงสองครั้ง** ทั้งที่โค้ดเรียก `PERFORM SUB-PARA.` แค่ครั้งเดียว!

### อธิบายสาเหตุของบั๊ก

1. `PERFORM SUB-PARA.` ทำงานปกติ: กระโดดไป `SUB-PARA`, แสดง "Sub-para running." ครั้งที่ 1 แล้ว
   ย้อนกลับมาที่ `MAIN-PARA` ต่อจาก `PERFORM`
2. `DISPLAY "Main end.".` ทำงานตามปกติ
3. **`MAIN-PARA` จบคำสั่งสุดท้ายแล้ว แต่ไม่มี `STOP RUN`** ดังนั้นการควบคุมจึง**ไหลต่อไปยัง Paragraph
   ถัดไปในไฟล์โดยอัตโนมัติ** ซึ่งก็คือ `SUB-PARA` — ทำให้ `SUB-PARA` ถูกรันเป็น**ครั้งที่สอง**
   (แสดง "Sub-para running." อีกครั้ง) แต่ครั้งนี้ไม่ได้มาจากคำสั่ง `PERFORM` เลย!
4. จากนั้นไหลต่อไปยัง `LAST-PARA` โดยอัตโนมัติเช่นกัน จนกระทั่งเจอ `STOP RUN` ในที่สุด

### เวอร์ชันที่แก้ไขแล้ว (compile และรันจริง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP139F.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== Fall-through fixed with STOP RUN ===".
           DISPLAY "Main start.".
           PERFORM SUB-PARA.
           DISPLAY "Main end.".
           STOP RUN.

       SUB-PARA.
           DISPLAY "Sub-para running.".

       LAST-PARA.
           DISPLAY "Last-para running.".
```

### ผลลัพธ์จริงที่ได้ (หลังแก้ไข)

```
=== Fall-through fixed with STOP RUN ===
Main start.
Sub-para running.
Main end.
```

เห็นได้ชัดว่าเมื่อเพิ่ม `STOP RUN.` เข้าไปหลัง "Main end." โปรแกรมจะหยุดทันทีที่จุดนั้น ไม่มีการไหล
ต่อไปยัง `SUB-PARA` หรือ `LAST-PARA` อีก — ผลลัพธ์ถูกต้องตามที่ตั้งใจไว้แต่แรก

### ข้อควรระวัง

- **กฎทองที่ต้องจำ**: `MAIN-PARA` (หรือ Paragraph หลักตัวแรกของโปรแกรม) ต้องมี `STOP RUN.` กำกับไว้
  เสมอที่จุดที่ต้องการให้โปรแกรมจบการทำงานจริง ๆ อย่าคิดว่าโปรแกรมจะ "หยุดเอง" เมื่อ Paragraph
  หลักทำงานครบทุกคำสั่งแล้ว
- ปัญหานี้ยิ่งอันตรายมากขึ้นเมื่อโปรแกรมมีหลาย Paragraph เรียงต่อกันหลายสิบตัว เพราะ fall-through
  อาจไหลผ่าน Paragraph หลายตัวติดต่อกัน ทำให้เกิดผลข้างเคียงที่คาดไม่ถึงจำนวนมาก (เช่น ตัวแปรถูก
  MOVE ค่าซ้ำโดยไม่ตั้งใจ, ตัวนับถูกเพิ่มค่าซ้ำ) และมักเป็นบั๊กที่หาสาเหตุยากมากในโปรแกรมขนาดใหญ่
- แนวทางป้องกันที่ดีคือใช้ธรรมเนียม Numbered Paragraph (ขั้นตอนที่ 136) ร่วมกับ Paragraph พิเศษ
  ชื่อ `xxxx-EXIT` ที่มีแค่คำสั่ง `EXIT.` (ดูตัวอย่างเต็มในขั้นตอนที่ 140) เพื่อทำเครื่องหมายจุดจบ
  ของแต่ละกลุ่ม Paragraph ให้ชัดเจน

### แบบฝึกหัดที่ 139.1

**โจทย์**: จงอธิบายว่าทำไมปัญหานี้จึงไม่เกิดขึ้นกับ `PERFORM paragraph-name` (ที่เรียนในขั้นตอนที่
131) แต่เกิดขึ้นกับการไหลจากบนลงล่างแบบธรรมชาติของโปรแกรม

**เฉลย**: `PERFORM paragraph-name` มีกลไกพิเศษที่**จดจำตำแหน่งที่เรียกไว้** และเมื่อ Paragraph
ที่ถูกเรียกทำงานจบ (ไม่ว่าจะจบด้วยการจบ Paragraph ตามธรรมชาติหรือจบไฟล์) COBOL จะ**ย้อนกลับมาที่
ตำแหน่งเดิมโดยอัตโนมัติเสมอ** ต่างจากการไหลของโปรแกรมแบบปกติจากบนลงล่าง (ไม่ผ่าน `PERFORM`) ที่ไม่มี
กลไกการ "จดจำและย้อนกลับ" ใด ๆ เลย เมื่อ Paragraph หนึ่งจบลงโดยไม่มี `STOP RUN`, `GO TO`, หรือคำสั่ง
ควบคุมอื่นใด การควบคุมจึงไหลต่อไปยัง Paragraph ถัดไปในไฟล์ตามธรรมชาติเสมอ

---

## ขั้นตอนที่ 140: แนวทางปฏิบัติที่ดีในการจัดโครงสร้างโปรแกรมขนาดใหญ่

### แนวคิด

ปิดท้าย Part นี้ด้วยการรวบรวมทุกเทคนิคที่เรียนมา (Paragraph, Section, `PERFORM...THRU`, Numbered
Convention, การป้องกัน fall-through) เข้าเป็นแนวทางปฏิบัติที่แนะนำสำหรับโปรแกรม COBOL ขนาดใหญ่:

1. ใช้ **Numbered Paragraph Convention** (ขั้นตอนที่ 136) แบ่งเป็นกลุ่ม 0000/1000/2000/.../9000
2. จัดกลุ่ม Paragraph ที่เกี่ยวข้องกันไว้ใน **Section** เดียวกัน (ขั้นตอนที่ 133)
3. ทุกกลุ่ม (Section) ควรมี Paragraph พิเศษชื่อ `xxxx-EXIT` ที่มีแค่คำสั่ง `EXIT.` ไว้ท้ายกลุ่มเสมอ
   เพื่อเป็น "จุดสิ้นสุดที่ชัดเจน" สำหรับใช้กับ `PERFORM ... THRU xxxx-EXIT` — วิธีนี้ทำให้ถึงแม้จะมี
   คนแทรก Paragraph ใหม่เข้ามาในกลุ่มภายหลัง ขอบเขตของ `PERFORM ... THRU` ก็ยังคงถูกต้อง เพราะอ้างอิง
   ถึง Paragraph พิเศษที่ทำหน้าที่ "หมุดหมาย" ไว้อย่างชัดเจนแทนชื่อ Paragraph ทั่วไป
4. `0000-MAIN-PROCESS` มีหน้าที่แค่เรียก Paragraph อื่นตามลำดับ ไม่ควรมี logic ซับซ้อนอยู่ในตัวเอง
5. Paragraph หลักต้องมี `STOP RUN.` ชัดเจนเสมอ เพื่อป้องกันปัญหา fall-through (ขั้นตอนที่ 139)
6. หลีกเลี่ยง `GO TO` ในโค้ดใหม่ทั้งหมด ใช้ `PERFORM`/`IF`/`EVALUATE` แทน (ขั้นตอนที่ 138)

### ตัวอย่างโค้ดสรุปรวม: โปรแกรมคำนวณยอดรวมสินค้าที่จัดโครงสร้างอย่างดี

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP140.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ITEM-COUNT       PIC 9(02) VALUE ZERO.
       01  WS-ITEM-IDX         PIC 9(02).
       01  WS-GRAND-TOTAL      PIC 9(07)V99 VALUE ZERO.
       01  WS-ITEM-PRICE-TABLE.
           05  WS-ITEM-PRICE   PIC 9(05)V99 OCCURS 3 TIMES.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS SECTION.
       0000-MAIN-PARA.
           PERFORM 1000-INITIALIZE THRU 1000-EXIT.
           PERFORM 2000-PROCESS-ITEMS THRU 2000-EXIT.
           PERFORM 9000-TERMINATE THRU 9000-EXIT.
           STOP RUN.

       1000-INITIALIZE-SECTION SECTION.
       1000-INITIALIZE.
           DISPLAY "1000: Initializing item prices...".
           MOVE 3 TO WS-ITEM-COUNT.
           MOVE 100.00 TO WS-ITEM-PRICE(1).
           MOVE 250.50 TO WS-ITEM-PRICE(2).
           MOVE 75.25  TO WS-ITEM-PRICE(3).
       1000-EXIT.
           EXIT.

       2000-PROCESS-ITEMS-SECTION SECTION.
       2000-PROCESS-ITEMS.
           DISPLAY "2000: Adding up all item prices...".
           PERFORM 2100-ADD-ONE-ITEM
                   VARYING WS-ITEM-IDX FROM 1 BY 1
                   UNTIL WS-ITEM-IDX > WS-ITEM-COUNT.
       2000-EXIT.
           EXIT.

       2100-ADD-ONE-ITEM.
           ADD WS-ITEM-PRICE(WS-ITEM-IDX) TO WS-GRAND-TOTAL.
           DISPLAY "  Added item " WS-ITEM-IDX ": "
                   WS-ITEM-PRICE(WS-ITEM-IDX).

       9000-TERMINATE-SECTION SECTION.
       9000-TERMINATE.
           DISPLAY "9000: Grand total = " WS-GRAND-TOTAL.
       9000-EXIT.
           EXIT.
```

### ผลลัพธ์จริงที่ได้

```
1000: Initializing item prices...
2000: Adding up all item prices...
  Added item 01: 00100.00
  Added item 02: 00250.50
  Added item 03: 00075.25
9000: Grand total = 0000425.75
```

### อธิบายโค้ดทีละส่วน

- โปรแกรมนี้แบ่งเป็น 4 Section ชัดเจน: Main, Initialize, Process, Terminate — อ่าน
  `0000-MAIN-PARA` เพียงอย่างเดียวก็เข้าใจภาพรวมของทั้งโปรแกรมได้ทันที
- `1000-EXIT.`, `2000-EXIT.`, `9000-EXIT.` เป็น Paragraph พิเศษที่มีแค่คำสั่ง `EXIT.` (คำสั่งที่ไม่ทำ
  อะไรเลย ใช้เป็น "หมุดหมาย" เท่านั้น) ทำให้ `PERFORM 1000-INITIALIZE THRU 1000-EXIT.` มีขอบเขตที่
  ชัดเจนแน่นอน แม้จะมีการแทรก Paragraph ใหม่เข้าไประหว่างทางในอนาคต (เช่น `1000-VALIDATE`) ตราบใดที่
  ยังคงอยู่ก่อน `1000-EXIT.` ขอบเขตของ `PERFORM ... THRU` ก็ยังคงถูกต้องเสมอ
- `2100-ADD-ONE-ITEM` เป็น Paragraph ย่อยที่ถูกเรียกด้วย `PERFORM ... VARYING` (เทคนิคจากขั้นตอนที่
  137) ผสานเข้ากับการเข้าถึงตาราง (`OCCURS`, เทคนิคจากขั้นตอนที่ 126) แสดงให้เห็นว่าเทคนิคทั้งหมด
  ที่เรียนมาใน Part 013-014 สามารถทำงานร่วมกันได้อย่างเป็นธรรมชาติ

### ข้อควรระวัง

- การใส่ `EXIT.` เปล่า ๆ ไว้ท้ายกลุ่มอาจดูเหมือนไม่มีประโยชน์ในโปรแกรมเล็ก ๆ แต่จะเห็นคุณค่าชัดเจน
  มากเมื่อโปรแกรมเติบโตขึ้นและมีทีมงานหลายคนมาแก้ไขโค้ดร่วมกันในระยะยาว ซึ่งเป็นสถานการณ์ปกติของ
  ระบบ COBOL ในองค์กรจริงที่อาจถูกดูแลต่อเนื่องนานหลายสิบปี (ทบทวนเนื้อหา Part 001 เรื่องอายุของ
  ระบบ Legacy)
- อย่าลืมว่า `EXIT` (ไม่มีคำอื่นตามหลัง) ใน COBOL มาตรฐานเป็นเพียง "no-operation statement"
  ไม่ใช่คำสั่งออกจากลูปแบบ `EXIT PERFORM` (ขั้นตอนที่ 127) หรือออกจากโปรแกรมแบบ `STOP RUN` — ทั้งสาม
  คำมีความหมายต่างกันโดยสิ้นเชิงแม้จะใช้คำว่า "EXIT" ร่วมกัน ต้องแยกแยะให้ถูกต้อง

### แบบฝึกหัดที่ 140.1

**โจทย์**: ปรับโปรแกรมให้รองรับสินค้า 5 รายการแทน 3 รายการ (เพิ่มราคาสินค้าใหม่ 2 รายการเอง)

**เฉลย**: แก้ `OCCURS 3 TIMES` เป็น `OCCURS 5 TIMES`, แก้ `MOVE 3 TO WS-ITEM-COUNT.` เป็น
`MOVE 5 TO WS-ITEM-COUNT.` แล้วเพิ่มการกำหนดราคาอีก 2 รายการใน `1000-INITIALIZE`:

```cobol
       01  WS-ITEM-PRICE-TABLE.
           05  WS-ITEM-PRICE   PIC 9(05)V99 OCCURS 5 TIMES.
       ...
       1000-INITIALIZE.
           DISPLAY "1000: Initializing item prices...".
           MOVE 5 TO WS-ITEM-COUNT.
           MOVE 100.00 TO WS-ITEM-PRICE(1).
           MOVE 250.50 TO WS-ITEM-PRICE(2).
           MOVE 75.25  TO WS-ITEM-PRICE(3).
           MOVE 40.00  TO WS-ITEM-PRICE(4).
           MOVE 199.99 TO WS-ITEM-PRICE(5).
       1000-EXIT.
           EXIT.
```

ส่วน `2000-PROCESS-ITEMS` และ `2100-ADD-ONE-ITEM` ไม่ต้องแก้ไขเลย เพราะใช้ `WS-ITEM-COUNT` เป็น
ขอบเขตของลูปอยู่แล้ว นี่คือประโยชน์ของการเขียนโค้ดแบบยืดหยุ่นที่ไม่ hard-code จำนวนรอบตายตัว

---

## สรุปท้ายบท

ใน Part นี้ คุณได้เรียนรู้:

- Paragraph คืออะไร และวิธีเรียกใช้ด้วย `PERFORM paragraph-name`
- การเรียก Paragraph ซ้ำด้วย `TIMES` และ `UNTIL`
- Section คือหน่วยจัดกลุ่ม Paragraph หลายตัว และการเรียกทั้ง Section ด้วย `PERFORM section-name`
- `PERFORM ... THRU` สำหรับรันช่วงของ Paragraph ตามลำดับในไฟล์ พร้อมความเสี่ยงเมื่อมีการแทรกโค้ดใหม่
- การเรียก Paragraph ซ้อนกันหลายชั้น (chained PERFORM) และข้อจำกัดสำคัญเรื่อง Recursion ใน COBOL
- ธรรมเนียมการตั้งชื่อ Paragraph แบบมีโครงสร้าง (Numbered Paragraph Convention: 0000/1000/.../9000)
- การผสาน `PERFORM VARYING` กับการเรียก Paragraph เพื่อแยกโครงลูปออกจากเนื้อหาการทำงาน
- ทำไม COBOL สมัยใหม่เลี่ยงการใช้ `GO TO` และใช้ `EVALUATE`/`PERFORM`/`IF` แทน
- กับดัก Fall-through ที่อันตรายที่สุดอย่างหนึ่งของ COBOL และวิธีป้องกันด้วยการใส่ `STOP RUN` ให้ครบ
- แนวทางปฏิบัติที่ดีในการจัดโครงสร้างโปรแกรมขนาดใหญ่ด้วย Section, Numbered Convention และ
  Paragraph พิเศษ `xxxx-EXIT`

ตอนนี้คุณมีเครื่องมือครบทั้งหมดที่จำเป็นแล้ว: ตัวแปรและชนิดข้อมูล (Part 005-006), การรับ-แสดงผล
(Part 007), การย้ายข้อมูล (Part 008), เลขคณิต (Part 009), เงื่อนไข (Part 010-011), การวนลูป
(Part 012-013), และการจัดโครงสร้างโปรแกรม (Part 014 นี้) ถึงเวลาแล้วที่จะนำทุกอย่างมารวมกันเป็น
**โปรเจกต์รวบยอดเฟส 1** ใน Part 015: การสร้างเครื่องคิดเลขและระบบคำนวณเกรดนักเรียนที่ใช้งานได้จริง!

**[← กลับไป Part 013: PERFORM VARYING และการวนลูปหลายชั้น](part-013-perform-varying.md)** | **[ไปยัง Part 015: โปรเจกต์เฟส 1 →](part-015-phase1-project.md)**
