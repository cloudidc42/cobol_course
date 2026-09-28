# Part 054: TSO/ISPF: การใช้งานเบื้องต้น (ขั้นตอนที่ 531–540)

## คำนำของ Part นี้

Part 052–053 สอนให้เราเขียน JCL เพื่อสั่งรันโปรแกรม COBOL แบบ Batch แต่คำถามที่ตามมาคือ:
โปรแกรมเมอร์ Mainframe เขียนไฟล์ JCL และไฟล์ source code COBOL เหล่านั้น **ด้วยเครื่องมืออะไร**
ในเมื่อ Mainframe ไม่มีหน้าจอกราฟิกแบบ Windows/Mac ที่เราคุ้นเคย? คำตอบคือ **TSO** และ **ISPF**
ซึ่งเป็นสภาพแวดล้อมเชิงโต้ตอบ (interactive) มาตรฐานที่โปรแกรมเมอร์ Mainframe ทั่วโลกใช้ทำงานประจำ
วันมาตั้งแต่ยุค 1970 จนถึงปัจจุบัน (2026) และยังคงเป็นเครื่องมือหลักในหลายองค์กร แม้จะเริ่มมี
Modernization ผ่าน IDE สมัยใหม่บ้างแล้วก็ตาม

Part นี้จะพาคุณทำความรู้จักกับ TSO/ISPF ในเชิงแนวคิดและหน้าตาการใช้งาน เพื่อให้เมื่อคุณต้องทำงาน
กับ Mainframe จริงในอนาคต (หรือสัมภาษณ์งานที่ถามถึงเครื่องมือเหล่านี้) คุณจะไม่รู้สึกแปลกหน้า

> **สำคัญมาก — อ่านก่อนเริ่ม**: เนื้อหาทั้งหมดใน Part นี้เป็น**เนื้อหาเชิงแนวคิด (conceptual)**
> ล้วน ๆ เพราะ TSO และ ISPF เป็นส่วนหนึ่งของระบบปฏิบัติการ z/OS โดยตรง ต้องรันบน Mainframe จริง
> หรือ Mainframe Emulator (เช่น Hercules + MVS/TK4-) เท่านั้น **สภาพแวดล้อม Linux Sandbox ของ
> หลักสูตรนี้ไม่มี TSO/ISPF ติดตั้งอยู่ และไม่สามารถจำลองการทำงานจริงได้แม้แต่น้อย** ภาพหน้าจอ
> ทั้งหมดในเอกสารนี้เป็น **ภาพจำลอง (ASCII mockup) ที่วาดขึ้นเพื่อประกอบความเข้าใจเท่านั้น**
> ไม่ใช่ภาพที่แคปเจอร์จากการรันจริงแต่อย่างใด (ใช้เทคนิคเดียวกับที่ Part 040 ใช้อธิบาย
> `SCREEN SECTION` ด้วยภาพจำลอง ASCII) โค้ด COBOL ที่ปรากฏเป็นตัวอย่างเนื้อหาที่ถูกแก้ไขในภาพ
> จำลองของ ISPF Editor นั้นเป็น COBOL มาตรฐานที่คุณสามารถนำไป **compile และรันได้จริงด้วย
> GnuCOBOL** บนเครื่องของคุณเอง (เพียงแต่ไม่ใช่ผ่านหน้าจอ ISPF)

---

## ขั้นตอนที่ 531: TSO คืออะไร — Time Sharing Option

### แนวคิดของ TSO

**TSO (Time Sharing Option)** คือส่วนประกอบของ z/OS ที่เปิดให้ผู้ใช้หลายคน**เข้าใช้งาน Mainframe
พร้อมกันแบบโต้ตอบ (interactive)** ผ่าน terminal ของตัวเอง ต่างจาก Batch Processing (ที่เรียนใน
Part 052-053) ซึ่งงานถูก submit เข้าคิวแล้วรันโดยไม่มีใครนั่งเฝ้าหน้าจอ TSO ทำให้ผู้ใช้พิมพ์คำสั่ง
แล้วเห็นผลลัพธ์ทันที เหมือนการใช้ shell/terminal ในโลก Linux

### ประวัติโดยย่อ

TSO เปิดตัวในช่วงปลายทศวรรษ 1960 เพื่อแก้ปัญหาว่าในยุคนั้น Mainframe ราคาแพงมากจนต้องใช้ร่วมกัน
หลายคน (time-sharing = "แบ่งเวลาใช้งานร่วมกัน") การมี TSO ทำให้ผู้ใช้แต่ละคนรู้สึกเหมือนได้
ครอบครองเครื่องทั้งเครื่องคนเดียว ทั้งที่จริงแล้วเบื้องหลังมีการสลับ (time-slicing) การประมวลผลระหว่าง
ผู้ใช้หลายร้อยหลายพันคนอย่างรวดเร็วมาก

### คำสั่ง TSO พื้นฐานที่ควรรู้จัก

| คำสั่ง | ความหมาย |
|---|---|
| `LOGON` | เข้าสู่ระบบด้วย TSO userid และ password |
| `LOGOFF` | ออกจากระบบ |
| `SUBMIT` | ส่ง JCL job เข้าคิว JES เพื่อรันแบบ Batch (เชื่อมโยงกับ Part 052-053) |
| `STATUS` | ตรวจสอบสถานะ job ที่ submit ไปแล้ว |
| `CANCEL` | ยกเลิก job ที่กำลังรออยู่ในคิวหรือกำลังรันอยู่ |
| `LISTDS` | แสดงรายละเอียดของ dataset (คล้าย `ls -l` ในโลก Linux) |
| `ALLOCATE` | จองพื้นที่สร้าง dataset ใหม่ |
| `HELP` | แสดงคำอธิบายการใช้งานคำสั่ง TSO |

### ภาพจำลอง: หน้าจอ TSO LOGON

```
+----------------------------------------------------------------+
|  TSO/E LOGON                                                    |
|                                                                  |
|  Enter LOGON parameters below:                RACF LOGON parms: |
|                                                                  |
|  Userid    ===> COBOL01                                         |
|  Password  ===>                                  New Password ==>|
|  Procedure ===> ISPFPROC                                        |
|  Group Ident ===>                                                |
|  Acct Nmbr ===> ACCT123                                          |
|  Size      ===> 4096                                            |
|  Perform   ===>                                                  |
|  Command   ===>                                                  |
|                                                                  |
|  Enter an 'S' before each option desired below:                 |
|  -Nomail    -Nonotice   -Reconnect  -OIDcard                    |
|                                                                  |
|  PF1/PF13 ==> Help    PF3/PF15 ==> Logoff  PA1  ==> Attention   |
+----------------------------------------------------------------+
```

### ข้อควรระวัง

- TSO ต่างจาก UNIX System Services (USS) บน z/OS ซึ่งเป็นเชลล์ UNIX แยกต่างหากที่ทำงานคู่กัน —
  หลายคนสับสนสองสิ่งนี้ แต่ TSO ใช้ชุดคำสั่งของตัวเองที่ไม่ใช่ UNIX shell แม้จะรันอยู่บน z/OS
  เดียวกัน
- TSO userid ไม่ใช่สิ่งเดียวกับ RACF userid เสมอไป (RACF จะสอนแนวคิดใน Part 069) แต่ในทางปฏิบัติ
  องค์กรส่วนใหญ่ใช้ userid เดียวกันสำหรับทั้งสองระบบเพื่อความง่ายในการบริหารจัดการสิทธิ์

### แบบฝึกหัดที่ 531.1

**โจทย์**: จงอธิบายว่าเพราะเหตุใด TSO จึงถูกออกแบบขึ้นมาในยุค 1960s ทั้งที่ Batch Processing มีอยู่
แล้ว

**เฉลย**: Batch Processing เหมาะกับงานที่รู้ล่วงหน้าว่าจะทำอะไร (เช่น รันรายงานประจำคืน) แต่ไม่
เหมาะกับงานที่ต้องการโต้ตอบทันที เช่น การแก้ไขโค้ดทีละบรรทัดแล้วดูผลทันที หรือค้นหาข้อมูลแบบ
เฉพาะหน้า TSO ถูกออกแบบมาเติมเต็มช่องว่างนี้ ทำให้ผู้ใช้แต่ละคนรู้สึกเหมือนมี Mainframe เป็นของ
ตัวเอง ทั้งที่ทรัพยากรจริงถูกแบ่งกันใช้ (time-sharing) ระหว่างผู้ใช้จำนวนมากพร้อมกัน

---

## ขั้นตอนที่ 532: ISPF คืออะไร — Interactive System Productivity Facility

### ความสัมพันธ์ระหว่าง TSO กับ ISPF

**ISPF (Interactive System Productivity Facility)** คือโปรแกรมเมนูแบบเต็มจอ (full-screen panel)
ที่ทำงาน**อยู่บน TSO อีกชั้นหนึ่ง** ถ้า TSO เปรียบเหมือน shell แบบ command-line ของ Linux
ISPF ก็เปรียบเหมือน text-based UI ที่ครอบ shell นั้นไว้อีกที (คล้ายเปรียบเทียบกับโปรแกรมอย่าง
`htop`/`mc` (Midnight Commander) ที่ทำงานอยู่บน shell) ทำให้ผู้ใช้ไม่ต้องจำคำสั่งจำนวนมาก
เพียงเลือกจากเมนูตัวเลข

### Primary Option Menu — หน้าแรกที่ทุกคนเห็นเมื่อเข้า ISPF

```
                      ISPF PRIMARY OPTION MENU
 OPTION ===>

  0  SETTINGS      Terminal and user parameters
  1  VIEW          Display source data or listings
  2  EDIT          Create or change source data
  3  UTILITIES     Perform utility functions
  4  FOREGROUND    Interactive language processing
  5  BATCH         Submit job for language processing
  6  COMMAND       Enter TSO or Workstation commands
  7  DIALOG TEST   Perform dialog testing
  9  IBM PRODUCTS  IBM program development products
  10 SCLM          Software Configuration and Library Manager
  X  EXIT          Terminate ISPF

  Userid   . : COBOL01                Time. . . . : 22:14
  Terminal . : 3278                   Terminal name: IBM-3278-2
  Screen . . : 1                      Language. . : ENGLISH
  Appl ID. . : ISR                    Appl ID . . : PROD

 Enter X to Terminate using log/list defaults
```

### อธิบายจุดสำคัญ

- `OPTION ===>` คือช่องกรอกคำสั่งหลัก (Command Line) ของทุกหน้าจอ ISPF ผู้ใช้พิมพ์ตัวเลขตัวเลือก
  หรือตัวย่อคำสั่งเข้าไปแล้วกด **Enter** เพื่อไปยังหน้าจอถัดไป
- Option ที่ใช้บ่อยที่สุดในงาน COBOL คือ **Option 2 (EDIT)** สำหรับแก้ไข source code และ
  **Option 3 (UTILITIES)** โดยเฉพาะ **Option 3.4** สำหรับดูรายชื่อ dataset (จะสอนในขั้นตอนที่
  533)
- ตัวย่อ **PF Key** (Program Function Key) เช่น `PF3` (มักหมายถึง "ย้อนกลับ/End") และ `PF1`
  (มักหมายถึง "Help") เป็นแป้นลัดมาตรฐานที่ใช้ทั่วทุกหน้าจอ ISPF ทำให้ผู้เชี่ยวชาญทำงานได้เร็วมาก
  โดยไม่ต้องพิมพ์คำสั่งเต็ม

### ทำไม ISPF จึงยังคงอยู่มาถึงปี 2026

แม้หน้าตาจะดูเก่ามาก (เป็น text-mode สีเขียว-ดำแบบคลาสสิกที่หลายคนคุ้นตาจากภาพยนตร์) แต่ ISPF ยัง
คงอยู่เพราะ **ประสิทธิภาพการพิมพ์คำสั่งล้วน ๆ ของโปรแกรมเมอร์ที่ชำนาญนั้นเร็วกว่าการคลิกเมาส์ผ่าน
เมนูกราฟิกมาก** เมื่อทำงานซ้ำ ๆ นับพันครั้งต่อวัน ความเร็วในการพิมพ์คำสั่งสั้น ๆ คุ้มค่ากว่าการ
เรียนรู้ระบบใหม่ทั้งหมด องค์กรจำนวนมากจึงยังคงใช้ ISPF เป็นเครื่องมือหลักคู่ขนานไปกับ Modernization
เครื่องมือสมัยใหม่ (เช่น IBM Developer for z/OS ที่เป็น GUI แบบ Eclipse-based)

### ข้อควรระวัง

- อย่าสับสน ISPF กับ IDE สมัยใหม่ — ISPF ไม่มี syntax highlighting อัตโนมัติ ไม่มี auto-complete
  แบบ IDE ทั่วไป (แม้เวอร์ชันใหม่ ๆ จะเริ่มมีบ้างแล้ว) ผู้ใช้ต้องจำ syntax COBOL/JCL ได้แม่นยำกว่า
  การเขียนบน IDE สมัยใหม่มาก
- หน้าจอ ISPF ในเอกสารนี้เป็นภาพจำลอง — รูปแบบจริงอาจแตกต่างกันเล็กน้อยระหว่างแต่ละองค์กร
  เนื่องจาก ISPF สามารถ customize เมนูและ panel เองได้

### แบบฝึกหัดที่ 532.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมเมอร์ Mainframe ที่ชำนาญจึงมักนิยมใช้ ISPF (ทั้งที่หน้าตาดูเก่า)
มากกว่าเปลี่ยนไปใช้ GUI สมัยใหม่ทั้งหมด

**เฉลย**: เพราะการทำงานผ่านแป้นพิมพ์ล้วน (keyboard-only) โดยไม่ต้องสลับมือไปจับเมาส์ ทำให้ทำงานซ้ำ
ๆ ได้เร็วกว่ามากเมื่อสะสมเวลาตลอดวันทำงาน นอกจากนี้ ISPF ยังเสถียรมาก ใช้ทรัพยากรน้อย และทำงานผ่าน
การเชื่อมต่อ terminal แบบ text ล้วนที่ทนทานต่อเครือข่ายไม่เสถียรได้ดีกว่า GUI ที่ต้องส่งข้อมูลภาพ
จำนวนมาก

---

## ขั้นตอนที่ 533: Option 3.4 — Dataset List Utility

### ความสำคัญของ Option 3.4

**Option 3.4** (หรือเรียกสั้น ๆ ว่า "3.4" หรือ "DSLIST") คือหน้าจอที่โปรแกรมเมอร์ Mainframe ใช้
บ่อยที่สุดเป็นอันดับต้น ๆ เพราะทำหน้าที่เหมือน `ls`/File Explorer: **แสดงรายชื่อ dataset ทั้งหมดที่
ตรงกับรูปแบบ (pattern) ที่ระบุ** พร้อมรายละเอียดขนาด, ประเภท, วันที่แก้ไขล่าสุด

### ภาพจำลอง: การเข้าสู่ Option 3.4

```
                      ISPF PRIMARY OPTION MENU
 OPTION ===> 3.4

  0  SETTINGS      Terminal and user parameters
  1  VIEW          Display source data or listings
  2  EDIT          Create or change source data
  3  UTILITIES     Perform utility functions
```

### ภาพจำลอง: หน้าจอ DSLIST หลังจากกรอก Dataset Name Pattern

```
 DSLIST - Data Sets Matching COBOL01.*                Row 1 of 6
 Command ===>

 Command - Enter "/" to select action              Message      Volume
 -------------------------------------------------------------------------
 COBOL01.CNTL
 COBOL01.CUSTOMER.MASTER
 COBOL01.JCL.LIB
 COBOL01.LOAD.LIBRARY
 COBOL01.SOURCE.COBOL
 COBOL01.TEST.OUTPUT
 ***************************** END OF DATA SET LIST ****************
```

### อธิบายจุดสำคัญ

- ชื่อ dataset บน Mainframe เขียนแบบมีจุด (`.`) คั่นแต่ละ "qualifier" คล้าย domain name (เช่น
  `COBOL01.SOURCE.COBOL`) — แต่ละ qualifier ยาวได้สูงสุด 8 ตัวอักษร และชื่อเต็มยาวไม่เกิน 44
  ตัวอักษร ต่างจากระบบไฟล์ทั่วไปที่ใช้ `/` คั่น directory
- คอลัมน์ซ้ายมือของแต่ละแถว (ที่ในภาพจำลองว่างอยู่) คือช่องให้พิมพ์ **Line Command** เพื่อกระทำ
  การกับ dataset นั้นโดยตรง เช่น `B` (Browse - ดูอย่างเดียว), `E` (Edit - แก้ไข), `D` (Delete
  - ลบ), `R` (Rename - เปลี่ยนชื่อ)
- Pattern ที่ใช้ค้นหาสามารถใช้ `*` เป็น wildcard ได้ เช่น `COBOL01.SOURCE.*` เพื่อกรองเฉพาะ
  dataset ที่มี qualifier ที่สองเป็น `SOURCE`

### ภาพจำลอง: การใช้ Line Command เพื่อ Edit ไฟล์

```
 DSLIST - Data Sets Matching COBOL01.*                Row 1 of 6
 Command ===>

 Command - Enter "/" to select action              Message      Volume
 -------------------------------------------------------------------------
 COBOL01.CNTL
 COBOL01.CUSTOMER.MASTER
 e    COBOL01.SOURCE.COBOL
 COBOL01.TEST.OUTPUT
```

พิมพ์ `e` ที่หน้า dataset ที่ต้องการแล้วกด Enter จะเปิด **ISPF Editor** ให้แก้ไข dataset นั้นทันที
(รายละเอียดของ Editor จะสอนในขั้นตอนที่ 534-535)

### ข้อควรระวัง

- ชื่อ dataset บน z/OS **ไม่สนใจตัวพิมพ์เล็ก-ใหญ่** (case-insensitive) และมักแสดงผลเป็นตัวพิมพ์
  ใหญ่ทั้งหมดเสมอ ต่างจากชื่อไฟล์ใน Linux/Windows ที่อาจสนใจตัวพิมพ์เล็ก-ใหญ่
  (Linux สนใจ, Windows ไม่สนใจ)
- การลบ dataset ผ่าน Line Command `D` **ไม่มีถังขยะ (Recycle Bin)** ให้กู้คืน ต้องระวังให้มากเมื่อ
  ใช้คำสั่งนี้กับข้อมูล Production จริง

### แบบฝึกหัดที่ 533.1

**โจทย์**: จงอธิบายความแตกต่างระหว่างการใช้ Line Command `B` (Browse) กับ `E` (Edit) บนหน้าจอ
DSLIST

**เฉลย**: `B` (Browse) เปิด dataset ในโหมดอ่านอย่างเดียว ป้องกันการแก้ไขโดยไม่ตั้งใจ เหมาะกับการ
ตรวจสอบเนื้อหาอย่างรวดเร็วโดยไม่เสี่ยงทำข้อมูลเสียหาย ส่วน `E` (Edit) เปิดในโหมดที่แก้ไขและบันทึก
การเปลี่ยนแปลงได้ เหมาะกับตอนที่ต้องการแก้ไข source code หรือ JCL จริง ๆ

---

## ขั้นตอนที่ 534: ISPF Editor เบื้องต้น — Line Commands

### แนวคิดของ ISPF Editor

**ISPF Editor** (เข้าถึงผ่าน Option 2 หรือ Line Command `E` จาก DSLIST) คือโปรแกรมแก้ไขข้อความ
เต็มจอที่มีมาตั้งแต่ยุค 1970 มีจุดเด่นเฉพาะตัวคือ **Line Command Area** — พื้นที่แคบ ๆ ทางซ้ายมือ
ของทุกบรรทัด ที่ใช้พิมพ์คำสั่งเพื่อกระทำการกับบรรทัดนั้นโดยตรง แทนที่จะต้องเลือก (select) ข้อความ
ด้วยเมาส์แบบ editor สมัยใหม่

### ภาพจำลอง: หน้าจอ ISPF Editor กำลังแก้ไขโปรแกรม COBOL

```
 EDIT       COBOL01.SOURCE.COBOL(TXNVALID) - 01.15        Columns 00001 00072
 Command ===>                                                 Scroll ===> CSR
 ****** ***************************** Top of Data ******************************
 000100        IDENTIFICATION DIVISION.
 000200        PROGRAM-ID. TXNVALID.
 000300        AUTHOR. COBOL-COURSE.
 000400
 000500        ENVIRONMENT DIVISION.
 000600        INPUT-OUTPUT SECTION.
 000700        FILE-CONTROL.
 000800            SELECT TXN-FILE ASSIGN TO TXNIN
 000900                ORGANIZATION IS SEQUENTIAL.
 001000
 001100        DATA DIVISION.
 001200        FILE SECTION.
 001300        FD  TXN-FILE.
 001400        01  TXN-RECORD           PIC X(80).
 001500
 001600        WORKING-STORAGE SECTION.
 001700        01  WS-ERROR-COUNT       PIC 9(5) VALUE 0.
 001800
 001900        PROCEDURE DIVISION.
 002000        MAIN-PARA.
 002100            DISPLAY "TRANSACTION VALIDATION STARTING".
 002200            STOP RUN.
 ****** **************************** Bottom of Data ****************************
```

### อธิบายจุดสำคัญ

- ตัวเลข 6 หลักทางซ้ายสุดของทุกบรรทัด (เช่น `000100`, `000200`) คือ **หมายเลขบรรทัด (sequence
  number)** ที่ ISPF Editor เติมให้อัตโนมัติ ไม่ใช่ส่วนหนึ่งของ column 1-72 ที่ COBOL fixed-format
  ใช้จริง (เป็นธรรมเนียมที่สืบทอดมาจากยุคบัตรเจาะรู that ต้องมีหมายเลขกำกับลำดับการ์ดกันสับสน)
- พื้นที่ที่พิมพ์ Line Command ได้จริงคือ **ก่อน** หมายเลขบรรทัดเหล่านี้ (ในภาพจำลองข้างต้นเป็น
  พื้นที่ว่างที่ยังไม่ได้พิมพ์คำสั่งใด ๆ)
- `Columns 00001 00072` มุมขวาบนยืนยันว่า Editor กำลังทำงานในโหมด Fixed-Format มาตรฐานของ COBOL
  (column 1-72) ตรงกับกฎที่หลักสูตรนี้ใช้มาตั้งแต่ Part 003

### Line Commands ที่ใช้บ่อยที่สุด

| Line Command | ความหมาย |
|---|---|
| `I` หรือ `I2` | Insert แทรกบรรทัดใหม่ (I2 = แทรก 2 บรรทัด) ใต้บรรทัดที่พิมพ์คำสั่ง |
| `D` หรือ `D3` | Delete ลบบรรทัดนั้น (D3 = ลบ 3 บรรทัดถัดไปรวมบรรทัดนี้) |
| `C` / `CC` | Copy คัดลอกบรรทัด (ใช้คู่ `CC`...`CC` เพื่อทำเครื่องหมายช่วงบรรทัด) |
| `M` / `MM` | Move ย้ายบรรทัด (ใช้คู่แบบเดียวกับ `CC`) |
| `R` หรือ `R5` | Repeat ทำซ้ำบรรทัดนั้น |
| `TE` | Text Entry เปิดโหมดพิมพ์ต่อเนื่องหลายบรรทัดจากจุดนั้น |
| `)` `(` | เลื่อนบรรทัดไปทางขวา/ซ้าย (shift columns) |

### ภาพจำลอง: การใช้ `I` เพื่อแทรกบรรทัดใหม่

```
 000700        FILE-CONTROL.
 i        000800            SELECT TXN-FILE ASSIGN TO TXNIN
 000900                ORGANIZATION IS SEQUENTIAL.
```

หลังกด Enter ที่บรรทัด `i` จะได้บรรทัดว่างใหม่ 1 บรรทัดแทรกอยู่**เหนือ** `000800` ให้พิมพ์เนื้อหา
ใหม่ลงไปได้ทันที

### ข้อควรระวัง

- Line Command ต้องพิมพ์ **ในพื้นที่ที่กำหนดไว้เท่านั้น** (ก่อนหมายเลขบรรทัด) ถ้าพิมพ์ผิดตำแหน่ง
  (เผลอพิมพ์ในเนื้อหาโค้ด) ระบบจะไม่ตีความว่าเป็นคำสั่ง แต่จะแทรกตัวอักษรนั้นเข้าไปในโค้ดจริง ๆ
- คู่คำสั่ง `CC`/`MM` ต้องใช้ **คู่กันเสมอ 2 ตำแหน่ง** เพื่อกำหนดจุดเริ่มต้นและจุดสิ้นสุดของช่วงที่
  จะ copy/move มิฉะนั้น ISPF จะแจ้ง error ว่าคำสั่งไม่สมบูรณ์

### แบบฝึกหัดที่ 534.1

**โจทย์**: จงอธิบายว่าต้องพิมพ์ Line Command ใดเพื่อ "ลบบรรทัดปัจจุบันและอีก 2 บรรทัดถัดไปรวมเป็น
3 บรรทัด"

**เฉลย**: พิมพ์ `D3` ที่ Line Command Area ของบรรทัดแรกที่ต้องการลบ ตัวเลขต่อท้าย (`3`) หมายถึง
จำนวนบรรทัดทั้งหมดที่จะถูกลบนับรวมบรรทัดที่พิมพ์คำสั่งด้วย

---

## ขั้นตอนที่ 535: ISPF Editor — Primary Commands (FIND, CHANGE, SAVE)

### Primary Command Line vs Line Command Area

ต่างจาก Line Command ในขั้นตอนที่ 534 (ที่พิมพ์ติดกับแต่ละบรรทัด) **Primary Command** คือคำสั่งที่
พิมพ์ที่ช่อง **`Command ===>`** บนสุดของหน้าจอ ใช้สำหรับกระทำการที่ไม่ผูกกับบรรทัดใดบรรทัดหนึ่ง
โดยเฉพาะ เช่น ค้นหาข้อความ บันทึกไฟล์ หรือออกจากโปรแกรม

### คำสั่งที่ใช้บ่อยที่สุด

| Primary Command | ความหมาย |
|---|---|
| `FIND text` (ย่อ `F`) | ค้นหาข้อความ `text` จากตำแหน่งปัจจุบันเป็นต้นไป |
| `CHANGE old new` (ย่อ `C`) | เปลี่ยน `old` เป็น `new` ครั้งแรกที่พบ |
| `CHANGE old new ALL` | เปลี่ยนทุกจุดที่พบทั้งไฟล์ (คล้าย find-and-replace-all) |
| `SAVE` | บันทึกการแก้ไขแต่ยังอยู่ใน Editor ต่อ |
| `END` (หรือกด PF3) | บันทึกและออกจาก Editor |
| `CANCEL` | ออกจาก Editor โดย**ไม่บันทึก**การเปลี่ยนแปลงใด ๆ |
| `SORT` | เรียงลำดับบรรทัดตามเงื่อนไขที่ระบุ |
| `NUM ON`/`NUM OFF` | เปิด/ปิดการแสดงหมายเลขบรรทัด |

### ภาพจำลอง: การใช้ FIND เพื่อค้นหาคำว่า "PROCEDURE"

```
 EDIT       COBOL01.SOURCE.COBOL(TXNVALID) - 01.15        Columns 00001 00072
 Command ===> find procedure                                 Scroll ===> CSR
 ****** ***************************** Top of Data ******************************
 000100        IDENTIFICATION DIVISION.
```

หลังกด Enter เคอร์เซอร์จะกระโดดไปยังบรรทัดแรกที่พบคำว่า `PROCEDURE` (ไม่สนใจตัวพิมพ์เล็ก-ใหญ่
ตามค่าเริ่มต้น) พร้อมข้อความยืนยันที่แถบสถานะ เช่น `PROCEDURE FOUND` และแสดงบรรทัดนั้นให้เห็นบน
หน้าจอทันที

### ภาพจำลอง: การใช้ CHANGE ALL

```
 Command ===> c TXN-FILE TRANS-FILE all
```

คำสั่งนี้เปลี่ยนทุกจุดที่มีคำว่า `TXN-FILE` เป็น `TRANS-FILE` ทั่วทั้งไฟล์ในคำสั่งเดียว — มีประโยชน์
มากเมื่อต้องเปลี่ยนชื่อตัวแปรหรือชื่อไฟล์ที่ใช้ซ้ำหลายจุดในโปรแกรมยาวหลายพันบรรทัด

### วงจรการทำงานทั่วไป: Edit → Save → End

1. เปิดไฟล์ผ่าน Option 2 หรือ Line Command `E` จาก DSLIST
2. แก้ไขเนื้อหาด้วย Line Command และพิมพ์ข้อความตามปกติ
3. พิมพ์ `SAVE` เป็นระยะเพื่อบันทึกความคืบหน้า (ป้องกันข้อมูลหายหากการเชื่อมต่อขาด)
4. เมื่อเสร็จสิ้น พิมพ์ `END` หรือกด **PF3** เพื่อบันทึกครั้งสุดท้ายและกลับสู่เมนูก่อนหน้า

### ข้อควรระวัง

- ถ้าต้องการยกเลิกการแก้ไขทั้งหมดโดยไม่บันทึก **ต้องใช้ `CANCEL` เท่านั้น** การกด `END`/PF3 จะ
  บันทึกเสมอแม้จะไม่ได้ตั้งใจ นี่คือกับดักที่มือใหม่พลาดบ่อยมาก
- คำสั่ง `CHANGE ... ALL` ควรใช้ด้วยความระมัดระวังเป็นพิเศษกับไฟล์ COBOL เพราะอาจเปลี่ยนคำที่ปรากฏ
  ในคอมเมนต์หรือ string literal โดยไม่ตั้งใจ ควรตรวจสอบผลลัพธ์ทุกครั้งหลังใช้

### แบบฝึกหัดที่ 535.1

**โจทย์**: จงเขียน Primary Command ที่ใช้ค้นหาคำว่า `WS-ERROR-COUNT` **ถอยหลัง** จากตำแหน่ง
เคอร์เซอร์ปัจจุบัน

**เฉลย**: `FIND WS-ERROR-COUNT PREV` (หรือย่อ `F WS-ERROR-COUNT PREV`) — คำว่า `PREV` บอกให้
ค้นหาย้อนขึ้นไปด้านบนแทนที่จะค้นลงด้านล่างตามค่าเริ่มต้น

---

## ขั้นตอนที่ 536: ISPF Utilities — Dataset Allocation, Copy, Rename (Option 3.2/3.3)

### ภาพรวมของ Utility Menu (Option 3)

```
                          UTILITY SELECTION MENU
 OPTION ===>

  1  LIBRARY     - Compress or print data set. Print index listing.
  2  DATA SET    - Allocate, rename, delete, catalog, uncatalog,
                    display information for data sets.
  3  MOVE/COPY   - Move, copy, or promote members or data sets.
  4  DSLIST      - Data set list, options, edit, browse.
  5  RESET       - Reset statistics for members of ISPF libraries.
  6  HARDCOPY    - Initiate hardcopy print request.
  ...
```

### Option 3.2 — Data Set Utility (Allocate, Delete, Rename)

**Option 3.2** ใช้จัดการ dataset ระดับ metadata เช่น การสร้าง (Allocate) dataset ใหม่ กำหนด
คุณสมบัติทางกายภาพ (Record Format, Record Length, Space) เปรียบได้กับคำสั่ง `mkfs`/`fallocate`
ผสมกับ `mv`/`rm` ในโลก Linux

```
 DATA SET UTILITY
 Option ===> A

    A  Allocate new data set              C  Catalog data set
    R  Rename entry                       U  Uncatalog data set
    D  Delete entry                       ...

 ISPF Library:
    Project  . . . COBOL01
    Group  . . . . SOURCE
    Type . . . . . COBOL

 Other Partitioned, Sequential or VSAM Data Set:
    Data Set Name . . . COBOL01.NEW.DATASET
```

### ภาพจำลอง: หน้าจอ Allocate New Data Set (หลังเลือก A)

```
 ALLOCATE NEW DATA SET
 Command ===>

 Data Set Name . . . : COBOL01.NEW.DATASET

 Management class . . .  (Blank for default management class)
 Storage class . . . .  (Blank for default storage class)
 Volume serial . . . .  (Blank for authorized default volume)
 Device type . . . . .  (Generic unit or device address)
 Data class. . . . . .  (Blank for default data class)
 Space units . . . . . TRACK      (BLKS, TRKS, CYLS, KB, MB, BYTES)
 Average record unit    (M, K, or U)
 Primary quantity . . . 10         (In above units)
 Secondary quantity     5          (In above units)
 Directory blocks . . . 5          (Zero for sequential data set)
 Record format . . . . FB
 Record length . . . . 80
 Block size  . . . . . 27920
 Data set name type . . PDS        (LIBRARY, PDS, or blank)
```

### อธิบายจุดสำคัญ

- **Record Format (RECFM)** `FB` หมายถึง Fixed Block (ความยาวคงที่ทุก record) — ตรงกับที่โค้ด
  COBOL fixed-format ใช้เสมอ ต่างจาก `VB` (Variable Block) ที่ความยาวแต่ละ record ไม่เท่ากัน
- **Directory Blocks** ที่ไม่เป็นศูนย์ หมายถึงกำลังสร้าง **PDS (Partitioned Data Set)** ซึ่งเป็น
  "library" ที่เก็บ member ย่อยได้หลายตัว (คล้าย directory ที่เก็บหลายไฟล์) ต่างจาก Sequential
  Data Set ที่เก็บข้อมูลเป็นก้อนเดียวไม่มี member ย่อย
- **Space Units** เป็นหน่วยเฉพาะของ Mainframe (`TRACK`, `CYLS`) ต่างจากการระบุขนาดเป็น MB/GB
  ตรง ๆ แบบระบบไฟล์ทั่วไป เพราะสืบทอดมาจากยุคที่ดิสก์กายภาพแบ่งเป็น Track/Cylinder จริง

### Option 3.3 — Move/Copy

**Option 3.3** ใช้คัดลอกหรือย้าย member/dataset ระหว่างกัน เทียบเท่า `cp`/`mv` ในโลก Linux
มีประโยชน์มากเมื่อต้องการคัดลอก source code จาก library ทดสอบไปยัง library ของ Production

### ข้อควรระวัง

- การกำหนด **Block Size ผิดพลาด** (ไม่ลงตัวกับ Record Length) เป็นข้อผิดพลาดที่มือใหม่เจอบ่อยเมื่อ
  Allocate dataset เอง หลายองค์กรตั้งค่าให้ `0` เพื่อให้ระบบคำนวณค่าที่เหมาะสมที่สุดให้อัตโนมัติ
  (เรียกว่า System Determined Block Size)
- การลบ/เขียนทับ dataset ผ่าน Utility เหล่านี้ **ไม่มีการยืนยันซ้ำเสมอไปและไม่มีถังขยะ** ต้องตรวจ
  สอบชื่อ dataset ให้ถูกต้องก่อนยืนยันทุกครั้ง

### แบบฝึกหัดที่ 536.1

**โจทย์**: จงอธิบายว่าทำไมการสร้าง PDS (Directory Blocks ไม่เป็นศูนย์) จึงเหมาะกับการเก็บ
source code COBOL ของหลายโปรแกรมมากกว่าการสร้างเป็น Sequential Data Set แยกไฟล์ละ dataset

**เฉลย**: PDS ทำหน้าที่เหมือน "โฟลเดอร์" ที่เก็บ member (แต่ละโปรแกรม) ไว้ภายใต้ชื่อ dataset เดียว
ทำให้จัดการง่ายกว่ามาก (เช่น `COBOL01.SOURCE.COBOL(TXNVALID)` และ
`COBOL01.SOURCE.COBOL(TXNUPDATE)` อยู่ใน PDS เดียวกัน) ต่างจากการสร้างเป็น Sequential Data
Set แยกทีละไฟล์ที่จะทำให้มี dataset นับพันชื่อกระจัดกระจาย ยากต่อการค้นหาและบริหารจัดการสิทธิ์

---

## ขั้นตอนที่ 537: SDSF เบื้องต้น — การดู Job Output

### SDSF คืออะไร

**SDSF (System Display and Search Facility)** คือเครื่องมือ (มักเข้าถึงผ่าน ISPF Option เพิ่มเติม
หรือคำสั่ง TSO `SDSF` โดยตรง) ที่ใช้**ตรวจสอบสถานะและผลลัพธ์ของ job ที่ submit เข้า JES**
(เชื่อมโยงโดยตรงกับ JCL ที่เรียนใน Part 052-053 — เมื่อ `SUBMIT` job แล้ว SDSF คือที่ที่ไปดูว่า
job รันสำเร็จหรือไม่ และดู `SYSOUT`/`SYSPRINT` ที่โปรแกรมพิมพ์ออกมา)

### ภาพจำลอง: หน้าจอ SDSF แสดงรายการ Job (Panel ST - Status)

```
 SDSF STATUS DISPLAY ALL CLASSES                    LINE 1-4 (4)
 COMMAND INPUT ===>                                   SCROLL ===> CSR
 PREFIX COBOL*   DEST (ALL)     OWNER *        SYSNAME -

 NP   JOBNAME  JobID    Owner    Prty Queue  C Pos    SAff ASys  Status
      NIGHTBAT JOB12345 COBOL01   1  PRINT     Z            SC70  OUTPUT
      DAYBATCH JOB12346 COBOL01   1  EXECUTING            SC70  ACTIVE
      TESTJOB1 JOB12347 COBOL01   1  INPUT             SC70  WAITING
      OLDJOB99 JOB12340 COBOL01   1  PRINT     A            SC70  ABEND S0C7
```

### อธิบายจุดสำคัญ

- คอลัมน์ **Status** บอกสถานะปัจจุบันของ job: `ACTIVE` (กำลังรันอยู่), `WAITING` (รออยู่ในคิว),
  `OUTPUT` (รันเสร็จแล้ว มี output รอให้ดู), หรือแสดง **Abend Code** โดยตรงถ้า job ล้มเหลว (เช่น
  `ABEND S0C7` หมายถึง Data Exception — ข้อผิดพลาดจากการประมวลผลข้อมูลที่ไม่ใช่ตัวเลขในฟิลด์
  ตัวเลข ซึ่งเป็น abend code ที่โปรแกรมเมอร์ COBOL เจอบ่อยมากในโลกจริง)
- คอลัมน์ **NP** (Next Page/Action) คือช่องสำหรับพิมพ์ Line Command เพื่อกระทำการกับ job นั้น
  เช่น `S` (Display, ดู output), `?` (Display Job Data Set list), `C` (Cancel), `P` (Purge, ลบ
  ออกจากคิวถาวร), `H` (Hold, พักไว้ก่อน)

### ภาพจำลอง: การดูรายละเอียด Output ของ Job (หลังพิมพ์ `S` ที่ NIGHTBAT)

```
 SDSF OUTPUT DISPLAY NIGHTBAT JOB12345    DSID  2    LINE 0    COLS 02-81
 COMMAND INPUT ===>                                     SCROLL ===> CSR
 J E S 2  J O B  L O G  --  S Y S T E M  S C 7 0  --  N O D E  N 1
 1 JOB12345 IRR010I  USERID COBOL01  IS ASSIGNED TO THIS JOB.
 1 JOB12345 ICH70001I COBOL01  LAST ACCESS AT 22:10:33 ON MONDAY
 1 JOB12345 $HASP373 NIGHTBAT STARTED - INIT 3 - CLASS A - SYS SC70
 -               STEP1   PROC=RUNCOBOL  PGM=TXNVALID   RC=0000
 -               STEP2   PROC=RUNCOBOL  PGM=TXNUPDATE  RC=0000
 -               STEP3   PROC=RUNCOBOL  PGM=SENDALERT  RC=0000
 -               STEP4   PROC=RUNCOBOL  PGM=DAILYRPT   RC=0000
 1 JOB12345 $HASP395 NIGHTBAT ENDED - RC=0000
```

สังเกตว่า job นี้คือตัวอย่างเดียวกับกรณีศึกษาใน Part 053 ขั้นตอนที่ 530 — SDSF คือหน้าจอที่จะแสดง
ผลลัพธ์จริงของแต่ละ step (`RC=0000` ของแต่ละ step ตรงกับค่า `RETURN-CODE` ที่โปรแกรม COBOL
กำหนดไว้)

### ข้อควรระวัง

- SDSF แสดงผลเฉพาะ job ที่ยังอยู่ใน **JES Spool** (พื้นที่จัดเก็บชั่วคราว) เท่านั้น หากผ่านไปนาน
  เกินนโยบายที่องค์กรกำหนด (retention policy) job เก่าจะถูกลบออกจาก spool อัตโนมัติและหาไม่เจอ
  ใน SDSF อีกต่อไป
- สิทธิ์การดู/ยกเลิก job ของผู้อื่นถูกควบคุมด้วย RACF (Part 069) — ผู้ใช้ทั่วไปมักเห็นได้เฉพาะ
  job ของตัวเอง ไม่สามารถ Cancel job ของคนอื่นได้หากไม่มีสิทธิ์พิเศษ

### แบบฝึกหัดที่ 537.1

**โจทย์**: จากภาพจำลองหน้าจอ SDSF STATUS ข้างต้น จง identify ว่า job ใดควรถูกตรวจสอบโดยด่วน
ที่สุด และเพราะเหตุใด

**เฉลย**: `OLDJOB99` (JobID `JOB12340`) เพราะสถานะแสดง `ABEND S0C7` ซึ่งหมายถึง job ล้มเหลว
จาก Data Exception ควรเปิดดู output ผ่าน Line Command `S` ทันทีเพื่อหาสาเหตุ (มักเกิดจากข้อมูล
ตัวเลขที่ผิดรูปแบบในไฟล์นำเข้า) ก่อนที่ปัญหาจะกระทบ job อื่นที่ต่อคิวรอ resource เดียวกัน

---

## ขั้นตอนที่ 538: SDSF Commands ขั้นสูง — Cancel, Purge, Hold, Filter

### คำสั่งจัดการ Job ผ่าน Line Command

| Line Command | ความหมาย |
|---|---|
| `C` | Cancel — ยกเลิก job ที่กำลังรันหรือรออยู่ในคิว |
| `P` | Purge — ลบ job ออกจาก spool ถาวรหลังจากดู output เสร็จแล้ว |
| `H` | Hold — พัก job ไว้ก่อน ยังไม่ให้เริ่มรัน (หรือหยุด output ไม่ให้พิมพ์ออก) |
| `R` | Release — ปลดจากสถานะ Hold ให้กลับมาทำงานต่อ |
| `?` | แสดงรายชื่อ Data Set (DD) ทั้งหมดที่ job นั้นสร้างขึ้น |
| `X` | Purge job ทันทีโดยไม่ต้องยืนยันซ้ำ (ระวังการใช้งาน) |

### ภาพจำลอง: การ Filter รายการ Job ด้วย PREFIX และ OWNER

```
 SDSF STATUS DISPLAY ALL CLASSES                    LINE 1-2 (2)
 COMMAND INPUT ===> PREFIX NIGHT* OWNER COBOL01
 PREFIX NIGHT*   DEST (ALL)     OWNER COBOL01     SYSNAME -

 NP   JOBNAME  JobID    Owner    Prty Queue  C Pos    SAff ASys  Status
      NIGHTBAT JOB12345 COBOL01   1  PRINT     Z            SC70  OUTPUT
```

พิมพ์ `PREFIX NIGHT*` ที่ Command Input เพื่อกรองให้เห็นเฉพาะ job ที่ชื่อขึ้นต้นด้วย `NIGHT`
และ `OWNER COBOL01` เพื่อกรองเฉพาะ job ที่ submit โดย userid `COBOL01` — มีประโยชน์มากในองค์กร
ที่มี job นับพันวิ่งพร้อมกันบนระบบเดียว

### ภาพจำลอง: การใช้ Line Command เพื่อ Cancel Job ที่ค้าง

```
 NP   JOBNAME  JobID    Owner    Prty Queue  C Pos    SAff ASys  Status
 c    STUCKJOB JOB12350 COBOL01   1  EXECUTING            SC70  ACTIVE
```

พิมพ์ `c` แล้วกด Enter ระบบจะแสดงข้อความยืนยันก่อน cancel จริงเสมอ (เพื่อป้องกันการกดพลาด)

### ทำไมต้อง Purge Job หลัง Cancel

หลาย SDSF panel แยกการ **Cancel** (หยุดการทำงาน) ออกจากการ **Purge** (ลบทิ้งออกจาก spool)
เป็นคนละขั้นตอนกัน เพราะบางครั้งแม้ job จะถูก cancel ไปแล้ว output ที่พิมพ์ไปก่อนหน้ายังมีประโยชน์
สำหรับการวิเคราะห์ปัญหา (root cause analysis) การรวม Cancel+Purge เป็นคำสั่งเดียว (เช่น `CP` ใน
บาง configuration) มีให้ใช้เพื่อความรวดเร็วเมื่อแน่ใจแล้วว่าไม่ต้องการ output นั้นอีกต่อไป

### ข้อควรระวัง

- `X` (Purge ทันทีไม่ถามยืนยัน) เป็นคำสั่งที่ควรใช้ด้วยความระมัดระวังสูงสุด เพราะลบ output ทิ้งถาวร
  ทันทีโดยไม่มีโอกาสกู้คืน
- การ Hold job ที่มีอยู่ในคิวการรันแบบ Interdependent (job อื่นรอผลจาก job นี้ผ่าน scheduler เช่น
  ที่จะสอนใน Part 066) อาจทำให้ job ต่อเนื่องทั้งสายค้างรอไปด้วย ควรตรวจสอบผลกระทบก่อน Hold เสมอ

### แบบฝึกหัดที่ 538.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `H` (Hold) กับ `C` (Cancel) ใน SDSF

**เฉลย**: `H` (Hold) เป็นการ "พักไว้ชั่วคราว" job ยังคงอยู่ในระบบและสามารถ Release (`R`) ให้กลับมา
ทำงานต่อได้ภายหลัง เหมาะกับสถานการณ์ที่ต้องการหยุดชั่วคราวเพื่อตรวจสอบบางอย่างก่อน ส่วน `C`
(Cancel) เป็นการ "ยกเลิกถาวร" job นั้นจะไม่ทำงานต่ออีก ถ้าต้องการรันใหม่ต้อง submit job นั้นเข้าไป
ใหม่ทั้งหมด

---

## ขั้นตอนที่ 539: Split Screen และ TSO Command Shell ภายใน ISPF

### แนวคิดของ Split Screen

ISPF รองรับการเปิดหลาย **"หน้าจอเสมือน" (logical screen)** พร้อมกันในเซสชันเดียว ผ่านคำสั่ง
`SPLIT` (มักผูกกับ PF Key เช่น PF2) ทำให้สามารถทำงาน 2 อย่างคู่ขนานได้ เช่น หน้าจอหนึ่งแก้ไข
source code COBOL ด้วย Editor ในขณะที่อีกหน้าจอดูผล SDSF ของ job ที่เพิ่ง submit ไป — คล้ายกับ
การแบ่งหน้าจอ Terminal เป็นหลาย pane ในเครื่องมือสมัยใหม่อย่าง tmux

### ภาพจำลอง: หน้าจอหลัง SPLIT

```
+------------------------ SCREEN 1 -------------------------+
| EDIT  COBOL01.SOURCE.COBOL(TXNVALID) - 01.15               |
| Command ===>                                                |
| 000100        IDENTIFICATION DIVISION.                     |
| 000200        PROGRAM-ID. TXNVALID.                         |
+--------------------------------------------------------------
+------------------------ SCREEN 2 -------------------------+
| SDSF STATUS DISPLAY ALL CLASSES                             |
| COMMAND INPUT ===>                                           |
| NIGHTBAT JOB12345 COBOL01   1  EXECUTING     SC70  ACTIVE   |
+--------------------------------------------------------------
```

สลับไปมาระหว่างหน้าจอด้วย PF Key (มักเป็น PF9 = SWAP) โดยไม่ต้องปิดหน้าจอใดหน้าจอหนึ่งก่อน

### TSO Command Shell ภายใน ISPF (Option 6)

**Option 6** ของ ISPF Primary Option Menu เปิดให้พิมพ์คำสั่ง TSO ได้โดยตรงโดยไม่ต้องออกจาก ISPF
ก่อน มีประโยชน์เมื่อต้องการรันคำสั่งสั้น ๆ อย่าง `LISTDS` หรือ `SUBMIT` โดยไม่อยากสลับไปมาระหว่าง
เมนูหลายชั้น

```
                      ISPF PRIMARY OPTION MENU
 OPTION ===> 6

TSO COMMAND ===> submit 'COBOL01.JCL.LIB(NIGHTBAT)'
```

### ข้อควรระวัง

- จำนวนหน้าจอ Split สูงสุดขึ้นกับการตั้งค่าของระบบ (มักจำกัดที่ 2-8 หน้าจอ) การเปิดหน้าจอมากเกินไป
  จะใช้ทรัพยากรระบบ (Region Size) มากขึ้นตามไปด้วย
- คำสั่ง TSO ที่พิมพ์ผ่าน Option 6 ทำงาน**ภายใต้สิทธิ์ userid เดียวกัน**กับที่ล็อกอินเข้า ISPF
  เสมอ ไม่ใช่การสลับ context ไปเป็นผู้ใช้อื่น

### แบบฝึกหัดที่ 539.1

**โจทย์**: จงอธิบายประโยชน์ของ Split Screen ในบริบทของ workflow "แก้โค้ด → submit job → ดูผล"
ที่เรียนมาตลอด Part นี้

**เฉลย**: Split Screen ทำให้ไม่ต้องปิด Editor เพื่อไปเปิด SDSF แล้วต้องเปิด Editor กลับมาใหม่ทุก
รอบ ซึ่งเสียเวลาและเสี่ยงทำโค้ดที่แก้ค้างอยู่หายถ้าลืม save ก่อนออก การมีทั้งสองหน้าจอเปิดพร้อมกัน
ทำให้ตรวจสอบผลลัพธ์แล้วย้อนกลับไปแก้โค้ดต่อได้ทันทีในรอบเดียว ตรงกับรูปแบบการทำงานวนซ้ำ (edit-
compile-test loop) ที่โปรแกรมเมอร์ทุกยุคสมัยต้องทำซ้ำหลายรอบต่อวัน

---

## ขั้นตอนที่ 540: กรณีศึกษารวม — วงจรการทำงานเต็มรูปแบบบน TSO/ISPF

### สถานการณ์จำลอง

โปรแกรมเมอร์ COBOL คนหนึ่งได้รับมอบหมายให้แก้ไขข้อบกพร่อง (bug) เล็กน้อยในโปรแกรม `TXNVALID`
(โปรแกรมเดียวกับกรณีศึกษา Part 053) แล้ว submit job ทดสอบ ต่อไปนี้คือลำดับขั้นตอนแบบเต็มรูปแบบ
ที่จำลองการทำงานจริงบน TSO/ISPF ตั้งแต่ต้นจนจบ

### ขั้นตอนที่ 1: LOGON เข้าสู่ระบบ (ขั้นตอนที่ 531)

```
Userid    ===> COBOL01
Password  ===> ********
```

### ขั้นตอนที่ 2: เข้า ISPF แล้วไปที่ Option 3.4 เพื่อค้นหาไฟล์ (ขั้นตอนที่ 532-533)

```
OPTION ===> 3.4
Dataset Name Pattern ===> COBOL01.SOURCE.*
```

### ขั้นตอนที่ 3: ใช้ Line Command E เพื่อเปิด Editor (ขั้นตอนที่ 534-535)

```
e    COBOL01.SOURCE.COBOL(TXNVALID)
```

### ขั้นตอนที่ 4: แก้ไขโค้ดภายใน ISPF Editor

```
 EDIT       COBOL01.SOURCE.COBOL(TXNVALID) - 01.16        Columns 00001 00072
 Command ===> find ws-error-count
 001700        01  WS-ERROR-COUNT       PIC 9(5) VALUE 0.
 001800        01  WS-MAX-ERRORS        PIC 9(5) VALUE 10.
```

โปรแกรมเมอร์เพิ่มบรรทัด `WS-MAX-ERRORS` ด้วย Line Command `I` แล้วพิมพ์ `SAVE` เพื่อบันทึก
จากนั้นพิมพ์ `END` (หรือ PF3) เพื่อออกจาก Editor กลับสู่ DSLIST

โค้ด COBOL ที่แก้ไขนี้เป็นภาษามาตรฐาน ตัวอย่างโปรแกรมเต็มที่สอดคล้องกันสามารถ **compile และรันได้
จริงบน GnuCOBOL**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TXNVALID.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ERROR-COUNT       PIC 9(5) VALUE 0.
       01  WS-MAX-ERRORS        PIC 9(5) VALUE 10.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "TRANSACTION VALIDATION STARTING".
           DISPLAY "MAX ERRORS ALLOWED: " WS-MAX-ERRORS.
           IF WS-ERROR-COUNT > WS-MAX-ERRORS
               DISPLAY "TOO MANY ERRORS - ABORTING"
               MOVE 16 TO RETURN-CODE
           ELSE
               DISPLAY "VALIDATION WITHIN LIMITS - RC=0"
               MOVE 0 TO RETURN-CODE
           END-IF.
           STOP RUN.
```

ทดสอบคอมไพล์และรันจริงด้วย GnuCOBOL (ยืนยันแล้ว):

```bash
cobc -x -o txnvalid txnvalid.cob
./txnvalid
```

**ผลลัพธ์จริง:**

```
TRANSACTION VALIDATION STARTING
MAX ERRORS ALLOWED: 00010
VALIDATION WITHIN LIMITS - RC=0
```

### ขั้นตอนที่ 5: Submit JCL job ผ่าน Option 6 หรือคำสั่ง TSO SUBMIT (ขั้นตอนที่ 531, 539)

```
TSO COMMAND ===> submit 'COBOL01.JCL.LIB(TESTJOB1)'
IKJ56250I JOB TESTJOB1(JOB12351) SUBMITTED
```

### ขั้นตอนที่ 6: ตรวจสอบผลลัพธ์ผ่าน SDSF (ขั้นตอนที่ 537-538)

```
COMMAND INPUT ===> PREFIX TESTJOB1 OWNER COBOL01

NP   JOBNAME  JobID    Owner    Prty Queue  C Pos    SAff ASys  Status
     TESTJOB1 JOB12351 COBOL01   1  PRINT     Z            SC70  OUTPUT
```

พิมพ์ `s` ที่หน้า `TESTJOB1` เพื่อดู output แล้วตรวจสอบว่า `RC=0000` ตรงกับที่ตั้งใจไว้ในโค้ด
COBOL — ปิดวงจรครบสมบูรณ์: **แก้โค้ดผ่าน Editor → submit ผ่าน TSO → ตรวจผลผ่าน SDSF**

### ข้อควรระวัง

- วงจรทั้งหมดนี้เป็นภาพจำลองการทำงานเชิงแนวคิด **ไม่สามารถทำตามได้จริงในสภาพแวดล้อมของหลักสูตร
  นี้** เนื่องจากไม่มี TSO/ISPF/JES ติดตั้งอยู่ — ส่วนที่ **ทดสอบคอมไพล์และรันได้จริง** คือเฉพาะ
  โค้ด COBOL ในขั้นตอนที่ 4 เท่านั้น ซึ่งคุณสามารถลองทำตามได้จริงด้วย GnuCOBOL บนเครื่องของคุณเอง
- ในการทำงานจริง ควรฝึกความเคยชินกับ PF Key มาตรฐาน (PF3=End, PF7/PF8=Scroll Up/Down,
  PF9=Swap) ให้คล่องตั้งแต่ต้น เพราะจะช่วยเพิ่มความเร็วในการทำงานอย่างมากเมื่อต้องใช้งาน ISPF
  ทุกวันในสภาพแวดล้อมจริง

### แบบฝึกหัดที่ 540.1

**โจทย์**: จงเรียงลำดับขั้นตอนต่อไปนี้ให้ถูกต้องตามวงจรการทำงานจริงบน TSO/ISPF: (ก) ดูผลลัพธ์ผ่าน
SDSF (ข) LOGON เข้าสู่ระบบ (ค) แก้ไข source code ผ่าน ISPF Editor (ง) Submit JCL job

**เฉลย**: ข → ค → ง → ก (LOGON ก่อนเสมอ ตามด้วยแก้โค้ด จากนั้น submit job แล้วจึงไปตรวจสอบ
ผลลัพธ์เป็นลำดับสุดท้าย)

---

## สรุปท้ายบท

ใน Part นี้ เราได้ทำความรู้จักกับสภาพแวดล้อมเชิงโต้ตอบมาตรฐานของ Mainframe:

- TSO (Time Sharing Option) และแนวคิดการแบ่งเวลาใช้งาน Mainframe ร่วมกันหลายคน
- ISPF และ Primary Option Menu ที่เป็นประตูสู่ทุกฟีเจอร์การทำงาน
- Option 3.4 (Dataset List Utility) สำหรับค้นหาและจัดการ dataset
- ISPF Editor: Line Commands (`I`, `D`, `C`, `M`) สำหรับแก้ไขทีละบรรทัด
- ISPF Editor: Primary Commands (`FIND`, `CHANGE`, `SAVE`, `END`, `CANCEL`)
- Option 3.2/3.3 สำหรับ Allocate/Copy/Rename dataset ระดับ metadata
- SDSF สำหรับตรวจสอบสถานะและ output ของ job ที่ submit เข้า JES
- SDSF Commands ขั้นสูง: Cancel, Purge, Hold, Release, Filter
- Split Screen และ TSO Command Shell (Option 6) สำหรับทำงานคู่ขนาน
- กรณีศึกษารวมที่แสดงวงจรการทำงานเต็มรูปแบบ: LOGON → Edit → Submit → ตรวจผลผ่าน SDSF

**ย้ำอีกครั้ง**: ภาพหน้าจอทั้งหมดใน Part นี้เป็นภาพจำลอง ASCII ที่วาดขึ้นเพื่อประกอบความเข้าใจ
ไม่สามารถทดสอบใช้งานจริงได้ในสภาพแวดล้อมของหลักสูตรนี้ ส่วนโค้ด COBOL ในขั้นตอนที่ 540 ได้ทดสอบ
คอมไพล์และรันจริงด้วย GnuCOBOL แล้ว

Part ถัดไป (**Part 055**) จะพาคุณกลับสู่โลกของการจัดการไฟล์อีกครั้ง แต่คราวนี้ในระดับที่ลึกกว่า
Indexed/Relative File ที่เรียนใน Part 028-029: **VSAM (Virtual Storage Access Method)** ระบบ
จัดเก็บไฟล์ประสิทธิภาพสูงมาตรฐานของ Mainframe ที่แท้จริง พร้อมทั้งจะเชื่อมโยงให้เห็นว่าสิ่งที่คุณ
เรียนรู้ไปแล้วใน GnuCOBOL นั้นเป็น "แบบจำลองที่ใช้งานได้จริง" ของ VSAM อย่างไร

**[← กลับไป Part 053: JCL ขั้นสูง](part-053-jcl-advanced.md)** | **[ไปยัง Part 055: VSAM เบื้องต้น: KSDS, ESDS, RRDS →](part-055-vsam-basics.md)**
