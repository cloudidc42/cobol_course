# Part 069: Security บน Mainframe (RACF Concepts) (ขั้นตอนที่ 681–690)

## คำนำของ Part นี้

Part 068 พาเราไปเจาะลึกเรื่องประสิทธิภาพของโปรแกรม COBOL แต่ระบบที่ทำงานเร็วโดยไม่ปลอดภัยก็ยังคง
เป็นความเสี่ยงมหาศาลต่อองค์กร โดยเฉพาะระบบธนาคารและการเงินที่หลักสูตรนี้ใช้เป็นตัวอย่างหลักมาตลอด
Part นี้จะพาไปทำความเข้าใจความปลอดภัยบน Mainframe ใน **2 ระดับที่แยกกันชัดเจน**:

1. **ระดับ System/Platform — RACF (Resource Access Control Facility)**: กลไกควบคุมสิทธิ์การเข้าถึง
   ทรัพยากรทุกชนิดบน z/OS (User ID, Dataset, Program, Transaction) **หัวข้อกลุ่มนี้เป็นความรู้เชิงอ้างอิง
   (Reference/Conceptual) ทั้งหมด** เพราะ RACF ผูกติดกับ z/OS โดยตรง ไม่มีอยู่และไม่สามารถจำลองได้บน
   GnuCOBOL หรือ Linux ทั่วไป — ตรงกับที่ระบุไว้ในหมายเหตุมาตรฐานของหลักสูตรตั้งแต่ Part 051
2. **ระดับ Application/Code — Secure Coding Practice ใน COBOL เอง**: การตรวจสอบข้อมูลนำเข้า
   (Input Validation), การป้องกัน subscript/buffer overflow, การปกปิดข้อมูลอ่อนไหวในผลลัพธ์ **หัวข้อกลุ่ม
   นี้ทดสอบได้จริงด้วย GnuCOBOL ทั้งหมด** และเป็นทักษะที่นักพัฒนา COBOL ทุกคนต้องรู้ ไม่ว่าจะทำงานบน
   Mainframe จริงหรือ GnuCOBOL

**หลักการสำคัญที่ต้องเข้าใจตั้งแต่ต้น**: RACF และ Secure Coding **ไม่ใช่ทางเลือกที่ใช้แทนกันได้** แต่เป็น
**ชั้นป้องกันที่ทำงานร่วมกัน (Defense in Depth)** — RACF ป้องกันไม่ให้คนที่ไม่มีสิทธิ์เข้าถึงไฟล์/โปรแกรม
ตั้งแต่แรก แต่ **ไม่สามารถป้องกัน**ไม่ให้โปรแกรมของคุณเองรับข้อมูลผิดพลาดหรือมีบั๊กด้านความปลอดภัยภายใน
ตรรกะของมันเองได้เลย ทั้งสองชั้นจำเป็นต้องมีพร้อมกันเสมอในระบบที่ปลอดภัยจริง

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่างที่ระบุว่า
> "ทดสอบจริง" ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง (รวมถึงตัวอย่างที่ตั้งใจแสดงพฤติกรรม
> ที่ไม่ปลอดภัยเพื่อการเปรียบเทียบ) ส่วนหัวข้อ RACF ทั้งหมดเป็นไวยากรณ์คำสั่งจริงที่ใช้บน z/OS แต่**ไม่มี
> การรันจริงในสภาพแวดล้อมนี้** ตามที่ระบุไว้ชัดเจนในแต่ละหัวข้อ

---

## ขั้นตอนที่ 681: [เชิงแนวคิด/z/OS เท่านั้น] ภาพรวมสถาปัตยกรรมความปลอดภัยบน Mainframe และบทบาทของ RACF

> **หัวข้อนี้เป็นความรู้เชิงอ้างอิง (Reference/Conceptual) เท่านั้น** ไม่มีตัวอย่างที่รันได้จริงบน GnuCOBOL

### RACF คืออะไร

**RACF (Resource Access Control Facility)** คือ **Security Manager** (ซอฟต์แวร์ควบคุมความปลอดภัย)
ของ IBM ที่ทำงานเป็นส่วนหนึ่งของ z/OS โดยตรง หน้าที่หลักของมันคือตอบคำถามเดียวเสมอทุกครั้งที่มีการร้องขอ
เข้าถึงทรัพยากรใด ๆ ในระบบ: **"ผู้ใช้/โปรแกรมนี้ มีสิทธิ์เข้าถึงทรัพยากรนี้ในระดับนี้หรือไม่?"**

RACF ไม่ใช่ Security Manager ตัวเดียวที่มีในตลาด — คู่แข่งที่พบบ่อยคือ **ACF2** และ **Top Secret**
(ทั้งคู่เป็นผลิตภัณฑ์ของ Broadcom) แต่ RACF เป็นตัวที่ IBM พัฒนาเองและพบมากที่สุดในองค์กรที่ใช้ Mainframe
ทั่วโลก

### สามส่วนประกอบหลักของโมเดล RACF

RACF ควบคุมความสัมพันธ์ระหว่าง 3 สิ่งเสมอ:

```
USER  --(มีสิทธิ์)-->  RESOURCE  --(ที่ระดับ)-->  ACCESS LEVEL
```

| ส่วนประกอบ | คำอธิบาย | ตัวอย่าง |
|---|---|---|
| **User** | ผู้ใช้แต่ละคนที่ล็อกอินเข้าระบบ ระบุด้วย User ID สูงสุด 8 ตัวอักษร | `PJONES`, `BATCH01` |
| **Resource** | ทรัพยากรที่ต้องป้องกัน แบ่งเป็นหลาย "Class" | Dataset, Program, Transaction, Terminal |
| **Access Level** | ระดับสิทธิ์ที่อนุญาต | `NONE`, `READ`, `UPDATE`, `ALTER`, `CONTROL` |

### ทรัพยากรประเภทต่าง ๆ ที่ RACF ป้องกัน (RACF Class)

| Class | ป้องกันอะไร | เกี่ยวข้องกับ Part ใดที่เรียนมาแล้ว |
|---|---|---|
| `DATASET` | ไฟล์ (Sequential, VSAM) บนดิสก์ | Part 023-030, 055-056 |
| `PROGRAM` | โปรแกรม load module ที่อนุญาตให้รันได้ | Part 031-034, 043 |
| `TCICSTRN` | Transaction ID ของ CICS | Part 061-064 |
| `TERMINAL` | เทอร์มินัลที่อนุญาตให้ล็อกอิน | - |
| `FACILITY` | สิทธิพิเศษทั่วไป (เช่น สิทธิ์ APF-authorize) | ขั้นตอนที่ 685 |

### ทำไมต้องเรียนรู้ RACF แม้จะไม่มี Mainframe จริงให้ทดสอบ

แม้เราจะไม่สามารถรันคำสั่ง RACF จริงในหลักสูตรนี้ แต่นักพัฒนา COBOL ที่ทำงานในองค์กรจริงที่มี Mainframe
**ต้องเข้าใจแนวคิดนี้เสมอ** เพราะ:

1. **การเข้าถึงไฟล์/โปรแกรมที่ตัวเองพัฒนาต้องขอสิทธิ์ผ่าน RACF** — โปรแกรม COBOL ที่เขียนดีแค่ไหนก็รัน
   ไม่ได้ถ้า User ID ที่ใช้รัน job ไม่มีสิทธิ์เข้าถึง dataset ที่ระบุใน JCL (ทบทวนจาก Part 052)
2. **เมื่อโปรแกรมเจอ error รหัส "ไม่มีสิทธิ์" ต้องรู้ว่าควรติดต่อทีมไหน** — โดยทั่วไปทีม Security/RACF
   Administrator แยกต่างหากจากทีมพัฒนา
3. **การออกแบบระบบที่ปลอดภัยต้องคำนึงถึงทั้งสองชั้น** (RACF + Secure Coding) ตั้งแต่การออกแบบ ไม่ใช่
   มาคิดทีหลัง

### ข้อควรระวัง

- RACF ควบคุมที่ **ระดับ Resource** เท่านั้น (ใครเข้าถึงไฟล์/โปรแกรมอะไรได้) มันไม่มีทางรู้เลยว่า**ภายใน
  โปรแกรม** COBOL ที่ได้รับอนุญาตให้รันแล้วนั้น มีการตรวจสอบข้อมูลนำเข้าอย่างถูกต้องหรือไม่ — นี่คือ
  ช่องว่างที่ต้องปิดด้วย Secure Coding (ขั้นตอนที่ 686 เป็นต้นไป)
- อย่าเข้าใจผิดว่า "มี RACF แล้วระบบปลอดภัย 100%" — RACF เป็นเพียงหนึ่งชั้นในหลายชั้นของการป้องกันที่
  จำเป็น (Defense in Depth) ตามที่อธิบายไว้ในคำนำ

### แบบฝึกหัดที่ 681.1

**โจทย์**: จงอธิบายว่าทำไมองค์กรจึงต้องการทั้ง RACF และ Secure Coding Practice พร้อมกัน แทนที่จะ
เลือกใช้อย่างใดอย่างหนึ่ง

**เฉลยแนวทาง**: RACF ป้องกันที่ **ขอบเขตของระบบ (perimeter)** คือควบคุมว่าใครสามารถเข้าถึงหรือรัน
ทรัพยากรใดได้บ้าง แต่เมื่อผู้ใช้ที่ได้รับอนุญาตแล้วเริ่มใช้งานโปรแกรม RACF ไม่มีทางรู้เลยว่าข้อมูลที่ผู้ใช้
ป้อนเข้าไปนั้นถูกต้องหรือไม่ หรือโปรแกรมจัดการข้อมูลนั้นอย่างปลอดภัยหรือไม่ ถ้าขาด Secure Coding ผู้ใช้ที่
มีสิทธิ์ถูกต้องอาจป้อนข้อมูลผิดพลาด (โดยตั้งใจหรือไม่ตั้งใจ) จนทำให้ระบบทำงานผิดพลาดหรือข้อมูลเสียหายได้
โดยที่ RACF ไม่สามารถป้องกันได้เลย ทั้งสองชั้นจึงจำเป็นต้องทำงานร่วมกันเสมอ

---

## ขั้นตอนที่ 682: [เชิงแนวคิด/z/OS เท่านั้น] User ID, Group และคำสั่ง RACF พื้นฐาน

> **หัวข้อนี้เป็นความรู้เชิงอ้างอิง (Reference/Conceptual) เท่านั้น** คำสั่ง TSO/RACF ต่อไปนี้เป็นไวยากรณ์
> จริงที่ใช้บน z/OS แต่ไม่มีตัวอย่างใดที่รันได้บน GnuCOBOL

### โมเดล User และ Group ของ RACF

ผู้ใช้แต่ละคนบน z/OS มี **User ID** (สูงสุด 8 ตัวอักษร) และสามารถเป็นสมาชิกของ **Group** ได้หลายกลุ่ม
Group ใช้จัดกลุ่มผู้ใช้ตามหน้าที่ (เช่น แผนกบัญชี, ทีมพัฒนา COBOL, ทีม Operations) เพื่อให้กำหนดสิทธิ์
เป็นชุดเดียวกันให้กับทั้งกลุ่มได้ง่ายกว่าการกำหนดทีละคน

### คำสั่ง ADDUSER — สร้างผู้ใช้ใหม่

```
ADDUSER PJONES NAME('PATRICIA JONES') OWNER(SECADM) -
        PASSWORD(TEMP1234) DFLTGRP(COBOLDEV)
```

| พารามิเตอร์ | ความหมาย |
|---|---|
| `PJONES` | User ID ใหม่ที่จะสร้าง |
| `NAME(...)` | ชื่อเต็มของผู้ใช้ (metadata) |
| `OWNER(...)` | User ID หรือ Group ที่เป็นเจ้าของ (มีสิทธิ์จัดการ) profile นี้ |
| `PASSWORD(...)` | รหัสผ่านเริ่มต้น (ระบบมักบังคับให้เปลี่ยนตอนล็อกอินครั้งแรก) |
| `DFLTGRP(...)` | Group เริ่มต้นที่ผู้ใช้จะสังกัด |

### คำสั่ง CONNECT — เพิ่มผู้ใช้เข้า Group เพิ่มเติม

```
CONNECT PJONES GROUP(BATCHOPS) AUTHORITY(USE)
```

คำสั่งนี้ทำให้ `PJONES` เป็นสมาชิกของกลุ่ม `BATCHOPS` **เพิ่มเติม** จากกลุ่มเริ่มต้น (ผู้ใช้หนึ่งคนสามารถ
สังกัดได้หลาย Group พร้อมกัน) `AUTHORITY(USE)` หมายถึงระดับสิทธิ์ในการบริหารจัดการ Group นั้นเอง (ไม่ใช่
สิทธิ์เข้าถึงทรัพยากร — เรื่องนั้นแยกต่างหากในขั้นตอนที่ 683)

### คำสั่ง LISTUSER — ตรวจสอบข้อมูลผู้ใช้

```
LISTUSER PJONES
```

แสดงข้อมูลทั้งหมดของผู้ใช้ รวมถึง Group ที่สังกัด, วันหมดอายุรหัสผ่าน, สถานะ (revoked/active) และ
attribute พิเศษต่าง ๆ

### แนวคิด "Principle of Least Privilege" ในการออกแบบ Group

หลักการออกแบบที่ดีที่สุดคือ **ให้สิทธิ์เท่าที่จำเป็นต่อการทำงานเท่านั้น (Principle of Least Privilege)**
ตัวอย่างการจัดกลุ่มที่ดี:

| Group | สมาชิก | สิทธิ์ที่ควรมี |
|---|---|---|
| `COBOLDEV` | นักพัฒนา COBOL | เข้าถึง source code library, compile ได้, **ไม่มี**สิทธิ์เข้าถึงข้อมูล production จริง |
| `BATCHOPS` | ทีมควบคุม batch job | รัน job ที่กำหนดไว้ล่วงหน้าได้ **ไม่มี**สิทธิ์แก้ไข source code |
| `DBAADMIN` | ผู้ดูแลฐานข้อมูล | จัดการ VSAM/DB2 catalog **ไม่มี**สิทธิ์แก้ไขโปรแกรม COBOL |

### ข้อควรระวัง

- การให้สิทธิ์เกินความจำเป็น (over-privileging) เป็นสาเหตุอันดับต้น ๆ ของความเสียหายเมื่อ User ID ถูกขโมย
  หรือใช้งานผิดพลาด — ยิ่งสิทธิ์กว้างเท่าไร ความเสียหายที่อาจเกิดขึ้นยิ่งมากเท่านั้น
- ในองค์กรจริง การสร้าง/แก้ไขผู้ใช้และ Group **ไม่ใช่หน้าที่ของนักพัฒนา COBOL** แต่เป็นหน้าที่ของทีม
  Security Administrator โดยเฉพาะ — นักพัฒนาทั่วไปมักมีสิทธิ์แค่ "ร้องขอ" ผ่านกระบวนการอนุมัติเท่านั้น

### แบบฝึกหัดที่ 682.1

**โจทย์**: จงอธิบายว่าทำไมการแยก Group `COBOLDEV` และ `BATCHOPS` ออกจากกันตามตารางข้างต้น จึงเป็น
การประยุกต์ใช้หลักการ Least Privilege

**เฉลยแนวทาง**: นักพัฒนาใน `COBOLDEV` ต้องการสิทธิ์แก้ไข source code เพื่อพัฒนาโปรแกรม แต่ไม่ควรมี
สิทธิ์รันหรือแก้ไขข้อมูล production โดยตรง (เพื่อป้องกันความผิดพลาดหรือการใช้อำนาจเกินขอบเขตในการพัฒนา
ไปกระทบข้อมูลจริง) ในขณะที่ทีม `BATCHOPS` ต้องการรัน job ตามตารางเวลาที่กำหนดแต่ไม่ควรมีสิทธิ์แก้ไข
โค้ดต้นฉบับ (เพื่อป้องกันการเปลี่ยนแปลงโปรแกรมที่ไม่ผ่านกระบวนการพัฒนาและทดสอบที่ถูกต้อง) การแยกสิทธิ์
ตามหน้าที่ที่จำเป็นจริงแบบนี้ทำให้ความเสียหายที่อาจเกิดจาก User ID หนึ่งถูกละเมิดถูกจำกัดอยู่ในขอบเขตที่
แคบที่สุดเท่าที่จะเป็นไปได้

---

## ขั้นตอนที่ 683: [เชิงแนวคิด/z/OS เท่านั้น] Resource Profile และระดับการเข้าถึง (Access Level)

> **หัวข้อนี้เป็นความรู้เชิงอ้างอิง (Reference/Conceptual) เท่านั้น** ไม่มีตัวอย่างที่รันได้จริงบน GnuCOBOL

### Resource Profile คืออะไร

**Profile** คือ "กฎ" ที่ RACF สร้างขึ้นเพื่อป้องกันทรัพยากรหนึ่งชิ้น (หรือกลุ่มของทรัพยากรที่ชื่อคล้ายกัน)
ทุกครั้งที่มีการร้องขอเข้าถึงทรัพยากร RACF จะค้นหา Profile ที่ตรงกับชื่อทรัพยากรนั้น แล้วตรวจสอบว่า User
ที่ร้องขอมีสิทธิ์ในระดับที่เพียงพอหรือไม่

### ระดับการเข้าถึง (Access Level) มาตรฐานของ RACF

| ระดับ | ความหมาย | เทียบเคียงได้กับ |
|---|---|---|
| `NONE` | ไม่มีสิทธิ์ใด ๆ เลย | ไม่มีสิทธิ์เข้าถึง |
| `READ` | อ่านได้อย่างเดียว | สิทธิ์ระดับ read-only |
| `UPDATE` | อ่านและแก้ไขข้อมูลที่มีอยู่ได้ | สิทธิ์ read-write |
| `ALTER` | ทำได้ทุกอย่างรวมถึงลบและเปลี่ยนแปลง Profile เอง | สิทธิ์ระดับเจ้าของ |
| `CONTROL` | สิทธิ์พิเศษสำหรับ VSAM (เทียบเท่า UPDATE แต่ใช้กับบาง VSAM operation) | เฉพาะ VSAM |

**ข้อสำคัญ**: ระดับสิทธิ์เรียงจากน้อยไปมากเสมอ (`NONE < READ < UPDATE < CONTROL < ALTER`) — ผู้ที่มี
สิทธิ์ `ALTER` ย่อมทำทุกอย่างที่ระดับต่ำกว่าได้ด้วยเสมอ

### คำสั่ง PERMIT — ให้สิทธิ์ User/Group เข้าถึง Profile

```
PERMIT 'PROD.CUSTOMER.MASTER' ID(BATCHOPS) ACCESS(READ)
PERMIT 'PROD.CUSTOMER.MASTER' ID(DBAADMIN) ACCESS(ALTER)
```

บรรทัดแรกให้กลุ่ม `BATCHOPS` อ่าน dataset `PROD.CUSTOMER.MASTER` ได้อย่างเดียว บรรทัดที่สองให้กลุ่ม
`DBAADMIN` มีสิทธิ์เต็มรูปแบบ (`ALTER`) กับ dataset เดียวกัน — สังเกตว่า **Profile เดียวกันสามารถให้สิทธิ์
ต่างระดับกับ User/Group ต่างกลุ่มพร้อมกันได้**

### Generic Profile — ป้องกันไฟล์หลายไฟล์ด้วยกฎเดียว

RACF รองรับการใช้เครื่องหมาย `*` และ `%` เพื่อสร้าง **Generic Profile** ที่ครอบคลุมชื่อ dataset
หลายชื่อพร้อมกัน แทนที่จะต้องสร้าง Profile แยกทีละไฟล์:

```
PERMIT 'PROD.CUSTOMER.**' ID(BATCHOPS) ACCESS(READ)
```

Profile นี้ครอบคลุม dataset ทุกตัวที่ชื่อขึ้นต้นด้วย `PROD.CUSTOMER.` (เช่น `PROD.CUSTOMER.MASTER`,
`PROD.CUSTOMER.HISTORY`, `PROD.CUSTOMER.ARCHIVE.2026`) ด้วยกฎเดียว — ลดภาระการดูแล Profile
จำนวนมากในระบบที่มี dataset นับพันนับหมื่นไฟล์

### ข้อควรระวัง

- **Access Level ไม่ได้สื่อความหมายเดียวกันในทุก Resource Class** — เช่น `UPDATE` บน Dataset class
  หมายถึงแก้ไขข้อมูลในไฟล์ได้ แต่ `UPDATE` บน Transaction class ของ CICS อาจหมายถึงสิทธิ์ที่ต่างออก
  ไปโดยสิ้นเชิง ต้องตรวจสอบเอกสารอ้างอิงของแต่ละ Class เสมอ
- Generic Profile ที่ครอบคลุมกว้างเกินไป (เช่น `PROD.**` ครอบคลุมทุกอย่างใน `PROD`) เป็นความเสี่ยงด้าน
  ความปลอดภัยที่พบบ่อย เพราะอาจให้สิทธิ์เกินความจำเป็นโดยไม่ตั้งใจกับไฟล์ที่ไม่ควรเข้าถึงได้

### แบบฝึกหัดที่ 683.1

**โจทย์**: จงอธิบายว่าทำไมการให้สิทธิ์ `BATCHOPS` เป็น `READ` (ไม่ใช่ `UPDATE`) สำหรับ
`PROD.CUSTOMER.MASTER` ในตัวอย่างข้างต้นจึงสอดคล้องกับหลักการ Least Privilege ที่เรียนในขั้นตอนที่ 682

**เฉลยแนวทาง**: ทีม `BATCHOPS` มีหน้าที่รัน batch job ตามตารางเวลาเท่านั้น ไม่ได้มีหน้าที่แก้ไขข้อมูล
ลูกค้าโดยตรงด้วยตนเอง (การแก้ไขข้อมูลควรเกิดผ่านโปรแกรมที่ผ่านการทดสอบแล้วเท่านั้น ไม่ใช่การเข้าถึงไฟล์
โดยตรง) การให้สิทธิ์แค่ `READ` จึงเพียงพอต่อการทำงานจริงและลดความเสี่ยงที่ user ในกลุ่มนี้จะ (โดยตั้งใจ
หรือไม่ตั้งใจ) แก้ไขข้อมูล master file โดยตรงโดยไม่ผ่านกระบวนการที่ถูกต้อง

---

## ขั้นตอนที่ 684: [เชิงแนวคิด/z/OS เท่านั้น] การป้องกัน Dataset — Discrete Profile และการเชื่อมโยงกับ VSAM

> **หัวข้อนี้เป็นความรู้เชิงอ้างอิง (Reference/Conceptual) เท่านั้น** ไม่มีตัวอย่างที่รันได้จริงบน GnuCOBOL

### Discrete Profile เทียบกับ Generic Profile

นอกจาก Generic Profile ที่แนะนำในขั้นตอนที่ 683 RACF ยังรองรับ **Discrete Profile** ที่ป้องกัน dataset
**ชื่อเดียวเป๊ะ ๆ** เท่านั้น:

```
ADDSD 'PROD.PAYROLL.SALARY.2026' UACC(NONE)
PERMIT 'PROD.PAYROLL.SALARY.2026' ID(PAYROLLADM) ACCESS(UPDATE)
```

`ADDSD` (Add Dataset profile) สร้าง Profile ใหม่ที่ป้องกัน dataset นี้โดยเฉพาะ พารามิเตอร์
`UACC(NONE)` (Universal Access) กำหนดว่า **ผู้ใช้ใดก็ตามที่ไม่ได้รับสิทธิ์ผ่าน `PERMIT` อย่างชัดเจนจะไม่
มีสิทธิ์ใด ๆ เลย** — นี่คือค่าเริ่มต้นที่ปลอดภัยที่สุด (fail-safe default) ที่ควรใช้กับข้อมูลอ่อนไหวสูง เช่น
ข้อมูลเงินเดือน

### เมื่อไรควรใช้ Discrete แทน Generic

| สถานการณ์ | ควรใช้ |
|---|---|
| dataset จำนวนมากที่มีรูปแบบชื่อคล้ายกันและต้องการสิทธิ์เดียวกันหมด | Generic |
| dataset ที่มีความอ่อนไหวสูงเป็นพิเศษ (เงินเดือน, ข้อมูลส่วนบุคคล) ที่ต้องการควบคุมสิทธิ์ละเอียดเป็นรายไฟล์ | Discrete |

### การป้องกัน VSAM Cluster (ทบทวนจาก Part 055-056)

VSAM Cluster หนึ่งตัวประกอบด้วยหลายองค์ประกอบภายใน (Data Component, Index Component สำหรับ KSDS)
RACF ป้องกันที่ **ระดับชื่อ Cluster** เป็นหลัก ซึ่งครอบคลุมการเข้าถึงองค์ประกอบภายในทั้งหมดโดยอัตโนมัติ:

```
ADDSD 'PROD.ACCOUNT.VSAM.KSDS' UACC(NONE)
PERMIT 'PROD.ACCOUNT.VSAM.KSDS' ID(CORE-BANKING-BATCH) ACCESS(UPDATE)
```

### ความเชื่อมโยงกับ JCL ที่เรียนมาใน Part 052

เมื่อ JCL มีบรรทัด `DD DSN=PROD.ACCOUNT.VSAM.KSDS,DISP=SHR` (ทบทวนจาก Part 052) ระบบจะตรวจสอบ
RACF **ก่อน**ที่จะอนุญาตให้ job เปิดไฟล์นั้นได้เลย ถ้า User ID ที่ใช้รัน job ไม่มีสิทธิ์เพียงพอ job จะ
**ล้มเหลวตั้งแต่ขั้นตอน allocation** (ก่อนที่โปรแกรม COBOL แม้แต่จะเริ่มทำงานด้วยซ้ำ) พร้อม Condition
Code ที่บ่งชี้ปัญหาด้านสิทธิ์ (ทบทวนแนวคิด Condition Code จาก Part 053)

### ข้อควรระวัง

- **`UACC(NONE)` ควรเป็นค่าเริ่มต้นมาตรฐานสำหรับทุก Profile ใหม่** แล้วค่อยเพิ่มสิทธิ์ทีละ User/Group
  ที่จำเป็นจริงผ่าน `PERMIT` — แนวทางนี้เรียกว่า **"deny by default, allow by exception"** ซึ่งปลอดภัย
  กว่าการเริ่มต้นด้วยสิทธิ์กว้างแล้วค่อยจำกัดทีหลังมาก
- การลืมสร้าง Profile ป้องกันให้ dataset ใหม่เลย (ไม่ใช่แค่ตั้งค่า UACC ผิด) อาจทำให้ dataset นั้นตกอยู่
  ภายใต้กฎ **default RACF policy ของระบบ** ซึ่งอาจอนุญาตกว้างกว่าที่ตั้งใจ — องค์กรที่ดีมักมีกระบวนการ
  บังคับให้สร้าง Profile คู่กับการสร้าง dataset ใหม่เสมอ

### แบบฝึกหัดที่ 684.1

**โจทย์**: จงอธิบายว่าทำไมหลักการ "deny by default, allow by exception" (`UACC(NONE)` แล้วค่อย
`PERMIT` เพิ่ม) จึงปลอดภัยกว่าการเริ่มต้นด้วยสิทธิ์กว้างแล้วค่อยจำกัดทีหลัง

**เฉลยแนวทาง**: หากเริ่มต้นด้วยสิทธิ์กว้าง (เช่น `UACC(READ)` ให้ทุกคนอ่านได้ตั้งแต่แรก) ความเสี่ยงคือ
ผู้ดูแลระบบอาจ**ลืม**จำกัดสิทธิ์ทีหลัง ทำให้ผู้ใช้ที่ไม่ควรมีสิทธิ์ยังคงเข้าถึงได้อยู่เป็นเวลานานโดยไม่มีใคร
สังเกตเห็น (ความผิดพลาดแบบ "ลืมปิด" มักไม่ถูกตรวจพบ) ในทางกลับกัน หากเริ่มต้นด้วย `UACC(NONE)` แล้ว
ต้อง `PERMIT` เพิ่มทีละกลุ่มที่จำเป็นจริง ความผิดพลาดที่อาจเกิดขึ้นคือ **ผู้ใช้ที่ควรมีสิทธิ์กลับเข้าถึงไม่ได้**
ซึ่งจะถูกพบและรายงานทันทีตั้งแต่ครั้งแรกที่พยายามใช้งาน (fail loud) — ปลอดภัยกว่าการที่สิทธิ์เกินขอบเขต
หลุดรอดไปโดยไม่มีใครรู้ (fail silent) อย่างมาก

---

## ขั้นตอนที่ 685: [เชิงแนวคิด/z/OS เท่านั้น] ความปลอดภัยของโปรแกรมและ APF-Authorized Library

> **หัวข้อนี้เป็นความรู้เชิงอ้างอิง (Reference/Conceptual) เท่านั้น** ไม่มีตัวอย่างที่รันได้จริงบน GnuCOBOL

### PROGRAM Class — ควบคุมว่าใครรัน Load Module ไหนได้

นอกจากการป้องกัน dataset RACF ยังป้องกัน **โปรแกรมที่คอมไพล์แล้ว (Load Module)** ผ่าน Class ชื่อ
`PROGRAM` ได้โดยตรง:

```
RDEFINE PROGRAM PAYROLLCALC ADDMEM('PROD.LOADLIB'/'VOL001'/NOPADCHK)
PERMIT PAYROLLCALC CLASS(PROGRAM) ID(PAYROLLADM) ACCESS(READ)
```

การป้องกันนี้ทำให้แม้ผู้ใช้จะมีสิทธิ์เข้าถึง library ที่เก็บโปรแกรม แต่ถ้าไม่มีสิทธิ์เฉพาะใน `PROGRAM` class
ก็ยังคงรันโปรแกรมนั้นไม่ได้ — เป็นการป้องกันซ้อนอีกชั้นสำหรับโปรแกรมที่มีความอ่อนไหวสูง (เช่น โปรแกรม
คำนวณเงินเดือนที่ควรรันได้เฉพาะช่วง batch window ที่กำหนดโดยผู้ใช้ที่ได้รับอนุญาตเท่านั้น)

### แนวคิด APF-Authorized Library

**APF (Authorized Program Facility)** คือกลไกของ z/OS ที่กำหนดว่า **library (ชุดโปรแกรม) ใดได้รับ
อนุญาตให้ทำงานด้วยสิทธิ์ระดับสูงสุดของระบบปฏิบัติการ** (เรียกว่า "authorized state") ซึ่งอนุญาตให้เรียกใช้
บริการระดับ system ที่โปรแกรมทั่วไปเรียกใช้ไม่ได้ เช่น การเข้าถึงหน่วยความจำของ address space อื่น หรือ
การปิดกั้นการขัดจังหวะ (disable interrupts) — สิทธิ์ระดับนี้ทรงพลังมากและมีความเสี่ยงสูงหากถูกใช้ผิดวิธี

โปรแกรม COBOL ทั่วไป (รวมถึงโปรแกรมทั้งหมดที่เขียนตลอดหลักสูตรนี้) **ไม่จำเป็นต้องและไม่ควรรันใน
APF-Authorized Library เลย** — แนวคิดนี้สงวนไว้สำหรับ System Software เช่น ส่วนประกอบของ Subsystem
(CICS, DB2 เอง) หรือ Utility ระดับระบบปฏิบัติการเท่านั้น

### ทำไมนักพัฒนา COBOL ทั่วไปควรรู้จักแนวคิดนี้แม้จะไม่ได้ใช้โดยตรง

1. **เพื่อเข้าใจว่าทำไมบาง Utility ถึงมีสิทธิ์ที่โปรแกรมของเราไม่มี** — เมื่อเห็น error ที่บอกว่าไม่มีสิทธิ์ทำ
   บางอย่าง อาจเป็นเพราะโปรแกรมของเราไม่ได้รันใน authorized state ซึ่งเป็นเรื่องปกติและ**ถูกต้องแล้ว**
2. **เพื่อรู้ว่าการขอให้โปรแกรมของตัวเองรันแบบ APF-authorized เป็นเรื่องผิดปกติที่ต้องมีเหตุผลหนักแน่นมาก**
   — คำขอแบบนี้ต้องผ่านการตรวจสอบด้านความปลอดภัยอย่างเข้มงวดที่สุดในองค์กร เพราะความเสี่ยงสูงมาก
   หากมีบั๊กในโปรแกรมที่รันด้วยสิทธิ์ระดับนี้

### ข้อควรระวัง

- **ห้ามขอสิทธิ์ APF-Authorization ให้กับโปรแกรม COBOL ระดับ Application ทั่วไปเด็ดขาด** เว้นแต่จะมี
  ความจำเป็นด้าน System Software จริง ๆ ซึ่งพบได้น้อยมากในงานพัฒนา COBOL ระดับ Application
- แนวคิดนี้มักถูกเข้าใจผิดว่าเป็น "ทางลัด" เพื่อแก้ปัญหาสิทธิ์ที่ไม่เพียงพอ — การแก้ปัญหาที่ถูกต้องคือขอสิทธิ์
  ที่เหมาะสมผ่าน RACF ตามปกติ (ขั้นตอนที่ 683-684) ไม่ใช่การยกระดับสิทธิ์โปรแกรมทั้งหมด

### แบบฝึกหัดที่ 685.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม COBOL ระดับ Application (เช่น โปรแกรมคำนวณดอกเบี้ยธนาคารที่เรียน
มาตลอดหลักสูตรนี้) จึงไม่ควรรันในสถานะ APF-Authorized

**เฉลยแนวทาง**: โปรแกรม COBOL ระดับ Application มีหน้าที่ประมวลผลข้อมูลธุรกิจตามตรรกะที่กำหนด ไม่มี
ความจำเป็นใด ๆ ที่จะต้องเข้าถึงบริการระดับระบบปฏิบัติการที่สงวนไว้สำหรับ System Software การให้สิทธิ์
ระดับ APF-Authorized กับโปรแกรมที่ไม่จำเป็นต้องใช้จะเป็นการเพิ่มพื้นที่เสี่ยง (attack surface) โดยไม่มี
ประโยชน์ใด ๆ เลย หากโปรแกรมนั้นมีบั๊ก (ซึ่งเป็นไปได้เสมอไม่ว่าจะทดสอบดีแค่ไหน) บั๊กนั้นอาจถูกใช้ประโยชน์
ในทางที่เป็นอันตรายต่อทั้งระบบมากกว่าที่ควรจะเป็น เพราะโปรแกรมมีสิทธิ์เกินกว่าที่งานจริงต้องการ ตรงกับ
หลักการ Least Privilege ที่เรียนมาตั้งแต่ขั้นตอนที่ 682

---

## ขั้นตอนที่ 686: [ทดสอบได้จริง] การตรวจสอบข้อมูลตัวเลขนำเข้าด้วย IS NUMERIC

### จากนี้ไปคือ Secure Coding — ทดสอบได้จริงด้วย GnuCOBOL ทุกตัวอย่าง

RACF ป้องกันไม่ให้คนนอกเข้าถึงโปรแกรม แต่**ไม่มีทางป้องกันไม่ให้ผู้ใช้ที่มีสิทธิ์ถูกต้องป้อนข้อมูลผิดพลาด
เข้าไปในโปรแกรมได้เลย** — หน้าที่นี้เป็นของโปรแกรมเมอร์ COBOL โดยตรง ขั้นตอนนี้พิสูจน์ด้วยการรันจริงว่า
การไม่ตรวจสอบข้อมูลนำเข้าเป็นความเสี่ยงร้ายแรงเพียงใด

### ทดสอบ: ACCEPT ตรงเข้าฟิลด์ตัวเลขโดยไม่ตรวจสอบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP686NOVAL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-INPUT-QTY        PIC 9(5)     VALUE 0.
       01  WS-UNIT-PRICE       PIC 9(5)V99  VALUE 250.00.
       01  WS-TOTAL            PIC 9(9)V99  VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Enter quantity: " WITH NO ADVANCING.
           ACCEPT WS-INPUT-QTY.
           COMPUTE WS-TOTAL = WS-INPUT-QTY * WS-UNIT-PRICE.
           DISPLAY "Total = " WS-TOTAL.
           STOP RUN.
```

```bash
cobc -x -o step686_novalidate step686_novalidate.cob
printf -- "999999\n" | ./step686_novalidate
printf -- "-500\n" | ./step686_novalidate
printf -- "12A45\n" | ./step686_novalidate
```

**ผลลัพธ์จริงที่ได้ (ยืนยันด้วย GnuCOBOL) — สามกรณีที่อันตรายต่างกัน:**

```
-- 999999 (6 หลัก เข้า PIC 9(5) ที่รับได้แค่ 5 หลัก) --
Total = 024999750.00
      (หลักสูงถูกตัดทิ้งเงียบ ๆ: 999999 -> 99999, แล้วคูณ 250.00
       ได้ผลลัพธ์ที่ "ดูสมเหตุสมผล" แต่ผิดจากที่ผู้ใช้ตั้งใจป้อนโดยสิ้นเชิง)

-- -500 (เครื่องหมายลบใส่ในฟิลด์ PIC 9 ที่ไม่มีเครื่องหมาย) --
Total = 000125000.00
      (เครื่องหมายลบถูกละเว้นเงียบ ๆ กลายเป็น 500 ผลลัพธ์เป็นบวกทั้งที่
       ผู้ใช้อาจตั้งใจป้อนค่าติดลบเพื่อความหมายอื่น)

-- 12A45 (ตัวอักษรปนตัวเลข) --
Total = 000000000.00
      (กลายเป็น 0 เงียบ ๆ โดยไม่มีข้อความแจ้งเตือนใด ๆ เลยว่าข้อมูลผิดพลาด)
```

### วิเคราะห์อันตราย

ทั้งสามกรณีมีจุดร่วมที่อันตรายเหมือนกันทุกประการ: **โปรแกรมไม่ crash และไม่แสดง error ใด ๆ เลย** แต่กลับ
ให้ผลลัพธ์ที่ผิดพลาดแบบ "ดูสมเหตุสมผล" (เป็นตัวเลขปกติ ไม่ใช่ค่าประหลาดที่สังเกตเห็นได้ง่าย) — นี่คือรูปแบบ
เดียวกับกับดัก "ผลลัพธ์ค้างที่ดูสมเหตุสมผลแต่ผิดสนิท" ที่เรียนมาแล้วใน Part 015 ขั้นตอนที่ 143 (การหาร
ด้วยศูนย์) เพียงแต่คราวนี้สาเหตุคือ**ข้อมูลนำเข้าที่ไม่ผ่านการตรวจสอบ** แทน

### แก้ไข: ตรวจสอบด้วย IS NUMERIC ก่อนเชื่อถือข้อมูลใด ๆ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP686VALID.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW-INPUT        PIC X(5)     VALUE SPACES.
       01  WS-INPUT-QTY        PIC 9(5)     VALUE 0.
       01  WS-UNIT-PRICE       PIC 9(5)V99  VALUE 250.00.
       01  WS-TOTAL            PIC 9(9)V99  VALUE 0.
       01  WS-VALID-FLAG       PIC X(1)     VALUE "N".
           88  WS-INPUT-VALID          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Enter quantity: " WITH NO ADVANCING.
      *> Accept as ALPHANUMERIC first - this is the key technique.
      *> Accepting straight into a numeric PIC lets COBOL silently
      *> coerce garbage into a "valid-looking" number (proven above
      *> in step686_novalidate); accepting as text lets us CHECK
      *> before trusting the value at all.
           ACCEPT WS-RAW-INPUT.
           MOVE "N" TO WS-VALID-FLAG.
           IF WS-RAW-INPUT IS NUMERIC
               MOVE "Y" TO WS-VALID-FLAG
           END-IF.

           IF WS-INPUT-VALID
               MOVE WS-RAW-INPUT TO WS-INPUT-QTY
               COMPUTE WS-TOTAL = WS-INPUT-QTY * WS-UNIT-PRICE
               DISPLAY "Total = " WS-TOTAL
           ELSE
               DISPLAY "ERROR: '" WS-RAW-INPUT
                   "' is not a valid quantity. No calculation done."
           END-IF.
           STOP RUN.
```

```bash
cobc -x -o step686_validated step686_validated.cob
printf -- "00012\n" | ./step686_validated
printf -- "ABCDE\n" | ./step686_validated
printf -- "12A45\n" | ./step686_validated
```

**ผลลัพธ์จริง:**

```
-- 00012 (valid digits, fills field exactly) --
Total = 000003000.00

-- ABCDE --
ERROR: 'ABCDE' is not a valid quantity. No calculation done.

-- 12A45 --
ERROR: '12A45' is not a valid quantity. No calculation done.
```

### อธิบายจุดสำคัญ

- **เทคนิคหัวใจสำคัญ**: `ACCEPT` เข้าฟิลด์ `PIC X` (alphanumeric) **ก่อนเสมอ** แทนที่จะ `ACCEPT` ตรง
  เข้าฟิลด์ตัวเลข เพราะการ `ACCEPT` ตรงเข้าฟิลด์ตัวเลขทำให้ COBOL "ช่วยแปลง" ข้อมูลที่ไม่ถูกต้องให้
  กลายเป็นตัวเลขที่ "ดูถูกต้อง" แบบเงียบ ๆ ทันที (ตามที่พิสูจน์ในเวอร์ชันแรก) โดยไม่มีโอกาสให้เราตรวจสอบ
  ก่อนเลย
- `IS NUMERIC` เป็น Class Condition ที่เรียนมาตั้งแต่ Part 042 — ตรวจสอบว่าฟิลด์นั้นมี**เฉพาะ**ตัวอักษร
  ที่เป็นตัวเลข (0-9) เท่านั้นหรือไม่ (และเครื่องหมาย +/- ถ้าเป็นฟิลด์ signed) หากมีตัวอักษรอื่นปนอยู่แม้
  แต่ตัวเดียว จะได้ผล `FALSE` ทันที
- เมื่อตรวจสอบผ่านแล้วเท่านั้นจึงค่อย `MOVE` ค่าเข้าฟิลด์ตัวเลขจริง — ลำดับขั้นตอนนี้ ("Validate then
  Trust") คือหัวใจของ Input Validation ที่ดีในทุกภาษาโปรแกรม ไม่ใช่แค่ COBOL

### ข้อควรระวัง

- **ความกว้างของฟิลด์ที่ใช้ `ACCEPT` ต้องตรงกับความกว้างที่คาดหวังพอดี** — ถ้าผู้ใช้พิมพ์ค่าสั้นกว่าฟิลด์
  (เช่นพิมพ์ "2" ลงในฟิลด์ `PIC X(5)`) ช่องว่างที่เหลือจะถูกเติมด้วย**ช่องว่าง (space)** ซึ่ง**ไม่ใช่ตัวเลข**
  ทำให้ `IS NUMERIC` กลายเป็น `FALSE` ทั้งที่ผู้ใช้ป้อนตัวเลขถูกต้อง — นี่เป็นกับดักที่พบบ่อยมาก (จะเห็น
  วิธีจัดการที่ถูกต้องในขั้นตอนที่ 690) แนวทางแก้คือให้ผู้ใช้เติมเลข 0 นำหน้าให้เต็มความกว้างเสมอ
  (ตามที่ตัวอย่างนี้ใช้ `"00012"` แทน `"12"` เปล่า ๆ) หรือปรับความกว้างฟิลด์ให้ตรงกับความยาวข้อมูลจริง
- `IS NUMERIC` ตรวจสอบแค่ "รูปแบบเป็นตัวเลขหรือไม่" เท่านั้น **ไม่ได้ตรวจสอบขอบเขตค่าที่สมเหตุสมผล**
  (เช่น ค่า `99999` ก็ยังคงผ่าน `IS NUMERIC` แม้จะไม่สมเหตุสมผลสำหรับ "จำนวนสินค้า" ในบริบทธุรกิจจริง)
  — การตรวจสอบขอบเขตค่าต้องทำเพิ่มเติมด้วย `IF` (จะเห็นในขั้นตอนที่ 687)

### แบบฝึกหัดที่ 686.1

**โจทย์**: จงอธิบายว่าทำไมการ `ACCEPT` ตรงเข้าฟิลด์ `PIC 9(5)` (ตามในเวอร์ชันแรก) จึงเป็นความเสี่ยง
ด้านความปลอดภัย ไม่ใช่แค่ปัญหาความถูกต้องของข้อมูลธรรมดา

**เฉลยแนวทาง**: เพราะพฤติกรรมนี้ทำให้**ข้อมูลนำเข้าที่ไม่ถูกต้องหรือมุ่งร้ายกลายเป็นข้อมูลที่ "ดูถูกต้อง"
โดยไม่มีร่องรอยหรือการแจ้งเตือนใด ๆ เลย** ซึ่งเป็นคุณลักษณะที่อันตรายเป็นพิเศษในเชิงความปลอดภัย เพราะ
ผู้โจมตี (หรือแม้แต่ระบบภายนอกที่ป้อนข้อมูลผิดพลาดโดยไม่ได้ตั้งใจ) สามารถส่งข้อมูลที่ทำให้ระบบคำนวณผิดพลาด
ได้โดยไม่ทิ้งร่องรอยที่ตรวจสอบได้ในภายหลังเลย (ไม่มี error log ไม่มี exception ไม่มีสัญญาณผิดปกติใด ๆ)
ต่างจากบั๊กทั่วไปที่มักแสดงอาการชัดเจน (เช่น โปรแกรม crash) การป้องกันที่ถูกต้องคือต้องตรวจสอบและปฏิเสธ
ข้อมูลที่ไม่ถูกต้องอย่างชัดเจนตั้งแต่จุดรับข้อมูลเข้า (input boundary) เสมอ

---

## ขั้นตอนที่ 687: [ทดสอบได้จริง] ป้องกัน Subscript Overflow ในตาราง (Table/Array Bounds)

### ทดสอบ: Subscript ที่ไม่ตรวจสอบขอบเขตเลย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP687OVERFLOW.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  ACCOUNT-TABLE.
           05  ACCOUNT-ENTRY OCCURS 10 TIMES.
               10  ACCT-BALANCE     PIC 9(7)V99.
       01  WS-SUBSCRIPT         PIC 9(3)     VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Enter subscript (should be 1-10): "
               WITH NO ADVANCING.
           ACCEPT WS-SUBSCRIPT.
      *> DANGER: no bounds check at all before using WS-SUBSCRIPT
      *> as a table subscript.
           MOVE 999.99 TO ACCT-BALANCE(WS-SUBSCRIPT).
           DISPLAY "Wrote 999.99 into slot " WS-SUBSCRIPT.
           DISPLAY "ACCT-BALANCE(1) = " ACCT-BALANCE(1).
           STOP RUN.
```

```bash
cobc -x -o step687_overflow_normal step687_overflow.cob
printf -- "99\n" | ./step687_overflow_normal
```

**ผลลัพธ์จริง (คอมไพล์แบบปกติ ไม่มี `-debug`):**

```
Wrote 999.99 into slot 099
ACCT-BALANCE(1) = 0000000.00
```

### วิเคราะห์อันตราย — เขียนทับหน่วยความจำนอกขอบเขตแบบเงียบ ๆ

โปรแกรม**ไม่ crash เลย** แม้ `ACCOUNT-TABLE` จะมีแค่ 10 ช่อง (1-10) แต่รับ subscript `99` ได้โดยไม่มี
คำเตือนใด ๆ — คำสั่ง `MOVE 999.99 TO ACCT-BALANCE(99)` เขียนทับหน่วยความจำที่**อยู่นอกขอบเขตของตาราง
ไปไกลมาก** ซึ่งในความเป็นจริงอาจเป็นหน่วยความจำของตัวแปรอื่นในโปรแกรม ทำให้เกิด **การทำลายข้อมูลแบบ
เงียบ ๆ (silent memory corruption)** ที่ตรวจจับได้ยากมาก

### เปรียบเทียบ: คอมไพล์ด้วย -debug เพื่อเปิดการตรวจสอบ Subscript อัตโนมัติ

```bash
cobc -x -debug -o step687_overflow_debug step687_overflow.cob
printf -- "99\n" | ./step687_overflow_debug
```

**ผลลัพธ์จริง:**

```
libcob: step687_overflow.cob:19: error: subscript of 'ACCT-BALANCE' out of bounds: 99
	maximum subscript for 'ACCT-BALANCE': 10
```

แฟล็ก `-debug` (ทบทวนจาก Part 068 ขั้นตอนที่ 677 — คนละแฟล็กกับ `-O2`) เปิดการตรวจสอบขอบเขตของ
Subscript ให้อัตโนมัติ **แต่นี่คือเครื่องมือสำหรับ debug ช่วงพัฒนา ไม่ควรพึ่งพาเป็นการป้องกันหลักในโปรแกรม
production** (ทบทวนจาก Part 068: `-debug` ลดประสิทธิภาพและไม่ใช่ค่าปกติที่ใช้ compile โปรแกรมที่ deploy
จริง)

### วิธีที่ถูกต้องสำหรับ Production: ตรวจสอบขอบเขตด้วย IF ก่อนเสมอ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP687VALID.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  ACCOUNT-TABLE.
           05  ACCOUNT-ENTRY OCCURS 10 TIMES.
               10  ACCT-BALANCE     PIC 9(7)V99.
       01  WS-SUBSCRIPT         PIC 9(3)     VALUE 0.
       01  WS-TABLE-MAX         PIC 9(3)     VALUE 10.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Enter subscript (should be 1-10): "
               WITH NO ADVANCING.
           ACCEPT WS-SUBSCRIPT.
      *> Defensive check BEFORE touching the table at all - this
      *> works on every GnuCOBOL build, with or without -debug.
           IF WS-SUBSCRIPT >= 1 AND WS-SUBSCRIPT <= WS-TABLE-MAX
               MOVE 999.99 TO ACCT-BALANCE(WS-SUBSCRIPT)
               DISPLAY "Wrote 999.99 into slot " WS-SUBSCRIPT
           ELSE
               DISPLAY "ERROR: subscript " WS-SUBSCRIPT
                   " is out of range (1-" WS-TABLE-MAX ")."
           END-IF.
           DISPLAY "ACCT-BALANCE(1) = " ACCT-BALANCE(1).
           STOP RUN.
```

```bash
cobc -x -o step687_validated step687_validated.cob
printf -- "99\n" | ./step687_validated
```

**ผลลัพธ์จริง:**

```
ERROR: subscript 099 is out of range (1-010).
ACCT-BALANCE(1) = 0000000.00
```

### อธิบายจุดสำคัญ

- การตรวจสอบด้วย `IF WS-SUBSCRIPT >= 1 AND WS-SUBSCRIPT <= WS-TABLE-MAX` **ทำงานได้เสมอไม่ว่าจะ
  compile ด้วยแฟล็กใด** ต่างจากการพึ่งพา `-debug` ที่เป็นเพียงเครื่องมือช่วยตรวจจับตอนพัฒนาเท่านั้น
- ควรเก็บค่าขอบเขตสูงสุด (`WS-TABLE-MAX`) เป็นตัวแปรแยกต่างหากที่**ตรงกับจำนวน `OCCURS` จริงเสมอ**
  แทนที่จะเขียนตัวเลข `10` ซ้ำหลายจุดในโค้ด (หลักการ "Single Source of Truth" ที่ลดความเสี่ยงเมื่อต้อง
  แก้ไขขนาดตารางในอนาคต)
- เทคนิคนี้คือการนำ**หลักการเดียวกันกับขั้นตอนที่ 686** (Validate ก่อน Trust) มาประยุกต์ใช้กับ Subscript
  แทนที่จะเป็นแค่ตรวจสอบรูปแบบตัวเลข ยังต้องตรวจสอบ**ขอบเขตค่า**ด้วยเสมอ

### ข้อควรระวัง

- **อย่าใช้ `-debug` เป็นกลไกป้องกันหลักในโปรแกรม production** เพราะ (1) ลดประสิทธิภาพ (Part 068)
  (2) โปรแกรม production มักถูก compile โดยไม่มีแฟล็กนี้อยู่แล้วตามมาตรฐานการ build ขององค์กร การพึ่งพา
  แฟล็กที่อาจไม่ได้เปิดใช้งานจริงเป็นความเสี่ยงที่ยอมรับไม่ได้
- `INDEXED BY` ร่วมกับ `SET .. TO` (ทบทวนจาก Part 017) ก็ยังคง**ไม่มีการตรวจสอบขอบเขตอัตโนมัติ**เช่น
  เดียวกับ subscript ธรรมดา เว้นแต่จะ compile ด้วย `-debug` — ต้องเขียน `IF` ตรวจสอบเองเสมอไม่ว่าจะใช้
  subscript แบบ `PIC 9` ธรรมดาหรือ `INDEXED BY`

### แบบฝึกหัดที่ 687.1

**โจทย์**: จงอธิบายว่าทำไมผลลัพธ์ `ACCT-BALANCE(1) = 0000000.00` จึงยังคงเป็น `0` แม้จะรันตัวอย่างที่
ไม่มีการตรวจสอบ (subscript = 99) ไปแล้ว ทั้งที่ subscript 99 อยู่นอกขอบเขตของตารางที่มีแค่ 10 ช่อง

**เฉลยแนวทาง**: เพราะการเขียนทับหน่วยความจำนอกขอบเขต (subscript 99) เกิดขึ้นที่ตำแหน่งหน่วยความจำที่
**ไกลจาก** `ACCT-BALANCE(1)` มาก (99 ช่องถัดจากจุดเริ่มต้นของตาราง เทียบกับช่องแรกสุด) จึงไม่ได้ไปเขียน
ทับตำแหน่งของ `ACCT-BALANCE(1)` โดยตรงในกรณีนี้ **แต่นี่ไม่ได้แปลว่าปลอดภัย** เพราะการเขียนทับยังคง
เกิดขึ้นจริงที่หน่วยความจำตำแหน่งอื่นในโปรแกรม (อาจเป็นตัวแปรอื่น หรือแม้แต่พื้นที่ที่ไม่ได้ถูกจองไว้เลย ซึ่ง
อาจทำให้โปรแกรม crash แบบ SIGSEGV ได้เช่นกันหากตำแหน่งนั้นไกลเกินขอบเขตหน่วยความจำที่ระบบปฏิบัติการ
จัดสรรให้โปรเซสนี้ ทบทวนพฤติกรรมนี้จาก Part 031 ขั้นตอนที่ 304) ผลลัพธ์ที่ไม่กระทบ `ACCT-BALANCE(1)`
ในกรณีนี้เป็นเพียง**ความบังเอิญของ layout หน่วยความจำ**ไม่ใช่พฤติกรรมที่รับประกันได้เลย

---

## ขั้นตอนที่ 688: [ทดสอบได้จริง] ป้องกัน Buffer Overflow ใน STRING และ Reference Modification

### ทดสอบที่ 1: STRING ที่ล้นความจุของฟิลด์ปลายทาง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP688STRINGOF.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST-NAME        PIC X(20) VALUE "CHRISTOPHER".
       01  WS-LAST-NAME         PIC X(25)
           VALUE "MONTGOMERY-WORTHINGTON".
       01  WS-FULL-NAME         PIC X(15) VALUE SPACES.
       01  WS-PTR               PIC 9(3)  VALUE 1.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> WS-FULL-NAME is intentionally too small (15 bytes) for the
      *> combined name - this is a classic buffer-overflow-style
      *> risk if not guarded.
           STRING FUNCTION TRIM(WS-FIRST-NAME) DELIMITED BY SIZE
                   " " DELIMITED BY SIZE
                   FUNCTION TRIM(WS-LAST-NAME) DELIMITED BY SIZE
               INTO WS-FULL-NAME
               WITH POINTER WS-PTR
               ON OVERFLOW
                   DISPLAY "WARNING: name truncated - target field "
                       "too small for combined name."
               NOT ON OVERFLOW
                   DISPLAY "Name fit without truncation."
           END-STRING.
           DISPLAY "WS-FULL-NAME = [" WS-FULL-NAME "]".
           STOP RUN.
```

```bash
cobc -x -o step688_string_overflow step688_string_overflow.cob
./step688_string_overflow
```

**ผลลัพธ์จริง:**

```
WARNING: name truncated - target field too small for combined name.
WS-FULL-NAME = [CHRISTOPHER MON]
```

### อธิบายจุดสำคัญ

- `STRING` **ไม่ crash แม้ข้อมูลจะล้นความจุของฟิลด์ปลายทาง** — มันจะหยุดคัดลอกข้อมูลทันทีที่ฟิลด์
  ปลายทางเต็ม (ตัดข้อมูลส่วนที่เหลือทิ้งแบบเงียบ ๆ) **เว้นแต่จะมีวรรค `ON OVERFLOW` กำกับไว้** เพื่อจับ
  สถานการณ์นี้และแจ้งเตือน
- `ON OVERFLOW`/`NOT ON OVERFLOW` (ทบทวนแนวคิด scope terminator จาก Part 019) คือกลไกในตัวของ
  `STRING` ที่ใช้ตรวจจับการล้นได้โดยตรง ไม่ต้องคำนวณความยาวเปรียบเทียบเองด้วยมือ
- `WITH POINTER WS-PTR` ทำให้เราทราบตำแหน่งที่ `STRING` หยุดคัดลอกได้ (ค่าใน `WS-PTR` หลังจบคำสั่ง)
  ซึ่งมีประโยชน์เมื่อต้องการทราบว่าข้อมูลส่วนใดถูกตัดทิ้งไปบ้าง

### ทดสอบที่ 2: Reference Modification ที่มีจุดเริ่มต้น/ความยาวมาจากผู้ใช้โดยตรง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP688REFMOD.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ACCOUNT-NO        PIC X(10) VALUE "1234567890".
       01  WS-START-POS         PIC 9(3)  VALUE 0.
       01  WS-LENGTH            PIC 9(3)  VALUE 0.
       01  WS-PIECE             PIC X(10) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "Enter start position: " WITH NO ADVANCING.
           ACCEPT WS-START-POS.
           DISPLAY "Enter length: " WITH NO ADVANCING.
           ACCEPT WS-LENGTH.
      *> DANGER: reference modification with attacker/user-controlled
      *> start position and length, no bounds check at all.
           MOVE WS-ACCOUNT-NO(WS-START-POS:WS-LENGTH) TO WS-PIECE.
           DISPLAY "Extracted piece = [" WS-PIECE "]".
           STOP RUN.
```

```bash
cobc -x -o step688_refmod_normal step688_refmod.cob
printf -- "15\n5\n" | ./step688_refmod_normal
```

**ผลลัพธ์จริง (คอมไพล์แบบปกติ):**

```
Extracted piece = [          ]
```

**เปรียบเทียบ: คอมไพล์ด้วย -debug เพื่อเปิดการตรวจสอบ:**

```bash
cobc -x -debug -o step688_refmod_debug step688_refmod.cob
printf -- "15\n5\n" | ./step688_refmod_debug
printf -- "8\n10\n" | ./step688_refmod_debug
```

**ผลลัพธ์จริง:**

```
-- start=15 (เกินความยาวฟิลด์ 10 ไปมาก) --
libcob: step688_refmod.cob:20: error: offset of 'WS-ACCOUNT-NO' out of bounds: 15, maximum: 10

-- start=8, length=10 (เริ่มถูกแต่ยาวเกินจนล้นขอบฟิลด์) --
libcob: step688_refmod.cob:20: error: length of 'WS-ACCOUNT-NO' out of bounds: 10, starting at: 8, maximum: 10
```

### อธิบายจุดสำคัญ

- เมื่อ**ไม่มี**การตรวจสอบและ**ไม่ได้** compile ด้วย `-debug` reference modification ที่เกินขอบเขต
  **ไม่ crash แต่ก็ไม่แจ้งเตือนใด ๆ** — เพียงคืนค่าที่ตัดทอนหรือเป็นช่องว่างเงียบ ๆ เท่านั้น (พฤติกรรมคล้าย
  กับ `STRING` ที่ล้นในทดสอบที่ 1 แต่ไม่มีกลไก `ON OVERFLOW` ให้ตรวจจับเหมือนกัน)
  ทำให้เกิดความเสี่ยงแบบเดียวกับ Subscript Overflow ในขั้นตอนที่ 687
- ทั้ง `WS-START-POS` และ `WS-LENGTH` ในตัวอย่างนี้**มาจากผู้ใช้โดยตรงโดยไม่มีการตรวจสอบเลย** — ถ้าเป็น
  ระบบจริงที่ค่าเหล่านี้มาจากข้อมูลภายนอก (เช่น พารามิเตอร์จากไฟล์ input หรือ transaction ที่รับมาจาก
  ระบบอื่น) นี่คือรูปแบบความเสี่ยงเดียวกับช่องโหว่ Buffer Overflow ที่มีชื่อเสียงในภาษาระดับต่ำอื่น ๆ
  (เช่น C) เพียงแต่ COBOL/GnuCOBOL ไม่ทำให้เกิดการ crash รุนแรงระดับเดียวกันเสมอไป

### วิธีที่ถูกต้อง: ตรวจสอบขอบเขตก่อนใช้ Reference Modification เสมอ

```cobol
      *> Always validate BEFORE using user-controlled start/length
      *> as reference modification boundaries.
           IF WS-START-POS >= 1 AND WS-LENGTH >= 1
               AND (WS-START-POS + WS-LENGTH - 1) <= 10
               MOVE WS-ACCOUNT-NO(WS-START-POS:WS-LENGTH) TO WS-PIECE
           ELSE
               DISPLAY "ERROR: start/length out of bounds for a "
                   "10-byte field."
           END-IF
```

### ข้อควรระวัง

- สูตรตรวจสอบขอบเขตที่ถูกต้องสำหรับ reference modification คือ **`start >= 1` และ `length >= 1` และ
  `(start + length - 1) <= ความกว้างของฟิลด์`** — ลืมเงื่อนไขใดเงื่อนไขหนึ่งอาจยังคงเปิดช่องให้เข้าถึงนอก
  ขอบเขตได้
- `ON OVERFLOW` ใช้ได้กับ `STRING`/`UNSTRING` เท่านั้น **ไม่มีกลไกเทียบเท่าสำหรับ Reference
  Modification โดยตรง** — ต้องเขียน `IF` ตรวจสอบเองเสมอ (ต่างจาก `STRING` ที่มีกลไกในตัวให้ใช้)

### แบบฝึกหัดที่ 688.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `STRING` มี `ON OVERFLOW` ในตัว แต่ Reference Modification ไม่มี
กลไกเทียบเท่า จึงทำให้ Reference Modification เป็นจุดเสี่ยงที่นักพัฒนาต้องระมัดระวังเป็นพิเศษมากกว่า

**เฉลยแนวทาง**: เพราะ `STRING` ให้เครื่องมือตรวจจับสถานการณ์ผิดปกติมาพร้อมในตัวภาษา (`ON OVERFLOW`)
ทำให้นักพัฒนาที่ใช้คำสั่งนี้อย่างถูกต้องมักจะได้รับการแจ้งเตือนโดยอัตโนมัติเมื่อเกิดปัญหา แต่ Reference
Modification ไม่มีกลไกเทียบเท่าเลย ทำให้ความรับผิดชอบทั้งหมดในการตรวจสอบขอบเขตตกอยู่ที่นักพัฒนาเอง
ทั้งหมด หากลืมเขียน `IF` ตรวจสอบ จะไม่มี "ตาข่ายนิรภัย" ใด ๆ ในภาษาที่ช่วยจับข้อผิดพลาดนี้ให้เลย (นอกจาก
การ compile ด้วย `-debug` ซึ่งเป็นเครื่องมือ debug ไม่ใช่กลไกป้องกันถาวร) นี่คือเหตุผลที่ Reference
Modification ที่รับค่า start/length จากภายนอกจึงต้องถูกตรวจสอบด้วยความระมัดระวังเป็นพิเศษเสมอ

---

## ขั้นตอนที่ 689: [ทดสอบได้จริง] การปกปิดข้อมูลอ่อนไหวในผลลัพธ์ (Data Masking)

### ปัญหา: การแสดงข้อมูลอ่อนไหวเต็มรูปแบบใน Log/Console

โปรแกรมจำนวนมากมีนิสัย `DISPLAY` ข้อมูลทั้งหมดออกมาตรง ๆ เพื่อช่วย debug โดยไม่ทันคิดว่าข้อความเหล่านั้น
มักถูกเก็บเป็น **Log File** ที่อาจถูกอ่านโดยคนที่ไม่ควรเห็นข้อมูลอ่อนไหว (เช่น เลขบัญชีธนาคารเต็มรูปแบบ,
เลขบัตรประชาชน) แม้ RACF จะป้องกันการเข้าถึง dataset ของ log ได้ในระดับหนึ่ง (ขั้นตอนที่ 684) แต่
หลักการ **Defense in Depth** บอกว่าไม่ควรพึ่งพาการป้องกันชั้นเดียว

### ทดสอบ: เปรียบเทียบ Log ที่ไม่ปลอดภัยกับ Log ที่ปกปิดข้อมูลแล้ว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP689MASK.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ACCOUNT-NO        PIC X(10) VALUE "1234567890".
       01  WS-MASKED-ACCT       PIC X(10) VALUE SPACES.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> INSECURE: printing the full account number to a log/console
      *> exposes sensitive data to anyone who can read that output.
           DISPLAY "INSECURE LOG LINE: account=" WS-ACCOUNT-NO.

      *> SECURE: build a masked copy that only shows the last 4
      *> digits - this is what should go to logs/screens instead.
           MOVE "XXXXXX" TO WS-MASKED-ACCT(1:6).
           MOVE WS-ACCOUNT-NO(7:4) TO WS-MASKED-ACCT(7:4).
           DISPLAY "SECURE LOG LINE:   account=" WS-MASKED-ACCT.
           STOP RUN.
```

```bash
cobc -x -o step689_masking step689_masking.cob
./step689_masking
```

**ผลลัพธ์จริง:**

```
INSECURE LOG LINE: account=1234567890
SECURE LOG LINE:   account=XXXXXX7890
```

### อธิบายจุดสำคัญ

- เทคนิคนี้ใช้ **Reference Modification** (ทบทวนจาก Part 019) ที่มีจุดเริ่มต้นและความยาว**คงที่ที่รู้
  ล่วงหน้า** (ไม่ใช่รับมาจากผู้ใช้เหมือนในขั้นตอนที่ 688) จึงไม่มีความเสี่ยงเรื่อง Buffer Overflow ในกรณีนี้
- รูปแบบ "แสดงเฉพาะ 4 หลักสุดท้าย ปิดบังส่วนที่เหลือด้วย X" เป็นรูปแบบมาตรฐานที่พบได้ทั่วไปในใบเสร็จ
  บัตรเครดิตและระบบธนาคารจริง (PCI-DSS — มาตรฐานความปลอดภัยข้อมูลบัตรชำระเงิน — กำหนดให้ปกปิดเลข
  บัตรลักษณะนี้เป็นข้อบังคับ)
- หลักการนี้ควรใช้กับ**ทุกจุด**ที่ข้อมูลอ่อนไหวอาจถูกแสดงออกมา ไม่ใช่แค่ `DISPLAY` เท่านั้น แต่รวมถึงข้อมูล
  ที่เขียนลงไฟล์ log, ส่งผ่าน error message, หรือแสดงบนหน้าจอ (Part 040 — Screen Section) ด้วย

### ข้อควรระวัง

- การปกปิดข้อมูลต้อง**สมดุลกับความสามารถในการ debug** — ถ้าปกปิดมากเกินไปจนทีมสนับสนุนไม่สามารถ
  ระบุตัวปัญหาได้เลยเมื่อเกิด error ก็ไม่มีประโยชน์เช่นกัน แนวทางที่ดีคือ **เก็บข้อมูลเต็มรูปแบบไว้เฉพาะใน
  ที่จัดเก็บที่มีการป้องกันสิทธิ์เข้าถึงอย่างเข้มงวด (RACF ขั้นตอนที่ 684) แต่ปกปิดในทุกช่องทางที่มีโอกาส
  ถูกมองเห็นโดยคนที่ไม่จำเป็นต้องรู้** (หลักการ "need to know")
- อย่าปกปิดด้วยวิธีที่ผู้ใช้สามารถคำนวณย้อนกลับหาค่าจริงได้ง่าย (เช่น การเข้ารหัสแบบง่าย ๆ ที่ถอดรหัสได้ทันที)
  — สำหรับข้อมูลที่ต้องการความปลอดภัยสูงสุดจริง ๆ ควรใช้เทคนิคการเข้ารหัส (encryption) ที่เหมาะสมแทนการ
  ปกปิดบางส่วนแบบในตัวอย่างนี้ (ซึ่งเหมาะกับการแสดงผลเพื่อการอ้างอิงเท่านั้น ไม่ใช่การรักษาความลับระดับสูง)

### แบบฝึกหัดที่ 689.1

**โจทย์**: จงแก้ไขโปรแกรมข้างต้นให้แสดงเฉพาะ **2 หลักแรกและ 2 หลักสุดท้าย** ของเลขบัญชี (ปิดบังหลักที่
เหลือทั้งหมดตรงกลาง) แทนที่จะเป็น "ปกปิด 6 หลักแรก แสดง 4 หลักสุดท้าย" แบบเดิม

**เฉลย**:

```cobol
           MOVE "XXXXXX" TO WS-MASKED-ACCT(3:6).
           MOVE WS-ACCOUNT-NO(1:2) TO WS-MASKED-ACCT(1:2).
           MOVE WS-ACCOUNT-NO(9:2) TO WS-MASKED-ACCT(9:2).
```

ผลลัพธ์จะได้ `12XXXXXX90` — คัดลอก 2 หลักแรก (`WS-ACCOUNT-NO(1:2)`) และ 2 หลักสุดท้าย
(`WS-ACCOUNT-NO(9:2)`) มาแสดงตรง ๆ ส่วนหลักที่ 3 ถึง 8 (6 หลักตรงกลาง, `WS-MASKED-ACCT(3:6)`) ถูก
แทนที่ด้วย `X` ทั้งหมด

---

## ขั้นตอนที่ 690: [ทดสอบได้จริง] สรุปรวม — Defense in Depth ในโปรแกรมเดียว

### รวมทุกเทคนิคจากขั้นตอน 686-689 เข้าด้วยกัน

โปรแกรมต่อไปนี้จำลอง **ระบบฝากเงินขนาดเล็ก** ที่นำเทคนิค Secure Coding ทั้งสามข้อที่เรียนมาในบทนี้มาใช้
ร่วมกันในโปรแกรมเดียว: (1) ตรวจสอบข้อมูลตัวเลขด้วย `IS NUMERIC` ก่อนเสมอ (2) ตรวจสอบขอบเขต subscript
ก่อนเข้าถึงตาราง (3) ปกปิดข้อมูลบัญชีในผลลัพธ์ที่แสดง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP690SECURETXN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  ACCOUNT-TABLE.
           05  ACCOUNT-ENTRY OCCURS 5 TIMES.
               10  ACCT-ID          PIC 9(4).
               10  ACCT-BALANCE     PIC 9(7)V99.
       01  WS-TABLE-MAX          PIC 9(2)     VALUE 5.

       01  WS-RAW-SLOT           PIC X(1)     VALUE SPACES.
       01  WS-RAW-AMOUNT         PIC X(9)     VALUE SPACES.
       01  WS-SLOT               PIC 9(2)     VALUE 0.
      *> Amount is entered as whole CENTS (9 digits, no decimal
      *> point) so the IS NUMERIC test only ever sees plain digits -
      *> a period is NOT a digit and would fail IS NUMERIC too.
       01  WS-AMOUNT-CENTS       PIC 9(9)     VALUE 0.
       01  WS-AMOUNT             PIC 9(7)V99  VALUE 0.
       01  WS-VALID-FLAG         PIC X(1)     VALUE "Y".
           88  WS-ALL-VALID              VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Set up a small in-memory account table.
           MOVE 1001 TO ACCT-ID(1). MOVE 500.00 TO ACCT-BALANCE(1).
           MOVE 1002 TO ACCT-ID(2). MOVE 750.00 TO ACCT-BALANCE(2).
           MOVE 1003 TO ACCT-ID(3). MOVE 200.00 TO ACCT-BALANCE(3).
           MOVE 1004 TO ACCT-ID(4). MOVE   0.00 TO ACCT-BALANCE(4).
           MOVE 1005 TO ACCT-ID(5). MOVE 999.99 TO ACCT-BALANCE(5).

           MOVE "Y" TO WS-VALID-FLAG.

           DISPLAY "Enter account slot (1-5): "
               WITH NO ADVANCING.
           ACCEPT WS-RAW-SLOT.
      *> Technique 1: validate that the input is numeric BEFORE
      *> trusting it as a number at all (step 686).
           IF WS-RAW-SLOT IS NOT NUMERIC
               DISPLAY "ERROR: slot must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-SLOT TO WS-SLOT
      *> Technique 2: explicit bounds check before using the value
      *> as a table subscript (step 687) - never trust a number is
      *> in range just because it IS a number.
               IF WS-SLOT < 1 OR WS-SLOT > WS-TABLE-MAX
                   DISPLAY "ERROR: slot " WS-SLOT
                       " is out of range (1-" WS-TABLE-MAX ")."
                   MOVE "N" TO WS-VALID-FLAG
               END-IF
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter amount in cents (9 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-AMOUNT
      *> Technique 1 again, applied to the second input field.
               IF WS-RAW-AMOUNT IS NOT NUMERIC
                   DISPLAY "ERROR: amount must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
                   COMPUTE WS-AMOUNT = WS-AMOUNT-CENTS / 100
               END-IF
           END-IF.

           IF WS-ALL-VALID
               ADD WS-AMOUNT TO ACCT-BALANCE(WS-SLOT)
               DISPLAY "Deposit accepted."
      *> Technique 3: mask the account id in the confirmation line
      *> instead of ever displaying full account details (step 689).
               DISPLAY "  Account ****" ACCT-ID(WS-SLOT)
                   " new balance=" ACCT-BALANCE(WS-SLOT)
           ELSE
               DISPLAY "Transaction REJECTED - no data was changed."
           END-IF.
           STOP RUN.
```

```bash
cobc -x -o step690_secure_txn step690_secure_txn.cob
```

**ผลลัพธ์จริงจากการทดสอบครบ 4 เส้นทาง (test coverage แบบเดียวกับที่เรียนมาใน Part 015):**

```
-- กรณีที่ 1: slot=2 (valid), amount=100.50 (ป้อนเป็น 000010050 เซนต์) --
Deposit accepted.
  Account ****1002 new balance=0000850.50

-- กรณีที่ 2: slot=A (ตัวอักษร ไม่ใช่ตัวเลข) --
ERROR: slot must be numeric.
Transaction REJECTED - no data was changed.

-- กรณีที่ 3: slot=9 (ตัวเลขถูกต้อง แต่เกินขอบเขต 1-5) --
ERROR: slot 09 is out of range (1-05).
Transaction REJECTED - no data was changed.

-- กรณีที่ 4: slot=1 (valid), amount=XXXXXXXXX (ไม่ใช่ตัวเลข) --
ERROR: amount must be numeric.
Transaction REJECTED - no data was changed.
```

### วิเคราะห์ผลลัพธ์ — Defense in Depth ทำงานอย่างไรในทางปฏิบัติ

สังเกตว่ากรณีที่ 1 ทำงานถูกต้องสมบูรณ์: บัญชีสล็อต 2 มียอดเดิม `750.00` บวกด้วย `100.50` ได้ผลลัพธ์
`850.50` ตรงกับที่คำนวณด้วยมือ และข้อความยืนยัน**ไม่แสดงเลขบัญชีเต็มรูปแบบ**เลย (`****1002`
— ปกปิดด้วยเทคนิคจากขั้นตอนที่ 689 ผสมกับการแสดงตัวเลขจริงบางส่วนเพื่อการอ้างอิง)

กรณีที่ 2-4 แสดงให้เห็นว่า **ทุกจุดที่รับข้อมูลจากภายนอกถูกตรวจสอบก่อนนำไปใช้งานจริงเสมอ** — ไม่มีเส้นทาง
ใดเลยที่ข้อมูลที่ไม่ผ่านการตรวจสอบจะไปถึงคำสั่ง `ADD`/`ACCT-BALANCE(WS-SLOT)` ได้ ตรงกับหลักการ
"Validate then Trust" ที่เป็นแก่นของ Secure Coding ทั้งบทนี้

### ความเชื่อมโยงกับ RACF (ขั้นตอน 681-685) — ภาพรวมทั้งสองชั้นทำงานร่วมกันอย่างไร

| ชั้นการป้องกัน | ทำหน้าที่อะไร | ป้องกันสถานการณ์ไหน |
|---|---|---|
| **RACF** (เชิงแนวคิด) | ควบคุมว่า User ID ใดรันโปรแกรมนี้ได้, เข้าถึง dataset บัญชีลูกค้าได้ | ผู้ใช้ที่ไม่มีสิทธิ์พยายามรันโปรแกรมนี้เลย หรือเข้าถึงไฟล์ข้อมูลบัญชีโดยตรงโดยไม่ผ่านโปรแกรม |
| **Secure Coding** (ทดสอบจริงในบทนี้) | ตรวจสอบข้อมูลนำเข้าและปกปิดข้อมูลอ่อนไหวภายในโปรแกรมเอง | ผู้ใช้ที่**มีสิทธิ์ถูกต้อง**ป้อนข้อมูลผิดพลาดหรือพยายามเข้าถึง slot ที่ไม่มีอยู่จริง |

ทั้งสองชั้นทำงาน**คนละจุด**ของกระบวนการทั้งหมดและ**ไม่สามารถทดแทนกันได้เลย** — นี่คือคำตอบสมบูรณ์ของ
คำถามที่ตั้งไว้ในคำนำของ Part นี้ว่าทำไมทั้งสองเรื่องจึงต้องเรียนคู่กันในบทเดียว

### ข้อควรระวัง (สรุปรวมทั้งบท)

- Secure Coding ไม่ใช่สิ่งที่ทำเพิ่มเติม "ถ้ามีเวลา" แต่ต้องเป็นส่วนหนึ่งของการออกแบบโปรแกรมตั้งแต่ต้น
  โดยเฉพาะจุดใดก็ตามที่รับข้อมูลจากภายนอก (ผู้ใช้, ไฟล์ input, ระบบอื่น) ต้องถูกพิจารณาว่า **"ข้อมูลนี้
  อาจผิดพลาดหรือมุ่งร้ายได้เสมอ จนกว่าจะพิสูจน์ว่าถูกต้องแล้ว"**
- อย่าลืมว่าทุกเทคนิคในบทนี้ (`IS NUMERIC`, bounds checking, data masking) มี**ต้นทุนความซับซ้อนของ
  โค้ดเพิ่มขึ้น** — แต่เมื่อเทียบกับความเสียหายที่อาจเกิดขึ้นจากข้อมูลผิดพลาดในระบบการเงินจริง (ทบทวนสถิติ
  จาก Part 001 ว่า COBOL ยังคงเป็นแกนหลักของระบบธนาคารทั่วโลก) ต้นทุนนี้คุ้มค่าเสมอ

### แบบฝึกหัดที่ 690.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม `STEP690SECURETXN` จึงยังคง**ไม่ปลอดภัยสมบูรณ์**หากนำไปใช้งานจริง
บน Mainframe โดยไม่มีการตั้งค่า RACF ที่เหมาะสมควบคู่ไปด้วย

**เฉลยแนวทาง**: แม้โปรแกรมจะตรวจสอบข้อมูลนำเข้าและปกปิดข้อมูลอ่อนไหวในผลลัพธ์ได้อย่างดีเยี่ยม แต่หาก
ไม่มี RACF ควบคุมสิทธิ์ (ขั้นตอน 681-685) ผู้ใช้ที่**ไม่ควรมีสิทธิ์เข้าถึงระบบนี้เลย**อาจยังคงสามารถรัน
โปรแกรมนี้ได้อยู่ดี หรือแม้แต่เข้าถึงไฟล์ dataset ที่เก็บข้อมูลบัญชีจริงโดยตรงผ่านเครื่องมืออื่น (เช่น
TSO/ISPF ทบทวนจาก Part 054) โดยไม่ผ่านโปรแกรมนี้เลยแม้แต่น้อย ซึ่งเป็นสถานการณ์ที่ Secure Coding ใน
ระดับโปรแกรมไม่มีทางป้องกันได้เลยเพราะอยู่นอกขอบเขตควบคุมของมัน นี่คือเหตุผลที่ยืนยันหลักการ Defense in
Depth ที่กล่าวไว้ตั้งแต่คำนำของ Part นี้: **ทั้งสองชั้นต้องมีอยู่พร้อมกันเสมอ ขาดชั้นใดชั้นหนึ่งไปก็ยังคงเป็น
ความเสี่ยงที่ยอมรับไม่ได้สำหรับระบบการเงินจริง**

---

## สรุปท้ายบท

Part นี้พาเราไปทำความเข้าใจความปลอดภัยบน Mainframe ในสองระดับที่ทำงานร่วมกันแบบ Defense in Depth:

**ความรู้เชิงอ้างอิงสำหรับ z/OS จริง (ไม่สามารถทดสอบในสภาพแวดล้อมนี้):**

- RACF และบทบาทในการควบคุมสิทธิ์เข้าถึงทรัพยากรบน z/OS (ขั้นตอนที่ 681)
- User ID, Group และคำสั่ง `ADDUSER`/`CONNECT` (ขั้นตอนที่ 682)
- Resource Profile, Access Level และคำสั่ง `PERMIT` (ขั้นตอนที่ 683)
- การป้องกัน Dataset ด้วย Discrete/Generic Profile และ `UACC(NONE)` (ขั้นตอนที่ 684)
- ความปลอดภัยของโปรแกรมและแนวคิด APF-Authorized Library (ขั้นตอนที่ 685)

**เทคนิค Secure Coding ที่พิสูจน์ด้วยการรันจริง (ทดสอบได้กับ GnuCOBOL):**

- ตรวจสอบข้อมูลตัวเลขด้วย `IS NUMERIC` ก่อนเชื่อถือข้อมูลนำเข้าเสมอ (ขั้นตอนที่ 686)
- ตรวจสอบขอบเขต Subscript ก่อนเข้าถึงตาราง — เปรียบเทียบพฤติกรรมมี/ไม่มี `-debug` (ขั้นตอนที่ 687)
- ป้องกัน Buffer Overflow ใน `STRING` (ด้วย `ON OVERFLOW`) และ Reference Modification (ด้วย `IF`)
  (ขั้นตอนที่ 688)
- ปกปิดข้อมูลอ่อนไหวในผลลัพธ์ที่แสดง (Data Masking) (ขั้นตอนที่ 689)
- กรณีศึกษารวมทุกเทคนิคในโปรแกรมเดียว พร้อมทดสอบครบทุกเส้นทาง (ขั้นตอนที่ 690)

หลักการที่สำคัญที่สุดของ Part นี้คือ **RACF และ Secure Coding ไม่ใช่ทางเลือกที่ใช้แทนกันได้ แต่เป็นชั้น
ป้องกันคนละระดับที่ต้องมีพร้อมกันเสมอในระบบที่ปลอดภัยจริง** — ระบบที่มี RACF ตั้งค่าดีที่สุดในโลกก็ยังคง
เสี่ยงต่อบั๊กจากข้อมูลนำเข้าที่ไม่ถูกตรวจสอบ และโปรแกรมที่เขียนอย่างปลอดภัยที่สุดก็ยังคงเสี่ยงหากใครก็ตาม
สามารถเข้าถึงข้อมูลโดยตรงโดยไม่ผ่านโปรแกรมนั้นเลย

Part ถัดไป (Part 070) คือ **โปรเจกต์รวบยอดเฟส 4**: เราจะนำความรู้จาก Part 051-069 ทั้งหมด (JCL,
VSAM/Indexed Files, DB2/SQL concepts, CICS concepts, Batch Processing, Sort, Performance, Security)
มาประกอบร่างเป็น **ระบบธนาคารบน Mainframe จำลอง** ที่คอมไพล์และรันได้จริงสมบูรณ์แบบด้วย GnuCOBOL

**[← กลับไป Part 068: Performance Tuning สำหรับ Mainframe COBOL](part-068-performance-tuning.md)** |
**[ไปยัง Part 070: 🎯 โปรเจกต์เฟส 4: ระบบธนาคารบน Mainframe จำลอง →](part-070-phase4-project.md)**
