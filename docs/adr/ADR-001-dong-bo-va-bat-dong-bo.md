# ADR-001: order-service gọi catalog-service đồng bộ (REST) có timeout và cầu dao

- **Trạng thái:** Đã chấp nhận
- **Ngày:** 2026-11-14
- **Người quyết định:** Châu Minh Nhã (B2305596)
- **Liên quan:** Lab 3 (TH3), kiểm thử `CatalogCircuitBreakerTest`, `CatalogClientContractTest`

## 1. Bối cảnh
Khi đặt hàng, `order-service` cần giá sách hiện hành từ `catalog-service` để tính tổng tiền. Tuần trước một lần
catalog chậm 10 giây đã làm luồng đặt hàng treo: mỗi yêu cầu giữ một luồng Tomcat suốt thời gian chờ, các luồng
cạn dần và lỗi lan sang cả `order-service`. Ràng buộc: giá phải đúng tại thời điểm đặt (không dùng giá cũ), người
dùng cần phản hồi ngay (đơn có mã `ORD-…` và tổng tiền), và nhóm chỉ có một phòng thực hành nên cần giải pháp đơn giản.

## 2. Các phương án đã xem xét
| Phương án | Ưu điểm | Nhược điểm |
|---|---|---|
| A. Gọi đồng bộ REST, không bảo vệ | Đơn giản nhất, giá luôn mới | Catalog chậm thì treo luồng, lỗi lan truyền |
| B. Gọi đồng bộ REST + timeout + cầu dao (Resilience4j) | Giá luôn mới, lỗi nhanh (503), không lan truyền, vẫn đơn giản | Phải chỉnh số liệu cầu dao; catalog tắt thì không đặt được hàng |
| C. Bất đồng bộ qua hàng đợi/sự kiện, sao chép giá cục bộ | Chịu lỗi tốt nhất, catalog tắt vẫn nhận đơn | Giá có thể cũ, phức tạp (broker, nhất quán cuối cùng), vượt phạm vi TH3 |

## 3. Quyết định
Chúng tôi chọn **phương án B**: `order-service` gọi `catalog-service` đồng bộ bằng `RestClient` với timeout kết nối
1 s, timeout đọc 2 s, bọc trong cầu dao `catalog` (cửa sổ trượt 10 lời gọi, tối thiểu 5 lời gọi, ngưỡng lỗi 50 %,
mở 15 s, bỏ qua `UnknownBookException` vì 404 là lỗi nghiệp vụ). Khi catalog chậm, tắt hoặc cầu dao mở, dịch vụ trả
503 kèm `Retry-After: 10` ngay lập tức. Để tránh đặt trùng khi client gửi lại, `POST /api/v1/orders` hỗ trợ
`Idempotency-Key`, được kiểm tra trước khi gọi catalog.

## 4. Hệ quả
- Tích cực: sự cố catalog không làm treo `order-service`; giá luôn đúng; kiến trúc đủ đơn giản để vận hành.
- Tiêu cực / rủi ro chấp nhận: khi catalog ngừng hoạt động thì không đặt được hàng (mất tính sẵn sàng cho thao tác ghi).
- Việc phải làm tiếp theo: giám sát trạng thái cầu dao qua metrics; cân nhắc phương án C nếu cần nhận đơn khi catalog tắt.

## 5. Cách kiểm chứng
- `CatalogCircuitBreakerTest`: WireMock trả chậm 1500 ms, timeout đọc 300 ms, cửa sổ 4, tối thiểu 2 lời gọi →
  cả 5 lần POST đều 503 trong dưới 1400 ms và WireMock chỉ nhận đúng 2 yêu cầu.
- `CatalogClientContractTest` + `ContractVerifierTest`: hợp đồng `getBookFound`, `getBookNotFound` giữ cho hai bên
  thống nhất; nếu đổi hình dạng phản hồi của catalog thì build của catalog đỏ trước khi hỏng order-service.
- Xem xét lại khi p95 độ trễ của catalog vượt 1,5 s kéo dài hoặc khi cầu dao mở thường xuyên.
