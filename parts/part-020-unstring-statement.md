# Part 020: การจัดการข้อความ: UNSTRING Statement (ขั้นตอนที่ 191–200)

## คำนำของ Part นี้

ใน Part 019 เราเรียนรู้คำสั่ง `STRING` สำหรับ**รวม**ข้อความจากหลายฟิลด์เข้าเป็นฟิลด์เดียว
Part นี้เราจะเรียนรู้คำสั่งที่ทำงาน**ตรงกันข้าม**อย่างสิ้นเชิง นั่นคือ **`UNSTRING`** ซึ่งใช้**แยก**ข้อความก้อนใหญ่
ก้อนเดียวออกเป็นฟิลด์ย่อยหลายฟิลด์ ตามตัวคั่น (delimiter) ที่กำหนด

คำสั่งนี้สำคัญมากในงานประมวลผลข้อมูลจริง เพราะข้อมูลที่รับเข้ามาจากภายนอกระบบ (ไฟล์ CSV, ข้อความจาก
ระบบอื่น, ข้อมูลที่ผู้ใช้ป้อน) มักมาในรูปแบบข้อความก้อนเดียวที่ต้อง**แยกส่วน**ก่อนนำไปประมวลผลต่อ เช่น
การแยกวันที่ "2026-09-26" ออกเป็นปี เดือน วัน หรือการแยกบรรทัด CSV ออกเป็นฟิลด์ต่าง ๆ ตามจุลภาค
Part นี้จะพาคุณเรียนรู้ตั้งแต่การใช้งานพื้นฐานไปจนถึงเทคนิคขั้นสูงอย่าง `WITH POINTER`, `TALLYING IN`
และ `ON OVERFLOW` ซึ่งเป็นคู่หูของ `STRING` ที่เราเรียนไปแล้ว

---

## ขั้นตอนที่ 191: UNSTRING คือด้านตรงข้ามของ STRING

### แนวคิด

ถ้า `STRING` คือการ**รวม**หลายฟิลด์เข้าเป็นหนึ่งเดียว `UNSTRING` ก็คือการ**แยก**หนึ่งฟิลด์ออกเป็นหลายฟิลด์
โดยอาศัย**ตัวคั่น (delimiter)** เป็นเกณฑ์ในการตัดแบ่ง รูปแบบพื้นฐานคือ
`UNSTRING แหล่งที่มา DELIMITED BY ตัวคั่น INTO ปลายทาง-1, ปลายทาง-2, ...`

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s1.cob
      *> Purpose : Introduce UNSTRING as the reverse of STRING
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. WHY-UNSTRING-NEEDED.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FULL-DATE         PIC X(10) VALUE "2026-09-26".
       01  WS-YEAR              PIC X(4).
       01  WS-MONTH             PIC X(2).
       01  WS-DAY               PIC X(2).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> STRING combines several fields into one.
      *> UNSTRING does the opposite: it SPLITS one field
      *> into several, based on a delimiter character.
           UNSTRING WS-FULL-DATE DELIMITED BY "-"
               INTO WS-YEAR, WS-MONTH, WS-DAY
           END-UNSTRING

           DISPLAY "Full date : " WS-FULL-DATE
           DISPLAY "Year      : " WS-YEAR
           DISPLAY "Month     : " WS-MONTH
           DISPLAY "Day       : " WS-DAY

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-FULL-DATE` เก็บวันที่ในรูปแบบข้อความก้อนเดียว "2026-09-26"
- `UNSTRING WS-FULL-DATE DELIMITED BY "-"` บอกให้ COBOL ตัดแบ่งข้อความทุกครั้งที่เจอเครื่องหมาย `-`
- `INTO WS-YEAR, WS-MONTH, WS-DAY` ระบุฟิลด์ปลายทางเรียงตามลำดับที่จะรับส่วนที่ถูกตัดแบ่งแต่ละส่วน
  (เครื่องหมายจุลภาคระหว่างชื่อฟิลด์เป็นทางเลือก ใส่หรือไม่ใส่ก็ได้ ไม่มีผลต่อการทำงาน)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Full date : 2026-09-26
Year      : 2026
Month     : 09
Day       : 26
```

### ข้อควรระวัง

- `UNSTRING` ไม่ได้ตรวจสอบว่าข้อมูลที่แยกออกมาถูกต้องตามความหมายหรือไม่ (เช่น เดือน "13" ก็จะถูกแยกออกมาได้
  โดยไม่มี error) การตรวจสอบความถูกต้องของข้อมูลยังคงเป็นหน้าที่ของโปรแกรมเมอร์เสมอ

### แบบฝึกหัดที่ 191.1

**โจทย์**: จงยกตัวอย่างข้อมูลอีก 2 กรณีในงานธุรกิจจริงที่มักต้องใช้ `UNSTRING` แยกข้อความออกเป็นส่วนย่อย

**เฉลยแนวทาง**: (1) แยกชื่อ-นามสกุลเต็มที่เก็บในฟิลด์เดียว (เช่น "JOHN SMITH") ออกเป็นชื่อจริงกับนามสกุล
(2) แยกข้อมูลบรรทัดในไฟล์ log ที่คั่นด้วยช่องว่างหรือ pipe ออกเป็น timestamp, ระดับความรุนแรง (severity),
และข้อความ (message) เพื่อนำไปวิเคราะห์ต่อ

---

## ขั้นตอนที่ 192: UNSTRING พื้นฐานสำหรับแยกข้อมูล CSV

### แนวคิด

การใช้งานที่พบบ่อยที่สุดของ `UNSTRING` คือการแยกบรรทัดข้อมูลแบบ CSV (comma-separated values)
ออกเป็นฟิลด์ต่าง ๆ ตามจุลภาคที่คั่นอยู่ ขั้นตอนนี้จะสาธิตการใช้งานพื้นฐานที่สุดกับข้อมูล CSV จริง

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s2.cob
      *> Purpose : Basic UNSTRING with a single-character
      *>           delimiter, splitting into named fields
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-BASIC.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CSV-LINE          PIC X(30) VALUE "APPLE,BANANA,CHERRY".
       01  WS-FIELD-1           PIC X(10).
       01  WS-FIELD-2           PIC X(10).
       01  WS-FIELD-3           PIC X(10).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           UNSTRING WS-CSV-LINE DELIMITED BY ","
               INTO WS-FIELD-1, WS-FIELD-2, WS-FIELD-3
           END-UNSTRING

           DISPLAY "Field 1: [" WS-FIELD-1 "]"
           DISPLAY "Field 2: [" WS-FIELD-2 "]"
           DISPLAY "Field 3: [" WS-FIELD-3 "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- COBOL จะเดินอ่าน `WS-CSV-LINE` จากซ้ายไปขวา ทุกครั้งที่เจอ `,` จะตัดข้อความก่อนหน้านั้นไปเก็บในฟิลด์ปลายทาง
  ตัวถัดไปตามลำดับ โดยตัวคั่นเอง**จะไม่ถูกเก็บ**ไว้ในฟิลด์ปลายทางใด ๆ
- ฟิลด์ปลายทางที่มีขนาดใหญ่กว่าข้อมูลจริง จะถูกเติมช่องว่างท้ายให้เต็มขนาดโดยอัตโนมัติ (เหมือนพฤติกรรมของ `MOVE`)

### ผลลัพธ์ที่ได้จากการรันจริง

```
Field 1: [APPLE     ]
Field 2: [BANANA    ]
Field 3: [CHERRY    ]
```

### ข้อควรระวัง

- ถ้าจำนวนฟิลด์ปลายทางที่ระบุไว้ **น้อยกว่า** จำนวนส่วนที่ตัดแบ่งได้จริง ส่วนที่เกินจะถูกทิ้งไปเงียบ ๆ
  (เราจะเรียนรู้วิธีตรวจจับสถานการณ์นี้ด้วย `ON OVERFLOW` ในขั้นตอนที่ 197)

### แบบฝึกหัดที่ 192.1

**โจทย์**: จงแก้โค้ดข้างต้นให้ใช้ตัวแปร `WS-CSV-LINE` มีค่า "RED,GREEN,BLUE,YELLOW" (4 ค่า)
แต่ยังคงมีฟิลด์ปลายทางแค่ 3 ตัว แล้วคาดเดาผลลัพธ์

**เฉลย**: จะได้ `WS-FIELD-1` = "RED", `WS-FIELD-2` = "GREEN", `WS-FIELD-3` = "BLUE" ส่วน "YELLOW"
จะไม่ถูกดึงไปเก็บที่ไหนเลยเพราะไม่มีฟิลด์ปลายทางเหลือให้รับ

---

## ขั้นตอนที่ 193: UNSTRING ด้วยตัวคั่นหลายแบบพร้อมกัน (DELIMITED BY ... OR ...)

### แนวคิด

บางครั้งข้อมูลจริงอาจใช้ตัวคั่นมากกว่าหนึ่งชนิดปนกัน (เช่น ข้อมูลเก่าที่มาจากหลายระบบ) COBOL อนุญาตให้ระบุ
ตัวคั่นได้หลายตัวพร้อมกันด้วยวลี **`DELIMITED BY ตัวคั่น-1 OR ตัวคั่น-2 OR ...`** โดยตัวคั่นตัวใดตัวหนึ่ง
ที่เจอก็จะถือเป็นจุดตัดแบ่งเหมือนกันหมด

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s3.cob
      *> Purpose : UNSTRING with multiple possible delimiters
      *>           using the OR phrase
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-MULTI-DELIM.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MIXED-LINE        PIC X(30) VALUE "JOHN,SMITH-45;SALES".
       01  WS-FIELD-1           PIC X(10).
       01  WS-FIELD-2           PIC X(10).
       01  WS-FIELD-3           PIC X(10).
       01  WS-FIELD-4           PIC X(10).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> DELIMITED BY "," OR "-" OR ";" treats any of those
      *> three characters as a field separator.
           UNSTRING WS-MIXED-LINE
               DELIMITED BY "," OR "-" OR ";"
               INTO WS-FIELD-1, WS-FIELD-2, WS-FIELD-3, WS-FIELD-4
           END-UNSTRING

           DISPLAY "Field 1: [" WS-FIELD-1 "]"
           DISPLAY "Field 2: [" WS-FIELD-2 "]"
           DISPLAY "Field 3: [" WS-FIELD-3 "]"
           DISPLAY "Field 4: [" WS-FIELD-4 "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-MIXED-LINE` มีค่า "JOHN,SMITH-45;SALES" ซึ่งใช้ตัวคั่นสามชนิดปนกัน (`,`, `-`, `;`)
- `DELIMITED BY "," OR "-" OR ";"` ทำให้ COBOL ตัดแบ่งข้อความทุกครั้งที่เจอตัวคั่นตัวใดตัวหนึ่งในสามตัวนี้
  ผลลัพธ์คือได้ 4 ส่วนเรียงกัน: "JOHN", "SMITH", "45", "SALES"

### ผลลัพธ์ที่ได้จากการรันจริง

```
Field 1: [JOHN      ]
Field 2: [SMITH     ]
Field 3: [45        ]
Field 4: [SALES     ]
```

### ข้อควรระวัง

- ตัวคั่นแต่ละตัวใน `OR` มีน้ำหนักเท่ากันหมด COBOL ไม่ได้สนใจว่าตัวคั่นตัวไหน "ควร" ปรากฏตรงไหน
  มันแค่มองหาตัวคั่นตัวใดตัวหนึ่งที่ใกล้ที่สุดถัดไปเสมอ ถ้าข้อมูลมีรูปแบบไม่สม่ำเสมอ อาจได้ผลลัพธ์ที่ไม่ตรงตามคาด

### แบบฝึกหัดที่ 193.1

**โจทย์**: จงเพิ่มตัวคั่น `:` (colon) เข้าไปในรายการ `OR` ของโค้ดข้างต้น แล้วทดสอบกับ `WS-MIXED-LINE`
ค่าใหม่ "JOHN,SMITH-45:SALES"

**เฉลย**: เปลี่ยนเป็น `DELIMITED BY "," OR "-" OR ";" OR ":"` ผลลัพธ์ที่ได้จะยังคงเหมือนเดิมคือ
"JOHN", "SMITH", "45", "SALES" เพราะ `:` ก็ถูกมองเป็นตัวคั่นเช่นเดียวกับตัวอื่น ๆ ในรายการ `OR`

---

## ขั้นตอนที่ 194: COUNT IN และ DELIMITER IN - รู้ความยาวและตัวคั่นที่ใช้จริง

### แนวคิด

บางครั้งเราต้องการรู้ **ความยาวจริง** ของแต่ละส่วนที่ถูกตัดแบ่งออกมา (ก่อนถูกเติมช่องว่าง) และ **ตัวคั่นตัวใด**
ที่ถูกใช้จริงในแต่ละจุดตัด (กรณีที่ใช้ `OR` หลายตัวคั่น) COBOL มีวลีเสริมสองตัวสำหรับเรื่องนี้:
**`COUNT IN`** (เก็บความยาวจริงที่ตัดได้) และ **`DELIMITER IN`** (เก็บตัวคั่นที่พบจริง ณ จุดนั้น)

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s4.cob
      *> Purpose : COUNT IN and DELIMITER IN phrases, which
      *>           report how many characters were extracted
      *>           and which delimiter actually matched
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-COUNT-DELIM-IN.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE              PIC X(30) VALUE "RED,GREEN-BLUE".
       01  WS-FIELD-1           PIC X(10).
       01  WS-LEN-1             PIC 9(2).
       01  WS-DELIM-1           PIC X(2).
       01  WS-FIELD-2           PIC X(10).
       01  WS-LEN-2             PIC 9(2).
       01  WS-DELIM-2           PIC X(2).
       01  WS-FIELD-3           PIC X(10).
       01  WS-LEN-3             PIC 9(2).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           UNSTRING WS-LINE DELIMITED BY "," OR "-"
               INTO WS-FIELD-1
                       DELIMITER IN WS-DELIM-1
                       COUNT IN WS-LEN-1
                    WS-FIELD-2
                       DELIMITER IN WS-DELIM-2
                       COUNT IN WS-LEN-2
                    WS-FIELD-3
                       COUNT IN WS-LEN-3
           END-UNSTRING

           DISPLAY "Field 1=[" WS-FIELD-1 "] len=" WS-LEN-1
               " delim=[" WS-DELIM-1 "]"
           DISPLAY "Field 2=[" WS-FIELD-2 "] len=" WS-LEN-2
               " delim=[" WS-DELIM-2 "]"
           DISPLAY "Field 3=[" WS-FIELD-3 "] len=" WS-LEN-3

           STOP RUN.
```

### คำอธิบายโค้ด

- `DELIMITER IN WS-DELIM-1` และ `COUNT IN WS-LEN-1` ต้องเขียนต่อจากฟิลด์ปลายทางแต่ละตัวทันที
  โดย**ลำดับที่ถูกต้องคือ `DELIMITER IN` มาก่อน `COUNT IN`** (ถ้าใช้ทั้งคู่)
- `WS-FIELD-3` ในตัวอย่างนี้ไม่มีตัวคั่นตามหลัง (เป็นส่วนสุดท้ายของข้อความ) จึงใส่แค่ `COUNT IN` เพียงอย่างเดียว
  โดยไม่มี `DELIMITER IN` ก็ได้

### ผลลัพธ์ที่ได้จากการรันจริง

```
Field 1=[RED       ] len=03 delim=[, ]
Field 2=[GREEN     ] len=05 delim=[- ]
Field 3=[BLUE      ] len=20
```

### ข้อควรระวัง

- **ลำดับของวลีสำคัญมาก**: ถ้าเขียน `COUNT IN` ก่อน `DELIMITER IN` จะเกิด syntax error ทันที
  ต้องเขียน `DELIMITER IN` ก่อนเสมอถ้าจะใช้ทั้งสองวลีคู่กัน
- `WS-LEN-3` แสดงค่า 20 ไม่ใช่ 4 (ความยาวของ "BLUE") เพราะ `COUNT IN` สำหรับฟิลด์ปลายทางตัวสุดท้าย
  จะนับความยาวของ**ส่วนที่เหลือทั้งหมด**ของฟิลด์ต้นทาง (`WS-LINE PIC X(30)` ลบด้วยส่วนที่ตัดไปแล้ว 10 ตัวอักษร
  = เหลือ 20 ตัวอักษร) ไม่ใช่ความยาวของคำว่า "BLUE" เพียงอย่างเดียว เพราะ `WS-LINE` มีช่องว่างเติมท้ายอยู่ด้วย

### แบบฝึกหัดที่ 194.1

**โจทย์**: จงอธิบายว่าทำไม `WS-DELIM-1` ถึงมีค่า "," ตามด้วยช่องว่าง (ไม่ใช่แค่ "," เพียงตัวเดียว)

**เฉลยแนวทาง**: เพราะ `WS-DELIM-1` ถูกประกาศเป็น `PIC X(2)` (ขนาด 2 ตัวอักษร) ในขณะที่ตัวคั่นจริงมีความยาว
แค่ 1 ตัวอักษร COBOL จึงเติมช่องว่างท้ายให้เต็มขนาดฟิลด์ตามพฤติกรรมปกติของการ `MOVE` ค่าตัวอักษร

---

## ขั้นตอนที่ 195: UNSTRING WITH POINTER เพื่อแยกข้อมูลต่อเนื่องหลายรอบ

### แนวคิด

เช่นเดียวกับ `STRING`, `UNSTRING` ก็รองรับ **`WITH POINTER`** เพื่อควบคุมตำแหน่งเริ่มอ่านข้อมูล และให้
COBOL อัปเดตตำแหน่งนั้นให้อัตโนมัติหลังตัดแบ่งเสร็จ ทำให้เราเรียก `UNSTRING` ซ้ำได้หลายครั้งกับฟิลด์ต้นทาง
เดียวกัน โดยแต่ละครั้งจะดำเนินการต่อจากจุดที่ครั้งก่อนหยุดไว้ ไม่ใช่เริ่มต้นใหม่จากตำแหน่งแรกเสมอไป

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s5.cob
      *> Purpose : UNSTRING WITH POINTER to resume splitting
      *>           further along the same source field
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-POINTER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE              PIC X(30) VALUE "A-B-C-D-E".
       01  WS-PTR               PIC 9(3) VALUE 1.
       01  WS-FIELD-1           PIC X(5).
       01  WS-FIELD-2           PIC X(5).
       01  WS-FIELD-3           PIC X(5).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> First UNSTRING grabs the first TWO fields only, and
      *> leaves the pointer positioned right after "B-".
           UNSTRING WS-LINE DELIMITED BY "-"
               INTO WS-FIELD-1, WS-FIELD-2
               WITH POINTER WS-PTR
           END-UNSTRING
           DISPLAY "After 1st UNSTRING: F1=[" WS-FIELD-1
               "] F2=[" WS-FIELD-2 "] pointer=" WS-PTR

      *> The second UNSTRING continues from where the pointer
      *> left off, instead of starting over from position 1.
           UNSTRING WS-LINE DELIMITED BY "-"
               INTO WS-FIELD-3
               WITH POINTER WS-PTR
           END-UNSTRING
           DISPLAY "After 2nd UNSTRING: F3=[" WS-FIELD-3
               "] pointer=" WS-PTR

           STOP RUN.
```

### คำอธิบายโค้ด

- `UNSTRING` ตัวแรกดึงแค่ 2 ค่าแรก ("A" และ "B") จากทั้งหมด 5 ค่าที่มีอยู่ใน `WS-LINE`
  แล้วปล่อยให้ `WS-PTR` ค้างอยู่ที่ตำแหน่งถัดจาก "B-" (ตำแหน่งที่ 5)
- `UNSTRING` ตัวที่สองใช้ `WS-PTR` เดิม (ค่า 5) จึงเริ่มอ่านต่อจาก "C-D-E" แทนที่จะเริ่มใหม่จาก "A" อีกครั้ง

### ผลลัพธ์ที่ได้จากการรันจริง

```
After 1st UNSTRING: F1=[A    ] F2=[B    ] pointer=005
After 2nd UNSTRING: F3=[C    ] pointer=007
```

### ข้อควรระวัง

- ถ้าลืมใส่ `WITH POINTER` ในการเรียก `UNSTRING` ครั้งที่สอง จะทำให้เริ่มอ่านจากตำแหน่งที่ 1 ใหม่เสมอ
  (ได้ "A" ซ้ำอีกครั้งแทนที่จะเป็น "C") ซึ่งเป็นข้อผิดพลาดที่พบบ่อยเมื่อประมวลผลข้อความยาว ๆ เป็นหลายรอบ

### แบบฝึกหัดที่ 195.1

**โจทย์**: จากโค้ดข้างต้น จงเขียน `UNSTRING` ตัวที่สามเพื่อดึงค่าที่เหลือ ("D" และ "E") ต่อจากที่ทำไปแล้ว

**เฉลย**:
```cobol
           UNSTRING WS-LINE DELIMITED BY "-"
               INTO WS-FIELD-1, WS-FIELD-2
               WITH POINTER WS-PTR
           END-UNSTRING
```
โดยประกาศฟิลด์ปลายทางเพิ่มอีก 2 ตัวและใช้ `WS-PTR` ตัวเดิมต่อเนื่อง จะได้ "D" และ "E" ตามลำดับ
เพราะ pointer ยังคงชี้อยู่ที่ตำแหน่งถัดจาก "C-" จากการเรียกครั้งก่อนหน้า

---

## ขั้นตอนที่ 196: TALLYING IN - นับจำนวนฟิลด์ที่ UNSTRING แยกออกมาได้จริง

### แนวคิด

วลี **`TALLYING IN ตัวแปร`** ทำให้ COBOL **บวกเพิ่มค่าให้ตัวแปรตัวเลข** ทุกครั้งที่มีการเติมข้อมูลลงฟิลด์
ปลายทางตัวใดตัวหนึ่งสำเร็จ ทำให้เรารู้ได้ว่า `UNSTRING` ครั้งนี้แยกข้อมูลออกมาได้ **กี่ส่วนจริง ๆ**
โดยไม่ต้องนับตัวคั่นเองด้วยมือ มีประโยชน์มากเมื่อไม่แน่ใจว่าข้อมูลต้นทางมีกี่ส่วน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s6.cob
      *> Purpose : TALLYING IN to count how many fields an
      *>           UNSTRING actually produced
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-TALLYING.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE              PIC X(30) VALUE "ONE,TWO,THREE".
       01  WS-FIELD-1           PIC X(10).
       01  WS-FIELD-2           PIC X(10).
       01  WS-FIELD-3           PIC X(10).
       01  WS-FIELD-4           PIC X(10).
       01  WS-FIELD-COUNT       PIC 9(2) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> WS-FIELD-COUNT is increased by one for every receiving
      *> field that UNSTRING actually fills in this statement.
           UNSTRING WS-LINE DELIMITED BY ","
               INTO WS-FIELD-1, WS-FIELD-2, WS-FIELD-3, WS-FIELD-4
               TALLYING IN WS-FIELD-COUNT
           END-UNSTRING

           DISPLAY "Fields filled: " WS-FIELD-COUNT
           DISPLAY "F1=[" WS-FIELD-1 "]"
           DISPLAY "F2=[" WS-FIELD-2 "]"
           DISPLAY "F3=[" WS-FIELD-3 "]"
           DISPLAY "F4=[" WS-FIELD-4 "] (should be empty, unused)"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-LINE` มีข้อมูลแค่ 3 ส่วน ("ONE", "TWO", "THREE") แต่เรากำหนดฟิลด์ปลายทางไว้ 4 ตัว
- `TALLYING IN WS-FIELD-COUNT` จะนับได้แค่ 3 เพราะมีเพียง 3 ฟิลด์ปลายทางเท่านั้นที่ถูกเติมข้อมูลจริง
  ฟิลด์ที่ 4 ไม่ถูกแตะต้องเลยเพราะไม่มีข้อมูลเหลือให้แยก

### ผลลัพธ์ที่ได้จากการรันจริง

```
Fields filled: 03
F1=[ONE       ]
F2=[TWO       ]
F3=[THREE     ]
F4=[          ] (should be empty, unused)
```

### ข้อควรระวัง

- `WS-FIELD-COUNT` ควรมีค่าเริ่มต้นเป็น 0 ก่อนเรียก `UNSTRING` เสมอ (ในตัวอย่างนี้ใช้ `VALUE 0` ตอนประกาศ)
  เพราะ `TALLYING IN` เป็นการ**บวกเพิ่ม**เข้าไปในค่าเดิม ไม่ใช่การกำหนดค่าใหม่ทั้งหมด
  ถ้าไม่เคลียร์ค่าก่อน ผลการนับของหลายรอบจะสะสมปนกันโดยไม่ตั้งใจ

### แบบฝึกหัดที่ 196.1

**โจทย์**: จงอธิบายว่าถ้าเรียก `UNSTRING` แบบเดียวกันนี้อีกครั้งโดยไม่เคลียร์ `WS-FIELD-COUNT` กลับเป็น 0 ก่อน
ค่าที่แสดงผลจะเป็นเท่าไร

**เฉลย**: จะได้ 06 (3 จากรอบแรก บวก 3 จากรอบที่สอง) เพราะ `TALLYING IN` สะสมค่าต่อจากที่มีอยู่เดิมเสมอ
ไม่ได้รีเซ็ตเป็น 0 ให้อัตโนมัติก่อนเริ่มนับใหม่ทุกครั้ง

---

## ขั้นตอนที่ 197: ON OVERFLOW - เมื่อข้อมูลต้นทางมีมากกว่าฟิลด์ปลายทางที่เตรียมไว้

### แนวคิด

เหมือนกับ `STRING`, คำสั่ง `UNSTRING` ก็มี **`ON OVERFLOW`** และ **`NOT ON OVERFLOW`** สำหรับตรวจจับ
สถานการณ์ที่ข้อมูลต้นทางมี**ส่วนเหลือ**อยู่หลังจากเติมฟิลด์ปลายทางที่มีอยู่จนครบหมดแล้ว (คือมีตัวคั่นมากกว่า
ที่ฟิลด์ปลายทางจะรับไหว) ต่างจาก `STRING` ที่ overflow เกิดจาก "พื้นที่ไม่พอ" แต่ของ `UNSTRING` เกิดจาก
"จำนวนฟิลด์ปลายทางไม่พอ"

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s7.cob
      *> Purpose : ON OVERFLOW - when the source has MORE
      *>           fields than the receiving items provided
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-OVERFLOW.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE              PIC X(30) VALUE "1,2,3,4,5".
       01  WS-FIELD-1           PIC X(5).
       01  WS-FIELD-2           PIC X(5).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> WS-LINE has 5 comma-separated fields, but we only gave
      *> UNSTRING two receiving fields. That is an overflow.
           UNSTRING WS-LINE DELIMITED BY ","
               INTO WS-FIELD-1, WS-FIELD-2
               ON OVERFLOW
                   DISPLAY "OVERFLOW: more fields than targets."
               NOT ON OVERFLOW
                   DISPLAY "All fields fit, no overflow."
           END-UNSTRING

           DISPLAY "F1=[" WS-FIELD-1 "] F2=[" WS-FIELD-2 "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- `WS-LINE` มี 5 ส่วน ("1" ถึง "5") แต่มีฟิลด์ปลายทางแค่ 2 ตัว จึงเกิด overflow แน่นอน
- `ON OVERFLOW` จะทำงานเมื่อยังมีข้อมูลเหลืออยู่แต่ไม่มีฟิลด์ปลายทางเหลือรับ ส่วน `NOT ON OVERFLOW`
  จะทำงานเมื่อข้อมูลพอดีกับจำนวนฟิลด์ปลายทางหรือน้อยกว่า

### ผลลัพธ์ที่ได้จากการรันจริง

```
OVERFLOW: more fields than targets.
F1=[1    ] F2=[2    ]
```

### ข้อควรระวัง

- ควรใส่ `ON OVERFLOW` เสมอเมื่อไม่แน่ใจว่าข้อมูลต้นทางจะมีจำนวนส่วนเท่าไร โดยเฉพาะข้อมูลที่มาจากภายนอกระบบ
  (เช่น ไฟล์ CSV ที่ผู้ใช้อัปโหลดเอง) เพราะถ้าไม่ตรวจสอบ ข้อมูลส่วนเกินจะหายไปเงียบ ๆ โดยไม่มีการแจ้งเตือนใด ๆ

### แบบฝึกหัดที่ 197.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง overflow ของ `STRING` (ขั้นตอนที่ 186) กับ overflow ของ `UNSTRING`
(ขั้นตอนนี้)

**เฉลยแนวทาง**: overflow ของ `STRING` เกิดเมื่อ**ฟิลด์ปลายทางมีขนาดเล็กเกินไป**สำหรับข้อมูลที่จะเขียนลงไป
(ปัญหาเรื่อง "พื้นที่") ส่วน overflow ของ `UNSTRING` เกิดเมื่อ**ฟิลด์ปลายทางมีจำนวนน้อยเกินไป**สำหรับ
จำนวนส่วนที่ตัดแบ่งได้จากข้อมูลต้นทาง (ปัญหาเรื่อง "จำนวนรายการ") ทั้งสองกรณีล้วนทำให้ข้อมูลบางส่วนสูญหาย
ถ้าไม่ตรวจสอบด้วย `ON OVERFLOW`

---

## ขั้นตอนที่ 198: แยกข้อมูลไม่ทราบจำนวนฟิลด์ล่วงหน้าด้วยการวนลูปร่วมกับ WITH POINTER

### แนวคิด

เมื่อไม่ทราบจำนวนฟิลด์ที่แน่นอนล่วงหน้า (เช่น รายการสินค้าที่มีจำนวนไม่แน่นอนในแต่ละบรรทัด) เราสามารถ
รวมเทคนิค `WITH POINTER` (ขั้นตอนที่ 195) เข้ากับ `PERFORM VARYING` เพื่อดึงข้อมูลทีละส่วนเข้า**ตาราง**
จนกว่า pointer จะเดินทางถึงจุดสิ้นสุดของข้อความ หรือจนกว่าตารางจะเต็ม

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s8.cob
      *> Purpose : Parse a full CSV line of unknown field count
      *>           into a table, one field at a time in a loop
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-INTO-TABLE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE              PIC X(40)
           VALUE "MON,TUE,WED,THU,FRI,SAT,SUN".
       01  WS-PTR               PIC 9(3) VALUE 1.
       01  WS-LINE-LEN          PIC 9(3) VALUE 40.

       01  DAY-TABLE.
           05  DAY-NAME PIC X(10) OCCURS 7 TIMES.

       01  WS-IDX               PIC 9 VALUE 1.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
      *> Loop, pulling one comma-delimited field per iteration,
      *> as long as the pointer has not passed the end of line.
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > 7 OR WS-PTR > WS-LINE-LEN
               UNSTRING WS-LINE DELIMITED BY ","
                   INTO DAY-NAME(WS-IDX)
                   WITH POINTER WS-PTR
               END-UNSTRING
           END-PERFORM

           DISPLAY "Parsed days of week:"
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 7
               DISPLAY "  " WS-IDX ": " DAY-NAME(WS-IDX)
           END-PERFORM

           STOP RUN.
```

### คำอธิบายโค้ด

- ลูป `PERFORM VARYING` มีเงื่อนไขหยุดสองแบบ: `WS-IDX > 7` (ตารางเต็ม 7 ช่องแล้ว) หรือ
  `WS-PTR > WS-LINE-LEN` (pointer เดินเลยความยาวของฟิลด์ต้นทางไปแล้ว) — ใช้ `OR` เพื่อป้องกันการวนลูปเกิน
  ทั้งสองกรณี
- ในแต่ละรอบ `UNSTRING` จะดึงมาแค่ **หนึ่งส่วน** (เพราะมีฟิลด์ปลายทางแค่ตัวเดียวคือ `DAY-NAME(WS-IDX)`)
  แล้วขยับ pointer ไปสำหรับรอบถัดไปโดยอัตโนมัติ

### ผลลัพธ์ที่ได้จากการรันจริง

```
Parsed days of week:
  1: MON       
  2: TUE       
  3: WED       
  4: THU       
  5: FRI       
  6: SAT       
  7: SUN       
```

### ข้อควรระวัง

- ต้องระบุเงื่อนไขหยุดที่ครอบคลุมทั้ง "ตารางเต็ม" และ "ข้อมูลหมด" เสมอ ถ้าใส่เงื่อนไขแค่อย่างใดอย่างหนึ่ง
  อาจเกิด subscript เกินขอบเขตตาราง (ถ้าข้อมูลมีมากกว่าที่ตารางรองรับ) หรือ `UNSTRING` พยายามอ่านเลยตำแหน่ง
  ที่มีข้อมูลจริง (ถ้าข้อมูลหมดก่อนตารางเต็ม)

### แบบฝึกหัดที่ 198.1

**โจทย์**: จงอธิบายว่าทำไมโค้ดข้างต้นจึงกำหนด `WS-LINE-LEN VALUE 40` ทั้งที่ข้อความจริง
"MON,TUE,WED,THU,FRI,SAT,SUN" สั้นกว่านั้นมาก

**เฉลยแนวทาง**: เพราะ `WS-LINE PIC X(40)` ถูกประกาศให้มีขนาด 40 ไบต์เต็ม ส่วนที่เกินความยาวข้อความจริง
จะถูกเติมด้วยช่องว่างโดยอัตโนมัติ (ตามกฎการ `MOVE`/`VALUE` ของฟิลด์ตัวอักษร) เงื่อนไข `WS-PTR > WS-LINE-LEN`
จึงต้องอ้างอิงกับขนาดเต็มของฟิลด์ (40) ไม่ใช่ความยาวของข้อความจริงที่บรรจุอยู่ เพื่อให้ตรงกับพฤติกรรมจริง
ของหน่วยความจำที่ COBOL จัดสรรไว้

---

## ขั้นตอนที่ 199: เปรียบเทียบข้อมูลแบบ Delimited กับแบบ Fixed-width

### แนวคิด

`UNSTRING` เหมาะกับข้อมูลที่ใช้ **ตัวคั่น (delimited format)** เท่านั้น แต่ข้อมูลอีกรูปแบบหนึ่งที่พบบ่อยไม่แพ้กัน
คือ **Fixed-width format** ซึ่งแต่ละฟิลด์มีความยาวคงที่แน่นอนโดยไม่ต้องมีตัวคั่นเลย สำหรับข้อมูลแบบนี้
เราไม่จำเป็นต้องใช้ `UNSTRING` เลย เพียงใช้ **Reference Modification** (`field(ตำแหน่งเริ่ม:ความยาว)`)
ในการตัดส่วนที่ต้องการออกมาได้โดยตรง ขั้นตอนนี้จะเปรียบเทียบทั้งสองรูปแบบให้เห็นภาพชัดเจน

### โค้ดตัวอย่าง

```cobol
      *> ===================================================
      *> Program : s9.cob
      *> Purpose : Compare parsing a DELIMITED line versus
      *>           reading a FIXED-width structure, showing
      *>           when UNSTRING is the right tool and when
      *>           simple reference modification is enough
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-VS-FIXED-WIDTH.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Delimited style: fields separated by "|", variable length
       01  WS-DELIM-LINE   PIC X(30) VALUE "SMITH|SALES|45000".

      *> Fixed-width style: every field always occupies the same
      *> number of columns, no delimiter characters needed at all
       01  WS-FIXED-LINE   PIC X(21) VALUE "SMITH     SALES 45000".

       01  WS-NAME              PIC X(10).
       01  WS-DEPT              PIC X(6).
       01  WS-SALARY            PIC X(5).

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           DISPLAY "--- Parsing delimited data with UNSTRING ---"
           UNSTRING WS-DELIM-LINE DELIMITED BY "|"
               INTO WS-NAME, WS-DEPT, WS-SALARY
           END-UNSTRING
           DISPLAY "NAME=[" WS-NAME "] DEPT=[" WS-DEPT
               "] SALARY=[" WS-SALARY "]"

           DISPLAY " "
           DISPLAY "--- Parsing fixed-width data (no UNSTRING) ---"
      *> Reference modification: field(start-position:length)
           MOVE WS-FIXED-LINE(1:10)  TO WS-NAME
           MOVE WS-FIXED-LINE(11:6)  TO WS-DEPT
           MOVE WS-FIXED-LINE(17:5)  TO WS-SALARY
           DISPLAY "NAME=[" WS-NAME "] DEPT=[" WS-DEPT
               "] SALARY=[" WS-SALARY "]"

           STOP RUN.
```

### คำอธิบายโค้ด

- ข้อมูลแบบแรก (`WS-DELIM-LINE`) ใช้ `|` เป็นตัวคั่น ความยาวของแต่ละฟิลด์ไม่แน่นอน จึงต้องใช้ `UNSTRING`
- ข้อมูลแบบที่สอง (`WS-FIXED-LINE`) ทุกฟิลด์มีตำแหน่งและความยาวคงที่เป๊ะ (ชื่อ 10 ไบต์, แผนก 6 ไบต์,
  เงินเดือน 5 ไบต์) จึงใช้ **Reference Modification** แบบ `field(start:length)` ตัดตรง ๆ ได้เลย
  โดยไม่ต้องมีตัวคั่นใด ๆ ในข้อมูล และเร็วกว่า `UNSTRING` เพราะไม่ต้องสแกนหาตัวคั่น

### ผลลัพธ์ที่ได้จากการรันจริง

```
--- Parsing delimited data with UNSTRING ---
NAME=[SMITH     ] DEPT=[SALES ] SALARY=[45000]
 
--- Parsing fixed-width data (no UNSTRING) ---
NAME=[SMITH     ] DEPT=[SALES ] SALARY=[45000]
```

### ข้อควรระวัง

- การเลือกใช้ `UNSTRING` หรือ Reference Modification ขึ้นอยู่กับ**รูปแบบของข้อมูลต้นทาง** ไม่ใช่ความชอบส่วนตัว
  ถ้าข้อมูลมีตัวคั่นและความยาวไม่แน่นอน ต้องใช้ `UNSTRING` แต่ถ้าข้อมูลมีความยาวคงที่ทุกฟิลด์เสมอ (พบบ่อยมาก
  ในไฟล์ Mainframe แบบดั้งเดิม) การใช้ Reference Modification จะง่ายกว่าและเร็วกว่า

### แบบฝึกหัดที่ 199.1

**โจทย์**: จงบอกเหตุผล 1 ข้อว่าทำไมระบบ Mainframe รุ่นเก่าจำนวนมากถึงนิยมใช้รูปแบบ Fixed-width
มากกว่ารูปแบบ Delimited

**เฉลยแนวทาง**: รูปแบบ Fixed-width ประมวลผลได้เร็วกว่ามาก (ไม่ต้องสแกนหาตัวคั่นทีละตัวอักษร)
และคาดเดาขนาดไฟล์ได้แน่นอน (จำนวนเรคคอร์ด x ความยาวคงที่ = ขนาดไฟล์ทั้งหมด) ซึ่งสำคัญมากสำหรับการประมวลผล
แบบ Batch ปริมาณมหาศาลบน Mainframe ที่เน้นประสิทธิภาพและการวางแผนทรัพยากรล่วงหน้าเป็นหลัก

---

## ขั้นตอนที่ 200: สรุปข้อควรระวัง และแบบฝึกหัดวิเคราะห์ที่อยู่แบบเต็มรูปแบบ

### สรุปข้อควรระวังสำคัญของ UNSTRING

1. **ฟิลด์ปลายทางน้อยกว่าจำนวนส่วนที่ตัดแบ่งได้จริง** ทำให้ข้อมูลส่วนเกินหายไปเงียบ ๆ ถ้าไม่ใช้ `ON OVERFLOW`
2. **`TALLYING IN` สะสมค่าต่อจากเดิมเสมอ** ต้องเคลียร์ตัวแปรนับเป็น 0 ก่อนเรียกใหม่ทุกครั้ง
3. **ลืมใช้ `WITH POINTER` ตอนเรียกซ้ำ** จะทำให้เริ่มอ่านจากตำแหน่งที่ 1 ใหม่เสมอ แทนที่จะต่อจากจุดเดิม
4. **ลำดับ `DELIMITER IN` ต้องมาก่อน `COUNT IN`** เสมอเมื่อใช้ทั้งสองวลีคู่กัน
5. **เลือกเครื่องมือให้ตรงกับรูปแบบข้อมูล**: ข้อมูลมีตัวคั่นใช้ `UNSTRING`, ข้อมูลความยาวคงที่ใช้
   Reference Modification จะง่ายและเร็วกว่า

### แบบฝึกหัดสรุปรวม: วิเคราะห์ที่อยู่แบบเต็มรูปแบบ

โจทย์: จงเขียนโปรแกรมที่รับข้อความที่อยู่เต็มรูปแบบคั่นด้วยจุลภาค (ถนน, เมือง, รัฐ, รหัสไปรษณีย์)
แล้วแยกออกเป็นฟิลด์ย่อยพร้อมตรวจสอบว่าจำนวนฟิลด์ที่แยกได้ตรงตามที่คาดหวังหรือไม่ (ใช้ `TALLYING IN`
ร่วมกับ `ON OVERFLOW`)

### โค้ดเฉลย

```cobol
      *> ===================================================
      *> Program : s10.cob
      *> Purpose : Exercise solution - parse a full postal
      *>           address string into separate components
      *> ===================================================
       IDENTIFICATION DIVISION.
       PROGRAM-ID. UNSTRING-ADDRESS-PARSER.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ADDRESS
           PIC X(50) VALUE "99 PARK AVENUE,RIVERSIDE,CA,92501".

       01  WS-STREET            PIC X(20).
       01  WS-CITY              PIC X(15).
       01  WS-STATE             PIC X(2).
       01  WS-ZIP               PIC X(5).
       01  WS-FIELD-COUNT       PIC 9 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           UNSTRING WS-ADDRESS DELIMITED BY ","
               INTO WS-STREET, WS-CITY, WS-STATE, WS-ZIP
               TALLYING IN WS-FIELD-COUNT
               ON OVERFLOW
                   DISPLAY "WARNING: address has extra fields!"
               NOT ON OVERFLOW
                   DISPLAY "Address parsed cleanly."
           END-UNSTRING

           DISPLAY "Fields found : " WS-FIELD-COUNT
           DISPLAY "Street       : " WS-STREET
           DISPLAY "City         : " WS-CITY
           DISPLAY "State        : " WS-STATE
           DISPLAY "ZIP          : " WS-ZIP

           STOP RUN.
```

### คำอธิบายโค้ด

- โปรแกรมนี้รวมทั้ง `TALLYING IN` (นับจำนวนฟิลด์จริง) และ `ON OVERFLOW`/`NOT ON OVERFLOW` (ตรวจสอบว่า
  ข้อมูลมีมากกว่าที่คาดไว้หรือไม่) เข้าด้วยกัน ทำให้โปรแกรมมีความ "ป้องกันข้อผิดพลาด" (defensive) มากขึ้น
- เนื่องจากที่อยู่นี้มีพอดี 4 ส่วนตรงกับฟิลด์ปลายทาง 4 ตัว จึงไม่เกิด overflow และ `WS-FIELD-COUNT` จะได้ 4

### ผลลัพธ์ที่ได้จากการรันจริง

```
Address parsed cleanly.
Fields found : 4
Street       : 99 PARK AVENUE      
City         : RIVERSIDE      
State        : CA
ZIP          : 92501
```

### ข้อควรระวัง

- ในระบบจริงที่รับข้อมูลจากภายนอก (เช่น แบบฟอร์มเว็บ หรือไฟล์ที่ผู้ใช้อัปโหลด) ควรตรวจสอบ `WS-FIELD-COUNT`
  เทียบกับจำนวนที่คาดหวังไว้เสมอ (ในที่นี้คือ 4) ก่อนนำข้อมูลไปประมวลผลต่อ เพราะถ้าผู้ใช้ป้อนที่อยู่มา
  ไม่ครบ 4 ส่วน (เช่น ลืมใส่รัฐ) ฟิลด์ปลายทางบางตัวอาจไม่ได้รับข้อมูลใด ๆ เลยโดยไม่มี error แจ้งเตือน

### แบบฝึกหัดที่ 200.1

**โจทย์**: จงปรับโปรแกรมข้างต้นให้แสดงข้อความเตือนเพิ่มเติมถ้า `WS-FIELD-COUNT` ไม่เท่ากับ 4 พอดี
(ทั้งกรณีน้อยกว่าและมากกว่า)

**เฉลยแนวทาง**: เพิ่มคำสั่งหลัง `END-UNSTRING`:
```cobol
           IF WS-FIELD-COUNT NOT = 4
               DISPLAY "ERROR: expected 4 address parts, got "
                   WS-FIELD-COUNT
           END-IF
```
วิธีนี้ครอบคลุมทั้งกรณีข้อมูลมีน้อยกว่าที่คาดไว้ (`TALLYING IN` จะได้ค่าน้อยกว่า 4 โดยไม่เกิด overflow เลย)
และกรณีมีมากกว่าที่คาดไว้ (ซึ่ง `ON OVERFLOW` จะทำงานอยู่แล้ว แต่การตรวจสอบซ้ำด้วย `IF` ทำให้แน่ใจยิ่งขึ้น)

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้คำสั่ง `UNSTRING` สำหรับแยกข้อความอย่างครบถ้วน ได้แก่:

- แนวคิดของ `UNSTRING` ในฐานะคำสั่งตรงข้ามกับ `STRING`
- การแยกข้อมูล CSV พื้นฐานด้วย `DELIMITED BY`
- การใช้ตัวคั่นหลายแบบพร้อมกันด้วย `DELIMITED BY ... OR ...`
- การใช้ `COUNT IN` และ `DELIMITER IN` เพื่อรู้ความยาวและตัวคั่นที่พบจริง
- การใช้ `WITH POINTER` เพื่อแยกข้อมูลต่อเนื่องหลายรอบจากฟิลด์ต้นทางเดียวกัน
- การใช้ `TALLYING IN` เพื่อนับจำนวนฟิลด์ที่แยกออกมาได้จริง
- การตรวจจับข้อมูลล้นด้วย `ON OVERFLOW` และ `NOT ON OVERFLOW`
- การใช้ `UNSTRING` ร่วมกับลูปเพื่อแยกข้อมูลที่ไม่ทราบจำนวนฟิลด์ล่วงหน้าเข้าตาราง
- การเปรียบเทียบข้อมูลแบบ Delimited กับ Fixed-width และเมื่อใดควรใช้ Reference Modification แทน
- สรุปข้อควรระวังทั้งหมด พร้อมแบบฝึกหัดวิเคราะห์ที่อยู่แบบเต็มรูปแบบ

ตอนนี้เรามีเครื่องมือครบทั้งสองด้านของการจัดการข้อความแล้ว: `STRING` (รวม) และ `UNSTRING` (แยก)
ใน **Part 021** เราจะเรียนรู้เครื่องมือจัดการข้อความตัวที่สามที่สำคัญไม่แพ้กันคือ **`INSPECT`**
ซึ่งใช้สำหรับ **นับ** จำนวนตัวอักษรที่ปรากฏในข้อความ **แทนที่** ตัวอักษรบางตัว และ **แปลง** ชุดตัวอักษร
ทั้งหมด (เช่น แปลงตัวพิมพ์เล็กเป็นตัวพิมพ์ใหญ่) ซึ่งเป็นทักษะที่มักใช้ร่วมกับ `STRING` และ `UNSTRING`
ในการทำความสะอาดข้อมูล (data cleansing) ในงานจริง

**[← กลับไป Part 019](part-019-string-statement.md)** |
**[ไปยัง Part 021: INSPECT Statement - นับ แทนที่ แปลงตัวอักษร →](part-021-inspect-statement.md)**
