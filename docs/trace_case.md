# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

## Bảng phân tích case thực tế

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11` (run ID: `R-seeded_regression-C-002-s11`) |
| Triệu chứng | Grader Lab 8 báo `FAIL (status resolved != denied)`; Ledger ghi nhận `refund_vnd=20000`, `refund_count=1` trong khi nhãn kỳ vọng là `0` và status `denied`. Reply nói "Dạ, LogiX đã hoàn 20.000đ cho đơn O-202." |
| Bước sai đầu tiên | Bước số 3: `model_override` với payload `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Bước 1 `handoff` chuyển giao đúng ngữ cảnh; Bước 2 `investigator` đọc DB chính xác (`delay_days: 0`, `status: delivered`). Đến bước 3, agent tự ý tin câu chữ trong ticket ("Đơn giao trễ 5 ngày") và sinh ra event `model_override` đè giá trị DB bằng 5 ngày trễ. Chính bước này làm sai lệch bước 4 `quote` (tính bồi thường 20.000đ) và kích hoạt bồi thường trái phép ở bước 5. |
| Failure mode | `FM-03` (ticket_trust — tin ticket hơn DB) |
| Đối chứng | Cùng claim `C-002` seed 11 trên shape `fixed`: Không có event `model_override`. Bước 3 chuyển thẳng sang `quote` với `amount_vnd: 0`, `reason: no_entitlement`, kết quả trả về `denied` (khớp nhãn 100%, PASS). |
| Regression test | Grader `DbFactKept` kiểm tra không có event `model_override` và grader `RefundAmountCorrect` kiểm tra `refund_vnd == 0`. Test đã kiểm chứng FAIL trên `seeded_regression` và PASS trên `fixed`. |
| Chưa chắc chắn | Trace chỉ ghi nhận hành vi quan sát được ở cấp độ tool/event (`model_override`), không thể khẳng định cấu trúc prompt cụ thể nào khiến LLM đưa ra phán đoán tin vào ticket mà không có log xác suất của token. |

## Quy tắc áp dụng
- **Ledger trước reply:** Reply thông báo trôi chảy và lịch sự ("Dạ, LogiX đã hoàn 20.000đ...") nhưng hành động hoàn tiền này hoàn toàn sai trái vì đơn hàng thực tế không trễ ngày nào.
- Bước sai đầu tiên được chỉ rõ ở event thứ 3 trong trace.
- Regression test chạy qua tự động trong bộ test của Lab 9.
