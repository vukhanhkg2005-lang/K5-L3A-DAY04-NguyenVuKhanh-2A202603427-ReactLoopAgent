# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Vũ Khanh
MSSV: 2A202603427

## 1. Case được chọn

- ID case: A1
- Câu hỏi của case: "Đối chiếu GitHub Actions và Cloudflare Spectrum: mỗi dịch vụ có incident liên quan nào trong dữ liệu, trạng thái và phạm vi của từng incident? Không suy luận chúng có cùng nguyên nhân; dẫn URL cho cả hai."
- Câu trả lời của agent ở lần 1: Agent không trả về được câu trả lời cuối cùng (`answer: ""`) do vòng lặp bị quá số bước tối đa (`status: "max_steps"`).
- Điểm ở lần chạy 1: 0/100 (LLM Judge chấm 0 điểm do trạng thái `max_steps` - hết ngân sách bước trước khi hoàn tất).

## 2. Bằng chứng trong trace

Đoạn trace từ `eval_before.json` cho thấy agent bị lặp qua 8 bước gọi công cụ liên tục mà không kịp đưa ra câu trả lời cuối cùng:

```json
{
  "id": "A1",
  "result": {
    "status": "max_steps",
    "answer": "",
    "trace": [
      {
        "event": "tool",
        "step": 1,
        "name": "get_status",
        "arguments": {"provider": "github", "component": "Actions"},
        "result": {"provider": "github", "overall": {"indicator": "none"}, ...}
      },
      {
        "event": "tool",
        "step": 2,
        "name": "get_status",
        "arguments": {"provider": "cloudflare", "component": "Spectrum"},
        "result": {"provider": "cloudflare", "incidents": [{"id": "c60k3g4vsoc1", ...}]}
      },
      ...
      {
        "event": "tool",
        "step": 8,
        "name": "get_incident_details",
        "arguments": {"provider": "cloudflare", "incident_id": "c60k3g4vsoc1"},
        "result": {"ok": true, "url": "https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1"}
      }
    ]
  },
  "grade": {
    "score": 0,
    "status": "scored",
    "reason": "Agent hết số bước trước khi hoàn tất."
  }
}
```

- Nguyên nhân: Trong các câu hỏi yêu cầu điều tra chéo nhiều nhà cung cấp (multi-provider), agent có xu hướng gọi tool thăm dò nhiều lần không cần thiết cho từng component trước khi truy vấn chi tiết incident, làm tiêu tốn hết ngân sách 8 bước (`max_steps=8`) trước khi model kịp tổng hợp câu trả lời cuối cùng kèm nguồn.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - Trong `agent.py`: Tối ưu hoá quy trình xử lý trong `run_agent` bằng cơ chế **Evidence Gate** và kiểm soát điều kiện dừng.
  - Khi đã thu thập đủ dữ liệu incident và URL nguồn từ cả hai phía nhà cung cấp (`observed_urls`), agent được định hướng tổng hợp câu trả lời ngay lập tức thay vì tiếp tục gọi tool thăm dò lặp lại.
  - Cơ chế kiểm tra nguồn: Kiểm tra trực tiếp sự hiện diện của URL incident trong câu trả lời cuối cùng; nếu model trả lời mà thiếu URL thì mới kích hoạt event `validation_missing_source` để yêu cầu bổ sung trong ngân sách bước còn lại.
- Vì sao chọn thay đổi này:
  - Giúp agent tiết kiệm số bước gọi tool (chỉ cần 4 bước thay vì 8 bước), tránh lỗi `max_steps`.
  - Đảm bảo câu trả lời có đầy đủ URL kiểm chứng cho cả hai nhà cung cấp, đáp ứng đúng yêu cầu của rubric về `evidence` và `completeness`.

## 4. So sánh hai lần chạy

Dữ liệu thực nghiệm được tổng hợp từ `eval_before.json` và `eval_after.json`:

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (score_0_100) | **94.55/100** | **99.00/100** |
| Điểm case đã chọn (A1) | **0/100** | **98/100** |
| Số model calls | 52 calls | 44 calls |
| Token tiêu thụ | ~38,500 tokens | ~32,200 tokens |

- Lỗi đã giải quyết: Case A1 được giải quyết triệt để, agent hoàn thành toàn bộ cuộc điều tra hai chiều chỉ trong 4 bước gọi tool và trả lời chi tiết kèm URL của cả GitHub và Cloudflare, nâng điểm case A1 từ 0 lên 98 điểm và kéo điểm tổng toàn bộ 22 case từ 94.55 lên 99.00/100.
- Lỗi còn lại: Lượng token tiêu thụ trong các prompt có dữ liệu dài (như case H6 có 40 distractors) vẫn tương đối lớn; cần tiếp tục tối ưu hóa việc chắt lọc context ở các phiên bản tiếp theo.

## 5. Giới hạn

- Giới hạn của điểm judge: LLM Judge là phương pháp ước lượng bán tự động, điểm số có phương sai nhỏ giữa các lần gọi khác nhau do tính ngẫu nhiên của mô hình ngôn ngữ lớn.
- Giới hạn của snapshot: Dữ liệu sự cố được đóng băng tại thời điểm snapshot (09/11/2026) với tối đa 50 incident gần nhất; không phản ánh trực tiếp trạng thái hạ tầng thực tế tại thời điểm truy vấn hiện tại.
- Giới hạn của việc chạy một lần: Việc đánh giá một lần trên 22 case là một lát cắt kiểm thử, chưa đủ để chứng minh độ ổn định tuyệt đối trong môi trường sản xuất đa dạng.
