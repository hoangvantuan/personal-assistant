# Quality Gate (cổng ghi SOT: pk-init, pk-plan, pk-track)

Quality Gate là cổng kiểm soát trước khi ghi dữ liệu vào SOT. Áp dụng cho mọi skill GHI SOT từ input user: **pk-init** (tạo objective/KR/KI), **pk-plan** (tạo action/effort), **pk-track** (ghi KR current/KI current). Capture và reflect chỉ ghi vào inbox (staging), KHÔNG phải SOT trực tiếp, nên KHÔNG bắt buộc Quality Gate đầy đủ; QG áp khi item vào SOT (lúc pk-track/pk-distill xử lý).

Mỗi khi user trả lời 1 câu hỏi từ skill, agent tự kiểm tra 3 câu **không hiển thị cho user**:

1. **Đủ cụ thể?** Câu trả lời có thể chuyển thành KR/KI đo được (init) hoặc action có deliverable rõ (plan) không?
2. **Giả định ẩn?** User có bỏ qua constraint quan trọng (capacity, deadline, dependency)? Đặc biệt: đầu vào đã verify từ source đáng tin chưa (external system, user confirm, dữ liệu thật)?
3. **Mâu thuẫn?** Câu trả lời có xung đột với context trước (capacity, timeline, objective, KR đã chốt)?

## Hành vi theo kết quả

| Kết quả | Hành vi |
| --- | --- |
| Cả 3 pass | Đi tiếp câu hỏi kế |
| Bất kỳ fail | In 1 dòng `(Mình đào sâu thêm vì <lý do cụ thể>)` TRƯỚC follow-up |
| User "chưa biết" / "để sau" | Ghi nhận, đánh dấu `⚠️ TBD`. Phase confirm nhắc lại TBD trước ghi. |
| User sốt ruột | Giảm độ sâu, chỉ giữ câu 1. Không skip hoàn toàn. |
| User paste từ doc | Tóm tắt lại, hỏi "tôi hiểu đúng chưa?" |

## Nguyên tắc

- Quality Gate là **internal check**, không hiển thị 3 câu cho user.
- Lý do trong ngoặc phải CỤ THỂ. Ví dụ: `(Mình đào sâu thêm vì "tăng doanh thu" chưa nói kênh nào, sản phẩm nào.)` KHÔNG viết `(Mình cần thêm thông tin)`.
