# Part 058: Embedded SQL: EXEC SQL ใน COBOL (ขั้นตอนที่ 571–580)

## คำนำของ Part นี้

ใน Part 057 เราเรียนรู้ภาษา SQL และแนวคิดพื้นฐานของ DB2 แบบ "ยืนอิสระ" (standalone) คือมองผ่าน SQL
statement เฉย ๆ โดยยังไม่ได้เชื่อมกับโปรแกรม COBOL จริง คำถามที่ตามมาตามธรรมชาติคือ: แล้วโปรแกรม COBOL
บน Mainframe จะ "คุย" กับฐานข้อมูล DB2 ได้อย่างไร คำตอบคือเทคนิคที่เรียกว่า **Embedded SQL**
(SQL แบบฝังตัว) — การเขียนคำสั่ง SQL ปะปนอยู่ในโค้ด COBOL โดยตรง ผ่านบล็อก `EXEC SQL ... END-EXEC`

Part นี้จะพาคุณเข้าใจกลไกเบื้องหลัง Embedded SQL ตั้งแต่โครงสร้างไวยากรณ์ การประกาศตัวแปรที่ใช้ร่วมกัน
ระหว่าง COBOL กับ SQL (Host Variables) ไปจนถึงการตรวจสอบผลลัพธ์ผ่าน SQLCODE ก่อนที่ Part 059 จะต่อยอด
ไปสู่การประมวลผลหลายแถวด้วย Cursor และ Part 060 จะสอนการเขียน Stored Procedure ด้วย COBOL

> **โปรดอ่านกรอบคำเตือนด้านล่างก่อนเริ่มเรียนทุกครั้ง** เนื้อหาทั้งหมดใน Part 058–064 (Embedded SQL,
> DB2 Cursor, DB2 Stored Procedures, CICS) เป็นเทคโนโลยี Middleware ของ IBM Mainframe ที่ต้องอาศัย
> ซอฟต์แวร์เฉพาะทาง (DB2 for z/OS หรือ CICS Transaction Server) ซึ่ง**ไม่มีอยู่ในสภาพแวดล้อมที่ใช้เรียน
> หลักสูตรนี้**

---

## ⚠️ กรอบคำเตือนสำคัญ: ข้อจำกัดของสภาพแวดล้อมสำหรับ Part นี้

สภาพแวดล้อมที่ใช้เรียนหลักสูตรนี้คือ Linux พร้อม **GnuCOBOL** (คำสั่ง `cobc`) เท่านั้น — **ไม่มี z/OS จริง
ไม่มี DB2 Engine และไม่มี DB2 Precompiler** การคอมไพล์โปรแกรมที่มี Embedded SQL จริงบน Mainframe
ต้องผ่านขั้นตอนพิเศษที่เรียกว่า **DB2 Precompiler** (เช่น `DSNHPC` บน z/OS) ก่อนที่จะส่งต่อให้ COBOL
Compiler ตามปกติ — ขั้นตอนนี้ไม่มีอยู่ใน GnuCOBOL เลย

เราทดสอบจริงแล้วด้วยการคอมไพล์โค้ดที่มี `EXEC SQL INCLUDE SQLCA END-EXEC` ด้วย `cobc` ธรรมดาบน
สภาพแวดล้อมนี้ ผลคือ **compile error ทันที**:

```
error: SQLCA: No such file or directory
error: syntax error, unexpected Identifier or Literal, expecting .
error: PROCEDURE DIVISION header missing
```

เหตุผลคือ `cobc` ธรรมดาตีความ `EXEC SQL INCLUDE SQLCA END-EXEC` ตามตัวอักษร โดยพยายามมองหาไฟล์
copybook ชื่อ `SQLCA` มา `COPY` (เพราะ GnuCOBOL ไม่รู้จักไวยากรณ์ EXEC SQL เป็นพิเศษ) แทนที่จะประมวลผล
เป็นคำสั่ง SQL แบบที่ DB2 Precompiler เข้าใจ

**ดังนั้น**: ตัวอย่างโค้ด `EXEC SQL ... END-EXEC` **ทุกตัวอย่าง**ใน Part นี้เป็น**ไวยากรณ์อ้างอิง
(Reference Syntax)** ที่เขียนตามมาตรฐาน IBM DB2 จริงอย่างถูกต้องแม่นยำ เหมาะสำหรับการอ่านและทำความ
เข้าใจโครงสร้าง แต่**ไม่สามารถคอมไพล์หรือรันบนสภาพแวดล้อมนี้ได้** จะมีป้ายกำกับ
`⚠️ REFERENCE SYNTAX — ไม่สามารถคอมไพล์/รันได้ในสภาพแวดล้อมนี้` กำกับไว้เสมอ เพื่อไม่ให้สับสนกับโค้ด
GnuCOBOL ล้วนที่คอมไพล์และรันได้จริงใน Part ก่อนหน้า

ในบางขั้นตอน เราจะแยกส่วนโครงสร้างข้อมูล COBOL ล้วน ๆ (ที่ไม่มี `EXEC SQL` ปนอยู่) ออกมาคอมไพล์และ
รันจริงเพื่อพิสูจน์ว่าส่วนที่เป็น COBOL แท้ ๆ นั้นถูกต้องตามไวยากรณ์ — จุดใดที่ทำเช่นนี้จะระบุไว้อย่าง
ชัดเจนว่า **"ทดสอบจริงแล้ว"** ต่างจากไวยากรณ์ SQL ที่กำกับว่า **"อ้างอิงเท่านั้น"**

---

## ขั้นตอนที่ 571: Embedded SQL คืออะไร และวงจรการ Precompile

### แนวคิดพื้นฐาน

**Embedded SQL** คือการเขียนคำสั่ง SQL (SELECT, INSERT, UPDATE, DELETE) ปะปนอยู่ในซอร์สโค้ด COBOL
โดยตรง ล้อมด้วยคำสั่ง `EXEC SQL` และ `END-EXEC` ทำให้โปรแกรมเมอร์ COBOL สามารถดึงข้อมูลจากตาราง DB2
มาประมวลผลด้วยตรรกะ COBOL ตามปกติ (IF, PERFORM, COMPUTE ฯลฯ) ได้ในโปรแกรมเดียวกัน โดยไม่ต้องเขียน
โค้ดเชื่อมต่อฐานข้อมูลแบบ manual เหมือนภาษาอื่น (เช่น JDBC ใน Java)

### วงจรการแปลงโปรแกรม (Precompile → Compile → Bind)

จุดที่แตกต่างจากโปรแกรม COBOL ปกติมากที่สุดคือ โปรแกรมที่มี Embedded SQL **ไม่ได้ถูกส่งตรงไปที่ COBOL
Compiler** แต่ต้องผ่าน 3 ขั้นตอนตามลำดับก่อน:

```
[COBOL + EXEC SQL source]
        |
        v
  (1) DB2 PRECOMPILER (เช่น DSNHPC บน z/OS)
      - อ่านบล็อก EXEC SQL ทั้งหมด
      - แทนที่ด้วยคำสั่ง CALL ไปยัง DB2 runtime library (เช่น CALL 'DSNHLI')
      - แยกคำสั่ง SQL ออกไปเก็บเป็นไฟล์ต่างหากเรียกว่า DBRM
      - ผลลัพธ์: Modified COBOL Source (ไม่มี EXEC SQL หลงเหลือแล้ว)
        |
        v
  (2) COBOL COMPILER (เช่น IBM Enterprise COBOL)
      - คอมไพล์ Modified Source เป็น Object Module ตามปกติ
        |
        v
  (3) BIND (DB2 Utility)
      - นำ DBRM ไป BIND เข้ากับ DB2 เพื่อสร้าง PACKAGE/PLAN
      - DB2 จะตรวจสอบสิทธิ์และวางแผนการทำงาน (Access Path) ล่วงหน้าตอนนี้ ไม่ใช่ตอนรันจริง
        |
        v
  (4) LINK-EDIT: รวม Object Module + DB2 Runtime Library เป็น Load Module ที่รันได้จริง
```

จุดสำคัญที่ต้องจำคือ **EXEC SQL ไม่ใช่ COBOL มาตรฐาน** มันเป็น "ภาษาที่สอง" ที่ฝังอยู่ในซอร์สไฟล์
เดียวกัน แล้วอาศัย Precompiler แปลงให้กลายเป็น COBOL ปกติ (บวก CALL ไปยัง library) ก่อนที่ COBOL
Compiler ตัวจริงจะได้เห็นมันด้วยซ้ำ — นี่คือเหตุผลที่ GnuCOBOL ธรรมดา (ไม่มี Precompiler แบบ DB2)
ไม่สามารถประมวลผลไฟล์เหล่านี้ได้เลย

### ข้อควรระวัง

- อย่าสับสนระหว่าง **DB2 Precompiler** (แปลง EXEC SQL) กับ **CICS Translator** (แปลง EXEC CICS ที่จะ
  เรียนใน Part 061) ทั้งสองเป็นเครื่องมือคนละตัวกัน แต่ใช้แนวคิดคล้ายกันคือ "แปลงภาษาเฉพาะทางที่ฝังอยู่
  ให้กลายเป็น COBOL ธรรมดาก่อนคอมไพล์จริง" หากโปรแกรมหนึ่งมีทั้ง EXEC SQL และ EXEC CICS ต้องรันผ่าน
  ทั้งสองเครื่องมือตามลำดับที่ถูกต้อง (CICS Translator ก่อน แล้วค่อย DB2 Precompiler เป็นลำดับที่นิยม)
- DBRM (Database Request Module) ที่เกิดจากขั้นตอน Precompile เป็นคนละไฟล์กับ Load Module —
  ถ้าลืม BIND DBRM เข้า DB2 โปรแกรมจะคอมไพล์และ link-edit ผ่านได้ปกติ แต่จะ**รันไม่ได้**เพราะ DB2
  หา Package ที่ผูกกับโปรแกรมนี้ไม่เจอ (SQLCODE -805)

### แบบฝึกหัดที่ 571.1

**โจทย์**: จงเรียงลำดับ 4 ขั้นตอน (Precompile, Compile, Bind, Link-edit) ให้ถูกต้อง และอธิบายสั้น ๆ ว่า
แต่ละขั้นตอนทำอะไร

**เฉลย**: (1) **Precompile** — DB2 Precompiler อ่าน EXEC SQL ทั้งหมด แยกเป็น DBRM และแทนที่ด้วย CALL
ไปยัง DB2 runtime (2) **Compile** — COBOL Compiler คอมไพล์ Modified Source (ไม่มี EXEC SQL แล้ว) เป็น
Object Module (3) **Bind** — นำ DBRM ไปผูกเข้ากับ DB2 Catalog สร้างเป็น Package/Plan พร้อม Access Path
(4) **Link-edit** — รวม Object Module เข้ากับ DB2 Runtime Library เป็น Load Module ที่รันได้จริงบน z/OS

---

## ขั้นตอนที่ 572: ไวยากรณ์พื้นฐาน EXEC SQL...END-EXEC และตำแหน่งที่วางได้

### รูปแบบไวยากรณ์

ทุกคำสั่ง SQL ที่ฝังใน COBOL ต้องอยู่ในกรอบนี้เสมอ:

```
EXEC SQL
    <คำสั่ง SQL มาตรฐาน>
END-EXEC.
```

ข้อสังเกต: `END-EXEC` **ไม่มีเครื่องหมาย `-` คั่นแบบ scope terminator ทั่วไปของ COBOL-85**
(เช่น `END-IF`) แต่เป็นคำสงวนพิเศษเฉพาะของ Embedded SQL เอง และต้องปิดท้ายด้วย period (`.`)
เหมือนประโยค COBOL ปกติเมื่ออยู่ท้ายบล็อก statement

### ตำแหน่งที่ EXEC SQL วางได้

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP572-EXEC-SQL-POSITIONS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> (1) Position 1: in WORKING-STORAGE - for declaring SQLCA
      *>     and Host Variables (covered in detail in steps 573-574)
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID          PIC S9(5) COMP-3.
       01  HV-CUST-NAME        PIC X(20).
           EXEC SQL END DECLARE SECTION END-EXEC.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> (2) Position 2: in PROCEDURE DIVISION - for actual SQL statements
           EXEC SQL
               SELECT CUST_NAME
                 INTO :HV-CUST-NAME
                 FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           DISPLAY "NAME: " HV-CUST-NAME.
           STOP RUN.
```

### อธิบายจุดสำคัญ

- **EXEC SQL ปรากฏได้ 2 บริเวณหลัก**: (1) ใน `WORKING-STORAGE SECTION` สำหรับคำสั่งที่เกี่ยวกับการ
  ประกาศข้อมูล (`INCLUDE SQLCA`, `BEGIN/END DECLARE SECTION`, `DECLARE TABLE`) และ (2) ใน
  `PROCEDURE DIVISION` สำหรับคำสั่งที่ทำงานจริง (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `OPEN`,
  `FETCH`, `CLOSE` ของ Cursor ที่จะเรียนใน Part 059)
- DB2 Precompiler จะอ่านไฟล์ทั้งหมดผ่านตั้งแต่ต้นจนจบเพื่อตามหาบล็อก `EXEC SQL` ทุกจุด ไม่ว่าจะอยู่ Division
  ใดก็ตาม แล้วแทนที่ทีละบล็อกด้วยโค้ด COBOL ที่เทียบเท่า (ปกติจะแทรกเป็นคอมเมนต์คำสั่ง SQL เดิมไว้ข้างบน
  แล้วตามด้วย `CALL` และการเตรียม parameter block จริง)
- Column ของ Embedded SQL ยังคงต้องอยู่ในกฎ Fixed-Format คอลัมน์ 8–72 เหมือน COBOL ปกติทุกประการ
  เพราะ Precompiler อ่านไฟล์ในรูปแบบเดียวกับ COBOL Compiler

### ข้อควรระวัง

- ห้ามลืม period หลัง `END-EXEC` เมื่อเป็นการปิดท้ายประโยค (เว้นแต่กรณีพิเศษบางคำสั่งที่ผสมกับเงื่อนไข
  ของ COBOL ซึ่งไม่ได้ใช้บ่อย)
- SQL statement ภายในบล็อกต้องปิดท้ายด้วยการพบ `END-EXEC` เท่านั้น **ห้ามใส่ `;` แบบที่ SQL script
  ทั่วไปใช้** เพราะจะทำให้ Precompiler สับสน (ต่างจาก SQL ที่รันผ่าน command line หรือ SPUFI โดยตรง
  ซึ่งต้องใช้ `;` ปิดท้าย)

### แบบฝึกหัดที่ 572.1

**โจทย์**: จงชี้ว่าโค้ดต่อไปนี้ผิดตรงไหน และแก้ไขให้ถูกต้อง

```
EXEC SQL
    SELECT CUST_NAME INTO :HV-CUST-NAME FROM CUSTOMER;
END-EXEC
```

**เฉลย**: ผิด 2 จุด (1) มีเครื่องหมาย `;` ต่อท้ายคำสั่ง SQL ซึ่งไม่ควรมีในบล็อก Embedded SQL
(2) ไม่มี period หลัง `END-EXEC` ฉบับแก้ไข:

```
EXEC SQL
    SELECT CUST_NAME INTO :HV-CUST-NAME FROM CUSTOMER
END-EXEC.
```

---

## ขั้นตอนที่ 573: INCLUDE SQLCA และ SQLCODE — หัวใจของการตรวจผลลัพธ์ SQL

### SQLCA คืออะไร

**SQLCA (SQL Communication Area)** คือโครงสร้างข้อมูลมาตรฐานที่ DB2 ใช้ "รายงานผล" กลับมาให้โปรแกรม
COBOL รู้ทุกครั้งหลังสั่งคำสั่ง SQL ไม่ว่าจะสำเร็จ ล้มเหลว หรือไม่พบข้อมูล คำสั่ง
`EXEC SQL INCLUDE SQLCA END-EXEC` จะบอก Precompiler ให้แทรกโครงสร้างนี้เข้ามาในโปรแกรมโดยอัตโนมัติ
(เทียบเท่าการ `COPY` copybook มาตรฐานของ DB2 ที่ชื่อ `SQLCA`)

### โครงสร้างสำคัญของ SQLCA (แบบย่อ)

```cobol
      *> REFERENCE SYNTAX - standard SQLCA structure (abbreviated)
      *> generated automatically by EXEC SQL INCLUDE SQLCA END-EXEC
       01  SQLCA.
           05  SQLCAID          PIC X(8).
           05  SQLCABC          PIC S9(9)   COMP.
           05  SQLCODE          PIC S9(9)   COMP.
           05  SQLERRM.
               49  SQLERRML     PIC S9(4)   COMP.
               49  SQLERRMC     PIC X(70).
           05  SQLERRP          PIC X(8).
           05  SQLERRD          OCCURS 6 TIMES PIC S9(9) COMP.
           05  SQLWARN.
               10  SQLWARN0     PIC X.
               10  SQLWARN1     PIC X.
               10  SQLWARN2     PIC X.
               10  SQLWARN3     PIC X.
               10  SQLWARN4     PIC X.
               10  SQLWARN5     PIC X.
               10  SQLWARN6     PIC X.
               10  SQLWARN7     PIC X.
           05  SQLSTATE         PIC X(5).
```

### SQLCODE — ฟิลด์ที่สำคัญที่สุด

หลังทุกคำสั่ง `EXEC SQL` ที่ทำงานจริง (SELECT, INSERT, UPDATE, DELETE, OPEN, FETCH, CLOSE) DB2 จะเติม
ค่าตัวเลขลงใน `SQLCODE` เสมอโดยอัตโนมัติ (ไม่ต้องเขียนโค้ดอ่านค่าเอง) ค่าที่พบบ่อยที่สุด:

| SQLCODE | ความหมาย |
|---|---|
| `0` | สำเร็จสมบูรณ์ ไม่มีปัญหาใด ๆ |
| `100` | ไม่พบข้อมูล (Not Found) — เช่น SELECT ที่ไม่เจอแถวใดตรงเงื่อนไขเลย หรือ FETCH cursor จนหมดแถวแล้ว |
| ติดลบ (เช่น `-803`, `-811`, `-305`) | เกิดข้อผิดพลาด — ตัวเลขยิ่งเฉพาะเจาะจงบอกสาเหตุ (`-803` = ละเมิด unique constraint, `-811` = SELECT INTO ได้มากกว่า 1 แถว, `-305` = ค่าที่ได้เป็น NULL แต่ไม่มี indicator variable รองรับ) |
| บวกอื่น ๆ (นอกจาก 100) | สำเร็จแต่มีคำเตือน (Warning) เช่น ค่าถูกตัดทอน |

### รูปแบบการตรวจสอบมาตรฐาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               SELECT CUST_NAME
                 INTO :HV-CUST-NAME
                 FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   DISPLAY "FOUND: " HV-CUST-NAME
               WHEN 100
                   DISPLAY "NO CUSTOMER WITH THAT ID."
               WHEN OTHER
                   DISPLAY "SQL ERROR. SQLCODE=" SQLCODE
           END-EVALUATE.
```

### ข้อควรระวัง

- `SQLCODE` เป็นตัวแปรระดับ COBOL ธรรมดา (`PIC S9(9) COMP`) เข้าถึงได้เหมือนตัวแปรทั่วไป **ไม่ต้อง**
  ใส่ `:` นำหน้าเมื่ออ้างถึงมันในโค้ด COBOL ปกติ (ต่างจาก Host Variable ที่ต้องมี `:` เสมอเมื่ออยู่ใน
  ประโยค SQL — จะอธิบายละเอียดในขั้นตอนที่ 574) ทั้งนี้ห้ามอ้าง `SQLCODE` แบบมี `:` ในบล็อก SQL เอง
- **ต้องตรวจ `SQLCODE` หลัง `EXEC SQL` ทุกครั้งที่มีนัยสำคัญ** — โปรแกรมมือใหม่จำนวนมากลืมตรวจสอบ
  แล้วใช้ค่าในตัวแปรที่ SELECT มา ทั้งที่จริงคำสั่งนั้นล้มเหลว (SQLCODE ติดลบ) ทำให้ตัวแปรยังเป็นค่าเก่า
  ที่ค้างจากรอบก่อนหน้า และตรรกะของโปรแกรมผิดเพี้ยนโดยไม่มี error ปรากฏชัดเจน

### แบบฝึกหัดที่ 573.1

**โจทย์**: หากรัน `EXEC SQL SELECT ... INTO :HV-BAL FROM ACCOUNT WHERE ACCT_ID = :HV-ID END-EXEC`
แล้วได้ `SQLCODE = 100` หมายความว่าอย่างไร และโปรแกรมควรทำอย่างไรต่อ

**เฉลย**: `SQLCODE = 100` หมายถึงไม่พบแถวใดในตาราง `ACCOUNT` ที่ตรงกับ `ACCT_ID` ที่ค้นหา (Not Found)
ไม่ใช่ข้อผิดพลาดร้ายแรง แต่เป็นผลลัพธ์ปกติที่ต้องรองรับด้วยตรรกะทางธุรกิจ เช่น แสดงข้อความ
"ไม่พบบัญชีนี้" ให้ผู้ใช้ทราบ แทนที่จะพยายามใช้ค่า `HV-BAL` ที่ยังไม่ได้ถูกกำหนดค่าใหม่จริง (อาจเป็นค่า
ขยะจากรอบก่อนหน้า) มาคำนวณต่อ

---

## ขั้นตอนที่ 574: Host Variables — สะพานเชื่อมระหว่าง COBOL กับ SQL

### แนวคิด Host Variable

**Host Variable** คือตัวแปร COBOL ธรรมดาที่ประกาศใน `WORKING-STORAGE SECTION` ตามปกติ แต่ถูก "เปิด
เผย" ให้ DB2 Precompiler รู้จักผ่านกรอบ `EXEC SQL BEGIN DECLARE SECTION` ... `EXEC SQL END DECLARE
SECTION` เพื่อให้สามารถใช้ตัวแปรเหล่านี้ **รับค่าจาก** และ **ส่งค่าเข้าไปใน** คำสั่ง SQL ได้

เมื่ออ้างถึง Host Variable **ภายในประโยค SQL** (ระหว่าง `EXEC SQL` กับ `END-EXEC`) ต้องใส่เครื่องหมาย
colon (`:`) นำหน้าเสมอ เพื่อให้ DB2 แยกแยะได้ว่านี่คือชื่อตัวแปร COBOL ไม่ใช่ชื่อคอลัมน์ในตาราง

### ตัวอย่างการประกาศและใช้งาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID          PIC S9(5)      COMP-3.
       01  HV-CUST-NAME        PIC X(20).
       01  HV-BALANCE          PIC S9(9)V99   COMP-3.
           EXEC SQL END DECLARE SECTION END-EXEC.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Sending a value into SQL: HV-CUST-ID is an "input" host variable
           MOVE 10001 TO HV-CUST-ID.

           EXEC SQL
               SELECT CUST_NAME, BALANCE
                 INTO :HV-CUST-NAME, :HV-BALANCE
                 FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

      *> Receiving values back from SQL: HV-CUST-NAME, HV-BALANCE are "output"
           IF SQLCODE = 0
               DISPLAY "NAME: " HV-CUST-NAME " BAL: " HV-BALANCE
           END-IF.
```

### ทดสอบจริง: โครงสร้างข้อมูล Host Variable แบบ "ถอด EXEC SQL ออก" ก็เป็น COBOL ที่ใช้งานได้จริง

เพื่อพิสูจน์ว่าส่วนที่เป็น **โครงสร้างข้อมูล COBOL แท้ ๆ** (ไม่นับกรอบ `EXEC SQL BEGIN/END DECLARE
SECTION` ซึ่งเป็นเพียงกรอบบอก Precompiler) นั้นถูกต้องตามหลัก COBOL ทุกประการ เราทดสอบด้วยการคอมไพล์
และรันจริงบน GnuCOBOL โดยตัดกรอบ EXEC SQL ออก:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP574-HOST-VARS-PLAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> This mirrors a DB2 host-variable declare section, but with
      *> the EXEC SQL BEGIN/END DECLARE SECTION wrapper lines removed,
      *> so plain GnuCOBOL can compile and run it standalone. This
      *> proves the surrounding COBOL data description is syntactically
      *> valid; it does NOT prove the EXEC SQL statements themselves
      *> compile, since that requires a real DB2 precompiler.
       01  HV-CUST-ID          PIC 9(5).
       01  HV-CUST-NAME        PIC X(20).
       01  HV-BALANCE          PIC S9(7)V99 COMP-3.
       01  HV-NULL-IND         PIC S9(4) COMP.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO HV-CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO HV-CUST-NAME.
           MOVE 500.00 TO HV-BALANCE.
           MOVE 0 TO HV-NULL-IND.
           DISPLAY "HV-CUST-ID   : " HV-CUST-ID.
           DISPLAY "HV-CUST-NAME : " HV-CUST-NAME.
           DISPLAY "HV-BALANCE   : " HV-BALANCE.
           DISPLAY "HV-NULL-IND  : " HV-NULL-IND.
           STOP RUN.
```

คอมไพล์และรันจริง:

```bash
cobc -x -o step574 step574.cob
./step574
```

**ผลลัพธ์จริง (ทดสอบแล้ว):**

```
HV-CUST-ID   : 10001
HV-CUST-NAME : SOMCHAI JAIDEE
HV-BALANCE   : +0000500.00
HV-NULL-IND  : +0000
```

นี่ยืนยันว่า Host Variable เป็นเพียง **ตัวแปร COBOL มาตรฐาน** ทุกประการ (`PIC`, `COMP-3`, `COMP` ใช้ได้
ตามปกติ) สิ่งที่ทำให้มัน "พิเศษ" คือการถูกอ้างถึงด้วย `:` ภายในประโยค SQL เท่านั้น ส่วนบล็อก `EXEC SQL
SELECT ... END-EXEC` ที่ใช้ตัวแปรเหล่านี้จริงยังคงเป็นไวยากรณ์อ้างอิงที่ต้องมี DB2 Precompiler จึงจะ
ทำงานได้ (ดังที่พิสูจน์ไว้ในกรอบคำเตือนต้น Part)

### ข้อควรระวัง

- ชนิดข้อมูล COBOL ของ Host Variable ต้อง "เข้ากันได้" กับชนิดข้อมูลของคอลัมน์ DB2 ที่จะ map ด้วย
  เช่น DB2 `DECIMAL(9,2)` มักจับคู่กับ `PIC S9(7)V99 COMP-3`, DB2 `INTEGER` จับคู่กับ `PIC S9(9) COMP`,
  DB2 `CHAR(n)`/`VARCHAR(n)` จับคู่กับ `PIC X(n)`
- ห้ามลืม `:` เมื่ออ้าง Host Variable ในประโยค SQL — ถ้าลืม DB2 Precompiler จะเข้าใจผิดว่าชื่อนั้นคือ
  ชื่อคอลัมน์ในตาราง แล้วรายงาน error ว่าไม่พบคอลัมน์ชื่อนั้น
- **ห้ามใส่ `:` เมื่ออ้างตัวแปรเดียวกันนอกบล็อก SQL** (เช่นใน `IF`, `MOVE`, `DISPLAY` ของ COBOL ปกติ)
  เพราะ `:` มีความหมายเฉพาะภายในบล็อก `EXEC SQL...END-EXEC` เท่านั้น

### แบบฝึกหัดที่ 574.1

**โจทย์**: จงเขียนส่วนประกาศ Host Variable สำหรับคอลัมน์ `ORDER_ID INTEGER`, `ORDER_DATE DATE`,
`TOTAL_AMOUNT DECIMAL(9,2)`

**เฉลย**:

```
EXEC SQL BEGIN DECLARE SECTION END-EXEC.
01  HV-ORDER-ID        PIC S9(9)    COMP.
01  HV-ORDER-DATE       PIC X(10).
01  HV-TOTAL-AMOUNT     PIC S9(7)V99 COMP-3.
EXEC SQL END DECLARE SECTION END-EXEC.
```

(หมายเหตุ: DB2 `DATE` มักแทนด้วย Host Variable แบบ `PIC X(10)` ในรูปแบบ `YYYY-MM-DD` เมื่อไม่ได้ใช้
Host Variable ชนิดพิเศษของ DB2 เอง)

---

## ขั้นตอนที่ 575: DECLARE TABLE — เอกสารประกอบที่ช่วย Precompiler ตรวจสอบ

### วัตถุประสงค์ของ DECLARE TABLE

`EXEC SQL DECLARE TABLE ... END-EXEC` คือคำสั่งที่ **ไม่ได้สร้างอะไรใน DB2 จริง** และ**ไม่ได้ถูกแปลง
เป็นโค้ดที่รันได้เลย** แต่เป็น "เอกสารประกอบ" (documentation) ที่บอก DB2 Precompiler ว่าตารางที่โปรแกรม
จะใช้งานมีโครงสร้างคอลัมน์อย่างไร เพื่อให้ Precompiler สามารถ**ตรวจสอบความถูกต้องของชนิดข้อมูล**
ระหว่าง Host Variable กับคอลัมน์ตั้งแต่ตอน Precompile (ก่อนรันจริง) แทนที่จะรอให้ error ปรากฏตอนรัน

### ไวยากรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL
               DECLARE CUSTOMER TABLE
               (
                   CUST_ID       INTEGER      NOT NULL,
                   CUST_NAME     CHAR(20)     NOT NULL,
                   BALANCE       DECIMAL(9,2) NOT NULL,
                   EMAIL         VARCHAR(50)
               )
           END-EXEC.
```

### อธิบายจุดสำคัญ

- `DECLARE TABLE` ปกติจะวางไว้ใน `WORKING-STORAGE SECTION` ก่อนส่วนที่ `BEGIN DECLARE SECTION`
- โครงสร้างที่ระบุต้อง**ตรงกับตารางจริงใน DB2 Catalog เป๊ะ ๆ** (ชื่อคอลัมน์ ชนิดข้อมูล ลำดับ NOT NULL)
  มิฉะนั้น Precompiler อาจตรวจสอบผิดพลาดหรือปล่อยผ่านข้อผิดพลาดที่ควรจับได้ตั้งแต่ต้น
- ในทางปฏิบัติจริง หลายองค์กรใช้เครื่องมือ (เช่น DCLGEN ของ IBM) เพื่อ**สร้าง DECLARE TABLE +
  Host Variable Group ให้อัตโนมัติ** จากโครงสร้างตารางจริงใน DB2 Catalog แทนการพิมพ์มือ ลดความเสี่ยง
  ที่จะพิมพ์ผิดหรือลืมอัปเดตตามการเปลี่ยนแปลงตาราง

### ข้อควรระวัง

- `DECLARE TABLE` **ไม่ใช่** คำสั่งสร้างตาราง (ต่างจาก SQL DDL `CREATE TABLE` ที่เรียนใน Part 057)
  มันเป็นเพียงคำอธิบายให้ Precompiler ใช้ตรวจสอบเท่านั้น ต่อให้เขียนโครงสร้างผิดหรือไม่เขียนเลย โปรแกรม
  ก็ยังอาจ compile ผ่านได้ (เพียงแต่จะไม่ได้รับประโยชน์จากการตรวจสอบชนิดข้อมูลล่วงหน้า) เพราะ
  DECLARE TABLE ในหลาย compiler implementation ถือเป็นทางเลือก ไม่บังคับ
- อย่าลืมอัปเดต `DECLARE TABLE` ทุกครั้งที่โครงสร้างตารางจริงเปลี่ยน (เช่น เพิ่มคอลัมน์ใหม่) มิฉะนั้น
  การตรวจสอบชนิดข้อมูลจะอิงจากข้อมูลเก่าที่ไม่ตรงกับความจริงอีกต่อไป

### แบบฝึกหัดที่ 575.1

**โจทย์**: เพราะเหตุใด `DECLARE TABLE` จึงมีประโยชน์แม้จะไม่ได้ถูกแปลงเป็นโค้ดที่รันจริงเลย

**เฉลย**: เพราะมันช่วยให้ **DB2 Precompiler ตรวจสอบความเข้ากันได้ของชนิดข้อมูล** ระหว่าง Host Variable
กับคอลัมน์ตั้งแต่ขั้นตอน Precompile (compile-time) แทนที่จะปล่อยให้เกิดข้อผิดพลาดตอนรันจริง
(runtime) ซึ่งอาจเกิดในสภาพแวดล้อม Production ที่มีผลกระทบร้ายแรงกว่ามาก การจับข้อผิดพลาดให้เร็วที่สุด
เท่าที่เป็นไปได้เป็นหลักการวิศวกรรมซอฟต์แวร์ที่สำคัญ (Fail Fast)

---

## ขั้นตอนที่ 576: SELECT INTO — ดึงข้อมูลแถวเดียวจาก DB2

### แนวคิด Singleton SELECT

คำสั่ง `EXEC SQL SELECT ... INTO ... END-EXEC` (เรียกว่า **Singleton SELECT** หรือ **SELECT INTO**)
ใช้สำหรับดึงข้อมูล**แถวเดียว**จากตาราง DB2 มาเก็บใน Host Variable โดยตรง โดยไม่ต้องใช้ Cursor
(Cursor สำหรับดึงหลายแถวจะสอนใน Part 059) เหมาะกับกรณีที่รู้แน่ชัดว่าเงื่อนไขจะให้ผลลัพธ์ไม่เกิน 1 แถว
เสมอ เช่น ค้นหาด้วย Primary Key

### ตัวอย่างสมบูรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP576-SELECT-INTO.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID          PIC S9(5)    COMP-3.
       01  HV-CUST-NAME        PIC X(20).
       01  HV-BALANCE          PIC S9(9)V99 COMP-3.
           EXEC SQL END DECLARE SECTION END-EXEC.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO HV-CUST-ID.

           EXEC SQL
               SELECT CUST_NAME, BALANCE
                 INTO :HV-CUST-NAME, :HV-BALANCE
                 FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   DISPLAY "CUSTOMER: " HV-CUST-NAME
                   DISPLAY "BALANCE : " HV-BALANCE
               WHEN 100
                   DISPLAY "CUSTOMER NOT FOUND."
               WHEN OTHER
                   DISPLAY "SQL ERROR: " SQLCODE
           END-EVALUATE.

           STOP RUN.
```

### อธิบายจุดสำคัญ

- ลำดับคอลัมน์ที่ระบุใน `SELECT` ต้องตรงกับลำดับ Host Variable ใน `INTO` ทุกประการ (คอลัมน์ที่ 1
  → Host Variable ตัวที่ 1, คอลัมน์ที่ 2 → ตัวที่ 2 ตามลำดับ) ไม่ใช่จับคู่ตามชื่อ
- หากเงื่อนไข `WHERE` มีโอกาสให้ผลลัพธ์**มากกว่า 1 แถว** ต้องเปลี่ยนไปใช้ Cursor แทนทันที
  (ถ้ายังใช้ SELECT INTO จะได้ `SQLCODE = -811` ทันที) — นี่คือเหตุผลที่ Part 059 ต้องมีเรื่อง Cursor
  โดยเฉพาะ

### ข้อควรระวัง

- **ห้ามลืมตรวจ `SQLCODE = 100`** เสมอ (ไม่พบข้อมูล) เพราะถ้าลืม แล้วโปรแกรมพยายามใช้
  `HV-CUST-NAME`/`HV-BALANCE` ต่อ ค่าที่ได้จะเป็นค่าเก่าที่ค้างอยู่ในหน่วยความจำจากรอบก่อนหน้า (หรือ
  ค่าว่างถ้าเป็นครั้งแรก) ไม่ใช่ error ที่ตรวจจับได้ง่าย ๆ
- ค่า `SQLCODE = -811` (ได้มากกว่า 1 แถว) เป็นสัญญาณว่า `WHERE` clause ออกแบบผิดหรือควรใช้ Cursor
  แทน ไม่ควรพยายามแก้ปัญหานี้ด้วยการเติม `FETCH FIRST 1 ROW ONLY` โดยไม่พิจารณาตรรกะทางธุรกิจก่อน
  (เพราะอาจได้แถวที่ไม่ตรงกับที่ต้องการจริง ๆ)

### แบบฝึกหัดที่ 576.1

**โจทย์**: จงเขียน `EXEC SQL SELECT INTO` เพื่อดึงคอลัมน์ `PRODUCT_NAME` และ `UNIT_PRICE` จากตาราง
`PRODUCT` โดยค้นหาด้วย `PRODUCT_CODE` ที่เก็บใน Host Variable `HV-PROD-CODE`

**เฉลย**:

```
EXEC SQL
    SELECT PRODUCT_NAME, UNIT_PRICE
      INTO :HV-PRODUCT-NAME, :HV-UNIT-PRICE
      FROM PRODUCT
     WHERE PRODUCT_CODE = :HV-PROD-CODE
END-EXEC.
```

---

## ขั้นตอนที่ 577: INSERT, UPDATE, DELETE ผ่าน Embedded SQL

### แนวคิด

ต่างจาก `SELECT INTO` ที่ต้องมี `INTO` เสมอ คำสั่ง `INSERT`, `UPDATE`, `DELETE` แบบฝังตัวมีไวยากรณ์
ใกล้เคียงกับ SQL มาตรฐานมาก เพียงแทนค่าคงที่ด้วย Host Variable (มี `:` นำหน้า) และยังคงต้องตรวจ
`SQLCODE` หลังทำงานเสมอเช่นเดียวกัน

### ตัวอย่าง INSERT

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           MOVE 10010          TO HV-CUST-ID.
           MOVE "NAPAPORN SUK" TO HV-CUST-NAME.
           MOVE 0               TO HV-BALANCE.

           EXEC SQL
               INSERT INTO CUSTOMER (CUST_ID, CUST_NAME, BALANCE)
               VALUES (:HV-CUST-ID, :HV-CUST-NAME, :HV-BALANCE)
           END-EXEC.

           IF SQLCODE = 0
               DISPLAY "INSERT OK."
           ELSE
               DISPLAY "INSERT FAILED. SQLCODE=" SQLCODE
           END-IF.
```

### ตัวอย่าง UPDATE

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           MOVE 500.00 TO HV-BALANCE.
           MOVE 10001  TO HV-CUST-ID.

           EXEC SQL
               UPDATE CUSTOMER
                  SET BALANCE = BALANCE + :HV-BALANCE
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           DISPLAY "ROWS AFFECTED: " SQLERRD(3).
```

### ตัวอย่าง DELETE

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           MOVE 10010 TO HV-CUST-ID.

           EXEC SQL
               DELETE FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           IF SQLCODE = 0
               DISPLAY "DELETE OK."
           ELSE IF SQLCODE = 100
               DISPLAY "NO ROW MATCHED - NOTHING DELETED."
           ELSE
               DISPLAY "DELETE FAILED. SQLCODE=" SQLCODE
           END-IF.
```

### อธิบายจุดสำคัญ

- `SQLERRD(3)` (สมาชิกตัวที่ 3 ของ array `SQLERRD` ใน SQLCA) คือฟิลด์มาตรฐานที่ DB2 ใช้รายงาน
  **จำนวนแถวที่ได้รับผลกระทบ** จากคำสั่ง `INSERT`/`UPDATE`/`DELETE` ล่าสุด มีประโยชน์มากในการตรวจสอบ
  ว่าคำสั่งทำงานตามที่คาดหวังจริงหรือไม่ (เช่น UPDATE ควรกระทบ 1 แถวเท่านั้น ถ้าได้มากกว่านั้นอาจ
  แปลว่า WHERE clause กว้างเกินไป)
- คำสั่งเหล่านี้ **ไม่มี `COMMIT` อัตโนมัติ** ต้องสั่ง `EXEC SQL COMMIT END-EXEC` เองอย่างชัดเจน
  (หรือปล่อยให้ Transaction Manager เช่น CICS จัดการให้ ซึ่งจะกล่าวถึงใน Part 061) มิฉะนั้นการ
  เปลี่ยนแปลงจะยังไม่ถูกบันทึกถาวรจนกว่าโปรแกรมจะจบและมีการ commit โดยปริยายตามการตั้งค่าของระบบ

### ข้อควรระวัง

- อย่าลืมว่า `DELETE`/`UPDATE` แบบฝังตัวที่ไม่ผ่าน Cursor จะกระทบ**ทุกแถว**ที่ตรงเงื่อนไข `WHERE`
  ในครั้งเดียว (ต่างจาก `DELETE`/`UPDATE ... WHERE CURRENT OF` ผ่าน Cursor ที่กระทบแค่แถวที่ Cursor
  กำลังชี้อยู่ — จะสอนใน Part 059 ขั้นตอนที่ 589)
- ตรวจสอบ `SQLCODE = 100` เสมอสำหรับ `UPDATE`/`DELETE` เพราะไม่ใช่ error แต่หมายถึง "ไม่มีแถวใดตรง
  เงื่อนไขเลย" ซึ่งอาจเป็นเรื่องปกติทางธุรกิจ (เช่น ลบข้อมูลที่ถูกลบไปแล้วซ้ำ)

### แบบฝึกหัดที่ 577.1

**โจทย์**: จงเขียนคำสั่ง Embedded SQL เพื่อลด `BALANCE` ของบัญชีที่ `ACCT_ID` ตรงกับ `HV-ACCT-ID`
ลง `HV-AMOUNT` แล้วแสดงจำนวนแถวที่ได้รับผลกระทบ

**เฉลย**:

```
EXEC SQL
    UPDATE ACCOUNT
       SET BALANCE = BALANCE - :HV-AMOUNT
     WHERE ACCT_ID = :HV-ACCT-ID
END-EXEC.

DISPLAY "ROWS AFFECTED: " SQLERRD(3).
```

---

## ขั้นตอนที่ 578: การจัดการค่า NULL ด้วย Indicator Variable

### ปัญหา: COBOL ไม่มีแนวคิด NULL

ในโลกของ SQL คอลัมน์สามารถมีค่า **NULL** (หมายถึง "ไม่มีค่า" ไม่ใช่ศูนย์หรือช่องว่าง) ได้ แต่ COBOL
**ไม่มีแนวคิดเรื่อง NULL ในตัวแปรเลย** — ทุกตัวแปร COBOL ต้องมีค่าบางอย่างอยู่เสมอ (เช่น ตัวเลขเป็น 0,
ตัวอักษรเป็นช่องว่าง) วิธีแก้ปัญหานี้คือการใช้ **Indicator Variable** (ตัวแปรบอกสถานะ NULL) คู่กับ
Host Variable แต่ละตัวที่อาจได้ค่า NULL

### ไวยากรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID          PIC S9(5)  COMP-3.
       01  HV-EMAIL            PIC X(50).
       01  HV-EMAIL-IND        PIC S9(4)  COMP.
           EXEC SQL END DECLARE SECTION END-EXEC.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO HV-CUST-ID.

           EXEC SQL
               SELECT EMAIL
                 INTO :HV-EMAIL :HV-EMAIL-IND
                 FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           IF HV-EMAIL-IND < 0
               DISPLAY "EMAIL IS NULL - NOT PROVIDED."
           ELSE
               DISPLAY "EMAIL: " HV-EMAIL
           END-IF.
```

### กฎการตีความค่า Indicator Variable

| ค่าใน Indicator Variable | ความหมาย |
|---|---|
| `0` หรือมากกว่า | ค่าคอลัมน์ปกติ ไม่ใช่ NULL (ถ้ามากกว่า 0 แปลว่าถูกตัดทอน — string ยาวเกินความกว้างของ Host Variable) |
| `-1` (ค่าติดลบ) | คอลัมน์นั้นเป็น **NULL** — ค่าใน Host Variable ที่จับคู่กันไม่มีความหมายใด ๆ ห้ามนำไปใช้ต่อ |

### การส่งค่า NULL เข้า DB2 (INSERT/UPDATE)

Indicator Variable ใช้ได้ทั้งสองทิศทาง หากต้องการ **INSERT ค่า NULL** เข้าคอลัมน์ ให้ตั้งค่า
Indicator เป็น `-1` ก่อนแล้วอ้างคู่กันในประโยค SQL:

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           MOVE -1 TO HV-EMAIL-IND.

           EXEC SQL
               INSERT INTO CUSTOMER (CUST_ID, CUST_NAME, EMAIL)
               VALUES (:HV-CUST-ID, :HV-CUST-NAME, :HV-EMAIL :HV-EMAIL-IND)
           END-EXEC.
```

### ข้อควรระวัง

- **ถ้าคอลัมน์อาจเป็น NULL แต่โปรแกรมลืมใส่ Indicator Variable** เมื่อ `SELECT` แล้วเจอค่า NULL จริง
  จะได้ `SQLCODE = -305` ทันที (ข้อผิดพลาดร้ายแรงระดับ compile-time ไม่ได้ แต่รันไทม์แน่นอน) จึงควร
  ตรวจสอบเสมอว่าคอลัมน์ที่ประกาศใน DB2 อนุญาต NULL หรือไม่ (ดูจาก `DECLARE TABLE` หรือ Data
  Dictionary) แล้วเติม Indicator Variable ให้ครบทุกคอลัมน์ที่จำเป็น
- Indicator Variable ต้องเป็น `PIC S9(4) COMP` (halfword binary) เสมอตามมาตรฐาน ไม่ควรใช้ชนิดอื่น

### แบบฝึกหัดที่ 578.1

**โจทย์**: หาก `SELECT` คอลัมน์ `PHONE_NUMBER` ที่อนุญาต NULL แต่ลืมประกาศ Indicator Variable จะเกิด
อะไรขึ้นเมื่อข้อมูลจริงในแถวนั้นเป็น NULL

**เฉลย**: จะได้ `SQLCODE = -305` (บ่งบอกว่า "the value is null but no indicator variable is provided")
โปรแกรมจะไม่สามารถอ่านค่าคอลัมน์นั้นได้เลยและ Host Variable ที่รับค่าจะไม่ถูกกำหนดค่าใหม่ที่เชื่อถือได้
วิธีแก้คือเพิ่ม Indicator Variable คู่กับ Host Variable นั้นเสมอ แล้วตรวจสอบค่า Indicator ก่อนใช้งาน
ค่าใน Host Variable ทุกครั้ง

---

## ขั้นตอนที่ 579: รูปแบบการตรวจสอบ SQLCODE อย่างเป็นระบบ (Error-Checking Paragraph)

### ปัญหาของการตรวจสอบ SQLCODE แบบกระจัดกระจาย

หากโปรแกรมมี `EXEC SQL` หลายสิบจุด การเขียน `EVALUATE SQLCODE` ซ้ำ ๆ ทุกจุดทำให้โค้ดยาวและซ้ำซ้อนมาก
แนวทางที่นิยมในโลกจริงคือการรวมตรรกะตรวจสอบไว้ใน **Paragraph กลาง** แล้วเรียกผ่าน `PERFORM` หลังทุก
คำสั่ง SQL ที่สำคัญ

### รูปแบบมาตรฐาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

       01  WS-ABEND-SWITCH      PIC X VALUE "N".
           88  WS-SQL-ERROR     VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM FETCH-CUSTOMER-PARA.
           IF WS-SQL-ERROR
               DISPLAY "STOPPING DUE TO SQL ERROR."
               STOP RUN.
           END-IF.
           DISPLAY "PROCESSING CONTINUES NORMALLY."
           STOP RUN.

       FETCH-CUSTOMER-PARA.
           MOVE 10001 TO HV-CUST-ID.
           EXEC SQL
               SELECT CUST_NAME
                 INTO :HV-CUST-NAME
                 FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.
           PERFORM CHECK-SQLCODE-PARA.

       CHECK-SQLCODE-PARA.
           EVALUATE SQLCODE
               WHEN 0
                   CONTINUE
               WHEN 100
                   DISPLAY "SQL: RECORD NOT FOUND."
               WHEN OTHER
                   DISPLAY "SQL ERROR " SQLCODE
                       " AT " WS-CURRENT-PARA-NAME
                   DISPLAY "SQLSTATE: " SQLSTATE
                   DISPLAY "MESSAGE : " SQLERRMC
                   SET WS-SQL-ERROR TO TRUE
           END-EVALUATE.
```

### เทคนิคเสริม: บันทึก "จุดที่กำลังทำงาน" ก่อนแต่ละ EXEC SQL

ปัญหาสำคัญของ `CHECK-SQLCODE-PARA` แบบกลางคือ **ไม่รู้ว่า error เกิดจาก EXEC SQL จุดไหน** เทคนิคที่
นิยมใช้แก้ปัญหานี้คือการ `MOVE` ชื่อ paragraph หรือหมายเลขจุดเข้าตัวแปรก่อนเรียก SQL ทุกครั้ง แล้วให้
`CHECK-SQLCODE-PARA` แสดงค่านั้นออกมาด้วยเมื่อพบ error (ดังตัวอย่าง `WS-CURRENT-PARA-NAME` ข้างต้น)
เพื่อช่วยระบุจุดเกิดปัญหาได้รวดเร็วเมื่อไล่ debug ระบบขนาดใหญ่ที่มี EXEC SQL หลายร้อยจุด

### ข้อควรระวัง

- แนวทางนี้ (centralized error check) เป็น**รูปแบบยอดนิยมในทางปฏิบัติ** แต่ไม่ใช่มาตรฐานบังคับของ
  ภาษา — บางองค์กรใช้ `WHENEVER` แทน (จะสอนใน Part 059 ขั้นตอนที่ 586) ซึ่งเป็นวิธีการแบบ declarative
  มากกว่า แต่ละองค์กรมักมี "มาตรฐานการเขียน" (Coding Standard) ของตัวเองที่ต้องปฏิบัติตามเมื่อทำงานจริง
- ระวังอย่าให้ `CHECK-SQLCODE-PARA` เรียก `STOP RUN` ตรง ๆ ภายใน Paragraph กลาง เพราะจะทำให้ผู้เรียก
  ไม่มีโอกาส cleanup ทรัพยากร (เช่นปิด Cursor ที่เปิดค้างอยู่) ก่อนโปรแกรมจบ ควรตั้ง flag แล้วให้ผู้เรียก
  ตัดสินใจเองว่าจะจบโปรแกรมอย่างไร (ตามที่แสดงในตัวอย่างด้วย `WS-SQL-ERROR`)

### แบบฝึกหัดที่ 579.1

**โจทย์**: จงอธิบายข้อดีของการรวมตรรกะตรวจสอบ `SQLCODE` ไว้ใน Paragraph กลาง เทียบกับการเขียน
`EVALUATE SQLCODE` ซ้ำทุกจุด

**เฉลย**: ข้อดีหลักคือ **ลดความซ้ำซ้อนของโค้ด (DRY - Don't Repeat Yourself)** ทำให้แก้ไขตรรกะการ
รายงาน error เพียงจุดเดียวมีผลกับทุกจุดที่เรียก SQL ในโปรแกรม, ช่วยให้มาตรฐานการรายงาน error สม่ำเสมอ
กันทั้งโปรแกรม (ข้อความเดียวกัน รูปแบบเดียวกัน), และง่ายต่อการบำรุงรักษาระยะยาวเมื่อโปรแกรมมีขนาดใหญ่
และมีจุดเรียก SQL จำนวนมาก

---

## ขั้นตอนที่ 580: ภาพรวมวงจร Precompile-Bind-Compile และตำแหน่งใน JCL

### ทบทวนเชื่อมโยงกับ Part 052–053

Part 052–053 สอนเรื่อง JCL (JOB, EXEC, DD Statement) สำหรับสั่งรันโปรแกรม COBOL แบบ Batch บน
Mainframe ขั้นตอนนี้จะแสดงให้เห็นว่า JCL Job ที่ต้องคอมไพล์โปรแกรมที่มี Embedded SQL มีหน้าตาต่างจาก
Job คอมไพล์ COBOL ธรรมดาอย่างไร (แสดงเป็น **ไวยากรณ์อ้างอิง** เพื่อความเข้าใจภาพรวม ไม่ใช่โค้ดที่
รันได้ในสภาพแวดล้อมนี้ เนื่องจากไม่มี JCL Interpreter/z/OS จริงเช่นเดียวกับที่ระบุไว้ตั้งแต่ Part 052)

```
//COBSQL   JOB (ACCT),'DB2 COBOL COMPILE',CLASS=A,MSGCLASS=X
//*------------------------------------------------------------
//* STEP 1: DB2 PRECOMPILE - ประมวลผล EXEC SQL, สร้าง DBRM
//*------------------------------------------------------------
//PC       EXEC PGM=DSNHPC,PARM='HOST(COBOL),SOURCE'
//STEPLIB  DD DSN=DB2.SDSNLOAD,DISP=SHR
//SYSIN    DD DSN=&SYSUID..SRC(CUSTPGM),DISP=SHR
//SYSCIN   DD DSN=&&MODCOB,DISP=(NEW,PASS),
//            SPACE=(TRK,(5,5)),UNIT=SYSDA
//DBRMLIB  DD DSN=&SYSUID..DBRMLIB(CUSTPGM),DISP=SHR
//*------------------------------------------------------------
//* STEP 2: COBOL COMPILE - คอมไพล์ Modified Source ปกติ
//*------------------------------------------------------------
//COB      EXEC PGM=IGYCRCTL
//SYSIN    DD DSN=&&MODCOB,DISP=(OLD,DELETE)
//SYSLIN   DD DSN=&&OBJSET,DISP=(NEW,PASS),
//            SPACE=(TRK,(5,5)),UNIT=SYSDA
//*------------------------------------------------------------
//* STEP 3: BIND - ผูก DBRM เข้ากับ DB2
//*------------------------------------------------------------
//BIND     EXEC PGM=IKJEFT01,DYNAMNBR=20,COND=(4,LT,COB)
//STEPLIB  DD DSN=DB2.SDSNLOAD,DISP=SHR
//SYSTSPRT DD SYSOUT=*
//SYSTSIN  DD *
  DSN SYSTEM(DB2A)
  BIND PACKAGE(CUSTPKG) MEMBER(CUSTPGM) -
       ISOLATION(CS) VALIDATE(BIND) ACTION(REPLACE)
  END
/*
//*------------------------------------------------------------
//* STEP 4: LINK-EDIT - รวมเป็น Load Module ที่รันได้จริง
//*------------------------------------------------------------
//LKED     EXEC PGM=IEWL,COND=(4,LT,COB)
//SYSLIN   DD DSN=&&OBJSET,DISP=(OLD,DELETE)
//         DD DDNAME=SYSIN
//SYSLMOD  DD DSN=&SYSUID..LOADLIB(CUSTPGM),DISP=SHR
//SYSIN    DD *
  INCLUDE SYSLIB(DSNELI)
  ENTRY MAIN
/*
```

### อธิบายภาพรวม

- **STEP `PC`**: เรียก `DSNHPC` (DB2 Precompiler) อ่าน source ต้นฉบับที่มี `EXEC SQL` แล้วแยกเป็น 2
  output: (1) `SYSCIN` = Modified COBOL Source ที่ EXEC SQL ถูกแทนที่ด้วย CALL แล้ว (2) `DBRMLIB` =
  DBRM สำหรับใช้ BIND
- **STEP `COB`**: คอมไพล์ `SYSCIN` (ไม่ใช่ source ต้นฉบับ!) ด้วย COBOL Compiler ตามปกติทุกประการ
  เพราะ ณ จุดนี้ไม่มี `EXEC SQL` หลงเหลือให้สับสนอีกแล้ว
- **STEP `BIND`**: ใช้คำสั่ง `DSN` (DB2 command processor ภายใต้ TSO) เพื่อสั่ง `BIND PACKAGE`
  ผูก DBRM เข้ากับ DB2 Catalog สร้าง Access Path ที่ DB2 จะใช้ตอนรันจริง
- **STEP `LKED`**: Link-edit รวม Object Module เข้ากับ DB2 Runtime Library (`DSNELI` เป็นตัวอย่าง
  หนึ่งของ DB2 Language Interface module) กลายเป็น Load Module พร้อมรัน

### ข้อควรระวัง

- `COND=(4,LT,COB)` ใน STEP หลัง ๆ หมายถึง "ข้าม STEP นี้ถ้า Condition Code ของ STEP `COB` มีค่า
  น้อยกว่า 4 เป็นเท็จ" (ตรรกะ Condition Code ของ JCL ได้สอนละเอียดใน Part 053) การมี Condition Code
  ควบคุมเช่นนี้สำคัญมาก เพราะถ้า Compile ล้มเหลว (RC สูง) ก็ไม่ควรเดินหน้าไป Bind/Link-edit ต่อ
- ลำดับ STEP ต้องเป็น **Precompile → Compile → Bind → Link-edit เสมอ** สลับลำดับไม่ได้ เพราะแต่ละ
  STEP อาศัย output ของ STEP ก่อนหน้าโดยตรง (Bind ต้องการ DBRM จาก Precompile, Link-edit ต้องการ
  Object Module จาก Compile)

### แบบฝึกหัดที่ 580.1

**โจทย์**: จงอธิบายว่าทำไม STEP การคอมไพล์ (`COB`) ในตัวอย่าง JCL ข้างต้นจึงอ่าน `SYSCIN`
(output ของ Precompiler) แทนที่จะอ่านไฟล์ source ต้นฉบับโดยตรง

**เฉลย**: เพราะ COBOL Compiler ธรรมดา (`IGYCRCTL`) **ไม่รู้จักไวยากรณ์ `EXEC SQL`** เลย (เช่นเดียวกับ
ที่ GnuCOBOL ธรรมดาไม่รู้จักตามที่พิสูจน์ไว้ในกรอบคำเตือนต้น Part) หากป้อน source ต้นฉบับที่ยังมี
`EXEC SQL` อยู่ตรง ๆ จะเกิด syntax error ทันที ดังนั้นต้องให้ DB2 Precompiler (`DSNHPC`) แปลง
`EXEC SQL` ทั้งหมดให้กลายเป็น COBOL ปกติ (บวก CALL ไปยัง DB2 runtime) ก่อน แล้วจึงส่ง Modified Source
(`SYSCIN`) นี้ให้ COBOL Compiler ประมวลผลต่อ

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้:

- Embedded SQL คืออะไร และวงจร Precompile → Compile → Bind → Link-edit ที่แตกต่างจากโปรแกรม
  COBOL ธรรมดา
- ไวยากรณ์พื้นฐาน `EXEC SQL ... END-EXEC` และตำแหน่งที่วางได้ในโปรแกรม
- `INCLUDE SQLCA` และการตรวจสอบผลลัพธ์ผ่าน `SQLCODE` — หัวใจของการเขียน Embedded SQL ที่ปลอดภัย
- Host Variable และเครื่องหมาย `:` ที่เชื่อม COBOL กับ SQL เข้าด้วยกัน (พร้อมทดสอบจริงว่าโครงสร้าง
  ข้อมูลที่อยู่เบื้องหลังเป็น COBOL มาตรฐานทุกประการ)
- `DECLARE TABLE` เพื่อช่วย Precompiler ตรวจสอบชนิดข้อมูลล่วงหน้า
- `SELECT INTO` สำหรับดึงข้อมูลแถวเดียว และ `INSERT`/`UPDATE`/`DELETE` แบบฝังตัว
- การจัดการค่า NULL ด้วย Indicator Variable ซึ่งเป็นสิ่งที่ COBOL ไม่มีแนวคิดในตัวเอง
- รูปแบบการตรวจสอบ `SQLCODE` อย่างเป็นระบบผ่าน Paragraph กลาง
- ภาพรวมของ JCL ที่ใช้ Precompile-Compile-Bind-Link-edit โปรแกรมที่มี Embedded SQL

สิ่งสำคัญที่สุดที่ต้องจำจาก Part นี้คือ **Embedded SQL ต้องอาศัย DB2 Precompiler จริงเสมอ** —
ไม่มีทางลัดใด ๆ ที่จะคอมไพล์โค้ดเหล่านี้ด้วย GnuCOBOL ธรรมดาได้ ทุกตัวอย่างที่แสดงไว้จึงเป็นไวยากรณ์
อ้างอิงมาตรฐาน IBM DB2 เพื่อการเรียนรู้และเตรียมพร้อมสำหรับงานจริงบน Mainframe

Part 059 จะต่อยอดจากนี้ไปสู่ปัญหาที่ `SELECT INTO` ทำไม่ได้: การประมวลผล**หลายแถว**พร้อมกันด้วย
**DB2 Cursor**

**[← กลับไป Part 057: DB2 และ SQL เบื้องต้นสำหรับ COBOL](part-057-db2-sql-basics.md)**
**[ไปยัง Part 059: DB2 Cursor และการประมวลผลหลายแถว →](part-059-db2-cursor.md)**
