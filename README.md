# หลักสูตร COBOL ฉบับสมบูรณ์: จากพื้นฐานสู่ระดับมืออาชีพระดับโลก

> ✅ **หลักสูตรเขียนเสร็จสมบูรณ์ครบทั้ง 100 Part / 1000 ขั้นตอนแล้ว**

หลักสูตรนี้พาผู้เรียนเดินทางตั้งแต่ **ขั้นตอนที่ 1 ถึงขั้นตอนที่ 1000** ผ่านเนื้อหา **100 ส่วน (Part)**
ครอบคลุมตั้งแต่การเขียนโปรแกรม COBOL ขั้นพื้นฐาน ไปจนถึงการพัฒนาระบบระดับองค์กร (Enterprise),
Mainframe, ฐานข้อมูล, CICS, การเชื่อมต่อกับเทคโนโลยีสมัยใหม่ (REST API, Docker, Cloud) และการทำโปรเจกต์จริงระดับโลก

ทุกส่วนมีเนื้อหาที่ **ใช้งานได้จริง 100%** พร้อมโค้ดตัวอย่างที่คอมไพล์และรันได้จริงด้วย [GnuCOBOL](https://gnucobol.sourceforge.io/)
คำอธิบายทีละบรรทัด แบบฝึกหัดพร้อมเฉลย และโปรเจกต์ประกอบการเรียนรู้

## วิธีใช้หลักสูตรนี้

1. เริ่มจาก `parts/part-001-*.md` และเรียงลำดับไปเรื่อย ๆ
2. แต่ละ Part ครอบคลุม 10 ขั้นตอน (Step) ตามที่ระบุในสารบัญด้านล่าง
3. ลงมือพิมพ์โค้ดและรันจริงทุกตัวอย่าง อย่าเพียงอ่านผ่าน
4. ทำแบบฝึกหัดท้ายบทก่อนเปิดดูเฉลย
5. ทำโปรเจกต์ประจำเฟส (Milestone Project) เพื่อรวบยอดความรู้

ดูรายละเอียดสถาปัตยกรรมหลักสูตรและปรัชญาการออกแบบได้ที่ [`docs/COURSE-OUTLINE.md`](docs/COURSE-OUTLINE.md)

ดูสถานะความคืบหน้าการเขียนเนื้อหาได้ที่ [`docs/PROGRESS.md`](docs/PROGRESS.md)

## โครงสร้างหลักสูตร (6 เฟส / 100 Parts / 1000 Steps)

| เฟส | ชื่อเฟส | Parts | Steps | คำอธิบาย |
|---|---|---|---|---|
| 1 | พื้นฐาน (Foundations) | 001–015 | 1–150 | โครงสร้างโปรแกรม, ชนิดข้อมูล, เงื่อนไข, ลูป, I/O พื้นฐาน |
| 2 | ระดับกลาง (Intermediate) | 016–035 | 151–350 | ตาราง, ไฟล์, Subprogram, การจัดการข้อความ |
| 3 | ขั้นสูง (Advanced) | 036–050 | 351–500 | Intrinsic Functions, Report Writer, JSON/XML, Error Handling |
| 4 | Mainframe/Enterprise | 051–070 | 501–700 | JCL, VSAM, DB2, CICS, IMS, Performance Tuning |
| 5 | COBOL สมัยใหม่ (Modern) | 071–085 | 701–850 | REST API, Docker, Cloud, CI/CD, Testing, Refactoring |
| 6 | ระดับมืออาชีพ/โลก (Professional) | 086–100 | 851–1000 | สถาปัตยกรรมองค์กร, Case Study, อาชีพ, Capstone Project |

## สารบัญฉบับเต็ม (Full Table of Contents)

> เขียนเสร็จครบทั้ง 100 Part แล้ว โค้ดตัวอย่างทุก Part ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL
> (ยกเว้นเนื้อหาอ้างอิงมาตรฐาน JCL/EXEC SQL/EXEC CICS ในเฟส 4 ที่ระบุไว้ชัดเจนว่าต้องใช้ Mainframe จริง)

### เฟส 1: พื้นฐาน (Foundations) (Parts 001–015)

- [Part 001: ประวัติศาสตร์และโลกของ COBOL, ทำไมต้องเรียน COBOL ในปี 2026, ภาพรวมอุตสาหกรรม (ขั้นตอนที่ 1–10)](parts/part-001-history-and-why-cobol.md)
- [Part 002: ติดตั้งสภาพแวดล้อมพัฒนา — GnuCOBOL, VS Code, การคอมไพล์โปรแกรมแรก (ขั้นตอนที่ 11–20)](parts/part-002-environment-setup.md)
- [Part 003: โครงสร้างโปรแกรม COBOL: 4 Divisions, กฎการเขียนคอลัมน์ (Column Rules) (ขั้นตอนที่ 21–30)](parts/part-003-program-structure.md)
- [Part 004: IDENTIFICATION DIVISION และ ENVIRONMENT DIVISION แบบละเอียด (ขั้นตอนที่ 31–40)](parts/part-004-identification-environment-division.md)
- [Part 005: DATA DIVISION เบื้องต้น: WORKING-STORAGE SECTION (ขั้นตอนที่ 41–50)](parts/part-005-working-storage-section.md)
- [Part 006: PICTURE Clause และชนิดข้อมูลตัวเลข/ตัวอักษร (ขั้นตอนที่ 51–60)](parts/part-006-picture-clause.md)
- [Part 007: DISPLAY และ ACCEPT: คำสั่ง Input/Output พื้นฐาน (ขั้นตอนที่ 61–70)](parts/part-007-display-accept.md)
- [Part 008: MOVE Statement และกฎการย้ายข้อมูล (MOVE Rules) (ขั้นตอนที่ 71–80)](parts/part-008-move-statement.md)
- [Part 009: เลขคณิตพื้นฐาน: ADD, SUBTRACT, MULTIPLY, DIVIDE, COMPUTE (ขั้นตอนที่ 81–90)](parts/part-009-arithmetic.md)
- [Part 010: เงื่อนไข IF-ELSE และ Condition Names (88-level) (ขั้นตอนที่ 91–100)](parts/part-010-if-else-condition-names.md)
- [Part 011: EVALUATE Statement (เทียบเท่า switch/case) (ขั้นตอนที่ 101–110)](parts/part-011-evaluate.md)
- [Part 012: PERFORM พื้นฐานและ PERFORM UNTIL (ลูป) (ขั้นตอนที่ 111–120)](parts/part-012-perform-until.md)
- [Part 013: PERFORM VARYING และการวนลูปหลายชั้น (Nested Loops) (ขั้นตอนที่ 121–130)](parts/part-013-perform-varying.md)
- [Part 014: โครงสร้างโปรแกรมแบบมีโมดูล: Paragraph, Section, PERFORM...THRU (ขั้นตอนที่ 131–140)](parts/part-014-paragraphs-sections.md)
- [Part 015: 🎯 โปรเจกต์เฟส 1: เครื่องคิดเลขและระบบคำนวณเกรดนักเรียน (ขั้นตอนที่ 141–150)](parts/part-015-phase1-project.md)

### เฟส 2: ระดับกลาง (Intermediate) (Parts 016–035)

- [Part 016: ตาราง (Tables) และ OCCURS Clause (ขั้นตอนที่ 151–160)](parts/part-016-tables-occurs.md)
- [Part 017: Subscript กับ Index และ SEARCH / SEARCH ALL (ขั้นตอนที่ 161–170)](parts/part-017-subscript-search.md)
- [Part 018: ตารางหลายมิติ (Multi-dimensional Tables) (ขั้นตอนที่ 171–180)](parts/part-018-multidim-tables.md)
- [Part 019: การจัดการข้อความ: STRING Statement (ขั้นตอนที่ 181–190)](parts/part-019-string-statement.md)
- [Part 020: การจัดการข้อความ: UNSTRING Statement (ขั้นตอนที่ 191–200)](parts/part-020-unstring-statement.md)
- [Part 021: INSPECT Statement: นับ แทนที่ แปลงตัวอักษร (ขั้นตอนที่ 201–210)](parts/part-021-inspect-statement.md)
- [Part 022: REDEFINES Clause และการใช้หน่วยความจำซ้ำ (ขั้นตอนที่ 211–220)](parts/part-022-redefines.md)
- [Part 023: ไฟล์ตามลำดับ (Sequential Files) เบื้องต้น (ขั้นตอนที่ 221–230)](parts/part-023-sequential-files.md)
- [Part 024: FILE SECTION และ FD Entry แบบละเอียด (ขั้นตอนที่ 231–240)](parts/part-024-file-section-fd.md)
- [Part 025: OPEN, CLOSE, READ, WRITE, REWRITE, DELETE (ขั้นตอนที่ 241–250)](parts/part-025-open-close-read-write.md)
- [Part 026: การประมวลผลไฟล์แบบ Master-Detail (Matching Records) (ขั้นตอนที่ 251–260)](parts/part-026-master-detail-processing.md)
- [Part 027: SORT และ MERGE Statement (ขั้นตอนที่ 261–270)](parts/part-027-sort-merge.md)
- [Part 028: Indexed Files และ ISAM (ขั้นตอนที่ 271–280)](parts/part-028-indexed-files.md)
- [Part 029: Relative Files (ขั้นตอนที่ 281–290)](parts/part-029-relative-files.md)
- [Part 030: File Status Codes และการจัดการข้อผิดพลาดของไฟล์ (ขั้นตอนที่ 291–300)](parts/part-030-file-status-codes.md)
- [Part 031: Subprograms: CALL Statement เบื้องต้น (ขั้นตอนที่ 301–310)](parts/part-031-call-statement.md)
- [Part 032: การส่งพารามิเตอร์: BY REFERENCE, BY CONTENT, BY VALUE (ขั้นตอนที่ 311–320)](parts/part-032-parameter-passing.md)
- [Part 033: COPY Statement และการใช้ Copybooks (ขั้นตอนที่ 321–330)](parts/part-033-copy-copybooks.md)
- [Part 034: Nested Programs และ END PROGRAM (ขั้นตอนที่ 331–340)](parts/part-034-nested-programs.md)
- [Part 035: 🎯 โปรเจกต์เฟส 2: ระบบจัดการสินค้าคงคลัง (Inventory Management System) (ขั้นตอนที่ 341–350)](parts/part-035-phase2-project.md)

### เฟส 3: ขั้นสูง (Advanced) (Parts 036–050)

- [Part 036: Intrinsic Functions: ตัวเลขและวันที่ (FUNCTION) (ขั้นตอนที่ 351–360)](parts/part-036-intrinsic-functions-numeric-date.md)
- [Part 037: Intrinsic Functions: ข้อความและสถิติ (ขั้นตอนที่ 361–370)](parts/part-037-intrinsic-functions-string-stats.md)
- [Part 038: Report Writer Feature เบื้องต้น (ขั้นตอนที่ 371–380)](parts/part-038-report-writer-basics.md)
- [Part 039: Report Writer ขั้นสูง: Control Breaks (ขั้นตอนที่ 381–390)](parts/part-039-report-writer-control-breaks.md)
- [Part 040: Screen Section: สร้าง UI แบบ Text-mode (ขั้นตอนที่ 391–400)](parts/part-040-screen-section.md)
- [Part 041: Advanced EVALUATE และเงื่อนไขซับซ้อน (ขั้นตอนที่ 401–410)](parts/part-041-advanced-evaluate.md)
- [Part 042: Class Condition และ Sign Condition (ขั้นตอนที่ 411–420)](parts/part-042-class-sign-condition.md)
- [Part 043: Dynamic CALL และ Program Pointers (ขั้นตอนที่ 421–430)](parts/part-043-dynamic-call.md)
- [Part 044: Based Storage และ POINTER Data Item (ขั้นตอนที่ 431–440)](parts/part-044-based-storage-pointer.md)
- [Part 045: JSON GENERATE และ JSON PARSE ใน COBOL สมัยใหม่ (ขั้นตอนที่ 441–450)](parts/part-045-json-generate-parse.md)
- [Part 046: XML GENERATE และ XML PARSE (ขั้นตอนที่ 451–460)](parts/part-046-xml-generate-parse.md)
- [Part 047: การจัดการวันที่และเวลาขั้นสูง (ขั้นตอนที่ 461–470)](parts/part-047-advanced-date-time.md)
- [Part 048: Error Handling ขั้นสูง: USE Statement และ DECLARATIVES (ขั้นตอนที่ 471–480)](parts/part-048-use-declaratives.md)
- [Part 049: Compiler Directives และความแตกต่างระหว่าง COBOL Dialects (ขั้นตอนที่ 481–490)](parts/part-049-compiler-directives.md)
- [Part 050: 🎯 โปรเจกต์เฟส 3: ระบบบัญชีลูกหนี้ (Accounts Receivable System) (ขั้นตอนที่ 491–500)](parts/part-050-phase3-project.md)

### เฟส 4: Mainframe/Enterprise (Parts 051–070)

- [Part 051: แนะนำ Mainframe และ z/OS Ecosystem (ขั้นตอนที่ 501–510)](parts/part-051-mainframe-zos-intro.md)
- [Part 052: JCL เบื้องต้น: JOB, EXEC, DD Statement (ขั้นตอนที่ 511–520)](parts/part-052-jcl-basics.md)
- [Part 053: JCL ขั้นสูง: Procedures และ Condition Codes (ขั้นตอนที่ 521–530)](parts/part-053-jcl-advanced.md)
- [Part 054: TSO/ISPF: การใช้งานเบื้องต้น (ขั้นตอนที่ 531–540)](parts/part-054-tso-ispf.md)
- [Part 055: VSAM เบื้องต้น: KSDS, ESDS, RRDS (ขั้นตอนที่ 541–550)](parts/part-055-vsam-basics.md)
- [Part 056: VSAM ขั้นสูง: Alternate Index และ Cluster (ขั้นตอนที่ 551–560)](parts/part-056-vsam-advanced.md)
- [Part 057: DB2 และ SQL เบื้องต้นสำหรับ COBOL (ขั้นตอนที่ 561–570)](parts/part-057-db2-sql-basics.md)
- [Part 058: Embedded SQL: EXEC SQL ใน COBOL (ขั้นตอนที่ 571–580)](parts/part-058-embedded-sql.md)
- [Part 059: DB2 Cursor และการประมวลผลหลายแถว (ขั้นตอนที่ 581–590)](parts/part-059-db2-cursor.md)
- [Part 060: DB2 Stored Procedures ด้วย COBOL (ขั้นตอนที่ 591–600)](parts/part-060-db2-stored-procedures.md)
- [Part 061: CICS เบื้องต้น: Transaction Processing (ขั้นตอนที่ 601–610)](parts/part-061-cics-intro.md)
- [Part 062: CICS Commands: SEND, RECEIVE, MAP (ขั้นตอนที่ 611–620)](parts/part-062-cics-send-receive-map.md)
- [Part 063: CICS BMS Maps และการออกแบบหน้าจอ (ขั้นตอนที่ 621–630)](parts/part-063-cics-bms-maps.md)
- [Part 064: CICS ขั้นสูง: Pseudo-conversational Programming (ขั้นตอนที่ 631–640)](parts/part-064-cics-pseudo-conversational.md)
- [Part 065: IMS DB/DC เบื้องต้น (ขั้นตอนที่ 641–650)](parts/part-065-ims-intro.md)
- [Part 066: Batch Processing Patterns และ Job Scheduling (ขั้นตอนที่ 651–660)](parts/part-066-batch-processing-patterns.md)
- [Part 067: DFSORT และ Sort Utilities ขั้นสูง (ขั้นตอนที่ 661–670)](parts/part-067-dfsort-advanced.md)
- [Part 068: Performance Tuning สำหรับ Mainframe COBOL (ขั้นตอนที่ 671–680)](parts/part-068-performance-tuning.md)
- [Part 069: Security บน Mainframe (RACF Concepts) (ขั้นตอนที่ 681–690)](parts/part-069-mainframe-security.md)
- [Part 070: 🎯 โปรเจกต์เฟส 4: ระบบธนาคารบน Mainframe จำลอง (ขั้นตอนที่ 691–700)](parts/part-070-phase4-project.md)

### เฟส 5: COBOL สมัยใหม่ (Modern) (Parts 071–085)

- [Part 071: GnuCOBOL สมัยใหม่และ Open-source COBOL Ecosystem (ขั้นตอนที่ 701–710)](parts/part-071-gnucobol-modern-ecosystem.md)
- [Part 072: การเชื่อมต่อ COBOL กับภาษาอื่น (C, Java) ผ่าน CALL (ขั้นตอนที่ 711–720)](parts/part-072-call-c-java.md)
- [Part 073: COBOL และ REST API: การสร้าง Wrapper Service (ขั้นตอนที่ 721–730)](parts/part-073-cobol-rest-api-wrapper.md)
- [Part 074: COBOL และ Web Application: เชื่อมกับ Node.js/Python (ขั้นตอนที่ 731–740)](parts/part-074-cobol-web-integration.md)
- [Part 075: ฐานข้อมูลสมัยใหม่: COBOL กับ MySQL/PostgreSQL (ขั้นตอนที่ 741–750)](parts/part-075-cobol-modern-databases.md)
- [Part 076: Containerizing COBOL ด้วย Docker (ขั้นตอนที่ 751–760)](parts/part-076-docker-cobol.md)
- [Part 077: COBOL บน Cloud: แนวคิด Mainframe Modernization (ขั้นตอนที่ 761–770)](parts/part-077-cobol-cloud-modernization.md)
- [Part 078: CI/CD Pipeline สำหรับโปรเจกต์ COBOL (ขั้นตอนที่ 771–780)](parts/part-078-cicd-cobol.md)
- [Part 079: Unit Testing สำหรับ COBOL (ขั้นตอนที่ 781–790)](parts/part-079-unit-testing-cobol.md)
- [Part 080: Version Control (Git) และ Workflow สำหรับทีม COBOL (ขั้นตอนที่ 791–800)](parts/part-080-git-workflow-cobol.md)
- [Part 081: Code Refactoring และ Clean Code สำหรับ COBOL (ขั้นตอนที่ 801–810)](parts/part-081-refactoring-clean-code.md)
- [Part 082: Design Patterns ในโลก COBOL (ขั้นตอนที่ 811–820)](parts/part-082-design-patterns-cobol.md)
- [Part 083: Legacy System Analysis และ Reverse Engineering (ขั้นตอนที่ 821–830)](parts/part-083-legacy-analysis.md)
- [Part 084: Migration Strategies: COBOL to Java/C# (ขั้นตอนที่ 831–840)](parts/part-084-migration-strategies.md)
- [Part 085: 🎯 โปรเจกต์เฟส 5: ปรับปรุงระบบ Legacy ด้วยสถาปัตยกรรมสมัยใหม่ (ขั้นตอนที่ 841–850)](parts/part-085-phase5-project.md)

### เฟส 6: ระดับมืออาชีพ/โลก (Professional) (Parts 086–100)

- [Part 086: สถาปัตยกรรมระบบองค์กรขนาดใหญ่ด้วย COBOL (ขั้นตอนที่ 851–860)](parts/part-086-enterprise-architecture.md)
- [Part 087: Microservices และ COBOL ในระบบกระจาย (ขั้นตอนที่ 861–870)](parts/part-087-microservices-cobol.md)
- [Part 088: Performance Optimization ระดับสูง (ขั้นตอนที่ 871–880)](parts/part-088-performance-optimization.md)
- [Part 089: Security ระดับ Enterprise สำหรับแอปพลิเคชัน COBOL (ขั้นตอนที่ 881–890)](parts/part-089-enterprise-security.md)
- [Part 090: การจัดการโปรเจกต์ COBOL ขนาดใหญ่ (Project Management) (ขั้นตอนที่ 891–900)](parts/part-090-project-management.md)
- [Part 091: Case Study - Core Banking ตอนที่ 1: การออกแบบระบบ (ขั้นตอนที่ 901–910)](parts/part-091-corebanking-design.md)
- [Part 092: Case Study - Core Banking ตอนที่ 2: โมดูลบัญชีและธุรกรรม (ขั้นตอนที่ 911–920)](parts/part-092-corebanking-transactions.md)
- [Part 093: Case Study - Core Banking ตอนที่ 3: การรายงานและปิดบัญชี (ขั้นตอนที่ 921–930)](parts/part-093-corebanking-reporting.md)
- [Part 094: Case Study — ระบบประกันภัย (Insurance System) (ขั้นตอนที่ 931–940)](parts/part-094-insurance-case-study.md)
- [Part 095: Case Study — ระบบ ERP และห่วงโซ่อุปทาน (ขั้นตอนที่ 941–950)](parts/part-095-erp-supply-chain-case-study.md)
- [Part 096: การเตรียมตัวสอบใบรับรอง COBOL และแนวทางอาชีพ (ขั้นตอนที่ 951–960)](parts/part-096-certification-career.md)
- [Part 097: แนวข้อสอบสัมภาษณ์งาน COBOL Developer (ขั้นตอนที่ 961–970)](parts/part-097-interview-questions.md)
- [Part 098: Best Practices และมาตรฐานการเขียนโค้ดระดับโลก (ขั้นตอนที่ 971–980)](parts/part-098-best-practices.md)
- [Part 099: อนาคตของ COBOL และเทรนด์เทคโนโลยี (ขั้นตอนที่ 981–990)](parts/part-099-future-of-cobol.md)
- [Part 100: 🎯 Capstone Project สุดท้าย, บทสรุปหลักสูตร, และแหล่งเรียนรู้ต่อ (ขั้นตอนที่ 991–1000)](parts/part-100-final-capstone.md)

## ข้อกำหนดเบื้องต้น

- เครื่องคอมพิวเตอร์ Windows / macOS / Linux
- ติดตั้ง [GnuCOBOL](https://gnucobol.sourceforge.io/) (สอนวิธีติดตั้งใน Part 002)
- Text Editor เช่น VS Code (แนะนำ extension: COBOL Language Support)
- ไม่จำเป็นต้องมีประสบการณ์เขียนโปรแกรมมาก่อน แต่ถ้ามีจะช่วยให้เรียนเร็วขึ้น

## License / การนำไปใช้

เนื้อหานี้จัดทำขึ้นเพื่อการศึกษา สามารถนำไปใช้เรียนรู้และสอนต่อได้
