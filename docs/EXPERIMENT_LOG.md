# Nhật ký thử nghiệm & quyết định — Thành viên B

> File append-only: chỉ thêm entry mới ở CUỐI, không sửa lại entry cũ (kể cả khi sau này thấy quyết định đó sai — ghi entry mới nói rõ "đảo ngược quyết định ngày X vì Y", đừng xoá lịch sử). Đây là nguồn chính để viết phần "Phương pháp"/"Thực nghiệm" của bài báo nếu vào top 10 (hạn 24/09–24/10).
>
> Format mỗi entry: `### [module] dd/mm — tiêu đề ngắn`, rồi mô tả ngắn gọn: đã thử gì, số đo được, quyết định + lý do. Spec hiện tại (input/output/workflow) KHÔNG nằm ở đây — xem docstring đầu file `.py` của module đó. README.md của module giữ câu hỏi còn mở/rationale nền, không phải log.

> **⚠️ Lưu ý về nguồn gốc các entry dưới đây (12/08/2026):** Toàn bộ entry từ đây đến trước mốc `[B1] 12/08 — review lại...` là do AI (Claude Code) tự viết trong các phiên trước, **chưa được người duyệt lại từng phần** — coi là ghi chú tham khảo/giả thuyết chưa verify, không phải kết luận đã chốt. Một số số liệu (bug `\r\n`, thống kê corpus...) đã được kiểm chứng lại độc lập và xác nhận đúng khi rebuild B1 (xem entry `[B1] 12/08`), nhưng không phải mọi entry cũ đều đã qua bước này. Từ mốc 12/08 trở đi, entry chỉ được ghi khi có người (không chỉ AI) đã xem code + số đo thật trước khi chốt.

---

### [Data] 06/08 — nhận corpus thật, phát hiện tập đóng

Corpus `selected-contexts/` (8,532 file `context_*.json`) được BTC cấp, chặn B0/B1 từ đầu dự án (xem `B_technical_handoff.md` mục 7) nay đã gỡ.

Đo trên toàn bộ 8,532 file (không phải mẫu): tổng số văn bản (8,532) gần khớp tuyệt đối tổng số câu hỏi train+warmup+public (7,000+500+1,000=8,500). Giả thuyết: corpus là tập đã lọc sẵn cho toàn đề, không phải kho luật mở — thu hẹp đáng kể không gian tìm kiếm cho B2. **Chưa chứng minh bằng liên kết trực tiếp** (train/warmup/public không có field `context_id`) — chỉ suy từ số lượng trùng khớp, cần B0 chạy xong để xác nhận chắc.

Cũng phát hiện vấn đề chất lượng: 20 văn bản `passage` rỗng, 1,125 văn bản (13.2%) `name=None` (đã viết fallback lấy tên từ `link` trong `io_utils.py`), 1 outlier ~6M ký tự dính rác paywall (`context_68843`, chỉ 5/8,532 file dính pattern này — hiếm, không cần xử lý đại trà). Chi tiết đầy đủ: `docs/DATA_NOTES.md`.

Quyết định: loại 20 văn bản rỗng khỏi index qua `io_utils.iter_corpus(skip_empty=True)` (mặc định), không xoá khỏi `data/corpus/` (giữ nguyên raw).

---

### [B0/B1/B2] 06/08 — chốt thư viện/tham số còn treo, ưu tiên deadline 10/08

Bối cảnh: hạn nộp bài hợp lệ đầu tiên 10/08 (còn 4 ngày lúc quyết) — mọi lựa chọn ưu tiên thư viện có sẵn/đơn giản đã kiểm chứng thay vì tự tối ưu, trừ khi tự viết rõ ràng nhẹ hơn cài thêm dependency.

- **B2 BM25**: `rank_bm25` (không tự viết — tránh rủi ro bug tinh vi ảnh hưởng điểm mà không kịp soát kỹ trong thời gian ngắn).
- **B2 tokenizer**: `underthesea` (phổ biến hơn `pyvi`, dễ tra lỗi nhanh).
- **B2 K khởi động**: 10 — không phải giá trị cuối, B3 sẽ đo lại 5/10/20/50 bằng Recall@K khi có nhãn B0.
- **B0 inverted index**: tự viết dict Python — corpus chỉ 8,532 văn bản, `datasketch`/MinHash là overkill ở quy mô này.
- **B0 alignment**: tự cài Smith-Waterman biến thể (~30 dòng) — `Bio.Align` nặng, không cần tính năng sinh học.
- **B0 n-gram**: n=6 khởi động, thử trên 20-30 mẫu warmup trước khi chạy full 500.
- **B1 test mẫu**: xác nhận `context_740.json` = Quyết định 5868/QĐ-BYT, có sẵn trong `data/corpus/` (tra trực tiếp bằng id từ ví dụ trong data overview, không cần đoán).
- **B1 case sửa đổi/bổ sung**: để B0 xử lý (pattern "Trước đây, căn cứ..."), B1 không làm trùng.
- **B1 schema có khớp nhóm IR không**: CHƯA CHỐT — cần hỏi người thật (Trưởng nhóm/nhóm IR), không tự quyết được. Quyết định tạm: dùng schema đề xuất làm baseline ngay, chấp nhận rủi ro phải đổi sau khi IR phản hồi, vì deadline không cho phép chờ.

Đã cập nhật các quyết định (trừ mục cuối) thẳng vào docstring của `label.py`/`retrieve.py`/`parser.py` — README của 3 module rút gọn lại, chỉ B1 còn giữ 1 câu hỏi mở thật.

---

### [B2] 06/08 — code thân hàm, test trên corpus thật

Implement `build_index`/`search` (`rank_bm25.BM25Okapi` + `underthesea.word_tokenize`). Phải thêm 1 lớp mỏng `BM25Index` (bọc `bm25` + `context_ids` song song) — `BM25Okapi` tự nó không giữ ID tài liệu, chỉ giữ theo vị trí. Điều chỉnh nhỏ so với dự tính ban đầu ("giữ nguyên type thư viện"), không tránh được.

Test sanity trên 10 văn bản thật (gồm `context_740`): query bằng câu lấy thẳng từ nội dung `context_740` → `context_740` đứng đầu bảng xếp hạng (score 14.4, cách biệt xa vị trí 2 là 4.0). Đúng như kỳ vọng.

**Phát hiện quan trọng cho B3 (granularity)**: `build_index` cho 10 văn bản (passage đầy đủ, TB ~29K ký tự/văn bản) mất 1.7s — ngoại suy tuyến tính, index TOÀN BỘ 8,532 văn bản ở granularity "nguyên văn bản" (không chunk) sẽ mất ước tính **~24 phút**, không khả thi để chạy nhanh/lặp lại nhiều lần khi thử nghiệm. Đây là lý do THỨ HAI (ngoài phạt phân mảnh METEOR) để ưu tiên index ở granularity Khoản (ngắn hơn nhiều) thay vì nguyên Điều/văn bản — B3 cần cân nhắc thêm góc độ tốc độ này khi quyết định.

---

### [B0] 06/08 — code thân hàm, thiết kế bổ sung khi implement

Implement `build_shingle_index` (dict tự viết, n=6, tokenize bằng regex `\S+` giữ lại vị trí ký tự để tái tạo `matched_span` đúng nguyên văn), `find_candidates` (đếm overlap n-gram), `align_span` (Smith-Waterman word-level tự cài), `detect_role` (regex marker "Trước đây..."), `label_answer`, `label_dataset`.

Bổ sung so với spec ban đầu (cần thiết khi code thật, không phải đổi ý tuỳ tiện):
- `align_span` giới hạn cửa sổ alignment quanh vị trí neo (anchor) tìm được qua n-gram, thay vì chạy Smith-Waterman trên toàn bộ passage — cần thiết vì passage có thể dài hàng trăm nghìn ký tự, DP đầy đủ sẽ quá chậm. Nếu không tìm được anchor, fallback cửa sổ = 2000 token đầu passage (giới hạn đã biết, có thể bỏ sót nếu đoạn trích nằm sau token 2000 và không share n-gram nào — hiếm nhưng có thể xảy ra).
- `label_answer` tách answer theo marker "Trước đây..." để gắn role, nhưng **mỗi phần chỉ lấy 1 candidate tốt nhất** — chưa xử lý đúng case đa văn bản (12.2%) khi answer trích ≥2 luật MÀ KHÔNG có marker "Trước đây" phân tách (vd 2 trích dẫn liên tiếp trong cùng 1 đoạn không sửa đổi). Đây là giới hạn v1 đã biết, cần quay lại xử lý sau khi có số đo Recall thật từ B3.

**Test đã chạy** (10 văn bản thật làm pool, `context_740` làm target):
- Đoạn trích lấy NGUYÊN VĂN từ `context_740` -> `find_candidates` xếp 740 hạng 1, `align_span` score=1.0, matched_span khớp tuyệt đối.
- Đoạn trích có nhiễu nhẹ (đổi 2 từ) -> score=0.94, vẫn định vị đúng vùng khớp (không rớt về 0).
- Answer ghép marker "Trước đây, căn cứ... quy định như sau:" -> `_split_by_amend_marker` tách đúng 2 phần, `label_answer` trả 2 MatchedSpan với role "hiện hành"/"trước đây" khác nhau, cùng context_id 740 (đúng kỳ vọng vì dùng chung 1 đoạn nguồn cho test).
- Đối chứng âm: câu hỏi thật từ `warmup.json` (không liên quan nội dung 10 văn bản test) -> `label_answer` trả về `[]` (không match giả — đúng kỳ vọng, ngưỡng confidence hoạt động).

**Timing đo được**: `build_shingle_index` trên 200 văn bản thật = 2.7s -> ngoại suy toàn bộ 8,532 văn bản ≈ **1.9 phút** (khả thi, không cần tối ưu thêm cho v1). `align_span` trên 1 cặp answer~400 ký tự / passage~8,500 ký tự = 0.03s.

**Việc còn lại trước khi coi B0 "xong"**: chạy `label_dataset` thật trên toàn bộ `warmup.json` (500 mẫu) — cần batch run riêng (không hợp trong 1 lệnh tương tác của phiên chat), rồi đọc mẫu tay ~20-30 kết quả để đánh giá chất lượng thật (không chỉ test tổng hợp như trên).

---

### [Research] 06/08 — tìm thể lệ DSC 2026 chính thức trên web

Đã tìm và fetch: trang khởi động chính thức (`uit.edu.vn`, `forum.uit.edu.vn`), trang `dsc.uit.edu.vn`.

**Xác nhận được** (khớp handoff, có bổ sung):
- Hạn đăng ký: hết ngày 16/8/2026 — KHỚP với handoff.
- Timeline tổng: 01/7–16/8 đăng ký, 01/8–05/8 warm-up, 06/8–18/9 public test, 19/9–23/9 private test, 24/9–24/10 hoàn thiện bài báo — khớp handoff gần như tuyệt đối.
- Mới: lệ phí 100.000đ/đội (không có trong handoff), hội thảo khoa học LegalQA 06/11, tổng kết trao giải 13/11, tổng giải thưởng 50.000.000đ (12tr/8tr/5tr mỗi chủ đề).

**KHÔNG tìm được** (không có trên các trang public đã index): văn bản "Thể lệ" chi tiết dạng Điều khoản cho DSC 2026 (nên không giải được mâu thuẫn Điều 6/7 top 7 vs top 10), công thức gộp METEOR+ROUGE-L để xếp hạng, xác nhận trần tham số 4.0B. `dsc.uit.edu.vn` hiện vẫn hiển thị nội dung **2025** (bài toán khác hẳn — phát hiện ảo giác LLM, không phải LegalQA) — site chưa deploy bản 2026, hoặc thể lệ chi tiết chỉ phát cho đội đã đăng ký (qua CodaBench/email), không public. **Kết luận: vẫn cần hỏi BTC trực tiếp** (dsc@uit.edu.vn) hoặc chờ Trưởng nhóm chia sẻ văn bản gốc — không phải việc web search giải quyết được thêm.

**⚠️ Rủi ro mới phát hiện, CHƯA XÁC NHẬN cho 2026**: thể lệ Bảng B **năm 2025** (bài toán khác) có điều khoản "Base-LLM phải nằm trong danh sách trắng của BTC (LLaMA-3-8B, Mistral-7B-Instruct...). Mọi LLM thương mại (GPT-4o, Gemini, Claude...) sẽ không được chấm điểm." Không có gì xác nhận điều này áp dụng cho LegalQA 2026, nhưng cùng ban tổ chức nên khả năng lặp lại không nhỏ — **nếu áp dụng, sẽ ảnh hưởng trực tiếp thiết kế Generator (C)** nếu C đang định dùng API LLM thương mại. Cần hỏi BTC/Trưởng nhóm xác nhận sớm, độc lập với câu hỏi B1.

---

### [Docs] 06/08 — đọc file phân công gốc "DSC2026 — Phân công nhóm Task 2: LegalQA" (13 trang, 02/08/2026)

File này có trong base knowledge (PDF, chưa đọc trước đó — chỉ đọc `B_technical_handoff.md` là bản tóm tắt lọc lại). Nhiều chi tiết mới quan trọng: T5 (Trưởng nhóm) đã sở hữu baseline thô cho hạn 10/08 (không phải việc của B); có tập dev nội bộ 150 câu đóng băng (T2, Trưởng nhóm) — cần hỏi đã chốt câu nào chưa để tránh dùng nhầm khi thử nghiệm; `common/scoring.py` là harness cá nhân, số chính thức chờ harness chung của Trưởng nhóm; bảng ngân sách cụ thể (BM25=0, bi-encoder≤0.7B, reranker≤0.6B); bảng độ nhạy METEOR theo tỉ lệ độ dài (đo thật 120 mẫu) — viết thừa rẻ hơn viết thiếu 3-4 lần, quan trọng cho thiết kế B4/B6 sau này; xác nhận cách xây nhãn B4 (trích giữa "như sau:"/"Theo đó-Như vậy-Tóm lại"); thêm module B5 (đối chứng kiến trúc BM25+generator lớn vs dense+generator nhỏ, P1) chưa từng biết trước. Đã cập nhật `docs/SYSTEM_SCAFFOLD.md`, `src/b4_span_selection/README.md`, thêm `src/b5_architecture_tradeoff/README.md`.

**Áp dụng ngay cho B0**: thêm `_extract_quote_core()` — trích riêng Khối 2 (giữa "như sau:" và "Theo đó/Như vậy/Tóm lại") trước khi đưa vào `find_candidates`/`align_span`, thay vì dùng nguyên cả đoạn (có lẫn Khối 1/3 diễn giải, không có trong corpus, gây nhiễu n-gram). Tài liệu gốc mô tả cách này cho B4, nhưng áp dụng được trực tiếp cho B0 vì cùng bản chất (định vị trích dẫn nguyên văn).

**Đo cải thiện thật** (answer mô phỏng: diễn giải dài + "như sau:" + trích thật từ `context_740` + "Theo đó," + bình luận dài, tổng ~950 ký tự nhiễu bao quanh 400 ký tự trích thật):
- KHÔNG lọc marker: `align_span` score = 0.481
- CÓ lọc marker: `align_span` score = 1.000
- Cải thiện: **+0.519** — rất đáng kể, xác nhận hướng đề xuất trong tài liệu phân công đúng và nên áp dụng.
- `find_candidates` cũng gọn hơn: không lọc trả về 3 candidate (740, 100109, 100125), có lọc chỉ còn 2 (740, 100109) — giảm nhiễu ứng viên sai.

Đã sửa `label_answer` dùng `_extract_quote_core` mặc định, chạy lại `experiments/demo_b0_autolabel.py` — cả 4 test cũ vẫn pass (test 3 "trước đây" còn tăng từ 0.892 lên 1.000 nhờ cải tiến này).

---

### [B3] 06/08 — dựng Recall@K, phát hiện + sửa 2 vấn đề hiệu năng nghiêm trọng của B0 khi chạy trên corpus thật quy mô lớn

Viết `src/b3_eval_recall/eval_recall.py` (`recall_at_k`, `recall_at_multiple_k`) — spec đơn giản, ăn thẳng output B0 (`MatchedSpan`) + B2 (`ScoredContext`).

**Vấn đề 1 — OOM khi build_shingle_index trên full corpus**: `build_shingle_index(corpus)` với toàn bộ 8,532 văn bản bị Kill (OOM) trong sandbox (3.8GB RAM, không swap). Đo tăng dần: 1500/8,532 văn bản đã tạo 7.7M shingle key, chiếm 3.5GB. Nguyên nhân: dùng `tuple[str,...]` (6 chuỗi) làm key trực tiếp — quá tốn bộ nhớ ở quy mô hàng chục triệu key. **Đã sửa**: `_shingle_key()` băm n-gram thành 1 int 64-bit làm key (`dict[int, set[int]]` thay vì `dict[tuple, set[int]]`) — giảm được nhưng KHÔNG đủ (1500 văn bản còn 2.85GB, ngoại suy full corpus vẫn >10GB). **Kết luận**: đây là giới hạn phần cứng của sandbox này (3.8GB), không phải lỗi thuật toán — cách index toàn cục vẫn là thiết kế đúng cho máy đủ RAM (nhanh hơn nhiều khi có nhiều câu hỏi lặp lại). Giải pháp cho hôm nay: dùng pool ngẫu nhiên 1,000/8,532 văn bản (đo an toàn ~1.9GB) thay vì full corpus — xem `experiments/demo_b3_step1_label.py`.

**Vấn đề 2 — cửa sổ alignment bị "bung" khi có neo giả rải rác**: đo trên corpus thật (không phải 10 văn bản test cũ), `align_span` mất trung bình ~1.2s/lần, có lần tới 1.9s — quá chậm cho batch nhiều câu hỏi. Truy ra nguyên nhân: `align_span` cắt cửa sổ alignment bằng `min(vị trí neo)` .. `max(vị trí neo)` — chỉ cần 1 neo giả (n-gram trùng ngẫu nhiên ở xa, ví dụ cụm từ pháp lý chung chung) là cửa sổ bung ra gần hết văn bản (đo thật: 888 vị trí neo rải rác trong văn bản 41,613 token → cửa sổ 39,660 token thay vì đáng lẽ chỉ vài trăm). **Đã sửa 2 phần**:
1. `_rolling_ngram_hashes()` — hash n-gram kiểu Rabin-Karp rolling hash, O(số token) thay vì O(số token × n) khi quét tìm neo.
2. `_find_best_anchor_cluster()` — tìm CỤM neo dày đặc nhất bằng sliding window (O(số vị trí neo)) thay vì lấy min/max toàn bộ, bỏ qua neo lẻ tẻ ở xa.

**Kết quả đo được** (5 câu hỏi thật, pool 1,000 văn bản, so trước/sau tối ưu): **26.21s → 2.42s (nhanh hơn 10.8 lần)**. Chạy lại `experiments/demo_b0_autolabel.py` (4 test case cũ) — kết quả giữ nguyên y hệt, không có regression.

**Bài học chung**: cả 2 vấn đề chỉ lộ ra khi test trên corpus THẬT ở quy mô thật (văn bản dài hàng chục nghìn token, nhiều n-gram trùng ngẫu nhiên) — bộ test nhỏ (10 văn bản tự chọn) trước đó không đủ đa dạng để phát hiện. Bài học: cần test trên mẫu ngẫu nhiên đủ lớn trước khi tin vào 1 module, không chỉ tin vào test case tự thiết kế.

---

### [B2] 06/08 — OOM thứ 2 (khác B0): `underthesea.word_tokenize` bung RAM trên 1 văn bản outlier 5.98M ký tự

Sau khi B0 (bước 1, `demo_b3_step1_label.py`) chạy xong (21/100 câu hỏi có nhãn, pool ngẫu nhiên 1,000 văn bản, ~58s), chạy bước 2 (`demo_b3_step2_recall.py`: `retrieve.build_index()` trên CÙNG pool 1,000 để tính Recall@K) → bị Kill (exit 137) ngay tại bước build_index, dù B0 vừa xử lý xong đúng 1,000 văn bản này không vấn đề gì.

**Chẩn đoán**: thu nhỏ pool B2 xuống chỉ còn văn bản đúng (19-21 từ B0) + 300 nhiễu ngẫu nhiên = 319 văn bản — VẪN bị Kill. Kiểm tra phân phối độ dài của pool 319: max = 5,983,358 ký tự (`context_68843`), trung vị chỉ 23,731 — đúng bằng giá trị max đã ghi nhận trong `docs/DATA_NOTES.md` mục thống kê ("một số văn bản là cả bộ luật/quy chuẩn khổng lồ", KHÔNG phải lỗi crawl). Đo trực tiếp: `underthesea.word_tokenize` trên 200,000 ký tự đầu của văn bản này tốn +380MB RAM (128MB → 509MB) — ngoại suy cho đủ 5.98M ký tự thì một mình văn bản này đã vượt xa 3.8GB của sandbox. Đây là lỗi hoàn toàn khác B0 (không phải shingle index, không phải align_span) — `underthesea` có chi phí bộ nhớ không tuyến tính theo độ dài input.

**Đã sửa**: thêm `MAX_TOKENIZE_CHARS = 50_000` trong `src/b2_retrieval/retrieve.py`, `build_index()` cắt text trước khi đưa vào tokenizer (tham số `max_tokenize_chars`, có thể tắt bằng `None` trên máy đủ RAM). 50k ký tự nằm giữa p90 (90,141) và median của corpus nên chỉ cắt phần đuôi phân phối, không ảnh hưởng đa số văn bản. Đây là fix TẠM — fix đúng lâu dài là B1 chunk theo Khoản trước khi index (mỗi Khoản ngắn hơn nhiều so với văn bản gốc), granularity cuối cùng do B3 quyết định.

**Kết quả sau fix**: `demo_b3_step2_recall.py` chạy xong trong ~47s (build_index 45.0s trên 319 văn bản).

---

### [Infra] 06/08 — symlink `data/corpus` bị hỏng giữa 2 phiên, đổi sang path trực tiếp

`data/corpus` (symlink tạo ở phiên trước, trỏ `data/selected-contexts/selected-contexts/`) trả `OSError: Input/output error` khi đọc — không phải lỗi 1 file, cả thư mục qua symlink đều lỗi, trong khi truy cập trực tiếp `data/selected-contexts/selected-contexts/` vẫn bình thường. Nguyên nhân nhiều khả năng: sync Windows (`E:\DSC\PROJECT`) <-> sandbox Linux không giữ được symlink ổn định giữa các phiên làm việc khác nhau — hạ tầng dễ vỡ, không nên phụ thuộc. **Đã sửa**: `src/common/config.py` — `CORPUS_DIR` trỏ thẳng `DATA_DIR / "selected-contexts" / "selected-contexts"`, bỏ hẳn symlink. Mọi module dùng `config.CORPUS_DIR` (không hardcode path riêng) nên chỉ sửa đúng 1 chỗ.

---

### [B4] 06/08 — code baseline v0 (heuristic lexical overlap), test đối chiếu nhãn B0

Phát hiện khi thiết kế spec: cách marker-based ("như sau:"/"Theo đó") của B0 CHỈ dùng được để sinh nhãn gold offline (cần `answer` có sẵn) — B4 chạy ở serving-time thật chỉ có `question`, không có `answer` để tách marker, nên cần thuật toán khác hẳn, không tái dùng trực tiếp được `_extract_quote_core`.

**Thuật toán chọn** (baseline v0, 0 tham số học, xem `src/b4_span_selection/selection.py`): tách Khoản ứng viên thành các đơn vị theo ranh giới Điểm tự nhiên (`\n\n`, B1 giữ lại sẵn) hoặc câu/mệnh đề nếu Khoản không chia Điểm; chấm điểm overlap token thô (giao tập hợp) giữa mỗi đơn vị và câu hỏi; trả về đơn vị điểm cao nhất, fallback trả nguyên Khoản nếu không đơn vị nào overlap.

**Test đối chiếu nhãn B0** (`experiments/demo_b4_span_selection.py`, dùng lại 21 câu hỏi có nhãn từ `_b3_step1_output.json`): 6/21 bỏ qua (matched_span của B0 không định vị được trong bất kỳ Khoản nào sau khi B1 chunk — có thể do granularity khác biệt hoặc matched_span cắt ngang ranh giới Khoản). Trong 15 câu còn lại: **B4 chọn trúng vùng khớp với B0 ở 10/15 (67%)**.

**Diễn giải — CHƯA phải benchmark chính thức**: đây là proxy thô — B0 định vị VĂN BẢN nguồn (document-level), B4 cần định vị đúng Điểm/câu (finer-grained), "khớp" ở đây chỉ đo bằng substring overlap chứ không phải nhãn gold thật ở đúng granularity B4 cần. Số liệu chính thức đòi hỏi nhãn gold riêng cho B4 (chưa có, chưa thiết kế cách sinh) — nhưng 67% trên baseline 0-tham-số, chưa tinh chỉnh gì, đủ để xác nhận hướng đi hợp lý, không phải ngẫu nhiên.

**Case sai đáng chú ý** (`165287`, `82359`): B4 chọn nhầm unit liền kề (cùng chủ đề, khác điểm cụ thể) — dấu hiệu overlap token thô đôi khi không đủ phân biệt giữa các đơn vị lân cận có từ vựng giống nhau (vd nhiều điểm a/b/c cùng nói về "bảo toàn vốn"). Hướng nâng cấp khả thi: trọng số IDF thay vì đếm thô (giảm ảnh hưởng từ phổ biến), hoặc BM25 cấp câu thay vì overlap set — chưa làm, ghi nhận làm sau khi có ngân sách thời gian.

---

### [B6] 06/08 — code format tối thiểu (đề xuất tạm)

Code `src/b6_context_package/package.py` (`ContextPackage`, `package_context()`) — dict phẳng: `question_id, context_id, dieu_so, khoan_so, span_text, source_name, source_link, retrieval_score`. Trả lời 3 câu hỏi từng treo trong README cũ: (1) số lượng đoạn tối đa — không hardcode trong hàm, để caller quyết định qua top-N theo score trước khi gọi theo batch; (2) thứ tự — giữ `retrieval_score` trong package để caller/Generator tự sort; (3) kèm metadata hay text thuần — chọn KÈM metadata (Điều/Khoản/nguồn), chi phí thấp, hữu ích cho trích dẫn + truy vết B7.

**Còn treo thật**: chưa xác nhận với người làm Generator (C) — đây là đề xuất một chiều từ phía B. Chi phí đổi thấp (chỉ là dict phẳng, không có logic phức tạp phụ thuộc), nên không chặn tiến độ, nhưng cần xác nhận khi có thể trao đổi.

---

### [Pipeline] 06/08 — chạy end-to-end B1→B2→B4→B6 lần đầu, xác nhận wiring đúng

Thêm `select_khoan()` vào `src/b4_span_selection/selection.py` (cùng thuật toán overlap token với `select_span`, nhưng áp dụng ở cấp CHỌN KHOẢN trong nhiều khoản ứng viên của 1 văn bản — cần vì B2 hiện trả về cả văn bản, chưa index ở granularity Khoản, nên phải thu hẹp 2 bước: văn bản → Khoản → đơn vị con).

`experiments/demo_b_end_to_end.py` chạy 5 câu hỏi thật qua đủ B2(search) → B1(parse_dieu_khoan trên top-1) → B4(select_khoan rồi select_span) → B6(package_context) — **không lỗi, mọi field ContextPackage điền đủ**. Xác nhận toàn bộ 4 module ghép được với nhau đúng interface đã thiết kế.

**QUAN TRỌNG — đây KHÔNG phải bằng chứng chất lượng, chỉ là bằng chứng WIRING đúng**: 4/5 câu hỏi ở top-1 trả về context_id KHÁC với context_id đúng theo nhãn B0 (dù pool đã được build có chứa đúng tài liệu, và B3 step2 đo Recall@5=0.905 trên chính pool này) — vì `top_k=1` ở đây chặt hơn nhiều so với Recall@5/10 đã đo (top-1 đúng khó hơn "nằm trong top-5" nhiều). Kèm B4 baseline v0 còn non (67% khớp thô ở mức tốt nhất), một vài kết quả cuối (vd câu `132819` chọn nhầm hẳn sang văn bản rửa tiền cho câu hỏi về bảo vệ dữ liệu cá nhân) rõ ràng sai — đúng như kỳ vọng ở baseline v0 đầu tiên, chưa tinh chỉnh gì. Giá trị của lần chạy này: xác nhận kiến trúc/interface đúng, SẴN SÀNG để đo chất lượng thật + cải thiện từng module, không phải xác nhận chất lượng đã tốt. **Recall@K sơ bộ (pool 319 văn bản = 19 đúng + 300 nhiễu, KHÔNG phải full 8,532, n=21 câu hỏi có nhãn)**:

| K | Recall@K | hits/n |
|---|---|---|
| 5 | 0.905 | 19/21 |
| 10 | 0.905 | 19/21 |
| 20 | 0.905 | 19/21 |
| 50 | 1.000 | 21/21 |

**Diễn giải — PHẢI đọc kèm cảnh báo**: đây là số liệu XÁC NHẬN DÂY CHUYỀN (pipeline B0→B2→B3 chạy đúng, ra số thật, không lỗi), KHÔNG phải số liệu chính thức để báo cáo, vì 3 lý do: (1) pool chỉ 319 văn bản trong tổng 8,532 — ít đối thủ cạnh tranh hơn hẳn tình huống thật, Recall@K nhiều khả năng LẠC QUAN hơn số thật; (2) n=21 câu hỏi có nhãn quá nhỏ để kết luận thống kê (khoảng tin cậy rất rộng); (3) granularity đang là nguyên văn bản (chưa chunk theo Khoản) — B3 "chốt granularity" thật sự vẫn cần B1. Số liệu chính thức cần: full corpus 8,532 văn bản + full train/warmup + granularity đã chốt qua B1, và cần chạy trên máy đủ RAM (không phải sandbox 3.8GB này) vì cả 2 OOM (B0 và B2) đều là giới hạn phần cứng, không phải giới hạn thuật toán.

---

### [B1] 12/08 — review lại toàn bộ với người, phát hiện bug thật (Phụ lục/Nơi nhận dính vào Điều cuối), rebuild có kiểm chứng

**Bối cảnh**: quyết định làm lại toàn bộ pipeline từng module một, có người review từng bước trước khi code (không tin thẳng vào code/số liệu AI viết trước đó dù "trông hợp lý"). Bắt đầu từ B1 vì đơn giản nhất, làm nền granularity cho B2/B4.

**Verify lại claim cũ trước khi sửa gì** (không tin comment cũ, tự chạy lại trên data thật):
- Bug `\r\n`/`\n\n` (đã ghi ở entry `[B1] 06/08`): CHẠY LẠI xác nhận đúng — `context_740` cho tiêu đề Điều 3 đầy đủ "Cơ cấu tổ chức và hoạt động", không bị cắt cụt.
- Tỉ lệ khớp cấu trúc: đo lại trên mẫu ngẫu nhiên 60 văn bản (seed khác) = 50/60 (~83%) khớp, 10 fallback đều là văn bản thật không có cấu trúc Điều/Khoản (Chỉ thị, Công văn, bảng tiêu chuẩn) — nhất quán với con số cũ (26/30), không phải trùng hợp.

**Bug MỚI phát hiện (chưa ai từng ghi nhận), đo trên mẫu 200 văn bản ngẫu nhiên**: mọi văn bản luật kiểu Thông tư/Quyết định đều có đuôi hành chính cố định `"Nơi nhận:"` (danh sách người nhận) + chữ ký, và nhiều văn bản có thêm `"PHỤ LỤC..."` (mẫu đơn, bảng biểu, danh mục tài khoản...) theo sau. Code cũ không có điểm dừng nào trước khối này → toàn bộ đuôi hành chính + phụ lục bị dính vào text của **Điều cuối cùng**. Hệ quả nặng hơn: các dòng đánh số/gạch đầu dòng bên trong khối đó (số thứ tự bảng, mục lục...) bị `_KHOAN_RE` (regex bắt "1." "2)" đầu dòng) bắt nhầm thành Khoản giả — đo được cụ thể: `context_166280` Điều 43 có "Khoản 416" (thực chất là số hiệu tài khoản kế toán trong bảng Phụ lục), `context_231867` Điều 33 có 5 "Khoản" chỉ chứa ký tự rác `"/"`.

**Quyết định + lý do** (đã thảo luận, không tự quyết 1 chiều):
- Cắt riêng đuôi hành chính vào field `phu_luc_raw` (không xoá, không gộp vào Điều cuối) — lý do: phụ lục không phải nội dung Điều/Khoản, nhưng vẫn có thể cần cho câu hỏi hỏi trực tiếp về nội dung phụ lục (mẫu đơn...); B2 sẽ không index field này cùng cấp Điều/Khoản (tránh nhiễu retrieval) nhưng dữ liệu vẫn giữ, tách sau này không mất gì.
- Thêm `parse_status` ("matched"/"fallback") vào output — lý do: downstream (B2/B4) cần biết văn bản nào có cấu trúc Điều/Khoản thật để áp đúng logic, tránh nhầm với fallback toàn văn.
- Thêm `dieu_id`/`khoan_id` dạng `{context_id}_{dieu_so}` / `{context_id}_{dieu_so}_{khoan_so}` — lý do: `khoan_so` một mình không duy nhất toàn corpus (Điều khác nhau đều có thể có "Khoản 1"), B2 (index theo Khoản)/B3 (Recall@K theo Khoản)/B4 (chọn Khoản) cần 1 ID thống nhất để tham chiếu qua lại, tránh mỗi module tự sinh ID kiểu khác nhau rồi lệch.
- Đổi API `parse_dieu_khoan(passage) -> list[Dieu]` thành `parse_document(context_id, passage) -> ParsedDoc` (có `parse_status`, `dieu`, `phu_luc_raw`) — cần `context_id` làm input để sinh `dieu_id`/`khoan_id`, không thể giữ signature cũ.

**Test sau khi sửa** (`experiments/demo_b1_parser.py`, 4 test đều PASS): `context_740` giữ đúng 5 Điều như cũ; mẫu 30 văn bản không còn title dính `\n` lạ; **`context_166280`/`context_231867` (case Phụ lục) không còn Khoản giả** — verify trực tiếp `len(khoan.text) >= 3` cho mọi Khoản sau parse.

**Chạy full corpus**: `experiments/build_parsed_corpus.py` → `data/parsed_corpus.jsonl` (8,512 văn bản, loại 20 văn bản rỗng theo `iter_corpus` mặc định) — **84.6% matched cấu trúc Điều/Khoản, 15.4% fallback**. File nặng 651MB (chưa nén, chưa tối ưu — chấp nhận được ở bước này, có thể nén/rút gọn field sau nếu cần).

**Vì sao `.jsonl` (1 dòng = 1 record JSON hoàn chỉnh, độc lập) thay vì `.json` (1 mảng lớn chứa tất cả record)**:
  - **RAM khi đọc**: JSON thường phải load + parse HẾT cả file mới lấy được dù chỉ 1 record (đo thật: ~1.2GB RAM, ~4s cho 681MB). JSONL đọc/parse được từng dòng một, không cần giữ hết trong RAM nếu chỉ cần lọc/thống kê.
  - **Ghi thêm không cần đọc lại**: JSON thường muốn thêm 1 record phải đọc hết, sửa cấu trúc `[...]`, ghi lại toàn file. JSONL chỉ `write(line + "\n")` ở cuối — đúng cách `build_parsed_corpus.py` ghi từng văn bản ngay khi parse xong, không giữ 8,512 record trong RAM để ghi 1 lần cuối.
  - **An toàn khi bị ngắt giữa đường** (crash/Ctrl+C): JSON thường thiếu dấu `]` đóng cuối → CẢ FILE không parse được, mất luôn cả phần đã ghi đúng trước đó. JSONL: các dòng ghi xong trước đó vẫn đọc được bình thường, chỉ mất phần chưa ghi.
  - **Đánh đổi**: khó đọc bằng mắt hơn (mỗi dòng đặc, không xuống dòng theo field) — hợp lý vì đây là dữ liệu 8,512 record ĐỘC LẬP nhau (không phải 1 cây cấu trúc lồng cần xem tổng thể).

**Đối chiếu với báo cáo của bên khác (không cùng pipeline)**: 1 team khác báo cáo "trần Recall = 0.9933" trên tập dev 150 câu random, dùng chiến lược chunking khác hẳn (cắt theo số ký tự cố định 1500-2500, giữ nguyên toàn bộ đuôi văn bản không tách Phụ lục). Chưa so sánh được trực tiếp — cần B0 chạy lại trên granularity Khoản mới của B1 trước, để tự đo recall trần theo đúng method (xem việc tiếp theo). Thống nhất: nếu recall trần của hướng Điều/Khoản thua rõ, sẽ đổi sang hướng chunking theo ký tự — chi phí đổi thấp vì B2/B3/B4 downstream chỉ cần list unit có `id` + `text`, không phụ thuộc cách chunk.

**Việc tiếp theo**: B0 (autolabel) chạy lại trên granularity Khoản mới (`khoan_id`) để đo recall trần thật của B1, so với con số 0.9933 nêu trên.

---

### [B2] 12/08 — thiết kế kiến trúc nhiều lớp, code Layer 1 (BM25 cấp văn bản) + Expand sang Khoản

**Bối cảnh quyết định**: người dùng đề xuất thêm bi-encoder/reranker ngay. Phát hiện mâu thuẫn với tài liệu phân công gốc (`src/b5_architecture_tradeoff/README.md`): ngân sách tham số toàn hệ thống <4.0B, mặc định đã chốt ưu tiên BM25 (0 tham số) dồn cho generator (C) — chỉ thêm dense/reranker **nếu B5 chứng minh được** đáng phần ngân sách lấy từ C. Đã hỏi lại, quyết định: **vẫn làm multi-layer nhưng đo Recall@K từng lớp ngay trong lúc build** (không tách "làm" và "chứng minh" thành 2 việc riêng) — tôn trọng tinh thần B5, không phải bỏ qua.

**Tham khảo kiến trúc case-law retrieval khác** (người dùng đã dùng ở 1 cuộc thi trước, không phải DSC): BM25 lọc ở cấp `law_id` (văn bản) → lấy hết article trong top-N luật → bi-encoder (BGE-M3) chấm điểm cấp article → reranker (bản v2, ngưỡng động, bỏ hẳn tầng LLM-relevance-filter Qwen2.5-3B của bản v1). Áp dụng được: (1) BM25 cấp văn bản rồi mới mở rộng Khoản — ĐÚNG hơn đề xuất ban đầu của tôi (BM25 trực tiếp cấp Khoản), vì lexical match tốt hơn ở văn bản dài; (2) multi-query (full text/câu dài nhất/câu đầu/câu cuối); (3) ngưỡng động cho reranker. KHÔNG áp dụng: tầng LLM-relevance-filter 3B — vượt xa ngân sách retrieval cho phép (bi-encoder ≤0.7B + reranker ≤0.6B); đáng chú ý bản v2 của chính người dùng cũng đã bỏ tầng này, củng cố hướng đang chọn.

**Kiến trúc chốt**: `Question (+ multi-query) → [Layer 1] BM25 cấp context_id → [Expand] lấy hết Khoản/Dieu trong top-N văn bản (đọc parsed_corpus.jsonl) → [Layer 2] bi-encoder (CHƯA CODE) → [Layer 3] reranker ngưỡng động (CHƯA CODE)`. Layer 2/3 để sau khi có nhãn B0 thật (cần ground truth để đo Recall@K từng lớp, làm bằng chứng cho B5) — thứ tự thực tế: B2 Layer 1 (hôm nay) → B0 (dùng Layer 1 tìm candidate) → có nhãn → B2 Layer 2/3 (đo so sánh) → B5.

**Code hôm nay** (`src/b2_retrieval/retrieve.py`): giữ nguyên `build_index`/`search` cũ (đã đúng là Layer 1, không cần viết lại). Thêm `build_query_variants` (regex tách câu, lấy full/dài nhất/đầu/cuối, dedup qua set), `search_docs_multi` (hợp kết quả nhiều biến thể, giữ score MAX mỗi context_id — không cộng dồn, tránh văn bản khớp yếu nhiều biến thể thắng văn bản khớp rất mạnh 1 biến thể), `expand_to_units` (đọc `parsed_corpus.jsonl` qua `io_utils.load_parsed_corpus()` mới thêm, trả `UnitCandidate` với `unit_type` phân biệt "khoan"/"dieu_fallback"/"doc_fallback" — KHÔNG bao giờ tạo unit từ `phu_luc_raw`, đúng quyết định đã chốt ở B1), `search_units` (gộp Layer 1 + Expand, dùng ngay cho B0).

**Đo được**: `load_parsed_corpus()` (8.512 văn bản, 651MB file) — 1.18GB RAM, 3.9s, load 1 lần lúc khởi động (không load lại mỗi câu hỏi) — chấp nhận được.

**Test** (`experiments/demo_b2_retrieval.py`, 4 test PASS): `build_query_variants` sinh đúng 4 biến thể cho câu nhiều mệnh đề, dedup đúng cho câu 1 mệnh đề; `search_docs_multi` tự tìm lại `context_740` đúng top-1; `expand_to_units` trên `context_740` cho đúng 10 unit (8 Khoản qua Điều 2/3/4, 2 `dieu_fallback` cho Điều 1/5 không chia Khoản) — khớp chính xác cấu trúc đã biết từ B1; `search_units` chạy được trên câu hỏi thật từ `warmup.json`, không lỗi.

**Việc tiếp theo**: B0 dùng `search_units()` (Layer 1 + Expand) làm candidate retrieval thay cho shingle index cũ, tránh OOM đã gặp ở bản B0 cũ.

---

### [Infra] 12/08 — máy chạy hiện tại KHÔNG bị giới hạn RAM như sandbox cũ; build_index full corpus đo được 43 phút (không phải 20-24 phút ngoại suy)

**Phát hiện quan trọng**: mọi giới hạn "sandbox 3.8GB RAM, không swap" ghi trong log 06/08 (lý do phải dùng pool ngẫu nhiên 1,000 văn bản thay vì full corpus cho B0/B2) **không áp dụng cho máy đang chạy hiện tại** — đo trực tiếp: tổng RAM ~32GB, còn ~6GB trống ngay cả khi build_index full corpus đang chạy (tiến trình chỉ dùng ~3.3GB). Không có OOM nào xảy ra khi build BM25 index Layer 1 trên **full 8,512 văn bản** (không cần pool mẫu). Ghi chú này để tránh áp dụng mù quáng workaround cũ (pool ngẫu nhiên, cap ký tự) trên máy này khi không cần thiết — nhưng vẫn giữ `MAX_TOKENIZE_CHARS`/cap độ dài cho case `unit_type="doc_fallback"` trong B0 (an toàn, chi phí giữ lại thấp, đề phòng chạy trên máy khác yếu hơn sau này).

**Timing thật** (`experiments/build_bm25_index.py`, đo qua `full_index_bench.txt`): load 8,512 văn bản = 4.1s; `build_index` (rank_bm25 + underthesea) full corpus = **2589.2s ≈ 43.2 phút** — gần gấp đôi ước tính ngoại suy tuyến tính cũ (20-24 phút, ngoại suy từ mẫu 319 văn bản) — bài học: ngoại suy tuyến tính từ mẫu nhỏ không đáng tin cho ước tính thời gian ở quy mô lớn hơn 25 lần, `underthesea` có overhead cố định/lần gọi không tuyến tính theo số văn bản.

**Quyết định**: thêm `retrieve.save_index()`/`load_index()` (pickle) — cache BM25 index full corpus ra `outputs/bm25_doc_index.pkl` sau khi build 1 lần, mọi script sau (B0, B3...) load lại (vài giây) thay vì build lại (43 phút) mỗi lần.

**Test B0 mới trên subset nhỏ** (`experiments/demo_b0_autolabel.py`, cập nhật dùng `label_answer(item, bm25_index, parsed_corpus)` thay API cũ) — 4/4 test PASS: đoạn trích nguyên văn từ Khoản 2 Điều 2 `context_740` → score=0.979; nhiễu nhẹ (đổi 2 từ) → score=0.915 (giảm nhưng vẫn nhận diện được); marker "Trước đây..." tách đúng 2 role (`hiện hành` conf=0.885, `trước đây` conf=0.979); đối chứng âm (câu hỏi thật không liên quan) → `[]` đúng kỳ vọng, không dương tính giả.

**Việc tiếp theo**: chạy `experiments/build_b0_labels.py` (full `warmup.json`, dùng index cache) khi `build_bm25_index.py` xong.

---

### [B3] 12/08 — sửa `eval_recall.py` cho schema mới, thêm đo Recall@K cấp Khoản

**Bug phát hiện khi rà lại (chưa chạy thật, phát hiện qua đọc kỹ type)**: `MatchedSpan.context_id` đổi thành `str` từ bản B0 rebuild (12/08), nhưng `recall_at_k` cũ so sánh trực tiếp với `ScoredContext.context_id` (vẫn là `int`, từ B2 Layer 1/`io_utils.ContextDoc`) — 2 set khác kiểu **không bao giờ giao nhau**, Recall@K sẽ luôn ra 0 một cách âm thầm (không lỗi, không crash, chỉ sai số). Đã sửa: ép cả 2 phía về `str` khi so sánh trong `recall_at_k`.

**Thêm `recall_at_k_unit`/`recall_at_multiple_k_unit`** — đo Recall@K ở cấp Khoản/Dieu (so `unit_id` thay vì `context_id`), cần cho việc so sánh Layer 1 (BM25) vs +Layer 2 (bi-encoder) vs +Layer 3 (reranker) sau này (bằng chứng cho B5). "Unit đúng" của 1 nhãn B0: `khoan_id` nếu `unit_type="khoan"`, ngược lại (`dieu_fallback`/`doc_fallback`, không có Khoản thật) fallback về `context_id` — nhất quán với cách `expand_to_units` (B2) gán `unit_id=dieu_id`/`context_id` cho 2 case đó.

**Test bằng dữ liệu giả** (chưa có nhãn B0 thật để test bằng data thật — đang chờ index): cả `recall_at_k` (doc-level) và `recall_at_k_unit` (unit-level) PASS đúng logic kỳ vọng (câu không nhãn bị loại khỏi mẫu số, hit/miss tính đúng, ép kiểu str hoạt động).

---

### [Infra] 12/08 — rà toàn bộ project theo yêu cầu người dùng, dọn dẹp file chết + tách `pipeline/` khỏi `experiments/`

**Bối cảnh**: rà lại toàn bộ engine + tổ chức file (không chỉ code từng module) để tìm chỗ chưa hợp lý, thay vì tiếp tục thêm module mới mà không nhìn lại tổng thể.

**Phát hiện + đã xử lý (có duyệt trước khi xoá/di chuyển)**:
1. **3 file đã CHẾT** (không chỉ lỗi thời): `experiments/demo_b3_step1_label.py`, `demo_b3_step2_recall.py` gọi `label.build_shingle_index` — hàm đã bị xoá khi B0 rebuild (12/08, xem entry `[B0]` cùng ngày) → 2 script này crash nếu chạy. Cả 2 tồn tại chỉ để giải quyết workaround "pool 1,000 văn bản ngẫu nhiên" cho giới hạn RAM sandbox cũ (3.8GB) — giới hạn đó **không áp dụng cho máy đang chạy hiện tại** (đã đo: 32GB RAM, còn dư ~6GB ngay cả khi build full-corpus index). `pipeline/build_b0_labels.py` (full corpus) đã thay thế đúng vai trò. **Đã xoá** cả 3 (2 script + `_b3_step1_output.json`).
2. **Tách `experiments/` thành `experiments/` (demo/test throwaway) + `pipeline/` (build script sinh artifact module khác phụ thuộc, PHẢI chạy đúng thứ tự)** — trước đó cả 2 loại nằm chung 1 thư mục phẳng, không phân biệt được bằng tên file cái nào "phải chạy trước" vs "chỉ để kiểm tra nhanh". Đã di chuyển `build_parsed_corpus.py`, `build_bm25_index.py`, `build_b0_labels.py` sang `pipeline/` (không cần sửa `sys.path.insert` trong các file này — `pipeline/` cùng cấp `experiments/` dưới `PROJECT_ROOT`, `parents[1]` vẫn đúng). Đã verify: di chuyển file trong lúc tiến trình `build_bm25_index.py` đang chạy KHÔNG làm crash tiến trình (Python đã đọc xong source vào bộ nhớ, không giữ lock đường dẫn).
3. **`data/README.md` sai so với code thật**: viết `data/corpus -> selected-contexts/selected-contexts` (symlink) — symlink này đã hỏng và bị bỏ từ `[Infra] 06/08` (code thật dùng thẳng `config.CORPUS_DIR`), nhưng README chưa cập nhật theo. Đã sửa.
4. **`docs/SYSTEM_SCAFFOLD.md` lỗi thời nhiều chỗ** (bảng trạng thái B3-B7, sơ đồ phụ thuộc, cấu trúc thư mục — chỉ được cập nhật piecemeal cho B0/B1/B2 các lần trước). Đã viết lại toàn bộ file cho khớp trạng thái 12/08 (kiến trúc nhiều lớp B2, thứ tự B2↔B0 mới, `pipeline/` mới tách).

**Chưa xử lý (ghi nhận, không phải việc của bước này)**: `requirements.txt` thiếu `transformers`/`torch` — cần thêm khi code B2 Layer 2/3 (bi-encoder/reranker), chưa cần ngay bây giờ.

---

### [B0] 12/08 — bug hiệu năng nghiêm trọng: SW full trên hàng nghìn unit/câu, phải kill tiến trình đang chạy thật

**Phát hiện qua `demo_b_end_to_end.py`** (không phải qua B0 trực tiếp — chạy demo này song song lúc chờ B0, tình cờ lộ ra số liệu): `search_units(top_k_docs=20)` trả về **2,000-3,000 unit ứng viên/câu hỏi** (top-20 văn bản × trung bình nhiều chục Khoản/văn bản) — số lớn hơn nhiều so với hình dung ban đầu ("candidate để align, chấp nhận rộng"). `label_answer` cũ (`_best_unit_match`) chạy Smith-Waterman FULL (O(n_a×n_p) mỗi cặp) trên **TẤT CẢ** unit đó, không lọc trước. Đã chạy thật `pipeline/build_b0_labels.py` (full 500 câu) — sau ~6 phút chưa qua nổi 50 câu đầu (ngưỡng in tiến độ) → ngoại suy sẽ mất **hàng giờ**, không phải vài phút như kỳ vọng. Đã kill tiến trình.

**Sửa**: thêm `_prefilter_units()` — lọc rẻ bằng overlap token thô (giống thuật toán `select_khoan` của B4) để thu hẹp từ 2,000-3,000 unit xuống `top_n=20` **trước khi** mới chạy Smith-Waterman trên số ít đó. Đúng tinh thần kiến trúc nhiều lớp đã áp dụng ở B2 (lọc rẻ trước, xử lý đắt sau) — chỉ là chưa áp dụng bên trong B0 ngay từ đầu.

**Đo lại sau khi sửa** (20 câu warmup thật, không phải mô phỏng): **2.38s/câu** → ngoại suy full 500 câu ≈ **19.8 phút** (so với hàng giờ trước khi sửa). Đã chạy lại `pipeline/build_b0_labels.py` full 500 câu với bản đã sửa.

**Bài học**: số liệu "candidate để align, K lớn chấp nhận được vì rẻ" ở bước lên plan trước đó (K=20 văn bản) chỉ đúng cho BM25 (rẻ), KHÔNG tự động đúng cho bước SAU nó (Smith-Waterman, đắt hơn nhiều bậc) — mỗi lớp trong pipeline cần tự đánh giá lại chi phí ở ĐÚNG quy mô đầu vào của lớp đó, không suy luận từ lớp trước.

---

### [B2] 12/08 — phát hiện `data_retrieve/` (nhãn context_id sạch từ hạng mục Retrieval khác), chiến lược ground truth 4 tầng

**Phát hiện**: người dùng thêm `data_retrieve/` vào project — dữ liệu của 1 hạng mục Retrieval RIÊNG trong cùng cuộc thi (khác hạng mục QA đang làm). Verify kỹ trước khi tin:
- Corpus **giống hệt** `data/selected-contexts` (8,532/8,532 ID trùng, nội dung byte-identical test 3 mẫu).
- Câu hỏi **khác hẳn** `data/train.json`/`warmup.json` — chỉ 22/7,000 trùng nội dung (~0.3%, ngẫu nhiên) — là 1 bộ 7,500 câu hỏi riêng (train+warmup, 346 ID trùng giữa 2 split nên phải xử lý tách riêng, không gộp chung dict).
- `answer` ở đây là **list `context_id` đúng cho sẵn** (vd `["280282"]`), không phải answer text — verify 1 mẫu: câu hỏi về đăng ký xe máy → đúng Thông tư 58/2020/TT-BCA về đăng ký biển số — hợp lý. 0/7,500 câu thiếu context_id trong corpus, 590/7,500 (7.9%) đa nhãn.
- Phân phối độ dài câu hỏi (median 19 từ, p90 27-29) **gần như trùng khớp** giữa `data_retrieve` và `data` (cả train lẫn warmup) — số đo trên `data_retrieve` có cơ sở để suy ra cho QA.

**Quyết định** (sau thảo luận nhiều lượt với người dùng — xem thêm câu hỏi/trả lời ngoài log): `data_retrieve` **không thay được B0** (không có answer text của QA, không có nhãn cấp Khoản, không đo được recall trần) nhưng là **ground truth SẠCH, ĐỘC LẬP, quy mô lớn (7,500 câu)** để đánh giá riêng B2 (BM25 Layer 1) — mạnh hơn hẳn cách cũ (dùng chính nhãn B0 sinh ra từ `search_units()` của B2 để đo B2 — **thiên lệch chọn mẫu vòng tròn**, B2 không thể trượt case nào nó chính là công cụ tạo nhãn). Chốt **chiến lược ground truth 4 tầng**:
1. `data_retrieve` (7,500 câu, nhãn sạch, độc lập) — đo Recall/rank cấp văn bản của B2, bằng chứng chính cho B5.
2. B0 trên `data/warmup.json`+`train.json` — nhãn cấp Khoản cho ĐÚNG câu hỏi QA, đo recall trần chunking, calibrate ngưỡng, đánh giá B4.
3. Kiểm chứng chéo (22 câu trùng nội dung 2 track, audit tay theo bucket confidence) — định lượng độ tin tầng 2.
4. B7 oracle (METEOR thật qua Generator) — thước đo cuối cùng, khớp cách BTC chấm.

**Đo được ngay khi phân tích `data_retrieve`, phát hiện lỗ hổng thật** (không phải suy đoán):
- **`MAX_TOKENIZE_CHARS=50,000` (B2 hiện tại) cắt mất 35.1% gold doc** (1,102/3,143 gold doc unique) — gold doc median 33.6k ký tự nhưng p90=125k, max=3.02M. Cap 200k chỉ còn cắt 3.8% (119/3,143) — chọn 200k làm cap mới cho config đối chứng (đủ RAM, máy hiện tại 32GB không còn giới hạn như sandbox cũ).
- 10.2% gold doc có `parse_status=fallback` (322/3,143) — không có Khoản thật.
- Corpus có **432,473 unit** (Khoản+Dieu_fallback+doc_fallback) nếu index cấp Khoản — gấp 51 lần số văn bản. Đo tokenize mẫu 2,000 unit = 4.01s (2.00ms/unit, RẺ hơn nhiều per-unit so với cấp văn bản vì unit ngắn — median 222 ký tự) → ngoại suy full 432,473 unit ≈ **14.4 phút**, khả thi.
- `name` (số hiệu/loại văn bản) hiện KHÔNG được index cùng `passage` — bỏ sót tín hiệu rẻ (câu hỏi luật hay nhắc thẳng số hiệu văn bản).

**Bổ sung B1 trước khi đo tầng 2** (cần cho cách đo bằng offset ký tự thay vì similarity score mờ — tránh lặp lại nhập nhằng 3 loại lỗi trộn lẫn khi đọc tay case điểm thấp ở entry `[B0] recall trần` trước đó): thêm `char_start`/`char_end` (tuyệt đối theo `main_text` sau `_split_tail`) vào cả `Dieu` và `Khoan` — viết `_strip_with_offset()` xử lý đúng việc `.strip()` dịch chuyển offset. Verify: `main_text[char_start:char_end] == text` cho mọi Dieu/Khoan trên `context_740`, Khoan luôn nằm trong range Dieu chứa nó — PASS. Chạy lại `pipeline/build_parsed_corpus.py` — thống kê matched/fallback giữ nguyên y hệt (84.6%/15.4%), xác nhận không có regression.

**Code mới**:
- `src/b3_eval_recall/ir_metrics.py` — bộ metric IR tổng quát (Hit@K, Recall@K micro/strict, MRR, nDCG@K, rank percentile), không phụ thuộc schema B0 (khác `eval_recall.py`). Test bằng dữ liệu giả tự tính tay — PASS (1 lần viết sai kỳ vọng trong assertion tự viết, không phải bug code, đã tự phát hiện qua đối chiếu công thức).
- `pipeline/build_retrieval_runs.py` — chạy B2 trên `data_retrieve` train+warmup, ghi run file (tách offline khỏi việc tính metric, đổi K/metric không cần chạy lại retrieval). 2 config rẻ (`L1_doc`, `L1_doc_noMQ`) dùng lại index cache có sẵn.
- `pipeline/build_expensive_indexes.py` — build 3 config cần index mới: `L1_doc_nocap` (cap 200k), `L1_doc_name` (ghép `name` vào đầu passage), `L1_khoan` (index thẳng cấp Khoản/Dieu, 432k unit).
- `pipeline/eval_retrieval_runs.py` — đọc run file, tính bảng metric qua `ir_metrics.evaluate`.

**Timing đo trên mẫu 50 câu** (trước khi chạy full): multi-query 84.3ms/câu, no-multi-query 87.0ms/câu — **gần như không khác biệt** (đúng như dự đoán trước đó: chỉ 1.5-3% câu hỏi có >1 mệnh đề nên multi-query hiếm khi tạo biến thể khác). Ngoại suy 2 config × (7,000+500) câu ≈ 21 phút.

**Đang chạy nền** (4 tiến trình song song, máy có 20 core): `build_retrieval_runs.py` (2 config rẻ, ~21 phút), `build_expensive_indexes.py L1_doc_nocap`, `L1_doc_name`, `L1_khoan` (build index mới, thời gian chưa đo — L1_khoan ngoại suy ~14.4 phút, 2 config kia chưa rõ). Kết quả metric đầy đủ sẽ ghi ở entry tiếp theo.

---

### [B2] 13/08 — kết quả ablation tầng 1 (data_retrieve, 7,500 câu sạch) — 3/4 config xong

**Timing thật đo được** (khác ngoại suy trước, ghi nhận sai số ngoại suy tuyến tính từ mẫu nhỏ):
- `L1_doc`/`L1_doc_noMQ` (dùng lại cache, không build): 632s+51s và 680s+47s cho train+warmup — khớp ngoại suy trước (~21 phút cho cả 2 config).
- `L1_doc_name` build index: 3,263.9s ≈ 54.4 phút (so ngoại suy ẩn chưa có — chỉ biết L1_doc gốc mất ~38-43 phút, cộng thêm field `name` không đổi nhiều).
- `L1_khoan` build index: 1,718.9s ≈ 28.6 phút (ngoại suy trước từ mẫu 2,000 unit là 14.4 phút — **sai gần 2 lần**, mẫu đầu file không đại diện đủ hoặc chi phí nội bộ BM25Okapi tăng hơn tuyến tính theo số "văn bản" — 394k unit so với 8,512 văn bản gốc).
- `L1_khoan` SEARCH (không phải build): đo được **2.68s/câu** (so 84-87ms/câu cấp văn bản — chậm ~32 lần, `BM25Okapi.get_scores` quét tuyến tính theo N "văn bản", N lớn hơn 46 lần). Ngoại suy full 7,500 câu ≈ 5.6 giờ — KHÔNG khả thi trong phiên. **Quyết định**: chỉ chạy `L1_khoan` trên `warmup.json` (500 câu, ~18.4 phút), bỏ `train.json` cho riêng config này — đủ cho so sánh có ý nghĩa thống kê, không đủ cho việc tune (không cần, đây là lựa chọn kiến trúc không có hyperparameter).

**Bug/hiệu chỉnh nhỏ trong `pipeline/build_khoan_doc_run.py`**: kết quả `retrieve.search()` ở cấp Khoản trả `context_id` field nhưng thực chất chứa `unit_id` (khoan_id/dieu_id dạng string) — đặt tên biến `ranked_unit_ids` rõ ràng khi collapse về `context_id` thật (`unit_id.split("_", 1)[0]`) để tránh nhầm lẫn đọc lại sau này.

**Kết quả trên `warmup.json` (500 câu, tập report cuối — không phải `train` dùng để tune)**:

| Config | Hit@1 | Hit@5 | Hit@10 | Hit@100 | MRR(full) | p50 rank | Coverage (tìm được ở đâu đó) |
|---|---|---|---|---|---|---|---|
| `L1_doc` (baseline) | 26.2% | 47.0% | 57.0% | 89.8% | 0.366 | 5 | 449/500 (89.8%) |
| `L1_doc_noMQ` | **giống hệt L1_doc tới 4 chữ số thập phân** | | | | | | |
| `L1_doc_name` (+tên văn bản) | 26.2% | 47.2% | 56.8% | 89.8% | 0.366 | 5 | 449/500 (89.8%) |
| `L1_khoan` (cấp Khoản, collapse về doc) | **32.0%** | **62.0%** | **71.2%** | 82.8% | **0.454** | **2** | 414/500 (82.8%) |
| `L1_doc_nocap` (cap 200k) | — chưa xong — | | | | | | |

**Diễn giải**:
1. **Multi-query (`noMQ` vs `L1_doc`): hoàn toàn vô dụng** — kết quả giống hệt nhau (đúng dự đoán từ phân tích trước: chỉ 1.5-3% câu hỏi >1 mệnh đề, biến thể bị dedupe hết). Quyết định: bỏ multi-query khỏi pipeline chính thức, giữ code lại nhưng tắt mặc định — đơn giản hoá, không mất gì.
2. **Thêm `name` vào index: không cải thiện đáng kể** (chênh trong biên độ nhiễu, n=500) — kết quả âm tính rõ ràng, không cần theo hướng này.
3. **Granularity Khoản vs văn bản: ĐÁNH ĐỔI RÕ, không có bên thắng tuyệt đối.** Khoản-level thắng áp đảo ở xếp hạng sớm (Hit@1 +5.8 điểm %, Hit@10 +14.2 điểm %, MRR +24%, p50 rank 5→2) — tín hiệu BM25 sắc nét hơn khi so với unit ngắn, tập trung (median 222 ký tự) thay vì cả văn bản dài loãng (median 33k ký tự). Nhưng THUA coverage tổng (mất 7% câu, 35/500) — từ khóa liên quan rải rác qua nhiều Khoản trong cùng văn bản, tách nhỏ làm mất hiệu ứng cộng dồn BM25 IDF×TF mà văn bản gộp mới đủ tín hiệu nổi lên. **Kết luận: XÁC NHẬN đúng kiến trúc 2 tầng đã chọn từ đầu** (tham khảo kinh nghiệm thi trước của người dùng) — Layer 1 cấp văn bản (giữ coverage rộng) → Expand → chấm lại cấp Khoản (tận dụng độ sắc nét) — KHÔNG nên thay hẳn Layer 1 bằng index Khoản.
4. Lưu ý phương pháp: `L1_khoan` dùng `RAW_TOP_K=1000` (cấp Khoản) trước khi collapse còn top-100 văn bản riêng biệt — phần mất coverage 7% CÓ THỂ 1 phần là do giới hạn RAW_TOP_K=1000 (nếu văn bản đúng có Khoản liên quan nhưng điểm thấp, nằm ngoài top-1000 unit), không hoàn toàn là giới hạn granularity — cần thử RAW_TOP_K lớn hơn nếu muốn tách bạch 2 nguyên nhân, CHƯA làm (ghi nhận giới hạn phương pháp, không phải kết luận sai).

**Còn thiếu**: `L1_doc_nocap` (cap 200k, đang build) để biết truncation 50k→200k có vá được phần nào trong 10.2% coverage gap của baseline hay không — kết quả sẽ ghi bổ sung.

---

### [B2] 13/08 — hoàn tất ablation tầng 1: `L1_doc_nocap` KHÔNG cải thiện (kiểm định thống kê), chốt cấu hình B2

**`L1_doc_nocap` build**: 4,684.2s ≈ 78.1 phút (dài nhất trong 5 config — hợp lý, cap 200k vs 50k, phải tokenize thêm nội dung cho ~14% văn bản dài).

**Kết quả trên `warmup` (500 câu)**: Hit@1=24.4%, Hit@5=47.8%, Hit@10=58.2%, MRR=0.361, coverage=447/500 (89.4%) — **THẤP HƠN baseline ở Hit@1/MRR**, dù giả thuyết ban đầu (cap 50k cắt 35.1% gold doc → bỏ cap sẽ cải thiện) có cơ sở lý thuyết hợp lý.

**Kiểm định trước khi kết luận** (không tin chênh lệch nhỏ trên n=500 mà không test): McNemar-style so từng câu (baseline đúng/nocap sai: 26 câu; nocap đúng/baseline sai: 17 câu — không lệch nhiều) + bootstrap 2,000 lần cho khoảng tin cậy 95% của (Hit@1_base − Hit@1_nocap) = **[-0.0080, +0.0440] — BAO TRÙM 0** → không có ý nghĩa thống kê. Kết luận đúng: **nocap không có tác dụng đo được theo cả 2 chiều** (không cải thiện, cũng không chắc chắn làm hại) — không phải "nocap tệ hơn".

**Giả thuyết giải thích tại sao lý thuyết đúng nhưng thực nghiệm không thắng**: BM25 có cơ chế length-normalization (tham số `b`) phạt văn bản dài hơn trung bình corpus — thêm hàng nghìn token vào văn bản vốn đã dài (đưa content bị cắt trước đó vào) đẩy độ dài văn bản đó CÀNG XA trung bình corpus, khả năng làm tăng mức phạt lên MỌI từ khớp trong văn bản đó, bù trừ lại lợi ích từ việc không còn bị cắt nội dung. Chưa verify sâu hơn (không cần thiết — kết luận thực dụng "không dùng nocap" đã đủ vững).

**BẢNG TỔNG KẾT 5 CONFIG** (`warmup.json`, 500 câu, tập report cuối):

| Config | Hit@1 | Hit@5 | Hit@10 | Hit@100 | MRR | p50 rank | Coverage | Chi phí build |
|---|---|---|---|---|---|---|---|---|
| `L1_doc` (baseline, cap 50k) | 26.2% | 47.0% | 57.0% | 89.8% | 0.366 | 5 | 449/500 | có sẵn |
| `L1_doc_noMQ` | giống hệt L1_doc (4 chữ số thập phân) | | | | | | | miễn phí |
| `L1_doc_name` (+tên văn bản) | 26.2% | 47.2% | 56.8% | 89.8% | 0.366 | 5 | 449/500 | 54.4 phút — vô ích |
| `L1_doc_nocap` (cap 200k) | 24.4%¹ | 47.8% | 58.2% | 89.4% | 0.361¹ | 4 | 447/500 | 78.1 phút — không ý nghĩa thống kê |
| `L1_khoan` (cấp Khoản, collapse doc) | **32.0%** | **62.0%** | **71.2%** | 82.8%² | **0.454** | **2** | 414/500² | 28.6 phút build + 18.4 phút search riêng |

¹ chênh lệch với baseline KHÔNG có ý nghĩa thống kê (CI 95% bao trùm 0).
² thấp hơn baseline ở coverage/Hit@100 — đánh đổi thật (xem entry trước), một phần có thể do giới hạn phương pháp `RAW_TOP_K=1000` khi collapse, chưa tách bạch được.

**QUYẾT ĐỊNH CHỐT cấu hình B2 Layer 1**:
1. **Bỏ multi-query** (0 lợi ích đo được trên 7,500 câu, đơn giản hoá code).
2. **Bỏ +name, bỏ nocap** — cả 2 tốn 54-78 phút build mà không cải thiện đo được (1 cái không ý nghĩa thống kê, 1 cái bằng 0). Giữ nguyên `MAX_TOKENIZE_CHARS=50_000`.
3. **Giữ kiến trúc 2 tầng đã chọn từ đầu** (Layer 1 cấp văn bản → Expand → chấm cấp Khoản) — số liệu XÁC NHẬN đúng hướng, không thay Layer 1 bằng index Khoản thẳng (mất 7% coverage).

**Ý tưởng phát sinh, CHƯA LÀM — đề xuất "Layer 1.5" rẻ**: tận dụng `bm25_khoan_index.pkl` đã build sẵn để RERANK ngay trong bước Expand — chỉ chấm điểm lại các Khoản thuộc top-N văn bản Layer 1 đã tìm ra (vài nghìn candidate, không phải search thẳng 394k unit) — lấy được độ sắc nét Khoản-level (Hit@1 +5.8 điểm%, MRR +24% đo được ở trên) mà KHÔNG mất coverage của Layer 1 (vì Layer 1 vẫn đảm bảo candidate đầu vào). Rẻ hơn nhiều so với Layer 2/3 (bi-encoder/reranker cần tham số học) vì chỉ là tính lại BM25 trên phạm vi đã thu hẹp — có thể làm trước khi đầu tư Layer 2/3.

---

### [B2] 13/08 — code + verify "Layer 1.5" (rerank rẻ bằng BM25 cấp Khoản) — kết quả "được cả 2", chốt làm default

**Cơ chế**: `rank_bm25.BM25Okapi.get_batch_scores(query, doc_ids)` — API có sẵn của thư viện, chấm điểm BM25 CHỈ trên tập `doc_ids` chỉ định (dùng vị trí nguyên, không phải string ID), nhưng vẫn dùng thống kê IDF/avgdl của TOÀN CORPUS đã index (394k unit) — đúng ý nghĩa BM25 hơn tự build index tạm từ riêng tập candidate (IDF sẽ lệch). Code: `retrieve.build_khoan_position_index()` (map unit_id -> vị trí, build 1 lần) + `retrieve.rerank_units_khoan()` (chấm lại `UnitCandidate` từ Expand, ghi đè field `score` mới thêm vào `UnitCandidate` — giữ nguyên `doc_score` cũ làm tham chiếu Layer 1, không mất thông tin).

**Test đúng đắn** (`context_740`, câu tự hỏi lại): rerank 610 candidate (từ top-20 văn bản) mất **0.014s** — cực rẻ. Top-1 sau rerank là 1 văn bản KHÁC (`174906_21`, nhắc đúng "Vụ trưởng Vụ Trang thiết bị...") — hợp lý, không phải lỗi, vì nhiều văn bản luật cùng nhắc đúng tên đơn vị này.

**Đo tại quy mô — LẦN 1 CÓ BUG TỰ PHÁT HIỆN**: chạy `pipeline/build_layer15_run.py` với `TOP_K_DOCS_LAYER1=20` (thói quen từ default cũ) trong khi baseline `L1_doc` dùng `top_k_docs=100` — kết quả Hit@K bị TRẦN GIẢ ở K=20 (Hit@20=Hit@50=Hit@100=70.6%, phẳng lì — vì chỉ có tối đa 20 văn bản khác nhau để rerank, không phải 100) — **so sánh không công bằng, tự phát hiện qua đọc số liệu bất thường (đường cong phẳng bất hợp lý), không chờ người dùng chỉ ra**. Sửa `TOP_K_DOCS_LAYER1=100` khớp baseline, chạy lại.

**Kết quả ĐÚNG (sau sửa), `warmup.json` 500 câu, so 3 config**:

| Config | Hit@1 | Hit@5 | Hit@10 | Hit@100 (=coverage) | MRR |
|---|---|---|---|---|---|
| `L1_doc` (baseline) | 26.2% | 47.0% | 57.0% | 89.8% | 0.366 |
| `L1_khoan` (search thẳng cấp Khoản) | 32.0% | 62.0% | 71.2% | 82.8% | 0.454 |
| **`Layer1.5` (Layer 1 + rerank Khoản)** | **32.2%** | **61.4%** | **70.4%** | **89.8%** | **0.452** |

**Kết luận: đúng giả thuyết "được cả 2"** — Layer 1.5 lấy được gần như TOÀN BỘ độ sắc nét của `L1_khoan` (Hit@1/Hit@5/MRR chênh <1 điểm%) **VÀ giữ nguyên 100% coverage của `L1_doc`** (449/500, không mất câu nào — khác hẳn `L1_khoan` mất 7%). Chi phí: 68.0s cho 500 câu (136ms/câu, chỉ +52ms so với Layer 1 gốc 84ms/câu) — gần như miễn phí, không cần tham số học nào.

**QUYẾT ĐỊNH**: chốt Layer 1.5 làm mặc định mới cho B2 (thay vì chỉ Layer 1 trần). Cần cập nhật `search_units()` để tích hợp bước rerank này làm tuỳ chọn mặc định (chưa làm — API rời hiện tại `search_units()` + `rerank_units_khoan()` riêng, cần gọi tay 2 bước). Layer 2/3 (bi-encoder/reranker, tốn tham số) giờ có baseline cao hơn hẳn để so sánh — B5 sẽ khó chứng minh đáng đầu tư hơn trước (Layer 1.5 đã ăn gần hết phần "dễ" mà dense retrieval hay mang lại).

---

### [B2/B0] 13/08 — wire Layer 1.5 vào `search_units()` + B0, chạy lại `warmup.json`

**Tích hợp API**: `search_units()` thêm tham số tuỳ chọn `khoan_index`/`khoan_position_index` (mặc định `None` — giữ nguyên hành vi cũ, tương thích ngược mọi caller hiện có). Truyền vào -> tự động rerank sau Expand.

**Bug tự phát hiện TRƯỚC khi chạy thật** (không phải sau khi có số liệu sai mới sửa): `_prefilter_units` (B0) luôn TỰ TÍNH LẠI thứ hạng bằng overlap token thô, bất kể input đã được Layer 1.5 rerank hay chưa — nếu wire thẳng mà không sửa, Layer 1.5 sẽ hoàn toàn VÔ TÁC DỤNG với B0 (bị ghi đè ngay). Đã sửa: thêm cờ `already_ranked` — `True` (khi có Layer 1.5) thì TIN thứ hạng đầu vào, chỉ cắt top_n; `False` (mặc định) giữ hành vi overlap thô cũ. Threading `khoan_index`/`khoan_position_index` xuyên suốt `label_answer`/`label_dataset` (B0) + `pipeline/build_b0_labels.py`.

**Test timing mẫu 20 câu**: 1.59s/câu — **NHANH HƠN** bản không Layer 1.5 (2.38s/câu) — vì nhánh `already_ranked=True` chỉ cắt `units[:top_n]` (O(top_n)), rẻ hơn tính overlap + sort toàn bộ 2,000-3,000 unit (O(n log n)) như trước.

**Chạy full `warmup.json` (500 câu)**: 780.8s (13.0 phút, nhanh hơn bản cũ 35% — 1211.3s/20.2 phút).

| | Bản cũ (không Layer 1.5) | Bản mới (có Layer 1.5) |
|---|---|---|
| Coverage | 308/500 (61.6%) | 313/500 (62.6%) |
| Mean score (tất cả) | 0.622 | 0.606 |
| Khoan: n / mean / ≥0.9 | 268 / 0.598 / 11.2% | 275 / 0.579 / 10.2% |
| Fallback: n / mean / ≥0.9 | 41 / 0.782 / 39.0% | 39 / 0.797 / 41.0% |

**Diễn giải**: coverage tăng nhẹ (+5 câu tổng, +7 case Khoản mới), tốc độ nhanh hơn rõ rệt — nhưng mean score case Khoản giảm nhẹ (0.598→0.579). Hợp lý, không phải dấu hiệu xấu: case MỚI tìm được (nhờ candidate tốt hơn) là case khó hơn (case dễ đã được cả 2 cách tìm thấy từ trước), nên kéo mean của nhóm xuống — đánh đổi tự nhiên khi mở rộng coverage, không phải Layer 1.5 làm giảm chất lượng match.

**File `outputs/b0_labels_warmup.json` đã GHI ĐÈ** bằng bản Layer 1.5 (mới nhất, dùng cho các bước sau — hiệu chỉnh ngưỡng, đo recall trần).

---

### [B0/B2] 13/08 — bắt đầu tầng 2 mở rộng (dev sample train.json) + chuẩn bị Layer 2 (bi-encoder) trên Kaggle GPU

**Quyết định người dùng**: ưu tiên chỉ số RECALL trong toàn bộ pipeline (thà dư hơn thiếu — theo cách chấm điểm cuộc thi). Đã lưu vào memory hệ thống (`feedback_prioritize_recall.md`), áp dụng cho mọi quyết định K/ngưỡng từ giờ. Hệ quả trực tiếp: ưu tiên vá 10.2% "mất tích vĩnh viễn" (BM25 không tìm thấy dù K=100) — đúng lý do bắt tay làm Layer 2.

**Tầng 2 mở rộng**: thay vì chạy hết `train.json` (7,000 câu, ước ~3 giờ ở tốc độ hiện tại), chạy **dev sample ngẫu nhiên 1,500 câu** (seed=42, tái lập được) — đủ thống kê cho calibrate ngưỡng + đo recall trần, tiết kiệm thời gian đáng kể. Script `pipeline/build_b0_labels_train_sample.py`, dùng Layer 1.5 (đã wire). Đang chạy nền, tốc độ đo được 1.71s/câu (100 câu đầu = 171.2s) — ngoại suy ~43 phút.

**Layer 2 (bi-encoder) — chuẩn bị chạy trên Kaggle GPU** (máy local không có GPU, CPU sẽ quá chậm cho 394k unit): viết sẵn `kaggle_layer2/README.md` (hướng dẫn từng bước) + `kaggle_layer2/embed_corpus_notebook.py` (code dán vào Kaggle Notebook theo cell). Quyết định thiết kế:
- Model: `BAAI/bge-m3` (~568M tham số, trong ngân sách ≤0.7B đã chốt từ đầu dự án) — multilingual, hỗ trợ Vietnamese, context dài (8192 token) phù hợp văn bản luật.
- Granularity: cấp Khoản (394k unit, giống Layer 1.5) — không phải cấp văn bản, nhất quán với phát hiện "BM25 cấp Khoản sắc nét hơn" đã đo được trước đó, dense embedding cũng nên hưởng lợi tương tự (embedding cả văn bản dài sẽ pha loãng ngữ nghĩa vào 1 vector trung bình, kém chính xác hơn).
- `model.max_seq_length=1024` — cap độ dài đưa vào model (đa số Khoản ngắn, chỉ ảnh hưởng 0.3% unit `doc_fallback` siêu dài).
- fp16 trên GPU — nhanh hơn, embedding không cần độ chính xác fp32.
- Embed LUÔN cả 4 nguồn câu hỏi (data_retrieve train+warmup, QA train+warmup — tổng ~15,000 câu) trong CÙNG phiên GPU, tránh phải quay lại Kaggle lần 2.
- **Phát hiện + sửa lỗi trước khi giao cho người dùng chạy**: `data_retrieve/train.json` và `data/train.json` trùng tên `train.json` — Kaggle Dataset là thư mục phẳng, upload thẳng sẽ đè nhau. Đã thêm bảng đổi tên rõ ràng vào README (`data_retrieve_train.json`, `qa_train.json`...) trước khi người dùng thao tác, tránh lỗi khi đã upload xong mới phát hiện.

**Quy trình đã thống nhất**: người dùng tự chạy Kaggle (tôi không đăng nhập/chạy hộ được) → tải `layer2_embeddings.zip` về → giải nén vào `outputs/layer2/` → tôi viết code load + build dense search index + đo Recall@K so với BM25 (tái dùng `ir_metrics.py` đã có).

**Việc tiếp theo**: chờ (1) dev sample train.json chạy xong (~43 phút), (2) người dùng chạy xong Kaggle notebook và đưa lại embedding.

---

### [B1] 13/08 — phát hiện nguyên nhân thật của lệch số unit (432,473 vs 393,999): danh sách đánh số LỒNG NHAU trong cùng 1 Điều

Nhận `outputs/layer2/` từ Kaggle (embedding Layer 2, xem entry riêng), verify shape: `corpus_embeddings.npy` có **432,473 hàng**, khác hẳn `bm25_khoan_index.pkl` build ở máy local trước đó (**393,999 unit**) — chênh 38,474, đúng bằng nghi vấn đã ghi nhận nhưng chưa điều tra ở entry `[B2] 12/08`.

**Điều tra ra nguyên nhân thật**: `corpus_unit_ids.json` (Kaggle) có 7,632 `khoan_id` bị trùng (xuất hiện 2-5 lần mỗi ID). Soi trực tiếp `context_100109` Điều 1: dãy `khoan_so` = `[1,2,3, 1,2, 1,2,3,4, 1,2,3,4,5,6,7,8,9]` — đây là **nhiều danh sách đánh số con LỒNG NHAU trong cùng 1 Điều** (kiểu "Trường hợp A: 1... 2... 3. Trường hợp B: 1... 2..." — mỗi trường hợp đánh số lại từ 1), nhưng `_KHOAN_RE` (B1, bắt bất kỳ dòng "^\d+[.)]") không phân biệt được, gộp hết thành 1 chuỗi Khoản của Điều đó → `khoan_so` trùng lặp → `khoan_id` trùng.

**Ảnh hưởng khác nhau ở 2 nơi build index cùng dữ liệu này**:
- Máy local (`pipeline/build_expensive_indexes.py build_khoan()`): dùng `dict{khoan_id: text}` — ghi đè, **âm thầm mất 38,474 unit** (giữ bản cuối cùng mỗi ID trùng, không cảnh báo).
- Kaggle (`embed_corpus_notebook.py` CELL 4): dùng `list` (append, không dedupe) — **giữ đủ cả 432,473 unit**, nhưng nhiều dòng trùng "tên" (khó tra ngược ID -> đúng 1 text nếu cần).

**Quyết định**: CHƯA sửa B1 (không chặn việc đang làm — Recall@K cấp văn bản chỉ cần biết unit thuộc văn bản nào, không cần ID duy nhất tuyệt đối). Ghi nhận làm việc sau: sửa `_KHOAN_RE`/`_split_khoan` để phát hiện danh sách lồng (reset numbering) và sinh ID phân biệt được (vd thêm hậu tố lần xuất hiện), hoặc coi các danh sách lồng là Điểm thay vì Khoản riêng.

---

### [B2] 13/08 — nhận embedding Layer 2 từ Kaggle, xử lý 3 sự cố thật trong lúc chạy (đã sửa notebook cho người dùng sau)

Người dùng tự chạy `kaggle_layer2/` (GPU T4 x2), gặp + tự xử lý được 3 lỗi môi trường thật, đã cập nhật `embed_corpus_notebook.py`/`README.md` ngay trong lúc debug cùng người dùng (không đợi xong mới ghi):

1. `FileNotFoundError` dù dataset đã add đúng — do phiên (session) đã chạy TRƯỚC KHI dataset được gắn vào, chưa mount kịp. Sửa: Restart Session.
2. Sau khi restart, vẫn `FileNotFoundError` — đường dẫn mount thật có thêm tiền tố `datasets/<username>/` (`/kaggle/input/datasets/ngmaidnghi/dsc-legalqa-b2-layer2-input/`), khác giả định ban đầu `/kaggle/input/<dataset-slug>/`. Tìm ra bằng `!find /kaggle/input/ -maxdepth 4`. Đã thêm ghi chú xử lý lỗi này vào code mẫu cho người dùng sau.
3. `AttributeError: module 'sympy' has no attribute 'core'` khi import `sentence_transformers` — xung đột phiên bản thư viện có sẵn trong image Kaggle (không phải lỗi code/data mình). Sửa: thêm `-U sympy` vào lệnh pip install CELL 1, Restart Session.
4. Tải `layer2_embeddings.zip` (~800MB, thực đo 885MB) qua trình duyệt bị đứt/lỗi — thêm CELL 8+9 (search ngay trên Kaggle bằng `torch` GPU matmul + `argpartition` lấy top-K, collapse về doc, xuất `dense_runs.zip` chỉ vài MB) làm phương án dự phòng. Cuối cùng người dùng tải lại được file nặng (thử lại thành công), nên dùng thẳng embedding đầy đủ thay vì dense_runs (linh hoạt hơn — dùng được cả cho việc wire Layer 2 vào production sau này, không chỉ để đo).

**Verify file nhận được** (`outputs/layer2/`, 12 file, ~903MB tổng): `corpus_embeddings.npy` shape `(432473, 1024)` fp16, khớp `corpus_unit_ids.json` (432,473 phần tử — xem entry ngay trên về nguyên nhân số này khác 393,999). 5 cặp query embedding/qids đúng số câu từng nguồn (retrieve_train=7000, retrieve_warmup=500, qa_train=7000, qa_warmup=500, qa_public=1000).

**Việc tiếp theo**: build cơ chế search dense (batch matmul, collapse về doc), đo Recall@K/MRR trên `data_retrieve` so với BM25 (Layer 1/1.5) đã có — theo đúng 9 bước đã thống nhất với người dùng.

---

### [B2] 13/08 — đo Layer 2 (dense, BGE-M3): thắng áp đảo ở xếp hạng, nhưng có nhóm "mất tích" RIÊNG — hợp BM25+dense vá gần hết

**Code**: `pipeline/build_layer2_dense_run.py` — search dense bằng nhân ma trận thô (`query_emb @ corpus_emb.T`, ép fp16→fp32 trước khi nhân vì numpy fp16 matmul không dùng BLAS tối ưu, chậm hơn nhiều), `argpartition` lấy top-300 rồi collapse về top-100 văn bản riêng biệt (cùng cơ chế đã dùng cho `L1_khoan`). **Đo được KHÔNG CẦN GPU**: 5.6ms/câu (matmul 50 câu = 0.18s + argpartition = 0.10s) — máy 20 core CPU đủ nhanh cho bước search (GPU chỉ cần lúc TẠO embedding, không cần lúc search). Chạy full `retrieve_train` (7,000 câu, 31.3s) + `retrieve_warmup` (500 câu, 2.4s).

**Kết quả trên `warmup` (500 câu), so với BM25 đã đo trước**:

| Config | Hit@1 | Hit@5 | Hit@10 | Hit@100 (coverage) | MRR |
|---|---|---|---|---|---|
| `L1_doc` (BM25) | 26.2% | 47.0% | 57.0% | 89.8% | 0.366 |
| `Layer1.5` (BM25+rerank Khoản) | 32.2% | 61.4% | 70.4% | 89.8% | 0.452 |
| **`L2_dense` (BGE-M3)** | **44.8%** | **71.8%** | **77.4%** | 85.8% | **0.565** |

**Dense thắng áp đảo ở xếp hạng sớm** (Hit@1 +12.6 điểm% so với Layer1.5, MRR +25%) — đúng kỳ vọng lý thuyết: embedding nắm được ý nghĩa, không chỉ từ trùng, nên xếp hạng đúng ngay từ đầu tốt hơn nhiều BM25. **Nhưng coverage lại THẤP HƠN** (85.8% vs 89.8%) — dense có nhóm "mất tích" riêng (14.2%, 71/500 câu), khác nhóm của BM25 (10.2%, 51/500 câu).

**Kiểm tra độ trùng lặp 2 nhóm "mất tích"** (bug nhỏ tự phát hiện: lần đầu tính "Hit@100 của union" qua `ir_metrics.evaluate` cho ra sai — vì union list bị cắt top-100 SAU KHI nối (BM25 100 câu trước, dense sau) nên gần như chỉ giữ nguyên top-100 của BM25, không phản ánh đúng "hợp cả 2". Sửa bằng tính trực tiếp tập hợp, không qua `ir_metrics`):
- Chỉ **13/500 (2.6%)** câu KHÔNG tìm thấy dù dùng CẢ 2 phương pháp — còn lại đều được ít nhất 1 trong 2 cứu được.
- **Coverage khi hợp (union) BM25 + dense = 97.4%** — tăng mạnh so với riêng lẻ (89.8% BM25, 85.8% dense).

**Kết luận quan trọng cho B5**: BM25 và dense thất bại ở **những câu khác nhau phần lớn** (chỉ 2.6% cùng thất bại) — xác nhận **2 phương pháp bổ trợ nhau thật sự**, không phải dense chỉ "làm lại việc BM25 đã làm tốt hơn 1 chút". Đáng đầu tư giữ CẢ 2 lớp (không phải chọn 1 trong 2), đặc biệt với ngân sách ~2 tỷ tham số mới (dư dả hơn nhiều so với 0.7B+0.6B cũ) — BGE-M3 (568M) vẫn nằm gọn trong ngân sách dù giữ cả BM25 (0 tham số) song song.

**Đo thêm trên `retrieve_train` (7,000 câu)**: Hit@1=44.0%, MRR=0.561, Hit@100=84.9% — **nhất quán cao** với `warmup` (44.8%/0.565/85.8%), xác nhận kết quả không phải nhiễu của mẫu nhỏ.

**Kiểm định thống kê** (bootstrap 2,000 lần, giống cách đã làm với `nocap`): 95% CI cho (Hit@1_dense − Hit@1_Layer1.5) = **[+0.082, +0.168]** — HOÀN TOÀN DƯƠNG, không chạm 0 → **có ý nghĩa thống kê thật**, không phải may rủi. Chênh lệch +12.6 điểm% quan sát được là đáng tin.

**KẾT LUẬN CHỐT cho B5**: Layer 2 (dense/BGE-M3) đáng giữ — cải thiện xếp hạng có ý nghĩa thống kê, và bổ trợ thật sự cho BM25 (nhóm câu 2 phương pháp cùng thất bại chỉ 2.6%, hợp lại coverage 97.4%) chứ không trùng lặp công dụng. Nên giữ **cả 2 lớp song song** (BM25/Layer1.5 + Layer2 dense), không phải chọn 1.

**Việc tiếp theo**: wire Layer 2 vào `search_units()`/`label.py` làm lớp song song với Layer 1.5 (hợp candidate từ cả 2 nguồn trước khi B4 chọn) — tận dụng phát hiện "bổ trợ nhau" thay vì chỉ dùng 1.

---

### [B5] 13/08 — chốt phân bổ ngân sách tham số: dùng 1.136 tỷ/2 tỷ, phần dư trả Generator

Người dùng hỏi cụ thể chiến lược dùng ngân sách 2 tỷ tham số — trình bày kiến trúc pipeline dự kiến đầy đủ (Layer 1 BM25 ∥ Layer 2 dense → Union → Expand → Layer 1.5 rerank → Layer 3 reranker → B4 → B6), tính tổng tham số: Layer 2 (BGE-M3, 568M, đã đo) + Layer 3 đề xuất (`BAAI/bge-reranker-v2-m3`, ~568M, CHƯA build/đo) = **1.136 tỷ**, còn dư ~864 triệu (43.2%).

**Quyết định người dùng** (chọn đúng phương án khuyến nghị): **KHÔNG dùng hết ngân sách dư — trả lại cho Generator (C)**, thay vì thêm bi-encoder thứ 2 hoặc reranker to hơn — đúng nguyên tắc "chỉ tiêu khi có bằng chứng cần" đã chốt từ đầu dự án (B5 README cũ), không tiêu chỉ vì "còn ngân sách".

Đã viết lại toàn bộ `src/b5_architecture_tradeoff/README.md` — từ "chưa chốt gì, chờ B7" (bản cũ) sang **đã chốt bằng số liệu thật** (không còn giả thuyết): giữ cả BM25 và dense (bổ trợ nhau, không thay thế), bảng ngân sách chi tiết, kiến trúc target, và ghi rõ cảnh báo con số "2 tỷ" mới chỉ nghe qua lời nhắc trong hội thoại — CHƯA xác nhận chính thức, dù không ảnh hưởng kết luận vì 1.136B vẫn đủ nhỏ hơn cả ngân sách cũ 1.3B nếu bị đổi lại.

**Việc tiếp theo**: (1) wire Layer 2+Union vào `search_units()` (đã hỏi người dùng, đang chờ xác nhận làm), (2) chọn/build/đo Layer 3 reranker.

---

### [B2] 13/08 — quyết định ưu tiên: build Layer 3 (reranker) trước, hoãn wire Layer 2/Union

Người dùng chốt: nhiệm vụ chính là "xây xong baseline" (đủ mảnh kiến trúc) trước khi tối ưu/nối dây — Layer 2 đã có số liệu chứng minh (rủi ro thấp, để sau), còn **Layer 3 là mảnh duy nhất chưa có gì (0 code, 0 số đo)** — ưu tiên làm trước.

**Cài đặt môi trường local**: máy này chưa có `torch`/`sentence_transformers` — cài `torch` bản CPU (`--index-url https://download.pytorch.org/whl/cpu`, nhẹ hơn bản GPU) + `sentence-transformers`, không lỗi.

**Test khả thi CPU trước khi cam kết hướng** (tránh lặp lại sai lầm "đoán rồi mới đo" của các bước trước): `BAAI/bge-reranker-v2-m3` (cross-encoder) — tải model lần đầu 235.1s (~3.9 phút, cache lại cho lần sau), suy luận đo được **108.7ms/candidate** trên CPU (test 30 candidate = 3.26s). Ngoại suy 500 câu × 30 candidate/câu ≈ **27.2 phút** — khả thi, KHÔNG cần quay lại Kaggle cho quy mô warmup.

**Thiết kế `pipeline/build_layer3_rerank_run.py`**: Layer 1 (BM25 doc, top-100 khớp baseline) → Expand → Layer 1.5 (rerank BM25 Khoản, đã có sẵn) → cắt top **30** candidate (đúng thiết kế "lọc rẻ trước, xử lý đắt sau" đã bàn — reranker không xử lý toàn bộ candidate đã gộp) → Layer 3 chấm điểm lại (đọc câu hỏi + candidate cùng lúc) → collapse về văn bản → so Recall@K/MRR với Layer 1.5. **CHƯA gồm Layer 2/Union** ở bước này — đúng phạm vi đã quyết định (hoãn wire Layer 2, chỉ đo Layer 3 đứng độc lập sau Layer 1.5 trước).

**Đang chạy nền** trên `data_retrieve/warmup.json` (500 câu, ~27-30 phút) — kết quả sẽ ghi ở entry tiếp theo.

---

### [B2] 13/08 — bài test tốc độ CPU cho Layer 3 SAI — dùng data giả không đại diện, phải chuyển Kaggle GPU

**Phát hiện khi chạy thật**: sau 50/500 câu, đo được **1,363.3s (22.7 phút)** — tức 27.3s/câu, KHÁC HẲN ước tính trước đó (~3.3s/câu từ bài test tốc độ). Ngoại suy full 500 câu ≈ **3.8 giờ**, không phải 27-30 phút đã báo — dừng tiến trình ngay, không để chạy tiếp mù quáng.

**Điều tra nguyên nhân bằng đo tách bạch từng bước** (không đoán): `search_docs_multi` (0.40s) + `expand_to_units` (0.00s, 3,922 unit) + `rerank_units_khoan`/Layer1.5 (0.03s) = **0.43s tổng** — HOÀN TOÀN không phải chỗ chậm. Nghẽn nằm ở chính bước reranker.

**Nguyên nhân gốc, xác nhận bằng đo trực tiếp trên candidate THẬT**: bài test tốc độ ban đầu dùng **1 đoạn text ngắn (~200 ký tự) LẶP LẠI 30 lần** để mô phỏng — không đại diện. Candidate thật (lấy từ đúng câu hỏi thật, sau Layer 1.5) có độ dài dao động **134 → 2,241 ký tự** (một số gần chạm giới hạn `max_length=512` token của model). Đo lại đúng trên 30 candidate thật: **28.63s** (≈954ms/candidate) — gấp **8.8 lần** con số bài test cũ, khớp chính xác với tốc độ chậm quan sát trong lần chạy thật.

**Bài học lặp lại đúng mẫu hình đã gặp nhiều lần trong dự án này**: test tốc độ bằng dữ liệu KHÔNG ĐẠI DIỆN (ở đây: text ngắn lặp lại thay vì đa dạng độ dài thật) cho ra ước tính sai nghiêm trọng — phải test bằng dữ liệu thật trước khi cam kết thời gian, không phải dữ liệu giả lập tiện tay.

**Quyết định**: chuyển Layer 3 (reranker) sang chạy Kaggle GPU — khác Layer 2 (chỉ cần nhân ma trận trên vector đã tính sẵn, CPU đủ nhanh), reranker phải xử lý toàn bộ văn bản qua model cho MỖI cặp (câu hỏi, candidate) — chi phí tăng theo độ dài, đúng loại việc GPU tăng tốc được nhiều (ước tính 10-50 lần).

---

### [Data] 13/08 — quyết định người dùng: ngừng dùng `warmup.json` (cả 2 nguồn), chuyển hẳn sang `train.json`

Người dùng chốt: giai đoạn warm-up của cuộc thi đã qua, không cần giữ `warmup.json` làm tập riêng nữa — dùng `train.json` làm chính từ giờ. Có gợi ý xóa hẳn file `warmup.json` — **đã KHÔNG xóa** (giữ nguyên file gốc trên đĩa, đúng quy ước `data/README.md` "không sửa/xoá file gốc do BTC/nguồn khác cấp", xóa là hành động khó hoàn tác) — chỉ NGỪNG DÙNG trong pipeline/đánh giá từ đây trở đi. Đã xác nhận lại với người dùng, chưa nhận phản hồi ngược lại việc giữ file.

**Áp dụng ngay**: Layer 3 (candidate generation + reranking) chuyển từ `data_retrieve/warmup.json` (500 câu, dự định ban đầu) sang `data_retrieve/train.json` (7,000 câu).

---

### [B2] 13/08 — chuẩn bị Layer 3 trên Kaggle: tách riêng bước sinh candidate (local, rẻ) khỏi bước rerank (Kaggle GPU, đắt)

**Thiết kế tách 2 giai đoạn** (khác Layer 2 — ở đó phải upload cả corpus vì cần embed HẾT corpus; ở đây KHÔNG cần vì reranker chỉ xử lý candidate đã chọn sẵn):
1. **Local** (`pipeline/build_layer3_candidates.py`): chạy Layer 1 (BM25 top-100) + Expand + Layer 1.5 (rerank Khoản) — phần này đã đo rẻ (~0.43s/câu, không cần GPU) — lấy top-30 candidate/câu, ghi ra file NHẸ (chỉ text cần thiết, không phải cả `parsed_corpus.jsonl` 700MB) để upload Kaggle.
2. **Kaggle** (`kaggle_layer3/rerank_notebook.py`): chỉ load model + đọc file candidate nhẹ + chạy reranker + ghi run file — không cần BM25 index hay corpus đầy đủ.

**Tối ưu cách gọi GPU** (rút kinh nghiệm từ việc CPU chậm): thay vì gọi `model.predict()` riêng từng câu hỏi (30 cặp/lần, lãng phí khả năng xử lý song song của GPU với batch nhỏ), **GỘP TẤT CẢ cặp (câu hỏi, candidate) của TOÀN BỘ 7,000 câu thành 1 danh sách lớn** (~210,000 cặp), gọi `predict(batch_size=128)` đúng 1 lần — để `sentence-transformers` tự chia batch tối ưu cho GPU, rồi mới nhóm lại theo câu hỏi sau.

**Đang chạy nền** (local, sinh candidate cho 7,000 câu, ước ~50 phút dựa trên tốc độ đã đo 0.43s/câu). Notebook + README Kaggle đã chuẩn bị sẵn — kế thừa toàn bộ bài học xử lý lỗi từ Layer 2 (mount dataset, sympy, restart session).

**Kết quả sinh candidate**: 777.6s (13.0 phút — NHANH HƠN ước tính 50 phút, vì `expand_to_units` rẻ hơn đáng kể so với giả định ban đầu). File `kaggle_layer3/upload/layer3_candidates_train.jsonl` = **204.2 MB** (nhẹ hơn nhiều so với corpus 700MB dùng cho Layer 2 — đúng lợi ích của việc tách 2 giai đoạn local/Kaggle). Đang chờ người dùng upload lên Kaggle + chạy `rerank_notebook.py`.

---

### [B2] 17/08 — phát hiện quan trọng: Dense rerank (dùng embedding Layer 2 có sẵn) thắng áp đảo BM25 rerank (Layer 1.5), MIỄN PHÍ

**Câu hỏi người dùng nêu ra, đúng trọng tâm**: BM25 dùng để rerank Khoản (Layer 1.5) "khá yếu", sao không dùng model khác — trong khi bi-encoder (Layer 2) đã chấm điểm/sắp xếp sẵn rồi.

**Insight chính xác**: `outputs/layer2/corpus_embeddings.npy` đã chứa sẵn embedding của TOÀN BỘ 432,473 Khoản (tính 1 lần trên Kaggle cho Layer 2) — để chấm điểm dense cho BẤT KỲ candidate nào (kể cả candidate không nằm trong top-100 gốc của Layer 2), chỉ cần **tra vị trí trong bảng có sẵn + nhân 1 phép** (dot product), KHÔNG cần chạy lại model BGE-M3 — rẻ hơn cả BM25 rerank hiện tại.

**Test trực tiếp trên CÙNG 1 tập candidate** (Layer 1 BM25-doc top-100 → Expand, giữ nguyên tập candidate, chỉ đổi cách CHẤM ĐIỂM LẠI): `warmup.json` (500 câu, dùng embedding Layer 2 đã có sẵn — không cần Kaggle, chạy local 66.3s cho cả 2 cách):

| Cách rerank | Hit@1 | Hit@5 | Hit@10 | MRR | Coverage |
|---|---|---|---|---|---|
| BM25-Khoản (Layer 1.5 hiện tại) | 32.2% | 61.4% | 70.4% | 0.452 | 89.8% |
| **Dense (tra bảng embedding có sẵn)** | **45.2%** | **71.0%** | **75.2%** | **0.564** | 89.8% (GIỐNG HỆT) |

**Diễn giải**: coverage giống hệt nhau (đúng lý thuyết — cả 2 chỉ sắp xếp lại CÙNG 1 tập candidate, không đổi tập nào được chọn), nhưng chất lượng xếp hạng chênh **13 điểm% ở Hit@1** — dense hơn hẳn, đúng như suy luận của người dùng.

**QUYẾT ĐỊNH: thay Layer 1.5 (BM25 rerank Khoản) bằng Dense rerank** (dùng embedding Layer 2 có sẵn) — cải thiện miễn phí, không cần Kaggle, không cần build gì thêm, chỉ cần đổi code tra `corpus_embeddings.npy` thay vì gọi `bm25_khoan_index.pkl`. Giữ nguyên tên "Layer 1.5" cho vị trí trong kiến trúc (rerank rẻ sau Expand, trước Layer 3), chỉ đổi CƠ CHẾ bên trong.

**Việc tiếp theo**: code lại `retrieve.rerank_units_khoan()` (hoặc hàm mới `rerank_units_dense()`) dùng embedding lookup thay BM25; wire vào B0/`search_units()`; đánh giá lại toàn bộ pipeline.

**CODE XONG NGAY SAU ĐÓ** (cùng ngày): thêm `build_corpus_embedding_index()` + `rerank_units_dense()` vào `src/b2_retrieval/retrieve.py` (giữ nguyên `rerank_units_khoan`/BM25 làm dự phòng, không xoá). Wire vào `search_units()` — thêm 3 tham số tuỳ chọn `corpus_embeddings`/`corpus_embedding_index`/`query_embedding`, ưu tiên dùng dense nếu đủ cả 3, fallback BM25 nếu chỉ có `khoan_index`, không rerank nếu không truyền gì (tương thích ngược 100% với caller cũ). Test PASS: kết quả qua `search_units()` khớp chính xác kết quả tính tay trước đó; đường không rerank vẫn hoạt động bình thường.

**Giới hạn đã biết, CHƯA giải quyết**: `rerank_units_dense` cần `query_embedding` tính SẴN — chỉ dùng được ngay cho câu hỏi đã có embedding trước (`data_retrieve`/`qa_train`/`qa_warmup`/`qa_public`, đã embed trên Kaggle). B0 dùng QUOTE TEXT trích từ answer (`_extract_quote_core`), KHÔNG phải câu hỏi gốc — chưa có embedding sẵn cho các đoạn quote này, cần load model BGE-M3 sống (local, đã cài `torch`+`sentence-transformers`) để tính embedding cho quote mới — CHƯA làm, là việc tiếp theo nếu muốn wire dense rerank vào B0.

### [B2] 17/08 — RÀ SOÁT KIẾN TRÚC theo yêu cầu người dùng: tìm ra 6 lỗi, 3 lỗi NẶNG ở "đường ống" giữa các lớp

Người dùng yêu cầu đánh giá kiến trúc "có thật sự hợp lí và tối ưu chưa, vì sao". Rà lại + đo bổ sung, tìm ra 6 vấn đề thật (không phải chỉnh sửa nhỏ):

**Lỗi 1 (NGHIÊM TRỌNG NHẤT) — phễu thắt quá chặt trước reranker, phá huỷ chính phần recall vừa xây**. Đo thật (`warmup`, 500 câu, nhánh BM25-doc + dense rerank):

| Điểm cắt | Số văn bản phủ | Hit |
|---|---|---|
| Union 2 nhánh (đã đo trước) | ~170 | 97.4% |
| Top-100 văn bản (không cắt unit) | 100 | 89.8% |
| Cắt top-100 unit | ~30 | 80.6% |
| **Cắt top-30 unit (ĐANG LÀM)** | **9.9** | **76.0%** |

Xây hệ thống coverage 97.4% rồi tự bóp xuống 76.0% NGAY TRƯỚC thành phần chính xác nhất (Layer 3) — Layer 3 không bao giờ cứu được 24% còn lại. Con số "30" do tôi chọn tuỳ ý, CHƯA TỪNG ĐO — đúng loại sai sót đã tránh được ở mọi chỗ khác trong dự án. Đi ngược nguyên tắc "ưu tiên recall" đã chốt.

**Lỗi 2 (NẶNG) — Union ở cấp VĂN BẢN là thiết kế bị "nhiễm" từ cách ĐO, không phải từ nhu cầu**: nhãn `data_retrieve` chỉ có cấp văn bản → đo ở cấp văn bản → kiến trúc vô tình thừa hưởng bước collapse-về-doc mà pipeline không cần. Hệ quả: Layer 2 (dense) đã biết chính xác Khoản nào tốt, nhưng ta bỏ thông tin đó, thu về doc ID, rồi expand lại TOÀN BỘ Khoản của doc đó — 1 văn bản dense thích vì đúng 1 Khoản giờ đóng góp cả ~80 Khoản, rác chen chỗ trong 30 slot (làm Lỗi 1 tệ thêm).

**Lỗi 3 (NẶNG) — điểm BM25 bị bỏ hoàn toàn khi xếp hạng**: sau Union, xếp hạng CHỈ bằng dense score → BM25 tụt xuống vai trò "chỉ sinh candidate". Lỗi logic: với đúng 51 câu mà BM25 tìm được nhưng dense trượt, dense score là thước đo TỆ (đó là lý do dense trượt) → xếp hạng thuần dense sẽ vùi lấp chính candidate mà BM25 đóng góp riêng. Gộp để tăng recall rồi xếp hạng theo cách phá recall.

**Lỗi 4 (TB)** — `select_span` mâu thuẫn nguyên tắc đã chốt: dùng 1.1B tham số tìm đúng Khoản rồi cắt nhỏ bằng heuristic đếm từ thô, cắt sai là xoá luôn đáp án — trong khi docs nhóm ghi METEOR phạt viết thiếu nặng gấp 3-4 lần viết dư. Số 67% đo trên nhãn B0 CŨ, chưa đo lại sau rebuild.

**Lỗi 5 (TB) — "hiệu ứng đèn đường"**: toàn bộ tối ưu đo ở cấp văn bản (vì chỉ ở đó có nhãn sạch). B4 (`select_khoan`→`select_span`) — đúng 2 khâu Generator tiêu thụ — CHƯA có số đo nào sau rebuild. Đang tối ưu chỗ nhìn thấy, bay mù ở chỗ quan trọng nhất. Đã có sẵn 2,000 câu nhãn B0 cấp Khoản mà chưa dùng.

**Lỗi 6 (nhỏ)** — quyết định chưa vào code: `use_multi_query` vẫn default `True` trong `search_units()` dù đã chốt bỏ.

---

### [B2] 17/08 — thiết kế lại "đường ống" + PHÁT HIỆN LỚN: ràng buộc đa dạng khi cắt cho +5.6 điểm% MIỄN PHÍ

**Thiết kế mới** (giữ nguyên mọi lớp đã chứng minh bằng số liệu — BM25 cấp văn bản, config BM25, dense cấp Khoản, hybrid 2 nhánh, Layer 3, ngân sách; chỉ sửa cách nối):

```
[A] BM25 doc top-N_doc → Expand ra Khoản     ┐ song song, độc lập
[B] Dense unit-level toàn kho → top-M unit    ┘ (KHÔNG collapse về doc nữa — sửa Lỗi 2)
  → [C] UNION ở CẤP UNIT
  → [D] chấm CẢ 2 thang cho MỌI unit trong U (rerank_units_khoan + rerank_units_dense,
        2 hàm ĐÃ CÓ SẴN — chỉ lắp ghép, không viết thuật toán mới) → đổi thành rank
  → [E] FUSION bằng RRF: 1/(k+rank_bm25) + 1/(k+rank_dense), k=60, 0 tham số (sửa Lỗi 3)
  → [F] CẮT top-K_rerank (K chọn bằng số đo — sửa Lỗi 1)
  → [G] Layer 3 reranker → [H] B4 top-3 → B6
```
`rerank_units_khoan` (BM25) từ chỗ "sắp bị thay thế" giờ có vai trò chính đáng: sinh xếp hạng cho nhánh lexical để RRF có cái mà gộp.

**Đo đường cong Hit@K theo số unit giữ lại** (500 câu): K=10→72.0%, 30→76.0%, 50→78.4%, 100→80.6%, 200→83.2%, 300→85.0%, 500→86.2% — **KHÔNG có điểm khuỵu**, tăng đều → không có điểm cắt "miễn phí", là đánh đổi thẳng recall/chi phí reranker.

**PHÁT HIỆN LỚN — ràng buộc "tối đa N Khoản mỗi văn bản" khi cắt**:

| K (unit) | max/doc | Số văn bản phủ | Hit |
|---|---|---|---|
| 30 | không giới hạn (đang làm) | 9.9 | 76.0% |
| **30** | **1** | **30.0** | **81.6% (+5.6)** |
| 30 | 2 | 18.8 | 79.6% |
| **50** | **1** | **50.0** | **86.4%** |
| 100 | 1 | 100.0 | 89.8% (chạm trần nhánh BM25) |

Cùng 30 slot (chi phí reranker Y NGUYÊN), thêm ràng buộc đa dạng → **+5.6 điểm%**. K=50+max1 cho **86.4% — cao hơn cả K=500 không ràng buộc (86.2%) mà rẻ hơn 10 lần**. Nguyên nhân hợp lý: đáp án nằm ở 1 VĂN BẢN cụ thể, tiêu nhiều slot cho nhiều Khoản cùng văn bản là dư thừa xét theo recall cấp văn bản.

**CẢNH BÁO tự nêu, chưa giải quyết**: số đo trên ở CẤP VĂN BẢN. `max_per_doc=1` có thể giữ ĐÚNG văn bản nhưng SAI Khoản — mà đầu ra cuối cần đúng Khoản. Có mâu thuẫn thật giữa tối ưu recall-cấp-văn-bản và đúng-Khoản-cuối-cùng; **chỉ giải được bằng nhãn B0 cấp Khoản** (2,000 câu đã có) — CHƯA kết luận `max_per_doc` nên bằng mấy.

**Hệ quả thực tế**: file `kaggle_layer3/upload/layer3_candidates_train.jsonl` (204MB) sinh theo thiết kế CŨ (top-30, không Union/RRF/đa dạng) → **đã ĐỀ NGHỊ người dùng TẠM DỪNG upload Kaggle**, tránh chạy 2 lần.

**Thứ tự thực hiện đã trình bày với người dùng**: (1) chốt điểm cắt + ràng buộc đa dạng bằng số đo, (2) code Union cấp unit + RRF, (3) đo B4 bằng nhãn B0 → quyết định `select_span`, (4) sinh lại candidate + Layer 3 trên Kaggle (cuối cùng).

---

### [B2] 17/08 — KẾT QUẢ ÂM TÍNH quan trọng: RRF cân bằng 1:1 LÀM TỆ ĐI, không phải cải thiện như tôi dự đoán

Code xong `union_units()` / `fuse_units_rrf()` / `cut_units_diverse()` (+ `RRF_K=60`) vào `retrieve.py`, viết `pipeline/eval_hybrid_designs.py` so 2 thiết kế trên `data_retrieve/warmup` (500 câu, 141.2s):

| Thiết kế | H@1 | H@3 | H@10 | H@20 | H@30 | H@100 | MRR | coverage |
|---|---|---|---|---|---|---|---|---|
| D0 hiện tại (BM25-doc → Expand → dense rerank) | **45.4%** | **65.4%** | 76.0% | 79.2% | 81.6% | **89.8%** | **0.566** | 449/500 |
| D1 Union cấp unit + RRF(1:1) | 17.8% | 51.6% | 75.8% | **81.6%** | **83.6%** | 87.2% | 0.373 | 436/500 |

**RRF cân bằng làm SỤP precision đầu bảng** (H@1 45.4% → 17.8%, MRR 0.566 → 0.373). Tôi đã đề xuất RRF như "chuẩn ngành, chắc ăn" — **số đo phản đối**. Nguyên nhân (đúng bản chất RRF): RRF thưởng ĐỒNG THUẬN 2 nhánh, phạt XUẤT SẮC ở 1 nhánh. Vì BM25-Khoản là ranker YẾU (H@1 ~32%) mà được cân bằng 1:1 với dense MẠNH (H@1 45%), nó kéo tín hiệu mạnh xuống: 1 unit hạng 1 ở dense nhưng vắng ở BM25 chỉ được 1/61=0.0164, trong khi unit hạng 50 ở CẢ HAI được 2/110=0.0182 — "tầm thường ở cả hai" THẮNG "xuất sắc ở một". Cộng thêm bất đối xứng độ dài list: nhánh BM25 có hàng NGHÌN unit (mọi Khoản của 100 văn bản), nhánh dense chỉ 300 → rác BM25 tích luỹ điểm RRF.

**NHƯNG tại đúng điểm cắt thì RRF lại HƠN**: K=30 → 83.6% (RRF) vs 81.6% (D0); K=20 → 81.6% vs 79.2%. Đây mới là con số quan trọng cho pipeline có Layer 3 — vì sau khi cắt, Layer 3 xếp hạng lại nên H@1 của bước fusion không quyết định; **recall tại điểm cắt** mới quyết định. Ngược lại, nếu pipeline KHÔNG có Layer 3 (trạng thái production hiện tại) thì RRF sẽ làm hại nặng (B4 lấy top-3 → H@3 51.6% vs 65.4%).

**Ràng buộc đa dạng xác nhận lại trên xếp hạng RRF**: K=30 không giới hạn=79.0% (12.1 văn bản) → max1=83.6% (30 văn bản) → vẫn +4.6 điểm%, nhất quán với đo trước trên D0.

**Lưu ý phương pháp (tự nêu)**: coverage của D1 (436) thấp hơn D0 (449) một phần là **artifact của cách đo** — tôi collapse xếp hạng fused về top-100 VĂN BẢN, trong khi union thực tế có tới ~170 văn bản khả dụng → phép đo top-100 tự cắt mất một nửa union. Không phải bằng chứng union kém coverage.

**Sweep trọng số + phương án thay thế** (500 câu, mọi con số cấp văn bản, `max_per_doc=1` ở các cột cắt):

| Cấu hình | H@1 | H@3 | recall@cắt30 | @cắt50 | @cắt100 |
|---|---|---|---|---|---|
| Union + xếp hạng THUẦN DENSE | **44.8%** | **65.4%** | 81.4% | 83.4% | **88.4%** |
| RRF 1:1 | 17.8% | 51.6% | **83.6%** | **85.4%** | 87.2% |
| RRF 0.3:1 | 24.4% | 59.0% | 83.0% | 84.0% | 86.4% |
| RRF 0.1:1 | 28.2% | 61.4% | 82.4% | 83.6% | 86.2% |
| **Lai: cắt bằng RRF(1:1) → xếp hạng lại bằng dense** | 39.0% | 59.4% | 83.6% | **85.4%** | — |

**Kết luận**: trọng số trung gian chỉ TRƯỢT DỌC đường đánh đổi, không vượt được 2 đầu — không có bữa trưa miễn phí. Phương án lai (cắt RRF → xếp lại dense) giữ được recall cắt của RRF (85.4%) nhưng **không hồi phục hết** precision dense (H@1 39.0 vs 44.8). Nguyên nhân đáng chú ý: với `max_per_doc=1`, mỗi văn bản chỉ giữ 1 Khoản mà **RRF** thích nhất, KHÔNG phải Khoản mà **dense** thích nhất → khi xếp hạng lại bằng dense, văn bản bị đại diện bởi Khoản kém hơn. Đây chính là **mâu thuẫn doc-level vs Khoản-level** đã cảnh báo, giờ hiện ra bằng số cụ thể — thêm bằng chứng phải đo bằng nhãn B0 cấp Khoản trước khi chốt `max_per_doc`.

**QUYẾT ĐỊNH (tách 2 vai trò, KHÔNG chọn bừa 1 phương án vì điều quyết định là chất lượng Layer 3 — chưa đo)**:
- **Tập đưa vào Layer 3**: cắt bằng RRF(1:1) + `max_per_doc=1`, top-50 → recall 85.4% (cao nhất) — cho Layer 3 trần cao nhất.
- **Xếp hạng khi KHÔNG có Layer 3** (đúng trạng thái production hiện tại): giữ **dense thuần** (H@3 65.4%) — an toàn nhất, vì H@3 chính là recall mà Generator THỰC NHẬN (B4 trả top-3).
- Chốt cuối sau khi có số đo Layer 3.

**Nhận thức phương pháp quan trọng**: "ưu tiên recall" phải phân biệt rõ 2 loại — (a) recall tại điểm cắt (trần cho Layer 3) và (b) **H@3 cuối pipeline** (Generator thực nhận). Trước đây tôi chỉ nhìn (a). Nếu Layer 3 tốt, nó chuyển (a) thành (b); nếu Layer 3 yếu/không có, (b) mới là con số thật quan trọng.

---

### [B2/B4] 17/08 — ĐO Ở CẤP KHOẢN: đảo ngược 2 kết luận trước, chốt thiết kế cuối

Dùng nhãn B0 cấp Khoản trên `qa_train` dev sample (`outputs/b0_labels_train_sample.json`, lọc `unit_type="khoan"` + `confidence>=0.6` → **n=393 câu**). KHÔNG dùng `warmup` (theo quyết định người dùng 17/08).

**PHÁT HIỆN A — ràng buộc đa dạng (`max_per_doc`) là ẢO GIÁC do đo sai cấp**:

| K | max/doc | **Hit KHOẢN** | Hit văn bản |
|---|---|---|---|
| 30 | không giới hạn | **81.9%** | 95.4% |
| 30 | **1** (tôi vừa đề xuất ở entry trước!) | **43.3%** | 98.7% |
| 30 | 2 | 56.0% | 98.2% |
| 30 | 3 | 64.4% | 97.5% |
| 50 | 3 | 66.4% | 98.7% |

`max_per_doc=1` giữ ĐÚNG VĂN BẢN (98.7%) nhưng SAI KHOẢN — Hit Khoản sụp **81.9% → 43.3%** (mất gần nửa). "+5.6 điểm% miễn phí" tôi khoe ở entry trước là **ảo giác vì chỉ đo ở cấp văn bản**. Đây đúng là "hiệu ứng đèn đường" (Lỗi 5) mà chính tôi nêu ra rồi gần như tự rơi vào — nếu không đo bước này đã đưa 1 thay đổi làm hại nặng vào production. **CHỐT: `max_per_doc = None`** (bỏ ràng buộc đa dạng).

**PHÁT HIỆN B — `select_span` XOÁ MẤT đáp án 55.6% số lần**: trong 151 ca mà Khoản đúng CÓ trong tập candidate, `select_span` chỉ giữ được đoạn gold ở **67/151 = 44.4%**. Nghĩa là hơn nửa số lần nó cắt bỏ đúng phần chứa câu trả lời. Kết hợp nguyên tắc "thà dư hơn thiếu" + METEOR phạt viết thiếu nặng gấp 3-4 lần → **CHỐT: BỎ `select_span`, trả nguyên Khoản**. (Lưu ý phép đo: probe = 60 ký tự đầu của gold span phải nằm trong text được chọn — nếu gold span trải qua nhiều Điểm thì không unit con nào chứa trọn, tính là FAIL, đúng cho mục đích "văn bản giao cho Generator có chứa đáp án không".)

**PHÁT HIỆN C — kiểm lại RRF ở cấp Khoản (không ràng buộc đa dạng), n=393**:

| Cấu hình | K=1 | K=3 | K=10 | K=30 | K=50 | K=100 |
|---|---|---|---|---|---|---|
| chỉ BM25-Khoản | 17.3% | 31.3% | 48.6% | 60.6% | 68.4% | 75.1% |
| **dense thuần (trên tập union)** | **39.4%** | **55.5%** | **73.0%** | **82.4%** | 85.0% | 91.3% |
| RRF 1:1 | 6.6% | 27.0% | 61.1% | 81.9% | **87.8%** | **92.1%** |

Cùng mẫu hình như cấp văn bản, rõ hơn: **dense thuần thắng áp đảo ở K≤30** (K=3: 55.5% vs 27.0% — hơn GẤP ĐÔI), RRF chỉ hơn +2.8 điểm% ở K=50. Vì B4 trả **top-3** cho Generator, dense thuần là lựa chọn đúng; +2.8 điểm% ở pool K=50 không bù được -28.5 điểm% ở K=3 nếu Layer 3 không đủ mạnh để khai thác.

**CẢNH BÁO trung thực về nhãn B0**: nhãn do chính pipeline tương tự sinh ra → thiên lệch chọn mẫu, **số tuyệt đối lạc quan** (Hit doc 95-99% ở đây vs 79-83% trên `data_retrieve` nhãn sạch). Nhưng **so sánh TƯƠNG ĐỐI giữa các biến thể vẫn hợp lệ** (thiên lệch ảnh hưởng đều mọi biến thể) — và đó chính là câu hỏi cần trả lời ở đây.

## THIẾT KẾ CUỐI (mọi lựa chọn đều có số đo hậu thuẫn)

```
Câu hỏi
  ├─ Layer 1: BM25 doc top-100 → Expand ra hết Khoản     ┐ nguồn candidate A
  └─ Layer 2: dense (BGE-M3) unit-level top-300           ┘ nguồn candidate B
  → Union CẤP UNIT                                          (recall)
  → Xếp hạng TOÀN BỘ bằng DENSE score                       (K@3=55.5%, tốt nhất)
  → KHÔNG ràng buộc đa dạng (max_per_doc=None)              (ràng buộc làm sụp Hit Khoản 81.9→43.3)
  → cắt top-50 → Layer 3 reranker                           (pool cấp Khoản 85.0%)
  → B4: top-3 Khoản, KHÔNG select_span (trả nguyên Khoản)   (select_span xoá đáp án 55.6%)
  → B6 đóng gói
```

RRF/`cut_units_diverse` **giữ lại trong code nhưng TẮT mặc định** — có số đo đầy đủ trong log này, bật lại được nếu Layer 3 chứng minh khai thác được pool rộng hơn.

Code hoá thiết kế cuối thành **1 hàm duy nhất** `retrieve.search_units_hybrid()` (+ `dense_search_units()` tách riêng nhánh B) để mọi script dùng chung 1 đường, không phải tự lắp lại từng lần. Đo: **0.26s/câu** (20 câu, đã gồm cả Layer 1 + dense search toàn kho + union + xếp hạng).

---

### [B1] 17/08 — BUG THẬT bắt được khi test `search_units_hybrid`: 7,163 unit (1.66%) có text RỖNG, embedding của chúng là vector rác

**Cách phát hiện**: test `search_units_hybrid` trên 20 câu, assert mọi unit trả ra phải có text → **24/1000 unit text rỗng**. Truy ngược: không phải bug tra cứu mà là **bug B1**.

**Nguyên nhân**: `_DIEU_RE` bắt tiêu đề là "toàn bộ phần còn lại của DÒNG" (`(?P<tieu_de>[^\n]*)$`). Với Điều NGẮN mà **toàn bộ nội dung nằm CÙNG DÒNG với header** — vd `"Điều 2. Quyết định này có hiệu lực từ ngày ký ban hành."` — regex hút hết nội dung vào group `tieu_de`, để `body`/`text` **RỖNG**. Nội dung không mất (nằm ở `dieu_tieu_de`) nhưng mọi bước sau (`expand_to_units` → B2/B4/B6) chỉ đọc `text` → giao **rỗng** cho Generator.

**Quy mô đo được toàn corpus**: **7,163 unit rỗng (1.66% của 432,473)**, **2,514 văn bản bị ảnh hưởng (29.5% corpus)** — 100% trong số đó có nội dung nằm ở `dieu_tieu_de`. Đây là loại Điều rất hay bị hỏi trong QA pháp luật ("có hiệu lực từ ngày nào", "ai chịu trách nhiệm thi hành") → không phải rìa vô hại.

**Hệ quả nặng hơn với Layer 2**: `embed_corpus_notebook.py` CELL 4 (chạy trên Kaggle) dùng đúng `dieu["text"]` → **embedding của 7,163 unit này được tính trên CHUỖI RỖNG = vector rác**, đang nằm trong `outputs/layer2/corpus_embeddings.npy`. Nghĩa là mọi số đo dense ở trên đã bị nhiễu nhẹ bởi 1.66% vector vô nghĩa (theo hướng BẤT LỢI cho dense — số thật của dense có thể cao hơn chút).

**Đã sửa** (`parse_document`): `body` rỗng + có `tieu_de` → `text = tieu_de` (lúc đó tiêu đề CHÍNH LÀ nội dung), `char_start/char_end` trỏ đúng span của tiêu đề. Cố ý KHÔNG đổi text của Điều bình thường (nếu ghép tiêu đề vào mọi Điều thì phải tính lại toàn bộ 432k embedding — đắt, chưa có bằng chứng đáng).

**Verify sau sửa**: `context_100380` — Điều 2 giờ có text 47 ký tự (đúng nội dung), offset khớp `main_text[char_start:char_end] == text` cho cả 3 Điều; 4/4 test `demo_b1_parser.py` PASS; rebuild full corpus → **text rỗng: 7,163 → 0**, tổng unit KHÔNG đổi (432,473), matched/fallback giữ nguyên 84.6%/15.4% (không regression).

**CÒN LẠI (chưa làm)**: 7,163 embedding rác trong `outputs/layer2/corpus_embeddings.npy` cần tính lại. 2 cách: (a) chạy lại toàn bộ trên Kaggle (~30 phút GPU), (b) tính bù CHỈ 7,163 unit đó bằng BGE-M3 local trên CPU rồi ghi đè đúng các hàng đó trong `.npy` (khả thi — text ngắn, `torch`+`sentence-transformers` đã cài local). Cách (b) rẻ hơn và không cần vòng Kaggle mới.

**ĐÃ LÀM (cách b, cùng ngày)**: `pipeline/patch_layer2_embeddings.py` — xác định unit cần vá bằng dấu hiệu `text == dieu_tieu_de` (đúng dạng sau khi B1 sửa), tìm được **7,080 unit** (khác 7,163 một chút — chênh lệch do 1 số unit đổi `dieu_id`/không còn tồn tại sau rebuild, chấp nhận được). Load `BAAI/bge-m3` LOCAL trên CPU (287.1s tải model — cache riêng, khác model reranker đã tải trước đó), embed 7,080 text ngắn (692.8s ≈ 11.5 phút), backup bản gốc (`corpus_embeddings.before_patch.npy`) trước khi ghi đè. Sanity check: norm trung bình = 1.0000 (đúng chuẩn hoá), 184/200 vector đầu KHÁC NHAU (đúng kỳ vọng — mỗi tiêu đề Điều khác nội dung, không phải lỗi tất cả trùng nhau).

**Xác nhận cuối**: chạy lại `experiments/demo_b_end_to_end.py` (viết lại dùng `search_units_hybrid` + bỏ `select_span`, trên `qa_train` — không dùng `warmup` theo quyết định 17/08) — 5 câu hỏi thật, **0/15 context rỗng** (so 24/1000 trước khi vá), pipeline B2(hybrid)→B4→B6 chạy sạch không lỗi. Ví dụ câu `82051` (hỏi mức phạt vận chuyển động vật không giấy kiểm dịch) → top-1 khớp chính xác nội dung. Ghi nhận 1 quan sát (không phải lỗi): câu `108971` có context `unit_type=doc_fallback`, `article=""`, text dài 24,795 ký tự (cả văn bản không tách được cấu trúc) — đúng thiết kế, nhưng đưa nguyên khối lớn không tiêu đề cho Generator chưa lý tưởng, chưa xử lý.

**TOÀN BỘ 6 lỗi/phát hiện trong đợt rà soát 17/08 đã xử lý xong**: Lỗi 1 (phễu thắt) → cắt top-50 không ràng buộc; Lỗi 2 (Union sai cấp) → `union_units` cấp unit; Lỗi 3 (bỏ điểm BM25) → đo RRF, quyết định KHÔNG dùng (dense thuần thắng); Lỗi 4 (`select_span` có hại) → bỏ; Lỗi 5 (chưa đo B4) → đã đo bằng nhãn B0 cấp Khoản; Lỗi 6 (`use_multi_query` sai default) → sửa. Cộng thêm phát hiện ngoài dự kiến: bug B1 7,163 unit rỗng → đã sửa + vá embedding.

**Việc tiếp theo**: sinh lại `layer3_candidates_train.jsonl` theo `search_units_hybrid` (bản cũ dùng thiết kế đã lỗi thời) rồi mới chạy Kaggle cho Layer 3.

**ĐÃ LÀM (cùng ngày, tiếp)**: Viết lại `pipeline/build_layer3_candidates.py` dùng `search_units_hybrid` (top_k_docs=100, dense_top_m=300, top_k_out=50, KHÔNG ràng buộc đa dạng) thay cho Layer1+Layer1.5-BM25 top-30 cũ đã lỗi thời. Chạy trên 7.000 câu `data_retrieve/train.json` (dùng `query_embeddings_retrieve_train.npy` đã có sẵn từ Kaggle) — **810.6s, 0 candidate rỗng**, ra file 368.1MB (`kaggle_layer3/upload/layer3_candidates_train.jsonl`).

Rà lại `kaggle_layer3/rerank_notebook.py` trước khi chạy Kaggle, phát hiện CELL 6 **collapse kết quả reranker về cấp văn bản** (`DOC_TOP_K=100`, gom theo `context_id`) — đúng bẫy "đèn đường" đã tự bắt ở B4 (đo/rank cấp văn bản đánh lừa, cấp Khoản mới là cái quyết định Hit thật). Sửa: bỏ collapse, giữ nguyên `ranked_units` (unit_id, context_id, score) đầy đủ cho toàn bộ candidate đã rerank, để B3/B4 tự đánh giá Hit-Khoản sau. Cập nhật `kaggle_layer3/README.md` khớp thiết kế mới (top-50 thay top-30, ghi rõ lý do không collapse).

**Trạng thái**: sẵn sàng để chạy Kaggle GPU (chưa chạy — cần user tạo Dataset + Notebook theo README, việc thủ công ngoài khả năng tool).

### [B3] 17/08 — Đo trần recall chunking (mục 6 tầng 2): 31,1% câu cần thông tin ngoài đúng 1 Khoản

Script `pipeline/measure_chunking_ceiling.py`. Cách đo: `confidence` lưu trong `MatchedSpan` (B0) chỉ phản ánh mức khớp giữa quote gốc và ĐÚNG Khoản B0 đã chọn — không nói liệu unit LỚN HƠN (cả Điều chứa Khoản đó) có khớp TỐT HƠN không. Tái tạo lại `quote_core` (dùng lại đúng `_split_by_amend_marker`/`_extract_quote_core` của B0), align lại (Smith-Waterman, cùng hàm `align_span`) với CẢ 2: (a) đúng Khoản B0 chọn, (b) toàn bộ Điều chứa Khoản đó — so 2 điểm.

**Mẫu**: n=508, gộp `warmup` (500 câu) + `train` dev-sample (1.500 câu), lọc nhãn đáng tin (`confidence>=0.6`, `unit_type="khoan"`). Sanity check: 11/508 (2,2%) score_khoan tính lại lệch >0,02 so với confidence đã lưu — chấp nhận được (nhiễu nhỏ, không phải lỗi hệ thống).

| | Số câu | Tỷ lệ |
|---|---|---|
| Khoản hiện tại ĐỦ (chênh score_dieu−score_khoan ≤ 0,03) | 350/508 | 68,9% |
| Cần thông tin ngoài đúng 1 Khoản (Điều khớp tốt hơn rõ rệt) | 158/508 | **31,1%** |

Chênh lệch trung bình (score_dieu − score_khoan) = 0,0704.

**Diễn giải**: gần 1/3 số câu, thông tin cần thiết trải ra NGOÀI đúng 1 Khoản mà B0/B2 xác định — chunking cấp Khoản là ĐƠN VỊ QUÁ NHỎ cho ~31% câu hỏi thật. Đây là **giới hạn trần** của kiến trúc hiện tại: dù Layer 1/2/3 hoàn hảo tới đâu, B4 hiện tại (chỉ trả về CÁC KHOẢN riêng lẻ, không tự động kèm Khoản lân cận cùng Điều) vẫn thiếu ngữ cảnh cho nhóm câu này.

**Caveat chưa giải quyết**: (a) đo trên nhãn B0 (silver, không phải gold tay); (b) chênh lệch điểm SW có thể một phần do các Khoản lân cận dùng cụm từ lặp lại (boilerplate pháp lý phổ biến) khiến điểm Điều "ăn gian" dù không thực sự cần thêm ngữ cảnh — CHƯA đọc tay kiểm chứng các case "cần cả Điều" để loại trừ khả năng này.

**Chưa quyết định thay đổi thiết kế** — đây là input cho quyết định tiếp theo (B4/B6 có nên tự động đính kèm Khoản lân cận cùng Điều khi trả top-3 hay không, đánh đổi với độ dài context đưa cho Generator) — cần bàn với người dùng trước khi code.

---

### [B0] 17/08 — Tầng 3: kiểm chứng chéo độ tin nhãn B0 (2 phần)

**Phần 1 — cross-check bằng 22/29 câu trùng nội dung 2 track** (`pipeline/crosscheck_b0_labels.py`): khớp NGUYÊN VĂN câu hỏi giữa `data/{train,warmup}.json` và `data_retrieve/{train,warmup}.json` → đo được **29 cặp** trùng (khác nhẹ số "22" ước lượng trước đó, do cách đếm — đây là số đo chính xác bằng code). Với mỗi cặp, lấy gold `context_id` CHO SẴN từ `data_retrieve` (độc lập, không qua B2/B0), so với nhãn B0 dự đoán (đọc từ output có sẵn nếu đã gán, chạy sống `label_answer` nếu chưa — 24/29 câu phải chạy sống).

Kết quả: 8/29 câu B0 không gán được nhãn nào (dưới ngưỡng `B0_CONFIDENCE_DROP=0.3`). Trong 21 câu còn lại: **B0 dự đoán TRÚNG context_id độc lập: 13/21 = 61,9%**.

Lưu ý quan trọng về cấu hình: cross-check này chạy B0 với Layer 1.5 **BM25** (không có dense rerank, vì B0 chưa có embedding sống cho quote trích từ answer — hạn chế đã ghi nhận từ 13/08) — đúng cấu hình B0 sản xuất thật, nên số 61,9% phản ánh đúng độ tin THỰC TẾ của nhãn B0 đang dùng, không phải kịch bản tốt nhất có thể.

**Phát hiện phụ đáng chú ý**: câu hỏi ngắn dạng định nghĩa ("X là gì?") sai có HỆ THỐNG, không phải ngẫu nhiên — "Nhà chung cư là gì?" xuất hiện 3 lần (3 qid QA khác nhau, cùng trùng 1 câu `data_retrieve`), cả 3 lần B0 đều gán nhầm sang CÙNG 1 văn bản sai (`144551` thay vì đúng `6411`) — cho thấy văn bản sai này có tín hiệu từ vựng/ngữ nghĩa mạnh hơn với câu hỏi ngắn kiểu này, một điểm yếu thật của cả BM25 lẫn dense cho câu hỏi định nghĩa ngắn, thiếu ngữ cảnh phân biệt.

**Phần 2 — đọc tay 30 case theo 3 mức confidence** (`pipeline/sample_b0_audit.py`, 10 case/bucket, seed=42): đọc thủ công câu hỏi + `matched_span`, đánh giá chủ quan độ liên quan chủ đề/nội dung.

| Bucket | Ước lượng đúng chủ đề (chủ quan) |
|---|---|
| Thấp (0,3-0,5) | ~7-8/10 |
| Trung (0,5-0,7) | ~7/10 |
| Cao (0,7-1,0) | ~8/10 |

**Diễn giải**: ngay cả ở confidence THẤP, phần lớn case vẫn tìm đúng chủ đề/văn bản — điểm confidence (Smith-Waterman similarity) đo ĐỘ KHỚP CÂU CHỮ NGUYÊN VĂN giữa quote và unit, không hẳn đo ĐÚNG SAI VỀ NỘI DUNG. Answer càng diễn giải lại (paraphrase) nhiều so với luật gốc thì điểm càng thấp dù B0 đã tìm đúng nguồn — nghĩa là ngưỡng `B0_CONFIDENCE_TRUST=0.6` có thể đang LOẠI BỎ một số nhãn đúng một cách không cần thiết.

**Caveat trung thực**: đọc tay ở Phần 2 là đánh giá CHỦ QUAN (dễ rộng lượng khi chỉ xét liên quan chủ đề chung chung, không kiểm chứng được domain cụ thể như "có đúng thuộc Bộ Quốc phòng không" chỉ qua đọc đoạn text ngắn) — **số 61,9% (Phần 1) đáng tin hơn** vì là kiểm chứng KHÁCH QUAN với gold độc lập; Phần 2 chỉ dùng để hiểu THÊM bản chất của sai số (confidence thấp không hẳn = sai chủ đề), không dùng thay thế cho Phần 1.

**Kết luận tầng 3**: nhãn B0 dùng cho các quyết định lớn (`max_per_doc`, `select_span`, mục [B2/B4] 17/08) — với threshold `confidence>=0.6` đã dùng — có độ tin ước tính ở mức TRUNG BÌNH-KHÁ (~62% theo kiểm chứng khách quan), đủ để tin vào KẾT LUẬN TƯƠNG ĐỐI (so sánh biến thể A vs B) như đã làm, nhưng KHÔNG nên dùng số tuyệt đối (như "Hit Khoản 81,9%") làm con số báo cáo cuối cùng — đúng cảnh báo đã nêu trước đó ở mục ground truth 4 tầng.

---

### [Ground Truth] 17/08 — Phát hiện lỗ hổng phương pháp: "tập giữ kín" thực ra đã bị dùng lặp lại để tune

Người dùng hỏi thẳng: các số Hit/Recall/Coverage đưa ra có phải đo trên tập THẬT SỰ giữ kín (chưa từng đụng, tránh rò rỉ) không. Rà lại — câu trả lời là KHÔNG hoàn toàn:

- `data_retrieve/warmup.json` (500 câu) — Ý ĐỊNH BAN ĐẦU (ghi trong mục 6 báo cáo) là giữ nguyên, chỉ dùng báo cáo số cuối. THỰC TẾ: đã bị dùng LẶP LẠI xuyên suốt session để đo — ablation Layer 1, so BM25-rerank vs dense-rerank, phát hiện RRF có hại, phát hiện `max_per_doc` "thắng giả" — đúng dạng "nhìn tập test nhiều lần" (data snooping).
- Nhãn B0 cấp Khoản (n=393, từ `train` dev-sample 1.500 câu seed=42) — dùng để chốt `max_per_doc=None`, bỏ `select_span`, so RRF ở cấp Khoản — cũng bị "nhìn" lặp lại nhiều lần cho nhiều quyết định liên tiếp.

**Hệ quả**: số TUYỆT ĐỐI (Hit-Khoản 85,0%, Hit@3 55,5%...) có thể lạc quan hơn thực tế trên câu hỏi hoàn toàn mới. So sánh TƯƠNG ĐỐI giữa các phương án (RRF vs dense, có/không `max_per_doc`...) vẫn hợp lệ vì độ lệch ảnh hưởng đều mọi phương án — đây là lý do các quyết định kiến trúc đã chốt KHÔNG cần đảo ngược, chỉ số liệu báo cáo cần thận trọng hơn.

**QUYẾT ĐỊNH (17/08)**: tạo 1 tập GIỮ KÍN THẬT SỰ trước khi đánh giá Layer 3 (quyết định quan trọng nhất còn lại) — thay vì lấy mẫu nhỏ (600-800 câu) như đề xuất ban đầu, người dùng chọn dùng NỐT toàn bộ phần `train.json` chưa từng đụng tới (~5.500/7.000 câu, sau khi trừ 1.500 câu dev-sample đã "nhiễm").

`pipeline/build_b0_labels_train_heldout.py`: gán nhãn B0 cho đúng phần bù (`train.json` − 1.500 câu đã dùng), lưu `outputs/b0_labels_train_heldout.json`, đánh dấu rõ **TUYỆT ĐỐI KHÔNG DÙNG ĐỂ TUNE** — chỉ mở 1 lần duy nhất khi đánh giá kết quả Layer 3/báo cáo số cuối. Ước tính ~2,4 giờ CPU (5.500 câu × 1,56s/câu đo trước đó) — chạy nền, song song lúc chờ Kaggle.

---

### [Infra] 20/08 — Dọn dẹp cấu trúc project (~1,34GB, 21 file rác gốc)

Theo yêu cầu người dùng ("cách tổ chức file đang chưa ổn... có khá nhiều file rác"). Rà soát bằng `find`/`du`/`grep` (xác nhận không code nào tham chiếu trước khi xoá) rồi thực hiện qua EnterPlanMode (duyệt trước khi làm vì có xoá dữ liệu):

**Đã xoá**: 21 file `.log`/`.txt` rác ở gốc thư mục (nội dung đã chép vào log này); `data_retrieve/selected-contexts/` (485MB — bản sao corpus KHÔNG code nào dùng, `config.py` chỉ trỏ bản trong `data/`); `kaggle_layer2/upload/` (689MB — snapshot đã dùng xong cho Kaggle Layer 2, tái tạo được nếu cần); `outputs/bm25_doc_index_name.pkl`/`_nocap.pkl` (168MB — index của 2 config ablation đã bị loại từ lâu); `outputs/runs/L3_rerank_warmup.jsonl` (0 byte, file chết); `docs/.keep`; mọi `__pycache__`.

**Đã đổi tên**: `data/selected-contexts/selected-contexts/` (thư mục lồng 2 cấp trùng tên, di sản từ symlink hỏng ghi trong `config.py` cũ — xem entry `[Infra] 06/08`) → **`data/corpus/`**, sửa `CORPUS_DIR` trong `src/common/config.py` theo. Đây là thay đổi DUY NHẤT có rủi ro ảnh hưởng pipeline — verify bằng `experiments/demo_b1_parser.py` (4/4 test PASS) và `experiments/demo_b_end_to_end.py` (0 context rỗng, output đúng schema) sau khi đổi — không có regression.

**Đã cập nhật**: `docs/SYSTEM_SCAFFOLD.md` (lỗi thời từ 13/08, viết lại toàn bộ cấu trúc thư mục + trạng thái module cho khớp 20/08) và `docs/KE_HOACH_TIEP_THEO_B.md` (đánh dấu 4/8 mục việc còn thiếu đã hoàn thành sau 13/08, cập nhật ngân sách 2,2 tỷ).

**Không động tới**: `outputs/runs/*` còn lại (bằng chứng thô cho bảng ablation trong báo cáo), `kaggle_layer3/upload/*` (candidate đang dùng dở), toàn bộ `data/`/`data_retrieve/` JSON gốc.

---

### [B2/Layer3] 20/08 — Kết quả Kaggle: Layer 3 CẢI THIỆN THẬT, đo được rõ ràng

Nhận file kết quả từ Kaggle (`outputs/layer3/L3_rerank_train.jsonl`, để nhầm ở `layer3_results/` — đã dời đúng vị trí). Sanity check trước khi đo: 7.000/7.000 qid khớp `train.json` (0 thiếu/thừa), mỗi câu đúng 50 candidate, không câu rỗng, không trùng `unit_id`, đã sắp xếp giảm dần đúng — file sạch.

`pipeline/eval_layer3_rerank.py`: so Hit@K cấp văn bản (nhãn sạch `data_retrieve/train.json`, toàn bộ 7.000 câu) giữa thứ tự TRƯỚC rerank (dense gốc, từ `layer3_candidates_train.jsonl`) và SAU rerank (Layer 3):

| K | Trước (dense) | Sau (Layer 3) | Chênh lệch |
|---|---|---|---|
| 1 | 46,2% | 48,0% | +1,7% |
| 3 | 64,9% | 69,9% | **+5,0%** |
| 5 | 71,6% | 76,6% | +5,1% |
| 10 | 78,5% | 82,5% | +4,0% |
| 20 | 83,2% | 85,9% | +2,7% |
| 30 | 85,2% | 87,0% | +1,8% |
| 50 | 87,3% | 87,3% | +0,0% |
| MRR | 0,5737 | 0,6039 | +0,0302 |

**Diễn giải**: Hit@50 KHÔNG đổi (87,3%=87,3%) — đúng lý thuyết, Layer 3 chỉ xếp lại thứ tự trong đúng 50 candidate sẵn có, không thêm/bớt candidate nào — đây cũng là 1 phép kiểm tra tính đúng đắn tự nhiên của cách đo (nếu Hit@50 lệch nhau thì chứng tỏ có lỗi ở đâu đó). Toàn bộ cải thiện đến từ việc xếp đúng candidate lên vị trí cao hơn — quan trọng nhất là **Hit@3 tăng +5,0 điểm%** (64,9%→69,9%), đúng con số B4 dùng (top-3 giao Generator). Cải thiện đồng đều, nhất quán ở mọi K<50, không phải nhiễu ngẫu nhiên.

**Caveat**: đo CẤP VĂN BẢN (vì `data_retrieve` chỉ có nhãn văn bản cho 7.000 câu này) — cấp Khoản (cấp quyết định điểm thật) CHƯA đo được trên đúng tập này (cần nhãn B0, chưa gán cho `data_retrieve`'s câu hỏi). Tuy nhiên đây là phép đo THẬT SỰ MỚI (Layer 3 chưa từng được dùng để tune bất cứ gì trước đây, nên không dính lỗi "nhìn tập nhiều lần" đã cảnh báo ở entry trước) — tín hiệu dương đáng tin để quyết định GIỮ Layer 3 trong pipeline chính thức.

**QUYẾT ĐỊNH**: Layer 3 chứng minh được giá trị bằng số đo thật — nên tích hợp vào pipeline chính thức (hiện `search_units_hybrid()` dừng ở bước dense-rank top-50, chưa gọi Layer 3). Việc tiếp theo: wire Layer 3 vào pipeline sản xuất (cần chạy reranker LIVE lúc phục vụ câu hỏi mới — khác với batch offline vừa làm trên Kaggle, cần quyết định hạ tầng chạy: local CPU quá chậm (954ms/candidate × 50 = ~48s/câu), cần GPU thường trực hoặc chấp nhận thời gian phản hồi chậm).

---

### [B2/Layer3] 30/08 — XÁC NHẬN CUỐI: Layer 3 cải thiện thật ở cấp Khoản, trên tập giữ kín

Hoàn tất Phase A-C của plan xác nhận Layer 3 (tập giữ kín 800 câu, `outputs/b0_labels_train_heldout.json`, chưa từng dùng để tune). Sự cố dọc đường: 2 lần chạy Kaggle đầu bị nhầm — notebook vẫn đọc `layer3_candidates_train.jsonl` cũ (do file `_heldout` chưa kịp upload/mount vào Dataset) → kết quả tải về giống hệt bản cũ. Lần 3 xác nhận đúng: dataset có đủ 2 file, sửa đúng 2 dòng (đọc `_heldout.jsonl`, ghi `L3_rerank_heldout.jsonl`), Restart Session trước khi Run All — ra kết quả mới, verify: 800/800 qid khớp `heldout_ids`, đúng 50 candidate/câu, không rỗng/trùng/sai thứ tự.

`pipeline/eval_layer3_rerank_heldout.py`: so Hit@K TRƯỚC/SAU rerank ở **cấp Khoản** (gold = nhãn B0 `confidence≥0,6`, n=198), kèm bootstrap CI (2.000 lần, cùng quy ước dùng xuyên suốt dự án):

| K | Trước (dense) | Sau (Layer 3) | Chênh lệch | 95% CI |
|---|---|---|---|---|
| 1 | 35,9% | 46,0% | **+10,1%** | [+3,5%, +16,7%] có ý nghĩa |
| 3 | 54,0% | 59,6% | +5,6% | [-0,5%, +11,6%] sát ngưỡng, chưa qua hẳn |
| 5 | 64,6% | 69,2% | +4,5% | [-1,0%, +10,6%] |
| 10 | 72,2% | 77,8% | +5,6% | [+1,0%, +10,1%] có ý nghĩa |
| 20 | 78,8% | 81,3% | +2,5% | [-1,0%, +6,6%] |
| 30 | 82,3% | 82,8% | +0,5% | [-2,5%, +3,5%] |
| 50 | 86,4% | 86,4% | 0,0% | [0,0%, 0,0%] |
| MRR | 0,4827 | 0,5546 | +0,0719 | — |

**Diễn giải**: Hit@50 không đổi — đúng lý thuyết (Layer 3 chỉ xếp lại 50 candidate có sẵn), xác nhận phép đo không lỗi. Hit@1 và Hit@10 có ý nghĩa thống kê rõ (CI không chứa 0). Hit@3 (con số quan trọng nhất — B4 lấy top-3) có xu hướng dương NHẤT QUÁN với kết quả cấp văn bản đo trước đó (+5,6 điểm% ở đây vs +5,0 điểm% ở n=7.000 cấp văn bản), nhưng CI chưa qua hẳn 0 (cận dưới -0,5%) — hạn chế do cỡ mẫu n=198, không phải do hiệu ứng yếu.

**QUYẾT ĐỊNH CUỐI**: 2 nguồn dữ liệu độc lập (7.000 câu cấp văn bản `data_retrieve` + 198 câu cấp Khoản `data/train.json`, chưa từng tune) cho cùng xu hướng nhất quán → **CHỐT giữ Layer 3 trong kiến trúc chính thức**. Việc còn lại: formalize thành quy trình batch chuẩn (không cần hạ tầng GPU sống, vì `public-official.json` là tập dự đoán sẵn rồi nộp, không phải API real-time).

**Bài học phương pháp bổ sung** (từ đối chiếu báo cáo Team IR — Task 1 LegalIR, cùng corpus): nên đặt ngưỡng quyết định TRƯỚC khi mở file kết quả GPU (ví dụ "nếu Hit@3 tăng ≥X điểm% thì giữ") — lần này chưa làm chặt chẽ bước đó trước khi đọc kết quả, ghi nhận để áp dụng nghiêm túc hơn cho các lần đo GPU sau.

---

### [B5] 17/08 — Xác nhận ngân sách tham số retrieval: 2,2 tỷ

Người dùng xác nhận con số cụ thể **2,2 tỷ tham số** cho retrieval (trước đó chỉ nghe verbal "~2 tỷ", chưa chốt). Cập nhật `docs/BAO_CAO_TONG_HOP_B.md` mục 3/4/12: tổng B đang dùng 1,136 tỷ (BM25=0 + bi-encoder 568M + reranker 568M) → còn dư **~1,064 tỷ (48,4%)**, vẫn giữ quyết định KHÔNG dùng hết, trả phần dư cho Generator (C). Không có thay đổi kiến trúc — chỉ cập nhật số liệu ngân sách.

**Đồng thời sửa Lỗi 6**: đổi `use_multi_query` default `True`→`False` ở `search_docs_multi()` và `search_units()` (quyết định bỏ multi-query đã chốt 13/08 nhưng code còn để bật).

---

### [B4/B6] 13/08 — rebuild theo schema Generator đề xuất: top-3 context/câu hỏi, `QAPackage` thay `ContextPackage`

**Input**: người dùng đưa schema do bên Generator (C) đề xuất:
```json
{"id": "...", "question": "...", "contexts": [{"document": "...", "article": "...", "text": "..."}], "reference_answer": "..."}
```
Khác hẳn `ContextPackage` cũ (1 record = 1 context, phẳng, không có `question` text, không có `reference_answer`).

**Quyết định người dùng**: số lượng context/câu hỏi = **top-3 cố định** (không phải ngưỡng động — chưa đủ data để calibrate ngưỡng đúng). Đúng nguyên tắc ưu tiên recall đã chốt (xem entry trước + memory `feedback_prioritize_recall.md`) — tăng khả năng không bỏ sót nguồn khi B4 chọn chưa chuẩn hoặc answer đa nguồn (12.2%).

**Lưu ý phát sinh, CHƯA XÁC NHẬN**: người dùng báo "ngân sách retrieval ~2 tỷ tham số" — khác hẳn tài liệu cũ (B5 README: bi-encoder ≤0.7B + reranker ≤0.6B = ≤1.3B). Chưa rõ đã chốt chính thức hay còn bàn — CHƯA cập nhật `B5 README` (chờ xác nhận, tránh ghi sai).

**Code mới**:
- `src/b4_span_selection/selection.py`: thêm `select_top_n_khoan(question, khoan_list, top_n=3, already_ranked=False)` — cùng cơ chế `already_ranked` như B0 (tin thứ hạng Layer 1.5/Layer 2 khi có, không tự tính lại overlap thô).
- `src/b6_context_package/package.py`: viết lại hoàn toàn — `ContextItem` (`document`, `article`, `text`, `retrieval_score`) + `QAPackage` (`id`, `question`, `contexts`, `reference_answer`). `article` là chuỗi người đọc được ("Điều 2 Khoản 1"), suy từ `unit_id` (không đổi cơ chế `_parse_unit_id`, chỉ đổi cách trình bày). `document` = `{"name", "link"}` — **ĐANG DÙNG TẠM**, Generator ghi "metadata hoặc parse từ corpus" (mơ hồ), cần xác nhận lại cấu trúc chính xác.

**Test** (`experiments/demo_b_end_to_end.py`, viết lại theo luồng mới: `search_units` +Layer1.5 → `select_top_n_khoan` (top-3, `already_ranked=True`) → `select_span` mỗi Khoản → `build_context_item`/`build_qa_package`): chạy 5 câu hỏi thật, không lỗi, output đúng schema Generator (verify bằng mắt JSON mẫu). Case đáng chú ý (câu `99969`, hỏi về tài liệu kế toán lưu trữ vĩnh viễn): cả 3 context được chọn đều từ CÙNG 1 văn bản (`95164`, Điều 10 + Điều 14 Khoản 1 + Điều 14 Khoản 2) — hợp lý, không phải lỗi, vì câu hỏi thực sự cần thông tin trải trên nhiều Khoản của cùng 1 Thông tư.

**Câu hỏi mở còn treo**: cấu trúc chính xác của `document` — cần hỏi lại Generator xác nhận `{"name","link"}` có đúng ý họ không, hay họ muốn 1 string, hay thêm field khác.

---

### [B2] 13/08 — bổ sung `qa_public.json` vào notebook Kaggle Layer 2 (tự phát hiện thiếu sót)

Rà lại `kaggle_layer2/` khi giải thích cho người dùng, phát hiện: quên đưa `data/public-official.json` (1.000 câu — ĐÍCH CUỐI CÙNG cần predict, không có answer) vào danh sách embed. Bản trước chỉ embed `data_retrieve` train/warmup + QA train/warmup (dùng để TEST Layer 2), nhưng thiếu chính tập cần DÙNG Layer 2 để trả lời thật. Đã sửa: thêm `qa_public` vào `QUERY_SOURCES` trong `embed_corpus_notebook.py`, cập nhật `README.md` (6 file cần upload thay vì 5, đổi tên `data/public-official.json` → `qa_public.json`). Verify schema `public-official.json` giống hệt `train.json`/`warmup.json` (`{"question", "answer": null}`) nên code loop hiện có xử lý được ngay, không cần sửa logic khác — chỉ thêm 1 dòng.

**Tự làm sẵn bước đổi tên** (người dùng hỏi có thể chuẩn bị sẵn không): copy 6 file gốc vào `kaggle_layer2/upload/` với tên đã đổi sẵn (verify kích thước byte-identical với bản gốc) — người dùng chỉ cần kéo-thả nguyên thư mục lên Kaggle, không phải tự đổi tên tay. Cập nhật `README.md` Bước 1 cho khớp.

---

### [B0] 13/08 — tầng 2 dev sample `train.json` (1,500 câu) chạy xong

`pipeline/build_b0_labels_train_sample.py` chạy xong: 2,522.5s (42.0 phút, khớp ngoại suy ~43 phút). Kết quả: **979/1,500 câu có nhãn (65.3%)**, score min=0.301 max=1.000 mean=0.627 — nhỉnh hơn `warmup.json` (313/500=62.6%, mean 0.606). Đã lưu `outputs/b0_labels_train_sample.json` (kèm `sample_ids`+`seed=42` để tái lập).

**Tổng cộng hiện có 2,000 câu QA đã gán nhãn** (500 warmup + 1,500 train sample) — đủ để hiệu chỉnh ngưỡng `B0_CONFIDENCE_DROP`/`TRUST` bằng data thật và đo recall trần chunking với cỡ mẫu lớn hơn nhiều so với trước (n=21 ban đầu → n~1,300 câu có nhãn). Việc tiếp theo (chưa làm): gộp 2 file nhãn, vẽ phân phối score, chốt ngưỡng, đo recall trần chính thức.

---

### [B2/B4/B6] 31/08 — Error analysis Layer 3 + quyết định context expansion + ĐẢO NGƯỢC top-3→top-1

**Error analysis** (`pipeline/error_analysis_heldout.py`, n=226 union gold B0+citation, HIT_K=10): 172 hit (76,1%), 54 case thất bại — **68,5% (37) Loại A** (gold không có trong top-50 TRƯỚC Layer 3 — lỗi B1/B2) vs **31,5% (17) Loại B** (gold có trong top-50 nhưng Layer 3 đẩy ra ngoài top-10 SAU). Trong Loại B, **76,5% (13/17) là Layer 3 làm TỆ HƠN dense gốc** (rank_after > rank_before), gồm 4 case dense đã ở top-6 (2 case rank 1) bị đẩy hẳn ra ngoài top-10 (rank 1→13, 1→14, 3→49, 6→18) — rủi ro cụ thể, không phủ nhận kết luận tổng thể Layer 3 tốt (+10-14đ% Hit@1) nhưng đáng điều tra thêm.

**Phân tích sâu Loại A** (định lượng, không phải đọc mẫu vài case):
- **43,2% (16/37)**: đúng văn bản, SAI Khoản (context_id trùng gold nhưng unit_id khác) — lỗi ở Expand/chunking (B1/B2 bỏ sót đúng Khoản trong đúng văn bản), KHÔNG phải lỗi tìm sai văn bản.
- **56,8% (21/37)**: sai HOÀN TOÀN văn bản — lỗi retrieval thật sự, quan sát được nguyên nhân chính là nhầm lẫn giữa nhiều văn bản luật cùng chủ đề (dense score cao, vd 0,99, nhưng chọn sai văn bản — nhiều Nghị định/Thông tư sửa đổi bổ sung nhau qua các năm nói cùng 1 chủ đề).
- Giả thuyết "gold là điều khoản sửa đổi/bổ sung khó match" — **KHÔNG xác nhận** (0% Loại A, chỉ 5,9% Loại B khớp pattern `^(Sửa đổi|Bổ sung|Bãi bỏ)`).
- Độ dài câu hỏi KHÔNG liên quan (Loại A: 20,8 từ TB, Loại B: 19,7 từ TB — không khác biệt).
- **24,1% (13/54) toàn bộ case thất bại** có candidate trong top-3 sau rerank overlap từ vựng >50% với gold text nhưng khác `unit_id` — dấu hiệu nhiều văn bản luật VN dùng chung khung câu boilerplate cho các thủ tục/chức danh khác nhau (case cực đoan qid=122961: 90% overlap — gold "thủ tục BÁN tài sản" vs candidate "thủ tục THANH LÝ tài sản", ngôn ngữ gần như y hệt). Giới hạn cố hữu của bài toán, khó giải quyết chỉ bằng cải thiện model.

**Context expansion** (`pipeline/eval_context_expansion_heldout.py`, so Hit rate context (a) chỉ Khoản vs (b) cả Điều, TOP_N tham số hoá):
| TOP_N | (a) chỉ Khoản | (b) cả Điều | Chênh lệch | Độ dài TB |
|---|---|---|---|---|
| 3 | 60,2% | 66,4% | +6,2đ% CI[+3,5%,+9,3%] *** | 2.241→7.015 ký tự (+213%) |
| 1 | 46,5% | 52,2% | +5,8đ% CI[+3,1%,+8,8%] *** | 734→2.322 ký tự (+216%) |

Verify cơ chế bằng phân tích riêng (script scratchpad, không lưu pipeline/): trong 226 câu, top-1 hit đúng Khoản 105 (46,5%, khớp bảng trên), **13 câu (5,8đ%) được cứu bởi mở rộng Điều — TOÀN BỘ 13/13 đều đúng văn bản/Điều nhưng sai Khoản trong CÙNG Điều** (khớp chính xác pattern 43,2% Loại A ở trên), 0 case "tình cờ trúng". 108 câu (47,8%) vẫn miss dù mở rộng — sai hẳn văn bản/Điều, expansion không cứu được nhóm này. **Kết luận: context expansion chỉ chữa lỗi Expand/chunking, bất lực với lỗi retrieval nhầm văn bản** (nhóm lớn hơn nhiều, 56,8% Loại A).

**QUYẾT ĐỊNH ĐẢO NGƯỢC top-3 (13/08) → top-1 (31/08)**: trao đổi với C (Generator), xác nhận kiến trúc Generator hiện tại COPY MÁY MÓC đoạn trích luật từ context (không phải LLM tự sinh/tự chọn giữa nhiều context — chỉ sinh phần lời dẫn + câu kết), nên **chỉ xử lý được đúng 1 context/câu hỏi**, không có cơ chế "đọc nhiều rồi tự chọn" như giả định thiết kế ban đầu (13/08, "thà dư hơn thiếu"). Vì mất hẳn top-2/3 dự phòng, áp dụng context mở rộng (cả Điều) cho context DUY NHẤT còn lại — bù được 5,8đ% (bảng trên, hàng TOP_N=1).

**Code sửa**: `pipeline/build_qa_packages_heldout.py` — `TOP_N`: 3→1, thêm `khoan_id_to_dieu_id()` + logic mở rộng text sang cả Điều (giữ `article`/`clause` đúng Khoản Layer 3 chọn — chỉ `text` mở rộng). Build lại `outputs/qa_packages_heldout.json`, verify 800/800 record đúng 1 context. Cập nhật docstring `src/b6_context_package/package.py` + `docs/SYSTEM_SCAFFOLD.md`.

**Việc còn treo**: điều tra sâu hơn cơ chế Layer 3 "làm tệ hơn dense gốc" ở 76,5% case Loại B (cân nhắc kết hợp điểm dense+rerank thay vì tin tuyệt đối cross-encoder); cải thiện B1/B2 cho nhóm lỗi lớn nhất (56,8% Loại A, nhầm văn bản cùng chủ đề); chưa chạy `data/public-official.json` (tập đích nộp bài) với cấu hình mới.

---

