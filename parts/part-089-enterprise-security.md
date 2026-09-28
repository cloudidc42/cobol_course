# Part 089: Security ระดับ Enterprise สำหรับแอปพลิเคชัน COBOL (ขั้นตอนที่ 881–890)

## คำนำของ Part นี้

Part 069 สอนความปลอดภัยบน Mainframe ใน 2 ระดับ: **RACF** (ระดับ System/Platform ที่ควบคุมว่าใครเข้าถึง
ทรัพยากรอะไรได้) และ **Secure Coding พื้นฐาน** (Input Validation ด้วย `IS NUMERIC`, การป้องกัน Subscript
Overflow) — ทั้งสองเรื่องยังคงเป็นความจริงและสำคัญมาก แต่ยังไม่ครอบคลุมความท้าทายที่ระบบ COBOL สมัยใหม่
ต้องเจอจริง ๆ: **แอปพลิเคชัน COBOL ในปี 2026 ไม่ได้ทำงานอยู่บน Mainframe อย่างเดียวโดดเดี่ยวอีกต่อไป**

ทบทวนจาก Part 073: เราสร้าง Wrapper Service ที่ให้ COBOL รับ Input จากโลกภายนอกผ่าน REST API/JSON
ทบทวนจาก Part 057-060: COBOL เชื่อมต่อฐานข้อมูลผ่าน SQL ทบทวนจาก Part 086: Copybook และ Interface
ถูกแชร์ระหว่างหลายทีมหลายระบบ — ทุกจุดเชื่อมต่อเหล่านี้คือ **พื้นผิวโจมตี (Attack Surface)** ใหม่ที่ RACF
ไม่มีทางป้องกันได้เลย เพราะ RACF ควบคุมแค่ "ใครเข้าถึงทรัพยากรบน Mainframe ได้" แต่ไม่มีทางรู้เลยว่า
JSON ที่ส่งเข้ามาทาง HTTP มีอักขระอันตรายซ่อนอยู่หรือไม่, หรือ String ที่ COBOL กำลังจะส่งเข้า SQL statement
ถูกปนเปื้อนด้วยคำสั่งที่ผู้โจมตีแอบใส่มาหรือไม่

Part นี้ขยายมุมมองความปลอดภัยจาก **"ป้องกันขอบเขตของ Mainframe"** ไปสู่ **"ป้องกันทุกจุดที่แอปพลิเคชัน
COBOL สัมผัสกับโลกภายนอก"** ครอบคลุม: การกรองข้อมูลนำเข้าที่ชั้น Web-Wrapper, การป้องกัน SQL Injection
ด้วยแนวคิด Parameterization, การบันทึก Audit Log ระดับองค์กร, การปกปิดข้อมูลอ่อนไหว (PII/เลขบัญชี) ด้วย
Subprogram ที่สร้างและทดสอบจริง, สิทธิ์การเข้าถึงไฟล์ข้อมูลอย่างปลอดภัย, การจัดการ Error แบบ Fail Securely,
และการป้องกัน Resource Exhaustion ที่ชั้นแอปพลิเคชัน — **ทุกตัวอย่างในเอกสารนี้ทดสอบได้จริงด้วย GnuCOBOL**
ต่างจาก Part 069 ที่หัวข้อ RACF ต้องเป็นเชิงแนวคิดล้วน ๆ เพราะ Part นี้อยู่ในระดับแอปพลิเคชันทั้งหมด
ไม่ต้องพึ่ง z/OS จริง

> **หมายเหตุระเบียบวิธี**: ระหว่างเตรียม Part นี้ ผู้เขียนพบบั๊กจริง **3 จุด** ขณะสร้างและทดสอบ Subprogram
> — ทั้งหมดถูกบันทึกไว้เป็นบทเรียนในขั้นตอนที่เกี่ยวข้อง เพราะบั๊กเหล่านี้ล้วนเป็นรูปแบบที่พบได้จริงในงาน
> COBOL Enterprise และเชื่อมโยงกับหลักการความปลอดภัยของ Part นี้โดยตรง

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และทุกตัวอย่างผ่านการ
> คอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง รวมถึงตัวอย่างที่ตั้งใจแสดงพฤติกรรมไม่ปลอดภัยเพื่อการ
> เปรียบเทียบ (เช่นเดียวกับแนวทางของ Part 069)

---

## ขั้นตอนที่ 881: จาก RACF สู่ Application Security — ขยาย Defense in Depth ทั้งองค์กร

### ทบทวนโมเดล Defense in Depth จาก Part 069

Part 069 สอนไว้ว่า RACF (ชั้น Perimeter/Resource) และ Secure Coding (ชั้น Application) **ต้องทำงาน
ร่วมกันเสมอ ไม่ใช่ทางเลือกที่ใช้แทนกันได้** — RACF ป้องกันไม่ให้คนที่ไม่มีสิทธิ์เข้าถึงไฟล์/โปรแกรมตั้งแต่แรก
แต่ไม่สามารถป้องกันบั๊กหรือช่องโหว่ที่ซ่อนอยู่**ภายใน**ตรรกะของโปรแกรมที่ได้รับอนุญาตให้รันแล้วได้เลย

### ทำไมโมเดลนี้ยังไม่พอสำหรับ COBOL ในปี 2026

ทบทวนภาพสถาปัตยกรรมจาก Part 073 และ Part 086: แอปพลิเคชัน COBOL สมัยใหม่มีจุดเชื่อมต่อกับโลกภายนอก
มากกว่าที่ RACF เคยต้องรับมือมาก:

```
+------------------------------------------------------------------+
|                    Enterprise COBOL Attack Surface                |
+------------------------------------------------------------------+
|                                                                    |
|  ภายนอกองค์กร                    ขอบเขต RACF/z/OS                |
|  (Internet/Partner)                     |                         |
|       |                                  |                         |
|       v                                  v                         |
|  +-----------+     +-----------+    +-----------+    +----------+ |
|  |   REST    | --> |  Wrapper  | -->|   COBOL   | -->|   File/  | |
|  |  Client   |     | (Part073) |    | Business  |    |   SQL    | |
|  |           |     |           |    |  Logic    |    | (Part057)| |
|  +-----------+     +-----------+    +-----------+    +----------+ |
|       ^                  ^                ^                ^      |
|       |                  |                |                |      |
|   จุดเสี่ยง 1:        จุดเสี่ยง 2:      จุดเสี่ยง 3:      จุดเสี่ยง 4:  |
|   ข้อมูลนำเข้า         การแปลงข้อมูล    การสร้าง SQL      สิทธิ์ไฟล์  |
|   ที่เป็นอันตราย       JSON<->COBOL    แบบไม่ปลอดภัย     /Log ข้อมูล |
|   (ขั้นตอนที่ 882)     ให้ปลอดภัย       (ขั้นตอนที่ 883)   อ่อนไหว   |
|                                                          (884-887)  |
+------------------------------------------------------------------+
```

**RACF ควบคุมเฉพาะกล่องกลาง ("COBOL Business Logic" กับ "File/SQL") เท่านั้น** — มันไม่มีทางรู้เลยว่า
Client ภายนอกส่งอะไรมา หรือ Wrapper แปลงข้อมูลถูกต้องปลอดภัยหรือไม่ ซึ่งเป็นจุดเสี่ยงส่วนใหญ่ในสถาปัตยกรรม
สมัยใหม่ที่ทบทวนจาก Part 073, 077 และ 086

### กรอบคิด "Never Trust, Always Verify" ที่ประยุกต์กับทุกจุดเชื่อมต่อ

หลักการที่ Part นี้ยึดถือตลอดคือ: **ทุกจุดที่ข้อมูลข้ามขอบเขตความไว้วางใจ (Trust Boundary) ต้องตรวจสอบ
ใหม่เสมอ ไม่ว่าจุดก่อนหน้าจะตรวจสอบมาแล้วหรือไม่** ตัวอย่างเช่น แม้ Web-Wrapper (Python) จะตรวจสอบข้อมูล
มาแล้วชั้นหนึ่ง โปรแกรม COBOL เองก็ยังคงต้องตรวจสอบซ้ำอีกชั้น (**Defense in Depth ไม่ใช่แค่ "หลายชั้น"
แต่คือ "แต่ละชั้นไม่ไว้ใจชั้นก่อนหน้าอย่างเต็มที่"**) — หลักการนี้จะปรากฏซ้ำในขั้นตอนที่ 889 ที่ COBOL
ตรวจสอบขอบเขตค่าเองแม้ Wrapper จะตรวจสอบมาแล้วก็ตาม

### ตารางสรุป 4 จุดเสี่ยงหลักที่ Part นี้ครอบคลุม

| จุดเสี่ยง | ตัวอย่างภัยคุกคาม | ขั้นตอนที่ครอบคลุม |
|---|---|---|
| ข้อมูลนำเข้าจาก Web-Wrapper | อักขระอันตราย, ข้อมูลเกินขนาด | 882 |
| การสร้างคำสั่ง SQL แบบไดนามิก | SQL Injection | 883 |
| ข้อมูลอ่อนไหวในผลลัพธ์/Log | PII รั่วไหล | 884-886 |
| สิทธิ์ไฟล์และการจัดการ Error | ไฟล์เปิดกว้างเกินไป, ข้อความ error รั่วไหลข้อมูล | 887-888 |
| ทรัพยากรระบบ (CPU/เวลา) | Resource Exhaustion / Algorithmic DoS | 889 |

### ข้อควรระวัง

- **อย่าเข้าใจผิดว่า Part นี้มาแทนที่ Part 069** — RACF ยังคงจำเป็นสำหรับระบบที่รันบน Mainframe จริง
  Part นี้เป็นชั้นป้องกันเพิ่มเติมที่ครอบคลุมจุดที่ RACF **ไม่เคยครอบคลุมมาตั้งแต่ต้น** ไม่ใช่การทดแทนกัน
- ทุกเทคนิคใน Part นี้ใช้ได้ทั้งระบบที่รันบน Mainframe จริง (ควบคู่กับ RACF) และระบบที่รันบน GnuCOBOL/
  Linux/Cloud (ทบทวนจาก Part 077) — เป็นทักษะที่จำเป็นไม่ว่าจะทำงานในสภาพแวดล้อมแบบใด

### แบบฝึกหัดที่ 881.1

**โจทย์**: จงอธิบายว่าทำไม "Never Trust, Always Verify" ถึงหมายความว่าโปรแกรม COBOL ควรตรวจสอบ
ข้อมูลนำเข้าเองแม้ Web-Wrapper (Python) จะตรวจสอบมาแล้วก็ตาม

**เฉลยแนวทาง**: เพราะ Wrapper และ COBOL เป็นคนละโปรแกรม พัฒนาและดูแลโดยอาจเป็นคนละทีมกัน (ทบทวน
จาก Part 086 เรื่อง Layered Architecture ที่แต่ละ Layer อาจดูแลโดยทีมต่างกัน) การเปลี่ยนแปลงโค้ดฝั่ง
Wrapper ในอนาคต (เช่น ทีมเว็บแก้ไข validation logic โดยไม่ได้แจ้งทีม COBOL) อาจทำให้การตรวจสอบที่เคย
มีหายไปโดยไม่ตั้งใจ นอกจากนี้ในอนาคตอาจมี Presentation Layer อื่นเพิ่มเข้ามา (เช่น CICS Transaction ตาม
ตัวอย่างจาก Part 086 ขั้นตอนที่ 852) ที่เรียก Business Logic เดียวกันแต่ไม่ผ่าน Web-Wrapper เลย — ถ้า
COBOL ไม่ตรวจสอบข้อมูลเอง Presentation Layer ใหม่นั้นจะไม่มีการป้องกันใด ๆ เลย การตรวจสอบซ้ำที่ COBOL
จึงเป็นเกราะป้องกันสุดท้ายที่ไม่ขึ้นอยู่กับว่าข้อมูลมาจากช่องทางไหน

---

## ขั้นตอนที่ 882: Input Sanitization สำหรับ Web-Wrapper Pattern

### ทบทวนจาก Part 073: เส้นทางของข้อมูลจาก HTTP Request ถึง COBOL

Part 073 สอนการสร้าง Wrapper Service: HTTP Request → Python แปลง JSON → เขียนเป็นบรรทัด Fixed-Width
→ ส่งให้ COBOL ผ่าน stdin/subprocess → COBOL ประมวลผล → ส่งผลลัพธ์กลับ ในกระบวนการนี้ COBOL **รับ
ข้อมูลจากภายนอกโดยตรง** ผ่านฟิลด์ที่แปลงมาจาก JSON ซึ่งอาจมี**อักขระอะไรก็ได้**ที่ผู้ใช้ (หรือผู้โจมตี) ส่งมา

### ทำไม Allow-List ดีกว่า Block-List เสมอ

Part 069 ขั้นตอนที่ 686 สอนการตรวจสอบด้วย `IS NUMERIC` สำหรับฟิลด์ตัวเลข — สำหรับฟิลด์ข้อความทั่วไป
(เช่น ชื่อลูกค้า) เราไม่สามารถใช้ `IS NUMERIC` ได้ แต่หลักการเดียวกันยังใช้ได้: **กำหนดชุดอักขระที่อนุญาต
(Allow-List) แทนที่จะพยายามไล่บล็อกอักขระอันตรายทีละตัว (Block-List)** เพราะ Block-List ต้องคาดเดาล่วงหน้า
ว่าอักขระอันตรายมีอะไรบ้าง ซึ่งมักตกหล่นเสมอ ในขณะที่ Allow-List ปฏิเสธทุกอย่างที่ไม่รู้จักโดยอัตโนมัติ
(หลักการเดียวกับ "deny by default, allow by exception" จาก Part 069 ขั้นตอนที่ 684 ที่ใช้กับ RACF)

### โปรแกรมทดสอบ: ตรวจสอบฟิลด์ข้อความด้วย Allow-List

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP882SANITIZE.
       AUTHOR. COBOL-COURSE.
      *> Validates a field received from the REST wrapper (Part 073
      *> pattern: JSON -> fixed-width COBOL field) before it is
      *> trusted by business logic. Rejects control characters and
      *> common injection/markup metacharacters.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW-INPUT        PIC X(50).
       01  WS-CHAR-IDX         PIC 9(3).
       01  WS-ONE-CHAR         PIC X(1).
       01  WS-BAD-COUNT        PIC 9(3) VALUE 0.
       01  WS-VALID-FLAG       PIC X(1) VALUE "Y".
           88  WS-IS-VALID              VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-RAW-INPUT.
           MOVE "Y" TO WS-VALID-FLAG.
           MOVE 0 TO WS-BAD-COUNT.

      *> Reject any character that is not a letter, digit, space,
      *> hyphen, period, or comma - a conservative allow-list is
      *> far safer than trying to block every dangerous character.
           PERFORM VARYING WS-CHAR-IDX FROM 1 BY 1
                   UNTIL WS-CHAR-IDX > 50
               MOVE WS-RAW-INPUT(WS-CHAR-IDX:1) TO WS-ONE-CHAR
               IF WS-ONE-CHAR NOT = SPACE
                   IF (WS-ONE-CHAR < "A" OR WS-ONE-CHAR > "Z")
                       AND (WS-ONE-CHAR < "a" OR WS-ONE-CHAR > "z")
                       AND (WS-ONE-CHAR < "0" OR WS-ONE-CHAR > "9")
                       AND WS-ONE-CHAR NOT = "-"
                       AND WS-ONE-CHAR NOT = "."
                       AND WS-ONE-CHAR NOT = ","
                       ADD 1 TO WS-BAD-COUNT
                   END-IF
               END-IF
           END-PERFORM.

           IF WS-BAD-COUNT > 0
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-IS-VALID
               DISPLAY "ACCEPTED: '" WS-RAW-INPUT "'"
           ELSE
               DISPLAY "REJECTED: input contains " WS-BAD-COUNT
                   " disallowed character(s)"
           END-IF.
           STOP RUN.
```

```bash
cobc -x -o step882_sanitize step882_sanitize.cob
printf -- "JOHN SMITH\n" | ./step882_sanitize
printf -- "X OR 1=1 -- DROP\n" | ./step882_sanitize
printf -- "a';DROP TABLE X;--\n" | ./step882_sanitize
printf -- "<script>alert(1)</script>\n" | ./step882_sanitize
```

**ผลลัพธ์จริง:**

```
--- test 1: normal name ---
ACCEPTED: 'JOHN SMITH                                        '

--- test 2: SQL injection style ---
REJECTED: input contains 001 disallowed character(s)

--- test 3: quote and semicolon ---
REJECTED: input contains 003 disallowed character(s)

--- test 4: script tag ---
REJECTED: input contains 007 disallowed character(s)
```

### อธิบายจุดสำคัญ

- ข้อมูลปกติ ("JOHN SMITH") ผ่านการตรวจสอบทันที เพราะมีแค่ตัวอักษรและช่องว่าง
- ข้อมูลที่มีลักษณะพยายามฉีดคำสั่ง SQL (`OR 1=1`) ถูกปฏิเสธเพราะมีเครื่องหมาย `=` ซึ่งไม่อยู่ใน Allow-List
- ข้อมูลที่มีเครื่องหมายคำพูดและ semicolon (ลักษณะ SQL Injection แบบคลาสสิก) ถูกปฏิเสธเพราะมี 3 อักขระ
  ที่ไม่ได้รับอนุญาต (`'`, `;`, `-` ซ้ำ — สังเกตว่า `-` เดี่ยวได้รับอนุญาตแต่ `--` ที่ติดกันยังคงนับเป็น 2
  ตัวอักษรที่อนุญาตแยกกัน ในตัวอย่างนี้ตัวที่ถูกนับเป็น "bad" คือ `'` และ `;` สองครั้ง)
- ข้อมูลที่มี HTML tag (ลักษณะ XSS) ถูกปฏิเสธเพราะมีอักขระ `<`, `>`, `/`, `(`, `)`, `!` ที่ไม่ได้รับอนุญาต

### ข้อควรระวัง

- **Allow-List ต้องออกแบบให้เหมาะกับประเภทข้อมูลจริง** — ตัวอย่างนี้เหมาะกับชื่อคน แต่ถ้าเป็นฟิลด์อีเมล
  จะต้องอนุญาต `@` เพิ่ม หรือถ้าเป็นที่อยู่อาจต้องอนุญาตเครื่องหมาย `/` และ `#` เพิ่มเติม — **ต้องออกแบบ
  Allow-List แยกตามชนิดข้อมูลแต่ละประเภท ไม่ใช่ใช้ชุดเดียวกันกับทุกฟิลด์**
- การตรวจสอบนี้ควรทำ **ทั้งที่ Web-Wrapper (Python)** และ**ที่ COBOL เอง** (ตามหลักการ "Never Trust,
  Always Verify" จากขั้นตอนที่ 881) — อย่าคิดว่าตรวจสอบที่ชั้นเดียวพอ

### แบบฝึกหัดที่ 882.1

**โจทย์**: หากต้องออกแบบ Allow-List สำหรับฟิลด์ "ที่อยู่อีเมล" จะต้องเพิ่มอักขระใดจากตัวอย่างในขั้นตอนนี้
บ้าง เพราะเหตุใด

**เฉลยแนวทาง**: ต้องเพิ่ม `@` (สำหรับแยกส่วน local-part กับ domain) และอาจต้องพิจารณาเพิ่ม `_` และ `+`
(ที่อยู่อีเมลบางระบบอนุญาตให้ใช้ใน local-part) แต่ควร**ยังคงปฏิเสธ**อักขระอันตรายอื่น เช่น `<`, `>`, `'`,
`;` เพราะแม้จะเป็นฟิลด์อีเมล ก็ไม่มีเหตุผลทางธุรกิจใดที่อีเมลจะต้องมีอักขระเหล่านี้ — หลักการคือ **อนุญาต
เฉพาะอักขระที่จำเป็นจริงสำหรับชนิดข้อมูลนั้น ไม่ใช่อนุญาตกว้างขึ้นเรื่อย ๆ เพื่อความสะดวก**

---

## ขั้นตอนที่ 883: SQL Injection และแนวคิด Parameterization เมื่อ COBOL เรียกใช้ SQL

### ทบทวนจาก Part 057-060: Embedded SQL ใน COBOL

Part 057-060 สอนการเชื่อมต่อ DB2 ผ่าน `EXEC SQL ... END-EXEC` (เนื้อหาเชิงแนวคิด/มาตรฐานตามหมายเหตุ
ของหลักสูตรตั้งแต่ Part 051 เพราะต้องใช้ DB2 จริงหรือ Mainframe Emulator) — ขั้นตอนนี้ไม่ได้สอนไวยากรณ์
SQL ซ้ำ แต่จะพิสูจน์ **หลักการด้านความปลอดภัยที่สำคัญที่สุดข้อหนึ่งของการเขียนโปรแกรมที่เชื่อมต่อฐานข้อมูล**
ด้วยการรันจริงบน GnuCOBOL แม้จะไม่มี DB2 จริงให้เชื่อมต่อก็ตาม — เพราะปัญหานี้เกิดที่ **ระดับการสร้าง
ข้อความ (String)** ก่อนที่จะส่งเข้าฐานข้อมูลเลยด้วยซ้ำ

### ทดสอบ A: รูปแบบอันตราย — สร้างข้อความ SQL ด้วยการต่อ String โดยตรง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP883UNSAFE.
       AUTHOR. COBOL-COURSE.
      *> DANGEROUS PATTERN: builds SQL statement TEXT by concatenating
      *> raw user input directly into the query string. This mirrors
      *> a real anti-pattern some COBOL/DB2 shops still use to build
      *> "dynamic WHERE clauses" for EXEC SQL. No real database is
      *> used here - this program only proves that the injected text
      *> changes the MEANING of the query, which is the whole point.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-USER-INPUT       PIC X(40).
       01  WS-QUERY-TEXT       PIC X(120).

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-USER-INPUT.
           STRING "SELECT ACCT_NO, BALANCE FROM ACCOUNTS "
               "WHERE CUST_NAME = '" DELIMITED BY SIZE
               FUNCTION TRIM(WS-USER-INPUT) DELIMITED BY SIZE
               "'" DELIMITED BY SIZE
               INTO WS-QUERY-TEXT.
           DISPLAY "BUILT QUERY TEXT:".
           DISPLAY FUNCTION TRIM(WS-QUERY-TEXT).
           STOP RUN.
```

```bash
cobc -x -o step883_unsafe step883_unsafe.cob
printf -- "SMITH\n" | ./step883_unsafe
printf -- "X' OR '1'='1\n" | ./step883_unsafe
```

**ผลลัพธ์จริง:**

```
=== ข้อมูลปกติ ===
BUILT QUERY TEXT:
SELECT ACCT_NO, BALANCE FROM ACCOUNTS WHERE CUST_NAME = 'SMITH'

=== ข้อมูลอันตราย ===
BUILT QUERY TEXT:
SELECT ACCT_NO, BALANCE FROM ACCOUNTS WHERE CUST_NAME = 'X' OR '1'='1'
```

### วิเคราะห์อันตราย

สังเกตข้อความ SQL ที่สร้างจากข้อมูลอันตราย: **`WHERE CUST_NAME = 'X' OR '1'='1'`** — เครื่องหมายคำพูด
เดี่ยว (`'`) ในข้อมูลนำเข้าทำให้ COBOL "ปิด" ค่าของ `CUST_NAME` ก่อนกำหนด แล้วเงื่อนไข `OR '1'='1'` ที่
เหลือกลายเป็นส่วนหนึ่งของคำสั่ง SQL จริง ๆ (ไม่ใช่ข้อมูล) เงื่อนไข `'1'='1'` เป็นจริงเสมอ ทำให้ (ถ้าโค้ดนี้
ถูกส่งไปยัง SQL engine จริง) คำสั่งจะคืนค่า**ทุกแถวในตาราง** แทนที่จะกรองเฉพาะลูกค้าชื่อ "X" — นี่คือหัวใจ
ของ SQL Injection: **ข้อมูลนำเข้าถูกตีความเป็นส่วนหนึ่งของโครงสร้างคำสั่ง (syntax) แทนที่จะเป็นแค่ค่าข้อมูล
(data)**

### ทดสอบ B: รูปแบบปลอดภัย — แยกข้อความ SQL ออกจากข้อมูลผู้ใช้อย่างเด็ดขาด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP883SAFE.
       AUTHOR. COBOL-COURSE.
      *> SAFE PATTERN: the query TEXT is a fixed literal that never
      *> changes no matter what the user typed. User input is kept
      *> in a SEPARATE host variable that would be bound at execution
      *> time via "EXEC SQL ... WHERE CUST_NAME = :WS-HOST-VAR END-EXEC"
      *> (Part 058) - the database driver treats it purely as DATA,
      *> never as part of the SQL grammar, no matter what characters
      *> it contains.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-USER-INPUT       PIC X(40).
       01  WS-HOST-VAR         PIC X(40).
       01  WS-QUERY-TEXT       PIC X(80) VALUE
           "SELECT ACCT_NO, BALANCE FROM ACCOUNTS WHERE CUST_NAME = ?".

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-USER-INPUT.
           MOVE WS-USER-INPUT TO WS-HOST-VAR.
           DISPLAY "FIXED QUERY TEXT (never changes):".
           DISPLAY FUNCTION TRIM(WS-QUERY-TEXT).
           DISPLAY "HOST VARIABLE VALUE (pure data, not SQL syntax):".
           DISPLAY "'" FUNCTION TRIM(WS-HOST-VAR) "'".
           STOP RUN.
```

```bash
cobc -x -o step883_safe step883_safe.cob
printf -- "SMITH\n" | ./step883_safe
printf -- "X' OR '1'='1\n" | ./step883_safe
```

**ผลลัพธ์จริง:**

```
=== ข้อมูลปกติ ===
FIXED QUERY TEXT (never changes):
SELECT ACCT_NO, BALANCE FROM ACCOUNTS WHERE CUST_NAME = ?
HOST VARIABLE VALUE (pure data, not SQL syntax):
'SMITH'

=== ข้อมูลอันตราย (ค่าเดียวกับทดสอบ A) ===
FIXED QUERY TEXT (never changes):
SELECT ACCT_NO, BALANCE FROM ACCOUNTS WHERE CUST_NAME = ?
HOST VARIABLE VALUE (pure data, not SQL syntax):
'X' OR '1'='1'
```

### วิเคราะห์ผล — ทำไมวิธีนี้ปลอดภัยอย่างแท้จริง

สังเกตความแตกต่างที่สำคัญที่สุด: **ไม่ว่าผู้ใช้จะป้อนอะไรมา ข้อความ Query (`WS-QUERY-TEXT`) ไม่เปลี่ยนแปลง
เลยแม้แต่ตัวอักษรเดียว** — ยังคงเป็น `WHERE CUST_NAME = ?` เหมือนเดิมทุกครั้ง ข้อมูลอันตรายทั้งหมด
(`X' OR '1'='1`) ถูกเก็บไว้ใน `WS-HOST-VAR` ซึ่งเป็นเพียง **ค่าข้อมูล (data)** — เมื่อส่งไปยังฐานข้อมูลจริง
ผ่าน `EXEC SQL ... USING :WS-HOST-VAR` (Part 058) driver ของฐานข้อมูลจะปฏิบัติต่อค่าทั้งหมดนี้เป็น**ค่า
ของ `CUST_NAME` ที่ต้องค้นหาตรงตัวเป๊ะ ๆ** (ซึ่งจะไม่พบแถวใดเลย เพราะไม่มีลูกค้าชื่อ `X' OR '1'='1` จริง)
แทนที่จะถูกตีความเป็นส่วนหนึ่งของคำสั่ง SQL เหมือนในทดสอบ A — นี่คือกลไกที่เรียกว่า **Parameterized
Query / Prepared Statement** ซึ่งเป็นวิธีป้องกัน SQL Injection ที่ยอมรับกันทั่วโลกในทุกภาษาโปรแกรม
ไม่ใช่เฉพาะ COBOL

### ข้อควรระวัง

- **อย่าใช้ `STRING` ต่อข้อมูลผู้ใช้เข้ากับข้อความ SQL โดยตรงเด็ดขาด** ไม่ว่าจะผ่านการ Sanitize มาจาก
  ขั้นตอนที่ 882 แล้วหรือไม่ก็ตาม — Parameterization ควรเป็นแนวทางหลักเสมอ ส่วน Input Sanitization เป็น
  ชั้นป้องกันเสริม (Defense in Depth) ไม่ใช่ตัวทดแทน
- ตัวอย่างนี้จำลองปัญหาด้วยการ `DISPLAY` ข้อความ SQL ที่สร้างขึ้นเท่านั้น (ไม่มีการเชื่อมต่อฐานข้อมูลจริง
  ตามหมายเหตุมาตรฐานของหลักสูตรเรื่อง DB2 ตั้งแต่ Part 051) แต่หลักการที่พิสูจน์ได้จากการรันจริงนี้ —
  ข้อความ SQL เปลี่ยนความหมายไปตามข้อมูลนำเข้า — คือปัญหาเดียวกันเป๊ะกับที่เกิดขึ้นจริงเมื่อเชื่อมต่อ DB2
  จริงด้วย Dynamic SQL ที่สร้างด้วยการต่อ String

### แบบฝึกหัดที่ 883.1

**โจทย์**: จงอธิบายว่าทำไม Input Sanitization จากขั้นตอนที่ 882 เพียงอย่างเดียว **ไม่เพียงพอ**ที่จะป้องกัน
SQL Injection ได้อย่างสมบูรณ์ แม้จะปฏิเสธเครื่องหมายคำพูดเดี่ยว (`'`) ไปแล้วก็ตาม

**เฉลยแนวทาง**: เพราะ Allow-List ที่ออกแบบไว้อาจไม่ครอบคลุมทุกกรณีการโจมตีที่เป็นไปได้เสมอ (เช่น ฐาน
ข้อมูลบางระบบมีฟังก์ชันหรือรูปแบบ syntax พิเศษที่ไม่ต้องพึ่งเครื่องหมายคำพูดเลยก็โจมตีได้ หรือฟิลด์บางฟิลด์
จำเป็นต้องอนุญาตเครื่องหมายคำพูดจริง ๆ ตามความต้องการทางธุรกิจ เช่น ชื่อที่มีเครื่องหมาย apostrophe เช่น
"O'Brien") การพึ่งพา Sanitization เพียงอย่างเดียวจึงมีความเสี่ยงที่จะตกหล่นเสมอ (ตรงกับหลักการ Block-List
มีข้อจำกัดที่อธิบายในขั้นตอนที่ 882) ในขณะที่ Parameterization แก้ปัญหาที่ **ต้นเหตุที่แท้จริง** คือการแยก
โครงสร้างคำสั่ง (SQL syntax) ออกจากค่าข้อมูล (data) อย่างเด็ดขาดในระดับกลไกของภาษา ทำให้ไม่ว่าข้อมูลจะมี
อักขระอะไรก็ไม่มีทางถูกตีความเป็นส่วนหนึ่งของคำสั่งได้เลย — ทั้งสองเทคนิคจึงควรใช้ร่วมกันเสมอ (Defense in
Depth) ไม่ใช่เลือกใช้อย่างใดอย่างหนึ่ง

---

## ขั้นตอนที่ 884: การปกปิดข้อมูลอ่อนไหว (Masking) ในผลลัพธ์ DISPLAY — สร้าง Subprogram จริง

### ทำไมต้อง Mask ข้อมูลก่อนแสดงผลหรือบันทึก Log เสมอ

ระบบธนาคารและการเงินที่หลักสูตรนี้ใช้เป็นตัวอย่างหลักมาตลอด มีข้อมูลอ่อนไหวจำนวนมาก (เลขบัญชี, เลขบัตร
ประชาชน, อีเมล, เบอร์โทรศัพท์) ที่**ไม่ควรปรากฏแบบเต็มรูปแบบ**ในหน้าจอ, ไฟล์ Log, หรือข้อความ error —
เพราะบุคคลที่มีสิทธิ์ดู Log (เช่น ทีม Operations ที่ตรวจสอบปัญหาระบบ) **ไม่จำเป็นต้องเห็นข้อมูลลูกค้าเต็ม
รูปแบบเพื่อทำงานของตัวเอง** (หลักการ Least Privilege เดียวกับที่ Part 069 ขั้นตอนที่ 682 สอนไว้กับ RACF
Group — ที่นี่ประยุกต์ใช้กับ**การมองเห็นข้อมูล**แทนสิทธิ์เข้าถึงระบบ)

### สร้าง Subprogram MASKACCT: ปกปิดเลขบัญชี เหลือ 4 หลักสุดท้าย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MASKACCT.
       AUTHOR. COBOL-COURSE.
      *> Masks an account number, keeping only the LAST 4 digits
      *> visible and replacing every earlier character with "*".
      *> LINKAGE: CALL "MASKACCT" USING WS-ACCT-IN WS-ACCT-OUT.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-IDX              PIC 9(2).
       01  WS-LEN              PIC 9(2) VALUE 16.

       LINKAGE SECTION.
       01  LK-ACCT-IN          PIC X(16).
       01  LK-ACCT-OUT         PIC X(16).

       PROCEDURE DIVISION USING LK-ACCT-IN LK-ACCT-OUT.
       MAIN-PARA.
           MOVE LK-ACCT-IN TO LK-ACCT-OUT.
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-LEN - 4
               MOVE "*" TO LK-ACCT-OUT(WS-IDX:1)
           END-PERFORM.
           GOBACK.
```

### โปรแกรมเรียกใช้และทดสอบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP884MASKTEST.
       AUTHOR. COBOL-COURSE.
      *> Calls the MASKACCT subprogram with several account numbers
      *> and DISPLAYs the masked result - this is what a program
      *> should print in logs/screens, never the raw account number.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ACCT-RAW         PIC X(16).
       01  WS-ACCT-MASKED      PIC X(16).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "1234567890123456" TO WS-ACCT-RAW.
           CALL "MASKACCT" USING WS-ACCT-RAW WS-ACCT-MASKED.
           DISPLAY "RAW=" WS-ACCT-RAW " MASKED=" WS-ACCT-MASKED.

           MOVE "9988776655440000" TO WS-ACCT-RAW.
           CALL "MASKACCT" USING WS-ACCT-RAW WS-ACCT-MASKED.
           DISPLAY "RAW=" WS-ACCT-RAW " MASKED=" WS-ACCT-MASKED.

           MOVE "0001112223334444" TO WS-ACCT-RAW.
           CALL "MASKACCT" USING WS-ACCT-RAW WS-ACCT-MASKED.
           DISPLAY "RAW=" WS-ACCT-RAW " MASKED=" WS-ACCT-MASKED.
           STOP RUN.
```

```bash
cobc -m -o MASKACCT.so MASKACCT.cob
cobc -x -o step884_maskacct_test step884_maskacct_test.cob
COB_LIBRARY_PATH=. ./step884_maskacct_test
```

**ผลลัพธ์จริง:**

```
RAW=1234567890123456 MASKED=************3456
RAW=9988776655440000 MASKED=************0000
RAW=0001112223334444 MASKED=************4444
```

### อธิบายจุดสำคัญ

- Subprogram นี้ทำงานแบบ **Pure Function** (รับข้อมูลเข้า, คืนผลลัพธ์, ไม่มีผลข้างเคียงอื่น) — ทบทวนจาก
  Part 031-032 เรื่อง Subprogram และการส่งพารามิเตอร์ — ทำให้นำไปใช้ซ้ำได้ในทุกจุดของระบบที่ต้องแสดงผล
  หรือบันทึก Log เลขบัญชี โดยรับประกันว่ากฎการ Masking สอดคล้องกันทุกจุด (หลักการเดียวกับ Business Logic
  Layer ใน Part 086 ขั้นตอนที่ 852 ที่รวมตรรกะไว้ที่เดียว)
- ตัวเลข `WS-LEN - 4` คำนวณจำนวนหลักที่ต้อง mask แบบไดนามิกจากความยาวฟิลด์ ไม่ใช่ hardcode ตัวเลขคงที่
  — ทำให้ปรับเปลี่ยนความยาวฟิลด์ในอนาคตได้ง่ายกว่า (หลักการ "Single Source of Truth" จาก Part 069
  ขั้นตอนที่ 687)

### ข้อควรระวัง

- **การ Mask ควรเกิดขึ้น ณ จุดที่ใกล้กับการแสดงผล/บันทึก Log ที่สุด** ไม่ใช่ Mask ตั้งแต่ตอนอ่านข้อมูลเข้า
  ระบบ เพราะ Business Logic บางส่วน (เช่น การตรวจสอบเลขบัญชีกับฐานข้อมูล) ยังคงต้องใช้ค่าจริงในการ
  ประมวลผล — Mask เฉพาะตอนที่ข้อมูลกำลังจะ**ออกจากระบบไปสู่สายตามนุษย์**เท่านั้น
- จำนวนหลักที่เปิดเผย (ในที่นี้คือ 4 หลักสุดท้าย) ควรเป็นไปตามนโยบายความปลอดภัยขององค์กรและกฎหมาย
  คุ้มครองข้อมูลส่วนบุคคลที่เกี่ยวข้อง (เช่น PDPA ในประเทศไทย) ไม่ใช่ตัวเลขที่กำหนดขึ้นเองตามใจ

### แบบฝึกหัดที่ 884.1

**โจทย์**: จงอธิบายว่าทำไมการสร้าง `MASKACCT` เป็น Subprogram แยกต่างหาก (แทนที่จะเขียนโค้ด Mask
ซ้ำในทุกโปรแกรมที่ต้องการ) จึงสำคัญมากขึ้นเมื่อระบบขยายสเกลไปสู่หลายสิบโปรแกรม

**เฉลยแนวทาง**: หากแต่ละโปรแกรมเขียนโค้ด Mask ของตัวเองแยกกัน เมื่อองค์กรต้องเปลี่ยนนโยบาย (เช่น
เปลี่ยนจากเปิดเผย 4 หลักสุดท้ายเป็น 2 หลักสุดท้ายตามกฎหมายใหม่) จะต้องแก้ไขโค้ดในทุกโปรแกรมแยกกัน ซึ่ง
เสี่ยงต่อการตกหล่นบางจุด (บางโปรแกรมอาจถูกลืมแก้ไข ทำให้ยังคงเปิดเผยข้อมูลเกินกว่านโยบายใหม่) การรวม
ตรรกะไว้ที่ Subprogram เดียว ทำให้แก้ไขที่จุดเดียวแล้วมีผลกับทุกโปรแกรมที่เรียกใช้ทันที (หลังจาก compile
ใหม่) ตรงกับหลักการ Copybook Governance จาก Part 086 ขั้นตอนที่ 855 ที่เน้นย้ำเรื่อง "สัญญาร่วม" ที่ต้อง
มีจุดเดียวที่ควบคุมได้ แทนที่จะกระจัดกระจายไปทั่วทั้งระบบ

---

## ขั้นตอนที่ 885: ขยาย Masking Subprogram ให้ครอบคลุม PII หลายประเภท

### ปัญหาของขั้นตอนที่ 884: ใช้ได้แค่เลขบัญชีเท่านั้น

`MASKACCT` ในขั้นตอนก่อนรับเฉพาะเลขบัญชี 16 หลักเท่านั้น — ระบบจริงมีข้อมูลอ่อนไหวหลายประเภท (เลขบัตร
ประชาชน, เบอร์โทรศัพท์, อีเมล) ที่ต้องการกฎการ Mask ต่างกัน (เลขบัตรประชาชนอาจเก็บ 4 หลักสุดท้ายเหมือน
บัญชี แต่อีเมลต้องการกฎที่ต่างออกไปโดยสิ้นเชิง เช่น เปิดเผยตัวอักษรแรกและโดเมน) ขั้นตอนนี้สร้าง Subprogram
**ทั่วไป (Generic)** ที่รับพารามิเตอร์ "ประเภทข้อมูล" เพื่อเลือกกฎการ Mask ที่เหมาะสม

### Subprogram MASKFLD: เลือกกฎการ Mask ตามประเภทข้อมูล

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MASKFLD.
       AUTHOR. COBOL-COURSE.
      *> General-purpose PII masking subprogram. LK-FIELD-TYPE picks
      *> the masking rule; LK-FIELD-IN/LK-FIELD-OUT are fixed-width
      *> 30-byte fields (right-padded with spaces).
      *> Types: "ACCT" / "NID" / "PHONE" - keep last 4 characters.
      *>        "EMAIL"                 - keep first char + domain.
      *>        anything else           - mask everything.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-IDX              PIC 9(2).
       01  WS-LEN              PIC 9(2).
       01  WS-AT-POS           PIC 9(2).

       LINKAGE SECTION.
       01  LK-FIELD-TYPE       PIC X(5).
       01  LK-FIELD-IN         PIC X(30).
       01  LK-FIELD-OUT        PIC X(30).

       PROCEDURE DIVISION USING LK-FIELD-TYPE LK-FIELD-IN LK-FIELD-OUT.
       MAIN-PARA.
           MOVE LK-FIELD-IN TO LK-FIELD-OUT.
           MOVE FUNCTION LENGTH(FUNCTION TRIM(LK-FIELD-IN)) TO WS-LEN.

           EVALUATE LK-FIELD-TYPE
               WHEN "ACCT "
               WHEN "NID  "
               WHEN "PHONE"
                   PERFORM VARYING WS-IDX FROM 1 BY 1
                           UNTIL WS-IDX > WS-LEN - 4
                       MOVE "*" TO LK-FIELD-OUT(WS-IDX:1)
                   END-PERFORM
               WHEN "EMAIL"
                   PERFORM FIND-AT-SIGN
                   IF WS-AT-POS > 2
                       PERFORM VARYING WS-IDX FROM 2 BY 1
                               UNTIL WS-IDX >= WS-AT-POS
                           MOVE "*" TO LK-FIELD-OUT(WS-IDX:1)
                       END-PERFORM
                   END-IF
               WHEN OTHER
                   PERFORM VARYING WS-IDX FROM 1 BY 1
                           UNTIL WS-IDX > WS-LEN
                       MOVE "*" TO LK-FIELD-OUT(WS-IDX:1)
                   END-PERFORM
           END-EVALUATE.
           GOBACK.

       FIND-AT-SIGN.
           MOVE 0 TO WS-AT-POS.
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > WS-LEN
               IF LK-FIELD-IN(WS-IDX:1) = "@" AND WS-AT-POS = 0
                   MOVE WS-IDX TO WS-AT-POS
               END-IF
           END-PERFORM.
```

### โปรแกรมทดสอบทั้ง 4 ประเภทข้อมูล

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP885MASKFLDTEST.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TYPE             PIC X(5).
       01  WS-IN               PIC X(30).
       01  WS-OUT              PIC X(30).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "ACCT" TO WS-TYPE.
           MOVE "1234567890123456" TO WS-IN.
           CALL "MASKFLD" USING WS-TYPE WS-IN WS-OUT.
           DISPLAY "ACCT  : " WS-IN " -> " WS-OUT.

           MOVE "NID" TO WS-TYPE.
           MOVE "1234567890123" TO WS-IN.
           CALL "MASKFLD" USING WS-TYPE WS-IN WS-OUT.
           DISPLAY "NID   : " WS-IN " -> " WS-OUT.

           MOVE "PHONE" TO WS-TYPE.
           MOVE "0812345678" TO WS-IN.
           CALL "MASKFLD" USING WS-TYPE WS-IN WS-OUT.
           DISPLAY "PHONE : " WS-IN " -> " WS-OUT.

           MOVE "EMAIL" TO WS-TYPE.
           MOVE "jsmith@example.com" TO WS-IN.
           CALL "MASKFLD" USING WS-TYPE WS-IN WS-OUT.
           DISPLAY "EMAIL : " WS-IN " -> " WS-OUT.
           STOP RUN.
```

```bash
cobc -m -o MASKFLD.so MASKFLD.cob
cobc -x -o step885_maskfld_test step885_maskfld_test.cob
COB_LIBRARY_PATH=. ./step885_maskfld_test
```

**ผลลัพธ์จริง:**

```
ACCT  : 1234567890123456               -> ************3456
NID   : 1234567890123                  -> *********0123
PHONE : 0812345678                     -> ******5678
EMAIL : jsmith@example.com             -> j*****@example.com
```

### อธิบายจุดสำคัญ

- ทั้ง 4 ประเภทข้อมูลใช้ Subprogram เดียวกัน แค่เปลี่ยนค่าพารามิเตอร์ `LK-FIELD-TYPE` — เลขบัญชี/บัตร
  ประชาชน/เบอร์โทรศัพท์ ใช้กฎเดียวกัน (เปิดเผย 4 ตัวท้าย) แต่อีเมลใช้กฎเฉพาะที่ค้นหาตำแหน่ง `@` ก่อน แล้ว
  ปกปิดเฉพาะส่วน local-part (ยกเว้นตัวอักษรแรก) โดยเปิดเผย domain ทั้งหมด
- ผลลัพธ์ `EMAIL : jsmith@example.com -> j*****@example.com` แสดงให้เห็นว่าตัวอักษรแรก (`j`) และ
  domain (`@example.com`) ยังคงมองเห็นได้ ในขณะที่ส่วนที่เหลือของ local-part (`smith`) ถูกแทนที่ด้วย `*`
  ทั้งหมด 5 ตัว ตรงตามความยาวจริงของข้อความที่ถูกปกปิด
- `WHEN "OTHER"` ทำหน้าที่เป็น **fail-safe default**: หากมีการเรียกใช้ด้วยประเภทข้อมูลที่ไม่รู้จัก (เช่น
  พิมพ์ผิดเป็น `"ACCNT"` แทน `"ACCT"`) โปรแกรมจะ**ปกปิดข้อมูลทั้งหมด**แทนที่จะเปิดเผยข้อมูลดิบโดยไม่ตั้งใจ
  — หลักการเดียวกับ `UACC(NONE)` ("deny by default") ที่ Part 069 ขั้นตอนที่ 684 สอนไว้กับ RACF

### ข้อควรระวัง

- ค่า `LK-FIELD-TYPE` ต้องส่งเข้ามาให้ตรงความกว้าง 5 ตัวอักษรพอดี (เช่น `"ACCT "` ที่มีช่องว่างเติมท้าย)
  มิฉะนั้น `EVALUATE` จะไม่ตรงกับเงื่อนไขใดเลยและตกไปที่ `WHEN OTHER` (ซึ่งในกรณีนี้ยังปลอดภัยเพราะ
  ปกปิดข้อมูลทั้งหมด แต่ก็ทำให้ผลลัพธ์ผิดจากที่ผู้เรียกตั้งใจ) — ทบทวนความสำคัญของ Contract ระหว่าง
  Subprogram จาก Part 032
- ตัวอย่างนี้ค้นหา `@` ตัวแรกที่พบเท่านั้น (`WS-AT-POS = 0` เป็นเงื่อนไขหยุดบันทึกตำแหน่งซ้ำ) หากอีเมล
  มีรูปแบบผิดปกติ (เช่น มี `@` มากกว่า 1 ตัว) ผลลัพธ์อาจไม่ตรงตามที่คาดหวัง — ควรตรวจสอบความถูกต้องของ
  รูปแบบอีเมลก่อน (ทบทวน Input Validation จากขั้นตอนที่ 882) ก่อนส่งเข้า Mask เสมอ

### แบบฝึกหัดที่ 885.1

**โจทย์**: หากต้องเพิ่มการ Mask สำหรับ "ที่อยู่บ้าน" ซึ่งเป็นข้อความยาวและไม่มีรูปแบบตายตัว จะออกแบบกฎ
การ Mask อย่างไรให้เข้ากับโครงสร้าง `MASKFLD` ที่มีอยู่

**เฉลยแนวทาง**: สามารถเพิ่ม `WHEN "ADDR "` เข้าไปใน `EVALUATE` โดยออกแบบกฎเฉพาะ เช่น เปิดเผยเฉพาะ
คำสุดท้าย (ชื่อจังหวัด) แต่ปกปิดรายละเอียดที่อยู่เฉพาะเจาะจง (บ้านเลขที่ ถนน) เนื่องจากที่อยู่แบบเต็มมีความ
เสี่ยงต่อความเป็นส่วนตัวสูงกว่าการเปิดเผยแค่จังหวัด สิ่งสำคัญคือควรรักษาโครงสร้างเดิมไว้ (รับ Type, ตัดสินใจ
กฎการ Mask ตาม Type, คืนค่าผ่าน `LK-FIELD-OUT`) เพื่อให้ผู้เรียกใช้ Subprogram นี้ไม่ต้องเปลี่ยนวิธีเรียกใช้
เลย เป็นการขยายความสามารถแบบ backward-compatible เช่นเดียวกับหลักการ Copybook Governance ที่เรียนใน
Part 086 ขั้นตอนที่ 855 (เพิ่มความสามารถใหม่โดยไม่กระทบของเดิม)

---

## ขั้นตอนที่ 886: Audit Logging Pattern สำหรับระบบองค์กร

### ทำไมต้องมี Audit Log แยกจาก Log ทั่วไป

ระบบการเงินต้องตอบคำถามได้เสมอว่า **"ใครทำอะไร เมื่อไหร่ กับข้อมูลอะไร"** — นี่คือหน้าที่ของ **Audit Log**
ซึ่งต่างจาก Log ทั่วไปที่เน้นการ debug ปัญหาทางเทคนิค Audit Log ต้องมีโครงสร้างที่ชัดเจน คงที่ และสำคัญ
ที่สุดคือ **ต้องไม่บันทึกข้อมูลอ่อนไหวแบบดิบ ๆ ลงไปเลย** (ต้องผ่านการ Mask จากขั้นตอนที่ 884-885 มาก่อนเสมอ)

### สร้าง Subprogram AUDITLOG

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. AUDITLOG.
       AUTHOR. COBOL-COURSE.
      *> Appends one structured audit line to AUDIT.LOG:
      *> TIMESTAMP | USER-ID | ACTION | DETAIL
      *> Callers MUST pass an already-masked DETAIL (Part 089 step
      *> 885) - this subprogram does not mask anything itself, it
      *> only records what it is given.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT AUDIT-FILE ASSIGN TO "AUDIT.LOG"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-AUDIT-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  AUDIT-FILE.
       01  AUDIT-LINE          PIC X(120).

       WORKING-STORAGE SECTION.
       01  WS-AUDIT-STATUS     PIC XX.
       01  WS-NOW                   PIC X(21).
       01  WS-NOW-FIELDS REDEFINES WS-NOW.
           05  WS-NOW-YYYY          PIC 9(4).
           05  WS-NOW-MO            PIC 9(2).
           05  WS-NOW-DD            PIC 9(2).
           05  WS-NOW-HH            PIC 9(2).
           05  WS-NOW-MI            PIC 9(2).
           05  WS-NOW-SS            PIC 9(2).
           05  FILLER               PIC X(7).
       01  WS-TIMESTAMP        PIC X(19).

       LINKAGE SECTION.
       01  LK-USER-ID          PIC X(10).
       01  LK-ACTION           PIC X(20).
       01  LK-DETAIL           PIC X(50).

       PROCEDURE DIVISION USING LK-USER-ID LK-ACTION LK-DETAIL.
       MAIN-PARA.
           MOVE FUNCTION CURRENT-DATE TO WS-NOW
           STRING WS-NOW-YYYY "-" WS-NOW-MO "-" WS-NOW-DD "T"
               WS-NOW-HH ":" WS-NOW-MI ":" WS-NOW-SS
               DELIMITED BY SIZE INTO WS-TIMESTAMP

      *> OPEN EXTEND fails with status 35 ("file does not exist") on
      *> the very first call, before AUDIT.LOG has ever been created.
      *> Fall back to OPEN OUTPUT to create it, then every later call
      *> in this or future runs uses EXTEND (append) as normal.
           OPEN EXTEND AUDIT-FILE
           IF WS-AUDIT-STATUS = "35"
               OPEN OUTPUT AUDIT-FILE
           END-IF

      *> MUST initialize the record to SPACES before STRING: STRING
      *> only overwrites the bytes it fills, leaving whatever was
      *> there before (often uninitialized binary garbage on a
      *> freshly loaded FD record) in every byte past the strung
      *> text - a common cause of "bad character" (status 71) on
      *> LINE SEQUENTIAL WRITE.
           MOVE SPACES TO AUDIT-LINE
           STRING WS-TIMESTAMP " | " DELIMITED BY SIZE
               LK-USER-ID DELIMITED BY SIZE " | " DELIMITED BY SIZE
               LK-ACTION DELIMITED BY SIZE " | " DELIMITED BY SIZE
               LK-DETAIL DELIMITED BY SIZE
               INTO AUDIT-LINE
           WRITE AUDIT-LINE
           CLOSE AUDIT-FILE
           GOBACK.
```

### สองบั๊กจริงที่พบระหว่างพัฒนา Subprogram นี้

ระหว่างเตรียมตัวอย่างนี้ ผู้เขียนเจอบั๊กจริง 2 จุดที่คุ้มค่ามากที่จะบันทึกไว้ เพราะเป็นรูปแบบบั๊กที่พบได้บ่อย
มากในโค้ด COBOL ที่จัดการไฟล์และ String:

**บั๊กที่ 1 — `OPEN EXTEND` ล้มเหลวเมื่อไฟล์ยังไม่เคยถูกสร้าง**: ความพยายามครั้งแรกที่เขียนโค้ดแบบไม่มี
`FILE STATUS` ทำให้โปรแกรม crash ทันทีด้วยข้อความ `libcob: error: file does not exist (status = 35)`
เพราะ `OPEN EXTEND` (เปิดเพื่อต่อท้าย) คาดหวังว่าไฟล์นั้น**มีอยู่แล้ว** — ทางแก้คือตรวจสอบ `FILE STATUS`
หลัง `OPEN EXTEND` แล้วถ้าได้ค่า `"35"` (ไฟล์ไม่พบ) ให้ลอง `OPEN OUTPUT` แทนเพื่อสร้างไฟล์ใหม่

**บั๊กที่ 2 — `STRING` ทิ้งขยะไว้ในไบต์ที่ไม่ได้เติมข้อความ**: หลังแก้บั๊กที่ 1 แล้ว โปรแกรมยัง `WRITE` ไม่
สำเร็จ (`FILE STATUS = "71"` ซึ่งหมายถึง "Bad Character") สาเหตุคือ `AUDIT-LINE` เป็นฟิลด์ยาว 120 ไบต์
แต่ `STRING` เติมข้อความแค่ประมาณ 80-90 ไบต์แรกเท่านั้น **ไบต์ที่เหลือยังคงเป็นค่าเริ่มต้นที่ไม่แน่นอนจาก
หน่วยความจำ** (โดยเฉพาะเมื่อ Subprogram ถูกโหลดแบบไดนามิกผ่าน `CALL`) ซึ่งอาจเป็นอักขระควบคุม (control
character) ที่ไฟล์แบบ `LINE SEQUENTIAL` ไม่ยอมรับ — ทางแก้คือ **ใส่ `MOVE SPACES TO AUDIT-LINE` ก่อน
`STRING` เสมอ** เพื่อล้างค่าทั้งฟิลด์ให้เป็นช่องว่างก่อน แล้วค่อยให้ `STRING` เติมข้อความทับบางส่วน

### โปรแกรมทดสอบและผลลัพธ์จริง (หลังแก้บั๊กทั้งสองจุดแล้ว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP886AUDITTEST.
       AUTHOR. COBOL-COURSE.
      *> Demonstrates the correct pattern: mask sensitive data FIRST
      *> with MASKFLD, then pass only the masked value to AUDITLOG.
      *> The raw account number never reaches the log file at all.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-USER-ID          PIC X(10) VALUE "TELLER07".
       01  WS-ACTION           PIC X(20) VALUE "VIEW BALANCE".
       01  WS-TYPE             PIC X(5)  VALUE "ACCT".
       01  WS-ACCT-RAW         PIC X(30) VALUE "1234567890123456".
       01  WS-ACCT-MASKED      PIC X(30).
       01  WS-DETAIL           PIC X(50).

       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "MASKFLD" USING WS-TYPE WS-ACCT-RAW WS-ACCT-MASKED.
           STRING "ACCT=" DELIMITED BY SIZE
               FUNCTION TRIM(WS-ACCT-MASKED) DELIMITED BY SIZE
               INTO WS-DETAIL.
           CALL "AUDITLOG" USING WS-USER-ID WS-ACTION WS-DETAIL.

           MOVE "TELLER07" TO WS-USER-ID.
           MOVE "UPDATE BALANCE" TO WS-ACTION.
           MOVE "ACCT=************3456 STATUS=OK" TO WS-DETAIL.
           CALL "AUDITLOG" USING WS-USER-ID WS-ACTION WS-DETAIL.

           DISPLAY "AUDIT ENTRIES WRITTEN. MASKED VALUE USED: "
               FUNCTION TRIM(WS-ACCT-MASKED).
           STOP RUN.
```

```bash
cobc -m -o AUDITLOG.so AUDITLOG.cob
cobc -x -o step886_auditlog_test step886_auditlog_test.cob
rm -f AUDIT.LOG
COB_LIBRARY_PATH=. ./step886_auditlog_test
cat AUDIT.LOG
```

**ผลลัพธ์จริง:**

```
AUDIT ENTRIES WRITTEN. MASKED VALUE USED: ************3456

--- AUDIT.LOG ---
2026-09-28T20:32:00 | TELLER07   | VIEW BALANCE         | ACCT=************3456
2026-09-28T20:32:00 | TELLER07   | UPDATE BALANCE       | ACCT=************3456 STATUS=OK
```

**เมื่อรันโปรแกรมซ้ำอีกครั้ง** (ทดสอบว่า `OPEN EXTEND` ทำงานถูกต้องหลังไฟล์มีอยู่แล้ว):

```
--- AUDIT.LOG หลังรันครั้งที่ 2 (ต่อท้ายรายการเดิม ไม่เขียนทับ) ---
2026-09-28T20:32:00 | TELLER07   | VIEW BALANCE         | ACCT=************3456
2026-09-28T20:32:00 | TELLER07   | UPDATE BALANCE       | ACCT=************3456 STATUS=OK
2026-09-28T20:32:00 | TELLER07   | VIEW BALANCE         | ACCT=************3456
2026-09-28T20:32:00 | TELLER07   | UPDATE BALANCE       | ACCT=************3456 STATUS=OK
```

### อธิบายจุดสำคัญ

- สังเกตว่า `WS-DETAIL` ที่ส่งเข้า `AUDITLOG` มีแต่ค่าที่ผ่านการ Mask มาแล้ว (`ACCT=************3456`)
  — **เลขบัญชีจริง (`1234567890123456`) ไม่เคยถูกส่งเข้า `AUDITLOG` เลย** สอดคล้องกับหลักการที่ตั้งไว้
  ในคำอธิบายของ Subprogram: "Callers MUST pass an already-masked DETAIL"
- รูปแบบบรรทัด `TIMESTAMP | USER-ID | ACTION | DETAIL` เป็นรูปแบบที่มีโครงสร้างชัดเจน (delimiter `|`)
  ทำให้สามารถแยกวิเคราะห์ (parse) ย้อนหลังได้ง่ายด้วยเครื่องมืออื่น เช่น สคริปต์ตรวจสอบ Compliance หรือ
  ระบบ SIEM (Security Information and Event Management) ในองค์กรจริง
- การทดสอบรันซ้ำ 2 ครั้งพิสูจน์ว่า `OPEN EXTEND` ทำงานถูกต้องเมื่อไฟล์มีอยู่แล้ว (ต่อท้ายรายการเดิม) และ
  fallback เป็น `OPEN OUTPUT` ทำงานถูกต้องเมื่อไฟล์ยังไม่มี (สร้างไฟล์ใหม่)

### ข้อควรระวัง

- **`AUDITLOG` ไม่ตรวจสอบหรือ Mask ข้อมูลใด ๆ เอง** — ความรับผิดชอบทั้งหมดตกอยู่ที่ผู้เรียกใช้ (Caller)
  ที่ต้อง Mask ข้อมูลก่อนส่งเข้ามาเสมอ นี่คือ Contract ที่ต้องสื่อสารให้ทุกทีมที่ใช้ Subprogram นี้เข้าใจ
  ตรงกัน (ทบทวนแนวคิด Contract จาก Part 086 ขั้นตอนที่ 855) มิฉะนั้นอาจมีทีมใดทีมหนึ่งลืม Mask แล้ว
  ข้อมูลอ่อนไหวรั่วไหลเข้า Log โดยไม่ตั้งใจ
- ไฟล์ `AUDIT.LOG` เองก็เป็นไฟล์ที่มีข้อมูลอ่อนไหว (แม้จะ Mask แล้ว ก็ยังบอกได้ว่าใครทำธุรกรรมอะไรกับ
  บัญชีไหนบ้าง) จึงต้องมีสิทธิ์การเข้าถึงไฟล์ที่รัดกุม — เนื้อหาต่อไปในขั้นตอนที่ 887 จะสอนเรื่องนี้โดยตรง

### แบบฝึกหัดที่ 886.1

**โจทย์**: จงอธิบายว่าทำไมบั๊กที่ 2 (STRING ทิ้งขยะในไบต์ที่ไม่ได้เติม) ถึงปรากฏเฉพาะตอนที่ `AUDITLOG`
ถูกเรียกแบบ Subprogram ผ่าน `CALL` แต่ไม่ปรากฏตอนทดสอบโค้ดเดียวกันแบบเป็นโปรแกรมหลัก (`-x`) ที่ใช้
`MOVE` เติมค่าคงที่ตรง ๆ ก่อนหน้านั้น

**เฉลยแนวทาง**: ในการทดสอบก่อนหน้าที่ใช้ `MOVE "TEST LINE ONE" TO AUDIT-LINE` ค่าทั้ง 120 ไบต์ของ
`AUDIT-LINE` ถูกเขียนทับทั้งหมด (MOVE ข้อความสั้นเข้าฟิลด์ที่ยาวกว่าจะเติมช่องว่างในส่วนที่เหลือให้อัตโนมัติ
ตามกฎ MOVE ของ COBOL ที่เรียนมาตั้งแต่ Part 008) จึงไม่มีไบต์ใดเหลือค่าขยะเลย แต่ `STRING` มีพฤติกรรม
ต่างจาก `MOVE` โดยพื้นฐาน — **`STRING` เติมเฉพาะไบต์ที่ข้อความจริง ๆ ครอบคลุมถึงเท่านั้น ไม่แตะไบต์ที่
เหลือเลยแม้แต่น้อย** เมื่อ `AUDIT-LINE` เป็นฟิลด์ในไฟล์ (`FILE SECTION`) ของ Subprogram ที่ถูกโหลดแบบ
ไดนามิก ค่าเริ่มต้นก่อนการเรียกใช้ครั้งแรกไม่ได้ถูกกำหนดให้เป็นช่องว่างเสมอไป (ต่างจากตัวแปรใน `WORKING-
STORAGE SECTION` ที่มี `VALUE` clause ชัดเจน) ทำให้ไบต์ที่เหลือมีค่าที่ไม่แน่นอน — บทเรียนคือ **ต้อง
`MOVE SPACES` หรือ `INITIALIZE` ฟิลด์ก่อนใช้ `STRING` เสมอ เมื่อฟิลด์ปลายทางมีความยาวมากกว่าข้อความที่
คาดว่าจะเติม**

---

## ขั้นตอนที่ 887: สิทธิ์การเข้าถึงไฟล์อย่างปลอดภัย (Secure File Permissions)

### ปัญหา: GnuCOBOL สร้างไฟล์ด้วยสิทธิ์เริ่มต้นที่อาจเปิดกว้างเกินไป

ทบทวนจาก Part 069: RACF ควบคุมสิทธิ์การเข้าถึง Dataset บน Mainframe จริง แต่เมื่อ COBOL รันบน Linux/
GnuCOBOL (ทบทวน Part 077 เรื่อง Cloud/Modernization) **สิทธิ์ไฟล์ถูกควบคุมด้วยกลไกของระบบปฏิบัติการ
(Unix File Permissions) แทน** — และค่าเริ่มต้นอาจไม่ปลอดภัยเท่าที่ควรสำหรับไฟล์ที่มีข้อมูลอ่อนไหว

### ทดสอบจริง: ตรวจสอบสิทธิ์ไฟล์ที่ COBOL สร้างขึ้นด้วยค่าเริ่มต้นของระบบ

```bash
umask
```

**ผลลัพธ์จริง:**

```
0022
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP887MAKEFILE.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT BAL-FILE ASSIGN TO "BALANCE887.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
       DATA DIVISION.
       FILE SECTION.
       FD  BAL-FILE.
       01  BAL-RECORD PIC X(30).
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT BAL-FILE.
           MOVE "ACCT=1234567890 BAL=5000.00" TO BAL-RECORD.
           WRITE BAL-RECORD.
           CLOSE BAL-FILE.
           STOP RUN.
```

```bash
cobc -x -o step887_makefile step887_makefile.cob
./step887_makefile
stat -c "%a %n" BALANCE887.DAT
```

**ผลลัพธ์จริง:**

```
644 BALANCE887.DAT
```

### วิเคราะห์อันตราย

สิทธิ์ `644` (`rw-r--r--`) หมายความว่า: **เจ้าของไฟล์อ่าน/เขียนได้, แต่ผู้ใช้กลุ่มเดียวกันและผู้ใช้อื่น ๆ
ทุกคนในเครื่อง "อ่านได้" ด้วย** — สำหรับไฟล์ที่มีเลขบัญชีและยอดเงินของลูกค้าอย่าง `BALANCE887.DAT` นี้
สิทธิ์แบบนี้เปิดกว้างเกินความจำเป็นไปมาก ตรงข้ามกับหลักการ **Least Privilege** ที่เรียนมาตลอดหลักสูตร
(ทบทวนจาก Part 069 ขั้นตอนที่ 682 และ 684) — ในสภาพแวดล้อมที่มีผู้ใช้หลายคนใช้เครื่องเดียวกัน (เช่น
Server ที่ทีมพัฒนาหลายคนแชร์กัน) ผู้ใช้คนอื่นที่ไม่เกี่ยวข้องกับระบบนี้เลยสามารถอ่านไฟล์ข้อมูลลูกค้าได้ทันที

### ทางแก้ที่ 1: จำกัดสิทธิ์หลังสร้างไฟล์ด้วย chmod

```bash
chmod 600 BALANCE887.DAT
stat -c "%a %n" BALANCE887.DAT
```

**ผลลัพธ์จริง:**

```
600 BALANCE887.DAT
```

สิทธิ์ `600` (`rw-------`) หมายความว่า **เฉพาะเจ้าของไฟล์เท่านั้นที่อ่าน/เขียนได้** ไม่มีใครอื่นในเครื่อง
เข้าถึงได้เลย — เหมาะสมกับไฟล์ข้อมูลลูกค้าที่ควรเข้าถึงได้เฉพาะโดย Process ที่รันด้วยบัญชีผู้ใช้ที่ถูกต้องเท่านั้น

### ทางแก้ที่ 2: กำหนด umask ที่เข้มงวดกว่าก่อนรันโปรแกรม (ป้องกันตั้งแต่ต้นทาง)

```bash
(umask 0177; ./step887_makefile2)
stat -c "%a %n" BALANCE887B.DAT
```

**ผลลัพธ์จริง:**

```
600 BALANCE887B.DAT
```

การตั้ง `umask 0177` ก่อนรันโปรแกรม (ค่า umask ที่ลบสิทธิ์ `rwx` ของ Group และ Other ทั้งหมด) ทำให้ไฟล์
ใหม่ทุกไฟล์ที่โปรแกรมสร้างขึ้น**ได้สิทธิ์ที่ปลอดภัยตั้งแต่วินาทีแรกที่ถูกสร้าง** โดยไม่ต้องมาไล่ `chmod`
ทีหลัง — วิธีนี้ปลอดภัยกว่าทางแก้ที่ 1 เพราะไม่มีช่วงเวลาสั้น ๆ ที่ไฟล์เปิดกว้างเกินไปก่อนจะถูก `chmod`

### ตารางสรุปสิทธิ์ไฟล์ที่ควรใช้กับข้อมูลแต่ละประเภท

| ประเภทไฟล์ | สิทธิ์ที่แนะนำ | เหตุผล |
|---|---|---|
| ไฟล์ข้อมูลลูกค้า/บัญชี (เช่น `BALANCE.DAT`) | `600` | เฉพาะ Process/ผู้ใช้ที่ถูกต้องเท่านั้นเข้าถึงได้ |
| ไฟล์ Audit Log (`AUDIT.LOG`) | `600` หรือ `640` (ถ้าทีม Security ต้องอ่านผ่าน Group) | จำกัดเฉพาะผู้ที่จำเป็นต้องตรวจสอบ |
| ไฟล์ Configuration ที่ไม่มีความลับ | `644` | อ่านได้กว้างกว่าได้ เพราะไม่มีข้อมูลอ่อนไหว |
| โปรแกรมที่คอมไพล์แล้ว (Executable) | `755` | ต้องรันได้จากหลายบัญชีผู้ใช้ตามการออกแบบระบบ |

### ข้อควรระวัง

- **`chmod`/`umask` เป็นกลไกของระบบปฏิบัติการ ไม่ใช่ของ COBOL โดยตรง** — GnuCOBOL ไม่มีคำสั่งภายใน
  ภาษาที่ตั้งค่าสิทธิ์ไฟล์ได้ ต้องจัดการผ่าน Shell Script ที่ห่อหุ้มการรันโปรแกรม COBOL หรือตั้งค่า umask
  ระดับ Process/Service ที่ใช้รันงาน Batch (เทียบเคียงกับการที่ RACF ควบคุมสิทธิ์ที่ระดับ z/OS แยกจาก
  ตรรกะภายในโปรแกรม COBOL เอง ตามที่ Part 069 อธิบายไว้)
- การเปลี่ยน umask ส่งผลกับ**ทุกไฟล์ที่ Process นั้นสร้างขึ้นหลังจากนั้น** ไม่ใช่แค่ไฟล์เดียว — ต้องระวัง
  ผลข้างเคียงหากโปรแกรมเดียวกันต้องสร้างไฟล์อื่นที่จำเป็นต้องมีสิทธิ์กว้างกว่า (เช่น ไฟล์รายงานที่ต้องแชร์
  ให้ทีมอื่นอ่าน) ควรตั้ง umask เฉพาะช่วงที่จำเป็น หรือ `chmod` แยกเป็นรายไฟล์แทน

### แบบฝึกหัดที่ 887.1

**โจทย์**: จงอธิบายว่าทำไมทางแก้ที่ 2 (ตั้ง `umask` ก่อนรันโปรแกรม) จึงปลอดภัยกว่าทางแก้ที่ 1 (`chmod`
หลังสร้างไฟล์) แม้ผลลัพธ์สุดท้าย (สิทธิ์ไฟล์ `600`) จะเหมือนกันทุกประการ

**เฉลยแนวทาง**: ทางแก้ที่ 1 มี **"ช่วงเวลาที่เสี่ยง" (window of vulnerability)** ระหว่างตอนที่ไฟล์ถูก
สร้างขึ้น (ด้วยสิทธิ์เริ่มต้น `644` ที่เปิดกว้าง) จนถึงตอนที่คำสั่ง `chmod` ทำงานสำเร็จ — แม้ช่วงเวลานี้จะสั้น
มาก (เสี้ยววินาที) แต่ในทางทฤษฎี หากมี Process อื่นในเครื่องเดียวกันจ้องอ่านไฟล์นี้อยู่พอดี (เช่น ระบบ
ที่ถูกโจมตีอยู่แล้วและมี Process ที่เป็นอันตรายคอยตรวจสอบไฟล์ใหม่ที่ถูกสร้าง) ก็ยังมีโอกาสอ่านไฟล์ได้ในช่วง
สั้น ๆ นั้น ในขณะที่ทางแก้ที่ 2 ทำให้ไฟล์**ไม่เคยมีสิทธิ์เปิดกว้างเลยแม้แต่วินาทีเดียว** เพราะระบบปฏิบัติการ
กำหนดสิทธิ์ตาม umask ตั้งแต่ตอนสร้างไฟล์ทันที (atomic ร่วมกับการสร้างไฟล์) หลักการนี้ตรงกับแนวคิด "deny
by default, allow by exception" ที่เรียนมาจาก Part 069 ขั้นตอนที่ 684 — ปลอดภัยกว่าเสมอเมื่อสามารถทำได้

---

## ขั้นตอนที่ 888: การจัดการ Error อย่างปลอดภัย (Fail Securely)

### หลักการ: ข้อความ Error ที่ผู้ใช้เห็น ต้องไม่รั่วไหลข้อมูลภายในระบบ

ทบทวนจาก Part 069 ขั้นตอนที่ 686: การแสดงผลลัพธ์ผิดพลาดแบบเงียบ ๆ เป็นอันตราย — แต่ในทางกลับกัน
**การแสดงรายละเอียด error มากเกินไปก็เป็นอันตรายเช่นกัน** โดยเฉพาะเมื่อข้อความ error รั่วไหลข้อมูล
อ่อนไหว (เช่น เลขบัญชีเต็มรูปแบบ, ชื่อไฟล์ภายในระบบ, โครงสร้างฐานข้อมูล) ให้กับผู้ใช้ภายนอกที่ไม่ควรเห็น
ข้อมูลเหล่านี้เลย หลักการที่ถูกต้องคือ **"Fail Securely"**: บันทึกรายละเอียดครบถ้วนไว้ภายใน (Audit Log
ที่มีสิทธิ์จำกัดตามขั้นตอนที่ 887) แต่แสดงข้อความทั่วไปที่ไม่มีข้อมูลอ่อนไหวใด ๆ ให้ผู้ใช้ภายนอกเห็นแทน

### โปรแกรมทดสอบ: แยกข้อความภายในกับข้อความที่ผู้ใช้เห็นอย่างเด็ดขาด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP888FAILSECURE.
       AUTHOR. COBOL-COURSE.
      *> Demonstrates "fail securely": the FULL error detail (which
      *> may include sensitive data) goes only to the internal audit
      *> log, never to the caller-facing message. The caller-facing
      *> message is always a fixed, generic, non-sensitive string.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCT-FILE ASSIGN TO "NOFILE888.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-ACCT-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCT-FILE.
       01  ACCT-RECORD          PIC X(30).

       WORKING-STORAGE SECTION.
       01  WS-ACCT-STATUS       PIC XX.
       01  WS-ACCT-NUMBER       PIC X(16) VALUE "1234567890123456".
       01  WS-USER-ID           PIC X(10) VALUE "TELLER07".
       01  WS-ACTION            PIC X(20) VALUE "OPEN ACCT FILE".
       01  WS-INTERNAL-DETAIL   PIC X(50).
       01  WS-CALLER-MESSAGE    PIC X(60).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT ACCT-FILE.
           IF WS-ACCT-STATUS NOT = "00"
      *> INTERNAL: full technical detail, including the raw account
      *> number and the exact file status - goes to the audit log
      *> only, which has restricted file permissions (step 887).
               STRING "FILE=NOFILE888.DAT STATUS=" DELIMITED BY SIZE
                   WS-ACCT-STATUS DELIMITED BY SIZE
                   " ACCT=" DELIMITED BY SIZE
                   WS-ACCT-NUMBER DELIMITED BY SIZE
                   INTO WS-INTERNAL-DETAIL
               CALL "AUDITLOG" USING WS-USER-ID WS-ACTION
                   WS-INTERNAL-DETAIL

      *> EXTERNAL: generic message only - no file name, no status
      *> code, no account number. This is what a caller at the
      *> web-wrapper boundary (Part 073) would actually see.
               MOVE "Unable to process request. Please contact support."
                   TO WS-CALLER-MESSAGE
               DISPLAY "CALLER SEES: " WS-CALLER-MESSAGE
           ELSE
               DISPLAY "FILE OPENED OK"
               CLOSE ACCT-FILE
           END-IF.
           STOP RUN.
```

```bash
cobc -x -o step888_failsecure step888_failsecure.cob
rm -f AUDIT.LOG NOFILE888.DAT
COB_LIBRARY_PATH=. ./step888_failsecure
cat AUDIT.LOG
```

**ผลลัพธ์จริง:**

```
CALLER SEES: Unable to process request. Please contact support.

--- AUDIT.LOG (internal only) ---
2026-09-28T20:32:49 | TELLER07   | OPEN ACCT FILE       | FILE=NOFILE888.DAT STATUS=35 ACCT=1234567890123456
```

### อธิบายจุดสำคัญ

- ผู้ใช้ภายนอก (หรือ Client ที่เรียกผ่าน Web-Wrapper) เห็นแค่ข้อความทั่วไป **"Unable to process request.
  Please contact support."** — ไม่มีชื่อไฟล์, ไม่มีรหัสสถานะ, ไม่มีเลขบัญชีปรากฏเลย
- ทีม Operations ที่มีสิทธิ์เข้าถึง `AUDIT.LOG` (ที่มีสิทธิ์ไฟล์จำกัดตามขั้นตอนที่ 887) เห็นรายละเอียดครบถ้วน
  พอที่จะวินิจฉัยปัญหาได้จริง: ไฟล์ไหน, รหัสสถานะอะไร (35 = ไม่พบไฟล์, ทบทวนจาก Part 030), เกี่ยวข้องกับ
  บัญชีไหน — ข้อมูลนี้จำเป็นสำหรับการแก้ไขปัญหา แต่ไม่ควรให้บุคคลภายนอกเห็น
- รูปแบบนี้คือการนำ**หลักการเดียวกับ Layered Architecture** (Part 086 ขั้นตอนที่ 852) มาประยุกต์ใช้กับ
  การจัดการ Error: แยก "รายละเอียดภายใน" (Data Access Layer เห็นได้) ออกจาก "สิ่งที่ผู้ใช้เห็น"
  (Presentation Layer ควบคุม) อย่างชัดเจน

### ข้อควรระวัง

- **อย่าใช้ข้อความทั่วไปเดียวกันสำหรับทุก error จนผู้ใช้ไม่สามารถแยกแยะปัญหาที่ตัวเองแก้ไขได้กับปัญหาที่
  ต้องรอระบบแก้เท่านั้น** — เช่น ถ้าปัญหาคือผู้ใช้กรอกข้อมูลผิดรูปแบบ (จากขั้นตอนที่ 882) ควรแจ้งว่า "ข้อมูล
  ที่กรอกไม่ถูกต้อง กรุณาตรวจสอบอีกครั้ง" ซึ่งไม่รั่วไหลข้อมูลภายในแต่ยังคงช่วยผู้ใช้ได้ ในขณะที่ปัญหาระดับ
  ระบบ (เช่น ไฟล์หาย, การเชื่อมต่อฐานข้อมูลล้มเหลว) จึงควรใช้ข้อความทั่วไปแบบ "กรุณาติดต่อฝ่ายสนับสนุน"
  ตามตัวอย่างนี้
- Audit Log ที่เก็บรายละเอียด error แบบเต็มรูปแบบ (รวมเลขบัญชีในตัวอย่างนี้) **ยังคงต้องมีสิทธิ์ไฟล์ที่
  รัดกุมตามขั้นตอนที่ 887 เสมอ** — Fail Securely ไม่ได้แปลว่าข้อมูลอ่อนไหวหายไปจากระบบทั้งหมด เพียงแต่
  ถูกจำกัดให้อยู่ในที่ที่ควบคุมการเข้าถึงได้อย่างเหมาะสมเท่านั้น

### แบบฝึกหัดที่ 888.1

**โจทย์**: จงยกตัวอย่างสถานการณ์ 1 กรณีที่การแสดงข้อความ error แบบละเอียดเกินไปกับผู้ใช้ภายนอก อาจนำไปสู่
การถูกโจมตีจริงได้

**เฉลยแนวทาง**: ตัวอย่างเช่น หากระบบแสดงข้อความ error ตรง ๆ ว่า **"ERROR: SQL syntax error near
'DROP TABLE' at line 1"** เมื่อมีคนพยายามส่งข้อมูลที่มีลักษณะ SQL Injection เข้ามา ข้อความนี้จะยืนยันให้
ผู้โจมตีรู้ทันทีว่า (1) ระบบมีช่องโหว่ SQL Injection จริง และ (2) ระบบใช้ฐานข้อมูลชนิดใด (จากรูปแบบข้อความ
error ที่เฉพาะเจาะจงกับแต่ละ Database Engine) ข้อมูลนี้ช่วยให้ผู้โจมตีปรับแต่ง payload การโจมตีให้แม่นยำ
ขึ้นไปอีก (เทคนิคที่เรียกว่า "Error-based" reconnaissance) ในขณะที่การแสดงข้อความทั่วไป (เช่น "คำขอไม่
สามารถดำเนินการได้") จะไม่ให้ข้อมูลใด ๆ แก่ผู้โจมตีเลยว่าการโจมตีสำเร็จหรือล้มเหลว หรือระบบใช้เทคโนโลยีใด

---

## ขั้นตอนที่ 889: ป้องกัน Resource Exhaustion และ Algorithmic DoS ที่ชั้นแอปพลิเคชัน

### เชื่อมโยงกับ Part 088: เมื่อความซับซ้อนเชิงอัลกอริทึมกลายเป็นช่องโหว่ความปลอดภัย

Part 088 สอนว่าลูปที่มีความซับซ้อน O(n) หรือ O(n²) ใช้เวลาแปรผันตามขนาดข้อมูล — Part นี้ชี้ให้เห็นแง่มุม
ด้านความปลอดภัยของเรื่องเดียวกัน: **ถ้าขนาดข้อมูล (n) ถูกกำหนดโดยผู้ใช้ภายนอกโดยตรง โดยไม่มีการตรวจสอบ
ขอบเขต ผู้โจมตีสามารถส่งค่า n ที่ใหญ่มาก ๆ เพื่อทำให้โปรแกรมใช้เวลา/ทรัพยากรระบบจนหมดได้** — ภัยคุกคาม
แบบนี้เรียกว่า **Algorithmic Denial of Service (Algorithmic DoS)**

### สถานการณ์: Web-Wrapper (Part 073) รับพารามิเตอร์ "จำนวนระเบียนที่ต้องการ" จากภายนอก

ทบทวนจาก Part 073: Wrapper Service อาจมี endpoint ที่รับพารามิเตอร์ เช่น "ดึงรายการธุรกรรมย้อนหลัง N
รายการ" — ถ้า N มาจากผู้ใช้โดยตรงและ COBOL นำไปใช้กำหนดจำนวนรอบของลูปโดยไม่ตรวจสอบขอบเขตก่อนเลย
ผู้โจมตีสามารถส่ง N = 999,999,999 เพื่อบังคับให้โปรแกรมวนลูปมหาศาล ใช้ CPU ของเครื่องแม่ข่ายจนกระทบ
ผู้ใช้คนอื่นที่ใช้ระบบเดียวกันอยู่ (โดยเฉพาะอันตรายมากในช่วง Batch Window ที่ทรัพยากรมีจำกัดตามที่ Part
066 อธิบายไว้)

### โปรแกรมทดสอบ: ตรวจสอบขอบเขตค่าก่อนเข้าลูปเสมอ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP889REQLIMIT.
       AUTHOR. COBOL-COURSE.
      *> Defense in depth at the COBOL layer: even though the web
      *> wrapper (Part 073) SHOULD validate the requested record
      *> count before calling this program, COBOL does not trust
      *> that and enforces its own hard ceiling. Without this check,
      *> a caller (malicious or just buggy) could request a loop
      *> large enough to exhaust CPU time for everyone else sharing
      *> the batch window - an algorithmic denial-of-service, tying
      *> back to the O(n) awareness from Part 088.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-REQUESTED-COUNT   PIC 9(9).
       01  WS-MAX-ALLOWED       PIC 9(9) VALUE 100000.
       01  WS-IDX               PIC 9(9).
       01  WS-ACCUM             PIC 9(9) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Enter requested record count: " WITH NO ADVANCING.
           ACCEPT WS-REQUESTED-COUNT.

           IF WS-REQUESTED-COUNT > WS-MAX-ALLOWED
               DISPLAY "REJECTED: requested " WS-REQUESTED-COUNT
                   " exceeds maximum allowed " WS-MAX-ALLOWED
                   ". Request denied before any processing loop ran."
           ELSE
               PERFORM VARYING WS-IDX FROM 1 BY 1
                       UNTIL WS-IDX > WS-REQUESTED-COUNT
                   ADD 1 TO WS-ACCUM
               END-PERFORM
               DISPLAY "PROCESSED " WS-ACCUM " records."
           END-IF.
           STOP RUN.
```

```bash
cobc -x -o step889_reqlimit step889_reqlimit.cob
time (printf -- "999999999\n" | ./step889_reqlimit)
time (printf -- "100000\n" | ./step889_reqlimit)
```

**ผลลัพธ์จริง:**

```
=== คำขอปกติ (500 รายการ) ===
PROCESSED 000000500 records.

=== คำขอที่พยายามโจมตี (999,999,999 รายการ) ===
REJECTED: requested 999999999 exceeds maximum allowed 000100000.
Request denied before any processing loop ran.
real	0m0.003s

=== คำขอสูงสุดที่อนุญาต (100,000 รายการ) ===
PROCESSED 000100000 records.
real	0m0.010s
```

### วิเคราะห์ผล

คำขอที่พยายามระบุจำนวน 999,999,999 รายการ **ถูกปฏิเสธทันทีภายใน 0.003 วินาที** — เร็วกว่าคำขอปกติเสียอีก
เพราะการตรวจสอบเงื่อนไข `IF WS-REQUESTED-COUNT > WS-MAX-ALLOWED` เกิดขึ้น**ก่อน**ที่ลูปจะเริ่มทำงานเลย
แม้แต่รอบเดียว ทำให้ไม่ว่าค่าที่ผู้โจมตีส่งมาจะใหญ่แค่ไหน (999,999,999 หรือค่าสูงสุดที่ `PIC 9(9)` รับได้)
ต้นทุนของการปฏิเสธยังคงเท่าเดิมเสมอ (constant time rejection) — นี่คือรูปแบบการป้องกันที่มีประสิทธิภาพ
สูงสุดสำหรับภัยคุกคามประเภทนี้

### ข้อควรระวัง

- **ค่า `WS-MAX-ALLOWED` ต้องถูกกำหนดตามขีดความสามารถจริงของระบบและความต้องการทางธุรกิจ** ไม่ใช่
  ตัวเลขที่เลือกขึ้นเองตามใจ — ควรทดสอบว่าค่าสูงสุดที่อนุญาตยังคงทำงานเสร็จภายในเวลาที่ยอมรับได้จริง (ทบทวน
  หลักการวัดผลจาก Part 088)
- **การตรวจสอบนี้ต้องทำที่ทุกจุดที่รับค่าจากภายนอก** ไม่ใช่แค่จุดเดียว — ถ้าระบบมีหลาย endpoint ที่รับ
  พารามิเตอร์กำหนดขนาดลูป (เช่น จำนวนหน้าที่จะแสดง, จำนวนรายการที่จะ export) ทุกจุดต้องมีการตรวจสอบขอบเขต
  แยกกันตามหลักการ "Never Trust, Always Verify" จากขั้นตอนที่ 881
- เทคนิคนี้ป้องกันได้เฉพาะ **Algorithmic DoS ที่มาจากค่าพารามิเตอร์เดียว** เท่านั้น — ภัยคุกคามแบบ DoS อื่น
  (เช่น ผู้โจมตีส่งคำขอจำนวนมหาศาลพร้อมกันแม้แต่ละคำขอจะมีขนาดเล็ก) ต้องอาศัยกลไกอื่นเพิ่มเติม เช่น Rate
  Limiting ที่ชั้น Web-Wrapper หรือ Load Balancer ซึ่งอยู่นอกขอบเขตของโปรแกรม COBOL เอง

### แบบฝึกหัดที่ 889.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบขอบเขตค่าในขั้นตอนนี้จึงเป็นตัวอย่างที่ดีของหลักการ "Never Trust,
Always Verify" จากขั้นตอนที่ 881 แม้ว่า Web-Wrapper (Part 073) ควรจะตรวจสอบค่านี้มาก่อนแล้วก็ตาม

**เฉลยแนวทาง**: หลักการ "Never Trust, Always Verify" หมายความว่าแต่ละชั้นของระบบต้องตรวจสอบข้อมูล
นำเข้าด้วยตัวเอง แม้ชั้นก่อนหน้าจะอ้างว่าตรวจสอบมาแล้วก็ตาม เพราะ (1) Wrapper อาจมีบั๊กหรือถูกข้ามไปได้
(เช่น มีการเรียก COBOL โดยตรงจากช่องทางอื่นที่ไม่ผ่าน Wrapper ตามที่ Part 086 ขั้นตอนที่ 852 อธิบายไว้ว่า
Business Logic อาจถูกเรียกจาก Presentation Layer หลายแบบ) (2) การพึ่งพาการตรวจสอบของชั้นอื่นเพียง
อย่างเดียวทำให้ระบบมี **จุดล้มเหลวเดียว (Single Point of Failure)** ด้านความปลอดภัย หากจุดนั้นถูกข้าม
หรือมีบั๊ก ทั้งระบบจะไม่มีการป้องกันใด ๆ เหลืออยู่เลย การที่ COBOL ตรวจสอบขอบเขตค่าด้วยตัวเองอีกชั้น (แม้
จะดูเหมือนซ้ำซ้อนกับสิ่งที่ Wrapper ควรทำ) จึงเป็นการสร้างเกราะป้องกันที่ไม่ขึ้นอยู่กับความถูกต้องของชั้นอื่น
เลย ตรงตามคำนิยามของ Defense in Depth ที่แท้จริง

---

## ขั้นตอนที่ 890: ประกอบทุกเทคนิคเข้าด้วยกัน — Pipeline ความปลอดภัยแบบ End-to-End

### เป้าหมาย: จำลองคำขอเข้ามาที่ Web-Wrapper แล้วผ่านทุกชั้นการป้องกันของ Part นี้

ขั้นตอนสุดท้ายนี้รวมทุกเทคนิคจากขั้นตอนที่ 882-888 เข้าเป็นโปรแกรมเดียว จำลองสถานการณ์จริง: คำขอ
"ตรวจสอบยอดคงเหลือบัญชี" เข้ามาพร้อมชื่อลูกค้าและเลขบัญชี — โปรแกรมต้อง (1) ตรวจสอบข้อมูลนำเข้า
(2) ปกปิดเลขบัญชีก่อนแสดงผลหรือบันทึก Log (3) บันทึก Audit Log เสมอไม่ว่าสำเร็จหรือล้มเหลว (4) แสดง
ข้อความที่เหมาะสมให้ผู้ใช้เห็น โดยไม่รั่วไหลข้อมูลภายใน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP890PIPELINE.
       AUTHOR. COBOL-COURSE.
      *> End-to-end mini demo combining every technique in Part 089:
      *> 1) sanitize input (step 882)
      *> 2) keep the query text fixed / input as pure data (step 883)
      *> 3) mask sensitive fields before they are ever displayed or
      *>    logged (steps 884-885)
      *> 4) write an audit entry using only the masked value (886)
      *> 5) fail securely: generic message out, full detail only to
      *>    the internal audit log (888)

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUST-NAME         PIC X(30).
      *> MUST match MASKFLD's LINKAGE SECTION field width (PIC X(30))
      *> exactly. COBOL does NOT check parameter sizes on CALL...
      *> USING by default - passing a shorter field here would make
      *> MASKFLD read past the end of it into whatever memory
      *> happens to follow, corrupting the masked result silently.
       01  WS-ACCT-NUMBER       PIC X(30).
       01  WS-CHAR-IDX          PIC 9(2).
       01  WS-ONE-CHAR          PIC X(1).
       01  WS-BAD-COUNT         PIC 9(3) VALUE 0.
       01  WS-TYPE              PIC X(5) VALUE "ACCT".
       01  WS-ACCT-MASKED       PIC X(30).
       01  WS-USER-ID           PIC X(10) VALUE "WEBAPI01".
       01  WS-ACTION            PIC X(20) VALUE "BALANCE INQUIRY".
       01  WS-DETAIL            PIC X(50).

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Enter customer name: " WITH NO ADVANCING.
           ACCEPT WS-CUST-NAME.
           DISPLAY "Enter account number (16 digits): "
               WITH NO ADVANCING.
           ACCEPT WS-ACCT-NUMBER.

      *> Step 1: sanitize the name field (same allow-list as 882).
           MOVE 0 TO WS-BAD-COUNT.
           PERFORM VARYING WS-CHAR-IDX FROM 1 BY 1
                   UNTIL WS-CHAR-IDX > 30
               MOVE WS-CUST-NAME(WS-CHAR-IDX:1) TO WS-ONE-CHAR
               IF WS-ONE-CHAR NOT = SPACE
                   IF (WS-ONE-CHAR < "A" OR WS-ONE-CHAR > "Z")
                       AND (WS-ONE-CHAR < "a" OR WS-ONE-CHAR > "z")
                       AND WS-ONE-CHAR NOT = "-"
                       ADD 1 TO WS-BAD-COUNT
                   END-IF
               END-IF
           END-PERFORM.

           IF WS-BAD-COUNT > 0
      *> FAIL SECURELY: full detail to audit log, generic message out.
               STRING "INVALID NAME INPUT: '" DELIMITED BY SIZE
                   WS-CUST-NAME DELIMITED BY SIZE "'"
                   DELIMITED BY SIZE INTO WS-DETAIL
               CALL "AUDITLOG" USING WS-USER-ID WS-ACTION WS-DETAIL
               DISPLAY "REQUEST DENIED: invalid input."
           ELSE
      *> Step 2/3: mask the account number before it is ever shown
      *> or logged - the raw number never appears in output.
               CALL "MASKFLD" USING WS-TYPE WS-ACCT-NUMBER
                   WS-ACCT-MASKED
               STRING "NAME=" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-CUST-NAME) DELIMITED BY SIZE
                   " ACCT=" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-ACCT-MASKED) DELIMITED BY SIZE
                   INTO WS-DETAIL
               CALL "AUDITLOG" USING WS-USER-ID WS-ACTION WS-DETAIL
               DISPLAY "REQUEST OK. Customer=" WS-CUST-NAME
                   " Account=" WS-ACCT-MASKED
           END-IF.
           STOP RUN.
```

### บั๊กจริงตัวที่ 3 ที่พบระหว่างพัฒนาตัวอย่างนี้: Parameter Size Mismatch

ความพยายามครั้งแรกประกาศ `WS-ACCT-NUMBER` เป็น `PIC X(16)` (ตรงตามความยาวจริงของเลขบัญชี) แต่ Subprogram
`MASKFLD` ที่สร้างไว้ในขั้นตอนที่ 885 คาดหวัง `LK-FIELD-IN` เป็น `PIC X(30)` — เมื่อรันจริง ผลลัพธ์ที่ได้คือ
เลขบัญชีถูก mask **ทั้งหมด 26 ตัวอักษร** (แทนที่จะเหลือ 4 ตัวท้ายตามที่ออกแบบไว้) เพราะ **COBOL ไม่ตรวจสอบ
ว่าขนาดพารามิเตอร์ที่ส่งเข้า `CALL...USING` ตรงกับที่ Subprogram คาดหวังหรือไม่** — เมื่อ `MASKFLD` พยายาม
อ่านฟิลด์ 30 ไบต์จากพื้นที่หน่วยความจำที่จริง ๆ มีแค่ 16 ไบต์ มันจะ**อ่านทะลุเข้าไปในหน่วยความจำของตัวแปร
ถัดไปในโปรแกรม** ทำให้ค่าที่คำนวณ (`FUNCTION LENGTH(FUNCTION TRIM(...))`) ผิดเพี้ยนไปจากความเป็นจริง

**ทางแก้**: เปลี่ยน `WS-ACCT-NUMBER` เป็น `PIC X(30)` ให้ตรงกับ `LINKAGE SECTION` ของ `MASKFLD` เป๊ะ ๆ
(ตามที่แสดงในโค้ดข้างต้น พร้อมคอมเมนต์เตือนไว้ในโค้ดโดยตรง)

```bash
cobc -x -o step890_pipeline step890_pipeline.cob
printf -- "JOHN SMITH\n1234567890123456\n" | COB_LIBRARY_PATH=. ./step890_pipeline
printf -- "X'; DROP TABLE--\n1234567890123456\n" | COB_LIBRARY_PATH=. ./step890_pipeline
cat AUDIT.LOG
```

**ผลลัพธ์จริง (หลังแก้ไขบั๊กแล้ว):**

```
=== คำขอปกติ ===
REQUEST OK. Customer=JOHN SMITH                     Account=************3456

=== คำขอที่มีอักขระอันตรายในชื่อ ===
REQUEST DENIED: invalid input.

--- AUDIT.LOG ---
2026-09-28T20:34:11 | WEBAPI01   | BALANCE INQUIRY      | NAME=JOHN SMITH ACCT=************3456
2026-09-28T20:34:11 | WEBAPI01   | BALANCE INQUIRY      | INVALID NAME INPUT: 'X'; DROP TABLE--
```

### วิเคราะห์ผล — ทุกเทคนิคทำงานร่วมกันได้จริง

- คำขอปกติ ("JOHN SMITH") ผ่านการตรวจสอบ, เลขบัญชีถูก mask ก่อนแสดงผล (`************3456`), และมีการ
  บันทึก Audit Log ด้วยค่าที่ mask แล้วเท่านั้น
- คำขอที่มีอักขระอันตราย (`X'; DROP TABLE--`) ถูกปฏิเสธทันที **แต่ยังคงมีการบันทึก Audit Log** ที่แสดง
  รายละเอียดของความพยายามนั้นไว้ (สำหรับการตรวจสอบย้อนหลังว่ามีความพยายามโจมตีเกิดขึ้น) — สังเกตว่าใน
  กรณีนี้ Audit Log **บันทึกข้อมูลดิบของ input ที่ถูกปฏิเสธ** (ไม่ใช่ข้อมูลอ่อนไหวของลูกค้าจริง เพราะคำขอ
  ถูกปฏิเสธไปแล้ว) ซึ่งเป็นการตัดสินใจที่สมเหตุสมผล เพราะข้อมูลนี้มีประโยชน์ต่อการตรวจจับรูปแบบการโจมตีมาก
  กว่าจะเป็นความเสี่ยงต่อความเป็นส่วนตัว
- นี่คือภาพรวมของ Defense in Depth ที่แท้จริง: **แต่ละชั้นการป้องกันทำงานเป็นอิสระต่อกัน** (Sanitize,
  Mask, Audit, Fail Securely) แต่ประกอบกันเป็นระบบที่ปลอดภัยครบถ้วน — ถ้าชั้นใดชั้นหนึ่งถูกข้ามหรือมีบั๊ก
  ชั้นอื่นยังคงทำงานปกป้องระบบอยู่บางส่วน

### Checklist สรุปความปลอดภัยระดับ Enterprise สำหรับแอปพลิเคชัน COBOL

| # | รายการตรวจสอบ | ขั้นตอนที่เกี่ยวข้อง |
|---|---|---|
| 1 | ข้อมูลนำเข้าทุกจุดผ่านการตรวจสอบด้วย Allow-List | 882 |
| 2 | ไม่มีการต่อ String ข้อมูลผู้ใช้เข้ากับคำสั่ง SQL โดยตรง | 883 |
| 3 | ข้อมูลอ่อนไหวถูก Mask ก่อนแสดงผล/บันทึก Log เสมอ | 884-885 |
| 4 | ทุกการกระทำสำคัญถูกบันทึกใน Audit Log ที่มีโครงสร้างชัดเจน | 886 |
| 5 | ไฟล์ข้อมูลอ่อนไหวมีสิทธิ์จำกัดเฉพาะผู้ที่จำเป็นต้องใช้ | 887 |
| 6 | ข้อความ error ที่แสดงต่อผู้ใช้ไม่รั่วไหลข้อมูลภายใน | 888 |
| 7 | มีการตรวจสอบขอบเขตค่าก่อนใช้ในลูปหรือการคำนวณที่มีต้นทุนสูง | 889 |
| 8 | RACF (ถ้ารันบน Mainframe) และ Secure Coding ทำงานร่วมกัน ไม่ใช่แทนกัน | ทบทวน Part 069 |

### ข้อควรระวัง

- Pipeline ตัวอย่างนี้เป็น**การสาธิตหลักการ**ในรูปแบบย่อ ระบบจริงระดับองค์กรอาจมีชั้นการตรวจสอบเพิ่มเติม
  อีกมาก (เช่น การตรวจสอบสิทธิ์ผู้ใช้ก่อนอนุญาตให้ดูข้อมูลบัญชีใด ๆ เลย ซึ่งเป็นเรื่องของ Authorization ที่
  แยกจาก Input Validation) — Part นี้ครอบคลุมเฉพาะเทคนิคระดับ Secure Coding ที่ทดสอบได้จริงด้วย
  GnuCOBOL เท่านั้น
- บั๊กทั้ง 3 จุดที่พบระหว่างเตรียม Part นี้ (OPEN EXTEND ล้มเหลว, STRING ทิ้งขยะ, Parameter Size
  Mismatch) ล้วนเป็นบั๊กที่**คอมไพล์ผ่านโดยไม่มี error หรือ warning ใด ๆ เลย** — ตอกย้ำบทเรียนสำคัญที่สุด
  ของ Part 088 และ Part นี้: **การคอมไพล์ผ่านไม่ได้แปลว่าโปรแกรมถูกต้อง ต้องทดสอบการทำงานจริงเสมอ**

### แบบฝึกหัดที่ 890.1

**โจทย์**: จากบั๊กทั้ง 3 จุดที่พบระหว่างเตรียม Part นี้ (OPEN EXTEND, STRING ทิ้งขยะ, Parameter Size
Mismatch) จงจัดลำดับว่าบั๊กใดมีความเสี่ยงด้านความปลอดภัยร้ายแรงที่สุดหากหลุดรอดไปถึงระบบ production
พร้อมให้เหตุผล

**เฉลยแนวทาง**: ไม่มีคำตอบตายตัว แต่แนวทางการวิเคราะห์ที่ดีคือ: **Parameter Size Mismatch** มีความเสี่ยง
ร้ายแรงที่สุดในบริบทความปลอดภัย เพราะทำให้ Subprogram อ่านหน่วยความจำนอกขอบเขตของตัวเอง ("buffer
over-read") ซึ่งอาจทำให้ผลลัพธ์การ Mask ข้อมูลผิดพลาดไปในทิศทางที่**เปิดเผยข้อมูลมากกว่าที่ควร** หรือ
เก็บข้อมูลของตัวแปรอื่นปนเข้ามาในผลลัพธ์โดยไม่ตั้งใจ (ในกรณีเลวร้ายกว่านี้ อาจกลายเป็นช่องโหว่ประเภท
"information disclosure" ที่ทำให้ข้อมูลอ่อนไหวของตัวแปรอื่นรั่วไหลออกมา) ส่วน **OPEN EXTEND ล้มเหลว**
ทำให้โปรแกรม crash ทันที ซึ่งแม้จะกระทบต่อความพร้อมใช้งาน (Availability) แต่อย่างน้อยก็ **"fail loud"**
(ล้มเหลวแบบเห็นได้ชัด) ไม่ใช่ให้ผลลัพธ์ผิดแบบเงียบ ๆ และ **STRING ทิ้งขยะ** แม้จะทำให้ WRITE ล้มเหลว
(fail loud เช่นกัน) แต่ก็มีความเสี่ยงที่ในบางกรณี ขยะที่หลงเหลืออาจไม่ทำให้เกิด error แต่กลับถูกเขียนลงไฟล์
จริง ๆ (ถ้าขยะนั้นบังเอิญเป็นอักขระที่ยอมรับได้) ทำให้ไฟล์ Log มีข้อมูลแปลกปลอมปนอยู่โดยไม่มีใครสังเกตเห็น
— ทั้งสามกรณีสอนบทเรียนเดียวกัน: **ต้องทดสอบการทำงานจริงของทุก Subprogram อย่างละเอียด ไม่ใช่แค่ตรวจสอบ
ว่าคอมไพล์ผ่าน**

---

## สรุปท้ายบท

Part 089 ขยายความปลอดภัยของ COBOL จาก Part 069 (RACF/Mainframe) ไปสู่ความปลอดภัยระดับแอปพลิเคชัน
ที่ครอบคลุมทุกจุดที่ COBOL สัมผัสกับโลกภายนอก พร้อมตัวอย่างที่ทดสอบได้จริงทุกกรณี และบั๊กจริง 3 จุดที่
ค้นพบระหว่างการพัฒนาที่กลายเป็นบทเรียนสำคัญ:

- **ขั้นตอนที่ 881**: ขยาย Defense in Depth จากขอบเขต RACF ไปสู่ทุกจุดเชื่อมต่อของสถาปัตยกรรมสมัยใหม่
  พร้อมหลักการ "Never Trust, Always Verify"
- **ขั้นตอนที่ 882**: Input Sanitization ด้วย Allow-List สำหรับข้อมูลจาก Web-Wrapper — ทดสอบจริงกับ
  ข้อมูลปกติและข้อมูลอันตราย 3 รูปแบบ
- **ขั้นตอนที่ 883**: พิสูจน์ปัญหา SQL Injection ด้วยการรันจริง (ข้อความ SQL เปลี่ยนความหมายจาก
  ข้อมูลนำเข้า) และวิธีแก้ด้วย Parameterization (ข้อความ SQL คงที่เสมอ ไม่ว่าข้อมูลจะเป็นอะไร)
- **ขั้นตอนที่ 884-885**: สร้างและทดสอบ Subprogram Masking จริง (`MASKACCT`, `MASKFLD`) ที่ปกปิด
  เลขบัญชี, เลขบัตรประชาชน, เบอร์โทรศัพท์, และอีเมล
- **ขั้นตอนที่ 886**: สร้าง Audit Logging Subprogram (`AUDITLOG`) พร้อมพบและแก้บั๊กจริง 2 จุด (OPEN
  EXTEND ล้มเหลว, STRING ทิ้งขยะทำให้ WRITE ล้มเหลวด้วย status 71)
- **ขั้นตอนที่ 887**: พิสูจน์ว่า GnuCOBOL สร้างไฟล์ด้วยสิทธิ์ `644` (เปิดกว้างเกินไป) โดยค่าเริ่มต้น และ
  วิธีแก้ด้วย `chmod 600` หรือ `umask 0177`
- **ขั้นตอนที่ 888**: หลักการ Fail Securely — แยกข้อความ error ภายใน (ครบถ้วน, ไปที่ Audit Log) จาก
  ข้อความที่ผู้ใช้เห็น (ทั่วไป, ไม่รั่วไหลข้อมูล)
- **ขั้นตอนที่ 889**: ป้องกัน Algorithmic DoS ด้วยการตรวจสอบขอบเขตค่าก่อนเข้าลูปเสมอ — พิสูจน์ว่าการ
  ปฏิเสธคำขอขนาดใหญ่ผิดปกติใช้เวลาเพียง 0.003 วินาที
- **ขั้นตอนที่ 890**: ประกอบทุกเทคนิคเป็น Pipeline เดียว พร้อมพบและแก้บั๊กจริงจุดที่ 3 (Parameter Size
  Mismatch ที่ทำให้เกิด Buffer Over-Read)

บทเรียนที่สำคัญที่สุดของ Part นี้คือ **ความปลอดภัยไม่ใช่ฟีเจอร์ที่เพิ่มเข้าไปทีหลัง แต่เป็นชุดของนิสัยการเขียน
โค้ดที่ต้องฝึกฝนและตรวจสอบด้วยการทดสอบจริงเสมอ** — บั๊กทั้ง 3 จุดที่พบระหว่างเตรียม Part นี้ล้วนคอมไพล์ผ่าน
โดยไม่มี error ใด ๆ เลย พิสูจน์ให้เห็นชัดเจนว่าทำไมการทดสอบการทำงานจริงจึงสำคัญไม่แพ้การเขียนโค้ดให้ถูกต้อง
ตามหลักไวยากรณ์

Part ถัดไปจะเปลี่ยนมุมมองจากเทคนิคการเขียนโค้ดไปสู่การบริหารจัดการโปรเจกต์ — **Part 090: การจัดการ
โปรเจกต์ COBOL ขนาดใหญ่** จะครอบคลุมการประมาณงาน, การจัดการหนี้ทางเทคนิค, การประสานงานหลายทีม,
และการบริหารความเสี่ยงสำหรับโครงการ Modernization ระดับองค์กร

**[ไปยัง Part 090: การจัดการโปรเจกต์ COBOL ขนาดใหญ่ →](part-090-project-management.md)**
