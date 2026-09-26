# Part 031: Subprograms: CALL Statement เบื้องต้น (ขั้นตอนที่ 301–310)

## คำนำของ Part นี้

Part 023–030 พาเราเจาะลึกเรื่องการประมวลผลไฟล์อย่างครบวงจร ถึงเวลาแล้วที่จะเปลี่ยนหัวข้อไปสู่อีก
เสาหลักหนึ่งของการเขียนโปรแกรม COBOL ระดับองค์กร นั่นคือ **การแบ่งโปรแกรมขนาดใหญ่ออกเป็นโปรแกรมย่อย
(Subprograms)** ที่เรียกใช้ซึ่งกันและกันได้

ตลอด Part 001–030 เราแบ่งงานภายในโปรแกรมเดียวด้วย `PERFORM` (Paragraph/Section จาก Part 014)
ซึ่งเหมาะกับการจัดระเบียบโค้ด**ภายในไฟล์เดียวกัน** แต่เมื่อระบบใหญ่ขึ้นเรื่อย ๆ จนมีหลายทีมเขียนโค้ด
ร่วมกัน หรือมีตรรกะบางอย่าง (เช่น สูตรคำนวณภาษี, การตรวจสอบเลขบัตรประชาชน) ที่ต้องใช้ซ้ำใน**หลาย
โปรแกรมที่แยกกันคอมไพล์** `PERFORM` เพียงอย่างเดียวไม่เพียงพออีกต่อไป COBOL จึงมีกลไก **`CALL`
Statement** ที่ทำให้โปรแกรมหนึ่ง (Main Program หรือ Calling Program) เรียกใช้งานอีกโปรแกรมหนึ่ง
(Subprogram หรือ Called Program) ที่**คอมไพล์แยกกันเป็นไฟล์ต่างหาก**ได้ พร้อมส่งข้อมูลไปมาระหว่างกัน

Part นี้จะปูพื้นฐาน `CALL` ให้แน่น: วิธีเขียนโปรแกรมหลักและโปรแกรมย่อยคู่กัน วิธีคอมไพล์รวมสองไฟล์
เข้าด้วยกัน การส่งพารามิเตอร์เบื้องต้นผ่าน `LINKAGE SECTION`, ความแตกต่างสำคัญระหว่าง `GOBACK` กับ
`STOP RUN`, และกับดักที่พบบ่อยที่สุดของมือใหม่ ส่วนรายละเอียดเชิงลึกเรื่อง**วิธีการส่งพารามิเตอร์
ทั้ง 3 แบบ** (`BY REFERENCE`, `BY CONTENT`, `BY VALUE`) จะแยกไปสอนเต็มรูปแบบใน **Part 032**

> **หมายเหตุการคอมไพล์**: ทุกตัวอย่างใน Part นี้ต้องใช้ **สองไฟล์ต้นฉบับ** (โปรแกรมหลักและโปรแกรม
> ย่อย) คอมไพล์รวมกันด้วยคำสั่งรูปแบบ `cobc -x main.cob sub.cob -o program` เสมอ ไม่สามารถคอมไพล์
> ทีละไฟล์แยกกันแล้วรันได้ทันที (ต้อง link เข้าด้วยกันก่อน) รายละเอียดเพิ่มเติมอยู่ในแต่ละขั้นตอน

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่าง
> (รวมถึงตัวอย่างที่ตั้งใจแสดง runtime crash จริง) ผ่านการคอมไพล์และรันทดสอบจริงด้วย GnuCOBOL แล้ว
> ทุกตัวอย่าง

---

## ขั้นตอนที่ 301: CALL และ GOBACK พื้นฐาน — โปรแกรมสองไฟล์แรกของคุณ

### CALL ต่างจาก PERFORM อย่างไร

`PERFORM` (Part 012–014) กระโดดไปทำงานที่ paragraph/section **ภายในโปรแกรมเดียวกัน** เท่านั้น
ส่วน `CALL` คือคำสั่งที่ส่งการควบคุมไปยัง**โปรแกรมอื่นที่คอมไพล์แยกต่างหากเป็นไฟล์คนละไฟล์**
โปรแกรมที่ถูกเรียก (Subprogram) มี `IDENTIFICATION DIVISION` และ `PROCEDURE DIVISION` ของตัวเอง
ครบถ้วน เหมือนเป็นโปรแกรม COBOL อิสระตัวหนึ่ง เพียงแต่ไม่มี `PROGRAM-ID` ที่ตรงกับโปรแกรมที่ run
โดยตรง (ไม่ได้ถูกเรียกด้วยคำสั่ง `./program` แต่ถูกเรียกจากโปรแกรมอื่นด้วย `CALL` เท่านั้น)

**ไฟล์ที่ 1 — โปรแกรมหลัก (`step301main.cob`):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP301MAIN.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main program starting.".
      *> CALL hands control to a SEPARATELY COMPILED program named
      *> STEP301SUB. This is a different kind of "jump" than the
      *> PERFORM you have used since Part 012 - PERFORM only jumps
      *> within the SAME program's own PROCEDURE DIVISION.
           CALL "STEP301SUB".
           DISPLAY "Main program resumed after CALL.".
           STOP RUN.
```

**ไฟล์ที่ 2 — โปรแกรมย่อย (`step301sub.cob`):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP301SUB.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       SUB-MAIN-PARA.
           DISPLAY "  Subprogram STEP301SUB is running.".
      *> GOBACK returns control to whoever called this program -
      *> it does NOT end the whole run, unlike STOP RUN (step 306
      *> explains the difference and its dangers in detail).
           GOBACK.
```

**คอมไพล์ทั้งสองไฟล์รวมกันเป็นโปรแกรมเดียว:**

```bash
cobc -x step301main.cob step301sub.cob -o step301
./step301
```

**ผลลัพธ์:**

```
Main program starting.
  Subprogram STEP301SUB is running.
Main program resumed after CALL.
```

### อธิบายทีละส่วน

- `CALL "STEP301SUB"`: ชื่อโปรแกรมที่เรียกต้องอยู่ในเครื่องหมายคำพูด (literal) และต้องตรงกับ
  `PROGRAM-ID` ของโปรแกรมย่อยเป๊ะ ๆ (COBOL ไม่สนตัวพิมพ์ใหญ่เล็ก แต่ควรเขียนให้ตรงกันเพื่อความ
  ชัดเจน)
- `GOBACK`: คำสั่งที่ใช้ **"จบการทำงานของโปรแกรมย่อยแล้วคืนการควบคุมกลับไปยังผู้เรียก"** เทียบได้กับ
  `return` ในภาษาโปรแกรมอื่น ๆ
- สังเกตลำดับผลลัพธ์: `"Main program starting."` → `"Subprogram ... is running."` → `"Main
  program resumed ..."` — พิสูจน์ว่าการควบคุมไหลจาก main ไปยัง sub แล้วไหลกลับมาที่ main ต่อจาก
  จุดที่เรียก `CALL` พอดี (เหมือน `PERFORM` แต่ข้ามโปรแกรม)
- คำสั่ง `cobc -x step301main.cob step301sub.cob -o step301` คอมไพล์**ทั้งสองไฟล์พร้อมกัน**และ
  เชื่อมโยง (link) เข้าเป็นไฟล์ execute เดียว (`step301`) — ลำดับไฟล์ในคำสั่งไม่สำคัญ (จะสลับ
  `step301sub.cob` มาก่อน `step301main.cob` ก็ได้ผลเหมือนกัน) ตราบใดที่ทั้งสองไฟล์ถูกส่งเข้า `cobc`
  พร้อมกัน

### ข้อควรระวัง

- โปรแกรมย่อยที่ไม่มี `GOBACK` หรือ `STOP RUN` เลยจะทำให้เกิดพฤติกรรมไม่แน่นอน (ไหลตกไปยังอะไรก็ตาม
  ที่อยู่ถัดจาก `PROCEDURE DIVISION` — ถ้าไม่มีอะไรเหลือ compiler มักเติม return ให้อัตโนมัติ แต่ไม่
  ควรพึ่งพาพฤติกรรมนี้)
- ชื่อโปรแกรมใน `CALL "..."` เป็น **string literal คงที่** ไม่ใช่ชื่อไฟล์ (`step301sub.cob`) —
  COBOL อ้างอิงจาก `PROGRAM-ID` ที่ประกาศไว้ภายในไฟล์ ไม่ใช่ชื่อไฟล์บนดิสก์

### แบบฝึกหัดที่ 301.1

**โจทย์**: จงอธิบายว่าทำไมการรันคำสั่ง `cobc -x step301sub.cob -o step301sub` (คอมไพล์เฉพาะไฟล์
subprogram เพียงไฟล์เดียว) แล้วพยายามรัน `./step301sub` จึงไม่ทำงานตามที่คาดหวัง

**เฉลย**: เพราะ `STEP301SUB` ถูกออกแบบมาให้เป็น**โปรแกรมย่อยที่รอรับการเรียกจากโปรแกรมอื่น** ไม่ใช่
โปรแกรมที่ออกแบบมาให้รันเดี่ยว ๆ ด้วยตัวเอง แม้จะคอมไพล์ผ่านได้ (เพราะมีโครงสร้าง COBOL ที่ถูกต้อง
ครบ 4 Divisions พื้นฐาน) แต่เมื่อรันจริงจะเข้าสู่ `PROCEDURE DIVISION` ทำงาน `DISPLAY` แล้วเจอ
`GOBACK` ซึ่งพยายาม "คืนการควบคุมกลับไปยังผู้เรียก" — แต่เพราะรันเป็นโปรแกรมเดี่ยว **ไม่มีผู้เรียก
ให้คืนกลับไปหา** พฤติกรรมในจุดนี้ขึ้นอยู่กับ runtime ของแต่ละ compiler (GnuCOBOL มักจะจบการทำงาน
เหมือน `STOP RUN` ในกรณีนี้) แต่ไม่ใช่การใช้งานที่ถูกออกแบบมาให้ทำเช่นนั้น

---

## ขั้นตอนที่ 302: ส่งพารามิเตอร์ครั้งแรก — USING และ LINKAGE SECTION

### LINKAGE SECTION คือ Division ใหม่ที่ยังไม่เคยเห็น

ก่อนหน้านี้เรารู้จักแค่ `WORKING-STORAGE SECTION` (Part 005) สำหรับข้อมูลที่โปรแกรม**เป็นเจ้าของเอง**
และ `FILE SECTION` (Part 024) สำหรับโครงสร้างไฟล์ ขั้นตอนนี้แนะนำ **`LINKAGE SECTION`** ส่วนที่สาม
ซึ่งใช้อธิบาย**ข้อมูลที่รับมาจากภายนอก**ผ่าน `CALL ... USING` — LINKAGE SECTION ไม่ได้จองพื้นที่
หน่วยความจำใหม่ของตัวเอง แต่เป็นเพียง "ป้ายชื่อ" ที่ชี้ไปยังหน่วยความจำเดียวกันกับที่ผู้เรียกส่งมาให้

**ไฟล์ที่ 1 — โปรแกรมหลัก:**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP302MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-NAME         PIC X(20) VALUE "SOMCHAI JAIDEE".

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: about to CALL, name = " WS-CUSTOMER-NAME.
      *> USING passes WS-CUSTOMER-NAME to the subprogram - the
      *> subprogram's LINKAGE SECTION describes how it RECEIVES
      *> that same piece of memory.
           CALL "STEP302SUB" USING WS-CUSTOMER-NAME.
           DISPLAY "Main: name after CALL   = " WS-CUSTOMER-NAME.
           STOP RUN.
```

**ไฟล์ที่ 2 — โปรแกรมย่อย:**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP302SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
      *> LINKAGE SECTION describes data that comes from OUTSIDE
      *> this program (from whoever CALLs it) - it does NOT
      *> reserve new memory of its own, unlike WORKING-STORAGE.
       LINKAGE SECTION.
       01  LK-CUSTOMER-NAME         PIC X(20).

       PROCEDURE DIVISION USING LK-CUSTOMER-NAME.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: received name = " LK-CUSTOMER-NAME.
           MOVE "PRASERT KAEWTA" TO LK-CUSTOMER-NAME.
           DISPLAY "  Sub: changed it to = " LK-CUSTOMER-NAME.
           GOBACK.
```

```bash
cobc -x step302main.cob step302sub.cob -o step302
./step302
```

**ผลลัพธ์:**

```
Main: about to CALL, name = SOMCHAI JAIDEE
  Sub: received name = SOMCHAI JAIDEE
  Sub: changed it to = PRASERT KAEWTA
Main: name after CALL   = PRASERT KAEWTA
```

### อธิบายจุดสำคัญ

- `CALL "STEP302SUB" USING WS-CUSTOMER-NAME`: `USING` ตามด้วยรายการตัวแปรที่จะส่งไปให้โปรแกรมย่อย
  — ในที่นี้ส่งไปแค่ตัวเดียว
- `PROCEDURE DIVISION USING LK-CUSTOMER-NAME.`: ทุกโปรแกรมย่อยที่รับพารามิเตอร์ต้องประกาศ
  `USING` ต่อท้าย `PROCEDURE DIVISION` ของตัวเองด้วย โดยรายชื่อตัวแปรตรงนี้**อ้างอิงจาก
  `LINKAGE SECTION`** ไม่ใช่ `WORKING-STORAGE SECTION`
- **จุดที่สำคัญที่สุด**: ชื่อตัวแปรฝั่งผู้เรียก (`WS-CUSTOMER-NAME`) กับฝั่งผู้ถูกเรียก
  (`LK-CUSTOMER-NAME`) **ไม่จำเป็นต้องเหมือนกันเลย** สิ่งที่ต้องตรงกันคือ**ลำดับ**ของพารามิเตอร์และ
  **โครงสร้าง/ความกว้าง** ของข้อมูล (`PIC X(20)` ทั้งคู่)
- ผลลัพธ์แสดงให้เห็นว่าเมื่อโปรแกรมย่อยแก้ไข `LK-CUSTOMER-NAME` ค่าที่เปลี่ยนไป**สะท้อนกลับมาที่
  `WS-CUSTOMER-NAME` ในโปรแกรมหลักทันที** หลัง `CALL` จบ — นี่คือพฤติกรรมเริ่มต้นของ COBOL
  (`BY REFERENCE`) ที่จะอธิบายเหตุผลเชิงลึกใน **Part 032**

### ข้อควรระวัง

- ถ้าความกว้างของพารามิเตอร์ไม่ตรงกัน (เช่น ผู้เรียกส่ง `PIC X(20)` แต่โปรแกรมย่อยรับเป็น
  `PIC X(30)`) COBOL ส่วนใหญ่ **จะไม่แจ้ง error ตอนคอมไพล์** เพราะแต่ละไฟล์คอมไพล์แยกกัน ไม่มีการ
  ตรวจสอบข้าม compilation unit แบบเข้มงวดเหมือนภาษาสมัยใหม่ ปัญหาจะไปโผล่ตอน**รันจริง**เท่านั้น
  (จะพิสูจน์ให้เห็นจริงในขั้นตอนที่ 304)
- `LINKAGE SECTION` ต้องมาหลัง `WORKING-STORAGE SECTION` เสมอถ้ามีทั้งคู่ (ลำดับ Section ตายตัว
  ตามมาตรฐาน เหมือนที่ Part 003 สอนเรื่องลำดับ Division)

### แบบฝึกหัดที่ 302.1

**โจทย์**: จงอธิบายว่าทำไมการเปลี่ยนชื่อตัวแปรใน `LINKAGE SECTION` จาก `LK-CUSTOMER-NAME` ให้เป็น
ชื่ออื่น เช่น `LK-ANY-NAME` แล้วแก้ `PROCEDURE DIVISION USING` ให้ตรงกัน จึงไม่ทำให้โปรแกรมเสียหาย
เลยแม้แต่น้อย

**เฉลย**: เพราะการส่งพารามิเตอร์ระหว่างโปรแกรมใน COBOL ยึดตาม**ตำแหน่ง (Positional)** ไม่ใช่ชื่อ
(ต่างจากภาษาสมัยใหม่บางภาษาที่รองรับ named parameter) กล่าวคือพารามิเตอร์ตัวแรกใน `CALL ... USING`
ของผู้เรียก จะจับคู่กับพารามิเตอร์ตัวแรกใน `PROCEDURE DIVISION USING` ของผู้ถูกเรียกเสมอ ไม่ว่าจะ
ตั้งชื่อว่าอะไรก็ตาม ชื่อตัวแปรใน `LINKAGE SECTION` เป็นเพียงชื่อที่ใช้อ้างอิงภายในโปรแกรมย่อยนั้น
เองเท่านั้น ไม่มีผลต่อการจับคู่พารามิเตอร์แต่อย่างใด

---

## ขั้นตอนที่ 303: พารามิเตอร์หลายตัว — ลำดับต้องตรงกันเป๊ะ

### การส่งพารามิเตอร์ 3 ตัวพร้อมกัน

เมื่อต้องส่งข้อมูลมากกว่าหนึ่งตัว เพียงแค่เรียงรายการตัวแปรต่อกันหลัง `USING` และ
`PROCEDURE DIVISION USING` ต้องเรียงตามลำดับเดียวกันเป๊ะ

**ไฟล์ที่ 1 — โปรแกรมหลัก:**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP303MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE                 PIC 9(5)V99 VALUE 350.00.
       01  WS-QUANTITY              PIC 9(3) VALUE 4.
       01  WS-TOTAL                 PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: price=" WS-PRICE " qty=" WS-QUANTITY.
      *> Three parameters, in a fixed ORDER that must match the
      *> subprogram's PROCEDURE DIVISION USING order exactly - the
      *> NAMES do not need to match, only the ORDER and layout do.
           CALL "STEP303SUB" USING WS-PRICE WS-QUANTITY WS-TOTAL.
           DISPLAY "Main: total = " WS-TOTAL.
           STOP RUN.
```

**ไฟล์ที่ 2 — โปรแกรมย่อย:**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP303SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-PRICE                 PIC 9(5)V99.
       01  LK-QUANTITY              PIC 9(3).
       01  LK-TOTAL                 PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-PRICE LK-QUANTITY LK-TOTAL.
       SUB-MAIN-PARA.
      *> The subprogram calculates the total and stores it into
      *> LK-TOTAL - since it is the SAME memory as WS-TOTAL in the
      *> caller (Part 032 explains exactly why), the caller sees
      *> this result immediately after CALL returns.
           COMPUTE LK-TOTAL = LK-PRICE * LK-QUANTITY.
           DISPLAY "  Sub: computed total = " LK-TOTAL.
           GOBACK.
```

```bash
cobc -x step303main.cob step303sub.cob -o step303
./step303
```

**ผลลัพธ์:**

```
Main: price=00350.00 qty=004
  Sub: computed total = 0001400.00
Main: total = 0001400.00
```

### อธิบายจุดสำคัญ

- พารามิเตอร์ตัวที่ 3 (`WS-TOTAL`/`LK-TOTAL`) ในตัวอย่างนี้ทำหน้าที่เป็น **"ช่องส่งผลลัพธ์กลับ"**
  — COBOL `CALL` ไม่มีแนวคิดเรื่อง "ค่าที่ return" แบบฟังก์ชันในภาษาอื่น (เช่น `return x;`) การ
  "ส่งค่ากลับ" จึงทำผ่านพารามิเตอร์ที่ส่งเข้าไปแล้วให้โปรแกรมย่อยเขียนผลลัพธ์ทับลงไปแทน (จะเห็น
  ทางเลือกอื่นด้วย `RETURN-CODE` ในขั้นตอนที่ 308)
- สังเกตว่า `WS-TOTAL` เริ่มต้นด้วยค่า `0` (จาก `VALUE 0`) ก่อนเรียก `CALL` — เป็นนิสัยที่ดีเสมอที่
  จะกำหนดค่าเริ่มต้นให้ชัดเจนก่อนส่งเป็นพารามิเตอร์ แม้จะรู้ว่าโปรแกรมย่อยจะเขียนทับมันแน่นอนก็ตาม
- ลำดับพารามิเตอร์ที่ถูกต้อง: `price, quantity, total` ตรงกันทั้งสองฝั่งเป๊ะ ถ้าสลับลำดับ (เช่น
  ส่ง `quantity, price, total`) โปรแกรมจะยังคอมไพล์ผ่านได้ตามปกติ **แต่คำนวณผิดพลาดทันที**
  เพราะ field ที่ต่างขนาดกัน (`PIC 9(5)V99` กับ `PIC 9(3)`) จะถูกตีความข้อมูลผิดเพี้ยนไปคนละความ
  หมาย

### ข้อควรระวัง

- จำนวนพารามิเตอร์ที่ `CALL ... USING` ส่งไป **ต้องเท่ากับ** จำนวนที่ `PROCEDURE DIVISION USING`
  ของโปรแกรมย่อยรับไว้เป๊ะ (ขั้นตอนที่ 304 จะพิสูจน์ผลลัพธ์ร้ายแรงเมื่อจำนวนไม่ตรงกัน)
- การตั้งชื่อพารามิเตอร์ให้สื่อความหมายในทั้งสองฝั่ง (แม้จะไม่บังคับให้ตรงกัน) ช่วยลดความสับสนของ
  ทีมพัฒนาได้มาก โดยเฉพาะเมื่อพารามิเตอร์มีจำนวนมากกว่า 3–4 ตัว

### แบบฝึกหัดที่ 303.1

**โจทย์**: จงอธิบายว่าจะเกิดอะไรขึ้นถ้าในโปรแกรมหลักเรียก
`CALL "STEP303SUB" USING WS-QUANTITY WS-PRICE WS-TOTAL` (สลับตำแหน่ง `WS-QUANTITY` กับ
`WS-PRICE`) โดยไม่แก้ไขโปรแกรมย่อยเลย

**เฉลย**: โปรแกรมจะยังคอมไพล์ผ่านได้ตามปกติ (COBOL ไม่ตรวจสอบชนิด/ความหมายของพารามิเตอร์ข้าม
compilation unit) แต่ผลลัพธ์จะผิดพลาดทันที เพราะ `LK-PRICE` (ประกาศเป็น `PIC 9(5)V99` กว้าง 7
ไบต์) จะไปรับหน่วยความจำที่จริง ๆ แล้วเป็นของ `WS-QUANTITY` (ประกาศเป็น `PIC 9(3)` กว้างแค่ 3 ไบต์)
ทำให้ตีความข้อมูลผิดเพี้ยนตั้งแต่ต้น (อาจได้ค่าที่ดูเหมือนตัวเลขแต่ไม่มีความหมายที่ถูกต้องเลย หรือ
กรณีเลวร้ายกว่านั้นคือขนาดหน่วยความจำที่ส่งมาไม่พอกับที่ `LK-PRICE` ต้องการอ่าน ซึ่งอาจนำไปสู่ปัญหา
เดียวกับที่จะพิสูจน์ในขั้นตอนที่ 304)

---

## ขั้นตอนที่ 304: กับดักร้ายแรง — จำนวนพารามิเตอร์ไม่ตรงกันทำให้โปรแกรมล่ม

### การทดลอง: เรียกด้วยพารามิเตอร์น้อยกว่าที่โปรแกรมย่อยต้องการ

นี่คือกับดักที่อันตรายที่สุดของ `CALL` ใน COBOL: เพราะแต่ละไฟล์คอมไพล์แยกกัน **COBOL ไม่มีทางตรวจ
สอบตอน compile ได้เลยว่าจำนวนพารามิเตอร์ที่ผู้เรียกส่งมาตรงกับที่โปรแกรมย่อยต้องการหรือไม่** ลอง
พิจารณาโปรแกรมนี้ที่เรียก `STEP304SUB` (ต้องการ 3 พารามิเตอร์) ด้วยแค่ 2 ตัว:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP304BADMAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE                 PIC 9(5)V99 VALUE 350.00.
       01  WS-QUANTITY              PIC 9(3) VALUE 4.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Calling with only 2 args - sub expects 3!".
      *> STEP304SUB below expects THREE parameters (price, qty,
      *> total) but we only pass TWO here. COBOL does NOT check
      *> this across separately compiled programs - it compiles
      *> cleanly and crashes only when the program actually RUNS.
           CALL "STEP304SUB" USING WS-PRICE WS-QUANTITY.
           DISPLAY "Returned safely (this line never prints).".
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP304SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-PRICE                 PIC 9(5)V99.
       01  LK-QUANTITY              PIC 9(3).
       01  LK-TOTAL                 PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-PRICE LK-QUANTITY LK-TOTAL.
       SUB-MAIN-PARA.
           COMPUTE LK-TOTAL = LK-PRICE * LK-QUANTITY.
           DISPLAY "  Sub: computed total = " LK-TOTAL.
           GOBACK.
```

**คอมไพล์ (สำเร็จ ไม่มี error ใด ๆ เลย):**

```bash
cobc -x step304bad_main.cob step304sub.cob -o step304bad
./step304bad
```

**ผลลัพธ์จริงตอนรัน (ยืนยันด้วย GnuCOBOL):**

```
Calling with only 2 args - sub expects 3!

attempt to reference unallocated memory (signal SIGSEGV)
abnormal termination - file contents may be incorrect
```

### วิเคราะห์: เกิดอะไรขึ้นกันแน่

โปรแกรม**คอมไพล์ผ่านทั้งสองไฟล์โดยไม่มี warning หรือ error ใด ๆ เตือนล่วงหน้าเลย** แต่พอรันจริง
โปรแกรมล่มทันทีด้วย **SIGSEGV (Segmentation Fault)** — สาเหตุคือ `LK-TOTAL` ในโปรแกรมย่อยพยายาม
"จับคู่" กับพารามิเตอร์ตัวที่ 3 ที่ผู้เรียก**ไม่ได้ส่งมาให้เลย** เมื่อคำสั่ง `COMPUTE LK-TOTAL = ...`
พยายามเขียนค่าลงในหน่วยความจำที่ `LK-TOTAL` ควรจะชี้ไปถึง แต่ตำแหน่งนั้นไม่มีอยู่จริง (ไม่ได้ถูกจอง
ไว้โดยผู้เรียก) ระบบปฏิบัติการจึงบล็อกการเข้าถึงหน่วยความจำที่ไม่ได้รับอนุญาตทันที ทำให้โปรแกรมทั้ง
กระบวนการ (ไม่ใช่แค่ subprogram) ล่มโดยสมบูรณ์

### แก้ไข: ทำให้จำนวนพารามิเตอร์ตรงกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP304GOODMAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE                 PIC 9(5)V99 VALUE 350.00.
       01  WS-QUANTITY              PIC 9(3) VALUE 4.
       01  WS-TOTAL                 PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Calling with all 3 required args.".
      *> Fixed: the number AND order of arguments now match the
      *> subprogram's PROCEDURE DIVISION USING exactly.
           CALL "STEP304SUB" USING WS-PRICE WS-QUANTITY WS-TOTAL.
           DISPLAY "Returned safely. Total = " WS-TOTAL.
           STOP RUN.
```

**ผลลัพธ์หลังแก้ไข:**

```
Calling with all 3 required args.
  Sub: computed total = 0001400.00
Returned safely. Total = 0001400.00
```

### ข้อควรระวัง

- **นี่คือเหตุผลสำคัญที่สุดที่ทีมพัฒนา COBOL ต้องมี "สัญญา" (Contract) ที่ชัดเจนและมีเอกสารกำกับ**
  ว่าโปรแกรมย่อยแต่ละตัวต้องการพารามิเตอร์อะไรบ้าง กี่ตัว เรียงลำดับอย่างไร เพราะ compiler ไม่ช่วย
  ตรวจสอบให้เหมือนภาษาสมัยใหม่ (COPY Statement ใน Part 033 จะช่วยลดความเสี่ยงนี้ได้บางส่วนด้วยการ
  ใช้โครงสร้างข้อมูลร่วมกันจากไฟล์เดียว)
- ข้อผิดพลาดแบบนี้**อาจไม่ล่มทันทีเสมอไป** ขึ้นอยู่กับว่าหน่วยความจำที่อยู่ถัดจากพารามิเตอร์ตัว
  สุดท้ายที่ส่งมาจริงเป็นพื้นที่อะไร บางครั้งอาจรันผ่านไปได้โดยได้ค่าขยะ (Garbage Value) แทนที่จะ
  ล่มทันที ซึ่ง**อันตรายยิ่งกว่า**เพราะไม่มีการแจ้งเตือนใด ๆ เลย

### แบบฝึกหัดที่ 304.1

**โจทย์**: จงอธิบายว่าทำไม compiler ของ COBOL จึงไม่สามารถตรวจจับข้อผิดพลาดเรื่องจำนวนพารามิเตอร์
ไม่ตรงกันได้ตั้งแต่ตอน compile ทั้งที่ภาษาโปรแกรมสมัยใหม่หลายภาษาทำได้

**เฉลย**: เพราะโปรแกรมหลักและโปรแกรมย่อยใน COBOL **ถูกคอมไพล์แยกกันเป็นคนละไฟล์ ณ เวลาที่ต่างกัน
ได้** (Separate Compilation) — เมื่อ `cobc` คอมไพล์ `step304bad_main.cob` มันไม่รู้จักและไม่ได้อ่าน
เนื้อหาของ `step304sub.cob` เลยแม้แต่น้อย (และในทางกลับกันก็เช่นกัน) การเชื่อมโยง (Linking) ทั้งสอง
ไฟล์เข้าด้วยกันเกิดขึ้น**หลังจาก**ทั้งคู่ถูกคอมไพล์เป็น object code แล้ว ซึ่งในขั้นตอนนั้นข้อมูลเรื่อง
"จำนวนและชนิดของพารามิเตอร์ที่ควรจะเป็น" ได้สูญหายไปมากแล้ว (ต่างจากภาษาสมัยใหม่ที่มักมีระบบ Module/
Header ที่ตรวจสอบ signature ของฟังก์ชันข้ามไฟล์ได้ตั้งแต่ compile time) ภาระในการตรวจสอบความถูกต้อง
นี้จึงตกเป็นของโปรแกรมเมอร์เองทั้งหมด

---

## ขั้นตอนที่ 305: CALL ด้วยชื่อโปรแกรมที่เก็บในตัวแปร

### เมื่อชื่อโปรแกรมไม่ใช่ Literal คงที่

นอกจาก `CALL "ชื่อโปรแกรม"` แบบ literal แล้ว COBOL ยังอนุญาตให้เก็บชื่อโปรแกรมไว้ใน**ตัวแปร**แล้ว
เรียก `CALL` ด้วยชื่อตัวแปรนั้นแทนได้ ทำให้เลือกได้ว่าจะเรียกโปรแกรมไหนตอน**รันจริง**โดยไม่ต้อง
เขียนชื่อตายตัวในโค้ด (ยังคงเป็นการเรียกโปรแกรมที่ link รวมอยู่ในไฟล์ execute เดียวกันเหมือนเดิม
ส่วนการโหลดโปรแกรมภายนอกแบบ dynamic module จริง ๆ จะสอนเต็มรูปแบบใน Part 043)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP305MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> The program name can be stored in a data item instead of
      *> written as a literal. COBOL looks up whatever program name
      *> is CURRENTLY in this field at the moment CALL executes.
       01  WS-PROGRAM-NAME          PIC X(10) VALUE "STEP305SUB".
       01  WS-GREETING              PIC X(20) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: will CALL program named ["
               WS-PROGRAM-NAME "]".
      *> CALL <identifier> instead of CALL "literal" - the program
      *> name is resolved from the CONTENTS of WS-PROGRAM-NAME at
      *> run time. This is still resolved among programs linked
      *> into this same executable; truly loading an external
      *> module at run time is covered later in Part 043.
           CALL WS-PROGRAM-NAME USING WS-GREETING.
           DISPLAY "Main: got back [" WS-GREETING "]".
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP305SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-GREETING              PIC X(20).

       PROCEDURE DIVISION USING LK-GREETING.
       SUB-MAIN-PARA.
           MOVE "HI FROM SUBPROGRAM" TO LK-GREETING.
           GOBACK.
```

**ผลลัพธ์:**

```
Main: will CALL program named [STEP305SUB]
Main: got back [HI FROM SUBPROGRAM  ]
```

### อธิบายจุดสำคัญ

- `CALL WS-PROGRAM-NAME USING WS-GREETING`: ไม่มีเครื่องหมายคำพูดล้อมรอบ `WS-PROGRAM-NAME` เลย
  เพราะเป็นการอ้างอิงชื่อตัวแปร ไม่ใช่ literal — COBOL จะไปดู**ค่าปัจจุบัน**ที่เก็บอยู่ใน
  `WS-PROGRAM-NAME` ขณะรันคำสั่งนี้ แล้วนำค่านั้นไปค้นหาโปรแกรมที่ตรงกัน
- เทคนิคนี้มีประโยชน์เมื่อต้องเลือกโปรแกรมย่อยที่จะเรียกตามเงื่อนไข runtime (เช่น เลือกโปรแกรม
  คำนวณภาษีคนละสูตรตามประเทศที่ระบุในข้อมูล) โดยไม่ต้องเขียน `IF`/`EVALUATE` เพื่อเลือก `CALL`
  literal หลายก้อน
- `WS-PROGRAM-NAME PIC X(10)` ต้องมีความกว้างพอสำหรับชื่อโปรแกรมที่ยาวที่สุดที่จะใช้ พร้อมช่องว่าง
  เติมท้ายถ้าชื่อสั้นกว่า (COBOL จะเทียบชื่อโดยตัดช่องว่างท้ายออกให้อัตโนมัติ)

### ข้อควรระวัง

- ถ้าค่าที่เก็บใน `WS-PROGRAM-NAME` ไม่ตรงกับชื่อโปรแกรมใดที่ link ไว้ในไฟล์ execute เลย โปรแกรม
  จะเกิด runtime error ทันทีที่เรียก `CALL` (ต่างจากการพิมพ์ literal ผิดซึ่งบาง compiler อาจตรวจจับ
  ได้ตั้งแต่ compile time ถ้าเขียนชื่อที่ไม่มีอยู่จริงเลยในไฟล์ที่ link ด้วยกัน)
- เทคนิคนี้ยัง**ไม่ใช่** Dynamic Loading ที่แท้จริง (โหลดไฟล์ `.so`/module จากดิสก์ตอนรัน) เพราะ
  โปรแกรมย่อยทั้งหมดยังคงต้องถูก compile และ link เข้าไปในไฟล์ execute เดียวกันตั้งแต่ต้น ความ
  แตกต่างเรื่อง Static เทียบกับ Dynamic CALL อย่างแท้จริงจะอธิบายเต็มรูปแบบใน Part 043

### แบบฝึกหัดที่ 305.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมข้างต้นจึงต้องกำหนดความกว้างของ `WS-PROGRAM-NAME` เป็น `PIC
X(10)` พอดี (ไม่ใช่กว้างกว่าหรือแคบกว่านี้) เพื่อให้ตรงกับชื่อ `STEP305SUB`

**เฉลย**: `STEP305SUB` มีความยาว 10 ตัวอักษรพอดี การกำหนด `PIC X(10)` ทำให้ค่าที่เก็บตรงกับความยาว
ของชื่อโปรแกรมจริงเป๊ะโดยไม่มีช่องว่างเติมท้าย หากกำหนดกว้างกว่านี้ (เช่น `PIC X(15)`) ค่าที่เก็บจะ
กลายเป็น `"STEP305SUB     "` (มีช่องว่างเติมท้าย 5 ตัว) ซึ่งโดยทั่วไป COBOL จะยังคงจับคู่ชื่อโปรแกรม
ได้ถูกต้องเพราะตัดช่องว่างท้ายก่อนเปรียบเทียบ แต่ถ้ากำหนดแคบกว่าความยาวจริงของชื่อ (เช่น `PIC X(8)`)
ชื่อโปรแกรมจะถูกตัดท้าย กลายเป็น `"STEP305S"` ซึ่งไม่ตรงกับโปรแกรมใดเลย ทำให้เกิด runtime error
ทันที ดังนั้นความกว้างที่ปลอดภัยที่สุดคือกว้างพอหรือกว้างกว่าชื่อโปรแกรมที่ยาวที่สุดที่จะใช้เสมอ

---

## ขั้นตอนที่ 306: GOBACK กับ STOP RUN — ความแตกต่างที่อันตรายถ้าใช้ผิด

### ทดลอง: ใช้ STOP RUN ผิดที่ในโปรแกรมย่อย

นี่คือกับดักคลาสสิกอันดับต้น ๆ ของมือใหม่ที่เขียน Subprogram: การใช้ `STOP RUN` แทน `GOBACK` ใน
โปรแกรมย่อยโดยเข้าใจผิดว่าทั้งสองคำสั่งทำงานเหมือนกัน

**โปรแกรมหลัก (ใช้ร่วมกันทั้งสองกรณี):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP306MAIN.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: before CALL.".
           CALL "STEP306SUB".
           DISPLAY "Main: after CALL (does this line print?).".
           STOP RUN.
```

**โปรแกรมย่อยแบบผิด (ใช้ STOP RUN):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP306SUB.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: running, about to STOP RUN by mistake.".
      *> DANGER: STOP RUN inside a SUBPROGRAM ends the ENTIRE
      *> executable immediately, not just this subprogram. Control
      *> never returns to the caller at all.
           STOP RUN.
```

**ผลลัพธ์ (ยืนยันด้วยการรันจริง):**

```
Main: before CALL.
  Sub: running, about to STOP RUN by mistake.
```

สังเกตว่า **บรรทัด `"Main: after CALL ..."` ไม่ถูกแสดงเลย!** โปรแกรมทั้งหมดจบการทำงานไปตั้งแต่อยู่
ใน subprogram แล้ว

**โปรแกรมย่อยแบบถูกต้อง (ใช้ GOBACK):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP306SUB.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: running, using GOBACK correctly.".
      *> GOBACK returns control to the caller, letting the caller
      *> decide when the whole program actually ends.
           GOBACK.
```

**ผลลัพธ์หลังแก้ไข:**

```
Main: before CALL.
  Sub: running, using GOBACK correctly.
Main: after CALL (does this line print?).
```

### อธิบายจุดสำคัญ

| คำสั่ง | ผลลัพธ์ |
|---|---|
| `GOBACK` | คืนการควบคุมกลับไปยัง**ผู้เรียกโดยตรง** (ถ้าเป็นโปรแกรมหลักเรียกเอง ก็จบการทำงานทั้งหมดเหมือนกัน) |
| `STOP RUN` | จบการทำงานของ**กระบวนการทั้งหมดทันที** ไม่ว่าจะเรียกจากที่ไหนในลำดับการ `CALL` ก็ตาม |

- กฎทองคือ: **`STOP RUN` ควรปรากฏเฉพาะในโปรแกรมหลัก (Top-level Program) เท่านั้น** ส่วน
  `GOBACK` ใช้กับ**ทุกโปรแกรมย่อยที่ถูกเรียกด้วย `CALL`** เสมอ
- ในสถาปัตยกรรมที่ซับซ้อนกว่านี้ (โปรแกรม A เรียก B เรียก C) ถ้า C ใช้ `STOP RUN` ผิดพลาด จะทำให้
  ทั้ง A และ B หยุดทำงานไปด้วยทันที แม้ A และ B จะยังมีงานค้างอยู่ก็ตาม — เป็นบั๊กที่อันตรายมากใน
  ระบบใหญ่เพราะอาจทำให้ข้อมูลไม่สมบูรณ์ (เช่น ไฟล์ที่ควร `CLOSE` ให้เรียบร้อยไม่ได้ถูกปิดอย่างถูกต้อง)

### ข้อควรระวัง

- โค้ดรีวิว (Code Review) ในทีม COBOL มืออาชีพมักตรวจสอบจุดนี้เป็นพิเศษ: **ค้นหา `STOP RUN` ทุกจุด
  ในโปรแกรมย่อย แล้วเปลี่ยนเป็น `GOBACK` ทันที** เพราะเป็นข้อผิดพลาดที่ compiler ตรวจจับไม่ได้เลย
  (ทั้งคู่เป็นคำสั่งที่ถูกต้องตามไวยากรณ์)
- `EXIT PROGRAM` เป็นอีกคำสั่งหนึ่งที่ทำหน้าที่คล้าย `GOBACK` (คืนการควบคุมกลับไปยังผู้เรียก) แต่
  เป็นรูปแบบเก่ากว่าตามมาตรฐาน COBOL-74/85 ส่วน `GOBACK` เป็นรูปแบบที่แนะนำให้ใช้ในปัจจุบัน

### แบบฝึกหัดที่ 306.1

**โจทย์**: ถ้ามีโปรแกรม A เรียก `CALL` โปรแกรม B และโปรแกรม B เรียก `CALL` โปรแกรม C ต่ออีกทอดหนึ่ง
จงอธิบายว่าจะเกิดอะไรขึ้นถ้าโปรแกรม C ใช้ `STOP RUN` แทน `GOBACK`

**เฉลย**: โปรแกรมทั้งกระบวนการ (ทั้ง A, B, และ C) จะจบการทำงานทันทีตั้งแต่อยู่ใน C โดย**ไม่มีการ
คืนการควบคุมกลับไปยัง B หรือ A เลยแม้แต่น้อย** เพราะ `STOP RUN` สั่งจบกระบวนการทั้งหมดในระดับ
operating system ไม่ใช่แค่จบการทำงานของโปรแกรมย่อยปัจจุบัน ผลกระทบคือโค้ดใด ๆ ที่ควรทำงานต่อใน B
หลังจาก `CALL "C"` (เช่น `CLOSE` ไฟล์ที่เปิดค้างไว้ หรือแสดงผลสรุป) จะไม่มีโอกาสได้ทำงานเลย ซึ่งอาจ
ทำให้เกิดปัญหาข้อมูลไม่สมบูรณ์ตามที่อธิบายไว้ข้างต้น

---

## ขั้นตอนที่ 307: ส่งกลุ่มข้อมูล (Group Item) เป็นพารามิเตอร์

### ส่งทั้งโครงสร้างในคำสั่งเดียว

แทนที่จะส่งพารามิเตอร์ทีละฟิลด์ (เหมือนขั้นตอนที่ 303) เราสามารถส่ง **Group Item** ทั้งก้อนเป็น
พารามิเตอร์เดียวได้เลย เหมาะมากเมื่อข้อมูลที่เกี่ยวข้องกันมีหลาย field (คล้าย record ในไฟล์ที่เรียน
มาตั้งแต่ Part 024)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP307MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> A whole GROUP item can be passed as ONE parameter - the
      *> subprogram then describes the SAME layout in its LINKAGE
      *> SECTION to access the individual fields inside it.
       01  WS-CUSTOMER.
           05  WS-CUST-ID           PIC 9(5) VALUE 10001.
           05  WS-CUST-NAME         PIC X(15) VALUE "SOMCHAI JAIDEE".
           05  WS-CUST-BALANCE      PIC 9(7)V99 VALUE 500.00.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Main: before -> " WS-CUST-ID " "
               WS-CUST-NAME " " WS-CUST-BALANCE.
           CALL "STEP307SUB" USING WS-CUSTOMER.
           DISPLAY "Main: after  -> " WS-CUST-ID " "
               WS-CUST-NAME " " WS-CUST-BALANCE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP307SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
      *> This layout must match the CALLER's group item byte for
      *> byte (same field widths, same order) - the FIELD NAMES
      *> here can be different from the caller's, only the memory
      *> layout has to line up.
       01  LK-CUSTOMER.
           05  LK-ID                PIC 9(5).
           05  LK-NAME              PIC X(15).
           05  LK-BALANCE           PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-CUSTOMER.
       SUB-MAIN-PARA.
           DISPLAY "  Sub: applying a deposit of 250.00 to "
               LK-NAME.
           ADD 250.00 TO LK-BALANCE.
           GOBACK.
```

**ผลลัพธ์:**

```
Main: before -> 10001 SOMCHAI JAIDEE  0000500.00
  Sub: applying a deposit of 250.00 to SOMCHAI JAIDEE
Main: after  -> 10001 SOMCHAI JAIDEE  0000750.00
```

### อธิบายจุดสำคัญ

- `CALL "STEP307SUB" USING WS-CUSTOMER`: ส่ง group item ทั้งก้อน (`WS-CUSTOMER`) เป็นพารามิเตอร์
  เดียว ไม่ใช่ 3 พารามิเตอร์แยกกัน — ทำให้ `CALL` อ่านง่ายขึ้นมากเมื่อข้อมูลมีหลาย field
- `01 LK-CUSTOMER.` ฝั่งโปรแกรมย่อยต้องมี**โครงสร้างภายในตรงกันทุกประการ**กับ `01 WS-CUSTOMER.`
  ฝั่งผู้เรียก (ลำดับ field, ความกว้างของแต่ละ field ต้องตรงกัน) แม้ชื่อ 01-level และชื่อ field
  ย่อยจะต่างกันได้ตามที่เรียนในขั้นตอนที่ 302
- โปรแกรมย่อยแก้ไข `LK-BALANCE` (ซึ่งคือหน่วยความจำเดียวกับ `WS-CUST-BALANCE`) แล้วผลลัพธ์สะท้อน
  กลับมาที่โปรแกรมหลักทันที เหมือนหลักการเดิมจากขั้นตอนที่ 302 เพียงแต่ขยายมาเป็นระดับ Group แทน
  Elementary Item ตัวเดียว

### ข้อควรระวัง

- ถ้าโครงสร้างภายในไม่ตรงกัน (เช่น สลับลำดับ field หรือความกว้างไม่เท่ากัน) จะเกิดปัญหาเดียวกับที่
  พิสูจน์ในขั้นตอนที่ 304 — ข้อมูลจะถูกตีความผิดเพี้ยนโดยไม่มี error เตือนตอน compile
  แม้ผลรวมความกว้างทั้งก้อนจะเท่ากันพอดีก็ตาม (เช่น `PIC 9(5)` ตามด้วย `PIC X(15)` มีความกว้างรวม
  เท่ากับ `PIC X(15)` ตามด้วย `PIC 9(5)` แต่ความหมายของแต่ละไบต์ต่างกันโดยสิ้นเชิง)
- Part 033 (COPY Statement) จะแนะนำเทคนิคที่ช่วยแก้ปัญหานี้ได้อย่างมีประสิทธิภาพ: การเก็บโครงสร้าง
  ข้อมูลที่ใช้ร่วมกันไว้ในไฟล์ **Copybook** เดียว แล้วให้ทั้งโปรแกรมหลักและโปรแกรมย่อย `COPY`
  โครงสร้างเดียวกันเข้ามาใช้ แทนที่จะพิมพ์ซ้ำสองที่ (ลดความเสี่ยงเรื่องโครงสร้างไม่ตรงกันได้เกือบ
  สมบูรณ์)

### แบบฝึกหัดที่ 307.1

**โจทย์**: จงอธิบายว่าทำไมการส่ง `WS-CUSTOMER` (group item) เป็นพารามิเตอร์เดียว จึงสะดวกกว่าการ
ส่ง `WS-CUST-ID`, `WS-CUST-NAME`, `WS-CUST-BALANCE` แยกกัน 3 พารามิเตอร์ (แบบเดียวกับขั้นตอนที่ 303)

**เฉลย**: เพราะเมื่อจำนวน field ในโครงสร้างข้อมูลเพิ่มขึ้นเรื่อย ๆ (เช่น ในระบบจริงอาจมีมากกว่า 10
field ต่อ record) การส่งทีละ field แยกกันจะทำให้บรรทัด `CALL ... USING` ยาวมากและเสี่ยงต่อการ
เรียงลำดับผิดพลาดสูงขึ้นตามจำนวน field (ทบทวนความเสี่ยงจากขั้นตอนที่ 303–304) การส่งเป็น group
item ก้อนเดียวช่วยให้ (1) บรรทัด `CALL` สั้นกระชับอ่านง่าย (2) ลดความเสี่ยงเรื่องลำดับพารามิเตอร์
ผิดพลาดเพราะมีแค่พารามิเตอร์เดียวให้ดูแล และ (3) ถ้าต้องเพิ่ม field ใหม่ในอนาคต แก้แค่ในคำนิยาม
group ที่จุดเดียว ไม่ต้องแก้บรรทัด `CALL` เลย (ตราบใดที่โปรแกรมย่อยได้รับการปรับปรุงโครงสร้างให้ตรง
กันด้วย)

---

## ขั้นตอนที่ 308: ส่งค่ากลับด้วย RETURN-CODE

### ทางเลือกที่สองสำหรับการส่งผลลัพธ์กลับ

นอกจากการใช้พารามิเตอร์ (ขั้นตอนที่ 303) COBOL ยังมี **`RETURN-CODE`** ซึ่งเป็น **Special
Register** (ตัวแปรพิเศษที่ COBOL เตรียมไว้ให้อัตโนมัติ ไม่ต้องประกาศใน `DATA DIVISION` เอง) สำหรับ
ส่งค่าตัวเลขเล็ก ๆ กลับจากโปรแกรมย่อยไปยังผู้เรียก เหมาะมากสำหรับผลลัพธ์แบบ "รหัสสถานะ" (เช่น
0 = ผ่าน, 1 = ไม่ผ่าน) มากกว่าข้อมูลจริงจำนวนมาก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP308MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AGE                   PIC 9(3).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 25 TO WS-AGE.
           CALL "STEP308SUB" USING WS-AGE.
      *> RETURN-CODE is a special register (no DATA DIVISION entry
      *> needed) that COBOL automatically makes available after any
      *> CALL - subprograms use it as a simple built-in way to hand
      *> back a small numeric result, alongside or instead of USING
      *> parameters.
           DISPLAY "Age 25  -> RETURN-CODE = " RETURN-CODE.

           MOVE 150 TO WS-AGE.
           CALL "STEP308SUB" USING WS-AGE.
           DISPLAY "Age 150 -> RETURN-CODE = " RETURN-CODE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP308SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AGE                   PIC 9(3).

       PROCEDURE DIVISION USING LK-AGE.
       SUB-MAIN-PARA.
      *> MOVE a value into RETURN-CODE (a special register, just
      *> like using it in the caller - no declaration needed here
      *> either) - 0 means valid, 1 means invalid, by convention
      *> this program chooses itself.
           IF LK-AGE > 0 AND LK-AGE <= 120
               MOVE 0 TO RETURN-CODE
           ELSE
               MOVE 1 TO RETURN-CODE
           END-IF.
           GOBACK.
```

**ผลลัพธ์:**

```
Age 25  -> RETURN-CODE = +000000000
Age 150 -> RETURN-CODE = +000000001
```

### อธิบายจุดสำคัญ

- `RETURN-CODE` ใช้ได้ทั้งสองฝั่ง**โดยไม่ต้องประกาศเลย** ทั้งในโปรแกรมย่อย (`MOVE 0 TO
  RETURN-CODE`) และในโปรแกรมหลักหลัง `CALL` กลับมา (`DISPLAY RETURN-CODE`) เพราะเป็นตัวแปรพิเศษ
  ที่ COBOL runtime จัดสรรให้อัตโนมัติเสมอ
- สังเกตรูปแบบการแสดงผล `+000000000` และ `+000000001` — เพราะ `RETURN-CODE` เป็นฟิลด์ตัวเลขแบบมี
  เครื่องหมาย (signed) ความกว้างมาตรฐานภายในของ GnuCOBOL เมื่อ `DISPLAY` ตรง ๆ จึงเห็นเครื่องหมาย
  `+` และเลข 0 เติมเต็มความกว้างแบบนี้ ต่างจากตัวแปรที่เราประกาศเองด้วย `PIC 9(3)` เป็นต้น
- `RETURN-CODE` มีค่าคงอยู่ **ข้าม `CALL` แต่ละครั้ง** จนกว่าจะมีการ `CALL` ครั้งใหม่มา `MOVE`
  ค่าทับ จึงควรอ่านค่านี้ทันทีหลัง `CALL` กลับมาเสมอ ก่อนที่จะมีคำสั่งอื่นมาเปลี่ยนแปลงมันโดยไม่
  ตั้งใจ

### ข้อควรระวัง

- `RETURN-CODE` เหมาะกับค่าตัวเลขขนาดเล็กเท่านั้น (เช่น รหัสสถานะ 0–99) **ไม่เหมาะกับการส่งข้อมูล
  จำนวนมากหรือข้อความ** ถ้าต้องส่งข้อมูลจริงจำนวนมาก ควรใช้พารามิเตอร์ผ่าน `USING` แทน (ขั้นตอนที่
  303, 307)
- อย่าลืมว่า `RETURN-CODE` เป็น**ตัวแปรร่วมของทั้งโปรแกรม** (Global ในความหมายหนึ่ง) ถ้ามีการ `CALL`
  หลายโปรแกรมติดกัน ค่าที่เห็นหลัง `CALL` แต่ละครั้งคือค่าจาก**การ `CALL` ล่าสุดเท่านั้น** ไม่ใช่
  ค่าสะสมจากทุกการเรียกก่อนหน้า

### แบบฝึกหัดที่ 308.1

**โจทย์**: จงอธิบายว่าทำไมการอ่านค่า `RETURN-CODE` ควรทำ**ทันที**หลัง `CALL` แต่ละครั้ง แทนที่จะ
เก็บไว้อ่านทีหลังหลังจากมีคำสั่ง `CALL` อื่นแทรกเข้ามา

**เฉลย**: เพราะ `RETURN-CODE` เป็น Special Register ตัวเดียวที่ใช้ร่วมกันทั้งโปรแกรม ไม่ได้ผูกกับ
การ `CALL` ครั้งใดครั้งหนึ่งโดยเฉพาะ ทุกครั้งที่มีการ `CALL` โปรแกรมใด ๆ เกิดขึ้น ค่าเดิมใน
`RETURN-CODE` จะถูกเขียนทับด้วยค่าใหม่จากโปรแกรมย่อยล่าสุดที่เพิ่งทำงานเสร็จ หากมีการ `CALL`
โปรแกรมที่สองแทรกเข้ามาก่อนที่จะอ่านค่าจากการ `CALL` โปรแกรมแรก ค่าที่ต้องการจากโปรแกรมแรกจะสูญหาย
ไปอย่างถาวร (ถูกทับไปแล้ว) จึงควรอ่านและเก็บค่าไว้ในตัวแปรของตัวเองทันทีหากยังต้องใช้ค่านั้นต่อไป
ในภายหลัง

---

## ขั้นตอนที่ 309: เรียกโปรแกรมย่อยเดิมซ้ำหลายครั้ง — กับดักเรื่องค่าที่ค้างอยู่

### ทดลอง: ค่าใน WORKING-STORAGE ของ Subprogram ไม่รีเซ็ตทุกครั้งที่ CALL

นี่คือกับดักเชิงตรรกะที่ลึกซึ้งและพบบ่อยมากในโปรแกรมจริง: **`WORKING-STORAGE SECTION` ของโปรแกรม
ย่อยจะถูกกำหนดค่าเริ่มต้นตาม `VALUE` clause เพียงครั้งเดียวเท่านั้น คือครั้งแรกที่ถูก `CALL`**
การ `CALL` ครั้งต่อ ๆ ไปในรันเดียวกันจะ**ใช้หน่วยความจำเดิมที่ค้างค่าจากครั้งก่อนหน้า** ไม่รีเซ็ต
กลับไปที่ `VALUE` ที่ประกาศไว้เลย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP309MAIN.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Calling the SAME subprogram 3 times:".
           CALL "STEP309SUB".
           CALL "STEP309SUB".
           CALL "STEP309SUB".
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP309SUB.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> IMPORTANT: WORKING-STORAGE in a subprogram is set up with
      *> its VALUE clauses only ONCE - the first time it is CALLed.
      *> Every later CALL in the SAME run REUSES the same memory
      *> with whatever it was left holding last time, unlike a
      *> brand new run of a whole program (Part 023 step 221).
       01  WS-CALL-COUNTER          PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       SUB-MAIN-PARA.
           ADD 1 TO WS-CALL-COUNTER.
           DISPLAY "  Sub: this is call number " WS-CALL-COUNTER.
           GOBACK.
```

**ผลลัพธ์ (ยืนยันด้วยการรันจริง):**

```
Calling the SAME subprogram 3 times:
  Sub: this is call number 001
  Sub: this is call number 002
  Sub: this is call number 003
```

### วิเคราะห์ผลลัพธ์

ถ้าคาดหวังว่าแต่ละ `CALL` จะเป็น "การเริ่มต้นใหม่" เหมือนการรันโปรแกรมใหม่ทุกครั้ง (แบบที่ Part 023
ขั้นตอน 221 สอนเรื่อง `WORKING-STORAGE` หายไปทุกครั้งที่โปรแกรมจบ) ผลลัพธ์ควรเป็น `001, 001, 001`
สามครั้ง — **แต่ผลจริงคือ `001, 002, 003`** เพราะการ `CALL` ซ้ำในรันเดียวกันไม่ได้ "เริ่มโปรแกรมใหม่"
แต่เป็นการ**เรียกใช้โปรแกรมที่ยังคงอยู่ในหน่วยความจำเดิม**จากการ `CALL` ครั้งก่อน `WS-CALL-COUNTER`
จึงสะสมค่าต่อเนื่องข้ามการ `CALL` แต่ละครั้งได้ เสมือนเป็นตัวแปร "จำค่าข้ามการเรียก" (State ที่คง
อยู่) โดยไม่ได้ตั้งใจ

### ทางแก้: CANCEL เพื่อบังคับรีเซ็ต

ถ้าต้องการให้โปรแกรมย่อยเริ่มต้นใหม่จริง ๆ ในการ `CALL` ครั้งถัดไป ให้ใช้คำสั่ง **`CANCEL`**
(ทดสอบยืนยันแล้ว):

```cobol
           CALL "STEP309SUB".
           CANCEL "STEP309SUB".
           CALL "STEP309SUB".
```

ผลลัพธ์จากการทดสอบ: `Sub: this is call number 001` ปรากฏ**สองครั้ง** (ไม่ใช่ `001` แล้ว `002`)
เพราะ `CANCEL` สั่งให้ COBOL "ลืม" สถานะทั้งหมดของโปรแกรมย่อยนั้น ทำให้ `CALL` ครั้งถัดไปเริ่มต้น
`WORKING-STORAGE` ใหม่ตาม `VALUE` clause อีกครั้งราวกับเป็นการเรียกครั้งแรก

### อธิบายจุดสำคัญ

- พฤติกรรมนี้**มีประโยชน์**ในบางสถานการณ์ (เช่น ตัวอย่างนี้ที่ใช้เป็นตัวนับจำนวนครั้งที่ถูกเรียก
  ข้ามการ `CALL`) แต่ก็เป็น**กับดักร้ายแรง**ถ้าโปรแกรมเมอร์ไม่รู้ตัวและคาดหวังว่าตัวแปรจะรีเซ็ต
  ทุกครั้ง (เช่น ตัวแปรสะสมยอดรวมที่ควรเริ่มจาก 0 ทุกครั้งที่ประมวลผลลูกค้าคนใหม่)
- `CANCEL "ชื่อโปรแกรม"` สามารถเรียกได้จากโปรแกรมหลัก (หรือโปรแกรมใดก็ตามที่เคย `CALL` โปรแกรมนั้น)
  เพื่อบังคับให้การ `CALL` ครั้งถัดไปเริ่มต้นใหม่หมด
- นี่คือเหตุผลสำคัญที่โปรแกรมย่อยที่ออกแบบมาให้เรียกซ้ำหลายครั้งควร**หลีกเลี่ยงการพึ่งพาค่าที่ค้าง
  อยู่ใน `WORKING-STORAGE` ข้าม `CALL`** เว้นแต่จะตั้งใจออกแบบให้เป็นแบบนั้นจริง ๆ (เช่น ตัวนับ
  หรือ cache ข้อมูลที่ต้องการให้อยู่ข้ามการเรียก)

### ข้อควรระวัง

- การเรียก `CANCEL` มีต้นทุนด้าน performance เล็กน้อย (ต้องจัดสรรหน่วยความจำใหม่ในการ `CALL`
  ครั้งถัดไป) ไม่ควรเรียกพร่ำเพรื่อโดยไม่จำเป็นในระบบที่ต้องประมวลผลเร็วมาก
- อย่าเรียก `CANCEL` โปรแกรมที่**กำลังทำงานอยู่ในขณะนั้น** (เช่น โปรแกรมเรียก `CANCEL` ตัวเองระหว่าง
  ที่ตัวเองยังไม่ `GOBACK`) เพราะเป็นพฤติกรรมที่ไม่มีนิยามชัดเจนตามมาตรฐานและอาจทำให้โปรแกรมทำงาน
  ผิดพลาดได้

### แบบฝึกหัดที่ 309.1

**โจทย์**: จงอธิบายว่าทำไมพฤติกรรม "ค่าค้างข้าม CALL" ในขั้นตอนนี้จึงไม่ขัดแย้งกับสิ่งที่ Part 023
ขั้นตอน 221 สอนไว้ว่า "WORKING-STORAGE หายไปทุกครั้งที่โปรแกรมจบการทำงาน"

**เฉลย**: เพราะทั้งสองกรณีอธิบายสถานการณ์ที่ต่างกัน Part 023 ขั้นตอน 221 พูดถึงการ**รันโปรแกรมใหม่
ทั้งกระบวนการ** (เช่น เรียก `./program` ใหม่จาก command line อีกครั้ง) ซึ่งระบบปฏิบัติการจะจัดสรร
หน่วยความจำใหม่ทั้งหมดให้ process ใหม่เสมอ ในขณะที่ Part นี้พูดถึงการ `CALL` โปรแกรมย่อยซ้ำ ๆ
**ภายในกระบวนการเดียวกัน** (process เดียวกันที่ยังทำงานต่อเนื่องอยู่ตั้งแต่ `STOP RUN` ยังไม่ถูก
เรียก) หน่วยความจำของโปรแกรมย่อยจึงยังคงอยู่และไม่ถูกจัดสรรใหม่ระหว่างการ `CALL` แต่ละครั้ง เว้นแต่
จะสั่ง `CANCEL` ให้จัดสรรใหม่โดยเฉพาะ ทั้งสองกรณีจึงสอดคล้องกันอย่างสมบูรณ์เมื่อเข้าใจขอบเขตของแต่ละ
สถานการณ์ให้ถูกต้อง

---

## ขั้นตอนที่ 310: โปรแกรมรวบยอด — คลังโปรแกรมย่อยสำหรับคำนวณภาษี

### เป้าหมาย

ขั้นตอนสุดท้ายของ Part นี้รวมเทคนิคทั้งหมดที่เรียนมาเข้าด้วยกัน: โปรแกรมหลักเรียกใช้**โปรแกรมย่อย
สองตัวที่ทำงานร่วมกัน**เป็น "คลังโปรแกรมย่อย" (Subprogram Library) ขนาดเล็ก — `STEP310VALIDATE`
ตรวจสอบความถูกต้องของข้อมูลนำเข้า (ส่งผลผ่าน `RETURN-CODE`) และ `STEP310TAX` คำนวณภาษีมูลค่าเพิ่ม
7% (ส่งผลผ่านพารามิเตอร์แบบ Group Item) จำลองการออกแบบที่ใช้งานจริงในองค์กร ที่แยกตรรกะทางธุรกิจ
ออกเป็นหน่วยเล็ก ๆ ที่ทดสอบและดูแลรักษาแยกจากกันได้

**ไฟล์ที่ 1 — โปรแกรมหลัก (`step310main.cob`):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP310MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER.
           05  OR-AMOUNT            PIC 9(7)V99.
           05  OR-TAX               PIC 9(7)V99.
           05  OR-TOTAL             PIC 9(7)V99.
       01  WS-INDEX                 PIC 9(1).
       01  WS-AMOUNT-TABLE.
           05  FILLER               PIC 9(7)V99 VALUE 1000.00.
           05  FILLER               PIC 9(7)V99 VALUE 0.
           05  FILLER               PIC 9(7)V99 VALUE 250.50.
       01  WS-AMOUNT-REDEF REDEFINES WS-AMOUNT-TABLE.
           05  WS-AMOUNT-ENTRY OCCURS 3 TIMES PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> A small "library" of TWO cooperating subprograms:
      *> STEP310VALIDATE checks the input is usable (RETURN-CODE),
      *> and STEP310TAX does the actual calculation. Splitting work
      *> this way, instead of one giant subprogram, is the same
      *> Single Responsibility idea from Part 014 applied across
      *> separately compiled programs.
           PERFORM VARYING WS-INDEX FROM 1 BY 1
                   UNTIL WS-INDEX > 3
               MOVE WS-AMOUNT-ENTRY (WS-INDEX) TO OR-AMOUNT
               DISPLAY "Order " WS-INDEX ": amount = " OR-AMOUNT
               CALL "STEP310VALIDATE" USING OR-AMOUNT
               IF RETURN-CODE = 0
                   CALL "STEP310TAX" USING WS-ORDER
                   DISPLAY "  -> tax=" OR-TAX
                       " total=" OR-TOTAL
               ELSE
                   DISPLAY "  -> REJECTED: amount must be > 0"
               END-IF
           END-PERFORM.
           STOP RUN.
```

**ไฟล์ที่ 2 — ตัวตรวจสอบ (`step310validate.cob`):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP310VALIDATE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AMOUNT                PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT.
       SUB-MAIN-PARA.
           IF LK-AMOUNT > 0
               MOVE 0 TO RETURN-CODE
           ELSE
               MOVE 1 TO RETURN-CODE
           END-IF.
           GOBACK.
```

**ไฟล์ที่ 3 — ตัวคำนวณภาษี (`step310tax.cob`):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP310TAX.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-ORDER.
           05  LK-AMOUNT            PIC 9(7)V99.
           05  LK-TAX               PIC 9(7)V99.
           05  LK-TOTAL             PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-ORDER.
       SUB-MAIN-PARA.
      *> A 7% VAT calculation, kept in ONE place so every part of a
      *> larger system that needs it calls the same subprogram
      *> instead of re-implementing the formula in several places.
           COMPUTE LK-TAX = LK-AMOUNT * 0.07.
           COMPUTE LK-TOTAL = LK-AMOUNT + LK-TAX.
           GOBACK.
```

**คอมไพล์ทั้งสามไฟล์เข้าด้วยกัน:**

```bash
cobc -x step310main.cob step310validate.cob step310tax.cob -o step310
./step310
```

**ผลลัพธ์:**

```
Order 1: amount = 0001000.00
  -> tax=0000070.00 total=0001070.00
Order 2: amount = 0000000.00
  -> REJECTED: amount must be > 0
Order 3: amount = 0000250.50
  -> tax=0000017.53 total=0000268.03
```

### อธิบายภาพรวมของโปรแกรม

โปรแกรมนี้ประกอบด้วย **3 ไฟล์คอมไพล์แยกกัน** ที่ทำงานร่วมกันเป็นระบบเดียว:

1. **STEP310MAIN**: โปรแกรมหลักที่ควบคุมลำดับการทำงานทั้งหมด ใช้ `OCCURS`/`REDEFINES` (ทบทวนจาก
   Part 016, 022) เพื่อวนตรวจสอบยอดสั่งซื้อ 3 รายการ
2. **STEP310VALIDATE**: โปรแกรมย่อยขนาดเล็กมากที่ทำหน้าที่เดียว (ตรวจสอบว่ายอดเงินมากกว่า 0)
   ส่งผลผ่าน `RETURN-CODE` เท่านั้น (เทคนิคจากขั้นตอนที่ 308)
3. **STEP310TAX**: โปรแกรมย่อยที่คำนวณภาษีจริง รับ-ส่งข้อมูลผ่าน Group Item เดียว (เทคนิคจาก
   ขั้นตอนที่ 307)

ยอดสั่งซื้อรายการที่ 2 (`0.00`) ถูกปฏิเสธโดย `STEP310VALIDATE` ก่อนที่จะส่งต่อไปคำนวณภาษีเลยด้วยซ้ำ
— แสดงให้เห็นการทำงานร่วมกันของโปรแกรมย่อยหลายตัวที่แต่ละตัวรับผิดชอบเฉพาะส่วนของตัวเอง (Single
Responsibility) ตรงตามหลักการที่เน้นย้ำมาตลอดหลักสูตร

### ข้อควรระวัง

- ยอดรวมของรายการที่ 3 (`250.50 * 0.07 = 17.535`) แสดงผลเป็น `17.53` (ไม่ปัดขึ้นเป็น `17.54`)
  เพราะ `COMPUTE` ที่ไม่มี `ROUNDED` clause จะ**ตัดทศนิยมส่วนเกินทิ้งเสมอ** (ทบทวนกฎจาก Part 009)
  — ถ้าต้องการปัดเศษที่ถูกต้องตามหลักบัญชี ต้องเพิ่ม `ROUNDED` เข้าไปในทั้งสองบรรทัด `COMPUTE`
- การออกแบบให้มีไฟล์แยกกันหลายไฟล์แบบนี้ต้องอาศัยวินัยในการดูแล**ทุกไฟล์ให้สอดคล้องกัน**เสมอ
  (โดยเฉพาะโครงสร้างพารามิเตอร์) — ทีมพัฒนาจริงมักมีระบบ Build/CI (Part 078 จะสอน CI/CD Pipeline
  สำหรับ COBOL) ที่คอมไพล์และทดสอบทุกไฟล์ที่เกี่ยวข้องพร้อมกันโดยอัตโนมัติทุกครั้งที่มีการแก้ไข

### แบบฝึกหัดที่ 310.1

**โจทย์**: จงเพิ่ม `ROUNDED` เข้าไปในคำสั่ง `COMPUTE LK-TAX` ของ `STEP310TAX` แล้วอธิบายว่าผลลัพธ์
ของรายการที่ 3 จะเปลี่ยนจาก `17.53` เป็นอะไร

**เฉลย**: แก้ไขเป็น:

```cobol
           COMPUTE LK-TAX ROUNDED = LK-AMOUNT * 0.07.
```

ผลลัพธ์จะเปลี่ยนจาก `0000017.53` เป็น **`0000017.54`** เพราะค่าจริงก่อนปัดคือ `17.535` และ
`ROUNDED` ใช้กฎการปัดเศษมาตรฐาน (ปัดขึ้นเมื่อหลักถัดไปคือ 5 ขึ้นไป) ทำให้ได้ `17.54` แทนที่จะตัด
ทิ้งเหลือ `17.53` เหมือนเดิม ซึ่งจะทำให้ `LK-TOTAL` เปลี่ยนจาก `268.03` เป็น `268.04` ตามไปด้วย

---

## สรุปท้ายบท

Part นี้ปูพื้นฐานสำคัญของการเขียนโปรแกรม COBOL แบบแยกโมดูล (Modular Programming) ข้ามไฟล์คอมไพล์
ผ่าน `CALL` Statement สรุปสิ่งที่เรียนรู้:

- ความแตกต่างระหว่าง `PERFORM` (ภายในโปรแกรมเดียว) กับ `CALL` (ข้ามโปรแกรมที่คอมไพล์แยกกัน)
- `LINKAGE SECTION` และ `PROCEDURE DIVISION USING` สำหรับรับพารามิเตอร์ในโปรแกรมย่อย
- การส่งพารามิเตอร์หลายตัวที่ต้อง**เรียงลำดับให้ตรงกัน**ระหว่างผู้เรียกและผู้ถูกเรียก
- กับดักร้ายแรงที่สุด: จำนวนพารามิเตอร์ไม่ตรงกันทำให้เกิด Segmentation Fault ที่ compiler ตรวจจับ
  ไม่ได้เลย
- การ `CALL` ด้วยชื่อโปรแกรมที่เก็บในตัวแปรแทน literal
- ความแตกต่างที่อันตรายระหว่าง `GOBACK` (คืนการควบคุม) กับ `STOP RUN` (จบทั้งกระบวนการ) ในโปรแกรม
  ย่อย
- การส่ง Group Item ทั้งก้อนเป็นพารามิเตอร์เดียว แทนการส่งทีละ field
- `RETURN-CODE` special register สำหรับส่งรหัสผลลัพธ์ตัวเลขเล็ก ๆ กลับจากโปรแกรมย่อย
- กับดักเรื่องค่าที่ค้างอยู่ใน `WORKING-STORAGE` ของโปรแกรมย่อยข้ามการ `CALL` และการใช้ `CANCEL`
  เพื่อบังคับรีเซ็ต
- โปรแกรมรวบยอดที่จำลองคลังโปรแกรมย่อยขนาดเล็กสำหรับคำนวณภาษี ประกอบด้วย 3 ไฟล์ที่ทำงานร่วมกัน

Part ถัดไป (**Part 032**) จะเจาะลึกหัวข้อที่ Part นี้จงใจข้ามไปก่อน: **วิธีการส่งพารามิเตอร์ทั้ง 3
แบบของ COBOL** — `BY REFERENCE` (ค่าเริ่มต้นที่เราใช้มาตลอด Part นี้), `BY CONTENT`, และ
`BY VALUE` — พร้อมอธิบายว่าทำไมการเลือกวิธีส่งพารามิเตอร์ที่ถูกต้องจึงสำคัญต่อความปลอดภัยของข้อมูล
ในระบบขนาดใหญ่

**[← กลับไป Part 030](part-030-file-status-codes.md)** | **[ไปยัง Part 032: การส่งพารามิเตอร์ BY REFERENCE, BY CONTENT, BY VALUE →](part-032-parameter-passing.md)**
