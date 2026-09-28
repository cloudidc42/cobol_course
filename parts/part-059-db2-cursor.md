# Part 059: DB2 Cursor และการประมวลผลหลายแถว (ขั้นตอนที่ 581–590)

## คำนำของ Part นี้

Part 058 จบลงด้วยข้อจำกัดสำคัญของ `SELECT INTO`: มันใช้ได้เฉพาะเมื่อผลลัพธ์มี**แถวเดียวเท่านั้น**
ถ้าเงื่อนไข `WHERE` ให้ผลลัพธ์มากกว่า 1 แถว โปรแกรมจะได้ `SQLCODE = -811` ทันที แต่ในความเป็นจริงงาน
ประมวลผลข้อมูลทางธุรกิจส่วนใหญ่ต้องวนอ่าน**หลายแถว**พร้อมกัน เช่น "แสดงรายการคำสั่งซื้อทั้งหมดของ
ลูกค้าคนหนึ่ง" หรือ "ประมวลผลบัญชีทุกบัญชีที่ยอดคงเหลือต่ำกว่าเกณฑ์" — งานเหล่านี้ต้องใช้กลไกที่เรียกว่า
**Cursor**

Part นี้จะพาคุณเรียนรู้วงจรชีวิตแบบเต็มรูปแบบของ Cursor: `DECLARE CURSOR` → `OPEN` → `FETCH` (วน
หลายรอบ) → `CLOSE` ผสานเข้ากับ `PERFORM` ของ COBOL ที่คุณคุ้นเคยมาตั้งแต่เฟส 1 ของหลักสูตร ไปจนถึง
เทคนิคขั้นสูงอย่าง Cursor ที่แก้ไข/ลบข้อมูลได้โดยตรง (`WHERE CURRENT OF`)

> **ย้ำเตือน**: เนื้อหาทั้งหมดใน Part นี้ยังคงเป็น **Embedded SQL** ซึ่งใช้หลักการเดียวกับ Part 058
> ทุกประการ (ต้องมี DB2 Precompiler) ตัวอย่างโค้ดทุกตัวอย่างที่มี `EXEC SQL` จึงเป็น **ไวยากรณ์อ้างอิง
> (Reference Syntax)** ตามมาตรฐาน IBM DB2 ที่ถูกต้อง แต่**ไม่สามารถคอมไพล์หรือรันบนสภาพแวดล้อม
> GnuCOBOL ของหลักสูตรนี้ได้** เนื่องจากไม่มี DB2 Precompiler และ DB2 Engine จริง (พิสูจน์แล้วด้วยการ
> ทดสอบคอมไพล์จริงใน Part 058) จะกำกับด้วยป้าย `⚠️ REFERENCE SYNTAX` ไว้ทุกจุดเช่นเดิม ส่วนโค้ด
> COBOL ล้วน ๆ ที่จำลองตรรกะ `PERFORM UNTIL` แบบเดียวกับที่ใช้คู่กับ Cursor จะระบุว่า **"ทดสอบจริง
> แล้ว"** เมื่อมีการสาธิตแยกส่วน

---

## ขั้นตอนที่ 581: ทำไมต้องมี Cursor — ข้อจำกัดของ SELECT INTO

### ทบทวนปัญหาจาก Part 058

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               SELECT ORDER_ID, ORDER_DATE
                 INTO :HV-ORDER-ID, :HV-ORDER-DATE
                 FROM ORDERS
                WHERE CUST_ID = :HV-CUST-ID
           END-EXEC.
```

ถ้าลูกค้าคนนี้มีคำสั่งซื้อ 15 รายการ คำสั่งข้างต้นจะได้ `SQLCODE = -811` ทันที เพราะ `SELECT INTO`
ออกแบบมาให้รองรับแค่ 1 แถวเท่านั้น — มันไม่มีกลไกใดที่จะ "รับหลายแถวแล้ววนอ่านทีละแถว" ได้ในตัวเอง

### แนวคิดของ Cursor

**Cursor** คือ "ตัวชี้" (pointer) ที่ DB2 มอบให้โปรแกรมใช้ควบคุมการอ่าน**ผลลัพธ์ชุดหนึ่ง (Result Set)**
ทีละแถว เปรียบเทียบได้กับการอ่านไฟล์ Sequential ที่เรียนใน Part 023–028: คุณ `OPEN` ไฟล์ก่อน แล้ว
`READ` ทีละ record จนกว่าจะถึง `AT END` แล้วจึง `CLOSE` — Cursor ของ DB2 ก็ทำงานคล้ายกันทุกประการ
เพียงแต่ "ไฟล์" ในที่นี้คือผลลัพธ์จากคำสั่ง `SELECT` แทน

| แนวคิดไฟล์ (Part 023-028) | แนวคิด Cursor (DB2) |
|---|---|
| `OPEN INPUT file` | `EXEC SQL OPEN cursor-name END-EXEC` |
| `READ file ... AT END` | `EXEC SQL FETCH cursor-name INTO ... END-EXEC` (ตรวจ `SQLCODE = 100` แทน `AT END`) |
| `CLOSE file` | `EXEC SQL CLOSE cursor-name END-EXEC` |
| นิยาม record layout ใน `FD` | นิยาม column list ใน `DECLARE CURSOR` |

### วงจรชีวิตของ Cursor (ภาพรวม)

```
DECLARE CURSOR   -->   บอก DB2 ว่า Cursor นี้ชื่ออะไร ดึงข้อมูลด้วย SELECT statement ใด
       |                (เป็นเพียงการ "นิยาม" ยังไม่ได้ดึงข้อมูลจริง)
       v
OPEN             -->   สั่ง DB2 เริ่มประมวลผล SELECT statement จริง เตรียมผลลัพธ์ไว้
       |                (ยังไม่ได้อ่านแถวใดเข้าโปรแกรมเลย)
       v
FETCH (วนซ้ำ)     -->   ดึงข้อมูลมาทีละแถวใส่ Host Variable ตรวจ SQLCODE ทุกครั้ง
       |                (SQLCODE = 100 เมื่อหมดแถวแล้ว)
       v
CLOSE            -->   ปิด Cursor คืนทรัพยากรให้ DB2
```

### ข้อควรระวัง

- อย่าสับสน Cursor ของ DB2 กับ "เคอร์เซอร์" ในความหมายของหน้าจอ (เช่น cursor กระพริบบนหน้าจอ Screen
  Section ที่เรียนใน Part 040) — เป็นคนละแนวคิดกันโดยสิ้นเชิง แม้ใช้คำเดียวกัน
- Cursor เหมาะสำหรับผลลัพธ์ที่**คาดว่าจะมีหลายแถว**เท่านั้น หากรู้แน่ชัดว่าจะได้แถวเดียวเสมอ
  (เช่นค้นด้วย Primary Key) ควรใช้ `SELECT INTO` แบบ Part 058 ต่อไป เพราะเขียนโค้ดสั้นกว่าและมี
  overhead น้อยกว่า

### แบบฝึกหัดที่ 581.1

**โจทย์**: จงเปรียบเทียบแนวคิด Cursor ของ DB2 กับการอ่านไฟล์ Sequential ของ COBOL ที่เรียนใน
Part 023 ว่ามีขั้นตอนที่คล้ายกันอย่างไร

**เฉลย**: ทั้งสองมีวงจร 3 ขั้นตอนเหมือนกัน: (1) เปิดแหล่งข้อมูล (`OPEN` ไฟล์ / `OPEN` Cursor)
(2) วนอ่านทีละรายการจนกว่าจะหมด (`READ...AT END` / `FETCH` ตรวจ `SQLCODE = 100`) (3) ปิดแหล่งข้อมูล
(`CLOSE` ไฟล์ / `CLOSE` Cursor) ความแตกต่างหลักคือแหล่งข้อมูลของไฟล์คือดิสก์โดยตรง ส่วนแหล่งข้อมูลของ
Cursor คือผลลัพธ์จากคำสั่ง SQL ที่ DB2 Engine ประมวลผลให้

---

## ขั้นตอนที่ 582: DECLARE CURSOR — นิยาม Cursor และ SELECT ที่ผูกกับมัน

### ไวยากรณ์พื้นฐาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               DECLARE ORDER-CURSOR CURSOR FOR
                   SELECT ORDER_ID, ORDER_DATE, TOTAL_AMOUNT
                     FROM ORDERS
                    WHERE CUST_ID = :HV-CUST-ID
                    ORDER BY ORDER_DATE
           END-EXEC.
```

### อธิบายจุดสำคัญ

- `DECLARE CURSOR` เป็นเพียงการ**นิยาม** — ไม่มีการดึงข้อมูลใด ๆ เกิดขึ้นจริง ณ จุดนี้ เปรียบเสมือน
  การ `SELECT` ของไฟล์ใน `FILE-CONTROL` ที่ต้องมาก่อน `OPEN` จริง
- Host Variable ที่อ้างใน `WHERE` (เช่น `:HV-CUST-ID`) จะถูก "จับค่า" ตอน `OPEN` เท่านั้น
  **ไม่ใช่ตอน DECLARE** ดังนั้นสามารถเปลี่ยนค่า `HV-CUST-ID` ก่อน `OPEN` แต่ละครั้งเพื่อค้นหาลูกค้า
  คนละคนได้ (จะสาธิตในขั้นตอนที่ 590)
- ชื่อ Cursor (`ORDER-CURSOR` ในตัวอย่าง) เป็นชื่อที่โปรแกรมเมอร์ตั้งเอง ไม่ต้องตรงกับชื่อตารางหรือ
  คอลัมน์ใด ๆ และใช้ชื่อนี้อ้างอิงตลอดวงจร `OPEN`/`FETCH`/`CLOSE`
- **ตำแหน่งของ `DECLARE CURSOR`** ต้องอยู่**ก่อน**คำสั่ง `OPEN` ของ Cursor นั้นในลำดับของซอร์สไฟล์
  เสมอ (แต่ไม่จำเป็นต้องอยู่ใน `WORKING-STORAGE SECTION` เท่านั้น — DB2 Precompiler บางเวอร์ชัน
  อนุญาตให้วางใน `PROCEDURE DIVISION` ก่อนจุดใช้งานได้เช่นกัน ขึ้นกับ dialect)

### ORDER BY ใน Cursor

ต่างจาก `SELECT INTO` ที่มักไม่ใช้ `ORDER BY` (เพราะคาดหวังแถวเดียว) Cursor ที่จะวนอ่านหลายแถวมักใช้
`ORDER BY` เสมอเพื่อควบคุมลำดับการประมวลผลให้แน่นอน (Deterministic) — โดยเฉพาะเมื่อผลลัพธ์จะถูกนำไป
แสดงรายงานหรือทำ Control Break (เทียบเคียงกับเทคนิค Control Break ของ Report Writer ใน Part 039)

### ข้อควรระวัง

- `DECLARE CURSOR` ที่ระบุ `WHERE` โดยใช้ Host Variable แต่ยังไม่ได้ `MOVE` ค่าเริ่มต้นให้ Host
  Variable นั้นก่อน `OPEN` จะทำให้ค้นหาด้วยค่าขยะที่ค้างอยู่ในหน่วยความจำ (เหมือนปัญหาเดียวกับที่
  เตือนไว้ใน Part 058 ขั้นตอนที่ 573)
- Cursor หนึ่งตัวผูกกับ SELECT statement เดียวเท่านั้น ถ้าต้องการเปลี่ยนโครงสร้างคอลัมน์ที่ดึงมา ต้อง
  ประกาศ Cursor ใหม่ ไม่สามารถ "แก้ไข" `DECLARE CURSOR` ที่มีอยู่แล้วได้ระหว่างรัน

### แบบฝึกหัดที่ 582.1

**โจทย์**: จงเขียน `DECLARE CURSOR` ชื่อ `LOW-BALANCE-CURSOR` เพื่อดึง `ACCT_ID` และ `BALANCE`
จากตาราง `ACCOUNT` ที่ `BALANCE` น้อยกว่า `HV-THRESHOLD` เรียงจากน้อยไปมาก

**เฉลย**:

```
EXEC SQL
    DECLARE LOW-BALANCE-CURSOR CURSOR FOR
        SELECT ACCT_ID, BALANCE
          FROM ACCOUNT
         WHERE BALANCE < :HV-THRESHOLD
         ORDER BY BALANCE
END-EXEC.
```

---

## ขั้นตอนที่ 583: OPEN Cursor — เริ่มประมวลผลจริง

### ไวยากรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           MOVE 10001 TO HV-CUST-ID.

           EXEC SQL
               OPEN ORDER-CURSOR
           END-EXEC.

           IF SQLCODE NOT = 0
               DISPLAY "FAILED TO OPEN CURSOR. SQLCODE=" SQLCODE
               STOP RUN
           END-IF.
```

### สิ่งที่เกิดขึ้นเบื้องหลังเมื่อ OPEN

เมื่อสั่ง `OPEN` DB2 จะ:

1. อ่านค่า Host Variable ทุกตัวที่อ้างใน `WHERE` ของ `DECLARE CURSOR` (เช่น `:HV-CUST-ID`) ณ
   ขณะนั้นทันที (ไม่ใช่ตอน `DECLARE`)
2. เริ่มดำเนินการ Access Path ที่วางแผนไว้ตอน `BIND` (Part 058 ขั้นตอนที่ 580) เพื่อเตรียมผลลัพธ์
3. **ยังไม่ส่งข้อมูลแถวใดเข้าโปรแกรมเลย** — เป็นเพียงการ "เตรียมพร้อม" ให้ `FETCH` ตัวถัดไปดึงข้อมูล
   ได้ทันที

### ข้อควรระวัง

- **ห้าม OPEN Cursor ที่เปิดอยู่แล้วซ้ำ** โดยไม่ `CLOSE` ก่อน จะได้ `SQLCODE = -502`
  ("cursor already open") ปัญหานี้พบบ่อยมากเมื่อ Cursor ถูกเปิดใน Loop ที่ไม่ได้ปิดให้ถูกต้อง
  ก่อนวนรอบใหม่
- `SQLCODE` หลัง `OPEN` มักจะเป็น `0` เสมอแม้ผลลัพธ์จะไม่มีแถวเลยก็ตาม (ต่างจาก `SELECT INTO` ที่จะ
  ได้ `100` ทันทีถ้าไม่พบข้อมูล) — Cursor จะรายงาน "ไม่มีข้อมูล" ผ่าน `FETCH` ครั้งแรกแทน ไม่ใช่ตอน
  `OPEN`

### แบบฝึกหัดที่ 583.1

**โจทย์**: หากเรียก `OPEN ORDER-CURSOR` สองครั้งติดกันโดยไม่มี `CLOSE` คั่นกลาง จะเกิดอะไรขึ้น

**เฉลย**: จะได้ `SQLCODE = -502` ("cursor already open") เพราะ DB2 ไม่อนุญาตให้ Cursor ตัวเดียวกัน
ถูกเปิดซ้ำสองครั้งพร้อมกันโดยไม่ปิดก่อน วิธีแก้คือเรียก `CLOSE ORDER-CURSOR` ก่อนจะ `OPEN` ใหม่ทุกครั้ง
หรือออกแบบตรรกะให้ `OPEN` เพียงครั้งเดียวต่อรอบการใช้งานจริง

---

## ขั้นตอนที่ 584: FETCH Cursor — วนอ่านทีละแถวด้วย PERFORM

### ไวยากรณ์ FETCH

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               FETCH ORDER-CURSOR
               INTO :HV-ORDER-ID, :HV-ORDER-DATE, :HV-TOTAL-AMOUNT
           END-EXEC.
```

### รูปแบบมาตรฐาน: FETCH ผสาน PERFORM UNTIL

การประมวลผลหลายแถวในโลกจริงเกือบทั้งหมดใช้รูปแบบนี้ — สังเกตว่าโครงสร้างเหมือนกับการอ่านไฟล์
Sequential ทุกประการ (Part 023 ขั้นตอนที่ 224) เพียงแค่เปลี่ยนจาก `READ` เป็น `FETCH` และเปลี่ยนจาก
`AT END` เป็นการตรวจ `SQLCODE`:

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG          PIC X VALUE "N".
           88  END-OF-CURSOR    VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO HV-CUST-ID.

           EXEC SQL
               OPEN ORDER-CURSOR
           END-EXEC.

           PERFORM FETCH-ORDER-PARA
               UNTIL END-OF-CURSOR.

           EXEC SQL
               CLOSE ORDER-CURSOR
           END-EXEC.
           STOP RUN.

       FETCH-ORDER-PARA.
           EXEC SQL
               FETCH ORDER-CURSOR
               INTO :HV-ORDER-ID, :HV-ORDER-DATE, :HV-TOTAL-AMOUNT
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   DISPLAY "ORDER " HV-ORDER-ID
                       " DATE " HV-ORDER-DATE
                       " AMOUNT " HV-TOTAL-AMOUNT
               WHEN 100
                   SET END-OF-CURSOR TO TRUE
               WHEN OTHER
                   DISPLAY "FETCH ERROR: " SQLCODE
                   SET END-OF-CURSOR TO TRUE
           END-EVALUATE.
```

### ทดสอบจริง: ตรรกะ PERFORM UNTIL แบบเดียวกัน (จำลองด้วยตารางในหน่วยความจำ)

เพื่อพิสูจน์ว่ารูปแบบ `PERFORM ... UNTIL END-OF-CURSOR` ที่ใช้คู่กับ `FETCH` เป็นตรรกะ COBOL มาตรฐาน
ทุกประการ (ไม่ได้พึ่งพาสิ่งพิเศษใด ๆ จาก DB2) เราจำลองด้วยตาราง `OCCURS` ในหน่วยความจำแทนการดึงจาก
DB2 จริง แล้วคอมไพล์และรันจริงด้วย GnuCOBOL:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP584-FETCH-LOOP-SIM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> This simulates the SHAPE of an OPEN/FETCH-loop/CLOSE cursor
      *> pattern using an in-memory table instead of a real DB2
      *> result set, so plain GnuCOBOL can compile and run it. The
      *> real EXEC SQL FETCH statement itself still requires a DB2
      *> precompiler and is NOT demonstrated as runnable here.
       01  WS-ORDER-TABLE.
           05  WS-ORDER-ENTRY OCCURS 4 TIMES.
               10  WS-ORDER-ID       PIC 9(6).
               10  WS-ORDER-AMOUNT   PIC 9(7)V99.
       01  WS-ROW-COUNT             PIC 9(2) VALUE 4.
       01  WS-CURSOR-POSITION       PIC 9(2) VALUE 1.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-CURSOR        VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 500001 TO WS-ORDER-ID(1).
           MOVE 1250.00 TO WS-ORDER-AMOUNT(1).
           MOVE 500002 TO WS-ORDER-ID(2).
           MOVE 320.50 TO WS-ORDER-AMOUNT(2).
           MOVE 500003 TO WS-ORDER-ID(3).
           MOVE 999.00 TO WS-ORDER-AMOUNT(3).
           MOVE 500004 TO WS-ORDER-ID(4).
           MOVE 45.25 TO WS-ORDER-AMOUNT(4).

           DISPLAY "Simulated OPEN cursor.".
           PERFORM SIMULATED-FETCH-PARA
               UNTIL END-OF-CURSOR.
           DISPLAY "Simulated CLOSE cursor.".
           STOP RUN.

       SIMULATED-FETCH-PARA.
           IF WS-CURSOR-POSITION > WS-ROW-COUNT
               SET END-OF-CURSOR TO TRUE
           ELSE
               DISPLAY "FETCHED ORDER "
                   WS-ORDER-ID(WS-CURSOR-POSITION)
                   " AMOUNT "
                   WS-ORDER-AMOUNT(WS-CURSOR-POSITION)
               ADD 1 TO WS-CURSOR-POSITION
           END-IF.
```

คอมไพล์และรัน:

```bash
cobc -x -o step584 step584.cob
./step584
```

**ผลลัพธ์จริง (ทดสอบแล้ว):**

```
Simulated OPEN cursor.
FETCHED ORDER 500001 AMOUNT 0001250.00
FETCHED ORDER 500002 AMOUNT 0000320.50
FETCHED ORDER 500003 AMOUNT 0000999.00
FETCHED ORDER 500004 AMOUNT 0000045.25
Simulated CLOSE cursor.
```

ตรรกะ `PERFORM ... UNTIL` ที่ตรวจ flag ทุกรอบก่อนประมวลผลนี้คือโครงสร้างเดียวกันเป๊ะกับที่ใช้คู่กับ
`EXEC SQL FETCH` จริง เพียงแต่ในโลกจริงเงื่อนไข `END-OF-CURSOR` จะถูกตั้งค่าจาก `SQLCODE = 100`
แทนการเทียบตำแหน่งกับ `WS-ROW-COUNT`

### ข้อควรระวัง

- **ต้องตรวจ `SQLCODE` ทุกครั้งหลัง `FETCH`** ก่อนใช้ค่าใน Host Variable เสมอ เพราะ `FETCH` ครั้ง
  สุดท้าย (ที่ได้ `SQLCODE = 100`) จะ**ไม่มีการกำหนดค่าใหม่ให้ Host Variable เลย** ถ้าโปรแกรมพยายาม
  `DISPLAY` ค่าเหล่านั้นต่อโดยไม่ตรวจสอบก่อน จะได้ข้อมูลที่ซ้ำกับรอบก่อนหน้าโดยไม่รู้ตัว
- ระวัง **Infinite Loop** หากลืมตั้ง `END-OF-CURSOR` เมื่อ `SQLCODE` เป็นค่าอื่นที่ไม่ใช่ 0 (เช่น
  ค่าติดลบที่บ่งบอก error) โปรแกรมจะวนเรียก `FETCH` ที่ error ซ้ำไปเรื่อย ๆ ไม่จบ — ตัวอย่างข้างต้น
  จึงตั้ง `SET END-OF-CURSOR TO TRUE` ไว้ทั้งใน `WHEN 100` และ `WHEN OTHER`

### แบบฝึกหัดที่ 584.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจ `SQLCODE` ใน `WHEN OTHER` ต้องตั้งค่า `END-OF-CURSOR TO TRUE`
ด้วย ทั้งที่ `WHEN OTHER` หมายถึงเกิด error ไม่ใช่ "หมดข้อมูล"

**เฉลย**: เพราะถ้าไม่ตั้ง flag ให้ loop หยุด โปรแกรมจะวนเรียก `FETCH` ที่กำลัง error อยู่ซ้ำไปเรื่อย ๆ
โดยไม่มีทางออกจาก `PERFORM UNTIL END-OF-CURSOR` ได้เลย (เพราะเงื่อนไขการจบ loop คือ flag ตัวนี้เท่านั้น)
กลายเป็น Infinite Loop ที่โปรแกรมค้างตลอดไป การตั้ง flag ให้จบ loop เมื่อเกิด error จึงจำเป็นเสมอ
แม้เหตุผลที่จบจะต่างจากกรณี "หมดข้อมูลปกติ" ก็ตาม

---

## ขั้นตอนที่ 585: CLOSE Cursor — คืนทรัพยากรให้ DB2

### ไวยากรณ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               CLOSE ORDER-CURSOR
           END-EXEC.
```

### ทำไมต้อง CLOSE เสมอ

`OPEN` Cursor ทำให้ DB2 ต้อง**จองทรัพยากรระบบ** (เช่น หน่วยความจำสำหรับเก็บสถานะการอ่าน, ล็อกบางส่วน
ของข้อมูลตามระดับ Isolation ที่ตั้งไว้ตอน `BIND`) ตราบใดที่ Cursor ยังเปิดค้างอยู่ ทรัพยากรเหล่านี้
จะไม่ถูกคืน การลืม `CLOSE` Cursor ในโปรแกรมที่ทำงานต่อเนื่องนาน ๆ (เช่นโปรแกรม CICS ที่จะเรียนใน
Part 061 ซึ่งอาจมีอายุการทำงานยาวกว่าโปรแกรม Batch มาก) อาจทำให้ทรัพยากรของระบบ DB2 หมดลงเรื่อย ๆ

### รูปแบบที่ปลอดภัย: CLOSE ในทุกเส้นทางที่โปรแกรมจบ

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       MAIN-PARA.
           EXEC SQL OPEN ORDER-CURSOR END-EXEC.
           IF SQLCODE NOT = 0
               DISPLAY "OPEN FAILED."
               GO TO END-PARA
           END-IF.

           PERFORM FETCH-ORDER-PARA UNTIL END-OF-CURSOR.

       END-PARA.
      *> Close the cursor no matter which path got here (except when OPEN
      *> failed from the start - a cursor that never opened needs no CLOSE)
           EXEC SQL CLOSE ORDER-CURSOR END-EXEC.
           STOP RUN.
```

### ข้อควรระวัง

- **ไม่ต้อง `CLOSE` Cursor ที่ `OPEN` ไม่สำเร็จ** (`SQLCODE` ของ `OPEN` ไม่เท่ากับ 0) เพราะ DB2 ยัง
  ไม่ได้จองทรัพยากรใด ๆ ให้ Cursor นั้นจริงตั้งแต่แรก การพยายาม `CLOSE` Cursor ที่ไม่เคยเปิดสำเร็จจะ
  ได้ `SQLCODE = -501` ("cursor not open")
- โปรแกรม Batch ที่จบด้วย `STOP RUN` มักจะปิด Cursor ทั้งหมดให้อัตโนมัติโดย DB2 เมื่อ Unit of Work
  จบลง แต่การพึ่งพาพฤติกรรมนี้**ไม่ใช่แนวปฏิบัติที่ดี** ควร `CLOSE` อย่างชัดเจนเสมอเพื่อความชัดเจน
  ของโค้ดและป้องกันปัญหาในกรณีที่โปรแกรมถูกเรียกซ้ำหลายครั้งในหน่วยงานเดียว (เช่นถูก `CALL` จาก
  โปรแกรมอื่นหลายรอบโดยไม่จบโปรเซส)

### แบบฝึกหัดที่ 585.1

**โจทย์**: หากโปรแกรมเรียก `CLOSE ORDER-CURSOR` ทั้งที่ `OPEN ORDER-CURSOR` ก่อนหน้าล้มเหลว
(`SQLCODE` ไม่เท่ากับ 0) จะเกิดอะไรขึ้น

**เฉลย**: จะได้ `SQLCODE = -501` ("cursor not open") เพราะ DB2 ไม่เคยเปิด Cursor นั้นสำเร็จตั้งแต่ต้น
จึงไม่มีอะไรให้ปิด วิธีที่ถูกต้องคือตรวจสอบ `SQLCODE` ของ `OPEN` ก่อนเสมอ และออกแบบ logic (เช่นด้วย
`IF`/`GO TO` ตามตัวอย่างในขั้นตอนนี้) ให้ข้ามการเรียก `CLOSE` ไปเมื่อ `OPEN` ล้มเหลว

---

## ขั้นตอนที่ 586: WHENEVER Statement — การจัดการ Error แบบ Declarative

### แนวคิดของ WHENEVER

`EXEC SQL WHENEVER ... END-EXEC` เป็นคำสั่งพิเศษที่บอก DB2 Precompiler ว่า **"ทุกครั้งที่เกิดเงื่อนไข
X หลัง EXEC SQL statement ใด ๆ ที่ตามมา ให้แทรกโค้ดกระโดดไปยัง paragraph นี้ให้อัตโนมัติ"** — ต่างจาก
วิธีที่ Part 058-059 ใช้มาตลอด (ตรวจ `SQLCODE` เองด้วยมือทุกจุด) `WHENEVER` ทำให้ Precompiler**แทรก
โค้ดตรวจสอบให้อัตโนมัติ**หลัง `EXEC SQL` ทุกตัวที่ตามมาในซอร์สไฟล์

### ไวยากรณ์และเงื่อนไขที่รองรับ

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL WHENEVER SQLERROR GO TO ERROR-PARA END-EXEC.
           EXEC SQL WHENEVER NOT FOUND GO TO END-OF-CURSOR-PARA END-EXEC.
           EXEC SQL WHENEVER SQLWARNING CONTINUE END-EXEC.
```

| เงื่อนไข | ทำงานเมื่อ |
|---|---|
| `SQLERROR` | `SQLCODE` เป็นค่าติดลบ (เกิดข้อผิดพลาด) |
| `NOT FOUND` | `SQLCODE = 100` (ไม่พบข้อมูล / Cursor หมดแถว) |
| `SQLWARNING` | `SQLCODE` เป็นค่าบวกอื่นที่ไม่ใช่ 100 (สำเร็จแต่มีคำเตือน) |

การกระทำที่ระบุได้มี 2 แบบ: `GO TO paragraph-name` (กระโดดไป paragraph ที่ระบุ) และ `CONTINUE`
(ไม่ทำอะไรเลย ปล่อยให้โปรแกรมทำงานต่อตามปกติ)

### ตัวอย่างการใช้งานเต็มรูปแบบ

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       PROCEDURE DIVISION.
       MAIN-PARA.
           EXEC SQL WHENEVER SQLERROR GO TO SQL-ERROR-PARA END-EXEC.
           EXEC SQL WHENEVER NOT FOUND GO TO NO-MORE-ORDERS-PARA
               END-EXEC.

           MOVE 10001 TO HV-CUST-ID.
           EXEC SQL OPEN ORDER-CURSOR END-EXEC.

           PERFORM UNTIL 1 = 2
               EXEC SQL
                   FETCH ORDER-CURSOR
                   INTO :HV-ORDER-ID, :HV-TOTAL-AMOUNT
               END-EXEC
      *> If SQLCODE = 100 occurs, the program will "jump" to
      *> NO-MORE-ORDERS-PARA automatically (no manual EVALUATE needed)
               DISPLAY "ORDER: " HV-ORDER-ID " " HV-TOTAL-AMOUNT
           END-PERFORM.

       NO-MORE-ORDERS-PARA.
           EXEC SQL CLOSE ORDER-CURSOR END-EXEC.
           DISPLAY "DONE PROCESSING ORDERS.".
           STOP RUN.

       SQL-ERROR-PARA.
           DISPLAY "UNEXPECTED SQL ERROR: " SQLCODE.
           STOP RUN.
```

### ข้อควรระวัง — เหตุผลที่หลายองค์กรเลือก "ไม่ใช้" WHENEVER

แม้ `WHENEVER` จะลดโค้ดซ้ำซ้อนได้มาก แต่มีข้อเสียสำคัญที่ทำให้หลายมาตรฐานการเขียนโค้ดขององค์กร
**ห้ามใช้**:

- `WHENEVER` ใช้ `GO TO` ซึ่งเป็นการควบคุมการไหลแบบไม่มีโครงสร้าง (Unstructured Control Flow)
  ที่ COBOL-85 พยายามลดการใช้ลงด้วย Scope Terminator ต่าง ๆ (ทบทวนจาก Part 001 ขั้นตอนที่ 6)
  ทำให้โค้ดอ่านยากขึ้นเมื่อโปรแกรมมีขนาดใหญ่ เพราะการกระโดดเกิดขึ้น "แบบซ่อนเร้น" หลัง EXEC SQL
  ทุกตัว ไม่ใช่จุดที่มองเห็นชัดเจนในโค้ด
- ผลของ `WHENEVER` มีผล**ตั้งแต่จุดที่ประกาศเป็นต้นไปในซอร์สไฟล์เท่านั้น** (ไม่ย้อนหลัง) และมีผลกับ
  ทุก `EXEC SQL` ที่ตามมาโดยไม่เลือกปฏิบัติ ทำให้ยากต่อการควบคุมพฤติกรรมที่ต่างกันในแต่ละจุด
- แนวทางแบบ Paragraph กลาง (Part 058 ขั้นตอนที่ 579) ให้การควบคุมที่ชัดเจนกว่าและสอดคล้องกับ
  Structured Programming มากกว่า จึงเป็นที่นิยมมากกว่าในองค์กรที่เน้น Clean Code

### แบบฝึกหัดที่ 586.1

**โจทย์**: จงอธิบายข้อดีและข้อเสียของ `WHENEVER` เทียบกับการตรวจ `SQLCODE` ด้วยตนเองทุกจุด

**เฉลย**: **ข้อดี**: ลดโค้ดซ้ำซ้อนได้มาก ไม่ต้องเขียน `EVALUATE SQLCODE` ซ้ำหลัง `EXEC SQL` ทุกจุด
ทำให้โค้ดสั้นลง **ข้อเสีย**: ใช้ `GO TO` ซึ่งขัดกับหลัก Structured Programming ทำให้ติดตามการไหลของ
โปรแกรมยากขึ้น (การกระโดดไป error handler เกิดขึ้นแบบซ่อนเร้น ไม่ปรากฏชัดในจุดที่เขียนโค้ด),
มีผลกับทุก EXEC SQL ที่ตามมาในไฟล์แบบไม่เลือกปฏิบัติ ทำให้ปรับพฤติกรรมเฉพาะจุดได้ยากกว่า จึงเหมาะกับ
โปรแกรมขนาดเล็กที่ตรรกะ error handling เหมือนกันตลอดทั้งไฟล์ ส่วนโปรแกรมขนาดใหญ่มักนิยมใช้ Paragraph
กลางแทน

---

## ขั้นตอนที่ 587: รูปแบบสมบูรณ์ — DECLARE + OPEN + PERFORM UNTIL + FETCH + CLOSE

### รวมทุกอย่างเป็นโปรแกรมเดียว

ขั้นตอนนี้จะรวมทุกส่วนจาก ขั้นตอนที่ 582–585 เป็นโปรแกรมสมบูรณ์หนึ่งโปรแกรม เพื่อให้เห็นภาพรวมทั้ง
วงจรชีวิตของ Cursor ในบริบทเดียวกัน — รูปแบบนี้คือ**แม่แบบมาตรฐาน**ที่โปรแกรมเมอร์ COBOL/DB2 ใช้ซ้ำ
แล้วซ้ำเล่าตลอดอาชีพการทำงานบน Mainframe

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP587-FULL-CURSOR-PATTERN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-CUST-ID           PIC S9(5)     COMP-3.
       01  HV-ORDER-ID          PIC S9(9)     COMP.
       01  HV-ORDER-DATE        PIC X(10).
       01  HV-TOTAL-AMOUNT      PIC S9(7)V99  COMP-3.
           EXEC SQL END DECLARE SECTION END-EXEC.

       01  WS-EOF-FLAG           PIC X VALUE "N".
           88  END-OF-CURSOR     VALUE "Y".
       01  WS-ORDER-COUNT        PIC 9(5)     VALUE 0.
       01  WS-GRAND-TOTAL        PIC S9(9)V99 VALUE 0.

           EXEC SQL
               DECLARE ORDER-CURSOR CURSOR FOR
                   SELECT ORDER_ID, ORDER_DATE, TOTAL_AMOUNT
                     FROM ORDERS
                    WHERE CUST_ID = :HV-CUST-ID
                    ORDER BY ORDER_DATE
           END-EXEC.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO HV-CUST-ID.

           EXEC SQL OPEN ORDER-CURSOR END-EXEC.
           IF SQLCODE NOT = 0
               DISPLAY "OPEN FAILED. SQLCODE=" SQLCODE
               STOP RUN
           END-IF.

           PERFORM FETCH-AND-PROCESS-PARA
               UNTIL END-OF-CURSOR.

           EXEC SQL CLOSE ORDER-CURSOR END-EXEC.

           DISPLAY "TOTAL ORDERS PROCESSED: " WS-ORDER-COUNT.
           DISPLAY "GRAND TOTAL AMOUNT     : " WS-GRAND-TOTAL.
           STOP RUN.

       FETCH-AND-PROCESS-PARA.
           EXEC SQL
               FETCH ORDER-CURSOR
               INTO :HV-ORDER-ID, :HV-ORDER-DATE, :HV-TOTAL-AMOUNT
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   ADD 1 TO WS-ORDER-COUNT
                   ADD HV-TOTAL-AMOUNT TO WS-GRAND-TOTAL
                   DISPLAY "ORDER " HV-ORDER-ID
                       " (" HV-ORDER-DATE ") "
                       HV-TOTAL-AMOUNT
               WHEN 100
                   SET END-OF-CURSOR TO TRUE
               WHEN OTHER
                   DISPLAY "FETCH ERROR: " SQLCODE
                   SET END-OF-CURSOR TO TRUE
           END-EVALUATE.
```

### อธิบายจุดสำคัญ

- สังเกตว่านอกจากดึงข้อมูลมาแสดงแล้ว โปรแกรมยัง**สะสมผลรวม** (`WS-ORDER-COUNT`, `WS-GRAND-TOTAL`)
  ด้วยตัวแปร COBOL ธรรมดา ๆ นี่คือรูปแบบที่พบบ่อยที่สุดในงานจริง: Cursor ทำหน้าที่แค่ "ป้อนข้อมูล
  ทีละแถว" ส่วนตรรกะทางธุรกิจทั้งหมด (คำนวณ, ตรวจเงื่อนไข, สะสมผลรวม) ยังคงเป็น COBOL ปกติทั้งหมด
- โครงสร้างนี้ผสานทั้ง Embedded SQL (`DECLARE`, `OPEN`, `FETCH`, `CLOSE`) เข้ากับความสามารถของ COBOL
  ที่เรียนมาตั้งแต่เฟส 1-3 ของหลักสูตรอย่างไร้รอยต่อ — นี่คือจุดแข็งของ Embedded SQL ที่ทำให้ COBOL
  ยังคงเป็นภาษาที่ใช้เขียนระบบธุรกิจบน Mainframe อย่างแพร่หลาย

### ข้อควรระวัง

- ลำดับการวางโค้ดสำคัญมาก: `DECLARE CURSOR` ต้องมาก่อน `OPEN` เสมอในซอร์สไฟล์ (แม้จะยังไม่ถูกเรียก
  ทำงานจริงจนกว่าจะถึง `OPEN` ก็ตาม) และมักนิยมวาง `DECLARE CURSOR` ไว้ท้าย `WORKING-STORAGE
  SECTION` เพื่อให้เห็นชัดเจนแยกจากการประกาศตัวแปรทั่วไป
- ฟิลด์สะสมผลรวมอย่าง `WS-GRAND-TOTAL` ต้องตั้งค่าเริ่มต้นเป็น 0 อย่างชัดเจนก่อนเริ่ม loop เสมอ
  (ในตัวอย่างใช้ `VALUE 0` ตอนประกาศ) มิฉะนั้นอาจได้ค่าเริ่มต้นที่ไม่แน่นอนถ้าโปรแกรมถูกเรียกซ้ำ
  หลายรอบในหน่วยความจำเดียวกัน

### แบบฝึกหัดที่ 587.1

**โจทย์**: จงแก้ไขโปรแกรมข้างต้นให้แสดงจำนวนคำสั่งซื้อที่มียอด `TOTAL_AMOUNT` เกิน 1,000 บาท
แยกต่างหากจากยอดรวมทั้งหมดด้วย

**เฉลย**: เพิ่มตัวแปรนับและเงื่อนไข `IF` ใน `FETCH-AND-PROCESS-PARA`:

```
01  WS-HIGH-VALUE-COUNT   PIC 9(5) VALUE 0.
...
       WHEN 0
           ADD 1 TO WS-ORDER-COUNT
           ADD HV-TOTAL-AMOUNT TO WS-GRAND-TOTAL
           IF HV-TOTAL-AMOUNT > 1000.00
               ADD 1 TO WS-HIGH-VALUE-COUNT
           END-IF
           DISPLAY "ORDER " HV-ORDER-ID " " HV-TOTAL-AMOUNT
```

แล้วเพิ่ม `DISPLAY "HIGH VALUE ORDERS: " WS-HIGH-VALUE-COUNT.` ใน `MAIN-PARA` หลัง loop จบ

---

## ขั้นตอนที่ 588: Scrollable Cursor และ Cursor สำหรับแก้ไขข้อมูล (FOR UPDATE OF)

### Cursor แบบ Read-only (ค่าเริ่มต้น)

Cursor ทุกตัวที่แสดงมาจนถึงตอนนี้เป็น **Read-only Cursor** — ใช้ได้แค่อ่านข้อมูล ไม่สามารถแก้ไขหรือ
ลบแถวที่กำลังอ่านอยู่โดยตรงผ่าน Cursor นั้นได้ หากต้องการแก้ไข/ลบข้อมูลที่ Cursor กำลังชี้อยู่ (จะสอน
ละเอียดในขั้นตอนที่ 589) ต้องประกาศ Cursor ด้วยประโยค `FOR UPDATE OF` ตั้งแต่ตอน `DECLARE`

### FOR UPDATE OF

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               DECLARE ACCOUNT-CURSOR CURSOR FOR
                   SELECT ACCT_ID, BALANCE
                     FROM ACCOUNT
                    WHERE BRANCH_CODE = :HV-BRANCH-CODE
                    FOR UPDATE OF BALANCE
           END-EXEC.
```

`FOR UPDATE OF BALANCE` บอก DB2 ว่าโปรแกรมนี้ตั้งใจจะแก้ไขคอลัมน์ `BALANCE` ของแถวที่ Cursor
กำลังชี้อยู่ในภายหลัง ทำให้ DB2 ใส่ล็อก (Lock) ที่เข้มงวดกว่า Read-only Cursor ตั้งแต่ตอน `FETCH`
เพื่อป้องกันไม่ให้โปรแกรมอื่นแก้ไขแถวเดียวกันพร้อมกันจนข้อมูลขัดแย้งกัน

### Scrollable Cursor (มาตรฐาน SQL สมัยใหม่)

DB2 เวอร์ชันใหม่ (ตั้งแต่ DB2 for z/OS V8 เป็นต้นมา ตามมาตรฐาน SQL:1999) รองรับ **Scrollable
Cursor** ที่เลื่อนไปมาได้ทั้งสองทิศทาง ต่างจาก Cursor ปกติที่เดินหน้าทางเดียวเท่านั้น
(Forward-only):

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               DECLARE SCROLL-CURSOR SCROLL CURSOR FOR
                   SELECT ACCT_ID, BALANCE
                     FROM ACCOUNT
                    ORDER BY ACCT_ID
           END-EXEC.

           EXEC SQL OPEN SCROLL-CURSOR END-EXEC.

           EXEC SQL FETCH LAST FROM SCROLL-CURSOR
               INTO :HV-ACCT-ID, :HV-BALANCE
           END-EXEC.

           EXEC SQL FETCH PRIOR FROM SCROLL-CURSOR
               INTO :HV-ACCT-ID, :HV-BALANCE
           END-EXEC.
```

| รูปแบบ FETCH | ความหมาย |
|---|---|
| `FETCH FIRST` | ไปยังแถวแรกของผลลัพธ์ |
| `FETCH LAST` | ไปยังแถวสุดท้ายของผลลัพธ์ |
| `FETCH NEXT` | ไปแถวถัดไป (พฤติกรรมเริ่มต้นเหมือน `FETCH` เฉย ๆ) |
| `FETCH PRIOR` | ย้อนกลับไปแถวก่อนหน้า |
| `FETCH ABSOLUTE n` | ไปยังแถวที่ลำดับที่ n เป๊ะ ๆ |
| `FETCH RELATIVE n` | เลื่อนจากตำแหน่งปัจจุบันไป n แถว (บวก=ไปหน้า, ลบ=ถอยหลัง) |

### ข้อควรระวัง

- Scrollable Cursor ใช้ทรัพยากรและมี overhead สูงกว่า Forward-only Cursor มาก เพราะ DB2 ต้องรักษา
  สถานะผลลัพธ์ทั้งหมดไว้ให้เลื่อนไปมาได้ ไม่ใช่แค่ "อ่านแล้วทิ้ง" แบบ Cursor ปกติ ควรใช้เฉพาะเมื่อ
  ความต้องการทางธุรกิจจำเป็นต้องเลื่อนไปมาจริง ๆ (เช่นหน้าจอแสดงผลที่ต้อง "เลื่อนกลับไปหน้าก่อน")
  ไม่ควรใช้เป็นค่าเริ่มต้นทั่วไป
- `FOR UPDATE OF` ต้องระบุคอลัมน์ที่จะแก้ไขให้ครบ หากพยายาม `UPDATE ... WHERE CURRENT OF` คอลัมน์
  ที่ไม่ได้ระบุไว้ใน `FOR UPDATE OF` จะเกิด error

### แบบฝึกหัดที่ 588.1

**โจทย์**: จงอธิบายว่าทำไม Cursor ที่จะใช้แก้ไขข้อมูลภายหลังจึงต้องประกาศ `FOR UPDATE OF` ตั้งแต่ตอน
`DECLARE CURSOR` ไม่สามารถเพิ่มทีหลังตอน `OPEN` ได้

**เฉลย**: เพราะ DB2 ต้องรู้**ตั้งแต่ตอนวางแผน Access Path** (ซึ่งเกิดตอน BIND โดยอิงจาก DECLARE
CURSOR) ว่า Cursor นี้จะถูกใช้แก้ไขข้อมูลหรือไม่ เพื่อเลือกกลยุทธ์การล็อก (Locking Strategy) ที่
เหมาะสมตั้งแต่ต้น (ล็อกที่เข้มงวดกว่าสำหรับ Cursor ที่จะแก้ไข เพื่อป้องกัน race condition) การเปลี่ยน
พฤติกรรมนี้กลางคันหลัง `OPEN` ไปแล้วจะขัดกับสถาปัตยกรรมของ DB2 ที่วางแผนการทำงานไว้ล่วงหน้าตั้งแต่
ขั้นตอน Bind

---

## ขั้นตอนที่ 589: Positioned UPDATE/DELETE — WHERE CURRENT OF

### แนวคิด

`WHERE CURRENT OF cursor-name` คือประโยคพิเศษที่ใช้แทนที่ `WHERE` แบบเงื่อนไขปกติ เพื่อบอก DB2 ว่า
**"แก้ไข/ลบเฉพาะแถวที่ Cursor นี้กำลังชี้อยู่ ณ ขณะนี้เท่านั้น"** — เป็นวิธีที่**ปลอดภัยและแม่นยำที่สุด**
ในการแก้ไขแถวที่เพิ่ง `FETCH` มา เพราะรับประกันว่าจะกระทบ**แถวเดียวที่ถูกต้องแน่นอน** ไม่มีโอกาสพลาด
ไปกระทบแถวอื่นที่ Key ซ้ำกันโดยบังเอิญ

### ตัวอย่าง Positioned UPDATE

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "001" TO HV-BRANCH-CODE.
           EXEC SQL OPEN ACCOUNT-CURSOR END-EXEC.

           PERFORM APPLY-INTEREST-PARA
               UNTIL END-OF-CURSOR.

           EXEC SQL CLOSE ACCOUNT-CURSOR END-EXEC.
           EXEC SQL COMMIT END-EXEC.
           STOP RUN.

       APPLY-INTEREST-PARA.
           EXEC SQL
               FETCH ACCOUNT-CURSOR
               INTO :HV-ACCT-ID, :HV-BALANCE
           END-EXEC.

           IF SQLCODE = 100
               SET END-OF-CURSOR TO TRUE
           ELSE IF SQLCODE NOT = 0
               DISPLAY "FETCH ERROR: " SQLCODE
               SET END-OF-CURSOR TO TRUE
           ELSE
               COMPUTE HV-BALANCE = HV-BALANCE * 1.02

      *> This UPDATE affects only the row the cursor is currently on
      *> there is no need to repeat WHERE ACCT_ID = :HV-ACCT-ID
               EXEC SQL
                   UPDATE ACCOUNT
                      SET BALANCE = :HV-BALANCE
                    WHERE CURRENT OF ACCOUNT-CURSOR
               END-EXEC

               DISPLAY "UPDATED ACCT " HV-ACCT-ID
                   " NEW BALANCE " HV-BALANCE
           END-IF.
```

### ตัวอย่าง Positioned DELETE

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC SQL
               DELETE FROM ACCOUNT
                WHERE CURRENT OF ACCOUNT-CURSOR
           END-EXEC.
```

### อธิบายจุดสำคัญ

- Cursor ที่ใช้กับ `WHERE CURRENT OF` **ต้องประกาศด้วย `FOR UPDATE OF`** ตั้งแต่ตอน `DECLARE`
  (ขั้นตอนที่ 588) มิฉะนั้นจะได้ `SQLCODE` ที่บ่งบอกว่า Cursor ไม่รองรับการแก้ไข
- หลัง `UPDATE`/`DELETE` แบบ `WHERE CURRENT OF` สำเร็จ ตำแหน่งของ Cursor **ยังคงอยู่ที่แถวเดิม**
  (ยังไม่เลื่อนไปแถวถัดไป) การ `FETCH` ครั้งถัดไปจะดึงแถวถัดจากแถวที่เพิ่งแก้ไข/ลบไปตามปกติ
- วิธีนี้มีประสิทธิภาพดีกว่าการ `UPDATE ... WHERE ACCT_ID = :HV-ACCT-ID` แยกต่างหาก เพราะ DB2 ไม่
  ต้องค้นหาแถวใหม่อีกรอบ (รู้ตำแหน่งแถวอยู่แล้วจาก Cursor) และปลอดภัยกว่าในกรณีที่ Key ไม่ unique
  จริง ๆ (ซึ่งไม่ควรเกิดขึ้นถ้าออกแบบตารางถูกต้อง แต่เป็นการป้องกันสองชั้น)

### ข้อควรระวัง

- **ห้ามใช้ `WHERE CURRENT OF` กับ Cursor ที่ยังไม่ได้ `FETCH` สำเร็จเลยสักครั้ง** (เช่น เพิ่ง `OPEN`
  แต่ยังไม่ `FETCH`) เพราะ Cursor ยังไม่ได้ "ชี้" ไปที่แถวใดเลย จะได้ error ทันที
- หลัง `DELETE ... WHERE CURRENT OF` การเรียก `FETCH` ครั้งถัดไปยังคงทำงานได้ปกติ (DB2 จัดการเลื่อน
  ตำแหน่งให้เอง) แต่**ห้ามพยายาม `UPDATE`/`DELETE` ซ้ำที่ตำแหน่งเดิมอีกครั้ง**ก่อน `FETCH` ใหม่
  เพราะแถวนั้นถูกลบไปแล้ว

### แบบฝึกหัดที่ 589.1

**โจทย์**: จงอธิบายข้อดีของ `UPDATE ... WHERE CURRENT OF cursor-name` เทียบกับการเขียน
`UPDATE ... WHERE ACCT_ID = :HV-ACCT-ID` แยกต่างหากหลัง `FETCH`

**เฉลย**: ข้อดีหลัก 2 ประการ: (1) **ประสิทธิภาพดีกว่า** เพราะ DB2 รู้ตำแหน่งแถวจาก Cursor อยู่แล้ว
ไม่ต้องค้นหา (search) แถวใหม่อีกรอบผ่าน `WHERE ACCT_ID = ...` (2) **ปลอดภัยกว่า** เพราะรับประกันว่า
จะกระทบแถวที่ Cursor กำลังชี้อยู่จริง ๆ เท่านั้น แม้ในทางทฤษฎีจะมีแถวอื่นที่ค่า Key เดียวกันปรากฏขึ้น
โดยไม่คาดคิด (เช่น จาก bug อื่นของระบบ) ก็จะไม่กระทบแถวผิดโดยไม่ได้ตั้งใจ

---

## ขั้นตอนที่ 590: Cursor พร้อมพารามิเตอร์ — โปรแกรมประมวลผลบัญชีตามเกณฑ์แบบสมบูรณ์

### โจทย์ตัวอย่าง

เขียนโปรแกรมที่รับค่า Threshold จากผู้ใช้ แล้วค้นหาบัญชีทั้งหมดที่ยอดคงเหลือต่ำกว่า Threshold นั้น
พร้อมสรุปจำนวนบัญชีและยอดรวม — นี่คือรูปแบบที่ใช้ Cursor ร่วมกับ Host Variable ที่ "เปลี่ยนค่าได้ทุก
ครั้งที่ OPEN" อย่างเต็มรูปแบบ

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP590-PARAMETERIZED-CURSOR.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.

           EXEC SQL BEGIN DECLARE SECTION END-EXEC.
       01  HV-THRESHOLD          PIC S9(9)V99  COMP-3.
       01  HV-ACCT-ID            PIC S9(9)     COMP.
       01  HV-BALANCE            PIC S9(9)V99  COMP-3.
           EXEC SQL END DECLARE SECTION END-EXEC.

       01  WS-EOF-FLAG            PIC X VALUE "N".
           88  END-OF-CURSOR      VALUE "Y".
       01  WS-ACCOUNT-COUNT       PIC 9(5)      VALUE 0.
       01  WS-TOTAL-BALANCE       PIC S9(9)V99  VALUE 0.
       01  WS-USER-INPUT          PIC 9(9)V99.

           EXEC SQL
               DECLARE LOW-BALANCE-CURSOR CURSOR FOR
                   SELECT ACCT_ID, BALANCE
                     FROM ACCOUNT
                    WHERE BALANCE < :HV-THRESHOLD
                    ORDER BY BALANCE
           END-EXEC.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "ENTER BALANCE THRESHOLD: " WITH NO ADVANCING.
           ACCEPT WS-USER-INPUT.
           MOVE WS-USER-INPUT TO HV-THRESHOLD.

           EXEC SQL OPEN LOW-BALANCE-CURSOR END-EXEC.
           IF SQLCODE NOT = 0
               DISPLAY "OPEN FAILED. SQLCODE=" SQLCODE
               STOP RUN
           END-IF.

           PERFORM FETCH-ACCOUNT-PARA
               UNTIL END-OF-CURSOR.

           EXEC SQL CLOSE LOW-BALANCE-CURSOR END-EXEC.

           DISPLAY "=================================".
           DISPLAY "ACCOUNTS BELOW THRESHOLD: "
               WS-ACCOUNT-COUNT.
           DISPLAY "TOTAL BALANCE           : "
               WS-TOTAL-BALANCE.
           STOP RUN.

       FETCH-ACCOUNT-PARA.
           EXEC SQL
               FETCH LOW-BALANCE-CURSOR
               INTO :HV-ACCT-ID, :HV-BALANCE
           END-EXEC.

           EVALUATE SQLCODE
               WHEN 0
                   ADD 1 TO WS-ACCOUNT-COUNT
                   ADD HV-BALANCE TO WS-TOTAL-BALANCE
                   DISPLAY "ACCT " HV-ACCT-ID
                       " BALANCE " HV-BALANCE
               WHEN 100
                   SET END-OF-CURSOR TO TRUE
               WHEN OTHER
                   DISPLAY "FETCH ERROR: " SQLCODE
                   SET END-OF-CURSOR TO TRUE
           END-EVALUATE.
```

### อธิบายจุดสำคัญ

- `HV-THRESHOLD` ถูกกำหนดค่าจากผู้ใช้ **ก่อน** `OPEN LOW-BALANCE-CURSOR` — นี่คือจุดที่ยืนยันสิ่งที่
  กล่าวไว้ในขั้นตอนที่ 582 ว่า Host Variable ใน `WHERE` ของ `DECLARE CURSOR` จะถูก "จับค่า" ตอน
  `OPEN` เท่านั้น ทำให้ Cursor ตัวเดียวกันสามารถนำมาใช้ค้นหาด้วยเกณฑ์ต่างกันได้ในแต่ละรอบการทำงาน
  (เพียงแต่ต้อง `CLOSE` ก่อนจะ `OPEN` ใหม่ด้วยค่าอื่น)
- โปรแกรมนี้สาธิตการผสาน `ACCEPT` (Part 007), ตัวแปรสะสมผลรวม (Part 009), `EVALUATE` (Part 011),
  และ Embedded SQL Cursor (Part นี้) เข้าด้วยกันอย่างเป็นธรรมชาติ — สะท้อนให้เห็นว่า Embedded SQL
  ไม่ได้แทนที่ความรู้ COBOL พื้นฐานที่เรียนมาตลอดหลักสูตร แต่เป็น "ส่วนเสริม" ที่ทำงานร่วมกันได้อย่าง
  ลงตัว

### ข้อควรระวัง

- ถ้าต้องการรันโปรแกรมนี้ซ้ำหลายรอบด้วย Threshold ต่างกันในโปรแกรมเดียว (loop ทั้งหมด) ต้องเรียก
  `CLOSE LOW-BALANCE-CURSOR` ก่อนเปลี่ยนค่า `HV-THRESHOLD` แล้ว `OPEN` ใหม่ทุกครั้ง — ลืมขั้นตอนนี้
  จะทำให้ได้ `SQLCODE = -502` ตามที่เตือนไว้ในขั้นตอนที่ 583
- ต้องรีเซ็ต `WS-EOF-FLAG` กลับเป็น `"N"` ก่อนเริ่ม loop รอบใหม่ทุกครั้งเช่นเดียวกับที่เตือนไว้ใน
  Part 028 ขั้นตอนที่ 274 มิฉะนั้น loop รอบสองจะไม่ทำงานเลยเพราะ flag ยังค้างเป็น `"Y"` จากรอบก่อน

### แบบฝึกหัดที่ 590.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้จึงต้องกำหนดค่า `HV-THRESHOLD` **ก่อน** `OPEN` และจะเกิดอะไรขึ้น
ถ้ากำหนดค่า **หลัง** `OPEN` แทน

**เฉลย**: เพราะ DB2 จะ "อ่านค่า" Host Variable ทุกตัวที่ปรากฏใน `WHERE` ของ `DECLARE CURSOR` ทันที
ที่คำสั่ง `OPEN` ถูกเรียก แล้วใช้ค่านั้นตลอดวงจรชีวิตของ Cursor นั้น (จนกว่าจะ `CLOSE`) หากกำหนดค่า
`HV-THRESHOLD` **หลัง** `OPEN` การเปลี่ยนแปลงนั้นจะไม่มีผลใด ๆ ต่อผลลัพธ์ที่ Cursor กำลังส่งกลับมา
เลย เพราะ DB2 ได้ "ล็อก" ค่าเงื่อนไข ณ ขณะ `OPEN` ไปแล้ว ผลลัพธ์ที่ `FETCH` ได้จะยังคงอิงจากค่าเก่า
ที่ `HV-THRESHOLD` เคยมีตอน `OPEN` เท่านั้น

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้:

- เหตุผลที่ต้องมี Cursor — ข้อจำกัดของ `SELECT INTO` ที่รองรับได้แค่แถวเดียว
- `DECLARE CURSOR` สำหรับนิยาม Cursor และ SELECT statement ที่ผูกกับมัน
- `OPEN` สำหรับเริ่มประมวลผลจริง และสิ่งที่เกิดขึ้นเบื้องหลัง
- `FETCH` ผสาน `PERFORM UNTIL` — รูปแบบมาตรฐานที่ใช้ตลอดอาชีพ COBOL/DB2 (พร้อมทดสอบจริงว่าตรรกะ
  `PERFORM UNTIL` ที่ใช้คู่กันเป็น COBOL มาตรฐาน)
- `CLOSE` และเหตุผลที่ต้องปิด Cursor เสมอเพื่อคืนทรัพยากรให้ DB2
- `WHENEVER` — ทางเลือกแบบ Declarative สำหรับจัดการ error ที่มีทั้งข้อดีและข้อเสีย
- รูปแบบสมบูรณ์ DECLARE-OPEN-PERFORM-FETCH-CLOSE ในโปรแกรมเดียว
- Scrollable Cursor และ `FOR UPDATE OF` สำหรับ Cursor ที่ต้องแก้ไขข้อมูลได้
- Positioned UPDATE/DELETE ด้วย `WHERE CURRENT OF` — วิธีแก้ไข/ลบข้อมูลที่ปลอดภัยและมีประสิทธิภาพ
  ที่สุดสำหรับแถวที่ Cursor กำลังชี้อยู่
- Cursor พร้อมพารามิเตอร์ที่เปลี่ยนค่าได้ทุกครั้งที่ `OPEN`

Cursor คือกลไกที่ทำให้ COBOL สามารถประมวลผลข้อมูลจำนวนมากจาก DB2 ได้อย่างมีประสิทธิภาพและปลอดภัย
เทียบเท่ากับการอ่านไฟล์ Sequential ที่เรียนมาก่อนหน้านี้ทุกประการ

Part 060 จะต่อยอดความรู้นี้ไปสู่การเขียน **DB2 Stored Procedure ด้วย COBOL** — การย้ายตรรกะทางธุรกิจ
บางส่วนไปทำงาน "ใกล้ข้อมูล" มากขึ้นโดยตรงบน DB2 Engine

**[← กลับไป Part 058: Embedded SQL: EXEC SQL ใน COBOL](part-058-embedded-sql.md)**
**[ไปยัง Part 060: DB2 Stored Procedures ด้วย COBOL →](part-060-db2-stored-procedures.md)**
