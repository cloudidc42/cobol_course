# Part 087: Microservices และ COBOL ในระบบกระจาย (ขั้นตอนที่ 861–870)

## คำนำของ Part นี้

Part 086 ขยายมุมมองจาก "หนึ่งระบบ" (Part 085) ไปสู่ "หลายระบบในองค์กรเดียวกัน" ที่สื่อสารกันผ่าน
Layered Architecture, Batch Scheduling, Copybook Governance, และ Interface Versioning — Part นี้จะขยาย
มุมมองต่อไปอีกขั้น: **สถาปัตยกรรมแบบกระจาย (Distributed Architecture)** และ **Microservices** ซึ่งเป็น
รูปแบบสถาปัตยกรรมที่ระบบสมัยใหม่จำนวนมากใช้กัน — คำถามสำคัญของ Part นี้คือ **"COBOL ซึ่งถือกำเนิดมาก่อน
แนวคิด Microservices หลายสิบปี จะเข้าไปมีบทบาทในโลกสถาปัตยกรรมแบบนี้ได้อย่างไร"**

เราจะเรียนรู้ว่า COBOL สามารถทำหน้าที่เป็น **Bounded Context Service** หนึ่งตัวในระบบ Microservices ได้
โดยไม่ต้องเขียนใหม่ทั้งหมด (ทบทวนแนวคิด Wrap/Encapsulate จาก Part 077/084/085), แนวคิด **Event-driven
Architecture** และวิธีที่ COBOL เข้าร่วมรูปแบบนี้ได้ผ่านไฟล์ธรรมดาที่ทำหน้าที่แทน Message Queue (พร้อม
คำเตือนที่ตรงไปตรงมาว่านี่เป็นการจำลองแนวคิด ไม่ใช่ระบบ Message Broker จริง), และแนวคิดสำคัญที่สุดสำหรับ
การเชื่อมต่อระหว่างงาน Batch กับ Microservices: **Idempotency** (ความสามารถทำงานซ้ำได้โดยไม่เกิดผลข้าง
เคียงซ้ำ) และ **Retry** (การลองใหม่เมื่อล้มเหลว) — ทุกแนวคิดจะพิสูจน์ด้วยโปรแกรม Producer/Consumer ที่
คอมไพล์และรันได้จริง

---

## ขั้นตอนที่ 861: ภาพรวม — COBOL ในโลกสถาปัตยกรรมแบบกระจาย

### Microservices คืออะไรโดยสังเขป

**Microservices** คือแนวทางการออกแบบระบบที่แบ่งแอปพลิเคชันขนาดใหญ่ออกเป็นบริการ (service) ขนาดเล็กที่
ทำงานอิสระจากกัน แต่ละบริการรับผิดชอบ **Bounded Context** หนึ่งเรื่อง (เช่น บริการจัดการลูกค้า, บริการ
คำนวณดอกเบี้ย, บริการแจ้งเตือน) สื่อสารกันผ่าน API หรือ Message ที่ชัดเจน แทนที่จะเป็นแอปพลิเคชันก้อนใหญ่
ก้อนเดียว (Monolith) ที่ทุกส่วนผูกติดกันแน่น

### ทำไมคำถามนี้ถึงสำคัญ: COBOL กับ Microservices ดูเหมือนจะขัดกันโดยธรรมชาติ

COBOL ถือกำเนิดในปี 1959 (Part 001) ในยุคที่แนวคิด "บริการขนาดเล็กที่สื่อสารผ่านเครือข่าย" ยังไม่มีอยู่
ด้วยซ้ำ ระบบ COBOL ดั้งเดิมมักถูกออกแบบมาเป็น **Monolith ขนาดใหญ่** ที่ทำงานแบบ Batch เป็นหลัก — คำถามที่
หลายองค์กรเผชิญคือ: จะนำสถาปัตยกรรม Microservices สมัยใหม่มาใช้ได้อย่างไร โดยไม่ต้องทิ้ง COBOL ที่ทำงาน
ถูกต้องมานานหลายสิบปี

### คำตอบ: COBOL ไม่จำเป็นต้องเปลี่ยนตัวเอง แต่ต้อง "สวมบทบาท" ใหม่

คำตอบที่ใช้ได้ผลจริงในอุตสาหกรรมคือ: **มองระบบ COBOL หนึ่งระบบ (หรือกลุ่มโปรแกรมที่เกี่ยวข้องกัน) เป็น
"หนึ่ง Microservice"** ที่มี Bounded Context ชัดเจนของตัวเอง แล้วให้มันสื่อสารกับส่วนอื่นของระบบผ่าน
ช่องทางมาตรฐาน (REST API ตาม Part 073, หรือ Message-based ตามที่ Part นี้จะสอน) — นี่คือแนวทางเดียวกับ
ที่ Part 085 สร้าง AR-MINI ขึ้นมา เพียงแต่ Part นี้จะมองมันในบริบทที่กว้างขึ้น: เมื่อมี AR-MINI, บริการ
ลูกค้า, บริการแจ้งเตือน ฯลฯ หลายตัวทำงานร่วมกัน จะออกแบบการสื่อสารระหว่างกันอย่างไร

### แผนภาพภาพรวมของ Part นี้

```
+------------------+     HTTP/REST      +------------------+
|  Web/Mobile App  | -----------------> |  API Gateway     |
+------------------+                    +--------+---------+
                                                  |
                     +----------------------------+----------------------------+
                     v                            v                            v
          +--------------------+       +--------------------+       +--------------------+
          | COBOL Service A    |       | COBOL Service B     |       | Non-COBOL Service   |
          | (เช่น AR-MINI       |       | (เช่นบริการลูกค้า)    |       | (เช่น Notification   |
          |  จาก Part 085)      |       |                      |       |  Service, Python)   |
          +----------+----------+       +----------+-----------+       +----------+----------+
                     |                             |                              ^
                     |     เขียนเหตุการณ์ลง Queue    |                              |
                     +---------------> [Event Queue] <---------------------------+
                                       (Step 863-865:
                                        file-based analog)
```

### ข้อควรระวัง

- Part นี้**ไม่ได้สอนว่า COBOL ควรถูกเขียนใหม่เป็นสไตล์ Microservices ภายในตัวมันเอง** (เช่น แยก
  `PROGRAM-ID` ย่อย ๆ เป็น "Micro-paragraph") — Microservices เป็นแนวคิดระดับสถาปัตยกรรมของ**ทั้งระบบ**
  ไม่ใช่รูปแบบการเขียนโค้ดภายในโปรแกรมเดียว
- อย่าใช้คำว่า "Microservices" เป็นเป้าหมายในตัวมันเอง — เป้าหมายที่แท้จริงคือ**ความสามารถในการพัฒนาและ
  Deploy แต่ละส่วนของระบบได้อย่างอิสระจากกัน** สถาปัตยกรรมแบบใดที่ตอบโจทย์นี้ได้ก็ถือว่าใช้งานได้ ไม่
  จำเป็นต้องยึดติดกับคำนิยามทางทฤษฎีอย่างเคร่งครัด

### แบบฝึกหัดที่ 861.1

**โจทย์**: จงอธิบายว่าทำไมการมอง AR-MINI จาก Part 085 เป็น "หนึ่ง Bounded Context" (บริการคำนวณและ
บันทึกใบแจ้งหนี้) แทนที่จะพยายามรวมมันเข้ากับบริการลูกค้าทั้งหมดในระบบเดียว ถึงสอดคล้องกับหลักการ
Microservices

**เฉลยแนวทาง**: เพราะหลักการสำคัญของ Microservices คือ **High Cohesion, Low Coupling** ภายในหนึ่งบริการ
ควรมีความเกี่ยวข้องกันแน่นแฟ้น (คำนวณใบแจ้งหนี้และบันทึกยอดคงเหลือเป็นเรื่องเดียวกันในบริบททางธุรกิจ)
ในขณะที่ระหว่างบริการควรมีการพึ่งพากันน้อยที่สุด การรวมบริการคำนวณใบแจ้งหนี้เข้ากับบริการจัดการข้อมูล
ลูกค้าทั้งหมด (ที่อาจมีเรื่องที่อยู่, ประวัติการติดต่อ, สิทธิพิเศษ ฯลฯ) จะทำให้ Bounded Context กว้างเกิน
ไปและมีความรับผิดชอบหลายเรื่องปนกัน ขัดกับหลักการที่ต้องการให้แต่ละบริการมีเหตุผลเดียวในการเปลี่ยนแปลง
(Single Responsibility ในระดับบริการ)

---

## ขั้นตอนที่ 862: COBOL เป็น Bounded Context Service หลัง API Gateway

### ทบทวนแนวคิด API Gateway จาก Part 073

Part 073 สอนการสร้าง REST API Wrapper รอบ COBOL — ในบริบท Microservices แนวคิดนี้ขยายออกไปอีกขั้น:
แทนที่ Client จะเรียก Wrapper ของแต่ละบริการโดยตรง มักมี **API Gateway** เป็นจุดเข้าเดียว (single entry
point) ที่รับ Request ทั้งหมดจากภายนอก แล้วจึงส่งต่อ (route) ไปยังบริการที่ถูกต้องภายใน

```
                      +------------------+
   Client Request --> |   API Gateway    |
   (เช่น "คำนวณ         |   (จุดเดียวที่     |
   ใบแจ้งหนี้")          |   ภายนอกเห็น)     |
                      +--------+---------+
                               |
                path = /invoice/*
                               v
                      +------------------+
                      | AR-MINI Service  |
                      | (ar_server.py -> |
                      |  ARCALC.cob)     |
                      | จาก Part 085      |
                      +------------------+
```

### ทำไมต้องมี API Gateway แทนที่จะให้ Client เรียกแต่ละบริการตรง ๆ

1. **จุดเดียวสำหรับ Authentication/Authorization** — ตรวจสอบสิทธิ์ผู้เรียกครั้งเดียวที่ Gateway แทนที่
   จะต้องทำซ้ำในทุกบริการ (รวมถึง COBOL Wrapper ที่ Part 073/085 สร้างไว้ ซึ่งยังไม่มีการตรวจสอบสิทธิ์เลย
   ตามที่ Part 085 ขั้นตอนที่ 850 ระบุไว้ว่าเป็นข้อจำกัดที่ต้องเติมเต็มต่อไป)
2. **ซ่อนรายละเอียดภายในจาก Client** — Client ไม่จำเป็นต้องรู้ว่าบริการไหนเขียนด้วย COBOL บริการไหน
   เขียนด้วย Python เห็นแค่ URL เดียวที่สอดคล้องกัน
3. **จัดการ Cross-cutting Concerns อื่น ๆ ที่จุดเดียว** เช่น Rate Limiting, Logging, การแปลง Protocol

### ตัวอย่างแนวคิดการ Route คำขอ (ภาพประกอบเชิงแนวคิด)

```
Gateway Routing Table:
  /invoice/*        -> AR-MINI Service (COBOL, port 8085, Part 085)
  /customer/*       -> Customer Service (สมมติเป็น Java, port 8090)
  /notify/*         -> Notification Service (สมมติเป็น Python, port 8095)
```

จากมุมมองของ Client ทุกคำขอดูเหมือนไปที่ระบบเดียวกัน (`https://api.company.example/`) แต่ Gateway เป็น
ผู้ตัดสินใจว่าจะส่งต่อไปยังบริการใดจริง ๆ ตามเส้นทาง (path) ที่ระบุมา — Client ไม่มีทางรู้เลยว่าเบื้องหลัง
`/invoice/calculate` คือ Python ที่เรียก COBOL Subprocess อยู่

### สิ่งที่ COBOL Service ต้องมีเพื่อทำงานร่วมกับ API Gateway ได้ดี

1. **Health Check Endpoint** — Gateway (และเครื่องมือ Monitoring) ต้องตรวจสอบได้ว่าบริการยังทำงานอยู่
   หรือไม่ ผ่าน Endpoint ง่าย ๆ (เช่น `GET /health` ที่ตอบกลับทันทีโดยไม่ต้องเรียก COBOL Subprocess จริง)
2. **Timeout ที่เหมาะสม** — เนื่องจากการเรียก COBOL ผ่าน Subprocess (ตามที่ Part 085 ขั้นตอนที่ 847 ทำ)
   มี Overhead มากกว่าการเรียกฟังก์ชันในภาษาเดียวกันโดยตรง Gateway ต้องตั้งค่า Timeout ที่ยอมรับ Overhead
   นี้ได้ แต่ไม่นานเกินจนทำให้ผู้ใช้รอนาน
3. **Error Response ที่เป็นมาตรฐานเดียวกับบริการอื่น** — ตามที่ Part 085 ขั้นตอนที่ 847 ออกแบบไว้แล้ว
   (HTTP status code + JSON error message) ควรสอดคล้องกับรูปแบบที่บริการอื่นในระบบใช้ เพื่อให้ Client
   จัดการ Error ได้อย่างสม่ำเสมอไม่ว่าจะเรียกบริการใด

### ข้อควรระวัง

- API Gateway เป็น Single Point of Failure ที่อาจเกิดขึ้นได้หากออกแบบไม่ดี — ต้องมีการวางแผนเรื่องความ
  พร้อมใช้งานสูง (High Availability) ของตัว Gateway เองด้วย ไม่ใช่แค่บริการที่อยู่เบื้องหลัง
- อย่าให้ Business Logic ใด ๆ ไปอยู่ที่ Gateway — Gateway ควรทำหน้าที่ Route/Authenticate/Log เท่านั้น
  กฎทางธุรกิจทั้งหมดต้องอยู่ใน Business Logic Layer ของแต่ละบริการ (ทบทวนหลักการ Layered Architecture
  จาก Part 086 ขั้นตอนที่ 852)

### แบบฝึกหัดที่ 862.1

**โจทย์**: จงออกแบบ Health Check Endpoint อย่างง่ายที่ควรเพิ่มเข้าไปใน `ar_server.py` จาก Part 085 เพื่อ
ให้ API Gateway ตรวจสอบสถานะของบริการนี้ได้ โดยไม่ต้องเรียก COBOL Subprocess จริงทุกครั้ง

**เฉลยแนวทาง**: เพิ่มเมธอด `do_GET` ใน Handler ที่ตรวจสอบ `self.path == "/health"` แล้วตอบกลับ HTTP 200
พร้อม JSON `{"status": "ok"}` ทันที โดยไม่เรียก `subprocess.run([ARCALC_PATH], ...)` เลย — เหตุผลคือ
Health Check ควรตอบเร็วที่สุดเท่าที่จะทำได้เพื่อไม่ให้เป็นภาระต่อระบบ Monitoring ที่มักเรียกถี่มาก (เช่น
ทุก 5-10 วินาที) การตรวจสอบเพียงว่า "Python process ของ Wrapper ยังทำงานอยู่และตอบสนองได้" ก็เพียงพอแล้ว
สำหรับ Health Check พื้นฐาน ส่วนการตรวจสอบว่า COBOL Executable เรียกใช้งานได้จริงหรือไม่อาจทำเป็น Endpoint
แยกต่างหาก (เช่น `/health/deep`) ที่เรียกไม่บ่อยเท่า

---

## ขั้นตอนที่ 863: แนวคิด Event-Driven Architecture และข้อจำกัดของสภาพแวดล้อมการเรียนรู้นี้

### Event-Driven Architecture คืออะไร

แทนที่บริการหนึ่งจะเรียกอีกบริการหนึ่งโดยตรงแล้วรอผลลัพธ์ทันที (Synchronous, แบบที่ Part 073/085 ทำ)
**Event-Driven Architecture** ให้บริการหนึ่ง "ประกาศเหตุการณ์" (publish an event) เช่น "มีการสร้าง
ใบแจ้งหนี้ใหม่" ลงใน **Message Queue** หรือ **Message Broker** กลาง แล้วบริการอื่นที่สนใจเหตุการณ์นั้น
(subscriber) จะมารับไปประมวลผลเมื่อพร้อม — ผู้ส่งไม่ต้องรอผู้รับ และผู้รับไม่ต้องรู้จักผู้ส่งโดยตรง

```
Synchronous (Part 073/085):          Event-Driven (Part นี้):
Client -> Service A -> รอผลลัพธ์      Service A -> เขียนเหตุการณ์ -> [Queue]
       <- ผลลัพธ์กลับทันที                                              |
                                      Service B <- อ่านเหตุการณ์เมื่อพร้อม
```

### เครื่องมือจริงในอุตสาหกรรม: Kafka, RabbitMQ

ในโลกอุตสาหกรรมจริง Event-Driven Architecture มักใช้ **Message Broker** เฉพาะทาง เช่น **Apache Kafka**
หรือ **RabbitMQ** ซึ่งมีความสามารถขั้นสูงมากมาย: การรับประกันลำดับข้อความ (ordering), การกระจายภาระ
(partitioning), ความทนทานต่อความล้มเหลว (durability), และกลไกยืนยันการรับข้อความ (acknowledgment)
ที่ซับซ้อน

### คำประกาศที่ตรงไปตรงมา: Part นี้ไม่มี Message Broker จริงให้ใช้

**สภาพแวดล้อมการเรียนรู้ของหลักสูตรนี้ไม่มี Kafka หรือ RabbitMQ ติดตั้งอยู่** (ทั้งสองเป็นซอฟต์แวร์ขนาด
ใหญ่ที่ต้องติดตั้งแยกต่างหากและมักต้องใช้ทรัพยากรระบบมาก) ดังนั้น Part นี้จะใช้ **ไฟล์ธรรมดา** (Flat
File) เป็น **ตัวแทนแนวคิด (Conceptual Analog)** ของ Message Queue แทน — ต้องเข้าใจให้ชัดเจนว่า:

| คุณสมบัติ | Kafka/RabbitMQ จริง | ไฟล์ธรรมดาใน Part นี้ |
|---|---|---|
| การรับประกันลำดับ | มีกลไกซับซ้อนรองรับ | อาศัยลำดับการเขียนไฟล์แบบง่าย ๆ เท่านั้น |
| หลายผู้บริโภคพร้อมกัน | รองรับ Consumer Group | ไม่รองรับในตัวอย่างนี้ (ผู้บริโภคเดียว) |
| ประสิทธิภาพ | ออกแบบมาสำหรับปริมาณสูงมาก | เหมาะกับการเรียนรู้/ปริมาณต่ำเท่านั้น |
| ความทนทาน | มีกลไก Replication ข้าม Server | ขึ้นกับความทนทานของไฟล์ระบบเดียว |
| **แนวคิดพื้นฐาน (Producer/Consumer, ข้อความเข้าคิว)** | **มี** | **มี (นี่คือสิ่งที่ Part นี้สอน)** |

### ทำไมการสอนด้วยไฟล์ธรรมดายังคงมีคุณค่า

แม้จะไม่มีความสามารถขั้นสูงของ Message Broker จริง แต่ **แนวคิดพื้นฐานที่สำคัญที่สุด** — การแยก Producer
ออกจาก Consumer โดยสิ้นเชิง, การประมวลผลแบบ "อย่างน้อยหนึ่งครั้ง" (at-least-once delivery), และความจำเป็น
ของ **Idempotency** ในฝั่งผู้บริโภค — เป็นแนวคิดเดียวกันทุกประการไม่ว่าจะใช้ไฟล์ธรรมดาหรือ Kafka จริง
การเข้าใจแนวคิดเหล่านี้ผ่านตัวอย่างที่เรียบง่ายและทดสอบได้จริงจะช่วยให้เมื่อไปเจอ Message Broker จริงใน
งานจริง จะเข้าใจ "ทำไม" มันถูกออกแบบมาแบบนั้นได้เร็วขึ้นมาก

### ออกแบบระบบ "คิว" แบบไฟล์สำหรับ Part นี้

```
+-------------+       เขียนต่อท้าย        +---------------+       อ่านทีละบรรทัด        +-------------+
| PRODUCER.cob| -----------------------> |  QUEUE.DAT     | -------------------------> | CONSUMER.cob |
| (สร้าง        |  (append-only,           |  (แทน Topic/    |  (ตรวจสอบซ้ำด้วย            | (ประมวลผล    |
|  ธุรกรรมใหม่) |   เหมือน Kafka topic     |   Queue)        |   APPLIED.LOG ก่อนนำไป      |  แบบ idempotent)|
|             |   ในระดับแนวคิด)          |                |   ใช้จริงทุกครั้ง)          |             |
+-------------+                          +---------------+                            +-------------+
```

### ข้อควรระวัง

- **ห้ามนำโค้ดในหลักสูตรนี้ไปใช้แทน Message Broker จริงในระบบ Production** — มันขาดคุณสมบัติสำคัญมากมาย
  (การจัดการ Concurrent Access ที่ปลอดภัย, ความทนทานต่อความล้มเหลวของดิสก์, ประสิทธิภาพในระดับใหญ่)
  ที่ระบบจริงต้องการ นี่เป็นเพียงเครื่องมือสอนแนวคิดเท่านั้น
- ในงานจริง หากองค์กรตัดสินใจใช้ Event-Driven Architecture ควรพิจารณา Message Broker ที่พิสูจน์แล้วใน
  อุตสาหกรรม (Kafka, RabbitMQ, Amazon SQS ฯลฯ) และ COBOL จะเชื่อมต่อกับสิ่งเหล่านี้ผ่าน Client Library
  หรือ Wrapper ภาษาอื่น (คล้ายกับที่ REST Wrapper ใน Part 073/085 ทำหน้าที่แปลระหว่าง COBOL กับ HTTP)

### แบบฝึกหัดที่ 863.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ไฟล์ธรรมดาแทน Message Broker จริงถึง**เพียงพอ**สำหรับจุดประสงค์การเรียนรู้
แนวคิด Idempotency แม้จะไม่เพียงพอสำหรับระบบ Production จริง

**เฉลยแนวทาง**: เพราะแนวคิด Idempotency เกี่ยวข้องกับ**วิธีที่ผู้บริโภคจัดการกับข้อความที่อาจถูกส่งซ้ำ**
ไม่ได้เกี่ยวข้องกับกลไกภายในที่ซับซ้อนของ Message Broker เอง — ไม่ว่าข้อความจะมาจากไฟล์ธรรมดาหรือ Kafka
จริง หลักการที่ผู้บริโภคต้อง "ตรวจสอบว่าเคยประมวลผลข้อความนี้ไปแล้วหรือยังก่อนทำงานซ้ำ" นั้นเหมือนกัน
ทุกประการ สิ่งที่ไฟล์ธรรมดา**ขาดไป**เมื่อเทียบกับ Message Broker จริง (เช่น การรับประกันความทนทานของ
ข้อมูลข้าม Server, ประสิทธิภาพสูง) เป็นเรื่องของ**โครงสร้างพื้นฐาน** (infrastructure) ไม่ใช่เรื่องของ
**แนวคิดการออกแบบโปรแกรมผู้บริโภค** ซึ่งเป็นสิ่งที่ Part นี้ต้องการสอน

---

## ขั้นตอนที่ 864: สร้าง Producer — โปรแกรมเขียนข้อความเข้าคิว

### ออกแบบรูปแบบข้อความ (Message Format)

แต่ละข้อความในคิวแทนหนึ่ง "เหตุการณ์ชำระเงิน" ประกอบด้วย: `MSG-ID` (รหัสข้อความไม่ซ้ำกัน, สำคัญมากสำหรับ
Idempotency ในขั้นตอนถัดไป), `CUST-ID` (รหัสลูกค้า), และ `AMOUNT` (จำนวนเงิน)

### ซอร์สโค้ดฉบับเต็ม: PRODUCER.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PRODUCER.
       AUTHOR. COBOL-COURSE.
      *> Simple file-based "queue" producer. This is an HONEST
      *> stand-in for a real broker (Kafka/RabbitMQ): one append-
      *> only flat file plays the role of a topic/queue. Every
      *> message gets its own sequence number, generated here from
      *> a small counter file so repeated runs keep numbering up
      *> instead of restarting at 1.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT QUEUE-FILE ASSIGN TO "QUEUE.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-QUEUE-STATUS.
           SELECT SEQ-FILE ASSIGN TO "PRODSEQ.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-SEQ-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  QUEUE-FILE.
       01  QUEUE-LINE               PIC X(20).

       FD  SEQ-FILE.
       01  SEQ-LINE                 PIC 9(6).

       WORKING-STORAGE SECTION.
       01  WS-QUEUE-STATUS          PIC XX.
       01  WS-SEQ-STATUS            PIC XX.
       01  WS-NEXT-ID               PIC 9(6) VALUE 0.
       01  WS-HOW-MANY              PIC 9(2).
       01  WS-COUNTER               PIC 9(2).
       01  WS-CUST-ID               PIC 9(5) VALUE 10001.
       01  WS-AMOUNT                PIC 9(7)V99 VALUE 500.00.
       01  WS-OUT-REC.
           05  OUT-MSG-ID           PIC 9(6).
           05  OUT-CUST-ID          PIC 9(5).
           05  OUT-AMOUNT           PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-HOW-MANY FROM CONSOLE
           PERFORM READ-LAST-SEQUENCE

           OPEN EXTEND QUEUE-FILE
           IF WS-QUEUE-STATUS = "35"
               OPEN OUTPUT QUEUE-FILE
           END-IF
           PERFORM VARYING WS-COUNTER FROM 1 BY 1
                   UNTIL WS-COUNTER > WS-HOW-MANY
               ADD 1 TO WS-NEXT-ID
               MOVE WS-NEXT-ID TO OUT-MSG-ID
               MOVE WS-CUST-ID TO OUT-CUST-ID
               MOVE WS-AMOUNT  TO OUT-AMOUNT
               MOVE WS-OUT-REC TO QUEUE-LINE
               WRITE QUEUE-LINE
               DISPLAY "PRODUCED MSG-ID=" WS-NEXT-ID
               ADD 100.00 TO WS-AMOUNT
           END-PERFORM
           CLOSE QUEUE-FILE

           PERFORM SAVE-LAST-SEQUENCE
           STOP RUN.

       READ-LAST-SEQUENCE.
           OPEN INPUT SEQ-FILE
           IF WS-SEQ-STATUS = "00"
               READ SEQ-FILE
                   AT END MOVE 0 TO WS-NEXT-ID
                   NOT AT END MOVE SEQ-LINE TO WS-NEXT-ID
               END-READ
               CLOSE SEQ-FILE
           ELSE
               MOVE 0 TO WS-NEXT-ID
           END-IF.

       SAVE-LAST-SEQUENCE.
           OPEN OUTPUT SEQ-FILE
           MOVE WS-NEXT-ID TO SEQ-LINE
           WRITE SEQ-LINE
           CLOSE SEQ-FILE.
```

### อธิบายจุดสำคัญ

- **`OPEN EXTEND QUEUE-FILE`** — เปิดไฟล์แบบ**ต่อท้าย** (append) แทนที่จะเขียนทับ (`OPEN OUTPUT`) เพื่อ
  ให้ข้อความเก่าที่ผู้บริโภคยังไม่ได้อ่านไม่หายไป ตรงกับพฤติกรรมของ Message Queue จริงที่ข้อความจะคงอยู่
  จนกว่าจะถูกอ่าน (หรือหมดอายุตามนโยบาย retention)
- **`PRODSEQ.DAT`** เก็บเลขลำดับล่าสุดที่เคยใช้ไว้ต่างหาก ทำให้ `MSG-ID` **ไม่ซ้ำกันเลยแม้จะรัน Producer
  หลายครั้ง** — คุณสมบัติ "ไม่ซ้ำกัน" ของ `MSG-ID` นี้คือกุญแจสำคัญที่ทำให้ Consumer ตรวจสอบ Idempotency
  ได้ในขั้นตอนถัดไป
- **`IF WS-QUEUE-STATUS = "35" OPEN OUTPUT QUEUE-FILE`** — จัดการกรณีที่ไฟล์คิวยังไม่เคยถูกสร้างมาก่อน
  (การรันครั้งแรกสุด) เพราะ GnuCOBOL's `OPEN EXTEND` ต้องการให้ไฟล์มีอยู่แล้วเท่านั้น (ทบทวนเทคนิคการ
  ตรวจสอบ File Status จาก Part 030)

### ทดสอบจริง: ผลิตข้อความสองรอบ

```bash
cobc -x -o producer producer.cob
rm -f QUEUE.DAT PRODSEQ.DAT

echo "=== first batch of 3 (new file) ==="
echo "03" | ./producer
cat QUEUE.DAT

echo "=== second batch of 2 (extend) ==="
echo "02" | ./producer
cat QUEUE.DAT
```

**ผลลัพธ์จริงที่ได้:**

```
=== first batch of 3 (new file) ===
PRODUCED MSG-ID=000001
PRODUCED MSG-ID=000002
PRODUCED MSG-ID=000003
00000110001000050000
00000210001000060000
00000310001000070000
=== second batch of 2 (extend) ===
PRODUCED MSG-ID=000004
PRODUCED MSG-ID=000005
00000110001000050000
00000210001000060000
00000310001000070000
00000410001000050000
00000510001000060000
```

สังเกตว่าการรันครั้งที่สองเริ่มนับต่อจาก `MSG-ID=000004` ไม่ใช่เริ่มใหม่ที่ `000001` — และข้อความเก่าจาก
รอบแรก (3 บรรทัดแรก) ยังคงอยู่ในไฟล์ครบถ้วน ไม่ถูกเขียนทับ

### ข้อควรระวัง

- `PRODSEQ.DAT` เป็นจุดเดียวที่ Producer พึ่งพาเพื่อรับประกัน `MSG-ID` ไม่ซ้ำกัน — หากมี Producer หลาย
  Instance ทำงานพร้อมกัน (Concurrent) โดยต่างก็อ่าน/เขียน `PRODSEQ.DAT` เดียวกัน จะเกิด Race Condition
  ที่ทำให้ `MSG-ID` ซ้ำกันได้ (ปัญหานี้ Message Broker จริงอย่าง Kafka แก้ด้วยกลไกภายในที่ซับซ้อนกว่านี้มาก
  เช่น Partition-level Offset ที่จัดการโดย Broker เอง ไม่ใช่ผู้ผลิตข้อความ)
- ในตัวอย่างนี้ Producer ไม่มีกลไกตรวจสอบว่า Consumer ตามทันหรือไม่ — ไฟล์ `QUEUE.DAT` จะโตขึ้นเรื่อย ๆ
  หากไม่มีการลบข้อความเก่าที่ถูกประมวลผลไปแล้ว (Message Broker จริงมีนโยบาย Retention/Compaction จัดการ
  เรื่องนี้โดยอัตโนมัติ)

### แบบฝึกหัดที่ 864.1

**โจทย์**: จงอธิบายว่าทำไม `MSG-ID` ต้องมาจากไฟล์ตัวนับ (`PRODSEQ.DAT`) แทนที่จะใช้ค่าที่คำนวณจาก
เวลาปัจจุบัน (Timestamp) เช่น วัน-เวลาที่ผลิตข้อความ

**เฉลยแนวทาง**: เพราะ Timestamp มีความเสี่ยงที่จะซ้ำกันได้หากมีการผลิตข้อความมากกว่าหนึ่งข้อความในหน่วย
เวลาที่ Timestamp วัดได้ละเอียดถึง (เช่น ถ้า Timestamp ละเอียดแค่ระดับวินาที และมีการผลิต 2 ข้อความ
ภายในวินาทีเดียวกัน ทั้งสองจะได้ `MSG-ID` เดียวกัน) ในขณะที่ตัวนับที่เพิ่มค่าทีละ 1 อย่างต่อเนื่อง
(Sequential Counter) รับประกันได้ว่าทุกข้อความจะได้ค่าไม่ซ้ำกันอย่างแน่นอน ตราบใดที่มีเพียง Producer
เดียวที่เข้าถึงตัวนับนี้ในแต่ละครั้ง (ตามข้อจำกัดที่ระบุไว้ในข้อควรระวังข้างต้น)

---

## ขั้นตอนที่ 865: สร้าง Consumer — โปรแกรมอ่านข้อความแบบ Idempotent

### แนวคิด: ทำไม "อ่านแล้วประมวลผล" เพียงอย่างเดียวไม่เพียงพอ

หากออกแบบ Consumer แบบง่ายที่สุด (อ่านทุกบรรทัดในคิวแล้วประมวลผลทันที) จะเกิดปัญหาทันทีที่ Consumer
ต้องรันซ้ำ (เช่น รันผิดพลาดโดยไม่ได้ตั้งใจ, หรือระบบ Retry อัตโนมัติหลังจาก Timeout) — ข้อความเดิมจะถูก
ประมวลผล**ซ้ำ**ทำให้ยอดเงินถูกบวกซ้ำสองครั้งหรือมากกว่า นี่คือปัญหาคลาสสิกของระบบแบบ **At-Least-Once
Delivery** (ข้อความรับประกันว่าจะถูกส่ง/ประมวลผล "อย่างน้อยหนึ่งครั้ง" แต่อาจมากกว่าหนึ่งครั้งได้) ที่
Message Broker ส่วนใหญ่ในโลกจริงใช้เป็นค่าเริ่มต้น (เพราะการรับประกัน "พอดีหนึ่งครั้ง" หรือ
Exactly-Once นั้นทำได้ยากและมีต้นทุนสูงกว่ามาก)

**Idempotency** คือคุณสมบัติที่ทำให้การประมวลผลซ้ำ**ไม่ส่งผลกระทบซ้ำ** — คำตอบมาตรฐานคือให้ Consumer
บันทึกว่า `MSG-ID` ใดถูกประมวลผลไปแล้วบ้าง แล้วตรวจสอบก่อนทุกครั้งว่าเคยทำไปแล้วหรือยัง

### ซอร์สโค้ดฉบับเต็ม: CONSUMER.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CONSUMER.
       AUTHOR. COBOL-COURSE.
      *> File-based "queue" consumer with an idempotent apply step.
      *> Re-running this program against the SAME QUEUE.DAT (as a
      *> retry after a timeout or a crash would do) must not double
      *> count any message - APPLIED.LOG is the de-duplication
      *> ledger that makes "at-least-once delivery" safe to consume.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT QUEUE-FILE ASSIGN TO "QUEUE.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT APPLIED-FILE ASSIGN TO "APPLIED.LOG"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-APPLIED-STATUS.
           SELECT TOTAL-FILE ASSIGN TO "TOTAL.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-TOTAL-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  QUEUE-FILE.
       01  QUEUE-LINE                PIC X(20).

       FD  APPLIED-FILE.
       01  APPLIED-LINE              PIC 9(6).

       FD  TOTAL-FILE.
       01  TOTAL-LINE                PIC 9(9)V99.

       WORKING-STORAGE SECTION.
       01  WS-IN-REC.
           05  IN-MSG-ID             PIC 9(6).
           05  IN-CUST-ID            PIC 9(5).
           05  IN-AMOUNT             PIC 9(7)V99.

       01  WS-APPLIED-STATUS         PIC XX.
       01  WS-TOTAL-STATUS           PIC XX.
       01  WS-EOF                    PIC X VALUE "N".
       01  WS-RUNNING-TOTAL          PIC 9(9)V99 VALUE 0.

       01  WS-APPLIED-TABLE.
           05  WS-APPLIED-ENTRY PIC 9(6)
                   OCCURS 100 TIMES INDEXED BY APL-IDX.
       01  WS-APPLIED-COUNT          PIC 9(3) VALUE 0.
       01  WS-FOUND-FLAG             PIC X VALUE "N".
           88  ALREADY-APPLIED             VALUE "Y".

       01  WS-NEW-COUNT              PIC 9(4) VALUE 0.
       01  WS-SKIP-COUNT             PIC 9(4) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM LOAD-APPLIED-LOG
           PERFORM LOAD-RUNNING-TOTAL
           PERFORM PROCESS-QUEUE
           PERFORM SAVE-RUNNING-TOTAL

           DISPLAY "NEW MESSAGES APPLIED: " WS-NEW-COUNT
           DISPLAY "DUPLICATES SKIPPED:   " WS-SKIP-COUNT
           DISPLAY "RUNNING TOTAL:        " WS-RUNNING-TOTAL
           STOP RUN.

       LOAD-APPLIED-LOG.
           OPEN INPUT APPLIED-FILE
           IF WS-APPLIED-STATUS = "00"
               PERFORM UNTIL WS-EOF = "Y"
                   READ APPLIED-FILE
                       AT END MOVE "Y" TO WS-EOF
                       NOT AT END
                           ADD 1 TO WS-APPLIED-COUNT
                           MOVE APPLIED-LINE TO
                               WS-APPLIED-ENTRY(WS-APPLIED-COUNT)
                   END-READ
               END-PERFORM
               CLOSE APPLIED-FILE
           END-IF
           MOVE "N" TO WS-EOF.

       LOAD-RUNNING-TOTAL.
           OPEN INPUT TOTAL-FILE
           IF WS-TOTAL-STATUS = "00"
               READ TOTAL-FILE
                   AT END MOVE 0 TO WS-RUNNING-TOTAL
                   NOT AT END MOVE TOTAL-LINE TO WS-RUNNING-TOTAL
               END-READ
               CLOSE TOTAL-FILE
           END-IF.

       PROCESS-QUEUE.
           OPEN INPUT QUEUE-FILE
           OPEN EXTEND APPLIED-FILE
           IF WS-APPLIED-STATUS = "35"
               OPEN OUTPUT APPLIED-FILE
           END-IF
           PERFORM UNTIL WS-EOF = "Y"
               READ QUEUE-FILE INTO WS-IN-REC
                   AT END MOVE "Y" TO WS-EOF
                   NOT AT END PERFORM HANDLE-ONE-MESSAGE
               END-READ
           END-PERFORM
           CLOSE QUEUE-FILE
           CLOSE APPLIED-FILE.

       HANDLE-ONE-MESSAGE.
           MOVE "N" TO WS-FOUND-FLAG
           SET APL-IDX TO 1
           SEARCH WS-APPLIED-ENTRY VARYING APL-IDX
               AT END CONTINUE
               WHEN WS-APPLIED-ENTRY(APL-IDX) = IN-MSG-ID
                   SET ALREADY-APPLIED TO TRUE
           END-SEARCH

           IF ALREADY-APPLIED
               ADD 1 TO WS-SKIP-COUNT
               DISPLAY "SKIP (ALREADY APPLIED) MSG-ID="
                   IN-MSG-ID
           ELSE
               ADD IN-AMOUNT TO WS-RUNNING-TOTAL
               ADD 1 TO WS-APPLIED-COUNT
               MOVE IN-MSG-ID TO WS-APPLIED-ENTRY(WS-APPLIED-COUNT)
               MOVE IN-MSG-ID TO APPLIED-LINE
               WRITE APPLIED-LINE
               ADD 1 TO WS-NEW-COUNT
               DISPLAY "APPLIED MSG-ID=" IN-MSG-ID " AMOUNT="
                   IN-AMOUNT
           END-IF.

       SAVE-RUNNING-TOTAL.
           OPEN OUTPUT TOTAL-FILE
           MOVE WS-RUNNING-TOTAL TO TOTAL-LINE
           WRITE TOTAL-LINE
           CLOSE TOTAL-FILE.
```

### อธิบายจุดสำคัญ

- **`APPLIED.LOG`** คือหัวใจของ Idempotency — มันเก็บรายการ `MSG-ID` ทุกตัวที่เคยถูกประมวลผลสำเร็จแล้ว
  ก่อนที่จะประมวลผลข้อความใด ๆ Consumer ต้องโหลดไฟล์นี้เข้าตาราง (`WS-APPLIED-TABLE`, ทบทวนเทคนิค OCCURS
  จาก Part 016) ก่อนเสมอ
- **`SEARCH WS-APPLIED-ENTRY VARYING APL-IDX`** ใช้เทคนิค Sequential Search ที่เรียนมาใน Part 017 เพื่อ
  ตรวจสอบว่า `IN-MSG-ID` เคยอยู่ในตารางหรือไม่ — ถ้าเจอ (`ALREADY-APPLIED`) จะข้ามการประมวลผล ถ้าไม่เจอ
  จึงประมวลผลจริงและบันทึกลง `APPLIED.LOG` ทันที
- สังเกตว่า `WS-RUNNING-TOTAL` ถูกโหลดจาก `TOTAL.DAT` **ก่อน**เริ่มประมวลผล (ไม่ได้เริ่มจาก 0 ทุกครั้ง)
  ทำให้ยอดสะสมต่อเนื่องได้ข้ามการรันหลายครั้ง เหมือนกับ `ARLEDGER.cob` ใน Part 085

### ทดสอบจริง: End-to-End พิสูจน์ทั้งการทำงานปกติและ Idempotency

```bash
rm -f QUEUE.DAT PRODSEQ.DAT APPLIED.LOG TOTAL.DAT

echo "=== produce 3 messages ==="
echo "03" | ./producer

echo "=== consumer run 1 (should apply all 3) ==="
./consumer

echo "=== consumer run 2 - SIMULATED RETRY, same QUEUE.DAT (should skip all, no double count) ==="
./consumer

echo "=== produce 2 more messages ==="
echo "02" | ./producer

echo "=== consumer run 3 (should apply only the 2 new ones) ==="
./consumer
```

**ผลลัพธ์จริงที่ได้:**

```
=== produce 3 messages ===
PRODUCED MSG-ID=000001
PRODUCED MSG-ID=000002
PRODUCED MSG-ID=000003
=== consumer run 1 (should apply all 3) ===
APPLIED MSG-ID=000001 AMOUNT=0000500.00
APPLIED MSG-ID=000002 AMOUNT=0000600.00
APPLIED MSG-ID=000003 AMOUNT=0000700.00
NEW MESSAGES APPLIED: 0003
DUPLICATES SKIPPED:   0000
RUNNING TOTAL:        000001800.00
=== consumer run 2 - SIMULATED RETRY, same QUEUE.DAT (should skip all, no double count) ===
SKIP (ALREADY APPLIED) MSG-ID=000001
SKIP (ALREADY APPLIED) MSG-ID=000002
SKIP (ALREADY APPLIED) MSG-ID=000003
NEW MESSAGES APPLIED: 0000
DUPLICATES SKIPPED:   0003
RUNNING TOTAL:        000001800.00
=== produce 2 more messages ===
PRODUCED MSG-ID=000004
PRODUCED MSG-ID=000005
=== consumer run 3 (should apply only the 2 new ones) ===
SKIP (ALREADY APPLIED) MSG-ID=000001
SKIP (ALREADY APPLIED) MSG-ID=000002
SKIP (ALREADY APPLIED) MSG-ID=000003
APPLIED MSG-ID=000004 AMOUNT=0000500.00
APPLIED MSG-ID=000005 AMOUNT=0000600.00
NEW MESSAGES APPLIED: 0002
DUPLICATES SKIPPED:   0003
RUNNING TOTAL:        000002900.00
```

**ผลลัพธ์พิสูจน์สมมติฐานทุกข้อ**: Run 1 ประมวลผลข้อความใหม่ทั้ง 3 รายการ (รวม 500+600+700 = 1,800.00
ตรงกับ `RUNNING TOTAL`), Run 2 (จำลองการ Retry ด้วยคิวเดิมทุกประการ) **ข้ามทั้ง 3 รายการ ไม่มีการนับซ้ำ
เลย** (ยอดคงเดิม 1,800.00), และ Run 3 หลังผลิตข้อความใหม่เพิ่ม 2 รายการ ประมวลผล**เฉพาะรายการใหม่
เท่านั้น** (500+600=1,100.00 บวกเพิ่มจากยอดเดิม 1,800.00 = 2,900.00 ตรงกับผลลัพธ์)

### ข้อควรระวัง

- `WS-APPLIED-TABLE` มีขนาดจำกัดที่ `OCCURS 100 TIMES` — ในระบบจริงที่มีข้อความจำนวนมาก การโหลดประวัติ
  ทั้งหมดเข้าหน่วยความจำแบบนี้จะไม่ Scale ได้ดี (ทบทวนข้อจำกัดของ Fixed-size Table จาก Part 016) ระบบจริง
  มักใช้ Database หรือโครงสร้างข้อมูลที่ค้นหาได้เร็วกว่า Linear Search มาก (เช่น Hash-based Lookup หรือ
  Indexed File ตาม Part 028)
- การตรวจสอบ Idempotency ด้วย `MSG-ID` เพียงอย่างเดียวใช้ได้ก็ต่อเมื่อ `MSG-ID` **ไม่ซ้ำกันจริง** ตามที่
  ขั้นตอนที่ 864 ออกแบบไว้ — หากระบบภายนอกส่ง `MSG-ID` ที่ซ้ำกันโดยไม่ตั้งใจ (บั๊กที่ฝั่ง Producer) กลไก
  นี้จะป้องกันไม่ได้เลย

### แบบฝึกหัดที่ 865.1

**โจทย์**: จงอธิบายว่าทำไมการเช็ค `ALREADY-APPLIED` ต้องทำ**ก่อน**การ `ADD IN-AMOUNT TO WS-RUNNING-TOTAL`
เสมอ ไม่ใช่ทำสลับลำดับกัน

**เฉลยแนวทาง**: เพราะเป้าหมายของ Idempotency คือป้องกันไม่ให้ข้อความที่เคยประมวลผลไปแล้วถูกนำไปบวกเข้า
ยอดรวมซ้ำอีก หากทำการ `ADD` ก่อนแล้วค่อยตรวจสอบ (หรือลืมตรวจสอบก่อน) ข้อความที่ซ้ำจะถูกนำไปบวกเข้ายอดรวม
ไปแล้วก่อนที่ระบบจะรู้ตัวว่าซ้ำ ทำให้ยอดผิดพลาดไปแล้วไม่สามารถย้อนกลับได้ง่าย ๆ ลำดับที่ถูกต้องคือ
**ตรวจสอบก่อนเสมอ (Check-then-Act)**: เช็คว่าเคยประมวลผลหรือยัง ถ้ายังไม่เคยจึงทำการประมวลผลจริง แล้ว
บันทึกทันทีว่าประมวลผลแล้ว — ลำดับนี้คือรูปแบบมาตรฐานของการออกแบบระบบ Idempotent ทุกระบบ ไม่ว่าจะเขียน
ด้วยภาษาใดก็ตาม

---

## ขั้นตอนที่ 866: การรับมือกับความล้มเหลวกลางคัน — Retry แบบ Partial Batch

### แนวคิด: Retry ไม่ได้เกิดแค่ "รันทั้งคิวใหม่ทั้งหมด" เสมอไป

ขั้นตอนที่ 865 พิสูจน์ว่า Consumer ปลอดภัยเมื่อ**รันคิวเดิมซ้ำทั้งหมด** — แต่สถานการณ์ที่พบบ่อยกว่าใน
โลกจริงคือ Consumer **ล้มเหลวกลางคัน** (เช่น เครื่องดับระหว่างประมวลผลข้อความที่ 2 จาก 3 ข้อความ) แล้ว
ถูก Retry ใหม่ — คำถามสำคัญคือ: กลไก Idempotency แบบเดียวกันนี้ยังคุ้มครองกรณีนี้ได้หรือไม่?

### ทดสอบจริง: จำลองความล้มเหลวกลางคันด้วยเวอร์ชันทดสอบของ Consumer

เพื่อพิสูจน์สถานการณ์นี้ เราเพิ่มกลไกทดสอบชั่วคราว (`WS-STOP-AFTER`) เข้าไปใน Consumer เพื่อจำลองว่า
โปรแกรม "หยุดทำงานกะทันหัน" หลังประมวลผลข้อความใหม่ไปแล้ว N รายการ (เทคนิคเดียวกับที่ Part 086 ขั้นตอนที่
853-854 ใช้จำลอง Batch Failure):

```cobol
      *> (added inside PROCESS-QUEUE of CONSUMER.cob for testing
      *> only - not production code)
           PERFORM UNTIL WS-EOF = "Y"
               READ QUEUE-FILE INTO WS-IN-REC
                   AT END MOVE "Y" TO WS-EOF
                   NOT AT END
                       PERFORM HANDLE-ONE-MESSAGE
                       IF WS-STOP-AFTER > 0
                          AND WS-NEW-COUNT = WS-STOP-AFTER
                           DISPLAY "SIMULATED CRASH AFTER "
                               WS-NEW-COUNT " NEW MESSAGE(S)"
                           CLOSE QUEUE-FILE
                           CLOSE APPLIED-FILE
                           PERFORM SAVE-RUNNING-TOTAL
                           STOP RUN
                       END-IF
               END-READ
           END-PERFORM
```

### รันจริง: ประมวลผล 3 ข้อความ แต่ "ล้มเหลว" หลังจากข้อความที่ 2

```bash
rm -f QUEUE.DAT PRODSEQ.DAT APPLIED.LOG TOTAL.DAT

echo "=== produce 3 messages ==="
echo "03" | ./producer

echo "=== consumer run 1: crash after 2 applied ==="
echo "02" | ./consumer_crash

echo "=== consumer run 2 (retry): should apply only msg 3, no duplicates ==="
echo "00" | ./consumer_crash
```

**ผลลัพธ์จริงที่ได้:**

```
=== produce 3 messages ===
PRODUCED MSG-ID=000001
PRODUCED MSG-ID=000002
PRODUCED MSG-ID=000003
=== consumer run 1: crash after 2 applied ===
APPLIED MSG-ID=000001 AMOUNT=0000500.00
APPLIED MSG-ID=000002 AMOUNT=0000600.00
SIMULATED CRASH AFTER 0002 NEW MESSAGE(S)
=== consumer run 2 (retry): should apply only msg 3, no duplicates ===
SKIP (ALREADY APPLIED) MSG-ID=000001
SKIP (ALREADY APPLIED) MSG-ID=000002
APPLIED MSG-ID=000003 AMOUNT=0000700.00
NEW MESSAGES APPLIED: 0001
DUPLICATES SKIPPED:   0002
RUNNING TOTAL:        000001800.00
```

**กลไก Idempotency แบบเดียวกันคุ้มครองได้ทั้งสองสถานการณ์**: ทั้งกรณี "Retry ทั้งคิว" (ขั้นตอนที่ 865)
และกรณี "ล้มเหลวกลางคันแล้ว Retry" (ขั้นตอนนี้) — เพราะ `APPLIED.LOG` ถูกบันทึก**ทันทีหลังประมวลผลแต่ละ
ข้อความสำเร็จ** (ไม่ใช่บันทึกทีเดียวตอนจบทั้ง Batch) ทำให้ไม่ว่าจะล้มเหลว ณ จุดใดก็ตาม การ Retry จะรู้
เสมอว่าข้อความใดทำสำเร็จไปแล้วบ้าง — ยอดรวมสุดท้าย (1,800.00 = 500+600+700) ถูกต้องตรงตามที่คาดหวังทุก
ประการ ไม่มีการนับซ้ำหรือตกหล่นแม้แต่รายการเดียว

### หลักการสำคัญที่พิสูจน์ได้จากการทดลองนี้: บันทึกสถานะทันทีหลังทำสำเร็จ ไม่ใช่ทีเดียวตอนจบ

สังเกตความแตกต่างสำคัญจาก `SAVE-RUNNING-TOTAL` (ที่บันทึกทีเดียวตอนจบ Batch) เทียบกับ `WRITE
APPLIED-LINE` (ที่บันทึกทันทีทุกครั้งที่ประมวลผลสำเร็จหนึ่งข้อความ ภายใน `HANDLE-ONE-MESSAGE`) — **การ
บันทึกสถานะการ "ทำสำเร็จแล้ว" ทันทีต่อหน่วยงานย่อยที่เล็กที่สุด (ต่อข้อความ ไม่ใช่ต่อทั้ง Batch)** คือ
กุญแจสำคัญที่ทำให้ระบบทนทานต่อความล้มเหลวกลางคันได้จริง

### ข้อควรระวัง

- `WS-STOP-AFTER` ในตัวอย่างนี้เป็น**เครื่องมือทดสอบ**ที่เพิ่มเข้ามาเพื่อจำลองสถานการณ์เท่านั้น ไม่ควรมี
  อยู่ในโค้ด Production จริง (เทียบเท่ากับ `WS-FORCE-FAIL` ใน `JOBCTL.cob` และ `WS-STOP-AFTER` ใน
  `BATCHRUN.cob` จาก Part 086)
- แม้กลไกนี้จะทนทานต่อความล้มเหลว "ระหว่างข้อความ" ได้ดี แต่ยังมีจุดเสี่ยงเล็กน้อยที่ทฤษฎีล้วน ๆ: หาก
  โปรแกรมล้มเหลว**ระหว่าง**การ `ADD`/`WRITE` ของข้อความเดียวกัน (เช่น หลัง `ADD IN-AMOUNT TO
  WS-RUNNING-TOTAL` แต่ก่อน `WRITE APPLIED-LINE` เสร็จสมบูรณ์) อาจเกิดสถานะที่ไม่สอดคล้องกันได้ในทาง
  ทฤษฎี — ระบบ Production จริงมักใช้กลไก Transaction ของฐานข้อมูล (ทบทวน DB2 จาก Part 057-060) เพื่อ
  รับประกันว่าการเปลี่ยนแปลงหลายจุดเกิดขึ้น "พร้อมกันทั้งหมดหรือไม่เกิดขึ้นเลย" (All-or-Nothing) ซึ่งไฟล์
  ธรรมดาไม่มีกลไกนี้ในตัว

### แบบฝึกหัดที่ 866.1

**โจทย์**: จงอธิบายว่าทำไมการทดสอบในขั้นตอนนี้ (crash หลังข้อความที่ 2 จาก 3) ถึงเป็นกรณีทดสอบที่สำคัญ
กว่าการทดสอบ "crash หลังข้อความสุดท้าย (ข้อความที่ 3)"

**เฉลยแนวทาง**: เพราะการ crash หลังข้อความสุดท้ายเป็นกรณีที่ไม่มีอะไรน่ากังวล (ทุกข้อความประมวลผลสำเร็จ
หมดแล้วก่อน crash) แต่การ crash หลังข้อความที่ 2 จาก 3 ทดสอบสถานการณ์ที่**ยังมีงานค้างอยู่**ในคิว —
เป็นกรณีที่พิสูจน์ว่าระบบ (1) ไม่ประมวลผลข้อความที่เสร็จไปแล้วซ้ำ (ข้อความที่ 1-2) และ (2) ยังคงประมวลผล
ข้อความที่ยังไม่เสร็จต่อไปได้อย่างถูกต้อง (ข้อความที่ 3) พร้อมกันในการทดสอบเดียว ซึ่งเป็นสถานการณ์ที่
สมจริงกว่ามากและครอบคลุมทั้งสองแง่มุมของ Idempotency (ไม่ทำซ้ำ + ไม่ตกหล่น) ในการทดสอบเดียวกัน

---

## ขั้นตอนที่ 867: แนวคิด Retry ขั้นสูง — Backoff และขีดจำกัดการลองใหม่

### ปัญหา: การ Retry ทันทีซ้ำ ๆ อาจทำให้สถานการณ์แย่ลง

ขั้นตอนที่ 865-866 พิสูจน์ว่าการ Retry ปลอดภัยด้าน**ความถูกต้องของข้อมูล** — แต่ยังมีคำถามเชิงปฏิบัติการ
ที่สำคัญอีกข้อ: **ควร Retry บ่อยแค่ไหน และควรหยุด Retry เมื่อไหร่?** หากระบบที่ Consumer ต้องพึ่งพา (เช่น
ฐานข้อมูลปลายทาง) กำลังมีปัญหาชั่วคราว การ Retry ทันทีซ้ำ ๆ แบบไม่มีการหน่วงเวลาอาจยิ่งซ้ำเติมปัญหา (เพิ่ม
ภาระให้ระบบที่กำลังมีปัญหาอยู่แล้ว)

### แนวคิด Exponential Backoff

**Exponential Backoff** คือกลยุทธ์ที่เพิ่มระยะเวลารอคอยระหว่างการ Retry แต่ละครั้งแบบทวีคูณ แทนที่จะ
Retry ทันทีทุกครั้ง:

```
Retry ครั้งที่ 1: รอ 1 วินาที แล้วลองใหม่
Retry ครั้งที่ 2: รอ 2 วินาที แล้วลองใหม่ (ถ้าครั้งที่ 1 ยังล้มเหลว)
Retry ครั้งที่ 3: รอ 4 วินาที แล้วลองใหม่
Retry ครั้งที่ 4: รอ 8 วินาที แล้วลองใหม่
...
Retry ครั้งที่ N: รอ (2^(N-1)) วินาที หรือจนถึงเพดานสูงสุดที่กำหนดไว้
```

แนวคิดนี้มักถูกนำไปใช้ในระดับ **Job Scheduler** ที่ควบคุมว่าจะรัน Consumer ใหม่เมื่อไหร่หลัง Job ก่อนหน้า
ล้มเหลว (ทบทวน Batch Scheduling จาก Part 086 ขั้นตอนที่ 853) มากกว่าจะเขียนเป็น Loop รอภายในโปรแกรม
COBOL เอง — ในทางปฏิบัติ Scheduler ภายนอก (เช่น Cron ที่มีการตั้งค่าหน่วงเวลาเพิ่มขึ้น, หรือ Message
Broker ที่มีกลไก Retry ในตัว) มักรับผิดชอบเรื่องนี้แทน

### แนวคิดขีดจำกัดการ Retry (Retry Limit) และ Dead Letter Queue

หากข้อความหนึ่งล้มเหลวซ้ำแล้วซ้ำเล่าไม่ว่าจะ Retry กี่ครั้ง (เช่น ข้อมูลในข้อความนั้นผิดรูปแบบจนไม่
สามารถประมวลผลได้เลย ไม่ใช่ปัญหาชั่วคราวของระบบปลายทาง) การ Retry ต่อไปเรื่อย ๆ ไม่มีที่สิ้นสุดจะไม่มี
ประโยชน์และอาจกีดขวางการประมวลผลข้อความอื่นที่ตามมาในคิว (Head-of-Line Blocking) — แนวทางแก้ไขมาตรฐาน
คือ:

1. กำหนด **Retry Limit** (เช่น "ลองใหม่ได้สูงสุด 5 ครั้ง")
2. เมื่อเกินขีดจำกัด ให้ย้ายข้อความนั้นไปยัง **Dead Letter Queue (DLQ)** — คิวพิเศษสำหรับข้อความที่
   ประมวลผลไม่สำเร็จ เพื่อให้มนุษย์เข้ามาตรวจสอบภายหลัง โดยไม่บล็อกการประมวลผลข้อความปกติอื่น ๆ ในคิวหลัก

```
Message queue:  [MSG-1] [MSG-2 - FAILS REPEATEDLY] [MSG-3] [MSG-4]
                            |
                 (retry 5 times, still fails)
                            |
                            v
                    [Dead Letter Queue]
                       [MSG-2]
                            |
                  (มนุษย์ตรวจสอบภายหลัง)

Meanwhile: MSG-3, MSG-4 ถูกประมวลผลต่อไปตามปกติ ไม่ถูกบล็อกโดย MSG-2
```

### แนวคิดการนำ Dead Letter มาใช้กับระบบ COBOL ของเรา (เชิงแนวคิด)

ในระบบ `CONSUMER.cob` ของเรา หากต้องเพิ่มแนวคิด Dead Letter จะทำได้โดยเพิ่มการตรวจสอบความถูกต้องของ
ข้อความก่อนประมวลผล (คล้ายกับ `VALIDATE-ORDER` จาก Part 083) หากข้อความไม่ถูกต้อง (เช่น `IN-AMOUNT`
เป็นศูนย์ หรือรูปแบบผิดเพี้ยน) ให้เขียนลงไฟล์ `DEADLETTER.LOG` แยกต่างหากแทนที่จะพยายามประมวลผลซ้ำไม่รู้
จบ:

```cobol
      *> illustrative concept only - not part of the real CONSUMER.cob
           IF IN-AMOUNT = ZERO
               DISPLAY "MOVING TO DEAD LETTER: MSG-ID=" IN-MSG-ID
               PERFORM WRITE-TO-DEAD-LETTER-LOG
           ELSE
               PERFORM HANDLE-ONE-MESSAGE
           END-IF
```

### ข้อควรระวัง

- Exponential Backoff และ Dead Letter Queue เป็นแนวคิดที่ Message Broker จริง (Kafka, RabbitMQ, Amazon
  SQS) มีกลไกในตัวรองรับอยู่แล้ว — เมื่อองค์กรใช้ Message Broker จริง มักไม่จำเป็นต้องเขียนกลไกเหล่านี้
  เองในโปรแกรม COBOL แต่ควรตั้งค่าผ่าน Configuration ของ Broker แทน
- ต้องระวังไม่ให้ "ขีดจำกัดการ Retry" เข้มงวดเกินไปสำหรับปัญหาชั่วคราวที่แท้จริง (เช่น เครือข่ายสะดุด
  ชั่วขณะ) เพราะจะทำให้ข้อความที่ควรจะสำเร็จได้ถูกส่งเข้า Dead Letter Queue อย่างไม่จำเป็น ต้องสมดุล
  ระหว่างการให้โอกาส Retry เพียงพอ กับการไม่ปล่อยให้ข้อความที่มีปัญหาจริงบล็อกระบบไม่รู้จบ

### แบบฝึกหัดที่ 867.1

**โจทย์**: จงอธิบายว่าทำไมปัญหา "ข้อมูลผิดรูปแบบ" (เช่น `IN-AMOUNT` เป็นค่าที่แปลงเป็นตัวเลขไม่ได้) ควร
ถูกส่งเข้า Dead Letter Queue **ทันที** โดยไม่ต้อง Retry เลย ในขณะที่ปัญหา "ระบบปลายทางไม่ตอบสนองชั่วคราว"
ควร Retry ก่อนจะส่งเข้า Dead Letter Queue

**เฉลยแนวทาง**: เพราะทั้งสองปัญหามีธรรมชาติต่างกันโดยพื้นฐาน — ปัญหาข้อมูลผิดรูปแบบเป็น**ปัญหาถาวร**
(Permanent Error) ที่จะไม่มีวันหายไปเองไม่ว่าจะ Retry กี่ครั้งก็ตาม (ข้อมูลเดิมจะยังผิดรูปแบบเหมือนเดิม
ทุกครั้งที่ลองใหม่) การ Retry ในกรณีนี้จึงเป็นการเสียเวลาและทรัพยากรโดยเปล่าประโยชน์ ควรส่งเข้า Dead
Letter Queue ทันทีเพื่อให้มนุษย์ตรวจสอบต้นตอของข้อมูลที่ผิดพลาด ในขณะที่ปัญหาระบบปลายทางไม่ตอบสนอง
มักเป็น**ปัญหาชั่วคราว** (Transient Error เช่น เครือข่ายสะดุด, ระบบกำลัง Restart) ที่มีโอกาสสูงที่จะ
หายไปเองถ้ารอสักครู่แล้วลองใหม่ (ตามหลัก Exponential Backoff) การแยกแยะสองประเภทนี้ให้ถูกต้องจึงสำคัญ
มากในการออกแบบกลยุทธ์ Retry ที่มีประสิทธิภาพ

---

## ขั้นตอนที่ 868: เปรียบเทียบตัวอย่างในหลักสูตรกับ Message Broker จริงอย่างตรงไปตรงมา

### ตารางเปรียบเทียบโดยละเอียด

| คุณสมบัติ | `QUEUE.DAT` ในหลักสูตรนี้ | Kafka/RabbitMQ จริง |
|---|---|---|
| การเขียนข้อความ | `OPEN EXTEND` + `WRITE` ต่อท้ายไฟล์ | Producer API ส่งข้อความผ่านเครือข่ายไปยัง Broker |
| การอ่านข้อความ | อ่านทั้งไฟล์ทุกครั้งตั้งแต่บรรทัดแรก | Consumer ติดตาม Offset เฉพาะของตัวเอง อ่านต่อจากจุดเดิมได้โดยไม่ต้องอ่านซ้ำทั้งหมด |
| หลาย Consumer พร้อมกัน | ไม่รองรับ (จะอ่านข้อความเดียวกันซ้ำกันทุกตัว) | รองรับผ่าน Consumer Group ที่แบ่งงานกันอัตโนมัติ |
| ความทนทาน (Durability) | ขึ้นกับระบบไฟล์เดียว ไม่มี Replication | Replication ข้ามหลาย Broker/Server โดยอัตโนมัติ |
| ลำดับข้อความ (Ordering) | ตามลำดับการเขียนไฟล์ (ใช้ได้ในระบบเดียว) | รับประกันลำดับภายใน Partition เดียวกันเท่านั้น (ซับซ้อนกว่าที่คิด) |
| การจัดการ Consumer ล่ม | ต้องเขียนกลไก Checkpoint/Idempotency เอง (ตามที่สอนไปแล้ว) | มีกลไก Acknowledgment/Offset Commit ในตัว |
| ประสิทธิภาพ (Throughput) | จำกัดด้วยความเร็วการอ่าน/เขียนไฟล์แบบลำดับ | ออกแบบมาเพื่อรองรับข้อความหลายล้านต่อวินาที |
| ค่าใช้จ่ายในการติดตั้ง/ดูแล | ไม่มีเลย (ใช้ไฟล์ระบบธรรมดา) | ต้องติดตั้ง ดูแล และปรับแต่ง Cluster เฉพาะทาง |

### สิ่งที่ตัวอย่างในหลักสูตรนี้สอนได้ถูกต้อง (แม้ไม่มีโครงสร้างพื้นฐานจริง)

1. **แนวคิด Producer/Consumer แยกจากกันอย่างสิ้นเชิง** — ถูกต้องตรงกับหลักการจริงทุกประการ
2. **At-Least-Once Delivery และความจำเป็นของ Idempotency** — ถูกต้องตรงกับหลักการจริงทุกประการ (นี่คือ
   ปัญหาที่มีอยู่จริงแม้ใน Kafka/RabbitMQ ระดับ Production)
3. **Checkpoint/Retry สำหรับความล้มเหลวกลางคัน** — หลักการเดียวกับที่ Consumer Group ใน Kafka ใช้ Offset
   Commit เพื่อจดจำว่าอ่านไปถึงไหนแล้ว

### สิ่งที่ตัวอย่างในหลักสูตรนี้ "ไม่ได้" สอน (ต้องเรียนรู้เพิ่มเติมจากประสบการณ์จริงกับ Message Broker)

1. การจัดการ **Consumer Group** และการแบ่งงานระหว่างหลาย Consumer พร้อมกัน
2. การจัดการ **Partition** เพื่อกระจายภาระข้อความจำนวนมากในระบบขนาดใหญ่จริง
3. กลไก **Schema Registry** สำหรับควบคุมรูปแบบข้อความในระบบที่มีผู้ผลิต/ผู้บริโภคจำนวนมาก (แนวคิดใกล้เคียง
   กับ Interface Versioning ที่ Part 086 ขั้นตอนที่ 857 สอนไว้ แต่ในระดับที่ซับซ้อนกว่ามาก)
4. การตั้งค่า **Exactly-Once Semantics** ในกรณีพิเศษที่ต้องการความแม่นยำสูงสุด (ซับซ้อนกว่า
   At-Least-Once + Idempotency มาก และมีข้อจำกัดทางทฤษฎีที่สำคัญ)

### ข้อควรระวัง

- เมื่อต้องทำงานกับ Message Broker จริงในอนาคต **อย่าคาดหวังว่าโค้ด COBOL จะเชื่อมต่อกับมันโดยตรง** —
  ในทางปฏิบัติ มักต้องมี Wrapper ภาษาอื่น (เช่น Python หรือ Java ที่มี Client Library ของ Kafka/RabbitMQ
  พร้อมใช้) ทำหน้าที่เป็นสะพานเชื่อม คล้ายกับที่ `ar_server.py` ใน Part 085 ทำหน้าที่เชื่อม HTTP กับ COBOL
- แนวคิด Idempotency ที่เรียนในหลักสูตรนี้เป็นพื้นฐานสำคัญที่สุด แต่การนำไปใช้กับ Message Broker จริงอาจ
  ต้องพิจารณารายละเอียดเพิ่มเติม เช่น การใช้ Database Transaction ร่วมกับการ Commit Offset (Pattern ที่
  เรียกว่า "Transactional Outbox" ในบางระบบ) ซึ่งอยู่นอกขอบเขตของหลักสูตรระดับนี้

### แบบฝึกหัดที่ 868.1

**โจทย์**: หากองค์กรของคุณตัดสินใจเปลี่ยนจากไฟล์ `QUEUE.DAT` ไปใช้ Kafka จริง โค้ดส่วนใดใน `CONSUMER.cob`
ที่ต้องเปลี่ยนแปลงมากที่สุด และส่วนใดที่แนวคิดยังคงใช้ได้โดยไม่ต้องเปลี่ยนแปลงมาก

**เฉลยแนวทาง**: ส่วนที่ต้องเปลี่ยนแปลงมากที่สุดคือ**กลไกการอ่านข้อความ** (`OPEN INPUT QUEUE-FILE` /
`READ QUEUE-FILE`) ซึ่งต้องถูกแทนที่ด้วยการเรียก Kafka Consumer Client (ผ่าน Wrapper ภาษาอื่น เนื่องจาก
COBOL ไม่มี Native Kafka Client) ที่ดึงข้อความมาทีละชุดผ่านเครือข่ายแทนการอ่านไฟล์โดยตรง ในทางกลับกัน
**แนวคิดเรื่อง Idempotency** (การตรวจสอบ `APPLIED-LOG`/`ALREADY-APPLIED` ก่อนประมวลผลทุกครั้ง) ยังคง
ใช้ได้และ**จำเป็นต้องมีอยู่เหมือนเดิมทุกประการ** เพราะ Kafka เองก็เป็นระบบแบบ At-Least-Once Delivery
เช่นกัน (ยกเว้นจะตั้งค่า Exactly-Once Semantics แบบพิเศษ ซึ่งมีความซับซ้อนและข้อจำกัดของตัวเอง) — นี่คือ
เหตุผลที่ Part นี้เน้นย้ำเรื่อง Idempotency มากเป็นพิเศษ เพราะมันเป็นแนวคิดที่**คงทนต่อการเปลี่ยนแปลง
เทคโนโลยีเบื้องหลัง** ต่างจากรายละเอียดการอ่าน/เขียนข้อความที่ผูกติดกับเทคโนโลยีเฉพาะมากกว่า

---

## ขั้นตอนที่ 869: การออกแบบ COBOL Service ให้พร้อมสำหรับ Distributed Architecture

### Checklist คุณสมบัติที่ COBOL Service ควรมีก่อนเข้าร่วมระบบกระจาย

จากทุกแนวคิดที่เรียนมาใน Part นี้ สรุปเป็น Checklist คุณสมบัติที่ COBOL Service หนึ่งตัวควรมีก่อนที่จะ
ถูกดึงเข้าไปเป็นส่วนหนึ่งของสถาปัตยกรรมแบบกระจาย:

| ลำดับ | คุณสมบัติ | อ้างอิง |
|---|---|---|
| 1 | มี Bounded Context ที่ชัดเจน ไม่ปะปนความรับผิดชอบกับบริการอื่น | ขั้นตอนที่ 861 |
| 2 | เข้าถึงได้ผ่านช่องทางมาตรฐาน (REST API) ไม่ใช่การเรียกโปรแกรมตรง ๆ ข้ามระบบ | ขั้นตอนที่ 862, Part 073 |
| 3 | มี Health Check Endpoint แยกจาก Business Logic หลัก | ขั้นตอนที่ 862 |
| 4 | หากเข้าร่วม Event-driven Architecture ต้องเข้าใจว่าเป็น At-Least-Once Delivery เสมอ | ขั้นตอนที่ 863 |
| 5 | ทุกการประมวลผลที่มีผลข้างเคียง (side effect) ต้องออกแบบให้ Idempotent | ขั้นตอนที่ 865-866 |
| 6 | มีกลยุทธ์ Retry ที่เหมาะสม (Backoff + Retry Limit + Dead Letter) สำหรับข้อความที่ล้มเหลว | ขั้นตอนที่ 867 |
| 7 | เข้าใจข้อจำกัดของโครงสร้างพื้นฐานที่ใช้จริง เทียบกับสิ่งที่ระบบ Production ต้องการ | ขั้นตอนที่ 868 |

### กรณีศึกษาสรุป: AR-MINI จาก Part 085 พร้อมสำหรับ Distributed Architecture แค่ไหน

ลองประเมิน AR-MINI จาก Part 085 ด้วย Checklist นี้:

| ข้อ | สถานะของ AR-MINI | หมายเหตุ |
|---|---|---|
| 1 | ผ่าน | คำนวณใบแจ้งหนี้ + บันทึกยอดคงเหลือ เป็น Bounded Context ที่ชัดเจน |
| 2 | ผ่าน | มี REST API ผ่าน `ar_server.py` แล้ว |
| 3 | ยังไม่ผ่าน | ยังไม่มี Health Check Endpoint แยกต่างหาก (ตามที่ระบุในแบบฝึกหัดที่ 862.1) |
| 4 | ยังไม่เกี่ยวข้อง | AR-MINI เป็น Synchronous ล้วน ยังไม่ได้เข้าร่วม Event-driven Architecture |
| 5 | ผ่านบางส่วน | `ARLEDGER.cob` บวกยอดสะสมทุกครั้งที่เรียก ยังไม่มีการตรวจสอบ Idempotency แบบ `MSG-ID` เหมือน `CONSUMER.cob` ในเอกสารนี้ (เพราะ REST call แต่ละครั้งถือเป็นคำขอใหม่ที่ตั้งใจให้ประมวลผลจริง ไม่ใช่ Message ที่อาจถูกส่งซ้ำ) |
| 6 | ไม่เกี่ยวข้องในขอบเขตปัจจุบัน | ยังไม่ได้เชื่อมต่อกับ Message Queue ใด ๆ |
| 7 | ผ่าน | เอกสาร Part 085 ระบุข้อจำกัดของระบบไว้อย่างตรงไปตรงมาแล้ว |

สังเกตว่าข้อ 5 มีความแตกต่างที่น่าสนใจ: AR-MINI ออกแบบมาให้แต่ละ Request คือคำขอใหม่ที่ตั้งใจให้ผลลัพธ์
ต่างกัน (REST call แบบ Synchronous) ในขณะที่ `CONSUMER.cob` ใน Part นี้ออกแบบมาสำหรับ Event ที่อาจถูก
ส่งซ้ำโดยไม่ตั้งใจ (Asynchronous Messaging) — นี่คือตัวอย่างที่ดีว่า **Idempotency ไม่ใช่กฎที่ต้องใช้กับ
ทุกการทำงานเสมอไป แต่ขึ้นอยู่กับรูปแบบการสื่อสาร (Synchronous vs Asynchronous) ที่ระบบนั้นใช้**

### ข้อควรระวัง

- Checklist นี้เป็นจุดเริ่มต้นที่ดี แต่ไม่ใช่รายการที่สมบูรณ์แบบสำหรับทุกองค์กร — แต่ละองค์กรอาจมี
  ข้อกำหนดเพิ่มเติมตามบริบทของตัวเอง (เช่น มาตรฐานความปลอดภัยเฉพาะทางสำหรับอุตสาหกรรมการเงิน)
- อย่าพยายามทำให้ทุกอย่าง Idempotent โดยไม่จำเป็น — การออกแบบระบบให้ Idempotent มีต้นทุนเพิ่มเติมเสมอ
  (พื้นที่จัดเก็บสำหรับบันทึกประวัติ, ความซับซ้อนของโค้ดที่เพิ่มขึ้น) ควรใช้กับจุดที่มีความเสี่ยงจากการ
  ประมวลผลซ้ำจริง ๆ (โดยเฉพาะจุดที่เกี่ยวข้องกับเงินหรือ Asynchronous Messaging) ไม่ใช่ใช้พร่ำเพรื่อกับ
  ทุกฟังก์ชันในระบบ

### แบบฝึกหัดที่ 869.1

**โจทย์**: จงอธิบายว่าทำไม AR-MINI (ข้อ 5 ในตาราง) จึง "ไม่จำเป็น" ต้องมี Idempotency แบบ `MSG-ID`
เหมือน `CONSUMER.cob` ทั้งที่ทั้งสองระบบต่างก็เกี่ยวข้องกับการบวกยอดเงินสะสมเหมือนกัน

**เฉลยแนวทาง**: เพราะรูปแบบการสื่อสารต่างกันโดยพื้นฐาน — AR-MINI ใช้ REST API แบบ Synchronous ที่ Client
เรียกและรอผลลัพธ์ทันที หาก Client ต้องการคำนวณใบแจ้งหนี้ใหม่จริง ๆ สองครั้ง (เช่น ลูกค้าสั่งซื้อสองครั้ง
แยกกัน) นั่นคือสองธุรกรรมที่ตั้งใจให้เกิดขึ้นจริงสองครั้ง ไม่ใช่การส่งคำขอเดียวกันซ้ำโดยไม่ตั้งใจ ในขณะ
ที่ `CONSUMER.cob` รับข้อความจากคิวที่มีธรรมชาติเป็น **At-Least-Once Delivery** ซึ่งหมายความว่าข้อความ
เดียวกัน (เช่น "ลูกค้า A ชำระเงิน 500 บาท ครั้งที่ `MSG-ID=5`") อาจถูกส่งมาถึงผู้บริโภคมากกว่าหนึ่งครั้ง
โดยไม่ได้ตั้งใจ (เพราะกลไกการส่งข้อความของระบบคิว) การตรวจสอบ `MSG-ID` จึงจำเป็นเพื่อแยกแยะระหว่าง
"ข้อความเดิมที่ถูกส่งซ้ำ" กับ "ธุรกรรมใหม่จริง ๆ" — หาก AR-MINI ในอนาคตถูกออกแบบให้รับ Event จากคิวแทน
การเรียก REST API ตรง ๆ ก็จะต้องเพิ่มกลไก Idempotency แบบเดียวกันนี้เข้าไปทันที

---

## ขั้นตอนที่ 870: สรุป Checklist และก้าวต่อไปสู่การปรับจูนประสิทธิภาพ

### สรุปภาพรวมเส้นทางการเรียนรู้ของ Part 086-087

```
Part 086: MANY SYSTEMS ในองค์กรเดียวกัน
   - Layered Architecture, Batch Scheduling
   - Copybook Governance, Interface Versioning
              |
              v
Part 087: SYSTEMS ที่กระจายตัวและสื่อสารแบบ Asynchronous
   - COBOL เป็น Bounded Context Service หลัง API Gateway
   - Event-driven Architecture (ด้วยข้อจำกัดที่ระบุไว้ตรงไปตรงมา)
   - Idempotency: หัวใจสำคัญที่สุดของการเชื่อมต่อ Batch <-> Microservices
   - Retry, Backoff, Dead Letter Queue
```

### บทเรียนที่สำคัญที่สุดของ Part นี้

หากต้องสรุป Part 087 เหลือเพียงประโยคเดียว คือ: **"เมื่อการสื่อสารระหว่างระบบไม่รับประกันว่าจะเกิดขึ้น
พอดีหนึ่งครั้งเสมอ (ซึ่งเป็นเรื่องปกติของระบบกระจายที่ต้องทนทานต่อความล้มเหลว) ผู้รับต้องออกแบบให้
ปลอดภัยต่อการรับซ้ำเสมอ (Idempotent)"** — หลักการนี้พิสูจน์แล้วด้วยการทดสอบจริงตลอด Part นี้ ทั้งกรณี
Retry ทั้งคิว (ขั้นตอนที่ 865) และกรณี Retry หลังล้มเหลวกลางคัน (ขั้นตอนที่ 866) และเป็นหลักการที่**คงทน
ต่อการเปลี่ยนแปลงเทคโนโลยีเบื้องหลัง** ไม่ว่าจะใช้ไฟล์ธรรมดาหรือ Kafka จริงก็ตาม (ตามที่วิเคราะห์ไว้ใน
ขั้นตอนที่ 868)

### ก้าวต่อไป: Part 088

ตลอด Part 085-087 เราเน้นเรื่อง**ความถูกต้อง**ของระบบ (Correctness) เป็นหลัก — Golden Master Testing,
Unit Testing, Idempotency — แต่ยังมีมิติสำคัญอีกด้านที่ยังไม่ได้กล่าวถึงอย่างละเอียด: **ประสิทธิภาพ**
(Performance) เมื่อระบบต้องรองรับปริมาณงานที่สูงขึ้นมาก ๆ **Part 088: Performance Optimization ระดับสูง**
จะกลับไปทบทวนเทคนิคการปรับจูนประสิทธิภาพ COBOL ในเชิงลึก ต่อยอดจากพื้นฐานที่ Part 068 (Performance
Tuning สำหรับ Mainframe) เคยแนะนำไว้ในเฟส 4

### ข้อควรระวัง

- อย่าลืมว่า Idempotency มีต้นทุน (เช่น พื้นที่จัดเก็บสำหรับ `APPLIED.LOG` ที่โตขึ้นเรื่อย ๆ ตามจำนวน
  ข้อความที่เคยประมวลผล) — ระบบจริงต้องมีนโยบายการล้างประวัติเก่าที่ไม่จำเป็นแล้วออกเป็นระยะ (เช่น ข้อความ
  ที่เก่ากว่า 90 วันและยืนยันแล้วว่าจะไม่ถูกส่งซ้ำอีก)
- เนื้อหาทั้ง Part 085-087 วางรากฐานสถาปัตยกรรมที่ดีไว้แล้ว — Part 088 เป็นต้นไปจะเริ่มเจาะลึกในหัวข้อ
  เฉพาะทางมากขึ้น (Performance, Security, Project Management) ซึ่งล้วนต้องอาศัยรากฐานสถาปัตยกรรมที่มั่นคง
  จาก 3 Part ที่ผ่านมาเป็นพื้นฐาน

### แบบฝึกหัดที่ 870.1

**โจทย์**: จงสรุปเป็นคำพูดของคุณเอง 3-4 ประโยค อธิบายว่าทำไม "การรู้ว่าระบบของเราใช้การสื่อสารแบบ
Synchronous หรือ Asynchronous" ถึงเป็นคำถามแรกที่ควรถามก่อนตัดสินใจว่าจะต้องออกแบบ Idempotency หรือไม่

**เฉลยแนวทาง**: คำตอบขึ้นกับผู้เรียนแต่ละคน แต่ควรมีใจความสำคัญคือ: การสื่อสารแบบ Synchronous (เช่น
REST API ที่ Client รอผลลัพธ์ทันที) มักมีธรรมชาติที่ผู้เรียกรู้ตัวชัดเจนว่ากำลังส่งคำขออะไรและเมื่อไหร่
ความเสี่ยงจากการส่งซ้ำโดยไม่ตั้งใจมักเกิดจากพฤติกรรมของผู้เรียกเอง (เช่นกดปุ่มซ้ำ) ซึ่งมีวิธีป้องกันที่
ต่างออกไป (เช่น การปิดปุ่มหลังกดครั้งแรก) ในขณะที่การสื่อสารแบบ Asynchronous ผ่าน Message Queue มี
ธรรมชาติของระบบเองที่อาจส่งข้อความซ้ำได้ (At-Least-Once Delivery) โดยที่ทั้งผู้ส่งและผู้รับไม่ได้ทำอะไร
ผิดพลาดเลย เป็นเพียงกลไกการรับประกันการส่งข้อความของระบบคิวเอง ดังนั้นหากระบบใช้ Asynchronous Messaging
ผู้พัฒนาจึงควร**ถือว่า Idempotency เป็นข้อกำหนดพื้นฐานที่ต้องมีเสมอ** ไม่ใช่ทางเลือกเสริม

---

## สรุปท้ายบท

Part 087 พาเราสำรวจว่า COBOL เข้าร่วมสถาปัตยกรรมแบบกระจาย (Distributed Architecture) และ Microservices
ได้อย่างไร โดยไม่ต้องเขียนใหม่ทั้งหมดหรือทิ้งจุดแข็งด้าน Decimal Arithmetic ที่มีมาตั้งแต่ Part 001:

- ภาพรวมว่า COBOL สามารถ "สวมบทบาท" เป็น Bounded Context Service หนึ่งตัวในระบบ Microservices ได้อย่างไร
  (ขั้นตอนที่ 861)
- COBOL Service ทำงานร่วมกับ API Gateway อย่างไร ต่อยอดจาก REST Wrapper ของ Part 073 (ขั้นตอนที่ 862)
- แนวคิด Event-Driven Architecture พร้อมคำประกาศที่ตรงไปตรงมาว่าสภาพแวดล้อมนี้ไม่มี Kafka/RabbitMQ จริง
  และใช้ไฟล์เป็นตัวแทนแนวคิดแทน (ขั้นตอนที่ 863)
- สร้าง `PRODUCER.cob` ที่เขียนข้อความเข้าคิวแบบไฟล์จริง พร้อมกลไกเลขลำดับที่ไม่ซ้ำกัน (ขั้นตอนที่ 864)
- สร้าง `CONSUMER.cob` ที่ประมวลผลแบบ Idempotent พิสูจน์ด้วยการทดสอบจริงว่าการ Retry ทั้งคิวไม่ทำให้เกิด
  การนับซ้ำ (ขั้นตอนที่ 865)
- พิสูจน์เพิ่มเติมว่ากลไกเดียวกันคุ้มครองกรณีความล้มเหลวกลางคัน (Partial Batch Retry) ได้เช่นกัน
  (ขั้นตอนที่ 866)
- แนวคิด Retry ขั้นสูง: Exponential Backoff, Retry Limit, และ Dead Letter Queue (ขั้นตอนที่ 867)
- เปรียบเทียบตัวอย่างในหลักสูตรกับ Message Broker จริงอย่างตรงไปตรงมา ทั้งสิ่งที่เหมือนและต่างกัน
  (ขั้นตอนที่ 868)
- Checklist คุณสมบัติที่ COBOL Service ควรมีก่อนเข้าร่วมระบบกระจาย พร้อมประเมิน AR-MINI จาก Part 085
  เทียบกับ Checklist นี้ (ขั้นตอนที่ 869)
- สรุปบทเรียนสำคัญที่สุด (Idempotency เมื่อสื่อสารแบบ Asynchronous) และเชื่อมโยงสู่ Part ถัดไป
  (ขั้นตอนที่ 870)

**ทุกโปรแกรม COBOL ในเอกสารนี้ (`PRODUCER.cob`, `CONSUMER.cob`, และเวอร์ชันทดสอบการจำลองความล้มเหลว)
ผ่านการคอมไพล์และรันทดสอบจริงทั้งหมด** พิสูจน์ให้เห็นว่าแนวคิด Idempotency และ Retry ที่สำคัญต่อระบบ
กระจายสมัยใหม่ สามารถนำไปปฏิบัติจริงด้วย COBOL ได้อย่างเป็นรูปธรรม ไม่ใช่แค่ทฤษฎีที่จำกัดอยู่ในภาษา
โปรแกรมมิ่งสมัยใหม่เท่านั้น

**[ไปยัง Part 088: Performance Optimization ระดับสูง →](part-088-performance-optimization.md)**
