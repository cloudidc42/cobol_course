# Part 032: การส่งพารามิเตอร์: BY REFERENCE, BY CONTENT, BY VALUE (ขั้นตอนที่ 311–320)

## คำนำของ Part นี้

Part 031 แนะนำ `CALL` Statement และการส่งพารามิเตอร์เบื้องต้นผ่าน `LINKAGE SECTION` ไปแล้ว แต่จงใจ
ข้ามรายละเอียดสำคัญข้อหนึ่งไป: **เมื่อโปรแกรมย่อยแก้ไขค่าพารามิเตอร์ที่รับมา ทำไมค่านั้นถึงเปลี่ยน
กลับไปที่โปรแกรมหลักด้วยเสมอ?** คำตอบคือ Part 031 ใช้วิธีการส่งพารามิเตอร์แบบ **ค่าเริ่มต้น
(Default)** ของ COBOL ที่เรียกว่า **`BY REFERENCE`** โดยไม่ได้เขียนกำกับไว้ให้เห็นชัด ๆ

Part นี้จะเจาะลึกทั้ง **3 วิธีการส่งพารามิเตอร์**ที่ COBOL มีให้ใช้อย่างเป็นทางการ:

- **`BY REFERENCE`**: ส่ง "ที่อยู่หน่วยความจำจริง" ไปให้ — โปรแกรมย่อยแก้ไขค่าต้นฉบับของผู้เรียก
  ได้โดยตรง (ค่าเริ่มต้นที่ Part 031 ใช้มาตลอด)
- **`BY CONTENT`**: ส่ง "สำเนาชั่วคราวของค่า" ไปให้ — โปรแกรมย่อยแก้ไขได้แค่สำเนาของตัวเอง ต้นฉบับ
  ของผู้เรียกปลอดภัย 100%
- **`BY VALUE`**: คล้าย `BY CONTENT` แต่ใช้กลไกที่เข้ากันได้กับการเรียกโค้ดที่ไม่ใช่ COBOL (เช่น
  ฟังก์ชันภาษา C) เหมาะกับตัวเลขไบนารีล้วน ๆ

การเลือกวิธีส่งพารามิเตอร์ที่ถูกต้องคือทักษะสำคัญที่แยกโปรแกรมเมอร์ COBOL มืออาชีพออกจากมือใหม่
เพราะเลือกผิดอาจทำให้ข้อมูลสำคัญในระบบถูกแก้ไขโดยไม่ได้ตั้งใจ — ทุกตัวอย่างในนี้ผ่านการทดสอบจริง
เพื่อพิสูจน์ความแตกต่างของทั้ง 3 วิธีอย่างเป็นรูปธรรม ไม่ใช่แค่คำอธิบายทางทฤษฎี

> **หมายเหตุการคอมไพล์**: เช่นเดียวกับ Part 031 ทุกตัวอย่างต้องใช้สองไฟล์ต้นฉบับขึ้นไป (โปรแกรมหลัก
> และโปรแกรมย่อย) คอมไพล์รวมกันด้วย `cobc -x main.cob sub.cob -o program`

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่าง
> (รวมถึงตัวอย่างที่ตั้งใจแสดง runtime crash จริงและพฤติกรรมผิดพลาดแบบเงียบ) ผ่านการคอมไพล์และรัน
> ทดสอบจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 311: BY REFERENCE แบบชัดเจน — ทบทวนค่าเริ่มต้นที่ใช้มาตลอด Part 031

### เขียน BY REFERENCE ให้เห็นชัด ๆ

Part 031 ทุกตัวอย่างใช้ `CALL "..." USING ตัวแปร` เฉย ๆ โดยไม่ระบุวิธีส่ง ซึ่ง COBOL จะถือว่าเป็น
`BY REFERENCE` โดยอัตโนมัติเสมอ ขั้นตอนนี้จะเขียนคำว่า `BY REFERENCE` ให้เห็นชัดเจน เพื่อเตรียมเทียบ
กับ `BY CONTENT` ในขั้นตอนถัดไป

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP311MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-BALANCE                PIC 9(7)V99 VALUE 500.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: before CALL, balance = " WS-BALANCE.
      *> BY REFERENCE is the DEFAULT for every CALL in Part 031 -
      *> writing it explicitly here changes nothing, but makes the
      *> intent visible: the subprogram gets the ACTUAL ADDRESS of
      *> WS-BALANCE, not a copy, so it can change the caller's data
      *> directly.
           CALL "STEP311SUB" USING BY REFERENCE WS-BALANCE.
           DISPLAY "Main: after CALL,  balance = " WS-BALANCE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP311SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-BALANCE                PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-BALANCE.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: received balance = " LK-BALANCE.
           ADD 250.00 TO LK-BALANCE.
           DISPLAY "  Sub: after ADD, balance = " LK-BALANCE.
           GOBACK.
```

```bash
cobc -x step311main.cob step311sub.cob -o step311
./step311
```

**ผลลัพธ์:**

```
Main: before CALL, balance = 0000500.00
  Sub: received balance = 0000500.00
  Sub: after ADD, balance = 0000750.00
Main: after CALL,  balance = 0000750.00
```

### อธิบายจุดสำคัญ

- `LK-BALANCE` ในโปรแกรมย่อย**ไม่ใช่หน่วยความจำของตัวเอง** แต่เป็นเพียง "ป้ายชื่ออีกชื่อหนึ่ง"
  ที่ชี้ไปยังหน่วยความจำเดียวกันกับ `WS-BALANCE` ในโปรแกรมหลัก การ `ADD 250.00 TO LK-BALANCE`
  จึงเป็นการแก้ไขหน่วยความจำก้อนเดียวกันนั้นโดยตรง ไม่ใช่การแก้ไขสำเนา
- นี่คือเหตุผลที่ `BY REFERENCE` เร็วและประหยัดหน่วยความจำที่สุดในบรรดาทั้ง 3 วิธี เพราะไม่ต้อง
  คัดลอกข้อมูลใด ๆ เลย เพียงส่ง "ที่อยู่" (Address) ไปเท่านั้น ไม่ว่าข้อมูลจะมีขนาดใหญ่แค่ไหนก็ตาม
- ข้อดีนี้แลกมาด้วยความเสี่ยง: โปรแกรมย่อยใด ๆ ที่ได้รับพารามิเตอร์ `BY REFERENCE` มีอำนาจแก้ไขข้อมูล
  ต้นฉบับของผู้เรียกได้เต็มที่ ไม่ว่าจะตั้งใจหรือไม่ก็ตาม (จะพิสูจน์ปัญหาที่เกิดจากเรื่องนี้ในขั้นตอน
  ที่ 319)

### ข้อควรระวัง

- `BY REFERENCE` เหมาะที่สุดสำหรับพารามิเตอร์ที่ **ตั้งใจให้โปรแกรมย่อยแก้ไขจริง** (เช่น ค่าที่ใช้
  "ส่งผลลัพธ์กลับ" อย่าง `WS-TOTAL` ใน Part 031 ขั้นตอน 303) ไม่ใช่ค่าเริ่มต้นที่ควรใช้แบบไม่คิด
  สำหรับทุกพารามิเตอร์
- ต้องมั่นใจเสมอว่าโปรแกรมย่อยที่เรียกด้วย `BY REFERENCE` **น่าเชื่อถือและเข้าใจตรงกัน**ว่าจะแก้ไข
  หรือไม่แก้ไขค่านั้น เพราะผลกระทบสะท้อนกลับมาที่ผู้เรียกทันทีโดยไม่มีการแจ้งเตือนใด ๆ

### แบบฝึกหัดที่ 311.1

**โจทย์**: จงอธิบายว่าทำไม `BY REFERENCE` จึงเป็นวิธีที่ **"เร็วที่สุด"** ในบรรดาทั้ง 3 วิธีการส่ง
พารามิเตอร์ของ COBOL

**เฉลย**: เพราะ `BY REFERENCE` ส่งเพียง **ที่อยู่หน่วยความจำ (Memory Address)** ของตัวแปรไปให้
โปรแกรมย่อยเท่านั้น ไม่ว่าตัวแปรต้นฉบับจะมีขนาดใหญ่แค่ไหน (เช่น record ขนาดหลายพันไบต์) ที่อยู่
หน่วยความจำก็ยังคงมีขนาดคงที่เสมอ (โดยทั่วไป 4 หรือ 8 ไบต์บนระบบสมัยใหม่) ต่างจาก `BY CONTENT`
และ `BY VALUE` ที่ต้อง**คัดลอกข้อมูลทั้งก้อน**ไปไว้ในหน่วยความจำใหม่ก่อนส่ง ยิ่งข้อมูลมีขนาดใหญ่
เท่าไร การคัดลอกก็ยิ่งใช้เวลาและหน่วยความจำมากขึ้นตามไปด้วย

---

## ขั้นตอนที่ 312: BY CONTENT — ส่งสำเนา ปกป้องข้อมูลต้นฉบับ

### ทดลอง: โปรแกรมย่อยเดียวกัน เปลี่ยนแค่วิธีส่ง

ใช้โปรแกรมย่อยที่**เหมือนกันทุกประการ**กับขั้นตอนที่ 311 แต่เปลี่ยนวิธีส่งจาก `BY REFERENCE` เป็น
**`BY CONTENT`**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP312MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-BALANCE                PIC 9(7)V99 VALUE 500.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: before CALL, balance = " WS-BALANCE.
      *> BY CONTENT sends a TEMPORARY COPY of WS-BALANCE's current
      *> value - the subprogram can change its own copy all it
      *> wants, but WS-BALANCE in the caller is fully protected.
           CALL "STEP312SUB" USING BY CONTENT WS-BALANCE.
           DISPLAY "Main: after CALL,  balance = " WS-BALANCE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP312SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-BALANCE                PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-BALANCE.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: received balance = " LK-BALANCE.
           ADD 250.00 TO LK-BALANCE.
           DISPLAY "  Sub: after ADD, balance = " LK-BALANCE.
           GOBACK.
```

**ผลลัพธ์:**

```
Main: before CALL, balance = 0000500.00
  Sub: received balance = 0000500.00
  Sub: after ADD, balance = 0000750.00
Main: after CALL,  balance = 0000500.00
```

### วิเคราะห์ความแตกต่างที่สำคัญที่สุดของ Part นี้

เปรียบเทียบบรรทัดสุดท้ายของทั้งสองขั้นตอน: ขั้นตอนที่ 311 (`BY REFERENCE`) ได้ `0000750.00` แต่
ขั้นตอนนี้ (`BY CONTENT`) ยังคงเป็น **`0000500.00` เหมือนเดิมทุกประการ** ทั้งที่โปรแกรมย่อยแสดงผลว่า
"after ADD" กลายเป็น `0000750.00` เหมือนกัน! นี่คือหลักฐานที่ชัดเจนที่สุดว่า **`LK-BALANCE` ในกรณี
`BY CONTENT` เป็นเพียง "สำเนาชั่วคราว" ที่ถูกสร้างขึ้นมาใหม่สำหรับการ `CALL` ครั้งนี้เท่านั้น**
การแก้ไขสำเนานั้นไม่มีทางย้อนกลับไปกระทบต้นฉบับ (`WS-BALANCE`) ได้เลย

### อธิบายจุดสำคัญ

- โปรแกรมย่อย `STEP312SUB` **เหมือนกับ `STEP311SUB` ทุกตัวอักษร** — พิสูจน์ว่าพฤติกรรมการปกป้อง
  ข้อมูลนี้**ไม่ได้ขึ้นอยู่กับโค้ดในโปรแกรมย่อยเลย** แต่ขึ้นอยู่กับ**ตัวเลือกของผู้เรียก**ที่ระบุใน
  `CALL ... USING` เท่านั้น
- สำเนาที่สร้างขึ้นจาก `BY CONTENT` จะถูกทำลายทิ้งทันทีที่โปรแกรมย่อยจบการทำงาน (`GOBACK`) ไม่มี
  ทางเข้าถึงสำเนานั้นได้อีกจากที่ใดเลย
- `BY CONTENT` เหมาะมากสำหรับพารามิเตอร์ที่ต้องการให้โปรแกรมย่อย**อ่านได้อย่างเดียว** (Read-Only)
  โดยไม่ต้องกังวลว่าโปรแกรมย่อยจะไปแก้ไขข้อมูลต้นฉบับโดยไม่ได้ตั้งใจ

### ข้อควรระวัง

- `BY CONTENT` มีต้นทุนด้าน performance สูงกว่า `BY REFERENCE` เพราะต้องคัดลอกข้อมูลทุกครั้งที่
  `CALL` — สำหรับข้อมูลขนาดใหญ่มาก (เช่น record หลายพันไบต์) ควรพิจารณาผลกระทบด้านประสิทธิภาพหาก
  ต้อง `CALL` บ่อยครั้งในลูปขนาดใหญ่
- อย่าลืมว่า `BY CONTENT` ปกป้อง**เฉพาะพารามิเตอร์ที่ระบุ `BY CONTENT` เท่านั้น** ถ้ามีหลาย
  พารามิเตอร์ใน `CALL` เดียวกัน แต่ละตัวต้องระบุวิธีส่งของตัวเองแยกกัน (จะสาธิตในขั้นตอนที่ 315)

### แบบฝึกหัดที่ 312.1

**โจทย์**: จงอธิบายว่าทำไมค่า `LK-BALANCE` ในสาขา `"after ADD"` ของโปรแกรมย่อยจึงแสดงผลเป็น
`0000750.00` เหมือนกันทั้งในขั้นตอนที่ 311 และ 312 ทั้งที่ผลลัพธ์สุดท้ายที่ฝั่งโปรแกรมหลักต่างกัน

**เฉลย**: เพราะไม่ว่าจะส่งด้วยวิธีใด **โปรแกรมย่อยเองมองไม่เห็นความแตกต่างเลยแม้แต่น้อย** —
`LK-BALANCE` ในมุมมองของโปรแกรมย่อยคือหน่วยความจำก้อนหนึ่งที่มีค่าเริ่มต้น `500.00` เสมอ และการ
`ADD 250.00` ก็ทำให้กลายเป็น `750.00` เหมือนกันทุกครั้งไม่ว่าจะถูกเรียกด้วยวิธีใด ความแตกต่างอยู่ที่
**หน่วยความจำก้อนนั้นคือก้อนเดียวกับ `WS-BALANCE` ของผู้เรียกหรือไม่**: ถ้าเป็น `BY REFERENCE`
มันคือก้อนเดียวกัน การเปลี่ยนแปลงจึงสะท้อนกลับ แต่ถ้าเป็น `BY CONTENT` มันคือสำเนาที่แยกออกมาต่างหาก
การเปลี่ยนแปลงในสำเนาจึงไม่มีทางย้อนกลับไปถึงต้นฉบับได้เลย

---

## ขั้นตอนที่ 313: เปรียบเทียบแบบเคียงข้างกันในโปรแกรมเดียว

### พิสูจน์ให้เห็นชัดในรันเดียว

ขั้นตอนนี้รวมทั้งสองวิธีไว้ในโปรแกรมเดียว โดยเรียก**โปรแกรมย่อยตัวเดียวกัน**สองครั้งด้วยวิธีส่งที่
ต่างกัน เพื่อพิสูจน์ให้เห็นชัดเจนที่สุดว่าความแตกต่างอยู่ที่ผู้เรียก ไม่ใช่ผู้ถูกเรียก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP313MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-REF-BALANCE            PIC 9(7)V99 VALUE 500.00.
       01  WS-CONTENT-BALANCE        PIC 9(7)V99 VALUE 500.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Both start at 500.00.".

           CALL "STEP313SUB" USING BY REFERENCE WS-REF-BALANCE.
           CALL "STEP313SUB" USING BY CONTENT WS-CONTENT-BALANCE.

           DISPLAY "After the SAME subprogram ran on both:".
           DISPLAY "  BY REFERENCE result -> " WS-REF-BALANCE.
           DISPLAY "  BY CONTENT  result -> " WS-CONTENT-BALANCE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP313SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-BALANCE                PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-BALANCE.
       SUB-MAIN-PARA.
      *> This ONE subprogram never knows or cares HOW it was
      *> called - it just adds 250.00 to whatever memory it was
      *> given. Whether that change is visible to the caller
      *> afterward depends ENTIRELY on the CALLER's choice of
      *> BY REFERENCE vs BY CONTENT, not on anything written here.
           ADD 250.00 TO LK-BALANCE.
           GOBACK.
```

**ผลลัพธ์:**

```
Both start at 500.00.
After the SAME subprogram ran on both:
  BY REFERENCE result -> 0000750.00
  BY CONTENT  result -> 0000500.00
```

### อธิบายจุดสำคัญ

- โปรแกรมย่อย `STEP313SUB` **ถูกเรียกใช้แค่ไฟล์เดียว ไม่มีเวอร์ชันแยกสำหรับแต่ละวิธี** — นี่คือ
  หัวใจสำคัญที่สุดของบทเรียนนี้: **การเลือกวิธีส่งพารามิเตอร์เป็นความรับผิดชอบของผู้เรียก (Caller)
  ทั้งหมด ไม่ใช่ผู้ถูกเรียก (Callee)**
- โปรแกรมย่อยที่ดีไม่ควร "สันนิษฐาน" ว่าตัวเองจะถูกเรียกด้วยวิธีใดวิธีหนึ่งเสมอ เพราะไม่มีทางรู้
  หรือควบคุมได้เลยจากฝั่งของตัวเอง (ยกเว้นกรณี `BY VALUE` ที่ขั้นตอนที่ 314 จะแสดงว่าโปรแกรมย่อย
  ต้องประกาศรับรู้ด้วย)
- ผลลัพธ์นี้ตอบคำถามที่ Part 031 ทิ้งค้างไว้: "ทำไมการแก้ไขพารามิเตอร์ในโปรแกรมย่อยถึงสะท้อนกลับมา
  ที่ผู้เรียก" — คำตอบคือเพราะ Part 031 **ไม่เคยระบุวิธีส่ง** จึงใช้ค่าเริ่มต้น `BY REFERENCE`
  เสมอ

### ข้อควรระวัง

- อย่าออกแบบโปรแกรมย่อยที่ผลลัพธ์ถูกต้องได้เฉพาะเมื่อถูกเรียกด้วยวิธีใดวิธีหนึ่งเท่านั้น (เช่น
  โปรแกรมย่อยที่ "ต้อง" ถูกเรียกด้วย `BY REFERENCE` เท่านั้นจึงจะทำงานถูกต้อง) ควรมีเอกสารกำกับ
  ชัดเจนว่าโปรแกรมย่อยแต่ละตัวคาดหวังให้เรียกด้วยวิธีใด สำหรับพารามิเตอร์แต่ละตัว
- ทีมพัฒนาที่ดีมักตั้งชื่อพารามิเตอร์หรือเขียนคอมเมนต์กำกับไว้ในโปรแกรมย่อยว่าคาดหวังให้ถูกเรียก
  ด้วยวิธีใด (เช่น คำนำหน้า `IN-`, `OUT-`, `INOUT-` สำหรับพารามิเตอร์ที่เป็น input อย่างเดียว,
  output อย่างเดียว, หรือทั้งสองอย่าง)

### แบบฝึกหัดที่ 313.1

**โจทย์**: จงออกแบบชื่อพารามิเตอร์ในตัวอย่างนี้ใหม่ให้สื่อความหมายชัดเจนขึ้นว่าตัวไหนคาดหวังให้เรียก
ด้วย `BY REFERENCE` (เพื่อรับผลลัพธ์กลับ) และตัวไหนเหมาะกับ `BY CONTENT` (อ่านอย่างเดียว)

**เฉลย**: เนื่องจากในตัวอย่างนี้ `LK-BALANCE` ถูกใช้เป็น "ค่าที่ต้องอัปเดต" (in-out) จึงควรตั้งชื่อ
ว่า `LK-INOUT-BALANCE` เพื่อสื่อว่าโปรแกรมย่อยนี้ถูกออกแบบมาให้**คาดหวัง** `BY REFERENCE` เสมอ
(ไม่ใช่ `BY CONTENT` ที่จะทำให้การอัปเดตไม่มีความหมายใด ๆ เลยดังที่เห็นในผลลัพธ์) — การตั้งชื่อ
แบบนี้ช่วยเตือนโปรแกรมเมอร์ที่มาเรียกใช้ในภายหลังว่าไม่ควรใช้ `BY CONTENT` กับพารามิเตอร์ตัวนี้
เพราะจะทำให้ผลลัพธ์การคำนวณหายไปโดยไม่รู้ตัว

---

## ขั้นตอนที่ 314: BY VALUE — เหมาะกับตัวเลขไบนารีและการเชื่อมต่อโค้ดภายนอก

### BY VALUE ต่างจาก BY CONTENT อย่างไร

`BY VALUE` คล้ายกับ `BY CONTENT` ตรงที่ปกป้องข้อมูลต้นฉบับของผู้เรียกเหมือนกัน แต่ใช้กลไกการส่งข้อมูล
ที่**เข้ากันได้กับมาตรฐานการเรียกฟังก์ชันของภาษา C** (Calling Convention) ซึ่งเป็นสิ่งจำเป็นเมื่อต้อง
เชื่อมต่อ COBOL กับโค้ดภาษาอื่น (Part 072 จะสอนการเชื่อมต่อ COBOL กับ C อย่างเต็มรูปแบบ) ข้อแตกต่าง
สำคัญคือ **`BY VALUE` ทำงานได้น่าเชื่อถือที่สุดกับข้อมูลตัวเลขแบบไบนารี (`COMP-5`) เท่านั้น และ
โปรแกรมย่อยต้องประกาศ `BY VALUE` รับไว้ด้วยเช่นกัน** (ต่างจาก `BY CONTENT`/`BY REFERENCE` ที่
โปรแกรมย่อยไม่ต้องประกาศอะไรเป็นพิเศษ)

### ทดลอง: ลืมประกาศ BY VALUE ฝั่งโปรแกรมย่อย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP314BADMAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-QUANTITY               PIC 9(3) COMP-5 VALUE 10.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: before CALL, quantity = " WS-QUANTITY.
      *> The subprogram's PROCEDURE DIVISION USING does NOT say
      *> BY VALUE, so it expects a REFERENCE (an address). We are
      *> sending a raw VALUE instead - the two sides disagree on
      *> the calling convention.
           CALL "STEP314BADSUB" USING BY VALUE WS-QUANTITY.
           DISPLAY "Main: after CALL (never reached).".
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP314BADSUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-QUANTITY               PIC 9(3) COMP-5.

      *> Missing "BY VALUE" here - this line silently expects the
      *> DEFAULT calling convention (BY REFERENCE), mismatching
      *> what the caller actually sent.
       PROCEDURE DIVISION USING LK-QUANTITY.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: received quantity = " LK-QUANTITY.
           GOBACK.
```

**ผลลัพธ์จริง (ยืนยันด้วย GnuCOBOL):**

```
Main: before CALL, quantity = 00010

attempt to reference unallocated memory (signal SIGSEGV)
abnormal termination - file contents may be incorrect
```

โปรแกรมล่มทันทีด้วย SIGSEGV — เหตุผลเดียวกับ Part 031 ขั้นตอนที่ 304: ทั้งสองฝั่ง**ไม่ตกลงกัน**ว่า
จะสื่อสารกันด้วยกลไกแบบไหน (ค่าจริง หรือ ที่อยู่หน่วยความจำ) ทำให้เกิดการอ่านหน่วยความจำผิดตำแหน่ง

### แก้ไข: ประกาศ BY VALUE ให้ตรงกันทั้งสองฝั่ง พร้อมใช้ COMP-5

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP314GOODMAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-QUANTITY               PIC 9(3) COMP-5 VALUE 10.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: before CALL, quantity = " WS-QUANTITY.
      *> BY VALUE protects the caller's data like BY CONTENT, but
      *> uses the calling convention most compatible with non-COBOL
      *> code (such as calling a C function - Part 072 covers this
      *> in depth). It works reliably with binary (COMP-5) numeric
      *> items, and BOTH sides must declare BY VALUE - see step
      *> 314bad for what happens when the subprogram forgets to.
           CALL "STEP314GOODSUB" USING BY VALUE WS-QUANTITY.
           DISPLAY "Main: after CALL,  quantity = " WS-QUANTITY.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP314GOODSUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-QUANTITY               PIC 9(3) COMP-5.

      *> BY VALUE declared here too, matching the caller exactly.
       PROCEDURE DIVISION USING BY VALUE LK-QUANTITY.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: received quantity = " LK-QUANTITY.
           ADD 5 TO LK-QUANTITY.
           DISPLAY "  Sub: after ADD, quantity = " LK-QUANTITY.
           GOBACK.
```

**ผลลัพธ์หลังแก้ไข:**

```
Main: before CALL, quantity = 00010
  Sub: received quantity = 00010
  Sub: after ADD, quantity = 00015
Main: after CALL,  quantity = 00010
```

### อธิบายจุดสำคัญ

- สังเกตว่า `WS-QUANTITY` (`PIC 9(3) COMP-5`) แสดงผลเป็น `00010` (5 หลัก) ไม่ใช่ `010` (3 หลัก)
  ตามที่ `PICTURE` ประกาศไว้ — นี่คือพฤติกรรมจริงของ GnuCOBOL ที่ยืนยันจากการทดสอบ: **ฟิลด์
  `COMP-5` (ไบนารีล้วน) จะ `DISPLAY` ด้วยความกว้างเต็มของ "ขนาดหน่วยเก็บข้อมูล" ที่ compiler
  เลือกใช้ภายใน (เช่น 2 ไบต์ รองรับได้ถึง 5 หลัก) ไม่ใช่ความกว้างตาม `PICTURE` ที่ประกาศไว้**
  ดังนั้นในทางปฏิบัติควร `MOVE` ค่าจากฟิลด์ `COMP-5` ไปยังฟิลด์ `DISPLAY` ธรรมดาก่อนแสดงผลให้คนอ่าน
  เสมอ (จะสาธิตเทคนิคนี้ในขั้นตอนที่ 320)
- `BY VALUE` ยืนยันแล้วว่าปกป้องข้อมูลต้นฉบับได้เหมือน `BY CONTENT`: `WS-QUANTITY` ยังคงเป็น
  `00010` แม้โปรแกรมย่อยจะ `ADD 5` ให้กลายเป็น `00015` ในฝั่งของตัวเองก็ตาม

### ข้อควรระวัง

- **กฎทองของ `BY VALUE`**: ต้องประกาศ `BY VALUE` ทั้งฝั่งผู้เรียก (`CALL ... USING BY VALUE`) และ
  ฝั่งผู้ถูกเรียก (`PROCEDURE DIVISION USING BY VALUE`) ให้ตรงกันเสมอ มิฉะนั้นจะเกิด runtime crash
  ทันทีตามที่พิสูจน์ข้างต้น
- ควรใช้ `BY VALUE` กับฟิลด์ `USAGE COMP-5` (หรือ `BINARY`/`COMP`) เท่านั้นเพื่อความน่าเชื่อถือ
  การใช้กับฟิลด์ `DISPLAY` ธรรมดา (เช่น `PIC 9(3)` ไม่มี `COMP-5`) อาจให้ผลลัพธ์ที่ไม่ถูกต้อง
  เนื่องจากรูปแบบการจัดเก็บข้อมูลภายในไม่ตรงกับที่กลไก `BY VALUE` คาดหวัง

### แบบฝึกหัดที่ 314.1

**โจทย์**: จงอธิบายว่าทำไมการแสดงผล `WS-QUANTITY` (`PIC 9(3) COMP-5`) จึงได้ `00010` (5 หลัก)
แทนที่จะเป็น `010` (3 หลักตาม PICTURE)

**เฉลย**: เพราะ `COMP-5` เป็นชนิดข้อมูลไบนารีที่ GnuCOBOL จัดสรรพื้นที่จัดเก็บตาม**ขนาดหน่วยความจำ
มาตรฐานของระบบ** (เช่น 2 ไบต์, 4 ไบต์) ไม่ใช่ตามจำนวนหลักที่ `PICTURE` ระบุไว้เป๊ะ ๆ เหมือนฟิลด์
`DISPLAY` ทั่วไป จำนวนหลัก `9(3)` (สูงสุด 999) ยังพอดีกับพื้นที่เก็บข้อมูลขนาด 2 ไบต์ (ซึ่งรองรับค่า
ได้ถึง 65,535 หรือ 5 หลัก) เมื่อ `DISPLAY` ตรง ๆ จึงเห็นความกว้างเต็มของพื้นที่จัดเก็บ (5 หลัก)
แทนที่จะเห็นแค่ความกว้างตาม `PICTURE` ที่ประกาศไว้ (3 หลัก) — นี่คือเหตุผลที่แนะนำให้ `MOVE` ค่าจาก
ฟิลด์ `COMP-5` ไปยังฟิลด์ `DISPLAY` ธรรมดาก่อนแสดงผลให้ผู้ใช้เห็นเสมอ

---

## ขั้นตอนที่ 315: ผสมทั้ง 3 วิธีในคำสั่ง CALL เดียวกัน

### แต่ละพารามิเตอร์เลือกวิธีของตัวเองได้อิสระ

`CALL` หนึ่งคำสั่งสามารถมีพารามิเตอร์หลายตัวที่**แต่ละตัวใช้วิธีส่งต่างกันได้อย่างอิสระ** เลือกตาม
หน้าที่ของพารามิเตอร์ตัวนั้น ๆ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP315MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUST-NAME              PIC X(15) VALUE "SOMCHAI JAIDEE".
       01  WS-ORIGINAL-BALANCE       PIC 9(7)V99 VALUE 500.00.
       01  WS-NEW-BALANCE            PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Before: name=" WS-CUST-NAME
               " original=" WS-ORIGINAL-BALANCE
               " new=" WS-NEW-BALANCE.

      *> Each parameter can use a DIFFERENT passing method in the
      *> SAME CALL statement - chosen based on what each one is
      *> FOR: read-only display data (BY CONTENT, protect it),
      *> a snapshot that must not change (BY CONTENT again), and
      *> an output parameter that MUST change (BY REFERENCE).
           CALL "STEP315SUB" USING
               BY CONTENT   WS-CUST-NAME
               BY CONTENT   WS-ORIGINAL-BALANCE
               BY REFERENCE WS-NEW-BALANCE.

           DISPLAY "After:  name=" WS-CUST-NAME
               " original=" WS-ORIGINAL-BALANCE
               " new=" WS-NEW-BALANCE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP315SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-CUST-NAME              PIC X(15).
       01  LK-ORIGINAL-BALANCE       PIC 9(7)V99.
       01  LK-NEW-BALANCE            PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-CUST-NAME LK-ORIGINAL-BALANCE
               LK-NEW-BALANCE.
       SUB-MAIN-PARA.
      *> This subprogram tries to change ALL THREE parameters, but
      *> only the effect on LK-NEW-BALANCE will be visible to the
      *> caller afterward - the other two were sent BY CONTENT.
           MOVE "IMPOSTOR NAME  " TO LK-CUST-NAME.
           ADD 999.99 TO LK-ORIGINAL-BALANCE.
           COMPUTE LK-NEW-BALANCE = LK-ORIGINAL-BALANCE + 250.00.
           DISPLAY "  Sub sees -> name=" LK-CUST-NAME
               " original=" LK-ORIGINAL-BALANCE
               " new=" LK-NEW-BALANCE.
           GOBACK.
```

**ผลลัพธ์:**

```
Before: name=SOMCHAI JAIDEE  original=0000500.00 new=0000000.00
  Sub sees -> name=IMPOSTOR NAME   original=0001499.99 new=0001749.99
After:  name=SOMCHAI JAIDEE  original=0000500.00 new=0001749.99
```

### อธิบายจุดสำคัญ

- โปรแกรมย่อยพยายามแก้ไข**ทั้ง 3 พารามิเตอร์**อย่างเต็มที่ (เปลี่ยนชื่อ, บวกเงินเข้ายอดเดิม,
  คำนวณยอดใหม่) แต่ผลลัพธ์ที่ฝั่งผู้เรียกเห็นหลัง `CALL` กลับมามีแค่ **`WS-NEW-BALANCE` เท่านั้น**
  ที่เปลี่ยนแปลง — `WS-CUST-NAME` และ `WS-ORIGINAL-BALANCE` ยังคงค่าเดิมเป๊ะ เพราะถูกส่งด้วย
  `BY CONTENT`
- สังเกตว่า `LK-NEW-BALANCE` ที่คำนวณได้ (`1749.99`) มาจาก `LK-ORIGINAL-BALANCE` **ที่ถูกบวก
  เพิ่มไปแล้วในสำเนาของโปรแกรมย่อยเอง** (`500.00 + 999.99 = 1499.99` แล้วบวกอีก `250.00`) แสดงให้
  เห็นว่าสำเนาจาก `BY CONTENT` ยังคงทำงานได้ตามปกติทุกประการภายในโปรแกรมย่อย เพียงแต่ไม่มีทาง
  ย้อนกลับไปกระทบต้นฉบับได้เท่านั้น
- นี่คือรูปแบบการออกแบบที่**ใช้งานจริงมากในระบบธุรกิจ**: ส่งข้อมูลอ้างอิง/แสดงผลแบบ `BY CONTENT`
  (ป้องกันการแก้ไขโดยไม่ตั้งใจ) พร้อมกับพารามิเตอร์ผลลัพธ์แบบ `BY REFERENCE` (ต้องการให้แก้ไขจริง)
  ในคำสั่งเดียวกัน

### ข้อควรระวัง

- ความสามารถในการผสมวิธีส่งนี้ทำให้การอ่านโค้ด `CALL` **สำคัญมาก** — ต้องอ่านทุกบรรทัดของ
  `USING` อย่างละเอียดว่าพารามิเตอร์ตัวไหนใช้วิธีอะไร ไม่ควรสันนิษฐานว่าทั้งหมดใช้วิธีเดียวกัน
- การจัดรูปแบบโค้ดให้แต่ละพารามิเตอร์อยู่คนละบรรทัด (เหมือนตัวอย่างนี้) ช่วยให้อ่านและตรวจสอบทาน
  ได้ง่ายกว่าการเขียนทุกอย่างในบรรทัดเดียวยาว ๆ มาก

### แบบฝึกหัดที่ 315.1

**โจทย์**: จงอธิบายว่าถ้าเปลี่ยน `BY CONTENT WS-ORIGINAL-BALANCE` เป็น `BY REFERENCE
WS-ORIGINAL-BALANCE` (โดยไม่แก้ไขอย่างอื่นเลย) ผลลัพธ์บรรทัด `"After: ..."` จะเปลี่ยนไปอย่างไร

**เฉลย**: `WS-ORIGINAL-BALANCE` จะเปลี่ยนจาก `0000500.00` (ค่าเดิม) กลายเป็น `0001499.99`
(ค่าที่โปรแกรมย่อยคำนวณได้จาก `ADD 999.99 TO LK-ORIGINAL-BALANCE`) เพราะตอนนี้ `LK-ORIGINAL-
BALANCE` จะชี้ไปยังหน่วยความจำเดียวกับ `WS-ORIGINAL-BALANCE` โดยตรง (`BY REFERENCE`) การแก้ไขใน
โปรแกรมย่อยจึงสะท้อนกลับมาทันที ส่วน `WS-NEW-BALANCE` ยังคงเป็น `0001749.99` เหมือนเดิม (คำนวณจาก
`LK-ORIGINAL-BALANCE` ที่ค่า `1499.99` บวก `250.00`) ไม่เปลี่ยนแปลงจากเดิม

---

## ขั้นตอนที่ 316: ส่ง Literal ตรง ๆ ใน CALL — ข้อแตกต่างระหว่าง Literal ตัวอักษรกับตัวเลข

### Literal ก็ส่งเป็นพารามิเตอร์ได้โดยไม่ต้องเก็บในตัวแปรก่อน

COBOL อนุญาตให้ส่ง literal (ค่าคงที่ที่เขียนตรง ๆ ในโค้ด) เป็นพารามิเตอร์ใน `CALL ... USING` ได้เลย
โดยไม่ต้องเก็บไว้ในตัวแปรก่อน แต่มีรายละเอียดสำคัญที่ต่างกันระหว่าง **literal ตัวอักษร** (มีเครื่องหมาย
คำพูดคร่อม) กับ **literal ตัวเลข** (ไม่มีเครื่องหมายคำพูด) ที่ต้องเข้าใจให้ถูกต้อง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP316MAIN.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Case 1: a QUOTED (alphanumeric) literal passed straight
      *> into CALL USING - this works correctly with the ordinary
      *> BY REFERENCE default, because GnuCOBOL creates temporary
      *> storage holding the exact bytes of the literal.
           DISPLAY "Case 1: alphanumeric literal".
           CALL "STEP316TEXTSUB" USING "BANGKOK".

      *> Case 2: an UNQUOTED numeric literal passed to a receiver
      *> declared as an ordinary DISPLAY-format PIC 9 field with NO
      *> "BY VALUE" - this compiles cleanly but silently receives
      *> GARBAGE (blank), not the number 100. No crash, no warning -
      *> just wrong data. This is confirmed by testing, and it is
      *> more dangerous than step 314's crash precisely because it
      *> fails silently.
           DISPLAY "Case 2: numeric literal, WRONG way".
           CALL "STEP316NUMBADSUB" USING 100.

      *> Case 3: the SAME numeric literal, this time BY VALUE into
      *> a COMP-5 (binary) receiver that also declares BY VALUE -
      *> this is the combination confirmed to work correctly.
           DISPLAY "Case 3: numeric literal, RIGHT way".
           CALL "STEP316NUMGOODSUB" USING BY VALUE 100.

           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP316TEXTSUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-CITY                   PIC X(7).

       PROCEDURE DIVISION USING LK-CITY.
       SUB-MAIN-PARA.
           DISPLAY "  Sub received city = [" LK-CITY "]".
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP316NUMBADSUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-NUM                    PIC 9(3).

       PROCEDURE DIVISION USING LK-NUM.
       SUB-MAIN-PARA.
           DISPLAY "  Sub received number = [" LK-NUM "]".
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP316NUMGOODSUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-NUM                    PIC 9(3) COMP-5.

       PROCEDURE DIVISION USING BY VALUE LK-NUM.
       SUB-MAIN-PARA.
           DISPLAY "  Sub received number = [" LK-NUM "]".
           GOBACK.
```

**ผลลัพธ์ (ยืนยันด้วยการรันจริง):**

```
Case 1: alphanumeric literal
  Sub received city = [BANGKOK]
Case 2: numeric literal, WRONG way
  Sub received number = []
Case 3: numeric literal, RIGHT way
  Sub received number = [00100]
```

### วิเคราะห์ผลลัพธ์ที่สำคัญ

**Case 2 คือกับดักที่อันตรายที่สุดของขั้นตอนนี้**: การส่ง literal ตัวเลข `100` (ไม่มีเครื่องหมาย
คำพูด) แบบธรรมดา (ไม่ระบุ `BY VALUE`) ไปยังฟิลด์ `PIC 9(3)` แบบ `DISPLAY` ธรรมดา **compile ผ่านได้
ปกติทุกประการ ไม่มี warning ใด ๆ เลย แต่รันแล้วได้ค่าว่างเปล่า** ไม่ใช่ `100` ตามที่คาดหวัง — และที่
ร้ายกว่าขั้นตอนที่ 314 (ที่อย่างน้อยยังมีการล่มให้เห็นชัด ๆ) คือ**กรณีนี้ไม่มีการล่มเลย โปรแกรมทำงาน
ต่อไปได้ตามปกติด้วยข้อมูลที่ผิดพลาดแบบเงียบ ๆ**

### อธิบายจุดสำคัญ

- **Literal ตัวอักษร** (มีเครื่องหมายคำพูด เช่น `"BANGKOK"`) ทำงานได้ปกติกับการส่งแบบ default
  (`BY REFERENCE`) เพราะ GnuCOBOL จัดสรรพื้นที่หน่วยความจำชั่วคราวเก็บไบต์ของข้อความนั้นให้โดย
  อัตโนมัติ แล้วส่งที่อยู่ของพื้นที่นั้นไปให้ราวกับเป็นตัวแปรจริง
- **Literal ตัวเลข** (ไม่มีเครื่องหมายคำพูด เช่น `100`) ที่ปลอดภัยที่สุดคือส่งด้วย **`BY VALUE`**
  ไปยังฟิลด์รับที่เป็น `COMP-5` (ไบนารี) เท่านั้น ตามที่พิสูจน์ใน Case 3 — ตรงกับที่ขั้นตอนที่ 314
  อธิบายไว้ว่า `BY VALUE` ถูกออกแบบมาให้เข้ากันกับตัวเลขไบนารีเป็นหลัก
- กฎปฏิบัติที่ปลอดภัย: **ใช้ literal ตัวอักษรได้อย่างอิสระในเกือบทุกวิธีส่ง แต่ใช้ literal ตัวเลข
  กับ `BY VALUE` + `COMP-5` เท่านั้น**

### ข้อควรระวัง

- ความผิดพลาดแบบ Case 2 (ไม่ล่ม แค่ได้ค่าผิด) **อันตรายกว่าความผิดพลาดแบบ Case 2 ของขั้นตอนที่
  314 มาก** เพราะไม่มีสัญญาณเตือนใด ๆ เลยที่บอกว่าเกิดปัญหาขึ้น หากไม่ได้ทดสอบผลลัพธ์อย่างละเอียด
  อาจปล่อยบั๊กนี้เข้าสู่ระบบจริงได้โดยไม่รู้ตัว
- ควรหลีกเลี่ยงการส่ง literal ตัวเลขตรง ๆ ใน `CALL` เมื่อเป็นไปได้ ให้เก็บไว้ในตัวแปร `COMP-5` ที่
  ตั้งชื่อสื่อความหมายก่อนเสมอ (เหมือนที่ทำมาตลอดในขั้นตอนที่ 311–315) เพื่อลดความเสี่ยงและทำให้
  โค้ดอ่านง่ายขึ้นด้วย

### แบบฝึกหัดที่ 316.1

**โจทย์**: จงอธิบายว่าทำไมการส่ง literal ตัวอักษร (Case 1) จึงไม่มีปัญหาเหมือน literal ตัวเลข
(Case 2) ทั้งที่ทั้งคู่เป็น literal เหมือนกัน

**เฉลย**: เพราะ literal ตัวอักษร (Alphanumeric Literal) มีการจัดเก็บข้อมูลภายในของ COBOL ที่ตรงไป
ตรงมา คือ**ไบต์ต่อไบต์ตามที่เขียนไว้ในเครื่องหมายคำพูดเป๊ะ** ไม่มีการแปลงรูปแบบใด ๆ เพิ่มเติม ทำให้
การจัดสรรพื้นที่ชั่วคราวและส่งที่อยู่ไปทำงานได้อย่างตรงไปตรงมา ในขณะที่ literal ตัวเลข (Numeric
Literal) มีความซับซ้อนเรื่องการแทนค่าภายในมากกว่า (ต้องตัดสินใจว่าจะแทนด้วยรูปแบบ `DISPLAY`
(zoned decimal) หรือรูปแบบไบนารี และด้วยขนาดกี่ไบต์) ซึ่งการตัดสินใจนี้อาจไม่ตรงกับรูปแบบที่ฟิลด์
ผู้รับ (`LINKAGE SECTION`) คาดหวังไว้ หากไม่ระบุกลไกการส่งที่ชัดเจนและตรงกัน (`BY VALUE` กับ
`COMP-5`) จึงเกิดความไม่ตรงกันของรูปแบบข้อมูลขึ้นได้ง่ายกว่ามาก

---

## ขั้นตอนที่ 317: การใช้งานจริง — BY CONTENT เพื่อปกป้องข้อมูลจากโค้ดที่ไม่น่าไว้ใจ

### สถานการณ์จริง: ส่งข้อมูลไปบันทึก Log โดยไม่ไว้ใจโค้ดนั้นสมบูรณ์แบบ

ในระบบองค์กรขนาดใหญ่ โปรแกรมมักต้องเรียกใช้โปรแกรมย่อยหรือ Utility ที่เขียนโดยทีมอื่น (หรือเป็น
โค้ดเก่าที่ไม่มีใครกล้าแก้ไข) `BY CONTENT` คือเครื่องมือป้องกันตัวเองที่ดีที่สุดในสถานการณ์แบบนี้:
**แม้โปรแกรมย่อยนั้นจะมีบั๊กที่แก้ไขพารามิเตอร์ของตัวเองโดยไม่ได้ตั้งใจ ข้อมูลต้นฉบับของเราก็ยัง
ปลอดภัย 100%**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP317MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LIVE-BALANCE           PIC 9(7)V99 VALUE 1234.56.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Live balance before logging: "
               WS-LIVE-BALANCE.

      *> STEP317AUDITLOG is a THIRD-PARTY-style utility we do not
      *> fully control or trust to be bug-free. Passing its input
      *> BY CONTENT is a defensive design choice: even if that
      *> subprogram has a bug that changes its parameter, our real
      *> account balance is safe no matter what it does internally.
           CALL "STEP317AUDITLOG" USING BY CONTENT WS-LIVE-BALANCE.

           DISPLAY "Live balance after logging:  "
               WS-LIVE-BALANCE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP317AUDITLOG.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ROUNDED-FOR-DISPLAY    PIC 9(7).

       LINKAGE SECTION.
       01  LK-BALANCE                PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-BALANCE.
       SUB-MAIN-PARA.
      *> This subprogram carelessly ROUNDS DOWN its own parameter
      *> to write a whole-number line to the audit log - a real
      *> bug that throws away the cents. Because the caller used
      *> BY CONTENT, this mistake stays completely contained here.
           DIVIDE LK-BALANCE BY 1 GIVING WS-ROUNDED-FOR-DISPLAY.
           MOVE WS-ROUNDED-FOR-DISPLAY TO LK-BALANCE.
           DISPLAY "  [AUDIT LOG] balance (rounded) = "
               LK-BALANCE.
           GOBACK.
```

**ผลลัพธ์:**

```
Live balance before logging: 0001234.56
  [AUDIT LOG] balance (rounded) = 0001234.00
Live balance after logging:  0001234.56
```

### อธิบายจุดสำคัญ

- `STEP317AUDITLOG` มีบั๊กจริง ๆ: มันทำลายทศนิยม (`.56` หายไปกลายเป็น `.00`) ในสำเนาของตัวเองก่อน
  บันทึก log แต่เพราะโปรแกรมหลักเลือกใช้ `BY CONTENT` **ยอดเงินจริงของลูกค้า (`WS-LIVE-BALANCE`)
  จึงไม่ได้รับผลกระทบใด ๆ เลย** ยังคงเป็น `1234.56` ที่ถูกต้องเป๊ะหลัง `CALL` กลับมา
- นี่คือแนวคิด **Defensive Programming** (การเขียนโปรแกรมเชิงป้องกัน) ที่สำคัญมากในระบบที่ต้องพึ่งพา
  โค้ดจากภายนอกหรือทีมอื่น: **เมื่อไม่แน่ใจว่าโปรแกรมย่อยจะทำอะไรกับพารามิเตอร์บ้าง ให้ส่งแบบ
  `BY CONTENT` ไว้ก่อนเป็นค่าเริ่มต้นที่ปลอดภัย** แล้วเปลี่ยนเป็น `BY REFERENCE` เฉพาะเมื่อต้องการ
  ผลลัพธ์กลับจริง ๆ และมั่นใจในพฤติกรรมของโปรแกรมย่อยนั้นแล้วเท่านั้น
- ตัวอย่างนี้แสดงให้เห็นคุณค่าของ `BY CONTENT` ที่มากกว่าแค่ "ทฤษฎี" — มันคือเครื่องมือป้องกันบั๊ก
  ในโค้ดของคนอื่นไม่ให้ลามมาถึงข้อมูลสำคัญของเรา

### ข้อควรระวัง

- `BY CONTENT` ป้องกันได้เฉพาะการแก้ไข**ค่าของพารามิเตอร์**เท่านั้น ไม่ได้ป้องกันผลข้างเคียงอื่น ๆ
  ที่โปรแกรมย่อยอาจทำ เช่น การเขียนไฟล์ผิดพลาด หรือการเรียก `STOP RUN` โดยไม่ตั้งใจ (ทบทวนจาก
  Part 031 ขั้นตอน 306) — `BY CONTENT` แก้ปัญหาได้เฉพาะเรื่อง "ข้อมูลถูกแก้ไขโดยไม่ได้ตั้งใจ"
  เท่านั้น
- แม้ `BY CONTENT` จะปลอดภัยกว่า แต่ก็ไม่ควรใช้พร่ำเพรื่อกับพารามิเตอร์ทุกตัวโดยไม่คิด เพราะมีต้นทุน
  ด้าน performance สูงกว่า `BY REFERENCE` เสมอ (ตามที่อธิบายในขั้นตอนที่ 312)

### แบบฝึกหัดที่ 317.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `BY CONTENT` ในสถานการณ์นี้จึงเรียกว่าเป็นแนวทาง **"Defensive
Programming"**

**เฉลย**: เพราะแนวคิด Defensive Programming คือการเขียนโค้ดที่**ป้องกันความเสียหายล่วงหน้า แม้จะ
ไม่แน่ใจ 100% ว่าโค้ดส่วนอื่นจะทำงานถูกต้องหรือไม่ก็ตาม** ในสถานการณ์นี้ โปรแกรมหลักไม่ได้ตรวจสอบ
หรือไว้ใจว่า `STEP317AUDITLOG` เขียนถูกต้องปราศจากบั๊ก แต่เลือกใช้ `BY CONTENT` เป็นเกราะป้องกันไว้
ล่วงหน้า ทำให้ไม่ว่าโปรแกรมย่อยนั้นจะมีบั๊กร้ายแรงแค่ไหนในการจัดการพารามิเตอร์ของตัวเอง ข้อมูลสำคัญ
ของระบบ (`WS-LIVE-BALANCE`) ก็ยังคงปลอดภัยเสมอ นี่คือการป้องกันปัญหาที่ต้นเหตุด้วยการเลือกกลไกทาง
ภาษาที่เหมาะสม แทนที่จะหวังพึ่งการทดสอบโปรแกรมย่อยให้ครบทุกกรณีเพียงอย่างเดียว

---

## ขั้นตอนที่ 318: BY VALUE ในบริบทของการเชื่อมต่อโค้ดแบบ C — ตัวอย่างจำลอง

### จำลองรูปแบบการเรียกฟังก์ชันสไตล์ C

แม้ Part 072 จะสอนการเชื่อมต่อ COBOL กับภาษา C อย่างเต็มรูปแบบ แต่ขั้นตอนนี้จะแสดง**รูปแบบการเรียก**
ที่ใช้เมื่อ COBOL ต้องคุยกับโค้ดแบบ C: พารามิเตอร์ input ธรรมดา (ตัวเลข) ส่งด้วย `BY VALUE`
(ตรงกับที่ภาษา C ส่ง `int` เป็นค่าเสมอ) ส่วนพารามิเตอร์ output (ผลลัพธ์ที่ต้องเขียนกลับ) ยังคงใช้
`BY REFERENCE` (ตรงกับที่ภาษา C รับ pointer สำหรับค่าที่ฟังก์ชันต้องเขียนกลับ)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP318MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-WIDTH                  PIC 9(4) COMP-5 VALUE 12.
       01  WS-HEIGHT                 PIC 9(4) COMP-5 VALUE 8.
       01  WS-AREA                   PIC 9(8) COMP-5 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Width=" WS-WIDTH " Height=" WS-HEIGHT.
      *> This mirrors the calling shape used when linking to a C
      *> function (Part 072 covers real C interop): plain input
      *> numbers go BY VALUE, matching how C passes int arguments,
      *> while a result the callee must fill in still goes BY
      *> REFERENCE, matching how C receives an output pointer.
           CALL "STEP318AREACALC" USING
               BY VALUE     WS-WIDTH
               BY VALUE     WS-HEIGHT
               BY REFERENCE WS-AREA.
           DISPLAY "Area = " WS-AREA.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP318AREACALC.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-WIDTH                  PIC 9(4) COMP-5.
       01  LK-HEIGHT                 PIC 9(4) COMP-5.
       01  LK-AREA                   PIC 9(8) COMP-5.

       PROCEDURE DIVISION USING
               BY VALUE     LK-WIDTH
               BY VALUE     LK-HEIGHT
               BY REFERENCE LK-AREA.
       SUB-MAIN-PARA.
           COMPUTE LK-AREA = LK-WIDTH * LK-HEIGHT.
           GOBACK.
```

**ผลลัพธ์:**

```
Width=00012 Height=00008
Area = 0000000096
```

### อธิบายจุดสำคัญ

- `12 * 8 = 96` คำนวณถูกต้อง แม้ `LK-WIDTH` และ `LK-HEIGHT` จะเป็นเพียง**ค่าที่ส่งมาแบบอ่านอย่าง
  เดียว** (`BY VALUE`) ก็ยังนำมาใช้คำนวณได้ตามปกติทุกประการ เพราะการคำนวณไม่ได้ต้องการแก้ไขค่า
  ต้นฉบับของ `WS-WIDTH`/`WS-HEIGHT` เลย
- `LK-AREA` (`BY REFERENCE`) คือพารามิเตอร์เดียวที่ต้อง**เขียนผลลัพธ์กลับ** จึงเป็นตัวเดียวที่ใช้
  `BY REFERENCE` — รูปแบบผสมนี้ (input เป็น `BY VALUE`, output เป็น `BY REFERENCE`) คือรูปแบบ
  มาตรฐานที่ใช้เมื่อเขียน COBOL ให้เรียกใช้ไลบรารีภาษา C จริง ๆ
- ทั้ง `LK-WIDTH`, `LK-HEIGHT`, `LK-AREA` ล้วนเป็น `COMP-5` (ไบนารี) ทั้งหมด ซึ่งตรงกับที่อธิบายไว้
  ในขั้นตอนที่ 314–316 ว่า `BY VALUE` ทำงานน่าเชื่อถือที่สุดกับชนิดข้อมูลนี้

### ข้อควรระวัง

- ตัวอย่างนี้เป็นเพียง**การจำลองรูปแบบการเรียก**ด้วยโปรแกรม COBOL ล้วน ๆ ทั้งสองฝั่ง ยังไม่ใช่การ
  เชื่อมต่อกับโค้ด C จริง (ซึ่งต้องอาศัยเทคนิคเพิ่มเติมเรื่องการคอมไพล์และ link ข้ามภาษา จะสอนเต็ม
  รูปแบบใน Part 072)
- เมื่อเชื่อมต่อกับ C จริง ต้องระวังเรื่องขนาดของชนิดข้อมูลให้ตรงกันเป๊ะ (เช่น `PIC 9(8) COMP-5`
  ของ COBOL ต้องตรงกับ `int` หรือ `long` ของ C ตามขนาดไบต์จริงบนแพลตฟอร์มนั้น ๆ) ความไม่ตรงกันแม้
  เพียงเล็กน้อยอาจทำให้เกิดปัญหาเดียวกับที่พิสูจน์ในขั้นตอนที่ 304 และ 314

### แบบฝึกหัดที่ 318.1

**โจทย์**: จงอธิบายว่าทำไมพารามิเตอร์ `LK-AREA` ในตัวอย่างนี้จึงไม่สามารถใช้ `BY VALUE` แทน
`BY REFERENCE` ได้ ทั้งที่พารามิเตอร์อื่นใช้ `BY VALUE` ได้ตามปกติ

**เฉลย**: เพราะ `LK-AREA` มีหน้าที่เป็น**ผลลัพธ์ที่ต้องส่งกลับไปให้ผู้เรียก** (`WS-AREA` ในโปรแกรม
หลักต้องได้รับค่า `96` หลัง `CALL` จบ) ในขณะที่ `BY VALUE` (เหมือน `BY CONTENT`) ทำงานด้วยการส่ง
**สำเนา**ของข้อมูลเท่านั้น การแก้ไขสำเนานั้นในโปรแกรมย่อยจะไม่มีทางย้อนกลับไปถึงต้นฉบับได้เลย
(ตามที่พิสูจน์ในขั้นตอนที่ 312, 314) ถ้าเปลี่ยน `LK-AREA` เป็น `BY VALUE` โปรแกรมหลักจะยังคงเห็น
`WS-AREA` เป็น `0` เหมือนเดิมตลอดไป แม้โปรแกรมย่อยจะคำนวณค่า `96` ได้ถูกต้องในฝั่งของตัวเองก็ตาม
พารามิเตอร์ที่ต้อง "ส่งค่ากลับ" จึงต้องใช้ `BY REFERENCE` เท่านั้น

---

## ขั้นตอนที่ 319: กับดักจากการลืมใช้ BY CONTENT — บั๊กทางธุรกิจที่เกิดขึ้นจริง

### สถานการณ์: โปรแกรมพิมพ์ใบเสร็จที่ไปแก้ไขราคาต้นฉบับโดยไม่ตั้งใจ

นี่คือตัวอย่างบั๊กที่**เกิดขึ้นจริงในระบบธุรกิจ**เมื่อโปรแกรมเมอร์ลืมพิจารณาเรื่องวิธีส่งพารามิเตอร์
ให้รอบคอบ: โปรแกรมย่อยที่ควรมีหน้าที่แค่ "แสดงราคาที่มีส่วนลดบนใบเสร็จ" กลับไปแก้ไขราคาต้นฉบับ
(Master Price) อย่างถาวรโดยไม่ได้ตั้งใจ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP319BADMAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MASTER-PRICE           PIC 9(5)V99 VALUE 1000.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Master price before receipt print: "
               WS-MASTER-PRICE.
      *> BUG: the intent here is just to PRINT a discounted price
      *> on a receipt - WS-MASTER-PRICE should stay untouched. But
      *> CALL without BY CONTENT defaults to BY REFERENCE, so any
      *> change the subprogram makes leaks back into the master
      *> price permanently.
           CALL "STEP319PRINTRECEIPT" USING WS-MASTER-PRICE.
           DISPLAY "Master price after receipt print:  "
               WS-MASTER-PRICE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP319PRINTRECEIPT.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-PRICE                  PIC 9(5)V99.

       PROCEDURE DIVISION USING LK-PRICE.
       SUB-MAIN-PARA.
      *> Applies a 10 percent discount directly to the PARAMETER
      *> itself, intending it only for the printed receipt line.
      *> The programmer forgot that the caller must protect its own
      *> data with BY CONTENT - this subprogram cannot know or
      *> control how it was called.
           COMPUTE LK-PRICE = LK-PRICE * 0.90.
           DISPLAY "  Receipt line: discounted price = "
               LK-PRICE.
           GOBACK.
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน — บั๊กเกิดขึ้นจริง):**

```
Master price before receipt print: 01000.00
  Receipt line: discounted price = 00900.00
Master price after receipt print:  00900.00
```

### วิเคราะห์ความเสียหาย

สังเกตบรรทัดสุดท้าย: **`WS-MASTER-PRICE` เปลี่ยนจาก `1000.00` กลายเป็น `900.00` อย่างถาวร**
ทั้งที่เจตนาเดิมของโปรแกรมเมอร์คือแค่ "แสดงราคาที่มีส่วนลดบนใบเสร็จ" เท่านั้น ไม่ได้ตั้งใจจะแก้ไข
ราคาสินค้าจริงในระบบเลย — ในระบบธุรกิจจริง นี่คือบั๊กระดับร้ายแรงที่อาจทำให้ราคาสินค้าทั้งระบบค่อย ๆ
ลดลงเรื่อย ๆ ทุกครั้งที่มีการพิมพ์ใบเสร็จ โดยไม่มีใครสังเกตเห็นจนกว่าจะสายเกินไป

### แก้ไข: เพิ่ม BY CONTENT

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP319GOODMAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MASTER-PRICE           PIC 9(5)V99 VALUE 1000.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Master price before receipt print: "
               WS-MASTER-PRICE.
      *> Fixed: BY CONTENT sends a copy, so whatever the receipt
      *> subprogram does to its own parameter can never reach
      *> WS-MASTER-PRICE here.
           CALL "STEP319PRINTRECEIPT" USING
               BY CONTENT WS-MASTER-PRICE.
           DISPLAY "Master price after receipt print:  "
               WS-MASTER-PRICE.
           STOP RUN.
```

**ผลลัพธ์หลังแก้ไข:**

```
Master price before receipt print: 01000.00
  Receipt line: discounted price = 00900.00
Master price after receipt print:  01000.00
```

### อธิบายจุดสำคัญ

- โปรแกรมย่อย `STEP319PRINTRECEIPT` **ไม่ต้องแก้ไขโค้ดแม้แต่บรรทัดเดียว** — การแก้ไขทั้งหมดอยู่ที่
  ฝั่งโปรแกรมหลักเท่านั้น (เพิ่ม `BY CONTENT`) ตอกย้ำหลักการจากขั้นตอนที่ 313 ว่าการเลือกวิธีส่ง
  เป็นความรับผิดชอบของผู้เรียก
- บั๊กแบบนี้**อันตรายเพราะตรวจจับยากมาก**: โปรแกรมย่อยทำงาน "ถูกต้อง" ตามหน้าที่ของมันทุกประการ
  (คำนวณส่วนลดถูกต้อง แสดงผลถูกต้อง) ปัญหาอยู่ที่**การเลือกวิธีส่งพารามิเตอร์ผิด**ในฝั่งผู้เรียก
  เพียงจุดเดียว ซึ่งอาจไม่มีใครสังเกตเห็นจนกว่าจะมีคนสงสัยว่าทำไมราคาสินค้าถึงลดลงเรื่อย ๆ

### ข้อควรระวัง

- ทุกครั้งที่เขียน `CALL` ควรถามตัวเองเสมอ: **"พารามิเตอร์ตัวนี้ ฉันต้องการให้โปรแกรมย่อยแก้ไขค่า
  ต้นฉบับจริงหรือไม่?"** ถ้าคำตอบคือ "ไม่" ให้ใช้ `BY CONTENT` เสมอ ไม่ว่าโปรแกรมย่อยนั้นจะดูน่า
  เชื่อถือแค่ไหนก็ตาม
- Code Review ที่ดีควรตรวจสอบทุก `CALL` ที่ส่งพารามิเตอร์แบบ default (ไม่มี `BY` ระบุ) แล้วถามว่า
  ตั้งใจให้เป็น `BY REFERENCE` จริงหรือแค่ลืมพิจารณาเรื่องนี้ไปเฉย ๆ

### แบบฝึกหัดที่ 319.1

**โจทย์**: จงอธิบายว่าทำไมบั๊กประเภทนี้จึง**ตรวจจับได้ยากกว่า**บั๊กแบบ Segmentation Fault ที่พิสูจน์
ในขั้นตอนที่ 304 และ 314

**เฉลย**: เพราะบั๊กแบบ Segmentation Fault ทำให้โปรแกรม**ล่มทันทีอย่างชัดเจน** ผู้พัฒนาจะรู้ทันทีว่า
มีปัญหาเกิดขึ้นและต้องแก้ไข แต่บั๊กจากการเลือกวิธีส่งพารามิเตอร์ผิดแบบนี้**ไม่ทำให้โปรแกรมล่มเลย**
โปรแกรมยังคงทำงานสำเร็จ แสดงผลลัพธ์ที่ดู "สมเหตุสมผล" (ใบเสร็จแสดงราคาส่วนลดถูกต้อง) ปัญหาที่แท้จริง
(ราคาต้นฉบับถูกแก้ไขอย่างถาวร) จะไม่ปรากฏให้เห็นทันที แต่จะค่อย ๆ สะสมความเสียหายไปเรื่อย ๆ ในระบบ
จนกว่าจะมีคนสังเกตเห็นความผิดปกติของข้อมูลในภายหลัง (เช่น รายงานราคาสินค้าที่ลดลงเรื่อย ๆ อย่างผิด
ปกติ) ซึ่งกว่าจะสืบสาวหาสาเหตุกลับมาถึงจุดที่แท้จริงในโค้ดอาจใช้เวลานานมาก

---

## ขั้นตอนที่ 320: โปรแกรมรวบยอด — ผสมทั้ง 3 วิธีอย่างตั้งใจในระบบประมวลผลคำสั่งซื้อ

### เป้าหมาย

ขั้นตอนสุดท้ายของ Part นี้จำลองระบบประมวลผลคำสั่งซื้อที่ใช้ทั้ง 3 วิธีการส่งพารามิเตอร์**อย่างตั้งใจ
และมีเหตุผลรองรับแต่ละตัวชัดเจน** ในคำสั่ง `CALL` เดียวที่ถูกเรียกซ้ำในลูป:

- **`BY CONTENT`** สำหรับชื่อลูกค้า — ใช้แค่แสดงผล/บันทึก log เท่านั้น ไม่ควรถูกแก้ไข
- **`BY VALUE`** สำหรับยอดสั่งซื้อ — เป็นค่า input ล้วน ๆ เหมือนพารามิเตอร์แบบ C
- **`BY REFERENCE`** สำหรับยอดรวมสะสม — ต้องอัปเดตและพกพาค่าไปยังรอบถัดไปของลูป

**ไฟล์ที่ 1 — โปรแกรมหลัก (`step320main.cob`):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP320MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-NAME          PIC X(15).
       01  WS-ORDER-AMOUNT           PIC 9(6)V99 COMP-5.
       01  WS-RUNNING-TOTAL          PIC 9(8)V99 COMP-5 VALUE 0.
       01  WS-INDEX                  PIC 9(1).
       01  WS-PRINT-TOTAL            PIC 9(8)V99.

       01  WS-NAME-TABLE.
           05  FILLER PIC X(15) VALUE "SOMCHAI JAIDEE ".
           05  FILLER PIC X(15) VALUE "SUDA MEECHAI   ".
           05  FILLER PIC X(15) VALUE "PRASERT KAEWTA ".
       01  WS-NAME-REDEF REDEFINES WS-NAME-TABLE.
           05  WS-NAME-ENTRY OCCURS 3 TIMES PIC X(15).

       01  WS-AMOUNT-TABLE.
           05  FILLER PIC 9(6)V99 COMP-5 VALUE 500.00.
           05  FILLER PIC 9(6)V99 COMP-5 VALUE 1250.75.
           05  FILLER PIC 9(6)V99 COMP-5 VALUE 300.00.
       01  WS-AMOUNT-REDEF REDEFINES WS-AMOUNT-TABLE.
           05  WS-AMOUNT-ENTRY OCCURS 3 TIMES
                   PIC 9(6)V99 COMP-5.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> This one CALL deliberately uses all THREE passing methods
      *> together, each chosen for a specific reason:
      *>   BY CONTENT   - the customer name is shown in a log line
      *>                  only; it must never be changed by the
      *>                  logging code, no matter what that code
      *>                  does internally (step 317's lesson).
      *>   BY VALUE     - the order amount is a plain input number,
      *>                  passed the way a C-style calculation
      *>                  utility would expect it (step 318).
      *>   BY REFERENCE - the running total is an ACCUMULATOR that
      *>                  must be updated and carried into the next
      *>                  loop iteration (step 311).
           PERFORM VARYING WS-INDEX FROM 1 BY 1 UNTIL WS-INDEX > 3
               MOVE WS-NAME-ENTRY (WS-INDEX) TO WS-CUSTOMER-NAME
               MOVE WS-AMOUNT-ENTRY (WS-INDEX) TO WS-ORDER-AMOUNT
               CALL "STEP320PROCESS" USING
                   BY CONTENT   WS-CUSTOMER-NAME
                   BY VALUE     WS-ORDER-AMOUNT
                   BY REFERENCE WS-RUNNING-TOTAL
           END-PERFORM.

           MOVE WS-RUNNING-TOTAL TO WS-PRINT-TOTAL.
           DISPLAY "Final running total: " WS-PRINT-TOTAL.
           STOP RUN.
```

**ไฟล์ที่ 2 — โปรแกรมย่อย (`step320process.cob`):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP320PROCESS.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> COMP-5 (binary) fields DISPLAY with extra padding beyond
      *> their PICTURE width in this GnuCOBOL build - so, as usual
      *> practice, we MOVE them into ordinary DISPLAY-usage fields
      *> before printing anything for a human to read.
       01  WS-PRINT-AMOUNT           PIC 9(6)V99.

       LINKAGE SECTION.
       01  LK-CUSTOMER-NAME          PIC X(15).
       01  LK-ORDER-AMOUNT           PIC 9(6)V99 COMP-5.
       01  LK-RUNNING-TOTAL          PIC 9(8)V99 COMP-5.

      *> The callee only ever declares BY REFERENCE (the default,
      *> so it can be left out) or BY VALUE here - BY CONTENT is a
      *> CALLER-side choice only; from the subprogram's point of
      *> view, content and reference both simply arrive as normal
      *> addressable data.
       PROCEDURE DIVISION USING
               LK-CUSTOMER-NAME
               BY VALUE     LK-ORDER-AMOUNT
               BY REFERENCE LK-RUNNING-TOTAL.
       SUB-MAIN-PARA.
           MOVE LK-ORDER-AMOUNT TO WS-PRINT-AMOUNT.
           DISPLAY "  Processing " LK-CUSTOMER-NAME
               " amount=" WS-PRINT-AMOUNT.
           ADD LK-ORDER-AMOUNT TO LK-RUNNING-TOTAL.
           GOBACK.
```

**คอมไพล์และรัน:**

```bash
cobc -x step320main.cob step320process.cob -o step320
./step320
```

**ผลลัพธ์:**

```
  Processing SOMCHAI JAIDEE  amount=000500.00
  Processing SUDA MEECHAI    amount=001250.75
  Processing PRASERT KAEWTA  amount=000300.00
Final running total: 00002050.75
```

### อธิบายภาพรวมของโปรแกรม

- **สำคัญมาก**: สังเกตว่า `PROCEDURE DIVISION USING` ฝั่งโปรแกรมย่อย **ไม่มีคำว่า `BY CONTENT`
  เลย** สำหรับ `LK-CUSTOMER-NAME` — เพราะ GnuCOBOL (และมาตรฐาน COBOL) **ไม่อนุญาตให้ประกาศ
  `BY CONTENT` ในฝั่งผู้ถูกเรียก** (ทดสอบยืนยันแล้วว่าจะเกิด syntax error ทันทีถ้าเขียนไว้) —
  `BY CONTENT` เป็นเพียง**ทางเลือกของผู้เรียก**เท่านั้นว่าจะปกป้องข้อมูลต้นฉบับของตัวเองหรือไม่
  ฝั่งผู้ถูกเรียกไม่มีสิทธิ์รับรู้หรือกำหนดเรื่องนี้เลย เห็นได้แค่ `BY REFERENCE` (ค่าเริ่มต้น
  ที่ละไว้ได้) กับ `BY VALUE` (ต้องประกาศชัดเจนเสมอ) เท่านั้นในฝั่งนี้
- ผลรวม `500.00 + 1250.75 + 300.00 = 2050.75` ถูกต้องสมบูรณ์ พิสูจน์ว่า `WS-RUNNING-TOTAL`
  (`BY REFERENCE`) สะสมค่าข้ามทุกรอบของลูปได้อย่างถูกต้อง ในขณะที่ `WS-ORDER-AMOUNT` (`BY VALUE`)
  เปลี่ยนค่าใหม่ทุกรอบโดยไม่กระทบอะไรจากรอบก่อนหน้าเลย
- การ `MOVE LK-ORDER-AMOUNT TO WS-PRINT-AMOUNT` ก่อน `DISPLAY` คือการนำเทคนิคจากขั้นตอนที่ 314
  มาใช้จริง: **ไม่ `DISPLAY` ฟิลด์ `COMP-5` ตรง ๆ ให้ผู้ใช้เห็นเด็ดขาด** เพราะจะได้ความกว้างที่ผิด
  เพี้ยนไปจาก `PICTURE` ที่ตั้งใจไว้

### ข้อควรระวัง

- โปรแกรมนี้ใช้ `OCCURS`/`REDEFINES` (Part 016, 022) เพื่อจำลองข้อมูลตัวอย่างแบบง่าย ๆ ในระบบจริง
  ข้อมูลเหล่านี้มักมาจากไฟล์ (Part 023–030) ไม่ใช่ค่าคงที่ในโค้ด
- การตัดสินใจเลือกวิธีส่งพารามิเตอร์ทั้ง 3 แบบในโปรแกรมเดียวต้องมี**เหตุผลชัดเจนกำกับทุกจุด**
  (ตามที่คอมเมนต์ในโค้ดอธิบายไว้) ทีมพัฒนาที่ดีควรเขียนเอกสารหรือคอมเมนต์อธิบายเหตุผลเสมอ ไม่ใช่
  แค่เลือกตามความเคยชินโดยไม่คิด

### แบบฝึกหัดที่ 320.1

**โจทย์**: จงอธิบายว่าทำไม `WS-ORDER-AMOUNT` (ส่งด้วย `BY VALUE`) จึง**ไม่จำเป็นต้อง**ใช้
`BY CONTENT` แทน ทั้งที่จุดประสงค์คล้ายกัน (ป้องกันการแก้ไขต้นฉบับ)

**เฉลย**: เพราะทั้ง `BY VALUE` และ `BY CONTENT` ต่างก็ปกป้องข้อมูลต้นฉบับของผู้เรียกได้เหมือนกัน
(ทั้งคู่ส่ง "สำเนา" ไม่ใช่ "ที่อยู่จริง") ความแตกต่างอยู่ที่**กลไกภายในและความเข้ากันได้**เท่านั้น:
`BY VALUE` เลือกใช้ในตัวอย่างนี้เพราะ `WS-ORDER-AMOUNT` เป็นฟิลด์ตัวเลขไบนารี (`COMP-5`) ล้วน ๆ
ซึ่งเข้ากับ `BY VALUE` ได้อย่างเป็นธรรมชาติและตรงกับรูปแบบที่ใช้เชื่อมต่อกับโค้ดภายนอกแบบ C (ตาม
ที่แสดงในขั้นตอนที่ 318) หากเปลี่ยนเป็น `BY CONTENT` ผลลัพธ์การป้องกันข้อมูลจะเหมือนกันทุกประการ
เพียงแต่ไม่ได้สื่อความหมายเรื่อง "รูปแบบการเรียกแบบตัวเลขไบนารีล้วน ๆ" ให้ผู้อ่านโค้ดเข้าใจชัดเจน
เท่ากับการใช้ `BY VALUE` ในบริบทนี้

---

## สรุปท้ายบท

Part นี้เจาะลึกกลไกการส่งพารามิเตอร์ทั้ง 3 แบบของ COBOL อย่างครบถ้วน พร้อมพิสูจน์ความแตกต่างด้วย
การทดลองจริงทุกกรณี สรุปสิ่งที่เรียนรู้:

- **`BY REFERENCE`**: ส่งที่อยู่หน่วยความจำจริง เร็วที่สุด แต่โปรแกรมย่อยแก้ไขต้นฉบับได้โดยตรง
  (ค่าเริ่มต้นของ COBOL)
- **`BY CONTENT`**: ส่งสำเนาชั่วคราว ปกป้องต้นฉบับได้ 100% เหมาะกับพารามิเตอร์แบบอ่านอย่างเดียว
- **`BY VALUE`**: คล้าย `BY CONTENT` แต่ใช้กลไกที่เข้ากันได้กับการเรียกโค้ดแบบ C เหมาะกับตัวเลข
  ไบนารี (`COMP-5`) และต้องประกาศตรงกันทั้งสองฝั่งเสมอ
- การพิสูจน์ด้วยโปรแกรมย่อยตัวเดียวกันที่ถูกเรียกต่างวิธี ยืนยันว่า**การเลือกวิธีส่งเป็นความรับผิด
  ชอบของผู้เรียกทั้งหมด**
- กับดักร้ายแรงของ `BY VALUE`: ต้องประกาศให้ตรงกันทั้งสองฝั่ง มิฉะนั้นเกิด Segmentation Fault
- ข้อแตกต่างระหว่างการส่ง literal ตัวอักษร (ปลอดภัย) กับ literal ตัวเลข (ต้องใช้ `BY VALUE` +
  `COMP-5` เท่านั้น มิฉะนั้นได้ค่าผิดแบบเงียบ ๆ)
- การผสมทั้ง 3 วิธีในคำสั่ง `CALL` เดียวกัน แต่ละพารามิเตอร์เลือกวิธีตามหน้าที่ของตัวเอง
- การใช้ `BY CONTENT` เป็นแนวทาง Defensive Programming เพื่อป้องกันบั๊กจากโค้ดที่ไม่น่าไว้ใจ
- บั๊กทางธุรกิจจริงที่เกิดจากการลืมใช้ `BY CONTENT` และผลกระทบที่ตรวจจับได้ยาก
- โปรแกรมรวบยอดที่ผสมทั้ง 3 วิธีอย่างมีเหตุผลในระบบประมวลผลคำสั่งซื้อ

ด้วย Part 031–032 คุณได้เรียนรู้พื้นฐานการเขียนโปรแกรม COBOL แบบแยกโมดูลข้ามไฟล์อย่างครบถ้วนแล้ว
Part ถัดไป (**Part 033**) จะแนะนำ **COPY Statement และ Copybooks** — เทคนิคที่ช่วยแก้ปัญหาสำคัญ
ที่ Part 031–032 เจอมาตลอด: การต้องเขียนโครงสร้างข้อมูล (เช่น `01 WS-CUSTOMER` และ
`01 LK-CUSTOMER`) ซ้ำสองที่ในโปรแกรมหลักและโปรแกรมย่อย ซึ่งเสี่ยงต่อความผิดพลาดหากแก้ไขไม่ตรงกัน
`COPY` จะช่วยให้ทั้งสองฝั่งใช้โครงสร้างเดียวกันจากไฟล์ต้นฉบับเพียงไฟล์เดียว

**[← กลับไป Part 031](part-031-call-statement.md)** | **[ไปยัง Part 033: COPY Statement และ Copybooks →](part-033-copy-copybooks.md)**
