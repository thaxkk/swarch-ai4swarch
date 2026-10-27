**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| checkoutservice → productcatalogservice | `checkoutservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | `src/checkoutservice/main.go` (`prepOrderItems`, `GetProduct`) ใช้ข้อมูลสินค้าเพื่อคิดราคา |
| checkoutservice → shippingservice | `checkoutservice.yaml` (`SHIPPING_SERVICE_ADDR`) | ✅ | `src/checkoutservice/main.go` (`quoteShipping`, `shipOrder`) ใช้ทั้งขอราคาและสั่งส่ง |
| checkoutservice → paymentservice | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | `src/checkoutservice/main.go` (`chargeCard`, `Charge`) เรียกเก็บเงินก่อนส่งสินค้า |
| checkoutservice → emailservice | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | `src/checkoutservice/main.go` (`sendOrderConfirmation`) เรียกหลัง shipping และหากผิดพลาดเพียง log warning |
| checkoutservice → cartservice | `checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | `src/checkoutservice/main.go` (`GetCart`, `EmptyCart`) อ่านและล้างตะกร้า |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B: แผนภาพจัด `shoppingassistantservice` เป็น `System_Ext` ทั้งที่ไฟล์ที่ให้ดูไม่พิสูจน์ว่าอยู่ภายนอก; `loadgenerator` ปรากฏในแผนภาพแม้ถูกคอมเมนต์ใน `kustomization.yaml`; คำอธิบาย `checkoutservice → cartservice` ขาดการ `EmptyCart` (ดู `outputs/1B-verification.md`).

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. Run A เห็นชื่อ service และรูปแบบ request/response แต่ไม่รู้ว่า service ใดเรียกอีก service, มี Redis หรือ deploy อย่างไร เพราะ `.proto` บอกสัญญา API เท่านั้น. Run B จึงเติมความสัมพันธ์จาก `*_SERVICE_ADDR` ใน manifests ได้ แต่ความสัมพันธ์ที่ config ระบุยังต้องตรวจโค้ดอีกชั้นหนึ่ง.
2. การวาด `shoppingassistantservice` เป็นระบบภายนอกแน่นอนน่าจะทำให้คนใหม่เข้าใจ ownership และขอบเขตการ deploy ผิด. `frontend.yaml` มีเพียง address ส่วน `kustomization.yaml` comment component นี้ไว้ จึงควรทำเครื่องหมายว่าไม่ทราบสถานะและขอบเขต แทนการฟันธง.
