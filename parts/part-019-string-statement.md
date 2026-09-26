# Part 019: การจัดการข้อความ: STRING Statement (ขั้นตอนที่ 181–190)

## คำนำของ Part นี้

ใน Part 018 เราเรียนรู้การจัดการข้อมูลที่มีโครงสร้างซับซ้อนผ่านตารางหลายมิติ คราวนี้เราจะเปลี่ยนโฟกัส
มาที่การจัดการ**ข้อความ (Text/String)** ซึ่งเป็นทักษะที่สำคัญไม่แพ้กันในงานธุรกิจจริง ไม่ว่าจะเป็นการรวมชื่อ-นามสกุล
เข้าด้วยกัน การประกอบที่อยู่เต็มรูปแบบจากฟิลด์ย่อย หรือการสร้างบรรทัดรายงานที่มีทั้งข้อความและตัวเลขปนกัน

COBOL มีคำสั่งเฉพาะสำหรับงานเหล่านี้ชื่อว่า **`STRING`** ซึ่งทำหน้าที่ **รวมข้อความจากหลายแหล่งเข้าเป็นฟิลด์เดียว**
Part นี้จะพาคุณเรียนรู้ตั้งแต่พื้นฐานที่สุดของ `STRING` ไปจนถึงเทคนิคขั้นสูงอย่าง `WITH POINTER` และ `ON OVERFLOW`
พร้อมกับข้อผิดพลาดที่พบบ่อยที่สุดที่ผู้เริ่มต้นมักเจอ เพื่อให้คุณเขียนโค้ดจัดการข้อความได้อย่างมั่นใจ
ก่อนที่ Part 020 เราจะเรียนรู้คำสั่งตรงข้ามกันคือ `UNSTRING` ที่ใช้แยกข้อความออกเป็นส่วนย่อย

---

## ขั้นตอนที่ 181: ทำไมต้องมี STRING - ข้อจำกัดของ MOVE ในการรวมข้อความ

### แนวคิด

หลาย ๆ คนที่เริ่มเขียน COBOL ใหม่ ๆ มักคิดว่าสามารถใช้ `MOVE` หลาย ๆ ครั้งเพื่อ "รวม" ข้อความจากหลายฟิลด์
เข้าเป็นฟิลด์เดียวได้ แต่ความจริงแล้ว `MOVE` แต่ละครั้งจะ**เขียนทับ**ค่าที่ `MOVE` ไปแล้วก่อนหน้าทั้งหมด
ไม่ได้ต่อท้ายกันแต่อย่างใด นี่คือสาเหตุที่ COBOL ต้องมีคำสั่งพิเศษชื่อ `STRING` ที่ออกแบบมาเฉพาะสำหรับการ
**ต่อ (concatenate)** ข้อความจากหลายแหล่งเข้าด้วยกัน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s1.cob
      *> Purpose : Why MOVE alone cannot combine several fields
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. WHY-STRING-NEEDED.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST-NAME        PIC X(10) VALUE "JOHN".
       01  WS-LAST-NAME         PIC X(10) VALUE "SMITH".
       01  WS-FULL-NAME         PIC X(21).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> A single MOVE only copies ONE source. The second MOVE
      *> below simply overwrites the first: it does NOT combine.
           MOVE WS-FIRST-NAME TO WS-FULL-NAME
           MOVE WS-LAST-NAME TO WS-FULL-NAME
           DISPLAY "After two MOVEs, only the last one survives:"
           DISPLAY "[" WS-FULL-NAME "]"

           DISPLAY " "
           DISPLAY "We need a statement that joins fields together"
           DISPLAY "instead of overwriting them. That is STRING."

           STOP RUN.
```

### คำอธิบายโค้ด

- `MOVE WS-FIRST-NAME TO WS-FULL-NAME` เขียน "JOHN" (พร้อมช่องว่างเติมท้าย) ลงใน `WS-FULL-NAME` ทั้งหมด
- `MOVE WS-LAST-NAME TO WS-FULL-NAME` เขียนทับ "SMITH" ลงไปแทนที่ "JOHN" ที่เพิ่งใส่ไปทั้งหมด
  ไม่ได้ต่อท้ายชื่อแรกแต่อย่างใด

### ผลลัพธ์ที่ได้จากการรันจริง

```
After two MOVEs, only the last one survives:
[SMITH                ]
 
We need a statement that joins fields together
instead of overwriting them. That is STRING.
```

### ข้อควรระวัง

- นี่เป็นความเข้าใจผิดที่พบบ่อยที่สุดของผู้เริ่มต้น COBOL: `MOVE` ไม่ใช่การต่อข้อความ (concatenation)
  แต่เป็นการเขียนทับ (overwrite) เสมอ

### แบบฝึกหัดที่ 181.1

**โจทย์**: จงลองนึกสถานการณ์จริงอีก 2 กรณีในงานธุรกิจที่ต้องการ "รวมข้อความจากหลายฟิลด์" เข้าด้วยกัน

**เฉลยแนวทาง**: (1) การสร้างเลขที่ใบเสร็จจากรหัสสาขา + ปี + เลขลำดับ เช่น "BKK-2026-000123"
(2) การสร้างที่อยู่เต็มรูปแบบสำหรับพิมพ์ซองจดหมายจากฟิลด์ ถนน, เขต, จังหวัด, รหัสไปรษณีย์ที่แยกเก็บไว้คนละฟิลด์

---

## ขั้นตอนที่ 182: STRING พื้นฐานด้วย DELIMITED BY SIZE

### แนวคิด

รูปแบบพื้นฐานที่สุดของ `STRING` คือ `STRING แหล่งที่มา-1 แหล่งที่มา-2 ... DELIMITED BY SIZE INTO ปลายทาง`
คำว่า `DELIMITED BY SIZE` หมายความว่า **ให้คัดลอกข้อมูลทั้งหมดของแหล่งที่มานั้นตามขนาดจริง (SIZE)**
รวมถึงช่องว่างที่เติมท้าย (trailing spaces) ถ้าแหล่งที่มาเป็นฟิลด์ที่มีขนาดคงที่

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s2.cob
      *> Purpose : Basic STRING with DELIMITED BY SIZE
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-BASIC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-GREETING          PIC X(40).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> DELIMITED BY SIZE means: take the WHOLE literal/field,
      *> including any trailing spaces it has.
           STRING "HELLO" " " "COBOL" " " "WORLD" "!"
               DELIMITED BY SIZE
               INTO WS-GREETING
           END-STRING

           DISPLAY "Result: [" WS-GREETING "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `STRING` รับรายการของแหล่งที่มาได้หลายตัวพร้อมกัน (ในที่นี้คือ literal ข้อความ 6 ตัว)
- `DELIMITED BY SIZE` ใช้ตัวเดียวควบคุมทุกแหล่งที่มาที่อยู่ก่อนหน้ามันได้ (ไม่ต้องระบุซ้ำทุกตัวถ้าใช้กติกาเดียวกันหมด)
- `INTO WS-GREETING` ระบุฟิลด์ปลายทางที่จะเก็บผลลัพธ์ที่รวมกันแล้ว
- `END-STRING` คือ scope terminator ปิดคำสั่ง (มาตรฐานตั้งแต่ COBOL-85)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Result: [HELLO COBOL WORLD!                      ]
```

### ข้อควรระวัง

- ฟิลด์ปลายทาง (`WS-GREETING`) **ไม่ได้ถูกเคลียร์เป็นช่องว่างก่อน** โดยอัตโนมัติเมื่อเรียก `STRING`
  ในตัวอย่างนี้ใช้ได้เพราะเป็นการรันครั้งแรกและฟิลด์เริ่มต้นเป็นช่องว่างอยู่แล้ว แต่ถ้าเรียก `STRING`
  ซ้ำหลายครั้งกับฟิลด์เดิมโดยไม่ล้างค่าก่อน อาจมีตัวอักษรเก่าตกค้างอยู่ท้ายฟิลด์ (จะอธิบายเพิ่มในขั้นตอนที่ 190)

### แบบฝึกหัดที่ 182.1

**โจทย์**: จงแก้โค้ดข้างต้นให้แสดงข้อความ "COBOL IS FUN!" แทน โดยยังคงใช้ `DELIMITED BY SIZE`

**เฉลย**:
```cobol
           STRING "COBOL" " " "IS" " " "FUN" "!"
               DELIMITED BY SIZE
               INTO WS-GREETING
           END-STRING
```

---

## ขั้นตอนที่ 183: STRING ด้วย DELIMITED BY SPACE เพื่อตัดช่องว่างท้ายฟิลด์

### แนวคิด

ปัญหาที่พบบ่อยของ `DELIMITED BY SIZE` คือมันจะคัดลอก**ช่องว่างที่เติมท้าย**ของฟิลด์ที่มีขนาดคงที่ไปด้วยเสมอ
เช่น ถ้า `WS-FIRST-NAME PIC X(10)` เก็บค่า "JOHN" จริง ๆ จะมีช่องว่างอีก 6 ตัวตามหลัง เมื่อใช้ `DELIMITED BY SIZE`
ช่องว่างเหล่านั้นจะถูกคัดลอกไปด้วย ทำให้ผลลัพธ์มีช่องว่างแทรกอยู่ตรงกลางแบบไม่สวยงาม วิธีแก้คือใช้
**`DELIMITED BY SPACE`** ซึ่งจะหยุดคัดลอกทันทีที่เจอ**ช่องว่างตัวแรก**ของแหล่งที่มานั้น

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s3.cob
      *> Purpose : STRING DELIMITED BY SPACE to trim trailing spaces
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-DELIM-SPACE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST-NAME        PIC X(10) VALUE "JOHN".
       01  WS-LAST-NAME         PIC X(10) VALUE "SMITH".
       01  WS-FULL-NAME         PIC X(21) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> DELIMITED BY SIZE keeps trailing spaces from PIC X(10),
      *> which is usually NOT what we want for names. Compare:
           STRING WS-FIRST-NAME DELIMITED BY SIZE
                  " "           DELIMITED BY SIZE
                  WS-LAST-NAME  DELIMITED BY SIZE
               INTO WS-FULL-NAME
           END-STRING
           DISPLAY "DELIMITED BY SIZE  : [" WS-FULL-NAME "]"

      *> DELIMITED BY SPACE stops copying that source as soon as
      *> the first space is reached, trimming the padding.
           MOVE SPACES TO WS-FULL-NAME
           STRING WS-FIRST-NAME DELIMITED BY SPACE
                  " "           DELIMITED BY SIZE
                  WS-LAST-NAME  DELIMITED BY SPACE
               INTO WS-FULL-NAME
           END-STRING
           DISPLAY "DELIMITED BY SPACE : [" WS-FULL-NAME "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- แต่ละแหล่งที่มาใน `STRING` สามารถกำหนด `DELIMITED BY` ของตัวเองแยกกันได้ ไม่จำเป็นต้องใช้กติกาเดียวกันหมดทุกตัว
- ในตัวอย่างที่สอง `WS-FIRST-NAME DELIMITED BY SPACE` หยุดคัดลอกทันทีที่เจอช่องว่างตัวแรก (หลัง "JOHN")
  ในขณะที่ `" " DELIMITED BY SIZE` (literal ช่องว่างตรงกลาง) ยังคงคัดลอกตามขนาดปกติ (คือ 1 ช่องว่าง)

### ผลลัพธ์ที่ได้จากการรันจริง

```
DELIMITED BY SIZE  : [JOHN       SMITH     ]
DELIMITED BY SPACE : [JOHN SMITH           ]
```

### ข้อควรระวัง

- **กับดักสำคัญ**: `DELIMITED BY SPACE` หยุดที่ช่องว่าง**ตัวแรกที่เจอ** ถ้าข้อมูลจริงมีช่องว่างอยู่ตรงกลาง
  (เช่น "MARY ANN" หรือ "123 MAIN STREET") มันจะถูกตัดครึ่งโดยไม่ตั้งใจ! เราจะเห็นปัญหานี้ชัดเจนในขั้นตอนที่ 184

### แบบฝึกหัดที่ 183.1

**โจทย์**: ถ้า `WS-FIRST-NAME` มีค่า "MARY ANN" (มีช่องว่างตรงกลาง) แล้วใช้ `DELIMITED BY SPACE`
ผลลัพธ์ที่ได้ในฟิลด์ปลายทางจะเป็นอย่างไร

**เฉลย**: จะได้เพียง "MARY" เท่านั้น เพราะ `DELIMITED BY SPACE` หยุดทันทีที่เจอช่องว่างระหว่าง "MARY" กับ "ANN"
ส่วน "ANN" จะไม่ถูกคัดลอกไปด้วยเลย นี่คือเหตุผลที่ต้องระวังการใช้ `DELIMITED BY SPACE` กับฟิลด์ที่อาจมีช่องว่างภายใน

---

## ขั้นตอนที่ 184: STRING จากหลายฟิลด์ต้นทาง และกับดักช่องว่างภายในฟิลด์

### แนวคิด

ขั้นตอนนี้จะแสดงตัวอย่างที่ใหญ่ขึ้น: การประกอบที่อยู่เต็มรูปแบบจากฟิลด์ย่อยหลายตัว พร้อมเผยกับดักสำคัญที่ต่อ
จากขั้นตอนก่อนหน้า: เมื่อฟิลด์ต้นทางมีช่องว่างอยู่ **ภายใน** ข้อมูลจริงของมันเอง (เช่น "123 MAIN STREET")
การใช้ `DELIMITED BY SPACE` จะตัดข้อมูลขาดกลางคัน วิธีแก้ที่ถูกต้องคือใช้ **`FUNCTION TRIM`**
เพื่อตัดเฉพาะช่องว่างที่เติมท้าย (padding) ออกก่อน แล้วค่อยใช้ `DELIMITED BY SIZE` กับค่าที่ตัดแล้ว

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s4.cob
      *> Purpose : STRING several source fields into one field,
      *>           and a common trap when a field has an
      *>           embedded space (like "123 MAIN STREET")
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-MULTI-SOURCE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STREET            PIC X(20) VALUE "123 MAIN STREET".
       01  WS-CITY              PIC X(15) VALUE "SPRINGFIELD".
       01  WS-STATE             PIC X(2)  VALUE "IL".
       01  WS-ZIP               PIC X(5)  VALUE "62704".
       01  WS-ADDRESS-WRONG     PIC X(60) VALUE SPACES.
       01  WS-ADDRESS-RIGHT     PIC X(60) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> TRAP: DELIMITED BY SPACE stops at the FIRST space it
      *> finds. WS-STREET has an embedded space ("123 MAIN
      *> STREET"), so this only copies "123" and stops there!
           STRING WS-STREET DELIMITED BY SPACE
                  ", "      DELIMITED BY SIZE
                  WS-CITY   DELIMITED BY SPACE
                  ", "      DELIMITED BY SIZE
                  WS-STATE  DELIMITED BY SIZE
                  " "       DELIMITED BY SIZE
                  WS-ZIP    DELIMITED BY SIZE
               INTO WS-ADDRESS-WRONG
           END-STRING
           DISPLAY "Wrong  : [" WS-ADDRESS-WRONG "]"

      *> FIX: use FUNCTION TRIM to remove trailing padding first,
      *> then DELIMITED BY SIZE copies the trimmed value exactly,
      *> embedded spaces and all.
           STRING FUNCTION TRIM(WS-STREET) DELIMITED BY SIZE
                  ", "                     DELIMITED BY SIZE
                  FUNCTION TRIM(WS-CITY)   DELIMITED BY SIZE
                  ", "                     DELIMITED BY SIZE
                  WS-STATE                 DELIMITED BY SIZE
                  " "                      DELIMITED BY SIZE
                  WS-ZIP                   DELIMITED BY SIZE
               INTO WS-ADDRESS-RIGHT
           END-STRING
           DISPLAY "Correct: [" WS-ADDRESS-RIGHT "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-STREET` มีค่า "123 MAIN STREET" ซึ่งมีช่องว่าง**ภายใน**ข้อมูลจริงถึง 2 จุด
- เวอร์ชัน "Wrong" ใช้ `DELIMITED BY SPACE` กับ `WS-STREET` โดยตรง ทำให้หยุดคัดลอกแค่ "123" เท่านั้น
- เวอร์ชัน "Correct" ใช้ `FUNCTION TRIM(WS-STREET)` ก่อน ซึ่งจะตัดเฉพาะช่องว่างส่วนเกินที่ **หัวและท้าย**
  ของฟิลด์ทั้งหมดออก (ไม่แตะช่องว่างภายใน) แล้วจึงใช้ `DELIMITED BY SIZE` กับผลลัพธ์ที่ได้ ซึ่งจะคัดลอก
  ข้อความทั้งหมดของค่าที่ TRIM แล้วอย่างถูกต้อง รวมถึงช่องว่างภายในด้วย

### ผลลัพธ์ที่ได้จากการรันจริง

```
Wrong  : [123, SPRINGFIELD, IL 62704                                  ]
Correct: [123 MAIN STREET, SPRINGFIELD, IL 62704                      ]
```

### ข้อควรระวัง

- นี่คือกับดักที่พบบ่อยที่สุดอันดับต้น ๆ ของ `STRING`: การใช้ `DELIMITED BY SPACE` กับฟิลด์ที่มีช่องว่าง
  **ภายใน** ข้อมูลจริง (ชื่อ-นามสกุลที่มีช่องว่างคั่น, ที่อยู่ที่มีหลายคำ) จะทำให้ข้อมูลขาดหายไปอย่างเงียบ ๆ
  โดยไม่มี error ใด ๆ เตือน — โปรแกรมจะ compile ผ่านและรันได้ปกติ แต่ผลลัพธ์ผิด
- กฎง่าย ๆ ในการเลือก: ถ้าฟิลด์ต้นทางอาจมีช่องว่างภายในข้อมูลจริง ให้ใช้ `FUNCTION TRIM(...)` ร่วมกับ
  `DELIMITED BY SIZE` เสมอ แทนที่จะใช้ `DELIMITED BY SPACE` โดยตรงกับฟิลด์นั้น

### แบบฝึกหัดที่ 184.1

**โจทย์**: จงอธิบายว่าทำไม `WS-STATE` และ `WS-ZIP` ในโค้ดข้างต้นถึงใช้ `DELIMITED BY SIZE` ได้โดยตรง
โดยไม่ต้องใช้ `FUNCTION TRIM`

**เฉลยแนวทาง**: เพราะ `WS-STATE PIC X(2)` (ค่า "IL") และ `WS-ZIP PIC X(5)` (ค่า "62704") ถูกกำหนดขนาด
ให้พอดีกับข้อมูลจริงเป๊ะ ไม่มีช่องว่างเหลือให้เติมท้าย จึงไม่มีปัญหาเรื่องช่องว่างส่วนเกินหรือช่องว่างภายใน
การใช้ `DELIMITED BY SIZE` ตรง ๆ จึงปลอดภัยในกรณีนี้

---

## ขั้นตอนที่ 185: STRING WITH POINTER เพื่อควบคุมตำแหน่งเริ่มเขียนและต่อข้อมูลภายหลัง

### แนวคิด

ปกติ `STRING` จะเริ่มเขียนที่ตำแหน่งแรกสุดของฟิลด์ปลายทางเสมอ แต่ถ้าเราต้องการควบคุมว่าจะ**เริ่มเขียนตรงไหน**
หรือต้องการ **`STRING` ต่อจากตำแหน่งที่เขียนไปแล้วในคำสั่งก่อนหน้า** เราใช้วลี **`WITH POINTER ตัวแปร`**
ตัวแปร pointer นี้ต้องเป็นตัวเลข และ COBOL จะ**อัปเดตค่าของมันโดยอัตโนมัติ**ให้ชี้ไปยังตำแหน่งถัดจาก
ตัวอักษรตัวสุดท้ายที่เพิ่งเขียนไป ทำให้เราเรียก `STRING` ซ้ำหลายครั้งเพื่อ "ต่อ" ข้อมูลเข้าไปในฟิลด์เดียวกันได้

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s5.cob
      *> Purpose : STRING WITH POINTER to control the start
      *>           position, and to continue building later
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-POINTER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RESULT            PIC X(40) VALUE SPACES.
       01  WS-PTR               PIC 9(3) VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> WS-PTR tells STRING where to begin writing, and STRING
      *> updates it to just past the last character written.
           STRING "PART-1" DELIMITED BY SIZE
               INTO WS-RESULT
               WITH POINTER WS-PTR
           END-STRING
           DISPLAY "After first STRING, pointer = " WS-PTR
           DISPLAY "Result so far: [" WS-RESULT "]"

      *> Because we reused WS-PTR, this STRING continues writing
      *> right after "PART-1" instead of overwriting it.
           STRING "-PART-2" DELIMITED BY SIZE
               INTO WS-RESULT
               WITH POINTER WS-PTR
           END-STRING
           DISPLAY "After second STRING, pointer = " WS-PTR
           DISPLAY "Result now: [" WS-RESULT "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-PTR VALUE 1` เริ่มต้นชี้ที่ตำแหน่งที่ 1 (ตัวอักษรแรกสุด) ของ `WS-RESULT`
- หลัง `STRING` ตัวแรกเขียน "PART-1" (6 ตัวอักษร) เสร็จ `WS-PTR` จะถูกอัปเดตเป็น 7 โดยอัตโนมัติ
  (ตำแหน่งถัดจากตัวอักษรตัวสุดท้ายที่เขียนไป)
- `STRING` ตัวที่สองใช้ `WS-PTR` เดิม (ค่า 7) จึงเริ่มเขียน "-PART-2" ต่อจากตำแหน่งที่ 7 ทันที
  ไม่ได้เขียนทับ "PART-1" ที่มีอยู่แล้ว

### ผลลัพธ์ที่ได้จากการรันจริง

```
After first STRING, pointer = 007
Result so far: [PART-1                                  ]
After second STRING, pointer = 014
Result now: [PART-1-PART-2                           ]
```

### ข้อควรระวัง

- ตัวแปร pointer ต้องเป็นตัวเลขที่ **ไม่มีตำแหน่งทศนิยม** (integer) และมีขนาดใหญ่พอที่จะรองรับตำแหน่งสูงสุด
  ที่เป็นไปได้ของฟิลด์ปลายทาง เช่นฟิลด์ขนาด 40 ตัวอักษรควรใช้ `PIC 9(3)` เป็นอย่างน้อย
- ถ้าลืมตั้งค่าตัวแปร pointer กลับไปที่ 1 ก่อนเริ่ม `STRING` ชุดใหม่ (ที่ไม่เกี่ยวกับชุดก่อนหน้า)
  ข้อมูลจะเขียนต่อจากตำแหน่งเดิมโดยไม่ตั้งใจ (จะอธิบายเพิ่มเป็นข้อควรระวังหลักในขั้นตอนที่ 190)

### แบบฝึกหัดที่ 185.1

**โจทย์**: จงเพิ่ม `STRING` ตัวที่สามต่อจากโค้ดข้างต้น เพื่อเติม "-DONE" ต่อท้าย และแสดงผลลัพธ์สุดท้าย

**เฉลย**:
```cobol
           STRING "-DONE" DELIMITED BY SIZE
               INTO WS-RESULT
               WITH POINTER WS-PTR
           END-STRING
           DISPLAY "Final result: [" WS-RESULT "]"
```
ผลลัพธ์ที่ได้จะเป็น `Final result: [PART-1-PART-2-DONE ...]` (มีช่องว่างเติมท้ายตามขนาดฟิลด์ 40 ตัวอักษร)

---

## ขั้นตอนที่ 186: ON OVERFLOW - เมื่อฟิลด์ปลายทางเล็กเกินไป

### แนวคิด

ถ้าฟิลด์ปลายทางมีขนาดเล็กเกินกว่าจะรองรับข้อมูลทั้งหมดที่ `STRING` พยายามเขียนลงไป (นับจากตำแหน่งที่ pointer
ชี้อยู่) จะเกิดสถานการณ์ที่เรียกว่า **overflow** ในกรณีนี้ COBOL จะ **หยุดเขียนทันทีที่เต็มพื้นที่** และข้อมูล
ส่วนที่เกินจะหายไป (ไม่ error ทันที) เราสามารถตรวจจับสถานการณ์นี้ได้ด้วยวลี **`ON OVERFLOW`**
และ **`NOT ON OVERFLOW`**

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s6.cob
      *> Purpose : ON OVERFLOW clause when the target is too small
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-OVERFLOW.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SMALL-FIELD       PIC X(10) VALUE SPACES.
       01  WS-PTR               PIC 9(3) VALUE 8.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> WS-SMALL-FIELD only has 10 bytes. Starting at position 8
      *> leaves just 3 bytes, but we try to write more than that.
           STRING "OVERFLOW-TEST" DELIMITED BY SIZE
               INTO WS-SMALL-FIELD
               WITH POINTER WS-PTR
               ON OVERFLOW
                   DISPLAY "OVERFLOW detected! Not enough room."
               NOT ON OVERFLOW
                   DISPLAY "No overflow, everything fit."
           END-STRING

           DISPLAY "Field content: [" WS-SMALL-FIELD "]"
           DISPLAY "Pointer ended at: " WS-PTR

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-SMALL-FIELD PIC X(10)` มีความยาวแค่ 10 ตัวอักษร และ `WS-PTR` เริ่มต้นที่ตำแหน่ง 8 (เหลือที่แค่ 3 ตัวอักษร)
- ข้อความ "OVERFLOW-TEST" ยาวถึง 13 ตัวอักษร ซึ่งเกินพื้นที่ที่เหลืออยู่มาก จึงเกิด overflow
- `ON OVERFLOW` จะทำงานเมื่อเขียนไม่ครบ ส่วน `NOT ON OVERFLOW` จะทำงานเมื่อเขียนครบพอดีโดยไม่ล้น
  ทั้งสองวลีนี้ใช้แทนกันไม่ได้ (mutually exclusive) — จะทำงานแค่อันใดอันหนึ่งเท่านั้นในแต่ละครั้งที่เรียก

### ผลลัพธ์ที่ได้จากการรันจริง

```
OVERFLOW detected! Not enough room.
Field content: [       OVE]
Pointer ended at: 011
```

### ข้อควรระวัง

- เมื่อเกิด overflow ข้อมูลจะถูกเขียนเท่าที่พื้นที่เหลืออนุญาต (ในตัวอย่างนี้คือ "OVE" ที่ตำแหน่ง 8-10)
  แล้ว **หยุดทันที** โดยไม่มีข้อผิดพลาดร้ายแรง (runtime error) ใด ๆ ถ้าไม่ตรวจสอบด้วย `ON OVERFLOW`
  โปรแกรมจะเดินหน้าต่อไปพร้อมข้อมูลที่ขาดหายไปแบบเงียบ ๆ ซึ่งอันตรายมากในระบบการเงิน
- ควรใส่ `ON OVERFLOW` ทุกครั้งที่ไม่แน่ใจว่าฟิลด์ปลายทางมีขนาดเพียงพอเสมอ โดยเฉพาะเมื่อข้อมูลต้นทาง
  มาจากภายนอก (เช่น ไฟล์หรือผู้ใช้ป้อน) ที่ความยาวไม่แน่นอน

### แบบฝึกหัดที่ 186.1

**โจทย์**: จงแก้โค้ดข้างต้นโดยเปลี่ยนค่าเริ่มต้นของ `WS-PTR` เป็น 1 แทน 8 แล้วคาดเดาว่าจะเกิด overflow หรือไม่

**เฉลย**: จะไม่เกิด overflow เพราะเริ่มเขียนที่ตำแหน่ง 1 มีพื้นที่เหลือเต็ม 10 ตัวอักษร แต่ข้อความ
"OVERFLOW-TEST" ยาว 13 ตัวอักษรก็ยังคงเกินพื้นที่ 10 ตัวอักษรอยู่ดี ดังนั้น**ยังคงเกิด overflow เหมือนเดิม**
เพียงแต่คราวนี้จะเขียนได้ 10 ตัวอักษรแรกคือ "OVERFLOW-T" ก่อนจะหยุด ข้อคิดคือ overflow ขึ้นอยู่กับทั้งขนาด
ฟิลด์ปลายทางและตำแหน่งเริ่มต้นของ pointer ร่วมกัน ไม่ใช่แค่ขนาดฟิลด์อย่างเดียว

---

## ขั้นตอนที่ 187: ใช้ STRING ในลูปเพื่อสร้างบรรทัด CSV จากตาราง

### แนวคิด

เทคนิคที่มีประโยชน์มากคือการใช้ `STRING WITH POINTER` ภายในลูป `PERFORM` เพื่อสร้างบรรทัดข้อความยาว ๆ
จากข้อมูลในตาราง เช่น การแปลงตารางชื่อผลไม้ให้เป็นบรรทัด CSV (comma-separated values) ที่คั่นด้วยจุลภาค
ซึ่งเป็นรูปแบบไฟล์ที่ใช้กันแพร่หลายในการแลกเปลี่ยนข้อมูลระหว่างระบบ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s7.cob
      *> Purpose : Build a CSV line from a table using STRING
      *>           inside a loop, accumulating with a pointer
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-BUILD-CSV.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  FRUIT-TABLE.
           05  FRUIT-NAME PIC X(10) OCCURS 4 TIMES.

       01  WS-CSV-LINE          PIC X(60) VALUE SPACES.
       01  WS-PTR               PIC 9(3) VALUE 1.
       01  WS-IDX               PIC 9 VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE "APPLE"  TO FRUIT-NAME(1)
           MOVE "BANANA" TO FRUIT-NAME(2)
           MOVE "CHERRY" TO FRUIT-NAME(3)
           MOVE "DURIAN" TO FRUIT-NAME(4)

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 4
               IF WS-IDX > 1
                   STRING "," DELIMITED BY SIZE
                       INTO WS-CSV-LINE
                       WITH POINTER WS-PTR
                   END-STRING
               END-IF
               STRING FRUIT-NAME(WS-IDX) DELIMITED BY SPACE
                   INTO WS-CSV-LINE
                   WITH POINTER WS-PTR
               END-STRING
           END-PERFORM

           DISPLAY "CSV line: [" WS-CSV-LINE "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- ในลูป เราตรวจสอบก่อนว่า `WS-IDX > 1` หรือไม่ ถ้าใช่ (คือไม่ใช่รายการแรก) ให้เติมจุลภาค `,` ก่อนเสมอ
  เทคนิคนี้ป้องกันไม่ให้มีจุลภาคเกินที่ต้นหรือท้ายบรรทัด
- `WS-PTR` ถูกใช้ร่วมกันตลอดทั้งลูป ทำให้แต่ละครั้งที่เรียก `STRING` จะต่อท้ายจากตำแหน่งล่าสุดเสมอ
  โดยไม่ต้องคำนวณตำแหน่งเองด้วยมือ

### ผลลัพธ์ที่ได้จากการรันจริง

```
CSV line: [APPLE,BANANA,CHERRY,DURIAN                                  ]
```

### ข้อควรระวัง

- ต้องเคลียร์ `WS-CSV-LINE` เป็น `SPACES` และตั้ง `WS-PTR` กลับเป็น 1 ก่อนเริ่มลูปใหม่ทุกครั้ง
  ถ้าโปรแกรมนี้ถูกเรียกซ้ำ (เช่น อยู่ใน paragraph ที่ถูก `PERFORM` หลายรอบ) มิฉะนั้นบรรทัด CSV เก่าจะปนกับใหม่

### แบบฝึกหัดที่ 187.1

**โจทย์**: จงแก้โค้ดข้างต้นให้ใช้เครื่องหมาย `|` (pipe) แทนจุลภาคเป็นตัวคั่น

**เฉลย**: เปลี่ยนบรรทัด `STRING "," DELIMITED BY SIZE` เป็น `STRING "|" DELIMITED BY SIZE` เพียงจุดเดียว
ที่เหลือไม่ต้องแก้ไขอะไรเพิ่มเติม ผลลัพธ์จะได้ `[APPLE|BANANA|CHERRY|DURIAN ...]`

---

## ขั้นตอนที่ 188: เปรียบเทียบ STRING กับ FUNCTION CONCATENATE

### แนวคิด

COBOL สมัยใหม่ (COBOL-2002 เป็นต้นไป) มี intrinsic function ชื่อ `FUNCTION CONCATENATE` ที่ใช้ต่อข้อความ
ได้เช่นกัน แต่มีข้อจำกัดคือ**ไม่มีวลี `DELIMITED BY`** ให้เลือกแบบ `STRING` มันจะต่อข้อมูลตามขนาดจริง (SIZE)
ของแต่ละอาร์กิวเมนต์เสมอ เหมาะกับกรณีที่ฟิลด์ต้นทางถูก `FUNCTION TRIM` มาเรียบร้อยแล้วและต้องการโค้ดที่กระชับ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s8.cob
      *> Purpose : Compare STRING with FUNCTION CONCATENATE
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-VS-CONCAT-FUNC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST             PIC X(6)  VALUE "GOOD".
       01  WS-SECOND            PIC X(6)  VALUE "BYE".
       01  WS-RESULT-STRING     PIC X(20) VALUE SPACES.
       01  WS-RESULT-FUNC       PIC X(20) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Method 1: the STRING statement (flexible, many delimiters)
           STRING WS-FIRST DELIMITED BY SPACE
                  WS-SECOND DELIMITED BY SPACE
               INTO WS-RESULT-STRING
           END-STRING

      *> Method 2: FUNCTION CONCATENATE (simple, no delimiters,
      *> good for a quick one-line join of already-clean fields)
           MOVE FUNCTION CONCATENATE(
               FUNCTION TRIM(WS-FIRST)
               FUNCTION TRIM(WS-SECOND))
               TO WS-RESULT-FUNC

           DISPLAY "STRING result     : [" WS-RESULT-STRING "]"
           DISPLAY "CONCATENATE result: [" WS-RESULT-FUNC "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `FUNCTION CONCATENATE(...)` รับอาร์กิวเมนต์หลายตัวและต่อกันตรง ๆ โดยไม่มีการเลือก delimiter ใด ๆ
  จึงต้อง `FUNCTION TRIM` แต่ละอาร์กิวเมนต์เองก่อนเสมอถ้าไม่ต้องการช่องว่างส่วนเกิน
- ทั้งสองวิธีให้ผลลัพธ์เหมือนกันในกรณีนี้ แต่ `STRING` มีความยืดหยุ่นมากกว่ามากเมื่อโจทย์ซับซ้อนขึ้น
  (เช่น ต้องการ delimiter ต่างกันในแต่ละแหล่งที่มา หรือต้องการ `ON OVERFLOW`)

### ผลลัพธ์ที่ได้จากการรันจริง

```
STRING result     : [GOODBYE             ]
CONCATENATE result: [GOODBYE             ]
```

### ข้อควรระวัง

- `FUNCTION CONCATENATE` เหมาะกับงานง่าย ๆ ที่ต่อข้อความไม่กี่ตัวและไม่ต้องการ `DELIMITED BY` หรือ `ON OVERFLOW`
  แต่สำหรับงานที่ซับซ้อน (หลายแหล่งที่มา, ต้องการ pointer, ต้องการตรวจ overflow) `STRING` ยังคงเป็นตัวเลือกที่ดีกว่า

### แบบฝึกหัดที่ 188.1

**โจทย์**: จงบอกเหตุผล 2 ข้อว่าทำไมในระบบที่ต้องรวมข้อความจากมากกว่า 5 แหล่ง และต้องการ delimiter
คนละแบบในแต่ละแหล่ง ควรเลือกใช้ `STRING` มากกว่า `FUNCTION CONCATENATE`

**เฉลยแนวทาง**: (1) `STRING` อนุญาตให้กำหนด `DELIMITED BY` แยกกันในแต่ละแหล่งที่มา ทำให้ควบคุมการตัดข้อมูล
ได้ละเอียดกว่า ในขณะที่ `FUNCTION CONCATENATE` ต้อง `FUNCTION TRIM` ทุกอาร์กิวเมนต์ก่อนเสมอซึ่งทำให้โค้ดยาวขึ้น
เมื่อมีหลายแหล่งที่มา (2) `STRING` รองรับ `ON OVERFLOW` เพื่อตรวจจับกรณีฟิลด์ปลายทางเล็กเกินไป
ในขณะที่ `FUNCTION CONCATENATE` ไม่มีกลไกตรวจสอบนี้ในตัวเอง

---

## ขั้นตอนที่ 189: ใช้ STRING สร้างบรรทัดรายงานที่มีข้อความและตัวเลขปนกัน

### แนวคิด

ในรายงานธุรกิจจริง บรรทัดหนึ่งมักประกอบด้วยทั้งข้อความ (ชื่อสินค้า) และตัวเลขที่จัดรูปแบบแล้ว (numeric-edited)
เช่น จำนวนสินค้าพร้อมเครื่องหมายจุลภาค หรือราคาพร้อมสัญลักษณ์เงิน `STRING` สามารถรวมทั้งสองแบบเข้าด้วยกันได้
โดยต้อง `MOVE` ตัวเลขไปยังฟิลด์ numeric-edited (PICTURE ที่มี `Z`, `$`, `,` เป็นต้น) ก่อน แล้วจึงนำฟิลด์นั้นมา
`STRING` เป็นข้อความตามปกติ

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s9.cob
      *> Purpose : Use STRING to build a formatted report line
      *>           combining text and numeric-edited fields
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-REPORT-LINE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRODUCT           PIC X(12) VALUE "USB CABLE".
       01  WS-QTY               PIC 9(4)  VALUE 25.
       01  WS-QTY-EDIT          PIC ZZZ9.
       01  WS-PRICE             PIC 9(5)V99 VALUE 129.50.
       01  WS-PRICE-EDIT        PIC $$,$$9.99.
       01  WS-REPORT-LINE       PIC X(50) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           MOVE WS-QTY   TO WS-QTY-EDIT
           MOVE WS-PRICE TO WS-PRICE-EDIT

           STRING WS-PRODUCT    DELIMITED BY SPACE
                  " QTY:"       DELIMITED BY SIZE
                  WS-QTY-EDIT   DELIMITED BY SIZE
                  " PRICE:"     DELIMITED BY SIZE
                  WS-PRICE-EDIT DELIMITED BY SIZE
               INTO WS-REPORT-LINE
           END-STRING

           DISPLAY "Report line: [" WS-REPORT-LINE "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-QTY-EDIT PIC ZZZ9` และ `WS-PRICE-EDIT PIC $$,$$9.99` เป็นฟิลด์ numeric-edited ที่จัดรูปแบบตัวเลข
  ให้อ่านง่ายขึ้น (`Z` แทนเลขศูนย์นำหน้าด้วยช่องว่าง, `$` และ `,` แสดงสัญลักษณ์เงินและตัวคั่นหลักพัน)
- เมื่อ `MOVE` ตัวเลขดิบไปยังฟิลด์ edit แล้ว เราสามารถนำฟิลด์ edit นั้นมาใช้ใน `STRING` ได้เหมือนฟิลด์ข้อความทั่วไป
  โดยใช้ `DELIMITED BY SIZE` เพราะฟิลด์ edit มีความยาวคงที่ตาม PICTURE ที่กำหนดไว้พอดี

### ผลลัพธ์ที่ได้จากการรันจริง

```
Report line: [USB QTY:  25 PRICE:  $129.50                      ]
```

### ข้อควรระวัง

- ต้อง `MOVE` ค่าไปยังฟิลด์ edit **ก่อน** เรียก `STRING` เสมอ ห้ามใส่ตัวแปรตัวเลขดิบ (เช่น `WS-QTY PIC 9(4)`)
  ลงใน `STRING` โดยตรงถ้าต้องการรูปแบบที่จัดสวยแล้ว เพราะ `STRING` จะคัดลอกเฉพาะตัวเลขดิบตามที่เก็บจริง
  (มีเลขศูนย์นำหน้าเต็มจำนวนหลักตาม PICTURE) ไม่ได้จัดรูปแบบให้อัตโนมัติ

### แบบฝึกหัดที่ 189.1

**โจทย์**: จงทดลองเปลี่ยน `WS-QTY-EDIT` จาก `PIC ZZZ9` ไปใช้ตัวแปรดิบ `WS-QTY PIC 9(4)` ใน `STRING` โดยตรง
แล้วคาดเดาผลลัพธ์ที่จะได้

**เฉลย**: ผลลัพธ์จะเป็น `... QTY:0025 PRICE: ...` คือมีเลขศูนย์นำหน้า "0025" แทนที่จะเป็น "  25" ที่จัดรูปแบบสวยงาม
เพราะฟิลด์ดิบ `PIC 9(4)` เก็บค่าด้วยเลขศูนย์นำหน้าเต็มความยาวเสมอ ไม่มีการแปลงเป็นช่องว่างให้อัตโนมัติ

---

## ขั้นตอนที่ 190: ข้อควรระวังสรุปรวม และแบบฝึกหัดสร้างป้ายจ่าหน้าจดหมาย

### สรุปข้อควรระวังสำคัญของ STRING

1. **`MOVE` ไม่ใช่การต่อข้อความ** ต้องใช้ `STRING` เสมอเมื่อต้องการรวมหลายฟิลด์เข้าเป็นฟิลด์เดียว
2. **`DELIMITED BY SPACE` อันตรายกับฟิลด์ที่มีช่องว่างภายใน** ให้ใช้ `FUNCTION TRIM` ร่วมกับ `DELIMITED BY SIZE` แทน
3. **ต้องเคลียร์ฟิลด์ปลายทางเป็น `SPACES` ก่อน `STRING` ใหม่ทุกครั้ง** ที่ไม่ใช่การต่อท้ายของเดิมโดยตั้งใจ
   มิฉะนั้นตัวอักษรเก่าจะตกค้างปนกับข้อมูลใหม่
4. **ต้องตั้งตัวแปร pointer กลับเป็น 1 ก่อนเริ่ม `STRING` ชุดใหม่** ที่ไม่ต่อเนื่องจากชุดก่อนหน้า
   มิฉะนั้นข้อมูลจะเริ่มเขียนจากตำแหน่งเดิมที่ค้างอยู่โดยไม่ตั้งใจ
5. **ควรใส่ `ON OVERFLOW` เมื่อความยาวข้อมูลไม่แน่นอน** โดยเฉพาะเมื่อข้อมูลมาจากภายนอกระบบ

### แบบฝึกหัดสรุปรวม: สร้างป้ายจ่าหน้าจดหมาย

โจทย์: จงเขียนโปรแกรมที่รับข้อมูลชื่อ ที่อยู่ (ที่มีช่องว่างภายใน) เมือง รัฐ และรหัสไปรษณีย์ที่แยกเก็บไว้
คนละฟิลด์ แล้วประกอบเป็นป้ายจ่าหน้าจดหมาย 3 บรรทัด โดยระวังทั้งกับดักช่องว่างภายในฟิลด์ (ขั้นตอนที่ 184)
และกับดักเรื่องไม่ได้เคลียร์ฟิลด์/pointer ก่อนเริ่มใหม่ (ขั้นตอนนี้)

### โค้ดเฉลย

```cobol
      *> ===================================================
      *> Program : s10.cob
      *> Purpose : Exercise solution - build a full mailing label
      *>           from separate fields, showing common pitfalls
      *>           fixed (clearing target, resetting pointer)
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STRING-MAILING-LABEL.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME              PIC X(20) VALUE "JANE DOE".
       01  WS-STREET            PIC X(20) VALUE "99 PARK AVENUE".
       01  WS-CITY              PIC X(15) VALUE "RIVERSIDE".
       01  WS-STATE             PIC X(2)  VALUE "CA".
       01  WS-ZIP               PIC X(5)  VALUE "92501".

       01  WS-LABEL-LINE-1      PIC X(30) VALUE SPACES.
       01  WS-LABEL-LINE-2      PIC X(30) VALUE SPACES.
       01  WS-LABEL-LINE-3      PIC X(30) VALUE SPACES.
       01  WS-PTR               PIC 9(3).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Pitfall 1: forgetting to reset the target field before
      *> a fresh STRING leaves old characters mixed with new
      *> ones. We always start from SPACES.
      *> Pitfall 3: WS-NAME and WS-STREET have embedded spaces
      *> ("JANE DOE", "99 PARK AVENUE"), so DELIMITED BY SPACE
      *> would wrongly cut them at the first space. We use
      *> FUNCTION TRIM + DELIMITED BY SIZE instead.
           MOVE SPACES TO WS-LABEL-LINE-1
           STRING FUNCTION TRIM(WS-NAME) DELIMITED BY SIZE
               INTO WS-LABEL-LINE-1
           END-STRING

      *> Pitfall 2: forgetting to reset the pointer to 1 before
      *> reusing it in a new STRING makes writing start in the
      *> middle of the field instead of the beginning.
           MOVE SPACES TO WS-LABEL-LINE-2
           MOVE 1 TO WS-PTR
           STRING FUNCTION TRIM(WS-STREET) DELIMITED BY SIZE
               INTO WS-LABEL-LINE-2
               WITH POINTER WS-PTR
           END-STRING

           MOVE SPACES TO WS-LABEL-LINE-3
           MOVE 1 TO WS-PTR
           STRING WS-CITY  DELIMITED BY SPACE
                  ", "     DELIMITED BY SIZE
                  WS-STATE DELIMITED BY SIZE
                  " "      DELIMITED BY SIZE
                  WS-ZIP   DELIMITED BY SIZE
               INTO WS-LABEL-LINE-3
               WITH POINTER WS-PTR
           END-STRING

           DISPLAY "----- MAILING LABEL -----"
           DISPLAY WS-LABEL-LINE-1
           DISPLAY WS-LABEL-LINE-2
           DISPLAY WS-LABEL-LINE-3

           STOP RUN.
```

### คำอธิบายโค้ด

- บรรทัดที่ 1 (ชื่อ) และบรรทัดที่ 2 (ถนน) ใช้ `FUNCTION TRIM` ป้องกันปัญหาช่องว่างภายในตามที่เรียนในขั้นตอนที่ 184
- บรรทัดที่ 3 (เมือง, รัฐ, รหัสไปรษณีย์) ใช้ `DELIMITED BY SPACE` ได้อย่างปลอดภัย เพราะแต่ละฟิลด์ย่อย
  ไม่มีช่องว่างภายในตัวมันเอง
- ทุกบรรทัดเคลียร์ฟิลด์ปลายทางเป็น `SPACES` ก่อนเสมอ และบรรทัดที่ใช้ pointer ก็ตั้งค่ากลับเป็น 1 ก่อนเริ่มใหม่

### ผลลัพธ์ที่ได้จากการรันจริง

```
----- MAILING LABEL -----
JANE DOE                      
99 PARK AVENUE                
RIVERSIDE, CA 92501           
```

### ข้อควรระวัง

- สังเกตว่าบรรทัดที่ 1 ไม่ได้ใช้ `WITH POINTER` เพราะเป็นการ `STRING` แหล่งเดียวลงฟิลด์เดียวโดยไม่ต่อจากที่ใด
  จึงไม่จำเป็นต้องมี pointer เลย (`STRING` จะเริ่มเขียนที่ตำแหน่ง 1 โดยปริยายเมื่อไม่ระบุ `WITH POINTER`)
  การใช้ pointer ควรสงวนไว้เฉพาะกรณีที่ต้องเรียก `STRING` มากกว่าหนึ่งครั้งต่อฟิลด์ปลายทางเดียวเท่านั้น

### แบบฝึกหัดที่ 190.1

**โจทย์**: จงต่อยอดโปรแกรมป้ายจ่าหน้าจดหมายข้างต้น ให้รวมทั้ง 3 บรรทัดเข้าเป็นฟิลด์เดียวขนาดใหญ่
คั่นแต่ละบรรทัดด้วยเครื่องหมาย `/` แทนการ `DISPLAY` แยก 3 บรรทัด

**เฉลยแนวทาง**: ประกาศฟิลด์ใหม่ `01 WS-FULL-LABEL PIC X(95) VALUE SPACES.` และ `01 WS-PTR2 PIC 9(3) VALUE 1.`
จากนั้นใช้ `STRING` สามครั้งต่อเนื่องกัน โดยแต่ละครั้งใช้ `WS-LABEL-LINE-n DELIMITED BY SPACE` ตามด้วย
`"/" DELIMITED BY SIZE` (ยกเว้นครั้งสุดท้ายไม่ต้องเติม `/`) พร้อม `WITH POINTER WS-PTR2` ทุกครั้งเพื่อให้
ต่อท้ายกันถูกต้อง ระวังว่า `WS-LABEL-LINE-1` มีค่า "JANE DOE" ไม่มีช่องว่างภายในเหลือให้กังวลอีกแล้ว
เพราะผ่านการ `FUNCTION TRIM` มาตั้งแต่ตอนสร้างแล้ว จึงใช้ `DELIMITED BY SPACE` ได้อย่างปลอดภัยในขั้นตอนนี้

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้คำสั่ง `STRING` สำหรับจัดการข้อความอย่างครบถ้วน ได้แก่:

- เหตุผลที่ต้องมี `STRING` เพราะ `MOVE` ไม่สามารถต่อข้อความจากหลายฟิลด์ได้
- การใช้ `DELIMITED BY SIZE` เพื่อคัดลอกข้อมูลเต็มขนาด
- การใช้ `DELIMITED BY SPACE` เพื่อตัดช่องว่างท้ายฟิลด์ และกับดักเมื่อฟิลด์มีช่องว่างภายใน
- การแก้กับดักช่องว่างภายในด้วย `FUNCTION TRIM` ร่วมกับ `DELIMITED BY SIZE`
- การใช้ `WITH POINTER` เพื่อควบคุมตำแหน่งเริ่มเขียน และต่อข้อมูลในหลายคำสั่ง `STRING`
- การตรวจจับข้อมูลล้นด้วย `ON OVERFLOW` และ `NOT ON OVERFLOW`
- การสร้างบรรทัด CSV จากตารางด้วย `STRING` ในลูป
- การเปรียบเทียบ `STRING` กับ `FUNCTION CONCATENATE`
- การรวมข้อความกับตัวเลขที่จัดรูปแบบแล้ว (numeric-edited) ในบรรทัดรายงาน
- สรุปข้อควรระวังทั้งหมด พร้อมแบบฝึกหัดสร้างป้ายจ่าหน้าจดหมายแบบครบวงจร

ทักษะเรื่อง `STRING` นี้จะเป็นพื้นฐานสำคัญสำหรับ **Part 020** ที่เราจะเรียนรู้คำสั่งตรงข้ามกันคือ
**`UNSTRING`** ซึ่งใช้แยกข้อความก้อนใหญ่ (เช่น บรรทัด CSV ที่เพิ่งสร้างในบทนี้) ออกเป็นฟิลด์ย่อย ๆ
ทำให้เราสามารถทำงานทั้งสองทิศทาง (รวมและแยก) ของการจัดการข้อความได้อย่างสมบูรณ์

**[← กลับไป Part 018](part-018-multidim-tables.md)** |
**[ไปยัง Part 020: การจัดการข้อความ - UNSTRING Statement →](part-020-unstring-statement.md)**
