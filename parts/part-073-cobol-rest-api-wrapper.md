# Part 073: COBOL และ REST API: การสร้าง Wrapper Service (ขั้นตอนที่ 721–730)

## คำนำของ Part นี้

Part 072 พาเราเรียนรู้การเชื่อม COBOL เข้ากับ C และ Java ผ่านคำสั่ง `CALL` โดยตรงในระดับโปรแกรม
(in-process integration) แต่โลกของแอปพลิเคชันสมัยใหม่ส่วนใหญ่สื่อสารกันผ่าน **HTTP และ REST API**
ที่รับส่งข้อมูลเป็น JSON ปัญหาคือ **COBOL ไม่มีความสามารถพูดภาษา HTTP ได้โดยตรง** — ไม่มี library
มาตรฐานสำหรับเปิด socket รับ HTTP request, แปลง JSON, หรือจัดการ routing แบบที่ภาษาเช่น Python
(Flask/FastAPI) หรือ Node.js (Express) ทำได้ในไม่กี่บรรทัด

Part นี้จะสอนแนวทางที่องค์กรจริงใช้แก้ปัญหานี้มานานหลายสิบปี นั่นคือรูปแบบที่เรียกว่า
**Wrapper Service** (หรือบางที่เรียก **Integration Adapter** / **API Gateway Pattern**):
เขียนโปรแกรมเล็ก ๆ ด้วยภาษาที่ถนัดเรื่องเว็บ (ในที่นี้คือ Python) ทำหน้าที่เป็น "คนกลาง" — รับ
HTTP request จากภายนอก แปลงข้อมูลเป็นรูปแบบที่โปรแกรม COBOL เข้าใจ (ผ่าน stdin หรือไฟล์)
เรียกโปรแกรม COBOL ให้ทำงาน แล้วแปลงผลลัพธ์กลับเป็น JSON ส่งคืนไป โดยที่**ตัวโปรแกรม COBOL เองไม่
ต้องรู้จัก HTTP เลยแม้แต่น้อย** — นี่คือรูปแบบเดียวกับที่ธนาคารและองค์กรขนาดใหญ่จำนวนมากใช้จริงในการ
เปิดให้ระบบ Core Banking แบบ COBOL/Mainframe เดิม เชื่อมต่อกับแอปมือถือหรือเว็บสมัยใหม่ได้โดยไม่ต้อง
เขียน Core Logic ใหม่ทั้งหมด (ตามที่กล่าวถึงในภาพรวมของ Part 001 ขั้นตอนที่ 3 เรื่อง Modernization)

ทุกตัวอย่างใน Part นี้ถูกทดสอบจริงแบบ end-to-end ในสภาพแวดล้อมของหลักสูตร: คอมไพล์โปรแกรม COBOL
ด้วย GnuCOBOL จริง, รัน HTTP server ด้วย Python จริง, และยิง request ด้วย `curl` จริง แล้วคัดลอกผลลัพธ์
ที่ได้จริงมาแสดงไว้ทุกจุด (ไม่มีการ "จินตนาการ" ผลลัพธ์)

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่าง (คอมเมนต์, ชื่อตัวแปร, ข้อความใน `DISPLAY`)
> เป็นภาษาอังกฤษล้วน ส่วนโค้ด Python ที่ใช้เป็น Wrapper ก็เป็นภาษาอังกฤษล้วนเช่นกันตามธรรมชาติของ
> โค้ดจริง คำอธิบายทั้งหมดนอกโค้ดบล็อกเป็นภาษาไทยตามปกติของหลักสูตร

### เครื่องมือที่ใช้จริงใน Part นี้ (ตรวจสอบแล้ว)

ก่อนเริ่ม เราตรวจสอบสภาพแวดล้อมจริงที่จะใช้ทดสอบทุกตัวอย่างใน Part นี้ก่อนเสมอ (ตามวินัยที่ปลูกฝัง
มาตั้งแต่ Part 045):

```bash
cobc --version
python3 --version
curl --version
```

ผลลัพธ์จริงจากสภาพแวดล้อมที่ใช้พัฒนา Part นี้:

```
cobc (GnuCOBOL) 4.0-early-dev.0
Python 3.11.15
curl 8.5.0 (x86_64-pc-linux-gnu) libcurl/8.5.0 ...
```

ทั้งสามเครื่องมือพร้อมใช้งานเต็มรูปแบบ — GnuCOBOL สำหรับ business logic, Python 3 (มาพร้อม module
มาตรฐาน `http.server` โดยไม่ต้องติดตั้งอะไรเพิ่มเติมเลย ไม่ต้อง `pip install` แม้แต่ package เดียว)
สำหรับ Wrapper Service, และ `curl` สำหรับทดสอบยิง request

---

## ขั้นตอนที่ 721: ทำไม COBOL คุยกับ HTTP ตรงๆ ไม่ได้ — ภาพรวมของ Wrapper Service Pattern

### ปัญหาพื้นฐาน

HTTP คือโพรโทคอลที่ทำงานบน TCP socket โดยมีรูปแบบข้อความเฉพาะ (HTTP request/response,
headers, JSON body ฯลฯ) การจะ "พูด HTTP" ได้ โปรแกรมต้องมีความสามารถอย่างน้อย 3 อย่าง:

1. เปิด TCP socket รอรับการเชื่อมต่อ (listen)
2. แยกวิเคราะห์ (parse) ข้อความ HTTP request ตามมาตรฐาน RFC
3. แปลง Body ที่มักเป็น JSON ให้เป็นโครงสร้างข้อมูลที่ใช้งานได้ และแปลงกลับตอนตอบ

มาตรฐาน COBOL (แม้แต่ COBOL-2014) **ไม่มี** library สำหรับข้อ 1–3 นี้โดยตรง ต่างจาก Python ที่มี
`http.server` เป็นส่วนหนึ่งของ Standard Library ติดตั้งมาพร้อมตัวภาษาอยู่แล้ว (ทดสอบแล้วในเครื่องพัฒนา
Part นี้ว่าใช้งานได้ทันทีโดยไม่ต้อง `pip install` ใด ๆ)

### ทางออก: Wrapper Service Pattern

แนวคิดคือแบ่งงานออกเป็น 2 ส่วนที่ชัดเจน:

```
[ HTTP Client ]  --JSON-->  [ Python Wrapper Service ]  --stdin-->  [ COBOL Program ]
                                     (HTTP server,                  (business logic,
                                      JSON <-> text                  no HTTP knowledge
                                      translation)                   needed at all)
                 <--JSON--                                <--stdout--
```

- **COBOL Program**: ทำหน้าที่เดียวคือ business logic ล้วน ๆ (เช่น คำนวณราคา, ตรวจสอบยอดเงิน) รับ
  input ผ่านช่องทางง่าย ๆ ที่สุดที่ COBOL ถนัด นั่นคือ `ACCEPT ... FROM CONSOLE` (อ่านจาก stdin)
  และส่ง output ผ่าน `DISPLAY` (เขียนไปที่ stdout) — เทคนิคพื้นฐานที่เรียนไปแล้วตั้งแต่ Part 007
- **Wrapper Service**: เขียนด้วยภาษาที่ถนัดเรื่องเว็บ (Python ในที่นี้) ทำหน้าที่เป็นคนกลาง 4 ขั้นตอน:
  1. รับ HTTP request (เช่น `POST /api/order` พร้อม JSON body)
  2. แปลง JSON เป็นข้อความที่ COBOL program คาดหวังทาง stdin
  3. เรียกโปรแกรม COBOL เป็น subprocess (เหมือนรันคำสั่งใน terminal) แล้วรอผลลัพธ์
  4. แปลงผลลัพธ์ที่ COBOL แสดงออกทาง stdout กลับเป็น JSON แล้วตอบกลับ HTTP client

จุดที่สำคัญที่สุดของแนวคิดนี้คือ **ความรับผิดชอบถูกแบ่งแยกชัดเจน (Separation of Concerns)**:
COBOL program ไม่จำเป็นต้องรู้จัก HTTP, JSON, หรือ network เลยแม้แต่น้อย ทำให้โค้ด COBOL ที่มีอยู่
เดิมในระบบ Legacy (ซึ่งอาจเขียนมาหลายสิบปีก่อนที่ REST API จะถือกำเนิดขึ้นมาด้วยซ้ำ) สามารถถูกนำมา
เปิดเป็น API ได้โดย**แทบไม่ต้องแก้โค้ด COBOL เดิมเลย** เพียงแค่เขียน Wrapper Service ห่อหุ้มไว้ด้านนอก

### ตัวอย่างสถานการณ์จริงที่ใช้ Pattern นี้

- ธนาคารเปิด Mobile Banking App ใหม่ แต่ระบบคำนวณดอกเบี้ยและตรวจสอบยอดเงินยังคงเป็นโปรแกรม COBOL
  อายุ 20 ปีที่ผ่านการทดสอบมาอย่างละเอียดแล้ว การเขียน Wrapper Service ห่อหุ้มไว้ปลอดภัยกว่าการ
  Rewrite Logic ใหม่ทั้งหมด (ตามเหตุผลที่อธิบายไว้ใน Part 001 ขั้นตอนที่ 3)
- หน่วยงานรัฐเปิด Public API ให้ประชาชนตรวจสอบสถานะโดยไม่ต้องเปิดเผยว่าเบื้องหลังยังใช้ระบบ
  Mainframe/COBOL อยู่

### ข้อควรระวัง

- Wrapper Service Pattern เพิ่มความหน่วง (latency) เสมอ เพราะมีขั้นตอนแปลงข้อมูลและเรียก
  subprocess เพิ่มเข้ามา สำหรับระบบที่ต้องการความเร็วสูงมาก (high-frequency trading เป็นต้น) อาจต้อง
  พิจารณาวิธีอื่นที่ซับซ้อนกว่านี้ (จะกล่าวถึงในขั้นตอนที่ 729)
- Pattern นี้เหมาะกับการเรียกโปรแกรม COBOL แบบ "รับ input ครั้งเดียว คำนวณ ส่ง output ครั้งเดียว
  แล้วจบ" (stateless, batch-like) ไม่เหมาะกับโปรแกรมที่ต้องโต้ตอบแบบ interactive หลายรอบในคำขอเดียว

### แบบฝึกหัดที่ 721.1

**โจทย์**: จงอธิบายด้วยคำพูดของคุณเองว่าทำไม "การไม่ต้องแก้โค้ด COBOL เดิม" จึงเป็นข้อดีสำคัญของ
Wrapper Service Pattern ในบริบทขององค์กรขนาดใหญ่

**เฉลยแนวทาง**: โค้ด COBOL ในระบบองค์กรขนาดใหญ่ (โดยเฉพาะระบบการเงิน) มักผ่านการทดสอบและใช้งาน
จริงมานานหลายปีหรือหลายสิบปี การแก้ไขโค้ดเดิมมีความเสี่ยงสูงที่จะทำให้เกิดบั๊กใหม่ในส่วนที่เคยทำงาน
ถูกต้องมาตลอด การเพิ่ม Wrapper Service เป็นชั้นแยกต่างหากทำให้ทีมพัฒนาเปิด API ใหม่ได้โดยไม่ต้อง
แตะโค้ด COBOL เดิมเลย ลดความเสี่ยงและความจำเป็นในการทดสอบซ้ำระบบเดิมทั้งหมด

---

## ขั้นตอนที่ 722: ออกแบบและเขียนโปรแกรม COBOL Business Logic (ORDERCALC)

### โจทย์ทางธุรกิจ

เราจะสร้างระบบคำนวณยอดคำสั่งซื้อสินค้าอย่างง่าย: รับราคาต่อหน่วย (price) และจำนวน (quantity)
แล้วคำนวณ ยอดก่อนภาษี (subtotal), ภาษี 7% (tax), และยอดรวมสุทธิ (total) — logic แบบนี้คือตัวแทน
ของ business logic จริงที่มักพบในระบบ ERP/ขายสินค้าที่เขียนด้วย COBOL

### ออกแบบ Interface ของโปรแกรม COBOL

เราเลือกให้โปรแกรม COBOL รับข้อมูล 2 บรรทัดทาง stdin (ราคา แล้วตามด้วยจำนวน) และส่งผลลัพธ์
1 บรรทัดทาง stdout โดยคั่นแต่ละค่าด้วยเครื่องหมาย pipe (`|`) — รูปแบบเรียบง่ายที่สุดเท่าที่จะเป็นไปได้
เพื่อให้ทั้งฝั่ง COBOL (ด้วย `STRING`/`UNSTRING` ที่เรียนมาตั้งแต่ Part 019-020) และฝั่ง Python
(ด้วย `str.split("|")`) แปลงข้อมูลกันได้ง่ายที่สุด

### โค้ดโปรแกรม ORDERCALC

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDERCALC.
       AUTHOR. COBOL-COURSE.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-IN-PRICE         PIC X(10).
       01  WS-IN-QTY           PIC X(10).
       01  WS-PRICE            PIC 9(7)V99.
       01  WS-QTY              PIC 9(5).
       01  WS-SUBTOTAL         PIC 9(9)V99.
       01  WS-TAX-RATE         PIC V99 VALUE .07.
       01  WS-TAX              PIC 9(9)V99.
       01  WS-TOTAL            PIC 9(9)V99.
       01  WS-SUBTOTAL-EDIT    PIC Z(7)9.99.
       01  WS-TAX-EDIT         PIC Z(7)9.99.
       01  WS-TOTAL-EDIT       PIC Z(7)9.99.
       01  WS-OUT-LINE         PIC X(60).
       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-IN-PRICE FROM CONSOLE
           ACCEPT WS-IN-QTY FROM CONSOLE
           MOVE FUNCTION NUMVAL(WS-IN-PRICE) TO WS-PRICE
           MOVE FUNCTION NUMVAL(WS-IN-QTY) TO WS-QTY
           COMPUTE WS-SUBTOTAL = WS-PRICE * WS-QTY
           COMPUTE WS-TAX = WS-SUBTOTAL * WS-TAX-RATE
           COMPUTE WS-TOTAL = WS-SUBTOTAL + WS-TAX
           MOVE WS-SUBTOTAL TO WS-SUBTOTAL-EDIT
           MOVE WS-TAX TO WS-TAX-EDIT
           MOVE WS-TOTAL TO WS-TOTAL-EDIT
           STRING FUNCTION TRIM(WS-SUBTOTAL-EDIT) DELIMITED BY SIZE
                  "|" DELIMITED BY SIZE
                  FUNCTION TRIM(WS-TAX-EDIT) DELIMITED BY SIZE
                  "|" DELIMITED BY SIZE
                  FUNCTION TRIM(WS-TOTAL-EDIT) DELIMITED BY SIZE
               INTO WS-OUT-LINE
           DISPLAY FUNCTION TRIM(WS-OUT-LINE)
           STOP RUN.
```

### อธิบายทีละส่วน

- `ACCEPT WS-IN-PRICE FROM CONSOLE`: อ่านข้อความ 1 บรรทัดจาก standard input (stdin) เก็บไว้ใน
  field ข้อความธรรมดา (`PIC X(10)`) ก่อน — ยังไม่แปลงเป็นตัวเลขทันที เพราะ input จากภายนอกอาจมี
  รูปแบบไม่แน่นอน (เช่น มีช่องว่างนำหน้า) การอ่านเป็นข้อความก่อนแล้วค่อยแปลงปลอดภัยกว่า
- `FUNCTION NUMVAL(...)`: แปลงข้อความตัวเลข (เช่น `"100.00"`) ให้เป็นค่าตัวเลขจริงที่ใช้คำนวณได้
  (ทบทวนจาก Part 036) รองรับทั้งจำนวนเต็มและทศนิยม
- `WS-TAX-RATE PIC V99 VALUE .07`: `V` คือจุดทศนิยมสมมติ (implied decimal, ทบทวนจาก Part 006)
  ค่า `.07` แทนอัตราภาษี 7%
- `PIC Z(7)9.99`: Numeric-edited picture ที่ทำให้ผลลัพธ์มีจุดทศนิยมจริงปรากฏในข้อความ (จำเป็นสำหรับ
  ส่งออกไปให้ Python/JSON ใช้งานต่อ) และ `Z` ทำให้เลขศูนย์นำหน้ากลายเป็นช่องว่างแทน
- `STRING ... DELIMITED BY SIZE ... INTO WS-OUT-LINE`: ต่อทั้ง 3 ค่าเข้าด้วยกันคั่นด้วย `|`
  (ทบทวนเทคนิคนี้เต็มรูปแบบจาก Part 019)

### ข้อควรระวัง

- ต้อง `FUNCTION TRIM` ทุกค่าก่อนนำไปต่อด้วย `STRING` เสมอ เพราะ `PIC Z(7)9.99` จะมีช่องว่างนำหน้า
  เมื่อค่าน้อยกว่าค่าสูงสุดที่ field รองรับได้ — ถ้าลืม trim ผลลัพธ์จะมีช่องว่างปนอยู่กลางข้อความ
  ทำให้ Python `split("|")` ยังทำงานได้ แต่ค่าที่ได้จะมีช่องว่างเกินมา
- `COMPUTE` ใน GnuCOBOL **ตัดทศนิยมทิ้ง (truncate) โดยไม่ปัดเศษ** เป็นค่าเริ่มต้น เว้นแต่จะใส่คำว่า
  `ROUNDED` ต่อท้ายชื่อตัวแปรปลายทาง ตัวอย่างนี้จงใจไม่ใส่ `ROUNDED` เพื่อความเรียบง่าย แต่ระบบจริง
  ที่เกี่ยวกับการเงินควรพิจารณาใส่ `ROUNDED` เสมอ (ทบทวนจาก Part 009)

### แบบฝึกหัดที่ 722.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม `ORDERCALC` จึงไม่ต้อง `SELECT`/`FD`/`OPEN`/`CLOSE` ใด ๆ เลย
ทั้งที่ Part 023-030 เน้นย้ำเรื่องการจัดการไฟล์มาโดยตลอด

**เฉลย**: เพราะโปรแกรมนี้ไม่ได้อ่าน-เขียนไฟล์บนดิสก์เลย แต่ใช้ `ACCEPT FROM CONSOLE` และ `DISPLAY`
ซึ่งทำงานผ่าน standard input/output (stdin/stdout) ของ process โดยตรง — เป็นกลไกคนละแบบกับ
File Processing ที่เรียนใน Part 023-030 การออกแบบให้ใช้ stdin/stdout แทนไฟล์จริงบนดิสก์นี้เองที่ทำให้
โปรแกรมนี้เรียกจาก subprocess ภายนอก (เช่นจาก Python) ได้สะดวกมาก เพราะทุกภาษาโปรแกรมมีความสามารถ
ส่งข้อมูลเข้า stdin และอ่านค่าจาก stdout ของ subprocess ได้เป็นมาตรฐานอยู่แล้ว

---

## ขั้นตอนที่ 723: คอมไพล์และทดสอบ ORDERCALC แบบ Standalone

### คอมไพล์โปรแกรม

```bash
cobc -x -o ordercalc ordercalc.cob
```

คำสั่งนี้คอมไพล์ `ordercalc.cob` เป็นไฟล์ execute ได้ชื่อ `ordercalc` (ทบทวนจาก Part 002)

### ทดสอบด้วยการป้อนข้อมูลผ่าน pipe

เราจำลองการส่งข้อมูล 2 บรรทัดเข้า stdin ด้วยคำสั่ง `printf` ร่วมกับ pipe (`|`) ของ shell:

```bash
printf "100.00\n3\n" | ./ordercalc
```

**ผลลัพธ์จริงที่ได้:**

```
300.00|21.00|321.00
```

ตรวจทานผลลัพธ์: ราคาต่อหน่วย 100.00 บาท x จำนวน 3 = subtotal 300.00, ภาษี 7% ของ 300.00 = 21.00,
ยอดรวม = 300.00 + 21.00 = 321.00 ตรงกันทุกประการ

ลองอีกชุดข้อมูลที่มีทศนิยมซับซ้อนขึ้น:

```bash
printf "19.99\n5\n" | ./ordercalc
```

**ผลลัพธ์จริงที่ได้:**

```
99.95|6.99|106.94
```

ตรวจทาน: 19.99 x 5 = 99.95, ภาษี 7% ของ 99.95 = 6.9965 แต่เนื่องจาก `COMPUTE` ไม่ปัดเศษ (ตามที่
เตือนไว้ในขั้นตอนก่อนหน้า) ค่าจึงถูกตัดเหลือ 6.99 แทนที่จะปัดเป็น 7.00 และยอดรวม = 99.95 + 6.99 =
106.94 — นี่คือพฤติกรรมจริงที่ตรวจสอบแล้ว ไม่ใช่การปัดเศษแบบที่หลายคนคาดหวัง

### พิสูจน์กรณี Input ผิดพลาด (สำคัญมากสำหรับขั้นตอนถัดไป)

ลองป้อนข้อความที่ไม่ใช่ตัวเลขเข้าไปดู:

```bash
printf "abc\n3\n" | ./ordercalc
```

**ผลลัพธ์จริงที่ได้:**

```
0.00|0.00|0.00
```

### อธิบายจุดสำคัญ

- `FUNCTION NUMVAL` เมื่อได้รับข้อความที่ไม่ใช่ตัวเลขเลย (เช่น `"abc"`) ในบิลด์ GnuCOBOL นี้จะคืนค่า
  **0 อย่างเงียบ ๆ โดยไม่มี error หรือ exception ใด ๆ** และโปรแกรมยังคงจบด้วย exit code 0 ปกติ —
  นี่คือ "Silent Failure" ที่คล้ายกับปัญหา `STRING` overflow ที่เจอใน Part 045 ขั้นตอนที่ 450
- ข้อเท็จจริงนี้สำคัญมากสำหรับขั้นตอนที่ 728 ที่เราจะออกแบบการตรวจสอบ (validation) ที่ฝั่ง Wrapper
  Service ให้ดักจับ input ที่ไม่ถูกต้อง**ก่อน**ที่จะส่งเข้า COBOL program เพราะเราพิสูจน์แล้วว่า
  ตัวโปรแกรม COBOL เองจะไม่แจ้งเตือนเมื่อได้รับ input ที่ผิดรูปแบบ

### ข้อควรระวัง

- **ห้ามพึ่งพาให้โปรแกรม COBOL ตรวจสอบความถูกต้องของ input เองทั้งหมด** โดยเฉพาะเมื่อโปรแกรม
  COBOL นั้นถูกออกแบบมาให้รับ input ที่ "สะอาด" อยู่แล้วจากระบบภายในองค์กร (ซึ่งมักเป็นกรณีของ
  โปรแกรม Legacy จำนวนมาก) การเปิดโปรแกรมนี้ให้รับ input จากอินเทอร์เน็ตภายนอกโดยตรงโดยไม่ผ่านการ
  ตรวจสอบก่อนเป็นความเสี่ยงด้านความปลอดภัยและความถูกต้องของข้อมูลอย่างมาก
- ทดสอบโปรแกรม COBOL แบบ standalone ด้วย `printf | ./program` ก่อนเสมอ ก่อนที่จะเชื่อมต่อกับ
  Wrapper Service เพราะจะแยกปัญหาได้ชัดเจนว่าบั๊กอยู่ที่ฝั่ง COBOL หรือฝั่ง Wrapper

### แบบฝึกหัดที่ 723.1

**โจทย์**: จงทดสอบโปรแกรม `ORDERCALC` ด้วยราคา `50.00` และจำนวน `0` แล้วอธิบายว่าทำไมผลลัพธ์จึง
เป็นเช่นนั้น

**เฉลย**: รันคำสั่ง `printf "50.00\n0\n" | ./ordercalc` จะได้ผลลัพธ์ `0.00|0.00|0.00` เพราะ
`WS-SUBTOTAL = WS-PRICE * WS-QTY = 50.00 * 0 = 0` และเมื่อ subtotal เป็น 0 ภาษีและยอดรวมก็เป็น 0
ตามไปด้วยโดยธรรมชาติของการคำนวณ ไม่ใช่ error แต่อย่างใด เป็นผลลัพธ์ที่ถูกต้องตามตรรกะทางคณิตศาสตร์

---

## ขั้นตอนที่ 724: ออกแบบการแปลงข้อมูล JSON กับข้อมูล COBOL

### ทำไมต้องออกแบบการแปลงข้อมูลล่วงหน้า

ก่อนเขียนโค้ด Wrapper Service เราต้องตัดสินใจล่วงหน้าว่า field ของ JSON แต่ละตัวจะแปลงเป็นข้อมูล
ฝั่ง COBOL อย่างไร เพราะทั้งสองฝั่งมีข้อจำกัดต่างกันมาก: JSON มีชนิดข้อมูลแค่ string/number/boolean/
null/object/array ในขณะที่ COBOL มี `PIC` ที่ระบุความกว้างและตำแหน่งทศนิยมตายตัว (ทบทวนจาก Part 006)

### ตารางการแมปข้อมูลของระบบ ORDERCALC

| JSON field | ชนิดข้อมูล JSON | รูปแบบที่ส่งให้ COBOL (stdin) | field ปลายทางใน COBOL |
|---|---|---|---|
| `price` | number (float) | ข้อความ เช่น `"100.00"` ต่อท้ายด้วย newline | `WS-IN-PRICE PIC X(10)` แล้วแปลงด้วย `NUMVAL` |
| `quantity` | number (int) | ข้อความ เช่น `"3"` ต่อท้ายด้วย newline | `WS-IN-QTY PIC X(10)` แล้วแปลงด้วย `NUMVAL` |
| (ผลลัพธ์) `subtotal`, `tax`, `total` | number (float) | อ่านจาก stdout รูปแบบ `"300.00\|21.00\|321.00"` | มาจาก `PIC Z(7)9.99` ของ COBOL |

### หลักการออกแบบที่สำคัญ 3 ข้อ

1. **ความกว้างของ field ฝั่ง COBOL ต้องเผื่อไว้มากกว่าค่าที่คาดว่าจะได้รับจริงเสมอ** เช่น
   `WS-IN-PRICE PIC X(10)` รองรับได้ถึง 10 ตัวอักษร ซึ่งเพียงพอสำหรับราคาสูงสุด `9999999.99`
   (10 ตัวอักษรพอดี) หากคาดว่าราคาอาจสูงกว่านี้ต้องขยาย `PIC` ให้กว้างขึ้นตั้งแต่ตอนออกแบบ
2. **ใช้ตัวคั่น (delimiter) ที่ไม่มีทางปรากฏในข้อมูลจริง** เราเลือก `|` เพราะราคาและจำนวนสินค้า
   ไม่มีทางมีเครื่องหมาย pipe ปนอยู่ ถ้าข้อมูลจริงอาจมี `|` ปนอยู่ได้ (เช่น ชื่อสินค้า) ต้องเลือก
   ตัวคั่นอื่นหรือใช้เทคนิคความกว้างคงที่ (Fixed-Width Field) แทน
3. **แปลง JSON เป็นข้อความธรรมดาที่สุดเท่าที่จะทำได้ก่อนส่งให้ COBOL** อย่าส่ง JSON ดิบเข้าไปให้
   COBOL parse เอง เพราะ Part 045 พิสูจน์แล้วว่าบิลด์นี้ไม่รองรับ `JSON PARSE` — ให้ Wrapper Service
   (ฝั่ง Python) เป็นผู้รับผิดชอบแปลง JSON ทั้งหมด แล้วส่งเฉพาะค่าดิบ (raw value) เป็นข้อความธรรมดา
   ให้ COBOL เท่านั้น

### ตัวอย่างการแปลงข้อมูลด้วยมือ (ภาพจำลองก่อนเขียนโค้ดจริง)

```
JSON ขาเข้า:  {"price": 100.00, "quantity": 3}
                    |
                    v  (Wrapper แปลง)
stdin ของ COBOL:  "100.00\n3\n"
                    |
                    v  (COBOL ประมวลผล)
stdout ของ COBOL: "300.00|21.00|321.00"
                    |
                    v  (Wrapper แปลงกลับ)
JSON ขาออก:  {"price": 100.0, "quantity": 3, "subtotal": 300.0,
              "tax": 21.0, "total": 321.0}
```

### ข้อควรระวัง

- อย่าลืมต่อท้ายค่าที่ส่งเข้า stdin ด้วยอักขระขึ้นบรรทัดใหม่ (`\n`) เสมอ เพราะ `ACCEPT ... FROM
  CONSOLE` ใน COBOL อ่านข้อมูลทีละบรรทัด หากไม่มี `\n` คั่น โปรแกรม COBOL อาจรอข้อมูลค้างอยู่ไม่จบ
  (hang) จนกว่าจะ timeout หรือ crash
- ตัดสินใจเรื่องการแปลงข้อมูลให้ชัดเจนก่อนเขียนโค้ด Wrapper — การแก้ไขรูปแบบการสื่อสารภายหลังจาก
  ที่ทั้งสองฝั่งเขียนเสร็จแล้วมักทำให้ต้องแก้โค้ดทั้งสองฝั่งพร้อมกัน

### แบบฝึกหัดที่ 724.1

**โจทย์**: หากต้องการเพิ่ม field `"customer-name"` (ข้อความ ไม่ใช่ตัวเลข) เข้าไปในระบบ จงออกแบบว่า
ควรส่งค่านี้เข้า stdin ของ COBOL อย่างไร และมีข้อควรระวังอะไรเพิ่มเติมที่ต่างจากข้อมูลตัวเลข

**เฉลยแนวทาง**: ส่งเป็นบรรทัดข้อความเพิ่มอีก 1 บรรทัดเข้า stdin เช่นเดียวกับ `price`/`quantity`
แต่ต้องระวังเรื่อง **ความกว้างสูงสุดของชื่อ** (ต้องตัดหรือ validate ความยาวก่อนส่ง มิฉะนั้น
`ACCEPT` อาจตัดข้อความที่ยาวเกิน field ปลายทางทิ้งอย่างเงียบ ๆ) และต้องระวัง**อักขระพิเศษที่อาจ
ปนมากับชื่อ** เช่น เครื่องหมาย pipe ที่เราใช้เป็นตัวคั่นผลลัพธ์ หากชื่อลูกค้ามี `|` ปนอยู่ (แม้ไม่น่า
เกิดขึ้นในทางปฏิบัติ) จะทำให้การแยกผลลัพธ์ที่ฝั่ง Python สับสนได้

---

## ขั้นตอนที่ 725: สร้าง HTTP Wrapper ด้วย Python Standard Library (http.server)

### ทำไมเลือก http.server แทน Framework อื่น

Python มี module `http.server` เป็นส่วนหนึ่งของ Standard Library ติดตั้งมาพร้อมตัวแปลภาษาอยู่แล้ว
ไม่ต้อง `pip install` Flask, FastAPI หรือ library ภายนอกใด ๆ เลย — เหมาะมากสำหรับสภาพแวดล้อมที่
จำกัดการติดตั้ง package เพิ่มเติม (เช่น เครื่อง Mainframe หรือเซิร์ฟเวอร์ภายในองค์กรที่ควบคุมเข้มงวด)
และเพียงพอสำหรับสาธิตแนวคิด Wrapper Service Pattern อย่างสมบูรณ์

### โครงสร้างพื้นฐานของ Wrapper Service

```python
import json
import subprocess
from http.server import BaseHTTPRequestHandler, HTTPServer

COBOL_BINARY = "./ordercalc"


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/api/order":
            self.send_response(404)
            self.end_headers()
            return
        # Step 1: read the raw JSON body sent by the client.
        length = int(self.headers.get("Content-Length", 0))
        raw = self.rfile.read(length)
        data = json.loads(raw)
        # ... translate, call COBOL, respond (next steps) ...

    def log_message(self, fmt, *args):
        pass  # keep the demo output quiet


if __name__ == "__main__":
    server = HTTPServer(("127.0.0.1", 8073), Handler)
    server.serve_forever()
```

### อธิบายทีละส่วน

- `class Handler(BaseHTTPRequestHandler)`: คลาสนี้เป็นแกนหลักของ `http.server` — เราสืบทอด
  (inherit) มาแล้ว override method `do_POST` เพื่อจัดการ HTTP POST request โดยเฉพาะ (คล้ายแนวคิด
  OOP ที่แม้ COBOL จะมีใน COBOL-2002 แต่หลักสูตรนี้เน้น Procedural ตามที่กล่าวไว้ใน Part 001)
- `self.headers.get("Content-Length", 0)`: HTTP request จะบอกความยาวของ body มาใน header
  `Content-Length` เราต้องอ่านค่านี้ก่อนเพื่อรู้ว่าจะอ่าน body กี่ไบต์ (`self.rfile.read(length)`)
- `self.path`: เก็บ URL path ของ request (เช่น `/api/order`) ใช้สำหรับทำ routing แบบง่าย ๆ ด้วย
  `if` เทียบ path ตรง ๆ — เพียงพอสำหรับ endpoint เดียวในตัวอย่างนี้
- `log_message`: override ทิ้งไว้เป็นฟังก์ชันว่างเพื่อไม่ให้ `http.server` พิมพ์ access log ทุก
  request ออกทาง stderr รกหน้าจอระหว่างสาธิต (ในระบบจริงควรเก็บ log ไว้ใช้ debug แทนที่จะปิดทิ้ง)

### ข้อควรระวัง

- `HTTPServer` มาตรฐานเป็นแบบ **single-threaded**: รับ request ได้ทีละ 1 request เท่านั้น ถ้ามี
  request ที่สองเข้ามาระหว่างกำลังประมวลผล request แรก จะต้องรอคิว — ประเด็นนี้จะอธิบายเพิ่มเติมและ
  แก้ไขในขั้นตอนที่ 729
- การ bind ที่ `127.0.0.1` (localhost) แทน `0.0.0.0` หมายความว่า server จะรับ connection จากเครื่อง
  ตัวเองเท่านั้น ปลอดภัยสำหรับการทดสอบ แต่หากต้องการให้เครื่องอื่นในเครือข่ายเข้าถึงได้ต้องเปลี่ยนเป็น
  `0.0.0.0` (และต้องเพิ่มมาตรการความปลอดภัยอื่น ๆ ตามความเหมาะสม)

### แบบฝึกหัดที่ 725.1

**โจทย์**: จงอธิบายว่าทำไมต้อง override method `do_POST` แทนที่จะใช้ `do_GET`

**เฉลย**: เพราะเราออกแบบ API endpoint `/api/order` ให้รับข้อมูลคำสั่งซื้อ (price, quantity) ผ่าน
**HTTP request body** ซึ่งเป็นรูปแบบมาตรฐานของ HTTP method `POST` (ใช้ส่งข้อมูลไปสร้าง/ประมวลผล
บางอย่างที่ฝั่ง server) ในขณะที่ `GET` ตามธรรมเนียม HTTP ควรใช้สำหรับการ "ขอข้อมูล" อย่างเดียวโดยไม่มี
body และไม่ควรมีผลข้างเคียง (side effect) การเลือก method ให้ตรงกับความหมายของ action ก็เป็นส่วนหนึ่ง
ของการออกแบบ REST API ที่ดี

---

## ขั้นตอนที่ 726: เชื่อมต่อ subprocess — เรียก COBOL และจับผลลัพธ์

### หัวใจของ Wrapper Service: subprocess.run

Python มี module `subprocess` ในมาตรฐานที่ใช้เรียกโปรแกรมภายนอก (เหมือนรันคำสั่งใน terminal) แล้ว
รับผลลัพธ์กลับมาในโค้ด Python ได้โดยตรง — นี่คือกลไกเดียวกับที่เราทดสอบด้วยมือผ่าน `printf | ./ordercalc`
ในขั้นตอนที่ 723 เพียงแต่ทำผ่านโค้ดแทนการพิมพ์ใน terminal เอง

### โค้ดสมบูรณ์ของ Wrapper Service (เวอร์ชันแรก)

```python
import json
import subprocess
from http.server import BaseHTTPRequestHandler, HTTPServer

COBOL_BINARY = "./ordercalc"


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/api/order":
            self.send_response(404)
            self.end_headers()
            return
        length = int(self.headers.get("Content-Length", 0))
        raw = self.rfile.read(length)
        data = json.loads(raw)
        price = str(data["price"])
        qty = str(data["quantity"])

        # Build the two-line stdin text ORDERCALC expects, then
        # run it exactly like `printf "price\nqty\n" | ./ordercalc`
        # would from a shell.
        stdin_text = price + "\n" + qty + "\n"
        result = subprocess.run(
            [COBOL_BINARY],
            input=stdin_text,
            capture_output=True,
            text=True,
            timeout=5,
        )
        line = result.stdout.strip()
        subtotal, tax, total = line.split("|")

        response = {
            "price": float(price),
            "quantity": int(qty),
            "subtotal": float(subtotal),
            "tax": float(tax),
            "total": float(total),
        }
        body = json.dumps(response).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, fmt, *args):
        pass


if __name__ == "__main__":
    server = HTTPServer(("127.0.0.1", 8073), Handler)
    server.serve_forever()
```

### อธิบายจุดสำคัญของ subprocess.run

- `[COBOL_BINARY]`: รายการคำสั่งที่จะรัน (list ของ string) — ใช้ list แทน string เดี่ยวเพื่อ
  หลีกเลี่ยงปัญหาการตีความ shell ที่ซับซ้อน (shell injection) เพราะ `subprocess.run` ที่ได้รับ
  list จะเรียก program โดยตรงโดยไม่ผ่าน shell interpreter
- `input=stdin_text`: ค่าข้อความที่จะป้อนเข้า stdin ของ process ที่ถูกเรียก เทียบเท่ากับสิ่งที่
  `printf "..." |` ทำในขั้นตอนที่ 723 ทุกประการ
- `capture_output=True`: บอกให้ `subprocess.run` เก็บทั้ง stdout และ stderr ของโปรแกรมที่ถูกเรียก
  ไว้ในผลลัพธ์ (`result.stdout`, `result.stderr`) แทนที่จะปล่อยให้แสดงตรง ๆ ทาง terminal
- `text=True`: บอกให้ทำงานกับข้อมูลเป็น string (str) แทนที่จะเป็น bytes ดิบ ทำให้เรียก
  `result.stdout.strip()` และ `.split("|")` ได้ตรงไปตรงมา
- `timeout=5`: กำหนดเวลาสูงสุดที่จะรอ subprocess ทำงาน (5 วินาที) หาก COBOL program ค้าง
  (hang) ด้วยเหตุผลใดก็ตาม `subprocess.run` จะโยน `subprocess.TimeoutExpired` exception แทนที่
  จะรอตลอดไปและทำให้ Wrapper Service ค้างตามไปด้วย

### ข้อควรระวัง

- **ทุกครั้งที่เรียก subprocess คือการสร้าง process ใหม่ 1 process** ซึ่งมีต้นทุน (overhead) ด้าน
  เวลาและทรัพยากรมากกว่าการเรียกฟังก์ชันภายในโปรแกรมเดียวกันมาก สำหรับระบบที่ต้องรับ request ปริมาณ
  สูงมาก ต้นทุนนี้อาจกลายเป็นคอขวด (bottleneck) — ประเด็นนี้จะอธิบายเพิ่มเติมในขั้นตอนที่ 729
- ต้องกำหนด `timeout` เสมอสำหรับ subprocess ที่เรียกจาก HTTP handler ไม่เช่นนั้น request ที่ค้าง
  จะทำให้ HTTP connection ค้างตามไปด้วยไม่มีกำหนด

### แบบฝึกหัดที่ 726.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `subprocess.run([COBOL_BINARY], ...)` (แบบ list) จึงปลอดภัยกว่า
การใช้ `subprocess.run(COBOL_BINARY, shell=True)` (แบบ string ผ่าน shell) โดยเฉพาะเมื่อค่าบางส่วน
ของคำสั่งมาจาก input ของผู้ใช้

**เฉลย**: เมื่อใช้ `shell=True` ระบบจะส่งคำสั่งทั้งหมดผ่าน shell interpreter (เช่น `/bin/sh`) ก่อน
หากมีค่าจาก input ของผู้ใช้ปนอยู่ในคำสั่งนั้นโดยไม่ได้ระวัง (เช่น ผู้ใช้ส่งค่าที่มีอักขระพิเศษของ
shell อย่าง `;`, `|`, `` ` ``) อาจทำให้เกิดการรันคำสั่งอื่นที่ไม่ได้ตั้งใจ เรียกว่า **Shell/Command
Injection** ซึ่งเป็นช่องโหว่ความปลอดภัยร้ายแรง ในขณะที่การใช้ list (`[COBOL_BINARY]`) โดยไม่ผ่าน
shell ทำให้ระบบเรียกโปรแกรมโดยตรงด้วยชื่อไฟล์ที่กำหนดตายตัว ไม่มีการตีความอักขระพิเศษใด ๆ จึงปลอดภัย
กว่ามากเมื่อค่าบางส่วนมาจากภายนอก (ในตัวอย่างของเรา ค่าจากผู้ใช้ถูกส่งผ่าน `input=` ทาง stdin
ไม่ได้ถูกนำไปประกอบเป็นส่วนหนึ่งของคำสั่งเชลล์เลย จึงปลอดภัยยิ่งขึ้นไปอีกชั้น)

---

## ขั้นตอนที่ 727: รันระบบจริงแบบ End-to-End พร้อมทดสอบด้วย curl

### ขั้นตอนการรันระบบทั้งหมด

**Terminal ที่ 1 — คอมไพล์และเริ่ม Wrapper Service:**

```bash
cobc -x -o ordercalc ordercalc.cob
python3 wrapper.py
```

โปรแกรม `wrapper.py` จะเริ่มทำงานและรอรับ HTTP request ที่ `http://127.0.0.1:8073` ค้างอยู่ (ไม่มี
ข้อความแสดงผลใด ๆ เพราะเราปิด `log_message` ไว้แล้ว)

**Terminal ที่ 2 — ทดสอบด้วย curl:**

```bash
curl -s -X POST http://127.0.0.1:8073/api/order \
    -H "Content-Type: application/json" \
    -d '{"price": 100.00, "quantity": 3}'
```

**ผลลัพธ์จริงที่ได้ (ทดสอบแล้ว):**

```
{"price": 100.0, "quantity": 3, "subtotal": 300.0, "tax": 21.0, "total": 321.0}
```

ทดสอบชุดที่สองด้วยตัวเลขทศนิยม:

```bash
curl -s -X POST http://127.0.0.1:8073/api/order \
    -H "Content-Type: application/json" \
    -d '{"price": 19.99, "quantity": 5}'
```

**ผลลัพธ์จริงที่ได้:**

```
{"price": 19.99, "quantity": 5, "subtotal": 99.95, "tax": 6.99, "total": 106.94}
```

ผลลัพธ์ตรงกับที่ทดสอบ standalone ในขั้นตอนที่ 723 ทุกประการ (`99.95|6.99|106.94`) พิสูจน์ว่า
Wrapper Service แปลงข้อมูลไป-กลับได้ถูกต้อง

### ลำดับเหตุการณ์ที่เกิดขึ้นจริงเบื้องหลัง 1 คำขอ

1. `curl` ส่ง HTTP POST พร้อม JSON body `{"price": 19.99, "quantity": 5}` ไปที่ port 8073
2. `HTTPServer` ของ Python รับ connection แล้วเรียก `do_POST` ของ `Handler`
3. `do_POST` อ่าน body, แปลงเป็น dict ด้วย `json.loads`, ดึงค่า `price`/`quantity` ออกมาเป็น string
4. สร้างข้อความ `"19.99\n5\n"` แล้วส่งเป็น `input` ให้ `subprocess.run(["./ordercalc"], ...)`
5. ระบบปฏิบัติการสร้าง process ใหม่รัน `ordercalc`, ป้อน stdin, รอจนโปรแกรมจบ (`STOP RUN`)
6. `ordercalc` คำนวณและ `DISPLAY` ผลลัพธ์ `"99.95|6.99|106.94"` ออกทาง stdout
7. `subprocess.run` เก็บ stdout นี้ไว้ใน `result.stdout` แล้วคืนค่ากลับให้ `do_POST`
8. `do_POST` แยกค่าด้วย `.split("|")`, ประกอบเป็น JSON response, ส่งกลับไปให้ `curl`

### ข้อควรระวัง

- ต้องเปิด Wrapper Service (`python3 wrapper.py`) ค้างไว้ในอีก terminal หนึ่งเสมอก่อนยิง `curl`
  ทดสอบ — เป็นความสับสนที่พบบ่อยของผู้เริ่มต้นทำงานกับ client-server pattern
- ถ้า `curl` ได้รับ connection refused ให้ตรวจสอบว่า Wrapper Service ยังทำงานอยู่จริงและ bind
  port ตรงกับที่ `curl` เรียก (`8073` ในตัวอย่างนี้)

### แบบฝึกหัดที่ 727.1

**โจทย์**: จงเขียนคำสั่ง `curl` สำหรับทดสอบคำสั่งซื้อราคา `250.50` จำนวน `8` ชิ้น แล้วคำนวณด้วยมือ
ว่าผลลัพธ์ที่ควรได้คืออะไร

**เฉลย**: คำสั่ง:

```bash
curl -s -X POST http://127.0.0.1:8073/api/order \
    -H "Content-Type: application/json" \
    -d '{"price": 250.50, "quantity": 8}'
```

คำนวณด้วยมือ: subtotal = 250.50 x 8 = 2004.00, tax = 2004.00 x 0.07 = 140.28,
total = 2004.00 + 140.28 = 2144.28 ผลลัพธ์ JSON ที่คาดว่าจะได้คือ
`{"price": 250.5, "quantity": 8, "subtotal": 2004.0, "tax": 140.28, "total": 2144.28}`

---

## ขั้นตอนที่ 728: การจัดการ Input ที่ไม่ถูกต้อง — บั๊กจริงที่พบระหว่างการพัฒนา

### ปัญหาที่ตรวจพบจริงระหว่างทดสอบ

ระหว่างพัฒนา Part นี้ เราทดสอบส่งค่า `price` เป็นข้อความที่ไม่ใช่ตัวเลข (`"abc"`) เข้า Wrapper
Service เวอร์ชันแรก (จากขั้นตอนที่ 726) และพบปัญหาจริง:

```bash
curl -s -w "\n[http %{http_code}]\n" -X POST http://127.0.0.1:8073/api/order \
    -H "Content-Type: application/json" \
    -d '{"price": "abc", "quantity": 3}'
```

**ผลลัพธ์จริงที่ได้:**

```
[http 000]
```

`http 000` หมายความว่า `curl` **ไม่ได้รับการตอบกลับใด ๆ เลย** — connection ถูกตัดกลางคัน ตรวจสอบ
log ฝั่ง server พบ traceback จริงดังนี้:

```
Traceback (most recent call last):
  ...
  File "wrapper.py", line 38, in do_POST
    "price": float(price),
             ^^^^^^^^^^^^
ValueError: could not convert string to float: 'abc'
```

### วิเคราะห์สาเหตุ

เกิดอะไรขึ้น: `price = str(data["price"])` แปลง `"abc"` เป็น string `"abc"` (ไม่ error ณ จุดนี้)
จากนั้นส่งเข้า COBOL ทาง stdin — และตามที่พิสูจน์แล้วในขั้นตอนที่ 723 `FUNCTION NUMVAL("abc")`
คืนค่า 0 อย่างเงียบ ๆ ทำให้ COBOL คืนผลลัพธ์ `"0.00|0.00|0.00"` กลับมาได้ตามปกติ **แต่ปัญหาที่แท้จริง
เกิดตอนสร้าง JSON response**: โค้ดพยายาม `float(price)` โดย `price` ยังเป็น string เดิม `"abc"`
ที่ไม่เคยถูกแปลงเป็นตัวเลขเลยตั้งแต่ต้น ทำให้เกิด `ValueError` ที่ไม่ได้ถูกดักจับ (unhandled
exception) ส่งผลให้ `http.server` ตัด connection ทิ้งโดยไม่ตอบกลับ client เลย

นี่คือบทเรียนสำคัญ: **การตรวจสอบ input ต้องทำที่ Wrapper Service ก่อนเรียก COBOL เสมอ** ไม่ใช่หวัง
พึ่งให้ COBOL หรือขั้นตอนสร้าง response ตรวจจับปัญหาแทน

### โค้ดฉบับแก้ไข (เวอร์ชันที่ปลอดภัยกว่า)

```python
import json
import subprocess
from http.server import BaseHTTPRequestHandler, HTTPServer

COBOL_BINARY = "./ordercalc"


def send_json(handler, status, payload):
    body = json.dumps(payload).encode()
    handler.send_response(status)
    handler.send_header("Content-Type", "application/json")
    handler.send_header("Content-Length", str(len(body)))
    handler.end_headers()
    handler.wfile.write(body)


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/api/order":
            send_json(self, 404, {"error": "not found"})
            return

        length = int(self.headers.get("Content-Length", 0))
        raw = self.rfile.read(length)
        try:
            data = json.loads(raw)
            price = float(data["price"])
            qty = int(data["quantity"])
        except (ValueError, KeyError, TypeError):
            send_json(self, 400,
                      {"error": "price and quantity must be numbers"})
            return

        if price < 0 or qty < 0:
            send_json(self, 400,
                      {"error": "price and quantity must not be negative"})
            return

        stdin_text = "%.2f\n%d\n" % (price, qty)
        result = subprocess.run(
            [COBOL_BINARY], input=stdin_text,
            capture_output=True, text=True, timeout=5,
        )
        subtotal, tax, total = result.stdout.strip().split("|")
        send_json(self, 200, {
            "price": price, "quantity": qty,
            "subtotal": float(subtotal), "tax": float(tax),
            "total": float(total),
        })

    def log_message(self, fmt, *args):
        pass


if __name__ == "__main__":
    HTTPServer(("127.0.0.1", 8073), Handler).serve_forever()
```

### ทดสอบเวอร์ชันที่แก้ไขแล้ว — ผลลัพธ์จริง

```bash
curl -s -w "\n[http %{http_code}]\n" -X POST http://127.0.0.1:8073/api/order \
    -H "Content-Type: application/json" -d '{"price": "abc", "quantity": 3}'
```

**ผลลัพธ์จริง:**

```
{"error": "price and quantity must be numbers"}
[http 400]
```

ทดสอบกรณีค่าติดลบ:

```bash
curl -s -w "\n[http %{http_code}]\n" -X POST http://127.0.0.1:8073/api/order \
    -H "Content-Type: application/json" -d '{"price": -5, "quantity": 3}'
```

**ผลลัพธ์จริง:**

```
{"error": "price and quantity must not be negative"}
[http 400]
```

และคำขอปกติยังคงทำงานถูกต้องเหมือนเดิม:

```
{"price": 100.0, "quantity": 3, "subtotal": 300.0, "tax": 21.0, "total": 321.0}
```

### อธิบายการแก้ไข

- ย้ายการแปลงชนิดข้อมูล (`float(data["price"])`, `int(data["quantity"])`) ให้เกิดขึ้น**ก่อน**เรียก
  COBOL แทนที่จะเรียก COBOL ไปก่อนแล้วค่อยแปลงชนิดตอนสร้าง response — หาก `data["price"]` แปลง
  เป็น float ไม่ได้ จะโยน `ValueError` ทันทีตรงจุดที่ครอบด้วย `try/except` พอดี
- เพิ่มการตรวจสอบค่าติดลบ (`price < 0 or qty < 0`) เป็นตัวอย่างของ **Business Rule Validation**
  ที่ควรอยู่ฝั่ง Wrapper Service เพื่อป้องกันไม่ให้ค่าที่ไม่สมเหตุสมผลทางธุรกิจหลุดเข้าไปถึง COBOL
- สร้างฟังก์ชันช่วย `send_json` เพื่อลดโค้ดซ้ำซ้อนในการส่ง HTTP response ทุกจุด — หลักการ DRY
  (Don't Repeat Yourself) ที่ใช้ได้กับทุกภาษารวมถึง COBOL (เทียบเท่าการใช้ `PERFORM paragraph-name`
  เพื่อลดโค้ดซ้ำซ้อนที่เรียนมาตั้งแต่ Part 014)

### ข้อควรระวัง

- **บทเรียนสำคัญที่สุดของขั้นตอนนี้**: จุดที่มีโอกาส error ในการแปลงชนิดข้อมูล (type conversion)
  ต้องถูกครอบด้วย exception handling เสมอ โดยเฉพาะเมื่อค่านั้นมาจากภายนอกระบบ (external input)
  ที่เราไม่สามารถควบคุมความถูกต้องได้ล่วงหน้า
- Unhandled exception ใน HTTP handler ไม่ได้แค่ทำให้ request นั้นล้มเหลว แต่ยังทำให้ client ได้รับ
  "http 000" (ไม่มีการตอบกลับเลย) ซึ่งสร้างความสับสนมากกว่าการได้รับ error code ที่ชัดเจนอย่าง 400

### แบบฝึกหัดที่ 728.1

**โจทย์**: จงเพิ่มการตรวจสอบใน Wrapper Service เวอร์ชันแก้ไขแล้ว ให้ปฏิเสธคำขอที่ `quantity`
มากกว่า 10,000 ชิ้น ด้วย HTTP status 400 และข้อความ `"quantity too large"`

**เฉลย**: เพิ่มเงื่อนไขต่อจากการตรวจสอบค่าติดลบ:

```python
        if qty > 10000:
            send_json(self, 400, {"error": "quantity too large"})
            return
```

---

## ขั้นตอนที่ 729: Concurrency และประสิทธิภาพของ Wrapper Service

### ปัญหา: HTTPServer มาตรฐานรับได้ทีละ 1 คำขอ

`HTTPServer` พื้นฐานที่ใช้ตลอด Part นี้เป็นแบบ **single-threaded** — ขณะกำลังประมวลผล 1 คำขอ
(รวมถึงเวลาที่ใช้รอ subprocess COBOL ทำงานจนเสร็จ) คำขออื่นที่เข้ามาพร้อมกันจะต้องรอคิวจนกว่าคำขอ
ก่อนหน้าจะเสร็จสิ้น สำหรับระบบสาธิตหรือระบบภายในที่มี traffic น้อยนี่ไม่ใช่ปัญหา แต่สำหรับระบบที่
ต้องรองรับผู้ใช้จำนวนมากพร้อมกันจะกลายเป็นคอขวดทันที

### ทางแก้เบื้องต้น: ThreadingHTTPServer

Python standard library มีคลาส `ThreadingHTTPServer` ที่สร้าง thread ใหม่ให้ทุกคำขอโดยอัตโนมัติ
ทำให้รับหลายคำขอพร้อมกันได้ในระดับหนึ่ง แก้ไขโค้ดเพียงบรรทัดเดียว:

```python
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

# ... (Handler class เหมือนเดิมทุกประการ) ...

if __name__ == "__main__":
    ThreadingHTTPServer(("127.0.0.1", 8073), Handler).serve_forever()
```

### ข้อจำกัดที่ยังคงอยู่แม้ใช้ ThreadingHTTPServer

แม้ `ThreadingHTTPServer` จะแก้ปัญหาการรอคิวของ HTTP layer ได้ แต่ **ต้นทุนของการสร้าง
subprocess ใหม่ทุกครั้งที่เรียก COBOL ยังคงอยู่เหมือนเดิม** — แต่ละ request ยังคงต้อง:

1. สร้าง process ใหม่ (fork + exec) ซึ่งมีต้นทุนด้านเวลาและหน่วยความจำ
2. โหลดตัว executable COBOL เข้าหน่วยความจำใหม่ทุกครั้ง
3. รอ process จบแล้วทำลายทิ้ง (cleanup)

สำหรับระบบที่ต้องการประสิทธิภาพสูงมาก แนวทางที่ก้าวหน้ากว่านี้ (นอกเหนือขอบเขตของ Part นี้ แต่ควร
รู้จักไว้) ได้แก่:

- **Process Pool**: เปิด COBOL process ค้างไว้ล่วงหน้าหลาย process แล้วหมุนเวียนส่งงานเข้าไป
  แทนที่จะสร้าง process ใหม่ทุกครั้ง (ต้องออกแบบให้ COBOL program รับงานหลายรอบใน 1 process เดียว
  ผ่านลูปอ่าน stdin ต่อเนื่อง แทนที่จะ `STOP RUN` หลังงานเดียว)
- **Message Queue** (เช่น RabbitMQ, Apache Kafka): Wrapper Service ส่งงานเข้าคิวแทนการเรียก
  subprocess ตรง ๆ แล้วมี Worker แยกต่างหากดึงงานจากคิวไปประมวลผลด้วย COBOL แบบขนาน เหมาะกับงาน
  batch ปริมาณมากที่ไม่ต้องการคำตอบทันที (asynchronous processing)
- **CICS/Transaction Server**: ในสภาพแวดล้อม Mainframe จริง มักใช้ CICS (ทบทวนจาก Part 061-064)
  ที่ออกแบบมาสำหรับรับธุรกรรมปริมาณสูงมากโดยเฉพาะ แทนการเรียก subprocess แบบที่สาธิตใน Part นี้

### ข้อควรระวัง

- อย่าคิดว่า `ThreadingHTTPServer` แก้ปัญหาประสิทธิภาพได้ทั้งหมด — เป็นเพียงการแก้ปัญหาระดับ HTTP
  layer เท่านั้น ต้นทุนที่แท้จริงของ subprocess ยังคงอยู่และอาจกลายเป็นคอขวดใหม่หากมี thread จำนวน
  มากพยายามสร้าง subprocess พร้อมกันเกินกว่าทรัพยากรเครื่องจะรองรับได้
- Pattern ที่สอนใน Part นี้ (`subprocess.run` แบบตรง ๆ) เหมาะสำหรับ**การเรียนรู้แนวคิดและระบบที่มี
  ปริมาณ request ไม่สูงมาก** สำหรับระบบ production ที่ต้องรองรับ traffic สูง ควรศึกษาต่อยอดด้วย
  แนวทาง Process Pool หรือ Message Queue ที่กล่าวถึงข้างต้น

### แบบฝึกหัดที่ 729.1

**โจทย์**: จงอธิบายว่าทำไม Message Queue Pattern จึงเหมาะกับงาน "ที่ไม่ต้องการคำตอบทันที" มากกว่า
งาน "คำนวณราคาแบบ real-time" อย่าง `ORDERCALC` ในตัวอย่างของเรา

**เฉลยแนวทาง**: Message Queue Pattern ทำงานแบบ asynchronous คือผู้ส่งงาน (producer) ส่งงานเข้าคิว
แล้วเดินหน้าต่อทันทีโดยไม่รอผลลัพธ์ ส่วนผลลัพธ์จะถูกประมวลผลและส่งกลับในภายหลัง (เช่น ผ่าน callback,
polling, หรือ webhook) ซึ่งเหมาะกับงานที่ใช้เวลานาน เช่น การประมวลผล batch รายงานสิ้นเดือน แต่ไม่
เหมาะกับ `ORDERCALC` เพราะผู้ใช้ (เช่น หน้าเว็บแสดงราคาสินค้า) ต้องการเห็นผลลัพธ์การคำนวณ**ทันที**
เพื่อแสดงผลต่อผู้ซื้อ การรอคิวแบบ asynchronous จะทำให้ประสบการณ์ผู้ใช้แย่ลงอย่างมาก กรณีนี้จึงเหมาะกับ
รูปแบบ synchronous request-response แบบที่สาธิตใน Part นี้มากกว่า

---

## ขั้นตอนที่ 730: สรุปภาพรวม Pattern และการนำไปประยุกต์ใช้จริง

### สรุปสถาปัตยกรรมทั้งหมดของ Part นี้

```
curl / Mobile App / Web Frontend
        |
        | HTTP POST /api/order + JSON body
        v
Python Wrapper Service (http.server, port 8073)
  - แปลง JSON -> stdin text
  - subprocess.run(["./ordercalc"], input=..., timeout=5)
  - แปลง stdout text -> JSON
        |
        | stdin: "100.00\n3\n"
        v
ORDERCALC (โปรแกรม COBOL, compiled ด้วย cobc)
  - ACCEPT ... FROM CONSOLE
  - COMPUTE business logic
  - DISPLAY ผลลัพธ์
        |
        | stdout: "300.00|21.00|321.00"
        v
กลับขึ้นไปที่ Wrapper Service -> ประกอบเป็น JSON -> ตอบกลับ curl
```

### ตารางสรุปสิ่งที่แต่ละฝั่งรับผิดชอบ

| ฝั่ง | ภาษา | หน้าที่ | ไม่ต้องรู้เรื่อง |
|---|---|---|---|
| COBOL (ORDERCALC) | GnuCOBOL | Business logic การคำนวณล้วน ๆ | HTTP, JSON, network ใด ๆ ทั้งสิ้น |
| Wrapper Service | Python (stdlib เท่านั้น) | HTTP server, แปลงข้อมูล, subprocess | Business logic การคำนวณราคา/ภาษี |

### เมื่อไหร่ควรใช้ Pattern นี้ และเมื่อไหร่ไม่ควร

**ควรใช้เมื่อ:**
- มีโปรแกรม COBOL ที่ผ่านการทดสอบและใช้งานจริงมานาน ต้องการเปิดเป็น API โดยไม่แก้โค้ดเดิม
- ปริมาณ request ไม่สูงมาก (หลักสิบถึงหลักร้อยต่อวินาที ขึ้นกับทรัพยากรเครื่อง)
- ต้องการต้นแบบ (prototype) อย่างรวดเร็วโดยไม่ต้องติดตั้ง library หรือโครงสร้างพื้นฐานเพิ่มเติม

**ไม่ควรใช้ (หรือควรพิจารณาแนวทางอื่นเพิ่มเติม) เมื่อ:**
- ต้องการ throughput สูงมาก (หลักพัน-หมื่น request ต่อวินาที) — ควรพิจารณา Process Pool หรือ
  Message Queue ตามที่กล่าวในขั้นตอนที่ 729
- ต้องการ Transaction ที่ซับซ้อนข้ามหลายขั้นตอน (multi-step transaction) — ควรพิจารณา CICS หรือ
  ระบบ Transaction Processing เฉพาะทาง (ทบทวนจาก Part 061-064)

### ความปลอดภัยที่ต้องพิจารณาเพิ่มเติมในระบบจริง (นอกเหนือจากที่สาธิตแล้ว)

- **Authentication/Authorization**: ตัวอย่างใน Part นี้ไม่มีการตรวจสอบสิทธิ์ผู้เรียกใช้ API เลย
  ระบบจริงต้องเพิ่มกลไกยืนยันตัวตน เช่น API Key หรือ OAuth Token
- **Rate Limiting**: จำกัดจำนวน request ต่อวินาทีต่อผู้ใช้ เพื่อป้องกันการเรียก subprocess
  ถี่เกินไปจนทำให้เครื่องทำงานหนักเกินไป (resource exhaustion)
- **Input Sanitization ที่ครบถ้วนกว่านี้**: ตัวอย่างของเราตรวจสอบแค่ชนิดข้อมูลและค่าติดลบ ระบบจริง
  ควรตรวจสอบขอบเขตค่าที่สมเหตุสมผลทางธุรกิจอย่างละเอียดกว่านี้เสมอ

### แบบฝึกหัดที่ 730.1 (แบบฝึกหัดรวม Part นี้)

**โจทย์**: จงออกแบบ (โดยไม่ต้องเขียนโค้ดเต็ม) ว่าหากต้องการเพิ่ม endpoint ใหม่ `/api/discount`
ที่รับ `price`, `quantity`, และ `discount-percent` แล้วคำนวณราคาหลังหักส่วนลดก่อนคิดภาษี จะต้อง
แก้ไขทั้งฝั่ง COBOL และฝั่ง Wrapper Service อย่างไรบ้าง

**เฉลยแนวทาง**:
1. **ฝั่ง COBOL**: เขียนโปรแกรมใหม่ (เช่น `DISCOUNTCALC.cob`) หรือแก้ `ORDERCALC` ให้รับ input
   เพิ่มอีก 1 บรรทัดทาง stdin (`discount-percent`) แล้วปรับสูตรคำนวณเป็น
   `subtotal = price * qty`, `discounted = subtotal - (subtotal * discount-percent / 100)`,
   `tax = discounted * 0.07`, `total = discounted + tax` ก่อนส่งออกผลลัพธ์ทาง stdout ในรูปแบบ
   คั่นด้วย `|` เหมือนเดิม
2. **ฝั่ง Wrapper Service**: เพิ่ม routing สำหรับ path ใหม่ `/api/discount` ที่:
   - อ่านและตรวจสอบ (validate) ค่า `price`, `quantity`, `discount-percent` ทั้งสามตัว (รวมถึง
     ตรวจสอบว่า `discount-percent` อยู่ในช่วง 0-100 อย่างสมเหตุสมผล)
   - แปลงเป็นข้อความ 3 บรรทัดส่งเข้า stdin ของโปรแกรม COBOL ตัวใหม่
   - แปลงผลลัพธ์กลับเป็น JSON response ตามรูปแบบเดียวกับ `/api/order`

การออกแบบเช่นนี้แสดงให้เห็นว่า Pattern ที่เรียนไปสามารถขยาย (extend) เพิ่ม endpoint ใหม่ได้โดยไม่
กระทบโค้ดเดิมของ endpoint อื่น ตราบใดที่ยึดหลักการแบ่งความรับผิดชอบ (Separation of Concerns) ที่
วางไว้ตั้งแต่ขั้นตอนที่ 721

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้และทดสอบจริงแบบ end-to-end ว่า:

- ทำไม COBOL ไม่สามารถพูดภาษา HTTP ได้โดยตรง และแนวคิด **Wrapper Service Pattern** ที่แก้ปัญหานี้
  ในโลกจริงมานานหลายสิบปี
- ออกแบบและเขียนโปรแกรม COBOL business logic (`ORDERCALC`) ที่รับ input ทาง stdin และส่ง output
  ทาง stdout โดยไม่ต้องรู้จัก HTTP หรือ JSON เลย
- ทดสอบโปรแกรม COBOL แบบ standalone ด้วย `printf | ./program` และค้นพบพฤติกรรม "Silent Failure"
  จริงของ `FUNCTION NUMVAL` เมื่อได้รับ input ที่ไม่ใช่ตัวเลข
- ออกแบบการแมปข้อมูลระหว่าง JSON กับรูปแบบข้อความที่ COBOL เข้าใจได้ อย่างมีหลักการ
- สร้าง HTTP Wrapper Service ด้วย Python `http.server` (Standard Library ล้วน ไม่ต้อง
  `pip install`) ที่เชื่อมต่อกับ COBOL ผ่าน `subprocess.run`
- รันระบบทั้งหมดจริงและทดสอบด้วย `curl` ได้ผลลัพธ์ตรงตามที่คาดหวังทุกกรณี
- **พบและแก้บั๊กจริง**: unhandled exception จาก `float("abc")` ที่ทำให้ client ได้รับ "http 000"
  แทนที่จะได้รับ error message ที่ชัดเจน พร้อมโค้ดฉบับแก้ไขที่ตรวจสอบ input ก่อนเรียก COBOL เสมอ
- ข้อจำกัดด้าน concurrency ของ `HTTPServer` และแนวทางแก้ไข (`ThreadingHTTPServer`, Process Pool,
  Message Queue) สำหรับระบบที่ต้องรองรับ traffic สูงขึ้น
- ภาพรวมว่าเมื่อไหร่ควรและไม่ควรใช้ Wrapper Service Pattern พร้อมข้อพิจารณาด้านความปลอดภัยเพิ่มเติม

Pattern ที่เรียนใน Part นี้คือรากฐานสำคัญของการทำ Modernization ในโลกจริง ใน **Part 074** เราจะ
ต่อยอด Pattern นี้ให้ซับซ้อนขึ้น: แทนที่จะคำนวณจากค่าที่ส่งเข้ามาอย่างเดียว เราจะเชื่อมต่อกับ
**Indexed File** (ทบทวนจาก Part 028) เพื่อสร้างระบบค้นหาข้อมูลลูกค้าผ่านเว็บแบบสมบูรณ์ พร้อม
เปรียบเทียบการสร้าง Web Layer ด้วยทั้ง Python และ Node.js

**[← กลับไป Part 072: การเชื่อมต่อ COBOL กับภาษาอื่น (C, Java) ผ่าน CALL](part-072-call-c-java.md)** | **[ไปยัง Part 074: COBOL และ Web Application: เชื่อมกับ Node.js/Python →](part-074-cobol-web-integration.md)**
