# Part 061: CICS เบื้องต้น: Transaction Processing (ขั้นตอนที่ 601–610)

## คำนำของ Part นี้

ตลอดเฟส 4 ของหลักสูตร (Part 051 เป็นต้นมา) เราเรียนรู้เทคโนโลยีของโลก **Batch Processing**:
JCL สั่งรันโปรแกรม, VSAM/DB2 เก็บข้อมูล, โปรแกรมทำงานตั้งแต่ต้นจนจบโดยไม่มีใครโต้ตอบระหว่างทาง
เหมาะกับงานประมวลผลจำนวนมากตอนกลางคืน (เช่น ปิดบัญชีสิ้นวัน) แต่คำถามคือ: แล้วเมื่อพนักงานธนาคารที่
หน้าเคาน์เตอร์กดปุ่มถอนเงินให้ลูกค้า ระบบต้องตอบสนอง**ทันที**ภายในเสี้ยววินาที ระบบแบบ Batch ไม่
สามารถตอบโจทย์นี้ได้เลย

**CICS (Customer Information Control System)** คือคำตอบของ IBM สำหรับโลกที่สอง: **Online
Transaction Processing (OLTP)** — ระบบที่รองรับผู้ใช้นับพันนับหมื่นคนพร้อมกัน แต่ละคนกดปุ่ม กรอกข้อมูล
และคาดหวังคำตอบแทบจะทันที Part นี้จะพาคุณทำความรู้จักกับ CICS ตั้งแต่แนวคิดพื้นฐาน สถาปัตยกรรม
Transaction ID ไปจนถึงไวยากรณ์ Command-Level API (`EXEC CICS ... END-EXEC`) ที่ใช้เขียนโปรแกรม CICS
จริง

> **ย้ำเตือนเรื่องข้อจำกัดสภาพแวดล้อม**: CICS เป็น **Transaction Processing Monitor** ที่ต้องรันอยู่
> บน z/OS จริง (หรือ Emulator เฉพาะทางอย่าง Hercules + z/OS สำหรับผู้ที่จริงจังมาก) พร้อม **CICS
> Transaction Server** — ซอฟต์แวร์ที่**ไม่มีอยู่ในสภาพแวดล้อม GnuCOBOL ของหลักสูตรนี้เลย**
> โปรแกรมที่มี `EXEC CICS ... END-EXEC` ต้องผ่าน **CICS Translator** ก่อนคอมไพล์ (แนวคิดคล้ายกับ
> DB2 Precompiler ใน Part 058 แต่เป็นเครื่องมือคนละตัว) ซึ่งไม่มีในสภาพแวดล้อมนี้เช่นกัน — เราทดสอบ
> จริงแล้วว่า `cobc` ธรรมดาพยายามคอมไพล์ `EXEC CICS RETURN END-EXEC` แล้วได้ error ทันที:
> `error: unknown statement 'EXEC'`
>
> ตัวอย่าง `EXEC CICS ... END-EXEC` ทุกตัวอย่างใน Part 061-064 จึงเป็น **ไวยากรณ์อ้างอิง
> (Reference Syntax)** ตามมาตรฐาน IBM CICS ที่ถูกต้องแม่นยำ แต่**ไม่สามารถคอมไพล์หรือรันได้จริงบน
> สภาพแวดล้อมนี้** จะกำกับด้วยป้าย `⚠️ REFERENCE SYNTAX` เสมอ ส่วนกลไก COBOL มาตรฐานที่อยู่เบื้อง
> หลัง (เช่น `LINKAGE SECTION` สำหรับ COMMAREA) จะสาธิตแยกต่างหากโดยระบุชัดเจนว่า "ทดสอบจริงแล้ว"
> เมื่อทำได้

---

## ขั้นตอนที่ 601: CICS คืออะไร — Online Transaction Processing vs Batch

### นิยามของ CICS

**CICS (Customer Information Control System)** คือ **Transaction Processing Monitor** ของ IBM
ที่ทำงานอยู่บน z/OS ทำหน้าที่เป็น "ตัวกลาง" ระหว่างผู้ใช้ปลายทาง (ผ่านหน้าจอ Terminal หรือระบบภายนอก)
กับโปรแกรม COBOL/PL/I ที่ implement ตรรกะทางธุรกิจจริง โดยจัดการเรื่องยาก ๆ ที่โปรแกรมเมอร์ไม่ต้อง
กังวลเอง เช่น การจัดการผู้ใช้หลายพันคนพร้อมกัน, การจัดสรรหน่วยความจำ, การจัดคิวงาน, ความปลอดภัย,
และการกู้คืนระบบเมื่อเกิดปัญหา

### เปรียบเทียบ Batch กับ CICS Online

| คุณสมบัติ | Batch (JCL, Part 052-053) | CICS Online (Part 061-064) |
|---|---|---|
| เริ่มทำงานเมื่อไร | ตามตารางเวลาหรือสั่งรันครั้งเดียว | ทันทีที่ผู้ใช้พิมพ์ Transaction ID |
| ปฏิสัมพันธ์กับผู้ใช้ | ไม่มี — รันจนจบโดยไม่มีใครโต้ตอบ | โต้ตอบกับผู้ใช้แบบ real-time ผ่านหน้าจอ |
| ปริมาณข้อมูลที่ประมวลผล | จำนวนมาก (พันถึงล้าน record ต่อรอบ) | น้อยต่อครั้ง (1 ธุรกรรมต่อการเรียก) |
| จำนวนผู้ใช้พร้อมกัน | 1 (โปรแกรมเดียวรันอยู่) | หลายพันถึงหลายหมื่นคนพร้อมกัน |
| เวลาตอบสนองที่คาดหวัง | นาทีถึงชั่วโมง | มิลลิวินาทีถึงวินาที (Sub-second Response) |
| ตัวอย่างการใช้งาน | ปิดบัญชีสิ้นวัน, คำนวณดอกเบี้ยรายเดือน | ธุรกรรม ATM, หน้าจอพนักงานธนาคาร, จองตั๋วเครื่องบิน |

### ทำไมต้องมี CICS แยกจาก z/OS เฉย ๆ

z/OS เองมีความสามารถรันโปรแกรมได้อยู่แล้ว (ผ่าน JCL) แต่การรันโปรแกรมหนึ่งครั้งต่อผู้ใช้หนึ่งคนแบบ
Batch (เปิด-ปิด Address Space ทุกครั้ง) จะช้าเกินไปและสิ้นเปลืองทรัพยากรมหาศาลเมื่อมีผู้ใช้เป็นพัน
เป็นหมื่นคนพร้อมกัน **CICS แก้ปัญหานี้ด้วยการรัน "Region" เดียวที่คงอยู่ตลอดเวลา** (ไม่ต้องเปิด-ปิด
Address Space ใหม่ทุกธุรกรรม) แล้วให้โปรแกรม COBOL หลายพันชุดที่ compile ไว้ล่วงหน้าทำงานสลับกันไป
มาภายใน Region เดียวกันอย่างมีประสิทธิภาพ

### ข้อควรระวัง

- อย่าสับสนระหว่าง **CICS Region** (พื้นที่ทำงานที่ z/OS จัดสรรให้ CICS) กับ **DB2 Subsystem**
  (ที่เรียนใน Part 057-060) ทั้งสองเป็นซอฟต์แวร์คนละตัวที่ทำงานแยกกันบน z/OS แต่มักทำงานร่วมกันเสมอ
  ในระบบจริง (โปรแกรม CICS เรียก Embedded SQL ไปหา DB2 ได้ปกติ)
- CICS **ไม่ใช่ภาษาโปรแกรม** แต่เป็น Middleware/Monitor ที่โปรแกรมภาษาต่าง ๆ (COBOL, PL/I, Assembler,
  Java) เรียกใช้บริการผ่าน API เฉพาะของมัน

### แบบฝึกหัดที่ 601.1

**โจทย์**: จงอธิบายว่าทำไมระบบ ATM ธนาคารจึงต้องใช้ CICS (หรือเทคโนโลยี OLTP อื่นที่คล้ายกัน) แทนที่
จะใช้ระบบ Batch ตามที่เรียนมาใน Part 051-057

**เฉลย**: เพราะระบบ ATM ต้องการ**การตอบสนองแบบ real-time**ต่อผู้ใช้แต่ละคนที่มาใช้บริการในเวลาต่างกัน
โดยสุ่ม (ไม่สามารถรอให้ถึงรอบ Batch ตอนกลางคืนได้) และต้องรองรับผู้ใช้จำนวนมากพร้อมกันทั่วประเทศ/โลก
ระบบ Batch ออกแบบมาสำหรับประมวลผลข้อมูลจำนวนมากเป็นชุดตามตารางเวลาที่กำหนดไว้ล่วงหน้า ไม่ได้ออกแบบ
มาให้ตอบสนองต่อ "เหตุการณ์" (event) ที่เกิดขึ้นแบบสุ่มจากผู้ใช้จำนวนมากพร้อมกันในเวลาจริง จึงต้องใช้
CICS ซึ่งเป็น Transaction Processing Monitor ที่ออกแบบมาเพื่อการนี้โดยเฉพาะ

---

## ขั้นตอนที่ 602: สถาปัตยกรรม CICS — Region, Terminal, และ Transaction ID

### ภาพรวมสถาปัตยกรรม

```
[ Terminal / เครื่องพนักงาน ]
        |
        | ผู้ใช้พิมพ์ Transaction ID (เช่น "ATMW")
        v
[ CICS Region บน z/OS ]
        |
        | CICS ค้นหาว่า Transaction ID "ATMW" ผูกกับโปรแกรมชื่ออะไร
        | (ดูจาก Program Control Table / Resource Definition)
        v
[ โหลดและรันโปรแกรม COBOL ที่ผูกกับ Transaction นั้น ]
        |
        | โปรแกรมทำงาน อาจเรียก DB2, VSAM, ส่งหน้าจอกลับ ฯลฯ
        v
[ ส่งผลลัพธ์กลับไปที่ Terminal ]
```

### Transaction ID (Trans-ID)

**Transaction ID** คือรหัส 1-4 ตัวอักษรที่ผู้ใช้พิมพ์ (หรือระบบส่งมา) เพื่อ "เรียก" ให้ CICS เริ่ม
ทำงานธุรกรรมหนึ่ง ๆ เช่น `ATMW` (ATM Withdrawal), `TELR` (Teller) เป็นชื่อสั้น ๆ ที่องค์กรกำหนดขึ้นเอง
แล้วลงทะเบียนไว้ใน **CSD (CICS System Definition)** ให้ CICS รู้ว่า Trans-ID นี้ผูกกับโปรแกรม COBOL
ตัวไหน

### ตัวอย่างการลงทะเบียน Transaction (ไวยากรณ์อ้างอิง — คำสั่งของ Resource Definition Online, RDO)

```
⚠️ REFERENCE SYNTAX - ไม่สามารถคอมไพล์/รันได้ในสภาพแวดล้อมนี้

CEDA DEFINE TRANSACTION(ATMW)
     GROUP(BANKGRP)
     PROGRAM(ATMWDRAW)
     TWASIZE(0)
     TASKDATALOC(BELOW)
```

คำสั่งนี้บอก CICS ว่า: เมื่อมีผู้พิมพ์ `ATMW` ที่ Terminal ให้ CICS โหลดและเรียกโปรแกรมชื่อ
`ATMWDRAW` ขึ้นมาทำงาน (`CEDA` คือเครื่องมือ Online Resource Definition ที่ทำงานผ่านหน้าจอ 3270
เพื่อกำหนดค่า CICS Resource ต่าง ๆ)

### จุดที่ทำให้ CICS มีประสิทธิภาพสูง: Multi-threading ภายใน Region เดียว

CICS Region หนึ่งสามารถรันหลายๆ Transaction (จากผู้ใช้คนละคน) พร้อมกันได้ภายใน Address Space
เดียวโดยการสลับ (Multiplex) การทำงานอย่างรวดเร็ว ทำให้ไม่ต้องเปิด Address Space ใหม่ทุกครั้งที่มี
ผู้ใช้เข้ามาเหมือนโปรแกรม Batch — นี่คือกุญแจสำคัญที่ทำให้ CICS รองรับผู้ใช้จำนวนมากได้ด้วยทรัพยากร
ที่จำกัด

### ข้อควรระวัง

- Transaction ID ต้องไม่ซ้ำกันภายใน CICS Region เดียวกัน และความยาวสูงสุดคือ 4 ตัวอักษรตาม
  ข้อจำกัดดั้งเดิมของ CICS (สืบเนื่องจากยุคที่หน้าจอ Terminal มีทรัพยากรจำกัดมาก)
- โปรแกรมหนึ่งตัวสามารถถูกผูกกับหลาย Transaction ID ได้ (เช่น Trans-ID ต่างกันแต่เรียกโปรแกรมเดียวกัน
  ด้วยพารามิเตอร์เริ่มต้นต่างกัน) แต่ในทางกลับกัน Transaction ID หนึ่งต้องผูกกับโปรแกรมเริ่มต้นเพียง
  โปรแกรมเดียวเท่านั้น

### แบบฝึกหัดที่ 602.1

**โจทย์**: จงอธิบายว่าทำไม CICS Region เดียวจึงสามารถรองรับผู้ใช้หลายพันคนพร้อมกันได้ ทั้งที่ z/OS
มีทรัพยากร (CPU, memory) จำกัด

**เฉลย**: เพราะ CICS ใช้เทคนิค**การสลับงานหลายชุดภายใน Address Space เดียว** (คล้ายแนวคิด
Multi-threading) แทนที่จะเปิด Address Space ใหม่ทุกครั้งที่มีผู้ใช้เข้ามาเหมือนโปรแกรม Batch ที่ต้อง
เปิด Job/Address Space แยกทุกครั้ง วิธีนี้ลด Overhead การจัดสรรทรัพยากรระบบปฏิบัติการอย่างมาก ทำให้
CICS Region เดียวสามารถให้บริการผู้ใช้จำนวนมากพร้อมกันได้ด้วยทรัพยากรที่ประหยัดกว่าการเปิด Process
แยกสำหรับผู้ใช้แต่ละคนมาก

---

## ขั้นตอนที่ 603: EXEC CICS...END-EXEC และ CICS Translator

### ไวยากรณ์พื้นฐาน

เหมือนกับ Embedded SQL (Part 058) โปรแกรม CICS เขียนคำสั่งพิเศษปะปนอยู่ในโค้ด COBOL ผ่านกรอบ
`EXEC CICS ... END-EXEC`:

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               RETURN
           END-EXEC.
```

### CICS Translator — เทียบเคียงกับ DB2 Precompiler

เช่นเดียวกับที่ Part 058 อธิบายวงจร Precompile-Compile-Bind ของ DB2 โปรแกรม CICS ก็ต้องผ่าน
เครื่องมือพิเศษที่เรียกว่า **CICS Translator** ก่อนที่ COBOL Compiler ตัวจริงจะประมวลผลได้:

```
[COBOL + EXEC CICS source]
        |
        v
  CICS TRANSLATOR (เช่น DFHECP1$ หรือ Integrated Translator สมัยใหม่)
      - อ่านบล็อก EXEC CICS ทั้งหมด
      - แทนที่ด้วย CALL ไปยัง CICS runtime (ผ่าน DFHEIBLK/DFHCOMMAREA)
      - ผลลัพธ์: Modified COBOL Source
        |
        v
  COBOL COMPILER (คอมไพล์ตามปกติ)
        |
        v
  LINK-EDIT (รวมกับ CICS Runtime Library เป็น Load Module พร้อมรัน)
```

เราทดสอบจริงแล้วว่าเมื่อป้อน `EXEC CICS RETURN END-EXEC` ให้ `cobc` ธรรมดา (ไม่มี CICS Translator)
จะได้ error ทันที:

```
error: unknown statement 'EXEC'
```

ต่างจากกรณี `EXEC SQL` ใน Part 058 ที่ `cobc` พยายามตีความ (แล้วหา error เชิงโครงสร้างอื่น) กรณี
`EXEC CICS` นี้ `cobc` ไม่รู้จักคำว่า `EXEC` ในบริบทนี้เลยตั้งแต่แรก เพราะไม่ได้ตามด้วย `SQL`
ที่บาง compiler อาจมี built-in behavior เล็กน้อยรองรับ

### ความแตกต่างสำคัญจากโปรแกรม Batch: ไม่มี PROCEDURE DIVISION USING แบบ Main Program

โปรแกรม CICS ไม่ได้รับ parameter ผ่าน JCL แบบโปรแกรม Batch (Part 052) แต่รับข้อมูลผ่านกลไกพิเศษของ
CICS เอง 2 ทางหลัก: **EIB (Execute Interface Block)** ที่จะเรียนในขั้นตอนที่ 604 และ **COMMAREA**
ที่จะเรียนในขั้นตอนที่ 607

### ข้อควรระวัง

- ห้ามลืมว่า CICS Translator กับ DB2 Precompiler เป็นเครื่องมือคนละตัว หากโปรแกรมมีทั้ง `EXEC CICS`
  และ `EXEC SQL` ปนกัน (พบได้บ่อยมากในโปรแกรมจริงที่ทำทั้ง Transaction Processing และเข้าถึง DB2)
  ต้องรันผ่าน**ทั้งสองเครื่องมือตามลำดับที่ถูกต้อง** โดยทั่วไปนิยมรัน CICS Translator ก่อน แล้วตามด้วย
  DB2 Precompiler เนื่องจาก DB2 Precompiler สมัยใหม่ส่วนใหญ่ออกแบบมาให้ทำงานกับ output ของ CICS
  Translator ได้อย่างถูกต้อง
- คอลัมน์ Fixed-Format (8-72) ยังคงบังคับใช้กับ `EXEC CICS` เหมือน `EXEC SQL` ทุกประการ เพราะทั้งคู่
  เป็น "ข้อความในซอร์สไฟล์ COBOL" ที่ต้องอยู่ในกฎเดียวกัน

### แบบฝึกหัดที่ 603.1

**โจทย์**: จงเปรียบเทียบข้อความ error ที่ได้จากการคอมไพล์ `EXEC SQL` (Part 058) กับ `EXEC CICS`
(Part นี้) ด้วย `cobc` ธรรมดา และอธิบายว่าทำไมจึงต่างกัน

**เฉลย**: `EXEC SQL INCLUDE SQLCA END-EXEC` ให้ error `SQLCA: No such file or directory` เพราะ
`cobc` พยายามตีความคำว่า `INCLUDE` ว่าเป็นการ `COPY` ไฟล์ชื่อ `SQLCA` (เข้าใจผิดแต่ยังพยายามประมวลผล
ต่อ) ส่วน `EXEC CICS RETURN END-EXEC` ให้ error `unknown statement 'EXEC'` เพราะ `cobc` ไม่รู้จัก
คำสงวน `EXEC` ในบริบทของ Verb/Statement เลย (มองว่าเป็นคำที่ไม่มีความหมายในไวยากรณ์ COBOL มาตรฐาน)
ทั้งสองกรณีสะท้อนความจริงเดียวกันคือ **GnuCOBOL ธรรมดาไม่รองรับทั้ง EXEC SQL และ EXEC CICS** เพียง
แต่ error message ที่ปรากฏต่างกันไปตามรายละเอียดของ parser ภายใน

---

## ขั้นตอนที่ 604: EIB (Execute Interface Block) — ข้อมูลบริบทของ Transaction

### แนวคิดของ EIB

**EIB (Execute Interface Block)** คือโครงสร้างข้อมูลที่ CICS **ส่งให้โปรแกรมโดยอัตโนมัติทุกครั้งที่
Transaction เริ่มทำงาน** (ไม่ต้องประกาศเองใน `WORKING-STORAGE` เหมือน SQLCA) บรรจุข้อมูลบริบทสำคัญ
เกี่ยวกับ Transaction นั้น เช่น มันคือ Transaction อะไร เริ่มทำงานเมื่อไร ผู้ใช้กดปุ่มอะไรล่าสุด

### ฟิลด์สำคัญของ EIB (แบบย่อ)

```cobol
      *> REFERENCE SYNTAX - standard DFHEIBLK structure (abbreviated)
      *> supplied automatically by CICS - no need to declare it yourself
       01  DFHEIBLK.
           05  EIBTIME       PIC S9(7)   COMP-3.
           05  EIBDATE       PIC S9(7)   COMP-3.
           05  EIBTRNID      PIC X(4).
           05  EIBTASKN      PIC S9(7)   COMP-3.
           05  EIBTRMID      PIC X(4).
           05  EIBCPOSN      PIC S9(4)   COMP.
           05  EIBAID        PIC X(1).
           05  EIBRESP       PIC S9(8)   COMP.
           05  EIBRESP2      PIC S9(8)   COMP.
```

| ฟิลด์ | ความหมาย |
|---|---|
| `EIBTRNID` | Transaction ID ที่กำลังทำงานอยู่ (เช่น `ATMW`) |
| `EIBTASKN` | หมายเลข Task ที่ CICS กำหนดให้ Transaction นี้ (unique ต่อการรันแต่ละครั้ง ไม่ซ้ำ) |
| `EIBTRMID` | รหัส Terminal ที่ผู้ใช้กำลังใช้งาน |
| `EIBAID` | Attention Identifier — บอกว่าผู้ใช้กดปุ่มอะไรล่าสุด (ENTER, PF1-PF24 ฯลฯ — สอนละเอียดใน Part 062 ขั้นตอนที่ 618) |
| `EIBRESP` / `EIBRESP2` | รหัสผลลัพธ์ล่าสุดจากคำสั่ง `EXEC CICS` ที่เพิ่งเรียก (เทียบเท่า `SQLCODE` ของ DB2) |
| `EIBDATE` / `EIBTIME` | วันที่/เวลาที่ Task เริ่มทำงาน ในรูปแบบตัวเลขภายในของ CICS |

### ตัวอย่างการใช้งาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               ASKTIME
           END-EXEC.

           DISPLAY "TRANSACTION ID : " EIBTRNID.
           DISPLAY "TASK NUMBER    : " EIBTASKN.
           DISPLAY "TERMINAL ID    : " EIBTRMID.
```

### ข้อควรระวัง

- ห้าม `MOVE` ค่าเข้าไปแก้ไขฟิลด์ใน `DFHEIBLK` เอง — เป็นข้อมูลที่ **CICS เขียนให้เท่านั้น**
  (Read-only จากมุมมองโปรแกรมเมอร์) การพยายามแก้ไขจะไม่มีผลต่อพฤติกรรมจริงของ CICS และอาจทำให้
  โปรแกรมสับสนกับสถานะจริง
- `EIBRESP`/`EIBRESP2` มีความสำคัญมากในการตรวจสอบผลลัพธ์ของ `EXEC CICS` แต่ละคำสั่ง (คล้าย `SQLCODE`)
  — ค่า `EIBRESP = 0` หมายถึงสำเร็จ ส่วนค่าอื่นบ่งบอก Condition ต่าง ๆ ที่จะอธิบายเพิ่มเติมในขั้นตอน
  ถัดไป

### แบบฝึกหัดที่ 604.1

**โจทย์**: จงเปรียบเทียบ EIB ของ CICS กับ SQLCA ของ DB2 (Part 058 ขั้นตอนที่ 573) ว่ามีจุดร่วมและ
จุดต่างอย่างไร

**เฉลย**: **จุดร่วม**: ทั้งสองเป็นโครงสร้างข้อมูลที่ระบบ (CICS/DB2) เตรียมและอัปเดตให้โปรแกรม
**โดยอัตโนมัติ**หลังทำงาน ไม่ต้องเขียนโค้ดอัปเดตเอง และทั้งสองมีฟิลด์สำหรับตรวจสอบผลลัพธ์ล่าสุด
(`EIBRESP`/`EIBRESP2` เทียบกับ `SQLCODE`) **จุดต่าง**: SQLCA ต้องประกาศเองผ่าน `EXEC SQL INCLUDE
SQLCA END-EXEC` ในขณะที่ EIB (`DFHEIBLK`) CICS ส่งให้โดยอัตโนมัติตั้งแต่ Transaction เริ่มทำงานโดย
ไม่ต้องประกาศอะไรเลย นอกจากนี้ EIB ยังมีข้อมูลบริบทของ Transaction เอง (เช่น Trans-ID, Terminal ID,
ปุ่มที่กด) ซึ่งเป็นข้อมูลคนละประเภทจาก SQLCA ที่เน้นสถานะของคำสั่ง SQL ล่าสุดเท่านั้น

---

## ขั้นตอนที่ 605: RETURN Command — การจบ Task และการเตรียมพร้อมสำหรับ Pseudo-conversational

### ไวยากรณ์พื้นฐาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               RETURN
           END-EXEC.
```

`RETURN` คือคำสั่งที่บอก CICS ว่า **"Task นี้ทำงานเสร็จแล้ว คืนการควบคุมกลับไปที่ CICS"**
เทียบเคียงได้กับ `STOP RUN` ของโปรแกรม Batch — เมื่อ CICS ได้รับ `RETURN` จะปลด Task ปัจจุบันออกจาก
หน่วยความจำ (คืนทรัพยากรทั้งหมดที่ Task นี้ใช้ไป) และพร้อมรับ Transaction ใหม่จากผู้ใช้คนนี้หรือคน
อื่นต่อไป

### RETURN TRANSID — preview ของ Pseudo-conversational Programming

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               RETURN TRANSID('ATMW') COMMAREA(WS-STATE-DATA)
           END-EXEC.
```

`RETURN TRANSID(...)` เป็นรูปแบบพิเศษที่บอก CICS ว่า **"เมื่อผู้ใช้คนนี้กดปุ่มอะไรก็ตามครั้งถัดไป
(เช่น ENTER) ให้เรียก Transaction 'ATMW' นี้อีกครั้งโดยอัตโนมัติ"** พร้อมส่งข้อมูล `COMMAREA` (จะ
สอนในขั้นตอนที่ 607) ไปด้วยเพื่อให้โปรแกรม "จำ" ได้ว่าตัวเองทำอะไรไปแล้วก่อนหน้านี้

นี่คือหัวใจของแนวคิดที่เรียกว่า **Pseudo-conversational Programming** ซึ่งเป็นรูปแบบมาตรฐานของ
โปรแกรม CICS ที่ดี (จะสอนแบบละเอียดเต็มรูปแบบใน Part 064) — เหตุผลสำคัญคือ **โปรแกรม CICS ที่ "ค้าง
รอ" input จากผู้ใช้จริง ๆ (Conversational) จะจับหน่วยความจำและทรัพยากรไว้ตลอดเวลาที่ผู้ใช้กำลังคิด**
(อาจนานหลายวินาทีถึงหลายนาที) ทำให้ระบบรองรับผู้ใช้พร้อมกันได้น้อยลงมาก ในขณะที่ Pseudo-conversational
จะ **"ปล่อย" ทรัพยากรกลับคืนทุกครั้งที่รอ input จากผู้ใช้ แล้วค่อย "กลับมาทำงานใหม่" เมื่อผู้ใช้ตอบ
สนอง** ทำให้ประหยัดทรัพยากรกว่ามาก

### ข้อควรระวัง

- `RETURN` แบบธรรมดา (ไม่มี `TRANSID`) ที่เรียกจาก Transaction เริ่มต้น (ไม่ใช่ระหว่างสายการเรียก
  `LINK`/`XCTL` ที่จะสอนในขั้นตอนที่ 608) หมายถึง **"จบ Transaction นี้ไปเลย"** — ผู้ใช้จะต้องพิมพ์
  Transaction ID ใหม่เองถ้าต้องการทำธุรกรรมต่อ
- `EXEC CICS RETURN` **ต้องเป็นคำสั่งสุดท้าย**ที่ Path การทำงานของโปรแกรมไปถึงเสมอ (คล้าย `STOP RUN`
  ของ Batch) ไม่มีโค้ดใดหลังจากนี้ที่จะทำงานต่อได้ ยกเว้นในกรณีของโปรแกรมที่ `LINK` เรียกมา (ขั้นตอน
  ที่ 608) ซึ่ง `RETURN` จะหมายถึงคืนการควบคุมกลับไปที่โปรแกรมผู้เรียกแทน

### แบบฝึกหัดที่ 605.1

**โจทย์**: จงอธิบายว่าทำไม Pseudo-conversational Programming (ใช้ `RETURN TRANSID`) จึงประหยัด
ทรัพยากรระบบมากกว่าการเขียนโปรแกรมแบบ Conversational (ที่ค้างรอ input ผู้ใช้ตรง ๆ)

**เฉลย**: เพราะโปรแกรมแบบ Conversational จะ**จับ Task และทรัพยากรที่เกี่ยวข้อง (memory, locks) ไว้
ตลอดเวลา**ที่รอผู้ใช้ตอบสนอง ซึ่งอาจนานหลายวินาทีถึงนาทีต่อผู้ใช้หนึ่งคน หากมีผู้ใช้หลายพันคนทำแบบนี้
พร้อมกัน ทรัพยากรของ CICS Region จะหมดอย่างรวดเร็ว ในขณะที่ Pseudo-conversational จะ `RETURN
TRANSID` **คืน Task และทรัพยากรทั้งหมดกลับให้ CICS ทันที**ทุกครั้งที่ต้องรอผู้ใช้ แล้วเมื่อผู้ใช้
ตอบสนอง (กดปุ่ม) ค่อยเริ่ม Task ใหม่พร้อมข้อมูลที่ส่งต่อผ่าน COMMAREA ทำให้ CICS Region เดียวรองรับ
ผู้ใช้จำนวนมากพร้อมกันได้อย่างมีประสิทธิภาพมากกว่ามาก

---

## ขั้นตอนที่ 606: ABEND และการจัดการข้อผิดพลาดใน CICS

### EXEC CICS ABEND — การยุติ Task โดยตั้งใจ

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               ABEND ABCODE('BANK')
           END-EXEC.
```

`ABEND` คือคำสั่งที่ยุติ Task ปัจจุบันทันทีเมื่อเกิดปัญหาที่ร้ายแรงจนไม่สามารถทำงานต่อได้อย่างปลอดภัย
`ABCODE` คือรหัส 4 ตัวอักษรที่โปรแกรมเมอร์กำหนดเองเพื่อช่วยระบุสาเหตุ (ปรากฏใน CICS log และหน้าจอ
ผู้ใช้เป็นข้อความเช่น "TRANSACTION ATMW ABEND BANK")

### RESP Option — วิธีมาตรฐานสมัยใหม่ในการตรวจสอบผลลัพธ์

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       01  WS-RESP-CODE        PIC S9(8) COMP.

           EXEC CICS
               READ FILE('CUSTFILE')
                    INTO(CUSTOMER-RECORD)
                    RIDFLD(WS-CUST-ID)
                    RESP(WS-RESP-CODE)
           END-EXEC.

           EVALUATE WS-RESP-CODE
               WHEN DFHRESP(NORMAL)
                   DISPLAY "READ SUCCESSFUL."
               WHEN DFHRESP(NOTFND)
                   DISPLAY "CUSTOMER NOT FOUND."
               WHEN OTHER
                   DISPLAY "UNEXPECTED ERROR: " WS-RESP-CODE
           END-EVALUATE.
```

`RESP(...)` เป็น option ที่เติมท้ายคำสั่ง `EXEC CICS` เกือบทุกตัว เพื่อรับรหัสผลลัพธ์เข้าตัวแปรที่
กำหนดเอง แทนที่จะพึ่งพา `EIBRESP` ตรง ๆ (แม้ `EIBRESP` จะถูกอัปเดตพร้อมกันเสมอก็ตาม) `DFHRESP(...)`
เป็น Compiler Directive พิเศษของ CICS Translator ที่แปลงชื่อ Condition (เช่น `NORMAL`, `NOTFND`)
ให้กลายเป็นค่าตัวเลขจริงโดยอัตโนมัติตอน Translate ทำให้โค้ดอ่านง่ายกว่าการจำตัวเลขรหัสเอง

### HANDLE CONDITION — รูปแบบเก่าที่ควรรู้จักแต่ไม่แนะนำให้ใช้ใหม่

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
      *> Legacy style - still found in many older programs
           EXEC CICS
               HANDLE CONDITION
                   NOTFND(NOT-FOUND-PARA)
                   ERROR(GENERAL-ERROR-PARA)
           END-EXEC.

           EXEC CICS
               READ FILE('CUSTFILE') INTO(CUSTOMER-RECORD)
                    RIDFLD(WS-CUST-ID)
           END-EXEC.
      *> If NOTFND occurs, the program will "jump" to NOT-FOUND-PARA automatically
      *> the same idea as WHENEVER for Embedded SQL in Part 059, step 586
```

`HANDLE CONDITION` ทำงานคล้าย `WHENEVER` ของ Embedded SQL ทุกประการ (กระโดดแบบซ่อนเร้นด้วย
`GO TO` โดยปริยาย) และมีข้อเสียเดียวกัน: ขัดกับ Structured Programming ทำให้ติดตามการไหลของโปรแกรม
ยาก จึงเป็นเหตุผลที่มาตรฐานสมัยใหม่แนะนำ `RESP` option แทนเกือบทั้งหมด แต่โปรแกรมเมอร์ CICS ยังคง
ต้อง**อ่านโค้ดเก่าที่ใช้ `HANDLE CONDITION` ให้เป็น** เพราะระบบ Legacy จำนวนมหาศาลยังคงใช้รูปแบบนี้
อยู่

### ข้อควรระวัง

- `ABEND` ควรใช้เฉพาะกรณีที่ปัญหาร้ายแรงจริง ๆ จนไม่สามารถรายงานผ่านหน้าจอผู้ใช้ตามปกติได้ ไม่ควรใช้
  แทนการตรวจสอบเงื่อนไขทางธุรกิจทั่วไป (เช่น "ไม่พบลูกค้า" ไม่ควรทำให้ Task ABEND แต่ควรแสดงข้อความ
  ที่เหมาะสมผ่านหน้าจอแทน)
- `RESP` option ที่ดีต้องตรวจสอบ**ทุกครั้ง**หลัง `EXEC CICS` ที่มีโอกาสล้มเหลว เช่นเดียวกับวินัยที่
  สอนเรื่อง `SQLCODE` ใน Part 058-059 — การลืมตรวจสอบเป็นสาเหตุอันดับต้น ๆ ของบั๊กที่พบในโปรแกรม
  CICS จริง

### แบบฝึกหัดที่ 606.1

**โจทย์**: จงอธิบายว่าทำไมมาตรฐานการเขียนโค้ด CICS สมัยใหม่จึงนิยมใช้ `RESP` option แทน `HANDLE
CONDITION` ทั้งที่ `HANDLE CONDITION` เขียนสั้นกว่าในบางกรณี

**เฉลย**: เพราะ `HANDLE CONDITION` ใช้กลไก `GO TO` แบบซ่อนเร้นที่ทำงานโดยอัตโนมัติหลัง `EXEC CICS`
ทุกตัวที่ตามมาในซอร์สไฟล์ ทำให้ผู้อ่านโค้ดไม่สามารถเห็น "จุดกระโดด" ได้ชัดเจนจากตำแหน่งที่เขียนโค้ด
ต้องย้อนกลับไปดู `HANDLE CONDITION` ก่อนหน้าเสมอเพื่อเข้าใจว่าคำสั่งนี้จะไปจบที่ไหนถ้าเกิดปัญหา
ขัดกับหลัก Structured Programming ที่ COBOL-85 พยายามส่งเสริม (ทบทวนจาก Part 001) ในขณะที่ `RESP`
option ทำให้การตรวจสอบและตัดสินใจเกิดขึ้น**ในจุดเดียวกับที่เรียกคำสั่ง**อย่างชัดเจน อ่านและติดตาม
การไหลของโปรแกรมได้ง่ายกว่ามาก โดยเฉพาะในโปรแกรมขนาดใหญ่ที่มี `EXEC CICS` จำนวนมาก

---

## ขั้นตอนที่ 607: COMMAREA — พื้นที่สื่อสารข้อมูลระหว่างการเรียกซ้ำ

### ปัญหา: CICS ไม่มีตัวแปร "Global" ที่คงอยู่ข้ามการเรียก Task

ต่างจากโปรแกรม Batch ที่ตัวแปรใน `WORKING-STORAGE SECTION` คงอยู่ตลอดการรันโปรแกรม (Part 005)
โปรแกรม CICS แบบ Pseudo-conversational (ขั้นตอนที่ 605) จะ**เริ่ม Task ใหม่ทุกครั้ง**ที่ผู้ใช้กดปุ่ม
ทำให้ `WORKING-STORAGE SECTION` **ถูกสร้างขึ้นใหม่หมด**ทุกครั้งที่ Task เริ่มทำงาน (ค่าทุกตัวรีเซ็ต
กลับเป็นค่าเริ่มต้นตาม `VALUE` clause) — คำถามคือ แล้วโปรแกรมจะ "จำ" ข้อมูลจากรอบก่อนหน้าได้อย่างไร

### COMMAREA คือคำตอบ

**COMMAREA (Communication Area)** คือพื้นที่หน่วยความจำที่**คงอยู่ข้ามการเรียก Task** ส่งต่อผ่าน
`RETURN TRANSID(...) COMMAREA(...)` (ขั้นตอนที่ 605) แล้ว Task ใหม่ที่เริ่มทำงานจะรับข้อมูลนี้กลับมา
ผ่าน `LINKAGE SECTION` (ไม่ใช่ `WORKING-STORAGE SECTION`)

### ไวยากรณ์และรูปแบบการใช้งาน

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STATE-DATA.
           05  WS-STEP-NUMBER       PIC 9(2).
           05  WS-ACCOUNT-NUMBER    PIC 9(10).

       LINKAGE SECTION.
       01  DFHCOMMAREA.
           05  LK-STEP-NUMBER       PIC 9(2).
           05  LK-ACCOUNT-NUMBER    PIC 9(10).

       PROCEDURE DIVISION.
       MAIN-PARA.
           EXEC CICS
               HANDLE CONDITION
                   NOTFND(FIRST-TIME-PARA)
           END-EXEC.

      *> Check whether this is the first call (no COMMAREA sent at all)
      *> or a repeat call (COMMAREA present from a previous round)
           IF EIBCALEN = 0
               PERFORM FIRST-TIME-PARA
           ELSE
               MOVE DFHCOMMAREA TO WS-STATE-DATA
               PERFORM CONTINUE-PROCESSING-PARA
           END-IF.

           EXEC CICS
               RETURN TRANSID('ATMW') COMMAREA(WS-STATE-DATA)
           END-EXEC.

       FIRST-TIME-PARA.
           MOVE 1 TO WS-STEP-NUMBER.
           DISPLAY "SHOWING FIRST SCREEN.".

       CONTINUE-PROCESSING-PARA.
           DISPLAY "CONTINUING AT STEP " WS-STEP-NUMBER.
```

### EIBCALEN — ตรวจสอบว่ามี COMMAREA ส่งมาหรือไม่

`EIBCALEN` (ฟิลด์หนึ่งใน EIB จากขั้นตอนที่ 604) บอก**ความยาว**ของ COMMAREA ที่ส่งเข้ามาในการเรียก
ครั้งนี้ ถ้าเป็น `0` หมายถึง**ไม่มี COMMAREA ส่งมาเลย** ซึ่งมักหมายความว่านี่คือการเรียก Transaction
ครั้งแรก (ผู้ใช้เพิ่งพิมพ์ Trans-ID เอง ไม่ใช่ CICS เรียกต่อจาก `RETURN TRANSID`)

### ข้อควรระวัง

- ขนาดของ COMMAREA มีจำกัด (แม้ CICS สมัยใหม่จะรองรับขนาดใหญ่ขึ้นมากแล้ว แต่ยังควรออกแบบให้กระชับ
  เพราะข้อมูลนี้ต้องถูกคัดลอกไปมาทุกครั้งที่ Task ทำงาน)
- **ห้ามลืมตรวจสอบ `EIBCALEN`** ก่อนใช้ค่าใน `DFHCOMMAREA` เสมอ เพราะถ้าเป็นการเรียกครั้งแรก
  (`EIBCALEN = 0`) `DFHCOMMAREA` จะไม่มีข้อมูลที่มีความหมายใด ๆ เลย การพยายามอ่านค่าจากมันโดยไม่
  ตรวจสอบก่อนอาจทำให้เกิดพฤติกรรมที่ไม่คาดคิด

### แบบฝึกหัดที่ 607.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม CICS แบบ Pseudo-conversational จึงต้องใช้ COMMAREA แทนที่จะพึ่งพา
ตัวแปรใน `WORKING-STORAGE SECTION` เหมือนโปรแกรม Batch

**เฉลย**: เพราะโปรแกรม Pseudo-conversational จะ**เริ่ม Task ใหม่ทั้งหมดทุกครั้ง**ที่ผู้ใช้กดปุ่ม
(ตามที่อธิบายในขั้นตอนที่ 605) ซึ่งหมายความว่า `WORKING-STORAGE SECTION` จะถูกจัดสรรใหม่และรีเซ็ต
กลับเป็นค่าเริ่มต้นทุกครั้ง (ค่าที่เก็บไว้จากรอบก่อนหน้าจะหายไปหมด) ต่างจากโปรแกรม Batch ที่รันต่อ
เนื่องตั้งแต่ต้นจนจบในหน่วยความจำเดียวกันตลอด COMMAREA จึงเป็นกลไกเดียวที่ทำให้ข้อมูลสำคัญ (เช่น
"ตอนนี้อยู่ขั้นตอนไหนของธุรกรรม") ถูกส่งต่อข้าม Task ที่แยกจากกันโดยสิ้นเชิงในทางเทคนิคได้

---

## ขั้นตอนที่ 608: XCTL และ LINK — การส่งต่อการควบคุมระหว่างโปรแกรม CICS

### XCTL — ส่งต่อการควบคุมแบบ "ไปแล้วไม่กลับ"

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               XCTL PROGRAM('ATMMENU') COMMAREA(WS-STATE-DATA)
           END-EXEC.
```

`XCTL` (Transfer Control) ส่งการควบคุมไปยังโปรแกรม CICS อีกตัวหนึ่ง **โดยไม่กลับมาที่โปรแกรมเดิม
อีกเลย** (โปรแกรมปัจจุบันถูกปลดออกจากหน่วยความจำทันที) เปรียบเทียบได้กับ `GO TO` ระดับโปรแกรมทั้ง
โปรแกรม เหมาะกับกรณีที่ต้องการเปลี่ยนไปทำงานอื่นโดยสิ้นเชิง เช่น จากหน้าจอ "เมนูหลัก" ไปหน้าจอ
"ถอนเงิน" โดยไม่จำเป็นต้องย้อนกลับมาที่เมนูหลักโดยอัตโนมัติ

### LINK — เรียกโปรแกรมย่อยแบบ "ไปแล้วกลับมา"

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
           EXEC CICS
               LINK PROGRAM('VALIDPIN') COMMAREA(WS-PIN-DATA)
           END-EXEC.
      *> Control returns to the line right after this once VALIDPIN
      *> calls EXEC CICS RETURN (not RETURN TRANSID)
           DISPLAY "BACK FROM VALIDPIN. RESULT: " WS-PIN-DATA.
```

`LINK` เปรียบเทียบได้กับ `CALL` ของ COBOL ปกติ (Part 031) — เรียกโปรแกรมอื่นให้ทำงาน แล้ว**กลับมา
ทำงานต่อที่จุดเดิม**เมื่อโปรแกรมที่ถูกเรียกจบด้วย `EXEC CICS RETURN` (ไม่ใช่ `RETURN TRANSID` ซึ่ง
สงวนไว้สำหรับ Transaction ระดับบนสุดเท่านั้น)

### ตารางเปรียบเทียบ

| คำสั่ง | เปรียบเทียบกับ COBOL ปกติ | กลับมาที่โปรแกรมเดิมไหม | ใช้เมื่อ |
|---|---|---|---|
| `XCTL` | คล้าย `GO TO` ข้ามโปรแกรม | ไม่กลับ | เปลี่ยนหน้าจอ/ฟังก์ชันการทำงานทั้งหมด |
| `LINK` | คล้าย `CALL` | กลับมาทำงานต่อ | เรียกใช้ตรรกะย่อยที่ใช้ซ้ำได้ (เช่น ตรวจสอบ PIN) |

### ข้อควรระวัง

- โปรแกรมที่ถูก `LINK` เรียกมา เมื่อจบต้องใช้ `EXEC CICS RETURN` แบบธรรมดา (ไม่มี `TRANSID`)
  เพราะการใช้ `RETURN TRANSID` ในบริบทนี้จะเกิด error ทันที (เนื่องจากขัดกับความหมาย — `LINK`
  คาดหวังให้กลับมาที่โปรแกรมผู้เรียก ไม่ใช่เริ่ม Transaction ใหม่)
- การใช้ `XCTL`/`LINK` มากเกินไปโดยไม่ระวังอาจทำให้เกิด "Deeply Nested Call Chain" ที่ตามรอยยาก
  เช่นเดียวกับปัญหา Deeply Nested CALL ของโปรแกรม Batch ทั่วไป (ทบทวนแนวคิด Modularization ที่ดี
  จาก Part 014 และ Part 031)

### แบบฝึกหัดที่ 608.1

**โจทย์**: หากต้องการเขียนโปรแกรม "ตรวจสอบ PIN" ที่ใช้ซ้ำได้จากหลายหน้าจอ (เมนูถอนเงิน, เมนูโอนเงิน,
เมนูเช็คยอด) ควรใช้ `XCTL` หรือ `LINK` เรียกโปรแกรมนี้ เพราะเหตุใด

**เฉลย**: ควรใช้ **`LINK`** เพราะโปรแกรมตรวจสอบ PIN เป็นตรรกะย่อยที่ต้องการ**ผลลัพธ์กลับมา** (เช่น
PIN ถูกต้องหรือไม่) แล้วให้โปรแกรมผู้เรียก (เมนูถอนเงิน/โอนเงิน/เช็คยอด) **ทำงานต่อ**ตามผลลัพธ์นั้น
ต่างจาก `XCTL` ที่เหมาะกับการ "เปลี่ยนหน้าจอไปเลยโดยไม่กลับ" ซึ่งไม่ตรงกับความต้องการในกรณีนี้ที่
โปรแกรมเมนูต่าง ๆ ยังต้องทำงานต่อหลังตรวจสอบ PIN เสร็จ (เช่น ถ้า PIN ถูกต้องจึงดำเนินการถอนเงินต่อ)

---

## ขั้นตอนที่ 609: ภาพรวมคำสั่ง CICS พื้นฐานอื่น ๆ

### ตารางสรุปคำสั่ง CICS พื้นฐานที่พบบ่อยที่สุด

| คำสั่ง | หน้าที่ |
|---|---|
| `RETURN` | จบ Task / ส่งกลับไปที่ CICS (ขั้นตอนที่ 605) |
| `ABEND` | ยุติ Task โดยตั้งใจเมื่อเกิดปัญหาร้ายแรง (ขั้นตอนที่ 606) |
| `XCTL` | ส่งต่อการควบคุมไปโปรแกรมอื่นแบบไม่กลับ (ขั้นตอนที่ 608) |
| `LINK` | เรียกโปรแกรมอื่นแบบกลับมาทำงานต่อ (ขั้นตอนที่ 608) |
| `ASKTIME` | ขอวันที่/เวลาปัจจุบันจาก CICS เข้า EIB |
| `SEND MAP` / `RECEIVE MAP` | ส่ง/รับข้อมูลหน้าจอ (สอนเต็มรูปแบบใน Part 062) |
| `READ` / `WRITE` / `REWRITE` / `DELETE` | เข้าถึงไฟล์ VSAM ผ่าน CICS File Control (คล้ายแนวคิด Part 025 แต่ใช้ syntax ของ CICS) |
| `WRITEQ TS` / `READQ TS` | เขียน/อ่านข้อมูลชั่วคราวผ่าน Temporary Storage Queue |
| `WRITEQ TD` / `READQ TD` | เขียน/อ่านผ่าน Transient Data Queue (มักใช้ทำคิวงานหรือ log) |
| `START` | เริ่ม Transaction อื่นแบบไม่ประสาน (Asynchronous) |
| `SYNCPOINT` | สั่ง Commit ข้อมูลที่เปลี่ยนแปลงไปแล้วในหน่วยงานนี้ (คล้าย `EXEC SQL COMMIT`) |

### รูปแบบไวยากรณ์ร่วมกันของ EXEC CICS

สังเกตว่าทุกคำสั่งของ CICS มีรูปแบบไวยากรณ์ร่วมกันคือใช้ **keyword parameter** (ไม่ใช่ตำแหน่งของ
parameter แบบ positional เหมือน `CALL ... USING`) เช่น `FILE('CUSTFILE')`, `INTO(...)`,
`RIDFLD(...)`, `RESP(...)` ทำให้อ่านโค้ดเข้าใจง่ายแม้จะไม่ได้จำลำดับพารามิเตอร์แม่นยำนัก และสามารถ
ใส่ option เพิ่มเติมได้อย่างยืดหยุ่นโดยไม่กระทบลำดับของ option อื่น

### ตัวอย่าง ASKTIME + FORMATTIME

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       01  WS-DATE-TEXT         PIC X(10).

           EXEC CICS
               ASKTIME
           END-EXEC.

           EXEC CICS
               FORMATTIME ABSTIME(EIBTIME)
                          YYYYMMDD(WS-DATE-TEXT)
           END-EXEC.

           DISPLAY "CURRENT DATE: " WS-DATE-TEXT.
```

### ข้อควรระวัง

- คำสั่ง File Control ของ CICS (`READ`/`WRITE`/`REWRITE`/`DELETE`) เข้าถึงไฟล์ VSAM ที่**ลงทะเบียน
  ไว้แล้วใน CICS FCT (File Control Table)** ผ่านชื่อ Logical File (เช่น `'CUSTFILE'`) เท่านั้น —
  ไม่ใช้ `SELECT ... ASSIGN TO` แบบ `FILE-CONTROL` ของโปรแกรม Batch (Part 023) เพราะ CICS จัดการ
  การเปิด/ปิดไฟล์ทางกายภาพให้เองทั้งหมด โปรแกรมเมอร์ระบุแค่ "ชื่อ Logical File" เท่านั้น
- Temporary Storage Queue (`WRITEQ TS`/`READQ TS`) กับ Transient Data Queue (`WRITEQ TD`/`READQ TD`)
  ดูคล้ายกันแต่มีพฤติกรรมต่างกันมาก: TS Queue อ่านได้หลายครั้งไม่จำกัด (เหมือนไฟล์ที่เก็บชั่วคราว)
  ส่วน TD Queue มีพฤติกรรมแบบคิว FIFO ที่อ่านแล้วหายไป (เหมือนคิวงานจริง) — เลือกผิดประเภทจะทำให้
  ตรรกะของระบบผิดพลาดทันที

### แบบฝึกหัดที่ 609.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง Temporary Storage Queue (TS) กับ Transient Data Queue (TD)
ของ CICS

**เฉลย**: **TS Queue** ทำงานคล้ายพื้นที่เก็บข้อมูลชั่วคราวที่เข้าถึงได้แบบสุ่ม (random access) ผ่าน
หมายเลข item — อ่านซ้ำกี่ครั้งก็ได้โดยข้อมูลไม่หายไป เหมาะกับการเก็บสถานะชั่วคราวระหว่างหน้าจอ (เช่น
เก็บรายการที่ผู้ใช้เลือกไว้ระหว่างเลื่อนดูหลายหน้าจอ) **TD Queue** ทำงานแบบคิว FIFO (First-In-First-
Out) ที่**อ่านแล้วข้อมูลจะหายไปจากคิวทันที** (คล้ายการดึงงานออกจากคิวงานจริง) เหมาะกับสถานการณ์ที่
ต้องการส่งข้อมูลไปประมวลผลต่อแบบครั้งเดียว เช่น การส่ง log ไปให้ระบบ Batch อ่านไปประมวลผลภายหลัง หรือ
กระตุ้น (trigger) ให้ Transaction อื่นเริ่มทำงานเมื่อมีข้อมูลใหม่เข้าคิว

---

## ขั้นตอนที่ 610: ตัวอย่างโปรแกรม CICS แบบสมบูรณ์ — เปรียบเทียบกับโปรแกรม Batch เทียบเท่า

### โจทย์ตัวอย่าง

เขียนโปรแกรมง่าย ๆ ที่รับรหัสลูกค้าผ่าน COMMAREA ตรวจสอบว่ารหัสถูกต้องหรือไม่ (สมมติว่าถูกต้องคือ
`10001`) แล้วส่งผลลัพธ์กลับ — เขียนทั้งแบบ CICS (ไวยากรณ์อ้างอิง) และแบบ Batch เทียบเท่า (COBOL
ล้วน ๆ ที่คอมไพล์และรันได้จริง) เพื่อให้เห็นความแตกต่างของโครงสร้างชัดเจน

### เวอร์ชัน CICS (ไวยากรณ์อ้างอิง)

```cobol
      *> REFERENCE SYNTAX ONLY - cannot be compiled or run in this environment
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP610-CICS-VALIDATE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-VALID-CUST-ID     PIC 9(5) VALUE 10001.

       LINKAGE SECTION.
       01  DFHCOMMAREA.
           05  LK-CUST-ID       PIC 9(5).
           05  LK-VALID-FLAG    PIC X.
               88  LK-IS-VALID  VALUE "Y".
               88  LK-IS-INVALID VALUE "N".

       PROCEDURE DIVISION.
       MAIN-PARA.
           IF EIBCALEN = 0
               DISPLAY "ERROR: NO INPUT PROVIDED."
           ELSE
               IF LK-CUST-ID = WS-VALID-CUST-ID
                   SET LK-IS-VALID TO TRUE
               ELSE
                   SET LK-IS-INVALID TO TRUE
               END-IF
           END-IF.

           EXEC CICS
               RETURN
           END-EXEC.
```

### เวอร์ชัน Batch เทียบเท่า (ทดสอบจริง — คอมไพล์และรันได้)

เพื่อเปรียบเทียบโครงสร้าง เราเขียนโปรแกรม Batch ที่ทำตรรกะเดียวกันทุกประการ (ตรวจสอบรหัสลูกค้า)
แต่ใช้กลไก COBOL มาตรฐานที่คอมไพล์และรันได้จริงบน GnuCOBOL:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP610-BATCH-VALIDATE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-VALID-CUST-ID     PIC 9(5) VALUE 10001.
       01  WS-CUST-ID           PIC 9(5).
       01  WS-VALID-FLAG        PIC X.
           88  WS-IS-VALID      VALUE "Y".
           88  WS-IS-INVALID    VALUE "N".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> In a batch program there is no COMMAREA or EIBCALEN -
      *> input simply comes from ACCEPT, a file, or a parameter.
           DISPLAY "ENTER CUSTOMER ID: " WITH NO ADVANCING.
           ACCEPT WS-CUST-ID.

           IF WS-CUST-ID = WS-VALID-CUST-ID
               SET WS-IS-VALID TO TRUE
           ELSE
               SET WS-IS-INVALID TO TRUE
           END-IF.

           IF WS-IS-VALID
               DISPLAY "RESULT: VALID CUSTOMER."
           ELSE
               DISPLAY "RESULT: INVALID CUSTOMER."
           END-IF.
           STOP RUN.
```

คอมไพล์และรันจริง:

```bash
cobc -x -o step610batch step610batch.cob
echo "10001" | ./step610batch
echo "99999" | ./step610batch
```

**ผลลัพธ์จริง (ทดสอบแล้ว):**

```
ENTER CUSTOMER ID: RESULT: VALID CUSTOMER.
ENTER CUSTOMER ID: RESULT: INVALID CUSTOMER.
```

### เปรียบเทียบโครงสร้างทั้งสองเวอร์ชัน

| จุด | เวอร์ชัน CICS | เวอร์ชัน Batch |
|---|---|---|
| รับข้อมูลเข้า | `DFHCOMMAREA` ผ่าน `LINKAGE SECTION` | `ACCEPT` จาก terminal โดยตรง |
| ตรวจสอบว่ามีข้อมูลส่งมาไหม | `IF EIBCALEN = 0` | ไม่จำเป็น (`ACCEPT` รอ input เสมอ) |
| จบการทำงาน | `EXEC CICS RETURN` | `STOP RUN` |
| แสดงผลลัพธ์ | ส่งกลับผ่าน `DFHCOMMAREA` (ให้โปรแกรม/หน้าจออื่นอ่านต่อ) | `DISPLAY` ตรง ๆ ที่ terminal |
| ตรรกะทางธุรกิจหลัก (`IF`, `88-level`) | เหมือนกันทุกประการ | เหมือนกันทุกประการ |

จุดสำคัญที่ควรสังเกตคือ **ตรรกะทางธุรกิจแกนกลาง** (การเปรียบเทียบรหัสลูกค้าด้วย `IF` และ condition
name) **เหมือนกันทุกประการ**ในทั้งสองเวอร์ชัน สิ่งที่ต่างกันคือ**เปลือกรอบนอก**ที่ใช้รับข้อมูลเข้าและ
ส่งผลลัพธ์ออก — นี่คือเหตุผลที่ทักษะ COBOL พื้นฐานที่เรียนมาตลอดหลักสูตรนี้ (IF, EVALUATE, PERFORM,
88-level) มีค่าเท่ากันทั้งในโลก Batch และโลก CICS Online

### ข้อควรระวัง

- อย่าเข้าใจผิดว่าโปรแกรม CICS "ง่ายกว่า" หรือ "ยากกว่า" โปรแกรม Batch โดยรวม — มันเป็นเพียง
  **บริบทการทำงานที่ต่างกัน** (รับ-ส่งข้อมูลผ่าน COMMAREA/EIB แทน ACCEPT/DISPLAY/JCL parameter)
  ตรรกะทางธุรกิจหลักยังคงใช้ความรู้ COBOL แบบเดียวกันทั้งหมด
- ตัวอย่าง CICS ในขั้นตอนนี้ยังไม่ได้แสดงการส่งหน้าจอกลับให้ผู้ใช้เห็นจริง ๆ (ผ่าน `SEND MAP`) ซึ่ง
  เป็นสิ่งที่ระบบ CICS จริงเกือบทั้งหมดต้องทำ — Part 062 จะสอนเรื่องนี้อย่างละเอียด

### แบบฝึกหัดที่ 610.1

**โจทย์**: จงระบุว่าส่วนใดของโค้ดในเวอร์ชัน CICS และเวอร์ชัน Batch ข้างต้นที่ **"เหมือนกัน"** และ
ส่วนใดที่ **"ต่างกัน"** พร้อมอธิบายว่าทำไมส่วนที่เหมือนกันจึงสำคัญต่อการเรียนรู้ CICS

**เฉลย**: **ส่วนที่เหมือนกัน**: การประกาศ `88-level` (`LK-IS-VALID`/`WS-IS-VALID`,
`LK-IS-INVALID`/`WS-IS-INVALID`) และตรรกะ `IF ... = WS-VALID-CUST-ID` ที่ใช้ตัดสินใจ **ส่วนที่ต่าง
กัน**: วิธีรับข้อมูลเข้า (COMMAREA กับ ACCEPT), การตรวจสอบว่ามีข้อมูลส่งมาหรือไม่ (`EIBCALEN`),
วิธีจบการทำงาน (`EXEC CICS RETURN` กับ `STOP RUN`), และวิธีส่งผลลัพธ์กลับ ส่วนที่เหมือนกันสำคัญมาก
เพราะแสดงให้เห็นว่า**ทักษะ COBOL พื้นฐานที่เรียนมาตลอดหลักสูตรนี้ (เฟส 1-3) นำไปใช้ในโลก CICS ได้
โดยตรงทันที** สิ่งที่ต้องเรียนรู้เพิ่มเติมสำหรับ CICS คือ "เปลือกรอบนอก" ที่ใช้สื่อสารกับ CICS
Runtime เท่านั้น ไม่ใช่การเรียนภาษาใหม่ทั้งหมด

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้:

- CICS คืออะไร และความแตกต่างระหว่าง Online Transaction Processing กับ Batch Processing
- สถาปัตยกรรม CICS Region, Terminal, และ Transaction ID
- ไวยากรณ์ `EXEC CICS...END-EXEC` และ CICS Translator (พิสูจน์แล้วว่า GnuCOBOL ธรรมดาคอมไพล์ไม่ผ่าน)
- EIB (Execute Interface Block) และฟิลด์สำคัญ เช่น `EIBTRNID`, `EIBAID`, `EIBRESP`
- `RETURN` และ `RETURN TRANSID` — preview ของแนวคิด Pseudo-conversational Programming
- `ABEND`, `RESP` option, และ `HANDLE CONDITION` — สองแนวทางในการจัดการข้อผิดพลาด
- COMMAREA — กลไกสื่อสารข้อมูลข้าม Task ที่จำเป็นเพราะ WORKING-STORAGE ไม่คงอยู่ข้าม Task
- `XCTL` และ `LINK` — การส่งต่อการควบคุมระหว่างโปรแกรม CICS แบบไม่กลับ/กลับมา
- คำสั่ง CICS พื้นฐานอื่น ๆ ที่พบบ่อย (File Control, TS/TD Queue, SYNCPOINT)
- การเปรียบเทียบโครงสร้างโปรแกรม CICS กับ Batch ที่ทำตรรกะเดียวกัน แสดงให้เห็นว่าทักษะ COBOL
  พื้นฐานยังคงใช้ได้โดยตรงในทั้งสองโลก

Part 062 จะต่อยอดจากนี้ไปสู่การสื่อสารกับผู้ใช้ผ่านหน้าจอจริง ๆ — คำสั่ง `SEND MAP`/`RECEIVE MAP`
และแนวคิด BMS (Basic Mapping Support) ที่ทำให้โปรแกรม CICS แสดงผลและรับข้อมูลจากผู้ใช้ผ่านหน้าจอ
Terminal ได้

**[← กลับไป Part 060: DB2 Stored Procedures ด้วย COBOL](part-060-db2-stored-procedures.md)**
**[ไปยัง Part 062: CICS Commands: SEND, RECEIVE, MAP →](part-062-cics-send-receive-map.md)**
