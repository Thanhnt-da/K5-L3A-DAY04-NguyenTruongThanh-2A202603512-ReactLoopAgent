# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Trường Thành
MSSV: 2A202603512

## 1. Case được chọn

- ID case: H6
- Câu trả lời của agent: `eval_before` trả về `h02n9572mx9y` không ghi rõ nguyên nhân mà chỉ hướng đến `xr3myzj7ms51`,và cho rằng nguyên nhân chính là sự kết hợp của hai incidents (event-routing config change, feature flag). Phần giới hạn model nói rằng: _“Bản tóm tắt bị cắt (“truncated”: true) và chỉ hiển thị **4 trong 9 update**”_ (“the summary is truncated and only shows 4 of 9 updates”).
- Điểm ở lần chạy 1: **88** 

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
step 2  get_incident_details {"provider": "github", "incident_id": "xr3myzj7ms51"}
  result: updates_total = 9, updates returned = 3
    resolved      2026-11-04T17:01:11Z  truncated = true
    monitoring    2026-11-04T16:49:27Z  truncated = false
    investigating 2026-11-04T16:05:31Z  truncated = false
answer: "Bản tóm tắt bị cắt ("truncated": true) và chỉ hiển thị 4 trong 9 update"
```
Vấn đề là tool trả về 3 updates nhưng model lại trả về 4
Nguyên nhân là do: prompt chưa bảo model cách đọc `updates_total` và `len(updates)` và `truncated` nên model đã dữ đoán con số. Đây là vấn đề về prompt, không phải do loop hay data.
## 3. Thay đổi đã thực hiện

### Sửa gì, ở đâu:
- Thay đổi 1: thay trong `agent.py`, `SYSTEM_PROMPT`: báo lại đúng `update_total` và update nào bị `truncated`, không tìm thấy -> không tìm thấy trong 50 incident
- Thay đổi 2: trong def `run_agent`: thêm biên seen_urls để lấy được những incidents và lưu url. Giờ code cũng collect url từ những lần chạy `get_incident_details` thành công. Nếu đáp án không chứa Url nào thì `validation_missing_source` yêu cầu mô hình chạy lại và `continue` và khi hết step, mô hình trả về not complete, 37/37 test đã thành công
### Vì sao chọn thay đổi này:
- Thay đổi system_prompt rule giúp giải quyết vấn đề ở H6

## 4. So sánh hai lần chạy

| Chỉ số            | `eval_before.json`           | `eval_after.json` |
| ----------------- | ---------------------------- | ----------------- |
| Điểm tổng         | 92.91                        | 92.18             |
| Điểm case đã chọn | 88                           | 92                |
| Số model calls    | 57                           | 58                |
| Token (agent)     | 93,527 input / 18,071 output | 107,813 / 19,506  |
| Token (Judge)     | 66,250 input / 11,072 output | 74,683 / 10,657   |

### Lỗi đã giải quyết: 
- H6 -> giờ đã trả về đúng số updates
- H3 -> giờ đã trả về "Không tìm thấy... trong 50 incident gần nhất" thay vì trả lời là "Không có"
- A2: update timestamp giờ đã rõ hơn 
### Lỗi còn lại hoặc mới xuất hiện:
- A5 (93->78): một lỗi xuất hiện do thay đổi. Phần thêm trong `SYSTEM_PROMPT` về `created_at` dẫn đến agent mở 5 incidents để đọc ngày và dẫn đến việc cho rằng không có incidents nào trong 2025 dựa trên 50 incidents gần nhất dù nó mới chỉ check 5/23.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
- Khi em dùng model Sonnet 5.5 của Claude, đã đạt giới hạn 1024 token như trong adapter có ghi, cho một lần chạy run.py hỏi về tình hình của github. Vậy nên, em đã thêm dòng vào SYSTEM_PROMPT bảo nó hãy gói gọn hơn.
- Model mà phải chạy và trả bằng Tiếng Việt tốn token và đạt rất gần đến giới hạn 1024 token.
- Em sử dụng test E2 làm demo test, lần chạy đầu tiên cho 94, lần thứ 2 (có edit system prompt để câu trả lời ngắn gọn hơn để token trong phạm vi 1024) thì trả về 93, lần chạy thứ 3 là lần chạy đủ 22 eval thì có điểm là 94 (94->93->94) vậy "Judge" vẫn có noise khi chấm điểm.
- Giới hạn về model: Judge đang là cùng một model với Agent (Claude Sonnet 5.5) nên có thể xảy ra tình trạng bias.
- Mỗi version chỉ đang chạy eval một lần để tránh bị mất chi phí nhiều nhưng không thể kiểm soát là kết quả đang ổn định
- Dữ liệu data lấy từ ngày 9/11/2026, chưa phản ánh dữ liệu thực tế, và giới hạn 50 incident là trong khoảng 10 tuần của GitHub nhưng chỉ trong khoảng 2 tuần của Cloudflare.