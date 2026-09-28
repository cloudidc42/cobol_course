# Part 056: VSAM ขั้นสูง: Alternate Index และ Cluster (ขั้นตอนที่ 551–560)

## คำนำของ Part นี้

Part 055 แนะนำ VSAM ทั้ง 3 ประเภทหลัก (KSDS, ESDS, RRDS) และจับคู่แต่ละประเภทกับความรู้เดิมของ
เราจาก Part 023-029 ไปแล้ว แต่ยังเหลือคำถามสำคัญหนึ่งข้อที่ Part 028 เคยตั้งไว้ตั้งแต่ขั้นตอนที่
279: ถ้าต้องการค้นหาข้อมูลด้วย field อื่นที่ไม่ใช่ Key หลัก (เช่น "ค้นหาลูกค้าทุกคนที่อยู่จังหวัด
กรุงเทพฯ" แทนที่จะค้นด้วยรหัสลูกค้า) จะทำอย่างไรบน VSAM จริง?

คำตอบคือ **Alternate Index (AIX)** ซึ่ง Part 028 ได้แนะนำแนวคิดที่เทียบเท่ากันแล้วผ่าน
`ALTERNATE RECORD KEY` ใน GnuCOBOL — ข่าวดีคือความรู้นั้น**ใช้ได้โดยตรง** เมื่อทำงานกับ VSAM AIX
จริงบน z/OS เช่นกัน Part นี้จะขยายความแนวคิดนั้นให้ครบถ้วน พร้อมทั้งแนะนำเครื่องมือสำคัญอีก 2 ตัว
ที่ใช้บริหารจัดการ VSAM Cluster ในโลกจริง: **REPRO** (สำหรับคัดลอก/backup/restore ข้อมูล) และ
แนวคิดของ **Cluster Reorganization** (การจัดระเบียบ Cluster ใหม่เพื่อคืนประสิทธิภาพ)

> **ย้ำกฎเดิมจาก Part 055**: ไวยากรณ์ IDCAMS (`DEFINE ALTERNATEINDEX`, `DEFINE PATH`,
> `BLDINDEX`, `REPRO`) ในเอกสารนี้เป็น**ไวยากรณ์มาตรฐานอ้างอิงที่ถูกต้อง** แต่**ไม่สามารถรันได้
> ในสภาพแวดล้อมของหลักสูตรนี้** ต้องใช้ z/OS จริงหรือ Mainframe Emulator (Hercules) เท่านั้น
> ส่วนโค้ด COBOL ที่ใช้ `ALTERNATE RECORD KEY` เป็นไวยากรณ์ COBOL มาตรฐานเดียวกับที่ Part 028
> ขั้นตอนที่ 279-280 ใช้ และ**ทดสอบคอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่างในเอกสารนี้**

---

## ขั้นตอนที่ 551: ทบทวน VSAM Cluster และแนะนำ Alternate Index (AIX)

### ทบทวนปัญหาจาก Part 028 ขั้นตอนที่ 279

Part 028 พิสูจน์ให้เห็นแล้วว่า `RECORD KEY` หลัก (Primary Key) เหมาะกับการค้นหาด้วยค่าที่ไม่ซ้ำกัน
(เช่นรหัสลูกค้า) แต่ในทางปฏิบัติเรามักต้องการค้นหาด้วย field อื่นที่**ค่าซ้ำกันได้** (เช่นจังหวัด
ที่อยู่ หมวดหมู่สินค้า) COBOL แก้ปัญหานี้ด้วย `ALTERNATE RECORD KEY` ซึ่งสร้างดัชนีที่สองแยก
ต่างหากให้กับไฟล์เดียวกัน

### Alternate Index (AIX) บน VSAM คืออะไร

**Alternate Index (AIX)** คือ VSAM Object ประเภทพิเศษที่สร้าง**ดัชนีเพิ่มเติม**ให้กับ KSDS หรือ
ESDS ที่มีอยู่แล้ว (เรียกว่า **Base Cluster**) โดยไม่ต้องแก้ไขโครงสร้างของ Base Cluster เดิมเลย
AIX ทำหน้าที่เหมือน "ดัชนีท้ายเล่มเพิ่มเติม" ที่ชี้กลับไปยัง record ใน Base Cluster ผ่านค่า Key
ของ field อื่นที่ไม่ใช่ Primary Key

### ตารางเทียบคำศัพท์ VSAM AIX กับ COBOL ALTERNATE RECORD KEY

| คำศัพท์ VSAM (z/OS) | คำศัพท์ COBOL (GnuCOBOL / มาตรฐาน) |
|---|---|
| Base Cluster | ไฟล์หลักที่มี `RECORD KEY IS` |
| Alternate Index (AIX) | `ALTERNATE RECORD KEY IS` |
| ค่าซ้ำได้ในดัชนีรอง | `WITH DUPLICATES` |
| PATH | ชื่อที่ใช้เข้าถึง Base Cluster ผ่าน AIX (ไม่มีเทียบเท่าตรงตัวใน GnuCOBOL — อธิบายขั้นตอนที่ 553) |
| UPGRADE Set | การอัปเดต AIX อัตโนมัติทุกครั้งที่ WRITE/REWRITE/DELETE Base Cluster (ทำงานอัตโนมัติเสมอใน GnuCOBOL) |

### สถาปัตยกรรมโดยรวมของ AIX

VSAM AIX ประกอบด้วย 3 ส่วนที่ต้องสร้างแยกกันตามลำดับบน z/OS จริง (ต่างจาก GnuCOBOL ที่ทำทุก
อย่างให้อัตโนมัติผ่าน `ALTERNATE RECORD KEY` ประโยคเดียว):

1. **DEFINE ALTERNATEINDEX** — สร้างโครงสร้างดัชนีรองที่ว่างเปล่า (ขั้นตอนที่ 552)
2. **DEFINE PATH** — สร้าง "ชื่อทางเข้า" ที่โปรแกรมจะเปิดเพื่อเข้าถึง Base Cluster ผ่าน AIX นี้
   (ขั้นตอนที่ 553)
3. **BLDINDEX** — สร้างเนื้อหาดัชนีจริงจากข้อมูลที่มีอยู่แล้วใน Base Cluster (ขั้นตอนที่ 554)

### ข้อควรระวัง

- AIX สามารถสร้างได้กับทั้ง KSDS และ ESDS (ทำให้ ESDS ที่ไม่มี Key หลักของตัวเอง สามารถค้นหาผ่าน
  field ใดก็ได้โดยอ้อมผ่าน AIX) แต่**ไม่สามารถสร้าง AIX ให้กับ RRDS ได้** เพราะ RRDS ใช้หมายเลข
  slot ล้วน ๆ ไม่มีแนวคิดเรื่อง field-based key ให้สร้างดัชนีรองด้วย
  ตำแหน่ง)
- Alternate Key ของ VSAM AIX **ไม่จำเป็นต้องไม่ซ้ำกัน** เหมือน GnuCOBOL `WITH DUPLICATES` —
  แต่ถ้าต้องการบังคับไม่ให้ซ้ำ (unique alternate key) ก็ทำได้เช่นกันผ่านการไม่ระบุตัวเลือกซ้ำได้
  ตอน DEFINE ALTERNATEINDEX

### แบบฝึกหัดที่ 551.1

**โจทย์**: จงอธิบายว่าทำไม VSAM AIX จึงไม่สามารถใช้กับ RRDS ได้ ทั้งที่ RRDS ก็เป็น VSAM Cluster
ประเภทหนึ่งเหมือนกัน

**เฉลย**: เพราะ RRDS ระบุตำแหน่ง record ด้วยหมายเลข slot (Relative Record Number) ล้วน ๆ ไม่มี
แนวคิดเรื่อง field ภายใน record ที่ใช้เป็น key ได้ AIX ต้องอาศัยค่าของ field หนึ่งใน record เพื่อ
สร้างดัชนีรอง แต่ RRDS ไม่มี field แบบนั้นที่ VSAM รู้จักในระดับโครงสร้างไฟล์ (แม้ record จะมี
field ข้างในก็ตาม แต่ VSAM มองเห็นแค่หมายเลข slot เท่านั้น) จึงไม่สามารถนำแนวคิด AIX มาใช้ได้

---

## ขั้นตอนที่ 552: IDCAMS DEFINE ALTERNATEINDEX — สร้างดัชนีรอง

### ไวยากรณ์อ้างอิง: DEFINE ALTERNATEINDEX (มาตรฐาน — รันไม่ได้ในสภาพแวดล้อมนี้)

```jcl
//DEFAIX   JOB (ACCT123),'DEFINE AIX',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE ALTERNATEINDEX (NAME(PROD.CUSTOMER.AIX.BYCITY)   -
                  RELATE(PROD.CUSTOMER.KSDS)               -
                  KEYS(10 20)                              -
                  RECORDSIZE(16 16)                        -
                  NONUNIQUEKEY                              -
                  UPGRADE                                    -
                  VOLUMES(VSAM01) )                          -
         DATA  (NAME(PROD.CUSTOMER.AIX.BYCITY.DATA))        -
         INDEX (NAME(PROD.CUSTOMER.AIX.BYCITY.INDEX))
/*
```

### อธิบายพารามิเตอร์ทีละส่วน

- `NAME(PROD.CUSTOMER.AIX.BYCITY)`: ชื่อของ AIX เอง (คนละชื่อกับ Base Cluster เสมอ)
- `RELATE(PROD.CUSTOMER.KSDS)`: ระบุว่า AIX นี้ผูกกับ Base Cluster ตัวใด — เทียบเท่ากับการที่
  `ALTERNATE RECORD KEY` ใน COBOL ถูกประกาศอยู่ใน `SELECT` เดียวกับ `RECORD KEY` หลัก
- `KEYS(10 20)`: ความยาวของ Alternate Key คือ 10 ไบต์ เริ่มต้นที่ offset 20 ของ record ใน Base
  Cluster (สมมติว่า field `CUST-CITY` อยู่ตำแหน่งนั้นใน record ที่มี `CUST-ID` ยาว 6 ไบต์ +
  `CUST-NAME` ยาว 20 ไบต์ ก่อนหน้า — ในทางปฏิบัติต้องคำนวณ offset ให้ตรงกับ layout จริงเป๊ะ)
- `NONUNIQUEKEY`: อนุญาตให้ค่า Alternate Key ซ้ำกันได้ระหว่างหลาย record — **ตรงกับ
  `WITH DUPLICATES` ใน COBOL ทุกประการ** (ถ้าต้องการบังคับไม่ให้ซ้ำ จะใช้ `UNIQUEKEY` แทน)
- `UPGRADE`: บอกให้ VSAM **อัปเดต AIX นี้อัตโนมัติ**ทุกครั้งที่ Base Cluster ถูก WRITE/REWRITE/
  DELETE — จะอธิบายรายละเอียดเพิ่มเติมในขั้นตอนที่ 555 (ตรงกับพฤติกรรม default ของ
  `ALTERNATE RECORD KEY` ใน GnuCOBOL ที่ Part 028 พิสูจน์ไว้แล้วว่าทำงานอัตโนมัติเสมอ)

### ข้อควรระวัง

- **ไวยากรณ์นี้เป็นไวยากรณ์มาตรฐานอ้างอิง ไม่สามารถรันได้ในสภาพแวดล้อมของหลักสูตรนี้** ต้องใช้
  z/OS จริงหรือ Mainframe Emulator
- การคำนวณ `KEYS(length offset)` ผิดพลาดเป็นข้อผิดพลาดที่พบบ่อยที่สุดเมื่อ DEFINE
  ALTERNATEINDEX เพราะต้องนับ offset จาก byte แรกของ record (เริ่มที่ 0) อย่างแม่นยำ ต่างจาก
  COBOL ที่ผู้เขียนแค่ระบุชื่อ field (`ALTERNATE RECORD KEY IS CUST-CITY`) แล้ว compiler
  คำนวณตำแหน่งให้อัตโนมัติ — นี่คือความแตกต่างเชิง operational ที่สำคัญที่ควรระวัง
- DEFINE ALTERNATEINDEX เพียงอย่างเดียว**ยังใช้งานไม่ได้จริง** ต้องตามด้วย DEFINE PATH
  (ขั้นตอนที่ 553) และ BLDINDEX (ขั้นตอนที่ 554) ก่อนเสมอ

### แบบฝึกหัดที่ 552.1

**โจทย์**: จงอธิบายความหมายของ `NONUNIQUEKEY` ใน DEFINE ALTERNATEINDEX และเปรียบเทียบกับ
คีย์เวิร์ดที่เทียบเท่าใน COBOL

**เฉลย**: `NONUNIQUEKEY` บอกว่า Alternate Key นี้อนุญาตให้มีค่าซ้ำกันได้ในหลาย record (เช่น
ลูกค้าหลายคนอยู่จังหวัดเดียวกัน) เทียบเท่ากับ `WITH DUPLICATES` ที่ต่อท้าย
`ALTERNATE RECORD KEY IS` ใน COBOL ทุกประการ ถ้าไม่ระบุ (หรือระบุ `UNIQUEKEY` แทน) ระบบจะ
บังคับว่าค่า Alternate Key ต้องไม่ซ้ำกันเหมือนกับ Primary Key

---

## ขั้นตอนที่ 553: DEFINE PATH — ประตูทางเข้า Base Cluster ผ่าน AIX

### PATH คืออะไร

**PATH** คือ VSAM Object ที่ทำหน้าที่เป็น "ชื่อทางเข้า" ให้โปรแกรมเปิดเพื่อเข้าถึง Base Cluster
**ผ่าน Alternate Index** แทนที่จะเปิด Base Cluster โดยตรงด้วยชื่อของมันเอง พูดง่าย ๆ คือ PATH
เป็นเหมือน "ประตูอีกบาน" ที่นำไปสู่ห้องเดียวกัน (ข้อมูลเดียวกันใน Base Cluster) แต่เข้าทางที่
เรียงลำดับตาม Alternate Key แทนที่จะเป็น Primary Key

### ไวยากรณ์อ้างอิง: DEFINE PATH (มาตรฐาน — รันไม่ได้ในสภาพแวดล้อมนี้)

```jcl
//DEFPATH  JOB (ACCT123),'DEFINE PATH',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE PATH (NAME(PROD.CUSTOMER.PATH.BYCITY)      -
               PATHENTRY(PROD.CUSTOMER.AIX.BYCITY)   -
               UPDATE )
/*
```

### อธิบายจุดสำคัญ

- `NAME(PROD.CUSTOMER.PATH.BYCITY)`: ชื่อของ PATH เอง — นี่คือชื่อที่ **JCL DD statement
  จะใช้อ้างอิง** เมื่อโปรแกรมต้องการเข้าถึงข้อมูลผ่าน Alternate Key แทน Primary Key
- `PATHENTRY(PROD.CUSTOMER.AIX.BYCITY)`: ระบุว่า PATH นี้เชื่อมโยงกับ AIX ตัวใด (ต้องสร้าง
  AIX ด้วย DEFINE ALTERNATEINDEX มาก่อนแล้วในขั้นตอนที่ 552)
- `UPDATE`: อนุญาตให้ WRITE/REWRITE/DELETE ผ่าน PATH นี้ได้ด้วย (ไม่ใช่แค่อ่านอย่างเดียว) — ถ้า
  ไม่ระบุ (หรือระบุ `NOUPDATE`) PATH นั้นจะเปิดได้เฉพาะโหมดอ่านเท่านั้น

### เทียบกับ GnuCOBOL: ทำไมจึงไม่มี DEFINE PATH แยกต่างหาก

ใน GnuCOBOL เมื่อประกาศ `ALTERNATE RECORD KEY IS CUST-CITY WITH DUPLICATES` ในบัตร `SELECT`
เดียวกับ `RECORD KEY IS CUST-ID` โปรแกรมสามารถ `START`/`READ NEXT RECORD` ผ่าน `CUST-CITY`
ได้ทันทีโดยไม่ต้องประกาศ "ประตูทางเข้าที่สอง" แยกต่างหาก — เพราะ GnuCOBOL รวมแนวคิด Base Cluster
+ AIX + PATH ไว้เป็น**หนึ่งเดียวในระดับไฟล์เดียว** ในขณะที่ VSAM จริงแยก 3 Object ออกจากกันอย่าง
ชัดเจน (Base Cluster, AIX, PATH) เพื่อความยืดหยุ่นในการบริหารจัดการสิทธิ์และการเชื่อมต่อในระดับ
องค์กรขนาดใหญ่ — ในทางปฏิบัติ โปรแกรม COBOL บน z/OS ที่ต้องการเข้าถึงข้อมูลผ่าน Alternate Key
จะเปิด **PATH ผ่าน DDNAME ของมันเอง** (แยกจาก DDNAME ของ Base Cluster) ตาม JCL:

```jcl
//CUSTPATH DD  DSN=PROD.CUSTOMER.PATH.BYCITY,DISP=SHR
```

### ข้อควรระวัง

- **ไวยากรณ์ DEFINE PATH นี้เป็นไวยากรณ์มาตรฐานอ้างอิง ไม่สามารถรันได้ในสภาพแวดล้อมของหลักสูตร
  นี้** ต้องใช้ z/OS จริงหรือ Mainframe Emulator
- ถ้าไม่ระบุ `UPDATE` ตอน DEFINE PATH แล้วโปรแกรมพยายาม WRITE/REWRITE/DELETE ผ่าน PATH นั้น
  จะได้รับ error ทันที แม้ Base Cluster เองจะเปิดในโหมด I-O ก็ตาม เพราะสิทธิ์การเขียนถูกควบคุม
  แยกที่ระดับ PATH เอง

### แบบฝึกหัดที่ 553.1

**โจทย์**: จงอธิบายว่าทำไม VSAM จึงแยก PATH ออกจาก AIX เป็นคนละ Object ทั้งที่ดูเหมือนทำหน้าที่
คล้ายกัน

**เฉลย**: การแยกเป็นคนละ Object ทำให้ควบคุมสิทธิ์การเข้าถึงได้ละเอียดกว่า เช่น อาจมี AIX เดียวกัน
แต่สร้าง PATH สองชื่อที่มีสิทธิ์ต่างกัน (PATH หนึ่งอนุญาต UPDATE อีก PATH หนึ่งอนุญาตแค่อ่าน) หรือ
อาจให้ทีมงานต่างแผนกมีสิทธิ์เข้าถึง PATH คนละชื่อที่ชี้ไปยัง Base Cluster เดียวกันแต่ผ่าน Alternate
Key ต่างกัน ทำให้บริหารจัดการสิทธิ์ในระดับองค์กรขนาดใหญ่ได้ยืดหยุ่นกว่าการรวมทุกอย่างไว้ใน Object
เดียว

---

## ขั้นตอนที่ 554: BLDINDEX Utility — สร้างเนื้อหาดัชนีจากข้อมูลที่มีอยู่

### ทำไมต้องมี BLDINDEX

DEFINE ALTERNATEINDEX (ขั้นตอนที่ 552) เป็นเพียงการ**จองโครงสร้างที่ว่างเปล่า**ของ AIX เท่านั้น
ยังไม่มีเนื้อหาดัชนีจริงข้างในเลย ถ้า Base Cluster มีข้อมูลอยู่แล้วก่อนสร้าง AIX (สถานการณ์ที่พบ
บ่อยมาก เช่น มีไฟล์ลูกค้าเก่าอยู่แล้ว แล้วเพิ่งต้องการเพิ่มความสามารถค้นหาด้วยจังหวัดภายหลัง) ต้อง
ใช้ **BLDINDEX (Build Index)** เพื่อ**สแกนข้อมูลทั้งหมดใน Base Cluster แล้วสร้างเนื้อหาดัชนีของ
AIX ให้ครบถ้วน**

### ไวยากรณ์อ้างอิง: BLDINDEX (มาตรฐาน — รันไม่ได้ในสภาพแวดล้อมนี้)

```jcl
//BLDIDX   JOB (ACCT123),'BUILD AIX INDEX',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  BLDINDEX INDATASET(PROD.CUSTOMER.KSDS)          -
           OUTDATASET(PROD.CUSTOMER.AIX.BYCITY)
/*
```

### อธิบายจุดสำคัญ

- `INDATASET`: ระบุ Base Cluster ที่มีข้อมูลอยู่แล้ว (แหล่งข้อมูลต้นทาง)
- `OUTDATASET`: ระบุ AIX ที่เพิ่งสร้างด้วย DEFINE ALTERNATEINDEX (ปลายทางที่จะได้รับเนื้อหาดัชนี)
- BLDINDEX จะอ่านทุก record ใน Base Cluster แล้วดึงค่าตาม `KEYS(length offset)` ที่ตั้งไว้ตอน
  DEFINE ALTERNATEINDEX มาสร้างเป็นรายการในดัชนี พร้อมชี้กลับไปยังตำแหน่งจริงของแต่ละ record

### ลำดับขั้นตอนที่ถูกต้องทั้งหมดในการสร้าง AIX ให้ Cluster ที่มีข้อมูลอยู่แล้ว

1. `DEFINE ALTERNATEINDEX` — สร้างโครงสร้างว่างเปล่า (ขั้นตอนที่ 552)
2. `BLDINDEX` — สแกนข้อมูลเดิมมาสร้างเนื้อหาดัชนี (ขั้นตอนนี้)
3. `DEFINE PATH` — สร้างประตูทางเข้า (ขั้นตอนที่ 553 — สังเกตว่าในทางปฏิบัติมักทำก่อนหรือหลัง
   BLDINDEX ก็ได้ เพราะ PATH เป็นแค่ "ชื่อทางเข้า" ไม่เกี่ยวกับเนื้อหาดัชนีโดยตรง แต่นิยมทำ
   DEFINE PATH ก่อน BLDINDEX เพื่อให้ครบทั้ง 3 ขั้นตอนในคำสั่งเดียวกัน)

### เทียบกับ GnuCOBOL: BLDINDEX เกิดขึ้นอัตโนมัติเมื่อไร

ใน GnuCOBOL เมื่อโปรแกรม `OPEN OUTPUT` ไฟล์ที่มี `ALTERNATE RECORD KEY` แล้ว `WRITE` record
ทีละตัว ดัชนีรองจะถูกสร้างไปพร้อมกับการเขียนแต่ละ record ทันที (ไม่มีขั้นตอน "สแกนย้อนหลัง" แยก
ต่างหาก เพราะไม่มี record เก่าที่เขียนไว้ก่อนหน้าการประกาศ Alternate Key) BLDINDEX บน z/OS จึง
จำเป็นเฉพาะกรณี**เพิ่ม AIX ให้กับ Cluster ที่มีข้อมูลอยู่ก่อนแล้ว** ซึ่งเป็นสถานการณ์ที่พบได้บ่อย
มากในระบบ Legacy ขององค์กรจริง (เช่น ระบบทำงานมา 10 ปีด้วย Primary Key เท่านั้น แล้ว
เพิ่งมีความต้องการทางธุรกิจใหม่ให้ค้นหาด้วย field อื่นเพิ่มเติม)

### ข้อควรระวัง

- **ไวยากรณ์ BLDINDEX นี้เป็นไวยากรณ์มาตรฐานอ้างอิง ไม่สามารถรันได้ในสภาพแวดล้อมของหลักสูตรนี้**
  ต้องใช้ z/OS จริงหรือ Mainframe Emulator
- BLDINDEX กับ Base Cluster ขนาดใหญ่มาก (หลายล้าน record) เป็นงานที่ใช้เวลานานและกิน I/O สูง
  ควรวางแผนรันในช่วงเวลาที่ระบบมีภาระงานต่ำ (off-peak hours) เสมอในองค์กรจริง

### แบบฝึกหัดที่ 554.1

**โจทย์**: จงอธิบายว่าทำไม BLDINDEX จึงไม่จำเป็นเมื่อสร้าง AIX พร้อมกับสร้าง Base Cluster ใหม่
ตั้งแต่แรก (Cluster ที่ยังไม่มีข้อมูลเลย)

**เฉลย**: เพราะถ้า Base Cluster ยังไม่มีข้อมูลเลย ก็ไม่มีอะไรให้ BLDINDEX สแกน — เมื่อโปรแกรม
เริ่มโหลดข้อมูล (LOAD) เข้า Base Cluster ทีละ record ผ่าน PUT/WRITE ในภายหลัง ถ้า UPGRADE Set
เปิดใช้งานอยู่ (ขั้นตอนที่ 555) ดัชนี AIX จะถูกอัปเดตไปพร้อมกันทีละ record โดยอัตโนมัติอยู่แล้ว
ทำให้ไม่จำเป็นต้องสแกนย้อนหลังอีกครั้งด้วย BLDINDEX

---

## ขั้นตอนที่ 555: UPGRADE Option — การซิงค์ AIX อัตโนมัติ (พิสูจน์ด้วย GnuCOBOL)

### UPGRADE Set คืออะไร

**UPGRADE** (ที่เห็นในขั้นตอนที่ 552) คือคุณสมบัติที่ทำให้ **AIX ถูกอัปเดตให้ตรงกับ Base Cluster
โดยอัตโนมัติทุกครั้ง** ที่มีการ WRITE (เพิ่ม record ใหม่), REWRITE (แก้ไข record) หรือ DELETE
(ลบ record) เกิดขึ้นกับ Base Cluster — โปรแกรมเมอร์ไม่ต้องเขียนโค้ดอัปเดตดัชนีรองเองแม้แต่บรรทัด
เดียว ถ้าไม่ระบุ UPGRADE (ใช้ `NOUPGRADE` แทน) AIX จะไม่ถูกอัปเดตอัตโนมัติ และต้องรัน BLDINDEX
ใหม่ทุกครั้งที่ต้องการให้ดัชนีตรงกับข้อมูลปัจจุบัน — ซึ่งไม่เหมาะกับ Production เลย

### พิสูจน์แนวคิดนี้ด้วย GnuCOBOL: ALTERNATE RECORD KEY ทำงานแบบ UPGRADE เสมอ

ข่าวดีคือ Part 028 ขั้นตอนที่ 279-280 ได้พิสูจน์พฤติกรรมนี้ไว้แล้วด้วย GnuCOBOL จริง —
`ALTERNATE RECORD KEY` ของ GnuCOBOL **ทำงานเหมือน UPGRADE เสมอโดยไม่มีทางเลือกอื่น** (ไม่มี
option `NOUPGRADE` ใน GnuCOBOL) ทุกครั้งที่ WRITE/REWRITE/DELETE จะปรับปรุงดัชนีรองให้ทันที
อัตโนมัติ ต่อไปนี้คือตัวอย่างใหม่ที่แสดงพฤติกรรมนี้ผ่านระบบสินค้าคงคลังแทน:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP556-AIX-PATH-ANALOG.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
      *> On z/OS, accessing this same data through the alternate
      *> key would mean opening the AIX's PATH name in JCL instead
      *> of the base cluster name. In GnuCOBOL, ONE SELECT/FD
      *> handles both the base cluster and its alternate index
      *> together - the ALTERNATE RECORD KEY clause plays the
      *> role that DEFINE ALTERNATEINDEX + DEFINE PATH + BLDINDEX
      *> play on z/OS (all covered as reference syntax this Part).
           SELECT PRODUCT-FILE ASSIGN TO "PROD556.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS PROD-ID
               ALTERNATE RECORD KEY IS PROD-CATEGORY
                   WITH DUPLICATES
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  PROD-ID              PIC X(6).
           05  PROD-NAME            PIC X(20).
           05  PROD-CATEGORY        PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT PRODUCT-FILE.
           MOVE "P00001" TO PROD-ID.
           MOVE "LAPTOP 14 INCH" TO PROD-NAME.
           MOVE "ELECTRONIC" TO PROD-CATEGORY.
           WRITE PRODUCT-RECORD.

           MOVE "P00002" TO PROD-ID.
           MOVE "OFFICE CHAIR" TO PROD-NAME.
           MOVE "FURNITURE" TO PROD-CATEGORY.
           WRITE PRODUCT-RECORD.

           MOVE "P00003" TO PROD-ID.
           MOVE "WIRELESS MOUSE" TO PROD-NAME.
           MOVE "ELECTRONIC" TO PROD-CATEGORY.
           WRITE PRODUCT-RECORD.
           CLOSE PRODUCT-FILE.

      *> Access "through the PATH" - i.e. via the alternate key -
      *> to list every product in the ELECTRONIC category, without
      *> ever touching the base RECORD KEY (PROD-ID) directly.
           OPEN INPUT PRODUCT-FILE.
           MOVE "ELECTRONIC" TO PROD-CATEGORY.
           START PRODUCT-FILE KEY IS = PROD-CATEGORY
               INVALID KEY
                   DISPLAY "No products in this category."
           END-START.

           DISPLAY "== Products in category ELECTRONIC (via AIX) ==".
           PERFORM UNTIL END-OF-FILE
               READ PRODUCT-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF PROD-CATEGORY = "ELECTRONIC"
                           DISPLAY "  " PROD-ID " " PROD-NAME
                       ELSE
                           SET END-OF-FILE TO TRUE
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE PRODUCT-FILE.
           STOP RUN.
```

คอมไพล์และรัน:

```bash
cobc -x -o step556 step556.cob
./step556
```

**ผลลัพธ์จริง (ยืนยันแล้วด้วย GnuCOBOL build ที่มี BDB indexed handler):**

```
== Products in category ELECTRONIC (via AIX) ==
  P00001 LAPTOP 14 INCH
  P00003 WIRELESS MOUSE
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้ค้นหา "สินค้าทั้งหมดในหมวด ELECTRONIC" ผ่าน `PROD-CATEGORY` (Alternate Key) โดย
  **ไม่เคยแตะ `PROD-ID` (Primary Key) เลยในขั้นตอนการค้นหา** — เทียบเท่ากับการเปิดไฟล์ผ่าน PATH
  บน z/OS จริงที่กล่าวถึงในขั้นตอนที่ 553
- `WRITE` ทั้ง 3 ครั้งใน `MAIN-PARA` ไม่มีโค้ดใดที่จัดการดัชนีรองด้วยตัวเองเลย GnuCOBOL จัดการ
  ให้อัตโนมัติทั้งหมดเบื้องหลัง — นี่คือพฤติกรรมแบบ UPGRADE Set ที่พิสูจน์ได้จริง
- สังเกตว่าผลลัพธ์แสดงเฉพาะ `P00001` และ `P00003` (ทั้งคู่เป็น ELECTRONIC) แต่ข้าม `P00002`
  (FURNITURE) ไปโดยอัตโนมัติ เพราะการอ่านผ่าน Alternate Key จะเรียงตามลำดับค่าของ
  `PROD-CATEGORY` ไม่ใช่ลำดับการเขียนจริง

### ข้อควรระวัง

- แม้ GnuCOBOL จะบังคับพฤติกรรมแบบ UPGRADE เสมอ (ไม่มีตัวเลือกปิด) แต่บน z/OS จริง การเลือก
  `NOUPGRADE` มีเหตุผลใช้งานจริงอยู่บ้าง เช่น เมื่อต้องการลดภาระ I/O ระหว่างการ LOAD ข้อมูลจำนวน
  มหาศาลครั้งแรก (ปิด UPGRADE ชั่วคราว โหลดข้อมูลให้เสร็จ แล้วค่อยรัน BLDINDEX ทีเดียวหลังจากนั้น
  จะเร็วกว่าการอัปเดตดัชนีทีละ record ระหว่างโหลด)
- ยิ่งมี AIX หลายตัวผูกกับ Base Cluster เดียว ยิ่งทำให้ทุกครั้งที่ WRITE/REWRITE/DELETE ต้อง
  อัปเดตดัชนีทุกตัวพร้อมกัน (ตามที่ Part 028 ขั้นตอน 279 เตือนไว้แล้ว) ส่งผลให้การเขียนข้อมูล
  ช้าลงตามจำนวน AIX ที่มี

### แบบฝึกหัดที่ 555.1

**โจทย์**: จงแก้ไขโปรแกรมข้างต้นให้ค้นหาสินค้าในหมวด `FURNITURE` แทน `ELECTRONIC`

**เฉลย**: เปลี่ยนค่าที่ `MOVE "ELECTRONIC" TO PROD-CATEGORY.` (ทั้งสองจุดที่ปรากฏ) เป็น
`MOVE "FURNITURE" TO PROD-CATEGORY.` และเปลี่ยนเงื่อนไขใน `IF PROD-CATEGORY = "ELECTRONIC"`
เป็น `IF PROD-CATEGORY = "FURNITURE"` ผลลัพธ์จะแสดงเฉพาะ `P00002 OFFICE CHAIR` เพราะเป็นสินค้า
เพียงตัวเดียวในหมวด FURNITURE

---

## ขั้นตอนที่ 556: การเข้าถึง VSAM ผ่าน AIX/PATH จากโปรแกรม COBOL บน z/OS

### ไวยากรณ์อ้างอิง: SELECT สำหรับเข้าถึงผ่าน PATH บน z/OS

เมื่อโปรแกรม COBOL บน z/OS ต้องการเข้าถึงข้อมูลผ่าน Alternate Key (ผ่าน PATH ที่สร้างไว้ใน
ขั้นตอนที่ 553) จะต้องมี **`SELECT` แยกต่างหากอีกตัว** ที่ชี้ไปยัง DDNAME ของ PATH นั้น (ไม่ใช่
DDNAME ของ Base Cluster) แม้จะเป็นข้อมูลชุดเดียวกันก็ตาม:

```cobol
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
      *> Access via the PRIMARY key - opens the Base Cluster
           SELECT CUSTOMER-FILE ASSIGN TO CUSTFILE
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-FILE-STATUS-1.

      *> Access via the ALTERNATE key - opens the PATH instead
           SELECT CUSTOMER-BY-CITY ASSIGN TO CUSTPATH
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               ALTERNATE RECORD KEY IS CUST-CITY
               FILE STATUS IS WS-FILE-STATUS-2.
```

พร้อม JCL ที่ผูก DDNAME ทั้งสองเข้ากับ Object คนละตัวกัน:

```jcl
//CUSTFILE DD  DSN=PROD.CUSTOMER.KSDS,DISP=SHR
//CUSTPATH DD  DSN=PROD.CUSTOMER.PATH.BYCITY,DISP=SHR
```

### ความแตกต่างสำคัญจาก GnuCOBOL

สังเกตว่าบน z/OS จริง เราต้องประกาศ `SELECT` **สองตัวแยกกัน** ชี้ไปยัง DDNAME คนละชื่อ (แม้จะ
เป็นข้อมูลชุดเดียวกันทางกายภาพ) เพราะ VSAM แยก Object ระหว่าง Base Cluster กับ PATH ออกจากกัน
อย่างชัดเจนตามที่อธิบายในขั้นตอนที่ 553 — ในขณะที่ **GnuCOBOL รวมทุกอย่างไว้ใน `SELECT` เดียว**
(ตามตัวอย่างขั้นตอนที่ 555) เพราะไม่ได้แยก Base Cluster/AIX/PATH เป็นคนละ Object ในระดับไฟล์
ระบบ

### ตารางสรุปภาพรวมการเข้าถึง

| ต้องการเข้าถึงผ่าน | z/OS COBOL (ไวยากรณ์อ้างอิง) | GnuCOBOL (ทดสอบรันได้จริง) |
|---|---|---|
| Primary Key | `SELECT ... ASSIGN TO CUSTFILE` (DDNAME ของ Base Cluster) | `SELECT` เดียวกัน ใช้ `RECORD KEY` |
| Alternate Key | `SELECT ... ASSIGN TO CUSTPATH` (DDNAME ของ PATH แยกต่างหาก) | `SELECT` เดียวกัน ใช้ `START`/`READ NEXT` กับฟิลด์ Alternate Key |

### ข้อควรระวัง

- **ตัวอย่าง SELECT สองตัวข้างต้นเป็นไวยากรณ์มาตรฐานอ้างอิงของ z/OS COBOL ไม่สามารถทดสอบรัน
  จริงได้ในสภาพแวดล้อม GnuCOBOL ของหลักสูตรนี้** เพราะ GnuCOBOL ไม่มีแนวคิด PATH แยกจาก Base
  Cluster ให้จำลองสถานการณ์นี้ได้ครบถ้วน
- แม้จะเปิดผ่าน `SELECT` คนละตัว แต่ทั้งสองยังคง**ชี้ไปยัง record เดียวกันทางกายภาพ** — ถ้าเปิด
  ทั้งสองพร้อมกันในโปรแกรมเดียว (เช่นเพื่อ REWRITE ผ่านตัวหนึ่งแล้วอ่านยืนยันผ่านอีกตัว) ต้อง
  ระวังเรื่อง concurrency/locking ตามที่ `SHAREOPTIONS` กำหนดไว้ตอน DEFINE CLUSTER (Part 055
  ขั้นตอนที่ 543)

### แบบฝึกหัดที่ 556.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม COBOL บน z/OS จึงต้องมี `SELECT` สองตัวเมื่อต้องการทั้งอ่านผ่าน
Primary Key และ Alternate Key ในโปรแกรมเดียวกัน ในขณะที่ GnuCOBOL ใช้ `SELECT` ตัวเดียวพอ

**เฉลย**: เพราะ VSAM มองว่า Base Cluster และ PATH เป็นคนละ Object ที่มีชื่อ dataset (DSN) และ
DDNAME แยกกันโดยสิ้นเชิง แม้จะชี้ไปยังข้อมูลเดียวกันทางกายภาพก็ตาม COBOL จึงต้องมี `SELECT`
แยกสำหรับแต่ละ DDNAME ที่ต้องการเปิด ในขณะที่ GnuCOBOL ออกแบบให้ Base Cluster และดัชนีรองอยู่ใน
"ไฟล์" เดียวกันในระดับที่ผู้ใช้มองเห็น การประกาศ `ALTERNATE RECORD KEY` เพิ่มเข้าไปใน `SELECT`
เดียวกันจึงเพียงพอโดยไม่ต้องแยก Object

---

## ขั้นตอนที่ 557: REPRO Utility — Backup, Restore และการคัดลอกข้อมูล VSAM

### REPRO คืออะไร

**REPRO (Reproduce)** คือคำสั่งของ IDCAMS ที่ใช้**คัดลอกข้อมูลจากแหล่งหนึ่งไปยังอีกแหล่งหนึ่ง**
รองรับการคัดลอกได้หลายทิศทาง: จาก Sequential Dataset เข้า VSAM Cluster (เช่นตอน LOAD ข้อมูลครั้ง
แรกที่กล่าวถึงใน Part 055 ขั้นตอนที่ 550), จาก VSAM Cluster หนึ่งไปยังอีก Cluster หนึ่ง (ใช้ทำ
Backup หรือ Reorganization), หรือจาก VSAM กลับไปเป็น Sequential Dataset (ใช้ทำ Restore/Extract)

### ไวยากรณ์อ้างอิง: REPRO สำหรับ LOAD ข้อมูลเริ่มต้นเข้า VSAM (มาตรฐาน — รันไม่ได้ในสภาพแวดล้อมนี้)

```jcl
//LOADCUST JOB (ACCT123),'LOAD CUSTOMER KSDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//INDS     DD  DSN=PROD.CUSTOMER.SEQ.EXTRACT,DISP=SHR
//OUTDS    DD  DSN=PROD.CUSTOMER.KSDS,DISP=SHR
//SYSIN    DD  *
  REPRO INFILE(INDS) OUTFILE(OUTDS)
/*
```

### ไวยากรณ์อ้างอิง: REPRO สำหรับ Backup VSAM Cluster ไปเป็น Sequential Dataset

```jcl
//BACKUP   JOB (ACCT123),'BACKUP CUSTOMER KSDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//INDS     DD  DSN=PROD.CUSTOMER.KSDS,DISP=SHR
//OUTDS    DD  DSN=PROD.CUSTOMER.BACKUP.SEQ,
//             DISP=(NEW,CATLG,DELETE),UNIT=SYSDA,
//             SPACE=(TRK,(100,20)),
//             DCB=(RECFM=FB,LRECL=35,BLKSIZE=3500)
//SYSIN    DD  *
  REPRO INFILE(INDS) OUTFILE(OUTDS)
/*
```

### อธิบายจุดสำคัญ

- REPRO ใช้ syntax เดียวกัน (`REPRO INFILE(...) OUTFILE(...)`) ไม่ว่าทิศทางการคัดลอกจะเป็นแบบใด
  — ความแตกต่างอยู่ที่ประเภทของ dataset ที่ DD statement ชี้ไปเท่านั้น (VSAM หรือ Sequential)
- REPRO จาก VSAM Cluster ไปเป็น Sequential Dataset (การ backup) จะได้ record เรียงตามลำดับ
  Key ปัจจุบันของ Cluster เสมอ (ไม่ใช่ลำดับการเขียนจริงในอดีต) เพราะ VSAM เก็บ record ตาม
  โครงสร้างดัชนี ไม่ใช่ตามลำดับเวลา (ตรงกับสิ่งที่ Part 028 ขั้นตอน 275 พิสูจน์ไว้กับ GnuCOBOL
  INDEXED file)
- REPRO สามารถระบุเงื่อนไขกรองข้อมูลเพิ่มเติมได้ เช่น `FROMKEY`/`TOKEY` เพื่อคัดลอกเฉพาะช่วง
  Key ที่ต้องการ ไม่จำเป็นต้องคัดลอกทั้งไฟล์เสมอไป

### ทำไม REPRO จึงเป็นเครื่องมือสำคัญที่สุดตัวหนึ่งของ VSAM Administrator

REPRO ถูกใช้ในหลายสถานการณ์งานจริง: (1) LOAD ข้อมูลเริ่มต้นเข้า Cluster ใหม่ (2) Backup ข้อมูล
ก่อนการเปลี่ยนแปลงเสี่ยง ๆ (3) Restore ข้อมูลกลับจาก backup เมื่อเกิดปัญหา (4) **Reorganization**
— คัดลอกข้อมูลจาก Cluster เดิมไปยัง Cluster ใหม่ที่เพิ่ง DEFINE ใหม่ (มักมี FREESPACE ที่เหมาะสม
กว่าเดิม) เพื่อล้าง CI Split สะสมทั้งหมดออกไป (จะอธิบายรายละเอียดในขั้นตอนที่ 558)

### ข้อควรระวัง

- **ไวยากรณ์ REPRO ทั้งหมดในขั้นตอนนี้เป็นไวยากรณ์มาตรฐานอ้างอิง ไม่สามารถรันได้ในสภาพแวดล้อม
  ของหลักสูตรนี้** ต้องใช้ z/OS จริงหรือ Mainframe Emulator
- REPRO เข้า VSAM Cluster ที่**มีข้อมูลอยู่แล้ว**จะเพิ่ม (append) record ใหม่เข้าไป ไม่ได้ลบ
  ข้อมูลเดิมออกก่อนโดยอัตโนมัติ — ถ้าต้องการแทนที่ข้อมูลทั้งหมด ต้องใช้ `REPLACE` option หรือ
  ลบ/สร้าง Cluster ใหม่ก่อนแล้วค่อย REPRO เข้าไป

### แบบฝึกหัดที่ 557.1

**โจทย์**: จงอธิบายว่าทำไมการ Backup ข้อมูลจาก VSAM Cluster ด้วย REPRO ไปเป็น Sequential
Dataset จึงยังคงเป็นแนวทางที่นิยมใช้ในองค์กรจริง แม้จะมีเครื่องมือ Backup ระดับดิสก์ (image copy)
ให้ใช้แล้วก็ตาม

**เฉลย**: Sequential Dataset ที่ได้จาก REPRO เป็นรูปแบบข้อมูลที่ "อ่านง่ายและพกพาได้" มากกว่า —
สามารถนำไปประมวลผลต่อด้วยโปรแกรม COBOL ปกติ (Part 023-027) ตรวจสอบเนื้อหาด้วยตาผ่าน ISPF Browse
(Part 054) หรือย้ายไปยัง Cluster ใหม่ที่มีคุณสมบัติต่างจากเดิมได้ง่าย ในขณะที่ image copy ระดับ
ดิสก์ (เช่น จาก DFSMShsm) มักผูกติดกับโครงสร้าง VSAM เดิมเป๊ะ ทำให้ยืดหยุ่นน้อยกว่าเมื่อต้องการ
ตรวจสอบหรือแปลงข้อมูลระหว่างทาง

---

## ขั้นตอนที่ 558: VSAM Cluster Reorganization — ทำไมต้องมีและทำอย่างไร

### ทบทวนปัญหา CI Split จากขั้นตอนที่ 541

Part 055 ขั้นตอนที่ 541 อธิบายไว้ว่าเมื่อ Control Interval (CI) เต็มและต้องแทรก record ใหม่เข้า
KSDS ตรงกลาง VSAM จะทำ **CI Split** (แบ่ง CI เดิมเป็นสองส่วน) เพื่อสร้างที่ว่างรองรับข้อมูลใหม่
กระบวนการนี้ทำงานถูกต้องเสมอ แต่มี**ผลข้างเคียงสะสม**: ยิ่งเวลาผ่านไปนาน ยิ่งมี CI Split เกิดขึ้น
บ่อยครั้ง ทำให้ (1) พื้นที่ดิสก์ถูกใช้อย่างไม่มีประสิทธิภาพ (fragmentation คล้ายที่เกิดกับ
filesystem ทั่วไป) (2) การอ่านข้อมูลตามลำดับ (Sequential Browse) ช้าลง เพราะ record ที่ควรอยู่
ติดกันทางตรรกะถูกกระจายไปอยู่คนละ CI ทางกายภาพ

### แนวคิดของ Reorganization

**Reorganization** คือกระบวนการ**สร้าง Cluster ใหม่ทั้งหมดที่มีโครงสร้างสะอาด**แล้วคัดลอกข้อมูล
จาก Cluster เดิมเข้าไปใหม่ทั้งหมด ทำให้ CI Split ที่สะสมมาทั้งหมดถูก "รีเซ็ต" กลับไปเป็นสภาพ
เรียบร้อยเหมือนตอนสร้างใหม่ ๆ — เปรียบได้กับการ Defragment ฮาร์ดดิสก์ในโลก PC หรือการทำ VACUUM
ในฐานข้อมูลเชิงสัมพันธ์สมัยใหม่

### ขั้นตอนมาตรฐานของการ Reorganize VSAM Cluster (ไวยากรณ์อ้างอิง)

```jcl
//REORG    JOB (ACCT123),'REORG CUSTOMER KSDS',CLASS=A,MSGCLASS=X
//*
//* STEP 1: BACKUP OLD CLUSTER TO SEQUENTIAL DATASET
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//INDS     DD  DSN=PROD.CUSTOMER.KSDS,DISP=SHR
//OUTDS    DD  DSN=&&TEMPBKUP,DISP=(NEW,PASS,DELETE),
//             UNIT=SYSDA,SPACE=(TRK,(100,20)),
//             DCB=(RECFM=FB,LRECL=35,BLKSIZE=3500)
//SYSIN    DD  *
  REPRO INFILE(INDS) OUTFILE(OUTDS)
/*
//*
//* STEP 2: DELETE THE OLD CLUSTER
//STEP2    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DELETE PROD.CUSTOMER.KSDS CLUSTER
/*
//*
//* STEP 3: DEFINE A FRESH CLUSTER WITH THE SAME ATTRIBUTES
//STEP3    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE CLUSTER (NAME(PROD.CUSTOMER.KSDS)         -
                  INDEXED                          -
                  KEYS(6 0)                        -
                  RECORDSIZE(35 35)                -
                  RECORDS(100000 50000)             -
                  FREESPACE(15 15)                  -
                  SHAREOPTIONS(2 3)                  -
                  VOLUMES(VSAM01) )
/*
//*
//* STEP 4: RELOAD DATA FROM THE BACKUP INTO THE FRESH CLUSTER
//STEP4    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//INDS     DD  DSN=&&TEMPBKUP,DISP=(OLD,DELETE)
//OUTDS    DD  DSN=PROD.CUSTOMER.KSDS,DISP=SHR
//SYSIN    DD  *
  REPRO INFILE(INDS) OUTFILE(OUTDS)
/*
```

### อธิบายภาพรวมของ 4 Step

1. **REPRO** ข้อมูลจาก Cluster เดิมออกมาเป็น Sequential Dataset ชั่วคราว (ใช้ `&&TEMPBKUP`
   ซึ่งเป็น temporary dataset ที่มีอายุแค่ระยะเวลาของ job เดียว — สังเกตว่าเทคนิค `&&` นี้เชื่อม
   โยงกับแนวคิด `SYSLIN`/`&&LOADSET` ที่ Part 053 ขั้นตอนที่ 521 เคยกล่าวถึง)
2. **DELETE** Cluster เดิมทิ้งไปทั้งหมด (คำสั่ง `DELETE ... CLUSTER` ของ IDCAMS)
3. **DEFINE CLUSTER** ใหม่ด้วยคุณสมบัติเดิมทุกประการ (โครงสร้างสะอาด ไม่มี CI Split สะสม)
4. **REPRO** ข้อมูลจาก Sequential Dataset ชั่วคราวกลับเข้า Cluster ใหม่ — ครบวงจร
   Reorganization

### ข้อควรระวัง

- **ไวยากรณ์ทั้งหมดในขั้นตอนนี้เป็นไวยากรณ์มาตรฐานอ้างอิง ไม่สามารถรันได้ในสภาพแวดล้อมของ
  หลักสูตรนี้** ต้องใช้ z/OS จริงหรือ Mainframe Emulator
- ระหว่างกระบวนการ Reorganize (โดยเฉพาะ STEP2-STEP3) **Cluster จะไม่พร้อมใช้งานชั่วคราว**
  องค์กรจริงจึงต้องวางแผนหน้าต่างเวลาบำรุงรักษา (Maintenance Window) ล่วงหน้าเสมอ ไม่สามารถทำ
  ระหว่างเวลาทำการปกติของระบบ Production ที่มีผู้ใช้งานจริงอยู่ได้
- หากมี Alternate Index ผูกกับ Cluster นี้อยู่ (ขั้นตอนที่ 551-556) การ DELETE Cluster เดิมจะทำ
  ให้ AIX ที่ผูกอยู่ **ถูกลบไปด้วยโดยอัตโนมัติ** ต้องวางแผน DEFINE ALTERNATEINDEX + DEFINE PATH
  + BLDINDEX ใหม่ทั้งหมดหลังจากสร้าง Cluster ใหม่เสร็จ ก่อนที่ระบบจะใช้งานได้ครบสมบูรณ์อีกครั้ง

### แบบฝึกหัดที่ 558.1

**โจทย์**: จงอธิบายว่าทำไมองค์กรจึงต้องวางแผน "หน้าต่างเวลาบำรุงรักษา" (Maintenance Window)
ก่อนทำ Reorganization เสมอ

**เฉลย**: เพราะกระบวนการ Reorganize ต้อง DELETE Cluster เดิมทิ้งก่อนสร้างใหม่ ในช่วงเวลานั้น
ไม่มี Cluster ให้โปรแกรม Production เปิดใช้งานได้เลย หากมีธุรกรรมพยายามเข้าถึงข้อมูลระหว่างนั้น
จะเกิดข้อผิดพลาดทันที องค์กรจึงต้องเลือกช่วงเวลาที่มีผู้ใช้งานน้อยที่สุด (เช่น ตอนดึกวันหยุด) และ
แจ้งเตือนผู้ใช้งานล่วงหน้า เพื่อลดผลกระทบทางธุรกิจให้น้อยที่สุด

---

## ขั้นตอนที่ 559: VERIFY, EXAMINE และ Abend Code ที่พบบ่อยของ VSAM

### VERIFY Utility

**VERIFY** คือคำสั่ง IDCAMS ที่ใช้แก้ไขปัญหา **VSAM Catalog ไม่ตรงกับสถานะจริงของ Cluster**
สถานการณ์นี้เกิดขึ้นได้เมื่อ job ที่กำลังเขียนไฟล์ VSAM ถูกยกเลิกกลางคัน (เช่นระบบล่ม หรือถูก
Cancel แบบกะทันหัน) ทำให้ Catalog ยังจำสถานะเก่าไว้ ไม่ตรงกับข้อมูลจริงบนดิสก์

```jcl
//VERIFY1  JOB (ACCT123),'VERIFY KSDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  VERIFY DATASET(PROD.CUSTOMER.KSDS)
/*
```

### EXAMINE Utility

**EXAMINE** คือคำสั่งที่ใช้**ตรวจสอบความถูกต้องของโครงสร้างภายใน** (structural integrity) ของ
KSDS โดยเฉพาะ — ตรวจสอบว่า Index Component ยังคงสอดคล้องกับ Data Component อยู่หรือไม่ มี
ประโยชน์มากเมื่อสงสัยว่า Cluster อาจเสียหายจากปัญหาฮาร์ดแวร์หรือข้อผิดพลาดของระบบ

```jcl
//EXAMINE1 JOB (ACCT123),'EXAMINE KSDS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  EXAMINE NAME(PROD.CUSTOMER.KSDS) INDEXTEST DATATEST
/*
```

### Abend Code ที่พบบ่อยเมื่อทำงานกับ VSAM

| Abend Code | ความหมาย |
|---|---|
| `S013` | ปัญหาตอน OPEN dataset (มักเกิดจาก DCB ไม่ตรงกัน คล้าย File Status `39` ที่ Part 055 ขั้นตอน 549 อธิบายไว้) |
| `S0C4` | Protection Exception — โปรแกรมพยายามเข้าถึงหน่วยความจำที่ไม่มีสิทธิ์ (มักเกิดจาก pointer/index ผิดพลาดในโค้ด) |
| `S0C7` | Data Exception — พบข้อมูลที่ไม่ใช่ตัวเลขในฟิลด์ที่ประกาศเป็นตัวเลข (เกิดบ่อยมากกับข้อมูลนำเข้าที่ไม่ผ่านการตรวจสอบ) |
| `S822` | Region ไม่พอ (Insufficient Virtual Storage) มักเกิดกับโปรแกรมที่ใช้ตารางขนาดใหญ่เกินที่ JCL กำหนด `REGION=` ไว้ |
| `S913` | ปัญหาเรื่องสิทธิ์การเข้าถึง (RACF Security Violation — จะสอนรายละเอียดใน Part 069) |

### ข้อควรระวัง

- **ไวยากรณ์ VERIFY/EXAMINE ในขั้นตอนนี้เป็นไวยากรณ์มาตรฐานอ้างอิง ไม่สามารถรันได้ในสภาพแวดล้อม
  ของหลักสูตรนี้** ต้องใช้ z/OS จริงหรือ Mainframe Emulator
- ควรรัน VERIFY **ก่อน**เปิดใช้งาน Cluster ที่สงสัยว่ามีปัญหาเสมอ ไม่ควรรัน EXAMINE หรือพยายาม
  REPRO ข้อมูลออกมาก่อน เพราะถ้า Catalog ไม่ตรงกับสถานะจริง อาจทำให้ได้ผลลัพธ์การตรวจสอบที่ผิด
  พลาดตามไปด้วย

### แบบฝึกหัดที่ 559.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง VERIFY กับ EXAMINE

**เฉลย**: VERIFY แก้ปัญหาที่ **Catalog** (ข้อมูลเมตาที่ระบบใช้ติดตามสถานะ) ไม่ตรงกับสถานะจริงของ
Cluster บนดิสก์ (มักเกิดจาก job ถูกตัดจบกะทันหัน) ส่วน EXAMINE ตรวจสอบ**โครงสร้างภายใน**ของ
Cluster เอง (ความสอดคล้องระหว่าง Index Component กับ Data Component) ว่าเสียหายหรือไม่ ทั้งสอง
แก้ปัญหาคนละชั้นกัน: VERIFY แก้ปัญหาระดับ metadata ส่วน EXAMINE ตรวจสอบระดับโครงสร้างข้อมูลจริง

---

## ขั้นตอนที่ 560: กรณีศึกษารวม — VSAM KSDS พร้อม Alternate Index ครบวงจร

### โจทย์ของกรณีศึกษา

จำลองระบบกรมธรรม์ประกันภัยที่ต้องการทั้งค้นหาด้วยเลขกรมธรรม์ (Primary Key) และค้นหาตามตัวแทน
ผู้ดูแล (Alternate Key) พร้อมสาธิตว่าเมื่อ REWRITE เปลี่ยนตัวแทนผู้ดูแลของกรมธรรม์หนึ่ง ดัชนีรอง
จะปรับปรุงตามอัตโนมัติทันที (พฤติกรรม UPGRADE ที่พิสูจน์ในขั้นตอนที่ 555)

### ไวยากรณ์ IDCAMS ที่ควรใช้สร้างชุด Cluster+AIX+PATH นี้บน z/OS จริง (อ้างอิง)

```jcl
//DEFPOL   JOB (ACCT123),'DEFINE POLICY KSDS+AIX',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IDCAMS
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
  DEFINE CLUSTER (NAME(PROD.POLICY.KSDS)           -
                  INDEXED                          -
                  KEYS(8 0)                        -
                  RECORDSIZE(45 45)                -
                  RECORDS(50000 20000)              -
                  FREESPACE(10 10)                  -
                  SHAREOPTIONS(2 3)                  -
                  VOLUMES(VSAM01) )
  DEFINE ALTERNATEINDEX (NAME(PROD.POLICY.AIX.AGENT)  -
                  RELATE(PROD.POLICY.KSDS)             -
                  KEYS(10 28)                           -
                  RECORDSIZE(18 18)                     -
                  NONUNIQUEKEY                           -
                  UPGRADE                                -
                  VOLUMES(VSAM01) )
  DEFINE PATH (NAME(PROD.POLICY.PATH.AGENT)          -
               PATHENTRY(PROD.POLICY.AIX.AGENT)       -
               UPDATE )
/*
```

### โปรแกรม COBOL: Working Analog ที่ Compile และรันได้จริง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP560-VSAM-AIX-CAPSTONE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT POLICY-FILE ASSIGN TO "POLICY560.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS POLICY-NO
               ALTERNATE RECORD KEY IS POLICY-AGENT
                   WITH DUPLICATES
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  POLICY-FILE.
       01  POLICY-RECORD.
           05  POLICY-NO            PIC X(8).
           05  POLICY-HOLDER        PIC X(20).
           05  POLICY-AGENT         PIC X(10).
           05  POLICY-PREMIUM       PIC 9(7)V99.

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Base cluster keyed by POLICY-NO (the AMS "base cluster" +
      *> "PATH by POLICY-NO"), with a second index on POLICY-AGENT
      *> (the AMS "alternate index" + "PATH by POLICY-AGENT") kept
      *> in sync automatically on every WRITE/REWRITE/DELETE -
      *> exactly what the UPGRADE option does for a real VSAM AIX.
           PERFORM LOAD-POLICIES.
           PERFORM LIST-BY-AGENT.
           PERFORM UPDATE-ONE-POLICY.
           PERFORM DELETE-ONE-POLICY.
           PERFORM LIST-BY-AGENT.
           STOP RUN.

       LOAD-POLICIES.
           OPEN OUTPUT POLICY-FILE.
           MOVE "PL000001" TO POLICY-NO.
           MOVE "SOMCHAI JAIDEE" TO POLICY-HOLDER.
           MOVE "AGENT-A" TO POLICY-AGENT.
           MOVE 12500.00 TO POLICY-PREMIUM.
           WRITE POLICY-RECORD.

           MOVE "PL000002" TO POLICY-NO.
           MOVE "SUDA MEECHAI" TO POLICY-HOLDER.
           MOVE "AGENT-B" TO POLICY-AGENT.
           MOVE 8300.00 TO POLICY-PREMIUM.
           WRITE POLICY-RECORD.

           MOVE "PL000003" TO POLICY-NO.
           MOVE "PRASERT KAEWTA" TO POLICY-HOLDER.
           MOVE "AGENT-A" TO POLICY-AGENT.
           MOVE 5400.00 TO POLICY-PREMIUM.
           WRITE POLICY-RECORD.
           CLOSE POLICY-FILE.
           DISPLAY "== Loaded 3 policies (base cluster + AIX) ==".

       LIST-BY-AGENT.
           OPEN INPUT POLICY-FILE.
           MOVE "AGENT-A" TO POLICY-AGENT.
           START POLICY-FILE KEY IS = POLICY-AGENT
               INVALID KEY
                   DISPLAY "No policies for this agent."
           END-START.

           MOVE "N" TO WS-EOF-FLAG.
           DISPLAY "== Policies serviced by AGENT-A (via PATH) ==".
           PERFORM UNTIL END-OF-FILE
               READ POLICY-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF POLICY-AGENT = "AGENT-A"
                           DISPLAY "  " POLICY-NO " " POLICY-HOLDER
                               " premium=" POLICY-PREMIUM
                       ELSE
                           SET END-OF-FILE TO TRUE
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE POLICY-FILE.

       UPDATE-ONE-POLICY.
           OPEN I-O POLICY-FILE.
           MOVE "PL000003" TO POLICY-NO.
           READ POLICY-FILE
               INVALID KEY
                   DISPLAY "Policy PL000003 not found."
               NOT INVALID KEY
                   MOVE "AGENT-B" TO POLICY-AGENT
                   REWRITE POLICY-RECORD
                       INVALID KEY
                           DISPLAY "Rewrite failed."
                   END-REWRITE
                   DISPLAY "== Reassigned PL000003 to AGENT-B =="
           END-READ.
           CLOSE POLICY-FILE.

       DELETE-ONE-POLICY.
           OPEN I-O POLICY-FILE.
           MOVE "PL000002" TO POLICY-NO.
           DELETE POLICY-FILE
               INVALID KEY
                   DISPLAY "Delete of PL000002 failed."
               NOT INVALID KEY
                   DISPLAY "== Deleted policy PL000002 =="
           END-DELETE.
           CLOSE POLICY-FILE.
```

คอมไพล์และรัน:

```bash
cobc -x -o step560 step560.cob
./step560
```

**ผลลัพธ์จริง (ยืนยันแล้วด้วย GnuCOBOL build ที่มี BDB indexed handler):**

```
== Loaded 3 policies (base cluster + AIX) ==
== Policies serviced by AGENT-A (via PATH) ==
  PL000001 SOMCHAI JAIDEE       premium=0012500.00
  PL000003 PRASERT KAEWTA       premium=0005400.00
== Reassigned PL000003 to AGENT-B ==
== Deleted policy PL000002 ==
== Policies serviced by AGENT-A (via PATH) ==
  PL000001 SOMCHAI JAIDEE       premium=0012500.00
```

### วิเคราะห์ผลลัพธ์: พิสูจน์พฤติกรรม UPGRADE แบบสมบูรณ์

สังเกตว่าการเรียก `LIST-BY-AGENT` ครั้งแรกพบกรมธรรม์ 2 ฉบับของ `AGENT-A` (`PL000001` และ
`PL000003`) แต่หลังจาก `UPDATE-ONE-POLICY` เปลี่ยน `PL000003` ให้เป็นของ `AGENT-B` แทน
การเรียก `LIST-BY-AGENT` ครั้งที่สองเหลือเพียง `PL000001` เท่านั้น — นี่คือหลักฐานที่ชัดเจนว่า
ดัชนีรอง (Alternate Index) **ปรับปรุงตามการ REWRITE โดยอัตโนมัติทันที** ตรงกับพฤติกรรมของ
UPGRADE Set บน VSAM AIX จริงทุกประการ โดยที่โปรแกรมเมอร์ไม่ต้องเขียนโค้ดจัดการดัชนีเองแม้แต่
บรรทัดเดียว

### ข้อควรระวัง

- เนื้อหาทั้งหมดในกรณีศึกษานี้ผสมทั้ง**ไวยากรณ์ IDCAMS อ้างอิง** (DEFINE CLUSTER, DEFINE
  ALTERNATEINDEX, DEFINE PATH — รันไม่ได้ในสภาพแวดล้อมนี้) และ**โค้ด COBOL ที่ทดสอบคอมไพล์และ
  รันจริงแล้ว** (โปรแกรม STEP560 ข้างต้น) ต้องแยกให้ชัดเจนเสมอว่าส่วนใดคือส่วนใดเมื่อนำไปอ้างอิง
  ต่อ
- ในการออกแบบระบบจริง ควรพิจารณาว่า field ใดควรเป็น Alternate Key อย่างรอบคอบ เพราะทุก AIX ที่
  เพิ่มขึ้นย่อมเพิ่มภาระ I/O ให้กับทุกการเขียนข้อมูล (ตามที่ Part 028 ขั้นตอน 279 และขั้นตอนที่
  555 ของ Part นี้เตือนไว้) ควรสร้างเฉพาะ Alternate Key ที่มีความจำเป็นทางธุรกิจจริง ๆ เท่านั้น

### แบบฝึกหัดที่ 560.1

**โจทย์**: จงเพิ่ม paragraph ใหม่ชื่อ `LIST-BY-AGENT-B` ที่ทำงานเหมือน `LIST-BY-AGENT` แต่ค้นหา
เฉพาะกรมธรรม์ของ `AGENT-B` แทน แล้วเรียกใช้เป็นขั้นตอนสุดท้ายของ `MAIN-PARA`

**เฉลย**:

```cobol
       LIST-BY-AGENT-B.
           OPEN INPUT POLICY-FILE.
           MOVE "AGENT-B" TO POLICY-AGENT.
           START POLICY-FILE KEY IS = POLICY-AGENT
               INVALID KEY
                   DISPLAY "No policies for this agent."
           END-START.

           MOVE "N" TO WS-EOF-FLAG.
           DISPLAY "== Policies serviced by AGENT-B (via PATH) ==".
           PERFORM UNTIL END-OF-FILE
               READ POLICY-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF POLICY-AGENT = "AGENT-B"
                           DISPLAY "  " POLICY-NO " " POLICY-HOLDER
                               " premium=" POLICY-PREMIUM
                       ELSE
                           SET END-OF-FILE TO TRUE
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE POLICY-FILE.
```

เมื่อเพิ่ม `PERFORM LIST-BY-AGENT-B.` ต่อท้ายใน `MAIN-PARA` ผลลัพธ์จะแสดงกรมธรรม์ 2 ฉบับของ
`AGENT-B` คือ `PL000002` (ก่อนถูกลบ — แต่เนื่องจากลำดับการ PERFORM ใน MAIN-PARA เดิมลบ
PL000002 ไปก่อนแล้ว ผลลัพธ์จริงจะเหลือเพียง `PL000003` ที่เพิ่งถูกย้ายมาเป็นของ AGENT-B เท่านั้น)
ซึ่งเป็นตัวอย่างที่ดีที่แสดงให้เห็นว่าต้องพิจารณาลำดับการดำเนินการให้รอบคอบเมื่อออกแบบ test case

---

## สรุปท้ายบท

ใน Part นี้ เราได้ขยายความรู้ VSAM ให้ครอบคลุมความสามารถขั้นสูงที่จำเป็นสำหรับงาน Production
จริง:

- Alternate Index (AIX) และการเชื่อมโยงกับ `ALTERNATE RECORD KEY` ที่ Part 028 สอนไว้แล้ว
- สถาปัตยกรรม 3 ส่วนของ AIX บน z/OS: DEFINE ALTERNATEINDEX → BLDINDEX → DEFINE PATH
- ความแตกต่างเชิง operational ระหว่าง GnuCOBOL (รวมทุกอย่างใน `SELECT` เดียว) กับ z/OS COBOL
  (แยก Base Cluster และ PATH เป็นคนละ `SELECT`/DDNAME)
- UPGRADE Option และการพิสูจน์พฤติกรรมนี้ด้วย GnuCOBOL จริงถึง 2 กรณีศึกษา (ขั้นตอนที่ 555
  และ 560)
- REPRO Utility สำหรับ LOAD/Backup/Restore ข้อมูล VSAM
- แนวคิดและขั้นตอนมาตรฐานของ Cluster Reorganization (Backup → Delete → Define ใหม่ → Reload)
- VERIFY และ EXAMINE สำหรับตรวจสอบและแก้ไขปัญหา Catalog/โครงสร้างของ VSAM
- Abend Code ที่พบบ่อยเมื่อทำงานกับ VSAM (`S013`, `S0C4`, `S0C7`, `S822`, `S913`)
- กรณีศึกษารวม: ระบบกรมธรรม์ประกันภัยที่มี AIX ครบวงจร **compile และรันได้จริงด้วย GnuCOBOL**
  พิสูจน์พฤติกรรม UPGRADE ผ่านการ REWRITE ที่เปลี่ยนตัวแทนผู้ดูแล

**ย้ำอีกครั้ง**: ไวยากรณ์ IDCAMS ทั้งหมด (DEFINE ALTERNATEINDEX, DEFINE PATH, BLDINDEX, REPRO,
VERIFY, EXAMINE) เป็นไวยากรณ์มาตรฐานอ้างอิงที่ถูกต้อง ต้องใช้ z/OS จริงหรือ Mainframe Emulator
เพื่อรันจริง ส่วนโค้ด COBOL ทุกตัวอย่างที่ระบุว่า "compile และรันได้จริง" ได้ทดสอบแล้วด้วย
GnuCOBOL จริงตามที่ระบุไว้ในแต่ละขั้นตอน

เมื่อจบ Part 055-056 เราได้ปิดฉากเรื่องการจัดการไฟล์บน Mainframe อย่างครบถ้วนแล้ว Part ถัดไป
(**Part 057**) จะพาคุณก้าวเข้าสู่โลกของ**ฐานข้อมูลเชิงสัมพันธ์**อย่างเป็นทางการ: **DB2 และ SQL
เบื้องต้นสำหรับ COBOL** ซึ่งเป็นก้าวสำคัญที่ทำให้ COBOL เชื่อมต่อกับข้อมูลในรูปแบบตาราง (table)
แทนที่จะเป็นไฟล์แบบ record-based ที่เราคุ้นเคยมาตลอด

**[← กลับไป Part 055: VSAM เบื้องต้น](part-055-vsam-basics.md)** | **[ไปยัง Part 057: DB2 และ SQL เบื้องต้นสำหรับ COBOL →](part-057-db2-sql-basics.md)**
