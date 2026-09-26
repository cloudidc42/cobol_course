# Part 028: Indexed Files และ ISAM (ขั้นตอนที่ 271–280)

## คำนำของ Part นี้

ตลอด Part 023–027 เราเรียนรู้ **ไฟล์ตามลำดับ (Sequential File)** อย่างละเอียด ตั้งแต่การเขียน-อ่าน
พื้นฐาน ไปจนถึงอัลกอริทึม Master-Detail และการ `SORT`/`MERGE` ข้อมูล แต่ตลอดทางนั้นเราเจอข้อจำกัด
ใหญ่อยู่เสมอ: **ต้องอ่านหรือเขียนตามลำดับเท่านั้น** อยากอ่าน record ที่ 1,000 ต้องอ่านผ่าน record ที่
1–999 ก่อนเสมอ และที่แย่กว่านั้นคือ Part 025 ขั้นตอนที่ 247 พิสูจน์ให้เห็นแล้วว่า `DELETE` **ใช้กับ
ไฟล์ตามลำดับไม่ได้เลย** แม้แต่ในทางทฤษฎี เพราะข้อมูลถูกเก็บเป็นสายต่อเนื่อง (byte stream) ไม่มี
"ช่องว่างระหว่างกลาง" ให้ลบได้โดยไม่กระทบ record อื่น

Part นี้จะพาคุณก้าวข้ามข้อจำกัดเหล่านั้นทั้งหมดด้วยการแนะนำไฟล์ประเภทที่สองจากทั้งหมด 3 ประเภทของ
COBOL (ทบทวนตารางจาก Part 023 ขั้นตอน 222): **Indexed File** ซึ่งใช้เทคนิคที่เรียกว่า **ISAM**
(Indexed Sequential Access Method) ทำให้เราสามารถ **ค้นหา แก้ไข และลบ record ใดก็ได้โดยตรงผ่าน
Key โดยไม่ต้องอ่านไฟล์ทั้งหมดตั้งแต่ต้น** — นี่คือรากฐานสำคัญที่ทำให้เกิดฐานข้อมูลสมัยใหม่ในเวลาต่อมา
และยังคงเป็นรูปแบบไฟล์มาตรฐานบน Mainframe (ในชื่อ **VSAM KSDS** ที่จะสอนเต็มรูปแบบใน Part 055)

> **สำคัญมากสำหรับผู้ที่ฝึกตามด้วยตนเอง**: GnuCOBOL ที่ติดตั้งมาตรฐานจากตัวจัดการแพ็กเกจของบาง
> ระบบปฏิบัติการ (เช่น Ubuntu/Debian) **อาจปิดการรองรับ `ORGANIZATION IS INDEXED` ไว้โดยค่าเริ่มต้น**
> ด้วยเหตุผลด้านสัญญาอนุญาต (license) ของไลบรารี ISAM ตรวจสอบได้ด้วยคำสั่ง `cobc -info` แล้วดูแถว
> `indexed file handler` — หากขึ้นว่า `disabled` แปลว่าต้อง compile GnuCOBOL ใหม่เองพร้อมเปิดใช้
> `--with-db` (Berkeley DB) หรือไลบรารี ISAM ตัวอื่น (VBISAM, D-ISAM, C-ISAM, LMDB) เสียก่อน
> โค้ดทุกตัวอย่างใน Part นี้ทดสอบจริงด้วย GnuCOBOL ที่ build พร้อม **Berkeley DB (BDB) 5.3** เป็น
> ISAM handler (`indexed file handler : BDB version 5.3.28`) และให้ผลลัพธ์ตรงตามที่แสดงไว้ทุกประการ

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้ (คอมเมนต์, ชื่อตัวแปร, ข้อความใน
> DISPLAY) เป็นภาษาอังกฤษล้วน เพราะขีดจำกัดคอลัมน์ 72 ของ Fixed-Format COBOL นับเป็นไบต์ไม่ใช่
> ตัวอักษร โค้ดทุกตัวอย่างในนี้ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 271: ทำไมต้องมี Indexed File — แนวคิด ISAM และ RECORD KEY

### ข้อจำกัดที่ Sequential File แก้ไม่ได้

ลองนึกภาพระบบธนาคารที่มีลูกค้า 1 ล้านคนเก็บอยู่ในไฟล์ตามลำดับ ถ้าต้องการดูยอดเงินของลูกค้าคนที่
999,999 โปรแกรมต้อง `READ` ไล่ทีละ record ตั้งแต่คนแรกไปจนถึงคนที่ต้องการ — เฉลี่ยแล้วต้องอ่านครึ่งไฟล์
(500,000 record) ทุกครั้งที่ค้นหา 1 ครั้ง นี่คือปัญหาประสิทธิภาพร้ายแรงที่ไฟล์ตามลำดับไม่มีทางแก้ได้เอง

### ISAM คืออะไร

**ISAM (Indexed Sequential Access Method)** คือเทคนิคที่ COBOL ยืมแนวคิดมาจากดัชนีท้ายเล่มหนังสือ:
แทนที่จะเก็บแค่ข้อมูลดิบเรียงต่อกัน ระบบจะสร้าง **โครงสร้างดัชนี (Index Structure)** แยกต่างหากที่จับคู่
**ค่า Key** กับ **ตำแหน่งจริงของ record นั้นบนดิสก์** เมื่อค้นหา ระบบจะค้นในดัชนี (ซึ่งมักจัดเก็บเป็น
B-Tree ทำให้ค้นเร็วมากแม้มีข้อมูลนับล้าน record) แล้ว "กระโดด" ไปอ่าน record นั้นโดยตรง โดยไม่ต้อง
ไล่อ่านทีละ record เลย GnuCOBOL ใช้ไลบรารีภายนอก (เช่น Berkeley DB) เพื่อจัดการโครงสร้างดัชนีนี้ให้
โดยอัตโนมัติ — ตัวโปรแกรมเมอร์ COBOL ไม่ต้องเขียนอัลกอริทึม B-Tree เองแม้แต่บรรทัดเดียว

### ประกาศไฟล์ Indexed ด้วย ORGANIZATION IS INDEXED และ RECORD KEY

การประกาศไฟล์ Indexed ต่างจาก Sequential เพียงจุดเดียวใน `SELECT`: ต้องระบุ `RECORD KEY IS` ว่า
field ใดใน record จะถูกใช้เป็น Key หลัก (Primary Key) สำหรับการค้นหา

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP271-INDEXED-INTRO.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST271.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS CUST-ID.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).
           05  CUST-BALANCE         PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.

           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME.
           MOVE 500.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.

           MOVE 10002 TO CUST-ID.
           MOVE "SUDA MEECHAI" TO CUST-NAME.
           MOVE 2750.50 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.

           MOVE 10003 TO CUST-ID.
           MOVE "PRASERT KAEWTA" TO CUST-NAME.
           MOVE 1500.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.

           CLOSE CUSTOMER-FILE.
           DISPLAY "Wrote 3 records keyed by CUST-ID.".

           OPEN INPUT CUSTOMER-FILE.
           PERFORM 3 TIMES
               READ CUSTOMER-FILE NEXT RECORD
               DISPLAY CUST-ID " " CUST-NAME " " CUST-BALANCE
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

คอมไพล์และรัน:

```bash
cobc -x -o step271 step271.cob
./step271
```

**ผลลัพธ์:**

```
Wrote 3 records keyed by CUST-ID.
10001 SOMCHAI JAIDEE       0000500.00
10002 SUDA MEECHAI         0002750.50
10003 PRASERT KAEWTA       0001500.00
```

### อธิบายทีละส่วน

- `ORGANIZATION IS INDEXED`: บอก COBOL ว่านี่คือไฟล์ Indexed ไม่ใช่ Sequential ธรรมดา
- `RECORD KEY IS CUST-ID`: กำหนดว่า field `CUST-ID` (ต้องเป็น field ที่มีอยู่จริงภายใน
  `CUSTOMER-RECORD`) คือ Key หลักที่ใช้ระบุตัวตนของแต่ละ record และต้อง**ไม่ซ้ำกัน**ในไฟล์เดียวกัน
- `ACCESS MODE IS SEQUENTIAL`: โหมดการเข้าถึงนี้ (ค่าเริ่มต้นถ้าไม่ระบุ) ทำให้ `READ` อ่านเรียง
  ตามลำดับ **Key** (ไม่ใช่ตามลำดับที่เขียนจริง!) — จะเจาะลึกโหมดอื่น (`RANDOM`, `DYNAMIC`) ใน
  ขั้นตอนที่ 273–275
- `READ CUSTOMER-FILE NEXT RECORD`: สำหรับไฟล์ Indexed ต้องใช้คำว่า `NEXT RECORD` ต่อท้าย `READ`
  เมื่ออ่านแบบเรียงลำดับ (ต่างจากไฟล์ Sequential ที่ใช้ `READ` เฉย ๆ)

### ข้อควรระวัง

- **`WRITE` บนไฟล์ Indexed ที่เปิดด้วย `OPEN OUTPUT` และ `ACCESS MODE IS SEQUENTIAL` ต้องเขียน
  Key เรียงจากน้อยไปมากเสมอ** ทดสอบจริงแล้วว่าถ้าเขียนสลับลำดับ (เช่น 10003 ก่อน 10001) จะเกิด
  runtime error ทันที:
  ```
  libcob: error: key order not ascending (status = 21) for file CUSTOMER-FILE ('CUST271.DAT')
  ```
  ข้อจำกัดนี้จะหมดไปเมื่อใช้ `ACCESS MODE IS DYNAMIC` ร่วมกับ `OPEN I-O` (ขั้นตอนที่ 275)
- อย่าสับสน `RECORD KEY` (ชื่อ field ใน record) กับชื่อไฟล์ (`CUSTOMER-FILE`) — `RECORD KEY IS`
  ต้องระบุชื่อ field ที่ประกาศไว้ใน `01 CUSTOMER-RECORD` เท่านั้น

### แบบฝึกหัดที่ 271.1

**โจทย์**: จงอธิบายว่าทำไมการค้นหาลูกค้า 1 คนจากไฟล์ 1 ล้าน record ด้วย Indexed File จึงเร็วกว่า
ไฟล์ Sequential มาก แม้ทั้งสองแบบจะเก็บข้อมูลจำนวนเท่ากันบนดิสก์เหมือนกัน

**เฉลย**: เพราะไฟล์ Sequential ไม่มีโครงสร้างช่วยค้นหาใด ๆ เลย โปรแกรมต้อง `READ` ไล่ทีละ record
ตั้งแต่ต้นจนกว่าจะเจอ (Linear Search) ซึ่งในกรณีเลวร้ายที่สุดต้องอ่านทั้งล้าน record ในขณะที่ไฟล์
Indexed มีโครงสร้างดัชนีแบบ B-Tree แยกต่างหาก ที่จับคู่ Key กับตำแหน่งจริงบนดิสก์ การค้นหาจึงใช้
การเปรียบเทียบเพียงไม่กี่ครั้ง (ระดับ log ของจำนวน record) แล้วกระโดดไปอ่าน record นั้นโดยตรง
ไม่ต้องอ่าน record อื่นเลยแม้แต่ record เดียว

---

## ขั้นตอนที่ 272: การออกแบบ Key ให้ถูกต้อง — กับดักเรื่องลำดับการเปรียบเทียบ

### Key ถูกเปรียบเทียบแบบไบต์ต่อไบต์เสมอ (Byte Comparison)

จุดที่ผู้เริ่มต้นพลาดบ่อยที่สุดเมื่อออกแบบ `RECORD KEY` คือการลืมไปว่า **ไม่ว่า PICTURE ของ Key
จะเป็นตัวเลข (`9`) หรือตัวอักษร (`X`) ก็ตาม การเปรียบเทียบลำดับของ Key จะเทียบกันแบบ**ไบต์ต่อไบต์
ตามรหัส ASCII เสมอ**เหมือนเปรียบเทียบข้อความ ไม่ใช่เปรียบเทียบค่าตัวเลขทางคณิตศาสตร์**

ถ้า Key เป็น `PIC 9(5)` (ตัวเลขแท้) COBOL จะเติมเลข 0 ข้างหน้าให้เต็มความกว้างเสมอ (เช่น `10001`,
`10002`) ทำให้การเทียบไบต์ตรงกับการเทียบค่าตัวเลขพอดี **แต่ถ้า Key เป็น `PIC X` (ตัวอักษร) และค่าที่
เก็บไม่ได้เติมศูนย์นำหน้าให้ครบ** การเรียงลำดับจะผิดเพี้ยนไปจากที่คาดหวังทันที

### ทดลอง: กับดักจริงเมื่อ Key เป็น PIC X โดยไม่เติมศูนย์

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP272-KEY-SORT-ORDER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT BAD-FILE ASSIGN TO "BADKEY272.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS BAD-ID.

       DATA DIVISION.
       FILE SECTION.
       FD  BAD-FILE.
       01  BAD-RECORD.
           05  BAD-ID               PIC X(5).
           05  BAD-NAME             PIC X(15).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> BAD-ID is alphanumeric with NO leading zeros. As TEXT,
      *> "10" sorts BEFORE "2" because byte '1' < byte '2'.
      *> We must write in the file's own byte-sort order, so we
      *> write "10" before "2" here - already a warning sign.
           OPEN OUTPUT BAD-FILE.
           MOVE "1" TO BAD-ID.
           MOVE "FIRST CUSTOMER" TO BAD-NAME.
           WRITE BAD-RECORD.

           MOVE "10" TO BAD-ID.
           MOVE "TENTH CUSTOMER" TO BAD-NAME.
           WRITE BAD-RECORD.

           MOVE "2" TO BAD-ID.
           MOVE "SECOND CUSTOMER" TO BAD-NAME.
           WRITE BAD-RECORD.
           CLOSE BAD-FILE.

           DISPLAY "Reading back in KEY order (text compare):".
           OPEN INPUT BAD-FILE.
           PERFORM 3 TIMES
               READ BAD-FILE NEXT RECORD
               DISPLAY "  [" BAD-ID "] " BAD-NAME
           END-PERFORM.
           CLOSE BAD-FILE.
           STOP RUN.
```

**ผลลัพธ์จริง (ยืนยันแล้วด้วย GnuCOBOL):**

```
Reading back in KEY order (text compare):
  [1    ] FIRST CUSTOMER
  [10   ] TENTH CUSTOMER
  [2    ] SECOND CUSTOMER
```

### วิเคราะห์ปัญหา

สังเกตว่าลำดับที่ได้คือ `1`, `10`, `2` — **ไม่ใช่ลำดับตัวเลข 1, 2, 10 ตามสามัญสำนึก** เพราะ COBOL
เติมช่องว่าง (ไม่ใช่ศูนย์) ท้าย `PIC X(5)` ให้เต็มความกว้างเสมอ (`"1    "`, `"10   "`, `"2    "`)
แล้วเปรียบเทียบทีละไบต์จากซ้ายไปขวา: ไบต์แรกของ `"1    "` และ `"10   "` เป็น `'1'` เหมือนกัน แต่ไบต์
แรกของ `"2    "` คือ `'2'` ซึ่งมีค่า ASCII มากกว่า `'1'` ดังนั้น Key ที่ขึ้นต้นด้วย `1` (ทั้ง `"1"`
และ `"10"`) จะมาก่อน Key ที่ขึ้นต้นด้วย `2` เสมอ ไม่ว่าค่าตัวเลขจริงจะเป็นเท่าไรก็ตาม

### กฎที่ต้องจำ

> **หาก Key ต้องการให้เรียงตามค่าตัวเลข ให้ประกาศเป็น `PIC 9` (ตัวเลขแท้) เสมอ** เพราะ COBOL
> รับประกันว่าค่าตัวเลขจะถูกเติมเลข 0 นำหน้าให้เต็มความกว้างอัตโนมัติ (ตามที่เห็นในขั้นตอนที่ 271 ที่
> `10001 < 10002 < 10003` เรียงถูกต้องสมบูรณ์) ถ้าจำเป็นต้องใช้ Key แบบตัวอักษรผสมตัวเลข (เช่น
> รหัสสินค้า `"A001"`) ต้อง**เติมศูนย์นำหน้าด้วยตนเองในทุกค่าให้ความยาวเท่ากันเสมอ** (เช่น `"A001"`,
> `"A010"`, `"A100"` ไม่ใช่ `"A1"`, `"A10"`, `"A100"`)

### ข้อควรระวัง

- ปัญหานี้จะยิ่งซ่อนเร้นมากขึ้นเมื่อข้อมูลมีจำนวนน้อยตอนทดสอบ (เช่น Key แค่ 1 หลักทั้งหมด) แล้วมา
  แสดงอาการตอนข้อมูลจริงมีทั้ง 1, 2, และ 3 หลักปะปนกัน จึงควรออกแบบความกว้างของ Key ให้รองรับ
  จำนวนหลักสูงสุดที่เป็นไปได้ตั้งแต่วันแรก
- การแก้ไขปัญหานี้ภายหลัง (เปลี่ยนจาก `PIC X` เป็น `PIC 9` หรือเติมศูนย์) หมายถึงต้อง**สร้างไฟล์ใหม่
  ทั้งไฟล์**เสมอ เพราะโครงสร้างดัชนีผูกติดกับความกว้างและชนิดของ Key เดิมไปแล้ว

### แบบฝึกหัดที่ 272.1

**โจทย์**: ถ้าต้องเก็บรหัสสินค้าที่เป็นตัวอักษรผสมตัวเลข เช่น `"A1"`, `"A25"`, `"B3"` และต้องการให้
เรียงลำดับตามตัวเลขถูกต้องภายในแต่ละหมวดตัวอักษร ควรออกแบบ `PICTURE` ของ Key อย่างไร

**เฉลย**: ควรแยก Key เป็นสอง field ย่อยรวมกันเป็น group แล้วเติมศูนย์นำหน้าตัวเลขให้ความกว้างคงที่
เช่น `05 KEY-PREFIX PIC X(1)` ตามด้วย `05 KEY-NUMBER PIC 9(3)` แล้วเก็บค่าเป็น `"A001"`, `"A025"`,
`"B003"` (เติมศูนย์เองก่อน `MOVE`) วิธีนี้ทำให้การเปรียบเทียบไบต์ตรงกับลำดับที่ต้องการทุกกรณี ไม่ว่า
จะมีตัวเลขกี่หลักก็ตาม ตราบใดที่ไม่เกินความกว้างสูงสุดที่กำหนดไว้ (`9(3)` รองรับได้ถึง 999)

---

## ขั้นตอนที่ 273: การอ่านแบบสุ่มด้วย Key — ACCESS MODE IS RANDOM และ INVALID KEY

### จุดเด่นที่แท้จริงของ Indexed File

ขั้นตอนที่ 271–272 ยังใช้การอ่านแบบเรียงลำดับ (`READ ... NEXT RECORD`) ซึ่งยังไม่ได้ใช้ประโยชน์
จากดัชนีเต็มที่ ขั้นตอนนี้จะแนะนำ **`ACCESS MODE IS RANDOM`** ซึ่งเปิดให้เรา **กระโดดไปอ่าน record
ใดก็ได้โดยตรงผ่านค่า Key** โดยไม่ต้องอ่าน record ก่อนหน้าเลยแม้แต่ record เดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP273-RANDOM-READ.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST273.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS CUST-ID.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).
           05  CUST-BALANCE         PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Build the file first. OPEN OUTPUT + WRITE works the same
      *> way no matter what ACCESS MODE the SELECT clause names -
      *> ACCESS MODE only changes how READ/START behave later.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME.
           MOVE 500.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.
           MOVE 10002 TO CUST-ID.
           MOVE "SUDA MEECHAI" TO CUST-NAME.
           MOVE 2750.50 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.
           MOVE 10003 TO CUST-ID.
           MOVE "PRASERT KAEWTA" TO CUST-NAME.
           MOVE 1500.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

      *> Now open RANDOM: we jump DIRECTLY to any key we want,
      *> with no need to read the records before it.
           OPEN INPUT CUSTOMER-FILE.

           MOVE 10002 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "Customer 10002 not found."
               NOT INVALID KEY
                   DISPLAY "Found: " CUST-NAME " balance="
                       CUST-BALANCE
           END-READ.

           MOVE 99999 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "Customer 99999 not found."
               NOT INVALID KEY
                   DISPLAY "Found: " CUST-NAME
           END-READ.

           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Found: SUDA MEECHAI         balance=0002750.50
Customer 99999 not found.
```

### อธิบายจุดสำคัญ

- ขั้นตอนการอ่านแบบสุ่มคือ: (1) `MOVE` ค่า Key ที่ต้องการค้นหาเข้า field ของ `RECORD KEY` ก่อน
  (ในที่นี้คือ `CUST-ID`) (2) เรียก `READ` เฉย ๆ (ไม่มี `NEXT RECORD`) COBOL จะใช้ค่าใน `CUST-ID`
  ขณะนั้นเป็นตัวค้นหาโดยอัตโนมัติ
- `INVALID KEY` / `NOT INVALID KEY` คือคู่สาขาสำหรับไฟล์ Indexed/Relative เทียบเท่ากับ
  `AT END`/`NOT AT END` ของไฟล์ Sequential: **`INVALID KEY` ทำงานเมื่อไม่พบ Key ที่ค้นหา**
  (ในตัวอย่างนี้คือ `99999` ที่ไม่มีอยู่จริง) ส่วน **`NOT INVALID KEY` ทำงานเมื่อค้นพบและอ่านสำเร็จ**
- สังเกตว่าเรา `READ` หา `10002` ได้ทันทีโดยไม่ต้องอ่าน `10001` ก่อนเลย นี่คือประโยชน์ที่แท้จริงของ
  ISAM

### ข้อควรระวัง

- ในโหมด `ACCESS MODE IS RANDOM` **ห้ามใช้ `READ ... NEXT RECORD`** เพราะเป็นคนละแนวคิดกัน
  (โหมดนี้ออกแบบมาสำหรับการค้นหาแบบสุ่มล้วน ๆ) หากต้องการอ่านทั้งแบบสุ่มและแบบเรียงลำดับสลับกันใน
  โปรแกรมเดียว ต้องใช้ `ACCESS MODE IS DYNAMIC` แทน (ขั้นตอนที่ 275)
- ต้อง `MOVE` ค่า Key ที่ต้องการค้นหาเข้าไปใน field ของ `RECORD KEY` **ก่อน** เรียก `READ` เสมอ
  ลืมขั้นตอนนี้จะทำให้ค้นหาด้วยค่า Key เก่าที่ค้างอยู่จากการอ่านครั้งก่อนหน้าโดยไม่ได้ตั้งใจ

### แบบฝึกหัดที่ 273.1

**โจทย์**: จงแก้ไขโปรแกรมข้างต้นให้ค้นหาลูกค้า `10001` เพิ่มอีกหนึ่งรายการ ต่อจากการค้นหา `10002`

**เฉลย**: เพิ่ม `MOVE`/`READ` อีกชุดก่อน `CLOSE CUSTOMER-FILE.`:

```cobol
           MOVE 10001 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "Customer 10001 not found."
               NOT INVALID KEY
                   DISPLAY "Found: " CUST-NAME
           END-READ.
```

ไม่ว่าจะค้นหา Key ใดก่อนหลัง ผลลัพธ์จะถูกต้องเสมอเพราะแต่ละครั้งเป็นการค้นหาใหม่ที่เป็นอิสระต่อกัน
โดยสิ้นเชิง ต่างจากไฟล์ Sequential ที่ตำแหน่งการอ่านจะเลื่อนไปเรื่อย ๆ ตามลำดับเท่านั้น

---

## ขั้นตอนที่ 274: START Statement — กำหนดจุดเริ่มต้นการอ่านแบบเรียงลำดับ

### แนวคิดของ START

`START` คือคำสั่งที่ใช้ **"เลื่อนตำแหน่งตัวชี้ไฟล์" ไปยังตำแหน่งที่ตรงกับเงื่อนไข Key ที่กำหนด โดย
ยังไม่อ่าน record ใด ๆ เข้ามา** หลังจาก `START` สำเร็จ คำสั่ง `READ ... NEXT RECORD` ตัวถัดไปจะเริ่ม
อ่านจากตำแหน่งนั้นเป็นต้นไป เปรียบเหมือนการเปิดหนังสือแล้วใช้ดัชนีเปิดไปที่หน้าที่ต้องการ (ไม่ใช่เปิด
อ่านจากหน้าแรกทุกครั้ง) `START` มีประโยชน์มากเมื่อต้องการ**อ่านเฉพาะบางช่วง**ของไฟล์ ไม่ใช่ทั้งหมด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP274-START-STATEMENT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST274.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.  MOVE "SOMCHAI"  TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           MOVE 10002 TO CUST-ID.  MOVE "SUDA"     TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           MOVE 10003 TO CUST-ID.  MOVE "PRASERT"  TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           MOVE 10004 TO CUST-ID.  MOVE "WANNA"    TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           MOVE 10005 TO CUST-ID.  MOVE "ANAN"     TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

      *> ACCESS MODE DYNAMIC allows both READ (random) and
      *> READ NEXT (sequential) - START positions the file cursor
      *> WITHOUT reading a record itself.
           OPEN I-O CUSTOMER-FILE.

           DISPLAY "---- START at CUST-ID >= 10003 ----".
           MOVE 10003 TO CUST-ID.
           START CUSTOMER-FILE KEY IS >= CUST-ID
               INVALID KEY
                   DISPLAY "START failed - no such key range."
           END-START.

           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " CUST-ID " " CUST-NAME
               END-READ
           END-PERFORM.

           DISPLAY "---- START at CUST-ID > 10004 ----".
           MOVE "N" TO WS-EOF-FLAG.
           MOVE 10004 TO CUST-ID.
           START CUSTOMER-FILE KEY IS > CUST-ID
               INVALID KEY
                   DISPLAY "START failed - no such key range."
           END-START.
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " CUST-ID " " CUST-NAME
               END-READ
           END-PERFORM.

           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
---- START at CUST-ID >= 10003 ----
  10003 PRASERT
  10004 WANNA
  10005 ANAN
---- START at CUST-ID > 10004 ----
  10005 ANAN
```

### อธิบายจุดสำคัญ

- `START CUSTOMER-FILE KEY IS >= CUST-ID`: เลื่อนตำแหน่งไปยัง record **แรก**ที่มีค่า Key
  **มากกว่าหรือเท่ากับ** ค่าที่อยู่ใน `CUST-ID` ขณะนั้น เงื่อนไขที่ใช้ได้มีทั้ง `=`, `>`, `>=`,
  `<`, `<=` (`NOT <` ก็เขียนแทน `>=` ได้เช่นกันตามมาตรฐาน COBOL)
- `START` **ไม่ได้อ่าน record เข้ามาในหน่วยความจำ** เป็นเพียงการ "เล็ง" ตำแหน่งเท่านั้น ต้องตามด้วย
  `READ ... NEXT RECORD` เสมอเพื่ออ่านข้อมูลจริง
- `INVALID KEY` ของ `START` ทำงานเมื่อไม่มี record ใดตรงกับเงื่อนไขเลย (เช่น `START ... KEY IS >`
  ด้วยค่าที่มากกว่า Key สูงสุดในไฟล์)

### ข้อควรระวัง

- `START` ใช้ได้เฉพาะกับไฟล์ **Indexed** และ **Relative** เท่านั้น (Part 029) เพราะต้องอาศัย
  โครงสร้างดัชนีหรือหมายเลขลำดับที่คำนวณตำแหน่งได้ ไฟล์ Sequential ธรรมดาไม่มี `START` ให้ใช้
- อย่าลืมรีเซ็ต `WS-EOF-FLAG` กลับเป็น `"N"` ทุกครั้งก่อน `START`/`PERFORM UNTIL` รอบใหม่ เหมือนที่
  ทำในตัวอย่างนี้ก่อนรอบที่สอง มิฉะนั้นลูปจะไม่ทำงานเลยเพราะธงยังค้างเป็น `"Y"` จากรอบก่อน

### แบบฝึกหัดที่ 274.1

**โจทย์**: จงเขียน `START` ที่ทำให้อ่านได้เฉพาะลูกค้าที่มี `CUST-ID` **น้อยกว่า** `10003` เท่านั้น

**เฉลย**: ใช้เงื่อนไข `<` แล้วเริ่มอ่านจากต้นไฟล์ (Key ต่ำสุด) เป็นค่าตั้งต้น:

```cobol
           MOVE 10003 TO CUST-ID.
           START CUSTOMER-FILE KEY IS < CUST-ID
               INVALID KEY
                   DISPLAY "No customer below 10003."
           END-START.
```

ผลลัพธ์จะได้แค่ `10001 SOMCHAI` และ `10002 SUDA` เท่านั้น เพราะ `START ... KEY IS <` เลื่อนไปยัง
record แรกที่ Key น้อยกว่า `10003` (ซึ่งในกรณีนี้คือ `10002` เนื่องจากไฟล์เรียงจากน้อยไปมาก
`START` จะหาตัวที่ **มากที่สุด**ในกลุ่มที่ตรงเงื่อนไข `<` แล้ววาง cursor ไว้ที่นั่น จากนั้น
`READ NEXT` จะไล่ถอยไม่ได้ ต้องอ่านต่อไปข้างหน้าตามปกติ ผลคือได้ `10002` แล้วตามด้วย... ไม่มีอะไร
ต่อแล้วเพราะ 10003 ไม่ผ่านเงื่อนไขเริ่มต้น) — ในทางปฏิบัติหากต้องการ "10001 ถึง 10002 ทั้งคู่" ให้
เริ่มอ่านจากต้นไฟล์แล้วใช้ `IF` ตรวจสอบเงื่อนไขหยุดในลูปแทนจะปลอดภัยกว่า

---

## ขั้นตอนที่ 275: ACCESS MODE IS DYNAMIC และการเขียนแทรกแบบไม่เรียงลำดับ

### ปัญหาที่ทิ้งค้างจากขั้นตอนที่ 271

ในขั้นตอนที่ 271 เราเจอข้อจำกัดว่า `OPEN OUTPUT` กับ `ACCESS MODE IS SEQUENTIAL` บังคับให้ต้อง
`WRITE` เรียง Key จากน้อยไปมากเท่านั้น ขั้นตอนนี้จะแนะนำ **`ACCESS MODE IS DYNAMIC`** ซึ่งเป็นโหมด
ที่**ยืดหยุ่นที่สุด**: รองรับทั้ง `READ` แบบสุ่ม, `READ NEXT RECORD` แบบเรียงลำดับ, และที่สำคัญคือ
เมื่อเปิดด้วย `OPEN I-O` จะสามารถ **`WRITE` แทรก record ใหม่ที่ Key อยู่ตรงกลางไฟล์เดิมได้ทันที
โดยไม่ต้องเรียงลำดับก่อน**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP275-DYNAMIC-INSERT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST275.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID. MOVE "SOMCHAI" TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           MOVE 10005 TO CUST-ID. MOVE "ANAN"    TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

      *> With ACCESS MODE DYNAMIC and OPEN I-O, WRITE no longer
      *> requires ascending order - the ISAM index engine (BDB)
      *> puts the new record in its correct sorted slot no matter
      *> what order we insert in.
           OPEN I-O CUSTOMER-FILE.
           MOVE 10003 TO CUST-ID. MOVE "PRASERT" TO CUST-NAME.
           WRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY "Insert of 10003 failed."
           END-WRITE.
           MOVE 10002 TO CUST-ID. MOVE "SUDA"    TO CUST-NAME.
           WRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY "Insert of 10002 failed."
           END-WRITE.
           CLOSE CUSTOMER-FILE.

           DISPLAY "Reading the whole file back in KEY order:".
           OPEN INPUT CUSTOMER-FILE.
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " CUST-ID " " CUST-NAME
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Reading the whole file back in KEY order:
  10001 SOMCHAI
  10002 SUDA
  10003 PRASERT
  10005 ANAN
```

### อธิบายจุดสำคัญ

- แม้จะ `WRITE` ค่า `10003` ก่อน `10002` (ลำดับที่ "ผิด" ถ้าเทียบกับกฎในขั้นตอนที่ 271) โปรแกรมก็
  ยังทำงานได้ปกติ ไม่มี error ใด ๆ เพราะเปิดไฟล์ด้วย `OPEN I-O` ร่วมกับ `ACCESS MODE IS DYNAMIC`
- เมื่ออ่านไฟล์กลับด้วย `READ NEXT RECORD` ผลลัพธ์จะเรียงตาม **Key** เสมอ (`10001, 10002, 10003,
  10005`) ไม่ใช่ตามลำดับที่ `WRITE` จริง (`10001, 10005, 10003, 10002`) นี่คือหลักฐานว่าเครื่องมือ
  ISAM (ในที่นี้คือ Berkeley DB) จัดเรียงตำแหน่งทางกายภาพให้เราโดยอัตโนมัติเบื้องหลัง
- `ACCESS MODE IS DYNAMIC` คือโหมดที่ **แนะนำให้ใช้เป็นค่าเริ่มต้น** สำหรับโปรแกรมที่ต้องทำงานกับ
  ไฟล์ Indexed ในโลกจริง เพราะให้ความยืดหยุ่นสูงสุด (ผสมทั้ง `READ`, `READ NEXT RECORD`, `START`,
  `WRITE` แทรกกลาง, `REWRITE`, `DELETE` ได้ในโปรแกรมเดียว)

### ข้อควรระวัง

- ความยืดหยุ่นนี้ใช้ได้เฉพาะเมื่อเปิดด้วย `OPEN I-O` เท่านั้น หากยังเปิดด้วย `OPEN OUTPUT` (แม้จะ
  ตั้ง `ACCESS MODE IS DYNAMIC` ไว้ก็ตาม) กฎเรื่องการเขียนเรียงลำดับจากขั้นตอนที่ 271 ยังคงบังคับใช้
  อยู่ ทดสอบแล้วว่า `ACCESS MODE` เพียงอย่างเดียวไม่เพียงพอ ต้องคู่กับโหมด `OPEN` ที่ถูกต้องด้วย
- การ "แทรก" record กลางไฟล์แบบนี้ **ไม่ใช่การย้ายข้อมูลทางกายภาพ**เหมือนไฟล์ Sequential แต่เป็น
  การเพิ่มรายการใหม่เข้าไปในโครงสร้างดัชนี B-Tree ซึ่งมีต้นทุนต่ำกว่ามาก (ต่างจากที่อธิบายไว้ใน
  Part 025 ขั้นตอน 247 ว่าทำไมไฟล์ Sequential ทำแบบนี้ไม่ได้เลย)

### แบบฝึกหัดที่ 275.1

**โจทย์**: จงอธิบายว่าทำไมผลลัพธ์การอ่านไฟล์กลับจึงได้ลำดับ `10001, 10002, 10003, 10005` ทั้งที่
คำสั่ง `WRITE` ในโปรแกรมเขียนตามลำดับ `10001, 10005, 10003, 10002`

**เฉลย**: เพราะไฟล์ Indexed ไม่ได้เก็บ record ตามลำดับที่ `WRITE` เรียกจริง แต่จัดเก็บตำแหน่งทาง
กายภาพผ่านโครงสร้างดัชนีที่เรียงตามค่า `RECORD KEY` เสมอ เมื่อ `READ NEXT RECORD` ถูกเรียก ระบบ
จะไล่อ่านตามลำดับที่ดัชนีบอก (คือลำดับ Key จากน้อยไปมาก) ไม่ใช่ลำดับเวลาที่แต่ละ record ถูกเขียน
ลงไป นี่คือความแตกต่างพื้นฐานที่สุดระหว่างไฟล์ Indexed กับไฟล์ Sequential

---

## ขั้นตอนที่ 276: REWRITE บนไฟล์ Indexed — ปลอดภัยกว่า LINE SEQUENTIAL เสมอ

### ทบทวนปัญหาจาก Part 025

Part 025 ขั้นตอนที่ 246 พิสูจน์ให้เห็นว่า `REWRITE` บนไฟล์ `LINE SEQUENTIAL` อาจ**ตัดข้อมูลทิ้งแบบ
เงียบ ๆ** ถ้าบรรทัดเดิมบนดิสก์สั้นกว่าค่าที่ต้องการเขียนทับ (เพราะ `LINE SEQUENTIAL` ตัดช่องว่างท้าย
อัตโนมัติ) ขั้นตอนนี้จะพิสูจน์ว่า **ไฟล์ Indexed ไม่มีปัญหานี้เลยแม้แต่กรณีเดียว** เพราะเก็บข้อมูล
เป็น record ความยาวคงที่แบบไบนารีเสมอ ไม่มีการตัดช่องว่างท้ายบรรทัดใด ๆ ทั้งสิ้น

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP276-REWRITE-INDEXED.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST276.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).
           05  CUST-BALANCE         PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME.
           MOVE 500.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.
           MOVE 10002 TO CUST-ID.
           MOVE "SUDA MEECHAI" TO CUST-NAME.
           MOVE 2750.50 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

      *> Unlike LINE SEQUENTIAL (Part 025, step 246), an INDEXED
      *> file always stores the FULL fixed-length record - there
      *> is no trailing-space trimming at all, so REWRITE here is
      *> ALWAYS 100% safe no matter how long the new value is,
      *> as long as it still fits the declared PICTURE width.
           OPEN I-O CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "Not found."
               NOT INVALID KEY
                   ADD 250.00 TO CUST-BALANCE
                   REWRITE CUSTOMER-RECORD
                       INVALID KEY
                           DISPLAY "Rewrite failed."
                   END-REWRITE
                   DISPLAY "New balance: " CUST-BALANCE
           END-READ.
           CLOSE CUSTOMER-FILE.

           OPEN INPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "Not found."
               NOT INVALID KEY
                   DISPLAY "Confirmed on disk: " CUST-NAME " "
                       CUST-BALANCE
           END-READ.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
New balance: 0000750.00
Confirmed on disk: SOMCHAI JAIDEE       0000750.00
```

### อธิบายจุดสำคัญ

- ลำดับการทำงานเหมือนกับ `REWRITE` บนไฟล์ Sequential ทุกประการ (`READ` ก่อน แก้ค่า แล้ว
  `REWRITE`) แต่**ไม่มีความเสี่ยงเรื่องความยาวบรรทัดเลย** เพราะทุก record มีความกว้างคงที่เท่ากับ
  ผลรวมของทุก `PIC` ใน `01 CUSTOMER-RECORD` เสมอ (ในที่นี้คือ 5+20+9 = 34 ไบต์ ทุก record เสมอ)
- `REWRITE` บนไฟล์ Indexed **ยังต้องอ่าน record นั้นมาก่อนด้วย `READ`** เหมือนเดิมทุกประการ
  (กฎเดียวกับ Part 025 ขั้นตอน 245) และค่า `RECORD KEY` (`CUST-ID`) ใน record area **ห้ามถูกแก้ไข**
  ก่อน `REWRITE` เพราะ `REWRITE` ไม่อนุญาตให้เปลี่ยนค่า Key ของ record ที่มีอยู่แล้ว

### ข้อควรระวัง

- ถ้าพยายามแก้ไขค่า `CUST-ID` (Key) ก่อนเรียก `REWRITE` แล้วค่าที่แก้ไม่ตรงกับค่าที่ `READ` มา
  จะได้ `INVALID KEY` ทันที เพราะเท่ากับพยายาม "ย้าย" record ไปเป็นอีก Key หนึ่งซึ่ง `REWRITE` ทำ
  ไม่ได้ (ถ้าต้องการเปลี่ยน Key ต้อง `DELETE` ตัวเก่าแล้ว `WRITE` ใหม่ด้วย Key ใหม่แทน)
- แม้ `REWRITE` บนไฟล์ Indexed จะปลอดภัยเรื่องความยาว แต่ยังคงต้องเปิดไฟล์ด้วย `OPEN I-O` เท่านั้น
  เหมือนไฟล์ Sequential ทุกประการ (ทบทวนจาก Part 025 ขั้นตอน 241)

### แบบฝึกหัดที่ 276.1

**โจทย์**: จงอธิบายว่าทำไม `REWRITE` บนไฟล์ Indexed จึงไม่มีความเสี่ยงเรื่องการตัดข้อมูลแบบเงียบ ๆ
เหมือนที่เกิดกับไฟล์ `LINE SEQUENTIAL` ใน Part 025 ขั้นตอนที่ 246

**เฉลย**: เพราะไฟล์ `LINE SEQUENTIAL` เป็นไฟล์ข้อความที่ตัดช่องว่างท้ายบรรทัดออกอัตโนมัติตอนเขียน
ทำให้ความยาวจริงบนดิสก์อาจสั้นกว่าที่ `PIC` ประกาศไว้ ในขณะที่ไฟล์ Indexed เก็บทุก record เป็น
ก้อนไบนารีที่มีความกว้างคงที่เท่ากับผลรวมของทุก `PIC` ใน record เสมอ ไม่มีการตัดอะไรทิ้งเลยไม่ว่า
ข้อมูลจริงจะสั้นแค่ไหน (ช่องที่เหลือถูกเติมด้วยช่องว่างแต่ยังคงอยู่ในไฟล์จริง) `REWRITE` จึงมีพื้นที่
เต็มความกว้างให้เขียนทับได้เสมอ

---

## ขั้นตอนที่ 277: DELETE บนไฟล์ Indexed — ทำงานได้จริงตามที่ Sequential ทำไม่ได้

### ทบทวนข้อจำกัดจาก Part 025

Part 025 ขั้นตอนที่ 247 พิสูจน์ว่า `DELETE` **compile ไม่ผ่านเลย**สำหรับไฟล์ `LINE SEQUENTIAL`
เพราะไม่มีแนวคิดเรื่อง "ตำแหน่งว่างระหว่างกลาง" ขั้นตอนนี้จะแสดงให้เห็นว่าไฟล์ Indexed ไม่มีข้อจำกัด
นี้เลย เพราะโครงสร้างดัชนี B-Tree สามารถ "ทำเครื่องหมายว่าตำแหน่งนี้ว่างแล้ว" ได้โดยไม่กระทบ record
อื่นเลยแม้แต่รายการเดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP277-DELETE-INDEXED.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST277.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID. MOVE "SOMCHAI" TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           MOVE 10002 TO CUST-ID. MOVE "SUDA"    TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           MOVE 10003 TO CUST-ID. MOVE "PRASERT" TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

      *> DELETE removes the record whose key was set by the LAST
      *> successful READ - unlike LINE SEQUENTIAL (Part 025, step
      *> 247), this compiles AND runs fine on an INDEXED file,
      *> because BDB can mark a slot free without shifting anyone.
           OPEN I-O CUSTOMER-FILE.
           MOVE 10002 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "Cannot delete - not found."
               NOT INVALID KEY
                   DELETE CUSTOMER-FILE RECORD
                       INVALID KEY
                           DISPLAY "Delete failed."
                   END-DELETE
                   DISPLAY "Deleted customer 10002."
           END-READ.
           CLOSE CUSTOMER-FILE.

           DISPLAY "Remaining records:".
           OPEN INPUT CUSTOMER-FILE.
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " CUST-ID " " CUST-NAME
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.

           DISPLAY "Trying to READ deleted key 10002 again:".
           OPEN INPUT CUSTOMER-FILE.
           MOVE 10002 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "  Confirmed: 10002 no longer exists."
               NOT INVALID KEY
                   DISPLAY "  Still found?! " CUST-NAME
           END-READ.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Deleted customer 10002.
Remaining records:
  10001 SOMCHAI
  10003 PRASERT
Trying to READ deleted key 10002 again:
  Confirmed: 10002 no longer exists.
```

### อธิบายจุดสำคัญ

- **สำคัญ (ทดสอบยืนยันแล้วด้วย GnuCOBOL)**: ในโหมด `ACCESS MODE IS RANDOM` หรือ `DYNAMIC`
  `DELETE CUSTOMER-FILE RECORD` **ไม่จำเป็นต้อง `READ` มาก่อนเลย** — แค่ `MOVE` ค่า Key ที่ต้องการ
  ลบเข้า `CUST-ID` แล้วเรียก `DELETE` ตรง ๆ ก็ลบได้ทันที (กฎ "ต้อง `READ` ก่อน" ใช้เฉพาะกรณี
  `ACCESS MODE IS SEQUENTIAL` เท่านั้น ซึ่งจะลบ record ที่เพิ่งอ่านมาล่าสุด) โปรแกรมตัวอย่างนี้
  `READ` ก่อนโดยตั้งใจ **เพื่อตรวจสอบว่า record มีอยู่จริงก่อนลบเท่านั้น** ไม่ใช่ข้อบังคับทางไวยากรณ์
  (ดูการพิสูจน์แบบไม่ต้อง `READ` เลยในแบบฝึกหัดท้ายขั้นตอนนี้)
- หลัง `DELETE` สำเร็จ record นั้นจะหายไปจากทั้งการอ่านแบบเรียงลำดับ (`READ NEXT RECORD`) และการ
  อ่านแบบสุ่ม (`READ` ด้วย Key เดิม) ทันที ไม่มีร่องรอยหลงเหลือให้ค้นเจอได้อีกเลย
- `DELETE` ใช้คำสั่งรูปแบบ `DELETE <ชื่อไฟล์> RECORD` (มีคำว่า `RECORD` ต่อท้ายชื่อไฟล์) ต่างจาก
  `REWRITE`/`WRITE` ที่ใช้ชื่อ record (`CUSTOMER-RECORD`) ตามหลัง

### ข้อควรระวัง

- ต้องเปิดไฟล์ด้วย `OPEN I-O` เท่านั้นเช่นเดียวกับ `REWRITE` — `OPEN INPUT` ไม่รองรับ `DELETE`
- การลบด้วย `DELETE` เป็นการลบถาวรทันที **ไม่มีคำสั่ง UNDO ใน COBOL** ควรพิจารณาสำรองข้อมูล
  (backup) ก่อนหากเป็นระบบที่ใช้งานจริง
- ถ้า `DELETE` โดยไม่ `READ` ก่อน และ Key นั้นไม่มีอยู่จริง COBOL จะไม่ทำให้โปรแกรมล่ม แต่จะเข้า
  สาขา `INVALID KEY` ให้ตามปกติ (ทดสอบยืนยันแล้ว) — การ `READ` ก่อนจึงเป็นเพียง**รูปแบบการเขียนโค้ด
  ที่อ่านง่ายและปลอดภัยกว่า** ไม่ใช่ข้อกำหนดของภาษา

### แบบฝึกหัดที่ 277.1

**โจทย์**: จงอธิบายว่าทำไมการ `DELETE CUSTOMER-FILE RECORD` ด้วย `ACCESS MODE IS DYNAMIC` จึง
สามารถทำได้ทันทีโดยไม่ต้อง `READ` มาก่อน ทั้งที่ `REWRITE` ยังคงต้อง `READ` มาก่อนเสมอ

**เฉลย**: เพราะ `REWRITE` ต้องมี "เนื้อหาปัจจุบันของ record area" ให้เขียนทับ (ค่าที่ `MOVE` แก้ไข
แล้วหลัง `READ`) จึงจำเป็นต้อง `READ` เข้ามาก่อนเสมอไม่ว่าโหมดใดก็ตาม แต่ `DELETE` ไม่ต้องมีเนื้อหา
ใด ๆ มาเกี่ยวข้องเลย เพียงแค่บอกว่า "ลบ record ที่มี Key เท่ากับค่านี้" ก็เพียงพอแล้ว ในโหมด `RANDOM`
หรือ `DYNAMIC` COBOL จึงออกแบบให้ `DELETE` ใช้ค่าปัจจุบันใน field ของ `RECORD KEY` ค้นหาและลบได้เลย
โดยตรง (คนละกลไกกับโหมด `SEQUENTIAL` ที่ไม่มีแนวคิดเรื่อง "ค่า Key ปัจจุบัน" ชัดเจน จึงต้องอ้างอิง
จาก record ที่เพิ่ง `READ` มาล่าสุดแทน)

---

## ขั้นตอนที่ 278: ข้อผิดพลาด Duplicate Key และการจัดการด้วย FILE STATUS

### RECORD KEY ต้องไม่ซ้ำกันเสมอ

เพราะ `RECORD KEY` มีหน้าที่เป็น "ตัวระบุตัวตน" ของแต่ละ record ในไฟล์ COBOL จึงบังคับว่าค่า
`RECORD KEY` **ต้องไม่ซ้ำกัน**ในไฟล์เดียวกันเด็ดขาด (ต่างจาก `ALTERNATE RECORD KEY` ที่จะเรียนใน
ขั้นตอนถัดไป ซึ่งอนุญาตให้ซ้ำได้) ขั้นตอนนี้จะพิสูจน์ว่าเกิดอะไรขึ้นเมื่อพยายามฝ่าฝืนกฎนี้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP278-DUPLICATE-KEY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST278.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME.
           WRITE CUSTOMER-RECORD.
           DISPLAY "First WRITE status: " WS-FILE-STATUS.

      *> Trying to WRITE a SECOND record with the SAME key value
      *> is rejected - RECORD KEY must be unique by definition.
           MOVE 10001 TO CUST-ID.
           MOVE "IMPOSTOR RECORD" TO CUST-NAME.
           WRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY "Duplicate key rejected! Status: "
                       WS-FILE-STATUS
           END-WRITE.
           CLOSE CUSTOMER-FILE.

           OPEN INPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           READ CUSTOMER-FILE.
           DISPLAY "Record 10001 is still: " CUST-NAME.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
First WRITE status: 00
Duplicate key rejected! Status: 22
Record 10001 is still: SOMCHAI JAIDEE
```

### อธิบายจุดสำคัญ

- `FILE STATUS IS WS-FILE-STATUS` (แนะนำสั้น ๆ ใน Part 025 ขั้นตอน 242) เก็บรหัสผลลัพธ์ 2 หลัก
  ของทุกปฏิบัติการไฟล์ **`00` แปลว่าสำเร็จปกติ** และ **`22` แปลว่าพยายามเขียน Key ที่ซ้ำกับที่มี
  อยู่แล้ว** — Part 030 จะสอนตารางรหัส File Status ทั้งหมดอย่างละเอียด
- สังเกตว่าแม้ `WRITE` ตัวที่สองจะ "ล้มเหลว" (ถูกปฏิเสธ) แต่**โปรแกรมไม่ล่ม**เลย เพราะเราดักจับ
  ด้วย `INVALID KEY` ไว้แล้ว — record เดิม (`SOMCHAI JAIDEE`) ยังคงอยู่ไม่ถูกเขียนทับ
- การป้องกัน Key ซ้ำนี้เกิดขึ้น**อัตโนมัติในระดับไฟล์**โดยไม่ต้องเขียนโค้ดตรวจสอบเอง (ต่างจากการ
  ตรวจสอบข้อมูลซ้ำในไฟล์ Sequential ที่ต้องเขียนอัลกอริทึมตรวจสอบเองทั้งหมด)

### ข้อควรระวัง

- ถ้าไม่มี `INVALID KEY` clause คอยดักไว้ การ `WRITE` ที่ซ้ำ Key จะทำให้เกิด runtime error ที่
  รุนแรงกว่านี้ (โปรแกรมอาจหยุดทำงานทันที) จึงควรใส่ `INVALID KEY` ทุกครั้งที่ `WRITE` ลงไฟล์
  Indexed เป็นแนวปฏิบัติที่ดี
- อย่าลืมว่า `FILE STATUS` เป็น `PIC XX` (ตัวอักษร 2 ตัว) ไม่ใช่ตัวเลข แม้ค่าจะดูเหมือนตัวเลขก็ตาม
  (`"00"`, `"22"` คือข้อความ ไม่ใช่ตัวเลข 0 หรือ 22)

### แบบฝึกหัดที่ 278.1

**โจทย์**: จงอธิบายว่าทำไมค่า `WS-FILE-STATUS` หลัง `WRITE` ตัวแรกจึงเป็น `"00"` ทั้งที่ยังไม่มีการ
ตรวจสอบเงื่อนไขอะไรเลยในโค้ด

**เฉลย**: เพราะ **ทุกปฏิบัติการไฟล์**ใน COBOL (ไม่ว่าจะเป็น `OPEN`, `READ`, `WRITE`, `REWRITE`,
`DELETE`, `CLOSE`) จะอัปเดตค่าใน field ที่ระบุไว้ที่ `FILE STATUS IS` โดยอัตโนมัติทุกครั้งที่ถูก
เรียก ไม่ว่าผลลัพธ์จะสำเร็จหรือล้มเหลว ค่า `"00"` คือรหัสมาตรฐานที่แปลว่า "สำเร็จสมบูรณ์" ซึ่งจะถูก
ตั้งค่าให้อัตโนมัติทันทีที่ `WRITE` ตัวแรกทำงานเสร็จโดยไม่มีปัญหาใด ๆ (Key `10001` ยังไม่เคยมีมาก่อน
ในไฟล์ที่เพิ่งสร้างใหม่)

---

## ขั้นตอนที่ 279: ALTERNATE RECORD KEY — ค้นหาด้วย Key รอง

### ปัญหา: ต้องการค้นหาด้วย field อื่นที่ไม่ใช่ Key หลัก

`RECORD KEY` เดิม (`CUST-ID`) เหมาะกับการค้นหาด้วยรหัสลูกค้าที่ไม่ซ้ำกัน แต่ในทางปฏิบัติเรามักต้องการ
ค้นหาด้วย field อื่นด้วย เช่น "ลูกค้าทุกคนที่อยู่จังหวัดกรุงเทพฯ" ซึ่ง field `CUST-CITY` **ซ้ำกันได้**
ระหว่างหลายลูกค้า COBOL รองรับกรณีนี้ผ่าน **`ALTERNATE RECORD KEY`** ซึ่งสร้างดัชนีที่สองแยกต่างหาก
ให้กับไฟล์เดียวกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP279-ALTERNATE-KEY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST279.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               ALTERNATE RECORD KEY IS CUST-CITY
                   WITH DUPLICATES.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(15).
           05  CUST-CITY            PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI" TO CUST-NAME.
           MOVE "BANGKOK" TO CUST-CITY.
           WRITE CUSTOMER-RECORD.

           MOVE 10002 TO CUST-ID.
           MOVE "SUDA" TO CUST-NAME.
           MOVE "CHIANGMAI" TO CUST-CITY.
           WRITE CUSTOMER-RECORD.

           MOVE 10003 TO CUST-ID.
           MOVE "PRASERT" TO CUST-NAME.
           MOVE "BANGKOK" TO CUST-CITY.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

      *> Two customers share CUST-CITY = "BANGKOK" - this is fine
      *> because the alternate key was declared WITH DUPLICATES.
      *> We can now search directly by CITY instead of by ID.
           OPEN INPUT CUSTOMER-FILE.
           MOVE "BANGKOK" TO CUST-CITY.
           START CUSTOMER-FILE KEY IS = CUST-CITY
               INVALID KEY
                   DISPLAY "No customer found in that city."
           END-START.

           DISPLAY "Customers in BANGKOK:".
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF CUST-CITY = "BANGKOK"
                           DISPLAY "  " CUST-ID " " CUST-NAME
                       ELSE
                           SET END-OF-FILE TO TRUE
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Customers in BANGKOK:
  10001 SOMCHAI
  10003 PRASERT
```

### อธิบายจุดสำคัญ

- `ALTERNATE RECORD KEY IS CUST-CITY WITH DUPLICATES`: ประกาศดัชนีที่สองโดยใช้ field
  `CUST-CITY` คำว่า **`WITH DUPLICATES` จำเป็นต้องมี** เพราะค่าในนี้ซ้ำกันได้ (ถ้าไม่ใส่ COBOL จะ
  ถือว่าต้องไม่ซ้ำเหมือน `RECORD KEY` หลัก และจะ error ทันทีที่พยายามเขียนค่าซ้ำ)
  แต่ละไฟล์สามารถมี `ALTERNATE RECORD KEY` ได้หลายตัว (ไม่จำกัดแค่ตัวเดียว)
  ในทางปฏิบัติ ทำให้เกิดดัชนี B-Tree แยกกันหลายชุดสำหรับไฟล์เดียวกัน
- `START CUSTOMER-FILE KEY IS = CUST-CITY`: เมื่อ `CUST-CITY` (ไม่ใช่ `CUST-ID`) เป็น field ที่
  เรามีค่าอยู่ในมือขณะนั้น COBOL จะฉลาดพอที่จะรู้ว่าต้องใช้ดัชนีของ `ALTERNATE RECORD KEY`
  ในการค้นหาแทนดัชนีหลัก เพราะ field ที่ระบุตรงกับชื่อ Alternate Key ที่ประกาศไว้
- หลัง `START` ด้วย Alternate Key แล้ว `READ NEXT RECORD` จะไล่อ่านตาม**ลำดับของ Alternate Key
  นั้น** (เรียงตาม `CUST-CITY`) ไม่ใช่ลำดับของ `CUST-ID` อีกต่อไป จึงต้องมี `IF CUST-CITY =
  "BANGKOK"` คอยตรวจสอบว่ายังอยู่ในกลุ่มที่ต้องการหรือไม่ เพราะเมื่อพ้นกลุ่ม `"BANGKOK"` ไปแล้ว
  จะเจอเมืองอื่นปะปนต่อไปเรื่อย ๆ ในลำดับตัวอักษร

### ข้อควรระวัง

- ต้องตรวจสอบเงื่อนไขหยุด (`IF CUST-CITY = "BANGKOK"`) ด้วยตนเองเสมอ เพราะ `READ NEXT RECORD`
  จะไม่หยุดอัตโนมัติเมื่อพ้นกลุ่มค่าที่ค้นหา มันจะอ่านต่อไปเรื่อย ๆ จนสุดไฟล์ (`AT END`) ถ้าไม่มี
  การตรวจสอบเอง
- `ALTERNATE RECORD KEY` เพิ่มภาระให้กับทุกครั้งที่ `WRITE`/`REWRITE`/`DELETE` เพราะต้องปรับปรุง
  ดัชนีทุกตัวพร้อมกัน (ทั้ง Primary และ Alternate) ไฟล์ที่มี Alternate Key จำนวนมากจะเขียนช้าลง
  เล็กน้อยแต่แลกมาด้วยความเร็วในการค้นหาที่หลากหลายมากขึ้น

### แบบฝึกหัดที่ 279.1

**โจทย์**: จงอธิบายว่าทำไมการประกาศ `ALTERNATE RECORD KEY IS CUST-CITY` โดย**ไม่ใส่** `WITH
DUPLICATES` จะทำให้โปรแกรมข้างต้นเขียนไฟล์ไม่สำเร็จ

**เฉลย**: เพราะถ้าไม่ระบุ `WITH DUPLICATES` COBOL จะถือว่า `CUST-CITY` ต้องมีค่า**ไม่ซ้ำกัน**ทุก
record เหมือนกับกฎของ `RECORD KEY` หลัก ในตัวอย่างนี้ลูกค้า `10001` และ `10003` ต่างก็มี
`CUST-CITY` เป็น `"BANGKOK"` เหมือนกัน เมื่อ `WRITE` record ที่สองที่มีค่า `"BANGKOK"` ซ้ำ จะได้รับ
`INVALID KEY` (สถานะ `22` เช่นเดียวกับที่พิสูจน์ในขั้นตอนที่ 278) ทันที ทำให้เขียนไฟล์ไม่ครบตามที่
ตั้งใจไว้

---

## ขั้นตอนที่ 280: โปรแกรมรวบยอด — ระบบ Customer Master แบบ CRUD ครบวงจร

### เป้าหมาย

ขั้นตอนสุดท้ายของ Part นี้จะรวมทุกอย่างที่เรียนมาทั้ง 9 ขั้นตอนเข้าด้วยกันเป็นโปรแกรมเดียวที่จำลอง
วงจรชีวิตทั่วไปของ**ไฟล์หลัก (Master File)** ในระบบธุรกิจจริง: สร้างไฟล์ → เพิ่มข้อมูลใหม่
(`WRITE`) → แก้ไขข้อมูล (`REWRITE`) → ลบข้อมูล (`DELETE`) → พิมพ์รายงานสรุป (`READ NEXT RECORD`)
พร้อมทั้งใช้ `ALTERNATE RECORD KEY` และ `FILE STATUS` ควบคู่กันแบบที่ใช้งานจริงในองค์กร

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP280-CUSTOMER-MASTER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTMAST280.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               ALTERNATE RECORD KEY IS CUST-CITY
                   WITH DUPLICATES
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(15).
           05  CUST-CITY            PIC X(10).
           05  CUST-BALANCE         PIC 9(7)V99.

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-TOTAL-BALANCE         PIC 9(8)V99 VALUE 0.
       01  WS-DISPLAY-TOTAL         PIC ZZZ,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM BUILD-MASTER-FILE.
           PERFORM ADD-NEW-CUSTOMER.
           PERFORM UPDATE-CUSTOMER-BALANCE.
           PERFORM DELETE-CUSTOMER.
           PERFORM PRINT-FULL-REPORT.
           STOP RUN.

       BUILD-MASTER-FILE.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI" TO CUST-NAME.
           MOVE "BANGKOK" TO CUST-CITY.
           MOVE 5000.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.

           MOVE 10002 TO CUST-ID.
           MOVE "SUDA" TO CUST-NAME.
           MOVE "CHIANGMAI" TO CUST-CITY.
           MOVE 3200.50 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.

           MOVE 10003 TO CUST-ID.
           MOVE "PRASERT" TO CUST-NAME.
           MOVE "BANGKOK" TO CUST-CITY.
           MOVE 1500.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.
           DISPLAY "1) Built master file with 3 customers.".

       ADD-NEW-CUSTOMER.
           OPEN I-O CUSTOMER-FILE.
           MOVE 10004 TO CUST-ID.
           MOVE "WANNA" TO CUST-NAME.
           MOVE "PHUKET" TO CUST-CITY.
           MOVE 900.00 TO CUST-BALANCE.
           WRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY "2) Add failed - duplicate ID."
               NOT INVALID KEY
                   DISPLAY "2) Added customer 10004 (WANNA)."
           END-WRITE.
           CLOSE CUSTOMER-FILE.

       UPDATE-CUSTOMER-BALANCE.
           OPEN I-O CUSTOMER-FILE.
           MOVE 10002 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "3) Update failed - not found."
               NOT INVALID KEY
                   ADD 1000.00 TO CUST-BALANCE
                   REWRITE CUSTOMER-RECORD
                   DISPLAY "3) Updated 10002 balance to "
                       CUST-BALANCE
           END-READ.
           CLOSE CUSTOMER-FILE.

       DELETE-CUSTOMER.
           OPEN I-O CUSTOMER-FILE.
           MOVE 10003 TO CUST-ID.
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY "4) Delete failed - not found."
               NOT INVALID KEY
                   DELETE CUSTOMER-FILE RECORD
                   DISPLAY "4) Deleted customer 10003 (PRASERT)."
           END-READ.
           CLOSE CUSTOMER-FILE.

       PRINT-FULL-REPORT.
           DISPLAY "5) ---- Final Customer Master Report ----".
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT CUSTOMER-FILE.
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "   " CUST-ID " " CUST-NAME
                           " (" CUST-CITY ") bal="
                           CUST-BALANCE
                       ADD CUST-BALANCE TO WS-TOTAL-BALANCE
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
           MOVE WS-TOTAL-BALANCE TO WS-DISPLAY-TOTAL.
           DISPLAY "   Total balance remaining: "
               WS-DISPLAY-TOTAL.
```

**ผลลัพธ์:**

```
1) Built master file with 3 customers.
2) Added customer 10004 (WANNA).
3) Updated 10002 balance to 0004200.50
4) Deleted customer 10003 (PRASERT).
5) ---- Final Customer Master Report ----
   10001 SOMCHAI         (BANGKOK   ) bal=0005000.00
   10002 SUDA            (CHIANGMAI ) bal=0004200.50
   10004 WANNA           (PHUKET    ) bal=0000900.00
   Total balance remaining:      10,100.50
```

### อธิบายภาพรวมของโปรแกรม

โปรแกรมนี้แบ่งงานออกเป็น 5 paragraph ย่อยตามหลักการที่เรียนมาจาก Part 014 (Paragraph, Section,
`PERFORM ... THRU`) แต่ละ paragraph รับผิดชอบหน้าที่เดียวชัดเจน (Single Responsibility) ตรงตาม
รูปแบบที่ระบบธุรกิจจริงใช้บริหารจัดการ Master File:

1. **BUILD-MASTER-FILE**: สร้างไฟล์เริ่มต้น (เทียบเท่ากับการ Migrate ข้อมูลเข้าระบบครั้งแรก)
2. **ADD-NEW-CUSTOMER**: เพิ่มลูกค้าใหม่ระหว่างการใช้งานปกติ (`WRITE` ผ่าน `OPEN I-O`)
3. **UPDATE-CUSTOMER-BALANCE**: แก้ไขข้อมูลลูกค้าที่มีอยู่แล้ว (อ่านแล้วแก้แล้ว `REWRITE`)
4. **DELETE-CUSTOMER**: ลบลูกค้าที่ไม่ต้องการแล้วออกจากระบบ
5. **PRINT-FULL-REPORT**: สรุปผลลัพธ์สุดท้ายทั้งหมด พร้อมคำนวณยอดรวม (เทคนิคจาก Part 009)

สังเกตว่าทุก paragraph `OPEN`/`CLOSE` ไฟล์ของตัวเองอย่างอิสระ (ไม่เปิดค้างข้าม paragraph) ตามหลัก
วินัยการจัดการไฟล์ที่ดีซึ่งเน้นย้ำมาตั้งแต่ Part 025

### ข้อควรระวัง

- การเปิด-ปิดไฟล์บ่อยครั้ง (ทุก paragraph) มีต้นทุนด้าน performance เล็กน้อยเมื่อเทียบกับการเปิดไฟล์
  ครั้งเดียวค้างไว้ตลอดโปรแกรม ในระบบที่ต้องประมวลผลจำนวนมากอาจพิจารณาเปิดไฟล์ครั้งเดียวใน
  `MAIN-PARA` แล้วส่งต่อให้ paragraph ย่อยทำงานแทน — เป็นการตัดสินใจเชิง trade-off ระหว่างความ
  ปลอดภัย (แต่ละ paragraph อิสระ ทดสอบง่าย) กับ performance
- โปรแกรมนี้ไม่ได้ตรวจสอบ `WS-FILE-STATUS` อย่างเป็นระบบ (ใช้แค่ `INVALID KEY`/`NOT INVALID KEY`)
  ทั้งที่ประกาศ `FILE STATUS IS WS-FILE-STATUS` ไว้ — Part 030 จะสอนวิธีตรวจสอบและจัดการ
  `FILE STATUS` อย่างครบถ้วนเป็นระบบสำหรับโปรแกรมระดับองค์กรจริง

### แบบฝึกหัดที่ 280.1

**โจทย์**: จงเพิ่ม paragraph ใหม่ชื่อ `FIND-BY-CITY` ที่ค้นหาและแสดงลูกค้าทั้งหมดในเมือง
`"BANGKOK"` โดยใช้ `ALTERNATE RECORD KEY` แล้วเรียกใช้ต่อจาก `DELETE-CUSTOMER` ก่อน
`PRINT-FULL-REPORT`

**เฉลย**:

```cobol
       FIND-BY-CITY.
           OPEN INPUT CUSTOMER-FILE.
           MOVE "BANGKOK" TO CUST-CITY.
           START CUSTOMER-FILE KEY IS = CUST-CITY
               INVALID KEY
                   DISPLAY "No customer in BANGKOK."
           END-START.
           MOVE "N" TO WS-EOF-FLAG.
           DISPLAY "4.5) Customers in BANGKOK:".
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF CUST-CITY = "BANGKOK"
                           DISPLAY "   " CUST-ID " " CUST-NAME
                       ELSE
                           SET END-OF-FILE TO TRUE
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
```

และเพิ่ม `PERFORM FIND-BY-CITY.` ใน `MAIN-PARA` ต่อจาก `PERFORM DELETE-CUSTOMER.` เนื่องจากลูกค้า
`10003` (BANGKOK) ถูกลบไปแล้วก่อนหน้านี้ ผลลัพธ์จะแสดงเฉพาะ `10001 SOMCHAI` เท่านั้นที่ยังอยู่ใน
เมือง BANGKOK

---

## สรุปท้ายบท

Part นี้พาคุณข้ามผ่านข้อจำกัดที่ใหญ่ที่สุดของไฟล์ตามลำดับที่สะสมมาตั้งแต่ Part 023 ได้สำเร็จ ด้วย
การแนะนำ **Indexed File** และเทคนิค **ISAM** สรุปสิ่งที่เรียนรู้ทั้งหมด:

- แนวคิด ISAM: โครงสร้างดัชนีที่ทำให้ค้นหา record ได้โดยตรงผ่าน Key โดยไม่ต้องอ่านไฟล์ทั้งหมด
- การประกาศไฟล์ Indexed ด้วย `ORGANIZATION IS INDEXED` และ `RECORD KEY IS`
- กับดักสำคัญเรื่องการเปรียบเทียบ Key แบบไบต์ต่อไบต์ และทำไม Key ตัวเลขต้องเป็น `PIC 9` เสมอ
- การอ่านแบบสุ่ม (`ACCESS MODE IS RANDOM`) ด้วย `READ`/`INVALID KEY`/`NOT INVALID KEY`
- `START` สำหรับกำหนดจุดเริ่มต้นการอ่านแบบเรียงลำดับโดยไม่ต้องอ่านจากต้นไฟล์เสมอ
- `ACCESS MODE IS DYNAMIC` ที่ให้ความยืดหยุ่นสูงสุด รวมถึงการเขียนแทรก record โดยไม่ต้องเรียง
  ลำดับ พร้อมพิสูจน์ว่า ISAM จัดเรียงตำแหน่งทางกายภาพให้อัตโนมัติ
- `REWRITE` บนไฟล์ Indexed ที่ปลอดภัย 100% ไม่มีปัญหาตัดข้อมูลแบบไฟล์ `LINE SEQUENTIAL`
- `DELETE` ที่ใช้งานได้จริงบนไฟล์ Indexed ซึ่งเป็นสิ่งที่ไฟล์ Sequential ทำไม่ได้เลย
- `ALTERNATE RECORD KEY WITH DUPLICATES` สำหรับสร้างดัชนีค้นหาที่สองบน field ที่ค่าซ้ำกันได้
- โปรแกรมรวบยอด CRUD ครบวงจรที่จำลองการบริหารจัดการ Master File แบบที่ใช้งานจริงในองค์กร

Part ถัดไป (**Part 029**) จะแนะนำไฟล์ประเภทที่สามและประเภทสุดท้ายของ COBOL: **Relative File**
ซึ่งเข้าถึง record ด้วย **หมายเลขลำดับ (Relative Record Number)** แทนที่จะเป็น Key ที่มีความหมาย
ทางธุรกิจ เหมาะสำหรับกรณีที่ต้องการความเร็วสูงสุดในการเข้าถึงแบบสุ่มโดยรู้ตำแหน่ง record ล่วงหน้า
(เช่น ระบบจองที่นั่งที่อ้างอิงด้วยหมายเลขที่นั่งโดยตรง)

**[← กลับไป Part 027](part-027-sort-merge.md)** | **[ไปยัง Part 029: Relative Files →](part-029-relative-files.md)**
