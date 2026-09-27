# Báo cáo khả thi: EchoSight làm giá đỡ cho KUP

Đánh giá phục vụ bản redesign v6 của đề cương *KG-VQA Faithful Attribution Rebuild* · 21/09/2026

- **Đối chiếu với:** đề cương v5 (16/09/2026)
- **Nguồn:** README, `model/answer_generator.py`, `model/retriever.py`, `test/test_reranker.py`, `test/test_answer_generator.py`, `utils/evaluation_utils.py`, `utils/test_utils.py`, `dataset/dataset.py`, `requirements.txt`, và 30 issue trên GitHub `Go2Heart/EchoSight`
- **Repo đã clone tại:** `New plan/EchoSight` (commit `ac11c1a`, MIT License)

---

## 1. Kết luận

**Khả thi, với điều kiện đổi cách nhìn về EchoSight.** Trong đề cương v5, EchoSight được mô tả là "chạy inference được ngay, không cần huấn luyện". Thực tế:

- Phần đề tài *thực sự cần* — giao diện tri thức → generator — trong EchoSight chỉ là **một hàm ~30 dòng** (`llm_answering`), rất dễ chặn và thay thế.
- Phần *tốn công nhất* (retrieval: tải ảnh, FAISS index, reranker Q-Former) chỉ cần chạy **đúng một lần** để sinh ra file JSON chứa K, sau đó toàn bộ KUP không cần đến nó nữa.
- Có **3 lệch pha nghiêm trọng** giữa thiết kế v5 và code thật, trong đó lệch pha số 1 (generator text-only) làm vô hiệu một phần M1/E1 nếu không sửa.
- Script standalone VQA của EchoSight **crash ngay khi chạy**, và các issue #12/#18 cho thấy tái lập Recall của retrieval đã thất bại với người khác — rủi ro tuần 1 mà v5 xếp "Cao" là đúng.

Khuyến nghị cho v6: giữ EchoSight, nhưng **định vị lại thành "nguồn cung cấp K đã cache + hai LLM baseline"**, và viết harness KUP riêng thay vì mở rộng code của họ.

---

## 2. Những điểm khớp với đề tài

| Yêu cầu trong v5 | Có trong EchoSight | Vị trí trong code |
|---|---|---|
| Giao diện K → generator tường minh, model-agnostic | ✅ Prompt là chuỗi phẳng: `"Context: " + section + "\nQuestion: " + q + "\nThe answer is:"` | `model/answer_generator.py:165` |
| Hai model đúng bộ Structural Attention Tax | ✅ Mistral-7B-Instruct-v0.2 và LLaMA-3-8B-Instruct, load bằng HuggingFace thuần, không wrapper lạ | `MistralAnswerGenerator`, `LLaMA3AnswerGenerator` |
| C5 — K đóng băng sau retrieval | ✅ `--save_result` lưu `reranked_sections[:10]` và `reranked_entries[:20]` theo `data_id`; `--resume_from` nạp lại mà không cần retriever | `test/test_reranker.py:76-80`, `:204-209` |
| Đơn vị tri thức có ranh giới rõ (D1) | ✅ Mỗi section là một chuỗi `# Wiki Article: T\n## Section Title: S\n<text>` | `reconstruct_wiki_sections` |
| Gold ở mức section (D5, E-VQA) | ✅ Cột `evidence_section_id` trong CSV, code training đã dùng | `dataset/dataset.py:115` |
| Thứ hạng của reranker cho từng unit (RQ3 / C3) | ✅ Thứ tự trong `reranked_sections` chính là rank sau rerank; điểm retrieval gốc cũng được lưu | `retrieval_similarities` |
| Baseline "chỉ tiêu đề" 29.4% | ✅ Có sẵn nhánh prompt `Entity name: <title>` | `model/answer_generator.py:243` |
| Baseline "không retrieval" 21.0% | ✅ Nhánh prompt vanilla `Question: q` | `model/answer_generator.py:253` |
| Codebase đủ nhẹ để sửa | ✅ ~2.300 dòng ngoài `lavis/`; `lavis/` 3.8 MB chỉ phục vụ reranker | — |
| KB có cấu trúc section cho cả hai dataset | ✅ KB E-VQA 2M trang; KB InfoSeek 100K trang, cùng format `{title, url, section_titles, section_texts, ...}` | `WikipediaKnowledgeBaseEntry`, `model/retriever.py:184` |

---

## 3. Ba lệch pha nghiêm trọng giữa v5 và code

### 3.1 Generator là text-only — ảnh không bao giờ đi tới LLM

`llm_answering()` chỉ nhận `question` và `entry_section`. Ảnh chỉ được dùng ở retriever (EVA-CLIP) và reranker (Q-Former). Không có nhánh nào đưa pixel hay caption vào prompt của Mistral/LLaMA3.

**Hệ quả với v5:**

| Thành phần v5 | Tình trạng |
|---|---|
| M1 · can thiệp "ảnh xám", đại lượng VN | **Không tồn tại ở tầng generator.** Ảnh xám chỉ có thể tác động qua retrieval → sinh K khác → vi phạm C5 |
| E1 · thiết kế 2×2 `{K, ảnh} × {giữ, bỏ}` | Chỉ còn trục K; trục ảnh sập |
| H1 · "Nguồn bị can thiệp: {K, ảnh}" | Phải viết lại |
| Bộ lọc leakage "không ảnh, không K mà vẫn đúng" | Vẫn làm được — chính là prompt vanilla |
| D4 · tách nguồn K / prior / ảnh | Chỉ tách được K / prior |

**Mặt tích cực:** sau khi cache K, toàn bộ KUP **không cần ảnh**. Không phải tải iNaturalist 2021, GLDv2, OVEN (hàng trăm GB) nếu có được file retrieval result — hoặc chỉ tải đúng tập ảnh của 300–600 instance đã chọn.

**Hai phương án cho v6:**

| Phương án | Được | Mất |
|---|---|---|
| **(a) Bỏ VN, thu RQ1 về "K vs prior"** — khuyến nghị | Giữ nguyên hai model trùng Structural Attention Tax; không thêm rủi ro; nhất quán với "Không đo chất lượng retrieval" đã có ở mục Không làm | Mất trục ảnh trong D4; phải khai ở Scope và Threats |
| (b) Thêm MLLM (LLaVA-1.6 / Qwen2-VL / Idefics2) | Có VN thật; mở được Future Work "phân rã hiệu ứng của ảnh" | Mất luận điểm "cùng model với công trình đối thoại"; thêm một model phải tái lập; tăng ngân sách gọi model |

### 3.2 Decoding đang sampling, không greedy

| Model | Cấu hình trong code | Vị trí |
|---|---|---|
| Mistral | `do_sample=True, top_p=0.9, temperature=0.9` | `model/answer_generator.py:181-188` |
| LLaMA3 | `do_sample=True, top_p=0.9, temperature=0.6` | `model/answer_generator.py:266-272` |

KS định nghĩa là "đáp án đổi khi bỏ K". Với sampling, đáp án đổi ngay cả khi không bỏ gì → **KS dương giả**. Bắt buộc `do_sample=False`.

**Hệ quả phụ:** con số tái lập tuần 1 **sẽ không khớp 41.8%** của paper vì paper chạy có sampling. Cổng tuần 1 phải đổi tiêu chí (xem §6).

### 3.3 Không có log-probability — chỉ có text sinh ra

`llm_answering()` gọi `generate()` rồi decode. Không có hàm nào trả về `log p(a* | q, do(K,v))`. Trong khi đó:

- Surrogate thưa của ContextCite cần φ(v) = log-prob của a* dưới mỗi mask v
- Bản mềm ΔlogP của KS cần log-prob
- LDS cần φ(v) trên tập held-out

**Phải viết thêm** hàm *teacher-forced scoring*: nối a* vào sau prompt, một forward pass, cộng log-prob các token của a*. Điểm quan trọng cho ngân sách: scoring **rẻ hơn generate nhiều lần** (một forward, không autoregressive) — M2/M3 nên thiết kế quanh scoring, chỉ generate khi cần Y(v) dạng nhãn cứng.

---

## 4. Thiếu sót cần bổ sung, xếp theo mức chặn

| # | Thiếu | Ảnh hưởng tới đề tài | Việc phải làm | Ước lượng |
|---|---|---|---|---|
| 4 | Generator chỉ nhận **1 section** (`reranked_sections[0]`) | K trong v5 là tập n unit gồm cả distractor. Prompt đa-unit là **thiết kế của luận văn**, không phải của EchoSight → phải khai ở Threats | Viết template đa-unit có đánh số. Chú ý `_adjust_prompt_length` cắt cứng 4096 token **bằng tokenizer Mistral kể cả khi chạy LLaMA3** (`answer_generator.py:97`) — phải thay bằng tokenizer đúng model và cắt theo unit, không cắt giữa unit | 1 ngày |
| 5 | Không có cơ chế **self-citation / `<evidence>`** | Cite-PN, SAA, AH (E4) không có nguồn dữ liệu | Template yêu cầu model trích số unit; parser đầu ra; quy tắc khớp unit ↔ gold | 1 ngày |
| 6 | **InfoSeek không có triple gold** trong pipeline | E3/M0 — đóng góp chính — phụ thuộc `(subject, relation, object)` | CSV InfoSeek của EchoSight là bản "chuyển sang format E-VQA". Bản InfoSeek gốc chỉ phát hành entity QID + answer; relation nằm trong file type riêng. Issue #30 hỏi `infoseek_val_type.jsonl` **đang mở, chưa được trả lời**. Gần như chắc chắn phải map ngược qua Wikidata bằng (QID, answer) → tuần dự phòng trong v5 là **bắt buộc**, không phải tuỳ chọn | 1 tuần |
| 7 | **Evaluation InfoSeek chưa tồn tại** | `evaluate_example` ném `ValueError` khi `question_type == "infoseek"` vì `_QUESTION_TYPES = ['templated','automatic','multi_answer','2_hop']` (`evaluation_utils.py:395`) | Tự viết eval InfoSeek theo bản gốc (exact match cho string, khoảng dung sai cho số). Cân nhắc bỏ BEM (cần TF 2.16 + TF-Hub) và dùng EM/F1 thống nhất cho cả hai dataset | 0.5 ngày |
| 8 | **Tái lập retrieval có tiền lệ thất bại** | Issue #12, #18: R@1 8.3% thay vì 36.5% dù md5 FAISS đúng (`59d386530a52838b5cfa0b647cf92e08`); #26: tác giả từ chối cung cấp retrieval result JSON "do storage management" | Rủi ro số một, đúng như v5 xếp. Phương án B (dùng section gold + distractor cùng bài làm K, mất trục C3 với rank reranker) phải sẵn sàng từ ngày 1, không đợi cuối tuần 1 | — |
| 9 | `test/test_answer_generator.py` **crash ngay** | `args.vqa_results` không tồn tại (dòng 57); `retrieval_results["reranked_sections"][0]` sai cấu trúc vì JSON keyed theo `data_id` (dòng 33) | Script này phải sửa trước khi dùng. README "chạy inference được ngay" **không đúng** với script standalone | 0.5 ngày |
| 10 | Không có **batch inference** | README ghi "Releasing Soon", chưa bao giờ phát hành. Vòng lặp gọi `generate()` từng mẫu | 168K lần gọi theo vòng lặp đơn là không khả thi. Viết batched scorer (left-padding) cho phần log-prob; dùng vLLM cho phần generate nếu cần | 1–2 ngày |
| 11 | Môi trường **ghim cứng và dễ vỡ** | `transformers==4.37.2`, `torch==2.3.1`, `faiss-gpu==1.7.2` (wheel pip cũ, thường hỏng với CUDA 12.4+/Python 3.11+), `tensorflow==2.16.1` + `tensorflow-text` chỉ để chạy BEM; đường dẫn hard-code `/remote-home/share/...` trong `test_reranker.py:59,65` và `/PATH/TO/...` trong `utils/test_utils.py` | Tách hai env: **env-retrieval** (faiss + lavis + transformers 4.37, chạy một lần) và **env-kup** (torch + transformers mới, không TF) | 1 ngày |
| 12 | Hậu xử lý output bẩn | Mistral cắt `[:-4]` để bỏ `</s>`; LLaMA3 không cắt `<|eot_id|>` | Chuẩn hoá answer (strip special token, lowercase, bỏ dấu câu) trước khi so KS — nếu không, "đáp án đổi" sẽ bị nhiễu bởi token điều khiển | 0.5 ngày |
| 13 | Ảnh InfoSeek không đầy đủ | Issue #29: AToMiC-Images không chứa toàn bộ ảnh InfoSeek | Chỉ ảnh hưởng khối retrieval một lần; chọn instance có ảnh trước khi chạy | — |

**Tổng công bổ sung ước tính:** ~2 tuần người, trong đó ~1 tuần cho triple InfoSeek (mục 6). Con số này chưa nằm trong kế hoạch 6 tuần của v5.

---

## 5. Kiến trúc triển khai đề xuất cho v6

Tách hai khối rõ ràng, không dùng chung env, không dùng chung máy nếu cần.

```
┌─────────────────────────────── KHỐI A · chạy một lần ───────────────────────────────┐
│  EchoSight nguyên bản (env-retrieval)                                                │
│  ảnh + CSV ──▶ EVA-CLIP retrieve ──▶ Q-Former rerank ──▶ test_reranker.py            │
│                                                          --save_result               │
│                                                                │                     │
│                                                                ▼                     │
│                                      retrieval_result.json {data_id: {               │
│                                          reranked_sections[:10],                     │
│                                          reranked_entries[:20],                      │
│                                          retrieval_similarities }}                   │
└──────────────────────────────────────────────┬───────────────────────────────────────┘
                                               │  đóng băng (C5)
┌──────────────────────────────────────────────▼───────────────────────────────────────┐
│  KUP harness (env-kup) — viết mới, không sửa code EchoSight                          │
│                                                                                      │
│  M0 Renderer   : {section, câu đơn, triple thô, triple verbalize} + distractor        │
│  Prompt builder: template đa-unit có đánh số, cắt theo unit, tokenizer đúng model     │
│  Scorer        : teacher-forced log p(a* | q, do(K,v)), batched, greedy khi generate  │
│  M1 Gate       : {K, ∅} → KS, leakage    (không có VN — xem §3.1)                     │
│  M2 Sampler    : v ~ Bernoulli(p), Fid⁺(random-k), Fid_α                             │
│  M3 Estimator  : surrogate thưa, PS/PN/PNS_lb, LDS                                   │
│  M4 Anchor     : evidence_section_id (E-VQA) / triple (InfoSeek), rank, self-citation │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

Từ EchoSight chỉ tái sử dụng: (i) `WikipediaKnowledgeBaseEntry` để đọc KB, (ii) `reconstruct_wiki_sections` để tách unit, (iii) hai nhánh prompt vanilla và title-only làm baseline đối chiếu với số của paper. Mọi thứ khác viết mới.

---

## 6. Đề xuất sửa cụ thể cho đề cương v6

| Mục trong v5 | Sửa thành |
|---|---|
| **Objective & Scope · "Hệ thống làm giá đỡ"** | Thay "chạy inference không cần train" bằng "cung cấp K đã rerank (cache một lần), KB có cấu trúc section, và hai LLM baseline; harness đo viết mới" |
| **M1 · Source Gate** | Bỏ "ảnh xám". Còn hai lần gọi thêm: bỏ K · (bỏ K là leakage check luôn, vì generator không thấy ảnh). Nếu chọn phương án (b) §3.1 thì giữ nguyên và thêm MLLM vào bảng model |
| **H1** | IV: `K ∈ {giữ, bỏ}`; DV: KS, ΔlogP, tỉ lệ leakage. Bỏ VN. Thêm "decoding greedy, seed cố định" như điều kiện bắt buộc, không phải giữ-cố-định-mặc-nhiên |
| **D4** | Viết lại: "tách đóng góp của tri thức ngoài với trí nhớ sẵn có của model; ảnh nằm ngoài giao diện đo vì generator text-only" |
| **Không làm** | Thêm: "Không đưa ảnh vào generator; ảnh chỉ tác động qua retrieval, và retrieval đã đóng băng" |
| **Cổng tuần 1** | Từ "khớp 41.8%" thành: (i) Recall@1/5/10 của retrieval trên 200 câu nằm trong ±3 điểm so với Table 1; (ii) accuracy greedy top-1 section nằm trong dải [vanilla 21.0%, paper 41.8%] và cao hơn title-only 29.4%. Nếu (i) fail nhưng (ii) pass với section gold → chuyển phương án B ngay |
| **Kế hoạch tuần 1–2** | Chạy khối A (retrieval) và việc map triple Wikidata cho InfoSeek **song song** từ tuần 1, không đợi tuần 4 |
| **Năm con số tài nguyên · "Đã dựng được EchoSight chưa"** | Điền: *chưa; phần tốn nhất là ảnh + FAISS, không phải LLM; tách được thành khối chạy một lần* |
| **Chi phí mỗi instance** | Tính lại theo scoring thay vì generate: m lần forward teacher-forced (rẻ) + số lần generate chỉ khi cần Y(v) cứng. Con số 168K "lần gọi" nên tách thành "forward" và "generate" |
| **Threats · External** | Thêm: "Hệ được đo không phải EchoSight nguyên bản: prompt đa-unit, greedy decoding, template có đánh số là của chúng tôi. Nhất quán với việc đối tượng là *giao diện* chứ không phải sản phẩm, nhưng phải khai rõ" |
| **Rủi ro "Không dựng lại được EchoSight"** | Bổ sung bằng chứng: issue #12/#18 (R@1 8% vs 36.5%), #26 (không có retrieval JSON), #30 (eval InfoSeek chưa có). Phương án B chuẩn bị từ ngày 1 |

---

## 7. Checklist việc kỹ thuật trước khi có số đầu tiên

Thứ tự đã tính đến phụ thuộc.

- [ ] Tạo `env-retrieval` theo `requirements.txt` gốc; xác nhận `faiss-gpu` import được
- [ ] Tải KB InfoSeek 100K + FAISS index; kiểm md5 FAISS E-VQA nếu dùng E-VQA
- [ ] Sửa `test_reranker.py`: đường dẫn hard-code, `iNat_image_path`
- [ ] Chạy khối A trên 200 câu InfoSeek với `--save_result` → `retrieval_result.json`; so Recall với Table 1
- [ ] **Song song:** kiểm CSV InfoSeek của EchoSight có cột nào ngoài `question/answer/wikipedia_url/data_id` không; lấy QID từ InfoSeek gốc; dựng script map (QID, answer) → relation qua Wikidata SPARQL
- [ ] Tạo `env-kup`: torch + transformers mới, không TF
- [ ] Viết `scorer.py`: load model, `score(prompt, answer) → log-prob`, batched, greedy generate
- [ ] Viết `prompt.py`: template đa-unit có đánh số, cắt theo unit bằng tokenizer đúng model, chế độ có/không `<evidence>`
- [ ] Viết `eval.py`: chuẩn hoá answer, EM/F1, eval InfoSeek theo bản gốc
- [ ] Tái lập baseline greedy: vanilla, title-only, top-1 section — so với 21.0 / 29.4 / 41.8
- [ ] Chỉ khi ba dòng trên xong mới bắt đầu M1 (E1)

---

## 8. Câu hỏi cần Thầy quyết (cập nhật từ v5)

1. **Trục ảnh:** bỏ VN và giữ hai LLM text-only (khuyến nghị), hay thêm MLLM và chấp nhận mất luận điểm "cùng model với Structural Attention Tax"?
2. **Dataset chính:** InfoSeek có gold triple về cấu trúc nhưng phải map ngược Wikidata (~1 tuần); E-VQA có gold section sẵn nhưng không có triple → E3 khó hơn. Có nên chạy E1/E2 trên E-VQA trước để có kết quả sớm, và để E3 cho InfoSeek?
3. **Tiêu chí tái lập:** chấp nhận accuracy greedy khác paper (do đổi decoding) làm cổng tuần 1 không?
4. **GPU:** với scoring batched, ước tính lại sau khi có số thời gian một forward trên máy thật — cần con số VRAM để biết chạy 8B bf16 với prompt 10 unit có vừa không.

---

## Phụ lục · Chi tiết code đã kiểm

| File | Dòng | Ghi nhận |
|---|---|---|
| `model/answer_generator.py` | 97 | `_adjust_prompt_length` luôn dùng tokenizer Mistral |
| | 165, 243, 253 | Ba nhánh prompt: section / title-only / vanilla |
| | 181–188, 266–272 | Sampling decoding cho cả hai model |
| | 190, 274 | Mistral cắt `[:-4]`; LLaMA3 không strip `<|eot_id|>` |
| `test/test_reranker.py` | 59, 65 | Đường dẫn hard-code `/remote-home/share/...` |
| | 76–80, 204–209 | Cấu trúc `retrieval_result[data_id]` được lưu |
| | 246–248 | Generator chỉ nhận `reranked_sections[0]` |
| `test/test_answer_generator.py` | 33 | Index sai vào JSON keyed theo `data_id` |
| | 57 | `args.vqa_results` không tồn tại → crash |
| `utils/evaluation_utils.py` | 32, 395 | `"infoseek"` không nằm trong `_QUESTION_TYPES` → `ValueError` |
| `utils/test_utils.py` | 3–5 | `/PATH/TO/...` placeholder |
| `dataset/dataset.py` | 115 | `evidence_section_id` — gold section E-VQA |
| `model/retriever.py` | 184–209 | Schema KB entry |
| `requirements.txt` | — | `transformers==4.37.2`, `torch==2.3.1`, `faiss-gpu==1.7.2`, `tensorflow==2.16.1` |
| GitHub issues | #12, #18 | Không tái lập được Recall dù md5 FAISS đúng |
| | #26 | Không cung cấp retrieval result JSON |
| | #29 | Ảnh InfoSeek thiếu trong AToMiC |
| | #30 | Eval InfoSeek — mở, chưa trả lời |
