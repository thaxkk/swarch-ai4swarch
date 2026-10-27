**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่มี เพราะ run a ไม่ได้ context เพิ่ม | มี 500 ms, 300 ms, 200 orders/s, MailFast 1.5/8 s, 0.5% และ 50 emails/s (`outputs/2a-adr.md`; `outputs/2b-adr.md`) |
| ทางเลือกที่พิจารณา (ระบุ) | synchronous calls, message broker, job queue/workflow | synchronous gRPC, RabbitMQ event, database polling workers |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | พูดถึง idempotency แต่ไม่ออกแบบ transaction; ไม่รู้ rate limit | เสนอ `orderID` สำหรับ deduplication และ throttle ที่ 50 emails/s แต่ยังไม่แก้ช่องว่าง capacity ระหว่าง 200 กับ 50 และยังไม่กำหนดที่เก็บแต้มแบบ atomic |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่พบตัวเลขที่แต่งขึ้น; broker และ workflow เป็นทางเลือกเสนอ | RabbitMQ เป็นข้อเสนอที่สอดคล้องกับทักษะทีม ไม่ใช่ระบบเดิม; ร่างระบุผู้ตัดสินใจเองโดยไม่มีหลักฐานว่าได้รับมอบหมาย |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่มการคำนวณ backlog ที่ช่วงพีก: 200−50 = 150 emails/s | คิวไม่ได้เพิ่ม quota ของ MailFast; ต้องตกลงเวลารออีเมลหรือเพิ่ม quota (`docs/scenario.md`) |
| 2 | กำหนด durable order/outbox เป็นเงื่อนไขก่อนยอมรับ order | โค้ดปัจจุบันสร้าง `OrderResult` แล้วตอบโดยไม่มี order write ที่ทนทาน จึงเสี่ยง event หาย (`src/checkoutservice/main.go`, `PlaceOrder`) |
| 3 | กำหนด ledger ที่บันทึก `orderID` และแต้มใน transaction เดียว พร้อมทดสอบ crash window | `orderID` อย่างเดียวไม่ป้องกันการมอบแต้มซ้ำหลัง consumer ล้มก่อน ack (`docs/adr/0002-publish-order-placed-events.md`, Decision/Validation) |
