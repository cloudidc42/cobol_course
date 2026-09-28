# Part 070: 🎯 โปรเจกต์เฟส 4: ระบบธนาคารบน Mainframe จำลอง (ขั้นตอนที่ 691–700)

## คำนำของ Part นี้

เดินทางมาถึงจุดสำคัญที่สุดของเฟส 4! ตั้งแต่ Part 051 ถึง Part 069 เราเรียนรู้แนวคิดของโลก Mainframe
เกือบทั้งหมด: JCL (Part 052-053), TSO/ISPF (Part 054), VSAM (Part 055-056), DB2/SQL (Part 057-060),
CICS (Part 061-064), IMS (Part 065), Batch Processing Pattern (Part 066), DFSORT (Part 067),
Performance Tuning (Part 068) และ Security/RACF (Part 069) เนื้อหาส่วนใหญ่ของหัวข้อเหล่านี้เป็น**เชิง
แนวคิดและอ้างอิง**เพราะต้องใช้ z/OS จริงหรือ Emulator เท่านั้นจึงจะรันได้ 100% ตามที่ระบุไว้ตั้งแต่ต้นเฟส

Part นี้คือ**บทพิสูจน์**: เราจะนำแนวคิดทั้งหมดที่เรียนมาประกอบร่างเป็น **ระบบธนาคารที่ทำงานได้จริง
สมบูรณ์แบบ 100%** ด้วย GnuCOBOL

### ประกาศความซื่อสัตย์ที่สำคัญที่สุดของ Part นี้ (อ่านก่อนเริ่ม)

> **โปรเจกต์นี้จำลองเฉพาะ "ฝั่งการประมวลผลไฟล์และแบทช์" ของระบบ Mainframe COBOL เท่านั้น**
> เราใช้ **GnuCOBOL's Indexed File (ผ่าน Berkeley DB เป็น ISAM handler)** เป็นตัวแทนของ **VSAM KSDS**
> (ตามแนวทางเดียวกับที่ Part 028 อธิบายไว้อย่างละเอียด) เราใช้**ไฟล์ Sequential ธรรมดา**เป็น Audit
> Trail แทนที่จะเป็น VSAM/DB2 log จริง และเราใช้**โปรแกรมแบบโต้ตอบสั้น ๆ ที่รันแล้วจบ**เพื่อจำลองพฤติกรรม
> ของ CICS Transaction (ไม่ใช่ CICS จริง เพราะไม่มี CICS region ให้ทดสอบ)
>
> **สิ่งที่โปรเจกต์นี้ไม่ใช่และไม่ได้อ้างว่าเป็น**: นี่ไม่ใช่ระบบที่รันบน z/OS จริง ไม่มี JCL ที่รันได้จริง
> ไม่มี CICS transaction จริง ไม่มี DB2 database จริง และไม่มี RACF ควบคุมสิทธิ์จริง — องค์ประกอบเหล่านี้
> ถูกอธิบายไว้แล้วในเชิงแนวคิดตลอด Part 051-069 และจะไม่ถูกกล่าวอ้างว่า "รันได้จริง" ในโปรเจกต์นี้
>
> **สิ่งที่โปรเจกต์นี้เป็นจริง 100%**: ทุกโปรแกรมในนี้**คอมไพล์และรันได้จริงด้วย GnuCOBOL**, ทุกผลลัพธ์
> ที่แสดงคือผลลัพธ์จริงจากการรันจริง, ระบบไฟล์ Indexed ทำงานได้จริงด้วยกลไก ISAM จริง (Berkeley DB),
> การคำนวณดอกเบี้ย การกระทบยอด (reconciliation) และการสร้างใบแจ้งยอด (statement) ล้วนเป็นตรรกะ
> ทางธุรกิจที่ทำงานถูกต้องและตรวจสอบได้จริงทุกประการ **จุดประสงค์ของ Part นี้คือแสดงให้เห็นว่า "ฝั่งการ
> ประมวลผลไฟล์และแบทช์" ของระบบ COBOL แบบ Mainframe มีหน้าตาและตรรกะการทำงานอย่างไร** ผ่านเครื่องมือ
> ที่เข้าถึงได้จริงโดยไม่ต้องมี Mainframe จริงอยู่ตรงหน้า

### หมายเหตุสำคัญเรื่องการคอมไพล์ (อ่านก่อนลงมือ)

โปรเจกต์นี้ใช้ `ORGANIZATION IS INDEXED` อย่างหนักหน่วง เหมือนที่ Part 028 เตือนไว้แล้ว: **GnuCOBOL
มาตรฐานจากตัวจัดการแพ็กเกจของหลายระบบปฏิบัติการปิดการรองรับ Indexed File ไว้โดยค่าเริ่มต้น** ตรวจสอบ
ด้วย `cobc -info | grep -i indexed` — ถ้าขึ้นว่า `disabled` ต้อง build GnuCOBOL ใหม่เองพร้อมเปิดใช้
`--with-db` (Berkeley DB) ตามที่ Part 028 อธิบายไว้ ทุกตัวอย่างในเอกสารนี้ทดสอบจริงด้วย GnuCOBOL ที่
build พร้อม **Berkeley DB (BDB) 5.3.28** เป็น ISAM handler (`indexed file handler : BDB version
5.3.28`) และให้ผลลัพธ์ตรงตามที่แสดงไว้ทุกประการ

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่างผ่านการ
> คอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง ผลลัพธ์ทุกชิ้นที่แสดงคือผลลัพธ์จริงจากการรันจริง

### ภาพรวม 10 ขั้นตอนของโปรเจกต์นี้

| ขั้นตอน | สิ่งที่สร้าง |
|---|---|
| 691 | ออกแบบระบบทั้งหมด + Copybook `acctrec.cpy` + โปรแกรมตั้งค่าเริ่มต้น `ACCTINIT` |
| 692 | Subprogram ร่วม `WRTAUDIT` (เขียน Audit Trail) ผ่านเทคนิค CALL (Part 031-032) |
| 693 | `ACCTOPEN` — เปิดบัญชีใหม่ (online-style transaction) |
| 694 | `ACCTDEP` — ฝากเงิน |
| 695 | `ACCTWD` — ถอนเงิน (ป้องกันยอดติดลบ) |
| 696 | `ACCTINQ` — สอบถามยอด + `ACCTCLOSE` — ปิดบัญชี |
| 697 | `BATCHINT` — Batch คำนวณดอกเบี้ยรายคืน พร้อม Control Total Report |
| 698 | `STMTGEN` — Batch สร้างใบแจ้งยอดลูกค้า (SORT + Control Break + Master-Detail) |
| 699 | `RECONCILE` — Batch กระทบยอดควบคุม (Control Total Reconciliation) |
| 700 | ทดสอบวงจรทั้งระบบแบบครบวงจร (online + batch) และสรุปการเชื่อมโยงกับ Part 051-069 |

---

## ขั้นตอนที่ 691: ออกแบบระบบ, Copybook บัญชี, และโปรแกรมตั้งค่าเริ่มต้น

### การออกแบบข้อมูล (Data Design)

**ACCOUNT-MASTER** (Indexed File — ตัวแทนของ VSAM KSDS จาก Part 055): เก็บสถานะปัจจุบันของทุกบัญชี
Key หลักคือ `ACCT-ID` (ทบทวนกฎการออกแบบ Key จาก Part 028 ขั้นตอนที่ 272: ใช้ `PIC 9` แท้เพื่อให้เรียง
ลำดับตามค่าตัวเลขถูกต้องเสมอ)

| ฟิลด์ | PICTURE | ความหมาย |
|---|---|---|
| `ACCT-ID` | `9(6)` | รหัสบัญชี (Record Key) |
| `ACCT-NAME` | `X(24)` | ชื่อเจ้าของบัญชี |
| `ACCT-TYPE` | `X(1)` | `S`=Savings (ออมทรัพย์), `C`=Checking (กระแสรายวัน) |
| `ACCT-STATUS` | `X(1)` | `A`=Active, `C`=Closed |
| `ACCT-BALANCE` | `9(9)V99` | ยอดคงเหลือปัจจุบัน |
| `ACCT-INTEREST-RATE` | `9(1)V9(4)` | อัตราดอกเบี้ยต่อปี (เฉพาะ Savings) |
| `ACCT-OPEN-DATE` | `9(8)` | วันที่เปิดบัญชี (YYYYMMDD) |
| `ACCT-LAST-STMT-DATE` | `9(8)` | วันที่ประมวลผลดอกเบี้ย/statement ล่าสุด |

**TXNLOG.DAT** (Sequential File — Audit Trail): ไฟล์ต่อท้ายอย่างเดียว (append-only) บันทึกทุก
transaction ที่เกิดขึ้น ไม่ว่าจะสำเร็จหรือถูกปฏิเสธ นี่คือคู่หูของ Master File ตามสถาปัตยกรรม Master-Detail
ที่เรียนมาใน Part 026 — Master เก็บ "สถานะปัจจุบัน" ส่วน Sequential Log เก็บ "ประวัติเหตุการณ์ทั้งหมด"

| ฟิลด์ (ตำแหน่งไบต์ในบรรทัดข้อความ) | PICTURE | ความหมาย |
|---|---|---|
| คอลัมน์ 1-8 | `9(8)` | วันที่ทำรายการ |
| คอลัมน์ 10-15 | `9(6)` | เวลาทำรายการ |
| คอลัมน์ 17-22 | `9(6)` | รหัสบัญชี |
| คอลัมน์ 24-27 | `X(4)` | ประเภทรายการ: `OPEN`, `DEP`, `WD`, `INT`, `INQ`, `CLOS` |
| คอลัมน์ 29-39 | `9(9)V99` | จำนวนเงินของรายการนี้ |
| คอลัมน์ 41-51 | `9(9)V99` | ยอดคงเหลือหลังทำรายการ |
| คอลัมน์ 53-60 | `X(8)` | สถานะ: `OK` หรือ `REJECTED` |

### สถาปัตยกรรมโปรแกรมทั้งระบบ

```
                    ┌─────────────────┐
                    │   ACCTINIT       │  (run once - creates empty master)
                    └────────┬─────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │     ONLINE-STYLE TRANSACTIONS          │
         │  (short-lived, interactive, one per     │
         │   customer request - simulates CICS)    │
         │                                          │
         │  ACCTOPEN   ACCTDEP   ACCTWD             │
         │  ACCTINQ    ACCTCLOSE                    │
         │       │         │        │                │
         │       └────┬────┴────┬───┘                │
         │            ▼         ▼                    │
         │     ACCOUNT-MASTER  WRTAUDIT (CALL)        │
         │     (indexed, I-O)       │                 │
         │                          ▼                 │
         │                    TXNLOG.DAT               │
         │                    (sequential, append)     │
         └───────────────────────┬────────────────────┘
                                  │ (end of business day)
         ┌────────────────────────┼────────────────────────┐
         │        NIGHTLY BATCH WINDOW (Part 066)            │
         │                                                    │
         │   BATCHINT ──reads/rewrites──> ACCOUNT-MASTER       │
         │   STMTGEN  ──reads──> TXNLOG.DAT + ACCOUNT-MASTER    │
         │   RECONCILE──reads──> ACCOUNT-MASTER + TXNLOG.DAT     │
         └────────────────────────────────────────────────────┘
```

สังเกตว่าสถาปัตยกรรมนี้เลียนแบบรูปแบบ Mainframe จริงอย่างตรงไปตรงมา: **โปรแกรมออนไลน์ทำงานสั้น ๆ ทีละ
รายการระหว่างวัน แล้วโปรแกรมแบทช์มาทำงานหนักในช่วงกลางคืน** (ทบทวนแนวคิด Batch Window จาก Part 066)

### Copybook `acctrec.cpy` — โครงสร้างบัญชีที่ใช้ร่วมกันทั้งระบบ

ทุกโปรแกรมที่แตะ `ACCOUNT-MASTER` ต้องมีโครงสร้าง record **ตรงกันเป๊ะ** ไม่เช่นนั้นจะเกิดปัญหาแบบเดียว
กับที่ Part 031 ขั้นตอนที่ 304 พิสูจน์ไว้ (ข้อมูลตีความผิดเพี้ยนแบบไม่มี error เตือน) เราจึงใช้เทคนิค
**COPY Statement** จาก Part 033 เพื่อรับประกันว่าทุกโปรแกรม**ไม่มีทางเขียนโครงสร้างผิดเพี้ยนไปจากกัน**ได้
เลย เพราะทุกโปรแกรมดึงเนื้อหาจากไฟล์**เดียวกัน**ไฟล์นี้:

```cobol
      ******************************************************************
      * ACCTREC.CPY
      * Shared account master record layout - the ONE place that
      * defines what an account record looks like. Every program that
      * touches ACCOUNT-MASTER (indexed file, our stand-in for VSAM
      * KSDS) COPYs this into its FD entry, so the layout can never
      * drift apart between programs (Part 033 technique, applied for
      * real across a whole multi-program system).
      ******************************************************************
           05  ACCT-ID                 PIC 9(6).
           05  ACCT-NAME               PIC X(24).
           05  ACCT-TYPE               PIC X(1).
               88  ACCT-TYPE-SAVINGS       VALUE "S".
               88  ACCT-TYPE-CHECKING      VALUE "C".
           05  ACCT-STATUS             PIC X(1).
               88  ACCT-STATUS-ACTIVE      VALUE "A".
               88  ACCT-STATUS-CLOSED      VALUE "C".
           05  ACCT-BALANCE            PIC 9(9)V99.
           05  ACCT-INTEREST-RATE      PIC 9(1)V9(4).
           05  ACCT-OPEN-DATE          PIC 9(8).
           05  ACCT-LAST-STMT-DATE     PIC 9(8).
```

บันทึกไว้เป็นไฟล์ `acctrec.cpy` ในโฟลเดอร์เดียวกับไฟล์ `.cob` ทั้งหมด (GnuCOBOL ค้นหาไฟล์ COPY ในโฟลเดอร์
ปัจจุบันโดยอัตโนมัติเมื่อเขียนชื่อไฟล์ในเครื่องหมายคำพูด ไม่ต้องระบุ path เพิ่มเติม)

### `ACCTINIT.cob` — สร้างไฟล์ Master เปล่าครั้งแรก

โปรแกรมนี้เทียบเท่ากับขั้นตอน `DEFINE CLUSTER` ของ IDCAMS ที่ใช้สร้าง VSAM Cluster ครั้งแรกบน z/OS จริง
(ทบทวนจาก Part 055) — รันเพียงครั้งเดียวก่อนระบบเริ่มใช้งานจริง:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCTINIT.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * ACCTINIT - one-time setup program. Creates a brand-new, empty
      * ACCOUNT-MASTER indexed file. This is the COBOL/GnuCOBOL
      * equivalent of the first IDCAMS DEFINE CLUSTER step that would
      * create a VSAM KSDS on a real z/OS system (Part 055) - run
      * once before the system is used for the first time.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT ACCOUNT-MASTER.
           IF WS-FILE-STATUS = "00"
               DISPLAY "ACCTINIT: ACCTMAST.DAT created successfully."
           ELSE
               DISPLAY "ACCTINIT: ERROR creating file, status="
                   WS-FILE-STATUS
           END-IF.
           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

คอมไพล์และรัน:

```bash
cobc -x acctinit.cob -o acctinit
./acctinit
```

**ผลลัพธ์จริง (ยืนยันด้วย GnuCOBOL build ที่มี BDB 5.3.28):**

```
ACCTINIT: ACCTMAST.DAT created successfully.
```

การตรวจสอบว่า GnuCOBOL ที่ใช้รองรับ Indexed File จริง:

```bash
cobc -info | grep -i indexed
```

```
indexed file handler     : BDB version 5.3.28
```

### ข้อควรระวัง

- **ต้องรัน `ACCTINIT` เพียงครั้งเดียวเท่านั้น** ก่อนใช้งานระบบครั้งแรก — ถ้ารันซ้ำจะเป็นการ `OPEN OUTPUT`
  ทับไฟล์เดิม ลบข้อมูลบัญชีทั้งหมดที่มีอยู่ก่อนหน้าทิ้งทันที (พฤติกรรมเดียวกับ `OPEN OUTPUT` ที่เรียนมา
  ตั้งแต่ Part 025)
- ทุกโปรแกรมในระบบนี้ต้อง compile และ run ในโฟลเดอร์เดียวกัน (ที่มี `ACCTMAST.DAT`, `TXNLOG.DAT`
  และไฟล์ `.cpy` อยู่ครบ) เพราะทุกโปรแกรมอ้างอิงชื่อไฟล์แบบ relative path เดียวกันหมด

### แบบฝึกหัดที่ 691.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ Copybook `acctrec.cpy` ในระบบนี้จึงสำคัญกว่าตอนที่เรียน Part 033
เสียอีก เมื่อพิจารณาว่าระบบนี้มีโปรแกรมที่แตะ `ACCOUNT-MASTER` ถึง 8 โปรแกรม

**เฉลยแนวทาง**: ยิ่งจำนวนโปรแกรมที่ใช้โครงสร้างเดียวกันมากขึ้นเท่าไร ความเสี่ยงที่จะเขียนโครงสร้างให้
คลาดเคลื่อนไปจากกันโดยไม่ตั้งใจ (พิมพ์ผิด, ลืมอัปเดตบางไฟล์เมื่อโครงสร้างเปลี่ยน) ก็ยิ่งสูงขึ้นตามจำนวน
โปรแกรมนั้น หากมี 8 โปรแกรมพิมพ์ `01 ACCOUNT-RECORD` แยกกันเองทั้งหมด และวันหนึ่งต้องเพิ่มฟิลด์ใหม่
(เช่น `ACCT-BRANCH-CODE`) จะต้องแก้ไข 8 ที่พร้อมกันให้ตรงกันเป๊ะ ซึ่งเสี่ยงต่อความผิดพลาดสูงมาก การรวมไว้
ในไฟล์ `acctrec.cpy` ไฟล์เดียวทำให้แก้เพียงจุดเดียว แล้ว compile โปรแกรมทั้งหมดใหม่ก็เพียงพอ — ยิ่งระบบ
ใหญ่เท่าไร คุณค่าของ Copybook ยิ่งเพิ่มขึ้นตามสัดส่วน ไม่ใช่คงที่

---

## ขั้นตอนที่ 692: Subprogram ร่วม `WRTAUDIT` — เขียน Audit Trail ด้วยเทคนิค CALL

### ทำไมต้องแยกเป็น Subprogram

ทุกโปรแกรม online (ACCTOPEN, ACCTDEP, ACCTWD, ACCTINQ, ACCTCLOSE) และโปรแกรม batch (BATCHINT)
ต้องเขียนบรรทึก Audit Trail รูปแบบเดียวกันทุกครั้งที่มีการทำรายการ แทนที่จะเขียนโค้ดเปิด/เขียน/ปิดไฟล์
`TXNLOG.DAT` ซ้ำ ๆ กันใน 6 โปรแกรม (เสี่ยงต่อการเขียนรูปแบบไม่ตรงกันแบบเดียวกับปัญหา Copybook ใน
ขั้นตอนที่ 691) เรารวมตรรกะนี้ไว้ใน **Subprogram เดียว** ที่ทุกโปรแกรมเรียกผ่าน `CALL` (ทบทวนเทคนิคจาก
Part 031-032) — นี่คือการนำแนวคิด **Single Source of Truth** มาใช้กับ**ตรรกะ**ของโปรแกรม ไม่ใช่แค่
โครงสร้างข้อมูลเหมือน Copybook

### ซอร์สโค้ด: `wrtaudit.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. WRTAUDIT.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * WRTAUDIT - shared subprogram (Part 031-032 CALL technique).
      * Every online and batch program in this system CALLs WRTAUDIT
      * instead of writing to the audit trail sequential file itself,
      * so the LOG LINE FORMAT is defined in exactly one place. This
      * sequential text file is the "audit trail" side of the system:
      * an append-only record of every transaction attempt, matching
      * the classic Mainframe pattern of a master file (indexed,
      * random access) paired with a sequential transaction log.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TRANSACTION-LOG ASSIGN TO "TXNLOG.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  TRANSACTION-LOG.
       01  TRANSACTION-LINE        PIC X(80).

       WORKING-STORAGE SECTION.
       01  WS-LOG-STATUS           PIC X(2).
       01  WS-OUT-LINE             PIC X(80)  VALUE SPACES.

       LINKAGE SECTION.
       01  LK-TXN-DATE             PIC 9(8).
       01  LK-TXN-TIME             PIC 9(6).
       01  LK-TXN-ACCT-ID          PIC 9(6).
       01  LK-TXN-TYPE             PIC X(4).
       01  LK-TXN-AMOUNT           PIC 9(9)V99.
       01  LK-TXN-BALANCE-AFTER    PIC 9(9)V99.
       01  LK-TXN-STATUS           PIC X(8).

       PROCEDURE DIVISION USING LK-TXN-DATE LK-TXN-TIME
               LK-TXN-ACCT-ID LK-TXN-TYPE LK-TXN-AMOUNT
               LK-TXN-BALANCE-AFTER LK-TXN-STATUS.
       MAIN-PARA.
      *> Each CALL opens, appends one line, and closes again - this
      *> program simulates short-lived online transactions (like a
      *> CICS pseudo-conversational task, Part 064) that each run and
      *> finish quickly, not a long-running batch job that keeps a
      *> file open across many records.
           OPEN EXTEND TRANSACTION-LOG.
           IF WS-LOG-STATUS NOT = "00"
      *> File does not exist yet (first transaction ever) - create it.
               OPEN OUTPUT TRANSACTION-LOG
           END-IF.

           STRING LK-TXN-DATE           DELIMITED BY SIZE
                   " "                  DELIMITED BY SIZE
                   LK-TXN-TIME          DELIMITED BY SIZE
                   " "                  DELIMITED BY SIZE
                   LK-TXN-ACCT-ID       DELIMITED BY SIZE
                   " "                  DELIMITED BY SIZE
                   LK-TXN-TYPE          DELIMITED BY SIZE
                   " "                  DELIMITED BY SIZE
                   LK-TXN-AMOUNT        DELIMITED BY SIZE
                   " "                  DELIMITED BY SIZE
                   LK-TXN-BALANCE-AFTER DELIMITED BY SIZE
                   " "                  DELIMITED BY SIZE
                   LK-TXN-STATUS        DELIMITED BY SIZE
               INTO WS-OUT-LINE.
           MOVE WS-OUT-LINE TO TRANSACTION-LINE.
           WRITE TRANSACTION-LINE.
           CLOSE TRANSACTION-LOG.
           GOBACK.
```

### ทดสอบด้วยโปรแกรมเรียกทดลอง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTAUDIT.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE   PIC 9(8) VALUE 20260101.
       01  WS-TIME   PIC 9(6) VALUE 120000.
       01  WS-ACCT   PIC 9(6) VALUE 100001.
       01  WS-TYPE   PIC X(4) VALUE "OPEN".
       01  WS-AMT    PIC 9(9)V99 VALUE 500.00.
       01  WS-BAL    PIC 9(9)V99 VALUE 500.00.
       01  WS-STAT   PIC X(8) VALUE "OK".
       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "WRTAUDIT" USING WS-DATE WS-TIME WS-ACCT WS-TYPE
               WS-AMT WS-BAL WS-STAT.
           DISPLAY "done".
           STOP RUN.
```

```bash
cobc -x testaudit.cob wrtaudit.cob -o testaudit
./testaudit
./testaudit
cat TXNLOG.DAT
```

**ผลลัพธ์จริง:**

```
done
done
20260101 120000 100001 OPEN 00000050000 00000050000 OK
20260101 120000 100001 OPEN 00000050000 00000050000 OK
```

### อธิบายจุดสำคัญ

- คอมไพล์ด้วยคำสั่ง `cobc -x testaudit.cob wrtaudit.cob -o testaudit` — ส่ง**ทั้งสองไฟล์**เข้า `cobc`
  พร้อมกันเพื่อ link เป็น execute เดียว ตามเทคนิคที่เรียนมาใน Part 031 ขั้นตอนที่ 301
- `OPEN EXTEND` บนไฟล์ที่ยังไม่เคยมีมาก่อนจะได้ `WS-LOG-STATUS` ไม่เท่ากับ `"00"` ทำให้โปรแกรมสลับไป
  `OPEN OUTPUT` สร้างไฟล์ใหม่โดยอัตโนมัติ — นี่คือการใช้ FILE STATUS (Part 030) เพื่อจัดการสถานการณ์
  "ไฟล์อาจมีหรือไม่มีอยู่ก็ได้" อย่างสง่างาม โดยไม่ต้องให้ผู้เรียกกังวลเรื่องนี้เลย
- สังเกตว่า `LK-TXN-AMOUNT`/`LK-TXN-BALANCE-AFTER` เป็น `PIC 9(9)V99` (มี V) ในขณะที่ตัวแปรฝั่งผู้เรียก
  ก็เป็น `PIC 9(9)V99` เหมือนกัน — พารามิเตอร์ตัวเลขที่มีทศนิยมส่งผ่าน `CALL` ได้ตรงไปตรงมาเมื่อ
  PICTURE ทั้งสองฝั่งตรงกัน (ทบทวนกฎจาก Part 032)

### ข้อควรระวัง

- **ทุกโปรแกรมที่ `CALL "WRTAUDIT"` ต้องส่งพารามิเตอร์ครบ 7 ตัวตามลำดับเป๊ะ** มิฉะนั้นจะเกิดปัญหาแบบ
  เดียวกับที่ Part 031 ขั้นตอนที่ 304 พิสูจน์ไว้ (SIGSEGV หรือข้อมูลตีความผิดเพี้ยน) — ไม่มี compiler
  ตรวจสอบให้เพราะเป็นคนละไฟล์ compile กัน
- ระวังอย่าส่ง string literal ที่มีความยาวไม่ตรงกับ `LINKAGE SECTION` ของ `WRTAUDIT` โดยตรง (เช่น
  `CALL ... USING ... "OK" ...` ที่ความยาว literal ไม่ตรงกับ `PIC X(8)`) — ควรกำหนดค่าใส่ตัวแปร
  `WORKING-STORAGE` ที่มีความกว้างตรงกันก่อนเสมอ (ตามที่ทุกโปรแกรมในระบบนี้ทำ)

### แบบฝึกหัดที่ 692.1

**โจทย์**: จงอธิบายว่าทำไม `WRTAUDIT` จึง `OPEN` และ `CLOSE` ไฟล์**ทุกครั้ง**ที่ถูกเรียก แทนที่จะเปิดไฟล์
ค้างไว้ตลอดและให้ผู้เรียกสั่งปิดเองตอนจบโปรแกรม

**เฉลยแนวทาง**: เพราะโปรแกรม online แต่ละตัว (ACCTOPEN, ACCTDEP ฯลฯ) ถูกออกแบบให้เป็นโปรแกรมที่
รันครั้งเดียวจบต่อหนึ่ง transaction (จำลองพฤติกรรม CICS pseudo-conversational ตามที่ comment ในโค้ด
อธิบายไว้) แต่ละโปรแกรมจึงมักเรียก `WRTAUDIT` เพียงครั้งเดียวแล้วก็ `STOP RUN` ทันที การเปิด-เขียน-ปิด
ให้เสร็จสมบูรณ์ในตัว `WRTAUDIT` เองทำให้**ผู้เรียกไม่ต้องรับผิดชอบเรื่องการจัดการไฟล์เลย** (encapsulation)
ลดความเสี่ยงที่ผู้เรียกจะลืมปิดไฟล์ และทำให้ไฟล์ `TXNLOG.DAT` ปลอดภัยจากการเขียนค้างครึ่ง ๆ กลาง ๆ หาก
โปรแกรมเรียกจบการทำงานกะทันหันด้วยเหตุผลอื่น

---

## ขั้นตอนที่ 693: `ACCTOPEN` — เปิดบัญชีใหม่ (Online-style Transaction)

### แนวคิด

โปรแกรมนี้คือ transaction แรกของระบบ: เปิดบัญชีใหม่ ใช้เทคนิค Secure Coding ทั้งหมดจาก Part 069:
ตรวจสอบข้อมูลนำเข้าด้วย `IS NUMERIC` ก่อนเชื่อถือเสมอ (ขั้นตอนที่ 686), ตรวจสอบว่าบัญชีซ้ำหรือไม่ก่อน
สร้าง, และปฏิเสธข้อมูลที่ผิดพลาดโดยไม่แก้ไขอะไรเลย

### ซอร์สโค้ด: `acctopen.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCTOPEN.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * ACCTOPEN - online-style transaction: open a new account.
      * "Online-style" here means a short-lived, interactive program a
      * teller would run once per customer request - the closest thing
      * GnuCOBOL lets us demonstrate to a real CICS transaction
      * (Part 061-064) without an actual CICS region. Each run opens
      * the indexed master, does ONE piece of business, and exits -
      * exactly the shape of a pseudo-conversational CICS task.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).

       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-RAW-TYPE              PIC X(1)   VALUE SPACES.
       01  WS-RAW-AMOUNT            PIC X(9)   VALUE SPACES.
       01  WS-NEW-NAME              PIC X(24)  VALUE SPACES.
       01  WS-AMOUNT-CENTS          PIC 9(9)   VALUE 0.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".

      *> Fixed "today" for this whole demo system (no system clock
      *> dependency needed for reproducible output). A production
      *> system would use FUNCTION CURRENT-DATE (Part 036) instead.
       01  WS-TODAY                 PIC 9(8)   VALUE 20260401.
       01  WS-NOW                   PIC 9(6)   VALUE 90000.
       01  WS-TXN-TYPE-OPEN         PIC X(4)   VALUE "OPEN".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.

           DISPLAY "=== ACCTOPEN: Open New Account ===".
           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
      *> Secure coding technique from Part 069 step 686: validate
      *> BEFORE trusting any external input.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be 6 numeric digits."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
      *> Duplicate check: try to READ the key we are about to create.
      *> If it is FOUND, the account already exists - reject.
               READ ACCOUNT-MASTER
                   INVALID KEY
                       CONTINUE
                   NOT INVALID KEY
                       DISPLAY "ERROR: account " WS-RAW-ID
                           " already exists."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter account holder name (up to 24 chars): "
                   WITH NO ADVANCING
               ACCEPT WS-NEW-NAME
               DISPLAY "Enter type (S=Savings, C=Checking): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-TYPE
               IF WS-RAW-TYPE NOT = "S" AND WS-RAW-TYPE NOT = "C"
                   DISPLAY "ERROR: type must be S or C."
                   MOVE "N" TO WS-VALID-FLAG
               END-IF
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter opening deposit in cents (9 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-AMOUNT
               IF WS-RAW-AMOUNT IS NOT NUMERIC
                   DISPLAY "ERROR: opening deposit must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
               END-IF
           END-IF.

           IF WS-ALL-VALID
               MOVE WS-NEW-NAME TO ACCT-NAME
               MOVE WS-RAW-TYPE TO ACCT-TYPE
               SET ACCT-STATUS-ACTIVE TO TRUE
               COMPUTE ACCT-BALANCE = WS-AMOUNT-CENTS / 100
               IF ACCT-TYPE-SAVINGS
                   MOVE 0.0250 TO ACCT-INTEREST-RATE
               ELSE
                   MOVE 0.0000 TO ACCT-INTEREST-RATE
               END-IF
               MOVE WS-TODAY TO ACCT-OPEN-DATE
               MOVE WS-TODAY TO ACCT-LAST-STMT-DATE

               WRITE ACCOUNT-RECORD
                   INVALID KEY
                       DISPLAY "ERROR: could not write new account "
                           "(unexpected)."
                       MOVE "N" TO WS-VALID-FLAG
               END-WRITE
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Account " ACCT-ID " opened for "
                   FUNCTION TRIM(ACCT-NAME) " - balance="
                   ACCT-BALANCE
               CALL "WRTAUDIT" USING WS-TODAY WS-NOW ACCT-ID
                   WS-TXN-TYPE-OPEN ACCT-BALANCE ACCT-BALANCE
                   WS-STATUS-OK
           ELSE
               DISPLAY "Account NOT opened - no data was changed."
           END-IF.

           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

คอมไพล์ (ต้อง link กับ `wrtaudit.cob` เสมอ เพราะมีการ `CALL "WRTAUDIT"`):

```bash
cobc -x acctopen.cob wrtaudit.cob -o acctopen
```

### ทดสอบครบทุกเส้นทาง

```bash
printf -- "100001\nSOMCHAI JAIDEE\nS\n000050000\n" | ./acctopen
printf -- "100001\nDUPLICATE TEST\nS\n000010000\n" | ./acctopen
printf -- "100002\nSUDA MEECHAI\nC\n000075000\n" | ./acctopen
printf -- "100003\nBAD TYPE TEST\nX\n000010000\n" | ./acctopen
```

**ผลลัพธ์จริง:**

```
-- เปิดบัญชี 100001 สำเร็จ --
Account 100001 opened for SOMCHAI JAIDEE - balance=000000500.00

-- พยายามเปิดบัญชี 100001 ซ้ำ --
ERROR: account 100001 already exists.
Account NOT opened - no data was changed.

-- เปิดบัญชี 100002 สำเร็จ --
Account 100002 opened for SUDA MEECHAI - balance=000000750.00

-- ประเภทบัญชีไม่ถูกต้อง (X) --
ERROR: type must be S or C.
Account NOT opened - no data was changed.
```

### อธิบายจุดสำคัญ

- การตรวจสอบบัญชีซ้ำใช้เทคนิค `READ ... INVALID KEY / NOT INVALID KEY` (ทบทวนจาก Part 028 ขั้นตอนที่
  273) แบบ**สลับความหมาย**จากที่ใช้ปกติ: ปกติเรา `READ` เพื่อ "หาให้เจอ" แต่ที่นี่เรา `READ` เพื่อ
  "หวังว่าจะไม่เจอ" (`INVALID KEY` ที่แปลว่า "ยังไม่มี" คือกรณีที่ต้องการ)
- อัตราดอกเบี้ย 2.5% ต่อปี (`0.0250`) ถูกกำหนดอัตโนมัติสำหรับบัญชี Savings เท่านั้น บัญชี Checking ได้
  `0.0000` เสมอ — ตรรกะนี้จะถูกใช้จริงใน `BATCHINT` (ขั้นตอนที่ 697)
- สังเกตว่า **ทุกจุดที่ตรวจสอบไม่ผ่านจะทำให้ `WS-ALL-VALID` เป็น false และไปยังจุดสุดท้าย** (แสดง
  "Account NOT opened") โดยไม่มีเส้นทางใดที่ข้อมูลไม่สมบูรณ์จะไปถึงคำสั่ง `WRITE` ได้เลย — สถาปัตยกรรม
  "Validate then Trust" แบบเดียวกับที่เรียนใน Part 069 ขั้นตอนที่ 690

### ข้อควรระวัง

- โปรแกรมเปิดไฟล์ด้วย `OPEN I-O` (ไม่ใช่ `OPEN OUTPUT`) เพราะไฟล์ Master **มีข้อมูลอยู่แล้ว**จากการรัน
  ครั้งก่อน — การใช้ `OPEN OUTPUT` ผิดพลาดจะลบข้อมูลเดิมทั้งหมดทันที (ย้ำเตือนจากขั้นตอนที่ 691)
- `WS-AMOUNT-CENTS` รับค่าจากผู้ใช้เป็น**หน่วยสตางค์** (9 หลัก ไม่มีจุดทศนิยม) แล้วค่อยหารด้วย 100
  ก่อนเก็บลง `ACCT-BALANCE` — เทคนิคนี้ (ทบทวนจาก Part 069 ขั้นตอนที่ 690) หลีกเลี่ยงปัญหา `IS NUMERIC`
  ปฏิเสธจุดทศนิยม (`.`) เพราะจุดทศนิยมไม่ใช่ตัวเลข

### แบบฝึกหัดที่ 693.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบ "บัญชีซ้ำ" ต้องทำ**ก่อน**การถาม "ชื่อเจ้าของบัญชี" และ "จำนวนเงิน
เปิดบัญชี" แทนที่จะถามข้อมูลทั้งหมดให้ครบก่อนแล้วค่อยตรวจสอบทีเดียวตอนท้าย

**เฉลยแนวทาง**: การตรวจสอบเงื่อนไขที่ทำให้ transaction ล้มเหลว**แน่นอน**ให้เร็วที่สุดเท่าที่จะทำได้ (fail
fast) ช่วยประหยัดเวลาและความยุ่งยากของผู้ใช้งาน หากบัญชีซ้ำแล้ว การถามชื่อและจำนวนเงินเพิ่มเติมก่อนแจ้ง
error เป็นการเสียเวลาโดยเปล่าประโยชน์ (ผู้ใช้ต้องพิมพ์ข้อมูลทั้งหมดก่อนจะรู้ว่าล้มเหลว) ในขณะที่การตรวจสอบ
ก่อนทำให้ผู้ใช้ทราบปัญหาทันทีตั้งแต่ขั้นตอนแรก เป็นหลักการออกแบบ UX ที่ดีที่ใช้ได้ทั้งในโปรแกรม COBOL
แบบ text-mode และแอปพลิเคชันสมัยใหม่ทุกประเภท

---

## ขั้นตอนที่ 694: `ACCTDEP` — ฝากเงิน

### ซอร์สโค้ด: `acctdep.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCTDEP.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * ACCTDEP - online-style transaction: deposit into an account.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-RAW-AMOUNT            PIC X(9)   VALUE SPACES.
       01  WS-AMOUNT-CENTS          PIC 9(9)   VALUE 0.
       01  WS-AMOUNT                PIC 9(9)V99 VALUE 0.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".
       01  WS-TODAY                 PIC 9(8)   VALUE 20260401.
       01  WS-NOW                   PIC 9(6)   VALUE 91500.
       01  WS-TXN-TYPE-DEP          PIC X(4)   VALUE "DEP".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
       01  WS-STATUS-REJECTED       PIC X(8)   VALUE "REJECTED".
       01  WS-ZERO-AMOUNT           PIC 9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.

           DISPLAY "=== ACCTDEP: Deposit ===".
           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       DISPLAY "ERROR: account " WS-RAW-ID
                           " does not exist."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID AND NOT ACCT-STATUS-ACTIVE
               DISPLAY "ERROR: account " ACCT-ID " is closed."
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter deposit amount in cents (9 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-AMOUNT
               IF WS-RAW-AMOUNT IS NOT NUMERIC
                   DISPLAY "ERROR: amount must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
                   COMPUTE WS-AMOUNT = WS-AMOUNT-CENTS / 100
                   IF WS-AMOUNT-CENTS = 0
                       DISPLAY "ERROR: deposit amount must be "
                           "greater than zero."
                       MOVE "N" TO WS-VALID-FLAG
                   END-IF
               END-IF
           END-IF.

           IF WS-ALL-VALID
               ADD WS-AMOUNT TO ACCT-BALANCE
               REWRITE ACCOUNT-RECORD
                   INVALID KEY
                       DISPLAY "ERROR: rewrite failed (unexpected)."
                       MOVE "N" TO WS-VALID-FLAG
               END-REWRITE
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Deposit accepted. Account " ACCT-ID
                   " new balance=" ACCT-BALANCE
               CALL "WRTAUDIT" USING WS-TODAY WS-NOW ACCT-ID
                   WS-TXN-TYPE-DEP WS-AMOUNT ACCT-BALANCE
                   WS-STATUS-OK
           ELSE
               DISPLAY "Deposit REJECTED - no data was changed."
           END-IF.

           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

```bash
cobc -x acctdep.cob wrtaudit.cob -o acctdep
printf -- "100001\n000010050\n" | ./acctdep
printf -- "999999\n000010050\n" | ./acctdep
```

**ผลลัพธ์จริง:**

```
-- ฝาก 100.50 เข้าบัญชี 100001 (เดิม 500.00) --
Deposit accepted. Account 100001 new balance=000000600.50

-- ฝากเข้าบัญชีที่ไม่มีอยู่จริง --
ERROR: account 999999 does not exist.
Deposit REJECTED - no data was changed.
```

### อธิบายจุดสำคัญ

- ลำดับการทำงานคือ **READ ก่อน แก้ไขค่าในหน่วยความจำ แล้วค่อย REWRITE** — กฎเดิมที่เรียนมาตั้งแต่
  Part 025 ขั้นตอนที่ 245 และยังคงใช้ได้เหมือนเดิมกับไฟล์ Indexed (ทบทวนจาก Part 028 ขั้นตอนที่ 276)
- ตรวจสอบสถานะบัญชี (`NOT ACCT-STATUS-ACTIVE`) ก่อนอนุญาตให้ฝากเงิน — ป้องกันไม่ให้มีการทำรายการกับ
  บัญชีที่ปิดไปแล้ว เป็นกฎทางธุรกิจ (business rule) ที่สำคัญมากในระบบธนาคารจริง
- `500.00 + 100.50 = 600.50` — ผลลัพธ์ถูกต้องตรงกับการคำนวณด้วยมือทุกประการ

### ข้อควรระวัง

- `REWRITE` **ต้องตามหลัง `READ` ของ record เดียวกันในรอบการทำงานเดียวกันเสมอ** ห้าม `REWRITE` โดย
  ไม่ `READ` มาก่อน (ทบทวนกฎจาก Part 025 ขั้นตอนที่ 245) — โปรแกรมนี้ยึดกฎนี้เคร่งครัดทุกจุด

### แบบฝึกหัดที่ 694.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้ต้องตรวจสอบ `WS-AMOUNT-CENTS = 0` (ปฏิเสธการฝาก 0 บาท) ทั้งที่
`IS NUMERIC` ผ่านแล้ว (0 ก็เป็นตัวเลขที่ถูกต้อง)

**เฉลยแนวทาง**: `IS NUMERIC` ตรวจสอบแค่ "รูปแบบเป็นตัวเลขหรือไม่" เท่านั้น (ทบทวนจาก Part 069 ขั้นตอน
ที่ 686) แต่ไม่ได้ตรวจสอบว่า**ค่านั้นสมเหตุสมผลกับบริบททางธุรกิจหรือไม่** ในบริบทของการฝากเงิน จำนวนเงิน
0 บาทแม้จะเป็นตัวเลขที่ถูกต้องตามรูปแบบ แต่ไม่มีความหมายทางธุรกิจ (ไม่ใช่ "การฝากเงิน" จริง) และหาก
อนุญาตให้ผ่านไปได้ จะทำให้เกิดรายการ Audit Trail ที่ไม่มีความหมาย (DEP 0.00) ปะปนอยู่ในประวัติ ทำให้
รายงานและการตรวจสอบภายหลังยุ่งยากขึ้นโดยไม่จำเป็น การตรวจสอบขอบเขตค่าเพิ่มเติมแบบนี้คือสิ่งที่ต้องทำ
เสมอควบคู่กับการตรวจสอบรูปแบบ (ทบทวนหลักการเดียวกันจาก Part 069 ขั้นตอนที่ 687 เรื่อง subscript bounds)

---

## ขั้นตอนที่ 695: `ACCTWD` — ถอนเงิน (ป้องกันยอดติดลบ)

### ซอร์สโค้ด: `acctwd.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCTWD.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * ACCTWD - online-style transaction: withdraw from an account.
      * Rejects the withdrawal (does NOT touch the record) whenever
      * the account would go negative - COBOL's PIC 9 (unsigned) on
      * ACCT-BALANCE makes an accidental negative balance impossible
      * to store even by mistake, but we still check explicitly so
      * the customer gets a clear message instead of a SIZE ERROR.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-RAW-AMOUNT            PIC X(9)   VALUE SPACES.
       01  WS-AMOUNT-CENTS          PIC 9(9)   VALUE 0.
       01  WS-AMOUNT                PIC 9(9)V99 VALUE 0.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".
       01  WS-TODAY                 PIC 9(8)   VALUE 20260401.
       01  WS-NOW                   PIC 9(6)   VALUE 92000.
       01  WS-TXN-TYPE-WD           PIC X(4)   VALUE "WD".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.

           DISPLAY "=== ACCTWD: Withdrawal ===".
           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       DISPLAY "ERROR: account " WS-RAW-ID
                           " does not exist."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID AND NOT ACCT-STATUS-ACTIVE
               DISPLAY "ERROR: account " ACCT-ID " is closed."
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter withdrawal amount in cents (9 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-AMOUNT
               IF WS-RAW-AMOUNT IS NOT NUMERIC
                   DISPLAY "ERROR: amount must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
                   COMPUTE WS-AMOUNT = WS-AMOUNT-CENTS / 100
                   IF WS-AMOUNT-CENTS = 0
                       DISPLAY "ERROR: withdrawal amount must be "
                           "greater than zero."
                       MOVE "N" TO WS-VALID-FLAG
                   END-IF
               END-IF
           END-IF.

      *> The overdraft guard - checked BEFORE touching the balance,
      *> exactly like the bounds checks taught in Part 069 step 687.
           IF WS-ALL-VALID AND WS-AMOUNT > ACCT-BALANCE
               DISPLAY "ERROR: insufficient funds. Balance="
                   ACCT-BALANCE " requested=" WS-AMOUNT
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID
               SUBTRACT WS-AMOUNT FROM ACCT-BALANCE
               REWRITE ACCOUNT-RECORD
                   INVALID KEY
                       DISPLAY "ERROR: rewrite failed (unexpected)."
                       MOVE "N" TO WS-VALID-FLAG
               END-REWRITE
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Withdrawal accepted. Account " ACCT-ID
                   " new balance=" ACCT-BALANCE
               CALL "WRTAUDIT" USING WS-TODAY WS-NOW ACCT-ID
                   WS-TXN-TYPE-WD WS-AMOUNT ACCT-BALANCE
                   WS-STATUS-OK
           ELSE
               DISPLAY "Withdrawal REJECTED - no data was changed."
           END-IF.

           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

```bash
cobc -x acctwd.cob wrtaudit.cob -o acctwd
printf -- "100001\n000005000\n" | ./acctwd
printf -- "100001\n999999999\n" | ./acctwd
```

**ผลลัพธ์จริง:**

```
-- ถอน 50.00 จากบัญชี 100001 (เดิม 600.50) --
Withdrawal accepted. Account 100001 new balance=000000550.50

-- ถอนเกินยอดคงเหลือ --
ERROR: insufficient funds. Balance=000000550.50 requested=009999999.99
Withdrawal REJECTED - no data was changed.
```

### อธิบายจุดสำคัญ

- `ACCT-BALANCE` ประกาศเป็น `PIC 9(9)V99` (**ไม่มี** `S` นำหน้า แปลว่าไม่มีเครื่องหมาย) ทำให้**ไม่มีทาง
  เก็บค่าติดลบได้เลยแม้แต่ทางเทคนิค** — นี่คือการใช้ชนิดข้อมูลเพื่อบังคับกฎทางธุรกิจตั้งแต่ระดับโครงสร้าง
  ข้อมูล (ทบทวนแนวคิดนี้จาก Part 006) แต่โปรแกรมยังคง**ตรวจสอบด้วย `IF` อย่างชัดเจน**ก่อนเสมอ เพื่อให้
  ลูกค้าได้รับข้อความ error ที่สื่อความหมายชัดเจน แทนที่จะปล่อยให้เกิด `SIZE ERROR` ที่ไม่เป็นมิตร
- 600.50 - 50.00 = 550.50 ถูกต้อง และการถอน 9,999,999.99 (เกินยอด) ถูกปฏิเสธอย่างชัดเจนโดยไม่มีการ
  แก้ไขข้อมูลใด ๆ เลย

### ข้อควรระวัง

- ระวังการเปรียบเทียบ `WS-AMOUNT > ACCT-BALANCE` ต้องทำ**ก่อน** `SUBTRACT` เสมอ ไม่ใช่ `SUBTRACT`
  ไปก่อนแล้วค่อยตรวจสอบผลลัพธ์ทีหลัง (เพราะ `ACCT-BALANCE` แบบไม่มีเครื่องหมายจะไม่มีทาง "แสดง" ค่าติดลบ
  ออกมาให้ตรวจสอบได้เลย ต้องป้องกันไว้ก่อนเท่านั้น)

### แบบฝึกหัดที่ 695.1

**โจทย์**: จงอธิบายว่าจะเกิดอะไรขึ้นหากโปรแกรมนี้ใช้ `ACCT-BALANCE PIC S9(9)V99` (มีเครื่องหมาย) แทน
`PIC 9(9)V99` (ไม่มีเครื่องหมาย) แล้วลืมตรวจสอบเงื่อนไข `WS-AMOUNT > ACCT-BALANCE` ก่อน `SUBTRACT`

**เฉลยแนวทาง**: หากใช้ `PIC S9(9)V99` โปรแกรมจะสามารถ `SUBTRACT` จนได้ค่าติดลบได้จริงโดยไม่เกิด
error ใด ๆ เลย (เพราะฟิลด์รองรับเครื่องหมายลบ) ทำให้บัญชีมียอดติดลบซึ่งขัดกับกฎทางธุรกิจของระบบธนาคาร
โดยสิ้นเชิง และที่อันตรายยิ่งกว่าคือ**ไม่มี error ใด ๆ เตือนเลย** เพราะฟิลด์รองรับค่านั้นได้ทางเทคนิค
เปรียบเทียบกับการใช้ `PIC 9(9)V99` (ไม่มีเครื่องหมาย) ที่แม้จะลืมตรวจสอบ ก็ยังมีโอกาสสูงที่จะเกิด
`SIZE ERROR`หรือพฤติกรรม runtime ที่ผิดปกติชัดเจนเมื่อพยายามเก็บค่าติดลบ ทำให้ตรวจพบปัญหาได้ง่ายกว่า
มาก — นี่คือตัวอย่างของหลักการ "Defense in Depth" ที่เรียนมาใน Part 069: การเลือกชนิดข้อมูลให้ถูกต้อง
คือชั้นป้องกันหนึ่ง และการตรวจสอบด้วย `IF` อย่างชัดเจนคืออีกชั้นหนึ่งที่ต้องมีควบคู่กันเสมอ ไม่ควรพึ่งพา
เพียงอย่างใดอย่างหนึ่ง

---

## ขั้นตอนที่ 696: `ACCTINQ` — สอบถามยอด และ `ACCTCLOSE` — ปิดบัญชี

### `ACCTINQ.cob` — สอบถามยอด (Read-only) พร้อม Data Masking

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCTINQ.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * ACCTINQ - online-style transaction: balance inquiry (read-only)
      * Masks the account id in its own confirmation line, following
      * the data-masking technique from Part 069 step 689, and still
      * logs the inquiry itself to the audit trail - many real banking
      * regulations require logging WHO looked at an account, not just
      * who changed it.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-MASKED-ID             PIC X(6)   VALUE SPACES.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".
       01  WS-TODAY                 PIC 9(8)   VALUE 20260401.
       01  WS-NOW                   PIC 9(6)   VALUE 93000.
       01  WS-TXN-TYPE-INQ          PIC X(4)   VALUE "INQ".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
       01  WS-ZERO-AMOUNT           PIC 9(9)V99 VALUE 0.
       01  WS-STATUS-TEXT           PIC X(6)   VALUE SPACES.
       01  WS-TYPE-TEXT             PIC X(8)   VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN INPUT ACCOUNT-MASTER.

           DISPLAY "=== ACCTINQ: Balance Inquiry ===".
           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       DISPLAY "ERROR: account " WS-RAW-ID
                           " does not exist."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID
      *> Mask everything but the last 3 digits before printing.
               MOVE "XXX"   TO WS-MASKED-ID(1:3)
               MOVE ACCT-ID(4:3) TO WS-MASKED-ID(4:3)

               IF ACCT-STATUS-ACTIVE
                   MOVE "ACTIVE" TO WS-STATUS-TEXT
               ELSE
                   MOVE "CLOSED" TO WS-STATUS-TEXT
               END-IF

               IF ACCT-TYPE-SAVINGS
                   MOVE "SAVINGS " TO WS-TYPE-TEXT
               ELSE
                   MOVE "CHECKING" TO WS-TYPE-TEXT
               END-IF

               DISPLAY "Account ****" WS-MASKED-ID
                   " (" FUNCTION TRIM(ACCT-NAME) ") "
                   FUNCTION TRIM(WS-TYPE-TEXT) " "
                   FUNCTION TRIM(WS-STATUS-TEXT)
               DISPLAY "  Balance = " ACCT-BALANCE

               CALL "WRTAUDIT" USING WS-TODAY WS-NOW ACCT-ID
                   WS-TXN-TYPE-INQ WS-ZERO-AMOUNT ACCT-BALANCE
                   WS-STATUS-OK
           END-IF.

           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

### `ACCTCLOSE.cob` — ปิดบัญชี (บังคับให้ยอดต้องเป็นศูนย์ก่อน)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCTCLOSE.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * ACCTCLOSE - online-style transaction: close an account.
      * Business rule: an account can only be closed when its balance
      * is exactly zero - this prevents money from silently vanishing
      * when an account is closed, a classic banking control.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".
       01  WS-TODAY                 PIC 9(8)   VALUE 20260401.
       01  WS-NOW                   PIC 9(6)   VALUE 94500.
       01  WS-TXN-TYPE-CLOS         PIC X(4)   VALUE "CLOS".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".
       01  WS-ZERO-AMOUNT           PIC 9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.

           DISPLAY "=== ACCTCLOSE: Close Account ===".
           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       DISPLAY "ERROR: account " WS-RAW-ID
                           " does not exist."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID AND NOT ACCT-STATUS-ACTIVE
               DISPLAY "ERROR: account " ACCT-ID
                   " is already closed."
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID AND ACCT-BALANCE NOT = ZERO
               DISPLAY "ERROR: account " ACCT-ID
                   " balance is not zero (" ACCT-BALANCE
                   ") - withdraw remaining funds before closing."
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID
               SET ACCT-STATUS-CLOSED TO TRUE
               REWRITE ACCOUNT-RECORD
                   INVALID KEY
                       DISPLAY "ERROR: rewrite failed (unexpected)."
                       MOVE "N" TO WS-VALID-FLAG
               END-REWRITE
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Account " ACCT-ID " closed successfully."
               CALL "WRTAUDIT" USING WS-TODAY WS-NOW ACCT-ID
                   WS-TXN-TYPE-CLOS WS-ZERO-AMOUNT ACCT-BALANCE
                   WS-STATUS-OK
           ELSE
               DISPLAY "Close REJECTED - no data was changed."
           END-IF.

           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

```bash
cobc -x acctinq.cob wrtaudit.cob -o acctinq
cobc -x acctclose.cob wrtaudit.cob -o acctclose

printf -- "100001\n" | ./acctinq
printf -- "100001\n" | ./acctclose
```

**ผลลัพธ์จริง:**

```
-- สอบถามยอด 100001 (ยอด 550.50) --
Account ****XXX001 (SOMCHAI JAIDEE) SAVINGS ACTIVE
  Balance = 000000550.50

-- พยายามปิดบัญชี 100001 (ยังมียอดคงเหลือ) --
ERROR: account 100001 balance is not zero (000000550.50) - withdraw remaining funds before closing.
Close REJECTED - no data was changed.
```

หลังจากถอนเงินจนยอดเป็นศูนย์แล้วลองปิดใหม่:

```bash
printf -- "100001\n000055050\n" | ./acctwd
printf -- "100001\n" | ./acctclose
printf -- "100001\n" | ./acctinq
```

```
Withdrawal accepted. Account 100001 new balance=000000000.00
Account 100001 closed successfully.
Account ****XXX001 (SOMCHAI JAIDEE) SAVINGS CLOSED
  Balance = 000000000.00
```

### อธิบายจุดสำคัญ

- `ACCTINQ` เปิดไฟล์ด้วย `OPEN INPUT` (ไม่ใช่ `OPEN I-O`) เพราะเป็นการอ่านอย่างเดียว ไม่มีการแก้ไข
  ข้อมูลใด ๆ — เลือกโหมด `OPEN` ให้ตรงกับการใช้งานจริงเสมอ (ทบทวนจาก Part 025 ขั้นตอนที่ 241)
- Mask แบบ "แสดงเฉพาะ 3 หลักสุดท้าย" (`****XXX001`) ใช้เทคนิค Reference Modification ตรงตามที่เรียน
  ใน Part 069 ขั้นตอนที่ 689
- `ACCTCLOSE` ตรวจสอบ `ACCT-BALANCE NOT = ZERO` **ก่อน** `SET ACCT-STATUS-CLOSED` เสมอ — กฎทาง
  ธุรกิจนี้ป้องกันเงินหายไปเงียบ ๆ เมื่อปิดบัญชี (เงินที่เหลืออยู่จะไม่ถูก "ลบทิ้ง" ไปพร้อมกับการปิดบัญชี)

### ข้อควรระวัง

- `ACCTINQ` ก็ยัง `CALL "WRTAUDIT"` แม้จะเป็นแค่การอ่านข้อมูล — การบันทึก "ใครเข้าดูข้อมูลอะไรเมื่อไร"
  เป็นข้อกำหนดจริงในหลายกฎระเบียบธนาคาร (Regulatory Compliance) ไม่ใช่แค่การบันทึกเฉพาะรายการที่
  เปลี่ยนแปลงข้อมูล

### แบบฝึกหัดที่ 696.1

**โจทย์**: จงอธิบายว่าทำไมกฎ "ต้องยอดเป็นศูนย์ก่อนปิดบัญชี" ใน `ACCTCLOSE` จึงสำคัญกว่าการปิดบัญชีแล้ว
"คืนเงินที่เหลือให้ลูกค้าโดยอัตโนมัติ" ที่ดูสะดวกกว่าสำหรับผู้ใช้

**เฉลยแนวทาง**: การ "คืนเงินอัตโนมัติ" ต้องมีขั้นตอนเพิ่มเติมที่ซับซ้อนกว่ามาก (เช่น ต้องรู้ว่าจะคืนเงินไป
ที่ไหน เป็นเงินสดหรือโอนเข้าบัญชีอื่น ต้องมีการอนุมัติหรือไม่ ต้องบันทึกเป็นธุรกรรมแยกต่างหากที่มีร่องรอย
ชัดเจน) ซึ่งอยู่นอกเหนือขอบเขตของคำสั่ง "ปิดบัญชี" เพียงคำสั่งเดียว การบังคับให้ผู้ใช้**ถอนเงินออกก่อนด้วย
transaction แยกต่างหาก** (`ACCTWD`) ทำให้ทุกการเคลื่อนไหวของเงินมีร่องรอยชัดเจนแยกจากกันใน Audit Trail
(ทบทวนความสำคัญของ Audit Trail จาก Part 069 คำนำ) และทำให้ตรรกะของ `ACCTCLOSE` เรียบง่าย ตรวจสอบ
ได้ง่าย และมีความเสี่ยงต่ำกว่ามาก — หลักการ "แต่ละโปรแกรมทำหน้าที่เดียวให้ดีที่สุด" (Single Responsibility)
นี้เป็นแนวทางการออกแบบที่ดีในระบบธุรกิจจริงเสมอ

---

## ขั้นตอนที่ 697: `BATCHINT` — Batch คำนวณดอกเบี้ยรายคืน พร้อม Control Total Report

### แนวคิด

นี่คือโปรแกรมแรกที่เป็น **Batch** อย่างแท้จริง (ทบทวนแนวคิด Batch Processing Pattern จาก Part 066):
แทนที่จะรับ input จากผู้ใช้ทีละคน โปรแกรมนี้**อ่านทุก record ในไฟล์ Master เรียงตามลำดับ** (`READ ...
NEXT RECORD`, ทบทวนจาก Part 028 ขั้นตอนที่ 271) แล้วประมวลผล**ทุกบัญชี**ในการรันครั้งเดียว — รูปแบบ
เดียวกับที่ Batch Job รายคืนบน Mainframe จริงทำงาน

### ซอร์สโค้ด: `batchint.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BATCHINT.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * BATCHINT - nightly batch job: interest calculation.
      * This is the "batch window" side of the system - a job that
      * would run once every night on a real Mainframe (Part 066),
      * scheduled through JCL (Part 052-053), touching every account
      * in one sequential sweep rather than one at a time online.
      * We read the INDEXED master with READ ... NEXT RECORD in key
      * order (Part 028), which is exactly how a batch program reads
      * a VSAM KSDS from start to finish.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-EOF-FLAG              PIC X(1)   VALUE "N".
           88  WS-END-OF-FILE               VALUE "Y".

       01  WS-TODAY                 PIC 9(8)   VALUE 20260430.
       01  WS-NOW                   PIC 9(6)   VALUE 10000.
       01  WS-TXN-TYPE-INT          PIC X(4)   VALUE "INT".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".

       01  WS-INTEREST-AMOUNT       PIC 9(9)V99 VALUE 0.

      *> Control totals - the batch-processing equivalent of the
      *> secure-coding validation checks from Part 069: prove the
      *> job did exactly what it should, nothing more, nothing less.
       01  WS-ACCOUNTS-READ         PIC 9(5)    VALUE 0.
       01  WS-SAVINGS-PROCESSED     PIC 9(5)    VALUE 0.
       01  WS-CHECKING-SKIPPED      PIC 9(5)    VALUE 0.
       01  WS-CLOSED-SKIPPED        PIC 9(5)    VALUE 0.
       01  WS-TOTAL-INTEREST-PAID   PIC 9(9)V99 VALUE 0.
       01  WS-OPENING-TOTAL-BAL     PIC 9(11)V99 VALUE 0.
       01  WS-CLOSING-TOTAL-BAL     PIC 9(11)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM 1000-OPEN-FILES.
           PERFORM UNTIL WS-END-OF-FILE
               READ ACCOUNT-MASTER NEXT RECORD
                   AT END
                       SET WS-END-OF-FILE TO TRUE
                   NOT AT END
                       PERFORM 2000-PROCESS-ONE-ACCOUNT
               END-READ
           END-PERFORM.
           PERFORM 3000-PRINT-CONTROL-REPORT.
           CLOSE ACCOUNT-MASTER.
           STOP RUN.

       1000-OPEN-FILES.
           OPEN I-O ACCOUNT-MASTER.
           DISPLAY "=== BATCHINT: Nightly Interest Batch Run ===".
           DISPLAY "Run date: " WS-TODAY.

       2000-PROCESS-ONE-ACCOUNT.
           ADD 1 TO WS-ACCOUNTS-READ.
           ADD ACCT-BALANCE TO WS-OPENING-TOTAL-BAL.

           IF NOT ACCT-STATUS-ACTIVE
               ADD 1 TO WS-CLOSED-SKIPPED
           ELSE
               IF ACCT-TYPE-SAVINGS
      *> Simple monthly interest: annual rate / 12, applied to the
      *> CURRENT balance. Intentionally simple for a teaching demo -
      *> real systems have far more elaborate day-count rules.
                   COMPUTE WS-INTEREST-AMOUNT ROUNDED =
                       ACCT-BALANCE * ACCT-INTEREST-RATE / 12
                   ADD WS-INTEREST-AMOUNT TO ACCT-BALANCE
                   ADD WS-INTEREST-AMOUNT TO WS-TOTAL-INTEREST-PAID
                   ADD 1 TO WS-SAVINGS-PROCESSED
                   MOVE WS-TODAY TO ACCT-LAST-STMT-DATE

                   REWRITE ACCOUNT-RECORD
                       INVALID KEY
                           DISPLAY "ERROR: rewrite failed for "
                               "account " ACCT-ID
                   END-REWRITE

                   CALL "WRTAUDIT" USING WS-TODAY WS-NOW ACCT-ID
                       WS-TXN-TYPE-INT WS-INTEREST-AMOUNT
                       ACCT-BALANCE WS-STATUS-OK
               ELSE
                   ADD 1 TO WS-CHECKING-SKIPPED
               END-IF
           END-IF.

           ADD ACCT-BALANCE TO WS-CLOSING-TOTAL-BAL.

       3000-PRINT-CONTROL-REPORT.
           DISPLAY " ".
           DISPLAY "---- BATCHINT CONTROL TOTAL REPORT ----".
           DISPLAY "Accounts read from master     : "
               WS-ACCOUNTS-READ.
           DISPLAY "Savings accounts credited      : "
               WS-SAVINGS-PROCESSED.
           DISPLAY "Checking accounts skipped       : "
               WS-CHECKING-SKIPPED.
           DISPLAY "Closed accounts skipped        : "
               WS-CLOSED-SKIPPED.
           DISPLAY "Total interest paid this run   : "
               WS-TOTAL-INTEREST-PAID.
           DISPLAY "Opening total balance (all accts): "
               WS-OPENING-TOTAL-BAL.
           DISPLAY "Closing total balance (all accts): "
               WS-CLOSING-TOTAL-BAL.
           IF WS-CLOSING-TOTAL-BAL =
                   WS-OPENING-TOTAL-BAL + WS-TOTAL-INTEREST-PAID
               DISPLAY "Control total check: BALANCED"
           ELSE
               DISPLAY "Control total check: *** OUT OF BALANCE ***"
           END-IF.
```

```bash
cobc -x batchint.cob wrtaudit.cob -o batchint
./batchint
```

**ผลลัพธ์จริง (รันกับ 3 บัญชี: 100001 Savings, 100002 Checking, 100003 Savings):**

```
=== BATCHINT: Nightly Interest Batch Run ===
Run date: 20260430
 
---- BATCHINT CONTROL TOTAL REPORT ----
Accounts read from master     : 00003
Savings accounts credited      : 00002
Checking accounts skipped       : 00001
Closed accounts skipped        : 00000
Total interest paid this run   : 000000003.23
Opening total balance (all accts): 00000002300.50
Closing total balance (all accts): 00000002303.73
Control total check: BALANCED
```

### อธิบายจุดสำคัญ

- `READ ACCOUNT-MASTER NEXT RECORD` ไล่อ่านทุก record ตามลำดับ Key (ไม่ใช่การค้นหาแบบสุ่มเหมือน
  โปรแกรม online) — วนลูปจนกว่าจะถึง `AT END` แล้วประมวลผลทุกบัญชีในรอบเดียว
- **Control Total** คือหัวใจของ Batch Processing ที่เชื่อถือได้: โปรแกรมนับและสะสมยอดทั้ง**ก่อน**และ
  **หลัง**การประมวลผล แล้วตรวจสอบว่า `ยอดปิด = ยอดเปิด + ดอกเบี้ยที่จ่าย` ตรงกันหรือไม่ — ถ้าตัวเลขไม่
  ตรงกัน (`OUT OF BALANCE`) แปลว่ามีบางอย่างผิดพลาดในตรรกะของโปรแกรม (ทบทวนแนวคิดนี้เชื่อมโยงกับ
  SMF/RMF และการวิเคราะห์ปัญหาจาก Part 068 ขั้นตอนที่ 679)
- คำนวณดอกเบี้ยรายเดือน: `ยอดคงเหลือ × อัตราดอกเบี้ยต่อปี ÷ 12` พร้อม `ROUNDED` — 1000.00 × 0.0250 ÷
  12 = 2.0833... ปัดเป็น 2.08 (ทบทวนการปัดเศษจาก Part 009)

### ข้อควรระวัง

- โปรแกรมนี้เปิดไฟล์ด้วย `OPEN I-O` เพราะต้องทั้งอ่านและ `REWRITE` — ถ้าเปิดด้วย `OPEN INPUT` โปรแกรม
  จะ compile ผ่านแต่ `REWRITE` จะ error ทันทีตอนรัน (ทบทวนกฎจาก Part 025 ขั้นตอนที่ 241)
- Control Total ต้องคำนวณจาก**ค่าที่อ่านได้จริงระหว่างการประมวลผล** ไม่ใช่คำนวณแยกต่างหากทีหลัง — ถ้า
  แยกคำนวณ อาจไม่สะท้อนสิ่งที่โปรแกรมทำจริงหากมีบั๊กในตรรกะหลัก

### แบบฝึกหัดที่ 697.1

**โจทย์**: จงอธิบายว่าทำไม Control Total Report ในขั้นตอนนี้จึงยังคงมีประโยชน์ แม้ในกรณีที่โปรแกรมไม่มี
บั๊กเลยแม้แต่จุดเดียว

**เฉลยแนวทาง**: Control Total ไม่ได้มีประโยชน์แค่ตอนที่ **สงสัยว่ามีบั๊ก** เท่านั้น แต่ยังเป็นหลักฐานยืนยัน
(evidence) ต่อผู้ตรวจสอบบัญชี (auditor) และทีมปฏิบัติการว่า batch job รันสำเร็จและครบถ้วนจริงในแต่ละครั้ง
โดยไม่ต้องเชื่อคำบอกเล่าเฉย ๆ — ในองค์กรการเงินจริง รายงานแบบนี้มักถูกเก็บเป็นหลักฐานสำหรับการตรวจสอบ
ย้อนหลัง (audit trail ระดับ job ไม่ใช่แค่ระดับ transaction) และบางครั้งต้องมีการเซ็นชื่อรับรองโดยเจ้าหน้าที่
ที่มีอำนาจก่อนที่ผลของ batch run จะถือว่า "ผ่านการรับรอง" อย่างเป็นทางการ Control Total จึงเป็นส่วนหนึ่ง
ของกระบวนการควบคุมภายใน (internal control) ไม่ใช่แค่เครื่องมือ debug เท่านั้น

---

## ขั้นตอนที่ 698: `STMTGEN` — สร้างใบแจ้งยอดลูกค้า (SORT + Control Break + Master-Detail)

### แนวคิด

โปรแกรมนี้สาธิตรูปแบบ Batch ที่ซับซ้อนที่สุดของโปรเจกต์: อ่าน Audit Trail (Sequential, "Detail
records" ตามศัพท์ Master-Detail จาก Part 026), จัดเรียงด้วย `SORT` (Part 027), แล้วพิมพ์รายงานแบบ
**Control Break** (Part 039) ที่ขึ้นหัวใบแจ้งยอดใหม่ทุกครั้งที่เปลี่ยนบัญชี พร้อมค้นชื่อเจ้าของบัญชีจาก
Indexed Master (Part 028)

### กับดักจริงที่พบระหว่างพัฒนา (บทเรียนสำคัญที่ควรอ่านก่อนโค้ด)

ระหว่างพัฒนาโปรแกรมนี้ พบบั๊กจริงที่คุ้มค่าแก่การบันทึกไว้: `TXNLOG.DAT` เก็บจำนวนเงินเป็นข้อความ (เช่น
`"00000000208"` แทนค่า 2.08 ด้วย field `PIC 9(9)V99`) การแปลงข้อความนี้กลับเป็นตัวเลขทำโดย **Reference
Modification** (Part 019) ตัด substring 11 ตัวอักษรออกมา แล้ว `MOVE` ตรงเข้าฟิลด์ `PIC 9(9)V99`
โดยตรง — **ผลลัพธ์ที่ได้ผิดพลาดไปถึง 100 เท่า!** (208.00 แทนที่จะเป็น 2.08)

สาเหตุคือกฎการ `MOVE` ของ COBOL: **เมื่อย้ายข้อมูลจากฟิลด์ตัวอักษร (`PIC X`, รวมถึงผลลัพธ์ของ Reference
Modification) เข้าฟิลด์ตัวเลขที่มี `V`** COBOL จะจัดตำแหน่งข้อมูลราวกับว่าฟิลด์ต้นทางเป็น**จำนวนเต็ม**
(ไม่มีจุดทศนิยมโดยนัย) แล้วจึงจัดตำแหน่งตามจุดทศนิยมของฟิลด์ปลายทาง ทำให้ตัวเลข 11 หลักที่ควรจะเป็น
"9 หลักเต็ม + 2 หลักทศนิยม" กลับถูกตีความเป็น "จำนวนเต็ม 11 หลัก" แล้วเบียดเข้าไปในโครงสร้าง 9+2 อย่าง
ผิดตำแหน่ง — ค่าจึงคลาดเคลื่อนไป 100 เท่าโดยไม่มี compile error หรือ runtime error ใด ๆ เตือนเลย

**วิธีแก้ที่ถูกต้อง**: ต้องย้ายข้อมูลตัวอักษรเข้าฟิลด์ **ตัวเลขจำนวนเต็มล้วน** (`PIC 9(11)` ไม่มี `V`)
ก่อนเสมอ แล้วจึงใช้ `COMPUTE` หารด้วย 100 เพื่อปรับสเกลให้ถูกต้องด้วยตัวเราเอง

### ซอร์สโค้ด: `stmtgen.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STMTGEN.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * STMTGEN - nightly batch job: customer statement generation.
      * Demonstrates a classic Mainframe batch pattern: read the
      * sequential audit trail (the "detail" records, Part 026
      * Master-Detail terminology), SORT it into account/date order
      * (Part 027), then print one statement per account with a
      * control break (Part 039) whenever the account id changes,
      * looking up the account NAME from the indexed master (Part 028)
      * along the way.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT RAW-TXN-FILE ASSIGN TO "TXNLOG.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

           SELECT PARSED-FILE ASSIGN TO "STMTPRS.TMP"
               ORGANIZATION IS SEQUENTIAL.

           SELECT SORTED-FILE ASSIGN TO "STMTSRT.TMP"
               ORGANIZATION IS SEQUENTIAL.

           SELECT SORT-WORK-FILE ASSIGN TO "STMTWRK.TMP".

           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-MASTER-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  RAW-TXN-FILE.
       01  RAW-TXN-LINE             PIC X(80).

       FD  PARSED-FILE.
       01  PARSED-RECORD.
           05  PR-ACCT-ID           PIC 9(6).
           05  PR-DATE              PIC 9(8).
           05  PR-TIME              PIC 9(6).
           05  PR-TYPE              PIC X(4).
           05  PR-AMOUNT            PIC 9(9)V99.
           05  PR-BALANCE           PIC 9(9)V99.
           05  PR-STATUS            PIC X(8).

       FD  SORTED-FILE.
       01  SORTED-RECORD.
           05  SR-ACCT-ID           PIC 9(6).
           05  SR-DATE              PIC 9(8).
           05  SR-TIME              PIC 9(6).
           05  SR-TYPE              PIC X(4).
           05  SR-AMOUNT            PIC 9(9)V99.
           05  SR-BALANCE           PIC 9(9)V99.
           05  SR-STATUS            PIC X(8).

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SK-ACCT-ID           PIC 9(6).
           05  SK-DATE              PIC 9(8).
           05  SK-TIME              PIC 9(6).
           05  SK-TYPE              PIC X(4).
           05  SK-AMOUNT            PIC 9(9)V99.
           05  SK-BALANCE           PIC 9(9)V99.
           05  SK-STATUS            PIC X(8).

       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-MASTER-STATUS         PIC X(2).
       01  WS-EOF-FLAG              PIC X(1) VALUE "N".
           88  WS-END-OF-FILE               VALUE "Y".
       01  WS-FIRST-RECORD-FLAG     PIC X(1) VALUE "Y".
           88  WS-IS-FIRST-RECORD           VALUE "Y".
       01  WS-BREAK-ACCT-ID         PIC 9(6) VALUE 0.
       01  WS-STMT-COUNT            PIC 9(3) VALUE 0.
       01  WS-GRAND-TXN-COUNT       PIC 9(5) VALUE 0.

      *> IMPORTANT: RAW-TXN-LINE is alphanumeric (PIC X). MOVing an
      *> alphanumeric substring straight into a PIC 9(9)V99 field is
      *> a classic trap: COBOL aligns an alphanumeric-to-numeric MOVE
      *> as if the source were an INTEGER (no implied decimal point),
      *> so an 11-digit string meant to be "9 integer + 2 decimal"
      *> lands as a plain 11-digit integer instead - the value comes
      *> out 100 times too large. The fix is to land it in a PLAIN
      *> INTEGER field first, then COMPUTE the real value by dividing
      *> by 100 ourselves.
       01  WS-RAW-AMOUNT-INT        PIC 9(11) VALUE 0.
       01  WS-RAW-BALANCE-INT       PIC 9(11) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM 1000-PARSE-AUDIT-TRAIL.
           PERFORM 2000-SORT-BY-ACCOUNT.
           PERFORM 3000-PRINT-STATEMENTS.
           DISPLAY " ".
           DISPLAY "STMTGEN complete. Statements printed: "
               WS-STMT-COUNT " Total transactions: "
               WS-GRAND-TXN-COUNT.
           STOP RUN.

       1000-PARSE-AUDIT-TRAIL.
      *> Turn the human-readable text audit trail into a fixed-field
      *> file the SORT statement can key on (Reference Modification,
      *> Part 019, applied at the FIXED byte offsets WRTAUDIT always
      *> writes at).
           OPEN INPUT RAW-TXN-FILE.
           OPEN OUTPUT PARSED-FILE.
           PERFORM UNTIL WS-END-OF-FILE
               READ RAW-TXN-FILE
                   AT END
                       SET WS-END-OF-FILE TO TRUE
                   NOT AT END
                       MOVE RAW-TXN-LINE(1:8)   TO PR-DATE
                       MOVE RAW-TXN-LINE(10:6)  TO PR-TIME
                       MOVE RAW-TXN-LINE(17:6)  TO PR-ACCT-ID
                       MOVE RAW-TXN-LINE(24:4)  TO PR-TYPE
      *> See the WS-RAW-AMOUNT-INT comment above - land as an
      *> integer first, THEN rescale by dividing by 100.
                       MOVE RAW-TXN-LINE(29:11) TO WS-RAW-AMOUNT-INT
                       COMPUTE PR-AMOUNT = WS-RAW-AMOUNT-INT / 100
                       MOVE RAW-TXN-LINE(41:11) TO WS-RAW-BALANCE-INT
                       COMPUTE PR-BALANCE = WS-RAW-BALANCE-INT / 100
                       MOVE RAW-TXN-LINE(53:8)  TO PR-STATUS
                       WRITE PARSED-RECORD
               END-READ
           END-PERFORM.
           CLOSE RAW-TXN-FILE.
           CLOSE PARSED-FILE.

       2000-SORT-BY-ACCOUNT.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SK-ACCT-ID
               ON ASCENDING KEY SK-DATE
               ON ASCENDING KEY SK-TIME
               USING PARSED-FILE
               GIVING SORTED-FILE.
           MOVE "N" TO WS-EOF-FLAG.

       3000-PRINT-STATEMENTS.
           OPEN INPUT SORTED-FILE.
           OPEN INPUT ACCOUNT-MASTER.
           PERFORM UNTIL WS-END-OF-FILE
               READ SORTED-FILE
                   AT END
                       SET WS-END-OF-FILE TO TRUE
                   NOT AT END
                       PERFORM 3100-PROCESS-DETAIL-LINE
               END-READ
           END-PERFORM.
           CLOSE SORTED-FILE.
           CLOSE ACCOUNT-MASTER.

       3100-PROCESS-DETAIL-LINE.
      *> Control break: a new account STARTS a new statement, the
      *> exact same pattern taught for control breaks in Part 039.
           IF WS-IS-FIRST-RECORD OR SR-ACCT-ID NOT = WS-BREAK-ACCT-ID
               PERFORM 3200-PRINT-STATEMENT-HEADER
               MOVE SR-ACCT-ID TO WS-BREAK-ACCT-ID
               MOVE "N" TO WS-FIRST-RECORD-FLAG
           END-IF.
           DISPLAY "    " SR-DATE " " SR-TIME " " SR-TYPE
               " amount=" SR-AMOUNT " balance-after=" SR-BALANCE
               " " FUNCTION TRIM(SR-STATUS).
           ADD 1 TO WS-GRAND-TXN-COUNT.

       3200-PRINT-STATEMENT-HEADER.
           ADD 1 TO WS-STMT-COUNT.
           MOVE SR-ACCT-ID TO ACCT-ID.
           READ ACCOUNT-MASTER
               INVALID KEY
                   MOVE "(ACCOUNT NOT FOUND)" TO ACCT-NAME
           END-READ.
           DISPLAY " ".
           DISPLAY "==== STATEMENT: Account " SR-ACCT-ID
               " (" FUNCTION TRIM(ACCT-NAME) ") ====".
```

```bash
cobc -x stmtgen.cob -o stmtgen
./stmtgen
```

**ผลลัพธ์จริง (หลังแก้บั๊กแล้ว — ค่าถูกต้องทุกรายการ):**

```
==== STATEMENT: Account 100001 (SOMCHAI JAIDEE) ====
    20260401 090000 OPEN amount=000000500.00 balance-after=000000500.00 OK
    20260401 091500 DEP  amount=000000100.50 balance-after=000000600.50 OK
    20260401 092000 WD   amount=000000050.00 balance-after=000000550.50 OK
    20260430 010000 INT  amount=000000001.15 balance-after=000000551.65 OK
 
==== STATEMENT: Account 100002 (SUDA MEECHAI) ====
    20260401 090000 OPEN amount=000000750.00 balance-after=000000750.00 OK
    20260401 092000 WD   amount=000000750.00 balance-after=000000000.00 OK
    20260401 094500 CLOS amount=000000000.00 balance-after=000000000.00 OK
 
==== STATEMENT: Account 100003 (PRASERT KAEWTA) ====
    20260401 090000 OPEN amount=000001000.00 balance-after=000001000.00 OK
    20260401 093000 INQ  amount=000000000.00 balance-after=000001000.00 OK
    20260430 010000 INT  amount=000000002.08 balance-after=000001002.08 OK
 
STMTGEN complete. Statements printed: 003 Total transactions: 00010
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้มี**สามขั้นตอนเรียงกัน**: (1) `1000-PARSE-AUDIT-TRAIL` แปลงข้อความเป็น record คงที่
  (2) `2000-SORT-BY-ACCOUNT` เรียงลำดับด้วย `SORT ... USING ... GIVING` (3) `3000-PRINT-STATEMENTS`
  อ่านผลลัพธ์ที่เรียงแล้วมาพิมพ์แบบ Control Break — สังเกตว่าแม้ในไฟล์ต้นฉบับรายการของบัญชี 100002
  (`WD`, `CLOS`) จะแทรกอยู่ปะปนกับบัญชีอื่น แต่หลัง `SORT` ทุกรายการของบัญชีเดียวกันมารวมกันเป็นกลุ่ม
  เดียวเรียบร้อย พิสูจน์ว่า `SORT` ทำงานถูกต้อง (ทบทวนจาก Part 027)
- `SD SORT-WORK-FILE` คือไฟล์ที่ `SORT` Statement ใช้งานภายในเท่านั้น (ทบทวนไวยากรณ์จาก Part 027)
  โครงสร้าง `01 SORT-RECORD` ต้อง**ตรงกันทุกไบต์**กับทั้ง `PARSED-RECORD` (ทางเข้า) และ `SORTED-RECORD`
  (ทางออก) เพราะ `SORT` คัดลอกข้อมูลแบบ byte-for-byte ไม่ใช่ตามชื่อฟิลด์
- Control Break ใช้เงื่อนไขเดียวกับที่เรียนใน Part 039: **`SR-ACCT-ID NOT = WS-BREAK-ACCT-ID`** เป็น
  ตัวจุดชนวนให้ขึ้นหัวใบแจ้งยอดใหม่ พร้อมเงื่อนไขพิเศษสำหรับ record แรกสุดของทั้งไฟล์ (`WS-IS-FIRST-RECORD`)

### ข้อควรระวัง

- **จำกฎนี้ไว้ตลอดไป: ห้าม `MOVE` ข้อมูลจาก Reference Modification (หรือฟิลด์ `PIC X` ใด ๆ) ตรงเข้า
  ฟิลด์ที่มี `V` โดยตรงเด็ดขาด** เว้นแต่จะมั่นใจ 100% ว่าเข้าใจกฎการจัดตำแหน่งทศนิยมของ MOVE อย่าง
  ละเอียด วิธีที่ปลอดภัยที่สุดเสมอคือ: ย้ายเข้าฟิลด์จำนวนเต็มล้วนก่อน แล้วค่อย `COMPUTE` ปรับสเกลด้วยตัวเอง
- ไฟล์ชั่วคราว (`STMTPRS.TMP`, `STMTSRT.TMP`, `STMTWRK.TMP`) ถูกสร้างและอาจไม่ถูกลบอัตโนมัติ — ในระบบ
  จริงควรมีขั้นตอนทำความสะอาดไฟล์ชั่วคราวเหล่านี้หลังใช้งานเสร็จ (หรือใช้ JCL `DISP=(NEW,DELETE,DELETE)`
  บน Mainframe จริงเพื่อให้ระบบจัดการให้อัตโนมัติ)

### แบบฝึกหัดที่ 698.1

**โจทย์**: จงอธิบายว่าทำไมบั๊กเรื่องการ `MOVE` ทศนิยมผิดตำแหน่งในขั้นตอนนี้จึง**ไม่ถูกตรวจพบ**จากการ
คอมไพล์เลย ทั้งที่เป็นความผิดพลาดที่ร้ายแรงมาก (ค่าคลาดเคลื่อน 100 เท่า)

**เฉลยแนวทาง**: เพราะทั้ง `RAW-TXN-LINE(29:11)` (แหล่งข้อมูล) และ `PR-AMOUNT` (ปลายทาง) ต่างก็เป็น
PICTURE ที่ถูกต้องตามหลักไวยากรณ์ COBOL ทุกประการ — `MOVE` จากฟิลด์ `PIC X` ไปยังฟิลด์ `PIC 9(9)V99`
เป็นคำสั่งที่ **ถูกต้องตามหลักไวยากรณ์ 100%** เพียงแต่**พฤติกรรมการจัดตำแหน่งทศนิยม**ไม่ตรงกับสิ่งที่
โปรแกรมเมอร์คาดหวัง (คาดว่าจะรักษาตำแหน่งทศนิยมตามข้อมูลจริง แต่ COBOL จัดตำแหน่งตามกฎของตัวเอง)
compiler ไม่มีทางรู้ "เจตนา" ของโปรแกรมเมอร์ได้เลยว่าข้อมูลในฟิลด์ `PIC X` นั้น "ควรจะมี" ทศนิยมกี่ตำแหน่ง
— บั๊กประเภทนี้ (ตรงตามไวยากรณ์ แต่ผิดตามความหมายทางธุรกิจ) เป็นสาเหตุของบั๊กร้ายแรงจำนวนมากในโลกจริง
และเป็นเหตุผลสำคัญที่สุดที่ต้อง**ทดสอบด้วยข้อมูลจริงและตรวจสอบผลลัพธ์ด้วยมือเปรียบเทียบเสมอ** ไม่ใช่
เชื่อว่า "compile ผ่านแล้วต้องถูกต้อง"

---

## ขั้นตอนที่ 699: `RECONCILE` — Batch กระทบยอดควบคุม (Control Total Reconciliation)

### แนวคิด

นี่คือโปรแกรมแบทช์ตัวสุดท้ายของวงจร: **กระทบยอด (Reconcile)** ระหว่างสองแหล่งข้อมูลที่ควรจะสอดคล้องกัน
เสมอแต่คำนวณแยกจากกันโดยสิ้นเชิง — **Source 1**: ผลรวมยอดคงเหลือปัจจุบันจากไฟล์ Master (Indexed)
และ **Source 2**: ยอดที่คำนวณใหม่ทั้งหมดจากการ "เล่นย้อน" (replay) ทุกรายการใน Audit Trail
(Sequential) ตั้งแต่ต้น ถ้าตัวเลขทั้งสองไม่ตรงกัน แปลว่ามีบางอย่างผิดปกติที่ต้องตรวจสอบก่อนสิ้นสุดวัน —
นี่คือรูปแบบการควบคุมภายใน (internal control) แบบคลาสสิกที่สุดของระบบธนาคารบน Mainframe

### ซอร์สโค้ด: `reconcile.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RECONCILE.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * RECONCILE - nightly batch job: control total reconciliation.
      * Classic Mainframe batch pattern: independently recompute a
      * total TWO different ways and prove they agree, instead of
      * just trusting either source blindly.
      *   Total A: sum ACCT-BALANCE across every record currently on
      *             the INDEXED master (our VSAM KSDS stand-in).
      *   Total B: replay every line of the SEQUENTIAL audit trail
      *             from a starting point of zero and recompute what
      *             the total balance change SHOULD have been.
      * If A and B do not agree, that is a red flag that something
      * touched the master file outside of the normal transaction
      * programs (or a bug slipped through) - exactly what a real
      * end-of-day balancing job on a Mainframe is meant to catch.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-MASTER-STATUS.

           SELECT RAW-TXN-FILE ASSIGN TO "TXNLOG.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "acctrec.cpy".

       FD  RAW-TXN-FILE.
       01  RAW-TXN-LINE             PIC X(80).

       WORKING-STORAGE SECTION.
       01  WS-MASTER-STATUS         PIC X(2).
       01  WS-EOF-FLAG              PIC X(1)  VALUE "N".
           88  WS-END-OF-FILE               VALUE "Y".

       01  WS-ACCOUNTS-ON-MASTER    PIC 9(5)  VALUE 0.
       01  WS-MASTER-TOTAL-BAL      PIC 9(11)V99 VALUE 0.

       01  WS-TXN-TYPE              PIC X(4)  VALUE SPACES.
       01  WS-RAW-AMOUNT-INT        PIC 9(11) VALUE 0.
       01  WS-TXN-AMOUNT            PIC 9(9)V99 VALUE 0.
       01  WS-LEDGER-TOTAL          PIC 9(11)V99 VALUE 0.
       01  WS-TXN-LINE-COUNT        PIC 9(5)  VALUE 0.
       01  WS-OPEN-COUNT            PIC 9(5)  VALUE 0.
       01  WS-DEP-COUNT             PIC 9(5)  VALUE 0.
       01  WS-WD-COUNT              PIC 9(5)  VALUE 0.
       01  WS-INT-COUNT             PIC 9(5)  VALUE 0.
       01  WS-OTHER-COUNT           PIC 9(5)  VALUE 0.

       01  WS-DIFFERENCE            PIC S9(11)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== RECONCILE: End-of-Day Control Total Check ===".
           PERFORM 1000-SUM-MASTER-FILE.
           PERFORM 2000-REPLAY-AUDIT-TRAIL.
           PERFORM 3000-PRINT-RECONCILIATION.
           STOP RUN.

       1000-SUM-MASTER-FILE.
           OPEN INPUT ACCOUNT-MASTER.
           PERFORM UNTIL WS-END-OF-FILE
               READ ACCOUNT-MASTER NEXT RECORD
                   AT END
                       SET WS-END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-ACCOUNTS-ON-MASTER
                       ADD ACCT-BALANCE TO WS-MASTER-TOTAL-BAL
               END-READ
           END-PERFORM.
           CLOSE ACCOUNT-MASTER.
           MOVE "N" TO WS-EOF-FLAG.

       2000-REPLAY-AUDIT-TRAIL.
           OPEN INPUT RAW-TXN-FILE.
           PERFORM UNTIL WS-END-OF-FILE
               READ RAW-TXN-FILE
                   AT END
                       SET WS-END-OF-FILE TO TRUE
                   NOT AT END
                       PERFORM 2100-REPLAY-ONE-LINE
               END-READ
           END-PERFORM.
           CLOSE RAW-TXN-FILE.

       2100-REPLAY-ONE-LINE.
           ADD 1 TO WS-TXN-LINE-COUNT.
           MOVE RAW-TXN-LINE(24:4)  TO WS-TXN-TYPE.
      *> Same fix as STMTGEN: land the amount as a plain integer
      *> first, then rescale - never MOVE an alphanumeric substring
      *> straight into a PIC 9(9)V99 field.
           MOVE RAW-TXN-LINE(29:11) TO WS-RAW-AMOUNT-INT.
           COMPUTE WS-TXN-AMOUNT = WS-RAW-AMOUNT-INT / 100.

           EVALUATE FUNCTION TRIM(WS-TXN-TYPE)
               WHEN "OPEN"
                   ADD WS-TXN-AMOUNT TO WS-LEDGER-TOTAL
                   ADD 1 TO WS-OPEN-COUNT
               WHEN "DEP"
                   ADD WS-TXN-AMOUNT TO WS-LEDGER-TOTAL
                   ADD 1 TO WS-DEP-COUNT
               WHEN "WD"
                   SUBTRACT WS-TXN-AMOUNT FROM WS-LEDGER-TOTAL
                   ADD 1 TO WS-WD-COUNT
               WHEN "INT"
                   ADD WS-TXN-AMOUNT TO WS-LEDGER-TOTAL
                   ADD 1 TO WS-INT-COUNT
      *> INQ (read-only) and CLOS (balance is already zero by the
      *> business rule enforced in ACCTCLOSE) never change the total
      *> balance, so they are counted but not added or subtracted.
               WHEN OTHER
                   ADD 1 TO WS-OTHER-COUNT
           END-EVALUATE.

       3000-PRINT-RECONCILIATION.
           COMPUTE WS-DIFFERENCE =
               WS-MASTER-TOTAL-BAL - WS-LEDGER-TOTAL.

           DISPLAY " ".
           DISPLAY "---- Source 1: ACCOUNT-MASTER (indexed file) ----".
           DISPLAY "Accounts on master             : "
               WS-ACCOUNTS-ON-MASTER.
           DISPLAY "Sum of ACCT-BALANCE             : "
               WS-MASTER-TOTAL-BAL.
           DISPLAY " ".
           DISPLAY "---- Source 2: TXNLOG.DAT (sequential audit) ----".
           DISPLAY "Transaction lines replayed      : "
               WS-TXN-LINE-COUNT.
           DISPLAY "  OPEN=" WS-OPEN-COUNT " DEP=" WS-DEP-COUNT
               " WD=" WS-WD-COUNT " INT=" WS-INT-COUNT
               " OTHER(INQ/CLOS)=" WS-OTHER-COUNT.
           DISPLAY "Recomputed total from ledger    : "
               WS-LEDGER-TOTAL.
           DISPLAY " ".
           DISPLAY "Difference (master - ledger)    : " WS-DIFFERENCE.
           IF WS-DIFFERENCE = 0
               DISPLAY "RECONCILIATION RESULT: BALANCED - both "
                   "sources agree exactly."
           ELSE
               DISPLAY "RECONCILIATION RESULT: *** OUT OF BALANCE "
                   "*** - investigate before end of day."
           END-IF.
```

```bash
cobc -x reconcile.cob -o reconcile
./reconcile
```

**ผลลัพธ์จริง:**

```
=== RECONCILE: End-of-Day Control Total Check ===
 
---- Source 1: ACCOUNT-MASTER (indexed file) ----
Accounts on master             : 00003
Sum of ACCT-BALANCE             : 00000002303.73
 
---- Source 2: TXNLOG.DAT (sequential audit) ----
Transaction lines replayed      : 00008
  OPEN=00003 DEP=00001 WD=00001 INT=00002 OTHER(INQ/CLOS)=00001
Recomputed total from ledger    : 00000002303.73
 
Difference (master - ledger)    : +00000000000.00
RECONCILIATION RESULT: BALANCED - both sources agree exactly.
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้**ไม่เชื่อถือแหล่งข้อมูลใดเป็นพิเศษ** — คำนวณทั้งสองแหล่งอย่างเป็นอิสระต่อกันโดยสิ้นเชิง
  แล้วนำผลมาเทียบกันเท่านั้น นี่คือหลักการ "Trust but Verify" ที่ใช้กันในระบบควบคุมภายในของสถาบันการเงิน
  ทั่วโลก
- `EVALUATE` (ทบทวนจาก Part 011 และ Part 041) ใช้แยกประเภทรายการและกำหนดว่ารายการประเภทใด
  "เพิ่ม", "ลด" หรือ "ไม่กระทบ" ยอดรวม — ตรรกะทางธุรกิจนี้ต้อง**ตรงกับตรรกะจริง**ในโปรแกรม online ทุก
  ตัวเป๊ะ (เช่น `WD` ต้องลบ เพราะ `ACCTWD` ใช้ `SUBTRACT`) มิฉะนั้นการกระทบยอดจะผิดพลาดเอง
- `WS-DIFFERENCE` ใช้ `PIC S9(11)V99` (**มี** เครื่องหมาย) เพราะผลต่างอาจเป็นค่าลบได้ในกรณีที่ระบบผิดปกติ
  จริง (ต่างจากฟิลด์ยอดคงเหลือที่ไม่ควรติดลบตามกฎธุรกิจ) — เลือกใช้เครื่องหมายให้ตรงกับความหมายที่แท้จริง
  ของแต่ละฟิลด์เสมอ

### ข้อควรระวัง

- โปรแกรมนี้กระทบยอด**ทั้งระบบในภาพรวม** ไม่ได้ตรวจสอบ**ทีละบัญชี** — หากต้องการความละเอียดสูงกว่านี้
  (เช่น ตรวจสอบว่าบัญชีไหนที่ยอดไม่ตรงกันแบบเจาะจง) ต้องขยายโปรแกรมให้กระทบยอดแยกตาม `ACCT-ID` แต่ละ
  บัญชี (เทคนิคเดียวกับ Control Break ใน `STMTGEN`)
- `RECONCILE` ต้องรัน**หลัง** `BATCHINT` เสมอในวงจรแบทช์ประจำคืน เพราะดอกเบี้ยที่ `BATCHINT` จ่ายเข้า
  บัญชีถูกบันทึกใน Audit Trail ด้วย (ประเภท `INT`) — ถ้ารัน `RECONCILE` ก่อน `BATCHINT` ยอดจาก Master
  Source 1 (ยังไม่รวมดอกเบี้ย) จะไม่ตรงกับ Source 2 (จะไม่มีรายการ `INT` ให้นับเช่นกัน หากรันก่อนจริง ๆ
  ทั้งสองฝั่งจะยังคงตรงกันบังเอิญ แต่ถ้ารันแทรกกลางระหว่าง `BATCHINT` กำลังประมวลผลอยู่จะเกิดปัญหาการ
  อ่านข้อมูลที่ไม่สมบูรณ์ — ลำดับการรัน Batch Job ที่ถูกต้องจึงสำคัญมาก ตามที่จะสรุปในขั้นตอนที่ 700)

### แบบฝึกหัดที่ 699.1

**โจทย์**: จงอธิบายว่าทำไมรายการประเภท `INQ` (สอบถามยอด) และ `CLOS` (ปิดบัญชี) จึงถูกจัดเป็น
`WHEN OTHER` (ไม่กระทบยอดรวม) ในขณะที่ `OPEN`, `DEP`, `WD`, `INT` ล้วนกระทบยอดรวมทั้งสิ้น

**เฉลยแนวทาง**: `INQ` เป็นการอ่านข้อมูลอย่างเดียว (read-only) ไม่มีการเปลี่ยนแปลงยอดเงินใด ๆ เกิดขึ้น
เลยตามที่ออกแบบไว้ใน `ACCTINQ` (ขั้นตอนที่ 696) ส่วน `CLOS` แม้จะเป็นการเปลี่ยนแปลงสถานะบัญชี (Active
เป็น Closed) แต่**ไม่ได้เปลี่ยนแปลงยอดเงิน** เพราะกฎทางธุรกิจใน `ACCTCLOSE` บังคับให้ยอดต้องเป็นศูนย์
ก่อนปิดบัญชีเสมอ (ตรวจสอบใน `IF WS-ALL-VALID AND ACCT-BALANCE NOT = ZERO`) ดังนั้นทั้งสองประเภทนี้
จึงไม่ควรถูกนำไปบวกหรือลบใน `WS-LEDGER-TOTAL` เลย การรวมเข้าไปโดยผิดพลาด (เช่น เผลอ `ADD` ยอด
`CLOS` ที่เป็น 0 อยู่แล้วก็ไม่กระทบอะไร แต่ถ้าเป็นรายการอื่นที่มีค่าไม่เป็นศูนย์จะทำให้ยอดรวมผิดทันที) จึง
ต้องออกแบบ `EVALUATE` ให้สะท้อนตรรกะทางธุรกิจที่แท้จริงของแต่ละประเภทรายการอย่างระมัดระวัง

---

## ขั้นตอนที่ 700: ทดสอบวงจรทั้งระบบแบบครบวงจร และสรุปการเชื่อมโยงกับ Part 051-069

### การรันวงจรทั้งระบบตั้งแต่ต้นจนจบ (Full Day Cycle)

ขั้นตอนสุดท้ายนี้คือการรัน**ทุกโปรแกรม**ที่สร้างขึ้นตลอด Part นี้เรียงตามลำดับที่ระบบธนาคารจริงจะทำงาน
ในหนึ่งวัน: **เช้า-บ่าย (Online Transactions) → กลางคืน (Batch Window)**

```bash
# ===== ONE-TIME SETUP (run once when the system is brand new) =====
cobc -x acctinit.cob -o acctinit
cobc -x acctopen.cob  wrtaudit.cob -o acctopen
cobc -x acctdep.cob   wrtaudit.cob -o acctdep
cobc -x acctwd.cob    wrtaudit.cob -o acctwd
cobc -x acctinq.cob   wrtaudit.cob -o acctinq
cobc -x acctclose.cob wrtaudit.cob -o acctclose
cobc -x batchint.cob  wrtaudit.cob -o batchint
cobc -x stmtgen.cob   -o stmtgen
cobc -x reconcile.cob -o reconcile

./acctinit

# ===== DAYTIME: ONLINE TRANSACTIONS (simulating CICS-style tasks) =====
printf -- "100001\nSOMCHAI JAIDEE\nS\n000050000\n"   | ./acctopen
printf -- "100002\nSUDA MEECHAI\nC\n000075000\n"     | ./acctopen
printf -- "100003\nPRASERT KAEWTA\nS\n000100000\n"   | ./acctopen
printf -- "100001\n000010050\n"                       | ./acctdep
printf -- "100001\n000005000\n"                       | ./acctwd
printf -- "100002\n999999999\n"                       | ./acctwd
printf -- "100003\n"                                  | ./acctinq
printf -- "100001\nDUP\nS\n000010000\n"               | ./acctopen
printf -- "100002\n"                                  | ./acctclose
printf -- "100002\n000075000\n"                       | ./acctwd
printf -- "100002\n"                                  | ./acctclose

# ===== NIGHT: BATCH WINDOW (Part 066 pattern, run in this ORDER) =====
./batchint      # 1. interest calculation FIRST - it changes balances
./stmtgen       # 2. statement generation - reads the now-complete log
./reconcile     # 3. reconciliation LAST - checks everything ties out
```

**ผลลัพธ์จริงของขั้นตอน Online (ระหว่างวัน) — สรุปย่อ:**

```
Account 100001 opened for SOMCHAI JAIDEE - balance=000000500.00
Account 100002 opened for SUDA MEECHAI - balance=000000750.00
Account 100003 opened for PRASERT KAEWTA - balance=000001000.00
Deposit accepted. Account 100001 new balance=000000600.50
Withdrawal accepted. Account 100001 new balance=000000550.50
ERROR: insufficient funds. Balance=000000750.00 requested=009999999.99
Withdrawal REJECTED - no data was changed.
Account ****XXX003 (PRASERT KAEWTA) SAVINGS ACTIVE
  Balance = 000001000.00
ERROR: account 100001 already exists.
Account NOT opened - no data was changed.
ERROR: account 100002 balance is not zero (000000750.00) - withdraw remaining funds before closing.
Close REJECTED - no data was changed.
Withdrawal accepted. Account 100002 new balance=000000000.00
Account 100002 closed successfully.
```

**ผลลัพธ์จริงของขั้นตอน Batch (กลางคืน):**

```
=== BATCHINT: Nightly Interest Batch Run ===
Run date: 20260430
 
---- BATCHINT CONTROL TOTAL REPORT ----
Accounts read from master     : 00003
Savings accounts credited      : 00002
Checking accounts skipped       : 00000
Closed accounts skipped        : 00001
Total interest paid this run   : 000000003.23
Opening total balance (all accts): 00000001550.50
Closing total balance (all accts): 00000001553.73
Control total check: BALANCED

==== STATEMENT: Account 100001 (SOMCHAI JAIDEE) ====
    20260401 090000 OPEN amount=000000500.00 balance-after=000000500.00 OK
    20260401 091500 DEP  amount=000000100.50 balance-after=000000600.50 OK
    20260401 092000 WD   amount=000000050.00 balance-after=000000550.50 OK
    20260430 010000 INT  amount=000000001.15 balance-after=000000551.65 OK
 
==== STATEMENT: Account 100002 (SUDA MEECHAI) ====
    20260401 090000 OPEN amount=000000750.00 balance-after=000000750.00 OK
    20260401 092000 WD   amount=000000750.00 balance-after=000000000.00 OK
    20260401 094500 CLOS amount=000000000.00 balance-after=000000000.00 OK
 
==== STATEMENT: Account 100003 (PRASERT KAEWTA) ====
    20260401 090000 OPEN amount=000001000.00 balance-after=000001000.00 OK
    20260401 093000 INQ  amount=000000000.00 balance-after=000001000.00 OK
    20260430 010000 INT  amount=000000002.08 balance-after=000001002.08 OK
 
STMTGEN complete. Statements printed: 003 Total transactions: 00010

=== RECONCILE: End-of-Day Control Total Check ===
 
---- Source 1: ACCOUNT-MASTER (indexed file) ----
Accounts on master             : 00003
Sum of ACCT-BALANCE             : 00000001553.73
 
---- Source 2: TXNLOG.DAT (sequential audit) ----
Transaction lines replayed      : 00010
  OPEN=00003 DEP=00001 WD=00002 INT=00002 OTHER(INQ/CLOS)=00002
Recomputed total from ledger    : 00000001553.73
 
Difference (master - ledger)    : +00000000000.00
RECONCILIATION RESULT: BALANCED - both sources agree exactly.
```

ทุกตัวเลขสอดคล้องกันสมบูรณ์: ก่อนคืนดอกเบี้ย (ตอนเช้าวันถัดไป) ยอดรวมทั้งระบบคือ `550.50 (บัญชี
100001) + 0.00 (บัญชี 100002 ที่ปิดแล้ว) + 1000.00 (บัญชี 100003) = 1550.50` ตรงกับ "Opening total
balance" ของ `BATCHINT` พอดี หลังจ่ายดอกเบี้ยรวม `1.15 + 2.08 = 3.23` ยอดรวมกลายเป็น `1550.50 +
3.23 = 1553.73` ซึ่งตรงกับ "Closing total balance" ของ `BATCHINT`, ยอดรวมที่ `STMTGEN` แสดงต่อบัญชี
(`551.65 + 0.00 + 1002.08 = 1553.73`), และที่สำคัญที่สุดคือตรงกับทั้งสองแหล่งข้อมูลอิสระใน `RECONCILE`
(Master sum และ Ledger replay ทั้งคู่ได้ `1553.73` เป๊ะ) **RECONCILE ยืนยันว่าทั้งไฟล์ Master (Indexed)
และ Audit Trail (Sequential) เห็นตรงกันทุกสตางค์**

### ตารางสรุป: สิ่งที่จำลองได้จริง vs สิ่งที่ยังคงเป็นแนวคิด

| องค์ประกอบ Mainframe จริง | อยู่ใน Part ใด | สิ่งที่โปรเจกต์นี้ใช้แทน |
|---|---|---|
| VSAM KSDS | Part 055-056 | GnuCOBOL Indexed File (Berkeley DB ISAM) — **ทำงานได้จริง** |
| Sequential Log/QSAM | Part 023 | GnuCOBOL LINE SEQUENTIAL File — **ทำงานได้จริง** |
| JCL (JOB/EXEC/DD ควบคุมลำดับ) | Part 052-053 | ลำดับคำสั่งใน shell script (`bash`) — **ทำงานได้จริงในหลักการเดียวกัน (ควบคุมลำดับการรัน) แต่ไม่ใช่ JCL จริง** |
| CICS Transaction | Part 061-064 | โปรแกรมโต้ตอบสั้น ๆ ที่รันแล้วจบ — **จำลองพฤติกรรมเชิงแนวคิดเท่านั้น ไม่มี CICS region จริง** |
| DB2/SQL | Part 057-060 | ไม่ได้ใช้เลยในโปรเจกต์นี้ — ระบบใช้ไฟล์ COBOL ล้วน (ตรงกับยุคก่อน DB2 ของ Mainframe จริง) |
| Batch Window/Job Scheduling | Part 066 | ลำดับการรัน `batchint → stmtgen → reconcile` ด้วยมือ — **หลักการเดียวกัน ไม่มี Job Scheduler จริง** |
| DFSORT | Part 067 | COBOL `SORT` Statement ภายใน `stmtgen.cob` — **ทำงานได้จริงด้วยกลไก SORT ในตัวภาษา** |
| Performance Tuning | Part 068 | หลักการ SEARCH ALL/ชนิดข้อมูล ไม่ได้ใช้เจาะจงในระบบนี้ (ระบบเล็กเกินกว่าจะเห็นผลต่าง) แต่ Control Total คือรูปแบบหนึ่งของการตรวจสอบที่เรียนมา |
| RACF | Part 069 | ไม่ได้ใช้เลย (เชิงแนวคิดล้วน) — แต่ **Secure Coding** (IS NUMERIC, bounds check, masking) ถูกใช้จริงทุกโปรแกรม |

### ข้อควรระวัง

- **ลำดับการรัน Batch สำคัญมาก**: `batchint` ต้องรันก่อน `stmtgen`/`reconcile` เสมอ เพราะทั้งสอง
  โปรแกรมหลังอ่านผลลัพธ์ที่รวมดอกเบี้ยแล้ว หากสลับลำดับ ตัวเลขในรายงานจะไม่ตรงกับความเป็นจริงของวันนั้น
  (แม้จะยัง "BALANCED" ในตัวเองก็ตาม เพราะกระทบยอดกับสถานะ ณ ขณะนั้น ไม่ใช่สถานะสุดท้ายที่ถูกต้อง) —
  บน Mainframe จริงลำดับนี้ถูกบังคับด้วย JCL Job Scheduling (Part 052-053, 066) ไม่ใช่การจำได้ของ
  ผู้ปฏิบัติงาน
- โปรเจกต์นี้**ไม่ได้ใส่กลไกป้องกัน concurrent access** (สองโปรแกรมพยายามแก้ไขไฟล์ Master พร้อมกัน) —
  บน Mainframe จริง VSAM มีกลไก record-level locking ในตัว ส่วนระบบสาธิตนี้ออกแบบมาให้รันทีละโปรแกรม
  เท่านั้น (ตรงกับข้อจำกัดที่ควรตระหนักหากนำแนวคิดนี้ไปพัฒนาต่อ)

### แบบฝึกหัดที่ 700.1

**โจทย์**: จากตารางสรุปข้างต้น จงเลือกองค์ประกอบ Mainframe จริง 1 อย่างที่**ไม่ได้ถูกจำลองเลย**ในโปรเจกต์
นี้ (DB2/SQL หรือ RACF) และอธิบายว่าหากจะขยายโปรเจกต์นี้ให้รวมองค์ประกอบนั้นในเชิงแนวคิด (ไม่ต้องรันจริง)
จะออกแบบอย่างไร

**เฉลยแนวทาง**: ตัวอย่างคำตอบสำหรับ RACF: หากต้องการจำลองแนวคิด RACF ควบคู่กับระบบนี้ (โดยไม่ต้องมี
RACF จริง) อาจออกแบบเป็น**เอกสารนโยบายสิทธิ์** (access policy document) แยกต่างหากที่ระบุว่า User ID
กลุ่มใดควรมีสิทธิ์รันโปรแกรมใดได้บ้าง เช่น กลุ่ม `TELLER` มีสิทธิ์รัน `ACCTOPEN`, `ACCTDEP`, `ACCTWD`,
`ACCTINQ` เท่านั้น (ไม่มีสิทธิ์รัน `ACCTCLOSE` หรือโปรแกรม batch ใด ๆ เลย) ส่วนกลุ่ม `BATCHOPS` มีสิทธิ์
รันเฉพาะ `BATCHINT`, `STMTGEN`, `RECONCILE` เท่านั้น (ทบทวนหลักการ Least Privilege จาก Part 069
ขั้นตอนที่ 682) แม้จะไม่มีการบังคับใช้จริงด้วยซอฟต์แวร์ RACF แต่การออกแบบนโยบายสิทธิ์แบบนี้ไว้ล่วงหน้า
เป็นขั้นตอนสำคัญที่ทีมพัฒนาระบบธนาคารจริงต้องทำก่อนนำระบบไปติดตั้งบน Mainframe จริงที่มี RACF ควบคุม
สิทธิ์อยู่จริงในภายหลัง

---

## สรุปท้ายบทและสรุปรวมเฟส 4

โปรเจกต์นี้พิสูจน์ให้เห็นว่า **"ฝั่งการประมวลผลไฟล์และแบทช์" ของระบบ COBOL แบบ Mainframe** มีตรรกะการ
ทำงานที่แท้จริงอย่างไร ผ่านระบบธนาคารจำลองที่ประกอบด้วย:

**โปรแกรม Online-style (จำลองพฤติกรรม CICS Transaction):**
- `ACCTOPEN` — เปิดบัญชีใหม่ พร้อมตรวจสอบบัญชีซ้ำและ Secure Coding เต็มรูปแบบ
- `ACCTDEP` / `ACCTWD` — ฝาก/ถอนเงิน พร้อมป้องกันยอดติดลบ
- `ACCTINQ` — สอบถามยอด พร้อม Data Masking
- `ACCTCLOSE` — ปิดบัญชี พร้อมกฎยอดต้องเป็นศูนย์ก่อน

**โปรแกรม Batch (จำลอง Batch Window รายคืน):**
- `BATCHINT` — คำนวณดอกเบี้ยพร้อม Control Total Report
- `STMTGEN` — สร้างใบแจ้งยอดด้วย SORT + Control Break + Master-Detail
- `RECONCILE` — กระทบยอดควบคุมระหว่างสองแหล่งข้อมูลอิสระ

**โครงสร้างพื้นฐานร่วม:**
- `acctrec.cpy` — Copybook (Part 033) รับประกันโครงสร้างข้อมูลตรงกันทุกโปรแกรม
- `WRTAUDIT` — Subprogram (Part 031-032) รวมตรรกะการเขียน Audit Trail ไว้ที่เดียว

ทุกโปรแกรมคอมไพล์และรันได้จริงด้วย GnuCOBOL (build ที่มี Berkeley DB เป็น ISAM handler) และผ่านการ
ทดสอบ end-to-end ครบวงจรทั้งระบบ พร้อมพิสูจน์ตัวเลขทางบัญชีที่ถูกต้องตรงกันทุกจุด รวมถึงการค้นพบและแก้ไข
บั๊กจริงระหว่างการพัฒนา (การจัดตำแหน่งทศนิยมผิดพลาดใน `STMTGEN`) ซึ่งเป็นบทเรียนที่มีค่าไม่แพ้เนื้อหา
หลักของ Part นี้เลย

นี่คือการปิดฉากเฟส 4 (Mainframe/Enterprise) อย่างสมบูรณ์! ตั้งแต่ Part 071 เป็นต้นไป หลักสูตรจะเข้าสู่
**เฟส 5: COBOL สมัยใหม่ (Modern)** ที่จะพาคุณออกจากโลก Mainframe แบบดั้งเดิม ไปสู่การเชื่อมต่อ COBOL
กับเทคโนโลยีสมัยใหม่: REST API, ภาษาอื่น (C, Java), ฐานข้อมูลสมัยใหม่, Docker, Cloud และ CI/CD —
เนื้อหาทั้งหมดใน Part 071 เป็นต้นไปสามารถทดสอบได้จริง 100% ด้วย GnuCOBOL เช่นเดียวกับเฟส 1-3

**[← กลับไป Part 069: Security บน Mainframe (RACF Concepts)](part-069-mainframe-security.md)** |
**[ไปยัง Part 071: GnuCOBOL สมัยใหม่และ Open-source COBOL Ecosystem →](part-071-gnucobol-modern-ecosystem.md)**
