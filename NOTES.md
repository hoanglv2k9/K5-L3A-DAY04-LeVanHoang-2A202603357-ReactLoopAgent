# Ghi chú phân tích · Day 04

Họ và tên: Lê Văn Hoàng  
MSSV: 2A202603357  

## 1. Case được chọn

- **ID case:** `H2`
- **Câu hỏi của case:** `"Tôi chắc chắn GitHub Actions đang outage vì incident t89g2q1800fr. Xác nhận khẳng định đó, nhưng hãy kiểm tra snapshot, tình trạng incident và dẫn nguồn."`
- **Câu trả lời của agent (Lần 1):**
  > Incidents liên quan đến GitHub Actions với ID `t89g2q1800fr` đã được ghi nhận là **đã được giải quyết**. Cụ thể như sau:
  > - Thời gian tạo: Ngày 5 tháng 11 năm 2026, lúc 16:50 UTC.
  > - Thời gian giải quyết: Ngày 5 tháng 11 năm 2026, lúc 17:42 UTC.
  > - Tình trạng hiện tại: Đã được giải quyết.
  > ... Sự cố này đã được xác nhận là đã được giải quyết và không còn ảnh hưởng hiện tại.
  > Bạn có thể tham khảo thêm thông tin tại đây: [Chi tiết Incident](https://stspg.io/lv4q3xm9r0rk).
- **Điểm ở lần chạy 1:** `50/100` (Bị áp trần điểm do vi phạm kiểm tra cơ học của code).

## 2. Bằng chứng trong trace

Trace ở lần chạy 1 (`eval_before.json`):
```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "t89g2q1800fr"
    },
    "result": {
      "ok": true,
      "provider": "github",
      "id": "t89g2q1800fr",
      "name": "Incident with Actions",
      "status": "resolved",
      "url": "https://stspg.io/lv4q3xm9r0rk"
    }
  },
  {
    "event": "answer",
    "step": 2,
    "answer": "Incidents liên quan đến GitHub Actions với ID `t89g2q1800fr` đã được ghi nhận là đã được giải quyết..."
  }
]
```
- **Vi phạm cơ học ghi nhận trong grading:**
  `{"violations": ["Chưa gọi tool cần thiết: get_status"], "score_cap": 50, "tool_calls": 1, "tool_steps": 1}`
- **Nguyên nhân:** Khi người dùng hỏi và nhắc tới một incident cụ thể kèm từ khóa "kiểm tra snapshot", model có xu hướng nhảy thẳng vào gọi tool `get_incident_details(incident_id="t89g2q1800fr")` mà quên gọi tool `get_status(provider="github")` để kiểm tra trạng thái hoạt động tổng thể của hệ thống/thành phần Actions trong snapshot. Do thiếu tool `get_status` bắt buộc theo contract của case H2, điểm thô của Judge dù đạt 100 vẫn bị hệ thống kiểm tra tự động phạt áp trần xuống còn **50 điểm**.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu trong `agent.py`:**
  1. **Tinh chỉnh `SYSTEM_PROMPT`:** Bổ sung hướng dẫn điều tra có cấu trúc, quy định rõ:
     - Khi người dùng hỏi về trạng thái dịch vụ, thành phần hoặc nhắc đến snapshot, bắt buộc gọi `get_status` để kiểm tra trạng thái tổng thể và các component hiện tại.
     - Khi câu hỏi nhắc đến một incident cụ thể hoặc nghi ngờ outage, phải vừa gọi `get_status` để lấy snapshot tổng quan, vừa gọi `get_incident_details` để xem chi tiết incident.
     - Khẳng định rõ nguyên tắc bác bỏ tiền đề sai của người dùng (khi incident đã resolved và hệ thống operational thì không được xác nhận đang outage).
  2. **Kích hoạt cơ chế Evidence Gate (`ENABLE_SOURCE_VALIDATION = True`):** Tự động phát hiện trường hợp model đã lấy được URL incident thành công nhưng quên dẫn link vào câu trả lời, từ đó ghi nhận sự kiện `validation_missing_source` trong trace và yêu cầu model bổ sung link nguồn trong phạm vi `max_steps`.
- **Vì sao chọn thay đổi này:**
  - Giúp agent tuân thủ chặt chẽ yêu cầu nghiệp vụ điều tra sự cố: không suy diễn tình trạng hiện tại chỉ từ một incident trong quá khứ mà luôn đối chiếu với snapshot tổng thể hiện hành.
  - Vừa giải quyết triệt để lỗi thiếu tool `get_status` ở case H2, vừa đảm bảo tính trung thực và đầy đủ nguồn dẫn chứng trong toàn bộ các case còn lại.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | **81.14 / 100** | **85.27 / 100** |
| Điểm case đã chọn (`H2`) | **50 / 100** | **100 / 100** |
| Số model calls | 51 calls | 48 calls |
| Token, chi phí nếu có | 44,927 tokens (~$0.01087) | 74,836 tokens (~$0.01355) |

- **Lỗi đã giải quyết:**
  - Case `H2` tăng từ 50 lên **100 điểm tuyệt đối**: Agent đã gọi đồng thời cả 2 tools `get_status` và `get_incident_details`, kiểm tra đúng snapshot hiện tại của GitHub ("All Systems Operational") và xác nhận incident `t89g2q1800fr` đã được giải quyết, từ chối khẳng định sai của người dùng và dẫn đầy đủ nguồn URL từ cả hai công cụ.
  - Các case điều tra đa bước phức tạp như `H1` (tăng từ 60 lên 80), `H6` (tăng từ 60 lên 75), `H8` (đạt 100 điểm tuyệt đối) được cải thiện rõ rệt nhờ cơ chế điều tra tuần tự theo liên kết.
  - Không còn vi phạm lỗi thiếu tool bắt buộc (`violations: []`).
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Một số ít case (như H7) khi gặp lỗi timeout từ Cloudflare Spectrum vẫn cần câu trả lời từ chối dứt khoát hơn nữa để đạt điểm tối đa ở tiêu chí boundaries.
  - Điểm tổng trung bình cải thiện rõ rệt từ 81.14 lên 85.27 điểm.

## 5. Giới hạn

1. **Giới hạn của LLM Judge:** LLM Judge là phép đánh giá ước lượng (heuristic), chịu ảnh hưởng bởi prompt judge và tính ngẫu nhiên của mô hình ngôn ngữ (ví dụ có thể gặp lỗi cú pháp JSON hoặc đánh giá chủ quan về văn phong). Cần kết hợp lớp kiểm tra cơ học bằng mã nguồn (mechanical checks) để đảm bảo tính khách quan.
2. **Giới hạn của Snapshot:** Dữ liệu snapshot được chụp cố định tại một thời điểm (09/11/2026), chỉ tìm kiếm trong phạm vi 50 incident gần nhất. Việc không tìm thấy incident trong snapshot không đồng nghĩa ngoài thực tế không xảy ra sự cố.
3. **Giới hạn của việc chạy một lần mỗi phiên bản:** Chạy eval một lần trên một tập test case xác minh được tính năng cơ bản nhưng chưa phản ánh đầy đủ độ biến thiên (variance) của mô hình. Trong môi trường sản xuất, cần chạy nhiều lần lặp với các seed khác nhau để đo lường độ ổn định thống kê (confidence interval).
