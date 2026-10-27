Agent: GitHub Copilot; Model: ไม่ทราบ

# ADR-0002: ใช้ RabbitMQ สำหรับงานหลังวางคำสั่งซื้อแบบ asynchronous

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ทีมพัฒนา Online Boutique

## Context and problem

ตาม requirement R-12 หลังวางคำสั่งซื้อสำเร็จ ระบบต้องส่งอีเมลยืนยันผ่าน MailFast, ให้ `loyaltyservice` มอบ loyalty points และบันทึก analytics event ชนิด `order placed` โดยคำสั่งซื้อต้องไม่ล้มเหลวเพราะงานใดงานหนึ่งเหล่านี้หยุดทำงาน (`docs/scenario.md`, R-12)

ปัจจุบัน `checkoutservice.PlaceOrder` เรียก `paymentservice` และ `shippingservice` ก่อนเรียก `emailservice` ผ่าน gRPC ในเส้นทางเดียวกัน แม้การส่งอีเมลล้มเหลว checkout จะเพียง log warning และยังส่งคำตอบสำเร็จ (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`). การเพิ่ม MailFast, loyalty และ analytics แบบ synchronous จะทำให้เวลาและความล้มเหลวของบริการหลังคำสั่งซื้อกระทบ checkout โดยตรง

เป้าหมายคือให้ p95 ของ `PlaceOrder` ไม่เกิน 500 ms จากปัจจุบันประมาณ 300 ms ที่ peak 200 orders/s (`docs/scenario.md`). MailFast มี p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5% และจำกัด 50 emails/s. Loyalty points ต้องมอบภายใน 1 นาทีและ exactly once. ปีหน้าคาดว่าจะมี post-order actions เพิ่ม เช่น SMS, fraud scoring และการอัปเดต recommendations (`docs/scenario.md`). ทีมมีนักพัฒนา 4 คน, ถนัด Go และ gRPC และมี 2 คนเคยใช้ RabbitMQ; ระบบรันบน Kubernetes และต้องการ component แบบ open-source ที่ self-hosted (`docs/scenario.md`).

## Decision drivers

1. **Availability:** งานอีเมล loyalty และ analytics ต้องไม่ทำให้คำสั่งซื้อล้มเหลว
2. **Performance:** เส้นทาง `PlaceOrder` ต้องรักษา p95 ไม่เกิน 500 ms
3. **Correctness:** loyalty points ต้องถูกมอบภายใน 1 นาทีและไม่เกิดการมอบซ้ำ
4. **Scalability:** รองรับ 200 orders/s และจำกัดการส่ง MailFast ที่ 50 emails/s
5. **Modifiability:** เพิ่ม post-order actions ใหม่ได้โดยไม่แก้เส้นทาง checkout มาก
6. **Operational fit:** ใช้เทคโนโลยี open-source ที่รันบน Kubernetes และเหมาะกับทักษะทีม

## Considered options

1. เรียก MailFast, `loyaltyservice` และ analytics แบบ synchronous ผ่าน gRPC จาก `checkoutservice`
2. เผยแพร่ `OrderPlaced` event ไปยัง RabbitMQ แล้วให้ consumer แยกกันประมวลผล
3. ให้ `checkoutservice` เขียนงานลงฐานข้อมูลกลาง แล้วให้ worker polling งานแต่ละประเภท

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous gRPC fan-out | รูปแบบใกล้กับโค้ดปัจจุบัน (`main.go`, `PlaceOrder`); flow ตรงไปตรงมา; ใช้ gRPC ที่ทีมถนัด | MailFast อาจใช้เวลาถึง p99 8 วินาทีและทำให้ checkout ช้าหรือใช้ทรัพยากรค้าง; ความล้มเหลวและ dependency ของ post-order actions ผูกกับ checkout; การเพิ่ม action ทำให้ checkout ซับซ้อนขึ้น |
| RabbitMQ `OrderPlaced` event | แยก post-order actions ออกจาก checkout; broker รองรับการค้างงานเมื่อ consumer หรือ MailFast หยุด; แยก consumer เพื่อควบคุม MailFast ที่ 50 emails/s; เพิ่ม action ใหม่ได้โดย subscribe event เดิม | ต้องดูแล RabbitMQ และการส่งซ้ำ; ต้องมี idempotency สำหรับ loyalty และ retry/dead-letter policy; มีความซับซ้อนในการติดตาม event และการรับประกันว่าการ publish เกิดหลังคำสั่งซื้อสำเร็จ |
| ฐานข้อมูลกลางกับ polling workers | งานคงอยู่ใน storage และ retry ได้; worker แยกจาก checkout; ไม่ต้องเพิ่ม message broker | ต้องกำหนด schema, locking และ polling เอง; polling เพิ่ม latency และภาระฐานข้อมูลที่ peak 200 orders/s; การเพิ่ม action และ routing มีแนวโน้มซับซ้อน; ยังต้องมี idempotency และการดูแล state เพิ่มเติม |

## Decision

เลือก **RabbitMQ `OrderPlaced` event** เป็นกลไก asynchronous สำหรับ post-order actions

เมื่อคำสั่งซื้อผ่านขั้นตอน payment และ shipping แล้ว `checkoutservice` ควรทำให้คำสั่งซื้อสำเร็จโดยไม่รอ MailFast, `loyaltyservice` หรือ analytics จากนั้นเผยแพร่ event ที่มี `orderID` และข้อมูลซึ่ง consumer จำเป็นต้องใช้ Consumer ของ MailFast จะ retry งานที่ล้มเหลวและควบคุมอัตราส่งไม่เกิน 50 emails/s. Consumer ของ `loyaltyservice` จะใช้ `orderID` เป็น idempotency key และตรวจสอบ/บันทึกผลก่อนยืนยัน message เพื่อไม่ให้ retry มอบแต้มซ้ำ ส่วน analytics จะมี consumer แยกออกมา

RabbitMQ เหมาะที่สุดเพราะลด dependency synchronous ของ checkout, รองรับการประมวลผลภายหลังเมื่อ external provider ช้าหรือหยุดทำงาน และเปิดทางให้เพิ่ม SMS, fraud scoring หรือ recommendations โดยไม่เพิ่ม synchronous call ใน `PlaceOrder` การเลือกนี้สอดคล้องกับประสบการณ์ RabbitMQ ของสมาชิกทีมและข้อกำหนด self-hosted บน Kubernetes (`docs/scenario.md`).

## Consequences

- **Positive:** checkout ไม่ต้องรอ MailFast ที่ p95 1.5 วินาทีหรือ p99 8 วินาที จึงช่วยรักษา latency เป้าหมาย (performance); post-order outage ไม่ทำให้ order fail (availability).
- **Positive:** consumer แยกกัน scale และ retry ได้ รวมถึง throttle MailFast ที่ 50 emails/s ขณะที่รับ order ได้ถึง 200 orders/s (scalability).
- **Positive:** เพิ่ม consumer สำหรับ action ใหม่ได้โดยไม่เปลี่ยน flow หลักของ checkout (modifiability).
- **Positive:** ใช้ RabbitMQ แบบ self-hosted บน Kubernetes และใช้ทักษะที่ทีมมีอยู่ (operability, cost).
- **Negative:** ต้องเพิ่มและดูแล RabbitMQ รวมถึง monitoring, retry และ dead-letter handling (operational complexity, cost).
- **Negative:** ระบบเป็น eventually consistent; อีเมลและ analytics อาจเกิดภายหลัง แม้ loyalty ต้องเสร็จภายใน 1 นาที (consistency, user experience).
- **Risks and mitigations:** message delivery อาจซ้ำ จึงต้องทำ consumer ให้ idempotent โดยเฉพาะ loyalty และใช้ `orderID` เป็น key; MailFast ล้มเหลวหรือ rate limit จึงใช้ retry กับ backoff และ consumer ที่ throttle; ความล้มเหลวระหว่างบันทึก order กับ publish event ต้องมี durable publication mechanism ที่กำหนดในขั้นออกแบบรายละเอียด

## Validation

ทำ load test ที่ 200 orders/s และตรวจว่า p95 ของ `PlaceOrder` ไม่เกิน 500 ms แม้ MailFast จะช้าหรือหยุดทำงาน ตรวจ metric ของ RabbitMQ backlog และยืนยันว่า loyalty event ถูกประมวลผลภายใน 1 นาทีโดยไม่มอบแต้มซ้ำเมื่อส่ง message ซ้ำ ทดสอบ MailFast consumer ให้ไม่เกิน 50 emails/s และทดสอบ retry/dead-letter ของทั้งสาม consumer รวมถึงทดสอบการเพิ่ม consumer ใหม่โดยไม่แก้ flow ของ checkout

## Open questions

- ยังไม่ได้ระบุ durable storage หรือ schema สำหรับ idempotency record ของ `loyaltyservice`; ต้องเลือกและยืนยันก่อน implementation
- ยังไม่ได้ระบุรูปแบบการทำให้การบันทึก order และการ publish `OrderPlaced` เป็น atomic; ต้องกำหนดกลไกที่ป้องกัน event สูญหาย
- ยังไม่ได้ระบุ retention, replication และ disaster-recovery settings ของ RabbitMQ บน Kubernetes
- ยังไม่ได้ระบุ contract/schema ของ `OrderPlaced` event และข้อมูลที่ MailFast, loyalty และ analytics ต้องใช้
- ยังไม่ได้ระบุเจ้าของ analytics storage หรือ SLA ของ analytics event
