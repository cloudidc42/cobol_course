# Part 074: COBOL และ Web Application: เชื่อมกับ Node.js/Python (ขั้นตอนที่ 731–740)

## คำนำของ Part นี้

Part 073 สอนแนวคิดหลักของ **Wrapper Service Pattern** ผ่านตัวอย่างที่เรียบง่ายที่สุด: โปรแกรม
COBOL คำนวณตัวเลขจาก input ที่ส่งเข้ามา ไม่มีการอ่าน-เขียนข้อมูลถาวรใด ๆ เลย Part นี้จะต่อยอด
แนวคิดเดียวกันให้ใกล้เคียงระบบจริงมากขึ้น: เราจะสร้าง **ระบบค้นหาข้อมูลลูกค้าผ่านเว็บ (Customer
Lookup Web System)** ที่ COBOL program อ่านข้อมูลจาก **Indexed File** จริง (ทบทวนเทคนิคเต็ม
รูปแบบจาก Part 028) แทนการรับค่าคำนวณอย่างเดียว — นี่คือรูปแบบที่ใกล้เคียงกับระบบ Core Banking
หรือระบบ ERP จริงมากกว่า ที่ข้อมูลลูกค้า/สินค้า/บัญชีมักถูกเก็บอยู่ในไฟล์หรือฐานข้อมูลที่มีอยู่แล้ว
มานาน ไม่ใช่ค่าที่คำนวณสด ๆ จาก input เพียงอย่างเดียว

นอกจากนี้ Part นี้ยังสอนสิ่งที่ชื่อ Part บอกไว้ตรง ๆ: **การเชื่อมต่อ COBOL เข้ากับทั้ง Python และ
Node.js** — เราจะสร้าง Web Layer เดียวกันด้วยสองภาษา เพื่อให้เห็นชัดเจนว่า Wrapper Service Pattern
ที่เรียนใน Part 073 นั้น**ไม่ได้ผูกติดกับภาษาใดภาษาหนึ่ง** ตราบใดที่ภาษานั้นสามารถเรียก subprocess
ภายนอกและจัดการ HTTP ได้ ก็สามารถทำหน้าที่เป็น Wrapper ให้ COBOL ได้เหมือนกันทั้งสิ้น

ทุกตัวอย่างในนี้ถูกทดสอบจริงในสภาพแวดล้อมของหลักสูตร รวมถึงกับดักจริงที่พบระหว่างการทดสอบ (เรื่อง
`LD_LIBRARY_PATH` ของไฟล์ Indexed) ซึ่งเป็นปัญหาจริงที่ผู้พัฒนาระบบลักษณะนี้มักเจอในโลกจริงเช่นกัน

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่าง (คอมเมนต์, ชื่อตัวแปร, ข้อความใน DISPLAY) เป็น
> ภาษาอังกฤษล้วน โค้ด Python/JavaScript ก็เป็นภาษาอังกฤษล้วนตามธรรมชาติของโค้ดจริงเช่นกัน

---

## ขั้นตอนที่ 731: ทบทวนภาพรวม — เป้าหมายและสถาปัตยกรรมของระบบ Customer Lookup

### สิ่งที่จะสร้างใน Part นี้

```
[ Browser / curl ]
        |
        | GET /api/customer/10001
        v
[ Web Layer: Python http.server หรือ Node.js http module ]
        |
        | subprocess: ./custlookup 10001
        v
[ CUSTLOOKUP.cob: เปิด Indexed File, READ ด้วย KEY, คืนค่า record ]
        |
        v
[ CUST074.DAT: Indexed File ที่เก็บข้อมูลลูกค้าไว้ล่วงหน้า ]
```

ระบบนี้ต่างจาก Part 073 ตรงจุดสำคัญ: โปรแกรม COBOL **ไม่ได้คำนวณอะไรจาก input ที่ส่งเข้ามาโดยตรง**
แต่ใช้ input (customer ID) เป็น **Key** ในการค้นหา record ที่มีอยู่แล้วในไฟล์ที่สร้างไว้ล่วงหน้า —
รูปแบบนี้ใกล้เคียงกับสถานการณ์จริงมากกว่า เช่น "ค้นหายอดเงินคงเหลือของลูกค้าหมายเลขบัญชี XXX" ที่ระบบ
ธนาคารต้องทำหลายล้านครั้งต่อวัน

### ทำไมต้องใช้ Indexed File แทน Sequential File

ทบทวนจาก Part 028: **Indexed File** ใช้เทคนิค ISAM ทำให้ค้นหา record ได้โดยตรงผ่าน Key โดยไม่ต้อง
ไล่อ่านทีละ record ตั้งแต่ต้นไฟล์ (ต่างจาก Sequential File ที่เรียนใน Part 023) คุณสมบัตินี้สำคัญมาก
สำหรับระบบค้นหาแบบ on-demand เช่นนี้ เพราะผู้ใช้เว็บอาจค้นหาลูกค้าคนไหนก็ได้แบบสุ่ม (random access)
ไม่ได้เรียงลำดับ การใช้ Sequential File จะทำให้แต่ละคำขอค้นหาช้าลงเรื่อย ๆ เมื่อจำนวนลูกค้าเพิ่มขึ้น

### สิ่งที่ต้องตรวจสอบก่อนเริ่ม

Part 028 เตือนไว้แล้วว่า GnuCOBOL มาตรฐานจากบาง distro อาจปิดการรองรับ `ORGANIZATION IS INDEXED`
ไว้โดยค่าเริ่มต้น เราจะตรวจสอบสภาพแวดล้อมจริงของ Part นี้ในขั้นตอนถัดไปก่อนเขียนโค้ดต่อ

### ข้อควรระวัง

- อย่าข้ามการตรวจสอบสภาพแวดล้อมไปเขียนโค้ดทันที — วินัยนี้ถูกเน้นย้ำมาตั้งแต่ Part 045 และ Part 028
  และจะพิสูจน์ให้เห็นอีกครั้งในขั้นตอนถัดไปว่าสำคัญเพียงใด

### แบบฝึกหัดที่ 731.1

**โจทย์**: จงอธิบายว่าทำไมระบบค้นหาลูกค้าแบบ "สุ่มค้นหาคนไหนก็ได้" จึงเหมาะกับ Indexed File
มากกว่า Sequential File โดยเปรียบเทียบกับที่เรียนมาจาก Part 023 และ Part 028

**เฉลยแนวทาง**: Sequential File (Part 023) ต้องอ่าน record ตามลำดับตั้งแต่ต้นไฟล์เสมอ หากต้องการ
record ที่อยู่ลำดับท้าย ๆ ต้องอ่านผ่าน record ก่อนหน้าทั้งหมดก่อน เฉลี่ยแล้วยิ่งไฟล์ใหญ่ยิ่งช้า
ในขณะที่ Indexed File (Part 028) ใช้โครงสร้างดัชนี (ISAM) ทำให้กระโดดไปอ่าน record ที่ต้องการได้
โดยตรงผ่าน Key โดยไม่ขึ้นกับขนาดไฟล์มากนัก เหมาะกับสถานการณ์ที่ผู้ใช้เว็บค้นหาลูกค้าคนไหนก็ได้แบบสุ่ม
ไม่เรียงลำดับ ซึ่งตรงกับสถานการณ์ของระบบ Customer Lookup ที่จะสร้างใน Part นี้

---

## ขั้นตอนที่ 732: ตรวจสอบสภาพแวดล้อม — cobc มาตรฐานรองรับ Indexed File หรือไม่

### ทดสอบด้วยโปรแกรมง่าย ๆ

เราเขียนโปรแกรมทดสอบเล็ก ๆ ที่ประกาศไฟล์แบบ `ORGANIZATION IS INDEXED` แล้วลองคอมไพล์ด้วย `cobc`
ที่อยู่ใน `PATH` ตามปกติของระบบ:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. IDXTEST.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUST.IDX"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID.
       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
       01  CUST-REC.
           05 CUST-ID   PIC 9(5).
           05 CUST-NAME PIC X(20).
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUST-FILE
           MOVE 1 TO CUST-ID
           MOVE "TEST" TO CUST-NAME
           WRITE CUST-REC
           CLOSE CUST-FILE
           DISPLAY "OK WROTE INDEXED FILE"
           STOP RUN.
```

```bash
cobc -x -o idxtest idxtest.cob
```

**ผลลัพธ์จริงที่ได้ในสภาพแวดล้อมของหลักสูตร:**

```
idxtest.cob: error [-Werror]: compiler is not configured to support ORGANIZATION INDEXED; FD CUST-FILE
```

### ยืนยันด้วย cobc --info

```bash
cobc --info | grep -i indexed
```

**ผลลัพธ์จริง:**

```
indexed file handler     : disabled
```

ตรงตามที่ Part 028 เตือนไว้ทุกประการ — `cobc` มาตรฐานที่อยู่ใน `PATH` ของสภาพแวดล้อมนี้ **ปิดการ
รองรับ Indexed File ไว้** (คนละสาเหตุกับ JSON ใน Part 045 แต่เป็นปัญหาประเภทเดียวกัน: ข้อจำกัดของ
การ build คอมไพเลอร์ ไม่ใช่ปัญหาที่โค้ด)

### ทางแก้: ใช้ตัวคอมไพเลอร์ทางเลือกที่ build พร้อม ISAM Handler

สภาพแวดล้อมของหลักสูตรนี้มีตัวคอมไพเลอร์ GnuCOBOL อีกชุดหนึ่งที่ build มาพร้อมเปิดใช้งาน Berkeley DB
(BDB) เป็น ISAM handler ติดตั้งแยกไว้ต่างหากที่ `/opt/gnucobol-isam/bin/cobc` ลองตรวจสอบ:

```bash
/opt/gnucobol-isam/bin/cobc --info | grep -i indexed
```

**ผลลัพธ์จริง:**

```
indexed file handler     : BDB version 5.3.28
```

คอมไพล์โปรแกรมเดิมใหม่ด้วยตัวนี้:

```bash
/opt/gnucobol-isam/bin/cobc -x -o idxtest idxtest.cob
./idxtest
```

**ผลลัพธ์จริง:**

```
OK WROTE INDEXED FILE
```

สำเร็จ! ไฟล์ `CUST.IDX` (พร้อมไฟล์เมทาดาทา `CUST.IDX.dd` ที่ BDB สร้างเพิ่ม) ถูกสร้างขึ้นจริง

### อธิบายจุดสำคัญ

- ทั้งสองคอมไพเลอร์ (`cobc` ปกติ กับ `/opt/gnucobol-isam/bin/cobc`) คือ GnuCOBOL เวอร์ชันเดียวกัน
  (`4.0-early-dev.0`) เพียงแต่ build ด้วย configuration ต่างกัน — พิสูจน์อีกครั้งว่าความสามารถของ
  ฟีเจอร์ COBOL ขึ้นกับ**การตั้งค่าตอน build คอมไพเลอร์** ไม่ใช่ตัวมาตรฐานภาษาเอง (บทเรียนเดียวกับที่
  เรียนไปแล้วจาก Part 045 เรื่อง JSON)
- Part นี้ทั้งหมดจากนี้ไปจะใช้ `/opt/gnucobol-isam/bin/cobc` ในการคอมไพล์โปรแกรมทุกโปรแกรมที่
  เกี่ยวข้องกับ Indexed File

### ข้อควรระวัง

- **เก็บ path ของคอมไพเลอร์ที่ใช้ให้ชัดเจนเสมอ** ในระบบจริง ทีมพัฒนาควรบันทึกไว้ใน build script หรือ
  เอกสารโครงการว่าใช้ GnuCOBOL build ไหนที่มี ISAM handler เปิดใช้งาน เพื่อไม่ให้สมาชิกทีมคนอื่นสับสน
  ว่าทำไมคอมไพล์โปรแกรมเดียวกันบนเครื่องต่างกันแล้วได้ผลต่างกัน
- อย่าลบไฟล์ `.dd` ที่ BDB สร้างขึ้นมาคู่กับไฟล์ Indexed File หลัก เพราะเป็นส่วนหนึ่งของโครงสร้าง
  ไฟล์ที่ ISAM handler ต้องใช้ (รายละเอียดเพิ่มเติมอยู่ใน Part 028)

### แบบฝึกหัดที่ 732.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง error message ของ JSON ใน Part 045
(`compiler is not configured to support JSON`) กับ error message ของ Indexed File ใน Part นี้
(`compiler is not configured to support ORGANIZATION INDEXED`) ว่าทั้งสองกรณีมีสาเหตุร่วมกัน
อย่างไร

**เฉลย**: ทั้งสอง error message มีรูปแบบคล้ายกันมาก (`compiler is not configured to support ...`)
เพราะมีสาเหตุร่วมกันคือ **ฟีเจอร์นั้นต้องพึ่งพาไลบรารีภายนอกที่ต้องเปิดใช้งานตอน build ตัวคอมไพเลอร์**
— JSON ต้องพึ่ง JSON library (เช่น cJSON) ส่วน Indexed File ต้องพึ่ง ISAM library (เช่น Berkeley DB)
หากไลบรารีเหล่านี้ไม่ถูกเชื่อม (link) เข้ากับ `cobc` ตอน build ฟีเจอร์นั้นจะถูกปิดใช้งานทั้งหมดไม่ว่า
โค้ด COBOL จะเขียนถูกต้องตามมาตรฐานแค่ไหนก็ตาม การแก้ปัญหาทั้งสองกรณีจึงเหมือนกัน: เปลี่ยนไปใช้
คอมไพเลอร์ (หรือ build) ที่เปิดใช้งานไลบรารีที่จำเป็นไว้แล้ว

---

## ขั้นตอนที่ 733: สร้างไฟล์ Indexed File ตัวอย่างด้วย CUSTSETUP

### โค้ดโปรแกรมสร้างข้อมูลลูกค้าเริ่มต้น

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTSETUP.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUST074.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS CUST-ID.
       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
       01  CUST-REC.
           05 CUST-ID      PIC 9(5).
           05 CUST-NAME    PIC X(20).
           05 CUST-CITY    PIC X(15).
           05 CUST-BALANCE PIC 9(7)V99.
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUST-FILE
           MOVE 10001 TO CUST-ID
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME
           MOVE "BANGKOK" TO CUST-CITY
           MOVE 15250.75 TO CUST-BALANCE
           WRITE CUST-REC

           MOVE 10002 TO CUST-ID
           MOVE "MALEE SRISUK" TO CUST-NAME
           MOVE "CHIANG MAI" TO CUST-CITY
           MOVE 8900.00 TO CUST-BALANCE
           WRITE CUST-REC

           MOVE 10003 TO CUST-ID
           MOVE "PRASERT KAEWTA" TO CUST-NAME
           MOVE "KHON KAEN" TO CUST-CITY
           MOVE 230.50 TO CUST-BALANCE
           WRITE CUST-REC

           CLOSE CUST-FILE
           DISPLAY "CUST074.DAT created with 3 records"
           STOP RUN.
```

### คอมไพล์และรัน

```bash
/opt/gnucobol-isam/bin/cobc -x -o custsetup custsetup.cob
./custsetup
```

**ผลลัพธ์จริง:**

```
CUST074.DAT created with 3 records
```

### อธิบายจุดสำคัญ

- โครงสร้างและวิธีเขียนโค้ดนี้เหมือนกับ Indexed File ที่เรียนใน Part 028 ทุกประการ — ไม่มีสิ่งใหม่
  ในระดับ syntax เลย สิ่งที่ต่างคือ**บริบทการใช้งาน**: ไฟล์นี้จะถูกอ่านโดยโปรแกรมอื่นที่เรียกผ่าน
  Web Layer แทนที่จะอ่านโดยโปรแกรม COBOL ตัวเดียวกันเอง
- `ACCESS MODE IS SEQUENTIAL` ในโปรแกรม setup นี้ใช้สำหรับการ**เขียน**ข้อมูลเริ่มต้นเรียงตามลำดับ
  Key ที่เพิ่มขึ้น (10001, 10002, 10003) ซึ่งจำเป็นสำหรับการ `WRITE` ครั้งแรกที่สร้างไฟล์ (ทบทวนจาก
  Part 028 ขั้นตอนที่ 572 เรื่องข้อกำหนดการเรียง key จากน้อยไปมากตอนสร้างไฟล์ใหม่)

### ข้อควรระวัง

- โปรแกรมนี้ **ต้องรันเพียงครั้งเดียวก่อนเริ่มใช้งานระบบเว็บ** เพราะ `OPEN OUTPUT` จะลบไฟล์เดิม
  ทิ้งทั้งหมดเสมอ (ทบทวนจาก Part 023 ขั้นตอนที่ 224) หากรันซ้ำระหว่างที่ระบบเว็บกำลังทำงานอยู่จะทำให้
  ข้อมูลที่เคยมีหายไปหมด
- ต้องใช้ `/opt/gnucobol-isam/bin/cobc` ในการคอมไพล์เสมอสำหรับโปรแกรมที่เกี่ยวข้องกับไฟล์นี้ ตาม
  ที่พิสูจน์ไว้ในขั้นตอนที่ 732

### แบบฝึกหัดที่ 733.1

**โจทย์**: จงเพิ่มลูกค้าคนที่ 4 (ID `10004`, ชื่อ `"WANNA SRISUK"`, เมือง `"PHUKET"`, ยอดเงิน
`5000.00`) เข้าไปในโปรแกรม `CUSTSETUP`

**เฉลย**: เพิ่มชุด `MOVE`/`WRITE` อีกชุดก่อน `CLOSE CUST-FILE`:

```cobol
           MOVE 10004 TO CUST-ID
           MOVE "WANNA SRISUK" TO CUST-NAME
           MOVE "PHUKET" TO CUST-CITY
           MOVE 5000.00 TO CUST-BALANCE
           WRITE CUST-REC
```

---

## ขั้นตอนที่ 734: เขียนโปรแกรม CUSTLOOKUP — ค้นหาลูกค้าด้วย Command-Line Argument

### ออกแบบ Interface

ต่างจาก `ORDERCALC` ใน Part 073 ที่รับ input ทาง stdin โปรแกรมนี้จะรับ **customer ID ผ่าน
command-line argument** แทน (เช่น `./custlookup 10001`) ซึ่งเป็นอีกวิธีมาตรฐานหนึ่งที่ COBOL
program รับ input จากภายนอกได้ ผ่านคำสั่ง `ACCEPT ... FROM ARGUMENT-VALUE`

### โค้ดโปรแกรม CUSTLOOKUP

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTLOOKUP.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUST074.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-FILE-STATUS.
       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
       01  CUST-REC.
           05 CUST-ID      PIC 9(5).
           05 CUST-NAME    PIC X(20).
           05 CUST-CITY    PIC X(15).
           05 CUST-BALANCE PIC 9(7)V99.
       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS      PIC XX.
       01  WS-ARG              PIC X(10).
       01  WS-BALANCE-EDIT     PIC Z(6)9.99.
       01  WS-OUT-LINE         PIC X(80).
       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-ARG FROM ARGUMENT-VALUE
           MOVE FUNCTION NUMVAL(WS-ARG) TO CUST-ID
           OPEN INPUT CUST-FILE
           READ CUST-FILE
               INVALID KEY
                   DISPLAY "NOTFOUND"
               NOT INVALID KEY
                   MOVE CUST-BALANCE TO WS-BALANCE-EDIT
                   STRING FUNCTION TRIM(CUST-NAME) DELIMITED BY SIZE
                          "|" DELIMITED BY SIZE
                          FUNCTION TRIM(CUST-CITY) DELIMITED BY SIZE
                          "|" DELIMITED BY SIZE
                          FUNCTION TRIM(WS-BALANCE-EDIT)
                              DELIMITED BY SIZE
                       INTO WS-OUT-LINE
                   DISPLAY FUNCTION TRIM(WS-OUT-LINE)
           END-READ
           CLOSE CUST-FILE
           STOP RUN.
```

### อธิบายทีละส่วน

- `ACCESS MODE IS RANDOM`: ต่างจาก `CUSTSETUP` ที่ใช้ `SEQUENTIAL` โปรแกรมนี้ใช้ `RANDOM` เพราะ
  ต้องการ**กระโดดไปอ่าน record ตาม Key ที่ระบุโดยตรง** ไม่ใช่อ่านเรียงตามลำดับ (ทบทวนความแตกต่าง
  ของ ACCESS MODE ทั้งสามแบบจาก Part 028)
- `ACCEPT WS-ARG FROM ARGUMENT-VALUE`: อ่านค่า command-line argument ตัวแรกที่ส่งเข้ามาตอนรัน
  โปรแกรม (เช่น `10001` จาก `./custlookup 10001`) เก็บไว้เป็นข้อความก่อน แล้วแปลงเป็นตัวเลขด้วย
  `FUNCTION NUMVAL` เหมือนเทคนิคที่ใช้กับ stdin ใน Part 073
- `READ CUST-FILE ... INVALID KEY ... NOT INVALID KEY`: รูปแบบมาตรฐานสำหรับอ่าน Indexed File
  แบบ Random Access (ทบทวนเต็มรูปแบบจาก Part 028) — `INVALID KEY` ทำงานเมื่อไม่พบ Key ที่ค้นหา,
  `NOT INVALID KEY` ทำงานเมื่อพบ
- ผลลัพธ์ถูกออกแบบให้เป็นรูปแบบเดียวกับ Part 073: ข้อความคั่นด้วย `|` ทาง stdout หรือคำว่า
  `NOTFOUND` เมื่อไม่พบ — ง่ายต่อการแปลงเป็น JSON ที่ฝั่ง Web Layer ในขั้นตอนถัดไป

### คอมไพล์และทดสอบ Standalone

```bash
/opt/gnucobol-isam/bin/cobc -x -o custlookup custlookup.cob
./custlookup 10001
./custlookup 10002
./custlookup 99999
```

**ผลลัพธ์จริงทั้งสามกรณี:**

```
SOMCHAI JAIDEE|BANGKOK|15250.75
MALEE SRISUK|CHIANG MAI|8900.00
NOTFOUND
```

ผลลัพธ์ถูกต้องครบทุกกรณี รวมถึงกรณี ID ที่ไม่มีอยู่จริง (`99999`)

### ข้อควรระวัง

- **ต้อง `OPEN INPUT` เท่านั้น ไม่ใช่ `OPEN I-O`** สำหรับโปรแกรม lookup ที่มีหน้าที่แค่ค้นหาอย่าง
  เดียว เพื่อป้องกันไม่ให้โปรแกรมนี้แก้ไขข้อมูลโดยไม่ตั้งใจ (Principle of Least Privilege — ให้สิทธิ์
  เท่าที่จำเป็นเท่านั้น)
- `FILE STATUS IS WS-FILE-STATUS` ถูกประกาศไว้แต่ยังไม่ได้ใช้ตรวจสอบในตัวอย่างนี้เพื่อความกระชับ
  ระบบจริงควรตรวจสอบค่านี้หลัง `OPEN` เพื่อดักจับกรณีไฟล์ `CUST074.DAT` หายไปหรือเสียหาย (ทบทวน
  File Status Codes เต็มรูปแบบจาก Part 030)

### แบบฝึกหัดที่ 734.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม `CUSTLOOKUP` จึงไม่จำเป็นต้องมี End-of-File Flag (`88
END-OF-FILE`) เหมือนโปรแกรมอ่าน Sequential File ที่เรียนใน Part 023-026

**เฉลย**: เพราะโปรแกรมนี้อ่าน record **เพียง 1 ครั้งเดียว** ด้วย Random Access ผ่าน Key ที่ระบุ
ชัดเจน ไม่ได้วนลูปอ่านไปเรื่อย ๆ จนสุดไฟล์เหมือนการประมวลผล Sequential File ทั้งไฟล์ (ทบทวนจาก
Part 025-026) End-of-File Flag จำเป็นเฉพาะเมื่อต้องอ่านหลาย record ต่อเนื่องกันในลูปเท่านั้น
สำหรับการอ่านครั้งเดียวแบบนี้ ผลลัพธ์จาก `INVALID KEY`/`NOT INVALID KEY` ก็เพียงพอต่อการตัดสินใจ
แล้วว่าจะทำอย่างไรต่อ

---

## ขั้นตอนที่ 735: ออกแบบ Web Layer ด้วย Python (http.server)

### โครงสร้าง Endpoint ที่จะสร้าง

- `GET /` — หน้าเว็บ HTML อย่างง่าย มีฟอร์มค้นหาลูกค้าด้วย JavaScript เรียก API
- `GET /api/customer/<id>` — คืนค่า JSON ของข้อมูลลูกค้า หรือ HTTP 404 หากไม่พบ

### โค้ดสมบูรณ์ของ webapp.py

```python
import json
import os
import subprocess
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse

COBOL_BINARY = os.path.join(os.path.dirname(__file__), "custlookup")
ENV = dict(os.environ)
ENV["LD_LIBRARY_PATH"] = "/opt/gnucobol-isam/lib"

PAGE = """<!doctype html>
<html><body>
<h1>Customer Lookup</h1>
<form onsubmit="lookup(event)">
<input id="cid" placeholder="Customer ID e.g. 10001">
<button type="submit">Search</button>
</form>
<pre id="out"></pre>
<script>
async function lookup(e) {
  e.preventDefault();
  const id = document.getElementById('cid').value;
  const res = await fetch('/api/customer/' + id);
  const data = await res.json();
  document.getElementById('out').textContent = JSON.stringify(data, null, 2);
}
</script>
</body></html>
"""


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        parsed = urlparse(self.path)
        if parsed.path == "/":
            body = PAGE.encode()
            self.send_response(200)
            self.send_header("Content-Type", "text/html")
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()
            self.wfile.write(body)
            return

        if parsed.path.startswith("/api/customer/"):
            cust_id = parsed.path.rsplit("/", 1)[-1]
            result = subprocess.run(
                [COBOL_BINARY, cust_id],
                capture_output=True, text=True, timeout=5, env=ENV,
            )
            line = result.stdout.strip()
            if line == "NOTFOUND":
                body = json.dumps({"error": "customer not found"}).encode()
                status = 404
            else:
                name, city, balance = line.split("|")
                body = json.dumps({
                    "customer_id": cust_id, "name": name,
                    "city": city, "balance": float(balance),
                }).encode()
                status = 200
            self.send_response(status)
            self.send_header("Content-Type", "application/json")
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()
            self.wfile.write(body)
            return

        self.send_response(404)
        self.end_headers()

    def log_message(self, fmt, *args):
        pass


if __name__ == "__main__":
    HTTPServer(("127.0.0.1", 8074), Handler).serve_forever()
```

### อธิบายจุดสำคัญ

- `ENV = dict(os.environ)` แล้วเพิ่ม `ENV["LD_LIBRARY_PATH"] = "/opt/gnucobol-isam/lib"`: จุดนี้
  **สำคัญมากและเป็นกับดักจริงที่จะพิสูจน์ในขั้นตอนที่ 738** — โปรแกรม `custlookup` ที่คอมไพล์ด้วย
  `/opt/gnucobol-isam/bin/cobc` ต้องการ runtime library ของตัวเองตอนรัน หากไม่ตั้งค่านี้ให้ถูกต้อง
  โปรแกรมจะ crash
- `parsed.path.rsplit("/", 1)[-1]`: ดึงส่วนท้ายสุดของ URL path ออกมาเป็น customer ID เช่น
  `/api/customer/10001` จะได้ `10001`
- โครงสร้างเหมือนกับ Wrapper Service ใน Part 073 ทุกประการ (routing, subprocess, JSON response)
  เพียงแต่เปลี่ยนจาก `POST` เป็น `GET` เพราะการค้นหาข้อมูล (query) ไม่มีผลข้างเคียง (side effect)
  ต่อข้อมูล จึงเหมาะกับ HTTP method `GET` ตามหลัก REST (ต่างจาก `POST /api/order` ใน Part 073
  ที่ "สร้าง" ผลการคำนวณใหม่ทุกครั้ง)

### ข้อควรระวัง

- หน้า HTML (`PAGE`) ในตัวอย่างนี้เขียนแบบง่ายที่สุดเพื่อการสาธิต ไม่มีการจัดการ error ฝั่ง
  JavaScript (เช่น กรณี fetch ล้มเหลว) ระบบจริงควรเพิ่มการจัดการเหล่านี้

### แบบฝึกหัดที่ 735.1

**โจทย์**: จงอธิบายว่าทำไม endpoint ค้นหาลูกค้าจึงควรเป็น `GET` ไม่ใช่ `POST` ทั้งที่ Part 073 เรา
เลือกใช้ `POST` สำหรับ `/api/order`

**เฉลย**: ตามหลักการออกแบบ REST API มาตรฐาน `GET` ควรใช้สำหรับการร้องขอข้อมูล (retrieve) ที่**ไม่มี
ผลข้างเคียง**ต่อสถานะของระบบ (idempotent — เรียกกี่ครั้งก็ได้ผลลัพธ์เดิมเสมอโดยไม่เปลี่ยนแปลงข้อมูล)
การค้นหาลูกค้าด้วย ID เข้าข่ายนี้พอดี เพราะเป็นการอ่านข้อมูลอย่างเดียวไม่มีการแก้ไข ในขณะที่
`/api/order` ของ Part 073 แม้จะไม่ได้บันทึกข้อมูลถาวรจริง ๆ ในตัวอย่างนั้น แต่ในทางความหมายถือเป็น
การ "ส่งคำสั่งซื้อ" ที่ควรใช้ `POST` ตามธรรมเนียมของ REST เพื่อสื่อความหมายที่ถูกต้องให้ผู้ใช้ API
คนอื่นเข้าใจตรงกัน

---

## ขั้นตอนที่ 736: รันระบบจริงและทดสอบด้วย curl

### ขั้นตอนการรันทั้งหมด

```bash
/opt/gnucobol-isam/bin/cobc -x -o custsetup custsetup.cob
/opt/gnucobol-isam/bin/cobc -x -o custlookup custlookup.cob
./custsetup
python3 webapp.py
```

### ทดสอบหน้า HTML

```bash
curl -s http://127.0.0.1:8074/ | head -5
```

**ผลลัพธ์จริง:**

```
<!doctype html>
<html><body>
<h1>Customer Lookup</h1>
<form onsubmit="lookup(event)">
<input id="cid" placeholder="Customer ID e.g. 10001">
```

### ทดสอบ API ค้นหาลูกค้าที่มีอยู่จริง

```bash
curl -s http://127.0.0.1:8074/api/customer/10001
```

**ผลลัพธ์จริง:**

```
{"customer_id": "10001", "name": "SOMCHAI JAIDEE", "city": "BANGKOK", "balance": 15250.75}
```

```bash
curl -s http://127.0.0.1:8074/api/customer/10003
```

**ผลลัพธ์จริง:**

```
{"customer_id": "10003", "name": "PRASERT KAEWTA", "city": "KHON KAEN", "balance": 230.5}
```

### ทดสอบ API ค้นหาลูกค้าที่ไม่มีอยู่จริง

```bash
curl -s -w " [http %{http_code}]\n" http://127.0.0.1:8074/api/customer/99999
```

**ผลลัพธ์จริง:**

```
{"error": "customer not found"} [http 404]
```

ระบบทำงานถูกต้องครบทุกกรณี ทั้งการค้นพบข้อมูล (200) และไม่พบข้อมูล (404) — สังเกตว่า HTTP status
code สื่อความหมายตรงกับผลลัพธ์จริงอย่างถูกต้องตามหลัก REST

### ข้อควรระวัง

- ต้องรัน `./custsetup` **ก่อน** เริ่ม `python3 webapp.py` เสมอ เพื่อให้ไฟล์ `CUST074.DAT` มีอยู่
  ก่อนที่ Web Layer จะพยายามค้นหาข้อมูลจากมัน
- สังเกตว่า `curl` เรียก `webapp.py` ที่ port `8074` ไม่ใช่ `8073` ของ Part 073 — เมื่อรันทั้งสอง
  ระบบพร้อมกัน (เช่น ระหว่างทบทวนเนื้อหา) ต้องแยก port ให้ชัดเจนเพื่อไม่ให้ชนกัน

### แบบฝึกหัดที่ 736.1

**โจทย์**: จงเขียนคำสั่ง `curl` ที่ทดสอบค้นหาลูกค้า ID `10002` และคาดเดาผลลัพธ์ JSON ที่ควรได้
ก่อนรันจริงเพื่อตรวจคำตอบ

**เฉลย**: คำสั่ง `curl -s http://127.0.0.1:8074/api/customer/10002` ควรได้ผลลัพธ์
`{"customer_id": "10002", "name": "MALEE SRISUK", "city": "CHIANG MAI", "balance": 8900.0}`
ตามข้อมูลที่สร้างไว้ใน `CUSTSETUP` ขั้นตอนที่ 733

---

## ขั้นตอนที่ 737: ทางเลือก — สร้าง Web Layer เดียวกันด้วย Node.js

### ทำไมต้องสอนทั้งสองภาษา

ชื่อ Part นี้ระบุชัดเจนว่า "เชื่อมกับ Node.js/Python" — ประเด็นสำคัญที่ต้องเข้าใจคือ **Wrapper
Service Pattern ไม่ได้ผูกติดกับภาษาใดภาษาหนึ่ง** ตราบใดที่ภาษานั้นมีความสามารถ 2 อย่าง: (1) เปิด
HTTP server ได้ และ (2) เรียก subprocess ภายนอกได้ ก็สามารถทำหน้าที่เป็น Wrapper ให้ COBOL ได้
เหมือนกันทั้งสิ้น Node.js ก็มีทั้งสองความสามารถนี้ในตัว (`http` module และ `child_process` module)
มาพร้อมกับตัวติดตั้งมาตรฐานเช่นเดียวกับ Python

### ตรวจสอบเวอร์ชัน Node.js ที่ใช้ทดสอบ

```bash
node --version
```

**ผลลัพธ์จริง:**

```
v22.22.2
```

### โค้ดสมบูรณ์ของ webapp.js (เทียบเท่า webapp.py)

```javascript
const http = require("http");
const { execFile } = require("child_process");
const path = require("path");

const COBOL_BINARY = path.join(__dirname, "custlookup");
const ENV = Object.assign({}, process.env, {
  LD_LIBRARY_PATH: "/opt/gnucobol-isam/lib",
});

const server = http.createServer((req, res) => {
  const match = req.url.match(/^\/api\/customer\/(\d+)$/);
  if (!match) {
    res.writeHead(404);
    res.end();
    return;
  }
  const custId = match[1];
  execFile(COBOL_BINARY, [custId], { env: ENV }, (err, stdout) => {
    const line = stdout.trim();
    if (line === "NOTFOUND") {
      res.writeHead(404, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ error: "customer not found" }));
      return;
    }
    const [name, city, balance] = line.split("|");
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({
      customer_id: custId, name: name,
      city: city, balance: parseFloat(balance),
    }));
  });
});

server.listen(8075, "127.0.0.1");
```

### เปรียบเทียบโครงสร้างโค้ดสองภาษา

| แนวคิด | Python (`http.server`) | Node.js (`http` module) |
|---|---|---|
| สร้าง HTTP server | `HTTPServer((host, port), Handler)` | `http.createServer(callback)` |
| จัดการ routing | เช็ค `self.path` ใน `do_GET` | จับคู่ `req.url` ด้วย regular expression |
| เรียก subprocess | `subprocess.run([...], ...)` (synchronous) | `execFile(...)` (asynchronous, ใช้ callback) |
| ส่ง JSON response | `self.wfile.write(body)` | `res.end(JSON.stringify(...))` |

### รันและทดสอบ Node.js เวอร์ชัน

```bash
node webapp.js
```

```bash
curl -s http://127.0.0.1:8075/api/customer/10002
```

**ผลลัพธ์จริง:**

```
{"customer_id":"10002","name":"MALEE SRISUK","city":"CHIANG MAI","balance":8900}
```

```bash
curl -s -w " [http %{http_code}]\n" http://127.0.0.1:8075/api/customer/00001
```

**ผลลัพธ์จริง:**

```
{"error":"customer not found"} [http 404]
```

ผลลัพธ์ถูกต้องเช่นเดียวกับเวอร์ชัน Python — พิสูจน์ว่า COBOL program ตัวเดียวกัน (`custlookup`)
ถูกเรียกใช้ได้สำเร็จจากทั้งสองภาษาโดยไม่ต้องแก้ไขโค้ด COBOL แม้แต่บรรทัดเดียว

### อธิบายความแตกต่างสำคัญ: Synchronous กับ Asynchronous

จุดที่ต่างกันมากที่สุดระหว่างสองเวอร์ชันคือวิธีเรียก subprocess: Python `subprocess.run` เป็นแบบ
**synchronous** (บล็อกรอจนกว่า subprocess จะจบ) ในขณะที่ Node.js `execFile` เป็นแบบ
**asynchronous** (ใช้ callback, ไม่บล็อก thread หลัก) — นี่คือเหตุผลสำคัญที่ Node.js สามารถรับ
หลาย HTTP request พร้อมกันได้ดีตามธรรมชาติของ Node.js Event Loop แม้จะยังใช้ `http.createServer`
พื้นฐานแบบเดียวกับที่ `HTTPServer` ของ Python ทำ (ทบทวนประเด็น concurrency นี้เพิ่มเติมได้จาก
Part 073 ขั้นตอนที่ 729)

### ข้อควรระวัง

- ตัวแปร `custId` ที่ได้จาก regular expression `\d+` (ตัวเลขล้วน) ในตัวอย่างนี้ปลอดภัยจาก command
  injection อยู่แล้วในระดับหนึ่ง เพราะ `execFile` (ต่างจาก `exec`) ไม่ผ่าน shell interpreter — หลัก
  การเดียวกับที่อธิบายไว้ใน Part 073 ขั้นตอนที่ 726 เรื่อง `subprocess.run` แบบ list
- ต้องตั้งค่า `LD_LIBRARY_PATH` ใน `ENV` เหมือนกับเวอร์ชัน Python ทุกประการ มิฉะนั้นจะเจอปัญหา
  เดียวกับที่จะพิสูจน์ในขั้นตอนถัดไป

### แบบฝึกหัดที่ 737.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `execFile` แทน `exec` ใน Node.js จึงปลอดภัยกว่า เทียบเคียงกับ
เหตุผลที่ Part 073 อธิบายไว้เรื่อง `subprocess.run` แบบ list กับ `shell=True`

**เฉลย**: `child_process.exec` ของ Node.js รันคำสั่งผ่าน shell interpreter (คล้าย `shell=True`
ของ Python) ทำให้อักขระพิเศษของ shell ในค่าที่ประกอบเป็นคำสั่งอาจถูกตีความผิดวัตถุประสงค์ กลายเป็น
ช่องโหว่ Command Injection ได้ ในขณะที่ `execFile` เรียกโปรแกรมโดยตรงด้วยชื่อไฟล์และ array ของ
argument โดยไม่ผ่าน shell เลย จึงปลอดภัยกว่ามากเมื่อ argument บางส่วนมาจากภายนอก หลักการนี้ตรงกับ
เหตุผลที่ Part 073 ขั้นตอนที่ 726 อธิบายไว้เรื่อง `subprocess.run(list)` เทียบกับ
`subprocess.run(string, shell=True)` ทุกประการ เพียงแต่เป็นคนละภาษา

---

## ขั้นตอนที่ 738: กับดักจริงที่พบระหว่างทดสอบ — LD_LIBRARY_PATH และ Indexed File

### ปัญหาที่พบจริงระหว่างพัฒนา Part นี้

ระหว่างทดสอบโปรแกรม `custlookup` ที่คอมไพล์ด้วย `/opt/gnucobol-isam/bin/cobc` เราลองรันโปรแกรม
โดย**ไม่ตั้งค่า** `LD_LIBRARY_PATH` ดูว่าจะเกิดอะไรขึ้น:

```bash
./custlookup 10001
```

**ผลลัพธ์จริงที่ได้ (ข้อผิดพลาดจริง ไม่ใช่ผลลัพธ์ที่ถูกต้อง):**

```
libcob: error: ERROR I/O routine IXEXT is not present

attempt to reference unallocated memory (signal SIGSEGV)
abnormal termination - file contents may be incorrect
```

โปรแกรม **crash ด้วย Segmentation Fault (SIGSEGV)** ทันที ไม่ใช่แค่ error message ธรรมดา

### วิเคราะห์สาเหตุด้วย ldd

```bash
ldd custlookup | grep libcob
```

**ผลลัพธ์จริง:**

```
libcob.so.5 => /lib/x86_64-linux-gnu/libcob.so.5
```

พบสาเหตุ: แม้เราจะคอมไพล์ด้วย `/opt/gnucobol-isam/bin/cobc` (ที่มี ISAM handler เปิดใช้งาน) แต่
ตอน**รัน**โปรแกรม ระบบกลับไปโหลด `libcob.so.5` จาก path มาตรฐานของระบบ (`/lib/x86_64-linux-gnu/`)
ซึ่งเป็นเวอร์ชันที่**ไม่มี** ISAM handler แทน เพราะทั้งสองไฟล์มีชื่อเดียวกัน (`libcob.so.5`) และ
ระบบปฏิบัติการค้นหาตาม `LD_LIBRARY_PATH` และ path มาตรฐานตามลำดับ หากไม่ระบุ `LD_LIBRARY_PATH`
ให้ชี้ไปที่ `/opt/gnucobol-isam/lib` ก่อน ระบบจะเจอไฟล์เวอร์ชันมาตรฐานก่อนเสมอ

### ทางแก้ที่ทดสอบแล้วว่าใช้ได้จริง

```bash
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./custlookup 10001
```

**ผลลัพธ์จริง:**

```
SOMCHAI JAIDEE|BANGKOK|15250.75
```

ทำงานถูกต้องทันทีเมื่อกำหนด `LD_LIBRARY_PATH` ให้ชี้ไปที่โฟลเดอร์ที่มี runtime library ที่ตรงกับ
ตัวคอมไพเลอร์ที่ใช้คอมไพล์โปรแกรมนี้

### ทำไมโค้ด Python/Node.js ในขั้นตอนก่อนหน้าจึงทำงานได้โดยไม่มีปัญหานี้

สังเกตว่าทั้ง `webapp.py` (ขั้นตอนที่ 735) และ `webapp.js` (ขั้นตอนที่ 737) มีโค้ดตั้งค่า
`LD_LIBRARY_PATH` ไว้ล่วงหน้าอยู่แล้ว:

```python
ENV = dict(os.environ)
ENV["LD_LIBRARY_PATH"] = "/opt/gnucobol-isam/lib"
```

```javascript
const ENV = Object.assign({}, process.env, {
  LD_LIBRARY_PATH: "/opt/gnucobol-isam/lib",
});
```

นี่ไม่ใช่เรื่องบังเอิญ — เราเขียนโค้ดสองไฟล์นี้โดยรู้ล่วงหน้าแล้วว่าต้องตั้งค่านี้ (จากการทดสอบระหว่าง
พัฒนา Part นี้จริง) และนำมาสอนไว้ในขั้นตอนนี้เพื่อให้คุณเข้าใจ**สาเหตุที่แท้จริง**ว่าทำไมต้องมีบรรทัด
นี้อยู่ในโค้ด ไม่ใช่แค่ copy โค้ดมาโดยไม่เข้าใจ

### อธิบายจุดสำคัญ

- เมื่อ Web Layer เรียก `subprocess.run`/`execFile` โดย**ไม่ส่ง `env` ที่กำหนด `LD_LIBRARY_PATH`
  ไว้อย่างชัดเจน** subprocess ที่ถูกสร้างขึ้นจะสืบทอด (inherit) environment variable จาก process
  แม่ ซึ่งอาจไม่มี `LD_LIBRARY_PATH` ที่ถูกต้องอยู่เลย ทำให้เกิดปัญหาเดียวกับที่พิสูจน์ข้างต้น
- ปัญหานี้เป็นตัวอย่างคลาสสิกของ **"works on my machine" bug** — โปรแกรม COBOL ทำงานถูกต้องสมบูรณ์
  เมื่อรันตรง ๆ จาก terminal ที่ตั้งค่า `LD_LIBRARY_PATH` ไว้แล้ว (เช่นระหว่างพัฒนา) แต่ crash ทันที
  เมื่อถูกเรียกจาก process อื่นที่ไม่มีการตั้งค่านี้ (เช่น Web Server ที่รันเป็น background service)

### ข้อควรระวัง

- **กำหนด environment variable ที่จำเป็นทั้งหมดอย่างชัดเจนในโค้ด Wrapper Service เสมอ** อย่าพึ่งพา
  การตั้งค่าที่ shell ของผู้พัฒนาเพียงอย่างเดียว เพราะเมื่อนำระบบไป deploy จริง (เช่นผ่าน systemd
  service, Docker container ที่จะเรียนใน Part 076, หรือ process manager อื่น ๆ) environment ที่ใช้
  รันอาจไม่เหมือนกับตอนพัฒนาเลย
- Segmentation Fault ที่เกิดจากปัญหา library เวอร์ชันไม่ตรงกันแบบนี้ เป็นปัญหาที่**อ่าน error
  message เดียวไม่พอ** ต้องใช้เครื่องมืออย่าง `ldd` ช่วยตรวจสอบเพิ่มเติมเพื่อหาสาเหตุที่แท้จริง

### แบบฝึกหัดที่ 738.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบด้วย `ldd custlookup` จึงช่วยวินิจฉัยปัญหานี้ได้ แทนที่จะอ่าน
แค่ error message `"ERROR I/O routine IXEXT is not present"` เพียงอย่างเดียว

**เฉลย**: error message `"ERROR I/O routine IXEXT is not present"` บอกแค่**อาการ** (symptom) ว่า
libcob runtime ที่กำลังทำงานอยู่ไม่มีฟังก์ชันจัดการ Indexed File ให้ใช้ แต่ไม่ได้บอก**สาเหตุที่แท้จริง**
ว่าทำไมถึงเป็นเช่นนั้น การใช้ `ldd` ช่วยแสดงให้เห็นว่า dynamic linker กำลังโหลด shared library
(`libcob.so.5`) จาก path ไหนจริง ๆ ทำให้เห็นชัดเจนว่าปัญหาคือระบบกำลังใช้ library คนละตัวกับที่
ตัวคอมไพเลอร์ `/opt/gnucobol-isam/bin/cobc` ตั้งใจให้ใช้ นำไปสู่การแก้ไขที่ตรงจุด (ตั้งค่า
`LD_LIBRARY_PATH`) แทนการเดาสุ่มลองผิดลองถูก

---

## ขั้นตอนที่ 739: ประสิทธิภาพและแนวทางขยายระบบ (Scaling Considerations)

### ต้นทุนของการเปิดไฟล์ Indexed File ทุกครั้งที่มีคำขอ

สังเกตว่าโปรแกรม `custlookup` ทำ `OPEN INPUT` และ `CLOSE` ไฟล์ `CUST074.DAT` **ทุกครั้ง**ที่ถูก
เรียก (เพราะแต่ละ HTTP request สร้าง subprocess ใหม่ทั้งหมด) การเปิด-ปิดไฟล์ซ้ำ ๆ แบบนี้มีต้นทุน
ด้านเวลาแม้จะไม่มากนักสำหรับไฟล์ขนาดเล็ก แต่จะเห็นผลชัดเจนขึ้นเมื่อ:

- ไฟล์ Indexed File มีขนาดใหญ่มาก (หลายล้าน record) ทำให้การเปิดไฟล์แต่ละครั้งมีต้นทุนสูงขึ้น
- ปริมาณ request สูงมาก (การเปิด-ปิดไฟล์ซ้ำหลายพันครั้งต่อวินาทีสร้างภาระให้ระบบปฏิบัติการ)

### แนวทางที่ก้าวหน้ากว่า (แนวคิดเพื่อการศึกษาต่อยอด)

- **COBOL Server Program แบบ Long-Running**: ออกแบบโปรแกรม COBOL ให้เปิดไฟล์ครั้งเดียวตอนเริ่ม
  ทำงาน แล้ววนลูปรับคำขอค้นหาต่อเนื่องหลายครั้งทาง stdin (แทนที่จะ `STOP RUN` หลังค้นหา 1 ครั้ง)
  วิธีนี้ต้องออกแบบ Wrapper Service ให้สื่อสารกับ subprocess เดียวที่เปิดค้างไว้ต่อเนื่อง แทนที่จะ
  สร้าง subprocess ใหม่ทุกคำขอ (เทคนิคนี้ซับซ้อนกว่าขอบเขตของ Part นี้ แต่ควรรู้จักไว้เป็นแนวทาง)
- **Caching**: หากข้อมูลลูกค้าไม่เปลี่ยนแปลงบ่อย Wrapper Service สามารถเก็บผลลัพธ์ที่เคยค้นหาไว้ใน
  หน่วยความจำชั่วคราว (in-memory cache) เพื่อลดจำนวนครั้งที่ต้องเรียก subprocess ซ้ำสำหรับ ID
  เดียวกัน
- **Migration ไปสู่ฐานข้อมูลสมัยใหม่**: สำหรับระบบที่ต้องการค้นหาข้อมูลปริมาณสูงมากพร้อมกัน อาจ
  พิจารณาย้ายข้อมูลจาก Indexed File ไปสู่ฐานข้อมูลเชิงสัมพันธ์ที่มีกลไก connection pooling และ
  caching ในตัว (เนื้อหานี้จะสอนใน **Part 075** ถัดไป)

### เปรียบเทียบ: เมื่อไหร่ควรคง Indexed File ไว้ เมื่อไหร่ควรย้ายไปฐานข้อมูล

| ปัจจัย | คง Indexed File ไว้ | ย้ายไปฐานข้อมูลสมัยใหม่ |
|---|---|---|
| ปริมาณข้อมูล | เล็ก-กลาง | ใหญ่มาก ต้องการ scale แนวนอน |
| จำนวน concurrent request | ต่ำ-ปานกลาง | สูงมาก |
| ความต้องการ query ซับซ้อน (JOIN, aggregate) | ไม่มี หรือน้อยมาก | มี ต้องการ SQL เต็มรูปแบบ |
| ความเสี่ยงจากการย้ายระบบ | ระบบเดิมทำงานดีอยู่แล้ว ไม่คุ้มความเสี่ยง | ความต้องการทางธุรกิจเกินขีดจำกัดเดิมชัดเจน |

### ข้อควรระวัง

- อย่ารีบสรุปว่าต้อง "ย้ายไปฐานข้อมูลสมัยใหม่" เสมอ — Indexed File ที่ออกแบบมาอย่างดียังคงทำงานได้
  รวดเร็วและเสถียรสำหรับหลายสถานการณ์ในโลกจริง (ระบบ VSAM บน Mainframe ที่เรียนใน Part 055-056
  ก็ยังคงเป็นแกนหลักของหลายธนาคารทั่วโลกจนถึงปี 2026) การตัดสินใจต้องพิจารณาจากปัจจัยจริงของระบบ
  ไม่ใช่ตามกระแสความนิยม

### แบบฝึกหัดที่ 739.1

**โจทย์**: จงยกตัวอย่างสถานการณ์ธุรกิจ 1 ตัวอย่างที่การคง Indexed File ไว้ (ไม่ย้ายไปฐานข้อมูล)
ยังคงเป็นทางเลือกที่สมเหตุสมผล

**เฉลยแนวทาง**: ระบบตรวจสอบยอดคงเหลือสิทธิลาพักร้อนของพนักงานในบริษัทขนาดกลางที่มีพนักงานไม่กี่พันคน
และมีการค้นหาไม่บ่อยนัก (เช่น ฝ่ายบุคคลตรวจสอบวันละไม่กี่สิบครั้ง) กรณีนี้ Indexed File ที่มีอยู่เดิม
(ถ้าระบบเดิมใช้ COBOL/Mainframe) ยังคงเพียงพอและเสถียร การลงทุนย้ายไปฐานข้อมูลสมัยใหม่อาจไม่คุ้มค่า
กับความเสี่ยงและต้นทุนที่ต้องใช้ เมื่อเทียบกับประโยชน์ที่ได้ (เพราะปริมาณและความถี่ในการเข้าถึงข้อมูล
ต่ำมาก)

---

## ขั้นตอนที่ 740: สรุปเปรียบเทียบ Part 073 กับ Part 074 และแบบฝึกหัดรวม

### ตารางเปรียบเทียบสองระบบที่สร้างมา

| ประเด็น | Part 073 (ORDERCALC) | Part 074 (CUSTLOOKUP) |
|---|---|---|
| วิธีส่ง input เข้า COBOL | stdin (`ACCEPT FROM CONSOLE`) | command-line argument (`ACCEPT FROM ARGUMENT-VALUE`) |
| แหล่งข้อมูล | คำนวณสดจาก input ที่ส่งเข้ามา | อ่านจาก Indexed File ที่มีอยู่ก่อนแล้ว |
| HTTP Method ที่ใช้ | `POST` (ส่งคำสั่งซื้อ) | `GET` (ค้นหาข้อมูล) |
| ภาษาที่สาธิต Web Layer | Python เท่านั้น | Python และ Node.js |
| คอมไพเลอร์ที่ใช้ | `cobc` มาตรฐาน | `/opt/gnucobol-isam/bin/cobc` (ต้อง ISAM handler) |
| ความซับซ้อนของ runtime environment | ต่ำ (ไม่มี dependency พิเศษ) | สูงกว่า (ต้องตั้ง `LD_LIBRARY_PATH`) |

### สิ่งที่ยังคงเหมือนกันทั้งสอง Part — แก่นแท้ของ Wrapper Service Pattern

ไม่ว่ารายละเอียดจะต่างกันแค่ไหน แก่นสำคัญที่เหมือนกันทั้งสอง Part คือ:

1. โปรแกรม COBOL ไม่รู้จัก HTTP เลยแม้แต่น้อย
2. Web Layer ทำหน้าที่แปลข้อมูลและเรียก subprocess เท่านั้น ไม่มี business logic ของตัวเอง
3. การสื่อสารระหว่างสองฝั่งใช้ช่องทางง่ายที่สุดเท่าที่จะเป็นไปได้ (text ธรรมดาผ่าน stdin/stdout/
   argument/exit code) ไม่ต้องพึ่งพา library พิเศษใด ๆ ที่ COBOL ไม่รองรับ

### แบบฝึกหัดที่ 740.1 (แบบฝึกหัดรวม Part นี้)

**โจทย์**: จงออกแบบ (โดยไม่ต้องเขียนโค้ดเต็ม) endpoint ใหม่ `PUT /api/customer/<id>/balance`
ที่รับ JSON body `{"amount": 500.00}` แล้วนำไป**บวกเพิ่ม**เข้ากับยอดเงินคงเหลือของลูกค้าคนนั้นใน
`CUST074.DAT` จริง จงระบุว่าต้องแก้ไขอะไรบ้างทั้งฝั่ง COBOL และฝั่ง Web Layer

**เฉลยแนวทาง**:

1. **ฝั่ง COBOL**: ต้องเขียนโปรแกรมใหม่ (เช่น `CUSTUPDATE.cob`) ที่เปิดไฟล์ด้วย `OPEN I-O`
   (ไม่ใช่ `OPEN INPUT` เหมือน `CUSTLOOKUP` เพราะต้องการทั้งอ่านและเขียน) รับ customer ID และ
   จำนวนเงินที่จะบวกเพิ่มเป็น 2 argument, `READ` record ด้วย Key เพื่อดึงยอดปัจจุบัน, บวกค่าใหม่เข้า
   กับ `CUST-BALANCE`, แล้วใช้คำสั่ง `REWRITE` (ทบทวนจาก Part 025) เพื่อบันทึกค่าที่แก้ไขแล้วกลับลง
   ไฟล์ในตำแหน่งเดิม
2. **ฝั่ง Web Layer**: เพิ่ม routing สำหรับ HTTP method `PUT` ที่ path `/api/customer/<id>/balance`
   อ่านค่า `amount` จาก JSON body, ตรวจสอบว่าเป็นตัวเลขที่สมเหตุสมผล (validation เหมือนที่เรียนใน
   Part 073 ขั้นตอนที่ 728), แล้วเรียก `CUSTUPDATE` ผ่าน subprocess พร้อมส่ง customer ID และ
   amount เป็น argument, สุดท้ายแปลงผลลัพธ์ (ยอดเงินใหม่ที่ได้จาก stdout ของ COBOL) กลับเป็น JSON
   response

การออกแบบนี้แสดงให้เห็นว่า Wrapper Service Pattern รองรับการขยายไปสู่ operation ที่ซับซ้อนขึ้น
(การแก้ไขข้อมูล ไม่ใช่แค่การอ่านอย่างเดียว) ได้โดยยังคงหลักการแบ่งความรับผิดชอบเดิมไว้ครบถ้วน

---

## สรุปท้ายบท

ใน Part นี้ เราได้ต่อยอด Wrapper Service Pattern จาก Part 073 ให้ใกล้เคียงระบบจริงมากขึ้น:

- สร้างระบบ Customer Lookup ที่ COBOL program อ่านข้อมูลจริงจาก **Indexed File** (ทบทวนเทคนิคจาก
  Part 028) แทนการคำนวณจาก input อย่างเดียว
- ตรวจสอบและพิสูจน์จริงว่า `cobc` มาตรฐานในสภาพแวดล้อมนี้ไม่รองรับ `ORGANIZATION IS INDEXED`
  และแก้ปัญหาด้วยการใช้ `/opt/gnucobol-isam/bin/cobc` ที่ build พร้อม Berkeley DB
- เขียนโปรแกรม `CUSTSETUP` (สร้างข้อมูล) และ `CUSTLOOKUP` (ค้นหาข้อมูลด้วย
  `ACCEPT FROM ARGUMENT-VALUE` และ `READ ... INVALID KEY`) พร้อมทดสอบจริงทุกกรณี
- สร้าง Web Layer เดียวกันด้วยทั้ง **Python** (`http.server`) และ **Node.js** (`http` module +
  `child_process.execFile`) พิสูจน์ว่า Wrapper Service Pattern ไม่ผูกติดกับภาษาใดภาษาหนึ่ง
- **พบและแก้กับดักจริง**: Segmentation Fault ที่เกิดจาก `LD_LIBRARY_PATH` ไม่ถูกต้อง ทำให้ระบบโหลด
  `libcob.so.5` ผิดเวอร์ชัน พร้อมวิธีวินิจฉัยด้วย `ldd` และวิธีแก้ไขที่ทดสอบแล้วว่าใช้ได้จริง
- แนวคิดเรื่องประสิทธิภาพและการขยายระบบ (caching, long-running COBOL server, การย้ายไปฐานข้อมูล)
  พร้อมเกณฑ์การตัดสินใจว่าเมื่อไหร่ควรคง Indexed File ไว้และเมื่อไหร่ควรย้าย

ใน **Part 075** เราจะเจาะลึกทางเลือกสุดท้ายที่กล่าวถึงข้างต้น: การเชื่อมต่อ COBOL เข้ากับฐานข้อมูล
สมัยใหม่อย่าง **PostgreSQL** จริง ผ่านเทคนิค Shim Program ทั้งแบบเรียกคำสั่ง CLI และแบบ CALL
โปรแกรม C ที่ link กับ database client library โดยตรง

**[← กลับไป Part 073: COBOL และ REST API: การสร้าง Wrapper Service](part-073-cobol-rest-api-wrapper.md)** | **[ไปยัง Part 075: ฐานข้อมูลสมัยใหม่: COBOL กับ MySQL/PostgreSQL →](part-075-cobol-modern-databases.md)**
