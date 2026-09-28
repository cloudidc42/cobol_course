# Part 060: DB2 Stored Procedures ด้วย COBOL (ขั้นตอนที่ 591–600)

## คำนำของ Part นี้

Part 058–059 สอนให้เราเขียนโปรแกรม COBOL ที่ "ไปดึงข้อมูล" จาก DB2 มาประมวลผลในโปรแกรม (ผ่าน
Embedded SQL และ Cursor) แนวทางนี้เหมาะกับงาน Batch เป็นอย่างยิ่ง แต่ในโลกของระบบ Online/Transaction
(ที่จะเรียนเต็มรูปแบบใน Part 061 เรื่อง CICS) การส่งข้อมูลไปมาระหว่างโปรแกรมกับฐานข้อมูลซ้ำ ๆ ผ่าน
เครือข่ายมีต้นทุนสูง แนวคิดที่แก้ปัญหานี้คือ **Stored Procedure** — การย้าย**ตรรกะบางส่วน**ไปทำงาน
"ใกล้ข้อมูล" ที่สุด คือรันอยู่บน DB2 Engine เองโดยตรง แล้วให้โปรแกรมเรียกเพียงครั้งเดียวผ่านคำสั่ง
`CALL`

จุดที่น่าสนใจคือ DB2 for z/OS อนุญาตให้เขียน Stored Procedure **ด้วยภาษา COBOL** ได้โดยตรง (ไม่ต้อง
ใช้ SQL PL หรือภาษาอื่น) ทำให้ทักษะ COBOL ที่เรียนมาตลอดหลักสูตรนี้นำไปใช้เขียน Stored Procedure ได้
ทันที Part นี้จะพาคุณเรียนรู้ตั้งแต่การ `CREATE PROCEDURE` การส่งพารามิเตอร์แบบ IN/OUT/INOUT ไปจนถึง
การเรียกใช้งานจากทั้งโปรแกรม COBOL อื่นและจาก SQL โดยตรง

> **ย้ำเตือนเรื่องข้อจำกัดสภาพแวดล้อม**: เช่นเดียวกับ Part 058-059 คำสั่ง `CREATE PROCEDURE`,
> `EXEC SQL CALL`, และ SQL `CALL` เป็นฟีเจอร์ของ DB2 Engine จริงที่ต้องมี DB2 Precompiler และ DB2
> Server ทำงานอยู่จริง — **ไม่มีในสภาพแวดล้อม GnuCOBOL ของหลักสูตรนี้** ตัวอย่างที่มี `EXEC SQL`
> หรือ `CREATE PROCEDURE` ทั้งหมดจึงเป็น **ไวยากรณ์อ้างอิง (Reference Syntax)** ตามมาตรฐาน IBM DB2
> ที่ถูกต้องแม่นยำ แต่**ไม่สามารถคอมไพล์หรือรันได้จริงบนสภาพแวดล้อมนี้**
>
> อย่างไรก็ตาม **กลไกการส่งพารามิเตอร์ IN/OUT/INOUT ระหว่างโปรแกรม COBOL** (ขั้นตอนที่ 593-595)
> เป็นกลไก COBOL มาตรฐานล้วน ๆ (`LINKAGE SECTION` + `CALL ... USING`) ที่**ไม่ต้องพึ่งพา DB2 เลย**
> เราจึงสาธิตส่วนนี้ด้วยโปรแกรม COBOL สองตัวที่**คอมไพล์และรันได้จริง**บน GnuCOBOL เพื่อแสดงกลไกการ
> ส่งพารามิเตอร์ที่ DB2 Stored Procedure ใช้จริงเบื้องหลัง — จุดใดสาธิตแบบนี้จะระบุไว้ชัดเจนว่า
> **"ทดสอบจริงแล้ว (จำลองกลไกพารามิเตอร์)"** เพื่อไม่ให้สับสนกับการอ้างว่าได้ทดสอบ DB2 Stored
> Procedure จริง ซึ่งไม่เกิดขึ้น

---

## ขั้นตอนที่ 591: Stored Procedure คืออะไร และทำไมต้องใช้

### แนวคิด

**Stored Procedure** คือโปรแกรมที่ถูก**คอมไพล์และเก็บไว้ใน DB2 Server เอง** (ไม่ใช่รันบนเครื่อง
Client หรือ Mainframe batch job แยกต่างหาก) เมื่อโปรแกรมอื่นต้องการใช้งาน จะเรียกผ่านคำสั่ง SQL
`CALL procedure-name(...)` เพียงครั้งเดียว แล้ว DB2 จะรันตรรกะทั้งหมดที่อยู่ข้างในให้ (ซึ่งอาจมีคำสั่ง
SQL หลายสิบคำสั่งข้างใน) แล้วส่งผลลัพธ์กลับมาในคำตอบเดียว

### ปัญหาที่ Stored Procedure แก้ไข: Network Round Trip

ลองเปรียบเทียบสถานการณ์ที่โปรแกรม Client ต้องดึงยอดคงเหลือ ตรวจสอบเครดิต และบันทึก log การเข้าถึง
3 ขั้นตอน:

```
แบบไม่มี Stored Procedure (3 round trip ผ่านเครือข่าย):
  Client --SELECT BALANCE-->        DB2
  Client <--ผลลัพธ์-------          DB2
  Client --SELECT CREDIT LIMIT-->   DB2
  Client <--ผลลัพธ์-------          DB2
  Client --INSERT LOG-->            DB2
  Client <--ผลลัพธ์-------          DB2

แบบมี Stored Procedure (1 round trip เท่านั้น):
  Client --CALL CHECK-CREDIT(...)--> DB2 (รันครบทั้ง 3 ขั้นตอนภายใน Engine เอง)
  Client <--ผลลัพธ์สุดท้าย---------  DB2
```

ในระบบธนาคารที่มีธุรกรรมหลายล้านรายการต่อวัน การลด round trip เครือข่ายจาก 3 เหลือ 1 ต่อธุรกรรม
ส่งผลต่อประสิทธิภาพรวมของระบบอย่างมหาศาล

### เหตุผลอื่นที่องค์กรใช้ Stored Procedure

1. **ความปลอดภัย**: ให้สิทธิ์ผู้ใช้ `EXECUTE` เฉพาะ Stored Procedure โดยไม่ต้องให้สิทธิ์เข้าถึงตาราง
   จริงโดยตรง ควบคุมได้ละเอียดกว่า
2. **ใช้ตรรกะร่วมกัน**: หลายแอปพลิเคชัน (COBOL batch, CICS online, Java web app) เรียก Stored
   Procedure ตัวเดียวกันได้ ลดความเสี่ยงที่ตรรกะทางธุรกิจจะเขียนซ้ำแล้วไม่ตรงกันในแต่ละที่
3. **ประสิทธิภาพ**: ลด Network Traffic และใช้ประโยชน์จากการที่ DB2 Engine "อยู่ใกล้ข้อมูล" ที่สุด

### ทำไมต้องเป็น COBOL

DB2 for z/OS รองรับหลายภาษาสำหรับเขียน Stored Procedure (SQL PL, Java, C) แต่หลายองค์กรที่มีทีม
COBOL อยู่แล้วเลือกเขียนด้วย **COBOL** เพราะ: (1) ทีมมีความเชี่ยวชาญอยู่แล้ว ไม่ต้องฝึกภาษาใหม่
(2) นำโค้ด Business Logic ที่เขียนด้วย COBOL อยู่แล้วมาปรับใช้ซ้ำได้ (3) ประสิทธิภาพสูงเทียบเท่า
โปรแกรม COBOL ทั่วไปบน Mainframe

### แบบฝึกหัดที่ 591.1

**โจทย์**: จงอธิบายว่าทำไมการลดจำนวน Network Round Trip จึงสำคัญมากสำหรับระบบธนาคารที่มีธุรกรรม
จำนวนมหาศาลต่อวัน

**เฉลย**: แต่ละ Round Trip เครือข่ายมี Latency (เวลาหน่วง) ที่คงที่ระดับหนึ่งไม่ว่าข้อมูลจะเล็กแค่ไหน
เมื่อคูณด้วยจำนวนธุรกรรมหลายล้านรายการต่อวัน แม้ Latency ต่อครั้งจะน้อยมาก (เช่น 1-2 มิลลิวินาที)
ผลรวมของการลด Round Trip จาก 3 เหลือ 1 ต่อธุรกรรมจะประหยัดเวลาประมวลผลรวมได้มหาศาล ส่งผลโดยตรงต่อ
Throughput (จำนวนธุรกรรมที่ระบบรองรับได้ต่อวินาที) และประสบการณ์ผู้ใช้ปลายทาง (เช่น ความเร็วในการทำ
ธุรกรรม ATM)

---

## ขั้นตอนที่ 592: CREATE PROCEDURE LANGUAGE COBOL — โครงสร้างพื้นฐาน

### ไวยากรณ์

```sql
-- REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
CREATE PROCEDURE GET_CUSTOMER_BALANCE
    (IN  P-CUST-ID       INTEGER,
     OUT P-BALANCE       DECIMAL(9,2),
     OUT P-RETURN-CODE   INTEGER)
    LANGUAGE COBOL
    PARAMETER STYLE GENERAL
    DETERMINISTIC
    NO SQL
    EXTERNAL NAME GETBALPR
    COLLID CUSTPROC
    WLM ENVIRONMENT PRODWLM;
```

### อธิบายทีละส่วน

| ส่วน | ความหมาย |
|---|---|
| `CREATE PROCEDURE GET_CUSTOMER_BALANCE` | ชื่อ Stored Procedure ที่จะใช้เรียกผ่าน SQL `CALL` |
| `(IN ..., OUT ..., OUT ...)` | รายการพารามิเตอร์พร้อมทิศทาง (จะสอนละเอียดในขั้นตอนที่ 593) |
| `LANGUAGE COBOL` | บอก DB2 ว่าโปรแกรมภาษา COBOL คือตัวที่ implement ตรรกะจริง |
| `PARAMETER STYLE GENERAL` | รูปแบบการส่งพารามิเตอร์มาตรฐาน (มีแบบอื่นเช่น `GENERAL WITH NULLS`
  ที่รองรับ Indicator Variable ของแต่ละพารามิเตอร์ด้วย) |
| `DETERMINISTIC` / `NOT DETERMINISTIC` | บอกว่าถ้าป้อน input เดิม จะได้ output เดิมเสมอหรือไม่
  (ช่วย DB2 ตัดสินใจเรื่อง caching/optimization) |
| `NO SQL` / `CONTAINS SQL` / `READS SQL DATA` / `MODIFIES SQL DATA` | ระบุระดับการเข้าถึงข้อมูล
  ของโปรแกรม (จะอธิบายในขั้นตอนที่ 598) |
| `EXTERNAL NAME GETBALPR` | ชื่อจริงของ Load Module COBOL ที่คอมไพล์ไว้แล้วบน z/OS (สังเกตว่า
  ไม่เกิน 8 ตัวอักษร ตามข้อจำกัดชื่อโปรแกรมของ Mainframe ยุคเก่า) |
| `COLLID CUSTPROC` | Package Collection ID ที่ Stored Procedure นี้ผูกอยู่ (เกี่ยวข้องกับขั้นตอน
  Bind Package ที่เรียนใน Part 058 ขั้นตอนที่ 580) |
| `WLM ENVIRONMENT PRODWLM` | สภาพแวดล้อม Workload Manager ที่ z/OS ใช้แยก Address Space สำหรับรัน
  Stored Procedure นี้โดยเฉพาะ (แยกจาก DB2 Engine หลักเพื่อความปลอดภัยและเสถียรภาพ) |

### ข้อควรระวัง

- `CREATE PROCEDURE` เป็นคำสั่ง SQL DDL ที่รันผ่าน SQL client (เช่น SPUFI หรือเครื่องมือ Admin)
  **ไม่ได้เขียนอยู่ในไฟล์ COBOL source เดียวกันกับ EXTERNAL NAME** — โปรแกรม COBOL ที่ implement
  ตรรกะจริง (`GETBALPR`) ต้องถูกคอมไพล์และวางไว้ใน Load Library ที่ z/OS หาเจอแยกต่างหาก
- `EXTERNAL NAME` ต้องตรงกับชื่อ `PROGRAM-ID` ของโปรแกรม COBOL จริงเป๊ะ ๆ และชื่อ Load Module
  ที่วางไว้ในไลบรารี มิฉะนั้น DB2 จะหาโปรแกรมไม่เจอตอนเรียก `CALL` (`SQLCODE` ที่เกี่ยวข้องกับ
  "routine not found")

### แบบฝึกหัดที่ 592.1

**โจทย์**: จงอธิบายว่าทำไม `CREATE PROCEDURE` จึงต้องแยกคำสั่ง SQL DDL ออกจากไฟล์ COBOL source
ของโปรแกรมจริง แทนที่จะเขียนรวมกันในไฟล์เดียว

**เฉลย**: เพราะ `CREATE PROCEDURE` เป็นคำสั่งที่**ลงทะเบียน metadata ใน DB2 Catalog**
(SYSIBM.SYSROUTINES และตารางที่เกี่ยวข้อง) บอกให้ DB2 รู้ว่ามี Stored Procedure ชื่ออะไร รับ
พารามิเตอร์แบบไหน และ Load Module จริงอยู่ที่ไหน ส่วนไฟล์ COBOL source คือ**ตัวโปรแกรมจริง**ที่ต้อง
ผ่านกระบวนการ Precompile-Compile-Bind-Link-edit (Part 058 ขั้นตอนที่ 580) แยกต่างหากก่อน จึงจะได้
Load Module พร้อมใช้งาน ทั้งสองส่วนทำงานคนละขั้นตอนและคนละเครื่องมือกัน จึงต้องแยกไฟล์กันตามธรรมชาติ
ของสถาปัตยกรรม DB2

---

## ขั้นตอนที่ 593: การส่งพารามิเตอร์: IN, OUT, INOUT

### ความหมายของแต่ละทิศทาง

| ทิศทาง | ความหมาย | เทียบเท่า COBOL Parameter Passing (Part 032) |
|---|---|---|
| `IN` | ค่าที่ส่ง**เข้าไปเท่านั้น** Stored Procedure อ่านได้แต่การแก้ไขค่าภายในจะไม่ส่งกลับมาที่ผู้เรียก | คล้าย `BY CONTENT` |
| `OUT` | ค่าที่ Stored Procedure **กำหนดค่าใหม่**แล้วส่งกลับให้ผู้เรียก ค่าเริ่มต้นที่ส่งเข้ามาไม่มีความหมายและไม่ควรอ่าน | คล้าย output-only ผ่าน `BY REFERENCE` |
| `INOUT` | ทั้งส่งค่าเข้าไปให้ Stored Procedure ใช้ **และ** รับค่าที่แก้ไขแล้วกลับมาด้วย | คล้าย `BY REFERENCE` เต็มรูปแบบ |

### ตัวอย่างการประกาศ

```sql
-- REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
CREATE PROCEDURE TRANSFER_FUNDS
    (IN    P-FROM-ACCT      INTEGER,
     IN    P-TO-ACCT        INTEGER,
     INOUT P-AMOUNT         DECIMAL(9,2),
     OUT   P-RETURN-CODE    INTEGER)
    LANGUAGE COBOL
    PARAMETER STYLE GENERAL
    EXTERNAL NAME XFERFUND
    COLLID CUSTPROC;
```

ในตัวอย่างนี้ `P-AMOUNT` เป็น `INOUT` เพราะผู้เรียกส่งจำนวนเงินที่ต้องการโอนเข้าไป แต่ Stored
Procedure อาจปรับค่านี้ก่อนส่งกลับ (เช่น ปัดเศษตามกฎธุรกิจ หรือหักค่าธรรมเนียมแล้วรายงานยอดสุทธิที่
โอนจริงกลับไป)

### การ Map พารามิเตอร์ในฝั่ง COBOL: LINKAGE SECTION

โปรแกรม COBOL ที่เป็น Stored Procedure จริง (`EXTERNAL NAME` ที่อ้างถึง) ต้องรับพารามิเตอร์ทั้งหมดผ่าน
`LINKAGE SECTION` และ `PROCEDURE DIVISION USING` ตามลำดับเดียวกับที่ประกาศใน `CREATE PROCEDURE`
เป๊ะ ๆ (ทบทวนกลไกนี้จาก Part 032 เรื่อง Parameter Passing):

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XFERFUND.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-FROM-ACCT        PIC S9(9)    COMP.
       01  LK-TO-ACCT          PIC S9(9)    COMP.
       01  LK-AMOUNT           PIC S9(7)V99 COMP-3.
       01  LK-RETURN-CODE      PIC S9(9)    COMP.

       PROCEDURE DIVISION
           USING LK-FROM-ACCT, LK-TO-ACCT,
                 LK-AMOUNT, LK-RETURN-CODE.
       MAIN-PARA.
      *> The real fund-transfer logic would go here (using EXEC SQL UPDATE
      *> on both accounts, as covered in Part 058)
           MOVE 0 TO LK-RETURN-CODE.
           GOBACK.
```

### ข้อควรระวัง

- ลำดับพารามิเตอร์ใน `PROCEDURE DIVISION USING` ต้อง**ตรงกับลำดับที่ประกาศใน `CREATE PROCEDURE`
  เป๊ะทุกตัว** ทั้งจำนวนและชนิดข้อมูล — สลับลำดับแม้เพียงสองตัวจะทำให้ค่าที่ได้รับผิดเพี้ยนไปทั้งหมด
  โดยไม่มี compile error เตือนใด ๆ (เพราะ DB2 กับ COBOL Compiler ตรวจสอบกันคนละจุด ไม่ได้ตรวจสอบ
  ไขว้กันโดยอัตโนมัติ)
- โปรแกรม Stored Procedure ควรจบด้วย `GOBACK` ไม่ใช่ `STOP RUN` เพราะ `STOP RUN` จะพยายามคืนสถานะ
  ระดับ Job ทั้งหมด ซึ่งไม่เหมาะกับบริบทของ Stored Procedure ที่ทำงานเป็นส่วนหนึ่งของ DB2 Address
  Space (แนวคิดเดียวกับความแตกต่างระหว่าง `STOP RUN` กับ `GOBACK`/`EXIT PROGRAM` ใน subprogram ที่
  เรียนใน Part 031)

### แบบฝึกหัดที่ 593.1

**โจทย์**: จงอธิบายว่าเมื่อใดควรใช้พารามิเตอร์แบบ `IN`, เมื่อใดควรใช้ `OUT`, และเมื่อใดควรใช้ `INOUT`
พร้อมยกตัวอย่างสถานการณ์ละ 1 ข้อ

**เฉลย**: **`IN`** ใช้เมื่อผู้เรียกต้องการแค่ "ส่งข้อมูลเข้าไปให้ประมวลผล" โดยไม่สนใจว่าค่านั้นจะถูก
แก้ไขภายในหรือไม่ เช่น รหัสลูกค้าที่ใช้ค้นหา (`P-CUST-ID`) **`OUT`** ใช้เมื่อผู้เรียกต้องการแค่ "ผลลัพธ์
ที่คำนวณใหม่" โดยไม่มีค่าเริ่มต้นที่มีความหมายส่งเข้าไปเลย เช่น ยอดคงเหลือที่คำนวณได้ (`P-BALANCE`)
**`INOUT`** ใช้เมื่อทั้งต้องส่งค่าเริ่มต้นเข้าไปให้ตรรกะใช้ **และ** ต้องการค่าที่แก้ไขแล้วกลับมาด้วย
เช่น จำนวนเงินที่ขอโอน ซึ่ง Stored Procedure อาจปรับ (หักค่าธรรมเนียม) แล้วรายงานยอดสุทธิกลับไป
(`P-AMOUNT` ในตัวอย่าง `TRANSFER_FUNDS`)

---

## ขั้นตอนที่ 594: เขียนโปรแกรม COBOL ที่เป็นเนื้อในของ Stored Procedure

### โครงสร้างโปรแกรมฉบับเต็ม

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GETBALPR.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID          PIC S9(9)     COMP.
       01  HV-BALANCE          PIC S9(9)V99  COMP-3.
           EXEC SQL END DECLARE SECTION END-EXEC.

       LINKAGE SECTION.
       01  LK-CUST-ID           PIC S9(9)     COMP.
       01  LK-BALANCE           PIC S9(9)V99  COMP-3.
       01  LK-RETURN-CODE       PIC S9(9)     COMP.

       PROCEDURE DIVISION
           USING LK-CUST-ID, LK-BALANCE, LK-RETURN-CODE.
       MAIN-PARA.
           MOVE LK-CUST-ID TO HV-CUST-ID.

           EXEC SQL
               SELECT BALANCE
                 INTO :HV-BALANCE
                 FROM CUSTOMER
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   MOVE HV-BALANCE TO LK-BALANCE
                   MOVE 0 TO LK-RETURN-CODE
               WHEN 100
                   MOVE 0 TO LK-BALANCE
                   MOVE 4 TO LK-RETURN-CODE
      *> RETURN CODE 4 = by business convention, means "customer not found"
               WHEN OTHER
                   MOVE 0 TO LK-BALANCE
                   MOVE 8 TO LK-RETURN-CODE
      *> RETURN CODE 8 = a system-level error occurred (SQL error)
           END-EVALUATE.

           GOBACK.
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้มี**ทั้ง** `LINKAGE SECTION` (รับพารามิเตอร์จากผู้เรียก DB2) **และ** Host Variable ใน
  `WORKING-STORAGE SECTION` (สำหรับสื่อสารกับ DB2 ผ่าน Embedded SQL) — ทั้งสองกลไกทำงานคู่ขนานกัน
  แยกจากกันชัดเจน: `LINKAGE SECTION` คุยกับ "ผู้เรียก Stored Procedure" ส่วน Host Variable คุยกับ
  "DB2 Engine ที่ Stored Procedure นี้เรียกใช้ต่อ"
- สังเกตว่าค่า Return Code ที่กำหนดเอง (`4`, `8`) เป็นแบบแผนที่**ทีมพัฒนาตกลงกันเอง** ไม่ใช่มาตรฐาน
  ตายตัวของ DB2 (คล้ายกับ Condition Code ของ JCL ที่เรียนใน Part 053 ที่ค่าความหมายขึ้นกับข้อตกลง
  ของแต่ละองค์กร) ผู้เรียก Stored Procedure ต้องรู้ล่วงหน้าว่าค่าใดหมายถึงอะไรผ่านเอกสารการออกแบบ
- โปรแกรม Stored Procedure ที่ดีควรมีตรรกะการตรวจสอบ error ครบถ้วนเหมือนโปรแกรม Batch ทั่วไป
  (ทบทวนแนวทาง Paragraph กลางจาก Part 058 ขั้นตอนที่ 579) เพราะข้อผิดพลาดที่ไม่ถูกจับจะทำให้
  Stored Procedure "เงียบ" ไม่รายงานปัญหาที่แท้จริงให้ผู้เรียกรู้

### ข้อควรระวัง

- Stored Procedure **ไม่ควรมี** `ACCEPT`/`DISPLAY` ที่คาดหวัง terminal โต้ตอบกับผู้ใช้ เพราะทำงานอยู่
  ใน DB2 Address Space โดยไม่มี terminal ที่แท้จริงผูกอยู่ (ต่างจากโปรแกรม Batch ทั่วไปที่อาจใช้
  `DISPLAY` ส่ง log ไปที่ SYSOUT ได้ปกติ) การรายงานผลต้องทำผ่านพารามิเตอร์ `OUT`/`INOUT` เท่านั้น
- ต้องคำนึงถึง **Isolation Level** และ **Commit Scope** ให้ดี — Stored Procedure ที่ทำงานภายใน
  Transaction ของผู้เรียกอาจไม่ควรสั่ง `COMMIT`/`ROLLBACK` เอง (ปล่อยให้ผู้เรียกเป็นผู้ตัดสินใจ)
  เว้นแต่ออกแบบให้เป็น Stored Procedure ที่ทำงานเป็น Transaction อิสระโดยเจตนา

### แบบฝึกหัดที่ 594.1

**โจทย์**: จงอธิบายว่าเหตุใดโปรแกรม Stored Procedure ในตัวอย่างจึงมีทั้ง `LINKAGE SECTION` และ
Host Variable ใน `WORKING-STORAGE SECTION` พร้อมกัน ทั้งที่ดูเหมือนทำหน้าที่คล้ายกัน (รับ-ส่งข้อมูล)

**เฉลย**: ทั้งสองทำหน้าที่รับ-ส่งข้อมูลจริง แต่คนละ "คู่สนทนา" กัน `LINKAGE SECTION` คุยกับ
**ผู้เรียก Stored Procedure** (ซึ่งอาจเป็นโปรแกรม COBOL อื่น หรือ SQL `CALL` โดยตรง) ผ่านกลไก
Parameter Passing มาตรฐานของ COBOL ส่วน Host Variable คุยกับ **DB2 Engine** ที่ Stored Procedure
เรียกใช้ต่อเพื่อดึงข้อมูลจริงจากตาราง (`EXEC SQL SELECT`) ทั้งสองเป็นความสัมพันธ์คนละคู่ที่เกิดขึ้นใน
โปรแกรมเดียวกัน: ผู้เรียก ↔ Stored Procedure (ผ่าน LINKAGE SECTION) และ Stored Procedure ↔ DB2
(ผ่าน Host Variable)

---

## ขั้นตอนที่ 595: เรียก Stored Procedure จากโปรแกรม COBOL ด้วย EXEC SQL CALL

### ไวยากรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID           PIC S9(9)     COMP.
       01  HV-BALANCE           PIC S9(9)V99  COMP-3.
       01  HV-RETURN-CODE       PIC S9(9)     COMP.
           EXEC SQL END DECLARE SECTION END-EXEC.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO HV-CUST-ID.

           EXEC SQL
               CALL GET_CUSTOMER_BALANCE
                   (:HV-CUST-ID, :HV-BALANCE, :HV-RETURN-CODE)
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   DISPLAY "BALANCE: " HV-BALANCE
                   DISPLAY "PROC RETURN CODE: " HV-RETURN-CODE
               WHEN OTHER
                   DISPLAY "CALL FAILED. SQLCODE=" SQLCODE
           END-EVALUATE.
```

### จุดสำคัญที่ต้องแยกให้ชัด: SQLCODE ของ CALL เอง vs Return Code ของ Stored Procedure

จุดที่ผู้เริ่มต้นสับสนบ่อยที่สุดคือ **มี 2 ระดับของ "ผลลัพธ์" ที่ต้องตรวจสอบแยกกัน**:

1. **`SQLCODE`** — บอกว่า**การเรียก `CALL` เอง**สำเร็จหรือไม่ (เช่น หา Stored Procedure เจอไหม,
   ส่งพารามิเตอร์ถูกชนิดไหม, DB2 Engine ทำงานได้ปกติไหม) เป็นเรื่องระดับ "โครงสร้างพื้นฐาน"
2. **`HV-RETURN-CODE`** (พารามิเตอร์ `OUT` ที่ Stored Procedure กำหนดค่าเอง) — บอกว่า**ตรรกะทาง
   ธุรกิจภายใน** Stored Procedure ทำงานเป็นอย่างไร (เช่น พบลูกค้าหรือไม่ ตามที่ออกแบบไว้ในขั้นตอน
   ที่ 594)

ต้องตรวจสอบ `SQLCODE` ก่อนเสมอ (ถ้า `CALL` ล้มเหลวระดับโครงสร้าง พารามิเตอร์ `OUT` ทั้งหมดจะไม่มี
ค่าที่เชื่อถือได้เลย) แล้วจึงตรวจสอบ `HV-RETURN-CODE` ต่อเพื่อดูผลลัพธ์ทางธุรกิจ

### ข้อควรระวัง

- ต้องประกาศ Host Variable ให้ครบทุกพารามิเตอร์ (`IN`, `OUT`, `INOUT`) แม้พารามิเตอร์ `OUT` จะยังไม่
  มีค่าที่มีความหมายก่อนเรียก `CALL` ก็ตาม — DB2 ต้องการ "ที่เก็บ" (storage) สำหรับใส่ค่าที่จะส่งกลับ
  มาเสมอ
- จำนวนและชนิดของ Host Variable ในวงเล็บของ `CALL` ต้องตรงกับที่ `CREATE PROCEDURE` ประกาศไว้ทุก
  ประการ ทั้งจำนวนพารามิเตอร์และลำดับ

### แบบฝึกหัดที่ 595.1

**โจทย์**: หาก `EXEC SQL CALL` ได้ `SQLCODE = 0` แต่ `HV-RETURN-CODE = 4` ควรตีความว่าอย่างไร

**เฉลย**: `SQLCODE = 0` หมายความว่า**การเรียก Stored Procedure สำเร็จในระดับโครงสร้าง** — DB2 หา
Stored Procedure เจอ ส่งพารามิเตอร์ถูกต้อง และตัว Stored Procedure ทำงานจบโดยไม่มี exception ใด ๆ
ส่วน `HV-RETURN-CODE = 4` คือค่าที่ Stored Procedure **จงใจกำหนดเอง**ตามตรรกะทางธุรกิจภายใน (ตาม
ตัวอย่างขั้นตอนที่ 594 ค่า 4 หมายถึง "ไม่พบลูกค้า") ซึ่งไม่ใช่ error ทางเทคนิค แต่เป็นผลลัพธ์ทาง
ธุรกิจปกติที่โปรแกรมผู้เรียกต้องรองรับตามที่ออกแบบไว้ (เช่น แสดงข้อความ "ไม่พบข้อมูลลูกค้า" ให้ผู้ใช้)

---

## ขั้นตอนที่ 596: เรียก Stored Procedure จาก SQL โดยตรง (นอก COBOL)

### บริบทการใช้งาน

Stored Procedure ไม่จำเป็นต้องถูกเรียกจากโปรแกรม COBOL เท่านั้น — เพราะมันถูกเก็บไว้ที่ระดับ DB2
Catalog ผู้ดูแลระบบหรือโปรแกรมเมอร์สามารถเรียกทดสอบได้โดยตรงผ่านเครื่องมือ SQL เช่น **SPUFI**
(SQL Processor Using File Input บน TSO/ISPF ที่แนะนำใน Part 054) หรือเครื่องมือ SQL client อื่น ๆ
โดยไม่ต้องเขียนโปรแกรม COBOL ใหม่เลย

### ไวยากรณ์การเรียกจาก SQL Script

```sql
-- REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
CALL GET_CUSTOMER_BALANCE(10001, ?, ?);
```

เครื่องหมาย `?` ในบริบทนี้คือ **Parameter Marker** ที่บอกเครื่องมือ SQL Client ว่าค่านี้เป็นพารามิเตอร์
`OUT` ที่จะได้รับผลลัพธ์กลับมา (ไม่ใช่ค่าคงที่ที่ส่งเข้าไป) เครื่องมือ Client แต่ละตัวมีวิธีแสดงผลลัพธ์
พารามิเตอร์ `OUT` เหล่านี้แตกต่างกันไปตามความสามารถของตัวเอง (เช่น SPUFI จะแสดงในหน้าจอผลลัพธ์แยก
ต่างหากจากผลลัพธ์ SELECT ปกติ)

### เปรียบเทียบสองวิธีการเรียก

| | จาก COBOL (`EXEC SQL CALL`) | จาก SQL Client โดยตรง (`CALL ... (?,?)`) |
|---|---|---|
| ใช้เพื่อ | Production — โปรแกรมอื่นเรียกใช้ตรรกะธุรกิจจริง | ทดสอบ/debug — ผู้พัฒนาทดลองเรียกดูผลลัพธ์เร็ว ๆ ระหว่างพัฒนา |
| ค่าที่ส่งเข้า | Host Variable (`:HV-xxx`) | ค่าคงที่ตรง ๆ หรือ Parameter Marker |
| ผลลัพธ์ที่ได้ | เก็บใน Host Variable ให้โปรแกรมใช้ต่อ | แสดงบนหน้าจอเครื่องมือ SQL |

### ข้อควรระวัง

- แม้จะเรียกทดสอบผ่าน SQL Client ได้สะดวก แต่**ต้องระวังผลข้างเคียง** (Side Effect) หาก Stored
  Procedure นั้นมีการ `INSERT`/`UPDATE`/`DELETE` ข้อมูลจริงอยู่ข้างใน (เช่นตัวอย่าง `TRANSFER_FUNDS`
  ในขั้นตอนที่ 593) การเรียกทดสอบใน Production Environment โดยไม่ระวังอาจทำให้ข้อมูลจริงเปลี่ยนแปลง
  โดยไม่ตั้งใจ — ควรทดสอบใน Environment แยกต่างหาก (Test/UAT) เสมอ
- สิทธิ์ `EXECUTE` บน Stored Procedure ต้องถูกให้ (`GRANT`) แยกต่างหากจากสิทธิ์การเข้าถึงตารางโดยตรง
  (จะกล่าวถึงในขั้นตอนที่ 599) ผู้ใช้ที่ไม่มีสิทธิ์ `EXECUTE` จะเรียก `CALL` ไม่ได้แม้จะมีสิทธิ์อ่าน
  ตารางที่เกี่ยวข้องก็ตาม

### แบบฝึกหัดที่ 596.1

**โจทย์**: จงอธิบายความแตกต่างระหว่างการเรียก Stored Procedure จากโปรแกรม COBOL กับการเรียกจาก
SQL Client โดยตรง และแต่ละแบบเหมาะกับสถานการณ์ใด

**เฉลย**: การเรียกจากโปรแกรม COBOL ผ่าน `EXEC SQL CALL` เหมาะกับ**การใช้งานจริงใน Production**
เพราะผลลัพธ์ถูกเก็บใน Host Variable ที่โปรแกรมนำไปประมวลผลต่อได้อัตโนมัติ เป็นส่วนหนึ่งของ Flow การ
ทำงานอัตโนมัติของระบบ ส่วนการเรียกจาก SQL Client โดยตรง (เช่น SPUFI) เหมาะกับ**การทดสอบและ debug**
ระหว่างพัฒนา เพราะทำได้รวดเร็วโดยไม่ต้องเขียนโปรแกรม COBOL ใหม่ทุกครั้งที่ต้องการทดลองเรียกดูผลลัพธ์
แต่ไม่เหมาะกับการใช้งานอัตโนมัติจริงเพราะต้องมีคนพิมพ์คำสั่งและอ่านผลลัพธ์เอง

---

## ขั้นตอนที่ 597: Result Set จาก Stored Procedure — WITH RETURN Cursor

### ปัญหา: บางครั้ง Stored Procedure ต้องส่งกลับ "หลายแถว" ไม่ใช่แค่พารามิเตอร์เดี่ยว ๆ

พารามิเตอร์ `OUT`/`INOUT` เหมาะกับการส่งค่าเดี่ยว ๆ กลับ (เช่น ยอดคงเหลือ 1 ค่า) แต่ถ้า Stored
Procedure ต้องการส่งกลับ **รายการทั้งชุด** (เช่น "รายการคำสั่งซื้อทั้งหมดของลูกค้า") ต้องใช้กลไกที่
เรียกว่า **Result Set** ผ่าน Cursor ที่ประกาศด้วย `WITH RETURN`

### ไวยากรณ์ฝั่ง Stored Procedure

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID          PIC S9(9)    COMP.
           EXEC SQL END DECLARE SECTION END-EXEC.

           EXEC SQL
               DECLARE ORDER-RESULT-CURSOR CURSOR
                   WITH RETURN FOR
                   SELECT ORDER_ID, ORDER_DATE, TOTAL_AMOUNT
                     FROM ORDERS
                    WHERE CUST_ID = :HV-CUST-ID
                    ORDER BY ORDER_DATE
           END-EXEC.

       LINKAGE SECTION.
       01  LK-CUST-ID           PIC S9(9)    COMP.

       PROCEDURE DIVISION USING LK-CUST-ID.
       MAIN-PARA.
           MOVE LK-CUST-ID TO HV-CUST-ID.

      *> Simply OPEN the cursor then GOBACK - no need to FETCH/CLOSE here
      *> DB2 will "hand off" this still-open cursor back to the caller
           EXEC SQL OPEN ORDER-RESULT-CURSOR END-EXEC.

           GOBACK.
```

### ไวยากรณ์ฝั่งผู้เรียก (ใน CREATE PROCEDURE ต้องระบุจำนวน Result Set)

```sql
-- REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
CREATE PROCEDURE GET_CUSTOMER_ORDERS
    (IN P-CUST-ID INTEGER)
    LANGUAGE COBOL
    PARAMETER STYLE GENERAL
    DYNAMIC RESULT SETS 1
    EXTERNAL NAME ORDERLST
    COLLID CUSTPROC;
```

`DYNAMIC RESULT SETS 1` บอก DB2 ว่า Stored Procedure นี้จะส่งกลับ Result Set 1 ชุด (เปิด Cursor
1 ตัวค้างไว้ตอนจบแล้วไม่ปิดเอง) ฝั่งผู้เรียกที่เป็นโปรแกรม COBOL อื่นต้องใช้ `ASSOCIATE LOCATORS` และ
`ALLOCATE CURSOR` เพื่อ "รับช่วงต่อ" Cursor นั้นมา `FETCH` ต่อได้

### อธิบายจุดสำคัญ

- ความแตกต่างสำคัญจาก Cursor ทั่วไป (Part 059) คือ Stored Procedure **เปิด Cursor แล้วไม่ปิดเอง**
  (ไม่มี `CLOSE` ก่อน `GOBACK`) เพราะเจตนาคือส่งต่อ Cursor ที่ยัง "มีชีวิต" อยู่ให้ผู้เรียกไปจัดการ
  ต่อ ถ้า Stored Procedure `CLOSE` ก่อนจบ ผู้เรียกจะไม่มีอะไรให้ `FETCH` เลย
- กลไกนี้ซับซ้อนกว่าพารามิเตอร์ `OUT`/`INOUT` ธรรมดามาก และมีความแตกต่างระหว่าง DB2 เวอร์ชันค่อนข้าง
  มาก จึงมักใช้เฉพาะเมื่อจำเป็นต้องส่งข้อมูลหลายแถวกลับจริง ๆ

### ข้อควรระวัง

- Result Set แบบนี้ **ใช้ทรัพยากรค้างไว้ (Cursor ที่ยังเปิดอยู่)** จนกว่าผู้เรียกจะ `CLOSE` เอง
  หากผู้เรียกลืมปิด จะเกิดปัญหาทรัพยากรค้างเช่นเดียวกับที่เตือนไว้ใน Part 059 ขั้นตอนที่ 585 แต่
  รุนแรงกว่าเพราะข้ามขอบเขตระหว่างสองโปรแกรม
- ไม่ใช่ทุกภาษา Client ที่รองรับการรับ Result Set จาก Stored Procedure ได้ง่ายเท่ากัน ควรตรวจสอบ
  เอกสารของ Driver/ภาษาที่ใช้ก่อนออกแบบให้ Stored Procedure ส่งกลับ Result Set แบบนี้

### แบบฝึกหัดที่ 597.1

**โจทย์**: จงอธิบายว่าทำไม Cursor ที่ใช้ส่ง Result Set กลับจาก Stored Procedure จึงไม่ถูก `CLOSE`
ก่อน `GOBACK` ทั้งที่ Part 059 สอนว่าควร `CLOSE` Cursor เสมอ

**เฉลย**: เพราะเจตนาของ Cursor แบบ `WITH RETURN` คือการ**ส่งต่อผลลัพธ์ที่ยังไม่ได้อ่านเลย**ให้ผู้เรียก
ไปประมวลผลต่อ ถ้า Stored Procedure `CLOSE` Cursor ก่อนจบ ผลลัพธ์ทั้งหมดจะถูกทิ้งไปและผู้เรียกจะไม่มี
อะไรให้ `FETCH` เลย คำแนะนำ "ควร CLOSE เสมอ" ใน Part 059 หมายถึง Cursor ที่ใช้**ภายในโปรแกรมเดียวกัน
จนจบการใช้งานแล้ว** ซึ่งต่างจากกรณีนี้ที่ Cursor ถูกออกแบบมาให้ "ส่งต่อ" ข้ามขอบเขตโปรแกรมโดยเจตนา —
ในกรณีนี้ **ผู้เรียก** (ไม่ใช่ Stored Procedure) คือฝ่ายที่มีหน้าที่ `CLOSE` Cursor เมื่อใช้งานเสร็จแล้ว

---

## ขั้นตอนที่ 598: การจัดการ Error ภายใน Stored Procedure และการส่งต่อสถานะ

### ระดับการเข้าถึงข้อมูล (SQL Data Access Clause)

ตอน `CREATE PROCEDURE` (ขั้นตอนที่ 592) มีตัวเลือกที่ต้องระบุเสมอว่าโปรแกรมนี้เข้าถึงข้อมูลระดับใด:

| ค่า | ความหมาย |
|---|---|
| `NO SQL` | ไม่มีคำสั่ง SQL ใด ๆ ภายในโปรแกรมเลย (ตรรกะล้วน ๆ) |
| `CONTAINS SQL` | มีคำสั่ง SQL ที่ไม่เข้าถึงข้อมูล เช่น `SET`, `VALUES` |
| `READS SQL DATA` | อ่านข้อมูล (`SELECT`) แต่ไม่แก้ไขข้อมูลใด ๆ |
| `MODIFIES SQL DATA` | แก้ไขข้อมูลได้ (`INSERT`/`UPDATE`/`DELETE`) |

การระบุระดับที่ถูกต้องช่วยให้ DB2 Optimizer และระบบรักษาความปลอดภัยตัดสินใจได้แม่นยำขึ้น เช่น Stored
Procedure ที่ประกาศเป็น `READS SQL DATA` แต่มีโค้ดพยายาม `UPDATE` ข้างในจะถูก DB2 ปฏิเสธตอนรันจริง
ทันที (ป้องกันการเขียนโค้ดผิดพลาดโดยไม่ตั้งใจ)

### รูปแบบการส่ง Error กลับให้ผู้เรียกอย่างสมบูรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       LINKAGE SECTION.
       01  LK-CUST-ID          PIC S9(9)    COMP.
       01  LK-BALANCE          PIC S9(9)V99 COMP-3.
       01  LK-RETURN-CODE      PIC S9(9)    COMP.
       01  LK-ERROR-MESSAGE    PIC X(80).

       PROCEDURE DIVISION
           USING LK-CUST-ID, LK-BALANCE,
                 LK-RETURN-CODE, LK-ERROR-MESSAGE.
       MAIN-PARA.
           MOVE SPACES TO LK-ERROR-MESSAGE.
           MOVE HV-CUST-ID TO LK-CUST-ID.

           EXEC SQL
               SELECT BALANCE INTO :HV-BALANCE
                 FROM CUSTOMER WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   MOVE HV-BALANCE TO LK-BALANCE
                   MOVE 0 TO LK-RETURN-CODE
               WHEN 100
                   MOVE 4 TO LK-RETURN-CODE
                   STRING "NO CUSTOMER FOUND FOR ID "
                       LK-CUST-ID DELIMITED BY SIZE
                       INTO LK-ERROR-MESSAGE
               WHEN OTHER
                   MOVE 8 TO LK-RETURN-CODE
                   STRING "SQL ERROR " SQLCODE DELIMITED BY SIZE
                       INTO LK-ERROR-MESSAGE
           END-EVALUATE.
           GOBACK.
```

### อธิบายจุดสำคัญ

- นอกจาก Return Code ตัวเลขแล้ว การเพิ่มพารามิเตอร์ `LK-ERROR-MESSAGE` (ข้อความอธิบาย error แบบ
  อ่านง่าย) เป็นแนวปฏิบัติที่ดีมาก เพราะช่วยให้ผู้เรียก (หรือทีม Support) เข้าใจปัญหาได้ทันทีโดยไม่ต้อง
  ไปไล่หาความหมายของตัวเลข Return Code จากเอกสารแยกต่างหาก
- ใช้ `STRING` (Part 019) เพื่อประกอบข้อความ error ที่มีทั้งข้อความคงที่และค่าตัวแปร (เช่น
  `SQLCODE`, `LK-CUST-ID`) เข้าด้วยกันในฟิลด์เดียว

### ข้อควรระวัง

- Stored Procedure ที่เกิด **ข้อผิดพลาดร้ายแรงระดับระบบ** (เช่น ABEND จาก Data Exception, การหารด้วย
  ศูนย์) จะทำให้ Address Space ของ Stored Procedure ล้มทั้งหมด และ DB2 จะรายงาน `SQLCODE` เชิงลบ
  ที่บ่งบอกความล้มเหลวของ Stored Procedure กลับไปยังผู้เรียกโดยอัตโนมัติ (โปรแกรมเมอร์ไม่จำเป็นต้อง
  เขียนโค้ดจับ ABEND เอง) แต่ก็ควรป้องกันไม่ให้เกิด ABEND ตั้งแต่ต้นด้วยการตรวจสอบข้อมูลก่อนคำนวณ
  เสมอ (ทบทวนหลัก Defensive Programming จาก Part 048 เรื่อง Error Handling ขั้นสูง)
- ความยาวของ `LK-ERROR-MESSAGE` ต้องเผื่อพื้นที่ให้พอสำหรับข้อความที่ยาวที่สุดที่เป็นไปได้ มิฉะนั้น
  `STRING` อาจตัดข้อความสำคัญทิ้งไปโดยไม่รู้ตัว (ทบทวนพฤติกรรมของ `STRING` เมื่อพื้นที่ปลายทางไม่พอ
  จาก Part 019)

### แบบฝึกหัดที่ 598.1

**โจทย์**: จงอธิบายประโยชน์ของการเพิ่มพารามิเตอร์ `LK-ERROR-MESSAGE` (ข้อความอธิบาย) นอกเหนือจาก
`LK-RETURN-CODE` (ตัวเลข) ในการออกแบบ Stored Procedure

**เฉลย**: ตัวเลข Return Code เพียงอย่างเดียวบอกได้แค่ "ประเภท" ของปัญหาแบบกว้าง ๆ (เช่น 4 = ไม่พบ
ข้อมูล, 8 = SQL error) แต่ไม่ได้บอกรายละเอียดเฉพาะเจาะจง เช่น ค้นหาด้วย ID อะไรที่ไม่พบ หรือ SQLCODE
ที่แท้จริงคือค่าอะไร การเพิ่มข้อความอธิบายช่วยให้ผู้พัฒนาหรือทีม Support **วิเคราะห์ปัญหาได้เร็วขึ้น
มาก**โดยไม่ต้องเปิดเอกสารแยกไปไล่หาความหมายของตัวเลข Return Code หรือต้องเพิ่ม logging แยกต่างหาก
เพื่อสืบสาเหตุ ซึ่งสำคัญมากในระบบ Production ที่ต้องแก้ปัญหาให้เร็วที่สุด

---

## ขั้นตอนที่ 599: การ Deploy — BIND PACKAGE, GRANT EXECUTE, DB2 Catalog

### ขั้นตอนการนำ Stored Procedure ขึ้นใช้งานจริง

```
(1) เขียนโปรแกรม COBOL (LINKAGE SECTION + EXEC SQL ภายใน)
        |
        v
(2) Precompile + Compile + Bind + Link-edit (เหมือน Part 058 ขั้นตอนที่ 580 ทุกประการ)
    ผลลัพธ์: Load Module พร้อม Package ที่ผูกกับ DB2 แล้ว
        |
        v
(3) CREATE PROCEDURE (ลงทะเบียน metadata ใน DB2 Catalog ชี้ไปที่ Load Module)
        |
        v
(4) GRANT EXECUTE ON PROCEDURE ... TO ผู้ใช้/กลุ่มที่ควรเรียกใช้ได้
        |
        v
(5) พร้อมให้โปรแกรมอื่นเรียกผ่าน CALL
```

### GRANT EXECUTE — ควบคุมสิทธิ์แยกจากสิทธิ์ตาราง

```sql
-- REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
GRANT EXECUTE ON PROCEDURE GET_CUSTOMER_BALANCE TO ROLE_TELLER_APP;
```

จุดสำคัญคือ องค์กรสามารถให้สิทธิ์ผู้ใช้กลุ่ม `ROLE_TELLER_APP` **เรียก** Stored Procedure นี้ได้ โดย
**ไม่ต้องให้สิทธิ์ `SELECT` บนตาราง `CUSTOMER` โดยตรง** — Stored Procedure ทำหน้าที่เป็น "ประตูทาง
เดียว" (Controlled Gateway) ที่จำกัดว่าผู้ใช้แต่ละกลุ่มเข้าถึงข้อมูลได้ผ่านวิธีที่กำหนดไว้ล่วงหน้าเท่านั้น
ไม่สามารถ query ตารางเองแบบอิสระได้

### SYSIBM.SYSROUTINES — ที่ที่ DB2 เก็บ Metadata ของ Stored Procedure

DB2 Catalog มีตารางระบบชื่อ `SYSIBM.SYSROUTINES` ที่เก็บข้อมูลของทุก Stored Procedure/Function
ที่ลงทะเบียนไว้ในระบบ ผู้ดูแลระบบสามารถ query ตารางนี้เพื่อตรวจสอบว่ามี Stored Procedure อะไรบ้าง
ชี้ไปที่ Load Module ใด และใครมีสิทธิ์เรียกใช้:

```sql
-- REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
SELECT ROUTINENAME, LANGUAGE, PARM_COUNT, CREATEDBY
  FROM SYSIBM.SYSROUTINES
 WHERE ROUTINESCHEMA = 'CUSTPROC';
```

### ข้อควรระวัง

- การเปลี่ยนแปลง Stored Procedure ที่ใช้งานอยู่จริง (แก้โค้ด COBOL แล้ว compile ใหม่) ต้องผ่านขั้น
  ตอน Bind/Deploy ใหม่ทั้งหมด และควรมีกระบวนการทดสอบใน Environment แยกก่อนเสมอ เพราะโปรแกรมอื่นที่
  เรียกใช้อยู่จะได้รับผลกระทบทันทีเมื่อ Load Module ถูกแทนที่ (ไม่มีการ "versioning" อัตโนมัติเว้นแต่
  จะออกแบบไว้เอง เช่น สร้างชื่อ Stored Procedure ใหม่ที่มีเวอร์ชันต่อท้าย)
- `GRANT EXECUTE` ที่ไม่ได้จำกัดให้แคบพอ (เช่นให้กับ `PUBLIC` แทนที่จะเป็น Role ที่เจาะจง) อาจเปิด
  ช่องโหว่ความปลอดภัยได้ ควรออกแบบสิทธิ์ตามหลัก Least Privilege เสมอ (ให้สิทธิ์เท่าที่จำเป็นเท่านั้น)

### แบบฝึกหัดที่ 599.1

**โจทย์**: จงอธิบายว่าทำไมการให้สิทธิ์ผ่าน `GRANT EXECUTE ON PROCEDURE` แทนการให้สิทธิ์ `SELECT`
บนตารางโดยตรง จึงช่วยเพิ่มความปลอดภัยให้ระบบ

**เฉลย**: เพราะ Stored Procedure ทำหน้าที่เป็น **ประตูทางเดียวที่ควบคุมได้** — ผู้ใช้ที่มีสิทธิ์
`EXECUTE` เท่านั้นจะเรียกได้ตามตรรกะและเงื่อนไขที่ Stored Procedure กำหนดไว้ล่วงหน้าเท่านั้น
(เช่น ดูได้แค่ยอดคงเหลือของ ID ที่ระบุ ไม่สามารถ query ข้อมูลอื่นในตารางเดียวกันได้) ต่างจากการให้
สิทธิ์ `SELECT` บนตารางโดยตรงที่ผู้ใช้สามารถเขียน SQL query อิสระใด ๆ ก็ได้ตามที่ต้องการ (เช่น ดึง
ข้อมูลทุกคอลัมน์ ทุกแถว โดยไม่มีการควบคุม) ซึ่งเสี่ยงต่อการรั่วไหลของข้อมูลที่ไม่ควรเปิดเผยมากกว่ามาก

---

## ขั้นตอนที่ 600: ตัวอย่างสมบูรณ์ครบวงจร — GetCustomerBalance

### ภาพรวมของระบบทั้งหมด

ขั้นตอนสุดท้ายของ Part นี้จะรวบรวมทุกส่วนที่เรียนมาเป็นตัวอย่างเดียวที่สมบูรณ์: การนิยาม Stored
Procedure, โปรแกรม COBOL ที่ implement ตรรกะ, และโปรแกรม COBOL ที่เรียกใช้ — แสดงทั้งไวยากรณ์อ้างอิง
เต็มรูปแบบ และการสาธิตกลไกพารามิเตอร์ที่ทดสอบจริงได้

### ส่วนที่ 1: CREATE PROCEDURE (ไวยากรณ์อ้างอิง)

```sql
-- REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
CREATE PROCEDURE GET_CUSTOMER_BALANCE
    (IN  P-CUST-ID       INTEGER,
     OUT P-BALANCE       DECIMAL(9,2),
     OUT P-RETURN-CODE   INTEGER)
    LANGUAGE COBOL
    PARAMETER STYLE GENERAL
    READS SQL DATA
    EXTERNAL NAME GETBALPR
    COLLID CUSTPROC
    WLM ENVIRONMENT PRODWLM;
```

### ส่วนที่ 2: โปรแกรม COBOL ฝั่ง Stored Procedure (ไวยากรณ์อ้างอิง — มี EXEC SQL จริง)

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GETBALPR.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.
           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID          PIC S9(9)    COMP.
       01  HV-BALANCE          PIC S9(9)V99 COMP-3.
           EXEC SQL END DECLARE SECTION END-EXEC.

       LINKAGE SECTION.
       01  LK-CUST-ID          PIC S9(9)    COMP.
       01  LK-BALANCE          PIC S9(9)V99 COMP-3.
       01  LK-RETURN-CODE      PIC S9(9)    COMP.

       PROCEDURE DIVISION
           USING LK-CUST-ID, LK-BALANCE, LK-RETURN-CODE.
       MAIN-PARA.
           MOVE LK-CUST-ID TO HV-CUST-ID.
           EXEC SQL
               SELECT BALANCE INTO :HV-BALANCE
                 FROM CUSTOMER WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.
           EVALUATE SQLCODE
               WHEN 0
                   MOVE HV-BALANCE TO LK-BALANCE
                   MOVE 0 TO LK-RETURN-CODE
               WHEN 100
                   MOVE 0 TO LK-BALANCE
                   MOVE 4 TO LK-RETURN-CODE
               WHEN OTHER
                   MOVE 0 TO LK-BALANCE
                   MOVE 8 TO LK-RETURN-CODE
           END-EVALUATE.
           GOBACK.
```

### ส่วนที่ 3: ทดสอบจริง — กลไกพารามิเตอร์ IN/OUT ล้วน ๆ (ไม่มี EXEC SQL)

เพื่อพิสูจน์ว่ากลไกการส่งพารามิเตอร์ IN/OUT ระหว่างโปรแกรม COBOL (LINKAGE SECTION +
`PROCEDURE DIVISION USING` + `CALL ... USING`) ที่ DB2 Stored Procedure ใช้อยู่เบื้องหลังนั้นเป็น
COBOL มาตรฐานแท้ ๆ เราจำลองด้วยโปรแกรมย่อยที่ตัด `EXEC SQL` ออก แทนที่การ `SELECT` จริงด้วยข้อมูล
จำลองในหน่วยความจำ แล้วคอมไพล์และรันจริง:

**โปรแกรมที่ 1 (จำลอง Stored Procedure body — ไม่มี EXEC SQL):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GETBAL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Simulates a database lookup with a fixed in-memory value,
      *> replacing the EXEC SQL SELECT that a real DB2 stored
      *> procedure would use. This proves the parameter-passing
      *> mechanics only, not the DB2 access itself.
       01  WS-SIMULATED-BALANCE PIC S9(7)V99 COMP-3 VALUE 500.00.

       LINKAGE SECTION.
       01  LK-CUST-ID           PIC S9(5)    COMP-3.
       01  LK-BALANCE           PIC S9(7)V99 COMP-3.
       01  LK-RETURN-CODE       PIC S9(4)    COMP.

       PROCEDURE DIVISION USING LK-CUST-ID, LK-BALANCE, LK-RETURN-CODE.
       MAIN-PARA.
           IF LK-CUST-ID = 10001
               MOVE WS-SIMULATED-BALANCE TO LK-BALANCE
               MOVE 0 TO LK-RETURN-CODE
           ELSE
               MOVE 0 TO LK-BALANCE
               MOVE -1 TO LK-RETURN-CODE
           END-IF.
           GOBACK.
```

**โปรแกรมที่ 2 (ผู้เรียก — จำลองผู้เรียก Stored Procedure ด้วย CALL ธรรมดา):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALLERPGM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUST-ID           PIC S9(5)    COMP-3.
       01  WS-BALANCE           PIC S9(7)V99 COMP-3.
       01  WS-RETURN-CODE       PIC S9(4)    COMP.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO WS-CUST-ID.
           CALL "GETBAL" USING WS-CUST-ID, WS-BALANCE, WS-RETURN-CODE.
           DISPLAY "RETURN CODE: " WS-RETURN-CODE.
           DISPLAY "BALANCE    : " WS-BALANCE.

           MOVE 99999 TO WS-CUST-ID.
           CALL "GETBAL" USING WS-CUST-ID, WS-BALANCE, WS-RETURN-CODE.
           DISPLAY "RETURN CODE: " WS-RETURN-CODE.
           DISPLAY "BALANCE    : " WS-BALANCE.
           STOP RUN.
```

คอมไพล์และรันจริง (รวมทั้งสองโปรแกรมเป็น executable เดียว):

```bash
cobc -x -o caller caller.cob getbal.cob
./caller
```

**ผลลัพธ์จริง (ทดสอบแล้ว):**

```
RETURN CODE: +0000
BALANCE    : +0000500.00
RETURN CODE: -0001
BALANCE    : +0000000.00
```

ผลลัพธ์นี้ยืนยันว่ากลไก IN (`WS-CUST-ID`/`LK-CUST-ID`) และ OUT (`WS-BALANCE`, `WS-RETURN-CODE` ที่
`LK-BALANCE`, `LK-RETURN-CODE` เขียนกลับมา) ทำงานถูกต้องตามที่คาดหวังทุกประการ — เมื่อ `CUST-ID`
เป็น `10001` (พบข้อมูล) ได้ `RETURN CODE = 0` และยอดคงเหลือ `500.00` กลับมา ส่วน `CUST-ID` เป็น
`99999` (ไม่พบ) ได้ `RETURN CODE = -1` และยอด `0.00` ตามตรรกะที่ออกแบบไว้ — นี่คือกลไกเดียวกันเป๊ะ ๆ
กับที่ DB2 ใช้ส่งผ่านพารามิเตอร์ `IN`/`OUT` ระหว่างผู้เรียกกับ Stored Procedure จริง เพียงแต่ในโลก
DB2 จริง การเรียกจะผ่าน `EXEC SQL CALL` แทน `CALL` ธรรมดา และ Stored Procedure จะมี `EXEC SQL
SELECT` แทนการจำลองด้วย `WS-SIMULATED-BALANCE`

### ข้อควรระวัง

- อย่าสับสนระหว่างสิ่งที่ทดสอบจริงได้ (กลไกพารามิเตอร์ COBOL-to-COBOL) กับสิ่งที่เป็นไวยากรณ์อ้างอิง
  เท่านั้น (`CREATE PROCEDURE`, `EXEC SQL CALL`, การเชื่อมต่อ DB2 จริง) — ทั้งสองส่วนมีความสำคัญคนละ
  ระดับ: กลไกพารามิเตอร์คือ "หัวใจ" ที่ทำให้ Stored Procedure ทำงานได้ ส่วนกลไก DB2/SQL คือ "เปลือก"
  ที่ห่อหุ้มให้มันทำงานในบริบทฐานข้อมูลได้จริง
- ในงานจริงบน Mainframe อย่าลืมว่าการ deploy Stored Procedure ต้องผ่านขั้นตอนครบทั้งหมดตามขั้นตอน
  ที่ 599 (Precompile-Compile-Bind-CREATE PROCEDURE-GRANT) ซึ่งซับซ้อนกว่าการคอมไพล์โปรแกรม COBOL
  ธรรมดามาก

### แบบฝึกหัดที่ 600.1

**โจทย์**: จงสรุปว่าองค์ประกอบใดของตัวอย่าง `GetCustomerBalance` ในขั้นตอนนี้เป็น "ไวยากรณ์อ้างอิง
เท่านั้น" และองค์ประกอบใด "ทดสอบจริงแล้ว" พร้อมอธิบายเหตุผล

**เฉลย**: **ไวยากรณ์อ้างอิงเท่านั้น**: (1) คำสั่ง `CREATE PROCEDURE` เพราะเป็นคำสั่ง DDL ของ DB2
ที่ต้องมี DB2 Engine จริงรันอยู่ (2) โปรแกรม `GETBALPR` ฉบับเต็มที่มี `EXEC SQL SELECT` เพราะต้องมี
DB2 Precompiler แปลงให้ก่อนจึงจะคอมไพล์ผ่าน (พิสูจน์แล้วว่า GnuCOBOL ธรรมดา compile ไม่ผ่านใน
Part 058) **ทดสอบจริงแล้ว**: กลไกการส่งพารามิเตอร์ IN/OUT ระหว่างโปรแกรม `CALLERPGM` กับ `GETBAL`
(เวอร์ชันจำลองที่ตัด EXEC SQL ออก) เพราะเป็น COBOL มาตรฐานล้วน ๆ ที่ใช้ `LINKAGE SECTION` และ
`CALL ... USING` ซึ่งไม่ต้องพึ่งพา DB2 เลย และคอมไพล์+รันได้จริงบน GnuCOBOL ดังที่แสดงผลลัพธ์ไว้

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้:

- Stored Procedure คืออะไร และเหตุผลที่องค์กรใช้ (ลด Network Round Trip, ความปลอดภัย, ใช้ตรรกะร่วม)
- `CREATE PROCEDURE LANGUAGE COBOL` และความหมายของแต่ละ clause
- การส่งพารามิเตอร์แบบ `IN`, `OUT`, `INOUT` และการ map เข้ากับ `LINKAGE SECTION` ของ COBOL
- โครงสร้างโปรแกรม COBOL ที่เป็นเนื้อในของ Stored Procedure (LINKAGE SECTION + Host Variable
  ทำงานคู่ขนานกัน)
- การเรียก Stored Procedure จากโปรแกรม COBOL อื่นผ่าน `EXEC SQL CALL` และการแยกตรวจ `SQLCODE`
  กับ Return Code ของตัว Stored Procedure เอง
- การเรียกจาก SQL Client โดยตรงเพื่อการทดสอบ
- Result Set จาก Stored Procedure ผ่าน Cursor แบบ `WITH RETURN`
- การจัดการ Error และการออกแบบการรายงานสถานะที่สมบูรณ์กลับให้ผู้เรียก
- ขั้นตอนการ Deploy: BIND, CREATE PROCEDURE, GRANT EXECUTE, และ DB2 Catalog
- ตัวอย่างสมบูรณ์ครบวงจร พร้อมการทดสอบจริงของกลไกพารามิเตอร์ที่ไม่ต้องพึ่งพา DB2

Part 058-060 ได้ปูพื้นฐานความรู้ DB2/COBOL ที่ครบถ้วนสำหรับงาน Batch และ Backend Logic แล้ว
Part 061 จะเปลี่ยนบริบทไปสู่โลกของ **CICS** — เทคโนโลยี Transaction Processing แบบ Online ที่ทำให้
ผู้ใช้ปลายทาง (เช่น พนักงานธนาคารที่หน้าเคาน์เตอร์) โต้ตอบกับระบบ COBOL/DB2 แบบ Real-time ได้

**[← กลับไป Part 059: DB2 Cursor และการประมวลผลหลายแถว](part-059-db2-cursor.md)**
**[ไปยัง Part 061: CICS เบื้องต้น: Transaction Processing →](part-061-cics-intro.md)**
