# Ghi chú phân tích · Day 04

Họ và tên: Phạm Trường Chinh
MSSV: 2A202603426

## 1. Case được chọn

- ID case: E4
- Câu trả lời của agent: Trích xuất các component bị ảnh hưởng gồm Webhooks, Actions, Pages; xác nhận Actions có liên quan và trạng thái sự cố đã resolved.
- Điểm ở lần chạy 1: 100/100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "incident_id": "xr3myzj7ms51",
      "provider": "github"
    },
    "result": {
      "ok": true,
      "id": "xr3myzj7ms51",
      "url": "https://stspg.io/pjgptg3cfc6n",
      "components": [
        "Webhooks",
        "Actions",
        "Pages"
      ]
    }
  },
  {
    "event": "validation_missing_source",
    "step": 2,
    "message": "Thiếu URL nguồn incident trong câu trả lời"
  }
]
```

- **Nguyên nhân có thể xảy ra**: Ở step 2, câu trả lời đầu tiên của model đưa ra nội dung chi tiết nhưng quên dẫn kèm đường link URL của incident. Cơ chế Evidence Gate phát hiện thiếu nguồn và gửi prompt nhắc nhở: `"Vui lòng cung cấp link URL incident làm bằng chứng nguồn trong câu trả lời."`. Tuy nhiên, model hiểu câu nhắc này là "chỉ gửi riêng URL" thay vì trả lời lại toàn bộ câu hỏi kèm URL, dẫn đến câu trả lời cuối cùng ở step 3 chỉ có link URL mà mất đi các thông tin component, khiến judge chấm điểm completeness thấp.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: 
  - Khởi tạo tập hợp `observed_urls` trong `run_agent` tại file `agent.py`.
  - Khi tool `get_incident_details` trả về thành công (`result.get('ok')` và `result.get('url')`), tự động lưu URL vào `observed_urls`.
  - Tại bước dừng (`finish == 'stop'`), kiểm tra nếu `observed_urls` có link nhưng câu trả lời chưa chứa link nguồn thì ghi nhận event `validation_missing_source` vào `trace` và nhắc model sửa đổi trong ngân sách `max_steps`.
- Vì sao chọn thay đổi này:
  - Đảm bảo tính xác thực thông tin (Evidence Gate), ngăn chặn agent đưa ra kết luận hoặc trích dẫn thông tin mà không dẫn link nguồn kiểm chứng, đồng thời thỏa mãn contract của `tests/test_validation.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 99.77 / 100 | 93.59 / 100 |
| Điểm case đã chọn (E4) | 100 / 100 | 40 / 100 |
| Số model calls | 54 calls | 58 calls |
| Token tiêu thụ | 62,147 tokens (49,876 prompt / 12,271 completion) | 72,497 tokens (59,520 prompt / 12,977 completion) |

- Lỗi đã giải quyết: Đảm bảo 100% các câu trả lời liên quan đến incident đều được kiểm tra bắt buộc phải có link nguồn xác thực trước khi kết thúc, vượt qua 37/37 unit tests.
- Lỗi còn lại hoặc mới xuất hiện: Khi được yêu cầu bổ sung link nguồn, model có xu hướng chỉ trả về duy nhất link URL mà quên giữ lại nội dung phân tích ban đầu, dẫn tới việc bị judge trừ điểm ở tiêu chí `completeness` (như ở case E3, E4, A7).

## 5. Giới hạn

- **Giới hạn của LLM Judge**: Đánh giá bằng LLM-as-a-judge mang tính xấp xỉ, phụ thuộc nhiều vào cách model judge hiểu prompt và có thể khắt khe khi cấu trúc câu trả lời của model thay đổi.
- **Giới hạn của Snapshot**: Dữ liệu status tĩnh chỉ chứa tối đa 50 incident gần nhất tại thời điểm chụp, không phản ánh được toàn bộ sự cố thời gian thực hoặc các sự cố lịch sử xa hơn.
- **Giới hạn của việc chỉ chạy một lần**: Độ ngẫu nhiên (sampling temperature) của LLM khiến kết quả giữa các lần chạy có độ dao động; một lần chạy không phản ánh hết độ ổn định tổng thể của agent.
