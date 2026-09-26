# Part 023: ไฟล์ตามลำดับ (Sequential Files) เบื้องต้น (ขั้นตอนที่ 221–230)

## คำนำของ Part นี้

ใน Part 022 เราได้เรียนรู้ `REDEFINES` ซึ่งเป็นเทคนิคสุดท้ายในกลุ่มการจัดการข้อมูลภายในหน่วยความจำ
(Working-Storage) ถึงเวลาแล้วที่เราจะก้าวข้ามขอบเขตของหน่วยความจำไปสู่ **การเก็บข้อมูลถาวรบนดิสก์**
ผ่านแนวคิดที่สำคัญที่สุดอย่างหนึ่งของ COBOL ตั้งแต่วันแรกที่ภาษานี้ถือกำเนิด นั่นคือ **การประมวลผลไฟล์
(File Processing)**

COBOL ถูกออกแบบมาตั้งแต่ต้นเพื่องานประมวลผลข้อมูลธุรกิจปริมาณมาก (Business Data Processing) และหัวใจ
ของงานเหล่านั้นคือการอ่าน-เขียนไฟล์ขนาดใหญ่อย่างมีประสิทธิภาพ ก่อนที่ COBOL จะมีฐานข้อมูลเชิงสัมพันธ์
(Relational Database) ให้ใช้งาน ระบบธุรกิจยุคแรกเก็บข้อมูลทั้งหมดไว้ใน **ไฟล์ตามลำดับ (Sequential File)**
บนเทปแม่เหล็กหรือดิสก์ และแม้ในปี 2026 ระบบ Batch Processing จำนวนมากในธนาคารและองค์กรขนาดใหญ่ก็ยังคง
ใช้ไฟล์ตามลำดับอยู่เป็นประจำสำหรับงานประมวลผลกลางคืน (Batch Job)

Part นี้จะพาคุณเริ่มต้นจากศูนย์: ทำไมโปรแกรมต้องมีไฟล์ ไฟล์ตามลำดับคืออะไร มีวิธีประกาศและใช้งานอย่างไร
ไปจนถึงการเขียน-อ่านไฟล์แบบครบวงจรด้วยตัวเอง เนื้อหาในนี้เป็น**พื้นฐานสำคัญที่สุด**ที่จะถูกใช้ซ้ำตลอด
Part 024–030 ที่จะเจาะลึกเรื่องไฟล์ต่อไป

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้ (คอมเมนต์, ชื่อตัวแปร, ข้อความใน DISPLAY)
> เป็นภาษาอังกฤษล้วน เพราะขีดจำกัดคอลัมน์ 72 ของ Fixed-Format COBOL นับเป็นไบต์ไม่ใช่ตัวอักษร ตามที่อธิบาย
> ไว้ใน Part 002 และ `docs/COURSE-OUTLINE.md` โค้ดทุกตัวอย่างในนี้ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL
> (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 221: ทำไมโปรแกรมต้องมีไฟล์ — จาก WORKING-STORAGE สู่ข้อมูลถาวร

### ปัญหาของ WORKING-STORAGE SECTION

ตลอด Part 001–022 เราเก็บข้อมูลทั้งหมดไว้ใน `WORKING-STORAGE SECTION` ซึ่งเป็นหน่วยความจำ (RAM)
ของโปรแกรมขณะทำงานเท่านั้น ปัญหาคือ **ข้อมูลใน WORKING-STORAGE จะหายไปทันทีที่โปรแกรมจบการทำงาน**
(`STOP RUN`) ลองดูตัวอย่างต่อไปนี้:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP221-NO-PERSIST.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-VISIT-COUNT      PIC 9(4) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           ADD 1 TO WS-VISIT-COUNT.
           DISPLAY "This program has been visited "
               WS-VISIT-COUNT " time(s) since it started.".
           DISPLAY "Run this program again - the number always".
           DISPLAY "resets to 1, because WORKING-STORAGE data".
           DISPLAY "disappears the moment STOP RUN executes.".
           STOP RUN.
```

คอมไพล์และรันด้วยคำสั่ง:

```bash
cobc -x -o step221 step221.cob
./step221
```

**ผลลัพธ์เมื่อรันครั้งแรก:**

```
This program has been visited 0001 time(s) since it started.
Run this program again - the number always
resets to 1, because WORKING-STORAGE data
disappears the moment STOP RUN executes.
```

**ผลลัพธ์เมื่อรันซ้ำอีกครั้ง (ค่ายังคงเป็น 0001 เสมอ):**

```
This program has been visited 0001 time(s) since it started.
Run this program again - the number always
resets to 1, because WORKING-STORAGE data
disappears the moment STOP RUN executes.
```

จะเห็นว่าไม่ว่าจะรันกี่ครั้ง ค่า `WS-VISIT-COUNT` จะเริ่มจาก 0 และกลายเป็น 1 เสมอ เพราะทุกครั้งที่โปรแกรม
เริ่มทำงานใหม่ หน่วยความจำถูกจัดสรรใหม่ทั้งหมดตาม `VALUE 0` ที่กำหนดไว้ ไม่มีการจดจำค่าจากการรันครั้งก่อน

### ทางออก: การเก็บข้อมูลลงไฟล์

เพื่อให้ข้อมูล **"อยู่รอด" ข้ามการรันโปรแกรมแต่ละครั้ง** (Persist) เราต้องเขียนข้อมูลนั้นลง **ไฟล์บนดิสก์**
ซึ่งเป็นสิ่งที่ COBOL รองรับโดยตรงผ่านกลไกที่เรียกว่า **File Processing** ตัวอย่างเช่น ระบบธนาคารต้องเก็บ
ยอดเงินคงเหลือของลูกค้าไว้ในไฟล์ (หรือฐานข้อมูล) ไม่ใช่ใน WORKING-STORAGE เพราะไม่เช่นนั้นยอดเงินทุกคน
จะกลายเป็นศูนย์ทันทีที่ระบบธนาคารรีสตาร์ท!

ตลอด Part 023–030 เราจะเรียนรู้การอ่าน-เขียนไฟล์ประเภทต่าง ๆ โดยเริ่มจากประเภทที่พื้นฐานที่สุดคือ
**ไฟล์ตามลำดับ (Sequential File)**

### ข้อควรระวัง

- อย่าสับสนระหว่าง "ตัวแปรหาย" กับ "โปรแกรม error" — การที่ค่าใน WORKING-STORAGE รีเซ็ตทุกครั้งที่รันใหม่
  เป็น**พฤติกรรมปกติ**ของภาษาโปรแกรมแทบทุกภาษา ไม่ใช่บั๊ก
- การเก็บข้อมูลถาวรไม่ได้จำกัดอยู่แค่ไฟล์เท่านั้น ในโลกจริงอาจเป็นฐานข้อมูล (Part 057–060 จะสอน DB2)
  แต่ไฟล์คือจุดเริ่มต้นที่ COBOL ถนัดที่สุดและใช้มานานที่สุด

### แบบฝึกหัดที่ 221.1

**โจทย์**: จงอธิบายว่าทำไมค่า `WS-VISIT-COUNT` ในตัวอย่างข้างต้นจึงไม่สามารถนับจำนวนครั้งที่โปรแกรมถูก
เรียกใช้งานสะสมได้ (เช่น รันครั้งที่ 5 ควรแสดง 5 แต่กลับแสดง 1 เสมอ)

**เฉลย**: เพราะ `WS-VISIT-COUNT` ถูกประกาศใน `WORKING-STORAGE SECTION` ซึ่งเป็นหน่วยความจำชั่วคราว
ที่ระบบปฏิบัติการจัดสรรให้โปรแกรมใหม่ทุกครั้งที่ถูกเรียกใช้งาน (process ใหม่) และคืนหน่วยความจำนั้นกลับ
เมื่อโปรแกรมจบการทำงานด้วย `STOP RUN` ค่าที่ตั้งไว้ด้วย `VALUE 0` จะถูกกำหนดใหม่ทุกครั้งเสมอ หากต้องการ
นับสะสมข้ามการรัน จำเป็นต้องเก็บค่าล่าสุดไว้ในไฟล์ก่อนจบโปรแกรม แล้วอ่านค่านั้นกลับมาตอนเริ่มโปรแกรมใหม่
ในครั้งถัดไป (เทคนิคนี้จะฝึกในขั้นตอนถัดไปของ Part นี้)

---

## ขั้นตอนที่ 222: ประเภทการจัดระเบียบไฟล์ใน COBOL และ LINE SEQUENTIAL

### ประเภทไฟล์หลัก 3 แบบใน COBOL

COBOL รองรับการจัดระเบียบไฟล์ (File Organization) 3 แบบหลัก ซึ่งกำหนดผ่านข้อกำหนด
`ORGANIZATION IS` ใน `FILE-CONTROL`:

| ประเภท | ORGANIZATION | ลักษณะ | สอนละเอียดใน |
|---|---|---|---|
| **Sequential** | `SEQUENTIAL` / `LINE SEQUENTIAL` | อ่าน-เขียนเรียงลำดับตั้งแต่ต้นจนจบเท่านั้น เหมือนม้วนเทป | Part 023–027 (Part นี้) |
| **Indexed** | `INDEXED` | ค้นหา record ด้วย Key ได้โดยตรง (ไม่ต้องอ่านทีละ record) | Part 028 |
| **Relative** | `RELATIVE` | เข้าถึง record ด้วยหมายเลขลำดับ (Relative Record Number) | Part 029 |

Part นี้เน้นเฉพาะ **ไฟล์ตามลำดับ (Sequential File)** ซึ่งเป็นรูปแบบพื้นฐานที่สุดและใช้งานง่ายที่สุด
จุดเด่นคือความเรียบง่าย ส่วนข้อจำกัดคือ **ต้องอ่านหรือเขียนตามลำดับเท่านั้น** จะกระโดดไปอ่าน record
ที่ 100 โดยไม่อ่าน record ที่ 1–99 ก่อนไม่ได้ (ต่างจาก Indexed File ใน Part 028)

### LINE SEQUENTIAL กับ SEQUENTIAL ต่างกันอย่างไร

GnuCOBOL มีตัวเลือกย่อยสองแบบสำหรับไฟล์ตามลำดับ:

- **`ORGANIZATION IS LINE SEQUENTIAL`**: เก็บข้อมูลเป็น**บรรทัดข้อความ**คล้ายไฟล์ `.txt` ทั่วไป
  แต่ละ record คั่นด้วยอักขระขึ้นบรรทัดใหม่ (`\n`) และ**ตัดช่องว่างท้ายบรรทัดทิ้งอัตโนมัติ**
  เปิดดูด้วย text editor ทั่วไปได้ทันที — **หลักสูตรนี้ใช้แบบนี้เป็นหลักตลอด Part 023–030**
  ตามที่กำหนดไว้ใน `docs/COURSE-OUTLINE.md` เพราะพกพาข้าม platform ได้ง่ายและเข้ากันได้กับ GnuCOBOL ดีที่สุด
- **`ORGANIZATION IS SEQUENTIAL`**: เก็บข้อมูลแบบ record ความยาวคงที่ (Fixed-Length Binary Record)
  ไม่มีตัวคั่นบรรทัด ใกล้เคียงพฤติกรรมไฟล์ตามลำดับบน Mainframe จริงมากกว่า แต่เปิดด้วย text editor
  แล้วจะอ่านไม่รู้เรื่อง

ลองพิสูจน์ความแตกต่างด้วยโค้ดต่อไปนี้ ซึ่งเขียนคำว่า `"ABC"` ลงในไฟล์ที่มี Record ขนาด 10 ไบต์
(`PIC X(10)`) ด้วยทั้งสองแบบ:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP222-ORG-COMPARE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TEXT-FILE ASSIGN TO "TEXT-STYLE.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT BINARY-FILE ASSIGN TO "BINARY-STYLE.DAT"
               ORGANIZATION IS SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  TEXT-FILE.
       01  TEXT-RECORD              PIC X(10).

       FD  BINARY-FILE.
       01  BINARY-RECORD            PIC X(10).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT TEXT-FILE.
           MOVE "ABC" TO TEXT-RECORD.
           WRITE TEXT-RECORD.
           CLOSE TEXT-FILE.

           OPEN OUTPUT BINARY-FILE.
           MOVE "ABC" TO BINARY-RECORD.
           WRITE BINARY-RECORD.
           CLOSE BINARY-FILE.

           DISPLAY "Both files written - compare their raw bytes".
           DISPLAY "with a hex dump tool outside the program.".
           STOP RUN.
```

**ผลลัพธ์การรัน:**

```
Both files written - compare their raw bytes
with a hex dump tool outside the program.
```

ตรวจสอบขนาดไฟล์จริงด้วยคำสั่ง `od -c` (แสดงไบต์ดิบ) และ `wc -c` (นับจำนวนไบต์):

```
--- TEXT-STYLE.DAT (LINE SEQUENTIAL) ---
0000000   A   B   C  \n
ขนาดไฟล์ = 4 ไบต์

--- BINARY-STYLE.DAT (SEQUENTIAL) ---
0000000   A   B   C                            
ขนาดไฟล์ = 10 ไบต์
```

จะเห็นชัดเจนว่า `LINE SEQUENTIAL` ตัดช่องว่างท้าย ("ABC" เหลือแค่ 3 ไบต์) แล้วเติม `\n` รวมเป็น 4 ไบต์
ในขณะที่ `SEQUENTIAL` เก็บ record เต็มความกว้าง 10 ไบต์ตามที่ประกาศไว้ใน `PIC X(10)` เป๊ะ ๆ โดยไม่มี `\n`

### ข้อควรระวัง

- อย่าเปิดไฟล์แบบ `SEQUENTIAL` (ไม่ใช่ LINE) ด้วย text editor ทั่วไปแล้วคาดหวังว่าจะอ่านง่าย เพราะไม่มี
  ตัวแบ่งบรรทัดและอาจมีช่องว่างปะปน
- ตลอดหลักสูตรตั้งแต่นี้ไป หากไม่ได้ระบุเป็นอย่างอื่น ให้ถือว่าใช้ `LINE SEQUENTIAL` เสมอ

### แบบฝึกหัดที่ 222.1

**โจทย์**: ถ้า record มี `PIC X(5)` และเก็บค่า `"HI"` (2 ตัวอักษร) ไฟล์แบบ `LINE SEQUENTIAL`
จะมีขนาดกี่ไบต์ และไฟล์แบบ `SEQUENTIAL` จะมีขนาดกี่ไบต์?

**เฉลย**: `LINE SEQUENTIAL` จะตัดช่องว่างท้ายออกก่อนเขียน เหลือ `"HI"` (2 ไบต์) แล้วเติม `\n` อีก 1 ไบต์
รวมเป็น **3 ไบต์** ส่วน `SEQUENTIAL` จะเก็บเต็มความกว้างของ `PIC X(5)` คือ `"HI   "` (มีช่องว่างเติมเต็ม)
รวมเป็น **5 ไบต์** พอดี โดยไม่มีตัวคั่นบรรทัดเพิ่ม

---

## ขั้นตอนที่ 223: ส่วนประกอบขั้นต่ำในการประกาศไฟล์

การจะใช้ไฟล์ใน COBOL ได้ ต้องประกาศ 3 จุดเสมอ:

1. **`ENVIRONMENT DIVISION` → `INPUT-OUTPUT SECTION` → `FILE-CONTROL`**: บอกว่าไฟล์ชื่ออะไร
   (ในโปรแกรม) ผูกกับไฟล์จริงชื่ออะไร (`ASSIGN TO`) และจัดระเบียบแบบไหน (`ORGANIZATION IS`)
2. **`DATA DIVISION` → `FILE SECTION` → `FD`**: อธิบายโครงสร้างของแต่ละ record ในไฟล์นั้น
   (จะเจาะลึกเต็มรูปแบบใน Part 024)
3. **`PROCEDURE DIVISION`**: คำสั่ง `OPEN`, `WRITE`/`READ`, `CLOSE` เพื่อใช้งานไฟล์จริง

ตัวอย่างโปรแกรมที่เล็กที่สุดเท่าที่จะเป็นไปได้ที่สร้างไฟล์ขึ้นมา 1 ไฟล์:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP223-MINIMAL-FILE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT GREETING-FILE ASSIGN TO "GREETING.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  GREETING-FILE.
       01  GREETING-RECORD         PIC X(40).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT GREETING-FILE.
           MOVE "Hello from a COBOL sequential file." TO
               GREETING-RECORD.
           WRITE GREETING-RECORD.
           CLOSE GREETING-FILE.
           DISPLAY "File GREETING.DAT has been created.".
           STOP RUN.
```

**ผลลัพธ์:**

```
File GREETING.DAT has been created.
```

ไฟล์ `GREETING.DAT` ที่ถูกสร้างขึ้นมีเนื้อหาดังนี้ (เปิดดูได้ด้วย `cat GREETING.DAT`):

```
Hello from a COBOL sequential file.
```

### อธิบายทีละส่วน

- `SELECT GREETING-FILE ASSIGN TO "GREETING.DAT"`: `GREETING-FILE` คือชื่อที่โปรแกรมใช้เรียกไฟล์นี้
  (Internal Name) ส่วน `"GREETING.DAT"` คือชื่อไฟล์จริงบนดิสก์ (External Name) ทั้งสองชื่อไม่จำเป็นต้อง
  เหมือนกัน
- `FD GREETING-FILE.`: ประกาศโครงสร้างของไฟล์ `GREETING-FILE` — สังเกตว่าใช้ชื่อเดียวกับใน `SELECT`
- `01 GREETING-RECORD PIC X(40).`: กำหนดว่าแต่ละ record ในไฟล์นี้มีความกว้าง 40 ตัวอักษร
- `OPEN OUTPUT` / `WRITE` / `CLOSE`: ลำดับการทำงานมาตรฐานที่จะอธิบายละเอียดในขั้นตอนถัดไป

### ข้อควรระวัง

- ชื่อไฟล์ภายใน (`GREETING-FILE`) กับชื่อ record (`GREETING-RECORD`) **ต้องไม่ซ้ำกัน** เพราะ COBOL
  ใช้พื้นที่ชื่อ (namespace) เดียวกันสำหรับชื่อข้อมูลทั้งหมดในโปรแกรม
- ไฟล์ที่ระบุใน `ASSIGN TO` เป็น **relative path** (เช่น `"GREETING.DAT"` ไม่ใช่ `"/home/user/..."`)
  จะถูกสร้าง/ค้นหาใน **โฟลเดอร์ปัจจุบัน (current working directory)** ที่รันโปรแกรมอยู่เสมอ

### แบบฝึกหัดที่ 223.1

**โจทย์**: แก้ไขโปรแกรมข้างต้นให้เขียนข้อความ 2 บรรทัดลงไฟล์แทนที่จะเป็น 1 บรรทัด

**เฉลย**: เพิ่ม `MOVE` และ `WRITE` อีกชุดหนึ่งก่อน `CLOSE`:

```cobol
           OPEN OUTPUT GREETING-FILE.
           MOVE "Hello from a COBOL sequential file." TO
               GREETING-RECORD.
           WRITE GREETING-RECORD.
           MOVE "This is the second line." TO GREETING-RECORD.
           WRITE GREETING-RECORD.
           CLOSE GREETING-FILE.
```

ทุกครั้งที่เรียก `WRITE` จะเป็นการเพิ่ม record ใหม่ 1 record ต่อจากที่เขียนไปแล้ว

---

## ขั้นตอนที่ 224: เขียนไฟล์ครั้งแรก — OPEN OUTPUT, WRITE, CLOSE โดยละเอียด

มาดูตัวอย่างที่สมจริงขึ้น: เขียนรายชื่อลูกค้า 3 คนลงไฟล์ `CUSTOMER.DAT`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP224-WRITE-CUSTOMERS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD         PIC X(30).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.

           MOVE "0001 SOMCHAI JAIDEE" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.

           MOVE "0002 SUDA MEECHAI" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.

           MOVE "0003 PRASERT KAEWTA" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.

           CLOSE CUSTOMER-FILE.
           DISPLAY "Wrote 3 records to CUSTOMER.DAT".
           STOP RUN.
```

**ผลลัพธ์:**

```
Wrote 3 records to CUSTOMER.DAT
```

**เนื้อหาของ `CUSTOMER.DAT` หลังรัน:**

```
0001 SOMCHAI JAIDEE
0002 SUDA MEECHAI
0003 PRASERT KAEWTA
```

### อธิบายลำดับการทำงาน (OPEN-PROCESS-CLOSE Pattern)

ทุกโปรแกรมที่ทำงานกับไฟล์จะมีรูปแบบ 3 ขั้นตอนเสมอ ซึ่งคุณจะเห็นซ้ำ ๆ ตลอดหลักสูตร:

1. **OPEN** — เปิดไฟล์ในโหมดที่ต้องการ (ในที่นี้คือ `OUTPUT` = เขียนไฟล์ใหม่)
2. **PROCESS** — ทำงานกับไฟล์ (ในที่นี้คือ `WRITE` ซ้ำ ๆ เพื่อเพิ่ม record)
3. **CLOSE** — ปิดไฟล์เสมอเมื่อทำงานเสร็จ (สำคัญมาก จะอธิบายเหตุผลใน Part 025)

### สำคัญ: OPEN OUTPUT จะลบไฟล์เดิมทิ้งเสมอ

หากไฟล์ `CUSTOMER.DAT` มีอยู่แล้วก่อนรันโปรแกรมนี้ `OPEN OUTPUT` จะ**ลบเนื้อหาเดิมทั้งหมดทิ้ง**และสร้าง
ไฟล์ใหม่เปล่า ๆ ก่อนเริ่มเขียน (คล้ายการเปิดไฟล์ในโหมด "write" ที่ทับไฟล์เดิมของภาษาอื่น) นี่คือความต่าง
สำคัญจากโหมด `EXTEND` ที่จะเรียนในขั้นตอนที่ 228

### ข้อควรระวัง

- ทุก `WRITE` เขียนได้ครั้งละ**หนึ่ง record เท่านั้น** ต้องการ 3 record ต้องเรียก `WRITE` 3 ครั้ง
- ต้อง `MOVE` ข้อมูลเข้า record **ก่อน** เรียก `WRITE` เสมอ เพราะ `WRITE` จะเขียนค่าปัจจุบันที่อยู่ใน
  record area ณ ขณะนั้นลงไฟล์
- ลืม `CLOSE` อาจทำให้ข้อมูลบางส่วนไม่ถูกเขียนลงดิสก์จริง (buffer ค้าง) — รายละเอียดเต็มอยู่ใน Part 025

### แบบฝึกหัดที่ 224.1

**โจทย์**: เพิ่มลูกค้าคนที่ 4 ชื่อ `"0004 WANNA SRISUK"` เข้าไปในโปรแกรมข้างต้น

**เฉลย**: เพิ่มคู่ `MOVE`/`WRITE` อีกชุดก่อน `CLOSE CUSTOMER-FILE.`:

```cobol
           MOVE "0004 WANNA SRISUK" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.
```

ผลลัพธ์ไฟล์จะมี 4 บรรทัดแทนที่จะเป็น 3 บรรทัด (ตรวจสอบด้วยการรันแล้ว `cat CUSTOMER.DAT`)

---

## ขั้นตอนที่ 225: อ่านไฟล์กลับ — OPEN INPUT, READ...AT END, CLOSE

การอ่านไฟล์กลับมาใช้รูปแบบเดียวกัน แต่เปลี่ยนโหมดเป็น `INPUT` และใช้คำสั่ง `READ` แทน `WRITE`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP225-READ-ONE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD         PIC X(30).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT CUSTOMER-FILE.

           READ CUSTOMER-FILE
               AT END
                   DISPLAY "File is empty!"
               NOT AT END
                   DISPLAY "First record: " CUSTOMER-RECORD
           END-READ.

           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์** (รันหลังจากไฟล์ `CUSTOMER.DAT` มี 4 record จากขั้นตอนที่แล้ว):

```
First record: 0001 SOMCHAI JAIDEE           
```

### อธิบาย AT END / NOT AT END

`READ` แต่ละครั้งจะอ่าน record **ถัดไป** จากตำแหน่งปัจจุบันของไฟล์ (COBOL จำตำแหน่งการอ่านให้อัตโนมัติ
เรียกว่า File Pointer) คำสั่ง `READ` มีสองสาขาเสมอ:

- **`AT END`**: ทำงานเมื่ออ่านไปจนสุดไฟล์แล้ว ไม่มี record เหลือให้อ่าน (เช่น ไฟล์ว่างเปล่า
  หรืออ่านครบทุก record แล้ว)
- **`NOT AT END`**: ทำงานเมื่ออ่าน record สำเร็จ ข้อมูลจะถูกใส่ไว้ใน record area (`CUSTOMER-RECORD`)
  ให้ใช้งานได้ทันที

`END-READ` คือ Scope Terminator (มาตรฐาน COBOL-85) ที่ปิดขอบเขตของคำสั่ง `READ` ให้ชัดเจน
(เทียบเท่าไม่ต้องพึ่ง period อย่างเดียว ตามที่อธิบายไว้ใน Part 001)

### ข้อควรระวัง

- ถ้าลืมเขียน `AT END` compiler จะรายงาน error ทันทีเพราะ `AT END` เป็น**ส่วนบังคับ**ของ `READ`
  สำหรับไฟล์ Sequential (ต่างจากไฟล์ Indexed ที่จะเรียนใน Part 028)
- `CUSTOMER-RECORD` ยังคงมีค่าจาก record สุดท้ายที่อ่านสำเร็จค้างอยู่แม้จะเข้า `AT END` แล้ว (ไม่ถูกล้าง)
  ดังนั้นอย่าใช้ค่านี้ต่อในสาขา `AT END`

### แบบฝึกหัดที่ 225.1

**โจทย์**: แก้ไขโปรแกรมให้แสดงข้อความ `"NO RECORDS FOUND"` แทน `"File is empty!"` เมื่อไฟล์ว่าง

**เฉลย**: แก้ข้อความในสาขา `AT END` โดยตรง:

```cobol
           READ CUSTOMER-FILE
               AT END
                   DISPLAY "NO RECORDS FOUND"
               NOT AT END
                   DISPLAY "First record: " CUSTOMER-RECORD
           END-READ.
```

---

## ขั้นตอนที่ 226: วนอ่านไฟล์ทั้งหมด — รูปแบบมาตรฐานที่ใช้ตลอดหลักสูตร

การอ่านทีละ record เดียวไม่มีประโยชน์มากนัก ในทางปฏิบัติเราต้องการ**วนอ่านทุก record จนสุดไฟล์**
รูปแบบมาตรฐาน (Standard Pattern) ที่ใช้ตลอดทั้งหลักสูตรนี้คือการใช้ **End-of-File Flag** ร่วมกับ
`PERFORM UNTIL`:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP226-READ-LOOP.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD         PIC X(30).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG             PIC X VALUE "N".
           88  END-OF-FILE         VALUE "Y".
       01  WS-RECORD-COUNT         PIC 9(4) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT CUSTOMER-FILE.

           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-RECORD-COUNT
                       DISPLAY "Record " WS-RECORD-COUNT ": "
                           CUSTOMER-RECORD
               END-READ
           END-PERFORM.

           CLOSE CUSTOMER-FILE.
           DISPLAY "Total records read: " WS-RECORD-COUNT.
           STOP RUN.
```

**ผลลัพธ์** (ไฟล์มี 3 record จากขั้นตอนที่ 224):

```
Record 0001: 0001 SOMCHAI JAIDEE           
Record 0002: 0002 SUDA MEECHAI             
Record 0003: 0003 PRASERT KAEWTA           
Total records read: 0003
```

### อธิบาย Pattern นี้อย่างละเอียด — จำรูปแบบนี้ให้แม่น

Pattern นี้คือ**หัวใจของการประมวลผลไฟล์ตามลำดับ**ทั้งหมดใน COBOL จะปรากฏซ้ำแล้วซ้ำเล่าตลอด
Part 023–030 ประกอบด้วย:

1. `01 WS-EOF-FLAG PIC X VALUE "N".` พร้อม `88 END-OF-FILE VALUE "Y".`: สร้างตัวแปรธง (Flag)
   และ Condition Name (จาก Part 010) เพื่อบอกว่า "อ่านจบไฟล์แล้วหรือยัง"
2. `PERFORM UNTIL END-OF-FILE`: วนซ้ำไปเรื่อย ๆ **จนกว่า** ธงจะเปลี่ยนเป็นจริง
3. ในสาขา `AT END`: `SET END-OF-FILE TO TRUE` เปลี่ยนธงเป็น "Y" เพื่อให้ลูปหยุดในรอบถัดไป
4. ในสาขา `NOT AT END`: ประมวลผล record ที่อ่านได้ตามปกติ

จุดสำคัญที่มือใหม่มักพลาดคือ **ต้องเรียก `READ` ครั้งแรกก่อนเข้าลูปด้วยหรือไม่?** ในรูปแบบนี้ที่ใช้
`PERFORM UNTIL` ธรรมดา (ไม่ใช่ `WITH TEST AFTER`) คำสั่ง `READ` ตัวแรกจะอยู่**ภายใน**ลูป ทำให้ลูป
ตรวจสอบเงื่อนไข (`END-OF-FILE` เป็นเท็จเสมอตอนเริ่ม) แล้วเข้าไป `READ` ทันที ถูกต้องสำหรับกรณีทั่วไป

### ข้อควรระวัง

- **ห้ามลืมตั้งค่าเริ่มต้น** `WS-EOF-FLAG` เป็น `"N"` ไม่เช่นนั้นถ้าค่าเริ่มต้นบังเอิญเป็น `"Y"`
  ลูปจะไม่ทำงานเลยแม้แต่ครั้งเดียว
- อย่าลืม `ADD 1 TO WS-RECORD-COUNT` **เฉพาะ**ในสาขา `NOT AT END` เท่านั้น ไม่ใช่นอกคำสั่ง `READ`

### แบบฝึกหัดที่ 226.1

**โจทย์**: จงอธิบายว่าจะเกิดอะไรขึ้นถ้าใส่ `SET END-OF-FILE TO TRUE` ผิดที่ไปอยู่ในสาขา `NOT AT END`
แทนที่จะเป็น `AT END`

**เฉลย**: ลูปจะหยุดทำงานทันทีหลังอ่าน record แรกสำเร็จเพียง record เดียว เพราะธง `END-OF-FILE`
จะถูกตั้งเป็นจริงทั้งที่ยังไม่ถึงจุดสิ้นสุดไฟล์จริง ทำให้โปรแกรมอ่านได้แค่ record เดียวเสมอไม่ว่าไฟล์
จะมีกี่ record ก็ตาม — นี่คือข้อผิดพลาดเชิงตรรกะที่พบบ่อยมากสำหรับผู้เริ่มต้น

---

## ขั้นตอนที่ 227: รวมเขียนและอ่านในโปรแกรมเดียว พร้อมคำนวณสรุปผล

โปรแกรมจริงมักต้อง **สร้างไฟล์ แล้วอ่านกลับมาประมวลผลต่อ** ในรันเดียวกัน ตัวอย่างต่อไปนี้เขียนสินค้า
3 รายการลงไฟล์ แล้วอ่านกลับมาคำนวณราคารวมทั้งหมด:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP227-WRITE-THEN-READ.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODUCT.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  PR-NAME             PIC X(15).
           05  PR-PRICE            PIC 9(5)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG             PIC X VALUE "N".
           88  END-OF-FILE         VALUE "Y".
       01  WS-TOTAL-PRICE          PIC 9(7)V99 VALUE 0.
       01  WS-DISPLAY-PRICE        PIC ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> ------- Write 3 product records into PRODUCT.DAT -------
           OPEN OUTPUT PRODUCT-FILE.
           MOVE "RICE BAG 5KG   " TO PR-NAME.
           MOVE 350.00 TO PR-PRICE.
           WRITE PRODUCT-RECORD.

           MOVE "COOKING OIL 1L " TO PR-NAME.
           MOVE 95.50 TO PR-PRICE.
           WRITE PRODUCT-RECORD.

           MOVE "INSTANT NOODLE " TO PR-NAME.
           MOVE 6.00 TO PR-PRICE.
           WRITE PRODUCT-RECORD.
           CLOSE PRODUCT-FILE.

      *> ------- Read them back and sum the price -------
           OPEN INPUT PRODUCT-FILE.
           PERFORM UNTIL END-OF-FILE
               READ PRODUCT-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY PR-NAME " " PR-PRICE
                       ADD PR-PRICE TO WS-TOTAL-PRICE
               END-READ
           END-PERFORM.
           CLOSE PRODUCT-FILE.

           MOVE WS-TOTAL-PRICE TO WS-DISPLAY-PRICE.
           DISPLAY "Total price of all products: " WS-DISPLAY-PRICE.
           STOP RUN.
```

**ผลลัพธ์:**

```
RICE BAG 5KG    00350.00
COOKING OIL 1L  00095.50
INSTANT NOODLE  00006.00
Total price of all products:     451.50
```

### อธิบายส่วนสำคัญ

- Record ในโปรแกรมนี้เป็น**กลุ่ม (Group Item)** ที่มี 2 field ย่อย (`PR-NAME`, `PR-PRICE`) แทนที่จะเป็น
  `PIC X` ตัวเดียวยาว ๆ เหมือนตัวอย่างก่อนหน้า — เนื้อหาเรื่องการออกแบบ record ที่ดีจะสอนละเอียดใน Part 024
- โปรแกรมเดียวสามารถ `OPEN`/`CLOSE` ไฟล์เดียวกันได้**หลายรอบ** ตราบใดที่ปิดไฟล์ก่อนเปิดใหม่ในโหมดอื่น
  (350.00 + 95.50 + 6.00 = 451.50 ตรงกับผลรวมที่คำนวณได้)

### ข้อควรระวัง

- `WS-TOTAL-PRICE PIC 9(7)V99` ต้องมีจำนวนหลักมากพอที่จะรองรับผลรวมของทุก record ไม่เช่นนั้นจะเกิด
  การ Overflow แบบเงียบ (ค่าตัดหลักบนทิ้งโดยไม่มี error แจ้งเตือน — ทบทวนเรื่อง MOVE Rules จาก Part 008)
- `PIC ZZZ,ZZ9.99` เป็น Numeric-Edited PICTURE สำหรับแสดงผลแบบมีลูกน้ำคั่นหลักพัน (Part 006–009 ทบทวน)

### แบบฝึกหัดที่ 227.1

**โจทย์**: เพิ่มสินค้ารายการที่ 4 ชื่อ `"SOAP BAR      "` ราคา `25.00` แล้วคำนวณราคารวมใหม่

**เฉลย**: เพิ่ม `MOVE`/`WRITE` อีกชุดก่อน `CLOSE PRODUCT-FILE.` ในส่วนเขียนไฟล์:

```cobol
           MOVE "SOAP BAR       " TO PR-NAME.
           MOVE 25.00 TO PR-PRICE.
           WRITE PRODUCT-RECORD.
```

ราคารวมใหม่จะกลายเป็น 451.50 + 25.00 = **476.50** โดยไม่ต้องแก้ไขส่วนอ่านไฟล์เลย เพราะลูป
`PERFORM UNTIL END-OF-FILE` จะอ่านทุก record ที่มีอยู่โดยอัตโนมัติไม่ว่าจะมีกี่ record ก็ตาม

---

## ขั้นตอนที่ 228: ต่อท้ายไฟล์เดิมด้วย OPEN EXTEND

จากขั้นตอนที่ 224 เราทราบแล้วว่า `OPEN OUTPUT` จะลบไฟล์เดิมทิ้งเสมอ แต่บ่อยครั้งเราต้องการ**เพิ่มข้อมูล
ต่อท้าย**ไฟล์ที่มีอยู่แล้วโดยไม่ลบของเดิม (เช่น เพิ่ม log รายวัน) ซึ่งทำได้ด้วย `OPEN EXTEND`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP228-EXTEND-FILE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD         PIC X(30).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG             PIC X VALUE "N".
           88  END-OF-FILE         VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Add one more customer to the END of the existing file
      *> without erasing what is already there.
           OPEN EXTEND CUSTOMER-FILE.
           MOVE "0004 WANNA SRISUK" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

           DISPLAY "After EXTEND, the full file now contains:".
           OPEN INPUT CUSTOMER-FILE.
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " CUSTOMER-RECORD
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์** (รันต่อจากไฟล์ที่มี 3 record จากขั้นตอนที่ 224):

```
After EXTEND, the full file now contains:
  0001 SOMCHAI JAIDEE           
  0002 SUDA MEECHAI             
  0003 PRASERT KAEWTA           
  0004 WANNA SRISUK
```

### สรุปโหมด OPEN ทั้งหมดที่เจอจนถึงตอนนี้

| โหมด | ผลลัพธ์ | ใช้เมื่อ |
|---|---|---|
| `OPEN OUTPUT` | สร้างไฟล์ใหม่ ลบของเดิมทั้งหมด | เริ่มไฟล์ใหม่ตั้งแต่ต้น |
| `OPEN INPUT` | เปิดอ่านอย่างเดียว ห้ามเขียน | ประมวลผล/อ่านข้อมูลที่มีอยู่ |
| `OPEN EXTEND` | เปิดต่อท้ายไฟล์เดิม | เพิ่มข้อมูลโดยไม่ลบของเก่า |

(โหมดที่ 4 คือ `OPEN I-O` สำหรับอ่าน-เขียนพร้อมกัน จะสอนละเอียดใน Part 025)

### ข้อควรระวัง

- `OPEN EXTEND` กับไฟล์ที่**ยังไม่เคยมีมาก่อน**จะสร้างไฟล์ใหม่ให้อัตโนมัติ (พฤติกรรมนี้อาจต่างกันไปตาม
  compiler แต่ละตัว ควรทดสอบเสมอ)
- ห้ามใช้ `OPEN EXTEND` เมื่อต้องการ**แก้ไข**ข้อมูลเดิมที่มีอยู่แล้ว (เช่น เปลี่ยนยอดเงินของลูกค้าคนที่ 2)
  เพราะ `EXTEND` เพิ่มได้แค่ท้ายไฟล์เท่านั้น การแก้ไข record กลางไฟล์ต้องใช้ `REWRITE` ผ่านโหมด `I-O`
  (Part 025)

### แบบฝึกหัดที่ 228.1

**โจทย์**: จงเขียนโปรแกรมที่ต่อท้ายลูกค้า 2 คนเข้าไปในไฟล์เดียวกันในการรันครั้งเดียว

**เฉลย**: เปิดไฟล์ครั้งเดียวด้วย `OPEN EXTEND` แล้วเรียก `WRITE` สองครั้งติดกันก่อน `CLOSE`:

```cobol
           OPEN EXTEND CUSTOMER-FILE.
           MOVE "0005 KANOKWAN SUK" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.
           MOVE "0006 ANAN CHOK" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.
```

ไม่จำเป็นต้อง `CLOSE`/`OPEN EXTEND` ใหม่ระหว่าง `WRITE` แต่ละครั้ง เปิดครั้งเดียวเขียนได้หลาย record

---

## ขั้นตอนที่ 229: จัดการไฟล์ที่อาจไม่มีอยู่จริงด้วย OPTIONAL

### ปัญหา: เปิดไฟล์ที่ไม่มีอยู่จริง

ลองดูว่าจะเกิดอะไรขึ้นถ้าเปิดไฟล์ที่ยังไม่เคยถูกสร้างมาก่อนด้วยโหมด `INPUT` แบบปกติ (ไม่มี `OPTIONAL`):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP229B-NO-OPTIONAL.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT LOG-FILE ASSIGN TO "MAYBE-MISSING.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  LOG-FILE.
       01  LOG-RECORD               PIC X(40).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT LOG-FILE.
           DISPLAY "Opened OK".
           CLOSE LOG-FILE.
           STOP RUN.
```

หากไฟล์ `MAYBE-MISSING.DAT` ไม่มีอยู่จริง โปรแกรมจะ**ล่ม (crash)** ทันทีที่ `OPEN` ด้วยข้อความ:

```
libcob: error: file does not exist (status = 35) for file LOG-FILE ('MAYBE-MISSING.DAT')
```

(ตัวเลข `35` คือ File Status Code ที่แปลว่า "ไฟล์ไม่มีอยู่จริง" — เนื้อหาเต็มเรื่อง File Status
อยู่ใน Part 030)

### ทางแก้: SELECT OPTIONAL

การเพิ่มคำว่า `OPTIONAL` หน้าชื่อไฟล์ใน `SELECT` บอก COBOL ว่า **"ไฟล์นี้อาจจะยังไม่มีอยู่ก็ได้
ถ้าเปิดแล้วไม่เจอ อย่าทำให้โปรแกรมล่ม"**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP229-OPTIONAL-FILE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OPTIONAL LOG-FILE ASSIGN TO "MAYBE-MISSING.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  LOG-FILE.
       01  LOG-RECORD               PIC X(40).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-RECORD-COUNT          PIC 9(4) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Because the SELECT clause has OPTIONAL, opening a file
      *> that does not exist yet does NOT cause a runtime error.
           OPEN INPUT LOG-FILE.

           PERFORM UNTIL END-OF-FILE
               READ LOG-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-RECORD-COUNT
               END-READ
           END-PERFORM.
           CLOSE LOG-FILE.

           IF WS-RECORD-COUNT = 0
               DISPLAY "MAYBE-MISSING.DAT was empty or did not exist."
               DISPLAY "The program kept running safely anyway."
           ELSE
               DISPLAY "Found " WS-RECORD-COUNT " existing record(s)."
           END-IF.
           STOP RUN.
```

**ผลลัพธ์เมื่อไฟล์ยังไม่มีอยู่จริง:**

```
MAYBE-MISSING.DAT was empty or did not exist.
The program kept running safely anyway.
```

**ผลลัพธ์เมื่อไฟล์มีอยู่แล้ว (เช่น คัดลอกไฟล์ `CUSTOMER.DAT` ที่มี 4 record มาทับ):**

```
Found 0004 existing record(s).
```

จะเห็นว่าโปรแกรมเดียวกันทำงานได้ถูกต้องทั้งสองกรณีโดยไม่ต้องแก้โค้ด

### หมายเหตุเรื่องตำแหน่งคำว่า OPTIONAL

สังเกตว่า `OPTIONAL` อยู่**ระหว่าง** `SELECT` กับชื่อไฟล์ (`SELECT OPTIONAL LOG-FILE ...`)
ไม่ใช่ท้ายประโยค การเขียนผิดตำแหน่งจะทำให้เกิด syntax error ทันทีตอนคอมไพล์

### ข้อควรระวัง

- `OPTIONAL` ใช้ได้เฉพาะกับการเปิดไฟล์ในโหมด `INPUT` เท่านั้นในทางปฏิบัติ (เปิดไฟล์ที่ยังไม่มีเพื่อเขียน
  ด้วย `OPTIONAL` ไม่ค่อยมีประโยชน์ เพราะ `OUTPUT`/`EXTEND` จะสร้างไฟล์ใหม่ให้อยู่แล้วถ้าไม่มี)
- แม้ใช้ `OPTIONAL` แล้ว **ต้องตรวจสอบ** `WS-RECORD-COUNT` หรือใช้ `READ...AT END` ตามปกติ เพราะไฟล์ที่
  "เปิดผ่านได้" กับไฟล์ที่ "มีข้อมูลจริง" เป็นคนละเรื่องกัน

### แบบฝึกหัดที่ 229.1

**โจทย์**: อธิบายว่าทำไม field `WS-RECORD-COUNT` ในตัวอย่างข้างต้นจึงเพียงพอที่จะบอกได้ว่าไฟล์
"ไม่มีอยู่จริง" หรือ "มีอยู่แต่ว่างเปล่า" โดยไม่ต้องแยกสองกรณีนี้ออกจากกัน

**เฉลย**: เพราะในทางปฏิบัติ ทั้งสองกรณีนำไปสู่ผลลัพธ์เดียวกันคือ **"ไม่มี record ให้ประมวลผล"**
ซึ่งเป็นสิ่งที่โปรแกรมส่วนใหญ่สนใจจริง ๆ (จะทำอะไรต่อเมื่อไม่มีข้อมูล) มากกว่าการแยกแยะสาเหตุทางเทคนิค
ว่าไฟล์หายไปจริง ๆ หรือแค่ว่างเปล่า หากต้องการแยกสองกรณีนี้ออกจากกันอย่างละเอียด จะต้องตรวจสอบ
FILE STATUS code ทันทีหลัง `OPEN` แทน (สอนใน Part 030)

---

## ขั้นตอนที่ 230: สรุปวงจรชีวิตของไฟล์ — โปรแกรมตัวอย่างครบวงจร

ขั้นตอนสุดท้ายของ Part นี้จะรวบรวมทุกอย่างที่เรียนมา (`OPEN OUTPUT`, `WRITE`, `CLOSE`, `OPEN INPUT`,
`READ` วนลูปพร้อม flag, การคำนวณสรุปผล) เข้าไว้ในโปรแกรมเดียวที่สมบูรณ์ โดยแบ่งเป็น paragraph
ย่อยตามแนวทาง Modularization ที่เรียนใน Part 014:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP230-FULL-CYCLE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT EMPLOYEE-FILE ASSIGN TO "EMPLOYEE.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  EMPLOYEE-FILE.
       01  EMPLOYEE-RECORD.
           05  EMP-NAME             PIC X(15).
           05  EMP-SALARY           PIC 9(6)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-RECORD-COUNT          PIC 9(4) VALUE 0.
       01  WS-TOTAL-SALARY          PIC 9(8)V99 VALUE 0.
       01  WS-AVERAGE-SALARY        PIC 9(8)V99 VALUE 0.
       01  WS-DISPLAY-TOTAL         PIC ZZZ,ZZ9.99.
       01  WS-DISPLAY-AVERAGE       PIC ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-EMPLOYEE-FILE.
           PERFORM READ-AND-SUMMARIZE.
           STOP RUN.

       CREATE-EMPLOYEE-FILE.
           OPEN OUTPUT EMPLOYEE-FILE.
           MOVE "SOMCHAI JAIDEE " TO EMP-NAME.
           MOVE 25000.00 TO EMP-SALARY.
           WRITE EMPLOYEE-RECORD.

           MOVE "SUDA MEECHAI   " TO EMP-NAME.
           MOVE 32000.50 TO EMP-SALARY.
           WRITE EMPLOYEE-RECORD.

           MOVE "PRASERT KAEWTA " TO EMP-NAME.
           MOVE 41500.75 TO EMP-SALARY.
           WRITE EMPLOYEE-RECORD.
           CLOSE EMPLOYEE-FILE.
           DISPLAY "Step 1: EMPLOYEE.DAT created with 3 records.".

       READ-AND-SUMMARIZE.
           OPEN INPUT EMPLOYEE-FILE.
           PERFORM UNTIL END-OF-FILE
               READ EMPLOYEE-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-RECORD-COUNT
                       ADD EMP-SALARY TO WS-TOTAL-SALARY
                       DISPLAY "  " EMP-NAME " earns " EMP-SALARY
               END-READ
           END-PERFORM.
           CLOSE EMPLOYEE-FILE.

           IF WS-RECORD-COUNT > 0
               COMPUTE WS-AVERAGE-SALARY =
                   WS-TOTAL-SALARY / WS-RECORD-COUNT
           END-IF.

           MOVE WS-TOTAL-SALARY TO WS-DISPLAY-TOTAL.
           MOVE WS-AVERAGE-SALARY TO WS-DISPLAY-AVERAGE.
           DISPLAY "Step 2: Read back " WS-RECORD-COUNT " record(s).".
           DISPLAY "Total salary   : " WS-DISPLAY-TOTAL.
           DISPLAY "Average salary : " WS-DISPLAY-AVERAGE.
```

**ผลลัพธ์:**

```
Step 1: EMPLOYEE.DAT created with 3 records.
  SOMCHAI JAIDEE  earns 025000.00
  SUDA MEECHAI    earns 032000.50
  PRASERT KAEWTA  earns 041500.75
Step 2: Read back 0003 record(s).
Total salary   :  98,501.25
Average salary :  32,833.75
```

(25000.00 + 32000.50 + 41500.75 = 98,501.25 หารด้วย 3 คน = 32,833.75 พอดี)

### ทบทวนภาพรวมทั้ง Part 023

โปรแกรมนี้รวมทุกแนวคิดของ Part 023 เข้าด้วยกัน:

- แยกงาน "สร้างไฟล์" และ "อ่าน+สรุปผล" เป็นคนละ paragraph (`CREATE-EMPLOYEE-FILE`,
  `READ-AND-SUMMARIZE`) ตามหลัก Modularization
- ใช้ OPEN-PROCESS-CLOSE pattern ในแต่ละ paragraph อย่างสมบูรณ์
- ใช้ End-of-File Flag + `PERFORM UNTIL` สำหรับวนอ่านทุก record
- คำนวณสรุปผล (ผลรวม, ค่าเฉลี่ย) จากข้อมูลที่อ่านจากไฟล์ — นี่คือรูปแบบงานจริงที่ COBOL ถนัดที่สุด

### ข้อควรระวังโดยรวมของ Part 023 (สรุปข้อผิดพลาดที่พบบ่อยที่สุด)

| ข้อผิดพลาด | ผลที่ตามมา |
|---|---|
| ลืม `AT END` ใน `READ` | Compile error ทันที (บังคับสำหรับไฟล์ Sequential) |
| ลืมตั้งค่าเริ่มต้น EOF-Flag เป็น "N" | ลูปอาจไม่ทำงานเลยถ้าค่าเริ่มต้นบังเอิญตรงกับ "Y" |
| ใช้ `OPEN OUTPUT` ทั้งที่ต้องการต่อท้ายไฟล์ | ข้อมูลเดิมทั้งหมดถูกลบทิ้งโดยไม่มีคำเตือน |
| เปิดไฟล์ที่ไม่มีอยู่จริงโดยไม่ใช้ `OPTIONAL` | โปรแกรมล่มทันที (status = 35) |
| ลืม `CLOSE` ไฟล์ | ข้อมูลอาจไม่ถูกเขียนลงดิสก์ครบถ้วน (รายละเอียดเต็มใน Part 025) |

### แบบฝึกหัดที่ 230.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงเพิ่มความสามารถให้โปรแกรม `STEP230-FULL-CYCLE` แสดงชื่อพนักงานที่มีเงินเดือน**สูงสุด**
ด้วย (นอกเหนือจากผลรวมและค่าเฉลี่ยที่มีอยู่แล้ว)

**เฉลย**: เพิ่มตัวแปรเก็บชื่อและเงินเดือนสูงสุดใน `WORKING-STORAGE`, แล้วเปรียบเทียบทุกครั้งที่อ่าน record
ใหม่ในลูป:

```cobol
       01  WS-MAX-SALARY            PIC 9(6)V99 VALUE 0.
       01  WS-MAX-NAME              PIC X(15) VALUE SPACES.
```

```cobol
                   NOT AT END
                       ADD 1 TO WS-RECORD-COUNT
                       ADD EMP-SALARY TO WS-TOTAL-SALARY
                       IF EMP-SALARY > WS-MAX-SALARY
                           MOVE EMP-SALARY TO WS-MAX-SALARY
                           MOVE EMP-NAME TO WS-MAX-NAME
                       END-IF
                       DISPLAY "  " EMP-NAME " earns " EMP-SALARY
```

แล้วเพิ่ม `DISPLAY "Highest paid  : " WS-MAX-NAME " (" WS-MAX-SALARY ")".` ต่อท้ายรายงานสรุป
ผลลัพธ์ที่ได้คือ `PRASERT KAEWTA` ด้วยเงินเดือน `041500.75` ซึ่งเป็นค่าสูงสุดในไฟล์

---

## สรุปท้ายบท

ใน Part นี้เราได้ปูพื้นฐานเรื่องไฟล์ตามลำดับซึ่งเป็นรากฐานสำคัญที่สุดของการประมวลผลไฟล์ใน COBOL:

- เข้าใจว่าทำไมข้อมูลใน `WORKING-STORAGE` จึงไม่ถาวร และทำไมต้องมีไฟล์
- รู้จักประเภทการจัดระเบียบไฟล์ทั้ง 3 แบบ (Sequential, Indexed, Relative) และความแตกต่างระหว่าง
  `LINE SEQUENTIAL` กับ `SEQUENTIAL` ที่พิสูจน์ด้วยการตรวจสอบไบต์จริงในไฟล์
- ประกาศไฟล์ครบ 3 จุด: `FILE-CONTROL`, `FD`, และการใช้งานใน `PROCEDURE DIVISION`
- เขียนไฟล์ด้วย `OPEN OUTPUT` / `WRITE` / `CLOSE` และอ่านไฟล์ด้วย `OPEN INPUT` / `READ` / `CLOSE`
- รูปแบบมาตรฐาน (Standard Pattern) สำหรับวนอ่านไฟล์ทั้งหมดด้วย End-of-File Flag และ `PERFORM UNTIL`
  ซึ่งจะใช้ซ้ำตลอดทั้งหลักสูตร
- การต่อท้ายไฟล์เดิมด้วย `OPEN EXTEND` และการจัดการไฟล์ที่อาจไม่มีอยู่จริงด้วย `SELECT OPTIONAL`
- เขียนโปรแกรมครบวงจรที่สร้างไฟล์ อ่านกลับ และคำนวณสรุปผลในโปรแกรมเดียว

Part ถัดไป (**Part 024**) จะพาคุณเจาะลึก `FILE SECTION` และ `FD Entry` แบบละเอียดที่สุด ตั้งแต่โครงสร้าง
record ที่ซับซ้อน การใช้ `FILLER`, `REDEFINES` ในไฟล์, ไปจนถึงข้อผิดพลาดสำคัญเรื่องหน่วยความจำที่ยัง
ไม่ได้กำหนดค่า (Uninitialized Data) ซึ่งเป็นสาเหตุของบั๊กที่พบได้บ่อยในโปรแกรมประมวลผลไฟล์จริง

**[← กลับไป Part 022](part-022-redefines.md)** | **[ไปยัง Part 024: FILE SECTION และ FD Entry แบบละเอียด →](part-024-file-section-fd.md)**
