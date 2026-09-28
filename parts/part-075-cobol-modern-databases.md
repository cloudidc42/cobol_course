# Part 075: ฐานข้อมูลสมัยใหม่: COBOL กับ MySQL/PostgreSQL (ขั้นตอนที่ 741–750)

## คำนำของ Part นี้

ใน Part 057–060 เราเรียนรู้การเชื่อมต่อ COBOL กับฐานข้อมูล **DB2** ผ่าน **Embedded SQL**
(`EXEC SQL ... END-EXEC`) ซึ่งเป็นวิธีมาตรฐานที่ใช้บน Mainframe จริงมานานหลายสิบปี — GnuCOBOL
เองก็รองรับ Embedded SQL ผ่าน precompiler ภายนอกสำหรับบางฐานข้อมูล แต่ในทางปฏิบัติของ GnuCOBOL
รุ่นมาตรฐานทั่วไป (รวมถึงบิลด์ที่ใช้ในหลักสูตรนี้) **ไม่มี native driver ในตัวสำหรับเชื่อมต่อกับ
MySQL หรือ PostgreSQL โดยตรง** ต่างจาก DB2 ที่ IBM ผลิต precompiler เฉพาะทางไว้ให้บน Mainframe

Part นี้จะสอนแนวทางที่ทีมพัฒนาในโลกจริงใช้แก้ปัญหานี้: **Shim Program Pattern** — เขียนโปรแกรม
ตัวกลางเล็ก ๆ (shim) ที่รู้จักวิธีคุยกับฐานข้อมูลจริง แล้วให้ COBOL program เรียกใช้ shim นี้ผ่าน
กลไกที่ COBOL มีอยู่แล้ว 2 แบบหลัก: (1) `CALL "SYSTEM"` เพื่อเรียก command-line client ของ
ฐานข้อมูลโดยตรง หรือ (2) `CALL` ไปยังโปรแกรม C ที่ link กับ client library ของฐานข้อมูลนั้น
โดยตรง (ต่อยอดจากเทคนิค COBOL-C interop ที่เรียนไปแล้วใน Part 072)

**สิ่งสำคัญที่สุดของ Part นี้คือความซื่อตรงทางเทคนิค**: เราตรวจสอบสภาพแวดล้อมจริงก่อนเขียนเนื้อหา
พบว่า **PostgreSQL ติดตั้งและใช้งานได้จริง** ในสภาพแวดล้อมของหลักสูตร (ทั้ง server และ client
library) จึงทดสอบทุกตัวอย่างที่เกี่ยวกับ PostgreSQL แบบ end-to-end จริง ส่วน **MySQL ไม่มีอยู่ใน
สภาพแวดล้อมนี้** เนื้อหาส่วน MySQL จึงถูกระบุไว้อย่างชัดเจนว่าเป็น**ไวยากรณ์อ้างอิงที่ยังไม่ได้ทดสอบ
รันจริง** (เช่นเดียวกับที่ Part 045 ทำกับ `JSON GENERATE`/`JSON PARSE`) แต่รูปแบบสถาปัตยกรรม
เหมือนกันทุกประการ เพียงเปลี่ยน client tool/library เท่านั้น

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างเป็นภาษาอังกฤษล้วน โค้ด C/SQL/bash ก็เป็น
> ภาษาอังกฤษล้วนตามธรรมชาติของโค้ดจริงเช่นกัน

---

## ขั้นตอนที่ 741: ทำไม GnuCOBOL คุยกับ MySQL/PostgreSQL โดยตรงไม่ได้

### ความแตกต่างจาก DB2 Embedded SQL ที่เรียนใน Part 057-060

DB2 Embedded SQL ทำงานได้เพราะ IBM (และผู้ผลิตเครื่องมือ) จัดทำ **precompiler เฉพาะทาง** ที่แปลง
`EXEC SQL ... END-EXEC` ให้กลายเป็นโค้ด COBOL ปกติที่เรียก DB2 client library ก่อนที่ `cobc` จะ
คอมไพล์จริง กลไกนี้ผูกติดกับ DB2 (และมาตรฐาน SQL/Embedded SQL ของ ISO) โดยเฉพาะ

**PostgreSQL และ MySQL ไม่มี precompiler มาตรฐานที่ผูกกับ GnuCOBOL โดยตรง** — GnuCOBOL รุ่นทั่วไป
(รวมถึงบิลด์ `4.0-early-dev.0` ที่ใช้ในหลักสูตรนี้) จึงไม่มีความสามารถส่ง SQL query ไปยัง
PostgreSQL/MySQL ได้เองเลยแม้แต่น้อย

### ตรวจสอบยืนยันด้วย cobc --info

```bash
cobc --info | grep -i -E "sql|db2"
```

ผลลัพธ์จากสภาพแวดล้อมของหลักสูตร**ไม่มีบรรทัดใดที่กล่าวถึงการรองรับ SQL client ของ MySQL หรือ
PostgreSQL เลย** — ยืนยันว่าฟีเจอร์นี้ไม่มีอยู่ในตัวคอมไพเลอร์โดยกำเนิด ต่างจาก JSON (Part 045) หรือ
Indexed File (Part 074) ที่อย่างน้อยมี configuration flag ให้เปิด-ปิดได้ กรณีนี้ไม่มี flag ใด ๆ
เกี่ยวข้องกับ MySQL/PostgreSQL ในสถาปัตยกรรมของ GnuCOBOL เลย

### ทางออก: Shim Program Pattern

```
[ COBOL Program ]
       |
       | (1) CALL "SYSTEM" เรียก CLI client
       | หรือ (2) CALL โปรแกรม C ที่ link กับ client library
       v
[ Shim: psql CLI หรือ C program ที่ link กับ libpq ]
       |
       | พูดภาษา SQL protocol จริงกับฐานข้อมูล
       v
[ PostgreSQL / MySQL Server ]
```

แนวคิดสำคัญคือ **แยกความรับผิดชอบ**: COBOL program ไม่จำเป็นต้องรู้จัก wire protocol ของฐานข้อมูล
เลย มีหน้าที่แค่ "ขอให้ shim ทำงานให้" แล้วรับผลลัพธ์กลับมาผ่านช่องทางง่าย ๆ (ไฟล์ หรือ parameter
ที่ส่งผ่าน CALL) — Pattern นี้คล้ายกับ Wrapper Service Pattern ใน Part 073-074 มาก เพียงแต่คราวนี้
COBOL เป็นฝ่าย**เรียก**ออกไปหาฐานข้อมูล แทนที่จะเป็นฝ่าย**ถูกเรียก**จากเว็บ

### ข้อควรระวัง

- อย่าสับสนระหว่าง "ไม่มี native driver" กับ "เชื่อมต่อฐานข้อมูลไม่ได้เลย" — Part นี้จะพิสูจน์ว่า
  COBOL เชื่อมต่อ PostgreSQL ได้จริงและทำงานได้สมบูรณ์ เพียงแต่ต้องผ่าน shim แทนการเชื่อมต่อโดยตรง
- ในโลกจริง องค์กรจำนวนมากที่ใช้ COBOL ร่วมกับฐานข้อมูลสมัยใหม่ก็ใช้แนวทาง Shim Program หรือ
  Middleware แบบนี้เช่นกัน ไม่ใช่เทคนิคที่ด้อยกว่าหรือเป็นทางลัดที่ไม่มืออาชีพแต่อย่างใด

### แบบฝึกหัดที่ 741.1

**โจทย์**: จงอธิบายว่าทำไม DB2 Embedded SQL (Part 057-060) จึงทำงานได้โดยตรงกับ COBOL แต่
PostgreSQL/MySQL ทำไม่ได้ ทั้งที่ทั้งหมดเป็นฐานข้อมูลเชิงสัมพันธ์เหมือนกัน

**เฉลย**: ความแตกต่างไม่ได้อยู่ที่ตัวฐานข้อมูลเอง แต่อยู่ที่ **การมีหรือไม่มี precompiler ที่ผูก
กับ COBOL โดยเฉพาะ** DB2 (ของ IBM) มี precompiler ที่แปลง `EXEC SQL` เป็นโค้ด COBOL ที่เรียก DB2
client library ให้อัตโนมัติ เพราะ IBM ลงทุนพัฒนาเครื่องมือนี้ควบคู่กับ COBOL compiler ของตนเองมานาน
(เนื่องจากทั้งคู่เป็นผลิตภัณฑ์ของ IBM ที่ออกแบบมาให้ทำงานร่วมกัน) ในขณะที่ PostgreSQL และ MySQL
เป็นโปรเจกต์ Open Source ที่ไม่มีความผูกพันโดยตรงกับ GnuCOBOL หรือคอมไพเลอร์ COBOL ตัวใดเป็นพิเศษ
จึงไม่มีใครพัฒนา precompiler แบบเดียวกันมาให้ใช้งานมาตรฐาน

---

## ขั้นตอนที่ 742: สำรวจฐานข้อมูลที่มีอยู่จริงในสภาพแวดล้อมนี้ (ตรวจสอบอย่างซื่อตรง)

### ตรวจสอบเครื่องมือที่มีจริง

ก่อนเขียนเนื้อหาต่อ เราตรวจสอบสภาพแวดล้อมจริงอย่างละเอียดด้วยคำสั่งต่อไปนี้:

```bash
which mysql mysqld
which psql pg_ctl
pg_lsclusters
```

**ผลลัพธ์จริงที่ได้:**

```
(ว่างเปล่า - ไม่พบคำสั่ง mysql หรือ mysqld ในระบบเลย)

/usr/bin/psql
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

### สรุปข้อเท็จจริงที่ตรวจพบ

| ฐานข้อมูล | สถานะในสภาพแวดล้อมนี้ | แนวทางของ Part นี้ |
|---|---|---|
| **PostgreSQL** | ติดตั้งครบทั้ง server (v16) และ client (`psql`, `libpq`) แต่ service ยังไม่ได้เปิด | **ทดสอบจริงทุกตัวอย่าง** |
| **MySQL** | ไม่มีการติดตั้งใด ๆ เลยในสภาพแวดล้อมนี้ | แสดงไวยากรณ์อ้างอิงเท่านั้น ระบุชัดเจนว่ายังไม่ได้ทดสอบ |

### เริ่มการทำงานของ PostgreSQL Service

```bash
service postgresql start
pg_lsclusters
```

**ผลลัพธ์จริงหลังเริ่ม service:**

```
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

สถานะเปลี่ยนจาก `down` เป็น `online` สำเร็จ — PostgreSQL พร้อมใช้งานจริงแล้ว

### อธิบายจุดสำคัญ

- ความซื่อตรงทางเทคนิคคือหลักการที่ยึดถือมาตลอดหลักสูตรนี้ (เห็นได้จาก Part 045 เรื่อง JSON และ
  Part 074 เรื่อง Indexed File) — การตรวจสอบสภาพแวดล้อมจริงก่อนเขียนเนื้อหาทำให้เราสามารถระบุได้
  อย่างชัดเจนว่าส่วนไหน "ทดสอบแล้วรันได้จริง" และส่วนไหน "เป็นไวยากรณ์อ้างอิงที่ยังไม่ได้ทดสอบ"
- การที่ MySQL ไม่มีในสภาพแวดล้อมนี้ไม่ได้แปลว่ารูปแบบสถาปัตยกรรมที่จะสอนใช้ไม่ได้กับ MySQL — เพียง
  แต่หมายความว่าตัวอย่างเฉพาะของ MySQL ใน Part นี้ (ขั้นตอนที่ 749) จะต้องระบุไว้อย่างชัดเจนว่าเป็น
  ไวยากรณ์อ้างอิงเท่านั้น เช่นเดียวกับที่ Part 045 ทำกับ `JSON GENERATE`

### ข้อควรระวัง

- ในระบบจริง ห้าม hardcode รหัสผ่านฐานข้อมูลไว้ในโค้ดเด็ดขาด (Part นี้ใช้รหัสผ่านง่าย ๆ เพื่อการสาธิต
  เท่านั้น) รายละเอียดเรื่องความปลอดภัยจะกล่าวถึงเต็มรูปแบบในขั้นตอนที่ 747
- คำสั่ง `service postgresql start` และการตั้งค่า authentication เป็นงานระดับ System
  Administrator ที่ต้องมีสิทธิ์ที่เหมาะสม ในระบบ production จริงมักมีทีม DBA ดูแลส่วนนี้แยกต่างหาก
  จากทีมพัฒนา COBOL

### แบบฝึกหัดที่ 742.1

**โจทย์**: จงอธิบายว่าทำไมหลักสูตรนี้จึงเลือกทดสอบด้วย PostgreSQL แทนที่จะแค่เขียนเนื้อหา MySQL
ทั้งหมดแบบอ้างอิงอย่างเดียว ทั้งที่ชื่อ Part คือ "MySQL/PostgreSQL"

**เฉลยแนวทาง**: หลักการสำคัญของหลักสูตรนี้ (ยึดถือมาตั้งแต่ Part 045) คือ**ทดสอบก่อนสอนเสมอเมื่อ
เป็นไปได้** เนื่องจากสภาพแวดล้อมจริงมี PostgreSQL พร้อมใช้งานเต็มรูปแบบ (ทั้ง server และ client
library) การทดสอบจริงกับ PostgreSQL จึงให้คุณค่ามากกว่าการเขียนทฤษฎีล้วน ๆ เพราะผู้เรียนจะได้เห็น
ผลลัพธ์จริง ข้อผิดพลาดจริง และวิธีแก้ปัญหาจริงที่เกิดขึ้นระหว่างการเชื่อมต่อ ในขณะที่ MySQL ซึ่งไม่มี
ในสภาพแวดล้อมนี้ ยังคงมีคุณค่าในการเรียนรู้สถาปัตยกรรม (pattern เดียวกันทุกประการ) แม้จะไม่สามารถ
พิสูจน์ด้วยการรันจริงได้ในบริบทนี้

---

## ขั้นตอนที่ 743: เตรียมฐานข้อมูล PostgreSQL จริงด้วย psql

### สร้างฐานข้อมูลและตารางตัวอย่าง

เราสร้างฐานข้อมูลชื่อ `cobolcourse` พร้อมตาราง `products` ที่จะใช้ทดสอบตลอด Part นี้:

```bash
export PGPASSWORD=cobol123
psql -h 127.0.0.1 -U postgres -c "CREATE DATABASE cobolcourse;"
psql -h 127.0.0.1 -U postgres -d cobolcourse -c "
CREATE TABLE products (
    product_id   INTEGER PRIMARY KEY,
    product_name VARCHAR(30) NOT NULL,
    price        NUMERIC(9,2) NOT NULL,
    qty_on_hand  INTEGER NOT NULL
);
INSERT INTO products VALUES
  (1, 'RICE BAG 5KG', 350.00, 120),
  (2, 'COOKING OIL 1L', 95.50, 300),
  (3, 'INSTANT NOODLE', 6.00, 5000);
"
```

**ผลลัพธ์จริง:**

```
CREATE DATABASE
CREATE TABLE
INSERT 0 3
```

### ตรวจสอบข้อมูลด้วย psql

```bash
psql -h 127.0.0.1 -U postgres -d cobolcourse -t -A -F'|' \
    -c "SELECT product_id, product_name, price, qty_on_hand FROM products ORDER BY product_id;"
```

**ผลลัพธ์จริง:**

```
1|RICE BAG 5KG|350.00|120
2|COOKING OIL 1L|95.50|300
3|INSTANT NOODLE|6.00|5000
```

### อธิบายตัวเลือกของ psql ที่สำคัญสำหรับ Part นี้

- `-t` (tuples only): ตัดหัวตารางและเส้นแบ่งออก แสดงเฉพาะข้อมูลดิบ
- `-A` (unaligned): ปิดการจัดคอลัมน์ให้ตรงกันสวยงาม (ซึ่งเติมช่องว่างจำนวนไม่แน่นอน) ทำให้ได้ข้อมูล
  ที่มีรูปแบบสม่ำเสมอ เหมาะสำหรับให้โปรแกรมอื่นอ่านต่อ (machine-readable)
- `-F'|'` (field separator): กำหนดตัวคั่นระหว่างคอลัมน์เป็นเครื่องหมาย pipe แทนที่จะเป็น tab
  ค่าเริ่มต้น — ทำให้ผลลัพธ์อยู่ในรูปแบบเดียวกับที่เราออกแบบไว้ใน Part 073-074 (คั่นด้วย `|`) ทำให้
  โปรแกรม COBOL แยกวิเคราะห์ (parse) ได้ง่ายด้วย `UNSTRING DELIMITED BY "|"` ที่คุ้นเคยอยู่แล้ว

ทั้งสามตัวเลือกนี้คือหัวใจสำคัญที่ทำให้ `psql` ทำหน้าที่เป็น **shim** ที่ส่งออกข้อมูลในรูปแบบที่
COBOL ประมวลผลต่อได้ง่ายที่สุด แทนที่จะเป็นตารางสวยงามสำหรับมนุษย์อ่านเพียงอย่างเดียว

### ข้อควรระวัง

- `PGPASSWORD` เป็น environment variable มาตรฐานที่ `psql` และเครื่องมือตระกูล PostgreSQL client
  อ่านค่ารหัสผ่านจากตรงนี้โดยอัตโนมัติ สะดวกสำหรับการสาธิต แต่**ไม่ปลอดภัยสำหรับระบบจริง** เพราะ
  environment variable อาจรั่วไหลผ่านการดู process list ของระบบปฏิบัติการได้ในบางกรณี (รายละเอียด
  เพิ่มเติมในขั้นตอนที่ 747)
- ต้องระบุ `-h 127.0.0.1` เสมอเมื่อต้องการเชื่อมต่อผ่าน TCP/IP พร้อมรหัสผ่าน หากไม่ระบุ host
  `psql` อาจพยายามเชื่อมต่อผ่าน Unix domain socket แทน ซึ่งอาจมีการตั้งค่า authentication ต่างกัน

### แบบฝึกหัดที่ 743.1

**โจทย์**: จงเขียนคำสั่ง `psql` ที่ export ข้อมูลเฉพาะสินค้าที่มี `qty_on_hand` มากกว่า 200 ชิ้น
ในรูปแบบคั่นด้วย `|` เหมือนตัวอย่างข้างต้น

**เฉลย**:

```bash
psql -h 127.0.0.1 -U postgres -d cobolcourse -t -A -F'|' \
    -c "SELECT product_id, product_name, price, qty_on_hand FROM products WHERE qty_on_hand > 200 ORDER BY product_id;"
```

ผลลัพธ์ที่คาดว่าจะได้ (อ้างอิงจากข้อมูลที่สร้างไว้): `2|COOKING OIL 1L|95.50|300` และ
`3|INSTANT NOODLE|6.00|5000`

---

## ขั้นตอนที่ 744: กลยุทธ์ Shim แบบที่ 1 — CALL "SYSTEM" เรียก psql CLI

### แนวคิด

วิธีที่เรียบง่ายที่สุดในการให้ COBOL คุยกับฐานข้อมูลคือใช้คำสั่ง **`CALL "SYSTEM"`** ที่ GnuCOBOL
มีในตัว เพื่อสั่งให้ระบบปฏิบัติการรันคำสั่ง shell ใด ๆ ก็ได้ — ในที่นี้คือรัน `psql` พร้อม query
ที่ต้องการ แล้ว**redirect ผลลัพธ์ลงไฟล์ข้อความธรรมดา** จากนั้น COBOL program จะอ่านไฟล์นั้นกลับมา
ด้วยเทคนิคการอ่านไฟล์ Sequential ที่เรียนไปแล้วตั้งแต่ Part 023-025

```
[ COBOL: STRING สร้างคำสั่ง shell ] --CALL "SYSTEM"--> [ psql รัน query, เขียนผลลง DBOUT.TXT ]
                                                                    |
[ COBOL: OPEN/READ ไฟล์ DBOUT.TXT ด้วยเทคนิค Part 023-025 ] <-------+
```

จุดเด่นของแนวทางนี้คือ **ไม่ต้องคอมไพล์อะไรเพิ่มเติม ไม่ต้อง link library ใด ๆ** ใช้ความสามารถ
มาตรฐานของ COBOL ล้วน ๆ ร่วมกับเครื่องมือ command-line ที่มีอยู่แล้วในระบบ

### โค้ดโปรแกรม DBQUERY

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DBQUERY.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT RESULT-FILE ASSIGN TO "DBOUT.TXT"
               ORGANIZATION IS LINE SEQUENTIAL.
       DATA DIVISION.
       FILE SECTION.
       FD  RESULT-FILE.
       01  RESULT-LINE          PIC X(100).
       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG          PIC X VALUE "N".
           88  END-OF-FILE      VALUE "Y".
       01  WS-SHELL-CMD         PIC X(250).
       01  WS-PART.
           05 WS-ID             PIC X(10).
           05 WS-NAME           PIC X(30).
           05 WS-PRICE          PIC X(15).
           05 WS-QTY            PIC X(10).
       01  WS-TOTAL-VALUE       PIC 9(9)V99 VALUE 0.
       01  WS-LINE-VALUE        PIC 9(9)V99.
       01  WS-TOTAL-EDIT        PIC Z(8)9.99.
       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Step 1: shim out to the psql CLI client (the real SQL
      *> engine, since GnuCOBOL has no native PostgreSQL driver)
      *> and redirect its query result into a plain text file.
           STRING
             "PGPASSWORD=cobol123 psql -h 127.0.0.1 -U postgres "
             DELIMITED BY SIZE
             "-d cobolcourse -t -A -F'|' -c "
             DELIMITED BY SIZE
             '"SELECT product_id, product_name, price, '
             DELIMITED BY SIZE
             'qty_on_hand FROM products ORDER BY product_id" '
             DELIMITED BY SIZE
             "> DBOUT.TXT"
             DELIMITED BY SIZE
             INTO WS-SHELL-CMD
           CALL "SYSTEM" USING WS-SHELL-CMD

      *> Step 2: read the shim's output back with plain
      *> sequential file processing, exactly as taught in
      *> Part 023-025.
           OPEN INPUT RESULT-FILE
           DISPLAY "PRODUCT REPORT (from PostgreSQL via psql shim)"
           DISPLAY "------------------------------------------------"
           PERFORM UNTIL END-OF-FILE
               READ RESULT-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       UNSTRING RESULT-LINE DELIMITED BY "|"
                           INTO WS-ID WS-NAME WS-PRICE WS-QTY
                       COMPUTE WS-LINE-VALUE ROUNDED =
                           FUNCTION NUMVAL(WS-PRICE) *
                           FUNCTION NUMVAL(WS-QTY)
                       ADD WS-LINE-VALUE TO WS-TOTAL-VALUE
                       DISPLAY FUNCTION TRIM(WS-ID) ": "
                           FUNCTION TRIM(WS-NAME) " x "
                           FUNCTION TRIM(WS-QTY) " @ "
                           FUNCTION TRIM(WS-PRICE)
               END-READ
           END-PERFORM
           CLOSE RESULT-FILE
           MOVE WS-TOTAL-VALUE TO WS-TOTAL-EDIT
           DISPLAY "------------------------------------------------"
           DISPLAY "Total inventory value: "
               FUNCTION TRIM(WS-TOTAL-EDIT)
           STOP RUN.
```

### อธิบายทีละส่วน

- `STRING ... INTO WS-SHELL-CMD`: ประกอบคำสั่ง shell ทั้งหมดเป็นข้อความเดียวก่อนส่งให้
  `CALL "SYSTEM"` — ใช้เทคนิคผสมเครื่องหมายคำพูดเดี่ยวกับคู่แบบเดียวกับที่เรียนไปแล้วใน Part 045
  ขั้นตอนที่ 443 (สร้าง JSON ด้วย `STRING`) เพื่อแทรกเครื่องหมายคำพูดคู่ (`"`) เข้าไปในคำสั่ง SQL
  ที่อยู่ภายในคำสั่ง shell อีกชั้นหนึ่ง
- `CALL "SYSTEM" USING WS-SHELL-CMD`: สั่งให้ระบบปฏิบัติการรันคำสั่งใน `WS-SHELL-CMD` ราวกับพิมพ์
  ใน terminal เอง — เทียบเท่ากับฟังก์ชัน `system()` ของภาษา C ที่ COBOL standard กำหนดให้รองรับ
- ส่วนอ่านไฟล์ผลลัพธ์กลับ (`OPEN INPUT`, `PERFORM UNTIL END-OF-FILE`, `READ ... AT END`) เป็น
  Pattern เดียวกันเป๊ะกับที่เรียนไปแล้วใน Part 023 ขั้นตอนที่ 226 — พิสูจน์ว่าเมื่อ shim ทำหน้าที่
  แปลงข้อมูลฐานข้อมูลให้เป็นไฟล์ข้อความธรรมดาแล้ว COBOL ก็ใช้ทักษะพื้นฐานที่เรียนมาตั้งแต่ต้น
  หลักสูตรจัดการต่อได้ทันทีโดยไม่ต้องเรียนรู้อะไรใหม่เลย

### คอมไพล์และรันจริง

```bash
cobc -x -o dbquery dbquery.cob
./dbquery
```

**ผลลัพธ์จริงที่ได้:**

```
PRODUCT REPORT (from PostgreSQL via psql shim)
------------------------------------------------
1: RICE BAG 5KG x 120 @ 350.00
2: COOKING OIL 1L x 300 @ 95.50
3: INSTANT NOODLE x 5000 @ 6.00
------------------------------------------------
Total inventory value: 100650.00
```

ตรวจทานผลรวม: 350.00×120 + 95.50×300 + 6.00×5000 = 42000.00 + 28650.00 + 30000.00 =
**100650.00** ตรงกันทุกประการ

### ข้อควรระวัง

- **สำคัญมาก**: บรรทัด `STRING` ในโค้ดต้นฉบับต้องไม่เกินคอลัมน์ 72 (ทบทวนกฎเหล็กจาก
  `docs/COURSE-OUTLINE.md`) — ระหว่างพัฒนา Part นี้พบข้อผิดพลาดจริงจากบรรทัด `DISPLAY` หนึ่งบรรทัด
  ที่ยาวเกิน 72 คอลัมน์ไปเพียง 1 ตัวอักษร ทำให้เกิด error สับสนที่ตำแหน่งไกลออกไปมาก
  (`unexpected STOP, expecting LEADING or TRAILING`) — บทเรียนคือ**เมื่อเจอ syntax error ที่ดูไม่
  สัมพันธ์กับโค้ดตรงจุดนั้นเลย ให้ตรวจสอบความยาวคอลัมน์ของบรรทัดก่อนหน้าเสมอ**
- `CALL "SYSTEM"` ไม่มีกลไกตรวจสอบว่าคำสั่งที่รันสำเร็จหรือล้มเหลวโดยอัตโนมัติในตัวอย่างนี้ ระบบจริง
  ควรตรวจสอบ exit code ที่ได้กลับมา (ผ่าน `RETURN-CODE` special register) ก่อนอ่านไฟล์ผลลัพธ์ต่อ

### แบบฝึกหัดที่ 744.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `-t -A -F'|'` ตอนเรียก `psql` (จากขั้นตอนที่ 743) จึงสำคัญต่อ
ความสำเร็จของโปรแกรม `DBQUERY` นี้

**เฉลย**: หากไม่ใช้ตัวเลือกเหล่านี้ `psql` จะพิมพ์ผลลัพธ์แบบตารางสวยงามสำหรับมนุษย์อ่าน (มีเส้นขอบ
ตาราง, หัวคอลัมน์, การจัดช่องว่างให้ตรงกัน) ซึ่งไม่ใช่รูปแบบที่ `UNSTRING DELIMITED BY "|"` ของ
COBOL คาดหวัง การมี `-t` (ตัดหัวตาราง), `-A` (ไม่จัดคอลัมน์), และ `-F'|'` (ใช้ `|` คั่น) ทำให้
ผลลัพธ์แต่ละบรรทัดมีรูปแบบสม่ำเสมอแน่นอน เช่น `1|RICE BAG 5KG|350.00|120` ที่ COBOL แยกวิเคราะห์
ด้วย `UNSTRING` ได้อย่างแม่นยำ หากไม่มีตัวเลือกเหล่านี้ โปรแกรม COBOL จะ parse ผลลัพธ์ผิดพลาดทันที

---

## ขั้นตอนที่ 745: เขียนข้อมูลกลับด้วย Dynamic SQL — โปรแกรม DBINSERT

### แนวคิด: COBOL สร้างคำสั่ง SQL แบบไดนามิกด้วย STRING

นอกจาก**อ่าน**ข้อมูลจากฐานข้อมูล เรายังสามารถให้ COBOL **เขียน**ข้อมูลใหม่ลงฐานข้อมูลได้ ด้วยการ
ประกอบคำสั่ง `INSERT INTO ...` เป็นข้อความด้วย `STRING` แล้วส่งให้ `psql` รันผ่าน `CALL "SYSTEM"`
เช่นเดียวกับขั้นตอนที่แล้ว

### โค้ดโปรแกรม DBINSERT

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DBINSERT.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NEW-ID       PIC 9(5) VALUE 4.
       01  WS-NEW-ID-EDIT  PIC Z(4)9.
       01  WS-NEW-NAME     PIC X(20) VALUE "SOAP BAR".
       01  WS-NEW-PRICE    PIC 9(5)V99 VALUE 25.00.
       01  WS-NEW-PRICE-EDIT PIC Z(3)9.99.
       01  WS-SQL-CMD      PIC X(250).
       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE WS-NEW-ID TO WS-NEW-ID-EDIT
           MOVE WS-NEW-PRICE TO WS-NEW-PRICE-EDIT
           STRING
             "PGPASSWORD=cobol123 psql -h 127.0.0.1 -U postgres "
             DELIMITED BY SIZE
             "-d cobolcourse -c " DELIMITED BY SIZE
             '"INSERT INTO products VALUES (' DELIMITED BY SIZE
             FUNCTION TRIM(WS-NEW-ID-EDIT) DELIMITED BY SIZE
             ", '" DELIMITED BY SIZE
             FUNCTION TRIM(WS-NEW-NAME) DELIMITED BY SIZE
             "', " DELIMITED BY SIZE
             FUNCTION TRIM(WS-NEW-PRICE-EDIT) DELIMITED BY SIZE
             ', 60)"' DELIMITED BY SIZE
             INTO WS-SQL-CMD
           DISPLAY "Running: " FUNCTION TRIM(WS-SQL-CMD)
           CALL "SYSTEM" USING WS-SQL-CMD
           STOP RUN.
```

### คอมไพล์และรันจริง

```bash
cobc -x -o dbinsert dbinsert.cob
./dbinsert
```

**ผลลัพธ์จริงที่ได้:**

```
Running: PGPASSWORD=cobol123 psql -h 127.0.0.1 -U postgres -d cobolcourse -c "INSERT INTO products VALUES (4, 'SOAP BAR', 25.00, 60)"
INSERT 0 1
```

ตรวจสอบด้วย `psql` ว่าข้อมูลถูกเพิ่มจริง:

```bash
psql -h 127.0.0.1 -U postgres -d cobolcourse -c "SELECT * FROM products ORDER BY product_id;"
```

**ผลลัพธ์จริง:**

```
 product_id |  product_name  | price  | qty_on_hand
------------+----------------+--------+-------------
          1 | RICE BAG 5KG   | 350.00 |         120
          2 | COOKING OIL 1L |  95.50 |         300
          3 | INSTANT NOODLE |   6.00 |        5000
          4 | SOAP BAR       |  25.00 |          60
(4 rows)
```

record ใหม่ (`SOAP BAR`) ถูกเพิ่มเข้าไปในฐานข้อมูลจริงสำเร็จ — พิสูจน์ว่า COBOL program สามารถ
**เขียน**ข้อมูลลง PostgreSQL ได้จริง ไม่ใช่แค่**อ่าน**อย่างเดียว

### อธิบายจุดสำคัญ

- โปรแกรมนี้แสดง `DISPLAY "Running: " ...` คำสั่งที่กำลังจะรันก่อนเสมอ — เทคนิคนี้มีประโยชน์มากตอน
  debug เพราะทำให้เห็นชัดเจนว่าคำสั่ง SQL ที่ COBOL ประกอบขึ้นมาจริง ๆ หน้าตาเป็นอย่างไร ก่อนที่จะถูก
  ส่งไปรันจริง
- ตัวเลขจำนวนที่นำมาจาก `PIC Z(4)9`/`PIC Z(3)9.99` ต้องผ่าน `FUNCTION TRIM` ก่อนนำไปต่อ `STRING`
  เสมอ (หลักการเดียวกับที่เรียนซ้ำหลายครั้งตั้งแต่ Part 045) เพื่อไม่ให้มีช่องว่างปะปนอยู่กลางคำสั่ง
  SQL ที่จะทำให้ค่าตัวเลขผิดรูปแบบ

### ข้อควรระวัง

- **ข้อควรระวังที่สำคัญที่สุดของขั้นตอนนี้จะอธิบายเต็มรูปแบบในขั้นตอนที่ 747**: การประกอบคำสั่ง SQL
  ด้วยการต่อข้อความ (string concatenation) แบบนี้มีความเสี่ยงต่อ **SQL Injection** อย่างมากหากค่า
  ที่นำมาต่อมาจาก input ภายนอกที่ไม่ได้ตรวจสอบ (เช่น มาจากผู้ใช้เว็บโดยตรง) ตัวอย่างนี้ปลอดภัยเพราะ
  ค่าทั้งหมดเป็นค่าคงที่ (`VALUE`) ที่กำหนดไว้ในโปรแกรมเอง ไม่ได้มาจากภายนอก
- ต้องแน่ใจว่า `WS-SQL-CMD PIC X(250)` มีความกว้างเพียงพอสำหรับคำสั่ง SQL ที่ยาวที่สุดที่อาจเกิดขึ้น
  จริง มิฉะนั้น `STRING` จะตัดคำสั่งทิ้งอย่างเงียบ ๆ เหมือนปัญหาที่พิสูจน์ไว้แล้วใน Part 019/045

### แบบฝึกหัดที่ 745.1

**โจทย์**: จงปรับโปรแกรม `DBINSERT` ให้เพิ่มสินค้ารายการที่ 5 (`SHAMPOO 200ML`, ราคา `89.00`,
จำนวน `150`) แทนที่ `SOAP BAR` เดิม

**เฉลยแนวทาง**: เปลี่ยนค่า `VALUE` ของทั้ง 4 ตัวแปร (`WS-NEW-ID` เป็น 5, `WS-NEW-NAME` เป็น
`"SHAMPOO 200ML"`, `WS-NEW-PRICE` เป็น `89.00`) แล้วแก้ตัวเลข `60` ในส่วนท้ายของ `STRING`
(จำนวนคงเหลือ) เป็น `150` จากนั้นคอมไพล์และรันใหม่ ผลลัพธ์คำสั่ง SQL ที่คาดว่าจะได้คือ
`INSERT INTO products VALUES (5, 'SHAMPOO 200ML', 89.00, 150)`

---

## ขั้นตอนที่ 746: กลยุทธ์ Shim แบบที่ 2 — CALL โปรแกรม C ที่ Link กับ libpq โดยตรง

### ทำไมต้องมีอีกแนวทางหนึ่ง

การเรียก `psql` ผ่าน `CALL "SYSTEM"` (ขั้นตอนที่ 744-745) ใช้งานง่ายแต่มีต้นทุนสูง: ทุกครั้งที่
เรียกต้องเปิด process ใหม่ (`psql`), เชื่อมต่อฐานข้อมูลใหม่ทั้งหมด, และเขียน-อ่านผ่านไฟล์ชั่วคราว
สำหรับระบบที่ต้องการประสิทธิภาพสูงกว่านี้ แนวทางที่สองคือ**เขียนโปรแกรม C ที่ link กับ client
library ของฐานข้อมูลโดยตรง** (`libpq` สำหรับ PostgreSQL) แล้วให้ COBOL program `CALL` เข้าไปที่
ฟังก์ชันนั้นโดยตรง ผ่านเทคนิค COBOL-C Interoperability ที่เรียนไปแล้วใน **Part 072**

```
[ COBOL Program ] --CALL "pggetprice"--> [ C function ที่ link กับ libpq ] --SQL protocol--> [ PostgreSQL ]
     (ใน process เดียวกัน ไม่ต้องเปิด subprocess ใหม่ ไม่ต้องผ่านไฟล์ชั่วคราว)
```

### โค้ด Shim ภาษา C (pgshim.c)

```c
#include <libpq-fe.h>
#include <string.h>
#include <stdio.h>

/* Entry point callable from COBOL via CALL "pggetprice" USING ... */
void pggetprice(char *prod_id, char *out_name, char *out_price)
{
    char idbuf[16];
    char conninfo[256];
    int i;

    /* prod_id arrives as a fixed-width COBOL PIC X field (space
       padded); trim trailing spaces to build a clean C string. */
    memcpy(idbuf, prod_id, 5);
    idbuf[5] = '\0';
    for (i = 4; i >= 0 && idbuf[i] == ' '; i--) {
        idbuf[i] = '\0';
    }

    memset(out_name, ' ', 20);
    memset(out_price, ' ', 10);

    snprintf(conninfo, sizeof(conninfo),
        "host=127.0.0.1 user=postgres password=cobol123 "
        "dbname=cobolcourse");

    PGconn *conn = PQconnectdb(conninfo);
    if (PQstatus(conn) != CONNECTION_OK) {
        PQfinish(conn);
        memcpy(out_name, "CONNECTION ERROR", 17);
        return;
    }

    char query[128];
    snprintf(query, sizeof(query),
        "SELECT product_name, price FROM products "
        "WHERE product_id = %s", idbuf);

    PGresult *res = PQexec(conn, query);
    if (PQresultStatus(res) == PGRES_TUPLES_OK && PQntuples(res) > 0) {
        char *name = PQgetvalue(res, 0, 0);
        char *price = PQgetvalue(res, 0, 1);
        memcpy(out_name, name, strlen(name) < 20 ? strlen(name) : 20);
        memcpy(out_price, price, strlen(price) < 10 ? strlen(price) : 10);
    } else {
        memcpy(out_name, "NOT FOUND", 9);
    }
    PQclear(res);
    PQfinish(conn);
}
```

### โค้ดโปรแกรม COBOL ที่เรียก Shim นี้ (DBCALL)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DBCALL.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PROD-ID       PIC X(5) VALUE "2".
       01  WS-OUT-NAME      PIC X(20).
       01  WS-OUT-PRICE     PIC X(10).
       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Looking up product ID: "
               FUNCTION TRIM(WS-PROD-ID)
           CALL "pggetprice" USING WS-PROD-ID WS-OUT-NAME
               WS-OUT-PRICE
           DISPLAY "Name : " FUNCTION TRIM(WS-OUT-NAME)
           DISPLAY "Price: " FUNCTION TRIM(WS-OUT-PRICE)
           STOP RUN.
```

### คอมไพล์ ลิงก์ และรันจริง

การคอมไพล์แบบนี้ต้อง (1) คอมไพล์ไฟล์ C ให้เป็น object file ก่อน (2) คอมไพล์ COBOL แล้วลิงก์
(link) เข้ากับทั้ง object file ของ C และ library `libpq`:

```bash
gcc -c -I$(pg_config --includedir) pgshim.c -o pgshim.o
cobc -x dbcall.cob pgshim.o -I$(pg_config --includedir) \
    -L$(pg_config --libdir) -lpq -o dbcall
./dbcall
```

**ผลลัพธ์จริงที่ได้:**

```
Looking up product ID: 2
Name : COOKING OIL 1L
Price: 95.50
```

ทดสอบกรณีไม่พบสินค้า (product ID `999`):

**ผลลัพธ์จริง:**

```
Looking up product ID: 999
Name : NOT FOUND
Price:
```

### อธิบายจุดสำคัญ

- `CALL "pggetprice" USING WS-PROD-ID WS-OUT-NAME WS-OUT-PRICE`: GnuCOBOL เรียกฟังก์ชัน C ชื่อ
  `pggetprice` โดยตรง ส่ง parameter แบบ `BY REFERENCE` (ค่าเริ่มต้นของ `CALL`, ทบทวนจาก Part 032)
  ทำให้ C function เข้าถึงและแก้ไข memory ของตัวแปร COBOL ได้โดยตรงผ่าน pointer — เทคนิคพื้นฐาน
  เดียวกับที่เรียนใน Part 072 เรื่องการเชื่อม COBOL กับ C
- `pg_config --includedir` และ `pg_config --libdir`: คำสั่งมาตรฐานของ PostgreSQL ที่คืนค่า path
  ของ header files (`libpq-fe.h`) และ library files (`libpq.so`) ตามลำดับ ช่วยให้ไม่ต้อง hardcode
  path เหล่านี้เอง
- **ข้อดีสำคัญของแนวทางนี้เทียบกับ `CALL "SYSTEM"`**: ทำงานอยู่ใน process เดียวกันตลอด ไม่ต้องเปิด
  subprocess ใหม่ทุกครั้ง ไม่ต้องเขียน-อ่านไฟล์ชั่วคราว จึงเร็วกว่ามากสำหรับการเรียกซ้ำหลายครั้ง
  ติดต่อกัน (แม้ในตัวอย่างนี้แต่ละครั้งที่เรียกยังคงเปิด connection ใหม่ผ่าน `PQconnectdb` — ระบบ
  จริงควรทำ Connection Pooling เพื่อลดต้นทุนนี้ต่อไปอีก)

### ข้อควรระวัง

- โค้ด C ตัวอย่างนี้ทำ **memory safety** เบื้องต้น (`strlen(name) < 20 ? ... : 20` เพื่อไม่ให้เขียน
  เกิน buffer ที่ COBOL เตรียมไว้) แต่ยังคงเรียบง่ายเพื่อการสาธิต ระบบจริงควรตรวจสอบข้อผิดพลาดและ
  ขอบเขตหน่วยความจำอย่างละเอียดกว่านี้มาก เพราะการเขียนเกิน buffer ในภาษา C คือช่องโหว่ความปลอดภัย
  ร้ายแรง (Buffer Overflow)
- การ hardcode รหัสผ่าน (`password=cobol123`) ไว้ในโค้ด C ตรง ๆ แบบนี้**ไม่ปลอดภัยสำหรับระบบจริง**
  เช่นเดียวกับที่เตือนไว้ในขั้นตอนก่อนหน้า จะอธิบายทางเลือกที่ปลอดภัยกว่าในขั้นตอนที่ 747

### แบบฝึกหัดที่ 746.1

**โจทย์**: จงอธิบายว่าทำไมแนวทาง "CALL โปรแกรม C ที่ link กับ libpq" จึงเร็วกว่าแนวทาง
"CALL SYSTEM เรียก psql" โดยเฉพาะเมื่อต้องเรียกค้นหาข้อมูลซ้ำหลายครั้งติดต่อกัน

**เฉลย**: แนวทาง `CALL "SYSTEM"` ต้องสร้าง process ใหม่ทั้งหมด (`psql`) ทุกครั้งที่เรียก ซึ่งรวมถึง
การโหลดโปรแกรม, เชื่อมต่อฐานข้อมูลใหม่ตั้งแต่ต้น (TCP handshake, authentication), รันคำสั่ง,
เขียนผลลัพธ์ลงไฟล์, แล้วปิดการเชื่อมต่อและ process ทิ้ง — ทำซ้ำขั้นตอนทั้งหมดนี้ทุกครั้ง ในขณะที่
แนวทาง `CALL` โปรแกรม C ทำงานอยู่ใน process COBOL เดียวกันตลอด ไม่มีต้นทุนการสร้าง process ใหม่
และหากออกแบบให้เปิด connection ค้างไว้ (Connection Pooling หรือเปิดครั้งเดียวตอนเริ่มโปรแกรม) ก็จะ
ไม่ต้องเสียเวลาเชื่อมต่อใหม่ทุกครั้งด้วย ทำให้การเรียกซ้ำหลายครั้งเร็วกว่าอย่างมีนัยสำคัญ

---

## ขั้นตอนที่ 747: ความปลอดภัย — SQL Injection และการจัดการรหัสผ่าน

### ปัญหา SQL Injection ใน Dynamic SQL

ในขั้นตอนที่ 745 เราสร้างคำสั่ง `INSERT` ด้วยการต่อข้อความ (`STRING`) โดยตรง หากค่าที่นำมาต่อ
**มาจาก input ภายนอกที่ไม่ผ่านการตรวจสอบ** (เช่น ชื่อสินค้าที่ผู้ใช้กรอกเข้ามาเอง) จะเกิดช่องโหว่
ร้ายแรงที่เรียกว่า **SQL Injection** ตัวอย่างเช่น หากผู้ใช้กรอกชื่อสินค้าเป็น:

```
SOAP'); DROP TABLE products; --
```

แล้วโค้ด COBOL นำค่านี้ไปต่อ `STRING` ตรง ๆ โดยไม่ตรวจสอบ คำสั่ง SQL ที่ได้จะกลายเป็น:

```sql
INSERT INTO products VALUES (4, 'SOAP'); DROP TABLE products; --', 25.00, 60)
```

ซึ่งเมื่อรันจริงจะ**ลบตาราง `products` ทั้งหมดทิ้ง** — ความเสียหายร้ายแรงที่เกิดจากการไม่ตรวจสอบ
input ก่อนนำไปประกอบเป็นคำสั่ง SQL

### แนวทางป้องกันที่ควรทำในระบบจริง

1. **ตรวจสอบและกรองอักขระอันตรายก่อนเสมอ**: ตรวจสอบว่าค่าที่รับเข้ามาไม่มีเครื่องหมายคำพูดเดี่ยว
   (`'`), เซมิโคลอน (`;`), หรือ comment marker ของ SQL (`--`) ปะปนอยู่ ก่อนนำไปต่อเป็นคำสั่ง SQL
   (คล้ายเทคนิคการตรวจสอบ input ที่เรียนไปแล้วใน Part 073 ขั้นตอนที่ 728 แต่ใช้กับบริบท SQL แทน
   HTTP request)
2. **จำกัดความกว้างของ field และตรวจสอบชนิดข้อมูลอย่างเข้มงวด**: เนื่องจาก COBOL ใช้ `PIC` ที่
   กำหนดความกว้างตายตัวอยู่แล้ว (ทบทวนจาก Part 006) นี่คือข้อได้เปรียบตามธรรมชาติของ COBOL ที่ช่วย
   ลดความเสี่ยงบางส่วนได้ เมื่อเทียบกับภาษาที่ field ความยาวไม่จำกัด
3. **ใช้ Prepared Statement เมื่อทำได้**: ในแนวทาง Shim แบบ C (ขั้นตอนที่ 746) library `libpq`
   รองรับฟังก์ชัน `PQexecParams` ที่แยกค่าพารามิเตอร์ออกจากคำสั่ง SQL อย่างชัดเจน (คล้าย placeholder
   `?` หรือ `$1` แทนการต่อข้อความ) ทำให้ปลอดภัยจาก SQL Injection โดยธรรมชาติ ควรใช้แทนการต่อ string
   เสมอเมื่อค่าที่ใช้มาจากภายนอก

### การจัดการรหัสผ่านฐานข้อมูลอย่างปลอดภัย

ตัวอย่างทั้งหมดใน Part นี้ใช้ `PGPASSWORD=cobol123` หรือ hardcode รหัสผ่านในโค้ด C ตรง ๆ **เพื่อ
ความง่ายในการสาธิตเท่านั้น** ระบบจริงควรใช้แนวทางที่ปลอดภัยกว่า ได้แก่:

- **ไฟล์ `.pgpass`**: PostgreSQL รองรับไฟล์ `~/.pgpass` ที่เก็บรหัสผ่านแยกจากคำสั่งหรือโค้ด โดยมี
  การจำกัดสิทธิ์การอ่านไฟล์ (`chmod 600`) ให้เฉพาะเจ้าของไฟล์เท่านั้น
- **Connection Service File** (`~/.pg_service.conf`): กำหนดค่าการเชื่อมต่อทั้งหมด (host, user,
  password, database) ไว้เป็นชื่อ service เดียว แล้วอ้างอิงแค่ชื่อ service ในโค้ด ไม่ต้องฝังค่า
  ที่ละเอียดอ่อนไว้ในโค้ดหรือ command เลย
- **Secret Management System** (เช่น HashiCorp Vault, AWS Secrets Manager): สำหรับระบบองค์กร
  ขนาดใหญ่ ดึงรหัสผ่านมาใช้แบบ dynamic ตอน runtime แทนการเก็บไว้แบบ static ที่ใดที่หนึ่ง

### ข้อควรระวัง

- **ห้าม commit รหัสผ่านจริงเข้า version control (Git) เด็ดขาด** แม้จะเป็นรหัสผ่านของสภาพแวดล้อม
  ทดสอบก็ตาม ควรฝึกนิสัยที่ถูกต้องตั้งแต่เนิ่น ๆ (เนื้อหาเรื่อง Git และ Version Control workflow
  ที่เหมาะสมสำหรับทีม COBOL จะสอนเต็มรูปแบบใน Part 080)
- แนวทาง `CALL "SYSTEM"` ที่ต่อ environment variable (`PGPASSWORD=...`) เข้ากับคำสั่งโดยตรงตามที่
  สาธิตใน Part นี้ มีความเสี่ยงเพิ่มเติมที่รหัสผ่านอาจปรากฏใน process list ของระบบปฏิบัติการชั่วขณะ
  หนึ่งระหว่างที่ `psql` กำลังทำงาน (ตรวจสอบได้ด้วยคำสั่ง `ps aux` จากผู้ใช้อื่นในเครื่องเดียวกัน)
  ระบบจริงควรหลีกเลี่ยงด้วยการใช้ `.pgpass` แทน

### แบบฝึกหัดที่ 747.1

**โจทย์**: จงอธิบายว่าทำไมการที่ COBOL `PIC X(20)` มีความกว้างตายตัวจึงช่วยลดความเสี่ยง SQL
Injection ได้บางส่วน แต่ไม่ได้ป้องกันได้ทั้งหมด

**เฉลย**: ความกว้างตายตัวของ `PIC X(20)` จำกัดว่าค่าที่รับเข้ามาต่อ field นั้นจะมีความยาวไม่เกิน
20 ตัวอักษรเสมอ (input ที่ยาวเกินจะถูกตัดทิ้งอัตโนมัติ) ทำให้การโจมตีที่ต้องใช้ payload ยาว ๆ
(เช่นคำสั่ง SQL ซับซ้อนหลายคำสั่ง) ทำได้ยากขึ้นในระดับหนึ่ง แต่**ไม่ได้ป้องกันได้ทั้งหมด** เพราะ
การโจมตีด้วยข้อความสั้น ๆ ก็เพียงพอที่จะสร้างความเสียหายได้ เช่นตัวอย่าง `SOAP'); DROP TABLE
products; --` ที่ยกมาข้างต้นมีความยาวไม่ถึง 20 ตัวอักษรด้วยซ้ำ ความกว้างที่จำกัดจึงเป็นเพียงเกราะ
ป้องกันชั้นหนึ่งเท่านั้น ไม่ใช่การป้องกัน SQL Injection ที่สมบูรณ์ — ต้องอาศัยการตรวจสอบเนื้อหา
(input validation) และ/หรือ Prepared Statement ควบคู่กันเสมอ

---

## ขั้นตอนที่ 748: เปรียบเทียบสองแนวทาง Shim และเลือกใช้ให้เหมาะสม

### ตารางเปรียบเทียบ CALL "SYSTEM" กับ CALL โปรแกรม C

| ประเด็น | CALL "SYSTEM" + psql CLI | CALL โปรแกรม C + libpq |
|---|---|---|
| ความยากในการเขียน | ง่ายมาก ไม่ต้องคอมไพล์ภาษาอื่น | ต้องเขียนและคอมไพล์ C เพิ่ม, ต้อง link library |
| ประสิทธิภาพ | ช้ากว่า (เปิด process ใหม่ทุกครั้ง) | เร็วกว่ามาก (อยู่ใน process เดียวกัน) |
| การจัดการ error | ต้องตรวจสอบผ่านไฟล์/exit code ทางอ้อม | ตรวจสอบได้ตรงจุดผ่าน return value ของ C function |
| ความเสี่ยง memory safety | ต่ำ (ไม่มีการจัดการ pointer โดยตรง) | สูงกว่า (ต้องระวัง buffer overflow ในโค้ด C) |
| เหมาะกับ | Batch job, รายงาน, งานที่ไม่ถี่มาก | ระบบ online ที่ต้องการ response เร็ว, งานที่เรียกถี่ |
| Dependency ตอน build | แค่มี `psql` ติดตั้งในเครื่อง | ต้องมี compiler C, header files, และ library เชื่อมโยง |

### แนวทางที่ 3 (แนวคิดเพื่อการศึกษาต่อยอด): Shim ด้วย Python ผ่าน subprocess

นอกจากสองแนวทางที่ทดสอบจริงข้างต้น ยังมีอีกแนวทางหนึ่งที่รวมข้อดีบางส่วนของทั้งสองแบบ: เขียน shim
เป็น Python script ที่ใช้ library `psycopg2` หรือ `pg8000` (ต้องติดตั้งเพิ่มเติมผ่าน `pip` ซึ่ง
ในสภาพแวดล้อมของหลักสูตรนี้ยังไม่ได้ติดตั้งไว้ จึงไม่ได้ทดสอบแนวทางนี้แบบเต็มรูปแบบ) แล้วให้ COBOL
`CALL "SYSTEM"` เรียก Python script นั้นแทน `psql` โดยตรง ข้อดีคือ Python เขียน logic การจัดการ
ข้อมูลที่ซับซ้อนได้ง่ายกว่าการเขียน SQL query ตรง ๆ ผ่าน `psql -c` แต่ยังคงมีข้อจำกัดด้านประสิทธิภาพ
เหมือนแนวทาง `CALL "SYSTEM"` เนื่องจากยังต้องเปิด process ใหม่ทุกครั้งอยู่ดี

### เกณฑ์การเลือกใช้ในทางปฏิบัติ

- เลือก **CALL "SYSTEM" + CLI client** เมื่อ: งาน batch ที่รันไม่บ่อย (เช่น รายงานสิ้นวัน), ต้องการ
  ความเรียบง่ายในการ maintain, ทีมพัฒนาไม่ถนัด C
- เลือก **CALL โปรแกรม C + native library** เมื่อ: ต้องการประสิทธิภาพสูง, ระบบ online ที่ต้องตอบ
  สนองเร็ว, มีทีมที่ถนัด C และเข้าใจเรื่อง memory management ดีพอ

### ข้อควรระวัง

- ไม่ว่าจะเลือกแนวทางใด **หลักการพื้นฐานเหมือนกันเสมอ**: COBOL program ไม่ควรรู้จักรายละเอียดของ
  SQL wire protocol เลย ให้ shim เป็นผู้รับผิดชอบส่วนนั้นทั้งหมด นี่คือแก่นสำคัญของ Shim Program
  Pattern ที่ Part นี้ต้องการสอน ไม่ใช่การท่องจำวิธีคอมไพล์หรือ syntax ของแต่ละแนวทาง

### แบบฝึกหัดที่ 748.1

**โจทย์**: จงพิจารณาระบบ "ตรวจสอบยอดเงินคงเหลือแบบ real-time สำหรับตู้ ATM" กับระบบ "สร้างรายงาน
ยอดขายประจำเดือนส่งอีเมลอัตโนมัติ" ว่าแต่ละระบบควรเลือกใช้แนวทาง Shim แบบใด พร้อมเหตุผล

**เฉลยแนวทาง**: ระบบตู้ ATM ควรเลือก **CALL โปรแกรม C + native library** เพราะต้องการความเร็วสูง
มาก (ผู้ใช้ตู้ ATM ไม่ควรต้องรอนานหลายวินาทีเพื่อดูยอดเงิน) และมักถูกเรียกใช้งานถี่มากตลอดทั้งวัน
ต้นทุนของการเปิด process ใหม่ทุกครั้งแบบ `CALL "SYSTEM"` จะสะสมกลายเป็นปัญหาประสิทธิภาพที่ชัดเจน
ในขณะที่ระบบรายงานยอดขายประจำเดือนควรเลือก **CALL "SYSTEM" + CLI client** เพราะรันเพียงครั้งเดียว
ต่อเดือน ไม่ต้องการความเร็วสูงมากนัก และความเรียบง่ายในการ maintain (ไม่ต้องดูแลโค้ด C เพิ่มเติม)
มีค่ามากกว่าประโยชน์ด้านประสิทธิภาพที่ได้ไม่คุ้มค่ากับความซับซ้อนที่เพิ่มขึ้น

---

## ขั้นตอนที่ 749: ตัวอย่างอ้างอิงสำหรับ MySQL (ยังไม่ได้ทดสอบในสภาพแวดล้อมนี้)

### ย้ำความซื่อตรงทางเทคนิคก่อนเริ่ม

ตามที่ตรวจสอบไว้แล้วในขั้นตอนที่ 742 **MySQL ไม่มีอยู่ในสภาพแวดล้อมของหลักสูตรนี้เลย** (ไม่มีทั้ง
`mysql` client และ `mysqld` server) เนื้อหาในขั้นตอนนี้จึงเป็น**ไวยากรณ์อ้างอิงที่ยังไม่ได้ทดสอบรัน
จริง** โดยอิงจากรูปแบบสถาปัตยกรรมเดียวกันทุกประการกับที่ทดสอบสำเร็จแล้วกับ PostgreSQL เพียงแต่
เปลี่ยน client tool และ syntax เล็กน้อยให้ตรงกับ MySQL — เช่นเดียวกับที่ Part 045 ทำกับไวยากรณ์
`JSON GENERATE`/`JSON PARSE` ที่ยืนยันไว้อย่างชัดเจนว่ายังไม่ได้ทดสอบรันในบิลด์ของหลักสูตร

### รูปแบบ Shim ด้วย mysql CLI (อ้างอิงเท่านั้น — ยังไม่ทดสอบ)

```
      *> REFERENCE SYNTAX ONLY - NOT TESTED IN THIS ENVIRONMENT.
      *> mysql client is not installed in this sandbox (verified
      *> in step 742). Pattern mirrors the tested PostgreSQL
      *> example in step 744, with mysql CLI syntax substituted.
       STRING
         "mysql -h 127.0.0.1 -u appuser -pSECRET "
         DELIMITED BY SIZE
         "cobolcourse -N -B -e "
         DELIMITED BY SIZE
         '"SELECT product_id, product_name, price '
         DELIMITED BY SIZE
         'FROM products ORDER BY product_id" '
         DELIMITED BY SIZE
         "> DBOUT.TXT"
         DELIMITED BY SIZE
         INTO WS-SHELL-CMD
       CALL "SYSTEM" USING WS-SHELL-CMD
```

### อธิบายความแตกต่างของตัวเลือกเมื่อเทียบกับ psql

| จุดประสงค์ | psql (PostgreSQL) | mysql (MySQL) |
|---|---|---|
| ตัดหัวตารางออก | `-t` | `-N` |
| ผลลัพธ์แบบไม่จัดคอลัมน์ | `-A` | `-B` (แสดงผลแบบ tab-separated โดยธรรมชาติ) |
| กำหนดตัวคั่นคอลัมน์เอง | `-F'|'` | ไม่มีตัวเลือกตรงตัว มักต้องใช้ `sed`/`tr` แปลง tab เป็น `|` เพิ่มเติม หรือปรับ `UNSTRING` ให้ใช้ TAB เป็น delimiter แทน |
| รันคำสั่ง SQL เดียว | `-c "SQL"` | `-e "SQL"` |
| ระบุรหัสผ่านผ่าน environment variable | `PGPASSWORD` | ไม่รองรับโดยตรง (MySQL แนะนำใช้ `--defaults-extra-file` แทนด้วยเหตุผลความปลอดภัย) |

### รูปแบบ Shim ด้วย C ที่ link กับ MySQL Client Library (อ้างอิงเท่านั้น — ยังไม่ทดสอบ)

```c
/* REFERENCE SYNTAX ONLY - NOT COMPILED OR TESTED IN THIS
   ENVIRONMENT (libmysqlclient-dev is not installed here,
   confirmed during development of this part). Shown to
   illustrate that the same CALL-based shim pattern from step
   746 applies to MySQL through its own client library
   (libmysqlclient) instead of libpq. */
#include <mysql/mysql.h>

void mysqlgetprice(char *prod_id, char *out_name, char *out_price)
{
    MYSQL *conn = mysql_init(NULL);
    mysql_real_connect(conn, "127.0.0.1", "appuser", "SECRET",
                        "cobolcourse", 0, NULL, 0);
    /* ... build query, mysql_query(conn, query), fetch a row via
       mysql_store_result()/mysql_fetch_row(), copy into out_name
       and out_price the same way step 746 does for PostgreSQL ... */
    mysql_close(conn);
}
```

### ข้อควรระวัง

- **ห้ามนำโค้ดในขั้นตอนนี้ไป compile ในสภาพแวดล้อมนี้โดยตรง** เพราะ `mysql` client และ
  `libmysqlclient-dev` ไม่มีติดตั้งอยู่ จะได้ error ประเภท "command not found" หรือ
  "libmysqlclient.h: No such file or directory" ทันที
- ก่อนนำไปใช้งานจริงกับ MySQL เสมอ ต้องทดสอบกับสภาพแวดล้อมที่มี MySQL ติดตั้งจริงก่อน และตรวจสอบ
  เวอร์ชันของ MySQL client ที่ใช้ เพราะตัวเลือก command-line อาจเปลี่ยนแปลงระหว่างเวอร์ชัน

### แบบฝึกหัดที่ 749.1

**โจทย์**: จงอธิบายว่าทำไมหลักสูตรนี้จึงยังคงสอนไวยากรณ์ของ MySQL Shim ทั้งที่ไม่สามารถทดสอบรันจริง
ในสภาพแวดล้อมของหลักสูตรได้ โดยเปรียบเทียบกับเหตุผลเดียวกันที่ Part 045 ให้ไว้เรื่อง JSON

**เฉลย**: เหตุผลเดียวกับที่ Part 045 อธิบายไว้: แม้จะทดสอบรันจริงไม่ได้ในสภาพแวดล้อมนี้ แต่ MySQL
เป็นฐานข้อมูลที่ใช้แพร่หลายมากในโลกจริง ผู้เรียนมีโอกาสสูงที่จะต้องทำงานกับระบบที่ใช้ MySQL ในอนาคต
การเข้าใจว่า**สถาปัตยกรรม Shim Program Pattern เดียวกันทุกประการ**สามารถนำไปปรับใช้กับ MySQL ได้
โดยเปลี่ยนแค่ client tool/library และ syntax เล็กน้อย มีคุณค่าทางการเรียนรู้มาก แม้จะไม่สามารถ
พิสูจน์ด้วยการรันจริงในบริบทนี้ก็ตาม หลักสูตรจึงเลือกที่จะสอนพร้อมระบุอย่างซื่อตรงชัดเจนว่าส่วนไหน
ทดสอบแล้วและส่วนไหนยังไม่ได้ทดสอบ แทนที่จะละเว้นไม่พูดถึงเลยหรือแอบอ้างว่าทดสอบแล้วทั้งที่ไม่จริง

---

## ขั้นตอนที่ 750: สรุปภาพรวม Shim Program Pattern และแบบฝึกหัดรวม

### สรุปสิ่งที่ทดสอบจริงทั้งหมดใน Part นี้

| โปรแกรม | หน้าที่ | ผลการทดสอบ |
|---|---|---|
| `DBQUERY.cob` | อ่านข้อมูลสินค้าทั้งหมดผ่าน psql shim, คำนวณมูลค่าคงคลังรวม | ทดสอบสำเร็จ ได้ผลลัพธ์ตรงตามการคำนวณด้วยมือ (100650.00) |
| `DBINSERT.cob` | เขียนข้อมูลสินค้าใหม่ผ่าน dynamic SQL + psql shim | ทดสอบสำเร็จ ตรวจสอบด้วย `psql` ยืนยันข้อมูลถูกเพิ่มจริง |
| `pgshim.c` + `DBCALL.cob` | ค้นหาสินค้ารายตัวผ่าน C shim ที่ link กับ libpq โดยตรง | ทดสอบสำเร็จทั้งกรณีพบและไม่พบข้อมูล |

### หลักการสำคัญที่ต้องจดจำจาก Part นี้

1. **GnuCOBOL ไม่มี native driver สำหรับ MySQL/PostgreSQL** — ต้องพึ่ง Shim Program เสมอ
2. **Shim ทำได้ 2 ระดับหลัก**: ผ่าน CLI client (`CALL "SYSTEM"`, ง่ายแต่ช้ากว่า) หรือผ่านโปรแกรม
   ที่ link กับ client library โดยตรง (`CALL`, เร็วกว่าแต่ซับซ้อนกว่า)
3. **ความปลอดภัยต้องมาก่อนเสมอ**: ระวัง SQL Injection เมื่อสร้างคำสั่ง SQL แบบไดนามิก และไม่เก็บ
   รหัสผ่านแบบ hardcode ในระบบจริง
4. **ความซื่อตรงทางเทคนิค**: ทดสอบทุกอย่างที่ทดสอบได้จริง (PostgreSQL) และระบุชัดเจนเมื่อมีข้อจำกัด
   ที่ทำให้ทดสอบไม่ได้ (MySQL) แทนที่จะกลบเกลื่อนหรือแอบอ้าง

### แบบฝึกหัดที่ 750.1 (แบบฝึกหัดรวม Part นี้)

**โจทย์**: จงออกแบบ (โดยไม่ต้องเขียนโค้ดเต็ม) ระบบที่ผสาน Part 074 (Web Layer ค้นหาลูกค้าผ่าน
Indexed File) เข้ากับ Part นี้ (Database Shim): แทนที่จะเก็บข้อมูลลูกค้าใน Indexed File ให้เปลี่ยน
ไปเก็บใน PostgreSQL แทน โดย Web Layer และ URL/API ยังคงเหมือนเดิมทุกประการ จงระบุว่าต้องแก้ไข
ส่วนใดบ้าง

**เฉลยแนวทาง**:

1. **ย้ายข้อมูล**: สร้างตาราง `customers` ใน PostgreSQL ที่มีโครงสร้างเทียบเท่ากับ record ใน
   `CUST074.DAT` เดิม (`customer_id`, `customer_name`, `city`, `balance`) แล้ว insert ข้อมูล
   ลูกค้าเดิมทั้งหมดเข้าไป (อาจเขียนโปรแกรม migration แยกต่างหาก หรือ export/import ผ่าน `psql`)
2. **แก้ไขโปรแกรม COBOL**: เปลี่ยน `CUSTLOOKUP.cob` จากที่เคย `OPEN`/`READ` ไฟล์ Indexed File
   โดยตรง (Part 074 ขั้นตอนที่ 734) ให้เปลี่ยนไปใช้เทคนิค Shim ที่เรียนใน Part นี้แทน — เช่น
   ประกอบคำสั่ง `SELECT ... WHERE customer_id = ?` ด้วย `STRING`, เรียกผ่าน `CALL "SYSTEM"` +
   `psql` (หรือ `CALL` โปรแกร C ที่ link กับ `libpq` สำหรับประสิทธิภาพที่ดีกว่า), แล้วแปลงผลลัพธ์
   เป็นรูปแบบคั่นด้วย `|` ทาง stdout **เหมือนเดิมทุกประการ**
3. **Web Layer (Python/Node.js) ไม่ต้องแก้ไขเลย**: เพราะ interface ระหว่าง Web Layer กับโปรแกรม
   COBOL (เรียกผ่าน argument, รับผลลัพธ์คั่นด้วย `|` ทาง stdout) ยังคงเหมือนเดิมทุกประการตามที่
   ออกแบบไว้ตั้งแต่ Part 074 — นี่คือประโยชน์สำคัญของการออกแบบ interface ที่ชัดเจนและแยกส่วน
   (Separation of Concerns) ตั้งแต่ต้น ทำให้เปลี่ยนแหล่งข้อมูลเบื้องหลัง (จาก Indexed File เป็น
   PostgreSQL) ได้โดยกระทบเฉพาะ COBOL program เพียงตัวเดียว ไม่กระทบ Web Layer เลยแม้แต่น้อย

การออกแบบเช่นนี้แสดงให้เห็นคุณค่าที่แท้จริงของสถาปัตยกรรมแบบแบ่งชั้น (layered architecture) ที่
เรียนมาตลอด Part 073-075: การเปลี่ยนแปลงในชั้นหนึ่ง (data storage) ไม่จำเป็นต้องกระทบชั้นอื่น (web
layer) ตราบใดที่ interface ระหว่างชั้นถูกออกแบบไว้อย่างชัดเจนและคงที่

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้และทดสอบจริงว่า COBOL เชื่อมต่อกับฐานข้อมูลสมัยใหม่ได้อย่างไร:

- ทำไม GnuCOBOL ไม่มี native driver สำหรับ MySQL/PostgreSQL (ต่างจาก DB2 Embedded SQL ใน
  Part 057-060 ที่มี precompiler เฉพาะทาง) และแนวคิด **Shim Program Pattern** ที่แก้ปัญหานี้
- ตรวจสอบสภาพแวดล้อมจริงอย่างซื่อตรง: พบว่า PostgreSQL ใช้งานได้เต็มรูปแบบ ส่วน MySQL ไม่มี
  ติดตั้งอยู่เลย และแบ่งเนื้อหาให้สอดคล้องกับข้อเท็จจริงนี้
- ทดสอบจริงแบบ end-to-end กับ **PostgreSQL**: อ่านข้อมูล (`DBQUERY`), เขียนข้อมูล (`DBINSERT`)
  ผ่าน `CALL "SYSTEM"` + `psql` CLI shim, และค้นหาข้อมูลรายตัว (`DBCALL`) ผ่าน `CALL` โปรแกรม C
  ที่ link กับ `libpq` โดยตรง — ทั้งสามโปรแกรมทำงานถูกต้องสมบูรณ์และให้ผลลัพธ์ตรงตามที่ตรวจสอบด้วยมือ
- เปรียบเทียบข้อดีข้อเสียของสองแนวทาง Shim (CLI vs native library) พร้อมเกณฑ์การเลือกใช้ในทาง
  ปฏิบัติ
- ความปลอดภัยที่สำคัญ: SQL Injection ในการสร้าง Dynamic SQL และแนวทางการจัดการรหัสผ่านที่ปลอดภัย
  กว่าการ hardcode
- ตัวอย่างอ้างอิงสำหรับ MySQL ที่ระบุไว้อย่างชัดเจนว่ายังไม่ได้ทดสอบรันจริงในสภาพแวดล้อมนี้ พร้อม
  ตารางเปรียบเทียบ syntax ระหว่าง `psql` กับ `mysql` CLI

ใน **Part 076** เราจะนำสิ่งที่สร้างมาตลอด Part 073-075 (Wrapper Service, Web Layer, Database
Shim) มา**บรรจุลงใน Docker Container** เพื่อให้ deploy ระบบ COBOL สมัยใหม่เหล่านี้ได้ง่ายและ
ทำซ้ำได้ (reproducible) ในทุกสภาพแวดล้อม

**[← กลับไป Part 074: COBOL และ Web Application: เชื่อมกับ Node.js/Python](part-074-cobol-web-integration.md)** | **[ไปยัง Part 076: Containerizing COBOL ด้วย Docker →](part-076-docker-cobol.md)**
