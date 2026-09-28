# Part 055: VSAM เบื้องต้น: KSDS, ESDS, RRDS (ขั้นตอนที่ 541–550)

## คำนำของ Part นี้

Part 028-029 สอนให้เรารู้จัก **Indexed File** (ISAM) และ **Relative File** ใน COBOL อย่างละเอียด
— ทั้งสองแบบนี้ให้เรา `READ`/`WRITE`/`REWRITE`/`DELETE` ผ่าน Key หรือหมายเลข slot ได้โดยตรง
โดยที่เราใช้ GnuCOBOL รันบนเครื่องส่วนตัวได้จริงทุกตัวอย่าง คำถามที่ตามมาตอนนี้คือ: แล้วบน Mainframe
จริง ระบบไฟล์ที่ให้ความสามารถแบบเดียวกันนี้เรียกว่าอะไร? คำตอบคือ **VSAM (Virtual Storage Access
Method)** — ระบบจัดการไฟล์มาตรฐานของ IBM z/OS ที่เป็นรากฐานของแทบทุกระบบ Production บน Mainframe
ทั่วโลก รวมถึงเป็นรากฐานที่ DB2 (Part 057) และ CICS (Part 061) เองก็ยังใช้ VSAM เป็นชั้นเก็บข้อมูล
เบื้องหลังในบางส่วน

ข่าวดีคือ **แนวคิดของ VSAM แทบจะเหมือนกับสิ่งที่คุณเรียนไปแล้วใน Part 028-029 ทุกประการ** เพียงแค่
เปลี่ยนชื่อเรียกและเพิ่มรายละเอียดเฉพาะของ Mainframe เข้ามา Part นี้จะแนะนำ VSAM ทั้ง 3 ประเภทหลัก
(KSDS, ESDS, RRDS) พร้อมทั้ง**จับคู่แต่ละประเภทกับสิ่งที่คุณรู้จักอยู่แล้วอย่างชัดเจน** เพื่อให้คุณ
เห็นภาพว่าความรู้เดิมของคุณมีค่าแค่ไหนเมื่อก้าวเข้าสู่โลก Mainframe จริง

> **สำคัญมาก — ขอบเขตของสภาพแวดล้อมนี้**: VSAM เป็นเทคโนโลยีเฉพาะของ z/OS ที่ต้องอาศัยระบบ
> จัดการพื้นที่ดิสก์ (Catalog, VTOC) ของ Mainframe จริงเท่านั้น สภาพแวดล้อม Linux Sandbox ของ
> หลักสูตรนี้**ไม่มี VSAM ของจริง และไม่สามารถ DEFINE CLUSTER หรือรัน IDCAMS ได้เลย** ไวยากรณ์
> IDCAMS ทุกชิ้นในเอกสารนี้เป็น**ไวยากรณ์มาตรฐานอ้างอิงที่ถูกต้อง**สำหรับให้คุณอ่านเป็นและเขียนได้
> แต่ต้องใช้ z/OS จริงหรือ Mainframe Emulator (Hercules) จึงจะรันได้จริง — ในทางกลับกัน **โค้ด
> COBOL ฝั่ง SELECT/FD ที่ใช้ `ORGANIZATION IS INDEXED`/`SEQUENTIAL`/`RELATIVE` เป็นไวยากรณ์
> COBOL มาตรฐานเดียวกับที่ Part 028-029 ใช้** และ**ทดสอบคอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุก
> ตัวอย่างในเอกสารนี้** (ใช้ build ที่เปิดใช้งาน indexed file handler ตามที่ Part 028 อธิบายไว้)

---

## ขั้นตอนที่ 541: VSAM คืออะไร — ภาพรวมและส่วนประกอบของ Cluster

### VSAM คืออะไร

**VSAM (Virtual Storage Access Method)** คือ Access Method (วิธีการเข้าถึงข้อมูลบนดิสก์) ของ
z/OS ที่เปิดตัวในช่วงต้นทศวรรษ 1970 เพื่อแทนที่ระบบเก่าอย่าง ISAM/BDAM ดั้งเดิมของ IBM ด้วย
สถาปัตยกรรมที่ยืดหยุ่นและมีประสิทธิภาพกว่า คำว่า "Virtual Storage" ในชื่อสะท้อนว่า VSAM ถูก
ออกแบบมาให้ทำงานร่วมกับหน่วยความจำเสมือน (Virtual Storage) ของ OS/VS ซึ่งเป็นเทคโนโลยีใหม่ในยุค
นั้น

### เปรียบเทียบ VSAM กับ Non-VSAM Access Methods

| Access Method | ลักษณะ | เทียบเท่าใน COBOL ที่เราเรียนไปแล้ว |
|---|---|---|
| **QSAM/BSAM** (Non-VSAM) | อ่าน/เขียนตามลำดับ ไม่มีดัชนี | `ORGANIZATION IS SEQUENTIAL` (Part 023-027) |
| **VSAM KSDS** | เก็บพร้อมดัชนีตาม Key ที่ไม่ซ้ำกัน | `ORGANIZATION IS INDEXED` (Part 028) |
| **VSAM ESDS** | เก็บตามลำดับการเขียนจริง ต่อท้ายเสมอ | `ORGANIZATION IS SEQUENTIAL` (แนวคิดคล้ายกัน) |
| **VSAM RRDS** | เก็บในตำแหน่ง (slot) ที่กำหนดด้วยหมายเลข | `ORGANIZATION IS RELATIVE` (Part 029) |

### ส่วนประกอบของ VSAM Cluster

เมื่อพูดถึงไฟล์ VSAM หนึ่งไฟล์ เราเรียกมันว่า **Cluster** ซึ่งจริง ๆ แล้วประกอบด้วยส่วนประกอบ
ภายในที่แยกจากกัน (แต่ผู้ใช้ทั่วไปมองเห็นเป็นก้อนเดียว):

- **Data Component**: พื้นที่จริงที่เก็บ record ข้อมูล
- **Index Component**: โครงสร้างดัชนี B-Tree ที่ใช้ค้นหา record อย่างรวดเร็ว (มีเฉพาะใน KSDS
  และ Alternate Index เท่านั้น — ESDS/RRDS ไม่มี Index Component เพราะเข้าถึงด้วยวิธีอื่น)

แนวคิดนี้ตรงกับสิ่งที่ Part 028 อธิบายไว้แล้วทุกประการ: GnuCOBOL ใช้ไลบรารี Berkeley DB (BDB)
จัดการโครงสร้าง B-Tree ให้อัตโนมัติเบื้องหลัง `ORGANIZATION IS INDEXED` ส่วน VSAM บน z/OS ก็ทำ
หน้าที่แบบเดียวกันนี้เอง เพียงแต่เป็นการ implement ของ IBM เองในระดับระบบปฏิบัติการ

### Control Interval (CI) และ Control Area (CA)

VSAM จัดสรรพื้นที่ดิสก์เป็นหน่วยเรียกว่า **Control Interval (CI)** — บล็อกข้อมูลขนาดคงที่ (เช่น
4096 หรือ 8192 ไบต์) ที่เป็นหน่วยพื้นฐานที่สุดในการอ่าน/เขียนจากดิสก์ในครั้งเดียว หลาย CI รวมกัน
เป็น **Control Area (CA)** เมื่อ CI เต็มและต้องแทรก record ใหม่เข้าไปตรงกลาง (สำหรับ KSDS) VSAM
จะทำ **CI Split** (แบ่งครึ่ง CI เดิมเป็นสองส่วน) โดยอัตโนมัติเพื่อสร้างที่ว่างรองรับข้อมูลใหม่ —
เป็นกลไกเบื้องหลังที่ทำให้แทรก record กลางไฟล์ได้โดยไม่ต้องย้ายข้อมูลทั้งไฟล์ (จะกล่าวถึงผลกระทบ
ของ CI Split ต่อประสิทธิภาพในขั้นตอนที่ 558 ของ Part 056)

### ข้อควรระวัง

- อย่าสับสนคำว่า "Cluster" ของ VSAM กับ "Cluster" ในความหมายอื่น (เช่น Server Cluster) — ในบริบท
  VSAM คำนี้หมายถึง "ไฟล์ VSAM หนึ่งไฟล์" เท่านั้น
- ขนาด CI ที่เล็กเกินไปทำให้เกิด CI Split บ่อย (performance แย่ลง) ส่วนขนาดใหญ่เกินไปทำให้เปลือง
  พื้นที่ดิสก์โดยไม่จำเป็น การเลือกขนาด CI ที่เหมาะสมเป็นงาน Performance Tuning ระดับ Mainframe
  System Programmer (จะกล่าวถึงเพิ่มเติมใน Part 068)

### แบบฝึกหัดที่ 541.1

**โจทย์**: จงอธิบายว่าทำไม ESDS และ RRDS จึงไม่มี Index Component ในขณะที่ KSDS มี

**เฉลย**: เพราะ ESDS เข้าถึงข้อมูลตามลำดับการเขียนจริงเท่านั้น (ไม่มีการค้นหาด้วย Key) และ RRDS
เข้าถึงข้อมูลด้วยการคำนวณตำแหน่ง slot โดยตรงจากหมายเลข Relative Record Number (คำนวณตำแหน่งทาง
กายภาพได้ทันทีโดยไม่ต้องค้นหา) ทั้งสองแบบจึงไม่จำเป็นต้องมีโครงสร้างดัชนี B-Tree แยกต่างหากเหมือน
KSDS ที่ต้องค้นหา record จากค่า Key ที่อาจไม่เรียงตามตำแหน่งทางกายภาพ

---

## ขั้นตอนที่ 542: KSDS (Key Sequenced Data Set) — แนวคิดและการ Map กับ COBOL INDEXED

### KSDS คืออะไร

**KSDS (Key Sequenced Data Set)** คือ VSAM Cluster ที่เก็บ record โดยเรียงตามค่าของ **Key
ที่ไม่ซ้ำกัน (unique key)** พร้อม Index Component สำหรับค้นหา record ใดก็ได้โดยตรงผ่านค่า Key
โดยไม่ต้องอ่านไล่ตั้งแต่ต้นไฟล์ — **นี่คือสิ่งเดียวกันทุกประการกับที่ Part 028 สอนผ่าน
`ORGANIZATION IS INDEXED`** เพียงแต่บน Mainframe จริงจะถูกสร้างผ่าน utility ชื่อ **IDCAMS**
(จะสอนไวยากรณ์ในขั้นตอนที่ 543) แทนที่จะสร้างจากภายในโปรแกรม COBOL โดยตรงแบบที่ GnuCOBOL ทำได้

### ตารางเทียบคำศัพท์ VSAM KSDS กับ COBOL INDEXED File

| คำศัพท์ VSAM (z/OS) | คำศัพท์ COBOL (GnuCOBOL / มาตรฐาน) |
|---|---|
| Cluster | ไฟล์ (`SELECT ... ASSIGN TO`) |
| Prime Key | `RECORD KEY IS` |
| Alternate Index (AIX) | `ALTERNATE RECORD KEY IS` (Part 028 ขั้นตอน 279, ขยายเพิ่มใน Part 056) |
| GET | `READ` |
| PUT (insert ใหม่) | `WRITE` |
| PUT (update ของเดิม) | `REWRITE` |
| ERASE | `DELETE` |
| Sequential Browse | `READ ... NEXT RECORD` |

### ตัวอย่างโค้ด: KSDS-Style Account File (Working Analog)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP542-KSDS-ANALOG.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-FILE ASSIGN TO "ACCT542.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-NUMBER.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-FILE.
       01  ACCOUNT-RECORD.
           05  ACCT-NUMBER          PIC 9(8).
           05  ACCT-HOLDER-NAME     PIC X(20).
           05  ACCT-BALANCE         PIC 9(9)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> This COBOL file layout is a direct working analog of a
      *> VSAM KSDS cluster: fixed-length records, one unique key
      *> field, direct access by key. On z/OS this SELECT clause
      *> would instead point to a VSAM KSDS via JCL DD, but the
      *> COBOL-level logic below is identical either way.
           OPEN OUTPUT ACCOUNT-FILE.
           MOVE 10000001 TO ACCT-NUMBER.
           MOVE "SOMCHAI JAIDEE" TO ACCT-HOLDER-NAME.
           MOVE 15000.00 TO ACCT-BALANCE.
           WRITE ACCOUNT-RECORD.

           MOVE 10000002 TO ACCT-NUMBER.
           MOVE "SUDA MEECHAI" TO ACCT-HOLDER-NAME.
           MOVE 8250.50 TO ACCT-BALANCE.
           WRITE ACCOUNT-RECORD.
           CLOSE ACCOUNT-FILE.

           OPEN I-O ACCOUNT-FILE.
           MOVE 10000002 TO ACCT-NUMBER.
           READ ACCOUNT-FILE
               INVALID KEY
                   DISPLAY "Account not found."
               NOT INVALID KEY
                   DISPLAY "KSDS-style direct read by key: "
                       ACCT-HOLDER-NAME " balance=" ACCT-BALANCE
           END-READ.
           CLOSE ACCOUNT-FILE.
           STOP RUN.
```

คอมไพล์และรัน (ต้องใช้ build ที่เปิด indexed file handler ตามที่ Part 028 อธิบาย):

```bash
cobc -x -o step542 step542.cob
./step542
```

**ผลลัพธ์จริง (ยืนยันแล้วด้วย GnuCOBOL build ที่มี BDB indexed handler):**

```
KSDS-style direct read by key: SUDA MEECHAI         balance=000008250.50
```

### อธิบายจุดสำคัญ

- โค้ดนี้**เหมือนกับที่เรียนใน Part 028 ทุกประการ** เพียงเปลี่ยนชื่อฟิลด์และเนื้อหาตัวอย่างให้เข้า
  บริบทระบบธนาคาร — จุดประสงค์คือเพื่อยืนยันว่าความรู้จาก Part 028 **ใช้ได้โดยตรง 100%** เมื่อ
  ทำความเข้าใจ KSDS
- บน z/OS จริง `SELECT ACCOUNT-FILE ASSIGN TO ...` จะระบุชื่อ **DDNAME** (เช่น `ACCTFILE`) แทน
  ชื่อไฟล์กายภาพ แล้วให้ JCL DD statement เป็นตัวเชื่อม DDNAME นั้นเข้ากับชื่อ VSAM Cluster จริง
  ผ่าน `//ACCTFILE DD DSN=PROD.ACCOUNT.KSDS,DISP=SHR` — Part 052 ได้แนะนำแนวคิด DD ไปแล้ว
- `RECORD KEY IS ACCT-NUMBER` ในโค้ด COBOL ตรงกับสิ่งที่ IDCAMS เรียกว่า **KEYS parameter**
  ตอน DEFINE CLUSTER (จะสอนไวยากรณ์ในขั้นตอนที่ 543)

### ข้อควรระวัง

- แม้ไวยากรณ์ COBOL จะเหมือนกันทุกประการ แต่**การจองพื้นที่ดิสก์จริงของ VSAM บน z/OS ต้องทำผ่าน
  IDCAMS ล่วงหน้าเสมอ** ก่อนโปรแกรม COBOL จะเปิดไฟล์ได้ ต่างจาก GnuCOBOL ที่ `OPEN OUTPUT`
  สามารถสร้างไฟล์ใหม่ได้เองจากภายในโปรแกรมโดยตรง — นี่คือความแตกต่างเชิง Operational ที่สำคัญ
  ที่สุดระหว่างสองสภาพแวดล้อม
- ค่า Key เปรียบเทียบแบบไบต์ต่อไบต์เช่นเดียวกับที่ Part 028 ขั้นตอน 272 พิสูจน์ไว้ — กฎ "ใช้
  `PIC 9` เพื่อการเรียงลำดับตัวเลขที่ถูกต้อง" ยังคงใช้ได้เต็มรูปแบบกับ VSAM KSDS เช่นกัน

### แบบฝึกหัดที่ 542.1

**โจทย์**: จงจับคู่คำศัพท์ VSAM ต่อไปนี้กับคำสั่ง COBOL ที่ตรงกัน: (ก) GET (ข) PUT สำหรับ record
ใหม่ (ค) ERASE

**เฉลย**: (ก) GET → `READ` (ข) PUT สำหรับ record ใหม่ → `WRITE` (ค) ERASE → `DELETE`
(ถ้าเป็น PUT สำหรับ record ที่มีอยู่แล้วเพื่ออัปเดต จะตรงกับ `REWRITE` แทน)

---

## ขั้นตอนที่ 543: IDCAMS DEFINE CLUSTER — สร้าง KSDS บน z/OS

### IDCAMS คืออะไร

**IDCAMS (Access Method Services)** คือ utility program มาตรฐานของ z/OS สำหรับจัดการ VSAM
Cluster ทุกประเภท (สร้าง ลบ คัดลอก แสดงรายละเอียด) เรียกใช้ผ่าน JCL โดยระบุ `PGM=IDCAMS` แล้ว
ป้อนคำสั่งผ่าน `SYSIN` — เปรียบได้กับการที่ GnuCOBOL ใช้ `OPEN OUTPUT` จากภายในโปรแกรมเพื่อสร้าง
ไฟล์ใหม่ แต่ VSAM แยกขั้นตอน "สร้างไฟล์" ออกจาก "เขียนข้อมูลลงไฟล์" อย่างชัดเจนคนละขั้นตอนเสมอ

### ไวยากรณ์ DEFINE CLUSTER สำหรับ KSDS (อ้างอิงมาตรฐาน — รันไม่ได้ในสภาพแวดล้อมนี้)

```jcl
//DEFKSDS  JOB (ACCT123),'DEFINE KSDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE CLUSTER (NAME(PROD.ACCOUNT.KSDS)          -
                  INDEXED                          -
                  KEYS(8 0)                        -
                  RECORDSIZE(37 37)                -
                  RECORDS(10000 5000)               -
                  FREESPACE(10 10)                  -
                  SHAREOPTIONS(2 3)                 -
                  VOLUMES(VSAM01) )                 -
         DATA  (NAME(PROD.ACCOUNT.KSDS.DATA))       -
         INDEX (NAME(PROD.ACCOUNT.KSDS.INDEX))
/*
```

### อธิบายพารามิเตอร์ทีละส่วน

- `NAME(PROD.ACCOUNT.KSDS)`: ชื่อ Cluster ตามกฎการตั้งชื่อ dataset ของ z/OS (Part 054
  ขั้นตอน 533)
- `INDEXED`: ระบุประเภท Cluster เป็น KSDS (เทียบเท่า `ORGANIZATION IS INDEXED` ใน COBOL)
- `KEYS(8 0)`: ความยาวของ Key คือ 8 ไบต์ เริ่มต้นที่ offset 0 ของ record (ตรงกับ `ACCT-NUMBER
  PIC 9(8)` ในตัวอย่างขั้นตอนที่ 542 ซึ่งกว้าง 8 ไบต์และอยู่ตำแหน่งแรกสุดของ record)
- `RECORDSIZE(37 37)`: ขนาด record ต่ำสุดและสูงสุดเท่ากันที่ 37 ไบต์ (fixed-length) ตรงกับผลรวม
  ของทุก PIC ใน `ACCOUNT-RECORD` (8+20+9 = 37 ไบต์พอดี)
- `RECORDS(10000 5000)`: จองพื้นที่เริ่มต้นสำหรับ 10,000 record และขยายเพิ่มครั้งละ 5,000 record
  เมื่อพื้นที่เต็ม (เทียบเท่าแนวคิด Primary/Secondary quantity ที่ Part 054 ขั้นตอน 536 แนะนำ)
- `FREESPACE(10 10)`: กันพื้นที่ว่าง 10% ไว้ในแต่ละ CI และ CA เพื่อรองรับการแทรก record ใหม่โดย
  ลดการเกิด CI Split (ขั้นตอนที่ 541)
- `SHAREOPTIONS(2 3)`: กำหนดว่าหลาย job/region เข้าถึง Cluster นี้พร้อมกันได้ในระดับใด (ค่า
  `(2 3)` หมายถึงอนุญาตให้หลาย region อ่านพร้อมกันได้ แต่เขียนได้ทีละ region เท่านั้น)
- `DATA (NAME(...))` และ `INDEX (NAME(...))`: กำหนดชื่อแยกให้ Data Component และ Index
  Component อย่างชัดเจน (ตามที่อธิบายไว้ในขั้นตอนที่ 541)

### ข้อควรระวัง

- **ไวยากรณ์ IDCAMS นี้เป็นไวยากรณ์มาตรฐานอ้างอิงที่ถูกต้องตามเอกสาร IBM แต่ไม่สามารถรันได้ใน
  สภาพแวดล้อมของหลักสูตรนี้** เนื่องจากไม่มี z/OS/VSAM ติดตั้งอยู่ ต้องใช้ z/OS จริงหรือ Mainframe
  Emulator (Hercules) เพื่อทดสอบรันจริง
- เครื่องหมาย `-` ท้ายบรรทัดคือ**ตัวต่อบรรทัด (continuation character)** ของ IDCAMS command
  (ต่างจาก column 7 continuation ของ COBOL fixed-format) จำเป็นเมื่อคำสั่งยาวเกินบรรทัดเดียว
- ต้องคำนวณ `RECORDSIZE`/`KEYS` ให้ตรงกับ FD ของโปรแกรม COBOL เป๊ะ หากไม่ตรงกัน โปรแกรมจะ
  `OPEN` ไฟล์ไม่สำเร็จ (File Status `39` — จะสอนละเอียดในขั้นตอนที่ 549)

### แบบฝึกหัดที่ 543.1

**โจทย์**: ถ้า record ของ COBOL มีขนาด 50 ไบต์คงที่เสมอ (ไม่ใช่ variable-length) จะต้องเขียน
`RECORDSIZE` อย่างไรใน DEFINE CLUSTER

**เฉลย**: `RECORDSIZE(50 50)` — ระบุค่าต่ำสุดและสูงสุดเท่ากันเพื่อบอกว่าเป็น fixed-length record
ทุก record ในไฟล์นี้มีขนาด 50 ไบต์เสมอไม่มีข้อยกเว้น

---

## ขั้นตอนที่ 544: ESDS (Entry Sequenced Data Set) — แนวคิดและการ Map กับ COBOL SEQUENTIAL

### ESDS คืออะไร

**ESDS (Entry Sequenced Data Set)** คือ VSAM Cluster ที่เก็บ record **เรียงตามลำดับการเขียนจริง
เท่านั้น** (physical insertion order) ไม่มี Key ใด ๆ ไม่มี Index Component และ**ไม่สามารถแทรก
record กลางไฟล์ได้** — record ใหม่จะถูกต่อท้ายไฟล์เสมอ (append-only) นี่คือสิ่งเดียวกันกับ
แนวคิดของ **`ORGANIZATION IS SEQUENTIAL`** ที่ Part 023-027 สอนไปแล้ว โดยเฉพาะพฤติกรรมที่ Part
025 ขั้นตอนที่ 247 พิสูจน์ว่า **`DELETE` ใช้กับไฟล์ Sequential ไม่ได้เลยแม้แต่ในทางทฤษฎี** —
ESDS ก็มีข้อจำกัดแบบเดียวกันทุกประการบน VSAM: **ห้าม DELETE record จาก ESDS ได้เช่นกัน**

### ตัวอย่างโค้ด: ESDS-Style Transaction Log (Working Analog)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP544-ESDS-ANALOG.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT LOG-FILE ASSIGN TO "TXNLOG544.DAT"
               ORGANIZATION IS SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  LOG-FILE.
       01  LOG-RECORD.
           05  LOG-SEQ-NO           PIC 9(6).
           05  LOG-TXN-TYPE         PIC X(10).
           05  LOG-AMOUNT           PIC 9(7)V99.

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> This is a working analog of a VSAM ESDS: records are
      *> physically appended in the order they are WRITTEN and
      *> can only be READ back in that same physical order - there
      *> is no key-based direct access at all, exactly like ESDS.
           OPEN OUTPUT LOG-FILE.
           MOVE 1 TO LOG-SEQ-NO.
           MOVE "DEPOSIT" TO LOG-TXN-TYPE.
           MOVE 500.00 TO LOG-AMOUNT.
           WRITE LOG-RECORD.

           MOVE 2 TO LOG-SEQ-NO.
           MOVE "WITHDRAW" TO LOG-TXN-TYPE.
           MOVE 200.00 TO LOG-AMOUNT.
           WRITE LOG-RECORD.

           MOVE 3 TO LOG-SEQ-NO.
           MOVE "DEPOSIT" TO LOG-TXN-TYPE.
           MOVE 1000.00 TO LOG-AMOUNT.
           WRITE LOG-RECORD.
           CLOSE LOG-FILE.

           DISPLAY "ESDS-style log, read back in physical write order:".
           OPEN INPUT LOG-FILE.
           PERFORM UNTIL END-OF-FILE
               READ LOG-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  #" LOG-SEQ-NO " " LOG-TXN-TYPE
                           " " LOG-AMOUNT
               END-READ
           END-PERFORM.
           CLOSE LOG-FILE.
           STOP RUN.
```

คอมไพล์และรัน (ไฟล์ Sequential ธรรมดา ไม่ต้องใช้ ISAM build พิเศษ):

```bash
cobc -x -o step544 step544.cob
./step544
```

**ผลลัพธ์จริง (ยืนยันแล้ว):**

```
ESDS-style log, read back in physical write order:
  #000001 DEPOSIT    0000500.00
  #000002 WITHDRAW   0000200.00
  #000003 DEPOSIT    0001000.00
```

### ทำไม ESDS จึงเหมาะกับงาน Transaction Log

ลักษณะ append-only ของ ESDS ตรงกับความต้องการของงาน **Audit Trail / Transaction Log** ในระบบ
ธนาคารพอดี: ต้องการบันทึกทุกธุรกรรมตามลำดับเวลาที่เกิดขึ้นจริง **ห้ามแก้ไขหรือลบข้อมูลย้อนหลัง**
(เพื่อความน่าเชื่อถือทางบัญชี/กฎหมาย) — ข้อจำกัดที่ดูเหมือนเป็นจุดอ่อน (แก้ไข/ลบไม่ได้) กลับ
กลายเป็นจุดแข็งที่พอดีกับความต้องการของงานประเภทนี้เป๊ะ

### ข้อควรระวัง

- แม้ ESDS จะไม่มี Key หลักของตัวเอง แต่**สามารถมี Alternate Index ผูกเข้ากับ field ใดก็ได้ใน
  record ได้** (จะสอนรายละเอียดเพิ่มเติมใน Part 056) ทำให้ค้นหาข้อมูลใน ESDS ผ่านค่า field
  บางตัวได้โดยอ้อม แม้ตัว ESDS เองจะไม่มีแนวคิด Primary Key ก็ตาม
- **RRN (Relative Byte Address)** คือค่าที่ VSAM ใช้อ้างอิงตำแหน่งของแต่ละ record ภายใน ESDS
  (คล้ายเลข offset ในไฟล์) แต่ต่างจาก Relative Record Number ของ RRDS (ขั้นตอนที่ 546) ตรงที่
  RBA ของ ESDS อิงตำแหน่ง byte จริงบนดิสก์ ไม่ใช่หมายเลข slot ลำดับที่

### แบบฝึกหัดที่ 544.1

**โจทย์**: จงอธิบายว่าทำไม ESDS จึงเหมาะกับงาน Transaction Log ของธนาคารมากกว่า KSDS ทั้งที่
KSDS ดูมีความสามารถมากกว่า (ค้นหาได้ทั้ง Insert/Update/Delete)

**เฉลย**: เพราะงาน Transaction Log ต้องการคุณสมบัติ **ห้ามแก้ไขหรือลบข้อมูลย้อนหลังเด็ดขาด**
เพื่อรักษาความถูกต้องของหลักฐานทางบัญชี ความสามารถ "แก้ไข/ลบได้" ของ KSDS กลับเป็นความเสี่ยง
(อาจมีคนแก้ไขปลอมแปลงข้อมูลย้อนหลังได้) ในขณะที่ข้อจำกัด append-only ของ ESDS **บังคับ**ให้ข้อมูล
เขียนตามลำดับเวลาเท่านั้นและไม่สามารถลบทิ้งได้ ตรงกับความต้องการด้านความน่าเชื่อถือของ Audit
Trail พอดี

---

## ขั้นตอนที่ 545: IDCAMS DEFINE CLUSTER สำหรับ ESDS และ RRDS

### DEFINE CLUSTER สำหรับ ESDS (อ้างอิงมาตรฐาน)

```jcl
//DEFESDS  JOB (ACCT123),'DEFINE ESDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE CLUSTER (NAME(PROD.TXNLOG.ESDS)           -
                  NONINDEXED                        -
                  RECORDSIZE(23 23)                 -
                  RECORDS(50000 20000)               -
                  SHAREOPTIONS(2 3)                  -
                  VOLUMES(VSAM01) )
/*
```

สังเกตว่าใช้ **`NONINDEXED`** แทน `INDEXED` เพื่อระบุว่าเป็น ESDS และ**ไม่มี** parameter
`KEYS(...)` เลย เพราะ ESDS ไม่มีแนวคิด Key และไม่มี Index Component ให้กำหนด

### DEFINE CLUSTER สำหรับ RRDS (อ้างอิงมาตรฐาน)

```jcl
//DEFRRDS  JOB (ACCT123),'DEFINE RRDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE CLUSTER (NAME(PROD.TELLER.RRDS)           -
                  NUMBERED                          -
                  RECORDSIZE(28 28)                 -
                  RECORDS(500 100)                   -
                  SHAREOPTIONS(2 3)                  -
                  VOLUMES(VSAM01) )
/*
```

ใช้ **`NUMBERED`** เพื่อระบุว่าเป็น RRDS — record แต่ละตัวจะถูกอ้างอิงด้วยหมายเลข **Relative
Record Number (RRN)** แทน Key และเช่นเดียวกับ ESDS คือไม่มี parameter `KEYS(...)`

### ตารางสรุปคีย์เวิร์ดที่แยกความแตกต่างของ VSAM ทั้ง 3 ประเภท

| ประเภท | คีย์เวิร์ดใน DEFINE CLUSTER | มี KEYS? | มี Index Component? |
|---|---|---|---|
| KSDS | `INDEXED` | มี | มี |
| ESDS | `NONINDEXED` | ไม่มี | ไม่มี |
| RRDS | `NUMBERED` | ไม่มี (ใช้ RRN แทน) | ไม่มี |

### ข้อควรระวัง

- **ไวยากรณ์ IDCAMS ทั้งหมดในขั้นตอนนี้เป็นไวยากรณ์มาตรฐานอ้างอิง ไม่สามารถรันได้ในสภาพแวดล้อม
  ของหลักสูตรนี้** ต้องใช้ z/OS จริงหรือ Mainframe Emulator เท่านั้น
- อย่าลืมว่า RRDS ต้องกำหนด `RECORDSIZE` แบบ fixed-length เท่านั้น (ค่าต่ำสุดและสูงสุดต้องเท่ากัน
  เสมอ) เพราะ VSAM ต้องคำนวณตำแหน่งของแต่ละ slot จากขนาด record ที่คงที่ ถ้า record ความยาวไม่
  เท่ากัน VSAM จะคำนวณตำแหน่ง slot ผิดพลาดทันที

### แบบฝึกหัดที่ 545.1

**โจทย์**: จงอธิบายว่าทำไม RRDS จึงต้องการ `RECORDSIZE` แบบ fixed-length เท่านั้น ในขณะที่ ESDS
สามารถรองรับ record ความยาวไม่เท่ากันได้ (variable-length)

**เฉลย**: เพราะ RRDS คำนวณตำแหน่งทางกายภาพของแต่ละ record จากสูตร "หมายเลข slot คูณด้วยขนาด
record คงที่" ถ้า record มีความยาวไม่เท่ากัน สูตรนี้จะคำนวณตำแหน่งผิดทันที ในขณะที่ ESDS เข้าถึง
record ผ่าน RBA (ตำแหน่ง byte จริงที่บันทึกไว้ตอนเขียน) ซึ่งไม่ต้องพึ่งพาการคำนวณจากขนาดคงที่ จึง
รองรับ record ที่ความยาวแตกต่างกันได้

---

## ขั้นตอนที่ 546: RRDS (Relative Record Data Set) — แนวคิดและการ Map กับ COBOL RELATIVE

### RRDS คืออะไร

**RRDS (Relative Record Data Set)** คือ VSAM Cluster ที่เก็บ record ในตำแหน่ง (slot) ที่มี
หมายเลขกำกับตายตัวตั้งแต่ 1 เป็นต้นไป การเข้าถึง record ทำผ่าน **Relative Record Number (RRN)**
โดยตรง — คำนวณตำแหน่งทางกายภาพได้ทันทีโดยไม่ต้องค้นหาผ่านดัชนีใด ๆ ทำให้เร็วที่สุดในบรรดา VSAM
ทั้ง 3 ประเภทเมื่อรู้หมายเลข slot ที่ต้องการล่วงหน้า — **นี่คือสิ่งเดียวกันทุกประการกับ
`ORGANIZATION IS RELATIVE`** ที่ Part 029 สอนไปแล้ว

### ตัวอย่างโค้ด: RRDS-Style Teller Slot File (Working Analog)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP546-RRDS-ANALOG.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SLOT-FILE ASSIGN TO "SLOT546.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS RANDOM
               RELATIVE KEY IS WS-SLOT-NO.

       DATA DIVISION.
       FILE SECTION.
       FD  SLOT-FILE.
       01  SLOT-RECORD.
           05  SLOT-TELLER-NAME     PIC X(20).
           05  SLOT-STATUS          PIC X(8).

       WORKING-STORAGE SECTION.
       01  WS-SLOT-NO               PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> This is a working analog of a VSAM RRDS: each record is
      *> stored in a fixed physical slot numbered 1, 2, 3, ... and
      *> is accessed directly by that RELATIVE KEY slot number -
      *> exactly the same addressing model RRDS uses on z/OS.
           OPEN OUTPUT SLOT-FILE.
           MOVE 1 TO WS-SLOT-NO.
           MOVE "TELLER WINDOW 1" TO SLOT-TELLER-NAME.
           MOVE "OPEN" TO SLOT-STATUS.
           WRITE SLOT-RECORD.

           MOVE 2 TO WS-SLOT-NO.
           MOVE "TELLER WINDOW 2" TO SLOT-TELLER-NAME.
           MOVE "CLOSED" TO SLOT-STATUS.
           WRITE SLOT-RECORD.

           MOVE 3 TO WS-SLOT-NO.
           MOVE "TELLER WINDOW 3" TO SLOT-TELLER-NAME.
           MOVE "OPEN" TO SLOT-STATUS.
           WRITE SLOT-RECORD.
           CLOSE SLOT-FILE.

           OPEN I-O SLOT-FILE.
           MOVE 2 TO WS-SLOT-NO.
           READ SLOT-FILE
               INVALID KEY
                   DISPLAY "Slot 2 not found."
               NOT INVALID KEY
                   DISPLAY "RRDS-style direct read of slot 2: "
                       SLOT-TELLER-NAME " " SLOT-STATUS
           END-READ.

           MOVE "MAINT" TO SLOT-STATUS.
           REWRITE SLOT-RECORD
               INVALID KEY
                   DISPLAY "Rewrite of slot 2 failed."
           END-REWRITE.
           CLOSE SLOT-FILE.

           OPEN INPUT SLOT-FILE.
           MOVE 2 TO WS-SLOT-NO.
           READ SLOT-FILE
               INVALID KEY
                   DISPLAY "Slot 2 not found."
               NOT INVALID KEY
                   DISPLAY "Confirmed on disk, slot 2: "
                       SLOT-TELLER-NAME " " SLOT-STATUS
           END-READ.
           CLOSE SLOT-FILE.
           STOP RUN.
```

คอมไพล์และรัน (ไฟล์ Relative ธรรมดา ไม่ต้องใช้ ISAM build พิเศษ):

```bash
cobc -x -o step546 step546.cob
./step546
```

**ผลลัพธ์จริง (ยืนยันแล้ว):**

```
RRDS-style direct read of slot 2: TELLER WINDOW 2      CLOSED
Confirmed on disk, slot 2: TELLER WINDOW 2      MAINT
```

### ทำไม RRDS จึงเหมาะกับงานที่มีจำนวน record คงที่และรู้ตำแหน่งล่วงหน้า

RRDS เหมาะกับสถานการณ์ที่ "หมายเลข" มีความหมายทางธุรกิจอยู่แล้วโดยธรรมชาติ เช่น หมายเลขช่องบริการ
ของธนาคาร (Teller Window), หมายเลขที่นั่งเครื่องบิน, หรือหมายเลขล็อตในคลังสินค้า — กรณีเหล่านี้ไม่
จำเป็นต้องมี Key แบบตัวอักษรหรือแบบซับซ้อน เพราะ "หมายเลขลำดับ" คือ Key ที่เป็นธรรมชาติของข้อมูล
นั้นอยู่แล้ว ทำให้ RRDS เร็วกว่า KSDS อย่างมีนัยสำคัญในสถานการณ์แบบนี้ เพราะไม่ต้องเสียเวลาค้นหา
ผ่าน B-Tree Index เลย

### ข้อควรระวัง

- RRDS จองพื้นที่**ตายตัวสำหรับทุก slot ตั้งแต่แรก** แม้ slot นั้นจะยังไม่มีข้อมูลจริง (unused
  slot) ทำให้ RRDS มักใช้พื้นที่ดิสก์มากกว่า KSDS/ESDS ที่มีจำนวนข้อมูลเท่ากัน หากจำนวน slot ที่
  ใช้จริงน้อยกว่าที่จองไว้มาก
- เช่นเดียวกับ Part 029 ที่สอนไว้ RELATIVE KEY ต้องเป็นตัวแปรใน `WORKING-STORAGE SECTION`
  เท่านั้น (ไม่ใช่ field ภายใน record ของ FD) — สังเกตว่า `WS-SLOT-NO` ประกาศอยู่ใน
  `WORKING-STORAGE SECTION` ไม่ใช่ใน `SLOT-RECORD`

### แบบฝึกหัดที่ 546.1

**โจทย์**: จงอธิบายว่าทำไมการค้นหาลูกค้าด้วยเลขบัตรประชาชน 13 หลักจึงเหมาะกับ KSDS มากกว่า RRDS
ทั้งที่ทั้งสองแบบเข้าถึงข้อมูลได้โดยตรงเหมือนกัน

**เฉลย**: เพราะ RRDS ต้องการหมายเลข slot ที่เป็น**ลำดับต่อเนื่องเริ่มจาก 1** โดยไม่มีช่องว่างมาก
เกินไป (มิฉะนั้นจะเปลืองพื้นที่มหาศาลจาก unused slot) แต่เลขบัตรประชาชน 13 หลักเป็นค่าที่กระจัด
กระจายมาก ไม่ต่อเนื่อง และไม่สามารถใช้เป็นหมายเลข slot ได้โดยตรง (ไม่มีทางจองพื้นที่ 10^13 slot)
จึงต้องใช้ KSDS ที่มีโครงสร้างดัชนี B-Tree รองรับค่า Key ที่กระจัดกระจายแบบใดก็ได้แทน

---

## ขั้นตอนที่ 547: COBOL SELECT Clause สำหรับ VSAM บน z/OS จริง

### ความแตกต่างระหว่าง GnuCOBOL SELECT กับ z/OS COBOL SELECT

ตลอด Part นี้เราใช้ `ASSIGN TO "filename.DAT"` ซึ่งเป็นรูปแบบที่ GnuCOBOL ใช้ผูกกับไฟล์บนดิสก์
ของ Linux โดยตรง แต่บน z/OS จริง การเขียน `SELECT` สำหรับไฟล์ VSAM จะไม่ระบุชื่อไฟล์กายภาพเลย
แต่จะระบุ **DDNAME** แทน แล้วปล่อยให้ JCL DD statement เป็นตัวกำหนดว่า DDNAME นั้นชี้ไปยัง VSAM
Cluster ตัวใดจริง ๆ

### ตัวอย่างไวยากรณ์อ้างอิง: SELECT สำหรับ VSAM KSDS บน z/OS

```cobol
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-FILE ASSIGN TO ACCTFILE
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-NUMBER
               FILE STATUS IS WS-FILE-STATUS.
```

สังเกตว่า **`ASSIGN TO ACCTFILE`** ไม่มีเครื่องหมายคำพูดครอบ และไม่ใช่ชื่อไฟล์ — `ACCTFILE` คือ
**DDNAME** ที่ผูกกับ dataset จริงผ่าน JCL:

```jcl
//ACCTFILE DD  DSN=PROD.ACCOUNT.KSDS,DISP=SHR
```

### เปรียบเทียบ SELECT ระหว่าง GnuCOBOL กับ z/OS COBOL

| ส่วนประกอบ | GnuCOBOL (สภาพแวดล้อมหลักสูตรนี้) | z/OS COBOL (Mainframe จริง) |
|---|---|---|
| `ASSIGN TO` | ชื่อไฟล์จริงบนดิสก์ (ในเครื่องหมายคำพูด) | DDNAME (ไม่มีเครื่องหมายคำพูด) |
| การเชื่อมกับไฟล์จริง | ทันทีจากชื่อไฟล์ | ผ่าน JCL `//DDNAME DD DSN=...` |
| `ORGANIZATION IS INDEXED/SEQUENTIAL/RELATIVE` | เหมือนกันทุกประการ | เหมือนกันทุกประการ |
| `RECORD KEY IS` / `RELATIVE KEY IS` | เหมือนกันทุกประการ | เหมือนกันทุกประการ |

### ทำไมการออกแบบนี้ถึงสำคัญ

การแยก "ชื่อที่โปรแกรมอ้างอิง" (DDNAME) ออกจาก "ชื่อไฟล์จริง" (DSN) ทำให้**เปลี่ยนไฟล์ที่โปรแกรม
ทำงานด้วยได้โดยไม่ต้อง compile โปรแกรมใหม่เลย** เช่น รันโปรแกรมเดียวกันกับไฟล์ทดสอบในวันนี้ และ
รันกับไฟล์ Production จริงในวันพรุ่งนี้ เพียงแค่เปลี่ยนค่า `DSN=` ใน JCL เท่านั้น — เป็นหลักการ
เดียวกับ **Dependency Injection** ในโลกโปรแกรมมิ่งสมัยใหม่ เพียงแต่ COBOL/JCL ทำสิ่งนี้มาตั้งแต่
ทศวรรษ 1960 ก่อนคำว่า "Dependency Injection" จะถูกบัญญัติขึ้นเสียอีก

### ข้อควรระวัง

- **ไวยากรณ์ `ASSIGN TO ACCTFILE` (ไม่มีเครื่องหมายคำพูด ชี้ไป DDNAME) เป็นไวยากรณ์มาตรฐาน
  ของ z/OS COBOL ที่ไม่สามารถทดสอบรันจริงได้ในสภาพแวดล้อม GnuCOBOL ของหลักสูตรนี้** ทุกตัวอย่าง
  ที่ compile และรันได้จริงในเอกสารนี้ (ขั้นตอนที่ 542, 544, 546, 550) ใช้รูปแบบ
  `ASSIGN TO "filename.DAT"` ของ GnuCOBOL แทน ซึ่งเป็นวิธีมาตรฐานที่ GnuCOBOL ใช้เมื่อไม่มี JCL
  รองรับ
- อย่าลืมว่า `FILE STATUS IS` ควรใส่ไว้เสมอในโค้ด VSAM ระดับ Production เพื่อตรวจจับข้อผิดพลาด
  เฉพาะของ VSAM ที่ Part 030 ยังไม่ได้ครอบคลุม (จะสอนรหัสเพิ่มเติมในขั้นตอนที่ 549)

### แบบฝึกหัดที่ 547.1

**โจทย์**: จงอธิบายประโยชน์เชิงปฏิบัติของการที่โปรแกรม COBOL อ้างอิง DDNAME แทนชื่อไฟล์จริง เมื่อ
ต้องย้ายโปรแกรมจากสภาพแวดล้อมทดสอบ (Test) ไปยังสภาพแวดล้อม Production

**เฉลย**: ทีมงานสามารถย้ายโปรแกรม (load module) ตัวเดียวกันจาก Test ไป Production ได้โดยไม่ต้อง
compile ใหม่เลย เพียงเปลี่ยนค่า `DSN=` ใน JCL ให้ชี้ไปยัง VSAM Cluster ของ Production แทน Test
ซึ่งลดความเสี่ยงที่โปรแกรม Production จะมีโค้ดต่างจากที่ผ่านการทดสอบมาแล้ว (เพราะเป็น binary
เดียวกันเป๊ะ เปลี่ยนแค่การผูกไฟล์ภายนอกเท่านั้น)

---

## ขั้นตอนที่ 548: LISTCAT — ตรวจสอบรายละเอียดของ VSAM Cluster

### LISTCAT คืออะไร

**LISTCAT (List Catalog)** คือคำสั่งของ IDCAMS ที่ใช้แสดงรายละเอียดของ VSAM Cluster ที่มีอยู่แล้ว
ในระบบ เปรียบได้กับคำสั่ง `ls -l` ผสมกับ `stat` ในโลก Linux ใช้บ่อยเมื่อต้องการตรวจสอบว่า
Cluster มีขนาดเท่าไร ใช้พื้นที่ไปเท่าไรแล้ว หรือ CI Split เกิดขึ้นบ่อยแค่ไหน

### ไวยากรณ์อ้างอิง: LISTCAT พื้นฐาน

```jcl
//LISTIT   JOB (ACCT123),'LISTCAT KSDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  LISTCAT ENTRIES(PROD.ACCOUNT.KSDS) ALL
/*
```

### ตัวอย่างผลลัพธ์ที่คาดว่าจะได้ (อ้างอิงรูปแบบมาตรฐาน — จำลองเพื่อประกอบความเข้าใจ)

```
CLUSTER ------- PROD.ACCOUNT.KSDS
      IN-CAT --- CATALOG.VSAM01
      HISTORY
        RELEASE----------------2
        OWNER-IDENT-----------(NULL)
      ATTRIBUTES
        KEYLEN----------------8         AVGLRECL---------37
        RKP--------------------0         MAXLRECL---------37
        INDEXED                          FREESPACE-%CI(10) %CA(10)
      STATISTICS
        REC-TOTAL-------------2450      SPLITS-CI-------------12
        REC-DELETED------------30       SPLITS-CA--------------3
        REC-INSERTED----------180
```

### อธิบายจุดสำคัญ

- **`SPLITS-CI`** และ **`SPLITS-CA`** ในผลลัพธ์คือตัวเลขสำคัญที่ System Programmer ใช้ประเมิน
  ประสิทธิภาพของ Cluster — ถ้าค่านี้สูงมากผิดปกติ แปลว่า `FREESPACE` ที่กำหนดไว้ตอน DEFINE
  CLUSTER น้อยเกินไป ควรพิจารณาทำ Reorganization (จะสอนใน Part 056 ขั้นตอนที่ 558)
- **`REC-TOTAL`**, **`REC-DELETED`**, **`REC-INSERTED`** ช่วยให้เห็นภาพว่า Cluster มีการ
  เปลี่ยนแปลงมากน้อยแค่ไหนตั้งแต่สร้างหรือ Reorganize ครั้งล่าสุด
- `KEYLEN` และ `RKP` (Relative Key Position) ยืนยันค่าที่ตั้งไว้ตอน DEFINE CLUSTER ตรงกับที่
  ระบุใน `KEYS(length offset)`

### ข้อควรระวัง

- **คำสั่ง LISTCAT และผลลัพธ์ตัวอย่างในขั้นตอนนี้เป็นการจำลองรูปแบบมาตรฐานเพื่อประกอบความเข้าใจ
  เท่านั้น ไม่สามารถรันได้จริงในสภาพแวดล้อมของหลักสูตรนี้** เนื่องจากไม่มี VSAM Catalog จริงให้
  ตรวจสอบ
- LISTCAT ต้องการสิทธิ์อ่าน Catalog ของ dataset นั้น ๆ ผ่าน RACF (Part 069) ผู้ใช้ที่ไม่มีสิทธิ์
  อาจได้รับข้อความ error แทนผลลัพธ์ปกติ

### แบบฝึกหัดที่ 548.1

**โจทย์**: หากพบว่าค่า `SPLITS-CI` ของ Cluster หนึ่งสูงผิดปกติ (เช่น หลักพันครั้งต่อเดือน)
ควรดำเนินการอย่างไรในเบื้องต้น

**เฉลย**: ควรพิจารณาเพิ่มค่า `FREESPACE` ตอน redefine Cluster ใหม่ (เพิ่มพื้นที่ว่างสำรองใน
แต่ละ CI/CA มากขึ้น) เพื่อลดความถี่ที่ VSAM ต้องแบ่ง CI ใหม่เมื่อมีการแทรก record จากนั้นทำ
REPRO (Part 056 ขั้นตอนที่ 557) เพื่อคัดลอกข้อมูลเดิมเข้า Cluster ใหม่ที่มี FREESPACE มากขึ้น
เป็นการ Reorganize ไฟล์ให้มีประสิทธิภาพกลับมาเหมือนตอนสร้างใหม่

---

## ขั้นตอนที่ 549: VSAM File Status Codes — ต่อยอดจาก Part 030

### ทบทวน File Status จาก Part 030

Part 030 สอนรหัส File Status พื้นฐานที่ใช้ได้กับทุกประเภทไฟล์ (`00` สำเร็จ, `10` End of File,
`23` ไม่พบ record, `35` ไฟล์ไม่มีอยู่จริงตอน OPEN ฯลฯ) รหัสเหล่านี้เป็นมาตรฐาน COBOL ที่ใช้ได้กับ
ทั้ง GnuCOBOL Indexed File และ VSAM บน z/OS เหมือนกันทุกประการ เพราะเป็นส่วนหนึ่งของมาตรฐานภาษา
COBOL เอง ไม่ใช่ส่วนเฉพาะของ implementation ใด

### รหัส File Status ที่พบบ่อยเป็นพิเศษเมื่อทำงานกับ VSAM

| File Status | ความหมาย | สาเหตุทั่วไปเมื่อเจอกับ VSAM |
|---|---|---|
| `22` | พยายามเขียน record ที่มี Key ซ้ำ (Duplicate Key) | เหมือนกับที่ Part 028 ขั้นตอน 278 พิสูจน์ไว้แล้วกับ GnuCOBOL |
| `23` | ไม่พบ record ที่ค้นหา | เหมือนกับ Part 030 |
| `24` | พยายามเขียนเกินพื้นที่ที่จองไว้ (Boundary Violation) | Cluster เต็มพื้นที่ที่ DEFINE ไว้ทั้ง Primary และ Secondary allocation |
| `30` | ข้อผิดพลาดระดับ Permanent I/O Error | ปัญหาฮาร์ดแวร์ดิสก์ หรือ VSAM Catalog เสียหาย |
| `37` | โหมดการเปิดไฟล์ไม่ตรงกับที่ DEFINE ไว้ (เช่น DEFINE เป็น REUSE แต่โปรแกรมเปิดผิดโหมด) | ความไม่สอดคล้องระหว่าง IDCAMS DEFINE กับ OPEN mode ในโปรแกรม |
| `39` | คุณลักษณะของไฟล์ไม่ตรงกับที่ FD ประกาศไว้ (Mismatch) | `RECORDSIZE`/`KEYS` ใน DEFINE CLUSTER ไม่ตรงกับ `01` record ใน FD — ตรงกับคำเตือนในขั้นตอนที่ 543 |
| `90`–`99` | ข้อผิดพลาดเฉพาะของ implementation (ไม่ใช่มาตรฐานสากล) | แต่ละ VSAM version/patch อาจนิยามต่างกันไป ต้องดูคู่มือเฉพาะระบบ |

### ตัวอย่างโค้ด: ตรวจสอบ File Status หลัง OPEN (ไวยากรณ์เดียวกับ Part 030 ที่ใช้ได้กับ VSAM ด้วย)

```cobol
           OPEN I-O ACCOUNT-FILE.
           IF WS-FILE-STATUS NOT = "00"
               DISPLAY "OPEN FAILED - STATUS: " WS-FILE-STATUS
               EVALUATE WS-FILE-STATUS
                   WHEN "35"
                       DISPLAY "CLUSTER DOES NOT EXIST - RUN IDCAMS DEFINE FIRST"
                   WHEN "39"
                       DISPLAY "RECORD LAYOUT MISMATCH - CHECK RECORDSIZE/KEYS"
                   WHEN OTHER
                       DISPLAY "UNEXPECTED VSAM ERROR"
               END-EVALUATE
               MOVE 16 TO RETURN-CODE
               STOP RUN
           END-IF.
```

> ส่วนของโค้ด COBOL ข้างต้นใช้ไวยากรณ์ `FILE STATUS`/`EVALUATE` มาตรฐานที่ **compile ได้จริงด้วย
> GnuCOBOL** (โครงสร้างเดียวกับที่ Part 030 พิสูจน์แล้ว) แต่ค่า File Status ที่จะปรากฏจริงในทาง
> ปฏิบัติ (`35`, `39` จากสาเหตุ VSAM เฉพาะ) ต้องทดสอบกับ VSAM จริงบน z/OS เท่านั้น เนื่องจาก
> GnuCOBOL ไม่มีแนวคิด IDCAMS/Catalog ให้จำลองสถานการณ์เหล่านี้ได้ครบถ้วน

### ข้อควรระวัง

- **รหัส `90`-`99` ไม่ใช่มาตรฐานสากล** ต่างจากรหัส `00`-`69` ที่เป็นมาตรฐาน COBOL แท้ ๆ ที่ใช้
  ความหมายเดียวกันทุก compiler/implementation ต้องตรวจสอบเอกสารเฉพาะของระบบเมื่อเจอรหัสในช่วงนี้
- เมื่อ Debug ปัญหา VSAM ควรตรวจสอบทั้ง File Status ที่โปรแกรม COBOL รายงาน **และ** ข้อความจาก
  ระบบ (System Messages เช่น `IEC161I`) ที่ปรากฏใน job log พร้อมกันเสมอ เพราะบางครั้ง File
  Status เพียงอย่างเดียวไม่เพียงพอต่อการวินิจฉัยสาเหตุที่แท้จริง

### แบบฝึกหัดที่ 549.1

**โจทย์**: หากโปรแกรม COBOL รายงาน File Status `39` เมื่อพยายาม `OPEN` ไฟล์ VSAM ควรตรวจสอบ
อะไรเป็นอันดับแรก

**เฉลย**: ควรตรวจสอบว่า `RECORDSIZE` และ `KEYS` ที่ระบุไว้ตอน IDCAMS DEFINE CLUSTER ตรงกับ
โครงสร้าง `01` record ใน FD ของโปรแกรม COBOL หรือไม่ (ทั้งความกว้างรวมของ record และตำแหน่ง/
ความกว้างของ Key field) เพราะ File Status `39` หมายถึงคุณลักษณะของไฟล์ที่ COBOL คาดหวังไม่ตรงกับ
ที่ VSAM Cluster ถูกกำหนดไว้จริง

---

## ขั้นตอนที่ 550: กรณีศึกษารวม — ออกแบบและใช้งาน VSAM KSDS สำหรับระบบลูกค้า

### โจทย์ของกรณีศึกษา

จำลองการบริหารจัดการ VSAM KSDS สำหรับเก็บข้อมูลลูกค้าธนาคาร ครอบคลุมวงจรชีวิตแบบเต็มรูปแบบ:
LOAD ข้อมูลเริ่มต้น → GET (อ่านตรง) → PUT UPDATE (แก้ไข) → ERASE (ลบ) → Full Cluster Scan
(อ่านทั้งไฟล์) โดยเขียนเป็นโปรแกรม COBOL เดียวที่รวมทุกการดำเนินการไว้ พร้อมคำอธิบายว่าแต่ละส่วน
เทียบเท่ากับปฏิบัติการ VSAM แบบใด

### ไวยากรณ์ IDCAMS ที่ควรใช้สร้าง Cluster นี้บน z/OS จริง (อ้างอิง — รันไม่ได้ในสภาพแวดล้อมนี้)

```jcl
//DEFCUST  JOB (ACCT123),'DEFINE CUSTOMER KSDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE CLUSTER (NAME(PROD.CUSTOMER.KSDS)         -
                  INDEXED                          -
                  KEYS(6 0)                        -
                  RECORDSIZE(35 35)                -
                  RECORDS(100000 50000)             -
                  FREESPACE(15 15)                  -
                  SHAREOPTIONS(2 3)                 -
                  VOLUMES(VSAM01) )                 -
         DATA  (NAME(PROD.CUSTOMER.KSDS.DATA))      -
         INDEX (NAME(PROD.CUSTOMER.KSDS.INDEX))
/*
```

### โปรแกรม COBOL: Working Analog ที่ Compile และรันได้จริง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP550-VSAM-CAPSTONE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTKSDS550.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(6).
           05  CUST-NAME            PIC X(20).
           05  CUST-BALANCE         PIC 9(9)V99.

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> This program is a full working analog of managing a VSAM
      *> KSDS cluster from COBOL: LOAD (initial build), then the
      *> same "GET/PUT/UPDATE/ERASE" operations VSAM exposes,
      *> spelled here in native COBOL as WRITE/READ/REWRITE/DELETE.
           PERFORM LOAD-INITIAL-DATA.
           PERFORM DIRECT-READ-DEMO.
           PERFORM UPDATE-DEMO.
           PERFORM DELETE-DEMO.
           PERFORM PRINT-FULL-CLUSTER.
           STOP RUN.

       LOAD-INITIAL-DATA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 100001 TO CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME.
           MOVE 15000.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.

           MOVE 100002 TO CUST-ID.
           MOVE "SUDA MEECHAI" TO CUST-NAME.
           MOVE 8250.50 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.

           MOVE 100003 TO CUST-ID.
           MOVE "PRASERT KAEWTA" TO CUST-NAME.
           MOVE 500.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.
           DISPLAY "== Cluster loaded with 3 records (LOAD) ==".

       DIRECT-READ-DEMO.
           OPEN I-O CUSTOMER-FILE.
           MOVE 100002 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "GET failed for 100002, status="
                       WS-FILE-STATUS
               NOT INVALID KEY
                   DISPLAY "== GET (direct read) 100002: "
                       CUST-NAME " " CUST-BALANCE " =="
           END-READ.

       UPDATE-DEMO.
           ADD 1000.00 TO CUST-BALANCE.
           REWRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY "PUT UPDATE failed, status="
                       WS-FILE-STATUS
           END-REWRITE.
           DISPLAY "== PUT UPDATE 100002: new balance="
               CUST-BALANCE " ==".

       DELETE-DEMO.
           MOVE 100003 TO CUST-ID.
           DELETE CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "ERASE failed for 100003, status="
                       WS-FILE-STATUS
               NOT INVALID KEY
                   DISPLAY "== ERASE (delete) 100003: done =="
           END-DELETE.
           CLOSE CUSTOMER-FILE.

       PRINT-FULL-CLUSTER.
           DISPLAY "== Full cluster scan after LOAD/PUT/ERASE ==".
           OPEN INPUT CUSTOMER-FILE.
           MOVE "N" TO WS-EOF-FLAG.
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " CUST-ID " " CUST-NAME " "
                           CUST-BALANCE
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
```

คอมไพล์และรัน:

```bash
cobc -x -o step550 step550.cob
./step550
```

**ผลลัพธ์จริง (ยืนยันแล้วด้วย GnuCOBOL build ที่มี BDB indexed handler):**

```
== Cluster loaded with 3 records (LOAD) ==
== GET (direct read) 100002: SUDA MEECHAI         000008250.50 ==
== PUT UPDATE 100002: new balance=000009250.50 ==
== ERASE (delete) 100003: done ==
== Full cluster scan after LOAD/PUT/ERASE ==
  100001 SOMCHAI JAIDEE       000015000.00
  100002 SUDA MEECHAI         000009250.50
```

### สรุปการจับคู่ของกรณีศึกษานี้

| ขั้นตอนในโปรแกรม | ปฏิบัติการ VSAM ที่เทียบเท่า | คำสั่ง COBOL ที่ใช้ |
|---|---|---|
| `LOAD-INITIAL-DATA` | LOAD (การบรรจุข้อมูลชุดแรกเข้า Cluster ที่เพิ่ง DEFINE) | `OPEN OUTPUT` + `WRITE` |
| `DIRECT-READ-DEMO` | GET (อ่านตรงผ่าน Key) | `READ` (พร้อม `INVALID KEY`) |
| `UPDATE-DEMO` | PUT UPDATE (แก้ไข record ที่มีอยู่) | `REWRITE` |
| `DELETE-DEMO` | ERASE (ลบ record) | `DELETE` |
| `PRINT-FULL-CLUSTER` | Sequential Browse (ไล่อ่านทั้ง Cluster ตามลำดับ Key) | `READ ... NEXT RECORD` |

สังเกตว่าผลลัพธ์สุดท้ายเหลือเพียง 2 record (`100001`, `100002`) เพราะ `100003` ถูก ERASE ไปแล้ว
และยอดคงเหลือของ `100002` เพิ่มขึ้นจาก PUT UPDATE — พฤติกรรมนี้ตรงกับสิ่งที่ VSAM KSDS จะทำจริง
บน z/OS ทุกประการ เพียงแต่รันผ่าน GnuCOBOL ที่เป็นแบบจำลองที่ใช้งานได้จริงบนเครื่องของเรา

### ข้อควรระวัง

- ในการใช้งานจริงบน z/OS ขั้นตอน `LOAD-INITIAL-DATA` มักไม่ทำผ่านโปรแกรม COBOL โดยตรง แต่ใช้
  **IDCAMS REPRO** (จะสอนใน Part 056 ขั้นตอนที่ 557) คัดลอกข้อมูลจากไฟล์ Sequential เข้า VSAM
  Cluster ที่เพิ่งสร้างแทน เพราะจัดการข้อมูลจำนวนมากได้มีประสิทธิภาพกว่า
- ทุกครั้งที่ทำ `DELETE` ตามด้วย `WRITE` Key เดิมใหม่ ต้องระวังว่า record ที่ถูกลบไปแล้วจะไม่
  สามารถกู้คืนได้ (ไม่มี soft-delete ในตัว VSAM เอง) หากต้องการ history ต้องออกแบบระบบ Audit
  Trail แยกต่างหาก (เช่นผ่าน ESDS ตามที่ขั้นตอนที่ 544 แนะนำ)

### แบบฝึกหัดที่ 550.1

**โจทย์**: จงเพิ่ม paragraph ใหม่ชื่อ `ADD-NEW-CUSTOMER` ที่เพิ่มลูกค้าใหม่ (`CUST-ID` 100004)
เข้าไปในไฟล์หลังจากขั้นตอน `DELETE-DEMO` แต่ก่อน `PRINT-FULL-CLUSTER`

**เฉลย**:

```cobol
       ADD-NEW-CUSTOMER.
           OPEN I-O CUSTOMER-FILE.
           MOVE 100004 TO CUST-ID.
           MOVE "WANNA SRISUK" TO CUST-NAME.
           MOVE 3000.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY "Insert of 100004 failed."
           END-WRITE.
           CLOSE CUSTOMER-FILE.
```

แล้วเพิ่ม `PERFORM ADD-NEW-CUSTOMER.` ต่อจาก `PERFORM DELETE-DEMO.` ใน `MAIN-PARA` ผลลัพธ์การ
สแกนไฟล์สุดท้ายจะแสดงลูกค้า 3 คน (`100001`, `100002`, `100004`) เพราะ `100003` ถูกลบไปแล้วและ
`100004` เพิ่งถูกเพิ่มใหม่หลังจากนั้น

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้ VSAM ทั้ง 3 ประเภทหลักบน Mainframe และเชื่อมโยงกับความรู้เดิมของเราจาก
Part 023-029 อย่างชัดเจน:

- ภาพรวมของ VSAM: Cluster, Data Component, Index Component, Control Interval/Area
- **KSDS** = `ORGANIZATION IS INDEXED` (Part 028) — เก็บพร้อมดัชนีตาม Key ที่ไม่ซ้ำกัน
- **ESDS** = `ORGANIZATION IS SEQUENTIAL` — เก็บตามลำดับเขียนจริง append-only ห้าม DELETE
- **RRDS** = `ORGANIZATION IS RELATIVE` (Part 029) — เก็บในตำแหน่ง slot ที่กำหนดด้วยหมายเลข
- ไวยากรณ์ IDCAMS `DEFINE CLUSTER` สำหรับสร้าง VSAM ทั้ง 3 ประเภท (`INDEXED`/`NONINDEXED`/
  `NUMBERED`)
- ความแตกต่างของ `SELECT ... ASSIGN TO` ระหว่าง GnuCOBOL (ชื่อไฟล์จริง) กับ z/OS COBOL
  (DDNAME ที่ผูกผ่าน JCL)
- `LISTCAT` สำหรับตรวจสอบสถิติและรายละเอียดของ Cluster
- VSAM File Status Codes ที่ต่อยอดจาก Part 030 (`22`, `24`, `37`, `39`)
- กรณีศึกษารวม: ระบบลูกค้าธนาคารแบบ KSDS ที่ **compile และรันได้จริงด้วย GnuCOBOL** ครอบคลุม
  LOAD/GET/PUT UPDATE/ERASE/Sequential Browse ครบวงจร

**ย้ำอีกครั้ง**: ไวยากรณ์ IDCAMS และ LISTCAT ทั้งหมดในนี้เป็นไวยากรณ์มาตรฐานอ้างอิงที่ถูกต้อง
ต้องใช้ z/OS จริงหรือ Mainframe Emulator เพื่อรันจริง ส่วนโค้ด COBOL ทุกตัวอย่างที่ระบุว่า
"compile และรันได้จริง" ได้ทดสอบแล้วด้วย GnuCOBOL จริงตามที่ระบุไว้ในแต่ละขั้นตอน

Part ถัดไป (**Part 056**) จะขยายความรู้ VSAM ของเราให้ลึกขึ้นอีกขั้น: **Alternate Index (AIX),
PATH, REPRO Utility และการ Reorganize Cluster** พร้อมทั้งเชื่อมโยงกับ `ALTERNATE RECORD KEY`
ที่ Part 028 ได้สอนไว้แล้วอย่างละเอียด

**[← กลับไป Part 054: TSO/ISPF](part-054-tso-ispf.md)** | **[ไปยัง Part 056: VSAM ขั้นสูง: Alternate Index และ Cluster →](part-056-vsam-advanced.md)**
