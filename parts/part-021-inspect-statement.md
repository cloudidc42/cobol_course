# Part 021: INSPECT Statement: นับ แทนที่ แปลงตัวอักษร (ขั้นตอนที่ 201–210)

## คำนำของ Part นี้

ใน Part 019 และ Part 020 เราได้เรียนรู้ `STRING` (รวมข้อความ) และ `UNSTRING` (แยกข้อความ) ไปแล้ว
Part นี้จะแนะนำเครื่องมือจัดการข้อความตัวที่สามที่สำคัญไม่แพ้กันคือ **`INSPECT`** ซึ่งทำหน้าที่แตกต่าง
จากสองคำสั่งก่อนหน้าโดยสิ้นเชิง: `INSPECT` ไม่ได้รวมหรือแยกข้อความ แต่ใช้สำหรับ **ตรวจสอบและแก้ไข
เนื้อหาภายในฟิลด์เดียว** ผ่านสามความสามารถหลัก ได้แก่ **นับ (`TALLYING`)** จำนวนครั้งที่ตัวอักษรหรือ
ข้อความปรากฏ **แทนที่ (`REPLACING`)** ตัวอักษรหรือข้อความบางส่วนด้วยค่าใหม่ และ **แปลง (`CONVERTING`)**
ชุดตัวอักษรทั้งหมดไปเป็นอีกชุดหนึ่ง (เช่น แปลงตัวพิมพ์เล็กเป็นตัวพิมพ์ใหญ่)

`INSPECT` เป็นเครื่องมือที่ใช้บ่อยมากในงาน **การทำความสะอาดข้อมูล (Data Cleansing)** ซึ่งเป็นขั้นตอน
สำคัญก่อนนำข้อมูลไปประมวลผลจริง ไม่ว่าจะเป็นการนับจำนวนตัวคั่นในข้อมูล CSV ก่อนเรียก `UNSTRING`
การมาตรฐานตัวพิมพ์ของข้อมูลที่ผู้ใช้ป้อน หรือการปิดบังข้อมูลอ่อนไหว (เช่น เลขบัตรเครดิต) ก่อนแสดงผล
Part นี้จะพาคุณเรียนรู้ทั้งสามความสามารถของ `INSPECT` อย่างละเอียด พร้อมตัวอย่างการใช้งานจริงที่ผสมผสาน
เข้ากับ `STRING`/`UNSTRING` ที่เรียนไปแล้ว

---

## ขั้นตอนที่ 201: แนะนำ INSPECT และภาพรวมสามความสามารถหลัก

### แนวคิด

`INSPECT` คือคำสั่งที่ COBOL มีให้สำหรับ**ตรวจสอบและแก้ไขเนื้อหาภายในฟิลด์เดียว** โดยไม่เปลี่ยนขนาด
ของฟิลด์นั้นเลย มันมีสามรูปแบบหลักที่ใช้แยกกันหรือรวมกันในคำสั่งเดียวก็ได้:

1. **`TALLYING`** — นับจำนวนครั้งที่ตัวอักษรหรือข้อความที่กำหนดปรากฏอยู่ ไม่แก้ไขข้อมูลใด ๆ
2. **`REPLACING`** — แทนที่ตัวอักษรหรือข้อความที่ตรงเงื่อนไขด้วยค่าใหม่ที่มี**ความยาวเท่ากัน**
3. **`CONVERTING`** — แปลงตัวอักษรแต่ละตัวในชุดหนึ่งไปเป็นตัวอักษรที่ตำแหน่งเดียวกันในอีกชุดหนึ่ง

ตัวอย่างแรกนี้แสดงรูปแบบพื้นฐานที่สุดคือ `TALLYING ... FOR ALL` เพื่อนับจำนวนตัวอักษรที่ปรากฏในข้อความ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s1.cob
      *> Purpose : Introduce INSPECT with a basic TALLYING FOR ALL
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-INTRO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEXT              PIC X(30) VALUE "MISSISSIPPI".
       01  WS-COUNT-S           PIC 9(2) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> INSPECT ... TALLYING counts how many times something
      *> appears in a field, WITHOUT changing the field itself.
           INSPECT WS-TEXT TALLYING WS-COUNT-S FOR ALL "S"

           DISPLAY "Text     : " WS-TEXT
           DISPLAY "Count of S: " WS-COUNT-S
           DISPLAY "Text after INSPECT (unchanged): " WS-TEXT

           STOP RUN.
```

### คำอธิบายโค้ด

- `INSPECT WS-TEXT TALLYING WS-COUNT-S FOR ALL "S"` สั่งให้ COBOL สแกน `WS-TEXT` ทั้งหมด แล้วบวกค่า
  ใน `WS-COUNT-S` ขึ้นทีละ 1 ทุกครั้งที่พบตัวอักษร "S" ปรากฏอยู่ (ไม่ว่าจะติดกันหรือแยกกัน)
- สังเกตว่า `WS-TEXT` ยังคงมีค่าเดิมทุกประการหลังคำสั่งทำงาน — `TALLYING` เป็นการอ่านและนับเท่านั้น
  ไม่แก้ไขข้อมูลต้นทางแต่อย่างใด

### ผลลัพธ์ที่ได้จากการรันจริง

```
Text     : MISSISSIPPI
Count of S: 04
Text after INSPECT (unchanged): MISSISSIPPI
```

### ข้อควรระวัง

- ตัวแปรที่ใช้เก็บผลนับ (`WS-COUNT-S`) **ไม่ถูกล้างเป็น 0 ให้อัตโนมัติ** ก่อนเริ่มนับ `TALLYING` จะ
  **บวกเพิ่ม**เข้าไปในค่าเดิมที่มีอยู่เสมอ ถ้าเรียก `INSPECT` ซ้ำหลายครั้งโดยไม่เคลียร์ค่าตัวแปรก่อน
  ผลนับของหลายรอบจะสะสมปนกัน (พฤติกรรมเดียวกับ `TALLYING IN` ของ `UNSTRING` ที่เรียนใน Part 020)

### แบบฝึกหัดที่ 201.1

**โจทย์**: จงแก้โค้ดข้างต้นให้นับจำนวนตัวอักษร "I" แทน "S" ใน "MISSISSIPPI"

**เฉลย**: เปลี่ยน `FOR ALL "S"` เป็น `FOR ALL "I"` ผลลัพธ์ที่ได้จะเป็น `Count of I: 04`
เพราะคำว่า "MISSISSIPPI" มีตัว I อยู่ 4 ตัวเช่นกัน

---

## ขั้นตอนที่ 202: TALLYING FOR LEADING และ FOR CHARACTERS

### แนวคิด

นอกจาก `FOR ALL` แล้ว `TALLYING` ยังมีวลีย่อยอีกสองแบบที่ให้ผลต่างกัน: **`FOR LEADING`** จะนับเฉพาะ
**ชุดตัวอักษรที่ซ้ำกันต่อเนื่องตั้งแต่ตำแหน่งแรกสุด**เท่านั้น พอเจอตัวอักษรอื่นที่ไม่ตรงก็จะหยุดนับทันที
(ต่างจาก `FOR ALL` ที่นับทุกตำแหน่งที่พบไม่ว่าจะอยู่ตรงไหน) ส่วน **`FOR CHARACTERS`** จะนับ**ทุกตำแหน่ง
ตัวอักษร**โดยไม่สนใจค่าของมันเลย มักใช้ร่วมกับ `BEFORE`/`AFTER` (ที่จะเรียนในขั้นตอนที่ 207) เพื่อวัด
ความยาวของข้อมูลจริงก่อนถึงจุดใดจุดหนึ่ง

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s2.cob
      *> Purpose : TALLYING FOR LEADING (counts only a run at
      *>           the start) versus FOR CHARACTERS (counts
      *>           every single character position)
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-TALLY-LEADING.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEXT              PIC X(20) VALUE "00012300".
       01  WS-LEAD-ZEROS        PIC 9(2) VALUE 0.
       01  WS-ALL-CHARS         PIC 9(2) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> FOR LEADING "0" stops counting the moment a non-"0"
      *> character is reached, so it only counts the run at
      *> the very beginning of the field.
           INSPECT WS-TEXT TALLYING WS-LEAD-ZEROS
               FOR LEADING "0"
           DISPLAY "Text          : " WS-TEXT
           DISPLAY "Leading zeros : " WS-LEAD-ZEROS

      *> FOR CHARACTERS counts every filled character position,
      *> regardless of its value (useful to measure actual length
      *> before trailing spaces, when combined with BEFORE SPACE
      *> as we will see later in this Part).
           INSPECT WS-TEXT TALLYING WS-ALL-CHARS
               FOR CHARACTERS BEFORE SPACE
           DISPLAY "Characters before first space: " WS-ALL-CHARS

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-TEXT` มีค่า "00012300" ตามด้วยช่องว่างเติมท้ายจนครบ 20 ไบต์ (`PIC X(20)`)
- `FOR LEADING "0"` นับได้แค่ 3 ตัว (เลข 0 สามตัวแรกสุด) เพราะพอเจอเลข "1" ที่ตำแหน่งที่ 4 มันก็หยุดนับ
  ทันที แม้ว่าจะมีเลข "0" อีกสองตัวอยู่ตรงกลาง ("300") ก็ไม่ถูกนับเพราะไม่ใช่ชุดต่อเนื่องจากจุดเริ่มต้น
- `FOR CHARACTERS BEFORE SPACE` นับทุกตัวอักษร (ไม่ว่าค่าอะไร) ก่อนจะเจอช่องว่างตัวแรก ได้ผลลัพธ์เป็น 8
  ซึ่งเท่ากับความยาวจริงของข้อมูล "00012300" ก่อนช่องว่างที่เติมท้าย

### ผลลัพธ์ที่ได้จากการรันจริง

```
Text          : 00012300
Leading zeros : 03
Characters before first space: 08
```

### ข้อควรระวัง

- อย่าสับสนระหว่าง `FOR ALL` กับ `FOR LEADING`: `FOR ALL "0"` กับข้อความ "00012300" จะนับได้ 5 ตัว
  (ทุกตำแหน่งที่มี "0") แต่ `FOR LEADING "0"` จะนับได้แค่ 3 ตัว (เฉพาะที่ต้นข้อความ) ความแตกต่างนี้
  สำคัญมากเมื่อใช้ตรวจสอบรูปแบบข้อมูล เช่น นับเลขศูนย์นำหน้าของรหัสไปรษณีย์หรือรหัสสินค้า

### แบบฝึกหัดที่ 202.1

**โจทย์**: จงคาดเดาผลลัพธ์ถ้าเปลี่ยน `FOR LEADING "0"` เป็น `FOR ALL "0"` กับข้อความ "00012300" เดิม

**เฉลย**: จะได้ผลลัพธ์เป็น 5 เพราะ "00012300" มีเลข 0 อยู่ทั้งหมด 5 ตำแหน่ง (สามตัวแรก และอีกสองตัว
ในส่วน "300") ในขณะที่ `FOR LEADING` นับได้แค่ 3 ตัวแรกเท่านั้นตามที่อธิบายไปแล้ว

---

## ขั้นตอนที่ 203: INSPECT REPLACING ALL - แทนที่ทุกตำแหน่งที่พบ

### แนวคิด

**`REPLACING ALL ค่าเดิม BY ค่าใหม่`** ใช้แทนที่**ทุกตำแหน่ง**ที่พบค่าเดิมด้วยค่าใหม่ทั่วทั้งฟิลด์
กฎสำคัญที่ต้องจำคือ **ค่าเดิมและค่าใหม่ต้องมีความยาวเท่ากันเสมอ** เพราะ `INSPECT` ไม่สามารถเปลี่ยนขนาด
ของฟิลด์ได้ — มันแค่เขียนทับตัวอักษรตำแหน่งเดิมด้วยตัวอักษรใหม่เท่านั้น การใช้งานที่พบบ่อยที่สุดคือ
การปรับรูปแบบตัวคั่น เช่น เปลี่ยนขีดกลางในเบอร์โทรศัพท์ให้เป็นช่องว่าง

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s3.cob
      *> Purpose : INSPECT REPLACING ALL - replace every match
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-REPLACE-ALL.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PHONE             PIC X(20) VALUE "081-234-5678".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "Before: " WS-PHONE

      *> REPLACING ALL swaps EVERY occurrence of the old value
      *> with the new one, across the whole field.
           INSPECT WS-PHONE REPLACING ALL "-" BY " "

           DISPLAY "After : " WS-PHONE

           STOP RUN.
```

### คำอธิบายโค้ด

- `REPLACING ALL "-" BY " "` เปลี่ยนขีดกลาง (`-`) ทุกตำแหน่งใน `WS-PHONE` ให้เป็นช่องว่าง (` `)
  ทั้งสองค่ามีความยาว 1 ตัวอักษรเท่ากัน จึงใช้งานได้ปกติ

### ผลลัพธ์ที่ได้จากการรันจริง

```
Before: 081-234-5678
After : 081 234 5678
```

### ข้อควรระวัง

- ถ้าพยายามใช้ค่าเดิมและค่าใหม่ที่มีความยาวไม่เท่ากัน (เช่น `REPLACING ALL "-" BY "--"`) จะเกิด
  compile error ทันที เพราะ COBOL ตรวจสอบความยาวของทั้งสองค่าตั้งแต่ตอน compile
- `REPLACING ALL` เปลี่ยนแปลงข้อมูลต้นฉบับโดยตรง (ต่างจาก `TALLYING` ที่ไม่แก้ไขอะไรเลย) ควรระวัง
  ไม่ให้เผลอ `REPLACING` ทับข้อมูลต้นฉบับที่ยังต้องใช้งานในรูปแบบเดิมต่อไป

### แบบฝึกหัดที่ 203.1

**โจทย์**: จงแก้โค้ดข้างต้นให้แทนที่ขีดกลางด้วยเครื่องหมายจุด (`.`) แทนช่องว่าง

**เฉลย**: เปลี่ยน `REPLACING ALL "-" BY " "` เป็น `REPLACING ALL "-" BY "."`
ผลลัพธ์ที่ได้จะเป็น `081.234.5678`

---

## ขั้นตอนที่ 204: REPLACING FIRST และ REPLACING LEADING

### แนวคิด

บางครั้งเราไม่ต้องการแทนที่**ทุก**ตำแหน่งที่พบ COBOL จึงมีวลีเสริมอีกสองแบบ: **`REPLACING FIRST`**
จะแทนที่**เฉพาะตำแหน่งแรกสุด**ที่พบเท่านั้น แล้วหยุดค้นหาทันที ส่วน **`REPLACING LEADING`** จะแทนที่
เฉพาะ**ชุดตัวอักษรต่อเนื่องตั้งแต่ต้นฟิลด์**เท่านั้น (เหมือนหลักการเดียวกับ `TALLYING FOR LEADING`
ในขั้นตอนที่ 202)

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s4.cob
      *> Purpose : REPLACING FIRST (only the first match) versus
      *>           REPLACING LEADING (only a run at the start)
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-REPLACE-FIRST-LEADING.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEXT-1            PIC X(20) VALUE "AXAXAX".
       01  WS-TEXT-2            PIC X(20) VALUE "000123".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> REPLACING FIRST changes only the FIRST match found,
      *> scanning left to right, then stops looking entirely.
           DISPLAY "Before (FIRST) : " WS-TEXT-1
           INSPECT WS-TEXT-1 REPLACING FIRST "X" BY "-"
           DISPLAY "After  (FIRST) : " WS-TEXT-1

      *> REPLACING LEADING only changes a run of matches starting
      *> at the very beginning of the field; it stops as soon as
      *> a different character is seen.
           DISPLAY " "
           DISPLAY "Before (LEADING): " WS-TEXT-2
           INSPECT WS-TEXT-2 REPLACING LEADING "0" BY "*"
           DISPLAY "After  (LEADING): " WS-TEXT-2

           STOP RUN.
```

### คำอธิบายโค้ด

- `REPLACING FIRST "X" BY "-"` กับ "AXAXAX" จะเปลี่ยนแค่ตัว X ตัวแรก (ตำแหน่งที่ 2) เท่านั้น
  ตัว X ตัวที่สองและสามยังคงอยู่เหมือนเดิม
- `REPLACING LEADING "0" BY "*"` กับ "000123" จะเปลี่ยนเลข 0 สามตัวแรกที่ต่อเนื่องกันตั้งแต่ต้น
  เป็น "***" แต่ไม่แตะเลข 0 อื่นที่อาจอยู่ตรงกลางข้อความ (ในตัวอย่างนี้ไม่มีเลข 0 ตรงกลาง)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Before (FIRST) : AXAXAX
After  (FIRST) : A-AXAX

Before (LEADING): 000123
After  (LEADING): ***123
```

### ข้อควรระวัง

- `REPLACING LEADING` และ `TALLYING FOR LEADING` มีพฤติกรรม "หยุดเมื่อไม่ตรง" เหมือนกันทุกประการ
  เป็นกับดักที่พบบ่อยเมื่อผู้เขียนโปรแกรมคาดหวังว่ามันจะทำงานเหมือน `ALL` แต่จริง ๆ แล้วมันดูแค่ต้นข้อความ
  เท่านั้น ควรเลือกใช้วลีให้ตรงกับความต้องการเสมอ

### แบบฝึกหัดที่ 204.1

**โจทย์**: จงอธิบายว่าถ้าใช้ `REPLACING ALL "X" BY "-"` แทน `REPLACING FIRST` กับข้อความ "AXAXAX"
ในตัวอย่างข้างต้น ผลลัพธ์จะต่างกันอย่างไร

**เฉลย**: จะได้ผลลัพธ์เป็น "A-A-A-" คือตัว X ทุกตัว (ทั้งสามตัว) ถูกแทนที่ด้วย "-" ทั้งหมด
ต่างจาก `REPLACING FIRST` ที่เปลี่ยนแค่ตัวแรกตัวเดียวเป็น "A-AXAX"

---

## ขั้นตอนที่ 205: INSPECT CONVERTING - แปลงชุดตัวอักษรทั้งชุด

### แนวคิด

**`CONVERTING ชุดที่1 TO ชุดที่2`** ทำงานต่างจาก `REPLACING` ตรงที่มันไม่ได้มองหา "ข้อความ" ที่ตรงกัน
แต่จะ**จับคู่ตัวอักษรทีละตัวตามตำแหน่ง**ระหว่างสองชุด แล้วแปลงทุกตัวอักษรในฟิลด์ที่ตรงกับชุดที่ 1
ไปเป็นตัวอักษรที่ตำแหน่งเดียวกันในชุดที่ 2 ทันที นี่คือเครื่องมือที่เหมาะที่สุดสำหรับงานแปลงตัวพิมพ์
เล็ก-ใหญ่ (case conversion) หรือการปิดบังตัวอักษรบางประเภท (เช่น แปลงทุกตัวเลขเป็นเครื่องหมาย `*`)

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s5.cob
      *> Purpose : INSPECT CONVERTING - translate an entire set
      *>           of characters into another set, position by
      *>           position (like a lowercase-to-uppercase map)
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-CONVERTING.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME              PIC X(20) VALUE "john smith".
       01  WS-CODE              PIC X(10) VALUE "AB12CD34".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> CONVERTING maps each character in the FROM list to the
      *> character at the SAME position in the TO list. Every
      *> lowercase letter here maps to its uppercase counterpart.
           DISPLAY "Before: " WS-NAME
           INSPECT WS-NAME CONVERTING
               "abcdefghijklmnopqrstuvwxyz"
               TO "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           DISPLAY "After : " WS-NAME

      *> CONVERTING is also handy to mask out digits, by mapping
      *> every digit character to the same placeholder character.
           DISPLAY " "
           DISPLAY "Before: " WS-CODE
           INSPECT WS-CODE CONVERTING "0123456789" TO "**********"
           DISPLAY "After : " WS-CODE

           STOP RUN.
```

### คำอธิบายโค้ด

- `CONVERTING "abc...z" TO "ABC...Z"` จับคู่ตัวอักษร a↔A, b↔B, c↔C ไปเรื่อย ๆ ตามตำแหน่ง ทำให้ทุก
  ตัวพิมพ์เล็กใน `WS-NAME` ถูกแปลงเป็นตัวพิมพ์ใหญ่ในคำสั่งเดียว โดยไม่ต้องเขียน `REPLACING` ทีละตัวอักษร
  26 ครั้ง
- `CONVERTING "0123456789" TO "**********"` แปลงตัวเลขทุกตัวให้เป็นเครื่องหมาย `*` เหมือนกันหมด
  เพราะทุกตำแหน่งในชุดที่ 2 เป็น `*` ซ้ำกัน (ไม่จำเป็นต้องมีการจับคู่แบบ 1-ต่อ-1 ที่ไม่ซ้ำกันเสมอไป)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Before: john smith
After : JOHN SMITH

Before: AB12CD34
After : AB**CD**
```

### ข้อควรระวัง

- ชุดตัวอักษรทั้งสองฝั่ง (`FROM` และ `TO`) ต้องมี**ความยาวเท่ากันเสมอ** (ในตัวอย่างแรกคือ 26 ตัวทั้งคู่)
  มิฉะนั้นจะเกิด compile error
- `CONVERTING` เหมาะกับการแปลง "ชุด" ตัวอักษรทั้งหมดในคราวเดียว แต่ถ้าต้องการแปลงแค่ "ข้อความเฉพาะ"
  (เช่น คำใดคำหนึ่ง ไม่ใช่ทุกตัวอักษรที่ตรงกัน) ควรใช้ `REPLACING` แทน เพราะ `CONVERTING` จะแปลง
  ทุกตัวอักษรที่ตรงกับชุด FROM โดยไม่สนใจบริบทรอบข้างเลย

### แบบฝึกหัดที่ 205.1

**โจทย์**: จงเขียน `INSPECT CONVERTING` ที่แปลงตัวพิมพ์ใหญ่กลับเป็นตัวพิมพ์เล็ก (ทิศทางตรงข้ามกับ
ตัวอย่างแรก)

**เฉลย**:
```cobol
           INSPECT WS-NAME CONVERTING
               "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
               TO "abcdefghijklmnopqrstuvwxyz"
```
เพียงสลับตำแหน่งชุด FROM และ TO กัน ผลลัพธ์จะแปลงตัวพิมพ์ใหญ่ทั้งหมดกลับเป็นตัวพิมพ์เล็ก

---

## ขั้นตอนที่ 206: BEFORE และ AFTER - จำกัดขอบเขตการสแกน

### แนวคิด

ทั้ง `TALLYING`, `REPLACING` และ `CONVERTING` สามารถเติมวลี **`BEFORE ตัวคั่น`** หรือ **`AFTER ตัวคั่น`**
ต่อท้ายเพื่อ**จำกัดขอบเขต**ที่ COBOL จะสแกนหาได้ **`BEFORE`** จะหยุดการสแกนทันทีที่เจอตัวคั่นที่ระบุ
(ทำงานเฉพาะส่วนก่อนหน้าตัวคั่นเท่านั้น) ส่วน **`AFTER`** จะเริ่มสแกนก็ต่อเมื่อผ่านตัวคั่นที่ระบุไปแล้ว
เทคนิคนี้มีประโยชน์มากเมื่อข้อมูลมีหลายส่วนปนกัน แต่เราต้องการดำเนินการแค่ส่วนใดส่วนหนึ่งเท่านั้น

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s6.cob
      *> Purpose : BEFORE / AFTER phrases limit the region of
      *>           the field that TALLYING or REPLACING scans
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-BEFORE-AFTER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE              PIC X(30) VALUE "NAME:JOHN;DEPT:SALES".
       01  WS-COLON-COUNT       PIC 9 VALUE 0.
       01  WS-LINE-2            PIC X(30) VALUE "2026-09-26T10:15:00".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> AFTER "NAME:" makes INSPECT skip everything up to AND
      *> including "NAME:" before it starts counting colons.
           INSPECT WS-LINE TALLYING WS-COLON-COUNT
               FOR ALL ":" AFTER "NAME:"
           DISPLAY "Line           : " WS-LINE
           DISPLAY "Colons after 'NAME:' : " WS-COLON-COUNT

      *> BEFORE "T" makes REPLACING stop working once it reaches
      *> the "T" character, leaving everything from "T" onward
      *> untouched (protecting the time part from the date part's
      *> replacement of "-" with "/").
           DISPLAY " "
           DISPLAY "Before: " WS-LINE-2
           INSPECT WS-LINE-2 REPLACING ALL "-" BY "/" BEFORE "T"
           DISPLAY "After : " WS-LINE-2

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-LINE` มีค่า "NAME:JOHN;DEPT:SALES" ซึ่งมีเครื่องหมาย `:` อยู่สองตำแหน่ง (หลัง NAME และหลัง DEPT)
  แต่ `AFTER "NAME:"` ทำให้ COBOL เริ่มนับก็ต่อเมื่อผ่านคำว่า "NAME:" ไปแล้วเท่านั้น จึงนับได้แค่ 1
  (เฉพาะ `:` ที่อยู่หลัง DEPT)
- `WS-LINE-2` เป็นวันที่-เวลาแบบ ISO 8601 ที่มีเครื่องหมาย `-` ทั้งในส่วนวันที่และเครื่องหมาย `:` ในส่วน
  เวลา `BEFORE "T"` ทำให้ `REPLACING ALL "-" BY "/"` ทำงานเฉพาะส่วนวันที่ (ก่อนตัว "T") เท่านั้น
  ส่วนเวลาที่อยู่หลัง "T" จึงไม่ถูกแตะต้องเลย

### ผลลัพธ์ที่ได้จากการรันจริง

```
Line           : NAME:JOHN;DEPT:SALES
Colons after 'NAME:' : 1

Before: 2026-09-26T10:15:00
After : 2026/09/26T10:15:00
```

### ข้อควรระวัง

- `BEFORE`/`AFTER` ใช้ตัวคั่นตัวแรก**ที่พบ**เป็นจุดอ้างอิงเสมอ ถ้าข้อความมีตัวคั่นซ้ำหลายจุด ต้อง
  ตรวจสอบให้แน่ใจว่าตำแหน่งแรกที่พบตรงกับที่ต้องการจริง ๆ ไม่เช่นนั้นผลลัพธ์อาจไม่ตรงตามที่คาดหวัง
- `BEFORE` และ `AFTER` สามารถใช้ร่วมกันได้ในบางกรณี (เช่น `AFTER "A" BEFORE "B"` เพื่อจำกัดขอบเขตทั้ง
  จุดเริ่มต้นและจุดสิ้นสุด) แต่ควรทดสอบผลลัพธ์จริงเสมอเพราะการซ้อนเงื่อนไขอาจทำให้อ่านโค้ดเข้าใจยากขึ้น

### แบบฝึกหัดที่ 206.1

**โจทย์**: จงเขียน `INSPECT` ที่นับจำนวนเครื่องหมาย `:` ใน `WS-LINE-2` ("2026-09-26T10:15:00")
เฉพาะส่วน**หลัง**ตัว "T" เท่านั้น (คือเฉพาะในส่วนเวลา)

**เฉลย**:
```cobol
       01  WS-TIME-COLON-COUNT  PIC 9 VALUE 0.
      ...
           INSPECT WS-LINE-2 TALLYING WS-TIME-COLON-COUNT
               FOR ALL ":" AFTER "T"
```
ผลลัพธ์ที่ได้จะเป็น 2 เพราะส่วนเวลา "10:15:00" มีเครื่องหมาย `:` อยู่ 2 ตำแหน่ง

---

## ขั้นตอนที่ 207: รวม TALLYING และ REPLACING ในคำสั่งเดียว

### แนวคิด

`INSPECT` หนึ่งคำสั่งสามารถมีทั้ง `TALLYING` และ `REPLACING` พร้อมกันได้ โดยแต่ละส่วนทำงานอิสระจากกัน
แต่ถูกประมวลผลในการสแกนฟิลด์เพียงรอบเดียว ทำให้มีประสิทธิภาพดีกว่าการเรียก `INSPECT` แยกสองคำสั่ง
เมื่อทั้งสองงาน (นับและแทนที่) เกี่ยวข้องกับข้อมูลเดียวกัน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s7.cob
      *> Purpose : Combine TALLYING and REPLACING in a SINGLE
      *>           INSPECT statement, each with its own phrase
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-TALLY-AND-REPLACE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEXT              PIC X(30) VALUE "A,B,,C,D".
       01  WS-COMMA-COUNT       PIC 9 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> One INSPECT statement can both COUNT and REPLACE at the
      *> same time. Each phrase acts on its own, independently,
      *> but both are evaluated as COBOL scans left to right.
           DISPLAY "Before: " WS-TEXT

           INSPECT WS-TEXT
               TALLYING WS-COMMA-COUNT FOR ALL ","
               REPLACING ALL "," BY ";"

           DISPLAY "After : " WS-TEXT
           DISPLAY "Commas found (before replacing): "
               WS-COMMA-COUNT

           STOP RUN.
```

### คำอธิบายโค้ด

- `TALLYING WS-COMMA-COUNT FOR ALL ","` และ `REPLACING ALL "," BY ";"` อยู่ในคำสั่ง `INSPECT` เดียวกัน
  โดยไม่ต้องมีคำเชื่อมพิเศษใด ๆ แค่เขียนวลีทั้งสองต่อกันตามลำดับ
- ผลลัพธ์คือ `WS-TEXT` มีจุลภาคทั้งหมดถูกเปลี่ยนเป็นเซมิโคลอน **และ** `WS-COMMA-COUNT` เก็บจำนวน
  จุลภาคที่พบไว้ด้วยในคราวเดียว

### ผลลัพธ์ที่ได้จากการรันจริง

```
Before: A,B,,C,D
After : A;B;;C;D
Commas found (before replacing): 4
```

### ข้อควรระวัง

- ลำดับการเขียน `TALLYING` ก่อนหรือหลัง `REPLACING` ในคำสั่งเดียวกันไม่ส่งผลต่อผลลัพธ์สุดท้าย
  เพราะทั้งสองอ้างอิงจากข้อมูล**ต้นฉบับก่อนแก้ไข**เสมอในระหว่างการสแกนรอบเดียวกัน แต่เพื่อความอ่านง่าย
  นิยมเขียน `TALLYING` ก่อน `REPLACING` เป็นธรรมเนียม

### แบบฝึกหัดที่ 207.1

**โจทย์**: จงเพิ่มการนับจำนวนตัวอักษร "A" ในคำสั่ง `INSPECT` เดียวกันนี้ด้วย (นอกเหนือจากนับจุลภาค
และแทนที่จุลภาค)

**เฉลยแนวทาง**: เพิ่มตัวแปร `01 WS-A-COUNT PIC 9 VALUE 0.` แล้วเติมวลี
`TALLYING WS-A-COUNT FOR ALL "A"` ต่อจาก `TALLYING WS-COMMA-COUNT FOR ALL ","` ในคำสั่ง `INSPECT`
เดียวกัน (สามารถมี `TALLYING` ได้หลายชุดในคำสั่งเดียว) ผลลัพธ์จาก "A,B,,C,D" จะได้ `WS-A-COUNT` = 1

---

## ขั้นตอนที่ 208: ใช้ INSPECT ช่วยตัดสินใจก่อนเรียก UNSTRING

### แนวคิด

เทคนิคที่มีประโยชน์มากในทางปฏิบัติคือการใช้ `INSPECT TALLYING FOR ALL` เพื่อ**นับจำนวนตัวคั่น**ในข้อมูล
CSV ก่อนที่จะเรียก `UNSTRING` จริง เพราะจำนวนฟิลด์ที่จะได้จากการแยกเท่ากับ**จำนวนตัวคั่น + 1** เสมอ
การรู้จำนวนฟิลด์ล่วงหน้าช่วยให้เราตรวจสอบได้ว่าข้อมูลมีรูปแบบถูกต้องหรือไม่ก่อนที่จะพยายามแยกจริง
ซึ่งป้องกันปัญหา overflow ของ `UNSTRING` ที่เรียนใน Part 020 ได้อย่างเป็นระบบ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s8.cob
      *> Purpose : Use INSPECT TALLYING to count delimiters in a
      *>           CSV line BEFORE deciding how to UNSTRING it
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-BEFORE-UNSTRING.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CSV-LINE          PIC X(40) VALUE "RED,GREEN,BLUE,YELLOW".
       01  WS-COMMA-COUNT       PIC 9 VALUE 0.
       01  WS-FIELD-COUNT       PIC 9 VALUE 0.

       01  COLOR-TABLE.
           05  COLOR-NAME PIC X(10) OCCURS 6 TIMES.
       01  WS-IDX               PIC 9 VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Number of fields = number of commas + 1. Knowing this
      *> ahead of time lets us size a table or validate input
      *> BEFORE calling UNSTRING at all.
           INSPECT WS-CSV-LINE TALLYING WS-COMMA-COUNT
               FOR ALL ","
           COMPUTE WS-FIELD-COUNT = WS-COMMA-COUNT + 1

           DISPLAY "CSV line     : " WS-CSV-LINE
           DISPLAY "Comma count  : " WS-COMMA-COUNT
           DISPLAY "Field count  : " WS-FIELD-COUNT

           IF WS-FIELD-COUNT > 6
               DISPLAY "ERROR: more fields than the table can hold."
           ELSE
               UNSTRING WS-CSV-LINE DELIMITED BY ","
                   INTO COLOR-NAME(1), COLOR-NAME(2),
                        COLOR-NAME(3), COLOR-NAME(4),
                        COLOR-NAME(5), COLOR-NAME(6)
               END-UNSTRING
               DISPLAY "Parsed colors:"
               PERFORM VARYING WS-IDX FROM 1 BY 1
                       UNTIL WS-IDX > WS-FIELD-COUNT
                   DISPLAY "  " COLOR-NAME(WS-IDX)
               END-PERFORM
           END-IF

           STOP RUN.
```

### คำอธิบายโค้ด

- `COMPUTE WS-FIELD-COUNT = WS-COMMA-COUNT + 1` คือสูตรมาตรฐานในการแปลง "จำนวนตัวคั่น" เป็น
  "จำนวนฟิลด์" (ถ้ามี 3 ตัวคั่น จะได้ 4 ฟิลด์เสมอ ตราบใดที่ไม่มีตัวคั่นติดกันสองตัว)
- การตรวจสอบ `IF WS-FIELD-COUNT > 6` ก่อนเรียก `UNSTRING` เป็นการป้องกันปัญหาล่วงหน้า แทนที่จะปล่อยให้
  `UNSTRING` ทำงานแล้วค่อยตรวจ `ON OVERFLOW` ภายหลัง ทั้งสองวิธีใช้ร่วมกันได้เพื่อความปลอดภัยสูงสุด
- `WS-IDX FROM 1 BY 1 UNTIL WS-IDX > WS-FIELD-COUNT` ใช้จำนวนฟิลด์ที่คำนวณได้จริงมาควบคุมลูปแสดงผล
  แทนที่จะ hardcode จำนวนไว้ตายตัว ทำให้โค้ดยืดหยุ่นกับข้อมูลที่มีจำนวนฟิลด์ไม่แน่นอน

### ผลลัพธ์ที่ได้จากการรันจริง

```
CSV line     : RED,GREEN,BLUE,YELLOW
Comma count  : 3
Field count  : 4
Parsed colors:
  RED
  GREEN
  BLUE
  YELLOW
```

### ข้อควรระวัง

- สูตร "จำนวนตัวคั่น + 1 = จำนวนฟิลด์" ใช้ได้เฉพาะเมื่อไม่มีตัวคั่นติดกันสองตัว (เช่น ",,") ถ้าข้อมูลมี
  ฟิลด์ว่างติดกัน (เช่น "A,,C") สูตรนี้ยังคงถูกต้อง (นับ "" เป็นฟิลด์ว่างหนึ่งฟิลด์) แต่ผู้เขียนโปรแกรม
  ควรตระหนักว่าฟิลด์ว่างเป็นไปได้เสมอในข้อมูลจริง และควรออกแบบการตรวจสอบเพิ่มเติมถ้าฟิลด์ว่างไม่ควรเกิดขึ้น

### แบบฝึกหัดที่ 208.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบ `WS-FIELD-COUNT` ก่อนเรียก `UNSTRING` ถึงดีกว่าการปล่อยให้เกิด
`ON OVERFLOW` แล้วค่อยจัดการทีหลัง

**เฉลยแนวทาง**: การตรวจสอบล่วงหน้าทำให้โปรแกรมสามารถ**หลีกเลี่ยง**การเรียก `UNSTRING` ที่ไม่ปลอดภัย
ไปเลยตั้งแต่ต้น (defensive programming) แทนที่จะปล่อยให้มันทำงานบางส่วนแล้วค่อยรู้ทีหลังว่าข้อมูลเกิน
ซึ่งในบางกรณี `UNSTRING` อาจเขียนข้อมูลบางส่วนลงในฟิลด์ปลายทางไปแล้วก่อนจะรู้ตัวว่า overflow
การตรวจสอบก่อนจึงทำให้ควบคุมสถานการณ์ได้ชัดเจนกว่า และยังใช้ค่า `WS-FIELD-COUNT` ต่อประโยชน์อื่น
(เช่น ควบคุมลูปแสดงผล) ได้อีกด้วย

---

## ขั้นตอนที่ 209: การทำความสะอาดข้อมูล (Data Cleansing) ด้วย INSPECT ร่วมกับ FUNCTION TRIM

### แนวคิด

ในงานจริง ข้อมูลที่รับเข้ามาจากแหล่งภายนอก (แบบฟอร์มเว็บ, ไฟล์นำเข้า) มักไม่สะอาดพอที่จะนำไปใช้ตรง ๆ
เสมอ ขั้นตอนนี้จะสาธิตการผสมผสาน `INSPECT CONVERTING` (มาตรฐานตัวพิมพ์) ร่วมกับ `FUNCTION TRIM`
(ตัดช่องว่างส่วนเกิน จาก Part 019) เพื่อทำความสะอาดข้อมูลก่อนเก็บบันทึก และสาธิตการใช้ `CONVERTING`
ร่วมกับ `BEFORE` เพื่อ**ปิดบังข้อมูลอ่อนไหว** (data masking) เช่น เลขบัตรเครดิต ซึ่งเป็นความต้องการ
ที่พบบ่อยมากในระบบการเงิน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s9.cob
      *> Purpose : Data-cleansing example combining INSPECT with
      *>           FUNCTION TRIM - normalize case/spacing, then
      *>           mask sensitive digits using CONVERTING + BEFORE
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-DATA-CLEANSING.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW-INPUT         PIC X(30) VALUE "  bangkok  ".
       01  WS-CARD-NO           PIC X(16) VALUE "4111222233334444".

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Step 1: uppercase the raw customer input using CONVERTING,
      *> a common first move before comparing against a reference
      *> table (so "bangkok", "Bangkok" and "BANGKOK" all match).
           DISPLAY "Raw input      : [" WS-RAW-INPUT "]"
           INSPECT WS-RAW-INPUT CONVERTING
               "abcdefghijklmnopqrstuvwxyz"
               TO "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           DISPLAY "Uppercased     : [" WS-RAW-INPUT "]"

      *> FUNCTION TRIM (Part 019) then removes the leading and
      *> trailing spaces that CONVERTING does not touch, since
      *> CONVERTING only maps character VALUES, not positions.
           MOVE FUNCTION TRIM(WS-RAW-INPUT) TO WS-RAW-INPUT
           DISPLAY "Trimmed        : [" WS-RAW-INPUT "]"

      *> Step 2: mask a card number, keeping only the last 4
      *> digits visible. CONVERTING maps every digit to "*", and
      *> BEFORE limits that mapping to the part BEFORE the last
      *> four digits, so "4444" itself is left untouched.
           DISPLAY " "
           DISPLAY "Card number    : " WS-CARD-NO
           INSPECT WS-CARD-NO CONVERTING "0123456789" TO
               "**********" BEFORE "4444"
           DISPLAY "Masked         : " WS-CARD-NO

           STOP RUN.
```

### คำอธิบายโค้ด

- ขั้นแรก `CONVERTING` แปลงตัวพิมพ์เล็กเป็นใหญ่ แต่**ไม่แตะช่องว่าง**ที่นำหน้าและตามหลัง เพราะ
  `CONVERTING` ทำงานกับ**ค่า**ของตัวอักษรเท่านั้น ไม่ได้ลบตำแหน่งใด ๆ ออกจากฟิลด์
- `FUNCTION TRIM` จึงยังคงจำเป็นต้องใช้ร่วมด้วยเพื่อจัดการเรื่องช่องว่างส่วนเกิน — นี่คือตัวอย่างที่ดี
  ที่แสดงว่า `INSPECT` และ `FUNCTION` ต่างมีจุดแข็งคนละด้าน และมักถูกใช้ร่วมกันในงานทำความสะอาดข้อมูลจริง
- `CONVERTING "0123456789" TO "**********" BEFORE "4444"` แปลงเฉพาะตัวเลขที่อยู่**ก่อน**การปรากฏครั้ง
  แรกของ "4444" เท่านั้น ทำให้เลข 4 หลักสุดท้ายซึ่งเป็นส่วนที่อนุญาตให้แสดงยังคงอ่านได้ปกติ

### ผลลัพธ์ที่ได้จากการรันจริง

```
Raw input      : [  bangkok                     ]
Uppercased     : [  BANGKOK                     ]
Trimmed        : [BANGKOK                       ]

Card number    : 4111222233334444
Masked         : ************4444
```

### ข้อควรระวัง

- เทคนิค `CONVERTING ... BEFORE "ค่าสุดท้ายที่รู้อยู่แล้ว"` ใช้ได้ดีเมื่อรู้ค่าที่ต้องการเก็บไว้แน่นอน
  แต่ถ้าเลข 4 หลักสุดท้ายบังเอิญไปซ้ำกับตัวเลขชุดอื่นที่ปรากฏก่อนหน้าในฟิลด์เดียวกัน `BEFORE` จะยึด
  ตำแหน่ง**แรกที่พบ**เป็นหลักเสมอ ซึ่งอาจทำให้ปิดบังผิดตำแหน่งได้ ในระบบจริงที่ต้องการความแม่นยำสูง
  (เช่น ปิดบังเลขบัตรเครดิตจริง) ควรพิจารณาใช้ **Reference Modification** (`field(start:length)`
  จาก Part 020) ร่วมด้วยเพื่อระบุตำแหน่งที่แน่นอน แทนการพึ่งพา `BEFORE` เพียงอย่างเดียว
- ควรระวังเรื่องลำดับการดำเนินการ (แปลงตัวพิมพ์ก่อนหรือ trim ก่อน) ให้เหมาะกับข้อมูลจริง เพราะบางกรณี
  ลำดับที่ต่างกันอาจให้ผลลัพธ์เหมือนกัน แต่บางกรณีอาจไม่เหมือนกัน ควรทดสอบให้แน่ใจเสมอ

### แบบฝึกหัดที่ 209.1

**โจทย์**: จงอธิบายว่าทำไมขั้นตอนนี้ต้องเรียก `FUNCTION TRIM` **หลัง** `CONVERTING` ไม่ใช่ก่อน

**เฉลยแนวทาง**: จริง ๆ แล้วในกรณีนี้จะเรียกก่อนหรือหลังก็ให้ผลลัพธ์เหมือนกัน เพราะ `CONVERTING`
ไม่ได้เปลี่ยนตำแหน่งหรือจำนวนช่องว่างเลย มันแค่เปลี่ยนค่าตัวอักษรที่ไม่ใช่ช่องว่างเท่านั้น อย่างไรก็ตาม
การเขียนเรียงตามลำดับที่เป็นธรรมชาติของกระบวนการ (แปลงรูปแบบก่อน แล้วค่อยตัดส่วนเกินออก) ช่วยให้โค้ด
อ่านและเข้าใจง่ายกว่า และเป็นแนวทางที่ปลอดภัยกว่าในกรณีทั่วไปที่อาจมีการแปลงอื่นที่ซับซ้อนกว่านี้
ซึ่งอาจได้รับผลกระทบจากลำดับการดำเนินการได้

---

## ขั้นตอนที่ 210: สรุปข้อควรระวัง และแบบฝึกหัดรายงานคุณภาพข้อมูล

### สรุปข้อควรระวังสำคัญของ INSPECT

1. **`TALLYING` สะสมค่าต่อจากเดิมเสมอ** ต้องเคลียร์ตัวแปรนับเป็น 0 ก่อนเรียกใหม่ทุกครั้ง
2. **`REPLACING`/`CONVERTING` ต้องการค่าเดิมและค่าใหม่ที่มีความยาวเท่ากันเสมอ** ไม่สามารถใช้ `INSPECT`
   เพื่อลบหรือเพิ่มความยาวข้อมูลได้ ถ้าต้องการเปลี่ยนความยาว ต้องใช้ `STRING`/`UNSTRING` แทน
3. **`FOR LEADING` ต่างจาก `FOR ALL` อย่างมาก** — `LEADING` หยุดทันทีที่ไม่ตรงกับเงื่อนไข ในขณะที่
   `ALL` ตรวจทุกตำแหน่งในฟิลด์
4. **`BEFORE`/`AFTER` ยึดตำแหน่ง**แรกที่พบ**ของตัวคั่นเป็นหลักเสมอ** ต้องระวังกรณีที่ตัวคั่นซ้ำหลายจุด
5. **`CONVERTING` แปลงค่าตัวอักษรเท่านั้น ไม่แตะตำแหน่งช่องว่างหรือความยาว** มักต้องใช้ร่วมกับ
   `FUNCTION TRIM` เพื่อจัดการช่องว่างส่วนเกิน

### แบบฝึกหัดสรุปรวม: รายงานคุณภาพข้อมูล (Data Quality Report)

โจทย์: จงเขียนโปรแกรมที่รับข้อมูลอีเมล 3 รายการ (บางรายการมีช่องว่างส่วนเกิน บางรายการไม่มีเครื่องหมาย
`@` เลย) แล้วตรวจสอบว่าแต่ละรายการมีเครื่องหมาย `@` อยู่กี่ตัว หากมีพอดี 1 ตัว ให้ถือว่าถูกต้อง
แปลงเป็นตัวพิมพ์เล็กและตัดช่องว่างส่วนเกิน แล้วนับจำนวนอีเมลที่ถูกต้องทั้งหมด

### โค้ดเฉลย

```cobol
      *> ===================================================
      *> Program : s10.cob
      *> Purpose : Final summary - a small data-quality report
      *>           that counts, replaces, and converts across a
      *>           batch of raw customer records
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSPECT-QUALITY-REPORT.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  RAW-RECORD-TABLE.
           05  RAW-RECORD PIC X(30) OCCURS 3 TIMES.

       01  WS-IDX               PIC 9 VALUE 1.
       01  WS-SPACE-COUNT       PIC 9(2) VALUE 0.
       01  WS-AT-COUNT          PIC 9 VALUE 0.
       01  WS-TOTAL-EMAILS      PIC 9 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "john.smith@mail.com"   TO RAW-RECORD(1)
           MOVE "no-at-sign-here"       TO RAW-RECORD(2)
           MOVE "  mary@test.co  "      TO RAW-RECORD(3)

           DISPLAY "===== Data Quality Report ====="
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               MOVE 0 TO WS-AT-COUNT
               MOVE 0 TO WS-SPACE-COUNT

      *> TALLYING FOR ALL "@" tells us whether this record looks
      *> like a valid single-address email (exactly one "@").
               INSPECT RAW-RECORD(WS-IDX)
                   TALLYING WS-AT-COUNT FOR ALL "@"

      *> TALLYING FOR LEADING SPACE measures accidental leading
      *> blanks that a web form or file import often introduces.
               INSPECT RAW-RECORD(WS-IDX)
                   TALLYING WS-SPACE-COUNT FOR LEADING SPACE

               DISPLAY "Record " WS-IDX ": [" RAW-RECORD(WS-IDX) "]"
               DISPLAY "  '@' count    : " WS-AT-COUNT
               DISPLAY "  leading spaces: " WS-SPACE-COUNT

               IF WS-AT-COUNT = 1
                   MOVE FUNCTION TRIM(RAW-RECORD(WS-IDX))
                       TO RAW-RECORD(WS-IDX)
                   INSPECT RAW-RECORD(WS-IDX) CONVERTING
                       "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
                       TO "abcdefghijklmnopqrstuvwxyz"
                   ADD 1 TO WS-TOTAL-EMAILS
                   DISPLAY "  STATUS: valid email -> ["
                       RAW-RECORD(WS-IDX) "]"
               ELSE
                   DISPLAY "  STATUS: rejected (needs exactly one @)"
               END-IF
           END-PERFORM

           DISPLAY " "
           DISPLAY "Total valid emails: " WS-TOTAL-EMAILS

           STOP RUN.
```

### คำอธิบายโค้ด

- โปรแกรมนี้วนลูปผ่านรายการทั้งสาม ใช้ `TALLYING FOR ALL "@"` ตรวจสอบว่ามีเครื่องหมาย `@` พอดี 1 ตัว
  หรือไม่ (เกณฑ์ง่าย ๆ สำหรับความถูกต้องของอีเมล) และใช้ `TALLYING FOR LEADING SPACE` วัดจำนวน
  ช่องว่างนำหน้าเพื่อรายงานคุณภาพข้อมูล
- รายการที่ผ่านเกณฑ์ (`WS-AT-COUNT = 1`) จะถูกทำความสะอาดด้วย `FUNCTION TRIM` และแปลงเป็นตัวพิมพ์เล็ก
  ด้วย `CONVERTING` ก่อนนับเป็นอีเมลที่ใช้งานได้ ส่วนรายการที่ไม่ผ่านจะถูกปฏิเสธพร้อมข้อความแจ้งเหตุผล

### ผลลัพธ์ที่ได้จากการรันจริง

```
===== Data Quality Report =====
Record 1: [john.smith@mail.com           ]
  '@' count    : 1
  leading spaces: 00
  STATUS: valid email -> [john.smith@mail.com           ]
Record 2: [no-at-sign-here               ]
  '@' count    : 0
  leading spaces: 00
  STATUS: rejected (needs exactly one @)
Record 3: [  mary@test.co                ]
  '@' count    : 1
  leading spaces: 02
  STATUS: valid email -> [mary@test.co                  ]

Total valid emails: 2
```

### ข้อควรระวัง

- ตัวแปรนับ (`WS-AT-COUNT`, `WS-SPACE-COUNT`) ต้อง `MOVE 0` ให้ก่อนเริ่มแต่ละรอบของลูปเสมอ ตามที่
  เน้นย้ำในข้อควรระวังข้อ 1 ข้างต้น มิฉะนั้นผลนับของรายการก่อนหน้าจะปนกับรายการปัจจุบัน ซึ่งเป็นบั๊ก
  ที่พบบ่อยมากเมื่อใช้ `INSPECT TALLYING` ภายในลูป
- เกณฑ์ "มี @ พอดี 1 ตัว" เป็นการตรวจสอบแบบง่ายเท่านั้น ในระบบจริงควรตรวจสอบรูปแบบอีเมลที่ละเอียดกว่านี้
  (เช่น ต้องมีจุดหลัง @ ด้วย) แต่หลักการใช้ `INSPECT` ในการตรวจสอบเบื้องต้นแบบนี้ยังคงเป็นเทคนิคที่มีค่า
  มากสำหรับการกรองข้อมูลเสียก่อนที่จะทำการตรวจสอบที่ซับซ้อนกว่าต่อไป

### แบบฝึกหัดที่ 210.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น ให้ปฏิเสธอีเมลที่มีช่องว่างนำหน้ามากกว่า 5 ตัวด้วย (นอกเหนือจาก
เกณฑ์เรื่องจำนวน `@`)

**เฉลยแนวทาง**: แก้เงื่อนไข `IF WS-AT-COUNT = 1` เป็น
`IF WS-AT-COUNT = 1 AND WS-SPACE-COUNT <= 5` เพื่อให้ทั้งสองเงื่อนไขต้องเป็นจริงพร้อมกันจึงจะถือว่า
เป็นอีเมลที่ใช้งานได้ ด้วยข้อมูลตัวอย่างนี้ผลลัพธ์จะไม่เปลี่ยนแปลง (เพราะไม่มีรายการใดมีช่องว่างนำหน้า
เกิน 5 ตัว) แต่หลักการนี้จะสำคัญมากขึ้นเมื่อข้อมูลจริงมีความไม่สม่ำเสมอมากกว่านี้

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้คำสั่ง `INSPECT` สำหรับนับ แทนที่ และแปลงตัวอักษรอย่างครบถ้วน ได้แก่:

- ภาพรวมสามความสามารถหลักของ `INSPECT`: `TALLYING`, `REPLACING`, และ `CONVERTING`
- `TALLYING FOR ALL` สำหรับนับทุกตำแหน่งที่พบ และ `FOR LEADING`/`FOR CHARACTERS` สำหรับกรณีพิเศษ
- `REPLACING ALL` สำหรับแทนที่ทุกตำแหน่ง และ `REPLACING FIRST`/`LEADING` สำหรับแทนที่บางส่วน
- `CONVERTING` สำหรับแปลงชุดตัวอักษรทั้งชุดในคำสั่งเดียว (เช่น แปลงตัวพิมพ์เล็ก-ใหญ่)
- `BEFORE`/`AFTER` สำหรับจำกัดขอบเขตการสแกนให้อยู่แค่บางส่วนของฟิลด์
- การรวม `TALLYING` และ `REPLACING` ในคำสั่งเดียวเพื่อประสิทธิภาพที่ดีขึ้น
- การใช้ `INSPECT` ช่วยตัดสินใจก่อนเรียก `UNSTRING` โดยนับจำนวนตัวคั่นล่วงหน้า
- การทำความสะอาดข้อมูล (data cleansing) จริงด้วย `INSPECT` ร่วมกับ `FUNCTION TRIM` และเทคนิคปิดบัง
  ข้อมูลอ่อนไหวด้วย `CONVERTING ... BEFORE`
- สรุปข้อควรระวังทั้งหมด พร้อมแบบฝึกหัดสร้างรายงานคุณภาพข้อมูล

ตอนนี้เรามีเครื่องมือจัดการข้อความครบทั้งสามตัวแล้ว: `STRING` (รวม), `UNSTRING` (แยก), และ `INSPECT`
(นับ/แทนที่/แปลง) ใน **Part 022** เราจะเปลี่ยนโฟกัสไปที่การจัดการ**หน่วยความจำ**ผ่าน **`REDEFINES`
Clause** ซึ่งเป็นเทคนิคสำคัญที่ทำให้เรามองข้อมูลชุดเดียวกันในหน่วยความจำเป็นได้หลายรูปแบบพร้อมกัน
เทคนิคนี้พบได้บ่อยมากในระบบ Legacy ที่ต้องประหยัดพื้นที่หน่วยความจำ และในการประมวลผล record
ที่มีหลายรูปแบบปนกันในไฟล์เดียว

**[← กลับไป Part 020](part-020-unstring-statement.md)** |
**[ไปยัง Part 022: REDEFINES Clause และการใช้หน่วยความจำซ้ำ →](part-022-redefines.md)**
