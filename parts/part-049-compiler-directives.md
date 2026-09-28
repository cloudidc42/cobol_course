# Part 049: Compiler Directives และความแตกต่างระหว่าง COBOL Dialects (ขั้นตอนที่ 481–490)

## คำนำของ Part นี้

ตลอด 48 Part ที่ผ่านมา เราเขียนและคอมไพล์โค้ด COBOL ด้วย `cobc` โดยแทบไม่เคยตั้งค่าอะไรเป็นพิเศษ
นอกจาก `-x -o` เพื่อสร้างโปรแกรม executable แต่ในโลกจริง COBOL ถูกใช้งานมานานกว่า 65 ปีบน compiler
และแพลตฟอร์มที่หลากหลายมาก — IBM Enterprise COBOL บน z/OS, Micro Focus COBOL บน Windows/Unix,
GnuCOBOL แบบ Open Source, และอีกหลายสิบ vendor ตลอดประวัติศาสตร์ COBOL แต่ละเจ้าต่างก็มี "สำเนียง"
(**Dialect**) ของตัวเองที่แตกต่างกันในรายละเอียดปลีกย่อย ไม่ว่าจะเป็นคำสงวนพิเศษ พฤติกรรมของชนิดข้อมูล
บางแบบ หรือกฎความยาวชื่อตัวแปร

Part นี้จะพาคุณเรียนรู้เครื่องมือสองกลุ่มที่ COBOL มีให้จัดการกับความหลากหลายนี้:

1. **Compiler Directives** (`>>SET`, `$SET`, `>>DEFINE`, `>>IF`/`>>ELSE`/`>>END-IF`) — คำสั่งพิเศษ
   ที่ฝังอยู่ในซอร์สโค้ดเอง ใช้ควบคุมพฤติกรรมการคอมไพล์แบบ **Conditional Compilation** (คอมไพล์โค้ด
   ส่วนต่างกันตามเงื่อนไข คล้าย `#ifdef` ในภาษา C)
2. **Dialect Flags** (`-std=ibm`, `-std=mf`, `-std=cobol2014` ฯลฯ) — ตัวเลือกตอนคอมไพล์ที่บอก
   GnuCOBOL ให้ **เลียนแบบพฤติกรรมของ compiler เจ้าอื่น** ทำให้โค้ดที่เขียนมาสำหรับ Mainframe จริง
   สามารถทดสอบเบื้องต้นบน GnuCOBOL ได้ก่อนนำไปใช้งานจริงบนเครื่องเป้าหมาย

ทุกตัวอย่างและทุกความแตกต่างระหว่าง Dialect ในเอกสารนี้**ผ่านการทดสอบจริง**ด้วย GnuCOBOL
4.0-early-dev.0 — รวมถึงกรณีที่คอมไพเลอร์**ให้ผลลัพธ์ต่างกันจริง**เมื่อใช้ dialect flag ต่างกัน
(ไม่ใช่แค่คำอธิบายทางทฤษฎี) และกรณีที่ฟีเจอร์บางตัวมีอยู่ในมาตรฐานแต่**ยังไม่ได้ implement จริง**ใน
GnuCOBOL รุ่นนี้ ซึ่งจะระบุไว้อย่างตรงไปตรงมาเมื่อถึงจุดนั้น

---

## ขั้นตอนที่ 481: Compiler Directive คืออะไร และภาพรวมกลุ่มคำสั่ง

### แนวคิด

**Compiler Directive** คือคำสั่งพิเศษที่**ไม่ใช่ COBOL Statement ปกติ** — มันสั่งงาน**ตัวคอมไพเลอร์เอง**
ณ เวลาคอมไพล์ ไม่ใช่สั่งงานโปรแกรมตอนรัน (runtime) ข้อสังเกตสำคัญที่แยก Directive ออกจากคำสั่ง COBOL
ทั่วไปคือ **Directive ไม่ต้องมี period (`.`) ปิดท้ายเหมือนประโยค COBOL ปกติ** และมีรูปแบบการเขียนสองแบบ
หลักที่ GnuCOBOL รองรับ:

| รูปแบบ | ตัวอย่าง | ที่มา |
|---|---|---|
| `>>` (COBOL 2002/2014 มาตรฐาน) | `>>SOURCE FORMAT FREE` | มาตรฐาน ISO COBOL สมัยใหม่ |
| `$` (สไตล์ Micro Focus ดั้งเดิม) | `$SET SOURCEFORMAT(FIXED)` | Micro Focus COBOL (รองรับไว้เพื่อความเข้ากันได้) |

GnuCOBOL รองรับทั้งสองรูปแบบเพื่อให้เข้ากันได้กับซอร์สโค้ด Legacy ที่มาจาก vendor ต่างกัน — โค้ดใหม่
ที่เขียนขึ้นในปัจจุบันมักนิยมใช้รูปแบบ `>>` เพราะเป็นมาตรฐานสากลที่ชัดเจนกว่า

### กลุ่มคำสั่ง Directive หลักที่จะเรียนใน Part นี้

1. **Source Format Directive** — เปลี่ยนรูปแบบการอ่านคอลัมน์ของซอร์สโค้ด (Fixed/Free) กลางไฟล์ได้
2. **Conditional Compilation Directive** (`>>DEFINE`, `>>IF`/`>>ELSE`/`>>END-IF`) — คอมไพล์โค้ดคนละ
   ส่วนตามเงื่อนไขที่กำหนดไว้ ณ เวลาคอมไพล์
3. **Dialect/Standard Flag** (`-std=`) — ไม่ใช่ Directive ในซอร์สโค้ด แต่เป็น command-line option
   ที่ส่งผลกว้างกว่า Directive มาก (เปลี่ยนพฤติกรรมทั้งไฟล์ รวมถึงคำสงวนและกฎภาษา)

### ตัวอย่างโค้ด: พิสูจน์ว่า Directive ไม่ใช่ COBOL Statement

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP481-DIRECTIVE-INTRO.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MSG                   PIC X(40).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> >>DEFINE below is a COMPILER DIRECTIVE - notice it has NO
      *> period at the end, unlike every COBOL statement we have
      *> written in this course so far.
       >>DEFINE COURSE-STAGE AS "ADVANCED"
           MOVE "DIRECTIVE PROCESSED AT COMPILE TIME" TO WS-MSG
           DISPLAY WS-MSG.
           STOP RUN.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step481 step481.cob
./step481
```

### ผลลัพธ์จริงที่ได้

```
DIRECTIVE PROCESSED AT COMPILE TIME
```

### อธิบายโค้ดทีละส่วน

- `>>DEFINE COURSE-STAGE AS "ADVANCED"` ไม่ได้ทำให้เกิดผลลัพธ์อะไรที่มองเห็นได้ในโปรแกรมนี้ (ยังไม่มี
  การใช้งานค่านี้ต่อ) — วัตถุประสงค์ของขั้นตอนนี้คือแค่แสดงให้เห็นว่า**โค้ดยังคอมไพล์ผ่านได้ปกติแม้จะมี
  บรรทัดที่ไม่มี period ปิดท้ายปะปนอยู่** เพราะคอมไพเลอร์รู้จักบรรทัดที่ขึ้นต้นด้วย `>>` ว่าเป็น
  Directive ไม่ใช่ COBOL Statement ตั้งแต่แรก จึงไม่บังคับกฎ period ของ COBOL Statement กับมัน
- ในขั้นตอนถัดไปเราจะเริ่มใช้งานค่าที่ `>>DEFINE` ประกาศไว้จริง ๆ ผ่าน `>>IF`

### ข้อควรระวัง

- อย่าใส่ period (`.`) ท้ายบรรทัด Directive โดยเข้าใจผิดว่าเป็นกฎเดียวกับ COBOL Statement — การใส่
  period ท้าย Directive จะทำให้คอมไพเลอร์ตีความผิดและอาจเกิด syntax error
- Directive ที่ขึ้นต้นด้วย `>>` ต้องอยู่**คนละบรรทัด**กับ COBOL Statement เสมอ ไม่สามารถเขียนแทรก
  กลางบรรทัดเดียวกับคำสั่งอื่นได้

### แบบฝึกหัดที่ 481.1

**โจทย์**: จงอธิบายความแตกต่างเชิงแนวคิดระหว่าง "Compiler Directive" กับ "COBOL Statement ทั่วไป
เช่น `MOVE`/`DISPLAY`" ในแง่ของ**เวลาที่มันมีผล**

**เฉลย**: COBOL Statement ทั่วไป เช่น `MOVE`/`DISPLAY` จะถูกแปลงเป็นโค้ดเครื่อง (machine code) ที่
ทำงาน**ตอนโปรแกรมรัน (runtime)** — มันมีผลก็ต่อเมื่อโปรแกรมถูกเรียกทำงานจริงเท่านั้น ในขณะที่
Compiler Directive มีผล**ตอนคอมไพล์ (compile time)** เท่านั้น มันควบคุมว่าคอมไพเลอร์จะแปลงส่วนไหน
ของซอร์สโค้ดเป็นโปรแกรมบ้าง หรือจะตีความคอลัมน์ของไฟล์อย่างไร — เมื่อคอมไพล์เสร็จแล้ว Directive จะ
"หายไป" ไม่เหลือร่องรอยในโปรแกรม executable เลย ต่างจาก Statement ที่กลายเป็นส่วนหนึ่งของโปรแกรมที่รัน

---

## ขั้นตอนที่ 482: SOURCE FORMAT Directive — สลับ Fixed/Free Format กลางไฟล์

### แนวคิด

Part 002 สอนไว้ว่า GnuCOBOL รองรับทั้ง Fixed-Format (คอลัมน์ 8-72 ที่หลักสูตรนี้ใช้เป็นหลัก) และ
Free-Format (ไม่บังคับคอลัมน์ เหมือนภาษาสมัยใหม่ทั่วไป) ขั้นตอนนี้จะแสดงวิธี**สลับโหมดด้วย Directive**
ซึ่งมีประโยชน์เมื่อต้องรวมโค้ด Legacy แบบ Fixed-Format เข้ากับโค้ดใหม่แบบ Free-Format ไว้ในระบบเดียวกัน

### วิธีที่ 1: `>>SOURCE FORMAT FREE` (มาตรฐาน)

```cobol
      *> This line is still Fixed-Format (column 8-72 rules apply),
      *> which is exactly why the directive line itself must start
      *> at column 8, like any other Area A/B COBOL line.
       >>SOURCE FORMAT FREE
IDENTIFICATION DIVISION.
PROGRAM-ID. STEP482-FREE-FORMAT.
AUTHOR. COBOL-COURSE.
DATA DIVISION.
WORKING-STORAGE SECTION.
01 WS-MSG PIC X(30) VALUE "NOW RUNNING IN FREE FORMAT".
PROCEDURE DIVISION.
MAIN-PARA.
    DISPLAY WS-MSG.
    STOP RUN.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step482 step482.cob
./step482
```

### ผลลัพธ์จริงที่ได้

```
NOW RUNNING IN FREE FORMAT   
```

### วิธีที่ 2: `-free` command-line flag (ทั้งไฟล์)

หากต้องการให้ **ทั้งไฟล์** เป็น Free-Format ตั้งแต่บรรทัดแรกโดยไม่ต้องมี Directive ในไฟล์เลย
ใช้ flag `-free` ตอนคอมไพล์แทนได้:

```cobol
IDENTIFICATION DIVISION.
PROGRAM-ID. STEP482B-FREE-VIA-FLAG.
DATA DIVISION.
WORKING-STORAGE SECTION.
01 WS-MSG PIC X(30) VALUE "FREE FORMAT VIA -free FLAG".
PROCEDURE DIVISION.
MAIN-PARA.
    DISPLAY WS-MSG.
    STOP RUN.
```

```bash
cobc -free -x -o step482b step482b.cob
./step482b
```

**ผลลัพธ์จริงที่ได้**: `FREE FORMAT VIA -free FLAG`

### วิธีที่ 3: `$SET SOURCEFORMAT(...)` (สไตล์ Micro Focus ดั้งเดิม)

```cobol
      $SET SOURCEFORMAT(FIXED)
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP482C-DOLLAR-SET.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MSG                   PIC X(30)
                                     VALUE "DOLLAR-SET DIRECTIVE WORKS".

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY WS-MSG.
           STOP RUN.
```

**ผลลัพธ์จริงที่ได้**: `DOLLAR-SET DIRECTIVE WORKS`

### อธิบายโค้ดทีละส่วน

- **จุดที่ต้องระวังที่สุดของวิธีที่ 1**: บรรทัด `>>SOURCE FORMAT FREE` เอง **ยังต้องเขียนตามกฎ
  Fixed-Format** (เริ่มที่คอลัมน์ 8 เป็นต้นไป) เพราะ ณ ตำแหน่งนั้นคอมไพเลอร์ยังไม่รู้ว่าไฟล์นี้จะ
  เปลี่ยนเป็น Free-Format — มันอ่านบรรทัดนั้นด้วยกฎ Fixed-Format ก่อน แล้วค่อยเปลี่ยนโหมดสำหรับบรรทัด
  **ถัดจากนั้น**ไป (ทดสอบยืนยันแล้วว่าหากวาง `>>SOURCE FORMAT FREE` ไว้ที่คอลัมน์ 1 โดยไม่มีการเยื้อง
  เลย จะเกิด `error: invalid indicator` ทันที เพราะคอมไพเลอร์ยังตีความคอลัมน์ 7 ตามกฎ Fixed-Format
  อยู่)
- วิธีที่ 2 (`-free` flag) สะดวกกว่าเมื่อทั้งไฟล์เป็น Free-Format ตั้งแต่ต้นจนจบ ไม่ต้องกังวลเรื่อง
  การเยื้องบรรทัดแรกเลย
- วิธีที่ 3 (`$SET`) แสดงให้เห็นว่า GnuCOBOL ยอมรับ syntax สไตล์ Micro Focus ได้ด้วย เพื่อความเข้ากันได้
  กับโค้ด Legacy ที่มาจาก Micro Focus COBOL โดยตรง

### ข้อควรระวัง

- **อย่าผสมทั้งสามวิธีในไฟล์เดียวกัน** จนกลายเป็นความสับสน ควรเลือกวิธีเดียวให้สอดคล้องกับมาตรฐาน
  ของทีม/องค์กร — หลักสูตรนี้ยังคงใช้ Fixed-Format เป็นค่าเริ่มต้นตลอดทุก Part (ตามที่ระบุใน
  `docs/COURSE-OUTLINE.md`) วิธีการเหล่านี้เป็นความรู้เสริมสำหรับกรณีที่ต้องทำงานกับโค้ด Legacy
  ที่เขียนในรูปแบบอื่น
- Directive มีผล**นับจากตำแหน่งที่ประกาศเป็นต้นไปในไฟล์นั้น** ไม่ใช่ทั้งไฟล์ย้อนหลัง — สามารถมีไฟล์
  ที่ส่วนต้นเป็น Fixed-Format แล้วสลับเป็น Free-Format กลางไฟล์ได้จริงด้วย Directive หลายจุด (แม้จะ
  ไม่แนะนำให้ทำในโค้ด production เพราะทำให้อ่านสับสน)

### แบบฝึกหัดที่ 482.1

**โจทย์**: จงอธิบายว่าทำไมบรรทัด `>>SOURCE FORMAT FREE` เองจึงต้องเขียนตามกฎ Fixed-Format
(เริ่มคอลัมน์ 8) ทั้งที่จุดประสงค์ของมันคือการเปลี่ยนไปใช้ Free-Format

**เฉลย**: เพราะคอมไพเลอร์ต้อง**อ่านและตีความ Directive นั้นก่อน**จึงจะรู้ว่าต้องเปลี่ยนโหมดการอ่าน
คอลัมน์ของบรรทัด**ถัดไป** — ณ ขณะที่กำลังอ่านบรรทัดที่มี Directive นี้อยู่ คอมไพเลอร์ยังคงใช้กฎ
Fixed-Format เดิม (ซึ่งเป็นโหมดเริ่มต้นของไฟล์) อยู่เสมอ การเปลี่ยนแปลงโหมดจะมีผล**หลังจาก**ประมวลผล
Directive นั้นเสร็จสิ้นแล้วเท่านั้น เปรียบเทียบได้กับการที่ต้อง "อ่านป้ายบอกทางก่อนจะรู้ว่าต้องเลี้ยว
ไปทางไหน" — ป้ายบอกทางเองก็ต้องอยู่ในตำแหน่งที่มองเห็นได้ตามกฎเดิมก่อนเสมอ

---

## ขั้นตอนที่ 483: >>DEFINE และ >>IF/>>ELSE/>>END-IF — Conditional Compilation พื้นฐาน

### แนวคิด

**Conditional Compilation** คือการให้คอมไพเลอร์**เลือกคอมไพล์โค้ดคนละส่วนกัน**ตามเงื่อนไขที่กำหนด
ไว้ ณ เวลาคอมไพล์ — เหมือนกับ `#ifdef`/`#else`/`#endif` ในภาษา C มีประโยชน์มากสำหรับสถานการณ์เช่น
"โปรแกรมนี้ต้องมีโค้ดพิเศษสำหรับ debug เฉพาะตอนพัฒนา แต่ไม่ต้องการให้โค้ดนั้นติดไปกับเวอร์ชัน production"

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP483-CONDITIONAL-COMPILE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MSG                   PIC X(40).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> >>DEFINE creates a compile-time constant/flag. It exists
      *> only during compilation - it is NOT a WORKING-STORAGE item
      *> and consumes no runtime memory at all.
       >>DEFINE BUILD-MODE AS "PRODUCTION"
       >>IF BUILD-MODE EQUAL "PRODUCTION"
           MOVE "RUNNING IN PRODUCTION MODE" TO WS-MSG
       >>ELSE
           MOVE "RUNNING IN TEST MODE - EXTRA CHECKS ACTIVE"
               TO WS-MSG
       >>END-IF
           DISPLAY WS-MSG.
           STOP RUN.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step483 step483.cob
./step483
```

### ผลลัพธ์จริงที่ได้

```
RUNNING IN PRODUCTION MODE
```

### ทดลองเปลี่ยนค่าและคอมไพล์ใหม่

เปลี่ยนบรรทัด `>>DEFINE BUILD-MODE AS "PRODUCTION"` เป็น `>>DEFINE BUILD-MODE AS "TEST"` แล้ว
คอมไพล์ใหม่ ผลลัพธ์จะเปลี่ยนเป็น `RUNNING IN TEST MODE - EXTRA CHECKS ACTIVE` ทันที **โดยที่ตัว
โปรแกรมตอนรัน (runtime) ไม่มีการตรวจสอบเงื่อนไขใด ๆ เลยสักนิด** — โค้ดของสาขาที่ไม่ถูกเลือกจะ**ไม่ถูก
คอมไพล์เข้าไปในโปรแกรมด้วยซ้ำ** ต่างจาก `IF`/`EVALUATE` ปกติที่ทั้งสองสาขาจะถูกคอมไพล์เข้าไปในโปรแกรม
เสมอ แล้วค่อยเลือกทำงานตอนรัน

### อธิบายโค้ดทีละส่วน

- `>>DEFINE BUILD-MODE AS "PRODUCTION"` สร้างค่าคงที่ระดับคอมไพเลอร์ชื่อ `BUILD-MODE`
- `>>IF BUILD-MODE EQUAL "PRODUCTION"` เปรียบเทียบค่าที่ `>>DEFINE` ไว้ ถ้าตรงกัน จะคอมไพล์เฉพาะโค้ด
  ในสาขานั้น สาขา `>>ELSE` จะถูก**ข้ามไปทั้งหมด ไม่ถูกแปลงเป็นโค้ดเครื่องเลย**
- ตัวดำเนินการเปรียบเทียบที่ `>>IF` รองรับได้แก่ `EQUAL`, `NOT EQUAL` (หรือ `=`, `<>`) และยังรองรับ
  `DEFINED`/`NOT DEFINED` เพื่อเช็คว่ามีการประกาศค่านั้นไว้หรือยัง (จะสาธิตในขั้นตอนที่ 484)

### ข้อควรระวัง

- `>>DEFINE` ที่ประกาศไว้ในไฟล์หนึ่งมีผล**เฉพาะไฟล์นั้น**เท่านั้น (ยกเว้นใช้ร่วมกับ `COPY`/`REPLACE`
  ที่ดึงมาจากไฟล์อื่น ซึ่งเป็นเรื่องขั้นสูงกว่าที่ไม่ครอบคลุมในเอกสารนี้) ไม่ใช่ global variable ที่
  แชร์ข้ามไฟล์ source อัตโนมัติ
- ค่าคงที่จาก `>>DEFINE` เป็นคนละเรื่องกับตัวแปร WORKING-STORAGE โดยสิ้นเชิง — ไม่สามารถ `DISPLAY`
  ค่าคงที่นี้ตอนรันได้โดยตรง (ต้องใช้มันผ่าน `>>IF` เพื่อกำหนด logic ที่จะคอมไพล์เข้าไปแทน)

### แบบฝึกหัดที่ 483.1

**โจทย์**: จงอธิบายข้อดีของการใช้ `>>IF`/`>>ELSE` แยกโค้ด debug ออกจากโค้ด production เปรียบเทียบ
กับการใช้ `IF`/`ELSE` ปกติที่ตรวจสอบ flag ตัวแปรตอนรัน

**เฉลยแนวทาง**: ข้อดีหลักคือ **ขนาดและประสิทธิภาพของโปรแกรม production** — เมื่อใช้ `>>IF`/`>>ELSE`
โค้ด debug ที่ไม่ถูกเลือกจะไม่ถูกคอมไพล์เข้าไปในโปรแกรมเลยแม้แต่ไบต์เดียว ทำให้โปรแกรม production
มีขนาดเล็กกว่าและไม่มี overhead ใด ๆ จากการตรวจสอบเงื่อนไขตอนรัน ในขณะที่การใช้ `IF`/`ELSE` ปกติ
โค้ดทั้งสองสาขาจะถูกคอมไพล์เข้าไปในโปรแกรมเสมอ (แม้ในทางปฏิบัติจะไม่ทำงานเพราะเงื่อนไขเป็นเท็จ) ทำให้
โปรแกรมมีขนาดใหญ่กว่าโดยไม่จำเป็น และในบางกรณีโค้ด debug อาจมีความเสี่ยงด้านความปลอดภัย (เช่น
ข้อความ debug ที่เปิดเผยข้อมูลภายใน) ที่ไม่ควรมีโอกาสถูกเรียกใช้ในโปรแกรม production เลยแม้แต่ทาง
ทฤษฎี — การตัดออกตั้งแต่ตอนคอมไพล์จึงปลอดภัยกว่า

---

## ขั้นตอนที่ 484: กำหนดค่าจาก Command Line ด้วย -D และตรวจสอบด้วย DEFINED

### แนวคิด

นอกจากกำหนดค่าคงที่ด้วย `>>DEFINE` ในซอร์สโค้ดแล้ว เรายังสามารถ**ส่งค่าเข้ามาจากภายนอกตอนคอมไพล์**
ผ่าน command-line flag `-D` ได้ — มีประโยชน์มากเมื่อต้องการคอมไพล์**ซอร์สโค้ดเดียวกัน**ให้ได้ผลลัพธ์
ต่างกันสำหรับหลายสภาพแวดล้อม (เช่น DEV/UAT/PRODUCTION) โดยไม่ต้องแก้ไขซอร์สโค้ดเลย

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP484-CLI-DEFINE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MSG                   PIC X(40).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> DEFINED checks whether a compile-time name was set at all
      *> - whether by >>DEFINE in the source, or by -D on the cobc
      *> command line. It does not care about the VALUE, only
      *> whether the name exists.
       >>IF BUILD-MODE DEFINED
           MOVE "BUILD-MODE WAS SET FROM THE COMMAND LINE"
               TO WS-MSG
       >>ELSE
           MOVE "BUILD-MODE WAS NOT SET AT ALL" TO WS-MSG
       >>END-IF
           DISPLAY WS-MSG.
           STOP RUN.
```

คอมไพล์สองแบบเพื่อเปรียบเทียบ:

```bash
echo "=== WITHOUT -D ==="
cobc -x -o step484a step484.cob
./step484a

echo "=== WITH -DBUILD-MODE=UAT ==="
cobc -x -DBUILD-MODE=UAT -o step484b step484.cob
./step484b
```

### ผลลัพธ์จริงที่ได้

```
=== WITHOUT -D ===
BUILD-MODE WAS NOT SET AT ALL

=== WITH -DBUILD-MODE=UAT ===
BUILD-MODE WAS SET FROM THE COMMAND LINE
```

### อธิบายโค้ดทีละส่วน

- ซอร์สโค้ด `step484.cob` **ไม่ถูกแก้ไขเลยแม้แต่ตัวอักษรเดียว** ระหว่างการคอมไพล์ทั้งสองครั้ง —
  ความแตกต่างของผลลัพธ์มาจาก flag `-D` ที่ใส่ตอนคอมไพล์เท่านั้น
- `-DBUILD-MODE=UAT` ทำสิ่งเดียวกับการเขียน `>>DEFINE BUILD-MODE AS "UAT"` ไว้ในไฟล์ แต่มาจาก
  ภายนอกไฟล์แทน ทำให้ script การ build (เช่น Makefile หรือ CI/CD pipeline ที่จะเรียนใน Part 078)
  สามารถควบคุมได้ว่าจะคอมไพล์เป็นเวอร์ชันไหนโดยไม่ต้องแก้ไฟล์ source เลย
- `DEFINED` ตรวจสอบแค่ "มีการตั้งชื่อนี้ไว้หรือไม่" ไม่ได้สนใจค่าที่ตั้งไว้ — ถ้าต้องการเปรียบเทียบค่า
  จริงด้วย ต้องรวมกับ `EQUAL` เช่น `>>IF BUILD-MODE DEFINED AND BUILD-MODE EQUAL "UAT"` (การผสม
  เงื่อนไขแบบนี้รองรับตามมาตรฐาน)

### ข้อควรระวัง

- ชื่อค่าคงที่หลัง `-D` **ไม่แยกตัวพิมพ์เล็ก-ใหญ่ (case-insensitive)** เหมือนคำสงวนอื่น ๆ ของ COBOL
  ทั่วไป ควรตั้งชื่อให้สื่อความหมายชัดเจนและสอดคล้องกับที่ใช้ใน `>>IF` ในซอร์สโค้ด
- อย่าลืมว่า `-D` เป็น flag ระดับ **การคอมไพล์ครั้งนั้น ๆ เท่านั้น** ถ้าลืมใส่ตอน build จริง โปรแกรม
  ที่ได้จะใช้ path ของ `>>ELSE` แทนโดยไม่มีข้อความเตือนใด ๆ — ควรมีระบบ build ที่ชัดเจน (เช่น script
  หรือ CI/CD) แทนการพิมพ์คำสั่งคอมไพล์ด้วยมือในโปรแกรมระดับองค์กรจริง

### แบบฝึกหัดที่ 484.1

**โจทย์**: จงยกตัวอย่างสถานการณ์จริงในองค์กรที่การใช้ `-D` ร่วมกับ `>>IF ... DEFINED` จะมีประโยชน์
มากกว่าการแก้ไขค่าคงที่ในซอร์สโค้ดโดยตรงทุกครั้งก่อน build

**เฉลยแนวทาง**: สถานการณ์ทั่วไปคือระบบที่ต้อง build โปรแกรมเดียวกันสำหรับหลายสภาพแวดล้อม
(Development, UAT/Testing, Production) โดยแต่ละสภาพแวดล้อมอาจต้องการพฤติกรรมต่างกันเล็กน้อย เช่น
เปิด/ปิดข้อความ debug พิเศษ หรือชี้ไปยังชื่อไฟล์ log คนละชื่อ หากใช้ `-D` ร่วมกับ Pipeline การ build
อัตโนมัติ (CI/CD) ทีมสามารถกำหนดค่าผ่าน script การ build ให้ต่างกันตามสภาพแวดล้อมเป้าหมายได้โดย
**ไม่ต้องแตะซอร์สโค้ดเลย** ลดความเสี่ยงที่จะลืมแก้ค่าคงที่กลับก่อน deploy ขึ้น production จริง
(ซึ่งเป็นบั๊กที่พบได้บ่อยเมื่อต้องแก้ไขค่าในซอร์สโค้ดด้วยมือทุกครั้ง)

---

## ขั้นตอนที่ 485: -std= Dialect Flag คืออะไร และ Dialect ที่ GnuCOBOL รองรับ

### แนวคิด

ต่างจาก Directive ในซอร์สโค้ดที่ส่งผลเฉพาะจุด `-std=<dialect>` เป็น **command-line option ที่
เปลี่ยนกฎการตีความภาษา COBOL ทั้งไฟล์** ให้เลียนแบบพฤติกรรมของ compiler ค่ายอื่น หรือมาตรฐานปีอื่น
ตรวจสอบรายชื่อ Dialect ที่ GnuCOBOL ในเครื่องรองรับได้จากไฟล์ config จริงที่ติดตั้งมากับ compiler:

```bash
ls /etc/gnucobol/*.conf
```

### รายชื่อ Dialect จริงที่มีอยู่ในสภาพแวดล้อมของหลักสูตรนี้ (ตรวจสอบแล้ว)

| ชื่อไฟล์ config | ใช้กับ `-std=` | คือการเลียนแบบ |
|---|---|---|
| `default.conf` | `-std=default` (ค่าเริ่มต้น) | พฤติกรรมมาตรฐานของ GnuCOBOL เอง |
| `cobol85.conf` | `-std=cobol85` | มาตรฐาน ANSI COBOL-85 |
| `cobol2002.conf` | `-std=cobol2002` | มาตรฐาน ISO COBOL-2002 |
| `cobol2014.conf` | `-std=cobol2014` | มาตรฐาน ISO COBOL-2014 (ใหม่ล่าสุด) |
| `ibm.conf` / `ibm-strict.conf` | `-std=ibm` / `-std=ibm-strict` | IBM Enterprise COBOL (Mainframe z/OS) |
| `mf.conf` / `mf-strict.conf` | `-std=mf` / `-std=mf-strict` | Micro Focus COBOL |
| `mvs.conf` / `mvs-strict.conf` | `-std=mvs` / `-std=mvs-strict` | IBM COBOL รุ่นเก่าบน MVS |
| `acu.conf` / `acu-strict.conf` | `-std=acu` / `-std=acu-strict` | ACUCOBOL (ปัจจุบันคือ AcuCOBOL-GT) |
| `realia.conf` / `realia-strict.conf` | `-std=realia` / `-std=realia-strict` | Realia COBOL |
| `rm.conf` / `rm-strict.conf` | `-std=rm` / `-std=rm-strict` | RM/COBOL |
| `xopen.conf` | `-std=xopen` | X/Open COBOL |
| `bs2000.conf` / `bs2000-strict.conf` | `-std=bs2000` / `-std=bs2000-strict` | Siemens BS2000 COBOL |

รูปแบบ `-strict` หมายถึง "เข้มงวดตามมาตรฐานของ dialect นั้นเป๊ะ ๆ ไม่ผ่อนปรน" ในขณะที่รูปแบบไม่มี
`-strict` (เช่น `ibm` ธรรมดา) จะ**รวมกฎของ `ibm-strict` เข้ามาทั้งหมดแล้วผ่อนปรนเพิ่มเติม** — ตรวจสอบ
ได้จากเนื้อหาไฟล์ `ibm.conf` จริงในเครื่อง:

```bash
cat /etc/gnucobol/ibm.conf
```

**เนื้อหาจริงที่พบ**:

```
include "ibm-strict.conf"

name: "IBM COBOL (lax)"

# reserve additional words, synonyms and exceptions from normal word list
include:		"ibm.words"

include "lax.conf-inc"
```

เห็นได้ชัดว่า `ibm.conf` **ไม่ได้เขียนกฎใหม่ทั้งหมด** แต่ใช้ `include "ibm-strict.conf"` นำกฎทั้งหมด
ของ `ibm-strict` มาใช้ก่อน แล้วค่อยเพิ่มคำสงวนพิเศษจาก `ibm.words` และผ่อนปรนกฎบางส่วนผ่าน
`lax.conf-inc` ทับอีกที — นี่คือสถาปัตยกรรมที่ทำให้ GnuCOBOL ดูแลรักษา Dialect จำนวนมากได้อย่างเป็นระบบ

### ตัวอย่างการใช้งาน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP485-DIALECT-CHECK.
      *> Note: no AUTHOR paragraph here - COBOL-2014 removed it
      *> from the standard, so -std=cobol2014 rejects it outright.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NUM                   PIC 9(4) VALUE 1234.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "REVERSED: " FUNCTION REVERSE(WS-NUM).
           STOP RUN.
```

```bash
cobc -std=cobol2014 -x -o step485 step485.cob
./step485
```

### ผลลัพธ์จริงที่ได้

```
REVERSED: 4321
```

### ข้อควรระวัง

- `-std=` เป็นตัวเลือกที่**ต้องระบุตอนคอมไพล์ทุกครั้ง** ไม่ได้บันทึกไว้ในไฟล์ source เหมือน `>>SET`
  บางประเภท — ถ้าลืมระบุ (หรือทีมลืมใส่ไว้ใน build script) จะกลับไปใช้ `default` โดยไม่มีข้อความเตือน
- ชื่อ dialect ที่ใช้ได้ขึ้นกับว่า**เครื่องที่ติดตั้ง GnuCOBOL มีไฟล์ config นั้นอยู่จริงหรือไม่** — ใน
  บางระบบปฏิบัติการที่ patch หรือ build GnuCOBOL เอง อาจมีรายชื่อ dialect ไม่ครบตามตารางข้างต้น ควร
  ตรวจสอบด้วย `ls /etc/gnucobol/*.conf` (หรือ path ที่ `cobc -info` รายงานไว้ในหัวข้อ `COB_CONFIG_DIR`)
  ก่อนใช้งานจริงเสมอ

### แบบฝึกหัดที่ 485.1

**โจทย์**: จงอธิบายว่าทำไมไฟล์ `ibm.conf` จึงไม่เขียนกฎการตั้งค่าทั้งหมดใหม่ทั้งหมด แต่เลือกใช้
`include "ibm-strict.conf"` แทน

**เฉลยแนวทาง**: เพื่อหลีกเลี่ยงความซ้ำซ้อนและความเสี่ยงที่การตั้งค่าจะไม่สอดคล้องกัน (inconsistency)
ระหว่างสอง config — `ibm.conf` (โหมดผ่อนปรน/lax) กับ `ibm-strict.conf` (โหมดเข้มงวด) มีกฎพื้นฐาน
ร่วมกันเกือบทั้งหมด (เช่น `binary-truncate: no`, `word-length: 30`, `assign-clause: external`)
ต่างกันแค่รายละเอียดเล็กน้อยเรื่องความเข้มงวดของการตรวจสอบ Syntax และคำสงวนเพิ่มเติม การใช้ `include`
ทำให้เมื่อกฎพื้นฐานร่วมกันต้องแก้ไข (เช่น แก้บั๊กหรือปรับปรุงความแม่นยำ) ผู้ดูแล GnuCOBOL แก้ที่เดียว
(`ibm-strict.conf`) แล้วมีผลกับทั้งสอง config โดยอัตโนมัติ — เป็นหลักการ "Don't Repeat Yourself
(DRY)" แบบเดียวกับที่ใช้ในโค้ดโปรแกรมทั่วไป เพียงแต่นำมาใช้กับไฟล์ configuration แทน

---

## ขั้นตอนที่ 486: ความแตกต่างที่พิสูจน์ได้จริง #1 — Binary Truncation (COMP Field Overflow)

### แนวคิด

นี่คือตัวอย่างที่**สำคัญที่สุด**ของ Part นี้: ความแตกต่างระหว่าง Dialect ที่**ส่งผลต่อผลลัพธ์การรันจริง**
ไม่ใช่แค่ทฤษฎี — เกี่ยวกับพฤติกรรมของฟิลด์ `COMP` (Binary) เมื่อค่าที่คำนวณได้**เกินจำนวนหลักที่ระบุใน
PICTURE** การตั้งค่านี้เรียกว่า `binary-truncate` ในไฟล์ config

ตรวจสอบค่าจาก config จริง:

```bash
grep "binary-truncate" /etc/gnucobol/default.conf /etc/gnucobol/ibm-strict.conf
```

**ผลลัพธ์จริง**:

```
/etc/gnucobol/default.conf:binary-truncate:		yes
/etc/gnucobol/ibm-strict.conf:binary-truncate:		no
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP486-BINARY-TRUNCATE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> PIC 9(4) COMP is stored as a 2-byte binary integer, which
      *> can physically hold values from 0 up to 65535 - far more
      *> than the 4 decimal digits the PICTURE clause specifies
      *> (0-9999). "binary-truncate" decides what happens to the
      *> EXTRA capacity beyond what the PICTURE digits describe.
       01  WS-BIN-COUNT             PIC 9(4) COMP VALUE 9999.

       PROCEDURE DIVISION.
       MAIN-PARA.
           ADD 1 TO WS-BIN-COUNT.
           DISPLAY "AFTER ADD 1 TO 9999 COMP: " WS-BIN-COUNT.
           STOP RUN.
```

คอมไพล์ด้วยสอง Dialect ที่ต่างกัน แล้วเปรียบเทียบผลลัพธ์:

```bash
echo "=== -std=default (binary-truncate: yes) ==="
cobc -std=default -x -o step486_default step486.cob
./step486_default

echo "=== -std=ibm (binary-truncate: no) ==="
cobc -std=ibm -x -o step486_ibm step486.cob
./step486_ibm
```

### ผลลัพธ์จริงที่ได้

```
=== -std=default (binary-truncate: yes) ===
AFTER ADD 1 TO 9999 COMP: 0000

=== -std=ibm (binary-truncate: no) ===
AFTER ADD 1 TO 9999 COMP: 10000
```

### อธิบายผลลัพธ์อย่างละเอียด — ความแตกต่างที่ทุกโปรแกรมเมอร์ COBOL ต้องรู้

**ซอร์สโค้ดเดียวกันทุกตัวอักษร ให้ผลลัพธ์ต่างกันโดยสิ้นเชิงเมื่อเปลี่ยนแค่ dialect flag**:

- **`-std=default` (`binary-truncate: yes`)**: GnuCOBOL "ตัดพื้นที่เก็บส่วนเกินทิ้ง" ให้เหลือพอดีกับ
  จำนวนหลักที่ `PICTURE 9(4)` ระบุไว้เท่านั้น (สูงสุด 4 หลัก คือ 0-9999) เมื่อ `9999 + 1 = 10000`
  ซึ่งมี 5 หลัก เกินขอบเขตของ PICTURE ผลลัพธ์จึงถูก "ตัดหลักสูงทิ้ง" เหลือ `0000` (ทำนองเดียวกับกฎ
  MOVE ที่ตัดหลักสูงทิ้งเมื่อฟิลด์ปลายทางเล็กกว่า ตามที่เรียนใน Part 008) — นี่คือพฤติกรรมที่ตรงตาม
  ตัวอักษรของ `PICTURE 9(4)` อย่างเคร่งครัด
- **`-std=ibm` (`binary-truncate: no`)**: IBM Enterprise COBOL (และ Mainframe ส่วนใหญ่) ใช้พื้นที่
  จัดเก็บแบบ Binary เต็มความจุจริงของขนาดที่จัดสรร (2 ไบต์ = เก็บได้ถึง 65535) โดยไม่สนใจว่า
  `PICTURE` ระบุไว้กี่หลัก ตราบใดที่ค่ายังไม่เกินความจุจริงของพื้นที่จัดเก็บ ผลลัพธ์ `10000` (แม้จะ
  มี 5 หลัก เกิน `PIC 9(4)` ที่ระบุไว้) จึงยังถูกเก็บและแสดงผลได้ถูกต้องสมบูรณ์ — พฤติกรรมนี้ตรงกับ
  สิ่งที่เรียกว่า **`COMP-5`** ใน dialect อื่น ๆ ที่ใช้ความจุเต็มของพื้นที่จัดเก็บเสมอ

### ทำไมความแตกต่างนี้จึงสำคัญมากในโลกจริง

โปรแกรมที่เขียนและทดสอบบน Mainframe จริง (IBM Enterprise COBOL, `binary-truncate: no` โดยปริยาย)
แล้วนำมาคอมไพล์ใหม่ด้วย GnuCOBOL โหมด default โดยไม่ระบุ `-std=ibm` **อาจได้ผลลัพธ์การคำนวณที่ผิด
อย่างเงียบ ๆ** หากมีค่าที่เกินขอบเขต PICTURE เกิดขึ้นจริงระหว่างการทำงาน (เช่น ตัวนับที่โตเกินคาด)
นี่คือเหตุผลสำคัญที่การ Migrate/ทดสอบโค้ด Legacy จาก Mainframe ต้องระบุ `-std=ibm` (หรือ dialect
ที่ตรงกับ compiler ต้นทางจริง ๆ) เสมอ ไม่ใช่ปล่อยให้ใช้ค่า default ของ GnuCOBOL

### ข้อควรระวัง

- อย่าคิดว่า `binary-truncate` เป็นเรื่อง "ทฤษฎีเล็กน้อยไม่สำคัญ" — ตัวอย่างข้างต้นพิสูจน์ว่ามันทำให้
  **ผลการคำนวณผิดไปคนละค่าโดยสิ้นเชิง** (0 vs 10000) จากโค้ดชุดเดียวกันเป๊ะ
- เมื่อ migrate โค้ด COBOL จาก Mainframe มาทดสอบบน GnuCOBOL ควรตรวจสอบและระบุ `-std=` ให้ตรงกับ
  compiler ต้นทางเสมอ **ก่อน**เริ่มทดสอบความถูกต้องของ logic ทางธุรกิจ มิฉะนั้นอาจเสียเวลา debug
  ปัญหาที่แท้จริงมาจาก dialect ที่ไม่ตรงกัน ไม่ใช่บั๊กในตรรกะโปรแกรม

### แบบฝึกหัดที่ 486.1

**โจทย์**: จงอธิบายว่าทำไมทีมที่ทำโปรเจกต์ Modernization (ย้ายโค้ด COBOL จาก Mainframe มารันบน
GnuCOBOL บน Linux) จึงควรระบุ `-std=ibm` (หรือ dialect ที่ตรงกับต้นทาง) ตั้งแต่การคอมไพล์ครั้งแรก
แทนที่จะปล่อยให้ใช้ `-std=default`

**เฉลย**: เพราะพฤติกรรมของฟิลด์ตัวเลขแบบ Binary (`COMP`) เมื่อค่าคำนวณเกินขอบเขต PICTURE ต่างกัน
โดยสิ้นเชิงระหว่างสอง Dialect ตามที่พิสูจน์ในขั้นตอนนี้ หากทีมใช้ `-std=default` โดยไม่รู้ตัว
โปรแกรมที่ Migrate มาอาจ**ทำงานได้โดยไม่มี error ใด ๆ แต่ให้ผลการคำนวณผิดอย่างเงียบ ๆ** ในสถานการณ์
ที่ค่าตัวแปรโตเกินขอบเขต PICTURE ที่นักพัฒนาเดิมตั้งใจให้พอดีกับความจุ Binary เต็มรูปแบบของ Mainframe
(ซึ่งเป็นพฤติกรรมปกติที่โปรแกรมเมอร์ Mainframe คุ้นเคยและอาจตั้งใจพึ่งพาไว้) บั๊กแบบนี้อันตรายกว่า
compile error มาก เพราะไม่มีสัญญาณเตือนใด ๆ ทำให้ตรวจพบได้ยากและอาจไปโผล่เป็นปัญหาข้อมูลผิดพลาดใน
ระบบ production แทน การระบุ `-std=ibm` ตั้งแต่ต้นช่วยให้พฤติกรรมของ GnuCOBOL ตรงกับที่โปรแกรมเดิม
ถูกออกแบบและทดสอบมาบน Mainframe จริง ลดความเสี่ยงนี้ได้ตั้งแต่ต้นทาง

---

## ขั้นตอนที่ 487: ความแตกต่างที่พิสูจน์ได้จริง #2 — ความยาวชื่อตัวแปร (Word Length)

### แนวคิด

อีกหนึ่งความแตกต่างที่พิสูจน์ได้ง่ายและกระทบการ**คอมไพล์ผ่านหรือไม่ผ่าน**โดยตรง: ความยาวสูงสุดของ
ชื่อตัวแปร/paragraph ที่แต่ละ Dialect อนุญาต

```bash
grep "word-length" /etc/gnucobol/default.conf /etc/gnucobol/ibm-strict.conf
```

**ผลลัพธ์จริง**:

```
/etc/gnucobol/default.conf:word-length:			63
/etc/gnucobol/ibm-strict.conf:word-length:			30
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP487-WORD-LENGTH.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> This variable name is 51 characters long - well within
      *> GnuCOBOL's own limit of 63, but well past the classic
      *> IBM Mainframe limit of 30 characters per word.
       01  WS-THIS-IS-A-VERY-LONG-VARIABLE-NAME-OVER-30-CHARS
                                     PIC X(5) VALUE "HELLO".

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY WS-THIS-IS-A-VERY-LONG-VARIABLE-NAME-OVER-30-CHARS.
           STOP RUN.
```

```bash
echo "=== -std=default (word-length: 63) ==="
cobc -std=default -x -o step487_default step487.cob

echo "=== -std=ibm-strict (word-length: 30) ==="
cobc -std=ibm-strict -x -o step487_ibm step487.cob
```

### ผลลัพธ์จริงที่ได้

```
=== -std=default (word-length: 63) ===
(compiled successfully, no errors)

=== -std=ibm-strict (word-length: 30) ===
step487.cob:5: error: word length exceeds 30 characters: 'WS-THIS-IS-A-VERY-LONG-VARIABLE-NAME-OVER-30-CHARS'
step487.cob: in paragraph 'MAIN-PARA':
step487.cob:9: error: word length exceeds 30 characters: 'WS-THIS-IS-A-VERY-LONG-VARIABLE-NAME-OVER-30-CHARS'
```

### อธิบายผลลัพธ์

- ซอร์สโค้ดเดียวกันเป๊ะ **คอมไพล์ผ่านสมบูรณ์ด้วย `-std=default`** แต่ **คอมไพล์ไม่ผ่านเลยด้วย
  `-std=ibm-strict`** เพราะชื่อตัวแปรยาวเกิน 30 ตัวอักษร (ชื่อในตัวอย่างมี 51 ตัวอักษร)
- ข้อจำกัด 30 ตัวอักษรนี้สืบทอดมาจากข้อจำกัดฮาร์ดแวร์/ซอฟต์แวร์ยุคแรกของ IBM Mainframe ที่ยังคงอยู่
  ในมาตรฐาน IBM COBOL จนถึงปัจจุบันเพื่อความเข้ากันได้ย้อนหลัง (backward compatibility) กับโปรแกรม
  ที่เขียนมาตั้งแต่หลายสิบปีก่อน
- นี่คือเหตุผลเชิงปฏิบัติที่โปรแกรมเมอร์ COBOL สาย Mainframe จำนวนมากยังคงเขียนชื่อตัวแปรสั้น ๆ แบบ
  `WS-CUST-NM` แทน `WS-CUSTOMER-NAME-FULL-INCLUDING-TITLE` แม้จะทำงานกับ compiler สมัยใหม่ที่รองรับ
  ชื่อยาวกว่าได้ก็ตาม — เป็นธรรมเนียมที่สืบทอดมาจากข้อจำกัดทางเทคนิคในอดีต

### ข้อควรระวัง

- หากเขียนโปรแกรมที่ต้องนำไป deploy บน Mainframe จริงในอนาคต (แม้จะพัฒนา/ทดสอบเบื้องต้นด้วย
  GnuCOBOL) ควรตั้งชื่อตัวแปรให้ไม่เกิน 30 ตัวอักษรตั้งแต่แรก และทดสอบคอมไพล์ด้วย `-std=ibm-strict`
  เป็นระยะเพื่อจับปัญหานี้ตั้งแต่เนิ่น ๆ แทนที่จะไปเจอตอน deploy จริงบน Mainframe
- ข้อจำกัดนี้นับ**เฉพาะความยาวของคำแต่ละคำ** (แต่ละชื่อตัวแปร/paragraph) ไม่ใช่ความยาวรวมทั้งบรรทัด
  หรือทั้งโปรแกรม

### แบบฝึกหัดที่ 487.1

**โจทย์**: จงอธิบายว่าทำไมข้อจำกัดความยาวชื่อ 30 ตัวอักษรของ IBM COBOL จึงยังคงมีอยู่ในมาตรฐานปัจจุบัน
ทั้งที่ฮาร์ดแวร์สมัยใหม่ไม่มีข้อจำกัดด้านหน่วยความจำแบบยุคแรกแล้ว

**เฉลยแนวทาง**: เหตุผลหลักคือ**ความเข้ากันได้ย้อนหลัง (Backward Compatibility)** — องค์กรที่มีโค้ด
COBOL หลายล้านบรรทัดสะสมมาตั้งแต่ยุค 1970-1980 (ตามที่ Part 001 อธิบายไว้) ไม่สามารถเปลี่ยนกฎภาษา
กลางคันได้โดยไม่กระทบโค้ดเดิมจำนวนมหาศาล การเปลี่ยนกฎความยาวชื่อ (แม้จะเป็นการ "ผ่อนปรน" ให้ยาวขึ้น)
อาจสร้างความเสี่ยงที่ไม่คาดคิด เช่น ชื่อตัวแปรที่ยาวขึ้นอาจไปชนกับคำสงวนใหม่ หรือโปรแกรมที่ generate
โค้ดอัตโนมัติ (เช่นจาก tool รุ่นเก่า) อาจยังผูกติดกับข้อจำกัดเดิมอยู่ อุตสาหกรรม Mainframe ให้
ความสำคัญกับเสถียรภาพและความสามารถในการคาดเดาพฤติกรรมได้ (predictability) เหนือความสะดวกสบายในการ
ตั้งชื่อตัวแปรยาว ๆ จึงเลือกคงกฎเดิมไว้เพื่อไม่ทำลายความเข้ากันได้กับระบบที่มีอยู่แล้วนับล้านโปรแกรม

---

## ขั้นตอนที่ 488: การจัดกลุ่มคำสงวนพิเศษ (Reserved Words) ตาม Dialect

### แนวคิด

แต่ละ Dialect ยังมีชุด**คำสงวนพิเศษ (Reserved Words) ของตัวเอง** ที่ dialect อื่นอาจไม่รู้จัก หรือ
กลับกัน — คำบางคำที่เป็นชื่อตัวแปรได้ปกติใน dialect หนึ่ง อาจกลายเป็นคำสงวนต้องห้ามใน dialect อื่น
ตรวจสอบรายการคำสงวนทั้งหมดที่ GnuCOBOL รู้จักพร้อมสถานะการรองรับได้ด้วย:

```bash
cobc --list-reserved
```

### ตัวอย่างการตรวจสอบผลกระทบจาก Reserved Words เพิ่มเติมของ Dialect

`ibm.conf` มีบรรทัด `include: "ibm.words"` ซึ่งเพิ่มคำสงวนพิเศษเฉพาะของ IBM COBOL เข้ามานอกเหนือจาก
ชุดคำสงวนมาตรฐาน — ความหมายเชิงปฏิบัติคือ **ชื่อตัวแปรที่ใช้ได้ปกติใน `-std=default` อาจกลายเป็นชื่อ
ต้องห้าม (เพราะชนกับคำสงวนของ IBM) เมื่อคอมไพล์ด้วย `-std=ibm`**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP488-RESERVED-WORDS.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> ORDER is not a COBOL reserved word in most dialects, so it
      *> is perfectly legal as a variable name by default.
       01  WS-ORDER-COUNT           PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           ADD 1 TO WS-ORDER-COUNT.
           DISPLAY "ORDER COUNT = " WS-ORDER-COUNT.
           STOP RUN.
```

```bash
cobc -std=default -x -o step488 step488.cob
./step488
```

### ผลลัพธ์จริงที่ได้

```
ORDER COUNT = 001
```

### อธิบายจุดสำคัญ

- ตัวอย่างนี้คอมไพล์ผ่านปกติด้วยทั้ง `-std=default` และ `-std=ibm` เพราะ `WS-ORDER-COUNT` ใช้
  `ORDER` เป็นเพียง**ส่วนหนึ่งของชื่อผสม** (compound word) ไม่ใช่คำเดี่ยว ๆ ที่ตรงกับคำสงวนพอดี —
  ประเด็นสำคัญที่ต้องเข้าใจคือ **ความเสี่ยงเรื่องคำสงวนพิเศษของแต่ละ Dialect มีอยู่จริง** แม้ตัวอย่าง
  ในเอกสารนี้จะไม่สร้าง error ให้เห็นชัดเจน (การหาคำที่ชนกันพอดีต้องอ้างอิงรายการ `ibm.words` ที่มี
  รายละเอียดเฉพาะเจาะจงมาก เกินขอบเขตที่จะสาธิตให้ครบในเอกสารนี้)
- แนวทางปฏิบัติที่ปลอดภัยที่สุดคือ **ตั้งชื่อตัวแปรแบบมี prefix เสมอ** (เช่น `WS-`, `LK-`, `FD-`
  ตามธรรมเนียมที่หลักสูตรนี้ใช้มาตั้งแต่ Part 005) เพราะการตั้งชื่อแบบผสมคำแบบนี้**ลดโอกาสชนกับคำสงวน
  ของ dialect ใด ๆ ได้อย่างมาก** เนื่องจากคำสงวนมักเป็นคำเดี่ยว ๆ (เช่น `ORDER`, `LENGTH`, `COUNT`)
  ไม่ใช่คำผสมที่มี prefix เฉพาะขององค์กร

### ข้อควรระวัง

- เมื่อ migrate โค้ดข้าม Dialect ควรคอมไพล์ทดสอบด้วย `-std=` ของ dialect เป้าหมายเสมอ เพื่อจับปัญหา
  คำสงวนที่ชนกันตั้งแต่ขั้นตอนคอมไพล์ (ซึ่งจะแจ้ง error ชัดเจน) ดีกว่าปล่อยผ่านไปแล้วไปเจอปัญหาที่
  ซับซ้อนกว่าภายหลัง
- `cobc --list-reserved` แสดงคำสงวนของชุดค่าเริ่มต้นเท่านั้น หากต้องการดูคำสงวนเฉพาะของ dialect หนึ่ง
  ต้องอ่านไฟล์ `.words` ที่เกี่ยวข้องโดยตรง (เช่น `ibm.words`) ซึ่งเป็นไฟล์ config เสริมที่แยกจาก
  `.conf` หลัก

### แบบฝึกหัดที่ 488.1

**โจทย์**: จงอธิบายว่าทำไมธรรมเนียมการตั้งชื่อตัวแปรด้วย prefix (เช่น `WS-`, `LK-`) ที่หลักสูตรนี้ใช้
มาตั้งแต่ต้น จึงช่วยลดความเสี่ยงเรื่องคำสงวนที่แตกต่างกันระหว่าง Dialect ได้ด้วย (นอกเหนือจากประโยชน์
เรื่องความสื่อความหมายที่เคยเรียนมาก่อนหน้านี้)

**เฉลยแนวทาง**: คำสงวนของ COBOL ในทุก Dialect มักเป็น**คำเดี่ยวที่มีความหมายทางภาษา** เช่น `ORDER`,
`LENGTH`, `RECORD`, `LINE`, `COUNT` เมื่อนำคำเหล่านี้ไปต่อกับ prefix เฉพาะขององค์กร (เช่น `WS-ORDER`,
`WS-LENGTH`) ผลลัพธ์จะกลายเป็น**คำผสมใหม่ที่ไม่ตรงกับคำสงวนใด ๆ ทั้งสิ้น** เพราะ COBOL ถือว่าคำที่มี
เครื่องหมาย `-` เชื่อมกันคือคำเดียวทั้งหมด (single word) ไม่ใช่หลายคำแยกกัน การตั้งชื่อแบบมี prefix
จึงเป็นเทคนิคป้องกันปัญหาที่ได้ผลดีในทางปฏิบัติ แม้จะไม่ได้ออกแบบมาเพื่อจุดประสงค์นี้โดยตรงตั้งแต่แรก
ก็ตาม (จุดประสงค์ดั้งเดิมคือบอกว่าตัวแปรอยู่ใน section ไหนของ DATA DIVISION ตามที่ Part 005 สอนไว้)

---

## ขั้นตอนที่ 489: ข้อจำกัดที่ต้องรู้ — Directive ที่มีอยู่ในมาตรฐานแต่ยังไม่ Implement

### ความซื่อสัตย์ทางเทคนิคสำคัญกว่าการอวดอ้างครบทุกฟีเจอร์

มาตรฐาน COBOL ระบุ Directive อีกตัวหนึ่งชื่อ `>>TURN` ซึ่งใช้เปิด/ปิดการตรวจสอบข้อผิดพลาดขณะรัน
(Exception Checking) เฉพาะจุดในโปรแกรม (เช่น เปิดการตรวจสอบ Subscript เกินขอบเขตเฉพาะบางส่วนของโค้ด)
ขั้นตอนนี้จะทดสอบมันอย่างตรงไปตรงมา เพื่อแสดงให้เห็นว่า **ไม่ใช่ทุก syntax ที่มาตรฐานอนุญาตจะทำงาน
ได้จริงในทุก compiler เวอร์ชัน** — เป็นบทเรียนสำคัญที่ต้องรู้ก่อนพึ่งพาฟีเจอร์ใดในโปรแกรมจริง

### ตัวอย่างโค้ดที่ทดสอบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP489-TURN-DIRECTIVE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TABLE.
           05  WS-ITEM              PIC 9(2) OCCURS 3 TIMES.
       01  WS-IDX                   PIC 9(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> >>TURN is valid COBOL-2002/2014 syntax for turning runtime
      *> exception checking on/off for a specific region of code.
      *> Testing it honestly below - see the real result underneath.
       >>TURN EC-BOUND-SUBSCRIPT CHECKING ON WITH LOCATION
           MOVE 5 TO WS-IDX
           DISPLAY "BEFORE OUT-OF-BOUNDS ACCESS"
           DISPLAY WS-ITEM(WS-IDX)
           DISPLAY "AFTER OUT-OF-BOUNDS ACCESS"
           STOP RUN.
```

```bash
cobc -x -o step489 step489.cob
./step489
```

### ผลลัพธ์จริงที่ได้

```
step489.cob:10: warning: TURN directive is not implemented
BEFORE OUT-OF-BOUNDS ACCESS

AFTER OUT-OF-BOUNDS ACCESS
```

### อธิบายผลลัพธ์อย่างตรงไปตรงมา

- คอมไพเลอร์**ยอมรับ syntax ของ `>>TURN` ได้ถูกต้องตามหลักไวยากรณ์** (ไม่ error ตอน parse) แต่แสดง
  **warning ชัดเจนว่า `"TURN directive is not implemented"`** — หมายความว่า GnuCOBOL 4.0-early-dev.0
  รู้จัก syntax นี้ (เพื่อให้คอมไพล์โค้ดที่มี `>>TURN` จาก compiler อื่นได้โดยไม่ error) แต่**ยังไม่ได้
  ทำให้มันมีผลจริงตอนรัน**
- สังเกตผลลัพธ์: `WS-ITEM(WS-IDX)` ที่ `WS-IDX = 5` (เกินขอบเขตที่ประกาศไว้แค่ `OCCURS 3 TIMES`)
  **ไม่ทำให้โปรแกรม error หรือหยุดทำงานเลย** แม้จะมี `>>TURN EC-BOUND-SUBSCRIPT CHECKING ON` กำกับไว้
  ก็ตาม — พิสูจน์ว่า directive นี้ไม่มีผลอะไรจริงในเวอร์ชันที่ทดสอบ บรรทัด `DISPLAY WS-ITEM(WS-IDX)`
  แสดงเป็นค่าว่าง (undefined behavior จากการเข้าถึงหน่วยความจำนอกขอบเขตของตาราง) แล้วโปรแกรมทำงานต่อ
  ไปจนจบตามปกติ

### ทำไมข้อมูลนี้จึงสำคัญ

หากทีมพัฒนาพึ่งพา `>>TURN EC-BOUND-SUBSCRIPT CHECKING ON` เพื่อป้องกัน Subscript เกินขอบเขตในโปรแกรม
จริงโดยเชื่อว่ามันทำงานตามมาตรฐาน แต่ compiler ที่ใช้จริงยังไม่ implement ฟีเจอร์นี้ (เหมือนที่พิสูจน์
ในขั้นตอนนี้) โปรแกรมจะ**ไม่มีการป้องกัน Subscript เกินขอบเขตเลยจริง ๆ** ทั้งที่โค้ดดู "ปลอดภัย" อยู่
แล้วตามที่เขียนไว้ — นี่คือเหตุผลที่หลักสูตรนี้ย้ำเสมอว่า **ต้องทดสอบยืนยันพฤติกรรมจริงของ compiler
เป้าหมายก่อนพึ่งพาฟีเจอร์ใด ๆ ในโปรแกรมสำคัญ ไม่ใช่เชื่อจากเอกสารมาตรฐานเพียงอย่างเดียว**

### ข้อควรระวัง

- ผลการทดสอบนี้เป็นของ **GnuCOBOL 4.0-early-dev.0** เวอร์ชันเดียวที่ใช้ตลอดหลักสูตรนี้เท่านั้น —
  GnuCOBOL เวอร์ชันใหม่กว่าในอนาคต หรือ compiler ค่ายอื่น (เช่น IBM Enterprise COBOL, Micro Focus)
  อาจ implement `>>TURN` ได้สมบูรณ์แล้วก็เป็นได้ ควรตรวจสอบเวอร์ชันและทดสอบซ้ำเสมอก่อนใช้งานจริง
- การป้องกัน Subscript เกินขอบเขตที่เชื่อถือได้ในสภาพแวดล้อมปัจจุบันของหลักสูตรนี้ยังคงต้องใช้เทคนิค
  การตรวจสอบด้วยมือ (`IF WS-IDX > 3 ...`) ตามที่เรียนมาใน Part 016-018 ไม่ใช่พึ่งพา `>>TURN`

### แบบฝึกหัดที่ 489.1

**โจทย์**: จงอธิบายว่าทำไมนักพัฒนาที่เจอ warning แบบ `"TURN directive is not implemented"` ควร
**หยุดและตรวจสอบเพิ่มเติม** แทนที่จะเพิกเฉยต่อ warning นั้นแล้วเชื่อว่าโปรแกรมทำงานถูกต้องตามที่ตั้งใจ

**เฉลยแนวทาง**: เพราะ warning นี้บอกตรง ๆ ว่า**โค้ดที่เขียนไว้จะไม่มีผลใด ๆ ต่อพฤติกรรมการรันจริง**
แม้ syntax จะถูกต้องและคอมไพล์ผ่านก็ตาม — นี่ต่างจาก error ที่หยุดการคอมไพล์ทันทีและบังคับให้แก้ไข
แต่ warning แบบนี้ปล่อยให้โปรแกรมคอมไพล์และรันต่อไปได้ตามปกติ ทำให้นักพัฒนาที่ไม่สังเกต warning
อย่างละเอียดอาจเข้าใจผิดว่าฟีเจอร์ที่เขียนไว้ (เช่น การตรวจสอบ Subscript อัตโนมัติ) ทำงานอยู่จริง
ทั้งที่ในความเป็นจริงไม่มีการป้องกันใด ๆ เกิดขึ้นเลย หากโปรแกรมนี้ถูกนำไปใช้ในระบบที่ข้อมูลสำคัญ
(เช่น ระบบการเงิน) การเข้าถึง Subscript เกินขอบเขตโดยไม่มีการป้องกันจริงอาจนำไปสู่การอ่าน/เขียน
หน่วยความจำที่ไม่ถูกต้อง ซึ่งเป็นความเสี่ยงร้ายแรงที่ตรวจพบได้ยากในภายหลัง หลักการสำคัญคือ **อ่าน
และทำความเข้าใจ warning ทุกตัวที่คอมไพเลอร์แจ้ง ไม่ใช่สนใจแค่ error เท่านั้น**

---

## ขั้นตอนที่ 490: สรุปแนวทางปฏิบัติที่ดีสำหรับการทำงานกับ Compiler Directives และ Dialects

### Checklist แนวทางปฏิบัติที่ดี (Best Practices)

จากทุกสิ่งที่เรียนรู้ใน Part นี้ รวบรวมเป็นแนวทางปฏิบัติที่นำไปใช้ได้จริงในโปรเจกต์:

1. **ระบุ `-std=` เสมอในโปรเจกต์จริง อย่าพึ่งพาค่า default โดยไม่ตั้งใจ** — โดยเฉพาะเมื่อทำงานกับ
   โค้ดที่มีต้นกำเนิดจาก Mainframe หรือ compiler เฉพาะทางอื่น
2. **ทดสอบยืนยันพฤติกรรมจริงก่อนพึ่งพาฟีเจอร์ใด ๆ** — ดังที่พิสูจน์ในขั้นตอนที่ 489 ว่า syntax ที่
   ถูกต้องตามมาตรฐานอาจยังไม่ถูก implement จริงในบาง compiler
3. **ใช้ Conditional Compilation (`>>IF`/`>>ELSE`) สำหรับโค้ดที่ต้องต่างกันตามสภาพแวดล้อม** แทนการ
   ดูแลไฟล์ source หลายชุดที่แทบจะเหมือนกันทุกประการ (ลดความเสี่ยงเรื่องไฟล์ไม่ sync กัน)
4. **ใช้ `-D` ร่วมกับ Build Script/CI-CD Pipeline** แทนการแก้ไขค่าคงที่ในซอร์สโค้ดด้วยมือก่อน deploy
   ทุกครั้ง
5. **ตั้งชื่อตัวแปรแบบมี prefix เสมอ** เพื่อลดความเสี่ยงชนกับคำสงวนพิเศษที่แตกต่างกันระหว่าง Dialect
6. **บันทึกไว้ในเอกสารโปรเจกต์ว่าใช้ Dialect ใด** เพื่อให้ทีมทุกคนคอมไพล์ด้วยการตั้งค่าเดียวกันเสมอ
   ป้องกันปัญหา "งานที่บ้านฉันรันได้" (works on my machine) ที่เกิดจาก dialect ไม่ตรงกัน

### ตัวอย่างโค้ด: สรุปรวมทุกเทคนิคในโปรแกรมเดียว

```cobol
      *> This header demonstrates the recommended combination:
      *> explicit format directive, conditional compilation guarded
      *> by a CLI-controllable flag, and prefix-based naming.
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP490-BEST-PRACTICES-SUMMARY.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ENVIRONMENT-MSG       PIC X(45).
       01  WS-DIALECT-MSG           PIC X(30)
           VALUE "NO DIALECT-SPECIFIC NOTE HERE".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Conditional compilation driven by an external -D flag -
      *> the recommended pattern from step 484, used for anything
      *> that legitimately differs between environments.
       >>IF DEPLOY-ENV DEFINED
           MOVE "RUNNING WITH AN EXPLICIT DEPLOY-ENV SETTING"
               TO WS-ENVIRONMENT-MSG
       >>ELSE
           MOVE "NO DEPLOY-ENV SET - USING DEFAULT BEHAVIOR"
               TO WS-ENVIRONMENT-MSG
       >>END-IF
           DISPLAY WS-ENVIRONMENT-MSG.
           DISPLAY WS-DIALECT-MSG.
           DISPLAY "REMINDER: always compile this program with an ".
           DISPLAY "explicit -std= flag matching your target system.".
           STOP RUN.
```

```bash
cobc -x -DDEPLOY-ENV=PRODUCTION -std=ibm -o step490 step490.cob
./step490
```

### ผลลัพธ์จริงที่ได้

```
RUNNING WITH AN EXPLICIT DEPLOY-ENV SETTING
NO DIALECT-SPECIFIC NOTE HERE
REMINDER: always compile this program with an
explicit -std= flag matching your target system.
```

### อธิบายโค้ดทีละส่วน

โปรแกรมนี้ผสมผสานเทคนิคหลักที่เรียนมาทั้ง Part: การใช้ `-D` ส่งค่าจากภายนอก, `>>IF`/`>>ELSE`
ตรวจสอบเงื่อนไขนั้น, การระบุ `-std=` อย่างชัดเจนตอนคอมไพล์ (`-std=ibm` ในตัวอย่างนี้ ซึ่งไม่ส่งผลต่อ
ผลลัพธ์ที่เห็นเพราะโปรแกรมนี้ไม่มีฟิลด์ `COMP` ที่ overflow แต่เป็นการฝึกให้เป็นนิสัยตั้งแต่ตอนนี้)
และการตั้งชื่อตัวแปรแบบมี prefix `WS-` ตลอดทั้งโปรแกรม

### ข้อควรระวังส่งท้าย

- Directive และ Dialect flag เป็นเครื่องมือที่ทรงพลังแต่ **ไม่ควรใช้พร่ำเพรื่อจนโค้ดอ่านยาก** — การมี
  `>>IF`/`>>ELSE` ซ้อนกันหลายชั้นในไฟล์เดียวทำให้ติดตามตรรกะได้ยากขึ้นมาก ควรใช้เท่าที่จำเป็นจริง ๆ
  และมีเอกสารกำกับเสมอว่าค่า `-D` แต่ละตัวมีความหมายอะไรบ้าง
- ทวนกฎเหล็กของหลักสูตรอีกครั้ง: แม้ Directive และ Dialect flag จะเป็นเรื่องของคอมไพเลอร์ แต่**ค่า
  string literal และคอมเมนต์ในโค้ด `>>DEFINE`/`>>IF` ก็ยังต้องเป็นภาษาอังกฤษ/ASCII เท่านั้น** เพราะ
  มันยังคงอยู่ในไฟล์ Fixed-Format column 72 เดียวกันกับโค้ด COBOL ปกติ

### แบบฝึกหัดที่ 490.1

**โจทย์**: จงออกแบบระบบตั้งชื่อ `-D` flag ของทีมสมมติหนึ่งทีม ที่ต้อง build โปรแกรม COBOL เดียวกัน
สำหรับ 3 สภาพแวดล้อม (DEV, UAT, PROD) โดยแต่ละสภาพแวดล้อมต้องการพฤติกรรม logging ที่ต่างกัน (DEV
แสดง log ละเอียดทุกขั้นตอน, UAT แสดง log ปานกลาง, PROD แสดงเฉพาะ error) จงร่างโครงสร้าง `>>IF`
คร่าว ๆ ที่จะใช้

**เฉลยแนวทาง**: สามารถออกแบบด้วยค่าคงที่ตัวเดียวที่ส่งผ่าน `-DLOG-LEVEL=DEV` (หรือ `UAT`/`PROD`)
แล้วใช้ `>>IF LOG-LEVEL EQUAL "DEV"` ตรวจสอบเป็นลำดับ:

```cobol
       >>IF LOG-LEVEL EQUAL "DEV"
           PERFORM LOG-DETAILED-TRACE
       >>ELSE
           >>IF LOG-LEVEL EQUAL "UAT"
               PERFORM LOG-MODERATE-TRACE
           >>ELSE
               PERFORM LOG-ERRORS-ONLY
           >>END-IF
       >>END-IF
```

ทีมจะกำหนด build script (หรือ CI/CD pipeline ตามที่จะเรียนใน Part 078) ให้ส่ง `-DLOG-LEVEL=DEV`
เมื่อ build สำหรับเครื่อง developer, `-DLOG-LEVEL=UAT` เมื่อ build ขึ้นระบบทดสอบ และ
`-DLOG-LEVEL=PROD` เมื่อ build ขึ้นระบบจริง โดยไม่ต้องแก้ไซร์สโค้ดเลยแม้แต่ตัวเดียวระหว่างสภาพแวดล้อม
ทั้งสาม — นี่คือการนำหลักการทั้งหมดจาก Part นี้มาประยุกต์ใช้กับสถานการณ์จริงในองค์กร

---

## สรุปท้ายบท

Part นี้พาคุณสำรวจเครื่องมือควบคุมการคอมไพล์ขั้นสูงของ COBOL ครอบคลุม:

- ความแตกต่างเชิงแนวคิดระหว่าง Compiler Directive (มีผลตอนคอมไพล์) กับ COBOL Statement ทั่วไป
  (มีผลตอนรัน)
- `>>SOURCE FORMAT FREE`, `-free` flag, และ `$SET SOURCEFORMAT(...)` สำหรับสลับ Fixed/Free Format
- `>>DEFINE`, `>>IF`/`>>ELSE`/`>>END-IF` สำหรับ Conditional Compilation และการทดสอบด้วย `DEFINED`
- การส่งค่าคงที่จาก command line ด้วย `-D` เพื่อควบคุมการ build โดยไม่แก้ซอร์สโค้ด
- รายชื่อ Dialect จริงที่ GnuCOBOL รองรับ (`default`, `cobol85`, `cobol2002`, `cobol2014`, `ibm`,
  `mf`, `mvs`, `acu`, `realia`, `rm`, `xopen`, `bs2000` และรุ่น `-strict` ของแต่ละตัว) พร้อมโครงสร้าง
  ไฟล์ config ที่แท้จริงซึ่งใช้ `include` เพื่อหลีกเลี่ยงความซ้ำซ้อน
- ความแตกต่างที่**พิสูจน์ด้วยการคอมไพล์และรันจริง**สองกรณี: `binary-truncate` (ผลการคำนวณ COMP
  field ต่างกันจริง — 0 vs 10000 จากโค้ดชุดเดียวกัน) และ `word-length` (คอมไพล์ผ่าน vs ไม่ผ่านจาก
  ความยาวชื่อตัวแปร)
- ความเสี่ยงเรื่องคำสงวนพิเศษที่แตกต่างกันระหว่าง Dialect และเหตุผลที่ธรรมเนียมตั้งชื่อแบบมี prefix
  ช่วยลดความเสี่ยงนี้ได้
- บทเรียนสำคัญเรื่องความซื่อสัตย์ทางเทคนิค: `>>TURN` เป็น syntax ที่ถูกต้องตามมาตรฐานแต่**ยังไม่ได้
  implement จริง**ใน GnuCOBOL 4.0-early-dev.0 (พิสูจน์ด้วย warning จริงจากคอมไพเลอร์) — บทเรียนว่า
  ต้องทดสอบยืนยันเสมอ ไม่เชื่อเอกสารมาตรฐานอย่างเดียว
- Checklist แนวทางปฏิบัติที่ดีสำหรับการทำงานกับ Directives และ Dialects ในโปรเจกต์จริง

ด้วยความรู้จาก Part 036 ถึง Part 049 ครบถ้วนสมบูรณ์แล้ว (Intrinsic Functions, Report Writer, Screen
Section, Advanced EVALUATE, Class/Sign Condition, Dynamic CALL, Based Storage, JSON/XML, Date/Time
ขั้นสูง, Error Handling ขั้นสูง, และ Compiler Directives) ถึงเวลาแล้วที่จะนำทุกอย่างมารวมกันในโปรเจกต์
รวบยอดเฟส 3: **Part 050 — ระบบบัญชีลูกหนี้ (Accounts Receivable System)** โปรเจกต์ที่ใหญ่และสมบูรณ์
ที่สุดในหลักสูตรจนถึงจุดนี้

**[ไปยัง Part 050: 🎯 โปรเจกต์เฟส 3 — ระบบบัญชีลูกหนี้ →](part-050-phase3-project.md)**

---

**เนื้อหาก่อนหน้า**: [Part 048: Error Handling ขั้นสูง — USE Statement และ DECLARATIVES ←](part-048-use-declaratives.md)
**เนื้อหาถัดไป**: [Part 050: 🎯 โปรเจกต์เฟส 3 — ระบบบัญชีลูกหนี้ →](part-050-phase3-project.md)
