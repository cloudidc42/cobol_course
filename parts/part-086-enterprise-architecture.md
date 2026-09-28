# Part 086: สถาปัตยกรรมระบบองค์กรขนาดใหญ่ด้วย COBOL (ขั้นตอนที่ 851–860)

## คำนำของ Part นี้

ยินดีต้อนรับสู่ **เฟส 6: ระดับมืออาชีพ/โลก (Professional/World-class)** — เฟสสุดท้ายของหลักสูตรนี้!
เฟส 1-5 พาเราเดินทางจากพื้นฐานที่สุด (ตัวแปร, เงื่อนไข, ลูป) ผ่านระดับกลาง-ขั้นสูง (ไฟล์, Subprogram,
Report Writer) ไปจนถึง Mainframe/Enterprise (JCL, VSAM, DB2, CICS) และปิดท้ายด้วย COBOL สมัยใหม่
(REST API, Docker, CI/CD, Refactoring, Legacy Modernization) — Part 085 พิสูจน์ให้เห็นแล้วว่าเราสามารถ
นำระบบ Legacy **หนึ่งระบบ** มาปรับปรุงให้ทันสมัยได้อย่างสมบูรณ์

แต่คำถามที่ตามมาตามธรรมชาติคือ: **แล้วถ้ามีระบบแบบนี้นับร้อยนับพันระบบล่ะ?** องค์กรขนาดใหญ่จริงอย่าง
ธนาคารหรือบริษัทประกันภัยไม่ได้มี COBOL แค่โปรแกรมเดียว แต่มีระบบหลายพันโปรแกรมที่เขียนโดยทีมต่าง ๆ
ตลอดหลายสิบปี ใช้ Copybook ร่วมกันหลายร้อยไฟล์ รันเป็น Batch Job หลายพันงานทุกคืน และต้องสื่อสารกับ
ระบบอื่นทั้งภายในและภายนอกองค์กรอย่างต่อเนื่อง

Part นี้จะแนะนำแนวคิดสถาปัตยกรรมระดับองค์กร (Enterprise Architecture) ที่จำเป็นสำหรับการดูแลระบบ COBOL
ขนาดใหญ่แบบนี้: **Layered Architecture** ในบริบท COBOL, **Batch Scheduling Architecture** (การจัดลำดับ
และกู้คืนงาน Batch จำนวนมาก), **การกำกับดูแล Copybook ที่ใช้ร่วมกัน** (Copybook Governance) ในระดับ
องค์กร, และ **กลยุทธ์การควบคุมเวอร์ชันของ Interface** ระหว่างระบบ — เนื้อหาส่วนใหญ่เป็นแนวคิดเชิง
สถาปัตยกรรม แต่ทุกตัวอย่าง COBOL ที่ปรากฏยังคงผ่านการคอมไพล์และทดสอบจริงเช่นเดิม

---

## ขั้นตอนที่ 851: ภาพรวมความท้าทายของสถาปัตยกรรมระดับองค์กร

### จาก "หนึ่งระบบ" สู่ "ระบบนิเวศขององค์กร"

Part 085 แก้ปัญหาระดับ**หนึ่งระบบ**: ทำอย่างไรให้ `ARLEGACY.cob` หนึ่งโปรแกรมกลายเป็นระบบที่ดูแลง่าย
ทดสอบได้ และเข้าถึงได้ผ่าน API — แต่องค์กรขนาดใหญ่ต้องตอบคำถามที่ซับซ้อนกว่านั้นมาก:

1. ถ้ามี `ARDISC` แบบนี้ 200 ตัวที่แต่ละทีมเขียนแยกกัน จะรู้ได้อย่างไรว่าตัวไหนซ้ำซ้อนกัน?
2. ถ้า Copybook หนึ่งไฟล์ถูกใช้โดย 150 โปรแกรมจาก 12 ทีม การแก้ไขมันปลอดภัยแค่ไหน?
3. งาน Batch หลายพันงานที่ต้องรันทุกคืนควรจัดลำดับก่อนหลังอย่างไร ถ้างานหนึ่งล้มเหลวจะกระทบงานที่เหลือ
   แค่ไหน?
4. ระบบ A ที่ส่งข้อมูลให้ระบบ B ทุกวัน จะเปลี่ยนรูปแบบข้อมูลได้อย่างไรโดยไม่ทำให้ระบบ B พังกะทันหัน?

คำถามเหล่านี้ไม่มีคำตอบเดียวที่ถูกต้องตายตัว แต่มี**หลักการและรูปแบบ (pattern) ที่พิสูจน์แล้วว่าใช้ได้ผล**
ในอุตสาหกรรมจริง ซึ่ง Part นี้จะแนะนำทีละเรื่อง

### 4 เสาหลักของสถาปัตยกรรมองค์กรที่ Part นี้ครอบคลุม

```
+---------------------------------------------------------------+
|                Enterprise COBOL Architecture                   |
+---------------------------------------------------------------+
|                                                                 |
|  1. LAYERED ARCHITECTURE      2. BATCH SCHEDULING              |
|     (Step 852)                   ARCHITECTURE                  |
|     - Presentation Layer         (Step 853-854)                |
|     - Business Logic Layer       - Job dependency graphs       |
|     - Data Access Layer          - Condition code propagation  |
|                                   - Checkpoint/restart          |
|                                                                 |
|  3. COPYBOOK GOVERNANCE       4. INTERFACE VERSIONING          |
|     (Step 855-856)               (Step 857)                    |
|     - Safe evolution rules       - Explicit version fields      |
|     - Anti-patterns that          - Backward-compatible         |
|       silently corrupt data        parsing strategies           |
|                                                                 |
+---------------------------------------------------------------+
```

### ทำไมเรื่องเหล่านี้ถึงสำคัญมากขึ้นตามขนาดขององค์กร

สังเกตว่าทุกปัญหาข้างต้นมีลักษณะร่วมกันคือ: **ปัญหาที่ไม่ร้ายแรงในระบบขนาดเล็ก (เช่น Part 085) กลับ
กลายเป็นความเสี่ยงมหาศาลเมื่อขยายสเกล** — การแก้ Copybook หนึ่งไฟล์ในระบบที่มีโปรแกรมเดียวใช้มันไม่มี
ความเสี่ยงอะไรเลย แต่การแก้ Copybook เดียวกันในระบบที่มี 150 โปรแกรมใช้ร่วมกันอาจทำให้เกิดความเสียหาย
เป็นวงกว้างได้หากไม่มีกระบวนการกำกับดูแลที่ดี — นี่คือเหตุผลที่องค์กร Mainframe ขนาดใหญ่ลงทุนสร้าง
กระบวนการ กติกา และเครื่องมือเฉพาะทางสำหรับปัญหาเหล่านี้โดยเฉพาะ

### ข้อควรระวัง

- เนื้อหาใน Part นี้เป็นแนวคิดสถาปัตยกรรมที่บางส่วนต้องอาศัยบริบทขององค์กรจริง (ทีมงานหลายทีม, ระบบ
  Governance, เครื่องมือ Enterprise) จึงจะเห็นประโยชน์เต็มที่ — ตัวอย่าง COBOL ที่สาธิตในเอกสารนี้เป็น
  รูปแบบจำลองขนาดเล็กเพื่อให้เข้าใจหลักการ ไม่ใช่ระบบ Governance ขนาดเต็มรูปแบบขององค์กรจริง
- อย่าใช้แนวคิด "องค์กรขนาดใหญ่" เป็นข้ออ้างที่จะไม่นำหลักการเหล่านี้ไปใช้กับระบบขนาดเล็ก — การกำกับดูแล
  Copybook หรือการออกแบบ Layered Architecture ที่ดีตั้งแต่ระบบยังเล็กจะทำให้การขยายสเกลในอนาคตราบรื่น
  กว่ามาก (ทบทวนหลักการนี้จาก Part 085 ที่วางโครงสร้าง Layer ไว้ตั้งแต่ระบบยังมีแค่ 3 โปรแกรม)

### แบบฝึกหัดที่ 851.1

**โจทย์**: จงยกตัวอย่างเหตุการณ์ 1 กรณีที่คุณคิดว่า "ปัญหาเล็กในระบบขนาดเล็กที่กลายเป็นปัญหาใหญ่เมื่อ
ขยายสเกล" นอกเหนือจาก 4 ตัวอย่างที่กล่าวถึงในขั้นตอนนี้

**เฉลยแนวทาง**: คำตอบขึ้นกับผู้เรียน ตัวอย่างที่เป็นไปได้ เช่น การตั้งชื่อตัวแปรไม่สอดคล้องกัน (ในระบบ
เดียวไม่มีปัญหาเพราะมีแค่ทีมเดียวดูแล แต่เมื่อมี 50 ทีมตั้งชื่อ Copybook/ตัวแปรตามใจตัวเอง การค้นหาและ
ทำความเข้าใจโค้ดข้ามทีมจะยากขึ้นมาก), หรือการไม่มี File Status Checking ที่รัดกุม (ในระบบทดสอบที่ไฟล์
มีอยู่เสมอไม่มีปัญหา แต่ในสภาพแวดล้อม Production ที่มีระบบอื่นเขียนทับไฟล์เดียวกันพร้อมกันหลายโปรแกรม
ความเสี่ยงจาก Race Condition จะสูงขึ้นตามจำนวนโปรแกรมที่เข้าถึงไฟล์นั้น)

---

## ขั้นตอนที่ 852: Layered Architecture ในบริบท COBOL

### แนวคิด: สามชั้นสถาปัตยกรรมที่ใช้ได้แม้ไม่มี Framework สมัยใหม่

**Layered Architecture** เป็นแนวคิดพื้นฐานที่สุดของวิศวกรรมซอฟต์แวร์: แบ่งระบบออกเป็นชั้น (layer) ที่
แต่ละชั้นมีหน้าที่ชัดเจนและพึ่งพาชั้นถัดไปในทิศทางเดียวเท่านั้น แม้ COBOL จะไม่มี Framework อย่าง Spring
(Java) หรือ Django (Python) แต่แนวคิดเดียวกันนี้ใช้ได้จริง ตามที่ Part 085 พิสูจน์ไว้แล้ว:

```
+------------------------------------------+
|  PRESENTATION LAYER                       |
|  - REST API Wrapper (Part 073)            |
|  - CICS BMS Maps (Part 062-063)           |
|  - Screen Section (Part 040)              |
|  หน้าที่: รับ Input จากภายนอก แปลงรูปแบบ    |
|         ส่ง Output กลับในรูปแบบที่โลกภายนอก |
|         เข้าใจ (JSON, หน้าจอ, ฯลฯ)          |
+-------------------+------------------------+
                     |
                     v
+------------------------------------------+
|  BUSINESS LOGIC LAYER                     |
|  - Subprogram คำนวณล้วน (เช่น ARDISC.cob)  |
|  หน้าที่: กฎทางธุรกิจทั้งหมด ไม่รู้จักรูปแบบ |
|         ข้อมูลภายนอกหรือที่มาของข้อมูล      |
+-------------------+------------------------+
                     |
                     v
+------------------------------------------+
|  DATA ACCESS LAYER                        |
|  - Subprogram จัดการไฟล์/DB (เช่น ARLEDGER)|
|  - VSAM (Part 055-056), DB2 (Part 057-060) |
|  หน้าที่: อ่าน/เขียนข้อมูลถาวร ไม่รู้จักกฎ  |
|         ทางธุรกิจใด ๆ เลย                  |
+------------------------------------------+
```

### ย้อนดู AR-MINI จาก Part 085 ในฐานะตัวอย่างจริงของ Layered Architecture

สังเกตว่าโครงสร้างของ `ARCALC.cob` / `ARDISC.cob` / `ARLEDGER.cob` ที่สร้างไว้ใน Part 085 **ตรงกับ 3
ชั้นนี้เป๊ะ**:

| Layer | ไฟล์ใน Part 085 | หน้าที่ |
|---|---|---|
| Presentation | `ar_server.py` (REST wrapper) | แปลง JSON เป็นบรรทัด Fixed-Width และกลับกัน |
| Orchestration (ตัวเชื่อม) | `ARCALC.cob` | เรียก Business Logic แล้วส่งต่อไป Data Access |
| Business Logic | `ARDISC.cob` | คำนวณส่วนลด/ภาษี ไม่รู้จักไฟล์หรือ JSON เลย |
| Data Access | `ARLEDGER.cob` | อ่าน/เขียน `BALANCE.DAT` ไม่รู้จักสูตรคำนวณเลย |

### กฎทองของ Layered Architecture: การพึ่งพาต้องไหลทิศทางเดียว

กฎที่สำคัญที่สุดคือ **ชั้นบนพึ่งพาชั้นล่างได้ แต่ชั้นล่างห้ามพึ่งพาชั้นบนเด็ดขาด** — `ARDISC.cob` (Business
Logic) ต้องไม่มีโค้ดใดที่อ้างอิงถึง JSON หรือ HTTP เลย และ `ARLEDGER.cob` (Data Access) ต้องไม่มีสูตร
คำนวณส่วนลดปนอยู่เลย การละเมิดกฎนี้ (เช่น เขียนโค้ดตรวจสอบ HTTP header ไว้ใน `ARDISC.cob`) จะทำให้ชั้น
Business Logic "ผูกติด" กับ Presentation Layer ชั้นใดชั้นหนึ่งไปโดยไม่จำเป็น ทำให้นำไปใช้ซ้ำกับ
Presentation Layer อื่น (เช่น CICS Transaction แทน REST API) ได้ยากขึ้นมาก

### ประโยชน์ที่จับต้องได้ของ Layered Architecture ในองค์กรขนาดใหญ่

1. **ทีมต่างกันดูแลชั้นต่างกันได้** — ทีม Web/Mobile ดูแล Presentation Layer, ทีม Business Analyst
   ร่วมออกแบบ Business Logic Layer, ทีม Database/Storage ดูแล Data Access Layer โดยแต่ละทีมไม่จำเป็น
   ต้องเข้าใจรายละเอียดภายในของชั้นอื่นทั้งหมด
2. **เปลี่ยน Presentation ได้โดยไม่กระทบ Business Logic** — องค์กรอาจมี Presentation Layer หลายแบบ
   พร้อมกัน (REST API สำหรับเว็บ, CICS สำหรับพนักงานหน้าเคาน์เตอร์, Batch Interface สำหรับงานกลางคืน)
   ที่ทั้งหมดเรียกใช้ Business Logic Layer เดียวกัน — รับประกันว่ากฎธุรกิจ**สอดคล้องกันทุกช่องทาง**
3. **ทดสอบง่ายขึ้น** — Business Logic Layer ที่ไม่มี File I/O ทดสอบได้เร็วและง่ายที่สุด (ตามที่ Part 085
   ขั้นตอนที่ 846 พิสูจน์ไว้)

### ข้อควรระวัง

- Layered Architecture ไม่ได้แปลว่าต้องมี "3 ไฟล์เป๊ะ" เสมอไป — ระบบขนาดใหญ่อาจมีหลายสิบ Subprogram ใน
  แต่ละ Layer แนวคิดสำคัญคือ**การแบ่งหน้าที่ให้ชัดเจน** ไม่ใช่จำนวนไฟล์
- ระวัง "Leaky Abstraction" — สถานการณ์ที่รายละเอียดของ Layer ล่างรั่วไหลขึ้นไปยัง Layer บน (เช่น
  `ARLEDGER.cob` คืนค่า File Status Code ตรง ๆ ให้ `ARCALC.cob` นำไปแสดงผลโดยไม่แปลความหมายก่อน ทำให้
  Presentation Layer ต้อง "รู้" รายละเอียดภายในของ Data Access Layer ซึ่งขัดกับหลักการแยกชั้นที่ดี)

### แบบฝึกหัดที่ 852.1

**โจทย์**: หากต้องเพิ่ม Presentation Layer ใหม่ให้ AR-MINI จาก Part 085 เพื่อรองรับการเรียกผ่าน CICS
Transaction (แทน/เพิ่มเติมจาก REST API) จะต้องแก้ไข `ARDISC.cob` หรือ `ARLEDGER.cob` หรือไม่ เพราะเหตุใด

**เฉลยแนวทาง**: ไม่จำเป็นต้องแก้ไขทั้งสองไฟล์เลย เพราะ CICS Transaction เป็นเพียง Presentation Layer
อีกแบบหนึ่ง (แทนที่ REST API Wrapper) โปรแกรม CICS ใหม่สามารถ `CALL "ARDISC"` และ `CALL "ARLEDGER"`
ด้วยพารามิเตอร์เดียวกันได้โดยตรง เพราะ Business Logic Layer และ Data Access Layer ถูกออกแบบให้ไม่รู้จัก
Presentation Layer ใด ๆ เป็นการเฉพาะตั้งแต่แรก — นี่คือประโยชน์ที่จับต้องได้ของการแยก Layer อย่างถูกต้อง
ตามที่อธิบายไว้ในขั้นตอนนี้

---

## ขั้นตอนที่ 853: สถาปัตยกรรมการจัดตารางงาน Batch (Batch Scheduling Architecture)

### แนวคิด: องค์กรขนาดใหญ่รันงาน Batch นับพันงานทุกคืน

ทบทวนจาก Part 066 (Batch Processing Patterns) — องค์กรการเงินขนาดใหญ่มักมีงาน Batch หลายพันงานที่ต้อง
รันทุกคืนตามลำดับที่ถูกต้อง (เช่น งานคำนวณดอกเบี้ยต้องรันหลังงานปิดยอดธุรกรรมประจำวัน แต่ก่อนงานสร้าง
รายงานสรุป) การจัดลำดับนี้เรียกว่า **Job Dependency Graph** และเครื่องมือที่ควบคุมมันเรียกว่า **Job
Scheduler** (เช่น IBM Tivoli Workload Scheduler, Control-M เป็นต้นในโลกอุตสาหกรรมจริง)

### แผนภาพ Job Dependency Graph ตัวอย่าง

```
                   [DAILY-START]
                        |
          +-------------+-------------+
          v                           v
   [EXTRACT-TXN]                [EXTRACT-CUST]
   (ดึงธุรกรรมวันนี้)             (ดึงข้อมูลลูกค้า)
          |                           |
          +-------------+-------------+
                        v
                [VALIDATE-DATA]
                (ตรวจสอบความถูกต้อง)
                        |
                        v
                [CALC-INTEREST]
                (คำนวณดอกเบี้ย - ต้องรันหลัง VALIDATE
                 เสร็จสมบูรณ์เท่านั้น)
                        |
          +-------------+-------------+
          v                           v
   [UPDATE-LEDGER]              [GEN-REPORT]
   (ปรับปรุงบัญชี)                (สร้างรายงาน)
          |                           |
          +-------------+-------------+
                        v
                   [DAILY-END]
```

สังเกตว่า `CALC-INTEREST` **ต้องรอ** `VALIDATE-DATA` เสร็จสมบูรณ์ก่อนเสมอ (Dependency) แต่ `EXTRACT-TXN`
และ `EXTRACT-CUST` ทำงาน**ขนานกันได้** (ไม่มี Dependency ระหว่างกัน) — การออกแบบ Dependency Graph ที่ดี
ช่วยให้ Batch Window (ช่วงเวลากลางคืนที่มีให้รันงาน) ถูกใช้อย่างมีประสิทธิภาพสูงสุด

### Condition Code Propagation: กลไกที่ COBOL ใช้บอกสถานะให้ Scheduler รู้

ทบทวนจาก Part 053 (JCL ขั้นสูง) — แต่ละ Job Step คืนค่า **Condition Code** (ตัวเลข 0-4095 ตามธรรมเนียม
Mainframe) ให้ Scheduler ตัดสินใจว่าจะรันขั้นตอนถัดไปหรือไม่ ต่อไปนี้คือตัวอย่าง COBOL ที่จำลองกลไกนี้
และทดสอบได้จริงบน GnuCOBOL ผ่าน `RETURN-CODE`:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JOBCTL.
       AUTHOR. COBOL-COURSE.
      *> Illustrates mainframe-style condition-code propagation
      *> (Part 053) inside a single COBOL "batch controller" that
      *> simulates three job steps. Each step sets a severity code;
      *> the controller keeps the HIGHEST severity seen, exactly
      *> like JCL's COND=(n,operator) step-to-step checking.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FORCE-FAIL            PIC X(1).
       01  WS-STEP-CODE             PIC 9(2) VALUE 0.
       01  WS-WORST-CODE            PIC 9(2) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-FORCE-FAIL FROM CONSOLE
           DISPLAY "JOB START"

           PERFORM STEP-A
           PERFORM STEP-B
           PERFORM STEP-C

           DISPLAY "JOB END - WORST CONDITION CODE = " WS-WORST-CODE
           MOVE WS-WORST-CODE TO RETURN-CODE
           STOP RUN.

       STEP-A.
           DISPLAY "STEP-A: LOAD CUSTOMER EXTRACT ... CC=00"
           MOVE 0 TO WS-STEP-CODE
           PERFORM UPDATE-WORST-CODE.

      *> This step simulates a data-quality warning (CC=04), the
      *> mainframe convention for "completed, but check the output"
      *> - not a hard failure, unless WS-FORCE-FAIL asks for CC=12.
       STEP-B.
           IF WS-FORCE-FAIL = "Y"
               DISPLAY "STEP-B: VALIDATE RECORDS ... CC=12 (FAILED)"
               MOVE 12 TO WS-STEP-CODE
           ELSE
               DISPLAY "STEP-B: VALIDATE RECORDS ... CC=04 (WARNING)"
               MOVE 4 TO WS-STEP-CODE
           END-IF
           PERFORM UPDATE-WORST-CODE.

      *> Real job schedulers would skip this step if STEP-B's code
      *> exceeded a COND= threshold. This simplified controller
      *> always runs it, but a production version would guard it
      *> with "IF WS-WORST-CODE < 8" the same way JCL EXEC PGM=...
      *> COND=(8,LT) would.
       STEP-C.
           IF WS-WORST-CODE < 8
               DISPLAY "STEP-C: POST TO GENERAL LEDGER ... CC=00"
               MOVE 0 TO WS-STEP-CODE
           ELSE
               DISPLAY "STEP-C: SKIPPED (PRIOR STEP FAILED)"
           END-IF.

       UPDATE-WORST-CODE.
           IF WS-STEP-CODE > WS-WORST-CODE
               MOVE WS-STEP-CODE TO WS-WORST-CODE
           END-IF.
```

คอมไพล์และรันทั้งสองกรณี:

```bash
cobc -x -o jobctl jobctl.cob

echo "=== normal run ==="
echo "N" | ./jobctl; echo "exit=$?"

echo "=== forced failure ==="
echo "Y" | ./jobctl; echo "exit=$?"
```

**ผลลัพธ์จริงที่ได้:**

```
=== normal run ===
JOB START
STEP-A: LOAD CUSTOMER EXTRACT ... CC=00
STEP-B: VALIDATE RECORDS ... CC=04 (WARNING)
STEP-C: POST TO GENERAL LEDGER ... CC=00
JOB END - WORST CONDITION CODE = 04
exit=4
=== forced failure ===
JOB START
STEP-A: LOAD CUSTOMER EXTRACT ... CC=00
STEP-B: VALIDATE RECORDS ... CC=12 (FAILED)
STEP-C: SKIPPED (PRIOR STEP FAILED)
JOB END - WORST CONDITION CODE = 12
exit=12
```

สังเกตว่า `exit=4` และ `exit=12` ตรงกับ `RETURN-CODE` ที่โปรแกรมตั้งไว้พอดี — นี่คือกลไกเดียวกับที่ JCL
ใช้ตรวจสอบ `COND=(n,operator)` ระหว่าง Step (Part 053) และเป็นกลไกเดียวกับที่ CI Pipeline ใน Part 085
ขั้นตอนที่ 848 ใช้ตรวจสอบว่า Unit Test ผ่านหรือไม่ (`$?` ใน bash)

### ข้อควรระวัง

- ธรรมเนียม Condition Code แบบ Mainframe (`0` = สำเร็จ, `4` = คำเตือน, `8` = ข้อผิดพลาดระดับผู้ใช้,
  `12`+ = ข้อผิดพลาดร้ายแรง) เป็น**ธรรมเนียม**ไม่ใช่กฎตายตัว แต่ละองค์กรอาจกำหนดค่าที่ต่างกันได้ สิ่งสำคัญ
  คือทุกโปรแกรมในองค์กรเดียวกันต้องใช้ธรรมเนียมเดียวกันอย่างสม่ำเสมอ
- โปรแกรม `JOBCTL.cob` นี้เป็นการจำลองแนวคิดภายในโปรแกรมเดียว เพื่อให้ทดสอบบน GnuCOBOL ได้ง่าย — ใน
  Mainframe จริง แต่ละ "Step" มักเป็นคนละโปรแกรม/คนละ Job Step ที่ควบคุมโดย JCL ทั้งหมด (ทบทวน Part
  052-053) ไม่ใช่ย่อหน้าในโปรแกรมเดียวกันแบบนี้

### แบบฝึกหัดที่ 853.1

**โจทย์**: จงอธิบายว่าทำไม `STEP-C` ในโค้ดข้างต้นถึงตรวจสอบ `IF WS-WORST-CODE < 8` แทนที่จะตรวจสอบแค่
`WS-STEP-CODE` ของ `STEP-B` เพียงอย่างเดียว

**เฉลยแนวทาง**: เพราะในสายงาน Batch จริงที่มีหลาย Step ต่อเนื่องกัน แต่ละ Step ควรตรวจสอบ **สถานะสะสม
ของทั้งสาย** (worst condition code so far) ไม่ใช่แค่ผลลัพธ์ของ Step ก่อนหน้าเพียง Step เดียว เพราะ
ความล้มเหลวอาจเกิดขึ้นที่ Step ใดก็ได้ก่อนหน้า (`STEP-A`, `STEP-B`, หรือ Step อื่นที่อาจถูกเพิ่มเข้ามา
ในอนาคต) การตรวจสอบ `WS-WORST-CODE` ซึ่งสะสมค่าสูงสุดไว้ตลอด รับประกันว่า `STEP-C` จะถูกข้ามหากมี
ความล้มเหลวร้ายแรงเกิดขึ้น ณ จุดใดก็ตามในสายงานก่อนหน้า ไม่ใช่แค่ Step ที่อยู่ติดกันเท่านั้น

---

## ขั้นตอนที่ 854: รูปแบบ Checkpoint/Restart สำหรับงาน Batch ขนาดใหญ่

### แนวคิด: งาน Batch ที่ประมวลผลนานหลายชั่วโมงต้องกู้คืนได้โดยไม่เริ่มใหม่ทั้งหมด

งาน Batch ในองค์กรขนาดใหญ่บางงานประมวลผลข้อมูลหลายล้านระเบียนและใช้เวลาหลายชั่วโมง หากงานล้มเหลวกลาง
คัน (เช่น เครื่องดับ, ดิสก์เต็ม) การ**เริ่มประมวลผลใหม่ตั้งแต่ระเบียนแรก**จะเสียเวลามหาศาลและอาจไม่ทัน
Batch Window ที่มีอยู่จำกัด — รูปแบบ **Checkpoint/Restart** แก้ปัญหานี้โดยบันทึก "ตำแหน่งล่าสุดที่ทำสำเร็จ"
ไว้เป็นระยะ เมื่อโปรแกรมเริ่มทำงานใหม่ (Restart) มันจะอ่าน Checkpoint แล้ว**ข้ามงานที่ทำสำเร็จไปแล้ว**
ไปเริ่มทำงานต่อจากจุดที่ค้างไว้แทน

### ซอร์สโค้ดฉบับเต็ม: BATCHRUN.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BATCHRUN.
       AUTHOR. COBOL-COURSE.
      *> Checkpoint/restart pattern for batch scheduling (Step 854).
      *> Processes 5 fixed "customer" records; after each one it
      *> writes its position to CHECKPT.DAT. A real run always
      *> starts from the LAST checkpoint, not from record 1, so a
      *> job that dies partway through does not reprocess work that
      *> already completed - the same principle a JCL restart step
      *> (Part 053) relies on when re-running a failed job.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CHECKPOINT-FILE ASSIGN TO "CHECKPT.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-CKPT-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CHECKPOINT-FILE.
       01  CKPT-LINE                PIC 9(2).

       WORKING-STORAGE SECTION.
       01  WS-CKPT-STATUS           PIC XX.
       01  WS-LAST-DONE             PIC 9(2) VALUE 0.
       01  WS-STOP-AFTER            PIC 9(2).
       01  WS-IDX                   PIC 9(2).
       01  WS-CUSTOMERS.
           05  PIC X(10) VALUE "CUST-A0001".
           05  PIC X(10) VALUE "CUST-A0002".
           05  PIC X(10) VALUE "CUST-A0003".
           05  PIC X(10) VALUE "CUST-A0004".
           05  PIC X(10) VALUE "CUST-A0005".
       01  WS-CUSTOMER-TABLE REDEFINES WS-CUSTOMERS.
           05  WS-CUSTOMER-NAME PIC X(10) OCCURS 5 TIMES.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> WS-STOP-AFTER simulates a crash after N records (0 = run to
      *> completion normally). This is a TEST HOOK, not something a
      *> real batch job would have.
           ACCEPT WS-STOP-AFTER FROM CONSOLE
           PERFORM READ-CHECKPOINT
           DISPLAY "RESTARTING AFTER RECORD: " WS-LAST-DONE

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 5
               IF WS-IDX > WS-LAST-DONE
                   DISPLAY "PROCESSING " WS-CUSTOMER-NAME(WS-IDX)
                   PERFORM SAVE-CHECKPOINT
                   IF WS-STOP-AFTER > 0 AND WS-IDX = WS-STOP-AFTER
                       DISPLAY "SIMULATED CRASH AFTER RECORD "
                           WS-IDX
                       STOP RUN
                   END-IF
               ELSE
                   DISPLAY "SKIP (ALREADY DONE) "
                       WS-CUSTOMER-NAME(WS-IDX)
               END-IF
           END-PERFORM

           DISPLAY "BATCH COMPLETE"
           STOP RUN.

       READ-CHECKPOINT.
           OPEN INPUT CHECKPOINT-FILE
           IF WS-CKPT-STATUS = "00"
               READ CHECKPOINT-FILE
                   AT END MOVE 0 TO WS-LAST-DONE
                   NOT AT END MOVE CKPT-LINE TO WS-LAST-DONE
               END-READ
               CLOSE CHECKPOINT-FILE
           ELSE
               MOVE 0 TO WS-LAST-DONE
           END-IF.

       SAVE-CHECKPOINT.
           OPEN OUTPUT CHECKPOINT-FILE
           MOVE WS-IDX TO CKPT-LINE
           WRITE CKPT-LINE
           CLOSE CHECKPOINT-FILE.
```

### ทดสอบจริง: จำลองความล้มเหลวกลางคันแล้ว Restart

```bash
cobc -x -o batchrun batchrun.cob
rm -f CHECKPT.DAT

echo "=== first run, crash after record 3 ==="
echo "03" | ./batchrun

echo "=== restart, should resume from record 4 ==="
echo "00" | ./batchrun
```

**ผลลัพธ์จริงที่ได้:**

```
=== first run, crash after record 3 ===
RESTARTING AFTER RECORD: 00
PROCESSING CUST-A0001
PROCESSING CUST-A0002
PROCESSING CUST-A0003
SIMULATED CRASH AFTER RECORD 03
=== restart, should resume from record 4 ===
RESTARTING AFTER RECORD: 03
SKIP (ALREADY DONE) CUST-A0001
SKIP (ALREADY DONE) CUST-A0002
SKIP (ALREADY DONE) CUST-A0003
PROCESSING CUST-A0004
PROCESSING CUST-A0005
BATCH COMPLETE
```

การรันครั้งแรก "ล้ม" หลังประมวลผลระเบียนที่ 3 (จำลองด้วย `WS-STOP-AFTER = 3`) จากนั้นการรันครั้งที่สอง
(Restart) **ข้ามระเบียนที่ 1-3 ที่ทำสำเร็จไปแล้วโดยอัตโนมัติ** และประมวลผลต่อจากระเบียนที่ 4 พอดี — พิสูจน์
ว่ารูปแบบ Checkpoint/Restart ทำงานถูกต้องจริง

### ข้อควรระวัง

- Checkpoint ต้องถูกบันทึก **หลังจาก**งานของระเบียนนั้นเสร็จสมบูรณ์แล้วเท่านั้น หากบันทึก Checkpoint
  ก่อนแล้วเกิดความล้มเหลวระหว่างประมวลผลจริง ระเบียนนั้นจะถูก**ข้ามไปอย่างผิดพลาด**เมื่อ Restart (ข้อมูล
  สูญหาย) — ลำดับการบันทึก Checkpoint เทียบกับการทำงานจริงจึงสำคัญมาก (ในตัวอย่างนี้ยังมีจุดที่ควรปรับปรุง
  เพิ่มเติม: การบันทึก Checkpoint เกิดขึ้น**ก่อน**การ "ประมวลผลจริง" เสร็จสมบูรณ์ในบางกรณี ซึ่งเป็นเรื่อง
  ที่ควรพิจารณาอย่างละเอียดตามลักษณะงานจริงแต่ละประเภท)
- รูปแบบนี้ทำงานได้ดีกับงานที่ประมวลผลแบบ "ลำดับ" (sequential, ทีละระเบียนตามลำดับคงที่) เท่านั้น หากงาน
  ประมวลผลแบบขนาน (parallel) การออกแบบ Checkpoint จะซับซ้อนกว่านี้มาก (ต้องติดตามหลายตำแหน่งพร้อมกัน)

### แบบฝึกหัดที่ 854.1

**โจทย์**: จงอธิบายว่ารูปแบบ Checkpoint/Restart ในขั้นตอนนี้ มีความคล้ายคลึงและแตกต่างจากรูปแบบ
Idempotent Consumer ที่จะกล่าวถึงใน Part 087 (Microservices) อย่างไร

**เฉลยแนวทาง**: ทั้งสองรูปแบบมีเป้าหมายร่วมกันคือ**ป้องกันการทำงานซ้ำที่ไม่ตั้งใจ**เมื่อกระบวนการถูก
รันซ้ำ (retry/restart) — แต่ Checkpoint/Restart เหมาะกับงานที่ประมวลผลแบบ**ลำดับต่อเนื่องมีตำแหน่งชัดเจน**
(เช่น "ทำถึงระเบียนที่เท่าไหร่แล้ว") ในขณะที่ Idempotent Consumer (Part 087) เหมาะกับข้อความ/เหตุการณ์ที่
มาถึงแบบ**ไม่เรียงลำดับหรือซ้ำได้** (เช่นข้อความจาก Message Queue ที่อาจถูกส่งซ้ำ) จึงต้องตรวจสอบด้วย
"รหัสประจำตัว" ของแต่ละข้อความ (Message ID) แทนที่จะใช้แค่ "ตำแหน่งลำดับ" เพียงอย่างเดียว — ทั้งสอง
รูปแบบสามารถใช้ร่วมกันได้ในระบบเดียวกัน ขึ้นอยู่กับลักษณะของงานแต่ละส่วน

---

## ขั้นตอนที่ 855: การกำกับดูแล Copybook ที่ใช้ร่วมกันในระดับองค์กร (Copybook Governance)

### แนวคิด: Copybook คือ "สัญญา" (Contract) ระหว่างหลายโปรแกรม

ทบทวนจาก Part 033 — Copybook ทำให้หลายโปรแกรมใช้โครงสร้างข้อมูลเดียวกันได้โดยไม่ต้องพิมพ์ซ้ำ แต่เมื่อ
Copybook หนึ่งไฟล์ถูกใช้โดยโปรแกรมนับร้อยจากหลายทีม มันกลายเป็น **"สัญญา" (Contract)** ที่ทุกฝ่ายพึ่งพา
— การแก้ไข Copybook โดยไม่ระมัดระวังเปรียบเสมือนการแก้สัญญาโดยไม่แจ้งอีกฝ่าย ซึ่งอาจทำให้ระบบที่พึ่งพา
สัญญานั้นพังได้ทันที องค์กรขนาดใหญ่จึงต้องมี **กฎการวิวัฒน์ (Evolution Rules)** ที่ชัดเจนสำหรับ Copybook
ที่ใช้ร่วมกัน

### กฎทองข้อที่ 1: เพิ่มฟิลด์ใหม่ที่ "ท้ายสุด" เท่านั้น

พิสูจน์ด้วยการทดลองจริง: สมมติ Copybook เวอร์ชันเดิมมี 3 ฟิลด์ (`CUST-ID`, `CUST-NAME`,
`CUST-BALANCE`) แล้วมีทีมหนึ่งต้องการเพิ่มฟิลด์อีเมลเข้าไป — **วิธีที่ปลอดภัย** คือเพิ่มไว้ท้ายสุด:

```cobol
      *> Copybook: CUSTREC-V2.CPY (version 2)
      *> V2 adds CUST-EMAIL at the END of the V1 layout. Any program
      *> still compiled against the V1 layout (first three fields
      *> only) can keep reading files written by V2 without changes
      *> - this is the "safe evolution" rule governance boards
      *> enforce on shared copybooks.
       01  CUST-ID-V2              PIC 9(5).
       01  CUST-NAME-V2            PIC X(20).
       01  CUST-BALANCE-V2         PIC 9(7)V99.
       01  CUST-EMAIL-V2           PIC X(25).
```

โปรแกรม "V2 Writer" เขียนระเบียนด้วยโครงสร้างใหม่ (4 ฟิลด์):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. WRITERV2.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUSTV2.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
       01  CUST-REC-OUT.
           05  O-ID                PIC 9(5).
           05  O-NAME              PIC X(20).
           05  O-BAL               PIC 9(7)V99.
           05  O-EMAIL             PIC X(25).
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUST-FILE
           MOVE 10001 TO O-ID
           MOVE "SIAM TRADING CO" TO O-NAME
           MOVE 23000.00 TO O-BAL
           MOVE "AP@SIAMTRADING.EXAMPLE" TO O-EMAIL
           WRITE CUST-REC-OUT
           CLOSE CUST-FILE
           DISPLAY "V2 WRITER DONE".
```

โปรแกรม "V1 Reader" ที่**ไม่เคยถูกแก้ไขหรือคอมไพล์ใหม่เลย** (ยังคงรู้จักแค่ 3 ฟิลด์แรกตามโครงสร้างเดิม):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. READERV1.
      *> Compiled only against the ORIGINAL 3-field layout. Never
      *> touched or recompiled when CUST-EMAIL was added downstream.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUSTV2.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
       01  CUST-REC-IN.
           05  I-ID                PIC 9(5).
           05  I-NAME              PIC X(20).
           05  I-BAL               PIC 9(7)V99.
       WORKING-STORAGE SECTION.
       01  WS-EOF                  PIC X VALUE "N".
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT CUST-FILE
           PERFORM UNTIL WS-EOF = "Y"
               READ CUST-FILE
                   AT END MOVE "Y" TO WS-EOF
                   NOT AT END
                       DISPLAY "ID=" I-ID " NAME=" I-NAME
                           " BAL=" I-BAL
               END-READ
           END-PERFORM
           CLOSE CUST-FILE.
```

ทดสอบจริง: ให้ V2 Writer เขียนไฟล์ก่อน แล้วให้ V1 Reader (ที่ไม่รู้จักฟิลด์ใหม่เลย) มาอ่าน:

```bash
cobc -x -o writer_v2 writer_v2.cob
./writer_v2

cobc -x -o reader_v1 reader_v1.cob
./reader_v1
```

**ผลลัพธ์จริงที่ได้:**

```
V2 WRITER DONE
ID=10001 NAME=SIAM TRADING CO      BAL=0023000.00
```

**V1 Reader อ่านค่าถูกต้องครบทั้ง 3 ฟิลด์ แม้ไฟล์จะถูกเขียนด้วยโครงสร้าง V2 ที่มี 4 ฟิลด์ก็ตาม** — มัน
เพียงแค่ "มองไม่เห็น" ฟิลด์ที่ 4 (`CUST-EMAIL`) เท่านั้น ไม่มีข้อมูลใดเสียหายหรือเยื้องตำแหน่งผิดพลาดเลย
นี่คือเหตุผลที่ **"เพิ่มฟิลด์ท้ายสุดเท่านั้น"** เป็นกฎทองข้อที่ 1 ของ Copybook Governance

### ข้อควรระวัง

- แม้ V1 Reader จะยังทำงานได้ถูกต้อง แต่มันก็ **"ไม่เห็น" ข้อมูลใหม่** (`CUST-EMAIL`) เลย — ถ้าตรรกะทาง
  ธุรกิจในอนาคตจำเป็นต้องใช้อีเมล โปรแกรมที่ไม่เคยอัปเดตจะไม่มีวันรู้จักข้อมูลนี้จนกว่าจะถูกคอมไพล์ใหม่
  ด้วย Copybook เวอร์ชันล่าสุด — การเพิ่มฟิลด์ท้ายสุดแก้ปัญหาเรื่อง "ไม่พัง" แต่ไม่ได้แก้ปัญหาเรื่อง
  "ทุกโปรแกรมเห็นข้อมูลล่าสุดโดยอัตโนมัติ"
- กฎนี้ใช้ได้กับไฟล์แบบ `LINE SEQUENTIAL` และการเข้าถึงฟิลด์ตามตำแหน่ง (positional) เท่านั้น หากระบบ
  เปลี่ยนไปใช้รูปแบบข้อมูลที่มีชื่อฟิลด์กำกับชัดเจน (เช่น JSON ผ่าน `JSON GENERATE`/`PARSE` จาก Part 045)
  กฎการวิวัฒน์จะต่างออกไป (เพิ่มฟิลด์ใหม่ที่ใดก็ได้ใน JSON โดยทั่วไปไม่กระทบผู้อ่านเดิม เพราะเข้าถึงด้วย
  ชื่อ ไม่ใช่ตำแหน่ง)

### แบบฝึกหัดที่ 855.1

**โจทย์**: จงอธิบายว่าทำไมกฎ "เพิ่มฟิลด์ท้ายสุดเท่านั้น" ถึงใช้ไม่ได้กับการ**ลบ**ฟิลด์ที่มีอยู่เดิมออก
แม้จะลบฟิลด์ที่อยู่ท้ายสุดก็ตาม

**เฉลยแนวทาง**: เพราะการลบฟิลด์ (แม้จะเป็นฟิลด์ท้ายสุด) จะทำให้ความยาว Record ทั้งหมดสั้นลง หากโปรแกรม
ที่ยังไม่ได้อัปเดต (เช่น V1 Reader ที่รู้จัก 3 ฟิลด์) พยายามอ่านไฟล์ที่ถูกเขียนด้วยโครงสร้างที่ลบฟิลด์
ที่ 3 ออกไปแล้ว (เหลือแค่ 2 ฟิลด์) โปรแกรมนั้นจะพยายามอ่านฟิลด์ที่ 3 (`CUST-BALANCE`) จากตำแหน่งที่ไม่มี
ข้อมูลจริงอยู่แล้ว (อาจเป็นช่องว่างท้ายบรรทัดหรือข้อมูลของบรรทัดถัดไปในบางกรณี) ทำให้ได้ค่าที่ผิดพลาดหรือ
โปรแกรมอาจล้มเหลวขณะอ่านไฟล์ได้ — การลบฟิลด์ที่มีอยู่แล้วจึงเป็น **Breaking Change** เสมอ ไม่ว่าจะอยู่
ตำแหน่งใดก็ตาม ต้องมีกระบวนการแจ้งเตือนและปรับปรุงโปรแกรมที่เกี่ยวข้องทั้งหมดให้พร้อมกันก่อนเสมอ (คนละ
กรณีกับการเพิ่มฟิลด์ใหม่ท้ายสุดที่ปลอดภัยกว่ามาก)

---

## ขั้นตอนที่ 856: Anti-Pattern — ผลกระทบร้ายแรงจากการแทรกฟิลด์กลางโครงสร้าง

### กฎทองข้อที่ 2: ห้ามแทรกฟิลด์ใหม่ "กลาง" โครงสร้างเดิมเด็ดขาด

ขั้นตอนที่แล้วพิสูจน์ว่าการเพิ่มฟิลด์ท้ายสุดปลอดภัย — ขั้นตอนนี้จะพิสูจน์ **ด้านตรงข้าม**: การแทรกฟิลด์
ใหม่ไว้ **กลาง** โครงสร้างเดิม (แม้จะดูเหมือนเป็นการเปลี่ยนแปลงเล็กน้อย) ก่อให้เกิดความเสียหายร้ายแรง
แบบ **เงียบ (silent corruption)** โดยไม่มี Error ใด ๆ เตือนเลย

### ตัวอย่าง Anti-Pattern: แทรก CUST-PHONE ไว้กลางโครงสร้าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. WRITERV3.
      *> ANTI-PATTERN: inserts CUST-PHONE in the MIDDLE of the
      *> layout instead of at the end. Do not do this to a shared
      *> copybook - this step uses it to show exactly why.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUSTV3BAD.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
       01  CUST-REC-OUT.
           05  O-ID                PIC 9(5).
           05  O-NAME              PIC X(20).
           05  O-PHONE             PIC X(10).
           05  O-BAL               PIC 9(7)V99.
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUST-FILE
           MOVE 10002 TO O-ID
           MOVE "BANGKOK SUPPLIES LTD" TO O-NAME
           MOVE "0812345678" TO O-PHONE
           MOVE 17000.00 TO O-BAL
           WRITE CUST-REC-OUT
           CLOSE CUST-FILE
           DISPLAY "V3 (BAD) WRITER DONE".
```

สังเกตว่าโปรแกรมนี้แทรก `O-PHONE PIC X(10)` ไว้**ระหว่าง** `O-NAME` และ `O-BAL` — ดูผิวเผินอาจคิดว่า
"ก็แค่เพิ่มฟิลด์ใหม่เหมือนเดิม" แต่ตำแหน่งที่แทรกต่างจากขั้นตอนที่ 855 อย่างสิ้นเชิง

### ทดสอบจริง: ให้ "V1 Reader" ตัวเดิม (ไม่เปลี่ยนแปลง) มาอ่านไฟล์ที่เขียนด้วยโครงสร้างใหม่นี้

```bash
cobc -x -o writer_v3_bad writer_v3_bad.cob
./writer_v3_bad

# reader_v1.cob ตัวเดิมทุกประการจากขั้นตอนที่ 855 (rebuild ให้ชี้ไปที่ไฟล์ใหม่)
cobc -x -o reader_v1_against_v3 reader_v1_against_v3.cob
./reader_v1_against_v3
```

**ผลลัพธ์จริงที่ได้:**

```
V3 (BAD) WRITER DONE
ID=10002 NAME=BANGKOK SUPPLIES LTD BAL=0812345.67
```

### วิเคราะห์หายนะที่เกิดขึ้น

**`BAL=0812345.67` เป็นค่าที่ผิดพลาดอย่างสิ้นเชิง** — ยอดคงเหลือจริงคือ `17000.00` แต่โปรแกรมกลับอ่านได้
`812345.67`! เกิดอะไรขึ้น: เมื่อ `O-PHONE` (10 ตัวอักษร: `"0812345678"`) ถูกแทรกไว้กลางโครงสร้าง โปรแกรม
`READERV1` ที่ยังคงเชื่อว่า "หลัง `NAME` มาคือ `BAL` ทันที" จึงอ่านไบต์ 10 ตัวแรกของเบอร์โทรศัพท์
(`"0812345678"` — 9 ใน 10 หลักแรก) มาตีความเป็นตัวเลขของฟิลด์ `BAL` (`PIC 9(7)V99`, 9 หลัก) แทน! และ
ที่อันตรายที่สุดคือ **ไม่มี Error หรือ Warning ใด ๆ เกิดขึ้นเลย** — โปรแกรมทำงาน "สำเร็จ" แต่ให้ข้อมูลที่
ผิดพลาดอย่างสิ้นเชิงแบบเงียบ ๆ (Silent Data Corruption)

### เปรียบเทียบผลกระทบระหว่างสองขั้นตอน

| การเปลี่ยนแปลง | ตำแหน่งที่เพิ่ม | ผลกระทบต่อ Reader เดิม |
|---|---|---|
| ขั้นตอนที่ 855 (ปลอดภัย) | ท้ายสุดของโครงสร้าง | อ่านฟิลด์เดิมถูกต้องครบถ้วน มองไม่เห็นฟิลด์ใหม่เท่านั้น |
| ขั้นตอนที่ 856 (อันตราย) | กลางโครงสร้าง | **ข้อมูลทุกฟิลด์หลังจุดที่แทรกเสียหายทั้งหมด โดยไม่มี Error เตือน** |

### ข้อควรระวัง

- นี่คือเหตุผลสำคัญที่สุดที่องค์กรขนาดใหญ่ต้องมี**กระบวนการอนุมัติการเปลี่ยนแปลง Copybook** (Change
  Approval Process) อย่างเป็นทางการ ไม่ใช่ปล่อยให้ทีมใดทีมหนึ่งแก้ไข Copybook ที่ใช้ร่วมกันได้ตามใจชอบ
  — ความเสียหายแบบ Silent Corruption อาจไม่ถูกค้นพบจนกว่าจะสร้างปัญหาทางธุรกิจร้ายแรง (เช่น ยอดบัญชีลูกค้า
  ผิดพลาดโดยไม่มีใครรู้เป็นเวลานาน)
- Code Review ที่ดีสำหรับการเปลี่ยนแปลง Copybook ควรตรวจสอบเสมอว่า **ฟิลด์ใหม่ถูกเพิ่มที่ท้ายสุดเท่านั้น**
  และควรมีการทดสอบอัตโนมัติ (คล้ายกับที่สาธิตในขั้นตอนที่ 855-856) เป็นส่วนหนึ่งของ CI Pipeline (ทบทวน
  Part 078 และ Part 085 ขั้นตอนที่ 848) เพื่อจับความผิดพลาดแบบนี้ก่อนที่จะถึงมือ Production

### แบบฝึกหัดที่ 856.1

**โจทย์**: จงอธิบายว่าทำไมความเสียหายแบบนี้ถึงเรียกว่า "Silent" (เงียบ) ทั้งที่ค่าที่ได้ (`0812345.67`)
ดูผิดปกติมากจนน่าจะสังเกตเห็นได้ง่าย

**เฉลยแนวทาง**: คำว่า "Silent" ในที่นี้หมายถึง**ไม่มีกลไกใด ๆ ของระบบ** (ไม่ว่าจะเป็น Compiler, Runtime,
หรือ File I/O) ที่แจ้งเตือนข้อผิดพลาดออกมาโดยอัตโนมัติ — โปรแกรมคอมไพล์ผ่าน รันจบโดยไม่มี Error ใด ๆ
และ File Status Code ก็รายงานว่า "00" (สำเร็จ) ตามปกติทุกประการ ความผิดปกติของค่าตัวเลขเป็นสิ่งที่มนุษย์
ต้องสังเกตเห็นเอง**หลังจากดูผลลัพธ์**เท่านั้น ซึ่งในกรณีข้อมูลจำนวนมาก (เช่นระบบที่ประมวลผลล้านระเบียน
ต่อคืน) มนุษย์แทบเป็นไปไม่ได้ที่จะตรวจตาดูทุกค่าด้วยตาเปล่า — หากค่าที่ผิดพลาดบังเอิญอยู่ในช่วงที่ดู
"สมเหตุสมผล" (ไม่ใช่ค่าที่โดดเด่นผิดปกติชัดเจนแบบตัวอย่างนี้) ความเสียหายอาจไม่ถูกพบเลยเป็นเวลานานมาก
นี่คือเหตุผลที่คำว่า "Silent" ถึงน่ากลัวกว่าข้อผิดพลาดที่ทำให้โปรแกรม Crash ทันที (เพราะอย่างน้อย Crash
ก็ทำให้รู้ทันทีว่ามีปัญหาเกิดขึ้น)

---

## ขั้นตอนที่ 857: กลยุทธ์การควบคุมเวอร์ชันของ Interface ระหว่างระบบ

### แนวคิด: เมื่อ "เพิ่มท้ายสุด" ไม่เพียงพออีกต่อไป

ขั้นตอนที่ 855 สอนกฎ "เพิ่มฟิลด์ท้ายสุด" สำหรับการเปลี่ยนแปลงเล็กน้อย แต่บางครั้งการเปลี่ยนแปลงมีขนาด
ใหญ่กว่านั้นมาก (เช่น เปลี่ยนทั้งโครงสร้างข้อความที่ส่งระหว่างระบบ) ในกรณีนี้ **การใส่ฟิลด์ Version ไว้
ที่ต้นข้อความอย่างชัดเจน** เป็นแนวทางที่นิยมใช้ในการสื่อสารระหว่างระบบ (System-to-System Interface) เพื่อ
ให้ผู้รับสามารถเลือกวิธีตีความข้อความให้ถูกต้องตามเวอร์ชันได้

### ซอร์สโค้ดฉบับเต็ม: MSGVER.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MSGVER.
       AUTHOR. COBOL-COURSE.
      *> Illustrates an explicit VERSION field at the front of an
      *> inter-system interface record, so the receiving side can
      *> dispatch to the right parsing rule as the format evolves.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LINE                  PIC X(30).

       01  WS-V01-LAYOUT REDEFINES WS-LINE.
           05  V1-VERSION           PIC X(2).
           05  V1-CUST-ID           PIC 9(5).
           05  V1-AMOUNT            PIC 9(7)V99.

       01  WS-V02-LAYOUT REDEFINES WS-LINE.
           05  V2-VERSION           PIC X(2).
           05  V2-CUST-ID           PIC 9(5).
           05  V2-AMOUNT            PIC 9(7)V99.
           05  V2-CURRENCY          PIC X(3).

       01  WS-VERSION-CHECK         PIC X(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "0110001000050000    " TO WS-LINE
           PERFORM DISPATCH-BY-VERSION

           MOVE "0210002000060000USD " TO WS-LINE
           PERFORM DISPATCH-BY-VERSION

           STOP RUN.

       DISPATCH-BY-VERSION.
           MOVE WS-LINE(1:2) TO WS-VERSION-CHECK
           EVALUATE WS-VERSION-CHECK
               WHEN "01"
                   DISPLAY "V1 MSG - CUST=" V1-CUST-ID
                       " AMOUNT=" V1-AMOUNT " (NO CURRENCY FIELD)"
               WHEN "02"
                   DISPLAY "V2 MSG - CUST=" V2-CUST-ID
                       " AMOUNT=" V2-AMOUNT " CCY=" V2-CURRENCY
               WHEN OTHER
                   DISPLAY "UNKNOWN VERSION: " WS-VERSION-CHECK
           END-EVALUATE.
```

คอมไพล์และรัน:

```bash
cobc -x -o msgver msgver.cob
./msgver
```

**ผลลัพธ์จริงที่ได้:**

```
V1 MSG - CUST=10001 AMOUNT=0000500.00 (NO CURRENCY FIELD)
V2 MSG - CUST=10002 AMOUNT=0000600.00 CCY=USD
```

### อธิบายจุดสำคัญ

- `WS-LINE(1:2)` ใช้เทคนิค **Reference Modification** (ทบทวนแนวคิดใกล้เคียงจาก Part 019-020 เรื่อง
  STRING/UNSTRING) เพื่ออ่าน 2 ตัวอักษรแรกของข้อความมาตรวจสอบเวอร์ชัน**ก่อน**ที่จะตัดสินใจว่าจะตีความ
  ส่วนที่เหลือด้วยโครงสร้างแบบใด
- `WS-V01-LAYOUT` และ `WS-V02-LAYOUT` เป็น `REDEFINES` ของ `WS-LINE` ตัวเดียวกัน (ทบทวนเทคนิคจาก Part
  022) ทำให้หน่วยความจำก้อนเดียวถูกตีความได้หลายแบบขึ้นอยู่กับเวอร์ชันที่ตรวจพบ
- `EVALUATE WS-VERSION-CHECK` ทำหน้าที่เป็น **Dispatcher**: เลือกวิธีตีความข้อมูลที่ถูกต้องตามเวอร์ชัน
  โดยอัตโนมัติ — นี่คือรูปแบบพื้นฐานที่ระบบ Enterprise Integration จริงใช้กันอย่างแพร่หลายเมื่อ Interface
  ระหว่างระบบต้องวิวัฒน์ไปตามเวลา

### เปรียบเทียบกับกฎ "เพิ่มฟิลด์ท้ายสุด" จากขั้นตอนที่ 855

| แนวทาง | เหมาะกับ | ข้อจำกัด |
|---|---|---|
| เพิ่มฟิลด์ท้ายสุด (Step 855) | การเปลี่ยนแปลงเล็กน้อย ผู้รับเดิมไม่จำเป็นต้องรู้จักฟิลด์ใหม่ | ผู้รับเดิมจะไม่มีวันเห็นข้อมูลใหม่จนกว่าจะอัปเดต |
| Version Field + Dispatch (Step 857) | การเปลี่ยนแปลงโครงสร้างใหญ่ หรือต้องการให้ผู้รับ**รู้ตัว**ว่ากำลังรับข้อมูลเวอร์ชันใด | ผู้รับต้องมีโค้ด Dispatch รองรับทุกเวอร์ชันที่อาจได้รับ (เพิ่มความซับซ้อนของโค้ดผู้รับ) |

### ข้อควรระวัง

- ระบบที่รับข้อความหลายเวอร์ชันพร้อมกัน (เช่นตัวอย่างนี้ที่รองรับทั้ง V1 และ V2) ต้อง**คงรองรับเวอร์ชันเก่า
  ไว้จนกว่าจะมั่นใจว่าไม่มีผู้ส่งรายใดส่งเวอร์ชันเก่าอีกแล้ว** — การหยุดรองรับเวอร์ชันเก่ากะทันหันเป็น
  Breaking Change ที่ร้ายแรงไม่ต่างจาก Anti-pattern ในขั้นตอนที่ 856
- ควรมีนโยบายชัดเจนว่าจะสนับสนุนกี่เวอร์ชันพร้อมกัน (เช่น "รองรับเวอร์ชันปัจจุบันและเวอร์ชันก่อนหน้า
  เท่านั้น") และมีกำหนดเวลาที่ชัดเจนสำหรับการเลิกสนับสนุนเวอร์ชันเก่า (Deprecation Timeline) พร้อมแจ้ง
  ให้ทุกระบบที่เกี่ยวข้องทราบล่วงหน้า

### แบบฝึกหัดที่ 857.1

**โจทย์**: จงออกแบบ `WS-V03-LAYOUT` ใหม่ที่เพิ่มฟิลด์ `V3-EXCHANGE-RATE PIC 9V9999` ต่อจาก
`V2-CURRENCY` แล้วเขียนเงื่อนไข `WHEN "03"` เพิ่มเติมในย่อหน้า `DISPATCH-BY-VERSION`

**เฉลยแนวทาง**:

```cobol
       01  WS-V03-LAYOUT REDEFINES WS-LINE.
           05  V3-VERSION           PIC X(2).
           05  V3-CUST-ID           PIC 9(5).
           05  V3-AMOUNT            PIC 9(7)V99.
           05  V3-CURRENCY          PIC X(3).
           05  V3-EXCHANGE-RATE     PIC 9V9999.
```

และเพิ่มใน `EVALUATE`:

```cobol
               WHEN "03"
                   DISPLAY "V3 MSG - CUST=" V3-CUST-ID
                       " AMOUNT=" V3-AMOUNT " CCY=" V3-CURRENCY
                       " RATE=" V3-EXCHANGE-RATE
```

สังเกตว่า `V3-EXCHANGE-RATE` ถูกเพิ่มต่อจากฟิลด์เดิมทั้งหมดของ V2 (ไม่ได้แทรกกลาง) ตรงตามกฎทองข้อที่ 1
จากขั้นตอนที่ 855 ด้วยเช่นกัน — แนวทาง Version Field และกฎ "เพิ่มท้ายสุด" มักถูกใช้ร่วมกันในทางปฏิบัติ
จริง ไม่ใช่แยกจากกันโดยสิ้นเชิง

---

## ขั้นตอนที่ 858: ภาพรวม Enterprise Integration Patterns และการเชื่อมต่อระหว่างระบบ

### แนวคิด: ระบบองค์กรขนาดใหญ่ไม่เคยทำงานอย่างโดดเดี่ยว

ระบบ COBOL ในองค์กรขนาดใหญ่แทบไม่เคยทำงานเพียงลำพัง — มันต้องรับส่งข้อมูลกับระบบอื่นอยู่ตลอดเวลา:
รับไฟล์จากธนาคารพันธมิตร, ส่งรายงานให้หน่วยงานกำกับดูแล, รับคำสั่งซื้อจากเว็บไซต์ ฯลฯ รูปแบบการเชื่อมต่อ
ที่พบบ่อยที่สุดในโลก COBOL/Mainframe มีดังนี้:

| รูปแบบ | คำอธิบาย | ตัวอย่างที่เคยเรียนในหลักสูตร |
|---|---|---|
| **File Transfer** | ส่ง/รับไฟล์แบบ Batch เป็นระยะ (เช่นทุกคืน) | Part 023-030 (Sequential/Indexed Files) |
| **Database Sharing** | หลายระบบเข้าถึงฐานข้อมูลร่วมกัน | Part 057-060 (DB2) |
| **Synchronous API Call** | เรียกใช้งานแบบทันที รอผลลัพธ์กลับ | Part 073 (REST API), Part 072 (CALL ข้ามภาษา) |
| **Asynchronous Messaging** | ส่งข้อความ/เหตุการณ์โดยไม่ต้องรอผลลัพธ์ทันที | จะสอนละเอียดใน **Part 087** |

### ทำไม File Transfer ถึงยังคงเป็นรูปแบบที่พบบ่อยที่สุดในโลก Mainframe

แม้จะดูเหมือนเทคโนโลยีเก่า แต่ **File Transfer แบบ Batch** ยังคงเป็นรูปแบบการเชื่อมต่อที่ใช้กันแพร่หลาย
ที่สุดในอุตสาหกรรมการเงินจริง ด้วยเหตุผล:

1. **เหมาะกับปริมาณข้อมูลมหาศาล** — การส่งไฟล์ที่มีธุรกรรมนับล้านรายการครั้งเดียวมีประสิทธิภาพกว่าการ
   เรียก API ทีละรายการมาก
2. **ทนทานต่อความล้มเหลว** — หากระบบปลายทางล่มชั่วคราว ไฟล์ยังคงอยู่รอได้ ต่างจาก Synchronous API ที่
   ต้องมีทั้งสองฝั่งพร้อมทำงานพร้อมกัน
3. **ตรวจสอบย้อนหลังได้ง่าย** — ไฟล์ที่ส่งไปแล้วเก็บไว้เป็นหลักฐาน (audit trail) ได้ตามธรรมชาติ

### เชื่อมโยงสู่ Part 087: เมื่อ Batch File Transfer ไม่เพียงพออีกต่อไป

อย่างไรก็ตาม โลกสมัยใหม่ต้องการการตอบสนองที่**เร็วกว่า**รอบ Batch รายคืน — ระบบ E-commerce ต้องการรู้
ผลลัพธ์การอนุมัติเครดิตภายในไม่กี่วินาที ไม่ใช่รอถึงคืนถัดไป นี่คือจุดที่ **Asynchronous Messaging** (การ
ส่งข้อความ/เหตุการณ์แบบไม่ประสาน) เข้ามามีบทบาท — **Part 087** จะสอนว่า COBOL สามารถมีส่วนร่วมในสถาปัตยกรรม
แบบ Microservices/Event-driven ได้อย่างไร แม้จะไม่มี Message Broker จริง (Kafka/RabbitMQ) ในสภาพแวดล้อม
การเรียนรู้นี้ก็ตาม โดยใช้ไฟล์เป็นตัวแทนของ Message Queue อย่างตรงไปตรงมา

### ข้อควรระวัง

- อย่ามองว่า File Transfer เป็น "เทคโนโลยีล้าสมัยที่ควรเลิกใช้ทั้งหมด" — ในบริบทการประมวลผลข้อมูลปริมาณ
  มากแบบ Batch มันยังคงเป็นทางเลือกที่เหมาะสมที่สุดในหลายกรณี การเลือกรูปแบบการเชื่อมต่อควรพิจารณาจาก
  ลักษณะงานจริง ไม่ใช่ตามกระแสความนิยมของเทคโนโลยี
- การผสมผสานหลายรูปแบบในระบบเดียวกันเป็นเรื่องปกติมากในองค์กรจริง (เช่น รับคำสั่งซื้อผ่าน REST API
  แบบ Synchronous แต่ประมวลผลการจัดส่งแบบ Batch File Transfer ทุกคืน) ไม่จำเป็นต้องเลือกใช้แบบใดแบบหนึ่ง
  เพียงอย่างเดียวทั้งระบบ

### แบบฝึกหัดที่ 858.1

**โจทย์**: จงยกตัวอย่างสถานการณ์ทางธุรกิจ 1 กรณีที่ **File Transfer แบบ Batch** เหมาะสมกว่า
**Synchronous API Call** และอธิบายเหตุผล

**เฉลยแนวทาง**: ตัวอย่างเช่น การส่งรายงานสรุปธุรกรรมประจำวันของธนาคารให้หน่วยงานกำกับดูแล (Regulator)
เพื่อการตรวจสอบ — งานนี้ไม่ต้องการการตอบสนองทันที (หน่วยงานกำกับดูแลไม่ได้รอผลลัพธ์แบบเรียลไทม์) แต่
ต้องการความแม่นยำและครบถ้วนของข้อมูลทั้งหมดในรอบวันนั้น การส่งเป็นไฟล์ Batch ครั้งเดียวหลังปิดวันทำการ
จึงเหมาะสมกว่าการเรียก API ทีละธุรกรรมมาก เพราะไฟล์ Batch ให้ภาพรวมที่สมบูรณ์ ตรวจสอบง่าย และไม่สร้าง
ภาระด้าน Network Traffic ระหว่างวันทำการซึ่งควรสงวนไว้สำหรับธุรกรรมของลูกค้าจริงที่ต้องการความเร็ว

---

## ขั้นตอนที่ 859: กระบวนการกำกับดูแลและการจัดการเอกสารระดับองค์กร

### แนวคิด: เทคนิคดีแค่ไหนก็ไร้ผลถ้าไม่มีกระบวนการรองรับ

ทุกเทคนิคที่สอนมาใน Part นี้ (Layered Architecture, Batch Scheduling, Copybook Governance, Interface
Versioning) จะไม่มีประโยชน์อะไรเลยหากองค์กรไม่มี **กระบวนการ (Process)** ที่บังคับใช้มันอย่างสม่ำเสมอ
ขั้นตอนนี้จะสรุปกระบวนการกำกับดูแลหลักที่องค์กร Mainframe ขนาดใหญ่ใช้กันจริง

### กระบวนการที่ 1: Change Management (การจัดการการเปลี่ยนแปลง)

ทุกการเปลี่ยนแปลงต่อระบบที่ใช้งานจริง (โดยเฉพาะ Copybook ที่ใช้ร่วมกัน ตามขั้นตอนที่ 855-856) ต้องผ่าน
กระบวนการอนุมัติที่เป็นทางการ ซึ่งโดยทั่วไปประกอบด้วย:

1. **Request for Change (RFC)** — บันทึกคำขอเปลี่ยนแปลงพร้อมเหตุผลและผลกระทบที่คาดว่าจะเกิดขึ้น
2. **Impact Analysis** — วิเคราะห์ว่าการเปลี่ยนแปลงจะกระทบโปรแกรม/ทีมใดบ้าง (ใช้เทคนิค Cross-Reference
   จาก Part 083 ขั้นตอนที่ 826 ช่วยค้นหาผู้ใช้ Copybook ทั้งหมด)
3. **Approval** — ผู้มีอำนาจ (มักเป็นคณะกรรมการ Change Advisory Board หรือ CAB) อนุมัติก่อนดำเนินการ
4. **Scheduled Deployment** — นำการเปลี่ยนแปลงไปใช้ตามช่วงเวลาที่กำหนดไว้ล่วงหน้า (มักเป็นช่วงที่มี
   Traffic ต่ำ) พร้อมแผนย้อนกลับ (Rollback Plan) หากเกิดปัญหา

### กระบวนการที่ 2: Impact Analysis ด้วยเครื่องมือ

Impact Analysis ที่ดีต้องตอบคำถาม "ถ้าแก้ Copybook นี้ โปรแกรมใดบ้างจะได้รับผลกระทบ" ได้อย่างแม่นยำ
วิธีพื้นฐานที่สุด (ต่อยอดจากเทคนิค Part 083) คือค้นหาทุกโปรแกรมที่มีคำสั่ง `COPY` อ้างอิงถึงไฟล์นั้น:

```bash
grep -rl 'COPY "CUSTREC' /path/to/all/cobol/programs/
```

คำสั่งนี้ (แนวคิดเดียวกับที่ Part 083 ขั้นตอนที่ 826 สอนไว้) จะแสดงรายชื่อไฟล์ COBOL ทุกไฟล์ที่ `COPY`
Copybook ที่กำลังจะถูกแก้ไข — ในองค์กรขนาดใหญ่มักมีเครื่องมือ Enterprise เฉพาะทางที่ทำสิ่งนี้ได้ละเอียด
กว่านี้มาก (รวมถึงตรวจสอบ Dynamic CALL และ Copybook ที่ซ้อนกันหลายชั้น) แต่หลักการพื้นฐานเหมือนกันทุก
ประการ

### กระบวนการที่ 3: มาตรฐานการเขียนเอกสาร (Documentation Standards)

องค์กรขนาดใหญ่ควรมีมาตรฐานเอกสารที่สม่ำเสมอสำหรับทุกระบบ เพื่อไม่ให้เกิดปัญหาแบบที่ Part 083 เผชิญ
(ระบบ Legacy ที่ไม่มีเอกสารเหลืออยู่เลย) มาตรฐานที่ดีควรครอบคลุม:

- **Architecture Decision Records (ADR)** — บันทึกการตัดสินใจเชิงสถาปัตยกรรมทุกครั้ง (ตามรูปแบบที่
  Part 050 และ Part 085 สาธิตไว้)
- **Copybook Registry** — ทะเบียนกลางที่บันทึกว่า Copybook แต่ละไฟล์มีเวอร์ชันอะไร ใครเป็นเจ้าของ
  (owner) และโปรแกรมใดบ้างที่ใช้งาน
- **Runbook** — คู่มือขั้นตอนการแก้ไขปัญหาเฉพาะหน้าสำหรับทีมปฏิบัติการ (Operations) เมื่อระบบมีปัญหา
  นอกเวลาทำการ

### ข้อควรระวัง

- กระบวนการที่เข้มงวดเกินไปอาจทำให้ทีมพัฒนาทำงานช้าลงโดยไม่จำเป็น — องค์กรที่ดีต้องหาสมดุลระหว่าง
  **ความปลอดภัย** (ป้องกันความเสียหายแบบขั้นตอนที่ 856) กับ **ความคล่องตัว** (ไม่ให้กระบวนการอนุมัติ
  กลายเป็นคอขวดที่ทำให้ทีมทำงานไม่ทัน) — นี่คือเหตุผลที่หลายองค์กรแยกระดับความเข้มงวดของกระบวนการตาม
  ความเสี่ยงของการเปลี่ยนแปลง (เช่น การแก้ Comment ไม่ต้องผ่าน CAB แต่การแก้โครงสร้าง Copybook ที่ใช้
  ร่วมกัน 100 โปรแกรมต้องผ่าน)
- เอกสารที่ดีต้องถูก**ปรับปรุงอย่างต่อเนื่อง** ไม่ใช่เขียนครั้งเดียวแล้วปล่อยทิ้งไว้จนล้าสมัย (ทบทวนจาก
  Part 083 ขั้นตอนที่ 829 ที่เน้นย้ำว่าเอกสารสรุปการวิเคราะห์ควรเป็น "living document")

### แบบฝึกหัดที่ 859.1

**โจทย์**: จงอธิบายว่าทำไม Impact Analysis (กระบวนการที่ 2) ถึงควรทำ**ก่อน**ขอ Approval (กระบวนการที่ 1
ส่วนที่ 3) ไม่ใช่ทำหลังจากได้รับอนุมัติแล้ว

**เฉลยแนวทาง**: เพราะผู้มีอำนาจอนุมัติ (Change Advisory Board) ต้องมีข้อมูลผลกระทบที่ครบถ้วนประกอบการ
ตัดสินใจ หากอนุมัติไปก่อนโดยไม่รู้ขอบเขตผลกระทบที่แท้จริง อาจนำไปสู่การอนุมัติการเปลี่ยนแปลงที่ดูเหมือน
ไม่มีความเสี่ยง (เช่น "แค่เพิ่มฟิลด์เดียว") ทั้งที่จริง ๆ แล้วอาจกระทบโปรแกรมนับร้อยตัวที่ไม่มีใครคาดคิด
มาก่อน การทำ Impact Analysis ก่อนเสมอทำให้กระบวนการอนุมัติมีข้อมูลที่ถูกต้องครบถ้วน และช่วยให้วางแผน
Scheduled Deployment (กระบวนการที่ 4) ได้อย่างเหมาะสม เช่น รู้ว่าต้องแจ้งทีมใดบ้างล่วงหน้า และควร
เลือกช่วงเวลาใดที่ปลอดภัยที่สุดสำหรับการเปลี่ยนแปลงนี้

---

## ขั้นตอนที่ 860: สรุป Checklist สถาปัตยกรรมองค์กรและก้าวต่อไป

### Checklist สรุปแนวคิดทั้งหมดของ Part นี้

| ลำดับ | รายการตรวจสอบ | อ้างอิงขั้นตอน |
|---|---|---|
| 1 | ระบบแบ่งเป็น Layer ที่ชัดเจน (Presentation/Business/Data Access) และการพึ่งพาไหลทิศทางเดียว | 852 |
| 2 | มี Job Dependency Graph ที่ชัดเจนสำหรับงาน Batch ที่ต้องรันตามลำดับ | 853 |
| 3 | ใช้ Condition Code Propagation อย่างสม่ำเสมอเพื่อให้ Scheduler ตัดสินใจได้ถูกต้อง | 853 |
| 4 | งาน Batch ขนาดใหญ่มีกลไก Checkpoint/Restart รองรับความล้มเหลวกลางคัน | 854 |
| 5 | Copybook ที่ใช้ร่วมกันมีกฎการวิวัฒน์ที่ชัดเจน (เพิ่มฟิลด์ท้ายสุดเท่านั้น) | 855-856 |
| 6 | Interface ระหว่างระบบมีกลยุทธ์ควบคุมเวอร์ชันที่รองรับการเปลี่ยนแปลงโครงสร้างใหญ่ได้ | 857 |
| 7 | เข้าใจรูปแบบการเชื่อมต่อระหว่างระบบที่หลากหลาย และเลือกใช้ให้เหมาะกับลักษณะงาน | 858 |
| 8 | มีกระบวนการ Change Management, Impact Analysis, และมาตรฐานเอกสารที่ชัดเจน | 859 |

### สรุปภาพรวม: จาก Part 085 สู่ Part 086

Part 085 แก้ปัญหาระดับ**หนึ่งระบบ**อย่างสมบูรณ์ — Part 086 ขยายมุมมองไปสู่ **ความท้าทายเมื่อมีหลายระบบ
ทำงานร่วมกันในองค์กรขนาดใหญ่**:

```
Part 085: ONE SYSTEM                    Part 086: MANY SYSTEMS
+-------------------+                   +-------------------+
| ARCALC/ARDISC/    |                   | ระบบ A, B, C, ... Z |
| ARLEDGER          |     ขยายสเกล      | ที่ใช้ Copybook     |
| (3 โปรแกรม         |  ------------->  | ร่วมกัน, รัน Batch  |
|  ทำงานร่วมกัน)      |                   | ตามตาราง, สื่อสาร    |
|                    |                   | กันข้ามระบบ          |
+-------------------+                   +-------------------+
```

หลักการที่เรียนรู้ใน Part 085 (Layered Architecture, Golden Master Testing, Unit Testing, CI/CD) ยังคง
ใช้ได้กับทุกระบบใน Part 086 — เพียงแต่ต้องเพิ่มเติมกระบวนการและวินัยที่รองรับการทำงานร่วมกันของหลายระบบ
หลายทีมพร้อมกัน

### เชื่อมโยงสู่ Part 087

Part 086 กล่าวถึง **Asynchronous Messaging** ไว้เพียงสั้น ๆ ในขั้นตอนที่ 858 ว่าเป็นทางเลือกเมื่อ Batch
File Transfer ไม่เพียงพออีกต่อไป — **Part 087: Microservices และ COBOL ในระบบกระจาย** จะลงรายละเอียด
เรื่องนี้อย่างเต็มรูปแบบ: COBOL จะทำหน้าที่เป็น Bounded Context Service หลัง API Gateway ได้อย่างไร,
รูปแบบ Event-driven ที่ใช้ไฟล์เป็นตัวแทน Message Queue, และแนวคิด Idempotency/Retry ที่สำคัญมากสำหรับ
การเชื่อมต่อระหว่างงาน Batch กับ Microservices สมัยใหม่

### ข้อควรระวัง

- แนวคิดใน Part นี้เป็นภาพรวมระดับสถาปัตยกรรม เหมาะสำหรับสร้างความเข้าใจพื้นฐานก่อนเจาะลึกรายละเอียด
  เชิงเทคนิคเพิ่มเติมใน Part ถัดไป — ในสถานการณ์จริง แต่ละองค์กรอาจมีรายละเอียดกระบวนการที่แตกต่างกันไป
  ตามวัฒนธรรมองค์กรและเครื่องมือที่ใช้
- อย่าลืมว่าเป้าหมายสูงสุดของสถาปัตยกรรมองค์กรที่ดีไม่ใช่ "กฎระเบียบที่ซับซ้อนที่สุด" แต่คือ **การทำให้
  ระบบจำนวนมากทำงานร่วมกันได้อย่างน่าเชื่อถือ ในขณะที่ทีมงานยังคงพัฒนาต่อยอดได้อย่างมั่นใจ**

### แบบฝึกหัดที่ 860.1

**โจทย์**: จากทั้ง 4 เสาหลักของ Part นี้ (Layered Architecture, Batch Scheduling, Copybook Governance,
Interface Versioning) จงเลือกมา 1 เรื่องที่คุณคิดว่าสำคัญที่สุดสำหรับองค์กรที่เพิ่งเริ่มขยายจากระบบ
COBOL เดี่ยว ๆ ไปสู่หลายระบบ พร้อมให้เหตุผลประกอบ

**เฉลยแนวทาง**: ไม่มีคำตอบตายตัว ขึ้นกับบริบทขององค์กร แต่เหตุผลที่ดีควรเชื่อมโยงกับความเสี่ยงเฉพาะหน้า
ที่องค์กรนั้นเผชิญ เช่น หากองค์กรมีหลายทีมที่เริ่มใช้ Copybook ร่วมกันแล้วแต่ยังไม่มีกระบวนการควบคุม
Copybook Governance (ขั้นตอนที่ 855-856) มักเป็นเรื่องเร่งด่วนที่สุด เพราะความเสียหายแบบ Silent
Corruption ที่พิสูจน์ไว้ในขั้นตอนที่ 856 สามารถเกิดขึ้นได้ทันทีโดยไม่มีการเตือนล่วงหน้าใด ๆ เลย ในขณะที่
ปัญหาด้าน Batch Scheduling หรือ Interface Versioning มักแสดงอาการผิดปกติที่สังเกตเห็นได้ชัดเจนกว่า (เช่น
งานล้มเหลวทันที) ทำให้มีเวลาตอบสนองมากกว่า

---

## สรุปท้ายบท

Part 086 คือจุดเริ่มต้นของเฟส 6 "ระดับมืออาชีพ/โลก" — เราได้ขยายมุมมองจากการปรับปรุง**หนึ่งระบบ**ใน
Part 085 ไปสู่ความท้าทายของการดูแล**หลายระบบในองค์กรขนาดใหญ่**:

- ภาพรวมความท้าทาย 4 เสาหลักของสถาปัตยกรรมองค์กร และเหตุผลที่ปัญหาเล็กในระบบเล็กกลายเป็นความเสี่ยงใหญ่
  เมื่อขยายสเกล (ขั้นตอนที่ 851)
- Layered Architecture ในบริบท COBOL พร้อมการทบทวน AR-MINI จาก Part 085 เป็นตัวอย่างจริง (ขั้นตอนที่ 852)
- Batch Scheduling Architecture: Job Dependency Graph และ Condition Code Propagation ที่ทดสอบได้จริง
  ผ่าน `JOBCTL.cob` (ขั้นตอนที่ 853)
- รูปแบบ Checkpoint/Restart สำหรับงาน Batch ขนาดใหญ่ พิสูจน์การกู้คืนจากความล้มเหลวจริงด้วย
  `BATCHRUN.cob` (ขั้นตอนที่ 854)
- กฎทองของ Copybook Governance: เพิ่มฟิลด์ท้ายสุดเท่านั้น พิสูจน์ด้วยการทดสอบ Writer/Reader คนละเวอร์ชัน
  จริง (ขั้นตอนที่ 855)
- Anti-Pattern ที่อันตรายที่สุด: การแทรกฟิลด์กลางโครงสร้าง พิสูจน์ Silent Data Corruption ที่เกิดขึ้นจริง
  โดยไม่มี Error เตือน (ขั้นตอนที่ 856)
- กลยุทธ์ Interface Versioning ด้วย Version Field และ Dispatch Pattern ที่ทดสอบได้จริงผ่าน `MSGVER.cob`
  (ขั้นตอนที่ 857)
- ภาพรวม Enterprise Integration Patterns และเหตุผลที่ File Transfer ยังคงสำคัญในโลก Mainframe จริง
  (ขั้นตอนที่ 858)
- กระบวนการกำกับดูแล: Change Management, Impact Analysis, และมาตรฐานเอกสารระดับองค์กร (ขั้นตอนที่ 859)
- สรุป Checklist ทั้งหมดและเชื่อมโยงสู่ Part ถัดไป (ขั้นตอนที่ 860)

ทักษะเหล่านี้เป็นพื้นฐานสำคัญสำหรับ **Part 087: Microservices และ COBOL ในระบบกระจาย** ซึ่งจะลงลึกใน
เรื่อง Asynchronous Messaging ที่ Part นี้เพียงแค่แตะผิวไว้ในขั้นตอนที่ 858 พร้อมตัวอย่างการสร้างระบบ
"คิว" แบบไฟล์ที่ทำงานได้จริงระหว่าง Producer และ Consumer

**[ไปยัง Part 087: Microservices และ COBOL ในระบบกระจาย →](part-087-microservices-cobol.md)**
