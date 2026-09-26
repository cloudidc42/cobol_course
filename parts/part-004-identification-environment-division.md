# Part 004: IDENTIFICATION DIVISION และ ENVIRONMENT DIVISION แบบละเอียด (ขั้นตอนที่ 31–40)

## คำนำของ Part นี้

ใน Part 003 คุณได้เรียนรู้โครงสร้างภาพรวมของโปรแกรม COBOL ทั้ง 4 Divisions และกฎการเขียนคอลัมน์แบบละเอียด
ครบถ้วนแล้ว Part นี้จะพาคุณเจาะลึก **สอง Divisions แรก** ให้ครบทุกแง่มุม นั่นคือ **IDENTIFICATION DIVISION**
ซึ่งเป็น Division เดียวที่บังคับต้องมีเสมอ และ **ENVIRONMENT DIVISION** ซึ่งเป็น Division ที่เชื่อมโยงโปรแกรม
เข้ากับสภาพแวดล้อมของเครื่องจริงและไฟล์ภายนอก

แม้สอง Divisions นี้จะดูเหมือนเป็นแค่ "ส่วนหัวเอกสาร" ที่ไม่มีตรรกะซับซ้อน แต่ความจริงแล้วมีรายละเอียดที่
สำคัญมากซ่อนอยู่ เช่น การกำหนดให้โปรแกรมเป็น `INITIAL PROGRAM` ที่ส่งผลต่อพฤติกรรมของตัวแปรเมื่อถูกเรียกซ้ำ
หรือ `SPECIAL-NAMES` ที่เปลี่ยนกฎการตีความสัญลักษณ์ทศนิยมทั้งโปรแกรม ซึ่งเป็นความรู้ที่จำเป็นมากเมื่อคุณ
ต้องทำงานกับระบบ Legacy จริงในอนาคต

---

## ขั้นตอนที่ 31: IDENTIFICATION DIVISION และ PROGRAM-ID (paragraph บังคับหนึ่งเดียว)

### ภาพรวม

`IDENTIFICATION DIVISION` เป็น Division แรกสุดของทุกโปรแกรม COBOL และเป็น Division เดียวที่**บังคับ
ต้องมี**เสมอ ไม่มีข้อยกเว้น ภายใน Division นี้มี paragraph ย่อยได้หลายตัว แต่มีเพียง **`PROGRAM-ID`**
เท่านั้นที่บังคับ ส่วนที่เหลือ (AUTHOR, INSTALLATION, DATE-WRITTEN ฯลฯ) เป็นทางเลือกทั้งหมด (จะสอนใน
ขั้นตอนที่ 32)

### ตัวอย่างขั้นต่ำสุด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PAYROLL-CALC.

       PROCEDURE DIVISION.
           DISPLAY "PROGRAM-ID is the only required paragraph".
           DISPLAY "in IDENTIFICATION DIVISION.".
           STOP RUN.
```

**ผลลัพธ์**:

```
PROGRAM-ID is the only required paragraph
in IDENTIFICATION DIVISION.
```

### PROGRAM-ID มีหน้าที่อะไร

`PROGRAM-ID` กำหนด**ชื่อโปรแกรม** ซึ่งมีความสำคัญมากกว่าที่คิด เพราะ:

1. เป็นชื่อที่ระบบปฏิบัติการ/runtime ของ COBOL ใช้อ้างอิงเมื่อมีการ `CALL` โปรแกรมนี้จากโปรแกรมอื่น
   (จะสอนละเอียดใน Part 031)
2. ในระบบ Mainframe จริง ชื่อนี้มักต้องตรงกับชื่อไฟล์ Load Module ที่ใช้รันจริงบนเครื่อง
3. ปรากฏใน error message และ debugging log ต่าง ๆ ช่วยระบุว่าปัญหาเกิดจากโปรแกรมไหน

### กฎการตั้งชื่อ PROGRAM-ID

ชื่อ PROGRAM-ID ปฏิบัติตามกฎการตั้งชื่อ Identifier ทั่วไปที่เรียนไปใน Part 003 ขั้นตอนที่ 29
(ตัวอักษร, ตัวเลข, hyphen เท่านั้น ไม่เกิน 30 ตัวอักษรตามมาตรฐาน) แต่ในทางปฏิบัติ ระบบ Mainframe
มักจำกัดชื่อโปรแกรมไว้ที่ **8 ตัวอักษร** เนื่องจากข้อจำกัดของระบบไฟล์ z/OS แบบดั้งเดิม ดังนั้นแม้
GnuCOBOL จะยอมให้ตั้งชื่อยาวกว่านี้ได้ แต่ธรรมเนียมที่ดีคือพยายามตั้งชื่อโปรแกรมให้กระชับไว้ก่อน

### ข้อควรระวัง

- ห้ามลืม period หลัง `PROGRAM-ID. ชื่อโปรแกรม.` — ทั้งหลังคำว่า `PROGRAM-ID` (ถ้าเขียนแยกบรรทัด)
  และหลังชื่อโปรแกรมเอง
- ชื่อโปรแกรมไม่จำเป็นต้องตรงกับชื่อไฟล์ `.cob` แต่**ควร**ตั้งให้ตรงกันเพื่อความสะดวกในการดูแลรักษาโค้ด

### แบบฝึกหัดที่ 31.1

**โจทย์**: จงเขียนโปรแกรม COBOL ที่มี PROGRAM-ID ชื่อ `MY-FIRST-ID` และแสดงข้อความ "Program ID set."

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MY-FIRST-ID.

       PROCEDURE DIVISION.
           DISPLAY "Program ID set.".
           STOP RUN.
```

---

## ขั้นตอนที่ 32: Paragraph เสริมของ IDENTIFICATION DIVISION (AUTHOR, INSTALLATION, DATE-WRITTEN ฯลฯ)

### รายการ Comment Entries ที่ใช้ได้

นอกจาก `PROGRAM-ID` แล้ว IDENTIFICATION DIVISION ยังรองรับ paragraph เสริมอีก 5 ตัวที่เรียกรวมกันว่า
**Comment Entries** เพราะมีไว้เพื่อบันทึกข้อมูลเอกสารประกอบเท่านั้น **ไม่มีผลต่อการทำงานของโปรแกรมแม้แต่น้อย**:

| Paragraph | ความหมาย |
|---|---|
| `AUTHOR.` | ชื่อผู้เขียนโปรแกรม |
| `INSTALLATION.` | ชื่อหน่วยงาน/สถานที่ที่พัฒนาโปรแกรม |
| `DATE-WRITTEN.` | วันที่เขียนโปรแกรม |
| `DATE-COMPILED.` | วันที่คอมไพล์ (บางคอมไพเลอร์เติมค่านี้ให้อัตโนมัติ) |
| `SECURITY.` | ระดับความลับของโปรแกรม |

### ตัวอย่าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. COMMENT-ENTRIES-DEMO.
       AUTHOR. JANE-DEVELOPER.
       INSTALLATION. HEADQUARTERS-IT-DEPT.
       DATE-WRITTEN. 2026-01-15.
       DATE-COMPILED. 2026-01-15.
       SECURITY. NON-CONFIDENTIAL.

       PROCEDURE DIVISION.
           DISPLAY "AUTHOR, INSTALLATION, DATE-WRITTEN,".
           DISPLAY "DATE-COMPILED, SECURITY are documentation only.".
           DISPLAY "They do not affect how the program runs.".
           STOP RUN.
```

**ผลลัพธ์**:

```
AUTHOR, INSTALLATION, DATE-WRITTEN,
DATE-COMPILED, SECURITY are documentation only.
They do not affect how the program runs.
```

### ทำไมถึงยังต้องรู้จัก แม้จะเป็นแค่ Comment

แม้มาตรฐาน COBOL-2002 เป็นต้นไปจะประกาศให้ Comment Entries เหล่านี้เป็น**ฟีเจอร์ที่เลิกใช้แล้ว**
(Obsolete) อย่างเป็นทางการ แต่ GnuCOBOL และคอมไพเลอร์อื่น ๆ ส่วนใหญ่ยังคง**รองรับ**ไว้เพื่อความเข้ากันได้
กับโค้ด Legacy จำนวนมหาศาลที่เขียนไว้ตั้งแต่ยุค 1970-1990 ซึ่งใส่ paragraph เหล่านี้เต็มไปหมด
เมื่อคุณต้องอ่านหรือแก้ไขโค้ด Legacy จริงในอนาคต คุณจะพบเห็น paragraph เหล่านี้บ่อยมาก การรู้ว่ามันคือ
เอกสารประกอบเฉย ๆ (ไม่ใช่โค้ดที่ทำงานจริง) จะช่วยให้คุณอ่านโปรแกรมได้เร็วขึ้นโดยไม่ต้องกังวลว่าจะพลาด
ตรรกะสำคัญไป

### ข้อควรระวัง

- ห้ามเข้าใจผิดว่า `DATE-COMPILED` จะอัปเดตวันที่ให้อัตโนมัติทุกครั้งที่คอมไพล์ — ใน GnuCOBOL ค่านี้เป็นเพียง
  ข้อความคงที่ที่คุณพิมพ์เอง ไม่ใช่ค่าที่ระบบคำนวณให้ (คอมไพเลอร์บางตัวในอดีตเติมให้อัตโนมัติ แต่ไม่ใช่มาตรฐานสากล)
- อย่าพยายามอ่านค่าจาก paragraph เหล่านี้ในโค้ด (เช่น พยายามดึงค่า `AUTHOR` มาแสดงด้วย `DISPLAY`)
  เพราะสิ่งเหล่านี้ไม่ใช่ตัวแปรที่โปรแกรมมองเห็นได้ เป็นแค่ข้อความในเอกสารเฉย ๆ

### แบบฝึกหัดที่ 32.1

**โจทย์**: จงอธิบายว่าทำไม Comment Entries ถึงไม่ปรากฏใน output ของโปรแกรมเลย ทั้งที่เราพิมพ์ข้อมูล
เช่น ชื่อผู้เขียนไว้ในซอร์สโค้ด

**เฉลย**: เพราะ `AUTHOR`, `INSTALLATION` ฯลฯ ไม่ใช่คำสั่งที่ทำงาน (Executable Statement) แต่เป็นเพียง
ส่วนหนึ่งของ IDENTIFICATION DIVISION ที่มีไว้เพื่อบันทึกข้อมูลเชิงเอกสารสำหรับมนุษย์อ่านในซอร์สโค้ดเท่านั้น
โปรแกรมจะแสดงผลลัพธ์เฉพาะสิ่งที่ถูกเขียนด้วยคำสั่งใน PROCEDURE DIVISION เช่น `DISPLAY` เท่านั้น

---

## ขั้นตอนที่ 33: PROGRAM-ID ... IS INITIAL PROGRAM

### แนวคิด: ทำไม Subprogram ถึง "จำ" ค่าเดิมได้

เมื่อโปรแกรม COBOL หนึ่งเรียกใช้โปรแกรมอื่นด้วยคำสั่ง `CALL` (จะสอนละเอียดใน Part 031) โดยปกติแล้ว
**ค่าตัวแปรใน WORKING-STORAGE ของโปรแกรมที่ถูกเรียก (Subprogram) จะไม่รีเซ็ตกลับเป็นค่าเริ่มต้น**
ในการเรียกครั้งถัดไป มันจะ "จำ" ค่าจากการเรียกครั้งก่อนหน้าไว้ ซึ่งบางครั้งเป็นพฤติกรรมที่เราต้องการ
(เช่น ตัวนับสะสม) แต่บางครั้งก็เป็นบั๊กที่ไม่คาดคิด

การเพิ่ม clause `IS INITIAL PROGRAM` ต่อท้าย `PROGRAM-ID` จะบังคับให้ **WORKING-STORAGE ของ
Subprogram นั้นถูกรีเซ็ตกลับเป็นค่า VALUE เริ่มต้นทุกครั้งที่ถูกเรียก** ราวกับเป็นการเรียกครั้งแรกเสมอ

### ทดลองเปรียบเทียบ: มี IS INITIAL PROGRAM

ต้องคอมไพล์เป็น 2 ไฟล์แยกกัน — โปรแกรมย่อย (module) กับโปรแกรมหลัก (main):

**ไฟล์ที่ 1: `counter-sub.cob` (Subprogram)**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. COUNTER-SUB IS INITIAL PROGRAM.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CALL-COUNT          PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
           ADD 1 TO WS-CALL-COUNT.
           DISPLAY "Inside COUNTER-SUB, call count = " WS-CALL-COUNT.
           EXIT PROGRAM.
```

**ไฟล์ที่ 2: `counter-main.cob` (Main Program)**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INITIAL-PROGRAM-DEMO.

       PROCEDURE DIVISION.
           DISPLAY "Calling COUNTER-SUB three times:".
           CALL "COUNTER-SUB".
           CALL "COUNTER-SUB".
           CALL "COUNTER-SUB".
           DISPLAY "Because COUNTER-SUB IS INITIAL PROGRAM,".
           DISPLAY "its working-storage resets before every call,".
           DISPLAY "so the call count above always shows 001.".
           STOP RUN.
```

**คำสั่งคอมไพล์และรัน** (ต้องคอมไพล์ subprogram เป็น dynamic module ด้วย flag `-m` ก่อน แล้วจึงคอมไพล์
โปรแกรมหลักด้วย `-x` และรันโดยกำหนด `COB_LIBRARY_PATH` ให้ชี้ไปที่โฟลเดอร์ที่มีไฟล์ module อยู่):

```bash
cobc -m -o COUNTER-SUB.so counter-sub.cob
cobc -x -o counter-main counter-main.cob
COB_LIBRARY_PATH=. ./counter-main
```

**ผลลัพธ์จริงจากการรัน**:

```
Calling COUNTER-SUB three times:
Inside COUNTER-SUB, call count = 001
Inside COUNTER-SUB, call count = 001
Inside COUNTER-SUB, call count = 001
Because COUNTER-SUB IS INITIAL PROGRAM,
its working-storage resets before every call,
so the call count above always shows 001.
```

### ทดลองเปรียบเทียบ: ไม่มี IS INITIAL PROGRAM

**ไฟล์ที่ 1: `counter-sub2.cob`** (เหมือนเดิมทุกอย่างแต่ **ไม่มี** `IS INITIAL PROGRAM`)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. COUNTER-SUB2.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CALL-COUNT          PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
           ADD 1 TO WS-CALL-COUNT.
           DISPLAY "Inside COUNTER-SUB2, call count = " WS-CALL-COUNT.
           EXIT PROGRAM.
```

**ไฟล์ที่ 2: `no-initial-main.cob`**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. NO-INITIAL-DEMO.

       PROCEDURE DIVISION.
           DISPLAY "Calling COUNTER-SUB2 three times:".
           CALL "COUNTER-SUB2".
           CALL "COUNTER-SUB2".
           CALL "COUNTER-SUB2".
           DISPLAY "Without IS INITIAL PROGRAM, working-storage".
           DISPLAY "keeps its value between calls: 001, 002, 003.".
           STOP RUN.
```

**ผลลัพธ์จริงจากการรัน**:

```
Calling COUNTER-SUB2 three times:
Inside COUNTER-SUB2, call count = 001
Inside COUNTER-SUB2, call count = 002
Inside COUNTER-SUB2, call count = 003
Without IS INITIAL PROGRAM, working-storage
keeps its value between calls: 001, 002, 003.
```

### เปรียบเทียบผลลัพธ์ทั้งสองแบบ

| | COUNTER-SUB (IS INITIAL PROGRAM) | COUNTER-SUB2 (ปกติ) |
|---|---|---|
| เรียกครั้งที่ 1 | 001 | 001 |
| เรียกครั้งที่ 2 | 001 (รีเซ็ตใหม่) | 002 (สะสมต่อ) |
| เรียกครั้งที่ 3 | 001 (รีเซ็ตใหม่) | 003 (สะสมต่อ) |

การทดลองจริงนี้พิสูจน์ชัดเจนว่า `IS INITIAL PROGRAM` ทำให้ WORKING-STORAGE ของโปรแกรมนั้นกลับไปเป็น
ค่า VALUE เริ่มต้นทุกครั้งที่ถูก `CALL` ในขณะที่โปรแกรมปกติจะสะสมค่าต่อเนื่องข้ามการเรียกแต่ละครั้ง

### ข้อควรระวัง

- นี่เป็นแนวคิดที่จะมีประโยชน์มากที่สุดเมื่อคุณเรียน Subprogram อย่างละเอียดใน Part 031 ตอนนี้ขอให้จำแค่
  หลักการไว้ก่อน
- ในระบบที่มีการเรียก Subprogram ซ้ำจำนวนมาก (เช่นระบบ Transaction Processing บน CICS) การลืมใส่
  `IS INITIAL PROGRAM` เมื่อจำเป็น (หรือใส่โดยไม่จำเป็น) เป็นสาเหตุของบั๊กที่พบได้บ่อยและวินิจฉัยยากมาก
  เพราะค่าตัวแปรที่ "หลงเหลือ" จากการเรียกครั้งก่อนอาจทำให้ผลลัพธ์ผิดเพี้ยนโดยไม่มี error ใด ๆ แสดงออกมา

### แบบฝึกหัดที่ 33.1

**โจทย์**: จงอธิบายสถานการณ์ในโลกจริงสักหนึ่งตัวอย่างที่การใช้ `IS INITIAL PROGRAM` จะเป็นประโยชน์
และอีกหนึ่งตัวอย่างที่การ**ไม่**ใช้จะเป็นประโยชน์กว่า

**เฉลยแนวทาง**: ประโยชน์ของ `IS INITIAL PROGRAM` เช่น Subprogram ที่คำนวณภาษีของลูกค้าแต่ละราย
แบบอิสระต่อกัน (ไม่ต้องการให้ค่าของลูกค้าคนก่อนหลงเหลือมาปนกับลูกค้าคนถัดไป) ส่วนกรณีที่ไม่ต้องการ
เช่น Subprogram ที่ทำหน้าที่นับจำนวนธุรกรรมทั้งหมดที่ประมวลผลไปแล้วตลอดการทำงานของโปรแกรมหลัก
(ต้องการให้ตัวนับสะสมค่าต่อเนื่องข้ามการเรียกแต่ละครั้ง)

---

## ขั้นตอนที่ 34: ภาพรวม ENVIRONMENT DIVISION — สองส่วนหลัก

### โครงสร้างของ ENVIRONMENT DIVISION

`ENVIRONMENT DIVISION` ทำหน้าที่เชื่อมโยงโปรแกรม COBOL เข้ากับ**สภาพแวดล้อมภายนอก**ของเครื่องที่รัน
แบ่งออกเป็น 2 ส่วนหลัก (Section):

1. **CONFIGURATION SECTION** — ระบุข้อมูลเกี่ยวกับเครื่องคอมพิวเตอร์ที่ใช้คอมไพล์/รัน และการตั้งค่าพิเศษ
   ต่าง ๆ ผ่าน `SOURCE-COMPUTER`, `OBJECT-COMPUTER`, และ `SPECIAL-NAMES`
2. **INPUT-OUTPUT SECTION** — ระบุการเชื่อมต่อระหว่างชื่อไฟล์ในโปรแกรมกับไฟล์จริงบนดิสก์ ผ่าน
   `FILE-CONTROL` (จะสอนแบบเต็มรูปแบบใน Part 023-030)

### ตัวอย่างโครงร่าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ENV-SKELETON-DEMO.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
       SOURCE-COMPUTER. LINUX-PC.
       OBJECT-COMPUTER. LINUX-PC.

       INPUT-OUTPUT SECTION.
       FILE-CONTROL.

       PROCEDURE DIVISION.
           DISPLAY "ENVIRONMENT DIVISION has two sections:".
           DISPLAY "CONFIGURATION SECTION and INPUT-OUTPUT SECTION.".
           STOP RUN.
```

**ผลลัพธ์**:

```
ENVIRONMENT DIVISION has two sections:
CONFIGURATION SECTION and INPUT-OUTPUT SECTION.
```

สังเกตว่า `FILE-CONTROL.` ในตัวอย่างนี้ว่างเปล่า (ไม่มีการประกาศ `SELECT` ใด ๆ ข้างใน) ก็ยังคอมไพล์ผ่านได้
เพราะเรายังไม่มีการใช้งานไฟล์ในโปรแกรมนี้

### ทำไมต้องแยกเป็นสอง Section

การแยก CONFIGURATION SECTION (เกี่ยวกับเครื่อง/การตั้งค่าโปรแกรม) ออกจาก INPUT-OUTPUT SECTION
(เกี่ยวกับไฟล์) ทำให้โครงสร้างชัดเจนว่า "ถ้าจะปรับการตั้งค่าทศนิยมหรือชื่อเครื่อง ให้ดูที่ CONFIGURATION"
และ "ถ้าจะหาว่าโปรแกรมเปิดไฟล์อะไรบ้าง ให้ดูที่ INPUT-OUTPUT SECTION" ซึ่งมีประโยชน์มากเมื่อโปรแกรมมี
ขนาดใหญ่และมีไฟล์เกี่ยวข้องหลายไฟล์

### ข้อควรระวัง

- ลำดับภายใน ENVIRONMENT DIVISION ก็ตายตัวเช่นกัน: CONFIGURATION SECTION ต้องมาก่อน
  INPUT-OUTPUT SECTION เสมอ
- ถ้าโปรแกรมไม่ใช้ไฟล์เลย ไม่จำเป็นต้องเขียน `INPUT-OUTPUT SECTION.` หรือ `FILE-CONTROL.` เลยก็ได้
  (จะสอนกรณีละเว้นทั้ง ENVIRONMENT DIVISION ในขั้นตอนที่ 38)

### แบบฝึกหัดที่ 34.1

**โจทย์**: CONFIGURATION SECTION กับ INPUT-OUTPUT SECTION อันไหนต้องเขียนก่อนกัน

**เฉลย**: CONFIGURATION SECTION ต้องเขียนก่อนเสมอ ตามด้วย INPUT-OUTPUT SECTION หากมีทั้งคู่ในโปรแกรม
เดียวกัน

---

## ขั้นตอนที่ 35: SOURCE-COMPUTER และ OBJECT-COMPUTER

### ความหมายในอดีตและปัจจุบัน

ในยุคที่ COBOL ถูกออกแบบ (1959-1970s) เป็นเรื่องปกติมากที่โปรแกรมจะถูก**เขียนและคอมไพล์บนเครื่องหนึ่ง**
แต่นำไปรันจริงบน**เครื่องอีกยี่ห้อหนึ่ง** (เช่น เขียนบน IBM แต่รันบน Honeywell) paragraph
`SOURCE-COMPUTER` และ `OBJECT-COMPUTER` จึงถูกออกแบบมาเพื่อบันทึกข้อมูลนี้ไว้อย่างชัดเจน

### ตัวอย่าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SOURCE-OBJECT-DEMO.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
       SOURCE-COMPUTER. LINUX-PC.
       OBJECT-COMPUTER. LINUX-PC.

       PROCEDURE DIVISION.
           DISPLAY "SOURCE-COMPUTER names the machine that".
           DISPLAY "compiles this program.".
           DISPLAY "OBJECT-COMPUTER names the machine that".
           DISPLAY "will run the compiled program.".
           STOP RUN.
```

**ผลลัพธ์**:

```
SOURCE-COMPUTER names the machine that
compiles this program.
OBJECT-COMPUTER names the machine that
will run the compiled program.
```

### สถานะในปัจจุบัน (2026)

ในทางปฏิบัติสมัยใหม่ paragraph ทั้งสองนี้**แทบไม่มีผลเชิงฟังก์ชันใด ๆ** กับคอมไพเลอร์อย่าง GnuCOBOL
แล้ว (ชื่อเครื่องที่ใส่ เช่น `LINUX-PC` เป็นเพียงข้อความอิสระที่คุณตั้งเองได้ตามใจชอบ ไม่ต้องตรงกับชื่อ
เครื่องจริง) แต่ยังคงมีประโยชน์เชิงเอกสารในระบบ Mainframe บางระบบที่ใช้ clause เสริม เช่น
`WITH DEBUGGING MODE` แนบท้าย `OBJECT-COMPUTER` เพื่อเปิดใช้งานบรรทัด Debug (`D` ที่คอลัมน์ 7
ตามที่กล่าวถึงใน Part 003 ขั้นตอนที่ 23)

### ข้อควรระวัง

- อย่าคาดหวังว่าการเปลี่ยนชื่อใน `SOURCE-COMPUTER`/`OBJECT-COMPUTER` จะทำให้โปรแกรมทำงานต่างไปจากเดิม
  บน GnuCOBOL — สิ่งนี้เป็นข้อมูลเอกสารเป็นหลัก ไม่ใช่การตั้งค่าที่มีผลจริงเหมือน `SPECIAL-NAMES`
  ในขั้นตอนถัดไป

### แบบฝึกหัดที่ 35.1

**โจทย์**: จงอธิบายว่าเพราะเหตุใด paragraph `SOURCE-COMPUTER` และ `OBJECT-COMPUTER` จึงมีความสำคัญ
น้อยลงมากในยุคปัจจุบันเมื่อเทียบกับยุค 1960-1970

**เฉลยแนวทาง**: เพราะในอดีตฮาร์ดแวร์คอมพิวเตอร์แต่ละยี่ห้อมีสถาปัตยกรรมแตกต่างกันมาก (คำสั่งเครื่อง,
การจัดเก็บข้อมูล) การระบุเครื่องต้นทาง/ปลายทางจึงมีความหมายจริงจังเพื่อช่วยให้ทีมงานเข้าใจข้อจำกัดที่อาจ
เกิดขึ้นจากการย้ายโปรแกรมข้ามแพลตฟอร์ม แต่ในปัจจุบันคอมไพเลอร์อย่าง GnuCOBOL คอมไพล์โค้ดผ่าน
ภาษา C ที่ทำงานได้บนแทบทุกแพลตฟอร์มสมัยใหม่โดยไม่ต้องพึ่งพาข้อมูลนี้อีกต่อไป

---

## ขั้นตอนที่ 36: SPECIAL-NAMES — การปรับแต่งพฤติกรรมของภาษา

### SPECIAL-NAMES คืออะไร

`SPECIAL-NAMES` เป็น paragraph ใน CONFIGURATION SECTION ที่ทรงพลังที่สุด เพราะสามารถ**เปลี่ยนกฎ
การตีความบางอย่างของทั้งโปรแกรม** ได้ ตัวอย่างที่ใช้บ่อยที่สุดคือ `DECIMAL-POINT IS COMMA`

### ปัญหา: ประเทศที่ใช้จุดทศนิยมต่างจากสหรัฐฯ

ในสหรัฐฯ และไทย เราคุ้นเคยกับการใช้ `.` (period) เป็นจุดทศนิยม และ `,` (comma) เป็นตัวคั่นหลักพัน
เช่น `1,234.56` แต่หลายประเทศในยุโรป (เช่น เยอรมนี, ฝรั่งเศส) ใช้กลับกัน คือ `,` เป็นจุดทศนิยม และ `.`
เป็นตัวคั่นหลักพัน เช่น `1.234,56` การเขียน `DECIMAL-POINT IS COMMA` ทำให้โปรแกรม COBOL ตีความ
สัญลักษณ์ทั้งสองแบบสลับกันทันทีตลอดทั้งโปรแกรม

### ตัวอย่างทดสอบจริง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DECIMAL-COMMA-DEMO.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
       SPECIAL-NAMES.
           DECIMAL-POINT IS COMMA.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE               PIC 9(5)V99 VALUE 1234,56.
       01  WS-PRICE-EDITED        PIC Z.ZZZ9,99.

       PROCEDURE DIVISION.
           MOVE WS-PRICE TO WS-PRICE-EDITED.
           DISPLAY "Raw value  : " WS-PRICE.
           DISPLAY "Edited     : " WS-PRICE-EDITED.
           DISPLAY "Note: with DECIMAL-POINT IS COMMA, the comma".
           DISPLAY "becomes the decimal separator and the period".
           DISPLAY "becomes the thousands separator in PICTURE.".
           STOP RUN.
```

**ผลลัพธ์จริงจากการรัน**:

```
Raw value  : 01234,56
Edited     :   1234,56
Note: with DECIMAL-POINT IS COMMA, the comma
becomes the decimal separator and the period
becomes the thousands separator in PICTURE.
```

### อธิบายสิ่งที่เกิดขึ้น

- `VALUE 1234,56` ในโค้ดต้นฉบับใช้ `,` เป็นจุดทศนิยม แทนที่จะเป็น `.` ตามปกติ เพราะมี
  `DECIMAL-POINT IS COMMA` กำกับไว้แล้ว
- PICTURE `Z.ZZZ9,99` ก็สลับความหมาย: ตำแหน่งที่ปกติเป็นจุดทศนิยม (`,` ในที่นี้) ยังคงเป็นจุดทศนิยม
  ส่วน `.` ที่ปกติเป็นทศนิยมกลับกลายเป็นตัวคั่นหลักพันแทน
- ค่าที่แสดงออกมาจึงใช้ `,` คั่นทศนิยมตลอดทั้งโปรแกรม สอดคล้องกับธรรมเนียมยุโรปที่กล่าวถึงข้างต้น

### ข้อควรระวัง

- Clause นี้มีผล**ทั้งโปรแกรม** ไม่สามารถใช้แค่บางตัวแปรได้ ดังนั้นต้องตัดสินใจตั้งแต่ต้นว่าโปรแกรมนี้
  จะใช้ธรรมเนียมแบบไหน แล้วต้องเขียน literal ตัวเลขทศนิยมทั้งหมดในโปรแกรมให้สอดคล้องกันเสมอ
- `SPECIAL-NAMES` ยังมี clause อื่นที่มีประโยชน์ เช่น การกำหนด `CURRENCY SIGN` (สัญลักษณ์สกุลเงินอื่น
  แทน `$`) และการผูกชื่อ Switch/Condition กับอุปกรณ์ระบบ ซึ่งจะกล่าวถึงเพิ่มเติมเมื่อเนื้อหาเกี่ยวข้อง
  ปรากฏขึ้นในบทถัดไปของหลักสูตร
- ระวังอย่าสับสน `SPECIAL-NAMES.` (paragraph สำหรับตั้งค่า) กับคำว่า "special names" ทั่วไปในภาษาอังกฤษ
  — นี่คือ keyword เฉพาะของ COBOL ที่ต้องเขียนตามรูปแบบนี้เท่านั้น

### แบบฝึกหัดที่ 36.1

**โจทย์**: หากไม่ระบุ `SPECIAL-NAMES` เลยในโปรแกรม ค่าเริ่มต้นของ `DECIMAL-POINT` คือรูปแบบใด

**เฉลย**: ค่าเริ่มต้นคือ `DECIMAL-POINT IS PERIOD` (ใช้ `.` เป็นจุดทศนิยม และ `,` เป็นตัวคั่นหลักพัน)
ซึ่งเป็นค่าที่ใช้ในทุกตัวอย่างของหลักสูตรนี้ ยกเว้นตัวอย่างในขั้นตอนนี้ที่ตั้งใจสาธิตการเปลี่ยนค่านี้โดยเฉพาะ

---

## ขั้นตอนที่ 37: INPUT-OUTPUT SECTION และ FILE-CONTROL — ตัวอย่างเบื้องต้น

### ภาพรวม (จะสอนแบบเต็มใน Part 023-030)

`INPUT-OUTPUT SECTION` พร้อม paragraph `FILE-CONTROL` คือจุดที่เราประกาศว่า "ชื่อไฟล์ในโปรแกรม"
(logical file name) ผูกกับ "ไฟล์จริงบนดิสก์" (physical file) อย่างไร ผ่านประโยค `SELECT ... ASSIGN TO ...`
เนื้อหาการจัดการไฟล์แบบเต็มรูปแบบจะสอนใน Part 023-030 ของหลักสูตร แต่ในขั้นตอนนี้เราจะดูตัวอย่าง
ที่ใช้งานได้จริงเพื่อให้เห็นภาพว่า ENVIRONMENT DIVISION เชื่อมโยงกับ DATA DIVISION และ PROCEDURE DIVISION
อย่างไรเมื่อมีไฟล์เข้ามาเกี่ยวข้อง

### ตัวอย่างที่ใช้งานได้จริง: เขียนไฟล์แล้วอ่านกลับ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FILE-CONTROL-PREVIEW.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT LOG-FILE ASSIGN TO "demo-log.txt"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  LOG-FILE.
       01  LOG-RECORD             PIC X(40).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG            PIC X VALUE "N".

       PROCEDURE DIVISION.
           OPEN OUTPUT LOG-FILE.
           MOVE "First line written by COBOL." TO LOG-RECORD.
           WRITE LOG-RECORD.
           MOVE "Second line written by COBOL." TO LOG-RECORD.
           WRITE LOG-RECORD.
           CLOSE LOG-FILE.

           DISPLAY "File written. Now reading it back:".

           OPEN INPUT LOG-FILE.
           PERFORM UNTIL WS-EOF-FLAG = "Y"
               READ LOG-FILE
                   AT END MOVE "Y" TO WS-EOF-FLAG
                   NOT AT END DISPLAY LOG-RECORD
               END-READ
           END-PERFORM.
           CLOSE LOG-FILE.

           DISPLAY "SELECT/ASSIGN in FILE-CONTROL connects a".
           DISPLAY "COBOL file name to a real file on disk.".
           STOP RUN.
```

**ผลลัพธ์จริงจากการรัน**:

```
File written. Now reading it back:
First line written by COBOL.
Second line written by COBOL.
SELECT/ASSIGN in FILE-CONTROL connects a
COBOL file name to a real file on disk.
```

(และไฟล์ `demo-log.txt` จะถูกสร้างขึ้นจริงในโฟลเดอร์เดียวกับโปรแกรม มีเนื้อหา 2 บรรทัดตามที่เขียนไว้)

### อธิบายส่วนสำคัญ

- `SELECT LOG-FILE ASSIGN TO "demo-log.txt"` — ผูกชื่อ `LOG-FILE` ที่ใช้ในโปรแกรมเข้ากับไฟล์จริงชื่อ
  `demo-log.txt` บนดิสก์
- `ORGANIZATION IS LINE SEQUENTIAL` — บอกว่าไฟล์นี้เป็นไฟล์ข้อความปกติ อ่าน/เขียนทีละบรรทัด
  (คล้ายไฟล์ .txt ทั่วไป)
- `FD LOG-FILE.` ใน `FILE SECTION` (ส่วนหนึ่งของ DATA DIVISION) — อธิบายโครงสร้างของแต่ละ record
  ในไฟล์นี้
- `OPEN`, `WRITE`, `READ`, `CLOSE` ใน PROCEDURE DIVISION — คำสั่งจัดการไฟล์จริง

### ข้อควรระวัง

- ต้องมี `FD` ใน `FILE SECTION` ที่ตรงกับชื่อใน `SELECT` เสมอ มิฉะนั้นจะเกิด compile error
- หัวข้อนี้เป็นเพียงตัวอย่างเกริ่นนำเท่านั้น รายละเอียดเรื่อง File Status, Record Format, และการจัดการ
  ข้อผิดพลาดของไฟล์แบบเต็มรูปแบบจะอยู่ใน Part 023-030 ของหลักสูตร

### แบบฝึกหัดที่ 37.1

**โจทย์**: ในตัวอย่างข้างต้น หากลบ `FD LOG-FILE.` และ `01 LOG-RECORD PIC X(40).` ออกจาก
FILE SECTION แต่ยังคงเหลือ `SELECT LOG-FILE ...` ไว้ใน FILE-CONTROL คาดว่าจะเกิดอะไรขึ้น

**เฉลยแนวทาง**: จะเกิด compile error เนื่องจากคอมไพเลอร์คาดหวังว่าทุกชื่อไฟล์ที่ประกาศด้วย `SELECT`
ใน FILE-CONTROL ต้องมี `FD` ที่สอดคล้องกันใน FILE SECTION เพื่อกำหนดโครงสร้างของ record ในไฟล์นั้น
การมีแค่ SELECT โดยไม่มี FD ทำให้คอมไพเลอร์ไม่รู้ว่าจะอ่าน/เขียนข้อมูลในไฟล์นี้ด้วยโครงสร้างแบบใด

---

## ขั้นตอนที่ 38: เมื่อไหร่ที่ ENVIRONMENT DIVISION ไม่จำเป็นเลย

### ทดสอบ: ละเว้น ENVIRONMENT DIVISION ทั้งหมด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. NO-ENVIRONMENT-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MESSAGE             PIC X(45)
               VALUE "ENVIRONMENT DIVISION is entirely optional.".

       PROCEDURE DIVISION.
           DISPLAY WS-MESSAGE.
           DISPLAY "This program has no ENVIRONMENT DIVISION".
           DISPLAY "at all, and it still compiles and runs.".
           STOP RUN.
```

**ผลลัพธ์จริงจากการรัน**:

```
ENVIRONMENT DIVISION is entirely optional.
This program has no ENVIRONMENT DIVISION
at all, and it still compiles and runs.
```

โปรแกรมนี้กระโดดจาก `IDENTIFICATION DIVISION` ตรงไปยัง `DATA DIVISION` เลย โดยไม่มี
`ENVIRONMENT DIVISION.` ปรากฏอยู่เลยแม้แต่บรรทัดเดียว และยังคอมไพล์ผ่านรันได้ปกติทุกประการ

### กฎการตัดสินใจ: เมื่อไหร่ควรมี เมื่อไหร่ควรละเว้น

| สถานการณ์ | ต้องมี ENVIRONMENT DIVISION หรือไม่ |
|---|---|
| โปรแกรมไม่เปิด/อ่าน/เขียนไฟล์ใด ๆ เลย | ไม่จำเป็น (ละเว้นได้เลย) |
| โปรแกรมต้องเปิด/อ่าน/เขียนไฟล์ (Sequential, Indexed, Relative) | **จำเป็น** ต้องมี INPUT-OUTPUT SECTION |
| โปรแกรมต้องปรับ DECIMAL-POINT, CURRENCY SIGN หรือใช้ SPECIAL-NAMES อื่น | **จำเป็น** ต้องมี CONFIGURATION SECTION |
| โปรแกรมสาธิต/ฝึกหัดขนาดเล็กที่เน้นตรรกะล้วน ๆ | มักละเว้นได้ (เหมือนตัวอย่างนี้) |

### ทำไมเนื้อหาส่วนใหญ่ของ Part 001-002 และตัวอย่างก่อนหน้าถึงไม่มี ENVIRONMENT DIVISION

หากคุณสังเกตโค้ดตัวอย่างใน Part 001-002 และหลายตัวอย่างก่อนหน้าใน Part นี้เอง จะพบว่าบางตัวอย่าง
ไม่มี ENVIRONMENT DIVISION เลย เพราะเป็นโปรแกรมสาธิตง่าย ๆ ที่เน้นแสดงข้อความหรือคำนวณเท่านั้น
ไม่ได้ใช้ไฟล์หรือปรับแต่งการตั้งค่าพิเศษใด ๆ นี่จึงเป็นเหตุผลว่าทำไมโครงสร้าง "ขั้นต่ำสุด" ที่เรียนใน
Part 003 ขั้นตอนที่ 30 ถึงไม่มี ENVIRONMENT DIVISION ปรากฏอยู่ด้วย

### ข้อควรระวัง

- แม้จะละเว้นได้ แต่โปรแกรมจริงในองค์กรส่วนใหญ่**มักจะมี** ENVIRONMENT DIVISION เสมอ แม้จะว่างเปล่าก็ตาม
  เพื่อให้โครงสร้างโปรแกรมดูสมบูรณ์และเป็นธรรมเนียมมาตรฐานที่ทีมพัฒนาส่วนใหญ่ยึดถือ
- อย่าลืมว่าถ้าจะเพิ่มการใช้ไฟล์เข้ามาทีหลัง จะต้องเพิ่ม ENVIRONMENT DIVISION กลับเข้ามาพร้อม
  INPUT-OUTPUT SECTION ทันที มิฉะนั้นจะคอมไพล์ไม่ผ่าน

### แบบฝึกหัดที่ 38.1

**โจทย์**: จงพิจารณาว่าโปรแกรมคำนวณเกรดนักเรียนที่รับคะแนนจากผู้ใช้ผ่าน `ACCEPT` และแสดงผลด้วย
`DISPLAY` เท่านั้น (ไม่มีการอ่าน/เขียนไฟล์) จำเป็นต้องมี ENVIRONMENT DIVISION หรือไม่

**เฉลย**: ไม่จำเป็น เพราะโปรแกรมนี้ไม่ได้เชื่อมต่อกับไฟล์ภายนอกหรือปรับแต่งค่า SPECIAL-NAMES ใด ๆ
เลย การรับข้อมูลผ่าน `ACCEPT` และแสดงผลผ่าน `DISPLAY` ทำงานได้โดยไม่ต้องพึ่งพา ENVIRONMENT DIVISION
เลยแม้แต่น้อย

---

## ขั้นตอนที่ 39: ตัวอย่างโปรแกรมรวมทุกสิ่งที่เรียนมาใน Part นี้

### โปรแกรมสาธิตแบบครบวงจร

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FULL-COMBINED-DEMO.
       AUTHOR. COBOL-COURSE.
       DATE-WRITTEN. 2026-01-15.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
       SOURCE-COMPUTER. LINUX-PC.
       OBJECT-COMPUTER. LINUX-PC.
       SPECIAL-NAMES.
           DECIMAL-POINT IS COMMA.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-UNIT-PRICE          PIC 9(4)V99 VALUE 250,50.
       01  WS-QUANTITY            PIC 9(3)    VALUE 4.
       01  WS-TOTAL               PIC 9(6)V99 VALUE 0.
       01  WS-TOTAL-EDITED        PIC Z.ZZZ9,99.

       PROCEDURE DIVISION.
       MAIN-PARAGRAPH.
           COMPUTE WS-TOTAL = WS-UNIT-PRICE * WS-QUANTITY.
           MOVE WS-TOTAL TO WS-TOTAL-EDITED.
           DISPLAY "Unit price : " WS-UNIT-PRICE.
           DISPLAY "Quantity   : " WS-QUANTITY.
           DISPLAY "Total      : " WS-TOTAL-EDITED.
           STOP RUN.
```

**ผลลัพธ์จริงจากการรัน**:

```
Unit price : 0250,50
Quantity   : 004
Total      :   1002,00
```

### อธิบายภาพรวม

โปรแกรมนี้รวมทุกสิ่งที่เรียนมาใน Part 004 เข้าไว้ในที่เดียว:

- `AUTHOR.` และ `DATE-WRITTEN.` — Comment Entries จากขั้นตอนที่ 32
- `SOURCE-COMPUTER.` และ `OBJECT-COMPUTER.` — จากขั้นตอนที่ 35
- `SPECIAL-NAMES.` พร้อม `DECIMAL-POINT IS COMMA.` — จากขั้นตอนที่ 36
- การคำนวณราคารวมด้วย `COMPUTE` แล้วแสดงผลด้วย PICTURE ที่ปรับรูปแบบทศนิยมแบบยุโรป

### ข้อควรระวัง

- เมื่อรวมหลาย feature เข้าด้วยกัน ให้ตรวจสอบว่า literal ตัวเลขทุกตัวในโปรแกรม (เช่น `250,50` ใน VALUE)
  สอดคล้องกับการตั้งค่า `DECIMAL-POINT` ที่กำหนดไว้ มิฉะนั้นค่าจะผิดเพี้ยนโดยไม่มี error แจ้งเตือน

### แบบฝึกหัดที่ 39.1

**โจทย์**: จากผลลัพธ์ `Total : 1002,00` จงตรวจสอบด้วยการคำนวณมือว่าค่านี้ถูกต้องหรือไม่
(250.50 × 4 เท่ากับเท่าไหร่)

**เฉลย**: 250.50 × 4 = 1002.00 ซึ่งตรงกับผลลัพธ์ `1002,00` ที่แสดงผล (เพียงแต่ใช้ `,` แทน `.`
เป็นจุดทศนิยมตามการตั้งค่า `DECIMAL-POINT IS COMMA`) แสดงว่าโปรแกรมคำนวณถูกต้อง

---

## ขั้นตอนที่ 40: ข้อผิดพลาดที่พบบ่อยเกี่ยวกับ Division เหล่านี้และวิธีแก้

### กรณีที่ 1: END PROGRAM ไม่ตรงกับ PROGRAM-ID

เมื่อโปรแกรมมีการปิดท้ายด้วย `END PROGRAM ชื่อโปรแกรม.` (มักใช้เมื่อมีหลายโปรแกรมในไฟล์เดียวกัน หรือ
เขียนแบบ Nested Program ซึ่งจะสอนใน Part 034) ชื่อที่ระบุใน `END PROGRAM` ต้อง**ตรงกันทุกตัวอักษร**
กับชื่อใน `PROGRAM-ID`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BAD-END-NAME.

       PROCEDURE DIVISION.
           DISPLAY "This program has a mismatched END PROGRAM.".
           STOP RUN.

       END PROGRAM WRONG-NAME.
```

**ผลลัพธ์การคอมไพล์จริง (error)**:

```
s40-bad-program-id.cob:8: error: END PROGRAM 'WRONG-NAME' is different from PROGRAM-ID 'BAD-END-NAME'
```

**วิธีแก้**: แก้ชื่อใน `END PROGRAM` ให้ตรงกับ `PROGRAM-ID` ทุกตัวอักษร:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GOOD-END-NAME.

       PROCEDURE DIVISION.
           DISPLAY "END PROGRAM name matches PROGRAM-ID.".
           STOP RUN.

       END PROGRAM GOOD-END-NAME.
```

**ผลลัพธ์หลังแก้ไข**:

```
END PROGRAM name matches PROGRAM-ID.
```

### กรณีที่ 2: ลืมลำดับ CONFIGURATION SECTION ก่อน INPUT-OUTPUT SECTION

หากเขียน `INPUT-OUTPUT SECTION.` ไว้ก่อน `CONFIGURATION SECTION.` คอมไพเลอร์จะรายงาน syntax error
ทันที เพราะไวยากรณ์กำหนดลำดับตายตัวเช่นเดียวกับลำดับ 4 Divisions หลัก

### กรณีที่ 3: ใช้ SPECIAL-NAMES ผิดตำแหน่ง

`SPECIAL-NAMES` ต้องอยู่ภายใน `CONFIGURATION SECTION` เท่านั้น หากนำไปเขียนไว้นอก Section นี้
(เช่น เขียนแยกไว้เป็น Section ของตัวเอง) จะเกิด syntax error เนื่องจากไวยากรณ์ไม่รู้จัก `SPECIAL-NAMES`
เป็น Section ระดับบนสุด

### ตารางสรุป Error ที่พบบ่อยใน Part นี้

| Error message | สาเหตุ | วิธีแก้ |
|---|---|---|
| `END PROGRAM 'X' is different from PROGRAM-ID 'Y'` | ชื่อใน END PROGRAM ไม่ตรงกับ PROGRAM-ID | แก้ให้ชื่อตรงกันทุกตัวอักษร |
| `syntax error, unexpected INPUT-OUTPUT` | เขียน INPUT-OUTPUT SECTION ก่อน CONFIGURATION SECTION | สลับลำดับให้ถูกต้อง |
| `'ชื่อไฟล์' is not defined` (เมื่อใช้ FILE-CONTROL) | มี SELECT แต่ไม่มี FD ที่ตรงกัน | เพิ่ม FD ใน FILE SECTION |
| ค่าทศนิยมผิดเพี้ยนโดยไม่มี error | literal ตัวเลขไม่สอดคล้องกับ DECIMAL-POINT ที่ตั้งไว้ | ตรวจสอบให้ทุก literal ใช้สัญลักษณ์ตรงกับที่ตั้งค่าไว้ |

### ข้อควรระวัง

- เมื่อโปรแกรมมีหลายไฟล์ที่ประกอบกัน (multiple source files) เช่นในตัวอย่าง `IS INITIAL PROGRAM`
  ขั้นตอนที่ 33 การลืมคอมไพล์ subprogram ด้วย flag `-m` ก่อน หรือลืมตั้งค่า `COB_LIBRARY_PATH`
  ตอนรัน จะทำให้เกิด runtime error ประเภท "module not found" ซึ่งไม่ใช่ compile error แต่เป็น
  ปัญหาตอนรันโปรแกรม ต้องแยกแยะให้ออกว่าเป็นปัญหาช่วงคอมไพล์หรือช่วงรัน

### แบบฝึกหัดที่ 40.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงตั้งใจทำให้เกิด error ทั้ง 2 แบบต่อไปนี้ด้วยตัวเอง แล้วบันทึกข้อความ error ที่ได้จริง:
(1) ตั้งชื่อ `END PROGRAM` ให้ไม่ตรงกับ `PROGRAM-ID`
(2) สลับลำดับเขียน `INPUT-OUTPUT SECTION` ไว้ก่อน `CONFIGURATION SECTION`

**เฉลยแนวทาง**: กรณีที่ (1) จะได้ error ทำนอง
`END PROGRAM 'X' is different from PROGRAM-ID 'Y'` ตามที่แสดงในตัวอย่างของขั้นตอนนี้
กรณีที่ (2) จะได้ syntax error ที่ชี้ไปยังตำแหน่งของ `INPUT-OUTPUT SECTION` เนื่องจากคอมไพเลอร์คาดหวัง
ให้ `CONFIGURATION SECTION` ปรากฏก่อนตามลำดับไวยากรณ์ที่กำหนดไว้ การฝึกทำ error ทั้งสองแบบนี้ด้วยตัวเอง
จะช่วยให้คุณจดจำข้อความ error เหล่านี้ได้แม่นยำขึ้นเมื่อเจอในสถานการณ์จริง

---

## สรุปท้ายบท

ใน Part นี้ คุณได้เรียนรู้:

- IDENTIFICATION DIVISION และ PROGRAM-ID ซึ่งเป็น paragraph บังคับหนึ่งเดียวของทั้งโปรแกรม
- Comment Entries เสริม (AUTHOR, INSTALLATION, DATE-WRITTEN, DATE-COMPILED, SECURITY) ที่เป็น
  เอกสารประกอบเท่านั้น ไม่มีผลต่อการทำงาน
- `PROGRAM-ID ... IS INITIAL PROGRAM` และผลกระทบจริงต่อการรีเซ็ต WORKING-STORAGE เมื่อถูก CALL ซ้ำ
  พิสูจน์ด้วยการทดลองเปรียบเทียบสองโปรแกรมจริง
- โครงสร้างของ ENVIRONMENT DIVISION ที่แบ่งเป็น CONFIGURATION SECTION และ INPUT-OUTPUT SECTION
- SOURCE-COMPUTER และ OBJECT-COMPUTER และเหตุผลว่าทำไมความสำคัญลดลงในยุคปัจจุบัน
- SPECIAL-NAMES และการปรับ DECIMAL-POINT IS COMMA ที่มีผลต่อทั้งโปรแกรม
- FILE-CONTROL เบื้องต้นที่เชื่อมโยงชื่อไฟล์ในโปรแกรมกับไฟล์จริงบนดิสก์ (ตัวอย่างที่ใช้งานได้จริง)
- กรณีที่ ENVIRONMENT DIVISION ไม่จำเป็นต้องมีเลย และหลักการตัดสินใจว่าเมื่อไหร่ควรมี
- ตัวอย่างโปรแกรมรวมทุก feature และ error ที่พบบ่อยพร้อมวิธีแก้

ตอนนี้คุณเข้าใจ IDENTIFICATION DIVISION และ ENVIRONMENT DIVISION อย่างละเอียดครบถ้วนแล้ว
ใน **Part 005** เราจะเข้าสู่ **DATA DIVISION** อย่างเต็มรูปแบบ โดยเริ่มจาก WORKING-STORAGE SECTION
ซึ่งเป็นหัวใจสำคัญของการเก็บข้อมูลในโปรแกรม COBOL ทุกโปรแกรม

**[← กลับไป Part 003](part-003-program-structure.md)** | **[ไปยัง Part 005: DATA DIVISION เบื้องต้น →](part-005-working-storage-section.md)**
