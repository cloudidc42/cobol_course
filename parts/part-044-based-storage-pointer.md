# Part 044: Based Storage และ POINTER Data Item (ขั้นตอนที่ 431–440)

## คำนำของ Part นี้

Part 043 แนะนำให้เรารู้จัก `PROGRAM-POINTER` ซึ่งเป็นตัวชี้ (pointer) เฉพาะสำหรับเก็บที่อยู่ของ
**โปรแกรม** Part นี้จะขยายแนวคิดเรื่อง Pointer ให้กว้างขึ้นไปถึงการชี้ไปยัง**ข้อมูล** ด้วยชนิดข้อมูล
`POINTER` ทั่วไป การอ้างอิงตำแหน่งข้อมูลด้วย special register `ADDRESS OF` และเทคนิค `BASED` storage
ที่เปิดทางให้ COBOL จัดสรรหน่วยความจำแบบไดนามิก (Dynamic Memory Allocation) ผ่านคำสั่ง `ALLOCATE`
และ `FREE` — ความสามารถที่เพิ่มเข้ามาตั้งแต่มาตรฐาน COBOL-2002 เพื่อให้ COBOL สร้างโครงสร้างข้อมูล
แบบไดนามิกอย่าง Linked List และ Stack ได้ เหมือนที่ภาษา C ทำได้ด้วย `malloc()`/`free()`

เนื้อหาใน Part นี้เป็นเทคนิคขั้นสูงที่ไม่ได้ใช้บ่อยในงาน COBOL แบบดั้งเดิม (ซึ่งเน้นโครงสร้างข้อมูล
แบบคงที่ผ่าน `OCCURS`) แต่มีความสำคัญมากขึ้นเรื่อย ๆ เมื่อ COBOL ต้องทำงานร่วมกับข้อมูลที่มีขนาดไม่
แน่นอนล่วงหน้า หรือเมื่อต้องเชื่อมต่อกับไลบรารีภาษา C ผ่าน `CALL` (จะเรียนละเอียดใน Part 072)

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และผ่านการคอมไพล์
> และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 431: POINTER Data Item พื้นฐาน

### POINTER คือชนิดข้อมูลที่เก็บ "ที่อยู่" ไม่ใช่ "ค่า"

ตัวแปรทั่วไปใน COBOL (`PIC 9`, `PIC X` ฯลฯ) เก็บ**ค่าข้อมูลโดยตรง** แต่ `USAGE POINTER` เป็นชนิดข้อมูล
พิเศษที่เก็บ **ที่อยู่ในหน่วยความจำ** ของข้อมูลอีกตัวหนึ่ง คล้ายกับแนวคิด pointer ในภาษา C หรือ
reference ในภาษาอื่น ๆ ตัวแปรที่ประกาศเป็น `POINTER` จะไม่มี `PICTURE` clause เพราะไม่ได้เก็บตัวเลข
หรือข้อความตามความหมายทั่วไป

### ค่าพิเศษ NULL

`POINTER` มีค่าพิเศษชื่อ `NULL` ที่หมายถึง "ยังไม่ได้ชี้ไปที่ไหน" เปรียบเทียบได้เหมือนกับ `nullptr`
ใน C++ หรือ `None`/`null` ในภาษาอื่น ๆ

### ตัวอย่าง: ประกาศ POINTER และเปรียบเทียบกับ NULL

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PTR1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AMOUNT       PIC 9(5)V99 VALUE 12345.67.
       01  WS-PTR          USAGE POINTER.
       PROCEDURE DIVISION.
           SET WS-PTR TO ADDRESS OF WS-AMOUNT
           DISPLAY "Address captured (non-null check): "
           IF WS-PTR NOT EQUAL NULL
               DISPLAY "Pointer is not NULL, points to WS-AMOUNT"
           END-IF
           SET WS-PTR TO NULL
           IF WS-PTR EQUAL NULL
               DISPLAY "Pointer reset to NULL successfully"
           END-IF
           STOP RUN.
```

**ผลลัพธ์:**

```
Address captured (non-null check):
Pointer is not NULL, points to WS-AMOUNT
Pointer reset to NULL successfully
```

### อธิบายจุดสำคัญ

- `01 WS-PTR USAGE POINTER.`: ประกาศตัวแปรชนิด `POINTER` — ไม่มี `PICTURE` เพราะเก็บที่อยู่หน่วยความจำ
  ไม่ใช่ข้อมูลทั่วไป
- `WS-PTR NOT EQUAL NULL` / `WS-PTR EQUAL NULL`: การเปรียบเทียบ pointer กับ `NULL` ใช้ operator
  ปกติ (`EQUAL`, `NOT EQUAL`) ได้เหมือนตัวแปรทั่วไป
- `SET WS-PTR TO NULL`: กำหนดให้ pointer ไม่ชี้ไปที่ใดเลย ซึ่งเป็นแนวปฏิบัติที่ดีเมื่อยังไม่มีข้อมูล
  ให้ชี้ (จะเห็นประโยชน์ชัดเจนเมื่อสร้าง Linked List ในขั้นตอนที่ 437)

### ข้อควรระวัง

- **ห้าม dereference pointer ที่เป็น `NULL` หรือยังไม่เคยถูก `SET`** การพยายามเข้าถึงข้อมูลผ่าน
  pointer ที่ไม่ถูกต้องจะทำให้โปรแกรม crash (คล้ายกับ Segmentation Fault ในภาษา C) เสมอตรวจสอบด้วย
  `IF pointer NOT EQUAL NULL` ก่อนใช้งานทุกครั้งที่ไม่แน่ใจ
- Pointer ที่เพิ่งประกาศแต่ยังไม่ได้ `SET` ค่าใด ๆ จะมีค่าไม่แน่นอน (undefined) ไม่ใช่ `NULL` โดย
  อัตโนมัติเสมอไป ควร `SET TO NULL` อย่างชัดเจนตั้งแต่ต้นเพื่อความปลอดภัย

### แบบฝึกหัดที่ 431.1

**โจทย์**: จงอธิบายว่าทำไม `POINTER` จึงไม่มี `PICTURE` clause ทั้งที่ตัวแปร COBOL ทั่วไปเกือบทุกชนิด
ต้องมี

**เฉลย**: `PICTURE` clause ใช้กำหนดรูปแบบและขนาดของ**ข้อมูลที่เก็บโดยตรง** เช่น จำนวนหลักตัวเลขหรือ
ความยาวตัวอักษร แต่ `POINTER` ไม่ได้เก็บข้อมูลแบบนั้น มันเก็บ**ที่อยู่ในหน่วยความจำ**ซึ่งมีขนาดคงที่
ตามสถาปัตยกรรมเครื่อง (เช่น 8 ไบต์บนระบบ 64-bit) และตีความแตกต่างไปจากตัวเลขหรือข้อความทั่วไป
โดยสิ้นเชิง COBOL จึงจัดให้ `POINTER` เป็นหนึ่งใน `USAGE` clause พิเศษที่ไม่ต้องและไม่สามารถระบุ
`PICTURE` ได้

---

## ขั้นตอนที่ 432: ADDRESS OF Special Register

### ADDRESS OF คือ "ตัวดำเนินการขอที่อยู่"

`ADDRESS OF data-item` เป็น **special register** ที่ COBOL เตรียมไว้ให้เสมอสำหรับทุกรายการข้อมูลที่
ประกาศไว้ในระดับ 01 (หรือ 77) มันคืนค่าที่อยู่ในหน่วยความจำของรายการข้อมูลนั้น คล้ายกับ operator `&`
ในภาษา C

### ตัวอย่าง: ใช้ ADDRESS OF กับข้อมูลหลายตัว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ADDR1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST-VALUE  PIC 9(5) VALUE 100.
       01  WS-SECOND-VALUE PIC 9(5) VALUE 200.
       01  WS-PTR-A        USAGE POINTER.
       01  WS-PTR-B        USAGE POINTER.
       PROCEDURE DIVISION.
           SET WS-PTR-A TO ADDRESS OF WS-FIRST-VALUE
           SET WS-PTR-B TO ADDRESS OF WS-SECOND-VALUE
           IF WS-PTR-A NOT EQUAL WS-PTR-B
               DISPLAY "Two different items have two"
                   " different addresses"
           END-IF
           IF WS-PTR-A EQUAL ADDRESS OF WS-FIRST-VALUE
               DISPLAY "Address of WS-FIRST-VALUE is stable"
           END-IF
           STOP RUN.
```

**ผลลัพธ์:**

```
Two different items have two different addresses
Address of WS-FIRST-VALUE is stable
```

### อธิบายจุดสำคัญ

- `ADDRESS OF WS-FIRST-VALUE` ใช้ได้ทั้งฝั่งขวาของ `SET` (เพื่อดึงค่าที่อยู่มาเก็บใน pointer) และ
  ใช้เปรียบเทียบตรง ๆ ได้เลย (`IF WS-PTR-A EQUAL ADDRESS OF WS-FIRST-VALUE`)
- ที่อยู่ของแต่ละรายการข้อมูลจะ**คงที่ตลอดอายุการทำงานของโปรแกรม** (สำหรับข้อมูลใน
  `WORKING-STORAGE SECTION` ปกติที่ไม่ใช่ `BASED`) ทำให้เปรียบเทียบซ้ำได้ผลลัพธ์เดิมเสมอ

### ข้อควรระวัง

- `ADDRESS OF` ใช้ได้กับรายการข้อมูลระดับ 01 หรือ 77 เท่านั้นในหลาย implementation (การใช้กับ field
  ย่อยระดับลึกอาจไม่ได้รับการรองรับเท่ากันในทุกคอมไพเลอร์) ในบิลด์นี้เมื่อทดสอบกับ 01-level ทำงานได้
  ปกติสมบูรณ์
- อย่าสับสนระหว่าง "ที่อยู่ของตัวแปร" กับ "ค่าที่เก็บอยู่ในตัวแปร" — `ADDRESS OF` ให้ที่อยู่เสมอ ไม่ใช่
  ค่าข้อมูล

### แบบฝึกหัดที่ 432.1

**โจทย์**: จงเขียนโปรแกรมที่ประกาศตัวแปร `WS-X` และ `WS-Y` (ทั้งคู่ `PIC 9(3)`) แล้วตรวจสอบว่าที่อยู่
ของทั้งสองตัวไม่เท่ากัน พร้อมแสดงข้อความยืนยัน

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CHKADDR.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-X            PIC 9(3) VALUE 1.
       01  WS-Y            PIC 9(3) VALUE 2.
       01  WS-PX           USAGE POINTER.
       01  WS-PY           USAGE POINTER.
       PROCEDURE DIVISION.
           SET WS-PX TO ADDRESS OF WS-X
           SET WS-PY TO ADDRESS OF WS-Y
           IF WS-PX NOT EQUAL WS-PY
               DISPLAY "WS-X and WS-Y have different addresses"
           END-IF
           STOP RUN.
```

---

## ขั้นตอนที่ 433: ส่งที่อยู่ข้ามโปรแกรม — ทางเลือกแทน BY REFERENCE

### ทำไมต้องส่งที่อยู่แทนการส่งค่าตรง ๆ

Part 032 สอนการส่งพารามิเตอร์ด้วย `BY REFERENCE` ซึ่งโดยปริยายก็คือการส่ง "ที่อยู่" ของตัวแปรไปให้
โปรแกรมย่อยแก้ไขได้โดยตรงอยู่แล้ว อย่างไรก็ตาม การส่ง `POINTER` เป็นพารามิเตอร์อย่างชัดเจนมีประโยชน์
เมื่อโปรแกรมย่อยต้อง**เลือกได้เองว่าจะชี้ไปที่ข้อมูลใด** (ไม่ใช่แค่แก้ไขข้อมูลที่ส่งมาให้ตายตัว) ซึ่ง
เป็นรากฐานของการส่งต่อ "การอ้างอิงถึงข้อมูล" แบบยืดหยุ่นระหว่างโปรแกรม

### ตัวอย่าง: โปรแกรมย่อยรับ POINTER แล้วแก้ไขข้อมูลปลายทางผ่าน SET ADDRESS OF

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUBPTR.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TARGET    PIC 9(5) BASED.
       LINKAGE SECTION.
       01  LS-PTR       USAGE POINTER.
       PROCEDURE DIVISION USING LS-PTR.
           SET ADDRESS OF WS-TARGET TO LS-PTR
           MOVE 999 TO WS-TARGET
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAINPTR.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-VALUE     PIC 9(5) VALUE 111.
       01  WS-PTR       USAGE POINTER.
       PROCEDURE DIVISION.
           DISPLAY "Before: " WS-VALUE
           SET WS-PTR TO ADDRESS OF WS-VALUE
           CALL "SUBPTR" USING WS-PTR
           DISPLAY "After:  " WS-VALUE
           STOP RUN.
```

**ผลลัพธ์:**

```
Before: 00111
After:  00999
```

### อธิบายจุดสำคัญ

- `01 WS-TARGET PIC 9(5) BASED.` ใน `SUBPTR`: ประกาศ `WS-TARGET` แบบ `BASED` (จะอธิบายละเอียดใน
  ขั้นตอนที่ 434) ซึ่งหมายความว่า `WS-TARGET` **ไม่มีหน่วยความจำเป็นของตัวเอง** จนกว่าจะมีการกำหนด
  ที่อยู่ให้ด้วย `SET ADDRESS OF`
- `SET ADDRESS OF WS-TARGET TO LS-PTR`: สั่งให้ `WS-TARGET` "สวมทับ" ตำแหน่งหน่วยความจำเดียวกันกับ
  ที่ `LS-PTR` ชี้ไป (ซึ่งก็คือ `WS-VALUE` ใน `MAINPTR`)
- `MOVE 999 TO WS-TARGET`: เมื่อ `WS-TARGET` ถูกกำหนดที่อยู่แล้ว การแก้ไขค่าของมันก็คือการแก้ไขค่า
  ของ `WS-VALUE` ในโปรแกรมหลักโดยตรง — นี่คือการส่งต่อ "การอ้างอิงถึงข้อมูล" ผ่าน `POINTER` อย่าง
  ชัดเจน ต่างจาก `BY REFERENCE` ตรงที่ผู้เขียนโปรแกรมย่อยเป็นผู้ควบคุมเองว่าจะ bind ที่อยู่นั้นเข้ากับ
  ตัวแปรใดของตนเมื่อไหร่ (ยืดหยุ่นกว่าในกรณีซับซ้อน เช่น การเลือกกลุ่มข้อมูลปลายทางตามเงื่อนไข)

### ข้อควรระวัง

- เทคนิคนี้ทรงพลังแต่ก็เสี่ยงมากขึ้นตามไปด้วย: หากผู้เรียกส่ง `POINTER` ที่ไม่ถูกต้องมา (เช่น
  `NULL` หรือชี้ไปยังหน่วยความจำที่ถูกปล่อยคืนไปแล้ว) การ `SET ADDRESS OF ... TO LS-PTR` แล้วเขียน
  ทับข้อมูลจะทำให้โปรแกรม crash หรือเสียหายของข้อมูลที่ไม่เกี่ยวข้องได้ ควร validate `LS-PTR NOT
  EQUAL NULL` ก่อนใช้งานเสมอในโค้ด production

### แบบฝึกหัดที่ 433.1

**โจทย์**: จงปรับโปรแกรม `SUBPTR` ให้ตรวจสอบก่อนว่า `LS-PTR` ไม่ใช่ `NULL` ก่อนจะ `SET ADDRESS OF`
และแก้ไขข้อมูล หากเป็น `NULL` ให้แสดงข้อความเตือนแทน

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUBPTR2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TARGET    PIC 9(5) BASED.
       LINKAGE SECTION.
       01  LS-PTR       USAGE POINTER.
       PROCEDURE DIVISION USING LS-PTR.
           IF LS-PTR EQUAL NULL
               DISPLAY "SUBPTR2: received a NULL pointer, skip"
           ELSE
               SET ADDRESS OF WS-TARGET TO LS-PTR
               MOVE 999 TO WS-TARGET
           END-IF
           GOBACK.
```

---

## ขั้นตอนที่ 434: BASED Clause — การประกาศข้อมูลแบบ "ไม่มีที่อยู่ตายตัว"

### BASED คืออะไร

โดยปกติ ตัวแปรใน `WORKING-STORAGE SECTION` จะถูกจัดสรรหน่วยความจำให้ทันทีตั้งแต่โปรแกรมเริ่มทำงาน
(ที่อยู่คงที่ตลอดการทำงาน) แต่รายการข้อมูลที่ประกาศด้วย clause **`BASED`** จะแตกต่างออกไป:
**มันไม่มีหน่วยความจำเป็นของตัวเองเลยตั้งแต่แรก** ต้องรอให้มีการกำหนดที่อยู่ให้ก่อน (ผ่าน
`SET ADDRESS OF` หรือ `ALLOCATE`) จึงจะใช้งานได้จริง

### กฎสำคัญ: BASED ใช้ได้เฉพาะระดับ 01/77

```cobol
      *> This FAILS to compile: BASED must be at 01/77 level
       01  WS-NODE.
           05  WS-NODE-VALUE   PIC 9(5) BASED.
```

ทดสอบคอมไพล์โค้ดข้างต้นจริงจะได้ error:

```
error: BASED only allowed at 01/77 level
```

รูปแบบที่ถูกต้องต้องประกาศ `BASED` ที่ตัวรายการข้อมูลระดับ 01 โดยตรง:

```cobol
       01  WS-NODE-VALUE   PIC 9(5) BASED.
```

### ตัวอย่าง: ประกาศ BASED item และผูกที่อยู่ด้วย ADDRESS OF

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BASED0.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-REAL-DATA    PIC 9(5) VALUE 555.
       01  WS-VIEW         PIC 9(5) BASED.
       PROCEDURE DIVISION.
      *> WS-VIEW has no memory of its own until we point it
      *> somewhere with SET ADDRESS OF.
           SET ADDRESS OF WS-VIEW TO ADDRESS OF WS-REAL-DATA
           DISPLAY "Reading through WS-VIEW: " WS-VIEW
           MOVE 777 TO WS-VIEW
           DISPLAY "WS-REAL-DATA is now: " WS-REAL-DATA
           STOP RUN.
```

**ผลลัพธ์:**

```
Reading through WS-VIEW: 00555
WS-REAL-DATA is now: 00777
```

### อธิบายจุดสำคัญ

- `01 WS-VIEW PIC 9(5) BASED.`: ประกาศ `WS-VIEW` เป็น `BASED` item — จนกว่าจะมีการ `SET ADDRESS OF`
  มันจะไม่มีหน่วยความจำจริงรองรับ
- `SET ADDRESS OF WS-VIEW TO ADDRESS OF WS-REAL-DATA`: ทำให้ `WS-VIEW` "สวมทับ" ตำแหน่งเดียวกันกับ
  `WS-REAL-DATA` ตั้งแต่นี้เป็นต้นไป การอ่าน/เขียน `WS-VIEW` ก็คือการอ่าน/เขียน `WS-REAL-DATA` ตัวจริง
- เทคนิคนี้มีประโยชน์เมื่อเราต้องการ "มุมมองอื่น" (alternate view) ของข้อมูลเดียวกันโดยไม่ต้องคัดลอก
  ข้อมูล — แนวคิดใกล้เคียงกับ `REDEFINES` (Part 022) แต่ยืดหยุ่นกว่าตรงที่กำหนดที่อยู่ได้แบบไดนามิก
  ระหว่างการทำงาน ไม่ใช่ตายตัวตั้งแต่ตอน compile

### ข้อควรระวัง

- ตัวแปร `BASED` ที่ยังไม่เคยถูกกำหนดที่อยู่ (ยังไม่ `SET ADDRESS OF` หรือ `ALLOCATE`) แล้วถูกนำไปใช้
  งานทันที (อ่าน/เขียนค่า) จะทำให้พฤติกรรมไม่แน่นอนหรือโปรแกรม crash — **ต้องกำหนดที่อยู่ก่อนใช้งาน
  เสมอ** ไม่มีข้อยกเว้น
- อย่าใช้ `BASED` เพียงเพื่อความคุ้นเคยกับภาษาอื่นโดยไม่จำเป็น สำหรับงานส่วนใหญ่ที่ขนาดข้อมูลรู้
  ล่วงหน้า `OCCURS` และตัวแปรปกติยังคงเป็นทางเลือกที่ปลอดภัยและอ่านง่ายกว่ามาก `BASED` ควรสงวนไว้
  สำหรับกรณีที่ต้องการโครงสร้างข้อมูลแบบไดนามิกจริง ๆ (เช่น Linked List ในขั้นตอนที่ 437)

### แบบฝึกหัดที่ 434.1

**โจทย์**: จงอธิบายว่าทำไมโค้ดต่อไปนี้จึง compile ไม่ผ่าน และควรแก้ไขอย่างไร

```cobol
       01  WS-GROUP.
           05  WS-FIELD-A  PIC 9(3) BASED.
           05  WS-FIELD-B  PIC X(5).
```

**เฉลย**: `BASED` ถูกใส่ไว้ที่ field ย่อยระดับ 05 ซึ่งไม่ได้รับอนุญาต (BASED ใช้ได้เฉพาะระดับ 01
หรือ 77 เท่านั้น) วิธีแก้คือย้าย `BASED` ไปไว้ที่ระดับ 01 ของกลุ่มทั้งหมดแทน:

```cobol
       01  WS-GROUP BASED.
           05  WS-FIELD-A  PIC 9(3).
           05  WS-FIELD-B  PIC X(5).
```

---

## ขั้นตอนที่ 435: ALLOCATE และ FREE — การจัดสรรหน่วยความจำแบบไดนามิก

### ยืนยันแล้วว่า GnuCOBOL บิลด์นี้รองรับ ALLOCATE/FREE เต็มรูปแบบ

`ALLOCATE`/`FREE` เป็นคำสั่งที่เพิ่มเข้ามาตั้งแต่มาตรฐาน COBOL-2002 สำหรับจัดสรรและคืนหน่วยความจำ
ระหว่างการทำงานของโปรแกรม (คล้าย `malloc()`/`free()` ในภาษา C) จากการทดสอบจริง
**ยืนยันว่า `cobc (GnuCOBOL) 4.0-early-dev.0` รองรับคำสั่งนี้ทำงานได้ถูกต้องสมบูรณ์**

### ตัวอย่าง: ALLOCATE หน่วยความจำใหม่แล้ว FREE คืน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BASED2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NODE-PTR     USAGE POINTER.
       01  WS-NODE-VALUE   PIC 9(5) BASED.
       PROCEDURE DIVISION.
           ALLOCATE WS-NODE-VALUE RETURNING WS-NODE-PTR
           MOVE 42 TO WS-NODE-VALUE
           DISPLAY "Value in allocated storage: " WS-NODE-VALUE
           FREE WS-NODE-PTR
           STOP RUN.
```

**ผลลัพธ์:**

```
Value in allocated storage: 00042
```

### อธิบายจุดสำคัญ

- `ALLOCATE WS-NODE-VALUE RETURNING WS-NODE-PTR`: จัดสรรหน่วยความจำใหม่ขนาดเท่ากับ `WS-NODE-VALUE`
  (ซึ่งประกาศเป็น `BASED`) แล้ว**ผูกที่อยู่ให้กับ `WS-NODE-VALUE` โดยอัตโนมัติทันที** พร้อมทั้งคืนค่า
  ที่อยู่นั้นออกมาเก็บใน `WS-NODE-PTR` ผ่าน `RETURNING` — สังเกตว่าไม่ต้อง `SET ADDRESS OF` เองอีก
  เพราะ `ALLOCATE` ทำให้ในตัว
- `MOVE 42 TO WS-NODE-VALUE`: ใช้งาน `WS-NODE-VALUE` ได้ทันทีเหมือนตัวแปรปกติ เพราะตอนนี้มันมีที่อยู่
  หน่วยความจำจริงรองรับแล้ว
- `FREE WS-NODE-PTR`: คืนหน่วยความจำที่จัดสรรไว้กลับสู่ระบบ — **สำคัญมาก** เพราะถ้าไม่ `FREE` จะเกิด
  Memory Leak (หน่วยความจำรั่วไหล) สะสมไปเรื่อย ๆ ตลอดการทำงานของโปรแกรม

### ข้อควรระวัง

- **ทุก `ALLOCATE` ควรมี `FREE` คู่กันเสมอ** เมื่อไม่ต้องการใช้ข้อมูลนั้นแล้ว มิฉะนั้นจะเกิด Memory
  Leak โดยเฉพาะในโปรแกรมที่ `ALLOCATE` ซ้ำ ๆ ในลูปจำนวนมาก
- หลังจาก `FREE WS-NODE-PTR` แล้ว **ห้ามใช้งาน `WS-NODE-VALUE` อีก** เพราะที่อยู่ที่มันเคยผูกอยู่ถูก
  คืนกลับสู่ระบบไปแล้ว การเข้าถึงข้อมูลหลัง `FREE` (เรียกว่า "dangling pointer" หรือ "use-after-free")
  เป็นบั๊กร้ายแรงที่พบบ่อยในโปรแกรมที่จัดการหน่วยความจำเอง

### แบบฝึกหัดที่ 435.1

**โจทย์**: จงเขียนโปรแกรมที่ `ALLOCATE` ตัวแปร `PIC X(10) BASED` สองตัว เก็บข้อความ "FIRST" และ
"SECOND" ตามลำดับ แสดงผลทั้งสอง แล้ว `FREE` คืนทั้งคู่

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TWOALLOC.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PTR-1        USAGE POINTER.
       01  WS-PTR-2        USAGE POINTER.
       01  WS-TEXT-1       PIC X(10) BASED.
       01  WS-TEXT-2       PIC X(10) BASED.
       PROCEDURE DIVISION.
           ALLOCATE WS-TEXT-1 RETURNING WS-PTR-1
           ALLOCATE WS-TEXT-2 RETURNING WS-PTR-2
           MOVE "FIRST" TO WS-TEXT-1
           MOVE "SECOND" TO WS-TEXT-2
           DISPLAY "1: " WS-TEXT-1
           DISPLAY "2: " WS-TEXT-2
           FREE WS-PTR-1
           FREE WS-PTR-2
           STOP RUN.
```

หมายเหตุ: เนื่องจาก `WS-TEXT-1` และ `WS-TEXT-2` เป็นตัวแปร `BASED` คนละตัว ต้องมี `WS-PTR` แยกกันคนละ
ตัวสำหรับแต่ละ allocation จะใช้ pointer ตัวเดียวสลับกันไม่ได้

---

## ขั้นตอนที่ 436: ตัวเลือกเพิ่มเติมของ ALLOCATE — INITIALIZED และ CHARACTERS

### ALLOCATE ... INITIALIZED

ปกติหน่วยความจำที่ `ALLOCATE` มาใหม่อาจมีค่าขยะ (garbage) หลงเหลืออยู่ การเติมคำว่า `INITIALIZED`
จะสั่งให้ COBOL ล้างค่าเริ่มต้นให้ (เหมือนกับ `INITIALIZE` statement ที่เรียนมาก่อนหน้า) ยืนยันจากการ
ทดสอบว่าใช้งานได้จริงในบิลด์นี้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ALLOCINIT.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PTR      USAGE POINTER.
       01  WS-ITEM     PIC 9(5) BASED.
       PROCEDURE DIVISION.
           ALLOCATE WS-ITEM INITIALIZED RETURNING WS-PTR
           DISPLAY "Initialized value: " WS-ITEM
           FREE WS-PTR
           STOP RUN.
```

**ผลลัพธ์:**

```
Initialized value: 00000
```

### ALLOCATE ... CHARACTERS — จัดสรรหน่วยความจำตามจำนวนไบต์ที่กำหนดตอนรัน

รูปแบบ `ALLOCATE numeric-item CHARACTERS RETURNING pointer-item` ใช้จัดสรรหน่วยความจำแบบดิบ
(raw buffer) ตามจำนวนไบต์ที่ระบุ ซึ่งจำนวนนั้นสามารถเป็นค่าที่คำนวณได้ระหว่างการทำงาน (ไม่ต้องรู้
ล่วงหน้าตอน compile):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ALLOCCNT.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PTR       USAGE POINTER.
       01  WS-SIZE      PIC 9(4) VALUE 10.
       01  WS-BUFFER    PIC X BASED.
       PROCEDURE DIVISION.
           ALLOCATE WS-SIZE CHARACTERS RETURNING WS-PTR
           DISPLAY "Allocated a buffer of " WS-SIZE " characters"
           FREE WS-PTR
           STOP RUN.
```

**ผลลัพธ์:**

```
Allocated a buffer of 0010 characters
```

### อธิบายจุดสำคัญ

- `ALLOCATE WS-ITEM INITIALIZED RETURNING WS-PTR`: หลัง allocate ค่าใน `WS-ITEM` จะถูกล้างเป็น
  `00000` แทนที่จะเป็นค่าขยะ — แนะนำให้ใช้ `INITIALIZED` เสมอเมื่อความถูกต้องของค่าเริ่มต้นสำคัญ
- `ALLOCATE WS-SIZE CHARACTERS RETURNING WS-PTR`: จัดสรรหน่วยความจำขนาด `WS-SIZE` ไบต์ (ในที่นี้คือ
  10 ไบต์) โดยที่ `WS-SIZE` เป็นตัวแปรที่กำหนดค่าได้ระหว่างรัน — เหมาะสำหรับสร้างบัฟเฟอร์ที่ขนาดไม่
  ทราบล่วงหน้าตอนเขียนโปรแกรม เช่น ขนาดที่คำนวณจากข้อมูล input จริง

### ข้อควรระวัง

- `ALLOCATE ... CHARACTERS` ให้หน่วยความจำแบบดิบที่ยังไม่มีโครงสร้างข้อมูลกำกับชัดเจน (ในตัวอย่างใช้
  `WS-BUFFER PIC X BASED` เป็นเพียงตัวยึดสำหรับผูก `ADDRESS OF` เท่านั้น) การเข้าถึงข้อมูลในบัฟเฟอร์
  ลักษณะนี้อย่างปลอดภัยต้องอาศัยเทคนิค reference modification หรือ `REDEFINES` เพิ่มเติม ซึ่งเป็นเรื่อง
  ขั้นสูงที่ควรใช้ด้วยความระมัดระวัง
- อย่าลืม `FREE` หน่วยความจำที่ `ALLOCATE ... CHARACTERS` มาเช่นเดียวกับ `ALLOCATE` แบบปกติ

### แบบฝึกหัดที่ 436.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `ALLOCATE WS-ITEM RETURNING WS-PTR` (ไม่มี `INITIALIZED`)
กับ `ALLOCATE WS-ITEM INITIALIZED RETURNING WS-PTR`

**เฉลย**: แบบแรกจัดสรรหน่วยความจำให้ แต่**ไม่รับประกันค่าเริ่มต้น** ของข้อมูลในนั้น (อาจมีค่าขยะจาก
การใช้งานหน่วยความจำครั้งก่อนหน้าของระบบปฏิบัติการหลงเหลืออยู่) ส่วนแบบที่สองจะ**ล้างค่าเริ่มต้นให้
เป็นค่าว่างมาตรฐานตามชนิดข้อมูล** (เช่น ศูนย์สำหรับตัวเลข, ช่องว่างสำหรับข้อความ) ทันทีหลัง allocate
เสร็จ ทำให้ปลอดภัยกว่าเมื่อเราต้องพึ่งพาค่าเริ่มต้นที่แน่นอนของข้อมูล

---

## ขั้นตอนที่ 437: สร้าง Linked List ด้วย BASED Storage และ POINTER

### แนวคิด: Node ที่ชี้ไปยัง Node ถัดไป

โครงสร้างข้อมูลแบบ **Linked List** ประกอบด้วย "โหนด" (node) หลายตัวที่แต่ละตัวเก็บทั้งข้อมูลและ
**pointer ชี้ไปยังโหนดถัดไป** ทำให้เพิ่ม/ลบข้อมูลได้แบบไดนามิกโดยไม่ต้องกำหนดขนาดตายตัวล่วงหน้าเหมือน
`OCCURS` นี่คือตัวอย่างการนำ `BASED` + `POINTER` + `ALLOCATE` มาผสานกันเป็นโครงสร้างข้อมูลที่ใช้งาน
ได้จริง

### ตัวอย่าง: สร้างและเดินผ่าน (Traverse) Linked List

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LINKLIST.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HEAD-PTR         USAGE POINTER.
       01  WS-CURRENT-PTR      USAGE POINTER.
       01  WS-NEW-PTR          USAGE POINTER.
       01  WS-NODE BASED.
           05  WS-NODE-NAME    PIC X(10).
           05  WS-NODE-NEXT    USAGE POINTER.
       01  WS-I                PIC 9 VALUE 1.
       01  WS-NAMES.
           05  PIC X(10) VALUE "ANONG".
           05  PIC X(10) VALUE "BOONMEE".
           05  PIC X(10) VALUE "CHAIYA".
       01  WS-NAMES-TABLE REDEFINES WS-NAMES.
           05  WS-NAME-ITEM OCCURS 3 TIMES PIC X(10).
       PROCEDURE DIVISION.
           SET WS-HEAD-PTR TO NULL
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
      *> Allocate a new node, fill it in, then link it to the
      *> front of the list (this builds the list in reverse
      *> insertion order - a classic "push to head" pattern).
               ALLOCATE WS-NODE RETURNING WS-NEW-PTR
               SET ADDRESS OF WS-NODE TO WS-NEW-PTR
               MOVE WS-NAME-ITEM(WS-I) TO WS-NODE-NAME
               SET WS-NODE-NEXT TO WS-HEAD-PTR
               SET WS-HEAD-PTR TO WS-NEW-PTR
           END-PERFORM

           DISPLAY "Walking linked list from head:"
           SET WS-CURRENT-PTR TO WS-HEAD-PTR
           PERFORM UNTIL WS-CURRENT-PTR EQUAL NULL
               SET ADDRESS OF WS-NODE TO WS-CURRENT-PTR
               DISPLAY "  Node: " WS-NODE-NAME
               SET WS-CURRENT-PTR TO WS-NODE-NEXT
           END-PERFORM
           STOP RUN.
```

**ผลลัพธ์:**

```
Walking linked list from head:
  Node: CHAIYA
  Node: BOONMEE
  Node: ANONG
```

### อธิบายจุดสำคัญ

- `01 WS-NODE BASED.` มี field ย่อยสองตัว: `WS-NODE-NAME` (ข้อมูล) และ `WS-NODE-NEXT` (pointer ชี้ไป
  โหนดถัดไป) — โครงสร้างคลาสสิกของ Linked List Node
- ในลูปสร้างโหนด: `ALLOCATE WS-NODE RETURNING WS-NEW-PTR` จัดสรรโหนดใหม่ แล้ว
  `SET ADDRESS OF WS-NODE TO WS-NEW-PTR` ผูกให้ `WS-NODE` (ตัวแปรเดียวที่ใช้ซ้ำ) ชี้ไปยังโหนดที่
  เพิ่งสร้าง จากนั้นกรอกข้อมูลและเชื่อมโหนดใหม่เข้ากับหัวลิสต์เดิม (`WS-NODE-NEXT` ชี้ไปยัง
  `WS-HEAD-PTR` เก่า) แล้วขยับ `WS-HEAD-PTR` มาที่โหนดใหม่ — ผลลัพธ์คือลิสต์ที่เรียงย้อนลำดับการ
  เพิ่มข้อมูล (แทรกล่าสุดจะอยู่หัวลิสต์เสมอ จึงเห็นผลลัพธ์เป็น CHAIYA, BOONMEE, ANONG)
- ในลูปเดิน (traverse): `SET ADDRESS OF WS-NODE TO WS-CURRENT-PTR` ทำให้ `WS-NODE` "มอง" ไปยังโหนด
  ปัจจุบันที่กำลังเดินอยู่ อ่านชื่อออกมาแสดง แล้วขยับ `WS-CURRENT-PTR` ไปยังโหนดถัดไปผ่าน
  `WS-NODE-NEXT` จนกว่าจะถึง `NULL` (ปลายลิสต์)

### ข้อควรระวัง

- ตัวแปร `WS-NODE` เพียงตัวเดียวถูกใช้ "มอง" โหนดที่ต่างกันไปเรื่อย ๆ ตลอดโปรแกรม (ผ่านการเปลี่ยน
  `ADDRESS OF`) — เป็นรูปแบบการเขียนโค้ดที่มีประสิทธิภาพแต่ต้องระวังสับสน เพราะ ณ เวลาใดเวลาหนึ่ง
  `WS-NODE` จะสะท้อนเฉพาะโหนดที่ถูกชี้ล่าสุดเท่านั้น
- ตัวอย่างนี้**ไม่ได้ `FREE`** โหนดที่สร้างขึ้น เพื่อความกระชับของตัวอย่างสอน ในโปรแกรมจริงควรวนลูป
  `FREE` ทุกโหนดก่อนจบโปรแกรม (ดูตัวอย่างแบบเต็มวงจรใน ขั้นตอนที่ 440)

### แบบฝึกหัดที่ 437.1

**โจทย์**: จงอธิบายว่าทำไมผลลัพธ์การเดินลิสต์จึงได้ลำดับ CHAIYA, BOONMEE, ANONG แทนที่จะเป็น
ANONG, BOONMEE, CHAIYA (ลำดับที่ใส่เข้าไปจริง)

**เฉลย**: เพราะรูปแบบการเชื่อมโหนดในโปรแกรมนี้คือ "แทรกที่หัวลิสต์เสมอ" (`SET WS-NODE-NEXT TO
WS-HEAD-PTR` ตามด้วย `SET WS-HEAD-PTR TO WS-NEW-PTR`) ซึ่งหมายความว่าโหนดที่เพิ่มล่าสุดจะกลายเป็น
หัวลิสต์ใหม่เสมอ เมื่อเพิ่ม ANONG ก่อน มันจะกลายเป็นหัวลิสต์ชั่วคราว จากนั้นเพิ่ม BOONMEE ซึ่งจะแทรก
นำหน้า ANONG และสุดท้าย CHAIYA จะแทรกนำหน้าสุด ทำให้เมื่อเดินลิสต์จากหัว จึงได้ลำดับย้อนกลับ:
CHAIYA, BOONMEE, ANONG

---

## ขั้นตอนที่ 438: ส่ง POINTER ข้ามโปรแกรมเพื่อแก้ไขข้อมูลปลายทาง

### ทบทวนและขยายความจากขั้นตอนที่ 433

ขั้นตอนที่ 433 แสดงตัวอย่างพื้นฐานของการส่ง `POINTER` เป็นพารามิเตอร์ให้โปรแกรมย่อยแล้วใช้
`SET ADDRESS OF` แก้ไขข้อมูลปลายทาง ในขั้นตอนนี้เราจะดูมุมมองที่กว้างขึ้น: ทำไมเทคนิคนี้จึงมีประโยชน์
ในทางปฏิบัติ โดยเฉพาะเมื่อโปรแกรมย่อยต้อง "เลือก" ว่าจะแก้ไขข้อมูลก้อนไหนจากหลาย ๆ ก้อนที่อาจถูกส่ง
เข้ามา

### ตัวอย่าง: โปรแกรมย่อยเลือกปรับปรุงหนึ่งในสองก้อนข้อมูลตามเงื่อนไข

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UPDATER.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TARGET       PIC 9(5) BASED.
       LINKAGE SECTION.
       01  LS-USE-SECOND   PIC X.
       01  LS-PTR-FIRST    USAGE POINTER.
       01  LS-PTR-SECOND   USAGE POINTER.
       PROCEDURE DIVISION USING LS-USE-SECOND
           LS-PTR-FIRST LS-PTR-SECOND.
           IF LS-USE-SECOND = "Y"
               SET ADDRESS OF WS-TARGET TO LS-PTR-SECOND
           ELSE
               SET ADDRESS OF WS-TARGET TO LS-PTR-FIRST
           END-IF
           ADD 100 TO WS-TARGET
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAINUPD.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-BALANCE-A    PIC 9(5) VALUE 1000.
       01  WS-BALANCE-B    PIC 9(5) VALUE 2000.
       01  WS-PTR-A        USAGE POINTER.
       01  WS-PTR-B        USAGE POINTER.
       PROCEDURE DIVISION.
           SET WS-PTR-A TO ADDRESS OF WS-BALANCE-A
           SET WS-PTR-B TO ADDRESS OF WS-BALANCE-B
           DISPLAY "Before: A=" WS-BALANCE-A " B=" WS-BALANCE-B
           CALL "UPDATER" USING "Y" WS-PTR-A WS-PTR-B
           DISPLAY "After:  A=" WS-BALANCE-A " B=" WS-BALANCE-B
           STOP RUN.
```

**ผลลัพธ์:**

```
Before: A=01000 B=02000
After:  A=01000 B=02100
```

### อธิบายจุดสำคัญ

- `MAINUPD` ส่งทั้งสองที่อยู่ (`WS-PTR-A`, `WS-PTR-B`) และ flag (`"Y"`) ไปให้ `UPDATER`
- `UPDATER` **ตัดสินใจเองตอนรัน** ว่าจะแก้ไขข้อมูลก้อนไหนจากค่า `LS-USE-SECOND` — ในตัวอย่างนี้ส่ง
  `"Y"` มา จึงเลือกแก้ `LS-PTR-SECOND` (ซึ่งชี้ไปที่ `WS-BALANCE-B`) ทำให้ `WS-BALANCE-B` เพิ่มขึ้น
  100 เป็น 2100 ในขณะที่ `WS-BALANCE-A` ไม่เปลี่ยนแปลง
- รูปแบบนี้มีประโยชน์มากในสถานการณ์ เช่น โปรแกรมย่อยที่ต้อง "เลือก account ปลายทางที่จะปรับยอด" จาก
  เงื่อนไขทางธุรกิจที่ซับซ้อน โดยผู้เรียกไม่ต้องรู้ล่วงหน้าว่าโปรแกรมย่อยจะเลือกอันไหน

### ข้อควรระวัง

- ยิ่งจำนวน pointer ที่ส่งเข้าไปมากขึ้น ความซับซ้อนในการดูแลโค้ดก็ยิ่งสูงขึ้นตามไปด้วย ควรมี
  คอมเมนต์อธิบายสัญญา (contract) ของโปรแกรมย่อยอย่างชัดเจนเสมอว่าพารามิเตอร์แต่ละตัวมีความหมายว่า
  อย่างไร
- ในตัวอย่างนี้ `"Y"` ถูกส่งเป็น literal ตรง ๆ ผ่าน `USING` ซึ่งใช้ได้เพราะพารามิเตอร์ตัวแรกรับแบบ
  `BY REFERENCE` ปกติ (ไม่ใช่ `POINTER`) — GnuCOBOL จะสร้างพื้นที่ชั่วคราวเก็บ literal นั้นให้อัตโนมัติ

### แบบฝึกหัดที่ 438.1

**โจทย์**: จงปรับ `MAINUPD` ให้ส่ง `"N"` แทน `"Y"` แล้วคาดเดาว่าผลลัพธ์จะเปลี่ยนไปอย่างไร

**เฉลย**: เมื่อส่ง `"N"` เงื่อนไข `IF LS-USE-SECOND = "Y"` จะเป็นเท็จ ทำให้ `UPDATER` เลือก
`SET ADDRESS OF WS-TARGET TO LS-PTR-FIRST` แทน ส่งผลให้ `WS-BALANCE-A` ถูกเพิ่ม 100 กลายเป็น 1100
ในขณะที่ `WS-BALANCE-B` ยังคงเป็น 2000 เหมือนเดิม (ผลลัพธ์ตรงข้ามกับตัวอย่างต้นฉบับ)

---

## ขั้นตอนที่ 439: ข้อควรระวังและกับดักที่พบบ่อยของการเขียนโปรแกรมด้วย POINTER/BASED

### สรุปกับดักสำคัญที่ต้องระวัง

1. **Dangling Pointer (ตัวชี้ห้อยค้าง)**: การใช้งาน pointer ที่ชี้ไปยังหน่วยความจำที่ถูก `FREE`
   ไปแล้ว เป็นสาเหตุอันดับหนึ่งของบั๊กร้ายแรงในโปรแกรมที่จัดการหน่วยความจำเอง
2. **Memory Leak (หน่วยความจำรั่วไหล)**: ลืม `FREE` หน่วยความจำที่ `ALLOCATE` มา ทำให้โปรแกรมใช้
   หน่วยความจำเพิ่มขึ้นเรื่อย ๆ โดยไม่มีการคืน โดยเฉพาะอันตรายในโปรแกรมที่ทำงานต่อเนื่องเป็นเวลานาน
   (long-running process)
3. **NULL Dereference**: การใช้งาน pointer ที่เป็น `NULL` โดยไม่ตรวจสอบก่อน ทำให้โปรแกรม crash
4. **BASED item ที่ยังไม่ผูกที่อยู่**: การใช้งานตัวแปร `BASED` ก่อนที่จะ `SET ADDRESS OF` หรือ
   `ALLOCATE` ให้ ทำให้พฤติกรรมไม่แน่นอน
5. **ลืมว่า pointer สองตัวชี้ไปที่เดียวกัน (Aliasing)**: เมื่อ pointer หลายตัวชี้ไปยังข้อมูลก้อน
   เดียวกัน การแก้ไขผ่าน pointer ตัวหนึ่งจะกระทบกับอีกตัวโดยไม่รู้ตัว หากไม่ได้ออกแบบมาให้ตั้งใจ

### ตัวอย่าง: สาธิตปัญหา Aliasing (ตั้งใจแสดงพฤติกรรม ไม่ใช่บั๊ก)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ALIASDEMO.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SHARED       PIC 9(5) VALUE 1000.
       01  WS-PTR-ONE      USAGE POINTER.
       01  WS-PTR-TWO      USAGE POINTER.
       01  WS-VIEW-A       PIC 9(5) BASED.
       01  WS-VIEW-B       PIC 9(5) BASED.
       PROCEDURE DIVISION.
      *> Both pointers deliberately point to the SAME data.
           SET WS-PTR-ONE TO ADDRESS OF WS-SHARED
           SET WS-PTR-TWO TO ADDRESS OF WS-SHARED
           SET ADDRESS OF WS-VIEW-A TO WS-PTR-ONE
           SET ADDRESS OF WS-VIEW-B TO WS-PTR-TWO

           DISPLAY "Before: VIEW-A=" WS-VIEW-A
               " VIEW-B=" WS-VIEW-B
           MOVE 9999 TO WS-VIEW-A
           DISPLAY "After changing VIEW-A only:"
           DISPLAY "  VIEW-A=" WS-VIEW-A " VIEW-B=" WS-VIEW-B
           STOP RUN.
```

**ผลลัพธ์:**

```
Before: VIEW-A=01000 VIEW-B=01000
After changing VIEW-A only:
  VIEW-A=09999 VIEW-B=09999
```

### อธิบายจุดสำคัญ

- แม้จะแก้ไขผ่าน `WS-VIEW-A` เพียงตัวเดียว แต่ `WS-VIEW-B` ก็เปลี่ยนค่าตามไปด้วย เพราะทั้งคู่ผูกอยู่
  กับที่อยู่หน่วยความจำเดียวกัน (`WS-SHARED`) — นี่ไม่ใช่บั๊ก แต่เป็นพฤติกรรมที่ถูกต้องตามหลักการของ
  pointer ถ้าโปรแกรมเมอร์ไม่ได้ตั้งใจให้เกิดผลแบบนี้ อาจนำไปสู่บั๊กที่ตามรอยยากมาก เพราะจุดที่แก้ไข
  ข้อมูล (`MOVE 9999 TO WS-VIEW-A`) ดูเหมือนไม่เกี่ยวข้องกับ `WS-VIEW-B` เลยเมื่อมองแค่บรรทัดนั้น

### ข้อควรระวัง

- ก่อนใช้ pointer หลายตัวในโปรแกรมเดียวกัน ควรวาดแผนภาพหรือจดบันทึกไว้เสมอว่า pointer ตัวไหนชี้ไปยัง
  ข้อมูลก้อนไหนบ้าง โดยเฉพาะเมื่อโปรแกรมมีความซับซ้อนมากขึ้น เพื่อป้องกันความสับสนเรื่อง Aliasing
- ในโค้ด production ควรตั้งชื่อตัวแปร pointer และ based item ให้สื่อความหมายชัดเจนว่ามันเกี่ยวข้อง
  กับข้อมูลใด เพื่อลดความเสี่ยงจากความสับสนประเภทนี้

### แบบฝึกหัดที่ 439.1

**โจทย์**: จากตัวอย่าง `ALIASDEMO` หากต้องการให้ `WS-VIEW-A` และ `WS-VIEW-B` เป็นอิสระต่อกันอย่าง
แท้จริง (แก้ไขตัวหนึ่งไม่กระทบอีกตัว) ควรแก้ไขโค้ดส่วนใด

**เฉลย**: ต้องให้แต่ละ view ชี้ไปยังหน่วยความจำคนละก้อนกัน เช่น ประกาศตัวแปรข้อมูลจริงแยกกันสองตัว
(`WS-SHARED-A`, `WS-SHARED-B`) แล้วให้ `WS-PTR-ONE` ชี้ไปที่ `WS-SHARED-A` และ `WS-PTR-TWO` ชี้ไปที่
`WS-SHARED-B` แทนการชี้ไปยัง `WS-SHARED` ตัวเดียวกันทั้งคู่ หรือถ้าต้องการข้อมูลอิสระจริง ๆ ควร
`ALLOCATE` หน่วยความจำใหม่แยกกันให้แต่ละ view แทนการใช้ `ADDRESS OF` ตัวแปรเดียวกัน

---

## ขั้นตอนที่ 440: ตัวอย่างรวม — Stack แบบไดนามิกด้วย ALLOCATE/FREE

### ภาพรวมโปรแกรมสุดท้ายของ Part นี้

เราจะรวมทุกเทคนิคที่เรียนมา: `BASED` storage, `POINTER`, `ALLOCATE`, และ `FREE` เพื่อสร้าง
**Stack** (โครงสร้างข้อมูลแบบ LIFO — Last In, First Out) ที่จัดสรรและคืนหน่วยความจำอย่างถูกต้อง
ครบวงจร ไม่มี Memory Leak หลงเหลือ

### โปรแกรมสมบูรณ์: STACK1

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STACK1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TOP-PTR      USAGE POINTER.
       01  WS-NEW-PTR      USAGE POINTER.
       01  WS-NODE BASED.
           05  WS-NODE-DATA    PIC 9(5).
           05  WS-NODE-NEXT    USAGE POINTER.
       01  WS-VALUE        PIC 9(5).
       PROCEDURE DIVISION.
           SET WS-TOP-PTR TO NULL

      *> Push 3 values: 10, 20, 30
           ALLOCATE WS-NODE RETURNING WS-NEW-PTR
           SET ADDRESS OF WS-NODE TO WS-NEW-PTR
           MOVE 10 TO WS-NODE-DATA
           SET WS-NODE-NEXT TO WS-TOP-PTR
           SET WS-TOP-PTR TO WS-NEW-PTR

           ALLOCATE WS-NODE RETURNING WS-NEW-PTR
           SET ADDRESS OF WS-NODE TO WS-NEW-PTR
           MOVE 20 TO WS-NODE-DATA
           SET WS-NODE-NEXT TO WS-TOP-PTR
           SET WS-TOP-PTR TO WS-NEW-PTR

           ALLOCATE WS-NODE RETURNING WS-NEW-PTR
           SET ADDRESS OF WS-NODE TO WS-NEW-PTR
           MOVE 30 TO WS-NODE-DATA
           SET WS-NODE-NEXT TO WS-TOP-PTR
           SET WS-TOP-PTR TO WS-NEW-PTR

           DISPLAY "Popping the stack (should be 30, 20, 10):"
           PERFORM UNTIL WS-TOP-PTR EQUAL NULL
      *> Look at the top node, read its value, unlink it from
      *> the stack, then FREE it - a complete push/pop lifecycle
      *> with no memory leaks.
               SET ADDRESS OF WS-NODE TO WS-TOP-PTR
               MOVE WS-NODE-DATA TO WS-VALUE
               DISPLAY "  Popped: " WS-VALUE
               SET WS-NEW-PTR TO WS-TOP-PTR
               SET WS-TOP-PTR TO WS-NODE-NEXT
               FREE WS-NEW-PTR
           END-PERFORM
           STOP RUN.
```

**ผลลัพธ์:**

```
Popping the stack (should be 30, 20, 10):
  Popped: 00030
  Popped: 00020
  Popped: 00010
```

### อธิบายจุดสำคัญ

- **Push (การเพิ่มเข้า stack)**: ทำซ้ำ 3 ครั้งด้วยรูปแบบเดียวกัน — `ALLOCATE` โหนดใหม่, กรอกข้อมูล,
  เชื่อม `WS-NODE-NEXT` เข้ากับยอด stack เดิม (`WS-TOP-PTR`), แล้วขยับยอด stack มาที่โหนดใหม่
- **Pop (การดึงออกจาก stack)**: อ่านค่าจากโหนดบนสุด (`WS-TOP-PTR`) เก็บใส่ `WS-VALUE` ก่อน จากนั้น
  จำที่อยู่โหนดนั้นไว้ใน `WS-NEW-PTR` ชั่วคราว ขยับยอด stack ไปยังโหนดถัดไป (`WS-NODE-NEXT`) แล้วจึง
  `FREE WS-NEW-PTR` คืนหน่วยความจำของโหนดที่เพิ่งดึงออกมา — **ขั้นตอนนี้สำคัญมาก**: ต้องขยับ
  `WS-TOP-PTR` ไปยังโหนดถัดไป**ก่อน** `FREE` โหนดเดิม มิฉะนั้นจะเสียตำแหน่งอ้างอิงไปยังโหนดถัดไปอย่าง
  ถาวร (เพราะข้อมูลถูกคืนสู่ระบบไปแล้ว)
- ผลลัพธ์แสดงลำดับ 30, 20, 10 — ตรงข้ามกับลำดับที่ push เข้าไป (10, 20, 30) ซึ่งเป็นพฤติกรรมที่ถูกต้อง
  ของ Stack แบบ LIFO
- เมื่อลูป `PERFORM UNTIL WS-TOP-PTR EQUAL NULL` จบลง หมายความว่าทุกโหนดถูก `FREE` ครบถ้วนแล้ว **ไม่มี
  Memory Leak หลงเหลือ** — ตรงข้ามกับตัวอย่าง Linked List ในขั้นตอนที่ 437 ที่จงใจละเว้นการ `FREE`
  เพื่อความกระชับของตัวอย่าง

### ข้อควรระวัง

- ลำดับการทำงานใน loop pop มีความสำคัญมาก: **ต้องอ่านค่าและขยับ pointer ไปยังโหนดถัดไปก่อน แล้วจึง
  ค่อย `FREE` โหนดเดิม** หากสลับลำดับ (เช่น `FREE` ก่อนแล้วค่อยพยายามอ่าน `WS-NODE-NEXT`) จะเป็นการ
  ใช้งานข้อมูลหลัง `FREE` ไปแล้ว (use-after-free) ซึ่งเป็นพฤติกรรมที่ไม่แน่นอนและอันตราย
- ในระบบจริงที่ stack ต้องรองรับการเรียกพร้อมกันจากหลายจุด (concurrent access) จำเป็นต้องมีกลไก
  ป้องกันการเข้าถึงพร้อมกัน (locking) เพิ่มเติม ซึ่งเกินขอบเขตของ Part นี้แต่ควรตระหนักไว้เมื่อนำ
  แนวคิดนี้ไปใช้งานจริง

### แบบฝึกหัดที่ 440.1

**โจทย์**: จงปรับโปรแกรม `STACK1` ให้มีพารากราฟ `PUSH-VALUE` และ `POP-VALUE` แยกออกมา (ใช้ `PERFORM`
เรียก) แทนที่จะเขียนโค้ด push ซ้ำ 3 รอบตรง ๆ

**เฉลย** (แนวทาง — เนื่องจาก COBOL ไม่รองรับการส่งพารามิเตอร์ให้พารากราฟผ่าน `PERFORM` โดยตรง ต้องใช้
ตัวแปร working-storage ร่วมกันเป็นสื่อกลาง):

```cobol
       01  WS-PUSH-VALUE   PIC 9(5).
       ...
       PROCEDURE DIVISION.
           SET WS-TOP-PTR TO NULL
           MOVE 10 TO WS-PUSH-VALUE
           PERFORM PUSH-VALUE
           MOVE 20 TO WS-PUSH-VALUE
           PERFORM PUSH-VALUE
           MOVE 30 TO WS-PUSH-VALUE
           PERFORM PUSH-VALUE
           ...
           STOP RUN.

       PUSH-VALUE.
           ALLOCATE WS-NODE RETURNING WS-NEW-PTR
           SET ADDRESS OF WS-NODE TO WS-NEW-PTR
           MOVE WS-PUSH-VALUE TO WS-NODE-DATA
           SET WS-NODE-NEXT TO WS-TOP-PTR
           SET WS-TOP-PTR TO WS-NEW-PTR.
```

หมายเหตุ: หากต้องการพารากราฟที่รับพารามิเตอร์จริง ๆ (เหมือนฟังก์ชันในภาษาอื่น) ต้องใช้ `CALL`
subprogram พร้อม `LINKAGE SECTION` แทน (ทบทวนได้จาก Part 031-032 และ Part 043)

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้การจัดการหน่วยความจำแบบไดนามิกและ pointer ทั่วไปใน COBOL อย่างครบถ้วน:

- `POINTER` data item พื้นฐาน และค่าพิเศษ `NULL`
- `ADDRESS OF` special register สำหรับดึงที่อยู่ของรายการข้อมูล
- การส่งที่อยู่ข้ามโปรแกรมด้วย `POINTER` เป็นพารามิเตอร์ เทียบกับการส่งแบบ `BY REFERENCE` ปกติ
- `BASED` clause และกฎที่ต้องประกาศที่ระดับ 01/77 เท่านั้น (ยืนยันจาก compile error จริง)
- `ALLOCATE`/`FREE` สำหรับจัดสรรและคืนหน่วยความจำแบบไดนามิก — **ยืนยันแล้วว่าใช้งานได้เต็มรูปแบบใน
  GnuCOBOL 4.0-early-dev บิลด์นี้** รวมถึงตัวเลือก `INITIALIZED` และ `CHARACTERS`
- การสร้าง Linked List ด้วย `BASED` node ที่มี pointer ชี้ต่อกัน
- กับดักสำคัญ: Dangling Pointer, Memory Leak, NULL Dereference, และ Aliasing
- ตัวอย่างรวม Stack แบบไดนามิกที่จัดการวงจรชีวิตหน่วยความจำอย่างถูกต้องครบถ้วน (push/pop/free)

เทคนิคเหล่านี้เป็นรากฐานสำคัญที่ทำให้ COBOL สมัยใหม่เชื่อมต่อกับโครงสร้างข้อมูลแบบไดนามิกและไลบรารี
ภาษาอื่น (เช่น C) ได้อย่างมีประสิทธิภาพ ใน Part ถัดไปเราจะเปลี่ยนทิศทางไปสู่การแลกเปลี่ยนข้อมูลกับ
โลกภายนอกในรูปแบบที่ใช้กันแพร่หลายที่สุดในปัจจุบัน: **JSON** ผ่านคำสั่ง `JSON GENERATE` และ
`JSON PARSE` ที่ COBOL-2014 เพิ่มเข้ามาเพื่อให้ COBOL คุยกับ REST API สมัยใหม่ได้โดยตรง

**[← กลับไป Part 043: Dynamic CALL และ Program Pointers](part-043-dynamic-call.md)** | **[ไปยัง Part 045: JSON GENERATE และ JSON PARSE →](part-045-json-generate-parse.md)**
