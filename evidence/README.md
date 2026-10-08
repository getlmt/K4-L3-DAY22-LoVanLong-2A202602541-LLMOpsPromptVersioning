# Evidence — Day 22: LangSmith + Prompt Versioning

**Học viên:** Lò Văn Long — 2A202602541  
**LangSmith project:** `day22-lab` · **Prompt Hub:** `lo-van-long-rag-prompt-v1`, `lo-van-long-rag-prompt-v2`  
**Model:** `gpt-4o-mini` (LLM + RAGAS evaluator), embeddings `text-embedding-3-small`, FAISS 107 chunks (500/50), k = 3

## Danh sách evidence

| File | Nội dung |
|---|---|
| `01_langsmith_traces.png` | Project `day22-lab` với 50 traces `rag-query` (Nhiệm vụ 1) |
| `02_prompt_hub.png` | 2 prompt V1/V2 trên Prompt Hub |
| `02_ab_routing_log.txt` | Log push/pull từ Hub + 50 câu có nhãn `[prompt-v1]`/`[prompt-v2]`, routing V1=19 / V2=31 |
| `03_ragas_scores.png` | Bảng so sánh V1 vs V2 trên terminal |
| `03_ragas_report.json` | Bản sao `data/ragas_report.json` |
| `04_pii_demo_log.txt`, `04_json_demo_log.txt` | Log 6 test case PII + 5 test case JSON (một lần chạy, ghi ra cả 2 file) |

## Hai phiên bản prompt

- **V1 — ngắn gọn:** trả lời trực tiếp trong 2–4 câu, chỉ dùng thông tin trong context, không đủ thông tin thì nói không tìm thấy.
- **V2 — chuyên gia, có cấu trúc:** xác định facts liên quan, viết 3–5 câu (câu đầu nêu ý chính, các câu sau giải thích chi tiết), không suy đoán ngoài context.

## Kết quả RAGAS (50 cặp QA × 2 phiên bản)

| Metric | V1 | V2 | Chênh lệch |
|---|---|---|---|
| faithfulness | **0.9711** | 0.9188 | V1 +0.052 |
| answer_relevancy | **0.9105** | 0.8895 | V1 +0.021 |
| context_recall | 1.0000 | 1.0000 | bằng nhau |
| context_precision | 0.9417 | 0.9450 | ≈ bằng nhau |

Cả hai phiên bản đều đạt faithfulness ≥ 0.9.

## Phân tích: vì sao V1 cao hơn V2

Số liệu lấy từ trace của lần chạy RAGAS trên LangSmith (bước tách câu trả lời thành các "claim" và kiểm tra từng claim với context):

| | V1 | V2 |
|---|---|---|
| Độ dài trung bình câu trả lời | 44.8 từ, 2.1 câu | 78.3 từ, 3.6 câu |
| Số claim trung bình / câu trả lời | 6.0 | 8.8 |
| Claim không được context hỗ trợ | 8 / 298 | 37 / 441 |
| Số câu có faithfulness < 1.0 | 6 / 50 | 21 / 50 |

1. **faithfulness (V1 thắng):** V2 yêu cầu "các câu sau giải thích chi tiết", nên model viết dài hơn và thêm các câu diễn giải, khái quát mà context không nói thẳng. Ví dụ với câu hỏi về bias-variance, V2 thêm *"This balance allows for better generalization to unseen data"*; context không nhắc tới "generalization" nên claim này bị chấm là không được hỗ trợ. Câu trả lời càng dài thì càng nhiều claim, và tỉ lệ claim "suy ra thêm" càng cao. V1 bị giới hạn 2–4 câu và "trả lời trực tiếp" nên gần như chỉ nhắc lại nội dung context.
2. **answer_relevancy (V1 thắng nhẹ):** chỉ số này đánh giá câu trả lời có bám đúng trọng tâm câu hỏi không. Câu trả lời ngắn, đi thẳng vào ý của V1 bám câu hỏi sát hơn. Phần giải thích mở rộng của V2 làm loãng trọng tâm.
3. **context_recall và context_precision (gần như bằng nhau):** hai chỉ số này chỉ dùng câu hỏi, context đã truy xuất và đáp án chuẩn, **không dùng câu trả lời** (đã kiểm tra input của metric trong trace). Hai phiên bản dùng chung retriever (FAISS, k = 3) nên context giống hệt nhau. Chênh lệch 0.003 ở context_precision là dao động của LLM chấm điểm, không do prompt. Cột "Winner" trên terminal ghi `← V2` cho context_recall dù hai bên bằng nhau (1.0) vì code in bảng chọn V2 khi không có bên nào lớn hơn.

**Kết luận:** với hệ thống RAG cần bám sát tài liệu, **V1 phù hợp hơn**: trung thực hơn và đúng trọng tâm hơn. V2 cho câu trả lời đầy đủ hơn nhưng dễ "giải thích thêm" ngoài context. Nếu muốn giữ phong cách V2, nên ràng buộc chặt hơn, ví dụ yêu cầu mỗi câu giải thích phải dựa trên một câu cụ thể trong context.

## Ghi chú về lần chạy RAGAS

Lần chạy đầu tiên của Nhiệm vụ 3 gặp 3 lỗi kết nối OpenAI (`Connection error`) trong lúc chấm V2, nên faithfulness và context_precision của V2 ra `NaN`. Nhiệm vụ 3 được chạy lại với code giữ nguyên; `03_ragas_scores.png` và `03_ragas_report.json` là kết quả của lần chạy lại này.
