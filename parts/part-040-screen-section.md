# Part 040: Screen Section: สร้าง UI แบบ Text-mode (ขั้นตอนที่ 391–400)

## คำนำของ Part นี้

ตั้งแต่ Part 007 เราใช้ `DISPLAY` และ `ACCEPT` แบบพื้นฐานในการโต้ตอบกับผู้ใช้ — พิมพ์ข้อความออก
ทีละบรรทัด แล้วรับค่าเข้ามาทีละตัวแปร เรียงจากบนลงล่างตามลำดับ ซึ่งใช้ได้ดีสำหรับโปรแกรมง่าย ๆ
แต่ถ้าต้องการสร้าง **หน้าจอกรอกข้อมูลแบบฟอร์ม (form)** ที่มีหลายช่องกรอกพร้อมกัน จัดวางตำแหน่งได้
อิสระ มีสี มีการเน้นข้อความ หรือมีเมนูให้เลือก — COBOL มีฟีเจอร์เฉพาะทางสำหรับงานนี้เรียกว่า
**`SCREEN SECTION`**

`SCREEN SECTION` เป็นส่วนหนึ่งของ DATA DIVISION (คล้ายกับ `REPORT SECTION` ที่เรียนใน Part 038-039
แต่ใช้สำหรับหน้าจอโต้ตอบแบบ Text-mode แทนรายงาน) ให้เรา "ประกาศ" หน้าตาของฟอร์มทั้งหมดไว้ล่วงหน้า
— ตำแหน่งของแต่ละช่อง สี ข้อความป้ายกำกับ (label) — แล้วใช้ `DISPLAY`/`ACCEPT` กับชื่อกลุ่มหน้าจอ
เพื่อแสดงผลหรือรับข้อมูลทั้งฟอร์มในคำสั่งเดียว

### ข้อจำกัดสำคัญที่ต้องรู้ก่อนเริ่ม Part นี้

`SCREEN SECTION` ของ GnuCOBOL ทำงานผ่านไลบรารี **curses** ซึ่ง**ต้องการเทอร์มินัลจริงที่โต้ตอบได้**
เมื่อทดสอบรันโปรแกรมที่ใช้ `SCREEN SECTION` (หรือแม้แต่ `DISPLAY`/`ACCEPT` ธรรมดาที่มี clause
`LINE`/`COLUMN`) ในสภาพแวดล้อมแบบไม่มีเทอร์มินัลจริง (headless / รันผ่านสคริปต์อัตโนมัติแบบที่ใช้
ตรวจสอบเนื้อหาหลักสูตรนี้) โปรแกรมจะยังคง**คอมไพล์และรันจบได้สำเร็จ** แต่สิ่งที่พิมพ์ออกมาจะเป็น
**รหัสควบคุมเทอร์มินัล (terminal control codes / ANSI escape sequences)** ที่อ่านไม่ออกเป็นข้อความ
ปกติ แทนที่จะเป็นหน้าจอฟอร์มที่จัดวางสวยงาม

ทุกตัวอย่างใน Part นี้ **ทดสอบคอมไพล์จริงด้วย `cobc -x` แล้วว่าผ่านสำเร็จ 100%** (ยืนยันด้วย exit
code และไม่มี compile error) แต่ **ผลลัพธ์แสดงผลจริงบนหน้าจอไม่สามารถแคปเจอร์เป็นข้อความมาแสดงใน
เอกสารนี้ได้อย่างถูกต้อง** เนื่องจากข้อจำกัดข้างต้น — แทนที่จะกุผลลัพธ์ปลอมขึ้นมา ทุกขั้นตอนจะระบุ
ชัดเจนว่า **สิ่งใดคือสิ่งที่ทดสอบยืนยันแล้วจริง (การคอมไพล์)** และ **สิ่งใดคือพฤติกรรมที่คาดหวังตาม
เอกสาร COBOL มาตรฐาน (ควรลองรันเองบนเทอร์มินัลจริงของคุณ)** พร้อม **ภาพจำลอง (mockup)** แบบ
ASCII ประกอบเพื่อให้เห็นภาพว่าฟอร์มควรมีหน้าตาอย่างไร — โปรดสังเกตว่าภาพจำลองเหล่านี้ **ไม่ใช่**
ผลลัพธ์ที่แคปเจอร์จากการรันจริง

**คำแนะนำ**: หลังเรียน Part นี้ ขอให้คุณเปิด terminal จริง (ไม่ใช่ผ่านสคริปต์อัตโนมัติ) แล้วคอมไพล์
และรันทุกตัวอย่างด้วยตัวเอง เพื่อเห็นผลลัพธ์ภาพหน้าจอจริงตามที่ออกแบบไว้

---

## ขั้นตอนที่ 391: SCREEN SECTION พื้นฐาน — BLANK SCREEN และข้อความคงที่

### แนวคิด

`SCREEN SECTION` ประกาศอยู่ใน DATA DIVISION ต่อจาก `WORKING-STORAGE SECTION` (และหลัง
`REPORT SECTION` ถ้ามี) โครงสร้างพื้นฐานที่สุดประกอบด้วยกลุ่มรายการระดับ `01` ที่มีฟิลด์ย่อยระบุ
ตำแหน่งด้วย **`LINE`**/**`COLUMN`** และข้อความคงที่ด้วย **`VALUE`** — คล้ายกับ `REPORT SECTION`
ที่เรียนใน Part 038 มาก แต่ใช้กับหน้าจอโต้ตอบแทนไฟล์รายงาน

**`BLANK SCREEN`** เป็น clause พิเศษที่ล้างหน้าจอทั้งหมดให้ว่างก่อนวาดเนื้อหาใหม่ นิยมใส่ไว้ที่ฟิลด์
แรกของกลุ่มหน้าจอเสมอ เพื่อป้องกันไม่ให้เนื้อหาเก่าจากหน้าจอก่อนหน้าค้างอยู่

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-BASIC-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DUMMY   PIC X.

       SCREEN SECTION.
       01  SCR-WELCOME.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1  VALUE "========================".
           05  LINE 2 COLUMN 1  VALUE "  CUSTOMER MANAGEMENT   ".
           05  LINE 3 COLUMN 1  VALUE "========================".
           05  LINE 5 COLUMN 1  VALUE "WELCOME TO THE SYSTEM.".

       PROCEDURE DIVISION.
           DISPLAY SCR-WELCOME.
           STOP RUN.
```

### อธิบายโค้ด

- `SCREEN SECTION.` เริ่มต้นส่วนใหม่ใน DATA DIVISION เช่นเดียวกับ `FILE SECTION`,
  `WORKING-STORAGE SECTION`, `REPORT SECTION` ที่เรียนมาก่อนหน้า
- `01 SCR-WELCOME.` — ชื่อกลุ่มหน้าจอ (screen group) ระดับ 01 ตั้งชื่อตามธรรมเนียม `SCR-` เพื่อให้
  แยกจากตัวแปรและ report group ได้ง่ายเมื่ออ่านโค้ด
- `05 BLANK SCREEN.` — ล้างหน้าจอทั้งหมดก่อนวาดเนื้อหา
- `05 LINE n COLUMN m VALUE "...".` — วางข้อความคงที่ที่ตำแหน่งบรรทัด `n` คอลัมน์ `m` พอดี
  ไม่มีตัวแปรเกี่ยวข้องเลย เป็นเพียงข้อความตกแต่งหน้าจอ (label/decoration)
- `DISPLAY SCR-WELCOME.` — สั่งวาดกลุ่มหน้าจอทั้งหมดออกทางเทอร์มินัลในคำสั่งเดียว

### ผลการทดสอบคอมไพล์จริง

ทดสอบด้วยคำสั่ง:
```
cobc -x -o screen-basic-demo screen-basic-demo.cob
```
**ผลลัพธ์: คอมไพล์สำเร็จ ไม่มี error หรือ warning ใด ๆ** (exit code 0) เมื่อรันโปรแกรมนี้ในสภาพแวดล้อม
ทดสอบแบบ headless พบว่าโปรแกรมจบการทำงานตามปกติ (exit code 0) แต่สิ่งที่พิมพ์ออกมาทาง terminal
เป็นลำดับรหัสควบคุม (escape sequences) ของไลบรารี curses ไม่ใช่ข้อความอ่านง่าย — ตรงตามข้อจำกัด
ที่อธิบายไว้ในคำนำของ Part นี้

### ภาพจำลองหน้าจอที่คาดว่าจะเห็นเมื่อรันบนเทอร์มินัลจริง (mockup — ไม่ใช่ผลลัพธ์ที่แคปเจอร์จริง)

```
========================
  CUSTOMER MANAGEMENT
========================

WELCOME TO THE SYSTEM.
```

### ข้อควรระวัง

- **`SCREEN SECTION` ต้องมาหลัง `WORKING-STORAGE SECTION`** เสมอในลำดับของ DATA DIVISION
  (และหลัง `REPORT SECTION` ถ้ามีทั้งสองอย่างในโปรแกรมเดียวกัน)
- `LINE`/`COLUMN` เริ่มนับจาก 1 เสมอ เหมือนที่เรียนใน `REPORT SECTION`
- หากลืม `BLANK SCREEN` เนื้อหาจากการ `DISPLAY` ครั้งก่อนหน้า (ไม่ว่าจะเป็นจาก screen group อื่น
  หรือ `DISPLAY` ธรรมดา) อาจยังค้างอยู่บนหน้าจอปนกับเนื้อหาใหม่

### แบบฝึกหัดที่ 391.1

**โจทย์**: จงเขียน screen group ชื่อ `SCR-GOODBYE` ที่ล้างหน้าจอแล้วแสดงข้อความ "THANK YOU FOR
USING THIS SYSTEM." ที่บรรทัด 10 คอลัมน์ 5

**เฉลย**:
```cobol
       01  SCR-GOODBYE.
           05  BLANK SCREEN.
           05  LINE 10 COLUMN 5
               VALUE "THANK YOU FOR USING THIS SYSTEM.".
```

---

## ขั้นตอนที่ 392: ช่องกรอกข้อมูล — PIC และ USING Clause

### แนวคิด

ข้อความคงที่อย่างเดียวยังไม่ใช่ "ฟอร์ม" ที่แท้จริง สิ่งที่ทำให้ `SCREEN SECTION` มีประโยชน์คือ
**ช่องกรอกข้อมูล (input field)** ที่ผูกกับตัวแปรใน WORKING-STORAGE ผ่าน clause **`USING`**
พร้อม **`PIC`** กำหนดรูปแบบและความกว้างของช่อง ต่างจาก `SOURCE` ใน `REPORT SECTION` (Part 038)
ที่ดึงค่าไปแสดงอย่างเดียว — `USING` ทำงานได้ **สองทาง**: ทั้งแสดงค่าปัจจุบันของตัวแปรตอน `DISPLAY`
และรับค่าใหม่จากผู้ใช้กลับเข้าตัวแปรตอน `ACCEPT`

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-INPUT-FIELD-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-NAME  PIC X(20).
       01  WS-CUSTOMER-AGE   PIC 9(3).

       SCREEN SECTION.
       01  SCR-CUSTOMER-FORM.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1  VALUE "CUSTOMER ENTRY FORM".
           05  LINE 3 COLUMN 1  VALUE "NAME:".
           05  LINE 3 COLUMN 10 PIC X(20) USING WS-CUSTOMER-NAME.
           05  LINE 4 COLUMN 1  VALUE "AGE :".
           05  LINE 4 COLUMN 10 PIC 9(3)  USING WS-CUSTOMER-AGE.

       PROCEDURE DIVISION.
           MOVE "SOMCHAI"  TO WS-CUSTOMER-NAME.
           MOVE 30         TO WS-CUSTOMER-AGE.

      *> DISPLAY shows the current values of the USING fields.
           DISPLAY SCR-CUSTOMER-FORM.

      *> ACCEPT lets the user type over the fields, updating the
      *> underlying WORKING-STORAGE variables in place.
           ACCEPT SCR-CUSTOMER-FORM.

           DISPLAY "VALUE READ BACK: " WS-CUSTOMER-NAME.
           STOP RUN.
```

### อธิบายโค้ด

- `05 LINE 3 COLUMN 10 PIC X(20) USING WS-CUSTOMER-NAME.` — สร้างช่องกรอกข้อความกว้าง 20
  ตัวอักษร ที่ตำแหน่งบรรทัด 3 คอลัมน์ 10 ผูกกับตัวแปร `WS-CUSTOMER-NAME`
- `DISPLAY SCR-CUSTOMER-FORM.` — วาดฟอร์มทั้งหมด พร้อมแสดงค่าปัจจุบันของ `WS-CUSTOMER-NAME`
  ("SOMCHAI") และ `WS-CUSTOMER-AGE` (030) ที่เคย `MOVE` ไว้ก่อนหน้า ในช่องกรอกที่เกี่ยวข้อง
- `ACCEPT SCR-CUSTOMER-FORM.` — ให้ผู้ใช้พิมพ์แก้ไขค่าในทุกช่องกรอกของฟอร์มนี้ (เคอร์เซอร์จะ
  เคลื่อนจากช่องแรกไปช่องถัดไปตามลำดับที่ประกาศไว้) เมื่อผู้ใช้กดยืนยัน (มักเป็น Enter ที่ช่องสุดท้าย)
  ค่าที่พิมพ์จะถูกเก็บกลับเข้า `WS-CUSTOMER-NAME` และ `WS-CUSTOMER-AGE` โดยอัตโนมัติ

### ผลการทดสอบคอมไพล์จริง

`cobc -x -o screen-input-field-demo screen-input-field-demo.cob` — **คอมไพล์สำเร็จ** ไม่มี
error ทดสอบรันแบบ headless (ป้อนข้อมูลผ่าน pipe) พบว่าโปรแกรมจบด้วย exit code 0 เช่นกัน แต่
เอาต์พุตเป็นรหัสควบคุมเทอร์มินัลตามที่อธิบายในคำนำ ไม่สามารถยืนยันค่าที่ "อ่านกลับ" ได้ในสภาพแวดล้อม
นี้ — ควรทดสอบ `ACCEPT` แบบโต้ตอบจริงบนเทอร์มินัลของคุณเอง

### ภาพจำลองหน้าจอ (mockup)

```
CUSTOMER ENTRY FORM

NAME: SOMCHAI
AGE : 030
```

(เคอร์เซอร์จะกะพริบอยู่ที่ช่อง NAME ก่อน รอให้ผู้ใช้พิมพ์ทับหรือกด Tab/Enter ผ่านไปช่อง AGE)

### ข้อควรระวัง

- **`USING` แตกต่างจาก `SOURCE`** ที่เรียนใน Report Writer (Part 038) ตรงที่ `USING` ทำงานได้
  ทั้งขาเข้าและขาออก (`DISPLAY` แสดงค่า, `ACCEPT` รับค่ากลับ) ในขณะที่ `SOURCE` ใช้ได้เฉพาะขาออก
  (แสดงค่าอย่างเดียว) เท่านั้น
- ต้อง `MOVE` ค่าเริ่มต้นเข้าตัวแปรก่อน `DISPLAY` หากต้องการให้ฟอร์มแสดงค่าเริ่มต้นที่ไม่ใช่ค่าว่าง/ศูนย์
- ความกว้างของ `PIC` ในช่องกรอกจำกัดจำนวนตัวอักษรที่ผู้ใช้พิมพ์ได้สูงสุด (`PIC X(20)` รับได้ไม่เกิน
  20 ตัวอักษร) เช่นเดียวกับกฎ MOVE ที่เรียนใน Part 008

### แบบฝึกหัดที่ 392.1

**โจทย์**: จงเพิ่มช่องกรอกใหม่ `WS-CUSTOMER-EMAIL PIC X(30)` เข้าไปในฟอร์มข้างต้น ที่บรรทัด 5

**เฉลย**:
```cobol
       01  WS-CUSTOMER-EMAIL PIC X(30).
       ...
           05  LINE 5 COLUMN 1  VALUE "EMAIL:".
           05  LINE 5 COLUMN 10 PIC X(30) USING WS-CUSTOMER-EMAIL.
```

---

## ขั้นตอนที่ 393: สีสันและการเน้นข้อความ — FOREGROUND-COLOR, BACKGROUND-COLOR, HIGHLIGHT

### แนวคิด

`SCREEN SECTION` รองรับ clause ควบคุมสีและการเน้นข้อความหลายตัว ที่สำคัญที่สุดคือ:

- **`FOREGROUND-COLOR n`** / **`BACKGROUND-COLOR n`** — กำหนดสีตัวอักษร/สีพื้นหลัง โดย `n`
  เป็นตัวเลข 0-7 ตามรหัสสีมาตรฐานของเทอร์มินัล
- **`HIGHLIGHT`** — ทำให้ข้อความสว่างเด่นขึ้น (bold)
- **`REVERSE-VIDEO`** — สลับสีตัวอักษรกับพื้นหลัง (ใช้เน้นข้อความสำคัญ เช่น ข้อความ error)

| รหัสสี (n) | สี |
|---|---|
| 0 | ดำ (Black) |
| 1 | น้ำเงิน (Blue) |
| 2 | เขียว (Green) |
| 3 | ฟ้า/เขียวอมฟ้า (Cyan) |
| 4 | แดง (Red) |
| 5 | ม่วง (Magenta) |
| 6 | เหลือง (Yellow) |
| 7 | ขาว (White) |

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-COLOR-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME   PIC X(20).

       SCREEN SECTION.
       01  SCR-COLOR-FORM.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1
               VALUE "REGISTRATION FORM"
               FOREGROUND-COLOR 3 HIGHLIGHT.
           05  LINE 3 COLUMN 1  VALUE "NAME:".
           05  LINE 3 COLUMN 10 PIC X(20) USING WS-NAME
               FOREGROUND-COLOR 2.
           05  LINE 5 COLUMN 1
               VALUE "** ERROR: NAME CANNOT BE BLANK **"
               FOREGROUND-COLOR 7 BACKGROUND-COLOR 4 REVERSE-VIDEO.

       PROCEDURE DIVISION.
           DISPLAY SCR-COLOR-FORM.
           STOP RUN.
```

### อธิบายโค้ด

- `VALUE "REGISTRATION FORM" FOREGROUND-COLOR 3 HIGHLIGHT.` — หัวข้อฟอร์มแสดงด้วยสีฟ้าอมเขียว
  (cyan) แบบตัวหนา
- `PIC X(20) USING WS-NAME FOREGROUND-COLOR 2.` — ช่องกรอกชื่อแสดงตัวอักษรสีเขียว
- บรรทัดข้อความ error ใช้ `REVERSE-VIDEO` ร่วมกับสีขาว-แดง ทำให้ข้อความเด่นชัดมาก เหมาะกับ
  การแจ้งเตือนข้อผิดพลาดที่ผู้ใช้ต้องสังเกตเห็นทันที

### ผลการทดสอบคอมไพล์จริง

`cobc -x -o screen-color-demo screen-color-demo.cob` — **คอมไพล์สำเร็จ** ไม่มี error การแสดงสี
จริงต้องอาศัยเทอร์มินัลที่รองรับสี (terminal ทั่วไปในปัจจุบันรองรับหมด) และไม่สามารถแคปเจอร์เป็นภาพ
สีมาแสดงในเอกสาร Markdown นี้ได้ จึงขอให้ผู้เรียนรันด้วยตนเองเพื่อเห็นสีจริง

### ภาพจำลองหน้าจอ (mockup — สีแสดงเป็นคำอธิบายในวงเล็บแทนสีจริง)

```
REGISTRATION FORM              <- สีฟ้าอมเขียว ตัวหนา (cyan, highlight)

NAME: _____________             <- ช่องกรอกตัวอักษรสีเขียว

** ERROR: NAME CANNOT BE BLANK **   <- พื้นแดง ตัวอักษรขาว (reverse video)
```

### ข้อควรระวัง

- รหัสสีเป็นตัวเลข 0-7 มาตรฐาน แต่**สีจริงที่ปรากฏขึ้นอยู่กับการตั้งค่าของเทอร์มินัลผู้ใช้ด้วย**
  (บาง terminal theme อาจปรับสีให้ต่างจากที่คาดไว้เล็กน้อย)
- การใช้สีมากเกินไปในฟอร์มเดียวอาจทำให้ผู้ใช้ตาลายและมองข้ามสิ่งสำคัญจริง ๆ ควรใช้สีเน้นเฉพาะจุดที่
  ต้องการดึงความสนใจสูงสุดเท่านั้น (เช่น ข้อความ error)
- ไม่ใช่ทุกเทอร์มินัล/ทุก terminal emulator จะรองรับ `REVERSE-VIDEO` หรือสีได้ตรงกันเป๊ะ 100%
  ควรทดสอบบนเทอร์มินัลเป้าหมายจริงก่อนใช้งานจริง

### แบบฝึกหัดที่ 393.1

**โจทย์**: จงเขียนบรรทัดข้อความแจ้งเตือนสำเร็จ "SAVED SUCCESSFULLY!" ด้วยสีเขียว (2) แบบ HIGHLIGHT
ที่บรรทัด 10

**เฉลย**:
```cobol
       05  LINE 10 COLUMN 1
           VALUE "SAVED SUCCESSFULLY!"
           FOREGROUND-COLOR 2 HIGHLIGHT.
```

---

## ขั้นตอนที่ 394: AUTO, REQUIRED และ SECURE — ควบคุมพฤติกรรมช่องกรอก

### แนวคิด

นอกจากสีสัน `SCREEN SECTION` ยังมี clause ควบคุม**พฤติกรรม**ของช่องกรอกที่สำคัญมากในการสร้าง
ฟอร์มมืออาชีพ:

- **`AUTO`** — เมื่อผู้ใช้พิมพ์ครบความกว้างของช่อง (เต็ม `PIC`) เคอร์เซอร์จะข้ามไปช่องถัดไปโดยอัตโนมัติ
  ทันที ไม่ต้องกด Tab/Enter เอง
- **`REQUIRED`** — บังคับให้ผู้ใช้ต้องพิมพ์ข้อมูลในช่องนี้ ห้ามเว้นว่าง (ถ้าปล่อยว่างแล้วพยายามข้ามไป
  ช่องอื่น ระบบจะไม่ยอมให้ผ่าน)
- **`SECURE`** — ซ่อนตัวอักษรที่พิมพ์ (แสดงเป็นเครื่องหมาย เช่น `*` แทน) เหมาะสำหรับช่องรหัสผ่าน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-BEHAVIOR-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-USER-ID   PIC X(8).
       01  WS-PASSWORD  PIC X(10).
       01  WS-PIN-CODE  PIC 9(4).

       SCREEN SECTION.
       01  SCR-LOGIN-FORM.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1  VALUE "LOGIN".
           05  LINE 3 COLUMN 1  VALUE "USER ID :".
      *> AUTO: cursor jumps to the next field once 8 characters
      *> are typed, without waiting for Enter.
           05  LINE 3 COLUMN 12 PIC X(8) USING WS-USER-ID
               REQUIRED AUTO.
           05  LINE 4 COLUMN 1  VALUE "PASSWORD:".
      *> SECURE: typed characters are masked on screen.
           05  LINE 4 COLUMN 12 PIC X(10) USING WS-PASSWORD
               REQUIRED SECURE.
           05  LINE 5 COLUMN 1  VALUE "PIN CODE:".
           05  LINE 5 COLUMN 12 PIC 9(4)  USING WS-PIN-CODE
               REQUIRED SECURE AUTO.

       PROCEDURE DIVISION.
           DISPLAY SCR-LOGIN-FORM.
           ACCEPT SCR-LOGIN-FORM.
           STOP RUN.
```

### อธิบายโค้ด

- `REQUIRED AUTO` บนช่อง USER ID — ผู้ใช้ต้องพิมพ์อย่างน้อย 1 ตัวอักษร (`REQUIRED`) และเมื่อพิมพ์
  ครบ 8 ตัวอักษรพอดี เคอร์เซอร์จะกระโดดไปช่อง PASSWORD ทันที (`AUTO`) โดยไม่ต้องกด Enter
- `REQUIRED SECURE` บนช่อง PASSWORD — บังคับกรอก และซ่อนตัวอักษรที่พิมพ์ไม่ให้ใครแอบมองเห็น
  บนหน้าจอได้
- `REQUIRED SECURE AUTO` บนช่อง PIN CODE — รวมทั้งสามพฤติกรรมเข้าด้วยกัน: บังคับกรอก, ซ่อน
  ตัวเลขที่พิมพ์, และกระโดดออกจากช่องอัตโนมัติเมื่อครบ 4 หลัก

### ผลการทดสอบคอมไพล์จริง

`cobc -x -o screen-behavior-demo screen-behavior-demo.cob` — **คอมไพล์สำเร็จ** ทั้ง `AUTO`,
`REQUIRED`, `SECURE` เป็น clause มาตรฐานที่ GnuCOBOL รองรับโดยไม่มี error ใด ๆ พฤติกรรมจริง
(การกระโดดอัตโนมัติ, การบังคับกรอก, การซ่อนตัวอักษร) ต้องทดสอบบนเทอร์มินัลจริงเท่านั้น เนื่องจากเป็น
พฤติกรรมเชิงโต้ตอบที่ไม่มีทางแคปเจอร์เป็นข้อความนิ่งได้

### ภาพจำลองหน้าจอ (mockup)

```
LOGIN

USER ID : ________
PASSWORD: **********
PIN CODE: ****
```

### ข้อควรระวัง

- **`SECURE` ซ่อนเฉพาะบนหน้าจอเท่านั้น** ค่าจริงที่พิมพ์ยังคงถูกเก็บลงตัวแปร (`WS-PASSWORD`)
  ตามปกติทุกตัวอักษร ไม่ได้เข้ารหัสหรือป้องกันการอ่านค่าจากหน่วยความจำแต่อย่างใด — หากต้องการความ
  ปลอดภัยจริงจัง (เช่น ระบบ login งานจริง) ต้องมีการเข้ารหัส/hash รหัสผ่านแยกต่างหาก ซึ่งเป็นหัวข้อ
  นอกเหนือขอบเขตของ `SCREEN SECTION`
- `AUTO` สะดวกแต่ก็อาจทำให้ผู้ใช้พลาดได้ถ้าพิมพ์ผิดแล้วเคอร์เซอร์กระโดดไปช่องถัดไปแล้วก่อนจะทันสังเกต
  ควรพิจารณาความยาวช่องให้เหมาะสมกับข้อมูลจริงเสมอ
- `REQUIRED` ไม่ได้ตรวจสอบรูปแบบข้อมูล (เช่น ตัวเลขหรือตัวอักษร) เพียงแค่ตรวจว่าไม่ว่างเปล่าเท่านั้น
  การตรวจสอบรูปแบบเชิงลึกยังต้องใช้ Class Condition (`IS NUMERIC` เป็นต้น) ที่จะเรียนใน **Part 042**

### แบบฝึกหัดที่ 394.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `SECURE` กับการเข้ารหัสรหัสผ่าน (password encryption/hashing)
จริง และอธิบายว่าทำไมทั้งสองอย่างจึงจำเป็นสำหรับระบบ login งานจริง

**เฉลยแนวทาง**: `SECURE` เป็นเพียง clause ที่ควบคุม**การแสดงผลบนหน้าจอ**เท่านั้น (แสดง `*` แทน
ตัวอักษรจริงเพื่อไม่ให้คนที่มองข้ามไหล่เห็นรหัสผ่าน) แต่ค่าที่เก็บในตัวแปรและส่งต่อไปประมวลผลยังคงเป็น
ข้อความรหัสผ่านจริงแบบไม่เข้ารหัส (plain text) ในขณะที่การเข้ารหัส/hashing คือกระบวนการแปลงรหัสผ่าน
ให้อยู่ในรูปแบบที่ไม่สามารถย้อนกลับเป็นข้อความเดิมได้ง่าย ก่อนจะบันทึกลงฐานข้อมูลหรือส่งผ่านเครือข่าย
ระบบ login งานจริงจึงต้องใช้ทั้งสองอย่างร่วมกัน: `SECURE` เพื่อป้องกันการมองเห็นด้วยตาเปล่าขณะพิมพ์
และ hashing เพื่อป้องกันไม่ให้รหัสผ่านที่แท้จริงรั่วไหลแม้ฐานข้อมูลจะถูกขโมยไปก็ตาม

---

## ขั้นตอนที่ 395: PROMPT และ UNDERLINE — ตัวช่วยด้านการมองเห็นช่องกรอก

### แนวคิด

- **`PROMPT`** — แสดงตัวอักษรเติมเต็มช่องว่าง (ค่าเริ่มต้นคือ underscore `_`) เพื่อบอกผู้ใช้ว่าช่องนี้
  ยาวแค่ไหนและยังไม่ได้กรอกตรงไหนบ้าง ทำให้มองเห็นขอบเขตของช่องกรอกชัดเจนกว่าช่องว่างเปล่า ๆ
- **`UNDERLINE`** — ขีดเส้นใต้ข้อความ/ช่องกรอก ช่วยให้แยกช่องกรอกออกจากข้อความป้ายกำกับ (label)
  ได้ชัดเจนโดยไม่ต้องพึ่งสีอย่างเดียว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-PROMPT-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRODUCT-CODE  PIC X(6).
       01  WS-PRODUCT-NAME  PIC X(15).

       SCREEN SECTION.
       01  SCR-PRODUCT-FORM.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1  VALUE "PRODUCT ENTRY" UNDERLINE.
           05  LINE 3 COLUMN 1  VALUE "CODE:".
      *> PROMPT fills unfilled positions with underscores so the
      *> user can see exactly how many characters fit.
           05  LINE 3 COLUMN 10 PIC X(6) USING WS-PRODUCT-CODE
               PROMPT.
           05  LINE 4 COLUMN 1  VALUE "NAME:".
           05  LINE 4 COLUMN 10 PIC X(15) USING WS-PRODUCT-NAME
               PROMPT UNDERLINE.

       PROCEDURE DIVISION.
           DISPLAY SCR-PRODUCT-FORM.
           STOP RUN.
```

### อธิบายโค้ด

- `PIC X(6) USING WS-PRODUCT-CODE PROMPT.` — ช่องกรอกรหัสสินค้าจะแสดงเป็น `______` (6
  ขีดเส้นใต้) เพื่อบอกผู้ใช้ทันทีว่าช่องนี้กรอกได้สูงสุด 6 ตัวอักษร ก่อนที่ผู้ใช้จะพิมพ์อะไรเลย
- `PROMPT UNDERLINE` บนช่องชื่อสินค้า — รวมทั้งตัวเติมเต็ม (prompt character) และเส้นใต้เข้าด้วยกัน
  ทำให้มองเห็นขอบเขตของช่องกรอกชัดเจนที่สุด

### ผลการทดสอบคอมไพล์จริง

`cobc -x -o screen-prompt-demo screen-prompt-demo.cob` — **คอมไพล์สำเร็จ** `PROMPT` และ
`UNDERLINE` เป็น clause มาตรฐานที่รองรับโดยไม่มี error

### ภาพจำลองหน้าจอ (mockup)

```
PRODUCT ENTRY
-------------
CODE: ______
NAME: _______________
```

### ข้อควรระวัง

- ตัวอักษรเติมเต็มเริ่มต้นของ `PROMPT` คือ underscore (`_`) แต่สามารถระบุตัวอักษรอื่นได้ผ่าน
  `PROMPT CHARACTER IS "x"` ในบางคอมไพเลอร์ ควรตรวจสอบเอกสารเฉพาะของ GnuCOBOL เวอร์ชันที่ใช้จริง
  ก่อนพึ่งพา syntax ขั้นสูงนี้
- `UNDERLINE` บนข้อความ label (ไม่ใช่ช่องกรอก) เป็นเพียงการตกแต่งภาพเท่านั้น ไม่มีผลต่อการทำงาน
  ของฟอร์ม

### แบบฝึกหัดที่ 395.1

**โจทย์**: จงอธิบายว่าทำไม `PROMPT` จึงช่วยลดข้อผิดพลาดของผู้ใช้เมื่อกรอกฟอร์มที่มีหลายช่องความยาว
ต่างกัน

**เฉลยแนวทาง**: เพราะ `PROMPT` แสดงขอบเขตของแต่ละช่องกรอกให้เห็นชัดเจนตั้งแต่แรก (เช่น เห็นว่า
ช่องนี้มี 6 ขีด อีกช่องมี 15 ขีด) ผู้ใช้จึงรู้ล่วงหน้าว่าควรพิมพ์ข้อมูลยาวแค่ไหนโดยไม่ต้องลองพิมพ์แล้ว
พบว่าข้อมูลถูกตัดทิ้งเมื่อเกินขนาดช่อง (ปัญหาเดียวกับกฎ MOVE ที่เรียนใน Part 008) การเห็นขอบเขต
ล่วงหน้าจึงช่วยลดความสับสนและข้อผิดพลาดจากการกรอกข้อมูลเกินขนาดได้อย่างมีประสิทธิภาพ

---

## ขั้นตอนที่ 396: ACCEPT ทั้งฟอร์ม vs ACCEPT ทีละช่อง

### แนวคิด

เราเห็นมาแล้วว่า `ACCEPT screen-group-name` (เช่น `ACCEPT SCR-CUSTOMER-FORM`) รับข้อมูลได้
**ทุกช่องกรอกในกลุ่มเดียวคำสั่งเดียว** โดยเคอร์เซอร์จะไล่ไปทีละช่องตามลำดับที่ประกาศไว้ (ยกเว้นช่อง
ที่ไม่มี `USING` ซึ่งจะถูกข้าม) แต่บางสถานการณ์เราต้องการควบคุมการรับข้อมูล**ทีละช่องแยกกัน** เช่น
ต้องการตรวจสอบความถูกต้องของแต่ละช่องทันทีหลังกรอกเสร็จ ก่อนให้ผู้ใช้ไปช่องถัดไป — ทำได้โดยการ
`ACCEPT` ชื่อฟิลด์ย่อยแต่ละตัวแยกกัน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-WHOLE-VS-FIELD-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ITEM-NAME  PIC X(15).
       01  WS-ITEM-QTY   PIC 9(4).

       SCREEN SECTION.
       01  SCR-ITEM-FORM.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1  VALUE "ITEM ENTRY".
           05  LINE 3 COLUMN 1  VALUE "NAME:".
           05  SCR-ITEM-NAME LINE 3 COLUMN 10 PIC X(15)
               USING WS-ITEM-NAME.
           05  LINE 4 COLUMN 1  VALUE "QTY :".
           05  SCR-ITEM-QTY  LINE 4 COLUMN 10 PIC 9(4)
               USING WS-ITEM-QTY.

       PROCEDURE DIVISION.
      *> Option A: accept the whole group in one statement --
      *> cursor moves through every USING field automatically.
           DISPLAY SCR-ITEM-FORM.
           ACCEPT SCR-ITEM-FORM.

      *> Option B: accept ONE field at a time -- useful when each
      *> field needs its own validation before moving on.
           DISPLAY SCR-ITEM-FORM.
           ACCEPT SCR-ITEM-NAME.
           IF WS-ITEM-NAME = SPACES
               DISPLAY "NAME IS REQUIRED." LINE 6 COLUMN 1
           END-IF.
           ACCEPT SCR-ITEM-QTY.

           STOP RUN.
```

### อธิบายโค้ด

- `05 SCR-ITEM-NAME LINE 3 COLUMN 10 PIC X(15) USING WS-ITEM-NAME.` — สังเกตว่าฟิลด์ย่อยนี้
  มี**ชื่อของตัวเอง** (`SCR-ITEM-NAME`) ไม่ใช่ `FILLER` แบบตัวอย่างก่อนหน้า การตั้งชื่อให้ฟิลด์ย่อย
  ทำให้เราสามารถ `ACCEPT`/`DISPLAY` เฉพาะฟิลด์นั้นได้โดยตรง
- `ACCEPT SCR-ITEM-FORM.` (Option A) — รับข้อมูลทั้งฟอร์มในคำสั่งเดียว เหมาะกับฟอร์มง่าย ๆ ที่ไม่
  ต้องการตรวจสอบระหว่างทาง
- `ACCEPT SCR-ITEM-NAME.` แล้วตรวจสอบทันที ก่อน `ACCEPT SCR-ITEM-QTY.` (Option B) — เหมาะกับ
  ฟอร์มที่ต้องการ validation แต่ละช่องก่อนไปช่องถัดไป ให้ผู้ใช้แก้ไขได้ทันทีโดยไม่ต้องกรอกทั้งฟอร์มใหม่

### ผลการทดสอบคอมไพล์จริง

`cobc -x -o screen-whole-vs-field-demo screen-whole-vs-field-demo.cob` — **คอมไพล์สำเร็จ**
ทั้งการตั้งชื่อฟิลด์ย่อยและการ `ACCEPT`/`DISPLAY` แยกเป็นรายฟิลด์เป็นไวยากรณ์มาตรฐานที่ GnuCOBOL
รองรับโดยไม่มี error

### ข้อควรระวัง

- ฟิลด์ย่อยที่ต้องการ `ACCEPT`/`DISPLAY` แยกกันได้ **ต้องมีชื่อของตัวเอง** ไม่ใช่ `FILLER` — นี่คือ
  ความแตกต่างสำคัญจากตัวอย่างขั้นตอนก่อน ๆ ที่ใช้ label เป็น `FILLER` (ไม่มีชื่อ) เพราะไม่จำเป็นต้อง
  อ้างอิงถึงมันแยกภายหลัง
- การ `ACCEPT` ทีละฟิลด์ทำให้โค้ดยาวขึ้นเมื่อฟอร์มมีหลายช่อง แต่ให้ความยืดหยุ่นด้าน validation
  มากกว่ามาก ควรเลือกใช้ตามความซับซ้อนของกฎ validation ที่ต้องการจริง

### แบบฝึกหัดที่ 396.1

**โจทย์**: จงอธิบายว่าเมื่อไหร่ควรเลือกใช้ `ACCEPT` ทั้งกลุ่ม (whole group) และเมื่อไหร่ควรเลือกใช้
`ACCEPT` ทีละฟิลด์

**เฉลยแนวทาง**: ควรใช้ `ACCEPT` ทั้งกลุ่มเมื่อฟอร์มมีกฎการตรวจสอบข้อมูลไม่ซับซ้อน หรือต้องการ
ตรวจสอบทุกช่องพร้อมกันหลังกรอกครบแล้ว (เช่น ตรวจสอบความสัมพันธ์ระหว่างหลายฟิลด์) ในขณะที่ควรใช้
`ACCEPT` ทีละฟิลด์เมื่อแต่ละช่องมีกฎ validation เฉพาะตัวที่ต้องแจ้งเตือนผู้ใช้ทันทีก่อนไปช่องถัดไป
(เช่น ตรวจสอบว่ารหัสสินค้ามีอยู่จริงในระบบก่อนให้กรอกจำนวน) เพื่อประสบการณ์การใช้งานที่ดีกว่า

---

## ขั้นตอนที่ 397: ACCEPT/DISPLAY แบบ LINE/COLUMN โดยไม่ใช้ SCREEN SECTION

### แนวคิด

ก่อนที่ COBOL จะมี `SCREEN SECTION` เต็มรูปแบบ โปรแกรมเมอร์สามารถใช้ `DISPLAY`/`ACCEPT` ธรรมดา
(ที่เรียนใน Part 007) ร่วมกับ clause **`LINE`**/**`COLUMN`** เพื่อกำหนดตำแหน่งบนหน้าจอได้โดยตรง
โดยไม่ต้องประกาศ `SCREEN SECTION` เลย วิธีนี้เรียบง่ายกว่าแต่ขาดความสามารถขั้นสูง เช่น สี,
`REQUIRED`, `AUTO` ที่มีเฉพาะใน `SCREEN SECTION` เต็มรูปแบบ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-LINE-COLUMN-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME  PIC X(15).

       PROCEDURE DIVISION.
      *> No SCREEN SECTION at all -- LINE/COLUMN on plain
      *> DISPLAY/ACCEPT statements position text directly.
           DISPLAY "SIMPLE POSITIONED FORM" LINE 1 COLUMN 1.
           DISPLAY "NAME:" LINE 3 COLUMN 1.
           ACCEPT WS-NAME LINE 3 COLUMN 7.
           DISPLAY "YOU ENTERED:" LINE 5 COLUMN 1.
           DISPLAY WS-NAME LINE 5 COLUMN 14.
           STOP RUN.
```

### อธิบายโค้ด

- `DISPLAY "..." LINE 1 COLUMN 1.` — วางข้อความที่ตำแหน่งบรรทัด/คอลัมน์ที่ระบุ โดยไม่ต้องประกาศ
  โครงสร้างใด ๆ ล่วงหน้าใน DATA DIVISION เลย
- `ACCEPT WS-NAME LINE 3 COLUMN 7.` — รับค่าเข้า `WS-NAME` โดยวางเคอร์เซอร์ที่ตำแหน่งที่ระบุ
- วิธีนี้เหมาะกับโปรแกรมง่าย ๆ ที่ต้องการจัดตำแหน่งข้อความบ้างเล็กน้อย แต่ไม่ต้องการความสามารถ
  เต็มรูปแบบของฟอร์ม (สี, validation, ฯลฯ)

### ผลการทดสอบคอมไพล์จริงและการรัน

`cobc -x -o screen-line-column-demo screen-line-column-demo.cob` — **คอมไพล์สำเร็จ** เมื่อทดสอบ
รันแบบ headless (ป้อนข้อมูลผ่าน pipe เช่น `echo "SOMCHAI" | ./screen-line-column-demo`) พบว่า
โปรแกรมจบด้วย exit code 0 **แต่ยืนยันได้ชัดเจนว่าแม้แต่ `DISPLAY`/`ACCEPT` ธรรมดาที่มี clause
`LINE`/`COLUMN` ก็ทริกเกอร์การทำงานผ่านไลบรารี curses เช่นเดียวกับ `SCREEN SECTION`** — เอาต์พุต
ที่ได้เป็นรหัสควบคุมเทอร์มินัลเช่นเดียวกัน ไม่ใช่ข้อความอ่านง่าย ข้อจำกัดเรื่องเทอร์มินัลจริงที่กล่าวถึง
ในคำนำ Part นี้จึงครอบคลุมโครงสร้างนี้ด้วย ไม่ใช่แค่ `SCREEN SECTION` เท่านั้น

### เปรียบเทียบกับ Part 007

`DISPLAY`/`ACCEPT` แบบพื้นฐานที่เรียนใน Part 007 (ไม่มี `LINE`/`COLUMN`) ทำงานแบบ**เรียงลำดับ
ปกติ** (พิมพ์ที่บรรทัดถัดไปเสมอ) และ**ไม่ต้องพึ่งเทอร์มินัลแบบโต้ตอบ** จึงรันได้ปกติแม้ในสภาพแวดล้อม
headless (เหมือนที่เราใช้ตรวจสอบผลลัพธ์ตลอดหลักสูตรนี้) ในขณะที่การเพิ่ม `LINE`/`COLUMN` เข้าไป
เปลี่ยนพฤติกรรมให้กลายเป็นการควบคุมตำแหน่งบนจอแบบเต็มรูปแบบทันที ซึ่งต้องพึ่งเทอร์มินัลจริง

### ข้อควรระวัง

- **การเพิ่ม `LINE`/`COLUMN` แม้เพียง clause เดียวเข้าไปใน `DISPLAY`/`ACCEPT` ธรรมดา ก็เปลี่ยน
  พฤติกรรมของคำสั่งทั้งหมดให้ต้องพึ่งพาเทอร์มินัลจริงทันที** หากคุณเขียนโปรแกรมที่จะรันแบบ batch
  หรือถูกเรียกจากสคริปต์อัตโนมัติ (ไม่มีผู้ใช้นั่งหน้าจอโต้ตอบ) **ห้ามใช้ `LINE`/`COLUMN` เด็ดขาด**
  ให้ใช้ `DISPLAY`/`ACCEPT` แบบพื้นฐานจาก Part 007 แทน
- วิธีนี้ไม่รองรับสี, `REQUIRED`, `AUTO`, `SECURE` ฯลฯ — หากต้องการความสามารถเหล่านั้นต้องใช้
  `SCREEN SECTION` เต็มรูปแบบเท่านั้น

### แบบฝึกหัดที่ 397.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมที่จะถูกเรียกใช้งานแบบอัตโนมัติผ่านสคริปต์ (เช่น เป็นส่วนหนึ่งของ
batch job รายคืนที่เรียนถึงใน Part 023-030) จึงไม่ควรใช้ `DISPLAY`/`ACCEPT` ที่มี `LINE`/`COLUMN`
หรือ `SCREEN SECTION` เลย

**เฉลยแนวทาง**: เพราะทั้งสองอย่างต้องพึ่งพาเทอร์มินัลแบบโต้ตอบจริง (interactive terminal ผ่าน
ไลบรารี curses) ซึ่งไม่มีอยู่ในสภาพแวดล้อมที่โปรแกรมถูกเรียกโดยอัตโนมัติจากสคริปต์/scheduler
โดยไม่มีผู้ใช้นั่งหน้าจอ (เช่น cron job หรือ batch job ตอนกลางคืนที่เรียนถึงใน Part 066) การใช้
คำสั่งเหล่านี้ในบริบทนั้นจะทำให้โปรแกรมค้างรอการโต้ตอบที่ไม่มีทางเกิดขึ้น หรือได้ผลลัพธ์เป็นรหัสควบคุม
เทอร์มินัลที่ไม่มีความหมายในไฟล์ log เหมือนที่พบในการทดสอบของ Part นี้เอง

---

## ขั้นตอนที่ 398: การอัปเดตค่าและวาดหน้าจอใหม่ (Refreshing the Screen)

### แนวคิด

ฟอร์มแบบโต้ตอบส่วนใหญ่ต้องมีการ**อัปเดตข้อมูลแล้ววาดหน้าจอใหม่** เช่น หลังคำนวณผลรวมแล้วแสดง
ผลลัพธ์กลับไปที่ฟอร์มเดิม หรือแสดงข้อความสถานะที่เปลี่ยนไปตามการกระทำของผู้ใช้ วิธีทำคือ `MOVE`
ค่าใหม่เข้าตัวแปรที่ผูกกับ `USING` แล้วเรียก `DISPLAY` กลุ่มหน้าจอนั้นซ้ำอีกครั้ง — Report Writer
ไม่มีแนวคิดนี้ (รายงานเขียนทางเดียวจบ) แต่ `SCREEN SECTION` ออกแบบมาให้วาดซ้ำได้ตลอดเวลา

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-REFRESH-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE       PIC 9(6)V99 VALUE 0.
       01  WS-QTY         PIC 9(4)    VALUE 0.
       01  WS-TOTAL       PIC 9(8)V99 VALUE 0.
       01  WS-STATUS-MSG  PIC X(30)   VALUE SPACES.

       SCREEN SECTION.
       01  SCR-ORDER-FORM.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1  VALUE "ORDER CALCULATOR".
           05  LINE 3 COLUMN 1  VALUE "PRICE:".
           05  LINE 3 COLUMN 10 PIC ZZZ,ZZ9.99 USING WS-PRICE.
           05  LINE 4 COLUMN 1  VALUE "QTY  :".
           05  LINE 4 COLUMN 10 PIC ZZZ9        USING WS-QTY.
           05  LINE 6 COLUMN 1  VALUE "TOTAL:".
           05  LINE 6 COLUMN 10 PIC ZZ,ZZZ,ZZ9.99 USING WS-TOTAL.
           05  LINE 8 COLUMN 1  PIC X(30) USING WS-STATUS-MSG.

       PROCEDURE DIVISION.
           MOVE 129.50 TO WS-PRICE.
           MOVE 10     TO WS-QTY.
           MOVE "PRESS ENTER TO CALCULATE" TO WS-STATUS-MSG.

      *> First draw: shows initial price/qty, total still zero.
           DISPLAY SCR-ORDER-FORM.

      *> Recompute, then redraw the SAME screen group to refresh
      *> the on-screen TOTAL and status message.
           COMPUTE WS-TOTAL = WS-PRICE * WS-QTY.
           MOVE "CALCULATION COMPLETE." TO WS-STATUS-MSG.
           DISPLAY SCR-ORDER-FORM.

           STOP RUN.
```

### อธิบายโค้ด

- ครั้งแรกที่ `DISPLAY SCR-ORDER-FORM` ทำงาน `WS-TOTAL` ยังเป็น 0 (ยังไม่ได้คำนวณ) หน้าจอจึงแสดง
  TOTAL เป็น 0
- หลัง `COMPUTE WS-TOTAL = WS-PRICE * WS-QTY.` ค่าของ `WS-TOTAL` เปลี่ยนเป็น 1,295.00 และ
  `WS-STATUS-MSG` เปลี่ยนข้อความ
- `DISPLAY SCR-ORDER-FORM.` ครั้งที่สอง **วาดกลุ่มหน้าจอเดิมซ้ำ** โดยดึงค่าปัจจุบันของทุกตัวแปร
  `USING` มาแสดงใหม่ทั้งหมด ทำให้ผู้ใช้เห็น TOTAL และข้อความสถานะที่อัปเดตแล้วโดยไม่ต้องเขียน
  screen group ใหม่แยกต่างหาก

### ผลการทดสอบคอมไพล์จริง

`cobc -x -o screen-refresh-demo screen-refresh-demo.cob` — **คอมไพล์สำเร็จ** การ `DISPLAY`
กลุ่มหน้าจอเดิมซ้ำหลายครั้งเป็นรูปแบบที่ถูกต้องตามไวยากรณ์ COBOL มาตรฐานและ GnuCOBOL รองรับ
โดยไม่มี error

### ภาพจำลองหน้าจอ — ก่อนและหลังคำนวณ (mockup)

ก่อนคำนวณ:
```
ORDER CALCULATOR

PRICE:    129.50
QTY  :   0010

TOTAL:       0.00

PRESS ENTER TO CALCULATE
```

หลังคำนวณและวาดหน้าจอใหม่:
```
ORDER CALCULATOR

PRICE:    129.50
QTY  :   0010

TOTAL:   1,295.00

CALCULATION COMPLETE.
```

### ข้อควรระวัง

- การ `DISPLAY` ซ้ำจะวาดทับต้องอาศัย `BLANK SCREEN` ที่ต้นกลุ่มเสมอ มิเช่นนั้นเนื้อหาเก่าที่ตำแหน่ง
  เดียวกันอาจไม่ถูกล้างออกอย่างสมบูรณ์ในบางกรณี (เช่น ข้อความใหม่สั้นกว่าข้อความเก่าที่ตำแหน่งเดิม)
- ควรวางแผนว่าฟิลด์ไหนควรมีชื่อของตัวเองเพื่อให้ `ACCEPT`/`DISPLAY` แยกฟิลด์ได้ (ตามที่เรียนใน
  ขั้นตอนที่ 396) หากต้องการอัปเดตแค่บางฟิลด์โดยไม่วาดทั้งฟอร์มใหม่ทุกครั้ง

### แบบฝึกหัดที่ 398.1

**โจทย์**: จงอธิบายว่าทำไมการ "วาดหน้าจอซ้ำ" ด้วย `SCREEN SECTION` จึงเหมาะกับโปรแกรมแบบ
โต้ตอบ (interactive) มากกว่าแนวทาง Report Writer ที่เรียนใน Part 038-039

**เฉลยแนวทาง**: เพราะ Report Writer ถูกออกแบบมาสำหรับงานที่เขียนผลลัพธ์ลงไฟล์/กระดาษแบบ
**ทางเดียว** (write-once) — เขียนแล้วจบ ไม่มีแนวคิดเรื่อง "แก้ไขแล้ววาดใหม่" เพราะกระดาษที่พิมพ์ไป
แล้วแก้ไขไม่ได้ ในขณะที่ `SCREEN SECTION` ทำงานบนหน้าจอที่**เปลี่ยนแปลงได้ตลอดเวลา** เหมาะกับ
สถานการณ์ที่ผู้ใช้โต้ตอบไปเรื่อย ๆ (กรอกข้อมูล, ดูผลลัพธ์, แก้ไข, ดูผลลัพธ์ใหม่) ซึ่งเป็นธรรมชาติของ
โปรแกรมประเภท data entry หรือเมนูแบบ interactive ที่ผู้ใช้คาดหวังให้หน้าจออัปเดตทันทีตามการกระทำ
ของตน

---

## ขั้นตอนที่ 399: สร้างเมนูแบบโต้ตอบด้วย SCREEN SECTION และ EVALUATE

### แนวคิด

การผสาน `SCREEN SECTION` เข้ากับ `EVALUATE` (Part 011) และ `PERFORM UNTIL` (Part 012)
คือรูปแบบที่พบบ่อยที่สุดในการสร้าง**เมนูหลัก (main menu)** ของโปรแกรม Text-mode: วนลูปแสดงเมนู
รับตัวเลือกจากผู้ใช้ แล้วใช้ `EVALUATE` ตัดสินใจว่าจะทำอะไรต่อ จนกว่าผู้ใช้จะเลือกออกจากโปรแกรม

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-MENU-DEMO.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE  PIC 9(1) VALUE 0.
       01  WS-EXIT-FLAG    PIC X    VALUE "N".
       01  WS-CUST-NAME    PIC X(20).
       01  WS-CUST-AGE     PIC 9(3).

       SCREEN SECTION.
       01  SCR-MAIN-MENU.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1 VALUE "=== MAIN MENU ==="
               FOREGROUND-COLOR 3 HIGHLIGHT.
           05  LINE 3 COLUMN 1 VALUE "1. ADD CUSTOMER".
           05  LINE 4 COLUMN 1 VALUE "2. VIEW LAST CUSTOMER".
           05  LINE 5 COLUMN 1 VALUE "3. EXIT".
           05  LINE 7 COLUMN 1 VALUE "ENTER CHOICE (1-3):".
           05  LINE 7 COLUMN 21 PIC 9(1) USING WS-MENU-CHOICE
               REQUIRED.

       01  SCR-ADD-CUSTOMER.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1 VALUE "ADD CUSTOMER".
           05  LINE 3 COLUMN 1 VALUE "NAME:".
           05  LINE 3 COLUMN 10 PIC X(20) USING WS-CUST-NAME
               REQUIRED.
           05  LINE 4 COLUMN 1 VALUE "AGE :".
           05  LINE 4 COLUMN 10 PIC 9(3)  USING WS-CUST-AGE
               REQUIRED.

       01  SCR-VIEW-CUSTOMER.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1 VALUE "LAST CUSTOMER ADDED".
           05  LINE 3 COLUMN 1 VALUE "NAME:".
           05  LINE 3 COLUMN 10 PIC X(20) USING WS-CUST-NAME.
           05  LINE 4 COLUMN 1 VALUE "AGE :".
           05  LINE 4 COLUMN 10 PIC 9(3)  USING WS-CUST-AGE.
           05  LINE 6 COLUMN 1 VALUE "PRESS ENTER TO CONTINUE...".

       PROCEDURE DIVISION.
           PERFORM UNTIL WS-EXIT-FLAG = "Y"
               DISPLAY SCR-MAIN-MENU
               ACCEPT SCR-MAIN-MENU

               EVALUATE WS-MENU-CHOICE
                   WHEN 1
                       DISPLAY SCR-ADD-CUSTOMER
                       ACCEPT SCR-ADD-CUSTOMER
                   WHEN 2
                       DISPLAY SCR-VIEW-CUSTOMER
                       ACCEPT WS-MENU-CHOICE
                   WHEN 3
                       MOVE "Y" TO WS-EXIT-FLAG
                   WHEN OTHER
                       CONTINUE
               END-EVALUATE
           END-PERFORM.

           STOP RUN.
```

### อธิบายโค้ด

- `PERFORM UNTIL WS-EXIT-FLAG = "Y"` — วนลูปหลักของโปรแกรม ตราบใดที่ผู้ใช้ยังไม่เลือกตัวเลือก
  "3. EXIT"
- ในแต่ละรอบ: วาดเมนูใหม่ (`DISPLAY SCR-MAIN-MENU`), รับตัวเลือก (`ACCEPT SCR-MAIN-MENU`),
  แล้วใช้ `EVALUATE WS-MENU-CHOICE` (เทคนิคจาก Part 011) ตัดสินใจว่าจะแสดงหน้าจอไหนต่อ
- `WHEN 1` — เปลี่ยนไปแสดงฟอร์มเพิ่มลูกค้า แล้วรับข้อมูลกลับเข้า `WS-CUST-NAME`/`WS-CUST-AGE`
- `WHEN 2` — แสดงข้อมูลลูกค้าล่าสุดที่เคยกรอกไว้ (ถ้ายังไม่เคยกรอกเลยจะเห็นค่าว่าง/ศูนย์) แล้วรอ
  ผู้ใช้กด Enter ก่อนวนกลับไปเมนูหลัก
- `WHEN 3` — ตั้งค่า `WS-EXIT-FLAG` เป็น "Y" ทำให้ลูป `PERFORM UNTIL` จบการทำงานในรอบถัดไป
- `WHEN OTHER` — หากผู้ใช้กรอกตัวเลขนอกช่วง 1-3 ให้ `CONTINUE` (ไม่ทำอะไร) แล้ววนกลับไปแสดง
  เมนูใหม่อีกครั้ง

### ผลการทดสอบคอมไพล์จริง

`cobc -x -o screen-menu-demo screen-menu-demo.cob` — **คอมไพล์สำเร็จ 100%** การผสาน
`SCREEN SECTION` เข้ากับ `PERFORM UNTIL` และ `EVALUATE` เป็นไวยากรณ์ที่ถูกต้องสมบูรณ์และ
GnuCOBOL รองรับโดยไม่มี error ใด ๆ พฤติกรรมการวนเมนูจริงต้องทดสอบบนเทอร์มินัลแบบโต้ตอบเท่านั้น

### ข้อควรระวัง

- อย่าลืม `REQUIRED` บนช่องรับตัวเลือกเมนู มิเช่นนั้นผู้ใช้อาจกด Enter โดยไม่กรอกอะไรเลย ทำให้
  `WS-MENU-CHOICE` คงค่าเดิมจากรอบก่อนหน้าโดยไม่ได้ตั้งใจ
- ควรมี `WHEN OTHER` ในทุก `EVALUATE` ที่ควบคุมเมนู เพื่อป้องกันไม่ให้โปรแกรมค้างหรือพังเมื่อผู้ใช้
  กรอกค่าที่ไม่คาดคิด (หลักการเดียวกับที่เรียนใน Part 011 ขั้นตอนที่ 106)
- โครงสร้างนี้เป็นรากฐานสำคัญของโปรแกรม Text-mode UI แทบทุกโปรแกรมในโลก COBOL Legacy —
  ควรฝึกฝนให้คล่องเพราะจะได้ใช้ซ้ำบ่อยมากในเฟสถัดไปของหลักสูตร

### แบบฝึกหัดที่ 399.1

**โจทย์**: จงเพิ่มตัวเลือกเมนูใหม่ "4. SHOW SYSTEM INFO" ที่แสดงข้อความ "COBOL COURSE SCREEN
DEMO VERSION 1.0" แล้วรอผู้ใช้กด Enter ก่อนกลับไปเมนูหลัก

**เฉลยแนวทาง**:
```cobol
       01  SCR-MAIN-MENU.
           ...
           05  LINE 5 COLUMN 1 VALUE "3. EXIT".
           05  LINE 6 COLUMN 1 VALUE "4. SHOW SYSTEM INFO".
           05  LINE 8 COLUMN 1 VALUE "ENTER CHOICE (1-4):".
           05  LINE 8 COLUMN 21 PIC 9(1) USING WS-MENU-CHOICE
               REQUIRED.

       01  SCR-SYSTEM-INFO.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1 VALUE "SYSTEM INFO".
           05  LINE 3 COLUMN 1 VALUE
               "COBOL COURSE SCREEN DEMO VERSION 1.0".
           05  LINE 5 COLUMN 1 VALUE "PRESS ENTER TO CONTINUE...".
       ...
       PROCEDURE DIVISION.
           ...
               EVALUATE WS-MENU-CHOICE
                   ...
                   WHEN 4
                       DISPLAY SCR-SYSTEM-INFO
                       ACCEPT WS-MENU-CHOICE
                   WHEN OTHER
                       CONTINUE
               END-EVALUATE
```

---

## ขั้นตอนที่ 400: โปรแกรมรวบยอด — ระบบลงทะเบียนลูกค้าแบบฟอร์มเต็มรูปแบบ

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่รวมทุกเทคนิคของ `SCREEN SECTION` เข้าด้วยกัน: `BLANK SCREEN`,
ช่องกรอกหลายชนิด (`PIC X`, `PIC 9`), สีสัน, `REQUIRED`/`AUTO`/`SECURE`/`PROMPT`, การอัปเดต
และวาดหน้าจอใหม่ ผสานกับเมนูแบบ `EVALUATE`/`PERFORM UNTIL`

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SCREEN-CAPSTONE-REGISTRATION.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE   PIC 9(1) VALUE 0.
       01  WS-EXIT-FLAG     PIC X    VALUE "N".
       01  WS-CUST-ID       PIC X(6).
       01  WS-CUST-NAME     PIC X(20).
       01  WS-CUST-AGE      PIC 9(3).
       01  WS-CUST-PIN      PIC 9(4).
       01  WS-STATUS-MSG    PIC X(35) VALUE SPACES.
       01  WS-RECORD-COUNT  PIC 9(3)  VALUE 0.

       SCREEN SECTION.
       01  SCR-MAIN-MENU.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1 VALUE "CUSTOMER REGISTRATION SYSTEM"
               FOREGROUND-COLOR 3 HIGHLIGHT.
           05  LINE 3 COLUMN 1 VALUE "1. REGISTER NEW CUSTOMER".
           05  LINE 4 COLUMN 1 VALUE "2. EXIT".
           05  LINE 6 COLUMN 1 VALUE "RECORDS SO FAR:".
           05  LINE 6 COLUMN 17 PIC ZZ9 USING WS-RECORD-COUNT.
           05  LINE 8 COLUMN 1 VALUE "ENTER CHOICE (1-2):".
           05  LINE 8 COLUMN 21 PIC 9(1) USING WS-MENU-CHOICE
               REQUIRED.

       01  SCR-REGISTER-FORM.
           05  BLANK SCREEN.
           05  LINE 1 COLUMN 1 VALUE "NEW CUSTOMER REGISTRATION"
               FOREGROUND-COLOR 2 HIGHLIGHT.
           05  LINE 3 COLUMN 1  VALUE "CUSTOMER ID:".
           05  LINE 3 COLUMN 14 PIC X(6) USING WS-CUST-ID
               REQUIRED AUTO PROMPT.
           05  LINE 4 COLUMN 1  VALUE "FULL NAME  :".
           05  LINE 4 COLUMN 14 PIC X(20) USING WS-CUST-NAME
               REQUIRED PROMPT.
           05  LINE 5 COLUMN 1  VALUE "AGE        :".
           05  LINE 5 COLUMN 14 PIC 9(3) USING WS-CUST-AGE
               REQUIRED AUTO.
           05  LINE 6 COLUMN 1  VALUE "SECURITY PIN:".
           05  LINE 6 COLUMN 14 PIC 9(4) USING WS-CUST-PIN
               REQUIRED SECURE AUTO.
           05  LINE 8 COLUMN 1  PIC X(35) USING WS-STATUS-MSG
               FOREGROUND-COLOR 2.

       PROCEDURE DIVISION.
           PERFORM UNTIL WS-EXIT-FLAG = "Y"
               DISPLAY SCR-MAIN-MENU
               ACCEPT SCR-MAIN-MENU

               EVALUATE WS-MENU-CHOICE
                   WHEN 1
                       PERFORM REGISTER-NEW-CUSTOMER
                   WHEN 2
                       MOVE "Y" TO WS-EXIT-FLAG
                   WHEN OTHER
                       CONTINUE
               END-EVALUATE
           END-PERFORM.

           DISPLAY "TOTAL CUSTOMERS REGISTERED: " WS-RECORD-COUNT.
           STOP RUN.

       REGISTER-NEW-CUSTOMER.
           MOVE SPACES TO WS-STATUS-MSG.
           DISPLAY SCR-REGISTER-FORM.
           ACCEPT SCR-REGISTER-FORM.

           ADD 1 TO WS-RECORD-COUNT.
           MOVE "CUSTOMER REGISTERED SUCCESSFULLY!" TO WS-STATUS-MSG.
           DISPLAY SCR-REGISTER-FORM.
```

### อธิบายโค้ด

โปรแกรมนี้ผสานทุกเทคนิคของ Part 040: เมนูหลักที่มีตัวนับจำนวน record (`WS-RECORD-COUNT`)
แสดงผลแบบ real-time, ฟอร์มลงทะเบียนที่มีทั้งช่องข้อความ/ตัวเลข พร้อม `REQUIRED`, `AUTO`,
`SECURE`, `PROMPT` ครบทุกแบบ, การอัปเดตข้อความสถานะแล้ววาดฟอร์มซ้ำเพื่อยืนยันความสำเร็จ
(ตามเทคนิคขั้นตอนที่ 398), และโครงสร้างเมนูวนลูปด้วย `EVALUATE`/`PERFORM UNTIL` (ขั้นตอนที่ 399)

### ผลการทดสอบคอมไพล์จริง

ทดสอบด้วย `cobc -x -o screen-capstone-registration screen-capstone-registration.cob`
**ผลลัพธ์: คอมไพล์สำเร็จสมบูรณ์ ไม่มี compile error หรือ warning ใด ๆ** (exit code 0) ยืนยันว่า
โครงสร้างโปรแกรมทั้งหมด (เมนู, ฟอร์ม, paragraph แยก, การเรียก `PERFORM REGISTER-NEW-CUSTOMER`)
ถูกต้องตามไวยากรณ์ COBOL อย่างสมบูรณ์ ตามข้อจำกัดที่อธิบายไว้ในคำนำของ Part นี้ ผลลัพธ์การแสดงผล
หน้าจอจริงต้องทดสอบบนเทอร์มินัลแบบโต้ตอบด้วยตนเอง

### ภาพจำลองหน้าจอ (mockup) — เมนูหลัก

```
CUSTOMER REGISTRATION SYSTEM

1. REGISTER NEW CUSTOMER
2. EXIT

RECORDS SO FAR:  2

ENTER CHOICE (1-2):
```

### ภาพจำลองหน้าจอ (mockup) — ฟอร์มลงทะเบียนหลังบันทึกสำเร็จ

```
NEW CUSTOMER REGISTRATION

CUSTOMER ID: C00123
FULL NAME  : SIRIPORN SUKSAWAT
AGE        : 028
SECURITY PIN: ****

CUSTOMER REGISTERED SUCCESSFULLY!
```

### ข้อควรระวัง

- โปรแกรมนี้เก็บข้อมูลลูกค้าไว้เพียง**คนเดียวล่าสุด**ใน WORKING-STORAGE เท่านั้น (ไม่มีตาราง/ไฟล์
  เก็บประวัติ) หากต้องการเก็บลูกค้าหลายคน ต้องผสานกับความรู้เรื่อง Table/OCCURS (Part 016) หรือ
  การเขียนไฟล์ (Part 023-025) ที่เรียนมาก่อนหน้า
- ควรทดสอบโปรแกรมนี้จริงบนเทอร์มินัลของคุณเอง เพื่อฝึกความคุ้นเคยกับการโต้ตอบผ่าน `SCREEN
  SECTION` แบบเต็มรูปแบบ ซึ่งเป็นทักษะสำคัญสำหรับโปรแกรม COBOL แบบ Interactive ในโลกจริง

### แบบฝึกหัดที่ 400.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น โดยเพิ่มการตรวจสอบว่า `WS-CUST-AGE` ต้องมีค่าอย่างน้อย 18
ก่อนจะยอมรับการลงทะเบียน (ถ้าอายุน้อยกว่า 18 ให้แสดงข้อความ "REGISTRATION FAILED: AGE MUST
BE 18 OR OVER." แทนข้อความสำเร็จ และไม่นับเพิ่มใน `WS-RECORD-COUNT`)

**เฉลยแนวทาง**:
```cobol
       REGISTER-NEW-CUSTOMER.
           MOVE SPACES TO WS-STATUS-MSG.
           DISPLAY SCR-REGISTER-FORM.
           ACCEPT SCR-REGISTER-FORM.

           IF WS-CUST-AGE < 18
               MOVE "REGISTRATION FAILED: AGE MUST BE 18 OR OVER."
                   TO WS-STATUS-MSG
           ELSE
               ADD 1 TO WS-RECORD-COUNT
               MOVE "CUSTOMER REGISTERED SUCCESSFULLY!"
                   TO WS-STATUS-MSG
           END-IF.

           DISPLAY SCR-REGISTER-FORM.
```
สังเกตว่าเทคนิคนี้คือการผสาน `IF` (Part 010) เข้ากับการอัปเดต `WS-STATUS-MSG` แล้ววาดฟอร์มซ้ำ
(ขั้นตอนที่ 398) — เป็นรูปแบบมาตรฐานของการทำ validation ในฟอร์มแบบ `SCREEN SECTION`

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้ **`SCREEN SECTION`** สำหรับสร้าง UI แบบ Text-mode อย่างครบถ้วน:

- โครงสร้างพื้นฐาน `BLANK SCREEN`, `LINE`/`COLUMN`, ข้อความคงที่ด้วย `VALUE`
- ช่องกรอกข้อมูลด้วย `PIC` และ `USING` clause ที่ทำงานได้ทั้งขาเข้า-ขาออก
- สีสันและการเน้นข้อความ: `FOREGROUND-COLOR`, `BACKGROUND-COLOR`, `HIGHLIGHT`, `REVERSE-VIDEO`
- การควบคุมพฤติกรรมช่องกรอก: `AUTO`, `REQUIRED`, `SECURE`
- ตัวช่วยด้านการมองเห็น: `PROMPT`, `UNDERLINE`
- ความแตกต่างระหว่าง `ACCEPT` ทั้งกลุ่มกับ `ACCEPT` ทีละฟิลด์
- `DISPLAY`/`ACCEPT` แบบ `LINE`/`COLUMN` โดยไม่ใช้ `SCREEN SECTION` เต็มรูปแบบ
- การอัปเดตค่าและวาดหน้าจอใหม่ (refresh) ซึ่งเป็นแนวคิดที่ Report Writer ไม่มี
- การผสาน `SCREEN SECTION` กับ `EVALUATE`/`PERFORM UNTIL` สร้างเมนูแบบโต้ตอบ
- โปรแกรมรวบยอดระบบลงทะเบียนลูกค้าที่ผสานทุกเทคนิคเข้าด้วยกัน

**ข้อจำกัดสำคัญที่ควรจำจาก Part นี้**: `SCREEN SECTION` (และแม้แต่ `DISPLAY`/`ACCEPT` ธรรมดา
ที่มี `LINE`/`COLUMN`) ต้องการเทอร์มินัลจริงเสมอ ทุกตัวอย่างในนี้ทดสอบยืนยันแล้วว่า**คอมไพล์ผ่าน
สมบูรณ์ 100%** แต่ผลลัพธ์การแสดงผลจริงต้องลองรันด้วยตนเองบนเทอร์มินัลของคุณ — นี่คือความแตกต่าง
สำคัญจาก Part อื่น ๆ ในหลักสูตรที่สามารถแคปเจอร์ผลลัพธ์การรันจริงมาแสดงในเอกสารได้โดยตรง

ใน **Part 041** เราจะกลับไปเจาะลึก **`EVALUATE`** ที่ใช้ในเมนูของ Part นี้อีกครั้ง แต่คราวนี้ใน
ระดับที่ซับซ้อนขึ้นมาก — การใช้หลาย subject พร้อมกันด้วย `ALSO`, เงื่อนไขผสมภายใน `WHEN` เดียว,
และ `EVALUATE` ที่ซ้อนกันหลายชั้น ซึ่งจะเป็นประโยชน์มากเมื่อคุณต้องเขียนตรรกะทางธุรกิจที่ซับซ้อน
ขึ้นในโปรแกรมแบบโต้ตอบเหล่านี้

**[← กลับไป Part 039](part-039-report-writer-control-breaks.md)** | **[ไปยัง Part 041: Advanced EVALUATE และเงื่อนไขซับซ้อน →](part-041-advanced-evaluate.md)**
