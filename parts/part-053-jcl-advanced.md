# Part 053: JCL ขั้นสูง: Procedures และ Condition Codes (ขั้นตอนที่ 521–530)

## คำนำของ Part นี้

Part 052 พาเรารู้จัก JCL เบื้องต้นแล้ว: บัตร `//JOBNAME JOB`, `//stepname EXEC PGM=...`, และ
`//ddname DD ...` ที่ผูกไฟล์เข้ากับโปรแกรม COBOL ทีละ step แต่ในงานจริงระดับองค์กร งาน Batch หนึ่ง
งาน (job) มักมีหลาย step ที่ **ใช้โครงสร้าง DD ชุดเดิมซ้ำ ๆ กัน** (เช่น step คอมไพล์ตามด้วย step
link-edit ตามด้วย step รัน ที่แทบทุกโปรแกรมในองค์กรต้องทำเหมือนกันทุกตัวอักษร) และ step ถัดไปมัก
**ต้องรู้ว่า step ก่อนหน้าสำเร็จหรือล้มเหลว** ก่อนตัดสินใจว่าจะรันต่อหรือข้าม

Part นี้จะแนะนำสองกลไกสำคัญที่ทำให้ JCL ใช้งานได้จริงในระดับองค์กร:

1. **Procedure (PROC)** — กลไก "แม่แบบ" (template) ของ step ชุดหนึ่งที่เขียนครั้งเดียวแล้วเรียกใช้
   ซ้ำได้หลายร้อยหลายพัน job โดยไม่ต้องคัดลอกวาง DD statement เดิมซ้ำ ๆ
2. **Condition Code / COND / IF-THEN-ELSE** — กลไกควบคุมว่า step ใดควรรันหรือข้าม โดยอิงจาก
   ผลลัพธ์ตัวเลข (Return Code) ที่ step ก่อนหน้าส่งกลับมา

> **ย้ำกฎสำคัญของเฟส 4 (ตามที่ `docs/COURSE-OUTLINE.md` ระบุไว้)**: เนื้อหาทั้งหมดใน Part นี้เป็น
> **ไวยากรณ์ JCL มาตรฐานของ IBM z/OS อ้างอิงจากเอกสารทางการ** สภาพแวดล้อมของหลักสูตรนี้เป็น Linux
> Sandbox ที่มี GnuCOBOL แต่**ไม่มี z/OS, ไม่มี JES2/JES3, ไม่มี TSO จริง** จึง **ไม่สามารถ submit
> หรือรัน JCL ใด ๆ ในเอกสารนี้ได้จริง** ทุกตัวอย่าง JCL ในนี้เป็นไวยากรณ์อ้างอิงที่ถูกต้องตามมาตรฐาน
> เพื่อให้คุณอ่านและเขียนเป็น แต่ต้องใช้ z/OS จริงหรือ Mainframe Emulator (เช่น Hercules + MVS/TK4-)
> เพื่อทดสอบรันจริง — จะไม่มีการอ้างว่า "compile และรันสำเร็จ" กับ JCL ใด ๆ ใน Part นี้เด็ดขาด

---

## ขั้นตอนที่ 521: ปัญหาที่ Procedure แก้ไข — ทำไมต้องมี PROC

### ปัญหาการเขียน DD ซ้ำซาก

ลองนึกภาพองค์กรที่มีโปรแกรม COBOL หลายพันโปรแกรม ทุกโปรแกรมต้องผ่านขั้นตอนเดียวกันเสมอ:
คอมไพล์ (compile) → link-edit (ผูกเป็น executable) → รัน (execute) แต่ละขั้นตอนต้องมี DD statement
ที่ชี้ไปยัง dataset ระบบชุดเดิมทุกครั้ง (SYSLIB, SYSLIN, SYSPRINT ฯลฯ) ถ้าไม่มีกลไกใดช่วย
โปรแกรมเมอร์ทุกคนต้องคัดลอกวาง JCL แบบนี้ซ้ำหลายพันครั้งทั่วองค์กร:

```jcl
//COMPILE  EXEC PGM=IGYCRCTL
//STEPLIB  DD DSN=IGY.SIGYCOMP,DISP=SHR
//SYSLIN   DD DSN=&&LOADSET,DISP=(MOD,PASS),UNIT=SYSDA,
//            SPACE=(TRK,(3,3))
//SYSPRINT DD SYSOUT=*
//SYSIN    DD DSN=MY.SOURCE.LIB(PROGRAM1),DISP=SHR
```

ปัญหาคือ (1) ถ้า SYSLIB เปลี่ยน library ต้องแก้ไข JCL หลายพันไฟล์ (2) โอกาสพิมพ์ผิดสูงมาก
(3) โปรแกรมเมอร์รุ่นใหม่ต้องจำรายละเอียดทางเทคนิคของ compiler/linker โดยไม่จำเป็น

### แนวคิดของ Procedure

**Procedure (PROC)** คือการ "ห่อหุ้ม" step ชุดหนึ่งที่ใช้ซ้ำบ่อย ๆ ให้กลายเป็นหน่วยเดียวที่เรียกใช้
ด้วยชื่อสั้น ๆ คำเดียว โปรแกรมเมอร์ COBOL ทั่วไปที่ต้องการแค่คอมไพล์-ลิงก์-รันโปรแกรมของตัวเอง ไม่
จำเป็นต้องรู้รายละเอียดของ SYSLIB/SYSLIN ใด ๆ เลย เพียงเขียน:

```jcl
//STEP1    EXEC COBUCLG,PARM.COB='LIB,APOST'
//COB.SYSIN DD DSN=MY.SOURCE.LIB(PROGRAM1),DISP=SHR
```

โดย `COBUCLG` คือชื่อ Procedure ที่ทีม System Programmer เตรียมไว้ให้ล่วงหน้า (เป็นตัวย่อทั่วไป
ของ COBOL-Compile-Link-Go) ครอบคลุมทั้ง 3 ขั้นตอนไว้ภายในเรียบร้อยแล้ว

### ประเภทของ Procedure

| ประเภท | ความหมาย |
|---|---|
| **Cataloged Procedure** | เก็บไว้ถาวรใน Procedure Library (PROCLIB) ขององค์กร เรียกใช้ได้จากทุก job |
| **In-stream Procedure** | เขียนแทรกอยู่ใน JCL deck เดียวกันนั้นเอง ใช้ได้เฉพาะภายใน job นั้น |

Part นี้จะอธิบายทั้งสองแบบ เริ่มจาก In-stream Procedure ในขั้นตอนถัดไป เพราะเห็นภาพโครงสร้างได้ง่าย
กว่า ก่อนไปสู่ Cataloged Procedure ที่ใช้งานจริงในองค์กรส่วนใหญ่

### ข้อควรระวัง

- Procedure ไม่ใช่แนวคิดเฉพาะของ COBOL — เป็นกลไกของ JCL/JES ล้วน ๆ ใช้ได้กับโปรแกรมภาษาใดก็ได้
  บน Mainframe (PL/I, Assembler, C ฯลฯ)
- อย่าสับสน Procedure (PROC) ของ JCL กับ `PROCEDURE DIVISION` ของ COBOL — เป็นคนละแนวคิดกัน
  โดยสิ้นเชิงแม้ชื่อจะคล้ายกัน

### แบบฝึกหัดที่ 521.1

**โจทย์**: จงอธิบายด้วยคำพูดของตัวเองว่า Procedure ช่วยลดความเสี่ยงเรื่องใดบ้างเมื่อเทียบกับการ
เขียน JCL แบบคัดลอกวางซ้ำทุกโปรแกรม

**เฉลย**: (1) ลดการพิมพ์ผิดเพราะเขียน DD ที่ซับซ้อนเพียงครั้งเดียว (2) เมื่อ library หรือพารามิเตอร์
ระบบเปลี่ยน แก้ไขที่ Procedure จุดเดียว ก็มีผลกับทุก job ที่เรียกใช้ทันที ไม่ต้องไล่แก้ทีละไฟล์
(3) โปรแกรมเมอร์ทั่วไปไม่จำเป็นต้องรู้รายละเอียดทางเทคนิคของ compiler/linker เพียงเรียกชื่อ
Procedure ก็เพียงพอ

---

## ขั้นตอนที่ 522: In-Stream Procedure — PROC และ PEND

### โครงสร้างพื้นฐาน

**In-stream Procedure** คือ Procedure ที่เขียนแทรกอยู่ในบัตร JCL เดียวกัน เริ่มต้นด้วยบัตร `PROC`
และจบด้วยบัตร `PEND` (Procedure END) ทุก step ที่อยู่ระหว่างสองบัตรนี้จะถูกรวมเป็น "แม่แบบ" เดียว

```jcl
//MYJOB    JOB (ACCT123),'COBOL COURSE',CLASS=A,MSGCLASS=X
//*
//BACKUP   PROC
//STEP1    EXEC PGM=IEBGENER
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  DUMMY
//SYSUT1   DD  DSN=&&TEMPDS,DISP=SHR
//SYSUT2   DD  DSN=PROD.BACKUP.FILE,DISP=(NEW,CATLG,DELETE),
//             UNIT=SYSDA,SPACE=(TRK,(10,5)),
//             DCB=(RECFM=FB,LRECL=80,BLKSIZE=8000)
//BACKUP   PEND
//*
//STEPA    EXEC BACKUP
```

### อธิบายจุดสำคัญ

- `//BACKUP PROC` เริ่มต้น Procedure ชื่อ `BACKUP` (ชื่อนี้เลือกเองได้ ตั้งตามกฎการตั้งชื่อ JCL
  ทั่วไป คือ 1-8 ตัวอักษร ขึ้นต้นด้วยตัวอักษร)
- ระหว่าง `PROC` กับ `PEND` คือ step ปกติทุกประการ (ในที่นี้มี step เดียวชื่อ `STEP1` ที่เรียก
  `IEBGENER` ซึ่งเป็น utility มาตรฐานของ z/OS สำหรับคัดลอกไฟล์)
- `//BACKUP PEND` ปิดท้าย Procedure (ชื่อหลัง `//` ตรงนี้เลือกใส่หรือไม่ใส่ก็ได้ตามมาตรฐาน แต่นิยม
  ใส่ชื่อเดิมซ้ำเพื่อความชัดเจน)
- `//STEPA EXEC BACKUP` คือจุดที่ "เรียกใช้" Procedure — สังเกตว่าไม่มี `PGM=` เพราะ `BACKUP`
  ไม่ใช่ชื่อโปรแกรม แต่เป็นชื่อ Procedure ที่ขยายออกเป็นหลาย step โดยอัตโนมัติเมื่อ JES ประมวลผล

### ทำไมต้องมี In-Stream Procedure ทั้งที่มี Cataloged Procedure แล้ว

In-Stream Procedure เหมาะกับ 2 กรณีหลัก: (1) กำลังพัฒนา/ทดสอบ Procedure ใหม่ที่ยังไม่พร้อม
เผยแพร่เป็น Cataloged Procedure ขององค์กร (2) Procedure นั้นใช้เฉพาะ job นี้ job เดียว ไม่คุ้มที่จะ
นำไปเก็บถาวรใน PROCLIB ส่วนกลาง

### ข้อควรระวัง

- มาตรฐาน JCL จำกัดจำนวน In-stream Procedure ไว้สูงสุด **15 ตัวต่อ job** (ข้อจำกัดของ JES เอง)
  ถ้าต้องการมากกว่านั้นต้องใช้ Cataloged Procedure แทน
- ห้ามซ้อน Procedure ภายใน Procedure อีกชั้น (nested PROC ไม่รองรับใน JCL มาตรฐาน)
- ชื่อ step ภายใน Procedure (เช่น `STEP1`) จะกลายเป็นส่วนหนึ่งของการอ้างอิงแบบ qualified name
  เมื่อ override ภายหลัง (ขั้นตอนที่ 525) จึงควรตั้งชื่อให้สื่อความหมายตั้งแต่ต้น

### แบบฝึกหัดที่ 522.1

**โจทย์**: จงเขียน In-stream Procedure ชื่อ `PRTREPT` ที่มี step เดียวเรียก `PGM=IEBGENER`
เพื่อพิมพ์รายงานออก `SYSOUT=*` จาก dataset ชื่อ `PROD.REPORT.DAILY`

**เฉลย**:

```jcl
//PRTREPT  PROC
//STEP1    EXEC PGM=IEBGENER
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  DUMMY
//SYSUT1   DD  DSN=PROD.REPORT.DAILY,DISP=SHR
//SYSUT2   DD  SYSOUT=*
//PRTREPT  PEND
//*
//RUNIT    EXEC PRTREPT
```

---

## ขั้นตอนที่ 523: การเรียกใช้ PROC ด้วย Symbolic Parameters

### ปัญหา: แม่แบบที่ตายตัวเกินไป

Procedure ในขั้นตอนที่ 522 มีปัญหาคือชื่อ dataset (`PROD.BACKUP.FILE`) ถูก "ฝัง" ไว้ตายตัว ถ้ามี
job อื่นอยากใช้ Procedure เดียวกันแต่ backup คนละไฟล์ ก็ต้องคัดลอก Procedure ใหม่ทั้งชุด — นี่คือ
ปัญหาเดิมที่ Procedure ควรมาแก้ แต่ยังไม่ได้แก้เต็มที่

### Symbolic Parameters — ตัวแปรภายใน PROC

**Symbolic Parameter** คือตัวแปรที่ขึ้นต้นด้วยเครื่องหมาย `&` ใช้แทนค่าคงที่ภายใน Procedure
ทำให้ผู้เรียกใช้ Procedure สามารถ "ส่งค่า" เข้าไปแทนที่ตอน `EXEC` ได้

```jcl
//BACKUP   PROC SRCDSN=,TRGDSN=,UN=SYSDA
//STEP1    EXEC PGM=IEBGENER
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  DUMMY
//SYSUT1   DD  DSN=&SRCDSN,DISP=SHR
//SYSUT2   DD  DSN=&TRGDSN,DISP=(NEW,CATLG,DELETE),
//             UNIT=&UN,SPACE=(TRK,(10,5)),
//             DCB=(RECFM=FB,LRECL=80,BLKSIZE=8000)
//BACKUP   PEND
//*
//STEPA    EXEC BACKUP,SRCDSN='PROD.CUSTOMER.MASTER',
//             TRGDSN='PROD.CUSTOMER.BACKUP01'
//STEPB    EXEC BACKUP,SRCDSN='PROD.ACCOUNT.MASTER',
//             TRGDSN='PROD.ACCOUNT.BACKUP01'
```

### อธิบายจุดสำคัญ

- `//BACKUP PROC SRCDSN=,TRGDSN=,UN=SYSDA` ประกาศ symbolic parameter 3 ตัว: `SRCDSN`, `TRGDSN`
  (ไม่มีค่า default — ผู้เรียกต้องกำหนดเองทุกครั้ง) และ `UN` (มีค่า default เป็น `SYSDA`
  ถ้าผู้เรียกไม่ระบุ ใช้ `SYSDA` โดยอัตโนมัติ)
- ภายใน Procedure อ้างอิงตัวแปรด้วย `&SRCDSN`, `&TRGDSN`, `&UN` แทนค่าคงที่
- `STEPA` และ `STEPB` เรียกใช้ Procedure **เดียวกัน** แต่ส่งค่าต่างกัน ทำให้ backup dataset คนละคู่
  โดยไม่ต้องคัดลอก Procedure ใหม่เลยแม้แต่บรรทัดเดียว — นี่คือประโยชน์ที่แท้จริงของ Symbolic
  Parameters

### กฎการตั้งชื่อ Symbolic Parameter

- ต้องขึ้นต้นด้วย `&` ตามด้วยตัวอักษรและตัวเลข ความยาวรวมไม่เกิน 8 ตัวอักษร (ไม่นับ `&`)
- ห้ามใช้ชื่อที่ซ้ำกับคำสงวนของ JCL (เช่น `&SYSUID` ซึ่งเป็น system symbol ที่ระบบสร้างให้อัตโนมัติ
  หมายถึง TSO userid ของผู้ submit job — ต่างจาก symbolic parameter ที่ผู้ใช้กำหนดเอง)

### ข้อควรระวัง

- ถ้าไม่ระบุค่าให้ symbolic parameter ที่ไม่มี default (เช่นลืมส่ง `TRGDSN` ตอนเรียก `EXEC BACKUP`)
  JES จะปฏิเสธ job ตั้งแต่ขั้นตอน JCL conversion ก่อนแม้แต่จะเริ่มรัน step แรก
- ค่าที่มีอักขระพิเศษ (เช่นจุด `.` ในชื่อ dataset) ต้องครอบด้วยเครื่องหมายคำพูดเดี่ยว (`'...'`) เมื่อ
  ส่งค่าตอน `EXEC` เพื่อป้องกันการตีความผิดพลาด

### แบบฝึกหัดที่ 523.1

**โจทย์**: จงแก้ไข Procedure `BACKUP` ข้างต้นให้มี symbolic parameter เพิ่มเติมชื่อ `SP` สำหรับ
กำหนดขนาดพื้นที่ (`SPACE`) โดยมีค่า default เป็น `(TRK,(10,5))`

**เฉลย**:

```jcl
//BACKUP   PROC SRCDSN=,TRGDSN=,UN=SYSDA,SP=(TRK,(10,5))
//STEP1    EXEC PGM=IEBGENER
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  DUMMY
//SYSUT1   DD  DSN=&SRCDSN,DISP=SHR
//SYSUT2   DD  DSN=&TRGDSN,DISP=(NEW,CATLG,DELETE),
//             UNIT=&UN,SPACE=&SP,
//             DCB=(RECFM=FB,LRECL=80,BLKSIZE=8000)
//BACKUP   PEND
```

job ที่ต้องการพื้นที่มากกว่าเดิมสามารถ override เฉพาะ `SP` โดยไม่กระทบ parameter อื่น เช่น
`EXEC BACKUP,SRCDSN='X',TRGDSN='Y',SP=(TRK,(50,10))`

---

## ขั้นตอนที่ 524: Cataloged Procedures และ JCLLIB Statement

### จาก In-Stream สู่ Cataloged

**Cataloged Procedure** คือ Procedure แบบเดียวกับที่เรียนมา แต่ถูก**เก็บถาวรในไลบรารีระบบ**
(เรียกว่า **PROCLIB**) แทนที่จะแทรกอยู่ใน JCL ทุกครั้ง องค์กรส่วนใหญ่จะมี PROCLIB มาตรฐาน เช่น
`SYS1.PROCLIB` (ของระบบ) หรือ `PROD.PROCLIB` (ขององค์กรเอง) ที่ System Programmer ดูแล

เมื่อเป็น Cataloged Procedure แล้ว การเรียกใช้ **ไม่ต้องมีบัตร `PROC`/`PEND` ปรากฏใน JCL เลย**
เพราะ Procedure ถูกเก็บแยกไว้ต่างหากแล้ว:

```jcl
//MYJOB    JOB (ACCT123),'COBOL COURSE',CLASS=A,MSGCLASS=X
//*
//STEPA    EXEC BACKUP,SRCDSN='PROD.CUSTOMER.MASTER',
//             TRGDSN='PROD.CUSTOMER.BACKUP01'
```

เพียงเท่านี้ JES จะไปค้นหาสมาชิกชื่อ `BACKUP` ใน PROCLIB ที่ระบบตั้งค่าไว้โดยอัตโนมัติ แล้วนำเนื้อหา
มา "ขยาย" (expand) รวมเข้ากับ job นี้ระหว่างขั้นตอน JCL conversion ก่อนรันจริง

### JCLLIB Statement — ระบุ PROCLIB ที่ไม่ใช่ค่ามาตรฐาน

หากองค์กรมี Procedure เก็บไว้ใน library ส่วนตัวที่ไม่ใช่ PROCLIB มาตรฐานของระบบ ต้องระบุด้วยบัตร
**`JCLLIB`** ก่อนบัตร `EXEC` แรกที่ใช้ Procedure นั้น:

```jcl
//MYJOB    JOB (ACCT123),'COBOL COURSE',CLASS=A,MSGCLASS=X
//JCLLIB   ORDER=(PROD.TEAM.PROCLIB,PROD.DEPT.PROCLIB)
//*
//STEPA    EXEC BACKUP,SRCDSN='PROD.CUSTOMER.MASTER',
//             TRGDSN='PROD.CUSTOMER.BACKUP01'
```

`JCLLIB ORDER=(...)` บอก JES ให้ค้นหา Procedure ตามลำดับ library ที่ระบุ (จากซ้ายไปขวา) ก่อนจะไป
ค้นหาใน PROCLIB มาตรฐานของระบบเป็นลำดับสุดท้าย

### เปรียบเทียบ In-Stream กับ Cataloged

| ประเด็น | In-Stream Procedure | Cataloged Procedure |
|---|---|---|
| ที่เก็บ | อยู่ในบัตร JCL เดียวกัน (`PROC`...`PEND`) | เก็บแยกต่างหากใน PROCLIB |
| การใช้ซ้ำข้าม job | ทำไม่ได้ (ใช้ได้เฉพาะ job นั้น) | ใช้ซ้ำได้ทุก job ทั่วองค์กร |
| จำนวนสูงสุดต่อ job | 15 ตัว | ไม่จำกัด (เรียกกี่ครั้งก็ได้) |
| เหมาะกับ | ทดสอบ/ใช้ครั้งเดียว | มาตรฐานองค์กรที่ใช้ซ้ำระยะยาว |

### ข้อควรระวัง

- ถ้า Procedure ชื่อเดียวกันมีอยู่ทั้งแบบ In-Stream (ใน job เดียวกัน) และ Cataloged (ใน PROCLIB)
  ระบบจะให้ความสำคัญกับ **In-Stream ก่อนเสมอ**
- การแก้ไข Cataloged Procedure มีผลกระทบกว้างมาก (ทุก job ที่เรียกใช้) จึงต้องผ่านกระบวนการควบคุม
  การเปลี่ยนแปลง (Change Management) อย่างเข้มงวดในองค์กรจริง ไม่ใช่แก้ไขได้อิสระเหมือน In-Stream

### แบบฝึกหัดที่ 524.1

**โจทย์**: จงอธิบายว่าทำไมองค์กรขนาดใหญ่ส่วนมากจึงเลือกใช้ Cataloged Procedure เป็นหลัก
แทนที่จะใช้ In-Stream Procedure ในงาน Production

**เฉลย**: เพราะ Cataloged Procedure ใช้ซ้ำได้ข้าม job และข้ามทีมทั่วทั้งองค์กร แก้ไขที่จุดเดียว
มีผลทุกที่ที่เรียกใช้ ลดความเสี่ยงเรื่องความไม่สอดคล้องกันระหว่าง job ต่าง ๆ นอกจากนี้ยังไม่ติด
ข้อจำกัดเรื่องจำนวนสูงสุด 15 ตัวต่อ job แบบ In-Stream Procedure และสามารถควบคุมสิทธิ์การแก้ไขแยก
จากสิทธิ์การเขียน JCL ของโปรแกรมเมอร์ทั่วไปได้ชัดเจนกว่า

---

## ขั้นตอนที่ 525: Overriding DD Statements ภายใน Procedure Step

### ทำไมต้อง Override

บางครั้ง Procedure มาตรฐานเกือบจะตรงกับที่ต้องการ 100% แล้ว ขาดแค่จุดเล็ก ๆ จุดเดียว เช่น
Cataloged Procedure `COBUCLG` (compile-link-go มาตรฐาน) มี step ชื่อ `GO` ที่รันโปรแกรมจริง
แต่ DD `SYSOUT` ของ step `GO` ถูกตั้งเป็น class เริ่มต้น ในขณะที่ job นี้ต้องการ SYSOUT class อื่น
— แทนที่จะคัดลอก Procedure ทั้งชุดมาแก้ (ซึ่งเสียประโยชน์ของ Cataloged Procedure ไปหมด) JCL มี
กลไก **Override** ให้ "แก้ไขเฉพาะจุด" โดยไม่แตะเนื้อหาส่วนอื่นของ Procedure

### ไวยากรณ์ Qualified Reference: stepname.ddname

```jcl
//MYJOB    JOB (ACCT123),'COBOL COURSE',CLASS=A,MSGCLASS=X
//STEPA    EXEC COBUCLG
//COB.SYSIN   DD DSN=MY.SOURCE.LIB(PROGRAM1),DISP=SHR
//GO.SYSOUT   DD SYSOUT=H
//GO.CUSTFILE DD DSN=PROD.CUSTOMER.MASTER,DISP=SHR
```

### อธิบายจุดสำคัญ

- `COBUCLG` เป็นตัวอย่าง Cataloged Procedure มาตรฐานของ IBM ที่มี 3 step ภายใน: `COB` (compile),
  `LKED` (link-edit), `GO` (execute) — ชื่อ step เหล่านี้กำหนดไว้ตายตัวโดยผู้เขียน Procedure
- `//COB.SYSIN DD ...` คือการ **override** DD ชื่อ `SYSIN` ที่อยู่ **ภายใน step ชื่อ `COB`**
  เท่านั้น รูปแบบคือ `stepname.ddname` คั่นด้วยจุด (`.`)
- `//GO.SYSOUT DD SYSOUT=H` override DD `SYSOUT` ของ step `GO` ให้ส่งไปยัง output class `H`
  แทนค่าเดิมที่ Procedure กำหนดไว้
- `//GO.CUSTFILE DD DSN=...` คือการ **เพิ่ม DD ใหม่** ที่ไม่มีอยู่เดิมใน Procedure เข้าไปใน step
  `GO` — เป็นวิธีมาตรฐานที่โปรแกรม COBOL ที่รันผ่าน Cataloged Procedure ใช้ผูกไฟล์ข้อมูลจริงของ
  ตัวเองเข้ากับ step `GO` (เพราะ Procedure กลางไม่มีทางรู้ล่วงหน้าว่าแต่ละโปรแกรมจะใช้ไฟล์อะไรบ้าง)

### กฎการ Override ที่ต้องจำ

1. Override DD ที่**มีอยู่แล้ว**ใน Procedure จะ**แทนที่ทั้ง DD statement เดิม** ไม่ใช่ผสานค่าบางส่วน
2. DD ใหม่ที่**ไม่มีอยู่เดิม** จะถูก**เพิ่มเข้าไป**ใน step นั้นตามปกติ
3. ลำดับการเขียน DD override **ต้องเรียงตามลำดับ step ภายใน Procedure เสมอ** (เช่น DD ของ `COB`
   ต้องมาก่อน DD ของ `LKED` และ `GO` ตามลำดับที่ Procedure กำหนดไว้ ไม่ใช่ลำดับที่ผู้เขียนต้องการ)

### ข้อควรระวัง

- ถ้าเขียนชื่อ step ผิด (เช่นพิมพ์ `GOO.SYSOUT` แทน `GO.SYSOUT`) JES จะถือว่าเป็นความพยายาม
  override step ที่ไม่มีอยู่จริง และปฏิเสธ JCL ทันทีตั้งแต่ขั้น JCL conversion
- Symbolic Parameter (ขั้นตอนที่ 523) กับ DD Override (ขั้นตอนนี้) เป็นกลไกคนละแบบ: Symbolic
  Parameter ใช้แทนค่าคงที่ **ภายใน** ข้อความ (เช่น DSN, UNIT) ส่วน DD Override ใช้แทนที่หรือเพิ่ม
  **ทั้ง DD statement** เข้าไปทั้งก้อน — ทั้งสองอย่างมักใช้ร่วมกันในงานจริง

### แบบฝึกหัดที่ 525.1

**โจทย์**: สมมติ Cataloged Procedure ชื่อ `RUNCOBOL` มี step เดียวชื่อ `RUN` ที่มี DD ชื่อ
`REPTFILE` อยู่แล้ว (ชี้ไปยัง dataset ทดสอบ) จงเขียน JCL ที่เรียกใช้ `RUNCOBOL` แล้ว override
`REPTFILE` ให้ชี้ไปยัง `PROD.MONTHLY.REPORT` แทน

**เฉลย**:

```jcl
//STEPA    EXEC RUNCOBOL
//RUN.REPTFILE DD DSN=PROD.MONTHLY.REPORT,DISP=SHR
```

---

## ขั้นตอนที่ 526: RETURN-CODE — กลไกสื่อสารผลลัพธ์ระหว่าง Step

### แนวคิดของ Return Code

ทุกครั้งที่โปรแกรม (COBOL หรือ utility ใด ๆ) จบการทำงานบน z/OS โปรแกรมนั้นจะส่งค่าตัวเลขกลับไปให้
ระบบเรียกว่า **Return Code (RC)** หรือ **Condition Code** ค่า `0` หมายถึงสำเร็จสมบูรณ์
ค่าที่มากกว่า 0 หมายถึงระดับความผิดปกติที่มากขึ้นตามลำดับ (ไม่มีมาตรฐานตายตัวว่าค่าใดหมายถึงอะไร
ขึ้นกับแต่ละโปรแกรม/utility กำหนดเอง แต่ธรรมเนียมทั่วไปที่พบบ่อยคือ):

| ค่า RC ทั่วไป | ความหมายตามธรรมเนียม (ไม่ตายตัว) |
|---|---|
| 0 | สำเร็จสมบูรณ์ (Normal completion) |
| 4 | สำเร็จแต่มีคำเตือนเล็กน้อย (Warning) |
| 8 | เกิดข้อผิดพลาดระดับปานกลาง (Error, อาจยังรันต่อได้บางส่วน) |
| 12 | ข้อผิดพลาดร้ายแรง (Severe error) |
| 16 | ข้อผิดพลาดวิกฤต ทำให้ผลลัพธ์ใช้งานไม่ได้ (Terminating error) |

### การกำหนด Return Code จากภายในโปรแกรม COBOL

โปรแกรม COBOL สามารถกำหนดค่า Return Code เองได้ผ่าน Special Register ชื่อ **`RETURN-CODE`**
ก่อนจบโปรแกรมด้วย `STOP RUN` หรือ `GOBACK`:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. VALIDATE-BATCH.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ERROR-COUNT       PIC 9(5) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VALIDATE-ALL-RECORDS.

           IF WS-ERROR-COUNT = 0
               MOVE 0 TO RETURN-CODE
               DISPLAY "VALIDATION PASSED - RC=0"
           ELSE IF WS-ERROR-COUNT < 10
               MOVE 4 TO RETURN-CODE
               DISPLAY "VALIDATION WARNINGS - RC=4"
           ELSE
               MOVE 16 TO RETURN-CODE
               DISPLAY "VALIDATION FAILED - RC=16"
           END-IF.
           STOP RUN.

       VALIDATE-ALL-RECORDS.
      *> validation logic would increment WS-ERROR-COUNT here
           CONTINUE.
```

> **หมายเหตุความสามารถของสภาพแวดล้อมนี้**: `RETURN-CODE` เป็น special register มาตรฐานของ
> COBOL-85 ขึ้นไป ที่ GnuCOBOL รองรับเช่นกัน โค้ดส่วน COBOL ข้างต้น **compile และรันได้จริงบน
> GnuCOBOL ทั่วไป** (ทดสอบแล้ว) — แต่**การที่ step ถัดไปใน JCL จะเห็นและตัดสินใจจากค่านี้ได้จริง
> ต้องอาศัย JES ของ z/OS จริง** ซึ่งไม่มีในสภาพแวดล้อมนี้ ส่วนนี้ของเนื้อหาจึงยังเป็นไวยากรณ์
> อ้างอิงเมื่อพูดถึงฝั่ง JCL

ทดสอบคอมไพล์ฝั่ง COBOL (ตรวจสอบเฉพาะว่า syntax ถูกต้องและ `RETURN-CODE` ใช้งานได้จริง):

```bash
cobc -x -o validate validate.cob
./validate
echo "Exit code seen by shell: $?"
```

**ผลลัพธ์จริงที่ยืนยันแล้ว** (กรณี `WS-ERROR-COUNT` เป็น 0 ตามค่าเริ่มต้นในตัวอย่างนี้):

```
VALIDATION PASSED - RC=0
Exit code seen by shell: 0
```

สังเกตว่า `RETURN-CODE` ของ COBOL ถูกส่งต่อเป็น **exit code ของโปรเซสจริง** แม้ในสภาพแวดล้อม
Linux/GnuCOBOL ก็ตาม (`echo $?` ของ shell คือกลไกเทียบเท่า Return Code ของ z/OS ในโลก Linux)
นี่คือเหตุผลที่บอกว่าแนวคิด Return Code ไม่ใช่เรื่องเฉพาะ Mainframe แต่เป็นแนวคิดสากลของระบบ
ปฏิบัติการที่ COBOL ยืมมาใช้เต็มรูปแบบบน z/OS ผ่าน JCL

### ข้อควรระวัง

- `RETURN-CODE` ต้องเป็นค่าตัวเลขเต็มระหว่าง 0–4095 (ขึ้นกับ implementation) ค่าที่ติดลบไม่ควรใช้
- ต้องกำหนดค่า `RETURN-CODE` **ก่อน** `STOP RUN`/`GOBACK` เสมอ ถ้าลืมกำหนด ค่าเริ่มต้นจะขึ้นกับ
  compiler แต่ละตัว (ไม่ควรพึ่งพาพฤติกรรม default เพราะไม่มีมาตรฐานตายตัว)

### แบบฝึกหัดที่ 526.1

**โจทย์**: จงปรับโปรแกรมข้างต้นให้กำหนด `WS-ERROR-COUNT` เป็น 15 แล้วทดสอบดูว่า Return Code
และข้อความที่แสดงเปลี่ยนไปเป็นอะไร

**เฉลย**: เปลี่ยน `VALUE 0` เป็น `VALUE 15` ที่ `WS-ERROR-COUNT` ผลลัพธ์จะเข้าเงื่อนไข
`ELSE` (เพราะ 15 ไม่น้อยกว่า 10) ทำให้ได้ผลลัพธ์ `VALIDATION FAILED - RC=16` และ
`echo $?` จะแสดง `16`

---

## ขั้นตอนที่ 527: COND Parameter — ควบคุมการรัน Step จาก Return Code

### แนวคิดของ COND

**`COND`** คือ parameter ที่ใส่บน `EXEC` statement เพื่อกำหนดเงื่อนไขว่า **step นี้ควรถูก "ข้าม"
(bypass) หรือไม่** โดยอิงจาก Return Code ของ step ก่อนหน้า ข้อควรทำความเข้าใจที่สำคัญที่สุดคือ
**ตรรกะของ `COND` "กลับด้าน" จากที่คนส่วนใหญ่คาดคิด**: เงื่อนไขที่ระบุใน `COND` คือเงื่อนไขที่ทำให้
step ถูก**ข้าม (ไม่รัน)** ไม่ใช่เงื่อนไขที่ทำให้รัน

### ไวยากรณ์พื้นฐาน

```jcl
//STEP1    EXEC PGM=VALIDATE
//STEP2    EXEC PGM=UPDATE,COND=(4,LT)
```

`COND=(4,LT)` แปลว่า **"ข้าม STEP2 ถ้า Return Code ของ step ก่อนหน้าน้อยกว่า (LT) 4"**
พูดอีกแบบคือ STEP2 จะ**รัน**ก็ต่อเมื่อ RC ของ STEP1 **มากกว่าหรือเท่ากับ 4** เท่านั้น — ซึ่งฟังดู
สวนทางกับสามัญสำนึกที่มักคิดว่า "รันต่อเมื่อสำเร็จ (RC ต่ำ)"

### รหัสเปรียบเทียบที่ใช้ได้ใน COND

| รหัส | ความหมาย |
|---|---|
| `GT` | Greater Than (มากกว่า) |
| `GE` | Greater than or Equal (มากกว่าหรือเท่ากับ) |
| `EQ` | Equal (เท่ากับ) |
| `LT` | Less Than (น้อยกว่า) |
| `LE` | Less than or Equal (น้อยกว่าหรือเท่ากับ) |
| `NE` | Not Equal (ไม่เท่ากับ) |

### ตัวอย่างที่ถูกต้องตามความต้องการทั่วไป: "ข้ามถ้า step ก่อนหน้าล้มเหลว"

ถ้าต้องการความหมายตามสามัญสำนึกจริง ๆ คือ **"ข้าม step นี้ถ้า step ก่อนหน้า RC สูงเกินไป (ล้มเหลว)"**
ต้องเขียนกลับด้านแบบนี้:

```jcl
//STEP1    EXEC PGM=VALIDATE
//STEP2    EXEC PGM=UPDATE,COND=(8,GE)
```

`COND=(8,GE)` แปลว่า **"ข้าม STEP2 ถ้า RC ของ STEP1 มากกว่าหรือเท่ากับ 8"** — นั่นคือ STEP2 จะ
รันต่อเมื่อ STEP1 จบด้วย RC ต่ำกว่า 8 (คือสำเร็จหรือมีแค่ warning เล็กน้อยเท่านั้น) ตรงกับความ
ต้องการทั่วไปที่ว่า "รันต่อเมื่อ step ก่อนหน้าไม่ได้ล้มเหลวหนัก"

### ข้อควรระวัง

- **นี่คือกับดักที่มือใหม่พลาดบ่อยที่สุดของ JCL ทั้งหมด**: จำสูตรนี้ไว้เสมอ — `COND` อธิบาย
  เงื่อนไข **"ข้าม" (bypass)** ไม่ใช่เงื่อนไข **"รัน" (execute)** ถ้าคิดสลับกันจะเขียน `LT`/`GT`
  ผิดด้านและได้ผลตรงข้ามกับที่ต้องการ 100%
- `COND` บน `EXEC` ของ step หนึ่ง จะเทียบกับ RC ของ **ทุก step ก่อนหน้าใน job เดียวกัน** ไม่ใช่แค่
  step ก่อนหน้าเพียง step เดียว (ถ้าระบุหลายเงื่อนไขคั่นด้วยจุลภาค แต่ละเงื่อนไขจะถูกตรวจสอบกับ
  step ก่อนหน้าทั้งหมดที่ยังไม่ถูกข้าม)
- `COND` บนบัตร `JOB` (ระดับ job ทั้งหมด ไม่ใช่ระดับ step) มีความหมายคล้ายกันแต่ใช้ยุติทั้ง job
  ทันทีเมื่อเข้าเงื่อนไข ไม่ใช่แค่ข้าม step เดียว

### แบบฝึกหัดที่ 527.1

**โจทย์**: จงเขียน `COND` บน STEP3 ให้มีความหมายว่า "ข้าม STEP3 ถ้า RC ของ step ก่อนหน้าเท่ากับ
ศูนย์เป๊ะเท่านั้น (คือรันเฉพาะตอนมีปัญหาบางอย่างเกิดขึ้น เช่น step ทำความสะอาดหลังความล้มเหลว)"

**เฉลย**: `COND=(0,EQ)` — แปลว่า "ข้าม STEP3 ถ้า RC เท่ากับ 0" ดังนั้น STEP3 จะรันก็ต่อเมื่อ RC
ไม่เท่ากับ 0 เท่านั้น ตรงกับรูปแบบการเขียน step "cleanup on failure" ที่พบได้ทั่วไปในงาน Batch จริง

---

## ขั้นตอนที่ 528: COND ขั้นสูง — หลายเงื่อนไขและ COND=EVEN/ONLY

### การระบุหลายเงื่อนไขพร้อมกัน

`COND` รองรับการระบุหลายเงื่อนไขพร้อมกันได้สูงสุด 8 ชุด โดยแต่ละชุดคั่นด้วยจุลภาค ถ้า**เงื่อนไขใด
เงื่อนไขหนึ่ง**เป็นจริง (ตรรกะ OR) step นั้นจะถูกข้ามทันที:

```jcl
//STEP4    EXEC PGM=REPORT,COND=((4,LT,STEP1),(8,LT,STEP2))
```

หมายความว่า **"ข้าม STEP4 ถ้า RC ของ STEP1 น้อยกว่า 4 หรือ RC ของ STEP2 น้อยกว่า 8"** — สังเกต
ว่ารอบนี้ระบุชื่อ step ที่จะเทียบด้วย (`STEP1`, `STEP2`) ทำให้ตรวจสอบเจาะจงกับ step ใดก็ได้
ไม่จำกัดแค่ step ล่าสุด

### COND=EVEN — รัน step แม้ step ก่อนหน้าล้มเหลว

ปกติแล้ว เมื่อ step ใด abend (ผิดพลาดร้ายแรงจนหยุดกลางคัน) step ถัดไปจะถูกข้ามโดยอัตโนมัติเสมอ
ไม่ว่าจะตั้ง `COND` ไว้อย่างไร **ยกเว้น** จะระบุ `COND=EVEN` ซึ่งบังคับให้ step นั้นรัน **แม้ step
ก่อนหน้าจะ abend ก็ตาม** — ใช้บ่อยกับ step ที่ทำหน้าที่ "ทำความสะอาด" (cleanup) หรือ "แจ้งเตือน"
(notification) ที่ต้องรันเสมอไม่ว่าผลลัพธ์ก่อนหน้าจะเป็นอย่างไร:

```jcl
//STEP1    EXEC PGM=UPDATE-MASTER
//STEP2    EXEC PGM=SEND-ALERT,COND=EVEN
```

### COND=ONLY — รัน step ก็ต่อเมื่อ step ก่อนหน้าล้มเหลว/ถูกข้ามเท่านั้น

`COND=ONLY` คือขั้วตรงข้าม: step จะรัน **ก็ต่อเมื่อมี step ก่อนหน้า abend หรือถูกข้ามไปแล้ว**
เท่านั้น ถ้าทุก step ก่อนหน้าสำเร็จปกติทั้งหมด step ที่มี `COND=ONLY` จะถูกข้ามเสมอ:

```jcl
//STEP1    EXEC PGM=VALIDATE
//STEP2    EXEC PGM=UPDATE
//STEP3    EXEC PGM=ROLLBACK,COND=ONLY
```

`STEP3` (ROLLBACK) จะรันเฉพาะเมื่อมีบางอย่างผิดพลาดใน `STEP1` หรือ `STEP2` เท่านั้น เป็นรูปแบบ
มาตรฐานสำหรับ step กู้คืนข้อมูลเมื่อเกิดความล้มเหลวกลางทาง

### เปรียบเทียบสรุป

| การตั้งค่า | ความหมาย |
|---|---|
| `COND=(n,op)` | ข้าม step นี้ถ้าเข้าเงื่อนไขเปรียบเทียบกับ RC ก่อนหน้า |
| `COND=EVEN` | รัน step นี้เสมอ แม้ step ก่อนหน้าจะ abend |
| `COND=ONLY` | รัน step นี้เฉพาะเมื่อมี step ก่อนหน้า abend/ถูกข้ามเท่านั้น |
| ไม่ระบุ `COND` | rank ปกติ: ข้าม step อัตโนมัติถ้ามี step ก่อนหน้า abend |

### ข้อควรระวัง

- `COND=EVEN` และ `COND=ONLY` ใช้**แทนที่** การตรวจสอบ RC แบบตัวเลข ไม่สามารถผสมกับ
  `COND=(n,op)` ในบรรทัดเดียวกันได้ (ต้องเลือกรูปแบบใดรูปแบบหนึ่ง)
- ตั้งแต่มาตรฐาน JCL รุ่นใหม่ (z/OS สมัยใหม่) แนะนำให้ใช้ **`IF/THEN/ELSE`** (ขั้นตอนที่ 529)
  แทน `COND` แบบตัวเลขเมื่อ logic ซับซ้อนขึ้น เพราะ `COND` อ่านเข้าใจยากและมีตรรกะ "กลับด้าน"
  ที่สร้างความสับสนดังที่เห็นในขั้นตอนที่ 527

### แบบฝึกหัดที่ 528.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `COND=EVEN` กับการไม่ใส่ `COND` เลย

**เฉลย**: ถ้าไม่ใส่ `COND` เลย พฤติกรรม default ของ JCL คือข้าม step โดยอัตโนมัติทันทีที่มี step
ก่อนหน้า abend (แม้ผู้เขียนจะไม่ได้ตั้งใจ) ในขณะที่ `COND=EVEN` เป็นการสั่งอย่างชัดเจนว่า "ให้รัน
step นี้ต่อไปเสมอไม่ว่า step ก่อนหน้าจะ abend หรือไม่" ซึ่งมีประโยชน์มากสำหรับ step ที่ต้องทำงาน
ไม่ว่าผลลัพธ์ก่อนหน้าจะเป็นอย่างไร เช่น step ส่งอีเมลแจ้งเตือนผลการรัน job

---

## ขั้นตอนที่ 529: IF/THEN/ELSE/ENDIF — JCL เชิงเงื่อนไขสมัยใหม่

### ทำไมต้องมี IF/THEN/ELSE ทั้งที่มี COND แล้ว

`COND` มีข้อจำกัดสำคัญคือ **ตรวจสอบได้แค่ตัวเลข Return Code เท่านั้น** และตรรกะ "กลับด้าน" ที่
สร้างความสับสน (ขั้นตอนที่ 527) ตั้งแต่ z/OS เวอร์ชันใหม่ ๆ IBM จึงเพิ่มโครงสร้าง
**`IF/THEN/ELSE/ENDIF`** เข้ามาใน JCL ให้เขียนเงื่อนไขได้อ่านง่ายขึ้นมาก คล้ายภาษาโปรแกรมทั่วไป
และรองรับการตรวจสอบที่หลากหลายกว่า `COND` เช่น `ABEND`, `RC`, `RUN` และผสมเงื่อนไขด้วย
`AND`/`OR` ได้

### ไวยากรณ์พื้นฐาน

```jcl
//MYJOB    JOB (ACCT123),'COBOL COURSE',CLASS=A,MSGCLASS=X
//*
//STEP1    EXEC PGM=VALIDATE
//*
//IF1      IF (STEP1.RC = 0) THEN
//STEP2    EXEC PGM=UPDATE-MASTER
//ENDIF1   ENDIF
//*
//IF2      IF (STEP1.RC > 0) THEN
//STEP3    EXEC PGM=SEND-ALERT
//ELSE
//STEP4    EXEC PGM=PRINT-SUCCESS
//ENDIF2   ENDIF
```

### อธิบายจุดสำคัญ

- `IF (STEP1.RC = 0) THEN` อ่านตรงตัวตามสามัญสำนึกเลย: **"ถ้า RC ของ STEP1 เท่ากับ 0 ให้ทำสิ่ง
  ต่อไปนี้"** ไม่มีตรรกะกลับด้านแบบ `COND` อีกต่อไป
- ชื่อที่ขึ้นต้นบัตร (`IF1`, `ENDIF1`, `IF2`, `ENDIF2`) เป็นชื่อป้ายกำกับที่ผู้เขียนตั้งเอง (ไม่บังคับ
  ต้องมี แต่นิยมใส่เพื่ออ่านง่ายเมื่อมีหลายบล็อก IF ซ้อนกัน)
- `ELSE` ทำงานเหมือนภาษาโปรแกรมทั่วไป: ถ้าเงื่อนไขใน `IF` เป็นเท็จ ให้ทำ step ที่อยู่ใต้ `ELSE`
  แทน
- `ENDIF` ปิดท้ายบล็อกเสมอ คล้าย `END-IF` ของ COBOL

### ตัวดำเนินการเปรียบเทียบและคีย์เวิร์ดพิเศษที่ใช้ได้

| รูปแบบ | ความหมาย |
|---|---|
| `stepname.RC = n` | เปรียบเทียบ Return Code ของ step ที่ระบุกับค่า n (`=`,`>`,`<`,`>=`,`<=`,`¬=`) |
| `stepname.ABEND` | เป็นจริงถ้า step นั้น abend |
| `ABENDCC = Sxxx` | ตรวจสอบรหัส System Abend Code เจาะจง |
| `stepname.RUN` | เป็นจริงถ้า step นั้นถูกรัน (ไม่ถูกข้าม) |
| `AND`, `OR`, `NOT` | ใช้รวมหลายเงื่อนไขเข้าด้วยกัน |

### ตัวอย่างผสมเงื่อนไขซับซ้อน

```jcl
//IF3      IF (STEP1.RC = 0) AND (STEP2.RC <= 4) THEN
//STEP5    EXEC PGM=FINALIZE
//ENDIF3   ENDIF
```

### ข้อควรระวัง

- `IF/THEN/ELSE/ENDIF` ของ JCL เป็นฟีเจอร์ของ **z/OS เวอร์ชันใหม่เท่านั้น** (ตั้งแต่ z/OS V1R4
  ขึ้นไปตามเอกสาร IBM) องค์กรที่ยังใช้ z/OS เวอร์ชันเก่ามากอาจยังต้องพึ่งพา `COND` แบบดั้งเดิม
- โครงสร้าง `IF/ENDIF` ของ JCL **ซ้อนกันได้** (nested) ต่างจาก In-Stream Procedure ที่ห้ามซ้อน
  แต่ควรระวังเรื่องความอ่านง่ายเมื่อซ้อนลึกเกิน 2-3 ชั้น
- อย่าลืมว่า `IF/THEN/ELSE` ควบคุมว่า step **จะรันหรือไม่** เท่านั้น ไม่ได้เปลี่ยนแปลงเนื้อหาของ
  DD statement หรือพารามิเตอร์ภายใน step แบบที่ Symbolic Parameter ทำได้

### แบบฝึกหัดที่ 529.1

**โจทย์**: จงเขียน JCL เชิงเงื่อนไขที่มีความหมายว่า "ถ้า STEP1 abend ให้รัน step กู้คืนข้อมูล
(RECOVERY) มิฉะนั้นให้รัน step พิมพ์รายงานสรุป (SUMMARY) ตามปกติ"

**เฉลย**:

```jcl
//IFCHK    IF (STEP1.ABEND) THEN
//RECOVER  EXEC PGM=RECOVERY
//ELSE
//SUMMARY  EXEC PGM=SUMMARY-REPORT
//ENDIF    ENDIF
```

---

## ขั้นตอนที่ 530: กรณีศึกษารวม — Batch Job หลาย Step พร้อม Procedure และเงื่อนไขครบวงจร

### โจทย์ของกรณีศึกษา

จำลองงาน Batch ประจำคืนขององค์กรธนาคาร ที่ต้อง: (1) ตรวจสอบความถูกต้องของไฟล์ธุรกรรมนำเข้าก่อน
(2) อัปเดตไฟล์หลักถ้าตรวจสอบผ่าน (3) ส่งอีเมลแจ้งเตือนทีมงานเสมอไม่ว่าผลลัพธ์จะเป็นอย่างไร
(4) พิมพ์รายงานสรุปเฉพาะเมื่อทุกอย่างสำเร็จ โดยใช้ Cataloged Procedure สำหรับ step
compile-link-go ของแต่ละโปรแกรม COBOL

### JCL ฉบับสมบูรณ์ (ไวยากรณ์อ้างอิง)

```jcl
//NIGHTBAT JOB (ACCT456),'NIGHTLY BATCH',CLASS=A,MSGCLASS=X,
//             NOTIFY=&SYSUID
//JCLLIB   ORDER=(PROD.BANK.PROCLIB)
//*
//********************************************************
//* STEP 1: VALIDATE INCOMING TRANSACTION FILE           *
//********************************************************
//STEP1    EXEC RUNCOBOL,PGMNAME=TXNVALID
//RUN.TXNIN    DD DSN=PROD.BANK.TXN.DAILY,DISP=SHR
//RUN.ERRRPT   DD SYSOUT=*
//*
//********************************************************
//* STEP 2: UPDATE MASTER FILE - ONLY IF VALIDATION OK    *
//********************************************************
//IF1      IF (STEP1.RC = 0) THEN
//STEP2    EXEC RUNCOBOL,PGMNAME=TXNUPDATE
//RUN.TXNIN    DD DSN=PROD.BANK.TXN.DAILY,DISP=SHR
//RUN.MASTER   DD DSN=PROD.BANK.CUSTOMER.MASTER,DISP=OLD
//ENDIF1   ENDIF
//*
//********************************************************
//* STEP 3: SEND ALERT - ALWAYS RUNS, EVEN AFTER ABEND    *
//********************************************************
//STEP3    EXEC RUNCOBOL,PGMNAME=SENDALERT,COND=EVEN
//RUN.ALERTMSG DD SYSOUT=*
//*
//********************************************************
//* STEP 4: PRINT SUMMARY - ONLY WHEN STEP1 AND STEP2 OK  *
//********************************************************
//IF2      IF (STEP1.RC = 0) AND (STEP2.RC <= 4) THEN
//STEP4    EXEC RUNCOBOL,PGMNAME=DAILYRPT
//RUN.RPTOUT   DD SYSOUT=*
//ENDIF2   ENDIF
```

### อธิบายภาพรวม

- `RUNCOBOL` คือ Cataloged Procedure สมมติที่องค์กรเตรียมไว้ (คล้าย `COBUCLG` มาตรฐานของ IBM แต่
  ปรับแต่งเพิ่มพารามิเตอร์ `PGMNAME` ให้ระบุชื่อโปรแกรมที่จะรันในแต่ละ step) — ทุก step ไม่ต้อง
  เขียน DD ของ compiler/linker ซ้ำเลยเพราะอยู่ใน Procedure กลางหมดแล้ว
- `STEP1` ตรวจสอบไฟล์ธุรกรรม แล้วกำหนด `RETURN-CODE` ภายในโปรแกรม COBOL ของมันเอง (ตามรูปแบบ
  ขั้นตอนที่ 526)
- `IF1...ENDIF1` ควบคุมว่า `STEP2` (อัปเดตไฟล์หลัก) จะรันก็ต่อเมื่อ `STEP1` สำเร็จสมบูรณ์
  (RC=0) เท่านั้น — ใช้ไวยากรณ์ที่อ่านง่ายกว่า `COND` แบบดั้งเดิมมาก
- `STEP3` (ส่งอีเมล) ใช้ `COND=EVEN` เพื่อการันตีว่าทีมงานจะได้รับแจ้งเตือนเสมอ ไม่ว่า `STEP1`
  หรือ `STEP2` จะล้มเหลวหรือ abend ก็ตาม — เป็นรูปแบบมาตรฐานของ step แจ้งเตือนในงาน Batch จริง
- `IF2...ENDIF2` ควบคุมว่ารายงานสรุปจะพิมพ์ก็ต่อเมื่อทั้งการตรวจสอบและการอัปเดตสำเร็จ (ยอมรับ
  warning เล็กน้อยได้ถ้า RC ของ STEP2 ไม่เกิน 4)

### ข้อควรระวัง

- กรณีศึกษานี้เป็น **ไวยากรณ์อ้างอิงที่ถูกต้องตามมาตรฐาน** แต่ **ไม่สามารถ submit หรือรันจริงได้
  ในสภาพแวดล้อมของหลักสูตรนี้** เนื่องจากไม่มี z/OS/JES/PROCLIB จริง หากต้องการทดสอบรันจริง
  ต้องใช้ Mainframe จริงหรือ Mainframe Emulator (เช่น Hercules พร้อมระบบปฏิบัติการ MVS/TK4-)
- ในการออกแบบจริง ควรตั้งชื่อ step และ label ของ `IF`/`ENDIF` ให้สื่อความหมายชัดเจนเสมอ เพราะเมื่อ
  job มีหลายสิบ step ผสมกับเงื่อนไขซ้อนกันหลายชั้น ความอ่านง่ายจะกลายเป็นปัจจัยสำคัญมากในการ
  บำรุงรักษาระยะยาว

### แบบฝึกหัดที่ 530.1

**โจทย์**: จงเพิ่ม step ที่ 5 ต่อจาก `STEP4` ที่ทำหน้าที่ "เก็บ backup ไฟล์หลัก" โดยให้รันเฉพาะเมื่อ
`STEP2` (อัปเดตไฟล์หลัก) สำเร็จเท่านั้น ไม่สนใจผลลัพธ์ของ step อื่น

**เฉลย**:

```jcl
//IF3      IF (STEP2.RC = 0) THEN
//STEP5    EXEC RUNCOBOL,PGMNAME=BACKUPMSTR
//RUN.MASTER   DD DSN=PROD.BANK.CUSTOMER.MASTER,DISP=SHR
//RUN.BACKUP   DD DSN=PROD.BANK.CUSTOMER.BKUP,
//             DISP=(NEW,CATLG,DELETE),UNIT=SYSDA,
//             SPACE=(TRK,(50,10))
//ENDIF3   ENDIF
```

การตรวจสอบเฉพาะ `STEP2.RC = 0` (ไม่ผสมกับเงื่อนไขของ step อื่น) ทำให้ logic ของ step นี้เป็น
อิสระและชัดเจน: backup จะเกิดขึ้นก็ต่อเมื่อการอัปเดตไฟล์หลักสำเร็จสมบูรณ์จริง ๆ เท่านั้น

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้กลไก JCL ขั้นสูงที่ทำให้ Batch Job บน Mainframe ใช้งานได้จริงในระดับ
องค์กรขนาดใหญ่:

- ปัญหาการเขียน DD ซ้ำซากที่ Procedure เข้ามาแก้ไข
- In-Stream Procedure ผ่านบัตร `PROC`/`PEND` และข้อจำกัดสูงสุด 15 ตัวต่อ job
- Symbolic Parameters (`&name`) ที่ทำให้ Procedure เดียวใช้ซ้ำได้หลายบริบท
- Cataloged Procedure และ `JCLLIB ORDER=(...)` สำหรับ library ที่ไม่ใช่ค่ามาตรฐาน
- การ Override DD statement ภายใน Procedure step ด้วย qualified name (`stepname.ddname`)
- `RETURN-CODE` special register ของ COBOL ที่ **ทดสอบคอมไพล์และรันได้จริง** และเชื่อมโยงกับ
  แนวคิด exit code สากลของระบบปฏิบัติการ
- `COND` parameter และตรรกะ "กลับด้าน" ที่ต้องระวังเป็นพิเศษ (เงื่อนไขคือเงื่อนไข "ข้าม" ไม่ใช่
  "รัน")
- `COND=EVEN` และ `COND=ONLY` สำหรับ step แจ้งเตือน/กู้คืนข้อมูล
- `IF/THEN/ELSE/ENDIF` ของ JCL สมัยใหม่ที่อ่านเข้าใจง่ายกว่า `COND` มาก
- กรณีศึกษารวม Batch Job ธนาคารจำลองที่ผสานทุกเทคนิคเข้าด้วยกัน

**ย้ำอีกครั้ง**: เนื้อหาทั้งหมดใน Part นี้เป็นไวยากรณ์ JCL มาตรฐานอ้างอิงที่ถูกต้องตามเอกสาร IBM
แต่ต้องใช้ z/OS จริงหรือ Mainframe Emulator จึงจะ submit และรันได้จริง ส่วนโค้ด COBOL ที่เกี่ยวกับ
`RETURN-CODE` ได้ทดสอบคอมไพล์และรันจริงด้วย GnuCOBOL แล้วในขั้นตอนที่ 526

Part ถัดไป (**Part 054**) จะพาคุณออกจากโลกของ Batch Processing ชั่วคราว เพื่อทำความรู้จักกับ
**TSO/ISPF** — สภาพแวดล้อมเชิงโต้ตอบ (interactive) ที่โปรแกรมเมอร์ Mainframe ใช้แก้ไขโค้ด จัดการ
dataset และตรวจสอบผลลัพธ์ของ job ที่เพิ่งเรียนไปใน Part นี้

**[← กลับไป Part 052: JCL เบื้องต้น](part-052-jcl-basics.md)** | **[ไปยัง Part 054: TSO/ISPF: การใช้งานเบื้องต้น →](part-054-tso-ispf.md)**
