# I. Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Năng Thịnh
MSSV: 2A202604028

## 1. Case được chọn

- ID case: A6
- Câu trả lời của agent: Ở lần chạy 1, agent nhận ra incident `h02n9572mx9y` trỏ sang `xr3myzj7ms51` và đã gọi tool để đọc cả hai incident. Tuy nhiên, câu trả lời cuối chủ yếu giải thích rằng incident đầu tiên không cung cấp nguyên nhân, sau đó chỉ đưa hai đường dẫn tham khảo. Agent chưa tổng hợp cụ thể thời gian xảy ra sự cố, phạm vi dịch vụ bị ảnh hưởng và nguyên nhân được mô tả trong incident liên quan.
- Điểm ở lần chạy 1: 50/100

## 2. Bằng chứng trong trace
Trace lần chạy 1 (`eval_before.json`):

```text
Step 1:
Tool: get_incident_details
Arguments:
{
  "provider": "github",
  "incident_id": "h02n9572mx9y"
}

Result:
- Status: resolved
- Component: Actions
- Update mới nhất:
  "Please refer to the combined summary in this related incident:
   https://www.githubstatus.com/incidents/xr3myzj7ms51"

Step 2:
Tool: get_incident_details
Arguments:
{
  "provider": "github",
  "incident_id": "xr3myzj7ms51"
}

Result:
- Status: resolved
- Components: Webhooks, Actions, Pages
- Update chứa thông tin về thời gian,
  phạm vi ảnh hưởng và nguyên nhân.

Step 3:
Model finish: stop

Final answer:
Incident h02n9572mx9y chỉ trỏ tới tóm tắt
của incident xr3myzj7ms51 mà không cung cấp
chi tiết nguyên nhân.

Agent chỉ cung cấp hai đường dẫn tham khảo,
không tổng hợp các thông tin quan trọng
từ incident liên quan.
```

Nguyên nhân có thể xảy ra:

Agent đã thực hiện đúng việc gọi tool và truy xuất dữ liệu nhưng chưa tổng hợp đầy đủ thông tin từ kết quả trả về. System prompt baseline chủ yếu yêu cầu sử dụng tools, phân biệt incident hiện tại và lịch sử, phạm vi ảnh hưởng và dẫn nguồn; chưa có quy tắc đủ cụ thể để xử lý quan hệ giữa các incident.

Trace lần chạy 2 (`eval_after.json`):

```text
Step 1:
get_incident_details(h02n9572mx9y)

Step 2:
get_incident_details(xr3myzj7ms51)

Step 3:
Model finish: stop

Validation:
validation_missing_source
- https://stspg.io/7pwv28hvkmz8
- https://stspg.io/pjgptg3cfc6n

Step 4:
Model finish: stop

Final answer:
- Thời gian: 04/11/2026, 15:49–16:48 UTC
  và 17:20–17:55 UTC.
- Ảnh hưởng: Actions, Pages, Webhooks,
  Code Scanning và Copilot coding agent.
- Actions workflow bị trễ đến 25 phút.
- Pages deployments giảm còn khoảng 4%.
- Webhook deliveries giảm còn khoảng 58%.
- Nguyên nhân: thay đổi cấu hình trong
  dịch vụ internal event-routing, kết hợp
  với feature flags không được thiết lập đúng.
```

Kết quả lần chạy 2 cho thấy agent đã cải thiện đáng kể khả năng tổng hợp thông tin. Tuy nhiên, trace vẫn ghi nhận `validation_missing_source`, cho thấy cơ chế kiểm tra nguồn chưa hoàn toàn ổn định.


## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
    * Mở rộng SYSTEM_PROMPT trong `agent.py`: bổ sung quy tắc yêu cầu đọc incident liên quan trước khi kết luận về nguyên nhân, phạm vi và ảnh hưởng.

    * Yêu cầu agent phân biệt thông tin được xác nhận, suy luận và thông tin chưa được xác minh.

    * Bổ sung cơ chế `provider_from_url()` để phát hiện incident ID từ URL trong nội dung update.

    * Bổ sung `inspected_incident_ids` và `nudged_incidents` để theo dõi incident đã đọc và tránh yêu cầu đọc trùng.

    * Khi phát hiện incident liên quan, agent bổ sung yêu cầu vào conversation để LLM tiếp tục gọi `get_incident_details` trước khi đưa ra kết luận.

    * Bổ sung cơ chế kiểm tra URL nguồn trong câu trả lời cuối thông qua `validation_missing_source`.

- Vì sao chọn thay đổi này:

Lỗi của A6 nằm ở khả năng khai thác và tổng hợp kết quả từ nhiều lần gọi tool. Vì vậy tôi tập trung cải thiện quy trình điều tra của agent: phát hiện quan hệ giữa các incident, chủ động truy xuất dữ liệu liên quan và yêu cầu tổng hợp dựa trên bằng chứng thực tế. Cách tiếp cận này giúp agent không dừng lại ở việc tìm thấy dữ liệu mà còn phải sử dụng dữ liệu đó để trả lời đúng mục tiêu người dùng.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 77.73/100 | 85.91/100 |
| Điểm case A6 | 50/100 | 80/100 |
| Số model calls | 4 | 4 |
| Agent tokens | 4,649 | 6,094 |
| Judge tokens | 1,976 | 2,187 |
| Tổng tokens | 6,625 | 8,281 |

- Lỗi đã giải quyết:
    * Agent đã đọc và khai thác thông tin từ incident liên quan thay vì chỉ cung cấp đường dẫn.

    * Câu trả lời đã nêu được thời gian xảy ra sự cố.

    * Xác định được các dịch vụ bị ảnh hưởng và mức độ gián đoạn cụ thể.

    * Nêu được nguyên nhân được mô tả trong incident liên quan, thay vì kết luận không có thông tin.

    * Phân biệt được thông tin được xác nhận và thông tin chưa được xác nhận.

- Lỗi còn lại hoặc mới xuất hiện:
    * Trace lần chạy 2 vẫn ghi nhận `validation_missing_source`, cho thấy việc kiểm tra và đối chiếu URL giữa tool result và câu trả lời cuối còn hạn chế.

    * Nội dung incident liên quan có truncated: true, vì vậy agent chưa thể khẳng định đã đọc toàn bộ nội dung cập nhật.

    * Phần thông tin chưa được xác nhận chưa thực sự làm rõ giới hạn của bằng chứng, đặc biệt là những chi tiết không có trong snapshot.

    * Số token sử dụng tăng từ 4,649 lên 6,094 đối với agent, cho thấy chất lượng được cải thiện nhưng phải đánh đổi bằng lượng context và token lớn hơn.

## 5. Giới hạn

- Giới hạn của điểm judge: Điểm số được đánh giá bằng LLM-as-a-Judge nên có thể chịu ảnh hưởng từ cách diễn giải rubric, tính nhất quán và khả năng đánh giá của model. Điểm 80/100 không đồng nghĩa với câu trả lời hoàn toàn chính xác. Cần kiểm tra lại câu trả lời với trace và dữ liệu nguồn.

- Giới hạn của snapshot: Agent làm việc với dữ liệu snapshot được ghi nhận tại thời điểm 09/11/2026. Đây không phải dữ liệu thời gian thực. Ngoài ra, incident có thể chứa nhiều update nhưng chỉ một phần được trả về do giới hạn pagination hoặc truncation. Vì vậy, kết luận chỉ có giá trị trong phạm vi dữ liệu đã truy xuất.

- Giới hạn việc chỉ chạy một lần mỗi phiên bản: Kết quả có thể chịu ảnh hưởng từ tính không xác định của LLM, do đó chưa đủ cơ sở để khẳng định mức cải thiện là ổn định hoặc có ý nghĩa thống kê. Cần chạy nhiều lần với cùng cấu hình và so sánh điểm trung bình, độ lệch chuẩn cũng như chi phí.

- Kết luận: Việc cải tiến giúp agent chuyển từ khả năng truy xuất incident sang khả năng điều tra và tổng hợp thông tin từ các incident có liên quan. Điểm tổng tăng từ 77.73 lên 85.91 và case A6 tăng từ 50 lên 80. Tuy nhiên, vẫn cần cải thiện cơ chế kiểm chứng nguồn, xử lý dữ liệu bị cắt và đánh giá độ ổn định qua nhiều lần chạy.

# II. Ghi chú (cho phần Bonus):

**Kết quả cuối:** `eval_bonus.json` — 10/10 case được chấm, `score_0_100 = 77.7`, `bonus_points_0_15 = 11.65`. Provider model: `openai` / `gpt-4o-mini`, `--max-steps 16`, judge dùng cùng model với agent.

| Case | Điểm |
|---|---|
| B1 | 50 |
| B2 | 82 | 
| B3 | 75 | 
| B4 | 100 |
| B5 | 90 |
| B6 | 100 |
| B7 | 50 |
| B8 | 85 |
| B9 | 95 |
| B10 | 50 |

## 1. Retry và rate limit

Agent giới hạn tối đa 3 lần gọi cho cùng một `(tool, provider)` khi gặp lỗi tạm thời. Từ lần thứ 4, agent không gọi `tools.execute()` nữa mà trả thẳng `RETRY_LIMIT_REACHED` cho model — tránh gọi API liên tục khi đã biết chắc sẽ lỗi. Các lần bị chặn này được ghi trace với `event: "tool_blocked"` thay vì `"tool"`, để không bị tính trùng vào giới hạn `max_calls` của case. Nếu tool trả `retry_after`/`retry_after_seconds`, agent chờ `min(giá trị, 5s)` trước khi để model thử lại, tránh gọi dồn dập.

## 2. Pagination

Tool trả `next_offset` khi còn update chưa đọc. Agent gắn ghi chú (`_agent_note`) vào mỗi kết quả có `next_offset != null`, nhắc model đọc tiếp nếu câu hỏi cần, hoặc phải nói rõ mới đọc một phần nếu không đọc hết.

**B2** là bằng chứng rõ nhất: incident GitHub `93njd414rpbx` có 13 update, agent đọc đủ 7 trang (`offset` 0→12, `limit=2/3`) cho tới khi `next_offset = null` mới dừng, rồi mới trả lời — đúng yêu cầu không tuyên bố đã xem hết timeline khi còn trang chưa đọc. Điểm 82 (giảm so với lần chạy thử riêng lẻ là 100) — theo `grade.reason`, judge trừ điểm vì câu trả lời diễn đạt chưa đủ nhấn mạnh rằng incident đã resolved, không phải lỗi về đọc trang.

Agent cũng tự phát hiện incident liên quan: khi một update chứa link dạng `.../incidents/<id>`, agent trích đúng provider từ host trong URL (không gán nhầm provider của tool vừa gọi) và chèn một lượt nhắc model đọc incident liên quan đó trước khi kết luận nguyên nhân — đây chính là cách B2 tự tìm ra và đọc tiếp incident GitHub từ update của Vercel.

## 3. Context

Scenario `noisy` (B6) trả thêm 160 mục `distractors` giả lập vào `get_vercel_status`. Agent rút gọn mọi list dài hơn 20 phần tử xuống còn 3 phần tử đầu + một dòng `{"_omitted": "N mục bị lược bỏ..."}` trước khi đưa vào message gửi model và vào trace — giảm kích thước tool message từ ~18.7 KB xuống ~2 KB. Nhờ đó B6 đạt 100 điểm, agent bỏ qua đúng dữ liệu nhiễu và vẫn gọi `get_incident_details` cho cả hai bên trước khi kết luận.

Rule trong prompt cũng nói rõ: khi câu hỏi cần đối chiếu/nêu phạm vi một incident, `get_status` (chỉ có id/tên) không đủ — bắt buộc gọi `get_incident_details`. Trước khi thêm rule này, agent từng bỏ qua bước đó và bị `score_cap = 50` do thiếu tool bắt buộc; sau khi thêm, B6 gọi đủ và hết bị chặn điểm.

## 4. Giới hạn còn lại

- Model hay bỏ sót narration: Dữ liệu agent thu thập được hầu hết đúng (đối chiếu trace vs answer ở B1, B10 cho thấy số liệu chính xác), nhưng câu trả lời cuối cùng của `gpt-4o-mini`đôi khi không tường thuật đủ ("đã retry bao nhiêu lần", "incident đã resolved") hoặc nhầm provider trong câu kết luận (B10 nhầm Cloudflare/Vercel). Agent không có cách nào ép model diễn đạt đúng 100% ngoài việc nhắc trong prompt/nudge — đây là giới hạn của model, không sửa được chỉ bằng code agent.
- Judge không ổn định: B1 bị chấm 50 dù `answer` đối chiếu tay với snapshot là đúng và đầy đủ cho cả 5 provider (Cloudflare "Gián đoạn hệ thống phần nào" ứng với `major`, 4 bên còn lại `none`, đúng timestamp/nguồn từng bên). B2 giảm từ 100 (chạy case đơn lẻ) xuống 82 (chạy cả bộ) dù trace không đổi — vì judge cùng model với agent (`gpt-4o-mini`) nên điểm có thể dao động giữa các lần chấm.
- Regex injection chỉ bắt mẫu phổ biến: `INJECTION_RE` nhận diện các cụm tiếng Anh kiểu "ignore...instructions", "system override" — không bắt được injection diễn đạt khác đi hoặc bằng ngôn ngữ khác. Đây là lớp phòng thủ phụ; lớp chính vẫn là rule tool output là dữ liệu không đáng tin trong system prompt.
- Trần retry 3 lần là cố định theo yêu cầu đề bài, không tối ưu theo backoff/jitter thực tế, phù hợp cho mục đích so sánh giữa các lần chạy eval hơn là dùng ở production.
- Một lần chạy không chứng minh độ ổn định — B1/B2/B10 cho thấy điểm có thể lệch 15–20 điểm giữa các lần chạy dù hành vi agent (trace) không đổi, hoàn toàn do biến thiên của model tạo câu trả lời và của judge.
- Chi phí: toàn bộ 10 case bonus dùng 36 lượt gọi model (agent + judge), tổng khoảng 104.000 token với `gpt-4o-mini`.

## 5. Kết luận

Hai lỗi thật sự nằm ở phía agent (không ép retry trước khi kết luận ở B10/B3; đếm nhầm số lần
gọi tool vào `max_calls` khi lần gọi đã bị chặn bởi retry-limit ở B4) đã được xác định bằng
cách đối chiếu `mechanical.violations` và trace, rồi sửa trực tiếp trong `agent.py`. Các điểm
thấp còn lại (B1, B7, B9) đối chiếu tay cho thấy là do model diễn đạt chưa chặt hoặc do judge
không ổn định, không phải lỗi logic của agent.
