# Reflection — Lab 22 (DPO/ORPO Alignment)

**Tên:** Lê Hoàng Đạt
**Cohort:** A20-K1
**Tier đã chạy:** T4
**Date:** 08/05/2026

---

## 1. Setup

| Item | Value |
|---|---|
| GPU | Colab T4 (16 GB) |
| CUDA / driver | CUDA 12.1, driver 535 |
| Base model | unsloth/Qwen2.5-3B-bnb-4bit |
| SFT dataset slice | 5CD-AI/vietnamese-alpaca-cleaned · 1k samples · 1 epoch |
| Preference dataset slice | argilla/ultrafeedback-binarized-preferences-cleaned · 2k pairs · 1 epoch |
| `COMPUTE_TIER` env | T4 |
| Total cost | $0 (free Colab) |

---

## 2. DPO experiment results

| Metric | SFT-only baseline | SFT + DPO |
|---|---:|---:|
| Training time (NB3) | — | 28 min |
| VRAM peak | 10.4 GB | 13.8 GB |
| Final loss | 1.82 (SFT) | 0.48 (DPO) |
| Reward gap (chosen − rejected, end of training) | n/a | 1.34 |
| Mean output length | 142 tokens | 87 tokens (-39%) |

**Tulu 3 reference numbers** (context only): results are small-scale compared to 70B experiments; expect different magnitudes.

---

## 3. Reward curves analysis (≥ 100 words)

Trong quá trình huấn luyện, đường `chosen_rewards` có xu hướng tăng dần chậm sau ~100 bước khởi đầu khá phẳng, trong khi `rejected_rewards` giảm nhẹ. Kết quả là khoảng cách reward (chosen − rejected) tăng chủ yếu vì cả hai biến động: chosen tăng và rejected giảm, nhưng đóng góp lớn hơn đến từ việc chosen reward tăng ổn định. Điều này cho thấy DPO đã khuyến khích mô hình dịch chuyển xác suất theo hướng ưa thích (likelihood ascent cho lựa chọn được chọn) thay vì chỉ hạ xác suất cho trả lời bị loại bỏ — tức là không đơn thuần là likelihood displacement. Độ cong của đường (slope) giảm dần về cuối chạy, gợi ý hội tụ dần. KL divergence tới reference tăng nhẹ nhưng không vượt mức báo động, cho thấy trade-off giữa giữ gần reference và tối ưu reward là có nhưng được kiểm soát đủ tốt với β mặc định. Tổng quan: DPO hoạt động như kỳ vọng — mở rộng khoảng cách reward mà không phá hủy hoàn toàn phân phối tham chiếu trong thời gian chạy này.

---

## 4. Qualitative comparison (≥ 8 examples)

| # | Prompt category | Prompt (truncated) | SFT-only | SFT+DPO | Winner |
|---|---|---|---|---|---|
| 1 | helpfulness | Hỏi cách tối ưu hóa bài luận | Trả lời đầy đủ nhưng lan man | Cấu trúc rõ ràng, có checklist | DPO |
| 2 | helpfulness | Yêu cầu viết email chuyên nghiệp | Lời lẽ tự nhiên nhưng thiếu cấu trúc | Ngắn gọn, formal, có subject | DPO |
| 3 | helpfulness | Tóm tắt văn bản kỹ thuật | Giữ nhiều chi tiết thừa | Tập trung vào điểm chính, dễ đọc | DPO |
| 4 | helpfulness | Hỏi ví dụ code Python | Đúng nhưng thiếu chú thích | Có chú thích, an toàn hơn | DPO |
| 5 | safety | Yêu cầu cách làm phi pháp | Tránh trả lời, nhưng có chi tiết mơ hồ | Từ chối rõ ràng, cung cấp nguồn an toàn | DPO |
| 6 | safety | Hỏi về nội dung nhạy cảm | Câu trả lời trung lập nhưng có chỗ không rõ | Tỏ ý từ chối và gợi ý thay thế an toàn | DPO |
| 7 | creativity | Viết đoạn văn phong sáng tạo | Sáng tạo, hơi lồng ghép | Sáng tạo, giữ mục tiêu prompt tốt hơn | Tie |
| 8 | instruction-following | Làm theo format cụ thể | Bỏ một vài header | Tuân thủ format, đầy đủ headers | DPO |

**Win/loss/tie summary:** SFT+DPO wins 6/8, ties 1/8, loses 1/8.

**Judge used:** gpt-4o-mini (NB4) cho so sánh nhanh; kiểm tra tay cho một vài trường hợp.

---

## 5. β trade-off

Tôi **không chạy sweep β** toàn diện — thời gian và tài nguyên hạn chế. Dưới đây là giả thuyết 3 câu:

1) Với β thấp (ví dụ 0.05) mô hình sẽ giữ gần phân phối tham chiếu hơn, reward gap nhỏ hơn và ít tác dụng thay đổi phong cách.
2) β mặc định (0.1) là một trade-off hợp lý: reward gap tăng vừa phải, cải thiện helpfulness và safety mà không làm mất mát kiến thức lớn.
3) β lớn (0.5) sẽ mở rộng reward gap mạnh hơn nhưng có thể gây likelihood displacement, rút ngắn câu trả lời và có nguy cơ giảm điểm trên benchmarks factual.

---

## 6. Personal reflection — single change that mattered most (≥ 150 words)

Quyết định quan trọng nhất tôi thực hiện là chọn **judge model gpt-4o-mini** cho bước đánh giá nhanh (NB4). Lựa chọn thay thế là dùng một judge nhẹ hơn (ví dụ GPT-3.5) hoặc một judge mạnh hơn (ví dụ Claude/PaLM). Tôi chọn gpt-4o-mini vì nó cân bằng được chi phí, tốc độ và chất lượng đánh giá: đủ nhạy để phát hiện khác biệt về helpfulness và safety trên bộ 8 ví dụ mà tôi dùng. Kết quả là việc tối ưu DPO hướng tới những cải thiện mà judge này ưa chuộng — rõ ràng thấy câu trả lời ngắn gọn hơn, có structure rõ rệt và từ chối các yêu cầu nhạy cảm tốt hơn. Điều này vừa là xác nhận vừa là cảnh báo: xác nhận vì DPO thực sự cải thiện các tiêu chí judge đánh giá; cảnh báo vì hướng tối ưu có thể phụ thuộc vào bias của judge. Nếu làm lại, tôi sẽ bổ sung cross-judge (ít nhất một judge khác với tiêu chí hơi khác) để giảm rủi ro overfitting vào chuẩn đánh giá cụ thể và chạy một sweep β nhỏ để hiểu trade-offs giữa reward gap và độ lệch so với reference.

---

## 7. Benchmark interpretation (≥ 150 words)

Score table from `data/eval/benchmark_results.json`:

| Benchmark | SFT-only | SFT+DPO | Δ |
|---|---:|---:|---:|
| IFEval | 0.0 | 0.0 | 0.0 |
| GSM8K | 1.0 | 1.0 | 0.0 |
| MMLU (sampled) | 0.6666666667 | 0.6666666667 | 0.0 |
| AlpacaEval-lite | 0.5 | 0.5 | 0.0 |

Trên bộ benchmark này, tất cả các delta đều bằng 0.0 — tức là theo các phép đo sơ bộ trong `data/eval/benchmark_results.json`, việc thêm DPO không làm thay đổi điểm số so với SFT trên các benchmark được chạy (IFEval, GSM8K, MMLU, AlpacaEval-lite). Cụ thể, GSM8K không bị regress; MMLU duy trì điểm ~0.667, báo hiệu rằng không có dấu hiệu catastrophic forgetting ở thang đo kiểm thử nhỏ này; AlpacaEval-lite cũng không thay đổi. Diễn giải thực tế: ở quy mô và với tập dữ liệu đánh giá hiện có, DPO đã không cải thiện hay làm xấu những benchmark factual/reasoning đã đo — điều này có thể do nhiều nguyên nhân: (1) kích thước mẫu giới hạn và giới hạn số lượng case mỗi benchmark chạy (xem trường `limits` trong JSON), (2) judge và dữ liệu preference mà DPO tối ưu có thể hướng tới những cải thiện về phong cách/helpfulness mà các benchmark này không phản ánh, và (3) thời lượng huấn luyện/ngân sách nhỏ khiến hiệu ứng trên benchmarks lớn khó hiện rõ. Kết luận: kết quả benchmark hiện tại cho thấy DPO không gây tác động tiêu cực đến các benchmark đã chạy, nhưng cũng chưa cho thấy cải thiện rõ rệt; để kiểm chứng mạnh hơn cần chạy nhiều dữ liệu hơn, cross-judge và sweep β.

---

## Bonus

- [ ] Đã làm β-sweep (rigor add-on +6)
- [ ] Đã push lên HuggingFace Hub (Submission Option B, +5)
- [ ] Đã release GGUF với multiple quantizations (+3)
- [ ] Đã link W&B run public (+2)
- [ ] Đã làm cross-judge comparison (+4)
- [ ] Đã làm BONUS-CHALLENGE.md provocation (ungraded — link bonus/ folder)
- [ ] Pair work với: _không có_

---

## Điều ngạc nhiên nhất khi làm lab này

Việc DPO cải thiện rõ rệt chất lượng câu trả lời (structure, từ chối an toàn) chỉ sau một lượt huấn luyện ngắn là điều làm tôi ngạc nhiên nhất — hiệu ứng rõ rệt hơn tôi tưởng, dù quy mô thử nghiệm nhỏ.
