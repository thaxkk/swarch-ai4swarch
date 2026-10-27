Agent: Codex; Model: GPT-6

# ADR-0003: ใช้ gRPC แบบรอผลและสัญญา demo.proto ร่วมกันสำหรับ RPC ภายใน

- **Status:** Reconstructed
- **Date:** 2026-10-27
- **Deciders:** Not evidenced in the provided files

## Context and problem

- [C1] demo.proto นิยาม request/response RPC ของหลายบริการในไฟล์เดียว (system/online-boutique/protos/demo.proto, service CartService และ service CheckoutService).
- [C2] frontend เปิด HTTP routes ให้ผู้ใช้ และสร้าง gRPC clients ไปยังบริการภายในเจ็ดรายการ (system/online-boutique/src/frontend/main.go, main: r.HandleFunc และ mustConnGRPC). คำว่า “ทุกการสื่อสารเป็น gRPC” จึงไม่ตรงกับหลักฐานนี้.
- [C3] checkoutservice.PlaceOrder เรียกบริการตะกร้า สินค้า สกุลเงิน จัดส่ง ชำระเงิน และอีเมล โดยใช้ผลตอบกลับหรือจัดการข้อผิดพลาดก่อนคืนคำตอบ (system/online-boutique/src/checkoutservice/main.go, PlaceOrder และ helper functions).
- [C4] คำกล่าวว่า “ทุกบริการ” ใช้ gRPC และสร้าง client/server จาก demo.proto ฉบับเดียว: Not evidenced in the provided files. หลักฐานที่อนุญาตครอบคลุมเพียงสอง implementation และไม่แสดงขั้นตอนสร้างโค้ด.

## Decision drivers

- [C5] **[Inferred]** ความถูกต้องของลำดับ checkout ต้องใช้ข้อมูลตะกร้า ราคา ค่าจัดส่ง ผลชำระเงิน และเลขติดตามก่อนส่ง PlaceOrderResponse (system/online-boutique/src/checkoutservice/main.go, PlaceOrder).
- [C6] **[Inferred]** ความง่ายในการรักษารูปแบบข้อความร่วมกันอาจเป็นแรงจูงใจ เพราะ CartItem, Money, Address และ OrderResult ถูกอ้างถึงข้าม service definitions (system/online-boutique/protos/demo.proto, symbols ดังกล่าว).

## Considered options

1. [C7] ใช้ gRPC request/response ภายในและรวม service/message definitions ใน demo.proto เดียว: เป็นรูปแบบที่เห็น แต่ Not evidenced in the provided files ว่าผู้ตัดสินใจเคยเปรียบเทียบตัวเลือก.
2. [C8] **[Assumption]** ใช้ HTTP/REST ภายในและแยกสัญญาตามบริการ: เป็นทางเลือกเชิงออกแบบที่ไม่มีบันทึกการพิจารณาในหลักฐาน.
3. [C9] **[Assumption]** ใช้ asynchronous message broker และแยก schema ตามเหตุการณ์: เป็นทางเลือกเชิงออกแบบที่ไม่มีบันทึกการพิจารณาในหลักฐาน.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| [C10] 1 | **[Inferred]** ชนิดข้อมูลร่วมช่วยส่งค่าที่ checkout ต้องใช้ต่อได้ตรงรูปแบบ | **[Inferred]** ต้องรอ dependency ระหว่าง checkout; การเปลี่ยนสัญญารวมอาจกระทบหลายบริการ |
| [C11] 2 | **[Assumption]** แต่ละบริการจัดการสัญญาของตนได้อิสระขึ้น | **[Assumption]** ต้องดูแลรูปแบบข้อมูลข้ามบริการและอาจต้องแปลงข้อมูลเพิ่ม |
| [C12] 3 | **[Assumption]** ผู้ส่งงานไม่ต้องรอผู้รับในทุกขั้น | **[Assumption]** การคืนผล checkout ทันทีต้องมีกลไกติดตามสถานะและจัดการความสำเร็จบางส่วน |

## Decision

- [C13] **[Inferred]** รูปแบบที่สร้างขึ้นใหม่คือใช้ gRPC request/response สำหรับเส้นทางภายในที่ต้องใช้ผลทันที และรวมชนิดข้อความ/RPC ที่เกี่ยวข้องใน demo.proto; เหตุผลที่เป็นไปได้คือช่วยประกอบผล checkout และใช้ชนิดข้อมูลร่วมกัน (system/online-boutique/src/checkoutservice/main.go, PlaceOrder; system/online-boutique/protos/demo.proto, OrderResult). เหตุผลจริงและขอบเขต “ทุกบริการ”: Not evidenced in the provided files.

## Consequences

- **Positive:** [C14] **[Inferred]** การได้รับผล RPC ก่อนขั้นต่อไปช่วยตรวจข้อผิดพลาด ณ จุดเรียก (quality attribute: correctness); PlaceOrder ตรวจ err หลังขั้นเตรียมสินค้า ชำระเงิน และจัดส่ง (system/online-boutique/src/checkoutservice/main.go, PlaceOrder).
- **Negative:** [C15] **[Inferred]** การรอ RPC ตามลำดับอาจสะสม latency และการเปลี่ยนข้อความร่วมอาจเพิ่มต้นทุนแก้ไขหลายบริการ (quality attributes: performance, modifiability). ไม่มีผลวัดจริงในหลักฐาน.
- **Risks and mitigations:** [C16] **[Inferred]** dependency ตอบผิดพลาดอาจทำให้ checkout ล้ม; ควรวัด latency/ข้อผิดพลาดราย RPC และทบทวน deadline (quality attribute: availability). PlaceOrder คืนข้อผิดพลาดจากหลายขั้น แต่หลักฐานไม่แสดงมาตรการนี้ครบถ้วน (system/online-boutique/src/checkoutservice/main.go, PlaceOrder).

## Validation

- [C17] **[Inferred]** ทดสอบ checkout เมื่อแต่ละ dependency ตอบช้าหรือล้มเหลว ตรวจ latency รวมและข้อผิดพลาดของ PlaceOrder และตรวจความเข้ากันได้ของ message หลังปรับ demo.proto. ยังไม่มีผลทดสอบหรือเกณฑ์ผ่านในหลักฐาน.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| C1 | หลาย service definitions อยู่ใน demo.proto เดียว | Evidenced | system/online-boutique/protos/demo.proto (service CartService, service CheckoutService) |
| C2 | frontend รับ HTTP และเปิด gRPC clients เจ็ดบริการ | Evidenced | system/online-boutique/src/frontend/main.go (main: r.HandleFunc, mustConnGRPC) |
| C3 | PlaceOrder ใช้ผล RPC หลายขั้นก่อนตอบกลับ | Evidenced | system/online-boutique/src/checkoutservice/main.go (PlaceOrder, prepareOrderItemsAndShippingQuoteFromCart) |
| C4 | ทุกบริการใช้ gRPC และสร้างจาก proto เดียว | Inferred | Not evidenced in the provided files; ตรวจได้เพียง frontend/main.go และ checkoutservice/main.go ที่ระบุข้างต้น |
| C5 | ลำดับ checkout ต้องใช้ผลก่อนตอบ | Inferred | system/online-boutique/src/checkoutservice/main.go (PlaceOrder) |
| C6 | การรักษาชนิดข้อมูลร่วมเป็นแรงจูงใจ | Inferred | system/online-boutique/protos/demo.proto (CartItem, Money, Address, OrderResult) |
| C7 | ตัวเลือก gRPC/สัญญารวมตรงกับรูปแบบที่เห็น | Evidenced | system/online-boutique/protos/demo.proto (service definitions); system/online-boutique/src/checkoutservice/main.go (mustConnGRPC) |
| C8 | HTTP/REST และสัญญาแยกเป็นทางเลือก | Assumption | Not evidenced in the provided files |
| C9 | asynchronous broker และ schema แยกเป็นทางเลือก | Assumption | Not evidenced in the provided files |
| C10 | ข้อดี/เสียของ gRPC และสัญญารวม | Inferred | system/online-boutique/src/checkoutservice/main.go (PlaceOrder); system/online-boutique/protos/demo.proto (Money, OrderResult) |
| C11 | ข้อดี/เสียของ HTTP/REST แยกสัญญา | Assumption | Not evidenced in the provided files |
| C12 | ข้อดี/เสียของ asynchronous broker | Assumption | Not evidenced in the provided files |
| C13 | เหตุผลของรูปแบบที่สร้างขึ้นใหม่ | Inferred | system/online-boutique/src/checkoutservice/main.go (PlaceOrder); system/online-boutique/protos/demo.proto (OrderResult) |
| C14 | ตรวจข้อผิดพลาดระหว่าง checkout ได้ ณ จุดเรียก | Inferred | system/online-boutique/src/checkoutservice/main.go (PlaceOrder) |
| C15 | latency และต้นทุนการแก้สัญญาอาจเพิ่ม | Inferred | system/online-boutique/src/checkoutservice/main.go (PlaceOrder); system/online-boutique/protos/demo.proto (service definitions) |
| C16 | ความเสี่ยง dependency และแนวทางวัด/deadline | Inferred | system/online-boutique/src/checkoutservice/main.go (PlaceOrder) |
| C17 | วิธีทดสอบที่เสนอ | Inferred | system/online-boutique/src/checkoutservice/main.go (PlaceOrder); system/online-boutique/protos/demo.proto (service definitions) |
