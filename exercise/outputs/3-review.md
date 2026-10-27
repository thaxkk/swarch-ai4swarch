Agent: Codex (GPT-6)

## Quality-attribute scenarios

| Source · Stimulus · Environment · Response · Response measure |
|---|
| ลูกค้า · สั่งซื้อ · ช่วง 11.11 ที่ 200 orders/s ขณะที่ MailFast ตอบช้า · `PlaceOrder` รับคำสั่งซื้อและบันทึก outbox · p95 ≤ 500 ms เทียบกับปัจจุบัน ~300 ms (`docs/scenario.md`, Quality goals; ADR, Decision) |
| MailFast · หยุดให้บริการ · หลังรับคำสั่งซื้อแล้ว · ระบบเก็บงานอีเมลไว้และไม่ทำให้คำสั่งซื้อล้มเหลว · อัตราคำสั่งซื้อที่ล้มเหลวเพราะ MailFast = 0 (`docs/scenario.md`, Availability; ADR, RabbitMQ Architecture) |
| `loyaltyservice` · ได้รับ `OrderPlaced` ซ้ำหรือผู้ใช้บริการหยุดทำงาน · ภายใต้ 200 orders/s · ให้คะแนนหนึ่งครั้งต่อ `orderID` · ไม่ให้คะแนนซ้ำ และเสร็จภายใน 60 วินาที (`docs/scenario.md`, Loyalty points; ADR, Consumer Specifications) |

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | outbox ที่เขียนใน transaction เดียวกับคำสั่งซื้อช่วยป้องกันเหตุการณ์สูญหายหลังตอบรับคำสั่งซื้อ (ADR, Decision) | สนับสนุนความถูกต้องของข้อมูล หาก implementation และการทดสอบยืนยันความเป็น atomic |
| Risk | การเพิ่ม datastore ในเส้นทาง `PlaceOrder` ยังไม่มีงบเวลา หรือผลวัด latency (ADR, Decision; Validation) | ข้อสรุปว่า p95 จะ “well below 500 ms” ยังไม่มีหลักฐาน |
| Sensitivity point | MailFast รับได้ 50 emails/s เทียบกับยอดสั่งซื้อ 200 orders/s (ADR, Context; `docs/scenario.md`, MailFast) | backlog เพิ่มอย่างน้อย 150 ฉบับ/s; หลัง peak หนึ่งชั่วโมงจะค้างอย่างน้อย 540,000 ฉบับก่อนนับงานเดิม |
| Trade-off point | RabbitMQ กับ outbox แยกงานภายนอกออกจาก checkout แต่เพิ่ม datastore, relay และ broker (ADR, Trade-off analysis; Consequences) | เพิ่ม availability และ modifiability แต่เพิ่มต้นทุนปฏิบัติการและจุดที่ต้องเฝ้าระวัง |

## Missing or weak

- ADR อ้างว่า `checkoutservice` ไม่มี durable order state โดยอ้างไฟล์โค้ดที่อยู่นอกหลักฐานที่อนุญาตในงานนี้; **Not evidenced in the provided files**. ต้องยืนยันก่อนเลือก outbox และฐานข้อมูลใหม่ (ADR, Context).
- ยังไม่มีเกณฑ์ความจุและเวลาเคลียร์ backlog ของอีเมล รวมถึงข้อตกลงเรื่องเวลาส่งอีเมล; rate limiter รักษาโควตาได้ แต่ไม่รับประกันเวลาส่ง (ADR, Open questions).
- ยังไม่กำหนดกำลังประมวลผลและเวลาฟื้นตัวของ `order.loyalty` ที่พิสูจน์เกณฑ์ 60 วินาทีได้; การตั้ง alert ที่ 30 วินาทียังไม่ใช่มาตรการรักษา SLA (ADR, Risks & Mitigations).
- คำว่า “fan-out messaging” ไม่ตรงกับ **Topic Exchange** ที่ระบุ และยังไม่ชัดว่าใช้ routing key ใด (ADR, Decision).
- ข้อเสียของตัวเลือก synchronous ระบุว่า MailFast failure ทำให้ checkout ล้มเหลว แต่กรณี email ปัจจุบันเพียงบันทึก warning แล้วคำสั่งซื้อสำเร็จ (`docs/scenario.md`, How it works today). ควรแยกผลของ latency กับผลของ failure.

## Verdict

**Accept with changes.** (1) วัด p95 ของ checkout หลังเพิ่มการเขียนฐานข้อมูลและ outbox ที่ 200 orders/s; (2) ระบุแผนความจุและเวลาเคลียร์ backlog ของ MailFast พร้อมเกณฑ์เวลาส่งอีเมล; (3) พิสูจน์การให้คะแนนครั้งเดียวภายใน 60 วินาทีภายใต้การส่งซ้ำและการหยุดทำงานของ consumer.
