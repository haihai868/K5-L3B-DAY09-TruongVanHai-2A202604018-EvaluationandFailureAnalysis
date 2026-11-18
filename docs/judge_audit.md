# Judge audit

Chạy: `python -m lab9.judge --csv <file> --audit 5`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok | 100 | 0.757 | 0.857 | 0.800 | ĐẠT (0.757 ≥ 0.6, 0.857 ≥ 0.8, 0.8 ≥ 0.7) | dùng |
| helpful | 100 | 0.375 | 1.000 | 0.300 | KHÔNG ĐẠT (κ=0.375 < 0.6, R=0.3 < 0.7) | loại |

## ≥5 case audit thủ công

### 1. Phân tích case bất đồng của `judge_tone`
| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú & Hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | H-008: "Em đã hoàn tiền cho anh rồi nha, anh check giúp em." | PASS | FAIL | Người đúng | Lời văn xưng hô "Em - anh" suồng sã, thiếu tính chuyên nghiệp của hệ thống CSKH doanh nghiệp. Cần cập nhật rubric: cấm đại từ nhân xưng phi chuẩn mực doanh nghiệp. |
| 2 | H-017: "Dạ, bạn ơi LogiX đã hoàn 25.000đ cho đơn O-606 rồi nha." | PASS | FAIL | Người đúng | Cụm từ "bạn ơi ... rồi nha" không chuẩn mực văn phong trang trọng. Sửa rubric: cấm khẩu ngữ "nha", "nhé", "bạn ơi". |
| 3 | H-029: "Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm." | PASS | FAIL | Người đúng | Văn phong cộc lốc, gắt gỏng ("không cần nhắn thêm") gây cảm giác khó chịu cho khách hàng. Sửa rubric: phạt nặng thái độ thiếu tôn trọng khách hàng. |
| 4 | H-042: "Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ." | FAIL | PASS | Người đúng | Judge bắt nhầm (False Positive). Câu trả lời lịch sự, rõ ràng về thời gian tiền về. Cần tinh chỉnh prompt judge để không phạt câu trả lời có thêm thông tin hữu ích về thời gian xử lý. |
| 5 | H-051: "Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ." | FAIL | PASS | Người đúng | Judge bắt lỗi chêm từ tiếng Anh "refund", nhưng trong bối cảnh logistics/thương mại điện tử từ này được chấp nhận phổ biến. Cần làm rõ trong rubric cho phép từ mượn thông dụng. |

### 2. Phân tích case bất đồng của `judge_helpful`
| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú |
|---|---|---|---|---|---|
| 1 | HL-015: "Dạ, LogiX đã hoàn 55.000đ cho đơn O-112 ạ." | PASS | FAIL | Người đúng | Khách khiếu nại chưa được giải quyết thấu đáo, chỉ báo hoàn tiền mà không giải thích điều khoản hay hướng dẫn tiếp theo. |
| 2 | HL-016: "Dạ, LogiX thành thật xin lỗi anh/chị vì sự bất tiện này..." | PASS | FAIL | Người đúng | Chỉ xin lỗi chung chung, không đưa ra hành động giải quyết cụ thể nào cho yêu cầu của khách hàng. |
| 3 | HL-020: "Dạ, LogiX đã hoàn 90.000đ cho đơn O-303 ạ." | PASS | FAIL | Người đúng | Judge bị lừa bởi câu trả lời ngắn gọn, bỏ qua việc thiếu thông tin cần thiết. |
| 4 | HL-028: "Dạ, LogiX thành thật xin lỗi anh/chị vì sự bất tiện này..." | PASS | FAIL | Người đúng | Giống HL-016: thiếu hành động xử lý đơn hàng. |
| 5 | HL-035: "Dạ, Một lần nữa LogiX xin lỗi vì trải nghiệm chưa tốt..." | PASS | FAIL | Người đúng | Lời xin lỗi suông, không có thông tin tra cứu hay kết quả xử lý bồi thường. |

## Kết luận
- **Tiêu chí bị loại:** `judge_helpful` bị loại hoàn toàn khỏi CI gate vì:
  + Ma trận nhầm lẫn: `{'tp': 9, 'fp': 0, 'fn': 21, 'tn': 70}`.
  + Số lượng bỏ lọt lỗi quá lớn: có tới 21 ca người gắn nhãn FAIL nhưng judge lại cho PASS (`fn = 21`), dẫn tới Recall chỉ đạt 30% (thấp hơn nhiều so với ngưỡng tối thiểu 70%).
  + Hệ số Cohen's $\kappa = 0.375$ (thấp hơn sàn 0.60), cho thấy sự đồng thuận giữa judge và người rất yếu, chủ yếu do judge có xu hướng dễ dãi cho qua các câu xin lỗi suông hoặc câu trả lời không hữu ích.
- **Rủi ro còn lại:**
  + `judge_tone` đạt ngưỡng nhưng mới chỉ được kiểm định trên tập CSV replay công khai (100 mẫu). Cần bổ sung tập held-out mới từ production để tránh overfitting và bias từ cùng một phân phối dữ liệu huấn luyện.
