# Part 098: Best Practices และมาตรฐานการเขียนโค้ดระดับโลก (ขั้นตอนที่ 971–980)

## คำนำของ Part นี้

เดินทางมาเกือบสุดทางแล้ว ตลอด 97 Part ที่ผ่านมา หลักสูตรนี้แนะนำไวยากรณ์ แนวคิด และเทคนิคของ COBOL
ไปทีละชิ้นตั้งแต่ `IDENTIFICATION DIVISION` แรกสุดใน Part 003 จนถึงสถาปัตยกรรมองค์กรขนาดใหญ่ใน Part 086
และ Microservices ใน Part 087 — แต่ความรู้ทั้งหมดนั้นจะมีค่าจริงในที่ทำงานก็ต่อเมื่อถูกนำมาใช้อย่าง
**มีวินัยและสม่ำเสมอ** เท่านั้น โปรแกรมเมอร์สองคนที่รู้ไวยากรณ์เดียวกันทุกประการ สามารถเขียนโค้ดที่
ต่างกันราวฟ้ากับเหวได้ ขึ้นอยู่กับว่าใครยึดถือ "วินัยการเขียนโค้ด" (Coding Discipline) มากกว่ากัน

Part นี้ไม่มีไวยากรณ์ใหม่ให้เรียน แต่จะทำหน้าที่เป็น **"สารบัญรวมบทเรียนที่เจ็บปวดที่สุด"** ของทั้ง
หลักสูตร — เราจะรวบรวมกับดัก (pitfall) ที่หลักสูตรนี้พิสูจน์ด้วยการทดสอบจริงมาแล้วหลายสิบครั้งตลอด 97 Part
ที่ผ่านมา มาจัดเป็นมาตรฐานการเขียนโค้ดที่ทีมพัฒนา COBOL ระดับโลกใช้จริง พร้อม **Code Review Checklist**
ที่คุณสามารถพิมพ์ออกมาแปะข้างจอ หรือนำไปใช้ตรวจโค้ดของเพื่อนร่วมทีมได้ทันที

สิ่งที่คุณจะได้จาก Part นี้:

1. หลักการตั้งชื่อ (Naming Convention) ที่ใช้มาตลอดทั้งหลักสูตร พร้อมเหตุผลเบื้องหลัง
2. วินัยการเขียนคอมเมนต์ที่ดี (และกฎเหล็กเรื่องภาษาในซอร์สโค้ดที่ต้องไม่มีข้อยกเว้น)
3. การเขียนโปรแกรมเชิงป้องกัน (Defensive Programming): ตรวจสอบข้อมูลนำเข้าและขอบเขตของตาราง
4. การกำกับดูแล Copybook (Copybook Governance) ในทีมขนาดใหญ่
5. **ตาราง "กับดักยอดฮิต" (Greatest Hits)** — รวมบั๊กและพฤติกรรมแปลกของ COBOL ที่หลักสูตรนี้พิสูจน์จริง
   พร้อมอ้างอิง Part ที่พิสูจน์แต่ละเรื่องไว้
6. Code Review Checklist ที่ใช้งานได้จริง
7. หลักการเขียนโค้ดให้ดูแลรักษาง่าย (Maintainability) และแบบฝึกหัด Refactor ก่อน/หลัง

ทุกตัวอย่างโค้ดใน Part นี้ **คอมไพล์และรันได้จริงด้วย GnuCOBOL** (เวอร์ชัน 4.0-early ที่ใช้ทดสอบเนื้อหา
ทั้งหมดในหลักสูตรนี้) และในขั้นตอนที่ 976 คุณจะได้เห็น**บั๊กจริงอีกตัวหนึ่ง**ที่เกิดขึ้นระหว่างการเตรียม
เนื้อหา Part 100 (Capstone Project) — นำมาเล่าไว้ที่นี่เพื่อให้เห็นภาพว่าแม้แต่ในการเตรียมหลักสูตรระดับ
มืออาชีพก็ยังพลาดกับดักคลาสสิกได้ ถ้าไม่มีวินัยตรวจสอบให้ครบ

---

## ขั้นตอนที่ 971: หลักการตั้งชื่อ (Naming Convention) ที่ใช้มาตลอดหลักสูตร

### ทำไมชื่อตัวแปรถึงสำคัญกว่าที่คิด

โค้ดถูก**อ่าน**มากกว่าที่ถูก**เขียน** โปรแกรมเมอร์คนหนึ่งอาจเขียนโค้ดครั้งเดียว แต่คนอื่น (รวมถึงตัวเอง
ในอีก 6 เดือนข้างหน้า) จะต้องอ่านโค้ดนั้นซ้ำแล้วซ้ำอีกเพื่อแก้บั๊กหรือเพิ่มฟีเจอร์ ชื่อตัวแปรที่ดีคือ
เอกสารประกอบโค้ดที่ดีที่สุดที่ไม่ต้องเขียนแยกต่างหากเลย

ลองเปรียบเทียบโค้ดสองเวอร์ชันที่**ทำงานเหมือนกันทุกประการ**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BADNAME.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  A               PIC 9(5)V99  VALUE 0.
       01  B               PIC 9(3)     VALUE 0.
       01  C               PIC 9(7)V99  VALUE 0.
       01  D               PIC X(1)     VALUE "N".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 19.99 TO A
           MOVE 3 TO B
           COMPUTE C ROUNDED = A * B
           IF C > 50.00
               MOVE "Y" TO D
           END-IF
           DISPLAY "TOTAL=" C " FLAG=" D
           STOP RUN.
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
TOTAL=0000059.97 FLAG=Y
```

โค้ดข้างต้นคอมไพล์ผ่านและรันได้ถูกต้อง 100% — แต่ถ้าคุณเปิดไฟล์นี้ขึ้นมาใหม่ในอีก 6 เดือนข้างหน้า
โดยไม่มีบริบทใด ๆ คุณจะตอบคำถามเหล่านี้ได้ไหม: `A` คืออะไร? ทำไม `C` ต้องมากกว่า `50.00`? `D`
มีความหมายว่าอะไร? เทียบกับเวอร์ชันที่ตั้งชื่อสื่อความหมาย:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GOODNAME.
      *> Naming convention used throughout this course:
      *>   WS-  = Working-Storage data item
      *>   88   = condition-name (reads like a yes/no question)
      *> Names describe BUSINESS MEANING, not just data type.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-UNIT-PRICE        PIC 9(5)V99  VALUE 0.
       01  WS-QUANTITY          PIC 9(3)     VALUE 0.
       01  WS-EXTENDED-AMOUNT   PIC 9(7)V99  VALUE 0.
       01  WS-FREE-SHIP-FLAG    PIC X(1)     VALUE "N".
           88  WS-QUALIFIES-FOR-FREE-SHIP   VALUE "Y".
       01  WS-FREE-SHIP-THRESHOLD PIC 9(3)V99 VALUE 50.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 19.99 TO WS-UNIT-PRICE
           MOVE 3 TO WS-QUANTITY
           COMPUTE WS-EXTENDED-AMOUNT ROUNDED =
               WS-UNIT-PRICE * WS-QUANTITY
           IF WS-EXTENDED-AMOUNT > WS-FREE-SHIP-THRESHOLD
               SET WS-QUALIFIES-FOR-FREE-SHIP TO TRUE
           END-IF
           DISPLAY "TOTAL=" WS-EXTENDED-AMOUNT
               " FREE-SHIP=" WS-FREE-SHIP-FLAG
           STOP RUN.
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน (เหมือนเวอร์ชันแรกทุกประการ)**:

```
TOTAL=0000059.97 FREE-SHIP=Y
```

โค้ดสองเวอร์ชันนี้**ทำงานเหมือนกัน 100%** แต่เวอร์ชันที่สองอ่านแล้วเข้าใจทันทีโดยไม่ต้องเดา — นี่คือ
เป้าหมายของ Naming Convention: ไม่ได้ทำให้โปรแกรมรันเร็วขึ้นหรือถูกต้องขึ้นแม้แต่น้อย แต่ทำให้
**มนุษย์คนถัดไปที่อ่านโค้ดนี้เสียเวลาน้อยลง**

### กฎการตั้งชื่อที่หลักสูตรนี้ใช้ตลอดทั้ง 100 Part

| องค์ประกอบ | Prefix/รูปแบบ | ตัวอย่าง | อ้างอิง |
|---|---|---|---|
| Working-Storage ทั่วไป | `WS-` | `WS-CUSTOMER-NAME` | Part 005 |
| Linkage Section (รับส่งผ่าน CALL) | `LK-` | `LK-CUST-ID` | Part 032 |
| File Section (ระเบียนไฟล์) | ชื่อไฟล์เป็นคำนำหน้า หรือ `FD-` เมื่อ COPY ซ้ำ | `CUST-RECORD`, `FD-CUST-RECORD` | Part 024 |
| Condition-name (88-level) | อ่านออกเสียงเป็นประโยคคำถาม Yes/No | `WS-USER-WANTS-MORE` | Part 010 |
| ค่าคงที่ (Level 78) | บอกความหมายทางธุรกิจ ไม่ใช่ค่าตัวเลข | `TIER-DISCOUNT-PLATINUM` | Part 033 |
| Paragraph หลัก | เลขนำหน้าตามลำดับการทำงาน (Numbered Paragraph) | `1000-SHOW-WELCOME` | Part 014/015 |
| Subprogram (PROGRAM-ID) | คำกริยา+คำนามสั้น ๆ สื่อหน้าที่เดียว | `ORDCALC`, `CUSTREPO` | Part 031, Part 082 |

**หลักการสำคัญที่อยู่เบื้องหลังตารางนี้**: prefix ไม่ได้มีไว้ "ให้ดูเป็นทางการ" แต่มีไว้บอกทันทีว่า
**ตัวแปรนี้มาจากไหนและมีขอบเขตชีวิตแค่ไหน** — เห็น `LK-` ปุ๊บ รู้ทันทีว่าค่านี้มาจากผู้เรียกโปรแกรม
ภายนอก ไม่ใช่ค่าที่โปรแกรมนี้ประกาศเอง เห็น `WS-` ปุ๊บ รู้ว่าเป็นตัวแปรภายในโปรแกรมนี้เท่านั้น

### ข้อควรระวัง

- อย่าตั้งชื่อสั้นเกินไปเพื่อประหยัดการพิมพ์ (เช่น `A`, `TMP`, `X1`) — Part 085 ขั้นตอนที่ 842 แสดงให้
  เห็นแล้วว่าโค้ด Legacy จริงที่ใช้ชื่อแบบนี้ (`ARLEGACY.cob`) อ่านยากและแก้บั๊กยากกว่ามากเมื่อเทียบกับ
  เวอร์ชัน Refactor ที่ตั้งชื่อสื่อความหมาย (`ARDISC.cob`)
- อย่าตั้งชื่อยาวจนเกินคอลัมน์ 72 โดยไม่ระวัง — ชื่อตัวแปรที่ยาวเกินไปอาจทำให้บรรทัดที่อ้างอิงถึงมัน
  (เช่นในนิพจน์คำนวณ) เกินขอบเขตคอลัมน์ได้ง่าย ดูรายละเอียดกับดักคอลัมน์ 72 ในขั้นตอนที่ 977
- ความสม่ำเสมอสำคัญกว่าความสมบูรณ์แบบ — ทีมที่เลือก convention คนละแบบกับหลักสูตรนี้ไม่ใช่เรื่องผิด
  ตราบใดที่**ทุกคนในทีมใช้ระบบเดียวกันตลอด**ทั้งโค้ดเบส

### แบบฝึกหัดที่ 971.1

**โจทย์**: จงอธิบายว่าทำไม `88-level` ควรถูกตั้งชื่อให้ "อ่านออกเสียงเป็นประโยคคำถาม Yes/No" แทนที่จะ
ตั้งชื่อบอกแค่ค่าที่เก็บ เช่น เปรียบเทียบ `WS-USER-WANTS-MORE` กับ `WS-ANSWER-IS-Y`

**เฉลยแนวทาง**: `88-level` ถูกใช้ในเงื่อนไข `IF` เสมอ (เช่น `IF WS-USER-WANTS-MORE`) การตั้งชื่อให้
อ่านเป็นประโยคคำถามทำให้ทั้งบรรทัด `IF` อ่านเป็นภาษาอังกฤษธรรมชาติได้ทันที ("ถ้าผู้ใช้ต้องการมากกว่านี้")
ในขณะที่ `WS-ANSWER-IS-Y` บอกแค่ "รูปแบบของค่า" (เป็นตัว Y หรือไม่) ซึ่งผู้อ่านต้องแปลความหมายทางธุรกิจ
เพิ่มอีกชั้นหนึ่งว่า "Y" แปลว่าอะไรในบริบทนี้ — ยิ่งชื่อใกล้เคียงกับสิ่งที่โปรแกรมเมอร์ต้องการสื่อสารมากเท่าไร
ยิ่งลดภาระการตีความของผู้อ่านมากเท่านั้น

---

## ขั้นตอนที่ 972: วินัยการเขียนคอมเมนต์ที่ดี

### คอมเมนต์ควรอธิบาย "ทำไม" ไม่ใช่ "ทำอะไร"

โค้ดที่ดีบอก "ทำอะไร" อยู่แล้วในตัวมันเอง (ถ้าตั้งชื่อดีตามขั้นตอนที่ 971) สิ่งที่คอมเมนต์ควรเพิ่มเติมคือ
**เหตุผลเบื้องหลังการตัดสินใจ** ที่โค้ดเพียงอย่างเดียวบอกไม่ได้ ลองดูตัวอย่างจริงจาก `ORDVALID.cob`
ใน Part 100 (Capstone):

```cobol
      *> Never trust a subscript that came from outside this
      *> paragraph (a parameter, a calculation, user input) --
      *> always range-check it against the table's real bounds
      *> before subscripting. Without this check, GnuCOBOL with
      *> runtime checks off will silently read whatever memory
      *> happens to sit past the table -- garbage in, garbage out,
      *> with no error at all.
           IF WS-REQUESTED-IDX >= WS-MIN-INDEX
              AND WS-REQUESTED-IDX <= WS-MAX-INDEX
```

คอมเมนต์นี้ไม่ได้บอกว่า "ตรวจสอบว่า index อยู่ในช่วง" (อ่านจากโค้ด `IF` ก็เห็นอยู่แล้ว) แต่บอก **ทำไม
ต้องตรวจสอบ** และ **จะเกิดอะไรขึ้นถ้าไม่ตรวจสอบ** — นี่คือข้อมูลที่มีค่าที่สุดสำหรับคนอ่านโค้ดในอนาคต
โดยเฉพาะโปรแกรมเมอร์ใหม่ที่อาจคิดว่า "ทำไมต้องเขียนโค้ดเพิ่มอีก 2 บรรทัด ในเมื่อ GnuCOBOL เดี๋ยวก็
รันได้อยู่แล้ว"

### รูปแบบ Header Comment ที่ใช้ตลอดหลักสูตรนี้

ทุกโปรแกรมใน Part 100 (Capstone) ขึ้นต้นด้วย Header Comment รูปแบบเดียวกัน:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDCALC.
       AUTHOR. COBOL-COURSE.
      *> Business Logic Layer - pure calculation engine for one order.
      *> Zero file I/O, zero side effects: given line items and the
      *> customer's tier, it returns subtotal/discount/tax/total.
```

Header Comment ที่ดีตอบคำถาม 2 ข้อทันทีโดยไม่ต้องอ่านทั้งโปรแกรม: **โปรแกรมนี้อยู่ที่ชั้นไหนของ
สถาปัตยกรรม** (Business Logic / Data Access / Presentation) และ **มันทำอะไรเป็นหลัก**

### กฎเหล็กที่ไม่มีข้อยกเว้น: คอมเมนต์ในโค้ดต้องเป็นภาษาอังกฤษ/ASCII เท่านั้น

หลักสูตรนี้ยึดกฎเหล็กที่ระบุไว้ใน `docs/COURSE-OUTLINE.md` มาตั้งแต่ Part 002 อย่างเคร่งครัด:
**โค้ด COBOL ทุกส่วน รวมถึงคอมเมนต์ ต้องเป็นภาษาอังกฤษ/ASCII เท่านั้น ไม่มีข้อยกเว้น** เหตุผลไม่ใช่
เรื่องสไตล์ แต่เป็นเรื่องเทคนิคที่พิสูจน์แล้วจริงตั้งแต่ Part 002: GnuCOBOL แบบ Fixed-Format นับขอบเขต
คอลัมน์ 72 เป็น**ไบต์** ไม่ใช่ตัวอักษร ตัวอักษรที่ไม่ใช่ ASCII (เช่นภาษาไทยใน UTF-8) ใช้ 3 ไบต์ต่อ
ตัวอักษร ทำให้ข้อความที่ดูสั้นเมื่อมองด้วยตาอาจเกิน column 72 จริง ๆ และทำให้ compile error

นี่ยังสอดคล้องกับธรรมเนียมอุตสาหกรรมจริงด้วย เพราะระบบ Mainframe จำนวนมากยังใช้ Character Encoding
แบบ EBCDIC ซึ่งไม่รองรับภาษาไทยอยู่แล้ว (ทบทวนจาก Part 002 และ Part 049)

### ข้อควรระวัง

- อย่าปล่อยคอมเมนต์ที่ล้าสมัย (stale comment) ทิ้งไว้ — คอมเมนต์ที่บอกพฤติกรรมผิดจากโค้ดจริงอันตราย
  กว่าไม่มีคอมเมนต์เลย เพราะทำให้คนอ่านเข้าใจผิดโดยไม่รู้ตัว เมื่อแก้โค้ด ให้ตรวจทานคอมเมนต์ข้างเคียงด้วย
  ทุกครั้ง
- ระวังเครื่องหมายคำพูดโค้ง (curly quotes) หรือขีดยาว (em-dash) ที่อาจติดมาจากการพิมพ์ในโปรแกรม
  ประมวลผลคำบางตัว แล้ว copy-paste เข้าไฟล์ COBOL — สิ่งเหล่านี้ก็ไม่ใช่ ASCII เช่นกัน และหลุดรอด
  สายตาได้ง่าย (ดูวิธีสแกนอัตโนมัติในขั้นตอนที่ 977)

### แบบฝึกหัดที่ 972.1

**โจทย์**: คอมเมนต์ต่อไปนี้ผิดหลักการข้อใด? `*> Add 1 to the counter` เหนือบรรทัด `ADD 1 TO WS-COUNT`

**เฉลยแนวทาง**: ผิดหลักการ "อธิบายทำไม ไม่ใช่ทำอะไร" — คอมเมนต์นี้เพียงแค่แปลโค้ดเป็นภาษาอังกฤษคำต่อคำ
โดยไม่เพิ่มข้อมูลใหม่ใด ๆ เลย คนอ่าน `ADD 1 TO WS-COUNT` เข้าใจอยู่แล้วว่ากำลังบวก 1 เข้า counter
คอมเมนต์แบบนี้เป็น "noise" ที่ทำให้โค้ดยาวขึ้นโดยไม่มีประโยชน์ ควรลบทิ้งไป หรือถ้าจะเขียนคอมเมนต์ตรงนี้
ควรอธิบายว่า **ทำไมต้องนับ** เช่น `*> Track retry attempts so BATCHJOB can cap it at MAX-RETRY`

---

## ขั้นตอนที่ 973: Defensive Programming ส่วนที่ 1 — การตรวจสอบข้อมูลนำเข้า (Input Validation)

### หลักการ: อย่าเชื่อข้อมูลที่มาจากภายนอกโปรแกรมโดยไม่ตรวจสอบ

ข้อมูลที่ "มาจากภายนอก" หมายถึงทุกอย่างที่ไม่ได้ถูกกำหนดค่าโดยตรงในโค้ดของคุณเอง: ค่าที่ผู้ใช้พิมพ์ผ่าน
`ACCEPT`, พารามิเตอร์ที่รับผ่าน `CALL...USING`, ข้อมูลที่อ่านจากไฟล์หรือฐานข้อมูล, หรือ Response จาก
REST API ภายนอก (Part 073) — Part 042 สอนเรื่อง Class Condition (`IS NUMERIC`) และ Part 089 (Enterprise
Security) ขยายความเรื่องนี้ในบริบทองค์กร แต่หลักการพื้นฐานเหมือนกันเสมอ: **ตรวจสอบก่อนใช้งาน**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. VALIDATEINPUT.
      *> Demonstrates defensive input validation before using a value
      *> that came from outside the program (ACCEPT / a parameter /
      *> a file record). Callback to Part 042 (class condition,
      *> IS NUMERIC) and Part 010 (88-level condition names).

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW-INPUT         PIC X(5).
       01  WS-QUANTITY          PIC 9(5).
       01  WS-VALID-FLAG        PIC X(1)  VALUE "Y".
           88  WS-INPUT-IS-VALID          VALUE "Y".
       01  WS-MIN-QUANTITY       PIC 9(5) VALUE 00001.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "00042" TO WS-RAW-INPUT
           PERFORM VALIDATE-AND-SHOW

           MOVE "ABCDE" TO WS-RAW-INPUT
           PERFORM VALIDATE-AND-SHOW

           MOVE "00000" TO WS-RAW-INPUT
           PERFORM VALIDATE-AND-SHOW

           STOP RUN.

       VALIDATE-AND-SHOW.
           SET WS-INPUT-IS-VALID TO TRUE

      *> Rule 1: must be numeric before we ever MOVE it into a
      *> numeric PICTURE item -- moving non-numeric text into a
      *> numeric field is undefined/garbage-producing behavior.
           IF WS-RAW-INPUT IS NOT NUMERIC
               MOVE "N" TO WS-VALID-FLAG
               DISPLAY "REJECTED [" WS-RAW-INPUT
                   "]: not numeric"
           ELSE
               MOVE WS-RAW-INPUT TO WS-QUANTITY
      *> Rule 2: even a valid number can be out of the allowed
      *> business range -- boundary check is a separate concern
      *> from "is it numeric at all".
               IF WS-QUANTITY < WS-MIN-QUANTITY
                   MOVE "N" TO WS-VALID-FLAG
                   DISPLAY "REJECTED [" WS-RAW-INPUT
                       "]: below minimum quantity"
               END-IF
           END-IF

           IF WS-INPUT-IS-VALID
               DISPLAY "ACCEPTED [" WS-RAW-INPUT
                   "]: quantity=" WS-QUANTITY
           END-IF.
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
ACCEPTED [00042]: quantity=00042
REJECTED [ABCDE]: not numeric
REJECTED [00000]: below minimum quantity
```

### อธิบายโค้ดทีละส่วน

สังเกตว่าโปรแกรมนี้ตรวจสอบ**สองชั้นแยกกัน**อย่างชัดเจน: (1) **ความเป็นตัวเลข** ด้วย `IS NOT NUMERIC`
ก่อนที่จะกล้า `MOVE` ค่าเข้าฟิลด์ตัวเลข — ถ้าข้าม step นี้ไปตรงๆ การ `MOVE` ข้อความที่ไม่ใช่ตัวเลขเข้า
`PIC 9` อาจให้ผลลัพธ์ที่ไม่แน่นอนตามมาตรฐาน (implementation-defined) ในบางคอมไพเลอร์ (2) **ขอบเขตทาง
ธุรกิจ** (business range) ด้วยการเทียบค่ากับ `WS-MIN-QUANTITY` — นี่คือคนละเรื่องกับ "เป็นตัวเลขหรือไม่"
เพราะ `00000` เป็นตัวเลขที่ถูกต้องตามหลักไวยากรณ์ แต่ผิดกฎทางธุรกิจที่ว่าจำนวนต้องมากกว่า 0

### ข้อควรระวัง

- การตรวจสอบทั้งสองชั้นต้อง**แยกจากกันชัดเจน**เสมอ อย่ารวมเป็นเงื่อนไขเดียวจนอ่านยาก เพราะข้อความ
  แจ้งเตือนที่ผู้ใช้ควรได้รับต่างกัน ("ไม่ใช่ตัวเลข" กับ "ค่าต่ำเกินไป" ต้องการคำอธิบายคนละแบบ)
- อย่าลืมว่า `IS NUMERIC` ตรวจสอบแค่ "รูปแบบตัวเลขที่ถูกต้องตาม PICTURE" เท่านั้น ไม่ได้ตรวจสอบว่าค่า
  นั้นสมเหตุสมผลทางธุรกิจหรือไม่ — ทั้งสองการตรวจสอบจำเป็นต้องทำคู่กันเสมอ

### แบบฝึกหัดที่ 973.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้ต้องตรวจสอบ `IS NOT NUMERIC` ก่อน แล้วจึงค่อย `MOVE` เข้า
`WS-QUANTITY` แทนที่จะ `MOVE` เข้าไปก่อนแล้วค่อยตรวจสอบทีหลัง

**เฉลยแนวทาง**: เพราะการ `MOVE` ข้อความที่ไม่ใช่ตัวเลข (เช่น "ABCDE") เข้าฟิลด์ `PIC 9(5)` เป็นการ
กระทำที่ผลลัพธ์ไม่แน่นอนตามมาตรฐาน COBOL ตั้งแต่ก่อนที่เราจะมีโอกาสตรวจสอบค่านั้นด้วยซ้ำ ดังนั้นลำดับ
ที่ถูกต้องคือตรวจสอบรูปแบบ**ก่อน**ในขณะที่ค่ายังอยู่ในฟิลด์ตัวอักษร (`PIC X`) ซึ่งรับค่าอะไรก็ได้อย่าง
ปลอดภัยเสมอ แล้วจึงค่อย `MOVE` เข้าฟิลด์ตัวเลขเมื่อมั่นใจแล้วว่าปลอดภัย

---

## ขั้นตอนที่ 974: Defensive Programming ส่วนที่ 2 — การตรวจสอบขอบเขต (Boundary Checks)

### หลักการ: อย่าเชื่อ Subscript ที่มาจากภายนอกโดยไม่ตรวจสอบขอบเขตตาราง

Part 016 สอนเรื่อง `OCCURS` และ Part 089 (Security ระดับ Enterprise) เน้นย้ำเรื่องการป้องกันข้อมูล
ที่ไม่น่าเชื่อถือ — การเข้าถึงตารางด้วย subscript ที่เกินขอบเขตเป็นหนึ่งในบั๊กที่อันตรายที่สุด เพราะ
คอมไพเลอร์บางตัว (หรือบาง runtime configuration) จะ**ไม่ error ให้เห็นทันที** แต่จะอ่าน/เขียนหน่วยความจำ
ที่อยู่ถัดจากตารางไปเงียบ ๆ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BOUNDARYCHECK.
      *> Demonstrates checking a subscript against table bounds
      *> BEFORE using it, instead of trusting the caller. Callback
      *> to Part 016 (OCCURS) and Part 089 (enterprise security /
      *> defensive coding for untrusted input).

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRODUCT-TABLE.
           05  WS-PRODUCT-NAME OCCURS 5 TIMES PIC X(10).
       01  WS-MAX-INDEX          PIC 9(2) VALUE 5.
       01  WS-MIN-INDEX          PIC 9(2) VALUE 1.
       01  WS-REQUESTED-IDX      PIC 9(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "APPLE     " TO WS-PRODUCT-NAME(1)
           MOVE "BANANA    " TO WS-PRODUCT-NAME(2)
           MOVE "CHERRY    " TO WS-PRODUCT-NAME(3)
           MOVE "DATE      " TO WS-PRODUCT-NAME(4)
           MOVE "EGGPLANT  " TO WS-PRODUCT-NAME(5)

           MOVE 3 TO WS-REQUESTED-IDX
           PERFORM SHOW-PRODUCT-SAFE

           MOVE 9 TO WS-REQUESTED-IDX
           PERFORM SHOW-PRODUCT-SAFE

           STOP RUN.

       SHOW-PRODUCT-SAFE.
      *> Never trust a subscript that came from outside this
      *> paragraph (a parameter, a calculation, user input) --
      *> always range-check it against the table's real bounds
      *> before subscripting. Without this check, GnuCOBOL with
      *> runtime checks off will silently read whatever memory
      *> happens to sit past the table -- garbage in, garbage out,
      *> with no error at all.
           IF WS-REQUESTED-IDX >= WS-MIN-INDEX
              AND WS-REQUESTED-IDX <= WS-MAX-INDEX
               DISPLAY "PRODUCT(" WS-REQUESTED-IDX ") = "
                   WS-PRODUCT-NAME(WS-REQUESTED-IDX)
           ELSE
               DISPLAY "REJECTED: index " WS-REQUESTED-IDX
                   " is out of range 1-5"
           END-IF.
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
PRODUCT(03) = CHERRY
REJECTED: index 09 is out of range 1-5
```

### อธิบายโค้ดทีละส่วน

เมื่อขอ index 3 (อยู่ในขอบเขต 1-5) โปรแกรมแสดงค่าปกติ แต่เมื่อขอ index 9 (เกินขอบเขต) โปรแกรม
**ปฏิเสธอย่างชัดเจนแทนที่จะพยายามอ่านค่า** — สังเกตว่าการตรวจสอบนี้ใช้ตัวแปร `WS-MAX-INDEX` และ
`WS-MIN-INDEX` แทนการเขียนตัวเลข `5` และ `1` ตรง ๆ ในเงื่อนไข เพื่อให้ถ้าขนาดตารางเปลี่ยนในอนาคต
(เช่นจาก `OCCURS 5` เป็น `OCCURS 10`) จะต้องแก้แค่จุดเดียว ไม่ใช่ไล่หาทุกจุดที่เขียนเลข `5` ไว้ตรง ๆ

### เมื่อไรควรใช้ `INDEXED BY` แทน subscript ตัวเลขธรรมดา

Part 017 สอนไว้แล้วว่า `INDEXED BY` (ใช้กับ `SEARCH`/`SEARCH ALL`) ให้ประสิทธิภาพที่ดีกว่าและ
GnuCOBOL บางเวอร์ชันยังตรวจสอบขอบเขตให้อัตโนมัติเมื่อคอมไพล์ด้วยตัวเลือกตรวจสอบขอบเขต (runtime
bounds checking) — แต่**ห้ามพึ่งพาฟีเจอร์นี้เป็นเกราะป้องกันเดียว** เพราะตัวเลือกนี้มักถูกปิดในโหมด
production เพื่อประสิทธิภาพ การตรวจสอบขอบเขตด้วยมือใน business logic ยังคงจำเป็นเสมอ โดยเฉพาะ
เมื่อ subscript มาจากค่าที่คำนวณหรือรับมาจากภายนอก

### ข้อควรระวัง

- อย่าตรวจสอบขอบเขตแค่ด้านบน (`<=` ค่าสูงสุด) โดยลืมด้านล่าง (`>=` ค่าต่ำสุด) — subscript ที่เป็น 0
  หรือติดลบ (ถ้าฟิลด์รองรับ) อันตรายพอ ๆ กับ subscript ที่เกินขอบเขตบน
- ในโปรแกรมจริงที่มีหลายตาราง อย่าใช้ตัวแปรขอบเขตร่วมกันข้ามตาราง — แต่ละตารางควรมีค่าขอบเขตของ
  ตัวเอง มิฉะนั้นการแก้ขนาดตารางหนึ่งอาจกระทบการตรวจสอบของอีกตารางโดยไม่ตั้งใจ

### แบบฝึกหัดที่ 974.1

**โจทย์**: ถ้า `WS-REQUESTED-IDX` เป็น `PIC S9(2)` (มีเครื่องหมาย) แทนที่จะเป็น `PIC 9(2)` (ไม่มี
เครื่องหมาย) การตรวจสอบขอบเขตในโค้ดข้างต้นยังคงเพียงพอหรือไม่? เพราะเหตุใด

**เฉลยแนวทาง**: ยังคงเพียงพอ เพราะเงื่อนไข `WS-REQUESTED-IDX >= WS-MIN-INDEX` (ซึ่ง `WS-MIN-INDEX`
มีค่า 1) จะปฏิเสธค่าติดลบใด ๆ โดยอัตโนมัติอยู่แล้ว (เช่น -3 ย่อมไม่ `>= 1`) — นี่คือเหตุผลที่การตรวจสอบ
ทั้งขอบเขตบนและล่างพร้อมกันสำคัญ เพราะมันครอบคลุมกรณีค่าติดลบไปในตัวโดยไม่ต้องเขียนเงื่อนไขเพิ่มเติม
แยกต่างหากสำหรับ "ค่าติดลบ" โดยเฉพาะ

---

## ขั้นตอนที่ 975: Copybook Governance — การกำกับดูแล Copybook ในทีมขนาดใหญ่

### ทำไม Copybook ถึงต้องมี "เจ้าของ" ที่ชัดเจน

Part 033 สอนพื้นฐาน `COPY` Statement, Part 080 สอน Git Workflow สำหรับทีม COBOL, และ Part 086 ขยาย
ความเรื่อง Copybook Governance ในบริบทองค์กรขนาดใหญ่ — หลักการรวมทั้งหมดคือ: **Copybook คือสัญญา
(contract) ระหว่างหลายโปรแกรมที่ไม่รู้จักกัน** เมื่อโปรแกรม A และโปรแกรม B ทั้งคู่ `COPY "CUSTOMER.CPY"`
ทั้งสองโปรแกรมกำลัง "เชื่อใจ" ว่าโครงสร้างข้อมูลตรงกัน ถ้ามีใครแก้ Copybook โดยไม่ประสานงาน โปรแกรม
ที่ไม่ได้ถูกคอมไพล์ใหม่อาจยังใช้โครงสร้างเก่าอยู่ ทำให้เกิดข้อมูลเพี้ยนแบบเงียบ ๆ

หลักฐานที่พิสูจน์แล้วจริงในหลักสูตรนี้คือระบบ Capstone ของ Part 100 เอง: `EOMSCNST.CPY` (ค่าคงที่
ทางธุรกิจ), `CUSTOMER.CPY`, `PRODUCT.CPY`, และ `ORDERREC.CPY` ถูกใช้ร่วมกันในโปรแกรมมากกว่า 10 ไฟล์
(`CUSTREPO.cob`, `PRODREPO.cob`, `ORDREPO.cob`, `ORDSVC.cob`, `EOMSMAIN.cob` ฯลฯ) — ถ้าแก้ไขโครงสร้าง
`ORDERREC.CPY` แม้เพียงเล็กน้อยโดยไม่คอมไพล์ใหม่ทุกไฟล์ที่เกี่ยวข้อง ระบบทั้งหมดจะพังทันที

### กฎการกำกับดูแล Copybook ที่ทีมมืออาชีพใช้จริง

1. **หนึ่ง Copybook ต่อหนึ่งความรับผิดชอบ** — อย่ารวมทุกอย่างไว้ในไฟล์เดียว (เทียบ `EOMSCNST.CPY`
   ที่เก็บเฉพาะค่าคงที่ กับ `CUSTOMER.CPY` ที่เก็บเฉพาะโครงสร้างข้อมูลลูกค้า แยกกันชัดเจน)
2. **ระบุเวอร์ชัน/วันที่แก้ไขล่าสุดในคอมเมนต์หัวไฟล์** เพื่อให้ตรวจสอบย้อนกลับได้ว่าโปรแกรมที่คอมไพล์
   ไว้แล้วใช้ Copybook เวอร์ชันใด
3. **ห้ามลบหรือเปลี่ยนลำดับฟิลด์ที่มีอยู่แล้ว** — ถ้าจำเป็นต้องเพิ่มฟิลด์ใหม่ ให้เพิ่มต่อท้าย ไม่ใช่
   แทรกกลางหรือลบของเดิม เพราะโปรแกรมเก่าที่ยังไม่ได้คอมไพล์ใหม่อาจอ้างอิงตำแหน่ง byte offset เดิมอยู่
   (สำคัญมากสำหรับไฟล์ที่มีข้อมูลค้างอยู่บนดิสก์แล้ว เช่น Indexed File ตามที่ Part 028 สอน)
4. **Copybook ควรอยู่ใน Version Control เดียวกับโปรแกรมที่ใช้มัน** (Part 080) และการเปลี่ยนแปลง
   ควรผ่าน Code Review เสมอ เพราะผลกระทบกว้างกว่าการแก้โปรแกรมเดี่ยว ๆ มาก
5. **คอมไพล์ใหม่ทุกโปรแกรมที่ COPY ไฟล์นั้นทันทีที่ Copybook เปลี่ยน** — นี่คือเหตุผลที่ `ci.sh`
   ใน Part 100 คอมไพล์ทุกโมดูลใหม่ทั้งหมดทุกครั้ง (Part 078 CI/CD) แทนที่จะคอมไพล์เฉพาะไฟล์ที่แก้ไข

### ตัวอย่างจริงจาก Capstone: รูปแบบ Header ที่ Copybook ทุกไฟล์ควรมี

```cobol
      *> ================================================================
      *> Copybook: EOMSCNST.CPY
      *> Enterprise Order Management System (EOMS) - named constants.
      *> Single source of truth for every business rule threshold used
      *> across EOMS programs (Part 098 copybook governance practice:
      *> one place to change a rule, every program picks it up on next
      *> compile). Level-78 items, same technique as Part 033/085.
      *> ================================================================
       78  EOMS-MAX-ORDER-LINES     VALUE 5.
```

สังเกตว่า Header อธิบายชัดเจนว่า Copybook นี้คือ **"Single Source of Truth"** สำหรับค่าคงที่ทางธุรกิจ
— ถ้าต้องการเปลี่ยนอัตราภาษี หรือเกณฑ์ส่วนลด ผู้พัฒนาทุกคนรู้ทันทีว่าต้องมาแก้ที่ไฟล์นี้ไฟล์เดียว
ไม่ต้องไล่หาเลขที่ฝังอยู่กระจัดกระจายในหลายโปรแกรม (ปัญหา Magic Number ที่ Part 085 ขั้นตอนที่ 843
เคยพบใน `ARLEGACY.cob` มาแล้ว)

### ข้อควรระวัง

- Copybook ที่ใช้ทั้งใน `FILE SECTION` และ `LINKAGE SECTION` ของโปรแกรมเดียวกันต้องเปลี่ยนชื่อ
  top-level ด้วย `REPLACING` เพื่อไม่ให้ฟิลด์ย่อยชนกัน (ดูตัวอย่างจริงใน `CUSTREPO.cob` ของ Part 100
  ขั้นตอนที่ 995 ที่ใช้ `COPY "CUSTOMER.CPY" REPLACING ==CUST-RECORD== BY ==FD-CUST-RECORD==`)
- อย่าใช้ `COPY...REPLACING` เพื่อ "ปรับแต่ง" โครงสร้างข้อมูลให้ต่างกันไปในแต่ละโปรแกรม — นั่นทำลาย
  จุดประสงค์หลักของ Copybook ที่ต้องการให้ทุกโปรแกรม "เห็นข้อมูลตรงกัน" `REPLACING` ควรใช้แค่เปลี่ยน
  **ชื่อ** ไม่ใช่เปลี่ยน**โครงสร้าง**

### แบบฝึกหัดที่ 975.1

**โจทย์**: ทีมหนึ่งต้องการเพิ่มฟิลด์ `CUST-EMAIL` เข้าไปใน `CUSTOMER.CPY` ที่มีโปรแกรมใช้งานอยู่แล้ว
15 โปรแกรม และมีไฟล์ `CUSTMAST.DAT` ที่มีข้อมูลลูกค้าอยู่แล้วหลายพันระเบียนบนดิสก์ จงอธิบายขั้นตอน
ที่ปลอดภัยที่สุดในการทำสิ่งนี้

**เฉลยแนวทาง**: (1) เพิ่มฟิลด์ `CUST-EMAIL` ต่อท้ายฟิลด์สุดท้ายเดิมใน Copybook เท่านั้น ห้ามแทรกกลาง
(2) เขียนโปรแกรมแปลงไฟล์ (conversion program) ที่อ่านระเบียนเก่าทีละรายการแล้วเขียนใหม่ลงไฟล์ที่มี
ความยาวระเบียนใหม่ (รวมพื้นที่สำหรับ `CUST-EMAIL` ที่อาจเป็นค่าว่างในตอนแรก) คล้ายเทคนิคที่ Part 028
สอนเรื่องการจัดการ Indexed File (3) คอมไพล์ใหม่ทั้ง 15 โปรแกรมพร้อมกันหลัง Copybook เปลี่ยน ห้ามปล่อย
ให้บางโปรแกรมใช้ Copybook เก่าและบางโปรแกรมใช้ใหม่พร้อมกัน เพราะจะอ่าน/เขียนความยาวระเบียนไม่ตรงกัน
(4) ทดสอบด้วย Golden Master Testing (Part 084/085) เปรียบเทียบผลลัพธ์ก่อน/หลังเปลี่ยนแปลง

---

## ขั้นตอนที่ 976: กับดักจริงที่พบระหว่างเตรียม Part 100 — CLOSE เขียนทับ FILE STATUS

### เรื่องจริงจากการเตรียมเนื้อหาหลักสูตรนี้

Part 035 และ Part 085 เคยเล่าให้ฟังแล้วว่าระหว่างเตรียมเนื้อหา ทีมงานพบบั๊กจริงจากการทดสอบจริง
(ไม่ใช่ตัวอย่างสมมติ) — Part 100 (Capstone Project) ก็พบบั๊กแบบเดียวกันอีกครั้ง คราวนี้เกิดขึ้นใน
`CUSTREPO.cob` (Data Access Layer สำหรับไฟล์ลูกค้า) และคุ้มค่ามากที่จะนำมาเล่าไว้ที่นี่เพราะเป็นกับดัก
ที่แนบเนียนมาก แม้แต่โปรแกรมเมอร์ที่มีวินัยตรวจสอบ `FILE STATUS` ทุกครั้งตามที่ Part 030 สอนไว้ ก็ยัง
พลาดได้ถ้าไม่ระวังจุดนี้

### โค้ดที่มีบั๊ก (เวอร์ชันก่อนแก้ไข)

```cobol
       DO-READ.
           MOVE CUST-RECORD TO FD-CUST-RECORD
           OPEN INPUT CUST-FILE
           IF LK-IO-STATUS = "00"
               READ CUST-FILE
                   INVALID KEY
                       CONTINUE
               END-READ
               MOVE FD-CUST-RECORD TO CUST-RECORD
               CLOSE CUST-FILE
           END-IF.
```

โค้ดนี้ดู "ถูกต้อง" ทุกประการเมื่ออ่านผ่าน ๆ ตา: เปิดไฟล์ ตรวจสอบสถานะ อ่านระเบียน จัดการกรณี
`INVALID KEY` (Part 030) แล้วปิดไฟล์ — แต่เมื่อทดสอบจริงด้วยการอ่านระเบียนที่**ไม่มีอยู่จริง**
(เช่น customer ID ที่ไม่เคยถูกสร้าง) ผู้เรียกกลับได้รับ `LK-IO-STATUS = "00"` (สำเร็จ!) พร้อมข้อมูล
ที่เป็นขยะ (ค่า default ว่างเปล่า) แทนที่จะได้รับสถานะ error ตามที่ควรจะเป็น

### สาเหตุที่แท้จริง: `CLOSE` ก็ตั้งค่า FILE STATUS เหมือนกัน

`READ` ที่ล้มเหลว (`INVALID KEY`) จะตั้งค่า `LK-IO-STATUS` เป็น `"23"` (record not found) อย่างถูกต้อง
ตามที่ Part 030 สอนไว้ — **แต่บรรทัดถัดไปคือ `CLOSE CUST-FILE` ซึ่งสำเร็จ และ `CLOSE` ก็เขียนทับ
`FILE STATUS` ด้วยค่า `"00"` ของมันเองทันที!** เพราะ `FILE STATUS IS LK-IO-STATUS` ผูกกับ**ทุกคำสั่ง**
ที่กระทำกับไฟล์นั้น (`OPEN`, `READ`, `WRITE`, `REWRITE`, `CLOSE` ทั้งหมดใช้ตัวแปรเดียวกัน) ไม่ใช่แค่
`READ` เพียงอย่างเดียว ผลคือสถานะ `"23"` จาก `READ` ที่ล้มเหลวถูก**ลบทิ้งไปเงียบ ๆ** ก่อนที่ผู้เรียก
จะมีโอกาสตรวจสอบมันเลยด้วยซ้ำ

### วิธีแก้ที่พิสูจน์แล้วว่าถูกต้อง

```cobol
       DO-READ.
           MOVE CUST-RECORD TO FD-CUST-RECORD
           OPEN INPUT CUST-FILE
           IF LK-IO-STATUS = "00"
               READ CUST-FILE
                   INVALID KEY
                       CONTINUE
               END-READ
      *> Save the READ's status BEFORE closing the file. This is a
      *> real bug this capstone hit during testing: CLOSE also sets
      *> FILE STATUS, so closing right after a failed READ silently
      *> overwrote "23" (record not found) back to "00" - callers
      *> saw a false success. Never assume a status survives the
      *> next file operation; capture it the moment you need it.
               MOVE LK-IO-STATUS TO WS-SAVED-STATUS
               MOVE FD-CUST-RECORD TO CUST-RECORD
               CLOSE CUST-FILE
               MOVE WS-SAVED-STATUS TO LK-IO-STATUS
           END-IF.
```

**พิสูจน์ด้วยการทดสอบจริง**: หลังแก้ไข การเรียกอ่าน order ID ที่ไม่มีอยู่จริงผ่านโปรแกรม `ORDEXPORT.cob`
(Part 100 ขั้นตอนที่ 998) ให้ผลลัพธ์ที่ถูกต้องทันที:

```json
{"error":"order not found"}
```

แทนที่จะเป็น JSON ที่มีค่าตัวเลขเป็นศูนย์ทั้งหมดราวกับว่าพบระเบียนจริง (พฤติกรรมที่ผิดก่อนแก้ไข)

### บทเรียนที่นำไปใช้ได้ทั่วไป

**อย่าสมมติว่าค่าสถานะ (status) ใด ๆ จะ "คงอยู่" ข้ามการกระทำถัดไป** โดยเฉพาะเมื่อตัวแปรสถานะนั้นถูก
ใช้ร่วมกันโดยหลายคำสั่ง — ถ้าต้องการใช้ค่าสถานะไปตัดสินใจอะไรในภายหลัง ให้ **copy มันออกมาเก็บทันที**
ที่ได้ค่านั้น ก่อนที่จะมีคำสั่งอื่นใดมีโอกาสเขียนทับมัน หลักการนี้ใช้ได้กว้างกว่าแค่ `FILE STATUS` —
ใช้ได้กับตัวแปร return code ของ subprogram ที่ถูกเรียกซ้อนกันหลายชั้น หรือแม้แต่ Environment Variable
ที่ระบบภายนอกอาจเปลี่ยนแปลงได้

### ข้อควรระวัง

- บั๊กแบบนี้จะ**ไม่ปรากฏ**เลยถ้าทดสอบเฉพาะกรณีสำเร็จ (happy path) — มันจะโผล่ออกมาเฉพาะกรณี error
  ที่ตามด้วยการกระทำอื่นที่สำเร็จเท่านั้น นี่คือเหตุผลสำคัญที่ Part 079 (Unit Testing) เน้นย้ำเรื่อง
  การทดสอบ "boundary case" และ "error case" ให้ครบ ไม่ใช่แค่กรณีปกติ
- ตรวจสอบเอกสารของคอมไพเลอร์ของคุณเองว่า `CLOSE` ตั้งค่า `FILE STATUS` เป็นอะไรเมื่อสำเร็จ — พฤติกรรม
  นี้เป็นไปตามมาตรฐาน COBOL ทั่วไป แต่ผู้พัฒนาจำนวนมากอาจไม่เคยรู้มาก่อนจนกว่าจะเจอบั๊กแบบนี้เอง

### แบบฝึกหัดที่ 976.1

**โจทย์**: นอกจาก `CLOSE` แล้ว ยังมีคำสั่งไฟล์ตัวใดอีกบ้างที่ใช้ `FILE STATUS` ตัวแปรเดียวกัน และ
เหตุใดโปรแกรมเมอร์จึงควรระวังลำดับการเรียกคำสั่งเหล่านี้เสมอ

**เฉลยแนวทาง**: `OPEN`, `READ`, `WRITE`, `REWRITE`, `DELETE`, `START`, และ `CLOSE` ทั้งหมดใช้ตัวแปร
เดียวกันที่ระบุใน clause `FILE STATUS IS ...` ของ `SELECT` เพราะเป็นเพียง "ช่องเก็บผลลัพธ์ล่าสุด"
ของการกระทำกับไฟล์นั้น ไม่ใช่ประวัติสะสม ดังนั้นทุกครั้งที่มีคำสั่งไฟล์มากกว่าหนึ่งคำสั่งเรียงต่อกัน
(เช่น `READ` ตามด้วย `CLOSE`) โปรแกรมเมอร์ต้องถามตัวเองเสมอว่า "ฉันต้องการใช้สถานะของคำสั่งไหน"
และถ้าต้องการสถานะของคำสั่งที่ไม่ใช่คำสั่งสุดท้าย ต้อง copy ค่านั้นออกมาเก็บไว้ก่อนคำสั่งถัดไปจะทำงาน

---

## ขั้นตอนที่ 977: ตาราง "กับดักยอดฮิต" — สรุปรวมบั๊กที่หลักสูตรนี้พิสูจน์จริงตลอด 97 Part

### ทำไมต้องมีตารางนี้

ตารางด้านล่างคือ **"Greatest Hits"** ของกับดัก COBOL ที่อันตรายที่สุดที่หลักสูตรนี้เจอและพิสูจน์ด้วย
การคอมไพล์-รันจริงมาแล้ว ไม่ใช่ทฤษฎีที่ยกมาลอย ๆ — แต่ละแถวอ้างอิง Part ที่พิสูจน์เรื่องนั้นจริง
คุณสามารถใช้ตารางนี้เป็น **Checklist ป้องกันตัวเอง** ก่อนส่งโค้ดเข้า Production ได้ทันที

| # | กับดัก | อาการ | วิธีป้องกัน | พิสูจน์จริงที่ Part |
|---|---|---|---|---|
| 1 | Column 72 นับเป็นไบต์ ไม่ใช่ตัวอักษร | ภาษาไทย/Unicode ใน comment ทำให้เกิน column 72 แบบมองไม่เห็นด้วยตา เกิด `continuation character expected` | เขียนโค้ด COBOL (รวม comment) เป็น ASCII ล้วนเท่านั้น | Part 002 |
| 2 | MOVE ตัดหลักสูงทิ้งแบบเงียบ ๆ | ค่าตัวเลข/ข้อความที่ยาวเกิน PICTURE ถูกตัดทอนโดยไม่มี warning | ตรวจสอบขนาดฟิลด์ปลายทางให้ใหญ่พอเสมอ โดยเฉพาะหลัง MULTIPLY | Part 008, Part 009 |
| 3 | SIZE ERROR ทำให้ฟิลด์ปลายทาง "ค้างค่าเดิม" แบบเงียบ ๆ | หารด้วยศูนย์/overflow โดยไม่มี `ON SIZE ERROR` ทำให้ผลลัพธ์เป็นค่าเก่าจากการคำนวณครั้งก่อน ดูสมเหตุสมผลแต่ผิดสนิท | ตรวจสอบเงื่อนไขก่อนคำนวณ หรือใช้ `ON SIZE ERROR` เสมอ | Part 009, Part 015 |
| 4 | REWRITE ต้องมีความยาว/คีย์เดียวกับระเบียนที่ READ มาก่อน | เปลี่ยนคีย์หรือความยาวระเบียนก่อน REWRITE ทำให้เกิด error หรือพฤติกรรมไม่คาดคิด | แก้ไขฟิลด์ใน place บนระเบียนที่ READ มา อย่าสร้างระเบียนใหม่ทั้งชุด | Part 025, Part 100 |
| 5 | ไม่ตรวจสอบ FILE STATUS หลังทุกคำสั่งไฟล์ | โปรแกรมดำเนินต่อไปราวกับไม่มีอะไรผิดพลาดหลัง OPEN/READ/WRITE ล้มเหลว | ตรวจสอบ FILE STATUS (หรือ 88-level ที่ผูกกับมัน) ทุกครั้งหลังทุกคำสั่งไฟล์ | Part 030 |
| 6 | CLOSE เขียนทับ FILE STATUS ของคำสั่งก่อนหน้า | READ ที่ล้มเหลวตามด้วย CLOSE ที่สำเร็จ ทำให้สถานะ error หายไปเป็น "00" | Copy สถานะที่ต้องการเก็บออกมาทันทีก่อนคำสั่งไฟล์ถัดไป | Part 098 (ขั้นตอนที่ 976), Part 100 |
| 7 | GO TO ใช้ "ข้าม" โค้ดแทนโครงสร้างเงื่อนไขที่ถูกต้อง | ควบคุมการไหลของโปรแกรมยากต่อการติดตาม เสี่ยงบั๊กเมื่อแก้ไขในอนาคต | ใช้ `IF/EVALUATE` กับ Scope Terminator แทนแทบทุกกรณี | Part 010, Part 011, Part 098 |
| 8 | ลืม GOBACK/EXIT PROGRAM ท้าย Paragraph สุดท้ายของ Subprogram ที่ถูกเรียกจากหลาย Paragraph | โปรแกรมทำงาน "ตกหล่น" (fall through) เข้า Paragraph ถัดไปโดยไม่ตั้งใจ หลังลูปควบคุมจบ | จบทุกเส้นทางของ Subprogram ด้วย GOBACK/EXIT PROGRAM อย่างชัดเจนเสมอ | Part 100 (ORDVALID.cob) |
| 9 | Subscript ไม่ตรวจสอบขอบเขตก่อนใช้งาน | เข้าถึงตารางเกินขอบเขต อ่าน/เขียนหน่วยความจำที่ไม่ใช่ของตาราง แบบไม่มี error ชัดเจน | ตรวจสอบ subscript กับขอบเขตบน/ล่างของตารางก่อนใช้งานเสมอ | Part 016, Part 089, Part 098 |
| 10 | Indexed File (ORGANIZATION IS INDEXED) ถูกปิดใช้งานในบาง GnuCOBOL build | คอมไพล์ error หรือรันไม่ได้เพราะ ISAM handler ไม่ถูกเปิดใช้งาน | ตรวจสอบด้วย `cobc -info \| grep -i indexed` ก่อนเสมอ ใช้ build ที่เปิด `--with-db` เมื่อจำเป็น | Part 028, Part 070, Part 100 |
| 11 | ตารางที่ส่งผ่าน CALL ต้องห่อด้วย group item | ส่ง OCCURS item ระดับ 01 เปล่า ๆ ผ่าน `PROCEDURE DIVISION USING` ทำให้เกิด compile error "requires one subscript" | ห่อ OCCURS ไว้ใต้ group item เสมอเมื่อจะส่งผ่าน CALL boundary | Part 032, Part 100 |
| 12 | JSON GENERATE ไม่รองรับในทุก GnuCOBOL build | คอมไพล์ error "compiler is not configured to support JSON" แม้ไวยากรณ์ถูกต้องตามมาตรฐาน | ตรวจสอบ `cobc -info \| grep -i json` ก่อนใช้ หากปิดอยู่ให้สร้าง JSON ด้วย STRING แทน | Part 045, Part 100 |

### ข้อควรระวัง

- ตารางนี้**ไม่ใช่รายการที่สมบูรณ์ทั้งหมด** ของบั๊ก COBOL ที่เป็นไปได้ — แต่เป็นรายการที่**พิสูจน์แล้ว
  จริงด้วยมือของหลักสูตรนี้เอง** ตลอดกระบวนการเขียน 100 Part เมื่อคุณเจอกับดักใหม่ที่ไม่อยู่ในตารางนี้
  ในงานจริง ให้บันทึกมันไว้ในเอกสารทีมของคุณเองเช่นเดียวกัน
- สังเกตว่าหลายแถวในตารางมีธีมร่วมกัน: **"พฤติกรรมที่ดูเหมือนสำเร็จ แต่จริง ๆ แล้วผิดอย่างเงียบ ๆ"**
  (silent failure) นี่คือรูปแบบบั๊กที่อันตรายที่สุดในภาษาใด ๆ ก็ตาม ไม่ใช่เฉพาะ COBOL เพราะโปรแกรม
  ไม่ crash ให้เห็นทันที แต่สร้างข้อมูลผิดพลาดที่อาจถูกใช้ต่อในระบบธุรกิจจริงโดยไม่มีใครสังเกตเห็น

### แบบฝึกหัดที่ 977.1

**โจทย์**: จากตารางข้างต้น จงเลือกมา 3 กับดักที่คุณคิดว่า "อันตรายที่สุด" สำหรับทีมที่เพิ่งเริ่มเขียน
COBOL และอธิบายเหตุผล

**เฉลยแนวทาง**: ไม่มีคำตอบตายตัว แต่แนวทางที่สมเหตุสมผล เช่น กับดัก #3 (SIZE ERROR ทำให้ฟิลด์ค้างค่า
เดิม), #6 (CLOSE เขียนทับ FILE STATUS), และ #8 (ลืม GOBACK) มักถูกเลือกเป็นอันดับต้น ๆ เพราะทั้งสาม
กรณีนี้เป็น **silent failure** ที่โปรแกรมยังคง "ดูเหมือนทำงานถูกต้อง" ทำให้ทีมที่ไม่มีประสบการณ์อาจ
ไม่ทันสังเกตจนกว่าจะเกิดความเสียหายทางธุรกิจจริงแล้ว ต่างจากกับดักอย่าง #1 หรือ #11 ที่ทำให้คอมไพล์
error ทันที (ซึ่งแม้จะน่ารำคาญแต่ปลอดภัยกว่ามากในแง่ที่ถูกจับได้ทันทีก่อนขึ้น production)

---

## ขั้นตอนที่ 978: Code Review Checklist ที่ทีมนำไปใช้ได้จริง

### หลักการ: Checklist ที่ดีต้องตรวจสอบได้จริงในเวลาจำกัด

Code Review ที่มีประสิทธิภาพไม่ใช่การอ่านโค้ดทุกบรรทัดอย่างละเอียดโดยไม่มีจุดโฟกัส แต่คือการไล่ตรวจ
รายการที่ชัดเจนซึ่งครอบคลุมกับดักที่พบบ่อยที่สุด ต่อไปนี้คือ Checklist ที่รวบรวมจากบทเรียนทั้งหมดใน
Part นี้ ออกแบบให้ใช้ได้จริงภายในเวลา 15-20 นาทีต่อ Pull Request หนึ่งชุด

### EOMS Code Review Checklist (ใช้ได้กับโปรแกรม COBOL ทุกขนาด)

**A. ความถูกต้องพื้นฐาน (Correctness)**

- [ ] โค้ดคอมไพล์ผ่านโดยไม่มี warning (ไม่ใช่แค่ "ไม่มี error")
- [ ] ทุก `COMPUTE`/`ADD`/`SUBTRACT`/`MULTIPLY`/`DIVIDE` ที่มีความเสี่ยง overflow หรือหารศูนย์
      มีการป้องกันด้วย `IF` ตรวจสอบก่อน หรือ `ON SIZE ERROR`
- [ ] ทุกฟิลด์ปลายทางของการคำนวณมีขนาด (จำนวนหลัก) ใหญ่พอสำหรับผลลัพธ์สูงสุดที่เป็นไปได้

**B. การจัดการไฟล์และทรัพยากรภายนอก**

- [ ] ทุกคำสั่ง `OPEN`/`READ`/`WRITE`/`REWRITE`/`DELETE`/`CLOSE` มีการตรวจสอบ `FILE STATUS` ทันที
- [ ] ถ้าต้องการเก็บ `FILE STATUS` ไปใช้หลังคำสั่งไฟล์อื่น มีการ copy ค่าออกมาเก็บก่อนเสมอ
- [ ] `REWRITE` แก้ไขฟิลด์บนระเบียนที่ `READ` มาก่อนหน้า ไม่ได้สร้างระเบียนใหม่ทั้งชุด

**C. Defensive Programming**

- [ ] ข้อมูลนำเข้าจากภายนอก (ACCEPT, พารามิเตอร์, ไฟล์) ผ่านการตรวจสอบ `IS NUMERIC`/ขอบเขตก่อนใช้งาน
- [ ] Subscript ของทุกตารางถูกตรวจสอบกับขอบเขตบน/ล่างก่อน subscript จริง เมื่อ subscript มาจาก
      ค่าที่คำนวณหรือรับจากภายนอก

**D. โครงสร้างและการดูแลรักษา**

- [ ] ไม่มี `GO TO` ที่ใช้ "ข้าม" โค้ดแทนโครงสร้างเงื่อนไข (ยกเว้นกรณีพิเศษที่มีคอมเมนต์อธิบายชัดเจน)
- [ ] Subprogram ทุกเส้นทางจบด้วย `GOBACK`/`EXIT PROGRAM` อย่างชัดเจน ไม่มีจุดที่ "ตกหล่น" เข้า
      Paragraph ถัดไปโดยไม่ตั้งใจ
- [ ] ไม่มี Magic Number ฝังอยู่ในโค้ด — ค่าคงที่ทางธุรกิจอยู่ใน Copybook ระดับ 78 พร้อมชื่อสื่อความหมาย
- [ ] ชื่อตัวแปร/paragraph สื่อความหมายทางธุรกิจ ไม่ใช่แค่บอกชนิดข้อมูล

**E. การทดสอบ**

- [ ] มี Unit Test ครอบคลุมทั้งกรณีปกติ (happy path) และกรณีขอบเขต/error (boundary/error case)
- [ ] ถ้าเป็นการ Refactor โค้ดเดิม มีการเปรียบเทียบผลลัพธ์กับ Golden Master ก่อน/หลังแก้ไข
- [ ] โค้ดทุกตัวอย่างถูกคอมไพล์และรันจริงแล้ว ไม่ใช่แค่ "ดูน่าจะถูกต้อง"

**F. กฎเฉพาะหลักสูตรนี้**

- [ ] โค้ด COBOL ทั้งหมด (comment, ชื่อตัวแปร, string literal) เป็น ASCII ล้วน ไม่มีอักขระที่ไม่ใช่
      ASCII หลุดเข้าไปโดยไม่ตั้งใจ

### ข้อควรระวัง

- Checklist นี้ไม่ได้แทนที่วิจารณญาณของผู้ตรวจโค้ด — มันเป็นเพียง "ตาข่ายนิรภัยขั้นต่ำ" ที่ช่วยไม่ให้
  ลืมตรวจสอบสิ่งที่พิสูจน์แล้วว่าเป็นปัญหาบ่อย ประเด็นที่ซับซ้อนกว่านั้น (เช่น การออกแบบสถาปัตยกรรม)
  ยังต้องอาศัยประสบการณ์และการอภิปรายกับทีม
- ทีมควรปรับแต่ง Checklist นี้ตามบริบทของตัวเอง — เช่นทีมที่ทำงานกับ CICS อาจต้องเพิ่มหัวข้อเกี่ยวกับ
  Pseudo-conversational Programming (Part 064) เข้าไปด้วย

### แบบฝึกหัดที่ 978.1

**โจทย์**: จงนำ Checklist ข้างต้นไปตรวจสอบโค้ด `before_refactor` (BEFOREREFACTOR) ในขั้นตอนที่ 980
ถัดไป ก่อนที่จะอ่านคำเฉลยการ Refactor ว่าโค้ดนั้นผ่าน/ไม่ผ่านข้อใดบ้าง

**เฉลยแนวทาง**: ให้ผู้เรียนลองทำเองก่อนไปขั้นตอนที่ 980 — คำตอบคร่าว ๆ คือโค้ดนั้นจะไม่ผ่านข้อ "ไม่มี
GO TO ที่ใช้ข้ามโค้ด" (หมวด D) และ "ไม่มี Magic Number" (หมวด D) อย่างชัดเจน ซึ่งตรงกับสิ่งที่ขั้นตอน
ที่ 980 จะแก้ไข

---

## ขั้นตอนที่ 979: หลักการเขียนโค้ดให้ดูแลรักษาง่าย (Maintainability)

### หลักการ: โค้ดที่ดีต้อง "แก้ง่าย" ไม่ใช่แค่ "ทำงานถูกต้อง"

โค้ดที่ทำงานถูกต้องในวันนี้ อาจกลายเป็นภาระหนักในอีก 2 ปีข้างหน้าถ้าไม่ได้ออกแบบมาให้ดูแลรักษาง่าย
Part 081 (Refactoring และ Clean Code) และ Part 082 (Design Patterns) สอนเทคนิคเหล่านี้ไว้อย่างละเอียด
แล้ว — ในที่นี้เราจะรวบยอดหลักการที่สำคัญที่สุด 3 ข้อที่ใช้ตลอดทั้ง Capstone Project ของ Part 100:

1. **Single Responsibility** — หนึ่ง Paragraph/Subprogram ควรทำหน้าที่เดียว (เทียบ `ORDCALC.cob`
   ที่คำนวณอย่างเดียว ไม่ยุ่งกับไฟล์ กับ `CUSTREPO.cob` ที่จัดการไฟล์อย่างเดียว ไม่มีตรรกะทางธุรกิจ)
2. **Flat is better than nested** — `EVALUATE TRUE` แบบเรียบอ่านง่ายกว่า `IF/ELSE` ซ้อนหลายชั้น
3. **Named constants over magic numbers** — ค่าคงที่ทางธุรกิจควรมีชื่อ ไม่ใช่ตัวเลขลอย ๆ

### เปรียบเทียบ: GO TO กับ EVALUATE ในสถานการณ์เดียวกัน

โค้ดสองเวอร์ชันต่อไปนี้**ทำงานเหมือนกันทุกประการ** (พิสูจน์แล้วด้วยการทดสอบ input ชุดเดียวกัน) แต่
ต่างกันอย่างสิ้นเชิงในแง่การดูแลรักษา — เราจะเห็นเวอร์ชัน "ก่อน" ในที่นี้ และเวอร์ชัน "หลัง Refactor"
พร้อมแบบฝึกหัดเต็มรูปแบบในขั้นตอนที่ 980

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BEFOREREFACTOR.
      *> "Before" version: deeply nested IF, a magic number, and a
      *> GO TO used only to skip past code -- the kind of code this
      *> Part's checklist flags in review.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-STATUS      PIC X(1).
       01  WS-ORDER-TOTAL       PIC 9(7)V99.
       01  WS-MESSAGE           PIC X(40).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "O" TO WS-ORDER-STATUS
           MOVE 15000.00 TO WS-ORDER-TOTAL
           PERFORM CLASSIFY-ORDER
           DISPLAY WS-MESSAGE

           MOVE "X" TO WS-ORDER-STATUS
           PERFORM CLASSIFY-ORDER
           DISPLAY WS-MESSAGE
           STOP RUN.

       CLASSIFY-ORDER.
           IF WS-ORDER-STATUS = "O"
               IF WS-ORDER-TOTAL > 10000.00
                   MOVE "High-value open order - needs approval"
                       TO WS-MESSAGE
               ELSE
                   IF WS-ORDER-TOTAL > 0
                       MOVE "Open order" TO WS-MESSAGE
                   ELSE
                       GO TO SKIP-MESSAGE
                   END-IF
               END-IF
           ELSE
               IF WS-ORDER-STATUS = "C"
                   MOVE "Closed order" TO WS-MESSAGE
               ELSE
                   MOVE "Unknown order status" TO WS-MESSAGE
               END-IF
           END-IF.
       SKIP-MESSAGE.
           EXIT.
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
High-value open order - needs approval
Unknown order status
```

### ปัญหาที่ซ่อนอยู่ในโค้ดนี้

แม้โค้ดนี้จะรันได้ถูกต้อง แต่มีปัญหาด้าน Maintainability ชัดเจน 3 จุด: (1) `IF/ELSE` ซ้อนกัน 3 ชั้น
ทำให้ต้องนับวงเล็บ/`END-IF` เพื่อเข้าใจว่าเงื่อนไขไหนคู่กับอะไร (2) ค่า `10000.00` เป็น Magic Number
ที่ไม่มีใครรู้ความหมายทางธุรกิจถ้าไม่อ่านทั้งบริบท (3) `GO TO SKIP-MESSAGE` เป็นการ "กระโดดออก" ที่ทำ
หน้าที่เหมือน `ELSE` แต่เขียนด้วยเทคนิคที่ทำให้ผู้อ่านต้องไล่หาว่า `SKIP-MESSAGE` อยู่ตรงไหนของโปรแกรม

### ข้อควรระวัง

- `GO TO` ไม่ได้ "ผิดกฎ" เสมอไปในมาตรฐาน COBOL แต่การใช้เพื่อจำลองพฤติกรรมที่ `IF/EVALUATE` ทำได้
  อยู่แล้วเป็นสัญญาณเตือนที่ควรระวังในการ Code Review เสมอ (ทบทวนหมวด D ของ Checklist ในขั้นตอนที่ 978)
- อย่า Refactor โครงสร้างเงื่อนไขที่ซับซ้อนโดยไม่มีการทดสอบเปรียบเทียบผลลัพธ์ก่อน/หลัง — ยิ่งเงื่อนไข
  ซ้อนกันมากเท่าไร ยิ่งมีโอกาสพลาดลำดับการตรวจสอบตอน Refactor มากเท่านั้น

### แบบฝึกหัดที่ 979.1

**โจทย์**: ก่อนไปดูเฉลยการ Refactor ในขั้นตอนที่ 980 จงลองเขียนโครงร่าง (ไม่ต้องคอมไพล์) ของ
`CLASSIFY-ORDER` เวอร์ชันที่ใช้ `EVALUATE TRUE` แทน `IF/ELSE` ซ้อน และไม่มี `GO TO` เลย ด้วยตัวเอง

**เฉลยแนวทาง**: ให้ผู้เรียนลองทำเองก่อน — แนวทางคร่าว ๆ คือใช้ `EVALUATE TRUE` พร้อมเรียงเงื่อนไข
จากเฉพาะเจาะจงที่สุดไปทั่วไปที่สุด (`WS-ORDER-STATUS = "O" AND WS-ORDER-TOTAL > 10000` ก่อน แล้วค่อย
`WS-ORDER-STATUS = "O" AND WS-ORDER-TOTAL > 0` แล้วค่อย `WS-ORDER-STATUS = "O"` เฉย ๆ เป็นค่า default
ของกลุ่ม Open) และแทนที่ `10000.00` ด้วยชื่อค่าคงที่ เช่น `WS-HIGH-VALUE-LIMIT`

---

## ขั้นตอนที่ 980: แบบฝึกหัด Refactor เต็มรูปแบบ — สรุปรวมหลักการทั้งหมดของ Part นี้

### เฉลยเต็มรูปแบบ: เวอร์ชัน "หลัง Refactor"

นี่คือเฉลยของแบบฝึกหัดที่ 979.1 ที่นำหลักการทั้งหมดจาก Part นี้มารวมกัน: ตั้งชื่อสื่อความหมาย
(ขั้นตอนที่ 971), คอมเมนต์อธิบายเหตุผล (ขั้นตอนที่ 972), ไม่มี Magic Number (ขั้นตอนที่ 975/977),
และโครงสร้างแบบเรียบไม่ซ้อนลึก (ขั้นตอนที่ 979):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. AFTERREFACTOR.
      *> "After" version: flat EVALUATE (reads as a decision table),
      *> a named constant instead of a magic number, and no GO TO at
      *> all -- same behavior, proven by the same test inputs.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-STATUS      PIC X(1).
       01  WS-ORDER-TOTAL       PIC 9(7)V99.
       01  WS-MESSAGE           PIC X(40).
       01  WS-HIGH-VALUE-LIMIT  PIC 9(7)V99 VALUE 10000.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "O" TO WS-ORDER-STATUS
           MOVE 15000.00 TO WS-ORDER-TOTAL
           PERFORM CLASSIFY-ORDER
           IF WS-MESSAGE NOT = SPACES
               DISPLAY WS-MESSAGE
           END-IF

           MOVE "X" TO WS-ORDER-STATUS
           PERFORM CLASSIFY-ORDER
           IF WS-MESSAGE NOT = SPACES
               DISPLAY WS-MESSAGE
           END-IF
           STOP RUN.

       CLASSIFY-ORDER.
           MOVE SPACES TO WS-MESSAGE
           EVALUATE TRUE
               WHEN WS-ORDER-STATUS = "O"
                    AND WS-ORDER-TOTAL > WS-HIGH-VALUE-LIMIT
                   MOVE "High-value open order - needs approval"
                       TO WS-MESSAGE
               WHEN WS-ORDER-STATUS = "O" AND WS-ORDER-TOTAL > 0
                   MOVE "Open order" TO WS-MESSAGE
               WHEN WS-ORDER-STATUS = "O"
                   CONTINUE
               WHEN WS-ORDER-STATUS = "C"
                   MOVE "Closed order" TO WS-MESSAGE
               WHEN OTHER
                   MOVE "Unknown order status" TO WS-MESSAGE
           END-EVALUATE.
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน (เหมือนเวอร์ชัน "ก่อน" ทุกประการ)**:

```
High-value open order - needs approval
Unknown order status
```

### เปรียบเทียบก่อน/หลัง Refactor ตาม Code Review Checklist

| หัวข้อจาก Checklist (ขั้นตอนที่ 978) | ก่อน Refactor | หลัง Refactor |
|---|---|---|
| ไม่มี `GO TO` ที่ใช้ข้ามโค้ด | ❌ มี `GO TO SKIP-MESSAGE` | ✅ ใช้ `WHEN WS-ORDER-STATUS = "O" CONTINUE` แทน |
| ไม่มี Magic Number | ❌ `10000.00` ฝังตรง ๆ 1 จุด | ✅ ใช้ `WS-HIGH-VALUE-LIMIT` |
| โครงสร้างไม่ซ้อนลึก | ❌ `IF` ซ้อน 3 ชั้น | ✅ `EVALUATE TRUE` แบบเรียบ ระดับเดียว |
| ผลลัพธ์ถูกต้องตรงกัน | ✅ | ✅ (พิสูจน์ด้วยการทดสอบ input เดียวกัน) |

### บทเรียนสำคัญที่สุดของแบบฝึกหัดนี้

**การ Refactor ที่ดีไม่เปลี่ยนพฤติกรรมของโปรแกรมแม้แต่น้อย** — สังเกตว่าผลลัพธ์จากทั้งสองเวอร์ชัน
เหมือนกันทุกตัวอักษร นี่คือหลักการ **Behavior-Preserving Refactoring** ที่ Part 081/084/085 เน้นย้ำ
มาตลอด: การทำให้โค้ดสะอาดขึ้นกับการทำให้โค้ดทำงานถูกต้องขึ้นเป็นคนละเรื่องกัน Refactoring คือการ
ปรับปรุง**โครงสร้างภายใน**โดยไม่แตะ**พฤติกรรมภายนอก** — ถ้าต้องการเปลี่ยนพฤติกรรมด้วย ให้แยกเป็น
commit/PR คนละชุดกับการ Refactor เสมอ เพื่อให้ตรวจสอบได้ง่ายว่าอะไรเป็นสาเหตุของการเปลี่ยนแปลงจริง

### ข้อควรระวัง

- ในโค้ดจริงที่ซับซ้อนกว่าตัวอย่างนี้มาก การพิสูจน์ว่า "พฤติกรรมเหมือนเดิม" ต้องอาศัย Automated Test
  ที่ครอบคลุม (Part 079) ไม่ใช่แค่การรันด้วยตาดู 1-2 กรณีเหมือนตัวอย่างนี้ — ยิ่งโค้ดซับซ้อน ยิ่งต้อง
  พึ่งพา Test Suite ที่ครอบคลุมมากขึ้นเท่านั้น ไม่ใช่น้อยลง
- Checklist และหลักการทั้งหมดใน Part นี้ไม่ได้มีไว้ให้ทำตามแบบเคร่งครัดจนขาดวิจารณญาณ — เป้าหมาย
  สูงสุดคือ**โค้ดที่ทีมของคุณอ่านและแก้ไขได้อย่างมั่นใจ** ไม่ใช่การทำตาม rule เพื่อ rule เอง

### แบบฝึกหัดที่ 980.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงเลือกโปรแกรม COBOL หนึ่งไฟล์ที่คุณเคยเขียนในเฟสก่อนหน้าของหลักสูตรนี้ (Part 015, 035, 050,
070 หรือ 085) แล้วนำ Code Review Checklist จากขั้นตอนที่ 978 ไปตรวจสอบทีละข้อ บันทึกว่าผ่าน/ไม่ผ่าน
ข้อใดบ้าง และถ้าไม่ผ่าน จงลองแก้ไขให้ผ่านโดยพิสูจน์ด้วยการคอมไพล์-รันเปรียบเทียบผลลัพธ์ก่อน/หลัง

**เฉลยแนวทาง**: ไม่มีคำตอบตายตัวเพราะขึ้นกับโค้ดที่ผู้เรียนแต่ละคนเลือก แต่กระบวนการที่ถูกต้องคือ:
(1) รันโปรแกรมเดิมเก็บผลลัพธ์ไว้เป็น Golden Master ก่อนแก้ไขใด ๆ (2) ไล่ Checklist ทีละข้อ บันทึก
จุดที่ไม่ผ่าน (3) แก้ไขทีละจุดเล็ก ๆ (4) คอมไพล์-รันเปรียบเทียบผลลัพธ์กับ Golden Master ทุกครั้งหลัง
แก้ไข เพื่อยืนยันว่ายังคงพฤติกรรมเดิมไว้ครบถ้วน — นี่คือกระบวนการเดียวกันกับที่ Part 085 ใช้ Refactor
`ARLEGACY.cob` และที่ Part 100 ใช้แก้บั๊กจริงตลอดการพัฒนา Capstone Project

---

## สรุปท้ายบท

Part 098 ไม่ได้สอนไวยากรณ์ใหม่แม้แต่คำสั่งเดียว แต่รวบรวม**วินัยการเขียนโค้ด**ที่แยกโปรแกรมเมอร์มือ
อาชีพออกจากผู้ที่แค่ "รู้ไวยากรณ์" — สิ่งที่เราได้เรียนรู้ในบทนี้:

- หลักการตั้งชื่อ (`WS-`, `LK-`, `88-level`, Level 78) ที่ใช้สม่ำเสมอตลอดทั้ง 97 Part ก่อนหน้า
  (ขั้นตอนที่ 971)
- วินัยการเขียนคอมเมนต์ที่อธิบาย "ทำไม" ไม่ใช่ "ทำอะไร" พร้อมย้ำกฎเหล็กเรื่องภาษา ASCII ในซอร์สโค้ด
  (ขั้นตอนที่ 972)
- Defensive Programming สองแบบหลัก: ตรวจสอบข้อมูลนำเข้า (ขั้นตอนที่ 973) และตรวจสอบขอบเขตตาราง
  (ขั้นตอนที่ 974)
- Copybook Governance สำหรับทีมขนาดใหญ่ที่มีโปรแกรมหลายสิบไฟล์ใช้ Copybook ร่วมกัน (ขั้นตอนที่ 975)
- **บั๊กจริงอีกตัวหนึ่งที่พบระหว่างเตรียม Part 100**: `CLOSE` เขียนทับ `FILE STATUS` ของ `READ`
  ที่ล้มเหลว พร้อมวิธีแก้ที่พิสูจน์แล้ว (ขั้นตอนที่ 976)
- ตาราง "กับดักยอดฮิต" 12 รายการที่หลักสูตรนี้พิสูจน์จริงตลอด 97 Part พร้อมอ้างอิงย้อนกลับ
  (ขั้นตอนที่ 977)
- Code Review Checklist ที่ใช้งานได้จริงภายใน 15-20 นาทีต่อ Pull Request (ขั้นตอนที่ 978)
- หลักการ Maintainability: Single Responsibility, Flat over Nested, Named Constants
  (ขั้นตอนที่ 979)
- แบบฝึกหัด Refactor เต็มรูปแบบที่รวมทุกหลักการเข้าด้วยกัน พร้อมพิสูจน์ Behavior-Preserving
  Refactoring ด้วยการทดสอบจริง (ขั้นตอนที่ 980)

มาตรฐานเหล่านี้ไม่ใช่กฎที่ตายตัวหรือศักดิ์สิทธิ์ — แต่เป็น**บทสรุปของบทเรียนที่เจ็บปวดจริง**ที่กลั่นกรอง
มาจากการเขียน ทดสอบ และแก้บั๊กจริงตลอดการพัฒนาหลักสูตรนี้ทั้ง 100 Part เมื่อคุณนำหลักการเหล่านี้ไปใช้
ในงานจริง คุณจะเขียนโค้ดที่ไม่ใช่แค่ "ทำงานได้ในวันนี้" แต่ "ทำงานได้และดูแลรักษาง่ายไปอีกหลายปี" —
ซึ่งคือสิ่งที่แยกโค้ดระดับมืออาชีพออกจากโค้ดระดับผู้เริ่มต้นอย่างแท้จริง

Part ถัดไปจะพาเรามองไปข้างหน้า: อนาคตของ COBOL จะเป็นอย่างไรในโลกที่ AI เข้ามามีบทบาทมากขึ้นเรื่อย ๆ
และทักษะอะไรที่ควรจับคู่กับความเชี่ยวชาญ COBOL เพื่อความมั่นคงทางอาชีพในระยะยาว

**[← กลับไป Part 097](part-097-interview-questions.md)** | **[ไปยัง Part 099: อนาคตของ COBOL และเทรนด์เทคโนโลยี →](part-099-future-of-cobol.md)**
