# Biên bản đàm phán hợp đồng API

- Cặp đàm phán:
- Product: B
- Provider:Notification (B7)
- Consumer:Core Business (B6)
- Phiên: v1.0
- Ngày: 20/5/2026

---

## Issue #1

- Raised by: Consumer (Core Business)
- Endpoint: Event `alert.created`
- Concern: Consumer muốn gửi event với `severity` để Notification quyết định channel gửi, nhưng chưa thống nhất được enum severity.
- Proposal: Dùng enum 4 mức: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`. Consumer cam kết gửi đúng mức dựa trên policy.
- Resolution: Accepted 
- Rationale: Đủ mức để phân loại, không quá phức tạp. Notification sẽ map mỗi mức sang channel tương ứng.
- Impact: Consumer phải đảm bảo logic xác định severity đúng. Notification phải implement routing theo 4 mức này.

---

## Issue #2

- Raised by: Provider (Notification)
- Endpoint: Event `alert.created`
- Concern: Provider cần xác định gửi thông báo đến ai, nhưng Consumer chưa gửi kèm `userId` hoặc `recipient`.
- Proposal: Consumer bắt buộc gửi thêm field `userId` trong `data`. Notification sẽ tự tra cứu contact info (Telegram ID, Email) từ user database.
- Resolution: Accepted
- Rationale: Core Business là nơi biết user nào liên quan đến alert (ví dụ: user quẹt thẻ, user phụ trách khu vực). Notification không nên tự suy diễn.
- Impact: Consumer phải đảm bảo `userId` luôn có mặt. Notification phải có cơ chế tra cứu user contact.

---

## Issue #3

- Raised by: Consumer (Core Business)   
- Endpoint: Event `alert.escalated`
- Concern: Consumer muốn gửi event khi alert chưa được xử lý sau 5 phút, nhưng chưa rõ Provider có cần `escalatedBy` hay không.
- Proposal: Thêm field `escalatedBy` (string, optional) để ghi nhận ai/component nào kích hoạt escalation. Consumer gửi "auto-escalation-policy" nếu do system, hoặc admin ID nếu do người escalate thủ công.
- Resolution: Modified (chấp nhận optional, không bắt buộc)
- Rationale: Hữu ích cho audit nhưng không phải lúc nào cũng có. Không nên bắt buộc để tránh block event.
- Impact: Consumer có thể gửi hoặc không. Provider phải xử lý được cả 2 trường hợp (có hoặc không có field).


---

## Issue #4

- Raised by: Provider (Notification)
- Endpoint: Tất cả event
- Concern: Provider lo ngại Consumer gửi trùng event (do retry) dẫn đến gửi thông báo trùng lặp cho user.
- Proposal: Consumer bắt buộc gửi `eventId` (UUID). Provider dùng `eventId` để deduplicate trong vòng 5 phút. Nếu `eventId` trùng → bỏ qua, không gửi thông báo lần 2.
- Resolution: Accepted 
- Rationale: Giải pháp chuẩn cho idempotency trong queue async. Không yêu cầu Consumer nhớ state phức tạp.
- Impact: Consumer phải sinh `eventId` cho mỗi event. Provider phải implement deduplication cache (Redis hoặc in-memory).

---

## Issue #5

- Raised by: Consumer (Core Business)
- Endpoint: Event `alert.resolved`
- Concern: Consumer muốn gửi thông báo "đã giải quyết" đến user, nhưng không chắc Provider có cần `resolutionNote` hay không.
- Proposal: Thêm field `resolutionNote` (string, optional, max 500 chars) để ghi lý do xử lý. Consumer sẽ gửi nếu có, Provider sẽ gửi kèm trong thông báo.
- Resolution: Accepted 
- Rationale: Tăng trải nghiệm user (biết được alert được xử lý thế nào). Optional nên không ảnh hưởng đến luồng chính.
- Impact:  Consumer cần thu thập note từ admin nếu có. Provider cần hiển thị note trong thông báo (nếu có).

---

## Issue #6

- Raised by: Provider (Notification)
- Endpoint: Tất cả event
- Concern: Provider giới hạn xử lý tối đa 500 event/phút. Consumer lo ngại nếu có burst alert (ví dụ cháy, nhiều sensor kích hoạt) có thể vượt quá giới hạn.
- Proposal:Thống nhất:
  - Consumer cam kết gửi tối đa 500 event/phút (≈ 8-9 event/giây).
  - Nếu vượt quá, Consumer tự throttling hoặc gộp batch.
  - Provider sẽ reject event nếu queue full, Consumer phải retry.
- Resolution: Modified (thêm cơ chế backpressure)
- Rationale: Bảo vệ Provider khỏi overload, đồng thời Consumer phải có trách nhiệm kiểm soát lưu lượng.
- Impact: Consumer phải implement throttling hoặc batch. Provider phải có cơ chế reject và trả lỗi rõ ràng.

---

# Chốt hợp đồng v1.0

Provider sign-off:  _Notification Team (B7)_  
Consumer sign-off:  _Core Business Team (B6)_ 
Witness (GV/TA):    _FIT4110 Teaching Team_ 
Date: _2026-05-20_            

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có (cặp Queue async chưa dùng Spectral trong Lab 02) | Queue async chưa yêu cầu Spectral ở Lab 02, sẽ dùng AsyncAPI + Spectral cho async ở Lab 03 | Lab 03 sẽ bổ sung ruleset cho async |
| Thiếu example cho event `alert.escalated` | Chưa đàm phán xong format chi tiết | Bổ sung trong Lab 03 khi viết AsyncAPI |
