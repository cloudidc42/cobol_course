# โครงสร้างหลักสูตร COBOL แบบละเอียด (Course Outline)

เอกสารนี้คือ "แผนที่หลักสูตร" ทั้งหมด 100 Part / 1000 Steps ใช้เป็นแหล่งอ้างอิงเดียว (single source of truth)
สำหรับลำดับเนื้อหา ชื่อไฟล์ และขอบเขตของแต่ละ Part เพื่อให้การเขียนเนื้อหาแต่ละส่วนสอดคล้องกันทั้งหลักสูตร

รูปแบบชื่อไฟล์: `parts/part-XXX-slug-ภาษาอังกฤษสั้นๆ.md` (XXX = เลข Part แบบเติมศูนย์ 3 หลัก)

สถานะ: ✅ = เขียนเสร็จแล้ว, 🚧 = กำลังเขียน, ⬜ = ยังไม่เริ่ม (ดูสถานะสดที่ `docs/PROGRESS.md`)

---

## เฟส 1: พื้นฐาน (Foundations) — Parts 001–015 — Steps 1–150

| Part | Steps | ชื่อเรื่อง |
|---|---|---|
| 001 | 1–10 | ประวัติศาสตร์และโลกของ COBOL, ทำไมต้องเรียน COBOL ในปี 2026, ภาพรวมอุตสาหกรรม |
| 002 | 11–20 | ติดตั้งสภาพแวดล้อมพัฒนา: GnuCOBOL, VS Code, การคอมไพล์โปรแกรมแรก |
| 003 | 21–30 | โครงสร้างโปรแกรม COBOL: 4 Divisions, กฎการเขียนคอลัมน์ (Column Rules) |
| 004 | 31–40 | IDENTIFICATION DIVISION และ ENVIRONMENT DIVISION แบบละเอียด |
| 005 | 41–50 | DATA DIVISION เบื้องต้น: WORKING-STORAGE SECTION |
| 006 | 51–60 | PICTURE Clause และชนิดข้อมูลตัวเลข/ตัวอักษร (Numeric/Alphanumeric) |
| 007 | 61–70 | DISPLAY และ ACCEPT: คำสั่ง Input/Output พื้นฐาน |
| 008 | 71–80 | MOVE Statement และกฎการย้ายข้อมูล (MOVE Rules) |
| 009 | 81–90 | เลขคณิตพื้นฐาน: ADD, SUBTRACT, MULTIPLY, DIVIDE, COMPUTE |
| 010 | 91–100 | เงื่อนไข IF-ELSE และ Condition Names (88-level) |
| 011 | 101–110 | EVALUATE Statement (เทียบเท่า switch/case) |
| 012 | 111–120 | PERFORM พื้นฐานและ PERFORM UNTIL (ลูป) |
| 013 | 121–130 | PERFORM VARYING และการวนลูปหลายชั้น (Nested Loops) |
| 014 | 131–140 | โครงสร้างโปรแกรมแบบมีโมดูล: Paragraph, Section, PERFORM...THRU |
| 015 | 141–150 | 🎯 โปรเจกต์เฟส 1: เครื่องคิดเลขและระบบคำนวณเกรดนักเรียน |

## เฟส 2: ระดับกลาง (Intermediate) — Parts 016–035 — Steps 151–350

| Part | Steps | ชื่อเรื่อง |
|---|---|---|
| 016 | 151–160 | ตาราง (Tables) และ OCCURS Clause |
| 017 | 161–170 | Subscript กับ Index และ SEARCH / SEARCH ALL |
| 018 | 171–180 | ตารางหลายมิติ (Multi-dimensional Tables) |
| 019 | 181–190 | การจัดการข้อความ: STRING Statement |
| 020 | 191–200 | การจัดการข้อความ: UNSTRING Statement |
| 021 | 201–210 | INSPECT Statement: นับ แทนที่ แปลงตัวอักษร |
| 022 | 211–220 | REDEFINES Clause และการใช้หน่วยความจำซ้ำ |
| 023 | 221–230 | ไฟล์ตามลำดับ (Sequential Files) เบื้องต้น |
| 024 | 231–240 | FILE SECTION และ FD Entry แบบละเอียด |
| 025 | 241–250 | OPEN, CLOSE, READ, WRITE, REWRITE, DELETE |
| 026 | 251–260 | การประมวลผลไฟล์แบบ Master-Detail (Matching Records) |
| 027 | 261–270 | SORT และ MERGE Statement |
| 028 | 271–280 | Indexed Files และ ISAM |
| 029 | 281–290 | Relative Files |
| 030 | 291–300 | File Status Codes และการจัดการข้อผิดพลาดของไฟล์ |
| 031 | 301–310 | Subprograms: CALL Statement เบื้องต้น |
| 032 | 311–320 | การส่งพารามิเตอร์: BY REFERENCE, BY CONTENT, BY VALUE |
| 033 | 321–330 | COPY Statement และการใช้ Copybooks |
| 034 | 331–340 | Nested Programs และ END PROGRAM |
| 035 | 341–350 | 🎯 โปรเจกต์เฟส 2: ระบบจัดการสินค้าคงคลัง (Inventory Management System) |

## เฟส 3: ขั้นสูง (Advanced) — Parts 036–050 — Steps 351–500

| Part | Steps | ชื่อเรื่อง |
|---|---|---|
| 036 | 351–360 | Intrinsic Functions: ตัวเลขและวันที่ (FUNCTION) |
| 037 | 361–370 | Intrinsic Functions: ข้อความและสถิติ |
| 038 | 371–380 | Report Writer Feature เบื้องต้น |
| 039 | 381–390 | Report Writer ขั้นสูง: Control Breaks |
| 040 | 391–400 | Screen Section: สร้าง UI แบบ Text-mode |
| 041 | 401–410 | Advanced EVALUATE และเงื่อนไขซับซ้อน |
| 042 | 411–420 | Class Condition และ Sign Condition |
| 043 | 421–430 | Dynamic CALL และ Program Pointers |
| 044 | 431–440 | Based Storage และ POINTER Data Item |
| 045 | 441–450 | JSON GENERATE และ JSON PARSE ใน COBOL สมัยใหม่ |
| 046 | 451–460 | XML GENERATE และ XML PARSE |
| 047 | 461–470 | การจัดการวันที่และเวลาขั้นสูง |
| 048 | 471–480 | Error Handling ขั้นสูง: USE Statement และ DECLARATIVES |
| 049 | 481–490 | Compiler Directives และความแตกต่างระหว่าง COBOL Dialects |
| 050 | 491–500 | 🎯 โปรเจกต์เฟส 3: ระบบบัญชีลูกหนี้ (Accounts Receivable System) |

## เฟส 4: Mainframe / Enterprise — Parts 051–070 — Steps 501–700

| Part | Steps | ชื่อเรื่อง |
|---|---|---|
| 051 | 501–510 | แนะนำ Mainframe และ z/OS Ecosystem |
| 052 | 511–520 | JCL เบื้องต้น: JOB, EXEC, DD Statement |
| 053 | 521–530 | JCL ขั้นสูง: Procedures และ Condition Codes |
| 054 | 531–540 | TSO/ISPF: การใช้งานเบื้องต้น |
| 055 | 541–550 | VSAM เบื้องต้น: KSDS, ESDS, RRDS |
| 056 | 551–560 | VSAM ขั้นสูง: Alternate Index และ Cluster |
| 057 | 561–570 | DB2 และ SQL เบื้องต้นสำหรับ COBOL |
| 058 | 571–580 | Embedded SQL: EXEC SQL ใน COBOL |
| 059 | 581–590 | DB2 Cursor และการประมวลผลหลายแถว |
| 060 | 591–600 | DB2 Stored Procedures ด้วย COBOL |
| 061 | 601–610 | CICS เบื้องต้น: Transaction Processing |
| 062 | 611–620 | CICS Commands: SEND, RECEIVE, MAP |
| 063 | 621–630 | CICS BMS Maps และการออกแบบหน้าจอ |
| 064 | 631–640 | CICS ขั้นสูง: Pseudo-conversational Programming |
| 065 | 641–650 | IMS DB/DC เบื้องต้น |
| 066 | 651–660 | Batch Processing Patterns และ Job Scheduling |
| 067 | 661–670 | DFSORT และ Sort Utilities ขั้นสูง |
| 068 | 671–680 | Performance Tuning สำหรับ Mainframe COBOL |
| 069 | 681–690 | Security บน Mainframe (RACF Concepts) |
| 070 | 691–700 | 🎯 โปรเจกต์เฟส 4: ระบบธนาคารบน Mainframe จำลอง |

## เฟส 5: COBOL สมัยใหม่ (Modern) — Parts 071–085 — Steps 701–850

| Part | Steps | ชื่อเรื่อง |
|---|---|---|
| 071 | 701–710 | GnuCOBOL สมัยใหม่และ Open-source COBOL Ecosystem |
| 072 | 711–720 | การเชื่อมต่อ COBOL กับภาษาอื่น (C, Java) ผ่าน CALL |
| 073 | 721–730 | COBOL และ REST API: การสร้าง Wrapper Service |
| 074 | 731–740 | COBOL และ Web Application: เชื่อมกับ Node.js/Python |
| 075 | 741–750 | ฐานข้อมูลสมัยใหม่: COBOL กับ MySQL/PostgreSQL |
| 076 | 751–760 | Containerizing COBOL ด้วย Docker |
| 077 | 761–770 | COBOL บน Cloud: แนวคิด Mainframe Modernization |
| 078 | 771–780 | CI/CD Pipeline สำหรับโปรเจกต์ COBOL |
| 079 | 781–790 | Unit Testing สำหรับ COBOL |
| 080 | 791–800 | Version Control (Git) และ Workflow สำหรับทีม COBOL |
| 081 | 801–810 | Code Refactoring และ Clean Code สำหรับ COBOL |
| 082 | 811–820 | Design Patterns ในโลก COBOL |
| 083 | 821–830 | Legacy System Analysis และ Reverse Engineering |
| 084 | 831–840 | Migration Strategies: COBOL to Java/C# |
| 085 | 841–850 | 🎯 โปรเจกต์เฟส 5: ปรับปรุงระบบ Legacy ด้วยสถาปัตยกรรมสมัยใหม่ |

## เฟส 6: ระดับมืออาชีพ/โลก (Professional) — Parts 086–100 — Steps 851–1000

| Part | Steps | ชื่อเรื่อง |
|---|---|---|
| 086 | 851–860 | สถาปัตยกรรมระบบองค์กรขนาดใหญ่ด้วย COBOL |
| 087 | 861–870 | Microservices และ COBOL ในระบบกระจาย |
| 088 | 871–880 | Performance Optimization ระดับสูง |
| 089 | 881–890 | Security ระดับ Enterprise สำหรับแอปพลิเคชัน COBOL |
| 090 | 891–900 | การจัดการโปรเจกต์ COBOL ขนาดใหญ่ (Project Management) |
| 091 | 901–910 | Case Study — Core Banking ตอนที่ 1: การออกแบบระบบ |
| 092 | 911–920 | Case Study — Core Banking ตอนที่ 2: โมดูลบัญชีและธุรกรรม |
| 093 | 921–930 | Case Study — Core Banking ตอนที่ 3: การรายงานและปิดบัญชี |
| 094 | 931–940 | Case Study — ระบบประกันภัย (Insurance System) |
| 095 | 941–950 | Case Study — ระบบ ERP และห่วงโซ่อุปทาน |
| 096 | 951–960 | การเตรียมตัวสอบใบรับรอง COBOL และแนวทางอาชีพ |
| 097 | 961–970 | แนวข้อสอบสัมภาษณ์งาน COBOL Developer |
| 098 | 971–980 | Best Practices และมาตรฐานการเขียนโค้ดระดับโลก |
| 099 | 981–990 | อนาคตของ COBOL และเทรนด์เทคโนโลยี |
| 100 | 991–1000 | 🎯 Capstone Project สุดท้าย, บทสรุปหลักสูตร, และแหล่งเรียนรู้ต่อ |

---

## มาตรฐานการเขียนเนื้อหา (Content Standard) — ใช้กับทุก Part

ทุกไฟล์ `parts/part-XXX-*.md` ต้องมีโครงสร้างดังนี้:

1. **หัวเรื่อง**: `# Part XXX: ชื่อเรื่อง (ขั้นตอนที่ A–B)`
2. **คำนำของ Part**: อธิบายภาพรวมว่าจะได้เรียนอะไรบ้าง เชื่อมโยงกับ Part ก่อนหน้า
3. **เนื้อหาแบ่งตามขั้นตอน**: `## ขั้นตอนที่ N: หัวข้อย่อย` ครบทั้ง 10 ขั้นตอนต่อ Part โดยแต่ละขั้นตอนต้องมี:
   - คำอธิบายแนวคิด (concept) เป็นภาษาไทยที่เข้าใจง่าย
   - โค้ดตัวอย่างที่ **คอมไพล์และรันได้จริง** ด้วย GnuCOBOL (ใช้ fixed-format, คอลัมน์ 8–72 เป็นมาตรฐาน)
   - คำอธิบายโค้ดทีละส่วน/ทีละบรรทัดในจุดที่สำคัญ
   - ผลลัพธ์ที่คาดว่าจะได้ (sample output)
   - ข้อควรระวัง/ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)
   - แบบฝึกหัดสั้น 1 ข้อ พร้อมเฉลย
4. **สรุปท้ายบท**: สรุปสิ่งที่เรียนรู้ และลิงก์ไปยัง Part ถัดไป
5. **ความยาว**: 500–3000+ บรรทัดต่อไฟล์ (ขึ้นกับความซับซ้อนของหัวข้อ)
6. **Milestone Parts** (015, 035, 050, 070, 085, 100) เป็นโปรเจกต์รวบยอด: มีการออกแบบระบบ, Full source code แบ่งเป็นไฟล์ย่อยที่ประกอบกันเป็นระบบทำงานได้จริง, และคำอธิบายสถาปัตยกรรม

## หมายเหตุเรื่องมาตรฐาน COBOL ที่ใช้

- ใช้ **GnuCOBOL 3.x** เป็นคอมไพเลอร์อ้างอิงหลัก เนื่องจากเป็น Open Source ฟรี ติดตั้งง่าย ใช้ได้ทุกแพลตฟอร์ม
- ใช้รูปแบบ **Fixed-Format** (คอลัมน์ 8–72) เป็นค่าเริ่มต้นเพื่อความเป็นมาตรฐานสากลและใกล้เคียงกับ Mainframe จริง
  (มีสอน Free-Format ใน Part 002 เป็นทางเลือก)
- Part 051 เป็นต้นไป จะสอนแนวคิด Mainframe/JCL/CICS/DB2 แบบ **เชิงทฤษฎีและตัวอย่างโค้ดมาตรฐาน**
  เนื่องจากบางส่วนต้องใช้ Mainframe จริงหรือ Emulator (Hercules) จึงจะรันได้ 100% — เนื้อหาจะระบุชัดเจนว่าตัวอย่างใดรันบน GnuCOBOL ได้ทันที และตัวอย่างใดต้องใช้สภาพแวดล้อม Mainframe/Emulator
