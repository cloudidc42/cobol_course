# Part 007: DISPLAY และ ACCEPT: คำสั่ง Input/Output พื้นฐาน (ขั้นตอนที่ 61–70)

## คำนำของ Part นี้

ใน Part 006 เราเจาะลึก PICTURE Clause จนเข้าใจอย่างละเอียดว่าตัวแปรแต่ละชนิดเก็บข้อมูลอย่างไร
ตอนนี้ถึงเวลาที่จะทำให้โปรแกรมของเรา **"พูดคุย" กับผู้ใช้งานได้จริง** ผ่านคำสั่งที่คุณใช้มาตั้งแต่โปรแกรม
แรกสุดในหลักสูตรนี้โดยไม่รู้ตัว นั่นคือ `DISPLAY` (แสดงผลลัพธ์ออกหน้าจอ) และคำสั่งคู่กันที่ยังไม่ได้เรียน
อย่างเป็นระบบคือ `ACCEPT` (รับข้อมูลจากผู้ใช้งานผ่านแป้นพิมพ์)

แม้ `DISPLAY` จะดูเรียบง่ายและคุณใช้มาแล้วนับร้อยครั้งในตัวอย่างก่อนหน้า แต่ยังมีความสามารถซ่อนอยู่
อีกมาก เช่น การควบคุมการขึ้นบรรทัดใหม่ด้วย `WITH NO ADVANCING` ส่วน `ACCEPT` คือประตูสู่การเขียน
โปรแกรมแบบ**โต้ตอบ (interactive)** ที่แท้จริงเป็นครั้งแรกในหลักสูตรนี้ — โปรแกรมที่รอรับข้อมูลจากผู้ใช้
แล้วนำไปประมวลผลทันที แทนที่จะกำหนดค่าไว้ล่วงหน้าด้วย `VALUE` เพียงอย่างเดียวเหมือนที่ผ่านมา

Part นี้จะพาคุณเรียนรู้ `DISPLAY` และ `ACCEPT` อย่างครบถ้วน ตั้งแต่การใช้งานพื้นฐาน การรับค่าจาก
ระบบปฏิบัติการ (วันที่ เวลา) การตรวจสอบความถูกต้องของข้อมูลนำเข้า ไปจนถึงข้อผิดพลาดที่พบบ่อยที่สุด
ปิดท้ายด้วยโปรแกรมคำนวณ BMI แบบโต้ตอบที่ใช้ทั้งสองคำสั่งร่วมกันอย่างเต็มรูปแบบ

---

## ขั้นตอนที่ 61: DISPLAY พื้นฐาน — แสดง Literal และตัวแปร

### แนวคิด

`DISPLAY` เป็นคำสั่งแสดงผลข้อมูลออกทางหน้าจอ (standard output) รูปแบบพื้นฐานที่สุดคือ:

```
DISPLAY <รายการที่ 1> <รายการที่ 2> ...
```

โดยแต่ละรายการอาจเป็น **literal** (ข้อความหรือตัวเลขคงที่ที่เขียนตรง ๆ) หรือ **identifier**
(ชื่อตัวแปร) ก็ได้ และสามารถใส่ได้หลายรายการต่อกันในคำสั่งเดียว โดย COBOL จะนำมาต่อกันเป็น
บรรทัดเดียวโดยอัตโนมัติ (ไม่มีช่องว่างแทรกให้เอง เว้นแต่จะเขียนช่องว่างไว้เป็น literal เอง)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DISPLAY-BASIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CITY                PIC X(10) VALUE "BANGKOK".
       01  WS-POPULATION          PIC 9(8)  VALUE 10539000.

       PROCEDURE DIVISION.
           DISPLAY "SIMPLE LITERAL TEXT".
           DISPLAY WS-CITY.
           DISPLAY "CITY: " WS-CITY " POPULATION: " WS-POPULATION.
           DISPLAY "A" " " "B" " " "C".
           STOP RUN.
```

### อธิบายโค้ด

- `DISPLAY "SIMPLE LITERAL TEXT"` — แสดงข้อความคงที่ตรง ๆ บรรทัดเดียว
- `DISPLAY WS-CITY` — แสดงค่าที่เก็บอยู่ในตัวแปร (ทั้งฟิลด์ 10 ไบต์ รวมช่องว่างที่เติมทางขวา)
- `DISPLAY "CITY: " WS-CITY " POPULATION: " WS-POPULATION` — ผสม literal และตัวแปรในคำสั่ง
  เดียวกัน ทุกรายการถูกต่อกันเป็นบรรทัดเดียวตามลำดับที่เขียน
- `DISPLAY "A" " " "B" " " "C"` — แสดงให้เห็นว่าต้องใส่ literal ช่องว่าง `" "` เองหากต้องการเว้น
  วรรคระหว่างรายการ เพราะ COBOL ไม่เติมช่องว่างให้อัตโนมัติระหว่างรายการที่ต่อกันใน DISPLAY เดียว

### ผลลัพธ์ที่ได้จากการรันจริง

```
SIMPLE LITERAL TEXT
BANGKOK   
CITY: BANGKOK    POPULATION: 10539000
A B C
```

สังเกตบรรทัดที่สอง `BANGKOK   ` มีช่องว่างต่อท้าย 3 ช่อง เพราะ `WS-CITY` เป็น `PIC X(10)` แต่ค่าจริง
มีแค่ 7 ตัวอักษร (ตามกฎ PIC X ที่เรียนใน Part 006 ขั้นตอนที่ 53) และบรรทัดที่สามมีช่องว่างระหว่าง
"BANGKOK" กับ "POPULATION" มากกว่าปกติ เพราะช่องว่างที่เติมทางขวาของ `WS-CITY` ถูกแสดงออกมาด้วย

### ข้อควรระวัง

- **DISPLAY ไม่เติมช่องว่างระหว่างรายการให้อัตโนมัติ** ต้องใส่ literal ช่องว่างเองเสมอถ้าต้องการ
  เว้นวรรค มิเช่นนั้นข้อความจะติดกันเป็นพืด อ่านยาก
- ทุกครั้งที่มี `DISPLAY` แบบไม่มี `WITH NO ADVANCING` (จะเรียนในขั้นตอนถัดไป) โปรแกรมจะขึ้น
  บรรทัดใหม่โดยอัตโนมัติหลังจบคำสั่งเสมอ

### แบบฝึกหัดที่ 61.1

**โจทย์**: จงเขียนคำสั่ง DISPLAY เพื่อแสดงผล "TOTAL: 500 BAHT" โดยใช้ตัวแปร
`WS-AMOUNT PIC 9(3) VALUE 500`

**เฉลย**:
```cobol
DISPLAY "TOTAL: " WS-AMOUNT " BAHT".
```

---

## ขั้นตอนที่ 62: DISPLAY WITH NO ADVANCING — ควบคุมการขึ้นบรรทัดใหม่

### แนวคิด

โดยปกติทุกคำสั่ง `DISPLAY` จะขึ้นบรรทัดใหม่ (คล้ายการกด Enter) หลังแสดงผลเสร็จเสมอ แต่บางครั้งเรา
ต้องการให้ข้อความจากหลาย `DISPLAY` อยู่**บรรทัดเดียวกัน** เช่น การสร้าง prompt ก่อนรับข้อมูลจาก
ผู้ใช้ที่ควรให้เคอร์เซอร์กะพริบต่อท้ายข้อความถามทันที ไม่ใช่ขึ้นบรรทัดใหม่ก่อน — วลี **`WITH NO
ADVANCING`** ต่อท้าย DISPLAY แก้ปัญหานี้ได้โดยตรง

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DISPLAY-NO-ADVANCING-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME                PIC X(15) VALUE "MALEE".

       PROCEDURE DIVISION.
           DISPLAY "NORMAL DISPLAY MOVES TO A NEW LINE AFTER".
           DISPLAY "EACH STATEMENT, LIKE THESE TWO LINES.".

           DISPLAY "THIS TEXT " WITH NO ADVANCING.
           DISPLAY "STAYS ON THE SAME LINE.".

           DISPLAY "ENTER YOUR NAME: " WITH NO ADVANCING.
           DISPLAY WS-NAME.
           DISPLAY "^ THE PROMPT AND THE ANSWER SHARE ONE LINE".
           DISPLAY "WHEN WITH NO ADVANCING SUPPRESSES THE".
           DISPLAY "NEWLINE AFTER THE PROMPT.".
           STOP RUN.
```

### อธิบายโค้ด

- `DISPLAY "THIS TEXT " WITH NO ADVANCING` — แสดงข้อความแล้ว**ไม่ขึ้นบรรทัดใหม่** เคอร์เซอร์ยังอยู่
  ท้ายข้อความนั้น
- `DISPLAY "STAYS ON THE SAME LINE."` (บรรทัดถัดมา ไม่มี WITH NO ADVANCING) — จะต่อท้ายข้อความ
  ก่อนหน้าทันทีในบรรทัดเดียวกัน เพราะไม่มีการขึ้นบรรทัดใหม่คั่นกลาง
- รูปแบบ `DISPLAY "ENTER YOUR NAME: " WITH NO ADVANCING` ตามด้วย `DISPLAY WS-NAME` คือรูปแบบ
  ที่ใช้บ่อยที่สุดในทางปฏิบัติ: แสดงข้อความถาม (prompt) แล้วให้คำตอบปรากฏต่อท้ายในบรรทัดเดียวกัน
  ซึ่งเป็นรากฐานสำคัญก่อนที่เราจะนำไปใช้คู่กับ `ACCEPT` ในขั้นตอนถัดไป

### ผลลัพธ์ที่ได้จากการรันจริง

```
NORMAL DISPLAY MOVES TO A NEW LINE AFTER
EACH STATEMENT, LIKE THESE TWO LINES.
THIS TEXT STAYS ON THE SAME LINE.
ENTER YOUR NAME: MALEE          
^ THE PROMPT AND THE ANSWER SHARE ONE LINE
WHEN WITH NO ADVANCING SUPPRESSES THE
NEWLINE AFTER THE PROMPT.
```

### ข้อควรระวัง

- ลืม `WITH NO ADVANCING` เมื่อต้องการสร้าง prompt ก่อน `ACCEPT` เป็นความผิดพลาดที่พบบ่อยมาก
  ผลคือข้อความถามกับที่ที่ผู้ใช้พิมพ์คำตอบจะอยู่คนละบรรทัด ทำให้ประสบการณ์ผู้ใช้ (UX) ดูไม่เป็น
  มืออาชีพ แม้โปรแกรมจะทำงานถูกต้องทุกประการก็ตาม
- `WITH NO ADVANCING` ควบคุมเฉพาะการขึ้นบรรทัดใหม่ **ของ DISPLAY เท่านั้น** ไม่ได้เกี่ยวข้องกับ
  การเว้นวรรคระหว่างรายการภายใน DISPLAY เดียวกัน (ยังต้องใส่ literal ช่องว่างเองตามขั้นตอนที่ 61)

### แบบฝึกหัดที่ 62.1

**โจทย์**: จงเขียนคำสั่ง DISPLAY สองบรรทัดเพื่อให้ได้ผลลัพธ์ "PRICE: 100" ในบรรทัดเดียว
โดยบรรทัดแรกแสดง "PRICE: " และบรรทัดที่สองแสดงตัวแปร `WS-PRICE PIC 9(3) VALUE 100`

**เฉลย**:
```cobol
DISPLAY "PRICE: " WITH NO ADVANCING.
DISPLAY WS-PRICE.
```

---

## ขั้นตอนที่ 63: ACCEPT พื้นฐาน — รับข้อมูลจากผู้ใช้เข้าฟิลด์ Alphanumeric

### แนวคิด

`ACCEPT` เป็นคำสั่งคู่ตรงข้ามของ `DISPLAY` ใช้ **รับข้อมูลจากแป้นพิมพ์ (standard input)** เข้าไปเก็บ
ในตัวแปร รูปแบบพื้นฐานที่สุดคือ:

```
ACCEPT <ชื่อตัวแปร>
```

เมื่อโปรแกรมทำงานถึงคำสั่งนี้ จะ**หยุดรอ**จนกว่าผู้ใช้จะพิมพ์ข้อความแล้วกดปุ่ม Enter ข้อความที่พิมพ์
จะถูกเก็บเข้าตัวแปรที่ระบุไว้ทันที ตามกฎการ MOVE ของชนิดข้อมูลนั้น ๆ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCEPT-BASIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-USER-NAME           PIC X(20).

       PROCEDURE DIVISION.
           DISPLAY "ENTER YOUR NAME: " WITH NO ADVANCING.
           ACCEPT WS-USER-NAME.
           DISPLAY "HELLO, " WS-USER-NAME "! WELCOME TO COBOL.".
           STOP RUN.
```

### อธิบายโค้ด

- `DISPLAY "ENTER YOUR NAME: " WITH NO ADVANCING` — สร้าง prompt ให้เคอร์เซอร์รอรับคำตอบใน
  บรรทัดเดียวกัน (ตามเทคนิคจากขั้นตอนที่ 62)
- `ACCEPT WS-USER-NAME` — โปรแกรมหยุดรอผู้ใช้พิมพ์ชื่อ เมื่อกด Enter ข้อความที่พิมพ์จะถูกเก็บใน
  `WS-USER-NAME` ตามกฎ PIC X (เติมช่องว่างทางขวาถ้าพิมพ์สั้นกว่า 20 ตัวอักษร ตัดทิ้งถ้ายาวกว่า)
- `DISPLAY "HELLO, " WS-USER-NAME "! WELCOME TO COBOL."` — นำค่าที่รับมาไปแสดงผลต่อทันที

### ผลลัพธ์การรันจริง (เมื่อผู้ใช้พิมพ์ "Somchai" แล้วกด Enter)

เนื่องจากโปรแกรมนี้ต้องการ interactive input การทดสอบอัตโนมัติทำได้โดยส่งข้อความผ่าน pipe
เข้าไปแทนการพิมพ์จริง:

```bash
echo "Somchai" | ./accept-basic-demo
```

**ผลลัพธ์ที่ได้**:

```
ENTER YOUR NAME: HELLO, Somchai             ! WELCOME TO COBOL.
```

สังเกตว่า prompt "ENTER YOUR NAME: " กับผลลัพธ์ "HELLO, ..." อยู่บรรทัดเดียวกัน เพราะเมื่อรันจาก
terminal จริง ข้อความที่ผู้ใช้พิมพ์จะปรากฏต่อจาก prompt ก่อนกด Enter (ในการทดสอบผ่าน pipe แบบนี้
เราจะไม่เห็นข้อความที่ "พิมพ์" สะท้อนกลับมา เพราะไม่ได้มาจากแป้นพิมพ์จริง)

### ข้อควรระวัง

- **ต้องประกาศตัวแปรที่รับค่าให้มีขนาดใหญ่พอสำหรับข้อมูลที่คาดว่าจะได้รับ** เพราะ ACCEPT ก็ปฏิบัติตาม
  กฎการตัด/เติมของ PICTURE เหมือน MOVE ทุกประการ (จะสาธิตปัญหานี้ชัดเจนในขั้นตอนที่ 68)
- ในสภาพแวดล้อมการพัฒนาบางแบบ (เช่น รันผ่าน script อัตโนมัติที่ไม่มี stdin แบบ interactive)
  `ACCEPT` อาจได้รับค่าว่างทันทีโดยไม่รอ ควรทดสอบโปรแกรมที่ใช้ ACCEPT ด้วยการรันจาก terminal จริง
  เสมอเพื่อยืนยันพฤติกรรม interactive ที่ถูกต้อง

### แบบฝึกหัดที่ 63.1

**โจทย์**: จงเขียนโปรแกรมสั้น ๆ ที่ถามหา "เมืองที่คุณอาศัยอยู่" แล้วแสดงผล "YOU LIVE IN: <คำตอบ>"

**เฉลย**:
```cobol
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CITY                PIC X(20).

       PROCEDURE DIVISION.
           DISPLAY "WHICH CITY DO YOU LIVE IN? " WITH NO ADVANCING.
           ACCEPT WS-CITY.
           DISPLAY "YOU LIVE IN: " WS-CITY.
```

---

## ขั้นตอนที่ 64: ACCEPT เข้าฟิลด์ตัวเลข — พฤติกรรมที่แตกต่างจากฟิลด์ข้อความ

### แนวคิด

เมื่อ `ACCEPT` รับข้อมูลเข้าฟิลด์ `PIC X` (alphanumeric) มันจะคัดลอกอักขระดิบตรง ๆ ตามกฎ MOVE
ปกติ แต่เมื่อ `ACCEPT` รับข้อมูลเข้าฟิลด์ตัวเลข (`PIC 9`) GnuCOBOL จะ**แปลความข้อความที่พิมพ์เป็น
ตัวเลข**ก่อนเก็บ ซึ่งมีพฤติกรรมที่น่าสนใจและควรทดสอบให้เห็นด้วยตาตัวเองอย่างละเอียด

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCEPT-NUMERIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AGE                 PIC 9(3).
       01  WS-NEXT-YEAR-AGE       PIC 9(3).

       PROCEDURE DIVISION.
           DISPLAY "ENTER YOUR AGE: " WITH NO ADVANCING.
           ACCEPT WS-AGE.
           DISPLAY "STORED VALUE: [" WS-AGE "]".
           IF WS-AGE IS NUMERIC
               COMPUTE WS-NEXT-YEAR-AGE = WS-AGE + 1
               DISPLAY "NEXT YEAR YOU WILL BE " WS-NEXT-YEAR-AGE
           END-IF.
           STOP RUN.
```

### ทดสอบจริงด้วยข้อมูลนำเข้าหลายรูปแบบ

**กรณีที่ 1 — พิมพ์ตัวเลขปกติ "030"**:
```
ENTER YOUR AGE: STORED VALUE: [030]
NEXT YEAR YOU WILL BE 031
```

**กรณีที่ 2 — พิมพ์ตัวอักษรล้วน "ABC"**:
```
ENTER YOUR AGE: STORED VALUE: [000]
NEXT YEAR YOU WILL BE 001
```

**กรณีที่ 3 — พิมพ์ผสมตัวเลขกับตัวอักษร "12A"**:
```
ENTER YOUR AGE: STORED VALUE: [012]
NEXT YEAR YOU WILL BE 013
```

### อธิบายผลลัพธ์ที่ได้จากการทดสอบจริง

นี่คือจุดสำคัญที่ทดสอบยืนยันด้วย GnuCOBOL จริง: เมื่อ `ACCEPT` รับข้อมูลเข้าฟิลด์ `PIC 9` มันจะ
**พยายามดึงเฉพาะตัวเลขออกมาจากข้อความที่พิมพ์** แทนที่จะคัดลอกอักขระดิบทั้งหมดเหมือนฟิลด์ข้อความ:

- พิมพ์ "030" (ตัวเลขล้วน) → เก็บได้ถูกต้องเป็น `030`
- พิมพ์ "ABC" (ไม่มีตัวเลขเลย) → ตีความไม่ได้ จึงเก็บเป็น `000`
- พิมพ์ "12A" (มีตัวเลขปนตัวอักษร) → ดึงส่วนที่เป็นตัวเลขได้ `12` มาเก็บเป็น `012`

พฤติกรรมนี้เป็น**ส่วนขยายเฉพาะของ GnuCOBOL** ที่พยายามป้องกันโปรแกรม crash เมื่อผู้ใช้ป้อนข้อมูล
ผิดชนิด แต่ไม่ได้หมายความว่าทุกคอมไพเลอร์จะทำงานแบบนี้เหมือนกันเสมอไป

### ข้อควรระวัง

- **ห้ามไว้ใจว่า ACCEPT เข้าฟิลด์ตัวเลขจะ "กรอง" ข้อมูลผิดให้ปลอดภัยเสมอ** แม้ในตัวอย่างนี้
  GnuCOBOL จะแปลง "ABC" เป็น `000` แทนที่จะ crash แต่ค่า `000` ที่ได้อาจเป็นค่าที่ผิดความหมายทาง
  ธุรกิจโดยสิ้นเชิง (อายุ 0 ปี ไม่ใช่ค่าจริงที่ผู้ใช้ตั้งใจป้อน) ควรตรวจสอบข้อมูลนำเข้าอย่างรอบคอบเสมอ
  แทนที่จะพึ่งพากลไกอัตโนมัติของคอมไพเลอร์
- แนวทางที่ปลอดภัยกว่าคือ **รับข้อมูลเข้าฟิลด์ `PIC X` ก่อนเสมอ** แล้วตรวจสอบด้วย `IS NUMERIC`
  ก่อนนำไปแปลงเป็นตัวเลข (ตามที่จะสาธิตในขั้นตอนที่ 67) วิธีนี้ทำให้เราควบคุมพฤติกรรมการตรวจสอบ
  ได้เองอย่างชัดเจน ไม่ต้องพึ่งพากลไกภายในของคอมไพเลอร์ที่อาจแตกต่างกันไปในแต่ละระบบ

### แบบฝึกหัดที่ 64.1

**โจทย์**: จากผลการทดสอบข้างต้น หากผู้ใช้พิมพ์ "-5" เข้าฟิลด์ `WS-AGE PIC 9(3)` (ไม่มี `S` นำหน้า)
คุณคาดว่าผลลัพธ์จะเป็นอย่างไร (ลองเทียบกับกรณี "12A" ที่ทดสอบไว้)

**เฉลยแนวทาง**: จากการทดสอบจริง ผลลัพธ์คือ `005` เพราะฟิลด์ไม่มี `S` รองรับเครื่องหมายลบ
GnuCOBOL จึงดึงเฉพาะตัวเลข "5" ออกมาโดยไม่สนใจเครื่องหมายลบที่พิมพ์นำหน้า ซึ่งยืนยันอีกครั้งว่า
ไม่ควรพึ่งพาพฤติกรรมนี้ในการตรวจสอบความถูกต้องของข้อมูลเชิงธุรกิจ

---

## ขั้นตอนที่ 65: ACCEPT FROM DATE/TIME/DAY-OF-WEEK — รับค่าจากระบบปฏิบัติการ

### แนวคิด

นอกจากรับข้อมูลจากผู้ใช้ `ACCEPT` ยังมีรูปแบบพิเศษ **`ACCEPT ... FROM <special-register>`**
ที่ดึงค่าจากระบบปฏิบัติการโดยตรง โดยไม่ต้องรอผู้ใช้พิมพ์อะไรเลย ค่าที่ดึงได้บ่อยที่สุดคือวันที่และเวลา
ปัจจุบันของเครื่องที่รันโปรแกรม:

| แหล่งข้อมูล | รูปแบบผลลัพธ์ |
|---|---|
| `FROM DATE YYYYMMDD` | ปี-เดือน-วัน แบบ 8 หลัก (ค.ศ. 4 หลัก) |
| `FROM TIME` | ชั่วโมง-นาที-วินาที-เซนติวินาที แบบ 8 หลัก |
| `FROM DAY-OF-WEEK` | ตัวเลข 1 หลัก (1 = จันทร์ ... 7 = อาทิตย์) |

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCEPT-SYSTEM-VALUES-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TODAY-YYYYMMDD      PIC 9(8).
       01  WS-TODAY-GROUP REDEFINES WS-TODAY-YYYYMMDD.
           05  WS-TODAY-YEAR      PIC 9(4).
           05  WS-TODAY-MONTH     PIC 9(2).
           05  WS-TODAY-DAY       PIC 9(2).
       01  WS-CURRENT-TIME        PIC 9(8).
       01  WS-DAY-OF-WEEK         PIC 9(1).

       PROCEDURE DIVISION.
           ACCEPT WS-TODAY-YYYYMMDD FROM DATE YYYYMMDD.
           ACCEPT WS-CURRENT-TIME FROM TIME.
           ACCEPT WS-DAY-OF-WEEK FROM DAY-OF-WEEK.

           DISPLAY "TODAY (YYYYMMDD)   : " WS-TODAY-YYYYMMDD.
           DISPLAY "YEAR-MONTH-DAY     : " WS-TODAY-YEAR "-"
                   WS-TODAY-MONTH "-" WS-TODAY-DAY.
           DISPLAY "CURRENT TIME (HHMMSSHH): " WS-CURRENT-TIME.
           DISPLAY "DAY OF WEEK (1=MONDAY) : " WS-DAY-OF-WEEK.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-TODAY-GROUP REDEFINES WS-TODAY-YYYYMMDD` — เทคนิค `REDEFINES` (จะสอนละเอียดใน Part 022)
  ให้เรามองข้อมูล 8 หลักเดียวกันเป็นกลุ่มปี-เดือน-วันแยกส่วนได้ในตัว โดยไม่ต้องเสียพื้นที่หน่วยความจำ
  เพิ่ม ใช้ในตัวอย่างนี้เพื่อแยกแสดงผลปี เดือน วัน ให้อ่านง่ายขึ้น
- `ACCEPT WS-TODAY-YYYYMMDD FROM DATE YYYYMMDD` — ดึงวันที่ปัจจุบันของระบบมาเก็บในรูปแบบ
  ปี (4 หลัก) เดือน (2 หลัก) วัน (2 หลัก) ติดกัน
- `ACCEPT WS-CURRENT-TIME FROM TIME` — ดึงเวลาปัจจุบันมาเก็บในรูปแบบ ชม.-นาที-วินาที-เซนติวินาที
- `ACCEPT WS-DAY-OF-WEEK FROM DAY-OF-WEEK` — ดึงวันในสัปดาห์เป็นตัวเลข 1-7 โดย 1 คือวันจันทร์
  ตามมาตรฐาน ISO (ต่างจากบางระบบที่นับวันอาทิตย์เป็นวันแรก)

### ผลลัพธ์ที่ได้จากการรันจริง (รันวันที่ 26 กันยายน 2026)

```
TODAY (YYYYMMDD)   : 20260926
YEAR-MONTH-DAY     : 2026-09-26
CURRENT TIME (HHMMSSHH): 20131272
DAY OF WEEK (1=MONDAY) : 6
```

วันที่ 26 กันยายน 2026 ตรงกับวัน**เสาร์** ซึ่งเป็นวันที่ 6 ของสัปดาห์ตามมาตรฐาน ISO (1=จันทร์ ...
6=เสาร์ 7=อาทิตย์) ตรงกับผลลัพธ์ที่ได้จากการรันจริงบนเครื่องพอดี

### ข้อควรระวัง

- **ผลลัพธ์ของ `ACCEPT ... FROM DATE/TIME` ขึ้นอยู่กับนาฬิกาของเครื่องที่รันโปรแกรม** หากเครื่อง
  server ตั้งเขตเวลา (timezone) ไม่ตรงกับที่คาดหวัง ค่าที่ได้อาจคลาดเคลื่อนจากที่ผู้ใช้คาดหวังได้
  ในระบบที่สำคัญมักต้องตรวจสอบการตั้งค่า timezone ของเครื่องให้ถูกต้องก่อนพึ่งพาคำสั่งนี้
- `FROM DATE YYYYMMDD` ให้ปี ค.ศ. 4 หลักเต็ม ต่างจาก `FROM DATE` แบบไม่ระบุรูปแบบ (ซึ่งบางระบบ
  ให้ปีเพียง 2 หลักตามธรรมเนียมเก่าที่เคยสร้างปัญหา Year 2000 หรือ "Y2K" ที่มีชื่อเสียงในอดีต) ควร
  เขียน `YYYYMMDD` ให้ชัดเจนเสมอเพื่อหลีกเลี่ยงความกำกวมนี้

### แบบฝึกหัดที่ 65.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `ACCEPT WS-DATE FROM DATE` (ไม่ระบุ YYYYMMDD) อาจเป็นความเสี่ยง
ในโปรแกรมที่เขียนขึ้นในยุคปัจจุบัน

**เฉลยแนวทาง**: เพราะรูปแบบ `FROM DATE` แบบไม่ระบุ format ในบางคอมไพเลอร์/มาตรฐานเก่าให้ผลลัพธ์
เป็นปีแบบ 2 หลักเท่านั้น (เช่น "26" แทน "2026") ซึ่งเป็นสาเหตุหลักของปัญหา Year 2000 (Y2K) ในอดีต
ที่ระบบไม่สามารถแยกแยะปี 1926 กับ 2026 ได้อย่างถูกต้อง การระบุ `YYYYMMDD` อย่างชัดเจนเสมอจึงเป็น
แนวปฏิบัติที่ปลอดภัยกว่ามากสำหรับโปรแกรมที่เขียนขึ้นใหม่ในปัจจุบัน

---

## ขั้นตอนที่ 66: การรับข้อมูลหลายค่าต่อเนื่องกัน — โปรแกรมแบบฟอร์ม

### แนวคิด

โปรแกรมธุรกิจจริงมักต้องการข้อมูลหลายอย่างจากผู้ใช้ในคราวเดียว เช่น แบบฟอร์มบันทึกคำสั่งซื้อ
ที่ต้องการชื่อสินค้า จำนวน และราคา วิธีทำคือเรียง `DISPLAY` (prompt) คู่กับ `ACCEPT` ต่อเนื่องกัน
ไปทีละฟิลด์ตามลำดับที่ต้องการ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCEPT-MULTIPLE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRODUCT-NAME        PIC X(15).
       01  WS-QUANTITY            PIC 9(4).
       01  WS-UNIT-PRICE          PIC 9(5)V99.
       01  WS-TOTAL-PRICE         PIC 9(9)V99.

       PROCEDURE DIVISION.
           DISPLAY "ENTER PRODUCT NAME: " WITH NO ADVANCING.
           ACCEPT WS-PRODUCT-NAME.

           DISPLAY "ENTER QUANTITY    : " WITH NO ADVANCING.
           ACCEPT WS-QUANTITY.

           DISPLAY "ENTER UNIT PRICE (E.G. 00150.50): "
                   WITH NO ADVANCING.
           ACCEPT WS-UNIT-PRICE.

           COMPUTE WS-TOTAL-PRICE = WS-QUANTITY * WS-UNIT-PRICE.

           DISPLAY "-------- ORDER SUMMARY --------".
           DISPLAY "PRODUCT : " WS-PRODUCT-NAME.
           DISPLAY "QTY     : " WS-QUANTITY.
           DISPLAY "PRICE   : " WS-UNIT-PRICE.
           DISPLAY "TOTAL   : " WS-TOTAL-PRICE.
           STOP RUN.
```

### อธิบายโค้ด

โปรแกรมนี้ถาม-รับข้อมูล 3 ครั้งต่อเนื่องกัน: ชื่อสินค้า (ข้อความ), จำนวน (ตัวเลข), ราคาต่อหน่วย
(ตัวเลขมีทศนิยม) แล้วนำมาคำนวณราคารวมด้วย `COMPUTE` ก่อนสรุปผลทั้งหมดด้วย `DISPLAY` ต่อเนื่อง
หลายบรรทัด รูปแบบนี้คือโครงสร้างพื้นฐานของโปรแกรมป้อนข้อมูล (data entry) แบบ text-mode ที่ใช้กัน
อย่างแพร่หลายในระบบ Mainframe รุ่นเก่า

### ผลลัพธ์การรันจริง (ป้อน "WIRELESS MOUSE", "0010", "00150.50")

```bash
printf "WIRELESS MOUSE\n0010\n00150.50\n" | ./accept-multiple-demo
```

```
ENTER PRODUCT NAME: ENTER QUANTITY    : ENTER UNIT PRICE (E.G. 00150.50): -------- ORDER SUMMARY --------
PRODUCT : WIRELESS MOUSE 
QTY     : 0010
PRICE   : 00150.50
TOTAL   : 000001505.00
```

(หมายเหตุ: เมื่อรันบน terminal จริงแบบ interactive prompt แต่ละอันจะรอผู้ใช้พิมพ์ทีละบรรทัดตามลำดับ
โดยจะเห็นคำตอบที่พิมพ์ปรากฏต่อท้าย prompt ในบรรทัดเดียวกัน — ผลลัพธ์ข้างต้นมาจากการทดสอบผ่าน
pipe ที่ไม่แสดง echo ของข้อมูลนำเข้า)

10 × 150.50 = 1505.00 ตรงกับผลลัพธ์ `TOTAL : 000001505.00` ที่ได้พอดี

### ข้อควรระวัง

- **ลำดับของ `ACCEPT` ต้องตรงกับลำดับที่ผู้ใช้จะป้อนข้อมูลเสมอ** หากสลับลำดับผิด ข้อมูลจะเข้าผิดฟิลด์
  โดยไม่มี error ใด ๆ แจ้งเตือน (เช่น ถ้าสลับ QUANTITY กับ UNIT PRICE ตัวเลขจำนวนจะไปเข้าฟิลด์ราคา
  แทน) เพราะ COBOL ไม่ได้ตรวจสอบความหมายของข้อมูล เพียงแต่ทำตามคำสั่งที่เขียนไว้ตามลำดับ
- ควรมี prompt ที่ชัดเจนบอกรูปแบบที่ต้องการเสมอ (เช่น "E.G. 00150.50") เพราะผู้ใช้ทั่วไปไม่ทราบว่า
  ฟิลด์ตัวเลขที่มีทศนิยมโดยนัย (`V`) ต้องพิมพ์กี่หลักถึงจะถูกต้อง

### แบบฝึกหัดที่ 66.1

**โจทย์**: จงเพิ่มการรับข้อมูล "ชื่อลูกค้า" (`WS-CUSTOMER-NAME PIC X(20)`) เป็นรายการแรกสุดก่อน
ชื่อสินค้าในโปรแกรมข้างต้น

**เฉลย** (โปรแกรมฉบับเต็มที่คอมไพล์และรันจริงแล้ว):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCEPT-MULTIPLE-DEMO-V2.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-NAME       PIC X(20).
       01  WS-PRODUCT-NAME        PIC X(15).
       01  WS-QUANTITY            PIC 9(4).
       01  WS-UNIT-PRICE          PIC 9(5)V99.
       01  WS-TOTAL-PRICE         PIC 9(9)V99.

       PROCEDURE DIVISION.
           DISPLAY "ENTER CUSTOMER NAME: " WITH NO ADVANCING.
           ACCEPT WS-CUSTOMER-NAME.

           DISPLAY "ENTER PRODUCT NAME: " WITH NO ADVANCING.
           ACCEPT WS-PRODUCT-NAME.

           DISPLAY "ENTER QUANTITY    : " WITH NO ADVANCING.
           ACCEPT WS-QUANTITY.

           DISPLAY "ENTER UNIT PRICE (E.G. 00150.50): "
                   WITH NO ADVANCING.
           ACCEPT WS-UNIT-PRICE.

           COMPUTE WS-TOTAL-PRICE = WS-QUANTITY * WS-UNIT-PRICE.

           DISPLAY "-------- ORDER SUMMARY --------".
           DISPLAY "CUSTOMER: " WS-CUSTOMER-NAME.
           DISPLAY "PRODUCT : " WS-PRODUCT-NAME.
           DISPLAY "QTY     : " WS-QUANTITY.
           DISPLAY "PRICE   : " WS-UNIT-PRICE.
           DISPLAY "TOTAL   : " WS-TOTAL-PRICE.
           STOP RUN.
```

ผลลัพธ์จากการรันจริง (ป้อน "JOHN SMITH", "WIRELESS MOUSE", "0010", "00150.50"):

```
-------- ORDER SUMMARY --------
CUSTOMER: JOHN SMITH
PRODUCT : WIRELESS MOUSE
QTY     : 0010
PRICE   : 00150.50
TOTAL   : 000001505.00
```

---

## ขั้นตอนที่ 67: ตรวจสอบความถูกต้องของข้อมูลนำเข้าด้วย IS NUMERIC

### แนวคิด

จากบทเรียนในขั้นตอนที่ 64 เราเห็นแล้วว่าการ ACCEPT ตรงเข้าฟิลด์ตัวเลขอาจทำให้ข้อมูลผิดพลาด
กลายเป็นศูนย์อย่างเงียบ ๆ **แนวปฏิบัติที่ปลอดภัยกว่ามาก** คือ รับข้อมูลเข้าฟิลด์ `PIC X` ก่อนเสมอ
แล้วตรวจสอบด้วย Class Condition `IS NUMERIC` (ที่เรียนไปแล้วใน Part 010 ขั้นตอนที่ 93) ก่อนนำไป
ใช้งานต่อ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCEPT-VALIDATE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-INPUT-TEXT          PIC X(5).
       01  WS-VALID-FLAG          PIC X(1)  VALUE "N".
           88  WS-INPUT-IS-VALID          VALUE "Y".

       PROCEDURE DIVISION.
           DISPLAY "ENTER A 5-DIGIT NUMBER: " WITH NO ADVANCING.
           ACCEPT WS-INPUT-TEXT.

           IF WS-INPUT-TEXT IS NUMERIC
               SET WS-INPUT-IS-VALID TO TRUE
           END-IF.

           IF WS-INPUT-IS-VALID
               DISPLAY "VALID NUMERIC INPUT: " WS-INPUT-TEXT
           ELSE
               DISPLAY "INVALID: '" WS-INPUT-TEXT
                       "' IS NOT A PURE NUMBER."
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-INPUT-TEXT PIC X(5)` — รับข้อมูลเป็นข้อความล้วนก่อนเสมอ **ไม่รับตรงเข้าฟิลด์ตัวเลข** เพื่อ
  หลีกเลี่ยงพฤติกรรม "แปลงอัตโนมัติ" ที่ควบคุมไม่ได้ตามที่พบในขั้นตอนที่ 64
- `IF WS-INPUT-TEXT IS NUMERIC` — ตรวจสอบว่าทุกตัวอักษรในฟิลด์เป็นตัวเลข 0-9 ล้วนหรือไม่
  (ยังคงเป็น `PIC X` แต่ตรวจสอบเนื้อหาข้างในได้ตามที่เรียนใน Part 010)
- ใช้ 88-level `WS-INPUT-IS-VALID` ร่วมกับ `SET ... TO TRUE` ทำให้โค้ดอ่านง่ายเหมือนภาษาอังกฤษ
  ตามเทคนิคที่เรียนมาแล้วใน Part 010 ขั้นตอนที่ 96-98

### ผลลัพธ์การรันจริง

**กรณีป้อน "12345" (ตัวเลขล้วน)**:
```
ENTER A 5-DIGIT NUMBER: VALID NUMERIC INPUT: 12345
```

**กรณีป้อน "12A45" (มีตัวอักษรปน)**:
```
ENTER A 5-DIGIT NUMBER: INVALID: '12A45' IS NOT A PURE NUMBER.
```

### ข้อควรระวัง

- **รูปแบบนี้ (รับเป็น `PIC X` ก่อน แล้วตรวจสอบก่อนแปลง) คือแนวปฏิบัติมาตรฐานในอุตสาหกรรมจริง**
  สำหรับข้อมูลนำเข้าทุกชนิดที่มาจากภายนอกโปรแกรม ไม่ว่าจะจากผู้ใช้ ไฟล์ หรือระบบอื่น ควรยึดหลัก
  "ไม่ไว้ใจข้อมูลนำเข้า" (never trust input) เสมอ
- หลังตรวจสอบว่าเป็น `IS NUMERIC` แล้ว จึงค่อย `MOVE` เข้าฟิลด์ตัวเลขจริงเพื่อนำไปคำนวณต่อได้อย่าง
  ปลอดภัย (เพราะการ MOVE จากฟิลด์ alphanumeric ที่ยืนยันแล้วว่าเป็นตัวเลขล้วน เข้าฟิลด์ numeric
  จะทำงานถูกต้องตามที่คาดหวังเสมอ)

### แบบฝึกหัดที่ 67.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น โดยเพิ่มการ `MOVE WS-INPUT-TEXT TO WS-NUMBER-FIELD` (โดยที่
`WS-NUMBER-FIELD PIC 9(5)`) เฉพาะกรณีที่ข้อมูลถูกต้องเท่านั้น

**เฉลย**:
```cobol
       01  WS-NUMBER-FIELD        PIC 9(5).
       ...
       IF WS-INPUT-IS-VALID
           MOVE WS-INPUT-TEXT TO WS-NUMBER-FIELD
           DISPLAY "VALID NUMERIC INPUT: " WS-NUMBER-FIELD
       ELSE
           DISPLAY "INVALID: '" WS-INPUT-TEXT
                   "' IS NOT A PURE NUMBER."
       END-IF.
```

---

## ขั้นตอนที่ 68: ข้อผิดพลาดที่พบบ่อยของ ACCEPT และ DISPLAY

### แนวคิด

ขั้นตอนนี้รวบรวมกับดักที่พบบ่อยที่สุดเมื่อทำงานกับ `ACCEPT`/`DISPLAY` และพิสูจน์ให้เห็นด้วยการรันจริง
2 กรณีสำคัญ: การพิมพ์ข้อมูลเกินขนาดฟิลด์ และการกด Enter โดยไม่พิมพ์อะไรเลย

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCEPT-PITFALL-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SHORT-FIELD         PIC X(5)  VALUE "XXXXX".
       01  WS-CODE                PIC X(8)  VALUE "OLDVALUE".

       PROCEDURE DIVISION.
           DISPLAY "FIELD BEFORE ACCEPT: [" WS-SHORT-FIELD "]".
           DISPLAY "TYPE MORE THAN 5 CHARACTERS THEN PRESS ENTER:".
           DISPLAY ">> " WITH NO ADVANCING.
           ACCEPT WS-SHORT-FIELD.
           DISPLAY "FIELD AFTER ACCEPT : [" WS-SHORT-FIELD "]".
           DISPLAY "EXTRA CHARACTERS TYPED PAST 5 ARE SILENTLY".
           DISPLAY "DROPPED -- ACCEPT NEVER OVERFLOWS A FIELD.".

           DISPLAY " ".
           DISPLAY "FIELD BEFORE ACCEPT: [" WS-CODE "]".
           DISPLAY "PRESS ENTER WITHOUT TYPING ANYTHING:".
           DISPLAY ">> " WITH NO ADVANCING.
           ACCEPT WS-CODE.
           DISPLAY "FIELD AFTER ACCEPT : [" WS-CODE "]".
           DISPLAY "AN EMPTY LINE OVERWRITES THE WHOLE FIELD".
           DISPLAY "WITH SPACES -- IT DOES NOT KEEP OLDVALUE.".
           STOP RUN.
```

### ผลลัพธ์การรันจริง (ป้อน "ABCDEFGHIJ" แล้วป้อนบรรทัดว่าง)

```
FIELD BEFORE ACCEPT: [XXXXX]
TYPE MORE THAN 5 CHARACTERS THEN PRESS ENTER:
>> FIELD AFTER ACCEPT : [ABCDE]
EXTRA CHARACTERS TYPED PAST 5 ARE SILENTLY
DROPPED -- ACCEPT NEVER OVERFLOWS A FIELD.
 
FIELD BEFORE ACCEPT: [OLDVALUE]
PRESS ENTER WITHOUT TYPING ANYTHING:
>> FIELD AFTER ACCEPT : [        ]
AN EMPTY LINE OVERWRITES THE WHOLE FIELD
WITH SPACES -- IT DOES NOT KEEP OLDVALUE.
```

### อธิบายผลลัพธ์: กับดักที่ 1 — ข้อมูลเกินขนาดฟิลด์

ผู้ใช้พิมพ์ "ABCDEFGHIJ" (10 ตัวอักษร) แต่ `WS-SHORT-FIELD` เก็บได้แค่ 5 ตัว ผลลัพธ์คือเก็บได้เฉพาะ
"ABCDE" (5 ตัวอักษรแรก) ส่วนที่เหลือ "FGHIJ" หายไปโดยไม่มีคำเตือนใด ๆ — ตรงตามกฎ `PIC X` truncation
ที่เรียนใน Part 006 ขั้นตอนที่ 53 ทุกประการ เพียงแต่คราวนี้เกิดจากข้อมูลผู้ใช้พิมพ์ ไม่ใช่ literal ในโค้ด

### อธิบายผลลัพธ์: กับดักที่ 2 — กด Enter โดยไม่พิมพ์อะไรเลย

`WS-CODE` มีค่าเดิมเป็น "OLDVALUE" ก่อน ACCEPT แต่เมื่อผู้ใช้กด Enter ทันทีโดยไม่พิมพ์อะไรเลย
ค่าทั้งหมดในฟิลด์ถูกแทนที่ด้วย**ช่องว่างล้วน** ไม่ใช่การ "คงค่าเดิมไว้" ตามที่มือใหม่หลายคนเข้าใจผิด
— `ACCEPT` เป็นการ**เขียนทับทั้งฟิลด์เสมอ** ไม่ว่าผู้ใช้จะพิมพ์อะไรหรือไม่ก็ตาม

### ข้อควรระวัง

- **ทั้งสองกับดักนี้ไม่ก่อให้เกิด compile error หรือ runtime error ใด ๆ เลย** ทำให้เป็นบั๊กประเภทที่
  ตรวจจับได้ยากที่สุด เพราะโปรแกรมทำงาน "ราบรื่น" แต่ข้อมูลผิดพลาดไปอย่างเงียบ ๆ
- หากต้องการให้ผู้ใช้ "กด Enter เพื่อคงค่าเดิม" ต้องเขียนโค้ดตรวจสอบเองอย่างชัดเจน เช่น ตรวจสอบว่า
  ค่าที่รับมาเป็น `SPACES` หรือไม่ แล้วค่อยตัดสินใจว่าจะคงค่าเดิมหรือไม่ (ไม่ใช่พฤติกรรมอัตโนมัติของ
  ACCEPT)
- ควรออกแบบขนาดฟิลด์ที่รับข้อมูลจากผู้ใช้ให้ใหญ่กว่าที่คาดว่าจะใช้จริงเล็กน้อยเสมอ เพื่อลดความเสี่ยง
  จากการตัดข้อมูลที่สำคัญทิ้งไปแบบไม่รู้ตัว

### แบบฝึกหัดที่ 68.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบ `IF WS-CODE = SPACES` หลัง `ACCEPT WS-CODE` จึงมีประโยชน์
ในการตรวจจับกรณีที่ผู้ใช้กด Enter โดยไม่พิมพ์อะไรเลย

**เฉลยแนวทาง**: เพราะจากการพิสูจน์ข้างต้น เมื่อผู้ใช้กด Enter ทันทีโดยไม่พิมพ์ ACCEPT จะเติมฟิลด์
ทั้งหมดด้วยช่องว่าง (SPACES) ดังนั้นเงื่อนไข `WS-CODE = SPACES` จะเป็นจริงเฉพาะกรณีนี้เท่านั้น
(หรือกรณีที่ผู้ใช้พิมพ์ช่องว่างล้วนซึ่งพบได้ยากกว่ามาก) ทำให้โปรแกรมมเมอร์สามารถเขียนโค้ดแยกกรณี
"ผู้ใช้ไม่ป้อนอะไร" ออกจากกรณี "ผู้ใช้ป้อนข้อมูลจริง" ได้อย่างชัดเจน ก่อนตัดสินใจว่าจะแจ้งเตือนหรือ
ใช้ค่าเริ่มต้นแทน

---

## ขั้นตอนที่ 69: เทคนิคการ Debug ด้วย DISPLAY

### แนวคิด

ก่อนที่จะมีเครื่องมือ debugger แบบกราฟิกที่ซับซ้อน เทคนิคพื้นฐานที่สุดและยังคงใช้กันแพร่หลายที่สุด
ในการหาข้อผิดพลาดของโปรแกรม COBOL (และภาษาโปรแกรมอื่น ๆ อีกมากมาย) คือการแทรก `DISPLAY`
เพื่อ "ส่องดู" ค่าของตัวแปรระหว่างการทำงานของโปรแกรม เรียกเทคนิคนี้ว่า **print debugging** หรือ
**trace debugging**

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DISPLAY-DEBUG-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE               PIC 9(5)V99 VALUE 250.00.
       01  WS-DISCOUNT-RATE       PIC V99     VALUE 0.10.
       01  WS-DISCOUNT-AMOUNT     PIC 9(5)V99.
       01  WS-FINAL-PRICE         PIC 9(5)V99.

       PROCEDURE DIVISION.
      *> DISPLAY statements used purely for tracing/debugging are
      *> a common and effective technique before a full debugger
      *> is available or needed.
           DISPLAY "DEBUG: WS-PRICE BEFORE CALC = " WS-PRICE.

           COMPUTE WS-DISCOUNT-AMOUNT = WS-PRICE * WS-DISCOUNT-RATE.
           DISPLAY "DEBUG: WS-DISCOUNT-AMOUNT   = "
                   WS-DISCOUNT-AMOUNT.

           COMPUTE WS-FINAL-PRICE = WS-PRICE - WS-DISCOUNT-AMOUNT.
           DISPLAY "DEBUG: WS-FINAL-PRICE       = " WS-FINAL-PRICE.

           DISPLAY "----------------------------------------".
           DISPLAY "FINAL PRICE AFTER DISCOUNT: " WS-FINAL-PRICE.
           STOP RUN.
```

### อธิบายโค้ด

โปรแกรมนี้แทรก `DISPLAY "DEBUG: ..."` ไว้หลังทุกขั้นตอนการคำนวณที่สำคัญ ทำให้เราเห็นค่าของตัวแปร
ณ จุดต่าง ๆ ระหว่างการทำงาน แทนที่จะเห็นเพียงผลลัพธ์สุดท้ายเท่านั้น หากพบว่าผลลัพธ์สุดท้ายผิดพลาด
เราจะสามารถไล่ดูค่า DEBUG ทีละบรรทัดเพื่อหาว่าขั้นตอนไหนเริ่มมีค่าผิดไปจากที่คาดหวัง

### ผลลัพธ์ที่ได้จากการรันจริง

```
DEBUG: WS-PRICE BEFORE CALC = 00250.00
DEBUG: WS-DISCOUNT-AMOUNT   = 00025.00
DEBUG: WS-FINAL-PRICE       = 00225.00
----------------------------------------
FINAL PRICE AFTER DISCOUNT: 00225.00
```

250.00 × 0.10 = 25.00 (ส่วนลด) และ 250.00 - 25.00 = 225.00 (ราคาสุทธิ) ตรงตามที่คำนวณไว้ทุกขั้น

### ข้อควรระวัง

- **ควรใส่ prefix ที่ชัดเจนอย่าง "DEBUG:" ในทุกบรรทัดที่ใช้เพื่อการ debug เท่านั้น** เพื่อให้แยกแยะ
  ได้ง่ายจากผลลัพธ์จริงของโปรแกรม และค้นหา/ลบออกได้สะดวกเมื่อ debug เสร็จแล้ว
- **อย่าลืมลบหรือปิดการทำงานของ DISPLAY debug ก่อนนำโปรแกรมขึ้นใช้งานจริง (production)** เพราะ
  ข้อความ debug จำนวนมากอาจทำให้ผลลัพธ์จริงอ่านยากขึ้น หรือในกรณีร้ายแรงอาจรั่วไหลข้อมูลที่ละเอียดอ่อน
  (เช่น รหัสผ่านหรือข้อมูลส่วนบุคคล) ออกไปในบันทึก (log) ของระบบโดยไม่ได้ตั้งใจ
- ในระบบขนาดใหญ่ที่ทันสมัยกว่า มักมีกลไก logging level (DEBUG, INFO, WARNING, ERROR) ที่ควบคุม
  การเปิด/ปิดข้อความ debug ได้โดยไม่ต้องแก้โค้ด แต่หลักการพื้นฐานของการ "ส่องดูค่าตัวแปรระหว่างทาง"
  ยังคงเหมือนกับเทคนิค DISPLAY debug นี้ทุกประการ

### แบบฝึกหัดที่ 69.1

**โจทย์**: จงอธิบายว่าทำไมการแทรก DISPLAY debug ไว้ **ระหว่าง** แต่ละขั้นตอนการคำนวณ (ไม่ใช่แค่
ก่อนกับหลังทั้งหมด) จึงช่วยหาจุดผิดพลาดได้เร็วกว่า

**เฉลยแนวทาง**: เพราะหากมีการคำนวณหลายขั้นตอนต่อเนื่องกันแล้วเกิดข้อผิดพลาด การเห็นเฉพาะค่า
เริ่มต้นกับผลลัพธ์สุดท้ายจะบอกได้แค่ว่า "ผลลัพธ์ผิด" แต่ไม่บอกว่า **ขั้นตอนไหน** ที่เริ่มผิดพลาด
การแทรก DISPLAY หลังทุกขั้นตอนย่อยทำให้เราเห็นค่ากลางทางทุกจุด จึงสามารถระบุได้ทันทีว่าขั้นตอนใด
ให้ผลลัพธ์ที่ไม่ตรงกับที่คาดหวัง ช่วยลดเวลาในการค้นหาสาเหตุของบั๊กได้อย่างมากในโปรแกรมที่มีการ
คำนวณซับซ้อนหลายขั้นตอน

---

## ขั้นตอนที่ 70: โปรแกรมรวบยอด — เครื่องคำนวณ BMI แบบโต้ตอบ

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่ผสาน `DISPLAY` และ `ACCEPT` เข้าด้วยกันอย่างเต็มรูปแบบในสถานการณ์จริง:
เครื่องคำนวณดัชนีมวลกาย (Body Mass Index หรือ BMI) แบบโต้ตอบ ที่รับน้ำหนักและส่วนสูงจากผู้ใช้
คำนวณค่า BMI แล้วจัดกลุ่มผลลัพธ์ตามเกณฑ์มาตรฐาน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BMI-CALCULATOR-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-WEIGHT-KG           PIC 9(3)V99.
       01  WS-HEIGHT-M            PIC 9(1)V99.
       01  WS-BMI                 PIC 9(2)V99.
       01  WS-BMI-CATEGORY        PIC X(20).

       PROCEDURE DIVISION.
           DISPLAY "===== BMI CALCULATOR =====".
           DISPLAY "ENTER WEIGHT IN KG (E.G. 065.50): "
                   WITH NO ADVANCING.
           ACCEPT WS-WEIGHT-KG.

           DISPLAY "ENTER HEIGHT IN METERS (E.G. 1.70): "
                   WITH NO ADVANCING.
           ACCEPT WS-HEIGHT-M.

           COMPUTE WS-BMI =
               WS-WEIGHT-KG / (WS-HEIGHT-M * WS-HEIGHT-M).

           IF WS-BMI < 18.50
               MOVE "UNDERWEIGHT" TO WS-BMI-CATEGORY
           ELSE
               IF WS-BMI < 23.00
                   MOVE "NORMAL" TO WS-BMI-CATEGORY
               ELSE
                   IF WS-BMI < 25.00
                       MOVE "OVERWEIGHT" TO WS-BMI-CATEGORY
                   ELSE
                       MOVE "OBESE" TO WS-BMI-CATEGORY
                   END-IF
               END-IF
           END-IF.

           DISPLAY "----------------------------------------".
           DISPLAY "WEIGHT   : " WS-WEIGHT-KG " KG".
           DISPLAY "HEIGHT   : " WS-HEIGHT-M " M".
           DISPLAY "BMI      : " WS-BMI.
           DISPLAY "CATEGORY : " WS-BMI-CATEGORY.
           STOP RUN.
```

### อธิบายโค้ด

- `WS-WEIGHT-KG PIC 9(3)V99` และ `WS-HEIGHT-M PIC 9(1)V99` — รับน้ำหนัก (สูงสุด 3 หลักจำนวนเต็ม)
  และส่วนสูงเป็นเมตร (1 หลักจำนวนเต็ม เช่น 1.70 เมตร) ทั้งคู่มีทศนิยม 2 ตำแหน่งตามที่เรียนเรื่อง `V`
  ใน Part 006 ขั้นตอนที่ 56
- `COMPUTE WS-BMI = WS-WEIGHT-KG / (WS-HEIGHT-M * WS-HEIGHT-M)` — สูตรมาตรฐานของ BMI คือ
  น้ำหนัก (กก.) หารด้วยส่วนสูง (เมตร) ยกกำลังสอง
- Nested IF สามชั้นจำแนกผลลัพธ์ BMI ตามเกณฑ์มาตรฐานสากล: ต่ำกว่า 18.50 คือ "ผอมกว่าเกณฑ์"
  (Underweight), 18.50-22.99 คือ "ปกติ" (Normal), 23.00-24.99 คือ "น้ำหนักเกิน" (Overweight),
  และตั้งแต่ 25.00 ขึ้นไปคือ "อ้วน" (Obese) — ใช้เทคนิค nested IF ตามที่เรียนใน Part 010 เพราะ
  ยังไม่ได้เรียน `EVALUATE` (ซึ่งจะเรียนใน Part 011 ถัดไป และจะเห็นว่าทำให้โค้ดแบบนี้กระชับขึ้นมาก)

### ผลลัพธ์การรันจริง (ป้อนน้ำหนัก "065.50" ส่วนสูง "1.70")

```bash
printf "065.50\n1.70\n" | ./bmi-calculator-demo
```

```
===== BMI CALCULATOR =====
ENTER WEIGHT IN KG (E.G. 065.50): ENTER HEIGHT IN METERS (E.G. 1.70): ----------------------------------------
WEIGHT   : 065.50 KG
HEIGHT   : 1.70 M
BMI      : 22.66
CATEGORY : NORMAL
```

65.50 ÷ (1.70 × 1.70) = 65.50 ÷ 2.89 = 22.66... ตรงกับผลลัพธ์ `BMI : 22.66` และจัดอยู่ในกลุ่ม
"NORMAL" (18.50-22.99) อย่างถูกต้องตามเกณฑ์

### ข้อควรระวัง

- **โปรแกรมนี้ยังไม่ได้ตรวจสอบว่าผู้ใช้ป้อนข้อมูลถูกต้องหรือไม่** (เช่น ป้อนตัวอักษรแทนตัวเลข หรือ
  ป้อนส่วนสูงเป็น 0 ซึ่งจะทำให้เกิดการหารด้วยศูนย์) ในสถานการณ์จริงควรเพิ่มการตรวจสอบด้วย
  `IS NUMERIC` ตามเทคนิคจากขั้นตอนที่ 67 ก่อนนำค่าไปคำนวณเสมอ
- การหารด้วยศูนย์ (`WS-HEIGHT-M` เป็น 0) จะทำให้เกิด runtime error ที่ทำให้โปรแกรม crash ทันที
  ซึ่งเป็นประเด็นสำคัญที่ต้องระวังเป็นพิเศษเมื่อรับข้อมูลตัวส่วน (denominator) จากผู้ใช้ — ควรตรวจสอบ
  ว่าค่ามากกว่า 0 ก่อนนำไปหารเสมอ (เทคนิคการดักจับข้อผิดพลาดเหล่านี้อย่างละเอียดจะเรียนต่อใน
  Part หลัง ๆ ของหลักสูตร)

### แบบฝึกหัดที่ 70.1

**โจทย์**: จงเพิ่มการตรวจสอบก่อนคำนวณว่า `WS-HEIGHT-M` ต้องมากกว่า 0 หากไม่มากกว่า 0 ให้แสดง
ข้อความ "INVALID HEIGHT, MUST BE GREATER THAN ZERO." แทนการคำนวณ

**เฉลยแนวทาง** (โปรแกรมฉบับเต็มที่คอมไพล์และรันจริงแล้ว):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BMI-CALCULATOR-V2.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-WEIGHT-KG           PIC 9(3)V99.
       01  WS-HEIGHT-M            PIC 9(1)V99.
       01  WS-BMI                 PIC 9(2)V99.
       01  WS-BMI-CATEGORY        PIC X(20).

       PROCEDURE DIVISION.
           DISPLAY "===== BMI CALCULATOR =====".
           DISPLAY "ENTER WEIGHT IN KG (E.G. 065.50): "
                   WITH NO ADVANCING.
           ACCEPT WS-WEIGHT-KG.

           DISPLAY "ENTER HEIGHT IN METERS (E.G. 1.70): "
                   WITH NO ADVANCING.
           ACCEPT WS-HEIGHT-M.

           IF WS-HEIGHT-M > 0
               COMPUTE WS-BMI =
                   WS-WEIGHT-KG / (WS-HEIGHT-M * WS-HEIGHT-M)

               IF WS-BMI < 18.50
                   MOVE "UNDERWEIGHT" TO WS-BMI-CATEGORY
               ELSE
                   IF WS-BMI < 23.00
                       MOVE "NORMAL" TO WS-BMI-CATEGORY
                   ELSE
                       IF WS-BMI < 25.00
                           MOVE "OVERWEIGHT" TO WS-BMI-CATEGORY
                       ELSE
                           MOVE "OBESE" TO WS-BMI-CATEGORY
                       END-IF
                   END-IF
               END-IF

               DISPLAY "----------------------------------------"
               DISPLAY "WEIGHT   : " WS-WEIGHT-KG " KG"
               DISPLAY "HEIGHT   : " WS-HEIGHT-M " M"
               DISPLAY "BMI      : " WS-BMI
               DISPLAY "CATEGORY : " WS-BMI-CATEGORY
           ELSE
               DISPLAY "INVALID HEIGHT, MUST BE GREATER THAN ZERO."
           END-IF.

           STOP RUN.
```

ผลลัพธ์จากการรันจริงกรณีส่วนสูงเป็น 0 (ป้อนน้ำหนัก "065.50" ส่วนสูง "0.00"):

```
INVALID HEIGHT, MUST BE GREATER THAN ZERO.
```

การตรวจสอบก่อนคำนวณเช่นนี้เป็นแนวปฏิบัติมาตรฐานเพื่อป้องกัน runtime error จากการหารด้วยศูนย์
สังเกตว่าคำสั่ง `DISPLAY` ทุกคำสั่งที่อยู่ภายใน `IF...ELSE...END-IF` **ต้องไม่มี period ปิดท้าย**
(ยกเว้นคำสั่งสุดท้ายที่ตามด้วย `END-IF.`) มิฉะนั้น period จะไปตัดจบ `IF` ทั้งก้อนก่อนถึง `ELSE`
ทำให้เกิด syntax error — นี่คือกับดักที่พบได้บ่อยเมื่อ "แปะ" DISPLAY หลายบรรทัดเข้าไปในเงื่อนไข

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้คำสั่ง Input/Output พื้นฐานที่สุดของ COBOL อย่างครบถ้วน:

- `DISPLAY` พื้นฐาน — แสดง literal และตัวแปรผสมกันได้ในคำสั่งเดียว โดยต้องใส่ช่องว่างเองหากต้องการ
  เว้นวรรค
- `WITH NO ADVANCING` — ควบคุมไม่ให้ขึ้นบรรทัดใหม่ ใช้สร้าง prompt ก่อน ACCEPT ที่ดูเป็นธรรมชาติ
- `ACCEPT` พื้นฐาน — รับข้อมูลจากผู้ใช้เข้าฟิลด์ alphanumeric ตามกฎ MOVE ปกติ
- พฤติกรรมพิเศษของ `ACCEPT` เข้าฟิลด์ตัวเลข ที่พยายามดึงตัวเลขออกมาจากข้อความที่พิมพ์ (ทดสอบ
  ยืนยันจริงด้วย GnuCOBOL)
- `ACCEPT ... FROM DATE/TIME/DAY-OF-WEEK` — ดึงค่าจากระบบปฏิบัติการโดยตรง
- การรับข้อมูลหลายค่าต่อเนื่องกันแบบโปรแกรมฟอร์ม
- การตรวจสอบความถูกต้องของข้อมูลนำเข้าด้วย `IS NUMERIC` ก่อนนำไปใช้งานจริง — แนวปฏิบัติที่ปลอดภัย
  กว่าการ ACCEPT ตรงเข้าฟิลด์ตัวเลข
- ข้อผิดพลาดที่พบบ่อยของ ACCEPT: ข้อมูลเกินขนาดฟิลด์ถูกตัดทิ้งเงียบ ๆ และการกด Enter เปล่า ๆ
  จะเขียนทับฟิลด์ด้วยช่องว่างเสมอ ไม่ใช่คงค่าเดิมไว้
- เทคนิคการ debug ด้วย DISPLAY เพื่อส่องดูค่าตัวแปรระหว่างการทำงานของโปรแกรม
- โปรแกรมรวบยอดเครื่องคำนวณ BMI แบบโต้ตอบที่ผสาน DISPLAY, ACCEPT, COMPUTE และ Nested IF
  เข้าด้วยกัน

ตอนนี้โปรแกรมของคุณสามารถ "พูดคุย" กับผู้ใช้งานได้แล้วอย่างแท้จริง ใน Part ถัดไปเราจะกลับไปเจาะลึก
คำสั่งที่คุณใช้มาตลอดหลาย Part ที่ผ่านมาโดยยังไม่ได้เรียนกฎอย่างละเอียด นั่นคือ **`MOVE` Statement**
และกฎการย้ายข้อมูล (MOVE Rules) ที่จะอธิบายทุกกรณีการ MOVE ระหว่างชนิดข้อมูลต่าง ๆ อย่างสมบูรณ์

**[← กลับไป Part 006](part-006-picture-clause.md)** | **[ไปยัง Part 008: MOVE Statement →](part-008-move-statement.md)**
