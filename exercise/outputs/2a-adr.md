Agent: GitHub Copilot; model: unknown

# ADR-0002: จัดการ post-order actions แบบ asynchronous

- **Status:** Proposed
- **Date:** 2026-10-23
- **Deciders:** <who>

## Context and problem

หลังจากลูกค้าสั่งซื้อสินค้า Online Boutique ต้องดำเนินการ post-order actions สามอย่างตาม requirement R-12:

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** ซึ่งเป็น external email provider ใหม่และมาแทนที่ logic ของ mock `emailservice`
2. มอบ loyalty points ผ่าน `loyaltyservice` ใหม่ที่ต้องสร้าง
3. บันทึก analytics event ชื่อ "order placed" ให้ทีมการตลาด

ในปีหน้าทีมการตลาดคาดว่าจะมี post-order actions เพิ่มขึ้น เช่น SMS, fraud scoring และการอัปเดต recommendations

ปัญหาที่ต้องตัดสินใจคือควรเชื่อม post-order actions เหล่านี้กับการสั่งซื้ออย่างไร เพื่อให้การดำเนินการหลังสั่งซื้อครบตาม requirement และรองรับการเพิ่ม action ในอนาคต

รายละเอียดเชิงตัวเลขของ latency, peak load, rate limit, availability และข้อจำกัดของทีมยังไม่ได้ให้มาใน requirement นี้

## Decision drivers

- การประมวลผล email, loyalty และ analytics ให้ครบตาม requirement
- รองรับการเพิ่ม post-order actions ในอนาคตโดยแก้ไขระบบเดิมให้น้อย
- รับประกันว่า loyalty points ถูกมอบให้สอดคล้องกับคำสั่งซื้อและไม่เกิดการมอบซ้ำ
- ทำให้ระบบสามารถติดตามและจัดการ action ที่ล้มเหลวภายหลังได้
- ความซับซ้อนในการพัฒนาและดูแลระบบ

## Considered options

1. เรียก `MailFast`, `loyaltyservice` และ analytics แบบ synchronous จากขั้นตอนการสั่งซื้อ
2. ใช้ asynchronous message broker โดย publish event เมื่อสั่งซื้อสำเร็จ แล้วให้ consumer แยกกันประมวลผลแต่ละ post-order action
3. ใช้ asynchronous job queue หรือ workflow component ที่มีตัวจัดการงานกลางสำหรับ post-order actions

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous calls | โครงสร้างตรงไปตรงมา; ติดตามผลของแต่ละ action ได้ทันที | ความล้มเหลวหรือความล่าช้าของ dependency อาจผูกกับขั้นตอนการสั่งซื้อ; การเพิ่ม action ใหม่ทำให้จุดเชื่อมต่อเพิ่มขึ้น |
| Asynchronous message broker | แยก lifecycle ของ order ออกจาก post-order actions; เพิ่ม consumer ใหม่ได้ง่าย; รองรับการ retry และการประมวลผลที่ล้มเหลวแยกจาก order ได้ | ต้องดูแล broker และ consumer; ต้องออกแบบ duplicate handling และการติดตามสถานะ; ผลลัพธ์ของ action ไม่เกิดขึ้นทันที |
| Asynchronous job queue or workflow component | ซ่อนการจัดคิวและ retry ไว้ใน component กลาง; อาจจัดลำดับและติดตาม workflow ได้ชัดเจน | เพิ่ม dependency และรูปแบบการปฏิบัติงานใหม่; การเพิ่ม action อาจยังต้องแก้ workflow กลาง; รายละเอียดผลิตภัณฑ์และความสามารถที่เหมาะสมยังไม่มีข้อมูล |

## Decision

เลือก **asynchronous message broker** โดยให้ระบบ publish event สำหรับคำสั่งซื้อ และให้ consumer แยกกันจัดการการส่งอีเมลผ่าน MailFast, การมอบ loyalty points และการบันทึก analytics event

ตัวเลือกนี้รักษาความเป็นอิสระระหว่างการสั่งซื้อกับ post-order actions และเปิดทางให้เพิ่ม action ใหม่ในอนาคตโดยเพิ่ม consumer แทนการเพิ่ม synchronous dependency ในขั้นตอนสั่งซื้อ การส่ง event และ consumer ต้องออกแบบให้รองรับการส่งซ้ำ เพื่อไม่ให้ loyalty points ถูกมอบซ้ำ

## Consequences

- **Positive:** การประมวลผล email, loyalty และ analytics แยกจากกันได้; ความล้มเหลวของ action หนึ่งไม่จำเป็นต้องหยุด action อื่น (quality attribute: availability)
- **Positive:** เพิ่ม post-order action ใหม่โดยเพิ่ม consumer ได้โดยไม่ผูกทุก action เข้ากับขั้นตอนสั่งซื้อ (quality attribute: modifiability)
- **Positive:** สามารถ retry งานที่ล้มเหลวและติดตามผลแยกจากคำสั่งซื้อได้ (quality attribute: reliability)
- **Negative:** ผู้ใช้หรือระบบอื่นอาจเห็นผลของ post-order action หลังจาก order สำเร็จแล้ว ไม่ใช่ทันที (quality attribute: consistency)
- **Negative:** ต้องมี infrastructure และ operational work สำหรับ broker, consumer, retry และ monitoring (quality attribute: cost and operability)
- **Risks and mitigations:** การส่ง event หรือ message ซ้ำอาจทำให้ loyalty points ถูกมอบซ้ำ; ให้ `loyaltyservice` รองรับ idempotency โดยใช้ order identity หรือ idempotency key และตรวจสอบผลด้วยการทดสอบกรณี retry และ duplicate delivery

## Validation

- ทดสอบพฤติกรรมเมื่อ MailFast, `loyaltyservice` หรือ analytics consumer หยุดทำงาน
- ทดสอบการ retry เมื่อ post-order action ล้มเหลว และตรวจสอบว่า event ที่ส่งซ้ำไม่ทำให้ loyalty points ถูกมอบซ้ำ
- ทดสอบการเพิ่ม consumer สำหรับ post-order action ใหม่โดยไม่แก้ไขลำดับการสั่งซื้อ
- กำหนดและตรวจสอบ metric สำหรับ event ที่ publish สำเร็จ, งานค้าง, retry, failure และ duplicate handling

## Open questions

- ต้องเลือก message broker หรือ queue technology ใด และมี infrastructure ใดที่องค์กรใช้อยู่แล้ว
- คำสั่งซื้อควรสำเร็จหรือหยุดเมื่อ post-order action ใด action หนึ่งล้มเหลวหรือไม่
- ต้องรับประกันการส่ง event อย่างไรเมื่อ order สำเร็จแต่การ publish event ล้มเหลว เช่น ใช้ transactional outbox หรือกลไกอื่น
- loyalty points ต้องถูกมอบภายในเวลาเท่าใด และ `loyaltyservice` จะเก็บสถานะ idempotency ที่ใด
- MailFast มี latency, failure rate, rate limit และวิธี authentication อย่างไร
- analytics event ต้องมี schema, retention และ delivery guarantee ระดับใด
- ต้องกำหนด timeout, retry policy, dead-letter handling และการแจ้งเตือนอย่างไร
- ใครเป็นผู้รับผิดชอบในการตัดสินใจและอนุมัติ ADR นี้