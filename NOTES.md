# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Văn Minh  
MSSV: 2A202603771  
Lớp: K5-L3B

## 1. Case được chọn

- ID case: **H6** (`noisy_status`).
- Câu trả lời của agent, trích đầu answer: “Dưới đây là thông tin bổ sung về các URL nguồn cho incident đã kiểm tra:”. Agent nêu nguyên nhân của incident liên kết nhưng bỏ status snapshot và cảnh báo dữ liệu thiếu trong câu trả lời cuối.
- Điểm lần 1: **45/100** (factuality=10, completeness=15, evidence=15, boundaries=5).

## 2. Bằng chứng trong trace

Tóm tắt trace H6 trong `eval_before.json`:

```text
Step 1: get_status(github, actions) → Actions operational,
        captured_at=2026-11-09T03:50:47.065020+00:00.
Step 1: get_incident_details(h02n9572mx9y) → 3/4 updates;
        update resolved dẫn sang incident xr3myzj7ms51.
Step 3: get_incident_details(xr3myzj7ms51) → 3/9 updates;
        body resolved bị cắt (truncated=true), vẫn có nguyên nhân
        thay đổi cấu hình dịch vụ định tuyến sự kiện và feature flags.
```

Nguyên nhân: lỗi tổng hợp ở prompt và kiểm tra trong loop. Agent đã đọc cả hai incident nhưng final thiếu thông tin; gate chỉ kiểm tra URL nên vẫn cho qua. Dữ liệu bị giới hạn thật, nhưng phần đã nhận đủ để nêu nguyên nhân, không phải hoàn toàn thiếu bằng chứng.

## 3. Thay đổi đã thực hiện

- **Prompt trong `agent.py`:** thêm `INVESTIGATION_RULES`, yêu cầu giữ snapshot, phân biệt nguyên nhân/ảnh hưởng, nối incident gốc với bản tổng hợp và viết lại đầy đủ câu trả lời sau mỗi lần sửa.
- **Loop trong `run_agent`:** theo dõi incident liên kết, kiểm tra URL từng incident; bổ sung kiểm tra snapshot và tỷ lệ update đã đọc/tổng số. Thiếu thông tin thì nhắc sửa trong `max_steps`.
- **Context:** bỏ 40 `distractors` khỏi nội dung gửi model, giữ nguyên trong trace; thêm metadata về số update và body bị cắt.
- **Mục đích:** giảm bỏ sót bằng chứng và giới hạn dữ liệu của H6. Kiểm tra offline: **37/37 test chính và 3/3 test bonus pass**. Không sửa tools, fixtures, cases hoặc grading.

## 4. So sánh hai lần chạy

Hai report cùng cấu hình: OpenRouter / `openai/gpt-4o-mini` cho agent và judge, `max_steps=8`, dataset core, cùng snapshot.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 81.82/100 | 88.95/100 (+7.13) |
| Điểm H6 | 45/100 | 80/100 (+35) |
| Case được chấm | 22/22 | 22/22 |
| Model calls có usage | 56 | 58 |
| Agent total tokens | 91,714 | 135,070 |
| Chi phí agent + judge | 0.02580570 USD | 0.03240450 USD |

Số lượt gọi được đếm từ usage vì baseline không ghi event LLM; token trong bảng chỉ tính agent, chi phí gồm cả agent và judge.

- **Kết quả:** H6 đọc incident liên kết ngay bước 2, giữ timestamp, nguồn và tỷ lệ 3/4, 3/9 updates trong final. Trong 22 case, có 9 case tăng điểm, 3 giảm, 10 giữ nguyên; không có mechanical violations.
- **Lỗi còn lại:** H6 cảnh báo phần bị cắt chưa rõ; query `GitHub Actions` trả `components=[]`. A6 cần thêm 4 lượt sửa; H7 vẫn kết luận Cloudflare operational sau timeout. Tổng token và chi phí tăng, chưa chứng minh tiết kiệm chi phí.

## 5. Giới hạn

- **Judge:** điểm mang tính ước lượng và có thể biến động; `completed` hoặc không có mechanical violations không chứng minh mọi kết luận đều đúng.
- **Snapshot:** dữ liệu tháng 11/2026, chỉ tìm trong 50 incident gần nhất, status liệt kê tối đa 5 mục; chi tiết chỉ trả một phần updates, mỗi body tối đa 900 ký tự. Không phản ánh trạng thái realtime hoặc lịch sử đầy đủ.
- **Validation:** kiểm tra URL, timestamp và tỷ lệ update chưa kiểm chứng được ý nghĩa câu trả lời; lỗi kết luận sai khi tool timeout vẫn có thể lọt qua.
- **Độ ổn định:** một lần chạy mỗi phiên bản chưa đủ kết luận cải thiện ổn định; không ghép điểm các lượt chạy riêng H6 vào report toàn bộ.
