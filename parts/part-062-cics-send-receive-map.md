# Part 062: CICS Commands: SEND, RECEIVE, MAP (ขั้นตอนที่ 611–620)

## คำนำของ Part นี้

Part 061 สอนให้เรารู้จักโครงสร้างพื้นฐานของโปรแกรม CICS: Transaction ID, EIB, COMMAREA, RETURN,
XCTL/LINK — แต่ตัวอย่างทั้งหมดยังไม่ได้แสดง**สิ่งที่ผู้ใช้เห็นบนหน้าจอจริง ๆ** เลย ทั้งที่นี่คือหัวใจ
สำคัญที่สุดของระบบ Online: พนักงานธนาคารต้องเห็นฟอร์มกรอกข้อมูลลูกค้า, ต้องพิมพ์ตัวเลขลงในช่องที่
กำหนด, ต้องกดปุ่ม Enter หรือ PF Key เพื่อยืนยัน — ทั้งหมดนี้คือหน้าที่ของ **BMS (Basic Mapping
Support)** และคำสั่ง `SEND MAP`/`RECEIVE MAP`

Part นี้จะพาคุณเข้าใจกลไกการสื่อสารกับหน้าจอ Terminal ของ CICS ตั้งแต่แนวคิดพื้นฐานของ "Map"
ไปจนถึงการอ่านค่าที่ผู้ใช้พิมพ์กลับมาและตรวจสอบว่าผู้ใช้กดปุ่มอะไร — เป็นการปูพื้นฐานที่จำเป็นก่อนที่
Part 063 จะสอนการ**ออกแบบ BMS Map** อย่างละเอียดครบถ้วน (การนิยาม Physical Map ด้วย Assembler
Macro และ Symbolic Map ที่ generate เป็น COBOL Copybook)

> **ย้ำเตือนเรื่องข้อจำกัดสภาพแวดล้อม**: เนื้อหาทั้งหมดใน Part นี้ยังคงเป็นเทคโนโลยี CICS ที่ต้อง
> อาศัย CICS Transaction Server จริงและ CICS Translator ในการคอมไพล์ ซึ่ง**ไม่มีอยู่ในสภาพแวดล้อม
> GnuCOBOL ของหลักสูตรนี้** (ยืนยันแล้วด้วยการทดสอบจริงใน Part 061 ว่า `cobc` ธรรมดาไม่รู้จัก `EXEC
> CICS` เลย) ตัวอย่างโค้ดทุกตัวอย่างที่มี `EXEC CICS` จึงเป็น **ไวยากรณ์อ้างอิง (Reference Syntax)**
> ตามมาตรฐาน IBM CICS ที่ถูกต้องแม่นยำ แต่**ไม่สามารถคอมไพล์หรือรันได้จริงบนสภาพแวดล้อมนี้** กำกับ
> ด้วยป้าย `⚠️ REFERENCE SYNTAX` เสมอเช่นเดิม

---

## ขั้นตอนที่ 611: Terminal I/O ใน CICS เทียบกับ DISPLAY/ACCEPT ของ Batch

### ทบทวน: โลก Batch ไม่มี "หน้าจอ" ที่แท้จริง

ตลอดหลักสูตรนี้ตั้งแต่เฟส 1 เราใช้ `DISPLAY` และ `ACCEPT` (Part 007) เพื่อโต้ตอบกับผู้ใช้ผ่าน
console/terminal เมื่อรันโปรแกรม COBOL แบบ Interactive แต่เมื่อโปรแกรมเดียวกันนี้ถูกรันเป็น Batch
Job ผ่าน JCL (Part 052) จริง ๆ บน Mainframe คำสั่ง `ACCEPT` จะไม่มี "คนนั่งพิมพ์" อยู่หน้าจอเลย —
ข้อมูลมักมาจากไฟล์หรือ JCL parameter (`SYSIN`) แทน ส่วน `DISPLAY` จะไปปรากฏใน SYSOUT (รายงานที่พิมพ์
หลังงานจบ) ไม่ใช่หน้าจอ real-time

### CICS: หน้าจอเป็นศูนย์กลางของทุกอย่าง

ในทางตรงกันข้าม โปรแกรม CICS ถูกออกแบบมาเพื่อ**โต้ตอบกับผู้ใช้ผ่านหน้าจอ Terminal แบบ real-time**
โดยตรง — Terminal มาตรฐานในยุคที่ CICS ถือกำเนิด (และยังคงใช้อยู่จริงในหลายองค์กรจนถึงปัจจุบัน) คือ
**จอ 3270** ซึ่งเป็นจอแบบ Block-mode (ต่างจากจอสมัยใหม่ที่ส่งทุกตัวอักษรที่พิมพ์ทันที) หมายความว่า
ผู้ใช้จะกรอกข้อมูลในหลายช่องพร้อมกันบนหน้าจอเดียว **แล้วกดปุ่ม Enter/PF Key ครั้งเดียว** เพื่อส่ง
ข้อมูล**ทั้งหน้าจอ**กลับไปที่โปรแกรมทีเดียว ไม่ใช่ทีละตัวอักษรแบบ interactive shell ทั่วไป

### เปรียบเทียบแนวคิด

| แนวคิด Batch (Part 007) | แนวคิด CICS Online |
|---|---|
| `DISPLAY "text"` | `EXEC CICS SEND MAP(...)` หรือ `SEND TEXT(...)` |
| `ACCEPT variable` | `EXEC CICS RECEIVE MAP(...)` |
| ส่ง/รับทีละบรรทัด | ส่ง/รับ**ทั้งหน้าจอ**พร้อมกันในครั้งเดียว |
| ไม่มีแนวคิดเรื่อง "ตำแหน่ง" บนหน้าจอ | มีตำแหน่ง (แถว/คอลัมน์) ที่แน่นอนสำหรับแต่ละฟิลด์ |
| ไม่มีแนวคิดเรื่อง "สี" หรือ "การป้องกันแก้ไข" | มี Attribute Byte ควบคุมสี ความสว่าง การป้องกันแก้ไข ฯลฯ |

### ข้อควรระวัง

- อย่าเข้าใจผิดว่า `SEND MAP` เทียบเท่า `DISPLAY` แบบตรงตัว — `SEND MAP` ส่ง**ทั้งหน้าจอ**ที่มี
  โครงสร้าง (label, ช่องกรอกข้อมูล, สี ฯลฯ) ในครั้งเดียว ในขณะที่ `DISPLAY` ส่งแค่ข้อความบรรทัดเดียว
  เท่านั้น
- จอ 3270 แบบ Block-mode ต่างจากจอ Terminal สมัยใหม่ (เช่น SSH terminal) ที่ส่งทุก keystroke ทันที
  — ความเข้าใจผิดนี้เป็นสาเหตุที่ทำให้ผู้เริ่มต้นสับสนว่าทำไมโปรแกรม CICS ถึง "ไม่รู้" ว่าผู้ใช้พิมพ์
  อะไรจนกว่าจะกด Enter/PF Key

### แบบฝึกหัดที่ 611.1

**โจทย์**: จงอธิบายว่าทำไมจอ 3270 แบบ Block-mode จึงเหมาะกับสถาปัตยกรรม CICS ที่ต้องรองรับผู้ใช้
จำนวนมากพร้อมกัน มากกว่าจอที่ส่งทุก keystroke ทันที

**เฉลย**: เพราะจอแบบ Block-mode ส่งข้อมูล**ทั้งหน้าจอในครั้งเดียว**เมื่อผู้ใช้กด Enter/PF Key เท่านั้น
ทำให้ CICS **ไม่ต้องเปิด Task ค้างไว้รอรับทีละตัวอักษร**ระหว่างที่ผู้ใช้กำลังพิมพ์ (ซึ่งอาจใช้เวลานาน)
สอดคล้องกับแนวคิด Pseudo-conversational Programming ที่เรียนใน Part 061 ขั้นตอนที่ 605 ที่ต้องการ
"ปล่อย" ทรัพยากรคืนให้ CICS ระหว่างรอผู้ใช้ ถ้าจอส่งทุก keystroke ทันทีแบบจอสมัยใหม่ CICS จะต้องเปิด
Task ค้างไว้ตลอดเวลาที่ผู้ใช้กำลังพิมพ์ ทำให้ประหยัดทรัพยากรได้ยากกว่ามาก

---

## ขั้นตอนที่ 612: BMS (Basic Mapping Support) คืออะไร — ภาพรวมก่อนลงรายละเอียดใน Part 063

### แนวคิดของ Map

**Map** คือ "แบบฟอร์มหน้าจอ" ที่กำหนดไว้ล่วงหน้าว่าแต่ละฟิลด์อยู่ตำแหน่งไหน (แถว, คอลัมน์), มีป้าย
กำกับ (label) ว่าอะไร, กว้างกี่ตัวอักษร, สีอะไร, แก้ไขได้หรือไม่ — คล้ายกับการออกแบบฟอร์มในโปรแกรม
Screen Section ที่เรียนใน Part 040 แต่ Screen Section เป็นฟีเจอร์ของ GnuCOBOL ที่ทำงานบน terminal
สมัยใหม่โดยตรง ส่วน Map ของ CICS ทำงานผ่านโปรโตคอลของจอ 3270 โดยเฉพาะ

### สอง "ครึ่ง" ของ BMS Map

```
[ Map Definition (เขียนด้วย Assembler Macro DFHMSD/DFHMDI/DFHMDF) ]
        |
        | ผ่าน BMS Macro Assembler (Assemble ครั้งเดียว)
        v
    แยกเป็น 2 ส่วน:
        |
        +---> (1) Physical Map (Load Module)
        |         เก็บไว้ใน CICS เพื่อใช้ "วาด" หน้าจอจริงตอน SEND MAP
        |
        +---> (2) Symbolic Map (COBOL Copybook)
                  ให้โปรแกรมเมอร์ COPY เข้าโปรแกรม COBOL เพื่ออ้างถึง
                  แต่ละฟิลด์ด้วยชื่อที่มีความหมาย (แทนตำแหน่งแถว-คอลัมน์)
```

จุดสำคัญคือ **Physical Map และ Symbolic Map เกิดจากคำนิยามเดียวกัน** แต่ถูกใช้งานคนละบริบท:
Physical Map ให้ CICS ใช้ตอนวาดหน้าจอจริง ส่วน Symbolic Map ให้โปรแกรมเมอร์ COBOL ใช้อ้างอิงชื่อ
ฟิลด์ในโค้ด (การออกแบบ Map แบบละเอียด รวมถึงการเขียน Assembler Macro เอง จะสอนเต็มรูปแบบใน Part 063)

### ตัวอย่างโครงสร้าง Symbolic Map ที่ Generate มาจาก BMS (ภาพรวมสั้น ๆ)

```cobol
      *> REFERENCE SYNTAX - sample symbolic map structure (abbreviated)
      *> generated automatically from BMS macros - not hand-typed
       01  CUSTMAPI.
           05  FILLER            PIC X(12).
           05  CUSTIDL           PIC S9(4) COMP.
           05  CUSTIDF           PIC X.
           05  FILLER REDEFINES CUSTIDF.
               10  CUSTIDA       PIC X.
           05  CUSTIDI           PIC X(5).
           05  CUSTNAML          PIC S9(4) COMP.
           05  CUSTNAMF          PIC X.
           05  FILLER REDEFINES CUSTNAMF.
               10  CUSTNAMA      PIC X.
           05  CUSTNAMI          PIC X(20).
       01  CUSTMAPO REDEFINES CUSTMAPI.
           05  FILLER            PIC X(12).
           05  CUSTIDO           PIC X(5).
           05  FILLER            PIC X.
           05  CUSTNAMO          PIC X(20).
```

อย่าเพิ่งกังวลกับรายละเอียดครบถ้วนของโครงสร้างนี้ — Part 063 จะอธิบายทุกส่วน (`L`, `F`, `A`, `I`,
`O` suffix) อย่างละเอียด ขั้นตอนนี้เพียงต้องการให้เห็นภาพว่า **Symbolic Map คือ COBOL Copybook ธรรมดา
ที่ `COPY` เข้าโปรแกรมได้** (แนวคิดเดียวกับ `COPY` ที่เรียนใน Part 033)

### ข้อควรระวัง

- Physical Map และ Symbolic Map **ต้อง Assemble/Generate ใหม่ทุกครั้ง**ที่แก้ไขการออกแบบหน้าจอ
  (เช่น ย้ายตำแหน่งฟิลด์ เปลี่ยนความกว้าง) — การแก้ไข Symbolic Map (COBOL Copybook) เองโดยตรงโดยไม่
  แก้ Map Definition ต้นฉบับจะทำให้ทั้งสองไม่ตรงกันและเกิดปัญหาตอนรันจริง
- ชื่อ Map และ Mapset (กลุ่มของ Map หลายตัวที่ Assemble รวมกัน) มีข้อจำกัดความยาวชื่อ (โดยทั่วไป 7-8
  ตัวอักษร) ตามธรรมเนียม Mainframe ยุคเก่าเช่นเดียวกับชื่อโปรแกรมและ Transaction ID

### แบบฝึกหัดที่ 612.1

**โจทย์**: จงอธิบายว่าทำไม BMS Map จึงต้องแยกเป็น Physical Map และ Symbolic Map สองส่วน แทนที่จะ
ใช้แค่ส่วนเดียว

**เฉลย**: เพราะทั้งสองส่วนถูกใช้งานโดย "ผู้บริโภค" คนละฝ่ายที่ต้องการรูปแบบข้อมูลต่างกัน **CICS
Runtime** ต้องการข้อมูลตำแหน่ง/คุณสมบัติของแต่ละฟิลด์ในรูปแบบไบนารีที่มีประสิทธิภาพสำหรับการ "วาด"
หน้าจอจริงผ่านโปรโตคอล 3270 (Physical Map) ในขณะที่ **โปรแกรมเมอร์ COBOL** ต้องการอ้างถึงฟิลด์แต่ละ
ตัวด้วยชื่อที่มีความหมายในภาษา COBOL ปกติ (Symbolic Map ในรูปแบบ Copybook) การแยกสองส่วนนี้ทำให้แต่
ละฝ่ายได้รูปแบบข้อมูลที่เหมาะกับการใช้งานของตัวเอง โดยยังคงสอดคล้องกันเพราะ generate มาจากคำนิยาม
เดียวกัน

---

## ขั้นตอนที่ 613: EXEC CICS SEND MAP — ส่งหน้าจอไปแสดงผล

### ไวยากรณ์พื้นฐาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               SEND MAP('CUSTMAP')
                    MAPSET('CUSTSET')
                    FROM(CUSTMAPO)
                    ERASE
           END-EXEC.
```

### อธิบายแต่ละ Option

| Option | ความหมาย |
|---|---|
| `MAP('CUSTMAP')` | ชื่อ Map ที่จะส่ง (ต้องตรงกับชื่อที่นิยามไว้ตอนสร้าง BMS Map) |
| `MAPSET('CUSTSET')` | ชื่อ Mapset (กลุ่มของ Map) ที่ Map นี้สังกัดอยู่ |
| `FROM(CUSTMAPO)` | ชื่อของ Output area ใน Symbolic Map ที่เก็บ**ค่าจริง**ที่จะแสดง (สังเกตจาก
  ขั้นตอนที่ 612 ว่า `CUSTMAPO` คือกลุ่มฟิลด์ Output ที่ REDEFINES จาก `CUSTMAPI`) |
| `ERASE` | ล้างหน้าจอเดิมทั้งหมดก่อนวาดหน้าจอใหม่ (ถ้าไม่ใส่ ข้อมูลเก่าบนหน้าจออาจค้างปะปนกับข้อมูล
  ใหม่) |

### ตัวอย่างการกำหนดค่าก่อน SEND MAP

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           MOVE "10001" TO CUSTIDO.
           MOVE "SOMCHAI JAIDEE" TO CUSTNAMO.

           EXEC CICS
               SEND MAP('CUSTMAP')
                    MAPSET('CUSTSET')
                    FROM(CUSTMAPO)
                    ERASE
           END-EXEC.
```

โปรแกรมต้อง `MOVE` ค่าที่ต้องการแสดงเข้าฟิลด์ Output (สังเกตชื่อลงท้ายด้วย `O` เช่น `CUSTIDO`,
`CUSTNAMO`) ของ Symbolic Map **ก่อน**เรียก `SEND MAP` เสมอ — เปรียบเทียบได้กับการกำหนดค่าตัวแปรก่อน
`DISPLAY` ในโปรแกรม Batch ทุกประการ เพียงแต่ในกรณีนี้คือการกำหนดค่าให้กับ**หลายฟิลด์พร้อมกัน**ก่อน
ส่งออกไปทั้งหน้าจอในคำสั่งเดียว

### Option อื่นที่พบบ่อย

- `DATAONLY`: ส่งเฉพาะข้อมูล (ไม่ส่งข้อความ label/สี/attribute ซ้ำ) เหมาะกับการ**อัปเดตข้อมูล**บน
  หน้าจอที่วาดโครงร่างไปแล้วครั้งหนึ่ง (เร็วกว่าการส่งทั้งหน้าจอใหม่)
- `CURSOR`: กำหนดตำแหน่งที่ cursor กระพริบจะไปอยู่หลังส่งหน้าจอ (ปกตินิยมชี้ไปที่ฟิลด์แรกที่ผู้ใช้
  ต้องกรอก)
- `FREEKB`: ปลดล็อกคีย์บอร์ดของผู้ใช้ (บางสถานการณ์ CICS จะล็อกคีย์บอร์ดชั่วคราวระหว่างประมวลผล)

### ข้อควรระวัง

- ลืมใส่ `ERASE` ในการส่งหน้าจอใหม่ (ที่มีโครงสร้างต่างจากหน้าจอก่อนหน้า) อาจทำให้ข้อความเก่าและใหม่
  ซ้อนทับกันจนอ่านไม่ออก — ควรใช้ `ERASE` เสมอเมื่อเปลี่ยนไปแสดง Map คนละตัว
- `FROM(...)` ต้องระบุชื่อ Output area ที่ถูกต้อง (ลงท้ายด้วย `O`) **ห้ามใช้ Input area (ลงท้ายด้วย
  `I`)** เพราะ Input area สงวนไว้สำหรับรับค่าที่ผู้ใช้พิมพ์กลับมาเท่านั้น (จะอธิบายในขั้นตอนถัดไป)

### แบบฝึกหัดที่ 613.1

**โจทย์**: จงอธิบายว่าทำไมต้อง `MOVE` ค่าเข้าฟิลด์ Output ก่อนเรียก `SEND MAP` เสมอ จะข้ามขั้นตอนนี้
ได้หรือไม่

**เฉลย**: ไม่ได้ เพราะ `SEND MAP` เป็นเพียงคำสั่งที่บอก CICS ว่า **"นำค่าที่อยู่ในพื้นที่หน่วยความจำ
ที่ระบุใน FROM(...) ไปวาดลงหน้าจอตามโครงร่างของ Map ที่กำหนด"** มันไม่ได้ "รู้" ค่าจริงที่ต้องการแสดง
เอง — ถ้าไม่ `MOVE` ค่าลงในฟิลด์ Output ก่อน ฟิลด์เหล่านั้นจะมีค่าว่างหรือค่าขยะที่ค้างอยู่ใน
หน่วยความจำ (โดยเฉพาะถ้าเป็น Task ใหม่ที่เพิ่งเริ่มทำงาน ค่าจะเป็นค่าเริ่มต้นตาม `VALUE` clause หรือ
ค่าว่างเปล่าตามธรรมชาติของ `WORKING-STORAGE`) ทำให้หน้าจอที่ผู้ใช้เห็นไม่มีข้อมูลที่ต้องการแสดงเลย

---

## ขั้นตอนที่ 614: EXEC CICS RECEIVE MAP — รับข้อมูลที่ผู้ใช้กรอกกลับมา

### ไวยากรณ์พื้นฐาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               RECEIVE MAP('CUSTMAP')
                       MAPSET('CUSTSET')
                       INTO(CUSTMAPI)
           END-EXEC.
```

### อธิบายแนวคิด

`RECEIVE MAP` คือคำสั่งที่ทำงาน**ตรงข้าม**กับ `SEND MAP` — รับข้อมูลที่ผู้ใช้กรอกไว้บนหน้าจอ (หลังกด
Enter/PF Key) กลับเข้าสู่โปรแกรม โดย CICS จะแกะข้อมูลดิบที่ส่งมาจากจอ 3270 แล้ว "แปลง" เป็นค่าในแต่
ละฟิลด์ของ Symbolic Map ให้อัตโนมัติ สังเกตว่า `INTO(CUSTMAPI)` ใช้ **Input area** (ลงท้ายด้วย `I`)
ตรงข้ามกับ `FROM(CUSTMAPO)` ของ `SEND MAP` ที่ใช้ Output area

### ตัวอย่างการใช้งานเต็มรูปแบบ

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       01  WS-RESP-CODE         PIC S9(8) COMP.

           EXEC CICS
               RECEIVE MAP('CUSTMAP')
                       MAPSET('CUSTSET')
                       INTO(CUSTMAPI)
                       RESP(WS-RESP-CODE)
           END-EXEC.

           EVALUATE WS-RESP-CODE
               WHEN DFHRESP(NORMAL)
                   DISPLAY "CUSTOMER ID ENTERED: " CUSTIDI
               WHEN DFHRESP(MAPFAIL)
                   DISPLAY "USER PRESSED ENTER WITH NO DATA."
               WHEN OTHER
                   DISPLAY "UNEXPECTED ERROR: " WS-RESP-CODE
           END-EVALUATE.
```

### L, F, A, I Suffix — โครงสร้างของแต่ละฟิลด์ใน Symbolic Map

ทบทวนจากโครงสร้างในขั้นตอนที่ 612 แต่ละฟิลด์บนหน้าจอจะมีกลุ่มฟิลด์ย่อยหลายตัวใน Symbolic Map:

| Suffix | ความหมาย |
|---|---|
| `L` (Length) | ความยาวจริงของข้อมูลที่ผู้ใช้พิมพ์เข้ามา (เช่น `CUSTIDL`) — ถ้าผู้ใช้ไม่พิมพ์อะไรเลยในช่องนี้ ค่าจะเป็น 0 |
| `F` (Attribute/Flag Byte ดิบ) | Attribute Byte ดิบของฟิลด์ ก่อนแยกเป็น `A` |
| `A` (Attribute) | ค่า Attribute ที่แปลแล้ว ใช้กำหนด/ตรวจสอบคุณสมบัติของฟิลด์ เช่น ป้องกันแก้ไข, สี (จะอธิบายละเอียดในขั้นตอนที่ 617) |
| `I` (Input) | ค่าจริงที่ผู้ใช้พิมพ์เข้ามา ใช้ตอน `RECEIVE MAP` |
| `O` (Output) | ค่าที่โปรแกรมต้องการแสดง ใช้ตอน `SEND MAP` |

### ข้อควรระวัง

- **ต้องตรวจสอบ `xxxxL` (ความยาว) ก่อนใช้ค่าใน `xxxxI` เสมอ** ถ้าผู้ใช้ไม่พิมพ์อะไรในช่องนั้นเลย
  `xxxxL` จะเป็น `0` และ `xxxxI` จะไม่มีค่าที่มีความหมาย (อาจเป็นค่าเก่าที่ค้างอยู่) การไม่ตรวจสอบ
  ความยาวก่อนเป็นสาเหตุบั๊กที่พบบ่อยมากในโปรแกรม CICS ของผู้เริ่มต้น
- Field ที่มีค่า `L = 0` ไม่ได้แปลว่า "ผิดพลาด" เสมอไป อาจหมายถึงผู้ใช้จงใจเว้นว่างไว้ (เช่น ฟิลด์
  optional) ต้องออกแบบตรรกะทางธุรกิจให้ตัดสินใจว่าค่าว่างยอมรับได้หรือไม่ในแต่ละกรณี

### แบบฝึกหัดที่ 614.1

**โจทย์**: หากผู้ใช้กด Enter โดยไม่พิมพ์อะไรเลยในช่อง Customer ID แล้วโปรแกรมเรียก `RECEIVE MAP`
`CUSTIDL` จะมีค่าเท่าไร และโปรแกรมควรทำอย่างไรต่อ

**เฉลย**: `CUSTIDL` จะมีค่าเป็น `0` (ไม่มีตัวอักษรใดถูกพิมพ์เข้าไปในช่องนี้เลย) โปรแกรมควรตรวจสอบ
เงื่อนไข `IF CUSTIDL = 0` ก่อนใช้ค่าใน `CUSTIDI` เสมอ หากเป็น `0` และฟิลด์นี้เป็นข้อมูลบังคับ ควรส่ง
ข้อความแจ้งเตือนกลับไปยังหน้าจอเดิม (ผ่าน `SEND MAP` อีกครั้งพร้อมข้อความ error) ให้ผู้ใช้กรอกใหม่
แทนที่จะพยายามประมวลผลต่อด้วยค่าที่ไม่มีความหมาย

---

## ขั้นตอนที่ 615: โครงสร้าง Symbolic Map แบบละเอียด — ทบทวนก่อนเข้า Part 063

### ภาพรวมของโครงสร้างที่ CICS Generate ให้

ขั้นตอนนี้จะอธิบายโครงสร้าง Symbolic Map ที่แสดงในขั้นตอนที่ 612 อย่างละเอียดยิ่งขึ้น เพื่อเตรียม
ความพร้อมก่อนที่ Part 063 จะสอนการ**ออกแบบ** BMS Map เองตั้งแต่ต้น (การเขียน Assembler Macro
`DFHMSD`, `DFHMDI`, `DFHMDF`)

```cobol
      *> REFERENCE SYNTAX - full symbolic map structure for one field
      *> (CUSTID field only, shown alone for clarity)
       01  CUSTMAPI.
           05  FILLER            PIC X(12).
      *> ---- Structure for the CUSTID field (Input side) ----
           05  CUSTIDL           PIC S9(4) COMP.
      *>    ^ actual length of what the user typed (0 up to max width)
           05  CUSTIDF           PIC X.
      *>    ^ raw attribute byte (rarely used directly)
           05  FILLER REDEFINES CUSTIDF.
               10  CUSTIDA       PIC X.
      *>        ^ the attribute form commonly used in programs (sets field properties)
           05  CUSTIDI           PIC X(5).
      *>    ^ the actual value the user typed (Input)

       01  CUSTMAPO REDEFINES CUSTMAPI.
           05  FILLER            PIC X(12).
      *> ---- Structure for the CUSTID field (Output side) ----
           05  FILLER            PIC X(2).
      *>    ^ overlaps with CUSTIDL+CUSTIDF on the Input side (not used directly)
           05  CUSTIDA           PIC X.
      *>    ^ used to set the attribute before sending (e.g. color, protection)
           05  CUSTIDO           PIC X(5).
      *>    ^ the value the program wants to display (Output)
```

### ทำไมต้องมี REDEFINES ระหว่าง CUSTMAPI กับ CUSTMAPO

สังเกตว่า `CUSTMAPO REDEFINES CUSTMAPI` — ทั้งสองกลุ่มใช้**หน่วยความจำเดียวกัน**เพียงแต่มองผ่าน "มุม
มอง" ที่ต่างกัน (ทบทวนแนวคิด `REDEFINES` จาก Part 022) เหตุผลคือ Input และ Output ของฟิลด์เดียวกัน
ไม่จำเป็นต้องมีโครงสร้างเหมือนกันทุกไบต์ (เช่น Input มี `CUSTIDL` ความยาว 2 ไบต์ที่ Output ไม่ต้องใช้)
การ `REDEFINES` ทำให้ประหยัดหน่วยความจำและสอดคล้องกับวิธีที่โปรโตคอล 3270 จัดเก็บข้อมูลจริง

### ข้อควรระวัง

- **ห้ามแก้ไข Symbolic Map ที่ Generate มาจาก BMS ด้วยมือโดยตรง** เว้นแต่จำเป็นจริง ๆ และเข้าใจ
  โครงสร้างอย่างละเอียด เพราะการแก้ไขที่ไม่ตรงกับ Physical Map จะทำให้เกิดพฤติกรรมที่ไม่คาดคิดตอนรัน
  จริง (ตำแหน่งข้อมูลเลื่อนคลาดเคลื่อน)
- ชื่อฟิลด์ Input/Output จะซ้ำกันในส่วน Attribute (`CUSTIDA` ปรากฏทั้งสองฝั่งในตัวอย่างเต็มรูปแบบ
  บางเวอร์ชันของ BMS) ต้องอ่านเอกสารหรือโครงสร้างจริงที่ generate ออกมาอย่างละเอียด เพราะรายละเอียด
  ปลีกย่อยอาจต่างกันไปตามเวอร์ชันของ CICS ที่ใช้

### แบบฝึกหัดที่ 615.1

**โจทย์**: จงอธิบายว่าทำไม Symbolic Map จึงต้องมีทั้งกลุ่ม Input (`CUSTMAPI`) และกลุ่ม Output
(`CUSTMAPO`) แยกกัน ทั้งที่มันเป็นฟิลด์เดียวกันบนหน้าจอเดียวกัน

**เฉลย**: เพราะข้อมูลที่ไหล **"เข้า"** โปรแกรม (จากผู้ใช้ผ่าน `RECEIVE MAP`) กับข้อมูลที่ไหล
**"ออก"** จากโปรแกรม (ไปยังผู้ใช้ผ่าน `SEND MAP`) มีโครงสร้างข้อมูลประกอบที่ต่างกันในรายละเอียด
(เช่น ฝั่ง Input ต้องมีฟิลด์ความยาว `L` เพื่อบอกว่าผู้ใช้พิมพ์มากี่ตัวอักษร ซึ่งฝั่ง Output ไม่จำเป็น
ต้องมี) การแยกกลุ่ม Input/Output ให้ชัดเจน (แม้จะใช้หน่วยความจำที่ทับซ้อนกันผ่าน `REDEFINES`) ทำให้
โปรแกรมเมอร์อ้างอิงชื่อฟิลด์ที่ตรงกับบริบทการใช้งาน (`RECEIVE` ใช้ชื่อ `I`, `SEND` ใช้ชื่อ `O`) ได้
อย่างชัดเจนไม่สับสน

---

## ขั้นตอนที่ 616: SEND TEXT — ทางเลือกที่ง่ายกว่าสำหรับข้อความไม่มีโครงสร้าง

### เมื่อไรควรใช้ SEND TEXT แทน SEND MAP

`SEND MAP` เหมาะกับหน้าจอที่มีโครงสร้างซับซ้อน (หลายฟิลด์ ตำแหน่งเฉพาะ สีต่าง ๆ) แต่บางสถานการณ์
โปรแกรมแค่ต้องการแสดง**ข้อความธรรมดา**อย่างรวดเร็ว (เช่น ข้อความแจ้งเตือนระบบ, หน้าจอสำหรับ debug)
โดยไม่จำเป็นต้องออกแบบ BMS Map เต็มรูปแบบ — กรณีนี้ใช้ `SEND TEXT` แทนได้

### ไวยากรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       01  WS-MESSAGE-TEXT      PIC X(60)
           VALUE "SYSTEM MAINTENANCE IN PROGRESS. PLEASE TRY AGAIN.".

           EXEC CICS
               SEND TEXT
                    FROM(WS-MESSAGE-TEXT)
                    LENGTH(60)
                    ERASE
           END-EXEC.
```

### เปรียบเทียบ SEND MAP กับ SEND TEXT

| | `SEND MAP` | `SEND TEXT` |
|---|---|---|
| ต้องออกแบบ BMS Map ก่อนไหม | ต้อง (Physical Map + Symbolic Map) | ไม่ต้อง |
| ควบคุมตำแหน่ง/สี/attribute ได้ไหม | ได้อย่างละเอียด | ไม่ได้ (ข้อความเรียงตามปกติ) |
| เหมาะกับ | หน้าจอธุรกิจที่มีฟอร์มซับซ้อน | ข้อความแจ้งเตือนง่าย ๆ, การ debug |
| ความซับซ้อนในการพัฒนา | สูงกว่า (ต้อง Assemble Map ก่อน) | ต่ำกว่ามาก (เขียนโค้ดอย่างเดียว) |

### ข้อควรระวัง

- `SEND TEXT` **ไม่รองรับ** Field ที่ผู้ใช้กรอกกลับมาได้อย่างมีโครงสร้าง — ถ้าต้องการรับข้อมูลกลับมา
  จากผู้ใช้อย่างเป็นระเบียบ (แยกเป็นฟิลด์ ๆ) ยังคงต้องใช้ `SEND MAP`/`RECEIVE MAP` เสมอ
- ในระบบ Production จริงส่วนใหญ่ `SEND TEXT` ใช้น้อยกว่า `SEND MAP` มาก เพราะหน้าจอธุรกิจเกือบทั้งหมด
  ต้องการโครงสร้างฟอร์มที่ชัดเจนเพื่อประสบการณ์ผู้ใช้ที่ดี

### แบบฝึกหัดที่ 616.1

**โจทย์**: จงยกตัวอย่างสถานการณ์ 2 แบบที่ควรใช้ `SEND TEXT` แทน `SEND MAP`

**เฉลย**: (1) **ข้อความแจ้งเตือนระบบชั่วคราว** เช่น "ระบบอยู่ระหว่างปิดปรับปรุง กรุณาลองใหม่ภายหลัง"
ที่ไม่ต้องการโครงสร้างฟอร์มซับซ้อน แค่ต้องการแสดงข้อความให้ผู้ใช้เห็นแล้วจบการทำงาน (2) **หน้าจอ
สำหรับพัฒนา/debug ชั่วคราว** ที่โปรแกรมเมอร์ต้องการแสดงค่าตัวแปรบางตัวอย่างรวดเร็วระหว่างพัฒนาโปรแกรม
โดยยังไม่ต้องการเสียเวลาออกแบบ BMS Map เต็มรูปแบบสำหรับหน้าจอที่อาจถูกลบทิ้งภายหลัง

---

## ขั้นตอนที่ 617: Attribute Byte — ควบคุมคุณสมบัติของแต่ละฟิลด์บนหน้าจอ

### แนวคิดของ Attribute Byte

ทุกฟิลด์บนหน้าจอ 3270 มี **Attribute Byte** กำกับอยู่ (ฟิลด์ `xxxxA` ในโครงสร้าง Symbolic Map จาก
ขั้นตอนที่ 615) ทำหน้าที่ควบคุมคุณสมบัติของฟิลด์นั้น เช่น แก้ไขได้หรือไม่, สว่างหรือมืด, ตัวเลขหรือ
ตัวอักษรทั่วไป

### ค่า Attribute ที่ใช้บ่อย (ผ่าน Copybook มาตรฐาน DFHBMSCA)

```cobol
      *> REFERENCE SYNTAX - standard constants from COPY DFHBMSCA
       EXEC CICS
           SEND MAP('CUSTMAP')
                MAPSET('CUSTSET')
                FROM(CUSTMAPO)
                ERASE
       END-EXEC.
```

```
DFHBMPRF   - Protected, ปกติ (ป้องกันไม่ให้ผู้ใช้แก้ไข, ความสว่างปกติ)
DFHBMUNP   - Unprotected, ปกติ (ผู้ใช้แก้ไขได้, ความสว่างปกติ)
DFHBMUNN   - Unprotected, ตัวเลขเท่านั้น (Numeric-only Unprotected)
DFHBMBRY   - Unprotected, ความสว่างสูง (Bright)
DFHBMDAR   - Protected, ซ่อนไม่ให้เห็น (Dark - ใช้กับรหัสผ่าน)
```

### ตัวอย่างการตั้งค่า Attribute ก่อนส่งหน้าจอ

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           MOVE DFHBMUNN TO CUSTIDA.
      *>   ^ lets the user edit it, and accepts numeric input only

           MOVE DFHBMDAR TO PINCODEA.
      *>   ^ hides the typed value from view (for PIN/password fields)

           EXEC CICS
               SEND MAP('CUSTMAP')
                    MAPSET('CUSTSET')
                    FROM(CUSTMAPO)
                    ERASE
           END-EXEC.
```

### MDT (Modified Data Tag) — บิตพิเศษที่บอกว่าฟิลด์ถูกแก้ไขหรือไม่

นอกจากคุณสมบัติข้างต้น Attribute Byte ยังมีบิตพิเศษเรียกว่า **MDT (Modified Data Tag)** ที่จอ 3270
ใช้บอกว่าผู้ใช้ได้แก้ไขค่าในฟิลด์นั้นหรือไม่นับตั้งแต่ครั้งล่าสุดที่ส่งหน้าจอมา ซึ่งเป็นกลไกสำคัญ
ที่ช่วยประหยัด Bandwidth: เมื่อผู้ใช้กด Enter จอ 3270 จะส่งกลับมา**เฉพาะฟิลด์ที่มี MDT ถูกตั้งค่า
เท่านั้น** (ฟิลด์ที่ไม่ถูกแตะต้องเลยจะไม่ถูกส่งข้อมูลกลับมาซ้ำ) — `RECEIVE MAP` จะจัดการเรื่องนี้ให้
อัตโนมัติโดยที่โปรแกรมเมอร์ไม่ต้องเขียนโค้ดจัดการ MDT เอง

### ข้อควรระวัง

- ฟิลด์ที่ตั้งเป็น `DFHBMPRF` (Protected) ผู้ใช้จะ**แก้ไขไม่ได้เลย** แม้จะพยายามกดคีย์บอร์ดในตำแหน่ง
  นั้นก็ตาม — เหมาะกับ Label/หัวข้อที่ไม่ควรถูกแก้ไข ตรงข้ามกับช่องกรอกข้อมูลที่ต้องเป็น Unprotected
  เสมอ
- ลืมตั้ง Attribute ให้ถูกต้องเป็นสาเหตุบั๊กที่พบบ่อยในการพัฒนาหน้าจอ CICS เช่น ลืมตั้ง
  `DFHBMUNP`/`DFHBMUNN` ให้ช่องกรอกข้อมูล ทำให้ผู้ใช้ไม่สามารถพิมพ์อะไรลงไปได้เลย (เพราะค่าเริ่มต้น
  จาก BMS Map มักเป็น Protected)

### แบบฝึกหัดที่ 617.1

**โจทย์**: จงอธิบายว่า MDT (Modified Data Tag) ช่วยประหยัดทรัพยากรเครือข่ายได้อย่างไร

**เฉลย**: เมื่อผู้ใช้กด Enter หลังกรอกข้อมูลในบางฟิลด์ (ไม่ใช่ทุกฟิลด์บนหน้าจอ) จอ 3270 จะส่งข้อมูล
กลับไปที่ CICS **เฉพาะฟิลด์ที่มีบิต MDT ถูกตั้งค่าไว้เท่านั้น** (คือฟิลด์ที่ผู้ใช้แตะต้อง/แก้ไขจริง ๆ)
โดยไม่ส่งข้อมูลของฟิลด์ที่ไม่ได้ถูกแก้ไขเลยกลับไปซ้ำ (เพราะ CICS ยังมีค่าฟิลด์เหล่านั้นอยู่แล้วจากที่
ส่งไปครั้งก่อน) ทำให้ปริมาณข้อมูลที่ส่งผ่านเครือข่ายต่อการโต้ตอบหนึ่งครั้งน้อยลงมาก ซึ่งสำคัญอย่างยิ่ง
ในยุคที่ CICS ถือกำเนิด (แบนด์วิดท์เครือข่ายราคาแพงและจำกัดมาก) และยังคงมีประโยชน์ในแง่ประสิทธิภาพ
จนถึงปัจจุบัน

---

## ขั้นตอนที่ 618: AID Keys — ตรวจสอบว่าผู้ใช้กดปุ่มอะไร

### แนวคิดของ AID (Attention Identifier)

เมื่อผู้ใช้กรอกข้อมูลบนหน้าจอ 3270 เสร็จแล้ว จะต้องกดปุ่มใดปุ่มหนึ่งเพื่อส่งข้อมูลกลับไปยัง CICS —
ปุ่มเหล่านี้เรียกว่า **AID Keys (Attention Identifier Keys)** และ CICS จะบันทึกว่าผู้ใช้กดปุ่มไหน
ไว้ในฟิลด์ `EIBAID` ของ EIB (ที่แนะนำใน Part 061 ขั้นตอนที่ 604)

### ปุ่มมาตรฐานที่พบบ่อย

| ปุ่ม | ค่าคงที่ (จาก COPY DFHAID) | ความหมายทั่วไป |
|---|---|---|
| Enter | `DFHENTER` | ยืนยันข้อมูล/ดำเนินการต่อ |
| PF1-PF24 | `DFHPF1` ถึง `DFHPF24` | ปุ่มฟังก์ชันที่กำหนดความหมายเอง (เช่น PF3 มักใช้แทน "ย้อนกลับ", PF12 มักใช้แทน "ยกเลิก") |
| Clear | `DFHCLEAR` | ล้างหน้าจอ (มักใช้แทน "ยกเลิกทั้งหมด") |
| PA1-PA3 | `DFHPA1` ถึง `DFHPA3` | ปุ่มที่ไม่ส่งข้อมูลกลับมาเลย (ใช้แค่แจ้งเหตุการณ์ ไม่ส่ง COMMAREA/ข้อมูลหน้าจอ) |

### ตัวอย่างการตรวจสอบ EIBAID

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               RECEIVE MAP('CUSTMAP')
                       MAPSET('CUSTSET')
                       INTO(CUSTMAPI)
           END-EXEC.

           EVALUATE EIBAID
               WHEN DFHENTER
                   PERFORM PROCESS-DATA-PARA
               WHEN DFHPF3
                   PERFORM RETURN-TO-MENU-PARA
               WHEN DFHPF12
                   PERFORM CANCEL-TRANSACTION-PARA
               WHEN DFHCLEAR
                   PERFORM CLEAR-SCREEN-PARA
               WHEN OTHER
                   MOVE "INVALID KEY. PLEASE USE ENTER OR PF3." 
                       TO WS-ERROR-MESSAGE
                   PERFORM RE-DISPLAY-PARA
           END-EVALUATE.
```

### ข้อควรระวัง

- ปุ่ม PA1-PA3 **ไม่ส่งข้อมูลหน้าจอกลับมาเลย** (ไม่มีการ transmit field ใด ๆ) ดังนั้นถ้าโปรแกรม
  เรียก `RECEIVE MAP` หลังผู้ใช้กดปุ่มนี้จะได้ `SQLCODE`/Response ที่บ่งบอกว่าไม่มีข้อมูล
  (`MAPFAIL`) — ควรตรวจสอบ `EIBAID` **ก่อน** ตัดสินใจว่าจะเรียก `RECEIVE MAP` หรือไม่ในบางกรณี
- ความหมายของปุ่ม PF Key แต่ละปุ่ม (PF3 = ย้อนกลับ, PF12 = ยกเลิก ฯลฯ) เป็น**ธรรมเนียมที่แต่ละ
  องค์กรกำหนดขึ้นเอง**ไม่ใช่มาตรฐานตายตัวของ CICS แม้จะมีธรรมเนียมที่นิยมใช้กันทั่วไปในอุตสาหกรรม
  ก็ตาม ควรตรวจสอบเอกสารการออกแบบของแต่ละระบบเสมอ

### แบบฝึกหัดที่ 618.1

**โจทย์**: จงอธิบายว่าทำไมปุ่ม PA1-PA3 จึงไม่ส่งข้อมูลหน้าจอกลับมาเลย ต่างจากปุ่ม Enter หรือ PF Key
ทั่วไป

**เฉลย**: ปุ่ม PA (Program Attention) ถูกออกแบบมาให้เป็น **"สัญญาณแจ้งเหตุการณ์" ล้วน ๆ** โดยไม่มี
เจตนาส่งข้อมูลใด ๆ กลับไปพร้อมกัน (ต่างจาก Enter/PF Key ที่มักใช้คู่กับการกรอกข้อมูลแล้วส่งกลับ)
ในทางเทคนิคของโปรโตคอล 3270 ปุ่มกลุ่มนี้ถูกกำหนดให้ทำงานแบบ "แจ้งแค่ว่ากดปุ่มอะไร" โดยไม่ส่ง field
data ตามมาด้วย ซึ่งเหมาะกับการใช้งานเช่น "ปุ่มด่วน" ที่ต้องการความเร็วสูงสุดในการแจ้งเหตุการณ์โดยไม่
ต้องรอส่งข้อมูลฟิลด์ทั้งหมด

---

## ขั้นตอนที่ 619: MAPFAIL และการจัดการ Error ของ SEND/RECEIVE

### เมื่อ RECEIVE MAP ไม่พบข้อมูลเลย — MAPFAIL

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       01  WS-RESP-CODE         PIC S9(8) COMP.
       01  WS-RESP2-CODE        PIC S9(8) COMP.

           EXEC CICS
               RECEIVE MAP('CUSTMAP')
                       MAPSET('CUSTSET')
                       INTO(CUSTMAPI)
                       RESP(WS-RESP-CODE)
                       RESP2(WS-RESP2-CODE)
           END-EXEC.

           EVALUATE WS-RESP-CODE
               WHEN DFHRESP(NORMAL)
                   PERFORM PROCESS-INPUT-PARA
               WHEN DFHRESP(MAPFAIL)
                   DISPLAY "NO DATA - USER PRESSED PA KEY OR CLEAR."
                   PERFORM RE-DISPLAY-EMPTY-SCREEN-PARA
               WHEN OTHER
                   DISPLAY "UNEXPECTED RESP: " WS-RESP-CODE
                       " RESP2: " WS-RESP2-CODE
                   EXEC CICS ABEND ABCODE('MAP1') END-EXEC
           END-EVALUATE.
```

`MAPFAIL` เกิดขึ้นเมื่อ `RECEIVE MAP` ไม่ได้รับข้อมูลใด ๆ กลับมาเลย — สถานการณ์ที่พบบ่อยที่สุดคือ
ผู้ใช้กดปุ่ม PA Key (ขั้นตอนที่ 618) หรือกด CLEAR โดยไม่มีการกรอกข้อมูลใด ๆ ก่อน โปรแกรมที่ดีต้อง
รองรับสถานการณ์นี้เสมอ ไม่เช่นนั้นจะเกิด Runtime Abend ที่ไม่คาดคิด

### RESP2 — ข้อมูลเสริมเมื่อ RESP ไม่เพียงพอ

`RESP2` เป็นฟิลด์เสริมที่บางคำสั่ง CICS ใช้ให้รายละเอียดเพิ่มเติมเมื่อ `RESP` เพียงอย่างเดียวไม่พอ
สื่อความหมาย (คล้ายกับที่ DB2 มี `SQLCODE` และ `SQLSTATE`/`SQLERRMC` เพื่อรายละเอียดเพิ่มเติมกว่า
ตัวเลขเดี่ยว ๆ)

### รูปแบบการป้องกัน Error ทั้งหมดใน SEND/RECEIVE

หลักการทั่วไปที่ควรยึดถือเมื่อเขียนโปรแกรม CICS ที่เกี่ยวข้องกับหน้าจอ:

1. ทุก `SEND MAP`/`RECEIVE MAP` ควรมี `RESP` เสมอ (ไม่พึ่งพา `HANDLE CONDITION` แบบเก่า ตามที่แนะนำ
   ใน Part 061 ขั้นตอนที่ 606)
2. ตรวจสอบ `EIBAID` ก่อนตัดสินใจว่าจะเรียก `RECEIVE MAP` หรือไม่ (บางปุ่มไม่มีข้อมูลให้รับ)
3. ตรวจสอบความยาว (`xxxxL`) ของแต่ละฟิลด์ก่อนใช้ค่าที่รับมา (ขั้นตอนที่ 614)
4. เตรียมรองรับ `MAPFAIL` เสมอในทุกหน้าจอที่มีการรับข้อมูล

### ข้อควรระวัง

- ห้ามละเลยการตรวจสอบ `MAPFAIL` โดยคิดว่า "ผู้ใช้ต้องกรอกข้อมูลก่อนกด Enter เสมอ" เพราะในทางปฏิบัติ
  ผู้ใช้อาจกดปุ่มผิดโดยไม่ตั้งใจได้เสมอ โปรแกรมที่ดีต้องทนทานต่อพฤติกรรมที่ไม่คาดคิดของผู้ใช้
  (Defensive Programming เช่นเดียวกับหลักการที่สอนใน Part 048)
- `ABEND` (ตามตัวอย่างใน `WHEN OTHER`) ควรใช้เฉพาะเมื่อเกิดข้อผิดพลาดที่ร้ายแรงเกินกว่าจะจัดการต่อได้
  อย่างปลอดภัยจริง ๆ เท่านั้น ไม่ควรใช้พร่ำเพรื่อสำหรับ error ที่คาดการณ์ได้ล่วงหน้า

### แบบฝึกหัดที่ 619.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม CICS ที่ดีจึงต้องรองรับเงื่อนไข `MAPFAIL` เสมอ แม้จะดูเหมือนเป็น
สถานการณ์ที่ไม่ค่อยเกิดขึ้น

**เฉลย**: เพราะผู้ใช้ระบบจริงสามารถกระทำสิ่งที่ผู้พัฒนาไม่คาดคิดได้เสมอ เช่น กดปุ่ม CLEAR หรือ PA Key
โดยไม่ตั้งใจ หรือกด Enter ทันทีโดยไม่ได้กรอกข้อมูลใด ๆ เลย หากโปรแกรมไม่รองรับเงื่อนไขนี้และปล่อยให้
`RECEIVE MAP` ล้มเหลวโดยไม่มีการตรวจสอบ (ไม่มี `RESP` หรือ `HANDLE CONDITION`) โปรแกรมจะเกิด Runtime
Abend ทันที ซึ่งในระบบ Production จริงหมายถึงผู้ใช้ (เช่น พนักงานธนาคารหรือลูกค้าที่ใช้ ATM) เห็น
ข้อความ error ที่ไม่เป็นมิตรหรือระบบค้าง สร้างประสบการณ์ที่แย่มาก การป้องกัน `MAPFAIL` ล่วงหน้าจึงเป็น
วินัยพื้นฐานที่จำเป็นสำหรับโปรแกรม CICS ทุกโปรแกรมที่มีการรับข้อมูลจากผู้ใช้

---

## ขั้นตอนที่ 620: ตัวอย่างสมบูรณ์ — เมนูอย่างง่ายด้วย SEND MAP/RECEIVE MAP และ EVALUATE บน EIBAID

### โจทย์ตัวอย่าง

เขียนโปรแกรม CICS ที่แสดงเมนูให้ผู้ใช้เลือก (ผ่าน `SEND MAP`) รับตัวเลือกที่ผู้ใช้พิมพ์ (ผ่าน
`RECEIVE MAP`) แล้วตัดสินใจว่าจะไปทำอะไรต่อตามปุ่มที่กด (`EIBAID`) — รวมทุกแนวคิดของ Part 061-062
เข้าด้วยกัน เป็นการ**สรุปรวบยอด**ก่อนที่ Part 063 จะสอนการออกแบบ BMS Map อย่างละเอียด และ Part 064
จะสอน Pseudo-conversational Programming แบบเต็มรูปแบบ

### โปรแกรมสมบูรณ์ (ไวยากรณ์อ้างอิง)

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP620-MENU-PROGRAM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY DFHAID.
           COPY DFHBMSCA.

       01  WS-RESP-CODE          PIC S9(8) COMP.
       01  WS-MENU-CHOICE        PIC 9.

       LINKAGE SECTION.
       01  DFHCOMMAREA           PIC X(1).

       PROCEDURE DIVISION.
       MAIN-PARA.
           EVALUATE TRUE
               WHEN EIBCALEN = 0
                   PERFORM SHOW-MENU-PARA
               WHEN EIBAID = DFHENTER
                   PERFORM RECEIVE-AND-ROUTE-PARA
               WHEN EIBAID = DFHPF3
                   EXEC CICS
                       SEND TEXT FROM('GOODBYE') LENGTH(7) ERASE
                   END-EXEC
                   EXEC CICS RETURN END-EXEC
               WHEN OTHER
                   PERFORM SHOW-MENU-PARA
           END-EVALUATE.

           EXEC CICS
               RETURN TRANSID('MENU') COMMAREA(DFHCOMMAREA)
           END-EXEC.

       SHOW-MENU-PARA.
           MOVE "1" TO MENUOPTO.
           EXEC CICS
               SEND MAP('MENUMAP')
                    MAPSET('MENUSET')
                    FROM(MENUMAPO)
                    ERASE
                    CURSOR
           END-EXEC.

       RECEIVE-AND-ROUTE-PARA.
           EXEC CICS
               RECEIVE MAP('MENUMAP')
                       MAPSET('MENUSET')
                       INTO(MENUMAPI)
                       RESP(WS-RESP-CODE)
           END-EXEC.

           IF WS-RESP-CODE NOT = DFHRESP(NORMAL)
               PERFORM SHOW-MENU-PARA
           ELSE
               IF MENUOPTL = 0
                   PERFORM SHOW-MENU-PARA
               ELSE
                   MOVE MENUOPTI TO WS-MENU-CHOICE
                   EVALUATE WS-MENU-CHOICE
                       WHEN 1
                           EXEC CICS
                               XCTL PROGRAM('ATMWDRAW')
                           END-EXEC
                       WHEN 2
                           EXEC CICS
                               XCTL PROGRAM('ATMDEPST')
                           END-EXEC
                       WHEN OTHER
                           PERFORM SHOW-MENU-PARA
                   END-EVALUATE
               END-IF
           END-IF.
```

### อธิบายภาพรวมของโปรแกรม

- `EVALUATE TRUE` ที่ตรวจ `EIBCALEN = 0` ก่อนเสมอ คือรูปแบบมาตรฐานของโปรแกรม Pseudo-conversational
  (ทบทวนจาก Part 061 ขั้นตอนที่ 607) — การเรียกครั้งแรก (ไม่มี COMMAREA) จะแสดงเมนูเปล่า ๆ
- `WHEN EIBAID = DFHENTER` คือกรณีที่ผู้ใช้กรอกตัวเลือกแล้วกด Enter — โปรแกรมจะ `RECEIVE MAP` แล้ว
  ตัดสินใจว่าจะ `XCTL` ไปโปรแกรมไหนต่อตามตัวเลขที่เลือก
- `WHEN EIBAID = DFHPF3` คือกรณีผู้ใช้ต้องการออกจากเมนู — ส่งข้อความลาก่อนแล้ว `RETURN` แบบธรรมดา
  (ไม่มี `TRANSID`) เพื่อจบ Transaction ไปเลย ต่างจากกรณีอื่นที่ `RETURN TRANSID('MENU')` เพื่อรอ
  ผู้ใช้กดปุ่มถัดไป
- สังเกตว่าโค้ดนี้ผสาน `EVALUATE` (Part 011), `COPY` (Part 033), `IF` ซ้อนกัน, และคำสั่ง CICS
  ทั้งหมดที่เรียนมาใน Part 061-062 เข้าด้วยกันอย่างเป็นธรรมชาติ

### ข้อควรระวัง

- โปรแกรมตัวอย่างนี้ยังไม่ได้แสดงการ "จดจำ" สถานะที่ซับซ้อนผ่าน COMMAREA อย่างเต็มรูปแบบ (ใช้แค่
  `PIC X(1)` เป็น placeholder) เพราะเมนูง่าย ๆ แบบนี้ไม่จำเป็นต้องจดจำอะไรมาก — โปรแกรมที่ซับซ้อนกว่า
  (เช่น กระบวนการโอนเงินหลายขั้นตอน) จะต้องออกแบบโครงสร้าง COMMAREA ให้ครอบคลุม "ขั้นตอนปัจจุบัน"
  อย่างละเอียด ซึ่งเป็นหัวใจของ Part 064
- การ `XCTL` ไปโปรแกรมอื่นโดยไม่ส่ง COMMAREA ต่อ (ตามตัวอย่าง) หมายความว่าโปรแกรมปลายทาง
  (`ATMWDRAW`, `ATMDEPST`) จะเริ่มต้นแบบ "ไม่รู้อะไรมาก่อนเลย" — ในระบบจริงมักต้องส่งข้อมูลบริบท
  ที่จำเป็น (เช่น รหัสลูกค้าที่ authenticated แล้ว) ผ่าน COMMAREA ไปด้วยเสมอ

### แบบฝึกหัดที่ 620.1

**โจทย์**: จงอธิบายว่าทำไมกรณี `WHEN EIBAID = DFHPF3` (ผู้ใช้กด PF3 เพื่อออก) จึงใช้
`EXEC CICS RETURN END-EXEC` แบบธรรมดา ในขณะที่กรณีอื่นใช้
`EXEC CICS RETURN TRANSID('MENU') COMMAREA(...) END-EXEC`

**เฉลย**: เพราะเจตนาของทั้งสองกรณีต่างกันโดยสิ้นเชิง กรณี PF3 คือผู้ใช้**ต้องการจบการใช้งานเมนูนี้**
ไปเลย จึงใช้ `RETURN` ธรรมดาเพื่อบอก CICS ว่า Transaction นี้จบแล้วอย่างสมบูรณ์ ไม่ต้องรอผู้ใช้กด
อะไรต่อ (ถ้าต้องการใช้งานอีกต้องพิมพ์ Transaction ID ใหม่เอง) ส่วนกรณีอื่น (แสดงเมนูรอให้ผู้ใช้เลือก)
โปรแกรมยัง**ต้องการรอ input ถัดไปจากผู้ใช้คนเดิม**อยู่ จึงใช้ `RETURN TRANSID('MENU')` เพื่อบอก CICS
ว่า "เมื่อผู้ใช้คนนี้กดปุ่มอะไรก็ตามต่อไป ให้เรียก Transaction 'MENU' นี้กลับมาทำงานอีกครั้ง" ตาม
แนวคิด Pseudo-conversational ที่อธิบายไว้ใน Part 061 ขั้นตอนที่ 605

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้:

- ความแตกต่างของ Terminal I/O ใน CICS (ทั้งหน้าจอพร้อมกัน ผ่านจอ 3270 แบบ Block-mode) เทียบกับ
  `DISPLAY`/`ACCEPT` ของ Batch (ทีละบรรทัด)
- BMS (Basic Mapping Support) และการแยกเป็น Physical Map กับ Symbolic Map
- `EXEC CICS SEND MAP` สำหรับส่งหน้าจอไปแสดงผล พร้อม option สำคัญ (`FROM`, `ERASE`, `CURSOR`)
- `EXEC CICS RECEIVE MAP` สำหรับรับข้อมูลกลับ และโครงสร้าง `L`/`F`/`A`/`I`/`O` ของ Symbolic Map
- `SEND TEXT` ทางเลือกที่ง่ายกว่าสำหรับข้อความไม่มีโครงสร้าง
- Attribute Byte และ MDT (Modified Data Tag) — กลไกควบคุมคุณสมบัติฟิลด์และประหยัดทรัพยากรเครือข่าย
- AID Keys และ `EIBAID` — การตรวจสอบว่าผู้ใช้กดปุ่มอะไร
- `MAPFAIL` และแนวปฏิบัติมาตรฐานในการป้องกันข้อผิดพลาดของ SEND/RECEIVE
- ตัวอย่างสมบูรณ์: โปรแกรมเมนูที่ผสานทุกแนวคิดของ Part 061-062 เข้าด้วยกัน

เนื้อหาใน Part 061-062 ได้ปูพื้นฐานที่จำเป็นทั้งหมดสำหรับการเขียนโปรแกรม CICS แบบโต้ตอบกับผู้ใช้
Part 063 จะพาคุณลงลึกไปสู่การ**ออกแบบ BMS Map เอง**ตั้งแต่ต้น (การเขียน Assembler Macro
`DFHMSD`/`DFHMDI`/`DFHMDF`) และ Part 064 จะสอน **Pseudo-conversational Programming** แบบเต็ม
รูปแบบ — รูปแบบการเขียนโปรแกรม CICS ที่มีประสิทธิภาพสูงสุดที่ใช้กันจริงในอุตสาหกรรม

**[← กลับไป Part 061: CICS เบื้องต้น: Transaction Processing](part-061-cics-intro.md)**
**[ไปยัง Part 063: CICS BMS Maps และการออกแบบหน้าจอ →](part-063-cics-bms-maps.md)**
