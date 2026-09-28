# Part 076: Containerizing COBOL ด้วย Docker (ขั้นตอนที่ 751–760)

## คำนำของ Part นี้

Part 073-075 สร้างระบบ COBOL สมัยใหม่ครบชุดแล้ว: Wrapper Service ที่เชื่อมกับ REST API (Part 073),
Web Layer ที่ค้นหาข้อมูลผ่าน Indexed File (Part 074), และการเชื่อมต่อฐานข้อมูล PostgreSQL ผ่าน
Shim Program (Part 075) แต่ระบบเหล่านี้ยังมีปัญหาหนึ่งที่คุ้นเคยกันดีในวงการพัฒนาซอฟต์แวร์:
**"มันทำงานบนเครื่องผม แต่ทำไมไม่ทำงานบนเครื่องอื่น?"** — ปัญหาที่เกิดจากความแตกต่างของสภาพแวดล้อม
เช่น เวอร์ชัน GnuCOBOL ที่ต่างกัน, `LD_LIBRARY_PATH` ที่ตั้งค่าต่างกัน (ตามที่พิสูจน์เป็นปัญหาจริง
ใน Part 074 ขั้นตอนที่ 738), หรือ library ที่ขาดหายไป

**Docker** คือเทคโนโลยี Containerization ที่แก้ปัญหานี้โดยการ "บรรจุ" โปรแกรมพร้อมกับทุกสิ่งที่มัน
ต้องการ (library, runtime, configuration) ไว้ในหน่วยเดียวที่เรียกว่า **Container Image** ทำให้
รันได้เหมือนกันทุกประการไม่ว่าจะ deploy ไปที่เครื่องไหนก็ตาม Part นี้จะสอนวิธีสร้าง Docker Image
สำหรับโปรแกรม COBOL อย่างถูกต้องตามมาตรฐาน โดยใช้เทคนิค **Multi-Stage Build** ที่แยกขั้นตอน
"คอมไพล์" ออกจากขั้นตอน "รัน" เพื่อให้ได้ image ขนาดเล็กและปลอดภัยที่สุด

> **ความซื่อตรงทางเทคนิคที่สำคัญมากสำหรับ Part นี้**: สภาพแวดล้อมที่ใช้พัฒนาหลักสูตรนี้มีคำสั่ง
> `docker` ติดตั้งอยู่ (Docker version 29.3.1) **แต่ไม่มี Docker daemon ทำงานอยู่เบื้องหลัง**
> (สภาพแวดล้อมแบบ Container-in-Container/sandbox จำนวนมากปิดความสามารถนี้ไว้ด้วยเหตุผลด้านความ
> ปลอดภัย) เมื่อทดสอบจริงด้วยคำสั่ง `docker build` จะได้ error ทันที (แสดงไว้ในขั้นตอนที่ 752)
> ดังนั้น **Dockerfile และ docker-compose.yml ทุกไฟล์ใน Part นี้เขียนขึ้นตามไวยากรณ์มาตรฐานที่
> ถูกต้องอย่างมั่นใจ และอ้างอิงชื่อ package จริงที่ตรวจสอบแล้วว่ามีอยู่จริงบนระบบปฏิบัติการที่ใช้ใน
> สภาพแวดล้อมนี้ (Ubuntu 24.04) แต่ไม่สามารถรัน `docker build`/`docker run` เพื่อพิสูจน์ผลลัพธ์
> จริงได้ในสภาพแวดล้อมนี้** — Part นี้จะระบุอย่างชัดเจนทุกจุดว่าอะไรทดสอบได้จริงและอะไรเป็นไวยากรณ์
> มาตรฐานที่ยังไม่ได้พิสูจน์ด้วยการรันจริง เช่นเดียวกับหลักการที่ยึดถือมาตลอดหลักสูตร (Part 045, 075)

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างเป็นภาษาอังกฤษล้วน ไฟล์ Dockerfile/YAML/
> โค้ด Python/JavaScript ก็เป็นภาษาอังกฤษล้วนตามธรรมชาติของไฟล์เหล่านี้เช่นกัน

---

## ขั้นตอนที่ 751: ทำไมต้อง Containerize COBOL — แนวคิดพื้นฐานของ Docker

### ปัญหาที่ Docker แก้ไข

ตลอด Part 073-075 เราพบปัญหาเรื่องสภาพแวดล้อมซ้ำแล้วซ้ำเล่า:

- Part 074 ขั้นตอนที่ 732: ต้องใช้ `/opt/gnucobol-isam/bin/cobc` แทน `cobc` มาตรฐาน เพราะบิลด์
  ต่างกันรองรับ feature ต่างกัน
- Part 074 ขั้นตอนที่ 738: โปรแกรมที่คอมไพล์ถูกต้อง crash ด้วย Segmentation Fault เมื่อรันโดยไม่
  ตั้งค่า `LD_LIBRARY_PATH` ให้ตรงกับ runtime library ที่ต้องการ
- Part 075: ต้องมี PostgreSQL client library (`libpq`) ติดตั้งไว้ก่อนจึงจะคอมไพล์ `pgshim.c` ได้

ปัญหาทั้งหมดนี้มีรากฐานเดียวกัน: **โปรแกรมพึ่งพาสภาพแวดล้อมที่มันถูกพัฒนาขึ้นมาก โดยไม่รู้ตัว**
เมื่อนำโปรแกรมไปรันบนเครื่องอื่นที่สภาพแวดล้อมไม่เหมือนกันเป๊ะ ปัญหาที่ไม่คาดคิดก็เกิดขึ้น

### Docker แก้ปัญหานี้อย่างไร

Docker บรรจุโปรแกรมพร้อมกับ **ทุกสิ่งที่จำเป็นสำหรับการรัน** (compiler runtime library, ระบบ
ปฏิบัติการฐาน, environment variable ที่ต้องตั้งค่า) ไว้ในหน่วยเดียวที่เรียกว่า **Image** เมื่อสร้าง
**Container** จาก Image นี้บนเครื่องไหนก็ตาม (ที่มี Docker Engine ติดตั้งอยู่) โปรแกรมจะทำงาน
เหมือนกันทุกประการ **ไม่ว่าเครื่องนั้นจะติดตั้ง GnuCOBOL เวอร์ชันอะไรอยู่ หรือไม่ได้ติดตั้งเลยก็ตาม**
เพราะทุกอย่างที่จำเป็นถูกบรรจุไว้ใน Image แล้ว

### Container ต่างจาก Virtual Machine (VM) อย่างไร

| คุณสมบัติ | Virtual Machine | Docker Container |
|---|---|---|
| สิ่งที่จำลอง | ฮาร์ดแวร์ทั้งชุด รวมถึง OS Kernel ของตัวเอง | ใช้ OS Kernel ร่วมกับเครื่อง host โดยตรง |
| ขนาด | ใหญ่ (หลาย GB ต่อ VM) | เล็กกว่ามาก (หลัก MB ถึงร้อย MB) |
| เวลาเริ่มทำงาน | ช้า (นาที) | เร็วมาก (วินาที) |
| ความหนาแน่น (จำนวนที่รันพร้อมกันได้) | น้อยกว่า (ใช้ทรัพยากรมาก) | มากกว่า (ใช้ทรัพยากรน้อยกว่า) |

Container จึงเหมาะกับการ deploy โปรแกรมจำนวนมากอย่างมีประสิทธิภาพ ซึ่งเป็นเหตุผลสำคัญที่วงการ
Modernization นิยมใช้ Docker/Container เป็นเป้าหมายในการย้ายระบบ Legacy ออกจากสภาพแวดล้อม
Mainframe/เซิร์ฟเวอร์เฉพาะทางเดิม

### ทำไม Container จึงสำคัญกับการทำ COBOL Modernization โดยเฉพาะ

ระบบ COBOL Legacy จำนวนมากถูกพัฒนาและ deploy บนเซิร์ฟเวอร์เฉพาะทางที่ตั้งค่าไว้ด้วยมือมานานหลายปี
(บางครั้งไม่มีเอกสารบันทึกการตั้งค่าไว้ครบถ้วนด้วยซ้ำ) การ Containerize ระบบเหล่านี้บังคับให้ทีม
พัฒนาต้อง**เขียนทุกขั้นตอนการติดตั้งและตั้งค่าลงในไฟล์ Dockerfile อย่างชัดเจน** ทำให้เกิดเอกสารที่
เป็น "ความจริง" (source of truth) ของสภาพแวดล้อมที่จำเป็น และทำให้ระบบนั้น**ทำซ้ำได้ (reproducible)**
บนเครื่องใดก็ได้ที่มี Docker — เป็นก้าวแรกที่สำคัญมากก่อนจะย้ายระบบไปสู่ Cloud (เนื้อหานี้จะต่อยอด
เต็มรูปแบบใน **Part 077**)

### ข้อควรระวัง

- Docker ไม่ได้ "แก้" ปัญหา business logic ที่ผิดพลาดในตัวโปรแกรม COBOL เอง มันแก้แค่ปัญหาเรื่อง
  **ความแตกต่างของสภาพแวดล้อมรัน (runtime environment)** เท่านั้น
- การ containerize ระบบ COBOL ไม่ได้แปลว่าต้องเปลี่ยนโค้ด COBOL เลยแม้แต่บรรทัดเดียว — Docker
  ทำงานที่ระดับ "สภาพแวดล้อมที่ห่อหุ้มโปรแกรม" ไม่ใช่ระดับตัวโปรแกรมเอง

### แบบฝึกหัดที่ 751.1

**โจทย์**: จงอธิบายว่าปัญหา Segmentation Fault ที่พบใน Part 074 ขั้นตอนที่ 738 (เรื่อง
`LD_LIBRARY_PATH`) จะไม่เกิดขึ้นอีกหากระบบถูก containerize ไว้อย่างถูกต้อง เพราะเหตุใด

**เฉลยแนวทาง**: หาก Dockerfile ถูกเขียนให้ติดตั้ง runtime library ที่ถูกต้อง (`libcob5t64` ที่มี
ISAM handler ตรงกับตัวคอมไพเลอร์ที่ใช้คอมไพล์โปรแกรม) และตั้งค่า `LD_LIBRARY_PATH` (หากจำเป็น) ไว้
ใน Dockerfile ตั้งแต่ตอนสร้าง Image แล้ว ทุก Container ที่สร้างจาก Image นั้นจะมีสภาพแวดล้อมเดียวกัน
ทุกประการเสมอ ไม่ว่าจะรันบนเครื่องไหนก็ตาม ปัญหาแบบ "เครื่องพัฒนาตั้งค่าไว้ถูกต้องแต่เครื่อง
production ลืมตั้งค่า" จะไม่เกิดขึ้นอีก เพราะการตั้งค่าทั้งหมดถูกบันทึกไว้ในไฟล์ Dockerfile ที่ใช้
สร้าง Image เดียวกันสำหรับทุกสภาพแวดล้อม

---

## ขั้นตอนที่ 752: ตรวจสอบสภาพแวดล้อม — Docker พร้อมใช้งานหรือไม่

### ตรวจสอบ Docker CLI

```bash
docker --version
```

**ผลลัพธ์จริง:**

```
Docker version 29.3.1, build c2be9cc
```

คำสั่ง `docker` มีอยู่จริงในสภาพแวดล้อมนี้ ดูเหมือนพร้อมใช้งาน — แต่การมีคำสั่ง `docker` ไม่ได้
แปลว่า Docker Engine (daemon) ที่ทำหน้าที่สร้างและรัน container จริง ๆ กำลังทำงานอยู่เสมอไป

### ทดสอบจริง — ลองสร้าง Image

```bash
docker build -t test076 .
```

**ผลลัพธ์จริงที่ได้ (error จริง ไม่ใช่การจำลอง):**

```
ERROR: failed to connect to the docker API at unix:///var/run/docker.sock; check if the path is correct and if the daemon is running: dial unix /var/run/docker.sock: connect: no such file or directory
```

### วิเคราะห์สาเหตุ

ข้อความ error บอกชัดเจนว่า Docker CLI พยายามเชื่อมต่อกับ **Docker daemon** ผ่าน Unix socket
(`/var/run/docker.sock`) แต่ไม่พบไฟล์ socket นี้เลย — หมายความว่า **Docker daemon ไม่ได้ทำงานอยู่
ในสภาพแวดล้อมนี้** สาเหตุที่พบบ่อยของสถานการณ์นี้คือสภาพแวดล้อมแบบ Sandbox/Container ที่ใช้พัฒนา
หลักสูตรนี้ทำงานอยู่**ภายใน container ของตัวเองอยู่แล้ว** และไม่ได้เปิดใช้งานความสามารถ
"Docker-in-Docker" (การรัน Docker daemon ซ้อนอยู่ภายใน container อีกชั้น) ไว้ ด้วยเหตุผลด้านความ
ปลอดภัยและความซับซ้อนของการตั้งค่า

ตรวจสอบเพิ่มเติมพบว่าไม่มีเครื่องมือทางเลือกอื่น (เช่น `podman`, `buildah`, `nerdctl`) ติดตั้งไว้ใน
สภาพแวดล้อมนี้เช่นกัน จึงยืนยันได้ว่า**ไม่มีวิธีสร้างหรือรัน Container ได้จริงในสภาพแวดล้อมของ
หลักสูตรนี้เลย**

### สิ่งที่หลักสูตรนี้จะทำต่อจากนี้ (และทำไมยังคุ้มค่าที่จะเรียน)

แม้จะรันไม่ได้จริง เนื้อหาที่เหลือของ Part นี้ยังคงมีคุณค่ามหาศาล เพราะ:

1. **ไวยากรณ์ของ Dockerfile และ docker-compose.yml เป็นมาตรฐานสากลที่ไม่เปลี่ยนแปลงบ่อย** —
   สิ่งที่เขียนถูกต้องในวันนี้ยังคงถูกต้องเมื่อคุณนำไปรันบนเครื่องของคุณเองที่มี Docker daemon
   ทำงานอยู่จริง
2. **ชื่อ package ที่อ้างอิงในไฟล์เหล่านี้ (`gnucobol4`, `libcob5t64`) ถูกตรวจสอบแล้วว่ามีอยู่จริง**
   บนระบบปฏิบัติการ Ubuntu 24.04 ที่ใช้เป็นฐานของสภาพแวดล้อมนี้เอง (ตรวจสอบด้วย `dpkg -l` ในระหว่าง
   การพัฒนา Part นี้) จึงมั่นใจได้ในระดับสูงว่าถูกต้อง แม้จะยังไม่ได้ผ่านการ build image จริง
3. เข้าใจ**แนวคิด**ของ Multi-Stage Build และการออกแบบ Image ที่ปลอดภัย มีค่ามากกว่าการท่องจำ syntax
   เพียงอย่างเดียว

### ข้อควรระวัง

- **หากคุณกำลังฝึกฝนบนเครื่องของตัวเองที่มี Docker Desktop หรือ Docker Engine ติดตั้งและทำงานอยู่
  จริง ให้ลองรันคำสั่งทุกคำสั่งใน Part นี้ด้วยตัวเองเพื่อพิสูจน์ผลลัพธ์จริง** — หลักสูตรนี้ยืนยันเพียง
  ความถูกต้องของไวยากรณ์ ไม่ได้ยืนยันว่าไม่มี edge case ใด ๆ ที่อาจเกิดขึ้นบนสภาพแวดล้อมที่ต่างออกไป
- อย่าเชื่อคำอธิบายใด ๆ ว่า "รันได้แน่นอน 100%" หากไม่มีการทดสอบจริงยืนยัน — นี่คือหลักการความซื่อ
  ตรงทางเทคนิคที่หลักสูตรนี้ยึดถือมาตลอด (Part 045, Part 075)

### แบบฝึกหัดที่ 752.1

**โจทย์**: จงอธิบายว่าทำไมข้อความ error `"dial unix /var/run/docker.sock: connect: no such file
or directory"` จึงบ่งบอกปัญหาระดับ "Docker daemon ไม่ทำงาน" ไม่ใช่ปัญหาระดับ "Dockerfile เขียนผิด"

**เฉลย**: error เกิดขึ้น**ก่อน**ที่ Docker CLI จะอ่านเนื้อหาของ Dockerfile ด้วยซ้ำ — ขั้นตอนแรกสุด
ของคำสั่ง `docker build` คือ CLI ต้องเชื่อมต่อกับ Docker daemon (ผ่าน Unix socket) เพื่อส่งคำสั่ง
สร้าง image ไปให้ daemon ทำงานจริง เมื่อ CLI หา socket ไฟล์นี้ไม่เจอเลย แปลว่ายังไม่ได้ไปถึงขั้นตอน
การอ่าน Dockerfile เลยด้วยซ้ำ จึงสรุปได้อย่างมั่นใจว่าปัญหาอยู่ที่การไม่มี daemon ทำงานอยู่เบื้องหลัง
ไม่เกี่ยวข้องกับเนื้อหาของ Dockerfile แต่อย่างใด (คล้ายกับหลักการวิเคราะห์ error message ที่เรียนมา
จาก Part 045 ขั้นตอนที่ 442: ต้องแยกแยะให้ออกว่า error เกิดจาก "โค้ด/ไฟล์ผิด" หรือ "สภาพแวดล้อมไม่
พร้อม")

---

## ขั้นตอนที่ 753: หลักการ Multi-Stage Build และทำไมต้องแยก Build Stage กับ Runtime Stage

### ปัญหาของ Single-Stage Build

หากเขียน Dockerfile แบบง่ายที่สุด (single stage) ที่ติดตั้งทั้ง compiler และ build tools ไว้ใน
image เดียวกับที่จะใช้รันจริง จะเกิดปัญหา: **Image สุดท้ายมีขนาดใหญ่เกินความจำเป็นมาก** เพราะบรรจุ
เครื่องมือสำหรับ "สร้าง" โปรแกรม (compiler, header files, build tools) ที่ไม่จำเป็นต้องใช้อีกเลย
หลังจากคอมไพล์เสร็จแล้ว — เปรียบเสมือนขนส่งทั้งโรงงานไปพร้อมกับสินค้าที่ผลิตเสร็จแล้ว ทั้งที่ผู้รับ
ปลายทางต้องการแค่ตัวสินค้าเท่านั้น

### แนวคิด Multi-Stage Build

Docker รองรับการเขียน Dockerfile ที่มีหลาย `FROM` ในไฟล์เดียว เรียกว่า **Multi-Stage Build**
แต่ละ `FROM` เริ่ม "stage" ใหม่ ทำให้เราแยกงานออกเป็น:

1. **Build Stage (Stage 1)**: ใช้ image ฐานที่มี compiler และเครื่องมือพัฒนาครบ (`gnucobol4`
   ซึ่งรวม `cobc` และ header files) คอมไพล์โปรแกรม COBOL ให้เป็นไฟล์ executable
2. **Runtime Stage (Stage 2)**: เริ่มจาก image ฐานใหม่ที่**เล็กที่สุดเท่าที่จำเป็น** (ไม่มี
   compiler เลย) แล้ว **copy เฉพาะไฟล์ executable ที่คอมไพล์เสร็จแล้ว** จาก Stage 1 มาไว้ พร้อม
   ติดตั้งเฉพาะ runtime library ที่จำเป็นสำหรับการรัน (`libcob5t64` เท่านั้น ไม่ต้องมี `gnucobol4`
   เต็มชุด)

### ตรวจสอบชื่อ Package จริงบนระบบปฏิบัติการที่ใช้เป็นฐาน

ก่อนเขียน Dockerfile เราตรวจสอบชื่อ package ที่ถูกต้องจริงบนสภาพแวดล้อม Ubuntu 24.04 (ระบบ
ปฏิบัติการเดียวกับที่ใช้พัฒนาหลักสูตรนี้) ด้วยคำสั่ง:

```bash
dpkg -l | grep -i cobol
dpkg -l | grep -i libcob
```

**ผลลัพธ์จริงที่ได้:**

```
ii  gnucobol4          4.0~early~20200606-6.1build1  amd64  COBOL compiler
ii  libcob5-dev:amd64  4.0~early~20200606-6.1build1  amd64  COBOL compiler - development files
ii  libcob5t64:amd64   4.0~early~20200606-6.1build1  amd64  COBOL compiler - runtime library
```

สามชื่อ package นี้ (`gnucobol4`, `libcob5-dev`, `libcob5t64`) คือชื่อจริงที่ยืนยันแล้วว่ามีอยู่ใน
Ubuntu 24.04 repository — จะนำมาใช้ใน Dockerfile ของขั้นตอนถัดไป (สังเกตว่า `libcob5t64` มีคำ
ต่อท้าย `t64` ซึ่งเป็นธรรมเนียมการตั้งชื่อของ Ubuntu 24.04 ที่เกี่ยวข้องกับการเปลี่ยนขนาด
`time_t` เป็น 64-bit บนสถาปัตยกรรมบางแบบ — เป็นรายละเอียดของระบบปฏิบัติการ ไม่ใช่เรื่องเฉพาะของ
GnuCOBOL)

### ประโยชน์ของ Multi-Stage Build สรุปเป็นตาราง

| ประเด็น | Single-Stage (ไม่แยก) | Multi-Stage (แยก Build/Runtime) |
|---|---|---|
| ขนาด Image สุดท้าย | ใหญ่ (มี compiler + build tools ติดไปด้วย) | เล็กกว่ามาก (มีแค่ runtime library) |
| พื้นที่โจมตี (Attack Surface) | กว้างกว่า (มี compiler ที่อาจถูกใช้โจมตีต่อได้หากมีช่องโหว่) | แคบกว่า (ไม่มี compiler ใน production image เลย) |
| ความเร็วในการ pull/deploy image | ช้ากว่า (ไฟล์เยอะกว่า) | เร็วกว่า |
| ความชัดเจนของ Dockerfile | ปนกันระหว่างขั้นตอน build กับ runtime | แยกชัดเจนเป็นสัดส่วน อ่านง่ายกว่า |

### ข้อควรระวัง

- ต้องระบุชื่อ stage อย่างชัดเจนด้วยคำสั่ง `AS <stage-name>` ต่อท้าย `FROM` เสมอ (เช่น
  `FROM ubuntu:24.04 AS builder`) เพื่อให้ stage ถัดไปอ้างอิงกลับมา copy ไฟล์จาก stage นี้ได้
  ด้วยคำสั่ง `COPY --from=builder ...`
- Multi-Stage Build ไม่ได้จำกัดแค่ 2 stage เท่านั้น — โครงการที่ซับซ้อนอาจมีหลาย stage มากกว่านี้
  (เช่น stage แยกสำหรับรัน unit test ก่อน build stage จริง) แต่สำหรับ Part นี้ 2 stage เพียงพอ
  สำหรับสาธิตแนวคิดหลัก

### แบบฝึกหัดที่ 753.1

**โจทย์**: จงอธิบายว่าทำไม image ที่ใช้ `libcob5t64` เพียงอย่างเดียว (ไม่มี `gnucobol4` เต็มชุด)
จึงยังคงรันโปรแกรม COBOL ที่คอมไพล์เสร็จแล้วได้ปกติ ทั้งที่ไม่มีตัวคอมไพเลอร์ `cobc` อยู่ใน image นั้น
เลย

**เฉลย**: เพราะไฟล์ executable ที่ได้จากการคอมไพล์ด้วย `cobc -x` (ทบทวนจาก Part 002) ไม่ใช่ไฟล์
COBOL source code ที่ต้องแปลใหม่ทุกครั้งที่รัน แต่เป็น**ไฟล์ binary ที่คอมไพล์เสร็จสมบูรณ์แล้ว**
คล้ายโปรแกรมที่เขียนด้วย C หรือภาษาอื่นที่คอมไพล์เป็น machine code สิ่งที่โปรแกรม binary นี้ต้องการ
ตอน**รัน**มีแค่ **runtime library** (`libcob.so` ที่มาจาก package `libcob5t64`) ที่มีฟังก์ชัน
พื้นฐานที่โปรแกรม COBOL เรียกใช้ตอนทำงาน (เช่นการจัดการ `DISPLAY`, การคำนวณ decimal arithmetic)
ไม่ต้องการตัวคอมไพเลอร์ `cobc` เลยแม้แต่น้อยในขั้นตอนการรัน เพราะการคอมไพล์เกิดขึ้นเสร็จสิ้นไปแล้ว
ใน Build Stage ก่อนหน้านี้

---

## ขั้นตอนที่ 754: เขียน Dockerfile แบบ Multi-Stage สำหรับ ORDERCALC (Part 073)

### โครงสร้างไฟล์ที่จะใช้

เราจะ containerize ระบบจาก Part 073 ทั้งหมด: โปรแกรม COBOL `ORDERCALC` และ Wrapper Service
`wrapper.py` (เวอร์ชันที่แก้ไขแล้วจากขั้นตอนที่ 728) โดยสมมติโครงสร้างไฟล์ในโปรเจกต์ดังนี้:

```
order-service/
  ├── ordercalc.cob
  ├── wrapper.py
  └── Dockerfile
```

### Dockerfile ฉบับสมบูรณ์

```dockerfile
# Stage 1: Build - compile the COBOL program using the full
# GnuCOBOL toolchain. This stage is discarded from the final
# image; only its compiled output is kept.
FROM ubuntu:24.04 AS builder

RUN apt-get update && \
    apt-get install -y --no-install-recommends gnucobol4 && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /build
COPY ordercalc.cob .
RUN cobc -x -o ordercalc ordercalc.cob

# Stage 2: Runtime - start from a fresh minimal base with only
# the COBOL runtime library and Python (for the wrapper service).
# No compiler, no build tools, no header files here.
FROM ubuntu:24.04 AS runtime

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libcob5t64 python3 && \
    rm -rf /var/lib/apt/lists/*

RUN useradd --create-home --shell /usr/sbin/nologin cobolapp
WORKDIR /app
COPY --from=builder /build/ordercalc .
COPY wrapper.py .
RUN chown -R cobolapp:cobolapp /app

USER cobolapp
EXPOSE 8073
CMD ["python3", "wrapper.py"]
```

### อธิบายทีละส่วน

- `FROM ubuntu:24.04 AS builder`: เริ่ม Stage 1 ชื่อ `builder` จาก Ubuntu 24.04 official image
  (base image เดียวกับสภาพแวดล้อมของหลักสูตรนี้เอง เพื่อความสอดคล้องของ package ที่ตรวจสอบไว้แล้ว)
- `apt-get install -y --no-install-recommends gnucobol4`: ติดตั้งเฉพาะ `gnucobol4` (ซึ่งรวม
  `cobc`, `cobcrun` และ library ที่จำเป็นสำหรับการคอมไพล์) ตัวเลือก `--no-install-recommends`
  ช่วยลดจำนวน package เสริมที่ไม่จำเป็นให้น้อยที่สุด
- `RUN cobc -x -o ordercalc ordercalc.cob`: คอมไพล์โปรแกรม COBOL ด้วยคำสั่งเดียวกับที่ใช้มาตลอด
  หลักสูตร (ทบทวนจาก Part 002) — เพียงแต่ครั้งนี้รันอยู่**ภายใน** Docker build process
- `FROM ubuntu:24.04 AS runtime`: เริ่ม Stage 2 ใหม่ทั้งหมดจาก base image เดียวกัน แต่**ไม่ได้
  สืบทอดสิ่งที่ติดตั้งไว้ใน Stage 1 เลย** — เป็นจุดเริ่มต้นที่สะอาดอย่างสมบูรณ์
- `apt-get install ... libcob5t64 python3`: ติดตั้งเฉพาะ runtime library ของ COBOL (ไม่มี
  compiler) และ Python 3 (สำหรับรัน `wrapper.py`) — สังเกตว่า**ไม่มี `gnucobol4` ในบรรทัดนี้เลย**
- `useradd --create-home --shell /usr/sbin/nologin cobolapp` และ `USER cobolapp`: สร้างผู้ใช้
  ธรรมดา (ไม่ใช่ `root`) และสั่งให้ container รันโปรแกรมด้วยสิทธิ์ผู้ใช้นี้แทน — หลักการความปลอดภัย
  พื้นฐานที่สำคัญมาก จะอธิบายเต็มรูปแบบในขั้นตอนที่ 758
- `COPY --from=builder /build/ordercalc .`: คำสั่งหัวใจของ Multi-Stage Build — คัดลอกเฉพาะไฟล์
  executable ที่คอมไพล์เสร็จแล้วจาก Stage `builder` มาไว้ใน Stage `runtime` โดยไม่ต้องนำ source
  code, compiler หรือ build tools ติดมาด้วยเลย
- `EXPOSE 8073`: ประกาศ (เชิงเอกสาร) ว่า container นี้ตั้งใจให้เข้าถึงผ่าน port 8073 — ตรงกับ
  port ที่ `wrapper.py` bind ไว้ตั้งแต่ Part 073
- `CMD ["python3", "wrapper.py"]`: คำสั่งที่จะรันเมื่อ container เริ่มทำงาน

### ข้อควรระวัง

- **สำคัญ**: โค้ด `wrapper.py` เดิมจาก Part 073 bind ที่ `("127.0.0.1", 8073)` ซึ่ง**ใช้ไม่ได้ใน
  บริบทของ Container** เพราะจะรับ connection ได้เฉพาะจากภายใน container เดียวกันเท่านั้น ทำให้
  เครื่อง host ที่รัน `docker run` เข้าถึงไม่ได้เลย ต้องแก้เป็น `("0.0.0.0", 8073)` เพื่อให้รับ
  connection จากภายนอก container ได้ (ทบทวนความแตกต่างของการ bind ที่ 127.0.0.1 กับ 0.0.0.0
  จาก Part 073 ขั้นตอนที่ 725)
- ลำดับของคำสั่งใน Dockerfile มีผลต่อ **build cache**: Docker cache ผลลัพธ์ของแต่ละบรรทัดไว้ หาก
  บรรทัดก่อนหน้าไม่เปลี่ยนแปลง การจัดเรียงให้ `COPY ordercalc.cob` และ `RUN cobc ...` มาก่อน
  `COPY wrapper.py` (ที่มักเปลี่ยนบ่อยกว่า) ช่วยให้ build ครั้งถัดไปเร็วขึ้นเมื่อแก้แค่ไฟล์ Python

### แบบฝึกหัดที่ 754.1

**โจทย์**: จงอธิบายว่าทำไมการรวม `python3` ไว้ใน Runtime Stage จึงไม่ขัดกับหลักการ "Runtime
Stage ควรมีแค่สิ่งจำเป็นสำหรับการรันเท่านั้น" ทั้งที่ `python3` ก็เป็น "ตัวแปลภาษา" เหมือนกับ
compiler

**เฉลย**: หลักการ "มีแค่สิ่งจำเป็นสำหรับการรัน" ไม่ได้หมายความว่าต้องไม่มีตัวแปลภาษาใด ๆ เลย
แต่หมายถึง **ไม่ควรมีเครื่องมือที่ใช้แค่ตอน "สร้าง" โปรแกรมแล้วไม่ได้ใช้อีกเลยตอน "รัน"** —
`gnucobol4` (ที่มี `cobc`) ใช้แค่ตอนคอมไพล์ COBOL source code ให้เป็น binary เพียงครั้งเดียว
หลังจากนั้นไม่จำเป็นอีกเลย ในขณะที่ `python3` ยังคง**จำเป็นตลอดเวลาที่ container ทำงานอยู่** เพราะ
`wrapper.py` เป็น Python script ที่ต้องถูก interpret (แปลความหมาย) ทุกครั้งที่รัน ไม่ได้ถูกคอมไพล์
เป็น binary ล่วงหน้าเหมือน COBOL ดังนั้น `python3` จึงจัดอยู่ในหมวด "runtime dependency" ที่ต้องมี
อยู่ใน Runtime Stage ไม่ใช่ "build tool" ที่ควรถูกตัดทิ้งเหมือน `gnucobol4`

---

## ขั้นตอนที่ 755: การจำลองขั้นตอน Build และ Run (Reference เท่านั้น — ไม่ได้ทดสอบจริงในสภาพแวดล้อมนี้)

### คำสั่งที่ควรใช้บนเครื่องที่มี Docker daemon ทำงานจริง

หากคุณกำลังฝึกฝนบนเครื่องที่มี Docker Desktop หรือ Docker Engine ทำงานอยู่จริง คำสั่งต่อไปนี้ควร
ใช้งานได้ (**อ้างอิงไวยากรณ์มาตรฐาน — ยังไม่ได้รันจริงในสภาพแวดล้อมของหลักสูตรนี้ ตามที่ระบุไว้ใน
ขั้นตอนที่ 752**):

```bash
# 1. Build the image from the Dockerfile in the current directory
docker build -t ordercalc-service:1.0 .

# 2. Run a container from that image, mapping host port 8073
#    to the container's port 8073
docker run -d -p 8073:8073 --name ordercalc ordercalc-service:1.0

# 3. Test it exactly like we did in Part 073 - from the host
#    machine, not from inside the container
curl -s -X POST http://127.0.0.1:8073/api/order \
    -H "Content-Type: application/json" \
    -d '{"price": 100.00, "quantity": 3}'

# 4. Check the container's logs if something goes wrong
docker logs ordercalc

# 5. Stop and remove the container when done
docker stop ordercalc
docker rm ordercalc
```

### ผลลัพธ์ที่คาดว่าจะได้ (คาดการณ์จาก logic ที่ทดสอบแล้วจริงใน Part 073 — ไม่ใช่ผลลัพธ์ที่รันจริง
ผ่าน Docker ในสภาพแวดล้อมนี้)

เนื่องจากโค้ด `ordercalc.cob` และ `wrapper.py` (หลังแก้ bind address ตามขั้นตอนที่ 754) เป็น
โค้ดชุดเดียวกันเป๊ะกับที่ทดสอบสำเร็จแบบ end-to-end นอก container ไปแล้วใน Part 073 ขั้นตอนที่ 727
ตรรกะทางโปรแกรมจึงควรให้ผลลัพธ์เดียวกัน:

```
{"price": 100.0, "quantity": 3, "subtotal": 300.0, "tax": 21.0, "total": 321.0}
```

**เหตุผลที่เรามั่นใจในระดับสูงว่าผลลัพธ์จะตรงกัน** (แม้ไม่ได้พิสูจน์ด้วย Docker จริง): Multi-Stage
Build เพียงแค่เปลี่ยน "ที่ที่โปรแกรมรัน" (จากรันตรงบนเครื่องพัฒนา เป็นรันภายใน container) ไม่ได้
เปลี่ยนแปลง logic ของโปรแกรมโค้ดเองเลยแม้แต่บรรทัดเดียว ตราบใดที่ runtime library ที่ container
ติดตั้งไว้ (`libcob5t64`) ตรงกับที่โปรแกรมต้องการ (ซึ่ง `gnucobol4` และ `libcob5t64` มาจาก
package version เดียวกันเสมอ เพราะเป็น package ชุดเดียวกันของ Ubuntu 24.04) ผลลัพธ์ก็ควรจะ
เหมือนกัน

### ข้อควรระวัง

- **ห้ามนำเสนอผลลัพธ์ในขั้นตอนนี้เป็นเหมือนกับผลลัพธ์ที่ทดสอบจริงในขั้นตอนอื่น ๆ ของหลักสูตร** —
  ผลลัพธ์ในขั้นตอนนี้คือ "การคาดการณ์อย่างมีเหตุผลจาก logic ที่ทดสอบแล้ว" เท่านั้น ไม่ใช่การยืนยัน
  ด้วยการรันจริงผ่าน Docker หากคุณมี Docker daemon ทำงานจริง ควรทดลองรันด้วยตัวเองเพื่อยืนยันผลลัพธ์
  อีกครั้งเสมอ
- หากผลลัพธ์จริงที่คุณได้ต่างจากที่คาดการณ์ไว้ สาเหตุที่เป็นไปได้มากที่สุดคือลืมแก้ `wrapper.py`
  ให้ bind ที่ `0.0.0.0` แทน `127.0.0.1` ตามที่เตือนไว้ในขั้นตอนที่ 754

### แบบฝึกหัดที่ 755.1

**โจทย์**: จงอธิบายว่าทำไมการที่เราทดสอบโค้ด `ordercalc.cob` และ `wrapper.py` แบบ end-to-end
สำเร็จมาแล้วใน Part 073 (นอก container) จึงเพิ่มความมั่นใจในระดับหนึ่งว่าจะทำงานได้ถูกต้องเมื่อรัน
ใน Docker container ด้วย แม้จะยังไม่ได้พิสูจน์ด้วยการรันจริงผ่าน Docker

**เฉลยแนวทาง**: เพราะตัวแปรที่ Docker เปลี่ยนแปลงคือ**สภาพแวดล้อมที่ห่อหุ้มโปรแกรม** (runtime
library, network namespace, filesystem) ไม่ใช่ตัว logic ของโปรแกรมโค้ดเอง เมื่อโค้ดเดิมทำงาน
ถูกต้องแล้วนอก container (พิสูจน์แล้วใน Part 073) และ Dockerfile ติดตั้ง runtime library ที่ตรง
กับที่โปรแกรมต้องการ (`libcob5t64` เวอร์ชันเดียวกับที่ `gnucobol4` ใช้คอมไพล์) ก็มีเหตุผลที่ดีมาก
ที่จะเชื่อว่าโปรแกรมจะทำงานเหมือนเดิม อย่างไรก็ตาม นี่ยังคงเป็นการคาดการณ์เชิงตรรกะ ไม่ใช่การพิสูจน์
ด้วยหลักฐานจริง — ความแตกต่างเล็กน้อยที่อาจเกิดขึ้นได้จริง (เช่น network namespace ของ container
ที่ทำให้ `127.0.0.1` ไม่เข้าถึงจากภายนอกได้) ก็เป็นตัวอย่างที่แสดงให้เห็นว่าทำไมการทดสอบจริงเสมอยัง
คงสำคัญ แม้จะมั่นใจในทางทฤษฎีแค่ไหนก็ตาม

---

## ขั้นตอนที่ 756: docker-compose.yml — จัดการหลาย Service พร้อมกัน

### ทำไมต้องใช้ Docker Compose

ระบบจริงมักประกอบด้วยหลาย service ทำงานร่วมกัน (เช่น Wrapper Service + ฐานข้อมูล) การรัน
`docker run` แยกทีละ service พร้อมจำ option ทั้งหมด (port mapping, network, environment
variable) ด้วยมือทุกครั้งนั้นยุ่งยากและเสี่ยงต่อความผิดพลาด **Docker Compose** แก้ปัญหานี้ด้วยการ
ให้เขียนการตั้งค่าทั้งหมดไว้ในไฟล์ YAML ไฟล์เดียว แล้วสั่งเริ่ม/หยุดทุก service พร้อมกันด้วยคำสั่ง
เดียว

### docker-compose.yml สำหรับระบบ ORDERCALC เพียงอย่างเดียว

```yaml
services:
  order-service:
    build: .
    ports:
      - "8073:8073"
    restart: unless-stopped
```

### อธิบายทีละส่วน

- `services:`: กำหนดรายการ service ทั้งหมดที่ระบบต้องการ ในที่นี้มีแค่ 1 service ชื่อ
  `order-service`
- `build: .`: บอกให้ Docker Compose สร้าง image จาก `Dockerfile` ในโฟลเดอร์ปัจจุบัน (เทียบเท่า
  การรัน `docker build .` ด้วยมือ)
- `ports: - "8073:8073"`: map port 8073 ของเครื่อง host เข้ากับ port 8073 ภายใน container
  (รูปแบบ `"host:container"`)
- `restart: unless-stopped`: หาก container หยุดทำงานโดยไม่คาดคิด (เช่น โปรแกรม crash) ให้ Docker
  เริ่ม container ใหม่โดยอัตโนมัติ เว้นแต่จะถูกสั่งหยุดด้วยมือ

### คำสั่งที่ใช้ควบคุม Docker Compose (Reference)

```bash
docker compose up -d     # สร้าง image (ถ้ายังไม่มี) และเริ่ม service ทั้งหมดในโหมด background
docker compose logs -f   # ดู log แบบ real-time
docker compose down      # หยุดและลบ container ทั้งหมด
```

### ข้อควรระวัง

- Docker Compose รุ่นใหม่ (v2 ขึ้นไป) ใช้คำสั่ง `docker compose` (มีช่องว่าง เป็น subcommand ของ
  `docker`) แทนคำสั่งเก่า `docker-compose` (มีขีดกลาง เป็นโปรแกรมแยกต่างหาก) ควรตรวจสอบเวอร์ชันของ
  Docker ที่ใช้งานอยู่ก่อนเสมอว่ารองรับรูปแบบไหน
- ไฟล์ YAML ไวต่อการเยื้องบรรทัด (indentation) มาก การใช้ tab ปนกับ space หรือเยื้องผิดจำนวนช่องว่าง
  เพียงเล็กน้อยอาจทำให้ Docker Compose อ่านไฟล์ผิดพลาดทันที

### แบบฝึกหัดที่ 756.1

**โจทย์**: จงเพิ่ม environment variable `TZ=Asia/Bangkok` เข้าไปใน `docker-compose.yml` ข้างต้น
เพื่อกำหนด timezone ของ container ให้ตรงกับประเทศไทย

**เฉลย**:

```yaml
services:
  order-service:
    build: .
    ports:
      - "8073:8073"
    environment:
      - TZ=Asia/Bangkok
    restart: unless-stopped
```

---

## ขั้นตอนที่ 757: ขยาย docker-compose.yml ให้รวม PostgreSQL (ต่อยอดจาก Part 075)

### เป้าหมาย

เราจะขยายระบบให้ประกอบด้วย 2 service ทำงานร่วมกัน: `order-service` (Wrapper Service + COBOL)
และ `db` (PostgreSQL ที่ทดสอบจริงใน Part 075) — นี่คือรูปแบบสถาปัตยกรรมที่ใกล้เคียงระบบจริงมากขึ้น

### docker-compose.yml ฉบับขยาย

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: cobol123
      POSTGRES_DB: cobolcourse
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  order-service:
    build: .
    ports:
      - "8073:8073"
    environment:
      - DB_HOST=db
      - DB_PASSWORD=cobol123
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pgdata:
```

### อธิบายทีละส่วน

- `image: postgres:16`: ใช้ official PostgreSQL image เวอร์ชัน 16 จาก Docker Hub โดยตรง แทนที่
  จะเขียน Dockerfile เอง เพราะ PostgreSQL มี official image ที่ดูแลรักษาอย่างดีอยู่แล้ว — ตรงกับ
  เวอร์ชัน PostgreSQL 16 ที่ทดสอบจริงใน Part 075
- `environment:`: กำหนดค่าเริ่มต้นสำหรับ PostgreSQL image (username, password, ชื่อฐานข้อมูล
  เริ่มต้น) — image ทางการของ PostgreSQL จะอ่านค่าเหล่านี้และสร้างฐานข้อมูลให้อัตโนมัติตอนเริ่มครั้ง
  แรก
- `volumes: - pgdata:/var/lib/postgresql/data`: เก็บข้อมูลฐานข้อมูลไว้ใน **Docker Volume**
  ชื่อ `pgdata` แทนที่จะเก็บไว้ในตัว container โดยตรง ทำให้ข้อมูลยังคงอยู่แม้ container จะถูกลบและ
  สร้างใหม่ (container คือสิ่งที่ "จบและตายได้" — สร้างใหม่ได้ตลอด แต่ volume คือที่เก็บข้อมูลถาวร
  ที่ควรแยกออกจากวงจรชีวิตของ container)
- `healthcheck:`: กำหนดวิธีตรวจสอบว่า PostgreSQL พร้อมรับ connection จริงหรือยัง (ด้วยคำสั่ง
  `pg_isready` ที่เป็นเครื่องมือมาตรฐานของ PostgreSQL) — สำคัญมากเพราะ container ของฐานข้อมูล
  อาจ "เริ่มทำงาน" (started) แล้วแต่ยังไม่ "พร้อมรับ connection จริง" (ready) ทันที ต้องรอสักครู่
- `depends_on: db: condition: service_healthy`: บอกให้ `order-service` **รอจนกว่า** `db` จะผ่าน
  healthcheck ก่อน จึงจะเริ่มทำงาน ป้องกันปัญหา `order-service` พยายามเชื่อมต่อฐานข้อมูลที่ยังไม่
  พร้อมใช้งาน
- `volumes: pgdata:` (ที่ระดับบนสุด): ประกาศชื่อ volume `pgdata` ให้ Docker จัดการเก็บไว้ในระบบ
  จริง (แยกต่างหากจาก filesystem ของ container ใด ๆ)

### เชื่อมต่อระหว่าง Service ด้วยชื่อ Service เป็น Hostname

จุดสำคัญมากที่ต้องเข้าใจ: ภายใน network ที่ Docker Compose สร้างให้อัตโนมัติ **ชื่อ service
(`db`) ใช้เป็น hostname ในการเชื่อมต่อระหว่าง container ได้โดยตรง** — ถ้า Shim Program ใน
`order-service` (ตามที่เรียนใน Part 075) ต้องเชื่อมต่อ PostgreSQL จะต้องแก้ connection string
จาก `host=127.0.0.1` (ที่ใช้ตอนทดสอบนอก container) เป็น `host=db` (ชื่อ service ใน
docker-compose.yml นี้) แทน เพราะ `127.0.0.1` ภายใน container ของ `order-service` หมายถึง
container นั้นเอง ไม่ใช่ container ของฐานข้อมูล

### ข้อควรระวัง

- **นี่คือจุดที่ผู้เริ่มต้นใช้ Docker Compose มักสับสนที่สุด**: ค่า `host` ที่ใช้เชื่อมต่อฐานข้อมูล
  ต้อง**เปลี่ยน**จาก `127.0.0.1`/`localhost` (ที่ใช้ตอนพัฒนานอก container ตามที่ทดสอบใน Part 075)
  เป็นชื่อ service (`db`) เมื่อย้ายมารันผ่าน Docker Compose เสมอ
- Environment variable ที่มีรหัสผ่าน (`POSTGRES_PASSWORD`, `DB_PASSWORD`) เขียนตรง ๆ ใน
  `docker-compose.yml` แบบนี้**เหมาะสำหรับการสาธิตหรือพัฒนาเท่านั้น** ระบบ production ควรใช้
  Docker Secrets หรือไฟล์ `.env` ที่แยกจาก version control (ทบทวนหลักการเดียวกันจาก Part 075
  ขั้นตอนที่ 747)

### แบบฝึกหัดที่ 757.1

**โจทย์**: จงอธิบายว่าทำไม `depends_on` แบบธรรมดา (ไม่มี `condition: service_healthy`) อาจไม่
เพียงพอสำหรับสถานการณ์นี้ และทำไมต้องใช้ `healthcheck` ร่วมด้วย

**เฉลย**: `depends_on` แบบธรรมดารับประกันแค่ **ลำดับการเริ่ม container** (เริ่ม `db` ก่อน แล้วค่อย
เริ่ม `order-service`) แต่ไม่รับประกันว่า PostgreSQL **พร้อมรับ connection จริงแล้ว** ภายใน
container ของฐานข้อมูล กระบวนการเริ่มต้นของ PostgreSQL (initialize ฐานข้อมูล, โหลดข้อมูล) ใช้เวลา
สักครู่หลังจาก container เริ่มทำงาน หาก `order-service` พยายามเชื่อมต่อทันทีที่ container ของ `db`
เริ่มทำงาน (แต่ PostgreSQL ข้างในยังไม่พร้อม) จะเกิด connection error ทันที การเพิ่ม `healthcheck`
พร้อม `condition: service_healthy` ทำให้ Docker Compose รอจนกว่า `pg_isready` จะยืนยันว่า
PostgreSQL พร้อมรับ connection จริง ๆ ก่อนจึงเริ่ม `order-service` แก้ปัญหา race condition นี้ได้
อย่างน่าเชื่อถือกว่า

---

## ขั้นตอนที่ 758: Security Best Practices สำหรับ Docker Image ที่มี COBOL

### หลักการความปลอดภัยที่สำคัญ 5 ข้อ

1. **ห้ามรันเป็น root**: Dockerfile ในขั้นตอนที่ 754 สร้างผู้ใช้ `cobolapp` และใช้คำสั่ง `USER
   cobolapp` ก่อนจะ `CMD` — หากไม่ทำเช่นนี้ container จะรันด้วยสิทธิ์ `root` โดยค่าเริ่มต้น ซึ่ง
   หากมีช่องโหว่ในโปรแกรมที่ทำให้ผู้โจมตีเข้าควบคุม process ได้ ผู้โจมตีจะได้สิทธิ์ `root` ภายใน
   container ทันที (และในบางกรณีอาจ escape ออกไปควบคุมเครื่อง host ได้หากมีช่องโหว่ของ container
   runtime เองร่วมด้วย)
2. **ใช้ Base Image ที่เล็กและอัปเดตสม่ำเสมอ**: Multi-Stage Build ที่เรียนในขั้นตอนที่ 753-754
   ช่วยเรื่องนี้โดยธรรมชาติอยู่แล้ว เพราะ Runtime Stage ไม่มี compiler หรือ build tools ที่อาจมี
   ช่องโหว่ปะปนอยู่
3. **Pin เวอร์ชันของ Base Image และ Package ให้ชัดเจน**: การใช้ `ubuntu:24.04` (ระบุเวอร์ชัน
   ชัดเจน) ดีกว่าการใช้ `ubuntu:latest` เพราะ `latest` อาจเปลี่ยนแปลงไปเรื่อย ๆ ทำให้ build ใน
   อนาคตอาจได้ package เวอร์ชันต่างจากที่ทดสอบไว้โดยไม่รู้ตัว
4. **ไม่เก็บ secret (รหัสผ่าน, API key) ไว้ใน Dockerfile หรือ Image**: ทุกสิ่งที่เขียนไว้ใน
   Dockerfile หรือ COPY เข้าไปใน image จะถูกเก็บถาวรในทุก layer ของ image นั้น แม้จะลบทิ้งใน
   คำสั่งถัดไปก็ยังคงกู้คืนออกมาดูได้จาก layer ก่อนหน้า ควรใช้ environment variable ที่ส่งเข้ามา
   ตอน `docker run`/Docker Compose หรือ Docker Secrets แทนเสมอ
5. **สแกนหาช่องโหว่ (Vulnerability Scanning) ก่อน deploy จริง**: เครื่องมืออย่าง `docker scout`
   หรือ Trivy สามารถสแกน image ที่สร้างเสร็จแล้วเพื่อตรวจหา package ที่มีช่องโหว่ความปลอดภัยที่รู้จัก
   (CVE) ก่อนนำไป deploy จริง

### ตัวอย่าง .dockerignore เพื่อป้องกันไฟล์ที่ไม่ควรอยู่ใน Image

```
.git
*.md
*.dat
*.idx
__pycache__/
*.pyc
```

ไฟล์ `.dockerignore` ทำงานคล้าย `.gitignore` — ป้องกันไม่ให้ไฟล์ที่ไม่จำเป็น (หรือมีความละเอียดอ่อน
เช่นไฟล์ข้อมูล `.dat`/`.idx` ที่อาจมีข้อมูลลูกค้าจริงจากการทดสอบ) ถูก `COPY` เข้าไปใน build context
โดยไม่ตั้งใจ

### ข้อควรระวัง

- Multi-Stage Build ช่วยเรื่องความปลอดภัยได้มาก แต่ **ไม่ได้แก้ปัญหาความปลอดภัยทั้งหมด** — ช่องโหว่
  ในตัวโค้ด COBOL/Python เอง (เช่น SQL Injection ที่เรียนใน Part 075 ขั้นตอนที่ 747) ยังคงต้อง
  ได้รับการแก้ไขที่ระดับโค้ด ไม่ใช่ที่ระดับ container
- การรันเป็นผู้ใช้ที่ไม่ใช่ `root` อาจทำให้เกิดปัญหาสิทธิ์การเข้าถึงไฟล์ (permission denied) หาก
  ลืม `chown` ไฟล์ที่ container ต้องเขียนได้ (เช่น ไฟล์ log หรือไฟล์ผลลัพธ์ชั่วคราวที่ Part 075
  เขียนด้วย `CALL "SYSTEM"`) ให้เป็นของผู้ใช้นั้นก่อน

### แบบฝึกหัดที่ 758.1

**โจทย์**: จงอธิบายว่าทำไมการเขียน `ENV DB_PASSWORD=cobol123` ไว้ตรง ๆ ใน Dockerfile จึงไม่
ปลอดภัย แม้จะลบบรรทัดนั้นออกในคำสั่งถัดไปด้วย `RUN unset DB_PASSWORD` ก็ตาม

**เฉลย**: Docker image ถูกสร้างขึ้นเป็นชั้น ๆ (layers) โดยแต่ละคำสั่งใน Dockerfile (`RUN`,
`COPY`, `ENV` ฯลฯ) สร้าง layer ใหม่ 1 ชั้นเสมอ และ**ทุก layer ที่เคยสร้างขึ้นมาแล้วจะยังคงถูกเก็บ
ไว้ถาวรภายใน image** แม้คำสั่งถัดไปจะพยายาม "ลบ" หรือ "unset" ค่านั้นก็ตาม เพราะการ unset เป็นแค่
การสร้าง layer ใหม่ที่ซ่อนค่านั้นจาก layer บนสุด แต่ layer ก่อนหน้าที่มีค่า `DB_PASSWORD=cobol123`
ยังคงอยู่และสามารถถูกดึงออกมาดูได้ด้วยเครื่องมือตรวจสอบ image layer (เช่น `docker history` หรือ
การ export image แล้วแกะดู layer แต่ละชั้น) ดังนั้นวิธีที่ถูกต้องคือไม่เขียน secret ลงใน
Dockerfile เลยตั้งแต่แรก ให้ส่งผ่าน environment variable ตอน `docker run`/Compose หรือใช้ Docker
Secrets/Build Secrets ที่ออกแบบมาเฉพาะเพื่อไม่ให้ค่านั้นถูกเก็บถาวรใน image layer แทน

---

## ขั้นตอนที่ 759: CI/CD และการเตรียมความพร้อมสำหรับ Production (ภาพรวม)

### เชื่อมโยงกับ Part 078 ที่กำลังจะมาถึง

Part 076 นี้สอนวิธี**สร้าง** Docker image สำหรับ COBOL แต่ยังไม่ได้ลงรายละเอียดเรื่องการนำ image
นี้เข้าสู่กระบวนการ CI/CD Pipeline (Continuous Integration/Continuous Deployment) อัตโนมัติ
เนื้อหานั้นจะสอนเต็มรูปแบบใน **Part 078** — Part นี้ให้แค่ภาพรวมสั้น ๆ เพื่อให้เห็นบริบทว่า
Dockerfile ที่เขียนวันนี้จะถูกนำไปใช้อย่างไรต่อในสายงานจริง

### ภาพรวมขั้นตอนที่ Dockerfile นี้มักถูกใช้ใน CI/CD Pipeline

```
Developer push โค้ด COBOL/Python เข้า Git repository
        |
        v
CI Server (เช่น GitHub Actions, GitLab CI, Jenkins) ตรวจจับการเปลี่ยนแปลง
        |
        v
รัน docker build อัตโนมัติ (Multi-Stage Build ที่เขียนใน Part นี้)
        |
        v
รัน automated test กับ image ที่สร้างเสร็จ (เนื้อหา Unit Testing สอนใน Part 079)
        |
        v
หากผ่านทุกการทดสอบ: push image ไปยัง Container Registry (เช่น Docker Hub, AWS ECR)
        |
        v
Deploy image เวอร์ชันใหม่ไปยัง Production (Kubernetes, Cloud Service เป็นต้น)
```

### เหตุผลที่ Multi-Stage Build สำคัญมากในบริบท CI/CD

CI Server มักมีข้อจำกัดด้านเวลาและทรัพยากร (มักคิดค่าใช้จ่ายตามเวลาที่ใช้ build) Multi-Stage
Build ที่ทำให้ image สุดท้ายมีขนาดเล็กกว่าช่วยลดเวลาในการ push/pull image ไปมาระหว่างขั้นตอนต่าง ๆ
ของ pipeline ได้อย่างมีนัยสำคัญ โดยเฉพาะเมื่อ pipeline รันซ้ำหลายสิบหรือหลายร้อยครั้งต่อวันในทีมที่
มีขนาดใหญ่

### ข้อควรระวัง

- Part นี้เป็นเพียงภาพรวมเพื่อเชื่อมโยงบริบทเท่านั้น รายละเอียดเชิงลึกเรื่องการตั้งค่า CI/CD Pipeline
  จริง (YAML ของ GitHub Actions, การจัดการ secret ใน pipeline, การทำ automated rollback) จะสอน
  แบบเต็มรูปแบบใน Part 078 — อย่าเพิ่งพยายามตั้งค่า pipeline เต็มรูปแบบจากเนื้อหาสั้น ๆ ในขั้นตอนนี้

### แบบฝึกหัดที่ 759.1

**โจทย์**: จงอธิบายว่าทำไม "การทดสอบ image ที่สร้างเสร็จแล้ว" (ในขั้นตอน CI/CD) จึงสำคัญกว่าแค่
"การทดสอบโค้ด COBOL แยกต่างหากก่อน containerize"

**เฉลยแนวทาง**: การทดสอบโค้ด COBOL แยกต่างหาก (เช่น รันโดยตรงบนเครื่อง CI Server) พิสูจน์ได้แค่ว่า
**logic ของโปรแกรมถูกต้อง** แต่ไม่ได้พิสูจน์ว่า**การ containerize สำเร็จและใช้งานได้จริง** ตัวอย่าง
เช่น ปัญหาแบบที่พบใน Part 074 ขั้นตอนที่ 738 (`LD_LIBRARY_PATH` ไม่ถูกต้อง) จะไม่ถูกตรวจพบเลยหาก
ทดสอบแค่โค้ดนอก container เพราะปัญหานั้นเกิดจากสภาพแวดล้อมของ container specifically การทดสอบ
image ที่สร้างเสร็จแล้วจริง (integration test ที่รันกับ container จริง) จึงช่วยจับปัญหาประเภทนี้ได้
ก่อนที่จะไปถึงขั้นตอน deploy จริงใน production

---

## ขั้นตอนที่ 760: สรุปภาพรวมและทางเลือกเมื่อไม่มี Docker Daemon

### สรุปสิ่งที่เรียนรู้ในเชิงสถาปัตยกรรม (ยืนยันความถูกต้องของไวยากรณ์)

- **Multi-Stage Build**: แยก Build Stage (มี `gnucobol4` เต็มชุด) ออกจาก Runtime Stage (มีแค่
  `libcob5t64`) ทำให้ image สุดท้ายเล็กและปลอดภัยกว่า
- **docker-compose.yml**: จัดการหลาย service (COBOL wrapper + PostgreSQL) พร้อมกันด้วยไฟล์
  configuration เดียว พร้อม healthcheck และ dependency management
- **Security Best Practices**: รันด้วยผู้ใช้ที่ไม่ใช่ root, ไม่เก็บ secret ใน image, ใช้
  `.dockerignore` และ pin เวอร์ชันของทุกอย่างให้ชัดเจน

### ทางเลือกเมื่อไม่มี Docker Daemon ทำงาน (สำหรับผู้เรียนที่อยู่ในสถานการณ์เดียวกับหลักสูตรนี้)

หากคุณอยู่ในสภาพแวดล้อมที่ไม่มี Docker daemon ทำงาน (เช่น Sandbox หรือ CI environment บางประเภท)
แต่ยังต้องการทดสอบแนวคิด Container ต่อไปนี้คือทางเลือกที่ควรรู้จักไว้:

- **Rootless container runtime อื่น** เช่น Podman หรือ Buildah ที่บางครั้งทำงานได้ในสภาพแวดล้อม
  ที่จำกัดกว่า Docker แบบดั้งเดิม (แม้ในสภาพแวดล้อมของหลักสูตรนี้จะตรวจสอบแล้วว่าไม่มีเครื่องมือ
  เหล่านี้ติดตั้งไว้เช่นกัน)
- **Cloud-based Build Service**: บริการอย่าง GitHub Actions, GitLab CI, หรือ Cloud Build ของผู้
  ให้บริการ Cloud มักมี Docker daemon พร้อมใช้งานเต็มรูปแบบในสภาพแวดล้อมของตัวเอง สามารถ push
  Dockerfile ที่เขียนไว้ไปให้ระบบเหล่านี้ build แทนได้
- **เครื่องพัฒนาของตัวเอง**: ติดตั้ง Docker Desktop (Windows/Mac) หรือ Docker Engine (Linux) บน
  เครื่องส่วนตัว แล้วทดสอบ Dockerfile ที่เรียนใน Part นี้ได้โดยตรง

### แบบฝึกหัดที่ 760.1 (แบบฝึกหัดรวม Part นี้)

**โจทย์**: หากคุณมีเครื่องที่ติดตั้ง Docker Desktop และทำงานได้จริง จงทำตามขั้นตอนต่อไปนี้เพื่อ
พิสูจน์ด้วยตัวเองว่า Dockerfile ที่เรียนใน Part นี้ถูกต้อง: (1) สร้างโฟลเดอร์โปรเจกต์พร้อมไฟล์
`ordercalc.cob`, `wrapper.py` (แก้ bind address เป็น `0.0.0.0` แล้ว), และ `Dockerfile` ตามที่สอน
ในขั้นตอนที่ 754 (2) รัน `docker build -t ordercalc-service .` (3) รัน
`docker run -d -p 8073:8073 ordercalc-service` (4) ทดสอบด้วย `curl` เหมือนที่ทำใน Part 073
(5) เปรียบเทียบผลลัพธ์ที่ได้กับผลลัพธ์ที่ Part 073 ทดสอบไว้นอก container

**เฉลยแนวทาง**: หากทำตามขั้นตอนถูกต้องทั้งหมด ผลลัพธ์จาก `curl` ควรตรงกับที่ Part 073 ขั้นตอนที่
727 ทดสอบไว้ทุกประการ (เช่น `{"price": 100.0, "quantity": 3, "subtotal": 300.0, "tax": 21.0,
"total": 321.0}`) เนื่องจาก logic ของโปรแกรมไม่เปลี่ยนแปลง มีเพียงสภาพแวดล้อมที่ห่อหุ้มเปลี่ยนไป
เท่านั้น หากผลลัพธ์ไม่ตรงกันหรือเกิด error ให้ตรวจสอบตามลำดับ: (ก) `wrapper.py` bind ที่
`0.0.0.0` แล้วหรือยัง (ข) `docker logs <container>` แสดง error อะไรบ้าง (ค) package
`gnucobol4`/`libcob5t64` ติดตั้งสำเร็จใน image หรือไม่ (ตรวจสอบด้วยการรัน
`docker exec -it <container> bash` เข้าไปดูภายใน container โดยตรง)

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้วิธี Containerize ระบบ COBOL สมัยใหม่ด้วย Docker:

- แนวคิดพื้นฐานของ Container และทำไมสำคัญกับการทำ COBOL Modernization โดยเฉพาะ (แก้ปัญหา "works
  on my machine" ที่พบจริงใน Part 074 เรื่อง `LD_LIBRARY_PATH`)
- **ตรวจสอบสภาพแวดล้อมอย่างซื่อตรง**: Docker CLI มีอยู่แต่ daemon ไม่ทำงาน (ยืนยันด้วย error
  message จริง) ทำให้เนื้อหาส่วนที่เหลือระบุชัดเจนว่าเป็นไวยากรณ์มาตรฐานที่ยังไม่ได้พิสูจน์ด้วยการ
  รันจริงในสภาพแวดล้อมนี้ แต่ตรวจสอบความถูกต้องของชื่อ package (`gnucobol4`, `libcob5t64`) กับ
  ระบบปฏิบัติการจริงแล้ว
- หลักการ **Multi-Stage Build**: แยก Build Stage (compiler ครบชุด) ออกจาก Runtime Stage (แค่
  runtime library) เพื่อ image ที่เล็กและปลอดภัยกว่า
- เขียน Dockerfile ฉบับสมบูรณ์สำหรับระบบ ORDERCALC จาก Part 073 พร้อมอธิบายทุกบรรทัด
- เขียน docker-compose.yml ทั้งแบบ service เดียวและแบบขยายรวม PostgreSQL จาก Part 075 พร้อม
  healthcheck และ dependency management
- **Security Best Practices**: ไม่รันเป็น root, ไม่เก็บ secret ใน image, ใช้ `.dockerignore`,
  pin เวอร์ชัน และสแกนช่องโหว่
- ภาพรวมความเชื่อมโยงกับ CI/CD Pipeline ที่จะสอนเต็มรูปแบบใน Part 078
- ทางเลือกสำหรับผู้เรียนที่อยู่ในสภาพแวดล้อมที่ไม่มี Docker daemon ทำงานเช่นเดียวกับหลักสูตรนี้

ใน **Part 077** เราจะยกระดับมุมมองขึ้นไปอีกขั้น จากการ Containerize โปรแกรมเดี่ยว ๆ ไปสู่แนวคิด
**การปรับปรุงระบบ Mainframe ทั้งองค์กรให้ทำงานบน Cloud** ผ่านแนวคิด AWS Mainframe Modernization,
Micro Focus Enterprise Server, และรูปแบบการโยกย้ายระบบแบบค่อยเป็นค่อยไปที่เรียกว่า
Strangler Fig Pattern

**[← กลับไป Part 075: ฐานข้อมูลสมัยใหม่: COBOL กับ MySQL/PostgreSQL](part-075-cobol-modern-databases.md)** | **[ไปยัง Part 077: COBOL บน Cloud: แนวคิด Mainframe Modernization →](part-077-cobol-cloud-modernization.md)**
