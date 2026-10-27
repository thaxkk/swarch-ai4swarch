# ผลการตรวจสอบ Run B

ตรวจ dependency จาก Run B จำนวน 7 รายการ โดยอ่าน configuration และsource code ไม่ได้ทดสอบระบบที่กำลังทำงาน: ยืนยันได้ 6 รายการ ส่วน Shopping Assistant ยืนยันการเรียก HTTP ในโค้ดได้ แต่ยังยืนยันการ deploy ปลายทางไม่ได้

กำหนด `K` = `system/online-boutique/kubernetes-manifests/` และ `S` = `system/online-boutique/src/`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่พบจากการตรวจไฟล์ |
|---|---|---|---|
| checkoutservice → productcatalogservice | `K/checkoutservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | บรรทัด 57–58 กำหนด `productcatalogservice:3550`; `S/checkoutservice/main.go:341–344` สร้าง `ProductCatalogServiceClient` และเรียก `GetProduct` |
| checkoutservice → shippingservice | `K/checkoutservice.yaml` (`SHIPPING_SERVICE_ADDR`) | ✅ | บรรทัด 59–60 กำหนด `shippingservice:50051`; `S/checkoutservice/main.go:314–315, 387` เรียกทั้ง `GetQuote` และ `ShipOrder` |
| checkoutservice → paymentservice | `K/checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | บรรทัด 61–62 กำหนด `paymentservice:50051`; `S/checkoutservice/main.go:370` เรียก `PaymentServiceClient.Charge` |
| checkoutservice → emailservice | `K/checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | บรรทัด 63–64 กำหนด `emailservice:5000`; `S/checkoutservice/main.go:380` เรียก `SendOrderConfirmation` |
| checkoutservice → currencyservice | `K/checkoutservice.yaml` (`CURRENCY_SERVICE_ADDR`) | ✅ | บรรทัด 65–66 กำหนด `currencyservice:7000`; `S/checkoutservice/main.go:360` เรียก `CurrencyServiceClient.Convert` |
| checkoutservice → cartservice | `K/checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | บรรทัด 67–68 กำหนด `cartservice:7070`; `S/checkoutservice/main.go:325, 333` เรียกทั้ง `GetCart` และ `EmptyCart` |
| frontend → shoppingassistantservice | `K/frontend.yaml` (`SHOPPING_ASSISTANT_SERVICE_ADDR`) | ❓ | บรรทัด 83–84 กำหนด endpoint จริง และ `S/frontend/handlers.go:464–472` ส่ง HTTP POST จริง จึงไม่ต้องอนุมาน HTTP จาก port 80 แต่ `K/kustomization.yaml:35` คอมเมนต์ component นี้ไว้ จึงยังยืนยันไม่ได้ว่าปลายทางถูก deploy |

สำหรับหกรายการของ `checkoutservice` โค้ดสร้าง gRPC connections ที่ `S/checkoutservice/main.go:118–123` และใช้ `grpc.NewClient` ที่บรรทัด 214–216 จึงมีหลักฐานเรื่อง protocol เพิ่มเติมจากชื่อ port ใน manifest

## สิ่งที่ผิดหรือขาดหายไปใน Run B

- **ขอบเขตของ Shopping Assistant ยังไม่มีหลักฐานรองรับ:** แผนภาพใช้ `System_Ext` แต่การมี endpoint ใน `K/frontend.yaml:83–84` ไม่ได้พิสูจน์ว่าเป็นระบบภายนอก ขอบเขตดังกล่าวไม่ปรากฏหลักฐานในไฟล์ที่ตรวจ จึงควรระบุว่าไม่ทราบ แทนการจัดเป็นระบบภายนอกอย่างแน่นอน
- **สถานะของ loadgenerator ในแผนภาพไม่ชัด:** Run B วาดเป็น container ปกติ แม้จะอธิบายข้อจำกัดไว้ใน Gaps แล้วก็ตาม; `K/kustomization.yaml:24` คอมเมนต์ resource นี้ไว้ จึงควรทำเครื่องหมายในแผนภาพว่าไม่ได้รวมในชุด Kustomize นี้ ส่วนการ deploy ผ่านวิธีอื่นยังยืนยันไม่ได้
- **รายละเอียดการใช้ cart ขาดไป:** แผนภาพ Run B ระบุ `checkoutservice → cartservice` ว่า “Reads cart” แต่โค้ดเรียก `EmptyCart` ด้วย (`S/checkoutservice/main.go:333`) ควรเปลี่ยนคำอธิบายเป็นการอ่านและล้างตะกร้า

## สะท้อนคิด

1. **เปรียบเทียบ A กับ B:** จาก `.proto` เพียงไฟล์เดียว agent รู้ชื่อ service, RPC และโครงสร้างข้อมูลได้ แต่ระบุไม่ได้ว่าใครเรียกใคร ใช้ภาษาอะไร มี Redis หรือ deploy อย่างไร เพราะไฟล์นี้กำหนดสัญญาของ API ไม่ใช่ implementation หรือ deployment (`system/online-boutique/protos/demo.proto`, symbols `CheckoutService`, `CartService`) ตัวอย่างเช่น การที่ `checkoutservice` เรียก `cartservice` ต้องตรวจเพิ่มจาก `K/checkoutservice.yaml:67–68` และ `S/checkoutservice/main.go:325, 333`
2. **ข้อผิดพลาดที่อาจทำให้สมาชิกใหม่เข้าใจผิดมากที่สุด:** การวาด Shopping Assistant เป็นระบบภายนอกอย่างแน่นอนอาจทำให้สมาชิกใหม่เข้าใจขอบเขตความรับผิดชอบของระบบผิด ทั้งที่หลักฐานยืนยันเพียง configured endpoint และโค้ด HTTP call (`K/frontend.yaml:83–84`; `S/frontend/handlers.go:464–472`) ควรระบุว่าขอบเขตและสถานะ deployment ยังไม่ยืนยัน เพราะ component นี้ถูกคอมเมนต์ไว้ใน `K/kustomization.yaml:35`

## ไฟล์ที่อ่านเพื่อจัดทำผลตรวจ

- `exercise/outputs/1A-architecture.md`
- `exercise/outputs/1B-architecture.md`
- `system/online-boutique/protos/demo.proto`
- `system/online-boutique/kubernetes-manifests/frontend.yaml`
- `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`
- `system/online-boutique/kubernetes-manifests/cartservice.yaml`
- `system/online-boutique/kubernetes-manifests/kustomization.yaml`
- `system/online-boutique/src/checkoutservice/main.go`
- `system/online-boutique/src/frontend/main.go`
- `system/online-boutique/src/frontend/handlers.go`
