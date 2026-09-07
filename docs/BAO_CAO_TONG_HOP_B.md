# Báo cáo tổng hợp — Module B (Retrieval & Span Selection), DSC 2026 Task 2 LegalQA

> Cập nhật 31/08/2026. Viết cho người CHƯA biết gì về phần việc này — mọi cơ chế, phương pháp đo, con số đều giải thích từ gốc, kèm căn cứ đo lường. Nguồn: `docs/EXPERIMENT_LOG.md` (tới 30/08) + `docs/SYSTEM_SCAFFOLD.md`, `README.md`, `outputs/LAYER3_EVALUATION_SUMMARY.md`, code thật trong `src/`/`pipeline/` (khối lượng việc lớn ngày 31/08 — ngân sách, schema B6, đảo ngược TOP_N, citation labels, error analysis — **CHƯA kịp ghi vào `EXPERIMENT_LOG.md`**, lấy trực tiếp từ code/output/`SYSTEM_SCAFFOLD.md` đã cập nhật; nên backfill vào log sau).

---

## I. Bối cảnh & phạm vi

### Bài toán chung

Cả nhóm giải bài toán **LegalQA** (DSC 2026 Task 2): cho 1 câu hỏi luật tiếng Việt, hệ thống phải trả lời dựa trên 1 corpus 8.512 văn bản pháp luật. Câu trả lời chuẩn có 3 khối nội dung:

- **Khối 1 — căn cứ pháp lý** (~7% trọng số điểm).
- **Khối 2 — trích nguyên văn điều luật liên quan** (~60% trọng số điểm) — phần lớn nhất.
- **Khối 3 — kết luận/diễn giải** (~25% trọng số điểm).

### Cách chấm điểm và ý nghĩa với thiết kế B

Chấm bằng **METEOR (chính)** + **ROUGE-L (phụ)**. **METEOR phạt câu trả lời THIẾU nội dung nặng hơn câu trả lời DƯ khoảng 3-4 lần.** Nguyên tắc **"ưu tiên recall, thà dư hơn thiếu"** là kim chỉ nam xuyên suốt — dù mục III.10 cho thấy nguyên tắc này vừa được **hiệu chỉnh lại** (không phải bỏ) sau khi hiểu rõ hơn cách Generator thực sự dùng context.

### Cấu trúc nhóm và phạm vi module B

Hệ thống có **4 phần**: **A** (chấm điểm/oracle eval), **B** (retrieval & span selection — chính là phần trong báo cáo này), **C** (generator, sinh câu trả lời cuối), **D**/Trưởng nhóm (điều phối chung). B chịu trách nhiệm: (1) tìm đúng văn bản luật, (2) tìm đúng đoạn (Khoản/Điều) trong đó, (3) đóng gói lại thành context chuẩn hoá giao cho C. **B dừng lại ở bước đóng gói** — không tự sinh câu trả lời, và **B7 (oracle eval qua Generator) KHÔNG PHẢI việc của B** (thuộc A) — B chỉ cần đảm bảo output đúng schema để A/C dùng được. Pipeline nội bộ B: B0 (tạo nhãn phục vụ đo lường), B1 (tách văn bản), B2 (retrieval), B3 (đo lường), B4 (chọn context), B5 (phân bổ ngân sách), B6 (đóng gói).

---

## II. Kiến trúc hiện tại

### Sơ đồ pipeline đầy đủ

```
Câu hỏi
   │
   ├──────────────────┬──────────────────┐
   ▼                  ▼
┌─────────────┐  ┌──────────────────┐
│ Layer 1      │  │ Layer 2           │   ← CHẠY SONG SONG, độc lập nhau
│ BM25 (từ)    │  │ Dense/BGE-M3      │
│ cấp văn bản  │  │ cấp UNIT (Khoản)  │
│ top-100 doc  │  │ top-300 unit      │
│ 0 tham số    │  │ 568 triệu tham số │
└─────────────┘  └──────────────────┘
   │ Expand ra Khoản     │
   └────────┬─────────────┘
            ▼
      Union ở CẤP UNIT (không collapse về văn bản)
            ▼
      Xếp hạng TOÀN BỘ bằng điểm DENSE
            ▼
      Cắt top-50, KHÔNG ràng buộc đa dạng văn bản
            ▼
      Layer 3 — Reranker cross-encoder   ✅ CHỐT GIỮ (3 nguồn nhãn độc lập đồng thuận, mục III.7-9)
      (BAAI/bge-reranker-v2-m3, ~568 triệu tham số)   🔶 batch qua Kaggle GPU, KHÔNG wire trực tiếp
                                                          (không cần suy luận sống — B nộp file, không
                                                          phải API thời gian thực)
            │
            ▼
      B4 — chọn ĐÚNG 1 context (TOP_N=1, đảo ngược từ top-3 ngày 31/08 — mục III.10)
            │
            ▼
      B6 — đóng gói QAPackage, context MỞ RỘNG trả cả Điều (không chỉ Khoản) — bù việc chỉ có 1 context
            │
            ▼
      Generator (C) — copy máy móc đoạn trích, không tự chọn giữa nhiều context
```

Luồng Layer 1+2+Union+xếp hạng gói gọn trong `retrieve.search_units_hybrid()` (~0,26 giây/câu, không cần GPU). Layer 3 chạy theo quy trình batch riêng (sinh candidate local → Kaggle GPU rerank → tải kết quả về) — hợp lý vì đầu ra cuối cùng của B là 1 file nộp, không phải API thời gian thực.

### Cơ chế từng tầng

**Layer 1 — BM25 (lexical/từ khoá).** `score(D,Q) = Σ IDF(t)·f(t,D)·(k1+1) / (f(t,D)+k1·(1−b+b·|D|/avgdl))`. 0 tham số học, nhanh, nhưng mù ngữ nghĩa.

**Layer 2 — Dense/bi-encoder (BGE-M3, 568 triệu tham số).** Chuyển mỗi đoạn text thành 1 vector sao cho 2 đoạn CÓ Ý NGHĨA GIỐNG NHAU thì vector GẦN NHAU, dù từ ngữ khác hẳn. Tạo vector cần GPU (1 lần cho 432.473 Khoản), tìm kiếm sau đó chỉ cần nhân ma trận (CPU, ~5,6ms/câu).

**Layer 3 — Cross-encoder/reranker (BAAI/bge-reranker-v2-m3, ~568 triệu tham số).** Đưa CẢ câu hỏi VÀ văn bản vào CÙNG 1 model chấm điểm — chính xác hơn nhưng đắt hơn nhiều (954ms/candidate trên CPU) — chỉ khả thi trên top-50 candidate đã thu hẹp.

### Ngân sách tham số (SỬA 31/08 — số cũ 2,2 tỷ đã lỗi thời)

**Xác nhận chính thức qua Generator (C), 31/08**: tổng ngân sách **toàn hệ thống là 4 tỷ tham số** — C (Generator) dùng **1,7 tỷ**, phần còn lại **2,3 tỷ dành cho B**.

| Layer | Model | Tham số |
|---|---|---|
| Layer 1 (BM25) | `rank_bm25` | 0 |
| Layer 1.5 (rerank Khoản) | dùng lại embedding Layer 2 | 0 |
| Layer 2 (bi-encoder) | `BAAI/bge-m3` | 568 triệu |
| Layer 3 (reranker) | `BAAI/bge-reranker-v2-m3` | ~568 triệu |
| **Tổng B đang dùng** | | **~1,136 tỷ / 2,3 tỷ được cấp (49,4%)** |

Còn dư **~1,164 tỷ (50,6% phần của B)**. **Quyết định: KHÔNG dùng hết, không tự động chuyển cho C** — chỉ tiêu khi có bằng chứng đo lường cần. Layer 3 đã xác nhận cải thiện +10-14 điểm% Hit@1 (mục III.7-9) mà chưa cần dùng thêm tham số nào ngoài 2 model hiện có.

---

## III. Các quyết định thiết kế lớn — kèm căn cứ đo lường

Nguyên tắc chung: mọi lựa chọn giữ/bỏ đều có bảng số đo thật đi kèm.

### 1. Ablation Layer 1 (BM25 cấp văn bản) — 5 cấu hình, 4 bị loại

Đo trên 500 câu `data_retrieve/warmup.json`:

| Config | Nội dung thử | Hit@1 | Hit@5 | Coverage | MRR | Kết luận |
|---|---|---|---|---|---|---|
| `L1_doc` (baseline) | BM25 cấp văn bản, top-100 | 26,2% | 47,0% | 89,8% | 0,366 | Giữ làm baseline |
| `L1_doc_noMQ` | Tắt multi-query | Giống hệt baseline | | | | Vô dụng — tắt mặc định |
| `L1_doc_name` | Thêm tên văn bản | 26,2% | 47,2% | 89,8% | 0,366 | Không cải thiện — không giữ |
| `L1_doc_nocap` | Bỏ cap 50k ký tự | 24,4% | 47,8% | 89,4% | 0,361 | Không ý nghĩa thống kê — giữ cap |
| `L1_khoan` | Index thẳng cấp Khoản | 32,0% | 62,0% | 82,8% | 0,454 | Sắc nét hơn NHƯNG mất 7% coverage |

### 2. Layer 1.5 — rerank rẻ cấp Khoản, đổi cơ chế từ BM25 sang Dense

| Cách rerank | Hit@1 | Hit@5 | Hit@10 | MRR | Coverage |
|---|---|---|---|---|---|
| BM25-Khoản (bản cũ) | 32,2% | 61,4% | 70,4% | 0,452 | 89,8% |
| **Dense (tra bảng embedding có sẵn)** | **45,2%** | **71,0%** | **75,2%** | **0,564** | 89,8% (giống hệt) |

Coverage giống hệt, Hit@1 chênh 13 điểm%, MIỄN PHÍ. **Chốt: dense rerank.**

### 3. Union — sửa lỗi thiết kế "bị nhiễm từ cách đo"

Union cấp văn bản làm loãng candidate (mở lại toàn bộ Khoản của 1 văn bản dù dense chỉ thích 1 Khoản trong đó). **Sửa: `union_units()` gộp ở CẤP UNIT.**

### 4. RRF (Reciprocal Rank Fusion) — kết quả ÂM TÍNH

| Thiết kế | H@1 | H@3 | H@10 | H@100 | MRR |
|---|---|---|---|---|---|
| Union → xếp hạng THUẦN DENSE | **45,4%** | **65,4%** | 76,0% | **89,8%** | **0,566** |
| Union → RRF 1:1 | 17,8% | 51,6% | 75,8% | 87,2% | 0,373 |

RRF thưởng đồng thuận, phạt xuất sắc 1 nhánh — BM25 yếu hơn dense nhiều nên bị cào bằng, kéo tín hiệu mạnh xuống. **Quyết định: dense thuần, giữ RRF trong code (tắt mặc định).**

### 5. Ràng buộc đa dạng văn bản (`max_per_doc`) — "thắng giả"

| K | max/văn bản | Hit cấp Khoản | Hit cấp văn bản |
|---|---|---|---|
| 30 | không giới hạn | **81,9%** | 95,4% |
| 30 | **1** | **43,3%** | 98,7% |

`max_per_doc=1` giữ đúng văn bản nhưng SAI Khoản — Hit cấp Khoản sụp gần một nửa (hiệu ứng đèn đường: đo ở chỗ dễ đo thay vì đúng chỗ quyết định). **Chốt: `max_per_doc = None`.**

### 6. `select_span` (cắt Khoản xuống 1 đoạn nhỏ) — đo có hại, loại bỏ

Xem chi tiết ở mục IV.2 — chỉ giữ đúng đáp án gold 44,4%/151 case, tức xoá mất 55,6% số lần. Đã loại khỏi pipeline chính từ 17/08.

### 7. Layer 3 — kết quả đo lần 1 (20/08, cấp văn bản, n=7.000): cải thiện thật

| K | Trước (dense) | Sau (Layer 3) | Chênh lệch |
|---|---|---|---|
| 1 | 46,2% | 48,0% | +1,7% |
| 3 | 64,9% | 69,9% | **+5,0%** |
| 10 | 78,5% | 82,5% | +4,0% |
| 50 | 87,3% | 87,3% | +0,0% |
| MRR | 0,5737 | 0,6039 | +0,0302 |

Hit@50 không đổi (đúng lý thuyết — chỉ xếp lại 50 candidate có sẵn). Cải thiện tập trung đúng chỗ quan trọng (Hit@3 lúc đó là con số B4 dùng).

### 8. Layer 3 — xác nhận cấp Khoản trên tập giữ kín (30/08, nhãn B0, n=198/800)

| K | Trước (dense) | Sau (Layer 3) | Chênh lệch | 95% CI |
|---|---|---|---|---|
| 1 | 35,9% | 46,0% | **+10,1%** | [+3,5%, +16,7%] có ý nghĩa |
| 3 | 54,0% | 59,6% | +5,6% | [-0,5%, +11,6%] sát ngưỡng |
| 10 | 72,2% | 77,8% | +5,6% | [+1,0%, +10,1%] có ý nghĩa |
| 50 | 86,4% | 86,4% | 0,0% | đúng lý thuyết |
| MRR | 0,4827 | 0,5546 | +0,0719 | — |

**Bài học pre-registration**: nên đặt ngưỡng quyết định TRƯỚC khi mở kết quả GPU — chưa làm chặt chẽ bước này ở lần đo trước, ghi nhận áp dụng nghiêm túc hơn cho các lần đo sau (mục V).

### 9. Layer 3 — xác nhận lần 2 bằng nguồn nhãn ĐỘC LẬP HOÀN TOÀN với B0 (31/08, citation-match, n=78/800)

**Nguồn nhãn mới**: `citation_labels_{train,validation,test}.json`, build từ `data123/` — **citation_metadata do chính Team C (Generator) tự parse sẵn** từ answer thật trong `data/train.json` (KHÔNG qua B2/search như B0 — độc lập hoàn toàn về mặt cơ chế, không chỉ độc lập về nguồn dữ liệu). Chỉ 2.946/7.000 câu (42,1%) đủ rõ để dùng ("baseline_eligible" — phần còn lại citation mơ hồ/trích nhiều văn bản không tách được), và trong 800 câu giữ kín chỉ 78 câu có citation label dùng được.

| K | Trước (dense) | Sau (Layer 3) | Chênh lệch | 95% CI |
|---|---|---|---|---|
| 1 | 33,3% | 47,4% | **+14,1%** | [+2,6%, +25,6%] có ý nghĩa |
| 3 | 56,4% | 62,8% | +6,4% | [-2,6%, +16,7%] |
| 10 | 66,7% | 73,1% | +6,4% | [+0,0%, +12,8%] |
| 50 | 80,8% | 80,8% | 0,0% | đúng lý thuyết |
| MRR | 0,4603 | 0,5616 | +0,1013 | — |

**So 2 nguồn trên phần chồng lấp** (50/800 câu có cả B0 lẫn citation label): B0 cho Hit cao hơn citation 4-6 điểm% ở mọi K — chênh lệch nhỏ, có thể do B0 match được context thứ 2-3 ngoài đúng Khoản trích dẫn chính, không phải mâu thuẫn nghiêm trọng.

**QUYẾT ĐỊNH CUỐI**: 3 nguồn độc lập (7.000 câu cấp văn bản, 198 câu cấp Khoản/B0, 78 câu cấp Khoản/citation) đều cho xu hướng dương nhất quán, "không đổi" đúng lý thuyết tại K=50 ở cả 3 → **CHỐT giữ Layer 3 trong kiến trúc chính thức** — đây là bằng chứng đáng tin nhất từ đầu dự án.

### 10. Đảo ngược TOP_N (3→1) + mở rộng context sang cả Điều (31/08) — thay đổi lớn nhất trong ngày

**Nguyên nhân**: làm rõ với Generator (C) cách Generator THỰC SỰ dùng context — **C xác nhận Generator copy máy móc đoạn trích luật, KHÔNG dùng LLM để tự chọn giữa nhiều context**. Nghĩa là quyết định top-3 chốt ngày 13/08 (dựa trên nguyên tắc "thà dư hơn thiếu") thực chất VÔ DỤNG với cách Generator hoạt động — 3 context đưa qua, Generator không biết chọn cái nào, khả năng cao chỉ dùng/copy nhầm.

**Quyết định mới**: B4 chỉ trả **ĐÚNG 1 context/câu hỏi** (`TOP_N=1`). Để bù lại việc mất "lưới an toàn" của top-2/3 (từng đặt ra để giảm rủi ro chọn sai Khoản, hoặc answer cần ≥2 nguồn), B6 **mở rộng context trả cả Điều** (không chỉ đúng Khoản đã chọn) — đây cũng là lời giải trực tiếp cho giới hạn trần chunking đã phát hiện ở mục IV.1 (31,1% câu cần thông tin ngoài 1 Khoản).

**Đo Hit rate**: xem phương pháp và số liệu đầy đủ ở mục IV.5 — kết quả cho cả 2 cấu hình TOP_N đều xác nhận mở rộng sang cả Điều cải thiện Hit rate có ý nghĩa thống kê.

**Đã áp dụng**: `build_qa_packages_heldout.py` sửa theo `TOP_N=1` + mở rộng Điều, build lại `outputs/qa_packages_heldout.json` (800 record, verify: đúng 1 context/record). Gói này gửi cho C/A dùng để chạy oracle eval nội bộ — **không phải bài nộp final**.

---

## IV. Phát hiện & giới hạn quan trọng

### 1. Giới hạn trần của chunking cấp Khoản: ~31,1% câu cần thông tin ngoài đúng 1 Khoản

Đo bằng Smith-Waterman so điểm khớp giữa quote gốc với (a) đúng Khoản B0 chọn, (b) toàn bộ Điều chứa Khoản đó, n=508 (nhãn B0 `confidence≥0,6`):

| | Số câu | Tỷ lệ |
|---|---|---|
| Khoản hiện tại ĐỦ | 350/508 | 68,9% |
| Cần thông tin ngoài đúng 1 Khoản | 158/508 | **31,1%** |

**Đã dẫn tới quyết định cụ thể**: đây là 1 trong 2 lý do trực tiếp khiến B6 quyết định mở rộng context sang cả Điều (mục III.10), không còn là câu hỏi bỏ ngỏ.

**Caveat chưa giải quyết**: đo trên nhãn silver (B0); có thể một phần "ăn gian điểm" do boilerplate pháp lý lặp lại giữa các Khoản lân cận, chưa đọc tay kiểm chứng.

### 2. `select_span` (cắt Khoản xuống 1 đoạn nhỏ nhất) xoá mất đáp án 55,6% số lần

**Đo thật** (151 case Khoản đúng CÓ trong candidate, nhãn B0): chỉ giữ đúng đáp án gold ở 67/151 = 44,4% — **55,6% số lần cắt bỏ đúng phần chứa câu trả lời**.

**Quyết định**: bỏ hẳn `select_span`, B4 trả nguyên văn Khoản (nay là cả Điều, mục III.10) cho B6/Generator. Hàm vẫn giữ trong code, dự phòng.

### 3. Case đặc biệt: văn bản không tách được cấu trúc (`doc_fallback`)

15,4% văn bản không khớp cấu trúc Điều/Khoản chuẩn — context trả về có thể RẤT DÀI (1 case đo được 24.795 ký tự, không có `article`). Đúng thiết kế, chưa lý tưởng về súc tích — ghi nhận, chưa xử lý.

### 4. Error analysis Layer 3 (31/08) — phân loại nguyên nhân case thất bại, Layer 3 không phải lúc nào cũng có lợi

**Mục đích**: sau khi chốt giữ Layer 3 (mục III.9), cần biết trong số case vẫn sai, sai do đâu — do B1/B2 chưa từng tìm ra đúng candidate (không thể trách Layer 3), hay do chính Layer 3 xếp hạng sai (candidate đã có sẵn nhưng bị đẩy xuống)? Câu trả lời quyết định nên đầu tư sửa retrieval hay sửa reranker tiếp theo (mục VIII.1.2-3).

**Nguồn gold**: hợp (union) 2 nguồn nhãn độc lập cấp Khoản — B0 (`confidence≥0,6`) + citation-match (mục V, Tầng 2b) — chỉ giữ `unit_type="khoan"` ở cả 2. Tổng **n=226 câu** có gold (198 từ B0, 78 từ citation, có phần chồng lấp giữa 2 nguồn).

**Cách phân loại** (`pipeline/error_analysis_heldout.py`, chỉ xét case Hit@10 sau Layer 3 = SAI):
- **Loại A (retrieval miss)**: gold KHÔNG nằm trong top-50 candidate GỐC (trước khi đưa vào Layer 3, tức từ `search_units_hybrid`) — lỗi thuộc B1/B2, Layer 3 không thể cứu vì gold chưa từng có mặt trong tập để rerank.
- **Loại B (reranker miss)**: gold CÓ mặt trong top-50 gốc, nhưng bị Layer 3 xếp xuống ngoài top-10 sau khi rerank — lỗi thuộc chính cross-encoder.

**Kết quả**: Hit@10 sau Layer 3 = 172/226 = 76,1%. Trong 54 case thất bại:

| Loại | Số case | Tỷ lệ trong nhóm thất bại |
|---|---|---|
| A — retrieval miss (lỗi B1/B2) | 37 | 68,5% |
| B — reranker miss (lỗi Layer 3) | 17 | 31,5% |

**Đào sâu thêm nhóm B** (tính thêm từ dữ liệu case-level lưu trong `outputs/error_analysis_heldout.json`, không phải số in sẵn của script gốc): trong 17 case Loại B, **13 case (76,5%) là do chính Layer 3 xếp gold XUỐNG THẤP HƠN vị trí gốc** (không chỉ "không đủ để lọt top-10" — thật sự đi lùi so với dense). Trong 13 case đó, **4 case gold đã ở top-6 TRƯỚC rerank** (rank gốc lần lượt 1, 1, 3, 6), trong đó **2 case gold đã ở đúng rank 1** (gần như chắc chắn đúng) — vậy mà Layer 3 đẩy xuống rank 13 và rank 14, ra hẳn ngoài top-10 (câu `122961`: rank 1→13; câu `88873`: rank 1→14).

**Ý nghĩa 2 chiều**:
1. Nguồn lỗi lớn nhất VẪN là retrieval (68,5%), không phải reranking — sửa B1/B2 (đặc biệt nhầm lẫn giữa văn bản cùng chủ đề — quan sát được là nguyên nhân phổ biến của Loại A: dense embedding cho điểm cao nhưng chọn sai văn bản cụ thể) có tiềm năng ảnh hưởng lớn hơn tinh chỉnh Layer 3 (mục VIII.1.3).
2. Nhưng Layer 3 không "vô hại" — có 1 nhóm nhỏ mà nó chủ động phá hỏng case gần như chắc chắn đúng. Kết luận tổng thể "giữ Layer 3" (III.9) vẫn đúng vì lợi ích ròng dương và có ý nghĩa thống kê, nhưng đây là bằng chứng cụ thể cho việc CHƯA nên tin tuyệt đối điểm cross-encoder — cân nhắc cơ chế an toàn (ví dụ không rerank nếu dense đã xếp hạng 1-2 với điểm cách biệt lớn) — xem mục VIII.1.2.

### 5. Context expansion (31/08) — đo đầy đủ, cơ sở cho quyết định III.10

**Mục đích**: sau khi biết TOP_N sẽ giảm còn 1 (vì Generator copy máy móc, mục III.10), cần biết nếu chỉ được đưa 1 context, nên đưa ĐÚNG Khoản đã chọn, hay đưa CẢ ĐIỀU chứa Khoản đó (rộng hơn, có khả năng chứa thêm thông tin mà chunking cấp Khoản đã bỏ lỡ — đúng vấn đề nêu ở mục IV.1, 31,1%)?

**Cách đo** (`pipeline/eval_context_expansion_heldout.py`): lấy CÙNG 1 danh sách Khoản đã chọn (top-N sau Layer 3, qua `select_top_n_khoan`), dựng 2 phiên bản context từ đúng danh sách đó — không chạy lại retrieval, nên đây là phép đo tách biệt hoàn toàn khỏi chất lượng Layer 3, chỉ đo tác động của cách B6 "cắt gọn":
- **(a) Hiện tại**: nối text của từng Khoản trong danh sách.
- **(b) Mở rộng**: với mỗi Khoản, thay bằng text của CẢ ĐIỀU chứa nó (loại trùng nếu 2 Khoản top-N cùng 1 Điều).

**Hit rate** = tỷ lệ câu mà text của gold Khoản (từ union gold, n=226, cùng nguồn mục IV.4) xuất hiện làm chuỗi con TRONG context đã dựng — khác Hit@K ở chỗ chỉ cần kiểm tra "có nằm trong văn bản context hay không", không cần khớp đúng `unit_id`.

**Kết quả — đo ở 2 cấu hình TOP_N** (đo cấu hình 3 trước, rồi đo lại ở cấu hình 1 sau khi đảo ngược quyết định):

| TOP_N | (a) chỉ Khoản | (b) cả Điều | Chênh lệch Hit rate | 95% CI (bootstrap 2.000 lần) | Độ dài TB ký tự (a→b) |
|---|---|---|---|---|---|
| 3 (cấu hình cũ, đã bỏ) | 60,2% | 66,4% | +6,2 điểm% | có ý nghĩa | 2.241 → 7.015 |
| **1 (đang dùng)** | 46,5% | **52,2%** | +5,8 điểm% | có ý nghĩa | 734 → 2.322 |

**Logic quyết định đặt sẵn trong script TRƯỚC khi đọc kết quả** (đúng bài học pre-registration, mục V): nếu chênh lệch Hit rate dương VÀ cận dưới khoảng tin cậy 95% > 0 → khuyến nghị áp dụng (b) mở rộng; ngược lại giữ (a). Cả 2 cấu hình TOP_N đều rơi vào trường hợp khuyến nghị áp dụng.

**Đánh đổi**: mở rộng sang cả Điều tăng Hit rate có ý nghĩa thống kê ở cả 2 cấu hình, nhưng cũng tăng độ dài context 3-3,1 lần. Với TOP_N=1, độ dài tuyệt đối sau mở rộng (2.322 ký tự) vẫn ngắn hơn nhiều so với context TOP_N=3 kể cả KHÔNG mở rộng (2.241 ký tự chỉ Khoản, càng ngắn hơn nhiều so với 7.015 ký tự nếu vừa top-3 vừa mở rộng Điều) — nên việc giảm TOP_N xuống 1 vừa đúng theo cách Generator dùng context, vừa "nhường chỗ" cho việc mở rộng Điều mà tổng độ dài/câu vẫn trong tầm kiểm soát. **Đã áp dụng** làm mặc định trong `build_qa_packages_heldout.py`.

---

## V. Phương pháp đo lường & độ tin cậy dữ liệu

### Vì sao cần nhiều nguồn đo, không chỉ 1

Đề thi chỉ cấp `câu hỏi` + `câu trả lời mẫu`, không cho biết trích từ văn bản/Khoản nào — phải tự tạo nguồn đo. Hiện có **3 nguồn độc lập** (trước đây chỉ 2):

### Tầng 1 — `data_retrieve`: nguồn đo sạch, độc lập, cấp văn bản

Bộ dữ liệu hạng mục thi khác, chung corpus, có sẵn `context_id` đúng — không qua thuật toán của nhóm. Quy mô: 7.500 câu. Dùng đo mọi ablation B2 (mục III.1-4, III.7).

### Tầng 2a — B0 (tự gán nhãn trên câu hỏi QA thật): cấp Khoản, có bias

**Cơ chế**: tách answer → trích lõi trích dẫn → tìm candidate qua `search_units()` (B2) → lọc rẻ → so khớp Smith-Waterman, `confidence` chuẩn hoá [0,1], ngưỡng drop 0,3. Quy mô: ~2.000 câu có nhãn. **Bias cố hữu**: dùng chính B2 để tìm candidate — nếu B2 sai từ đầu, B0 không tạo được nhãn cho câu đó, không dùng để đo lại chính B2.

### Tầng 2b — Citation-match (MỚI, 31/08): cấp Khoản, ĐỘC LẬP HOÀN TOÀN với B2/B0

Parse trực tiếp từ `citation_metadata` do Team C tự làm (đọc answer, trích số Điều/Khoản được nhắc tới) — không qua bất kỳ bước search/retrieval nào của B, nên độc lập cả về nguồn LẪN cơ chế (khác B0 chỉ độc lập về nguồn dữ liệu nhưng vẫn dùng chung cơ chế search). Coverage thấp hơn nhiều: 2.946/7.000 (42,1%) toàn tập, 78/800 trong tập giữ kín — vì `data123` chỉ parse chắc chắn được câu có trích dẫn rõ ràng, đơn giản.

### Tầng 3 — Kiểm chứng chéo độ tin của nhãn B0

Cross-check khách quan (29 cặp câu trùng `data`/`data_retrieve`): **B0 đúng 13/21 = 61,9%**. Đọc tay 30 case: ~70-80% đúng chủ đề ở mọi mức confidence. Kết luận: độ tin B0 ở mức trung bình-khá, đủ cho kết luận TƯƠNG ĐỐI, không dùng số tuyệt đối làm con số cuối cùng.

### Tầng 4 — B7 oracle (không phải việc của B, thuộc A)

Đưa context ĐÚNG vào Generator, đo METEOR/ROUGE-L thật. `outputs/qa_packages_heldout.json` (800 câu, mục III.10) chính là input B đã chuẩn bị sẵn để A/C chạy phép đo này — bản thân phép đo/kết quả chưa có (thuộc trách nhiệm A).

### Công thức các chỉ số đo (`src/b3_eval_recall/ir_metrics.py`)

- **Hit@K** = tìm được ít nhất 1 gold trong top-K.
- **Recall@K micro/strict**, **MRR@K**, **nDCG@K** — xem định nghĩa đầy đủ trong code, dùng khi cần phân tích sâu hơn Hit@K.
- **Coverage** = Hit@K tại K lớn nhất đã lấy.

### Phát hiện phương pháp luận: "nhìn tập test nhiều lần" (data snooping) và cách khắc phục

`data_retrieve/warmup.json` và nhãn B0 dev-sample (n=393) đã bị dùng LẶP LẠI để ra nhiều quyết định thiết kế — số TUYỆT ĐỐI có thể lạc quan hơn thực tế, so sánh TƯƠNG ĐỐI vẫn hợp lệ. **Khắc phục**: tạo tập GIỮ KÍN 800 câu (`outputs/b0_labels_train_heldout.json`), dùng đúng 1 lần cho Layer 3 (mục III.8-9) — không dùng để tune bất cứ gì trước đó.

**Bài học tiếp theo (pre-registration)**: nên chốt ngưỡng quyết định TRƯỚC khi mở kết quả GPU — áp dụng nghiêm túc hơn cho các lần đo tương lai, đặc biệt các thử nghiệm fine-tune ở mục VIII.

---

## VI. Trạng thái hiện tại (bảng done/pending)

| Module | Trạng thái | Việc còn lại |
|---|---|---|
| B0 (auto-label) | ✅ Ổn định, độ tin đã kiểm chứng khách quan (61,9%) — nhưng coverage thấp trên tập giữ kín (198/800 = 24,8% đạt confidence≥0,6) | Cân nhắc hạ `B0_CONFIDENCE_TRUST` |
| B1 (chunking) | ✅ Xong, full corpus, 3/4 bug đã sửa | Bug danh sách con đánh số trùng — hoãn |
| B2 (retrieval, Layer 1+1.5+2+Union) | ✅ Xong, đã wire vào `search_units_hybrid` | — |
| B2 Layer 3 (reranker) | ✅ CHỐT GIỮ — xác nhận bằng 3 nguồn nhãn độc lập (mục III.7-9) | Chưa formalize batch pipeline cho `public-official.json`; đã biết rủi ro làm tệ hơn ở 1 nhóm case nhỏ (IV.4) |
| B3 (đo lường) | ✅ Bộ metric đầy đủ; đã đo trần chunking (31,1%) | — |
| B4 (chọn context) | ✅ **TOP_N đảo ngược 3→1** (31/08, theo cách Generator dùng context thật) | — |
| B5 (ngân sách) | ✅ ĐÃ CHỐT — 4 tỷ toàn hệ thống, B được cấp 2,3 tỷ, đang dùng 1,136 tỷ (49,4%) | — |
| B6 (đóng gói) | ✅ Schema xác nhận với C 31/08 (`article`/`clause` tách rời, thêm `document_number`); context mở rộng cả Điều | Field `document` (`{"name","link"}`) vẫn còn treo, chưa Generator xác nhận |
| B7 (oracle) | Không phải việc của B (thuộc A) | B đã gửi `qa_packages_heldout.json` (800 câu) để A/C tự chạy |
| Citation labels (nguồn nhãn mới) | ✅ Đã build (`outputs/citation_labels_*.json`), dùng xác nhận Layer 3 lần 2 | Coverage thấp (42,1% toàn tập, 78/800 tập giữ kín) — có thể mở rộng nếu cần |
| Tập giữ kín (held-out) | ✅ 800 câu, đã dùng cho toàn bộ đánh giá Layer 3 + context expansion | — |
| **Chạy `public-official.json`** | ✅ **ĐÃ CHẠY (03/09/2026)** — 3 bản nộp LB: `v1_top3`=0,4506 (tốt nhất), `v2_top1`=0,4430, `v3_top1_raw` chờ điểm | Baseline coi như đóng |

---

## VII. Rủi ro & câu hỏi mở cần các thành viên khác

- ~~[RỦI RO CAO NHẤT] `data/public-official.json` chưa chạy qua pipeline~~ — **ĐÃ GIẢI QUYẾT (03/09/2026)**: chạy xong full pipeline, 3 bản nộp lên leaderboard (`v1_top3`=0,4506 tốt nhất). Baseline coi như đóng.
- **Generator (C)** cần xác nhận cấu trúc field `document` trong B6 (hiện dùng tạm `{"name","link"}`) — mọi field khác đã xác nhận 31/08.
- **Trưởng nhóm** cần xác nhận tập dev nội bộ 150 câu, và harness chấm điểm chung đã có chưa.
- **Layer 3 có thể làm tệ hơn** ở 1 nhóm case cụ thể (mục IV.4, 76,5% của case Loại B) — chưa có cơ chế giảm thiểu (ví dụ kết hợp điểm dense+rerank thay vì tin tuyệt đối cross-encoder).
- **Citation labels coverage thấp** (42,1% toàn tập, chỉ 78/800 tập giữ kín) — kết luận Layer 3 dựa 1 phần trên mẫu khá nhỏ ở nguồn nhãn "sạch nhất".
- **`EXPERIMENT_LOG.md` chưa được cập nhật** cho toàn bộ khối lượng việc ngày 31/08 (ngân sách, schema B6, TOP_N, citation labels, error analysis, context expansion) — nên backfill sớm để không mất dấu vết quyết định.
- **Bug B1 danh sách con đánh số trùng** (7.632 `khoan_id` trùng/432.473): còn tồn tại, hoãn.
- **Con số 31,1%** (trần chunking) đo trên nhãn silver — dù đã dẫn tới quyết định cụ thể (III.10), vẫn nên đọc tay kiểm chứng thêm nếu có thời gian.

---

## VIII. Kế hoạch tiếp theo

### 1. Việc kỹ thuật ưu tiên ngay

1. **[RỦI RO CAO — làm trước tiên] Chạy full pipeline cho `public-official.json`** (1.000 câu ĐÍCH nộp bài): sinh candidate → Kaggle Layer 3 → `qa_packages_public.json`, nhớ dùng đúng cấu hình mới nhất (TOP_N=1 + mở rộng Điều). Đây là việc duy nhất thực sự chặn khả năng nộp bài.
2. **Điều tra hiện tượng Layer 3 làm tệ hơn dense gốc** (mục IV.4, 76,5% case Loại B) — cân nhắc cơ chế kết hợp điểm dense+rerank (ví dụ chỉ áp dụng rerank khi chênh lệch điểm đủ lớn, hoặc không rerank nếu dense đã rất tự tin ở rank 1-2) thay vì tin tuyệt đối cross-encoder score.
3. **Cải thiện B1/B2 retrieval** — 68,5% case thất bại là retrieval miss hoàn toàn (không phải lỗi Layer 3). Nguyên nhân quan sát được: nhầm lẫn giữa nhiều văn bản luật cùng chủ đề. Hướng khả dĩ: xem mục 3 (fine-tune bi-encoder) bên dưới.
4. (Tuỳ chọn, không cấp bách) Mở rộng cỡ mẫu đánh giá Layer 3 bằng citation label từ 78 → 506 câu "fresh chưa tune" — cần sinh thêm candidate cho ~428 câu + chạy lại Kaggle.
5. Backfill `EXPERIMENT_LOG.md` cho khối lượng việc 31/08 chưa ghi.

### 2. Việc phối hợp (không phải kỹ thuật)

- Xác nhận cấu trúc `document` (B6) với Generator (C).
- Xác nhận tập dev nội bộ 150 câu + harness chấm điểm chung với Trưởng nhóm.

### 3. Đầu tư dài hạn hơn — kế hoạch fine-tune model

Ngân sách còn dư ~1,164 tỷ tham số (mục II) — chỉ làm nếu ablation chứng minh lợi ích thật.

**3.1 Fine-tune bi-encoder (Layer 2) cho domain pháp lý VN** — đáng ưu tiên hơn trước, vì error analysis (IV.4) cho thấy 68,5% thất bại là do retrieval nhầm lẫn giữa các văn bản cùng chủ đề — đúng loại lỗi mà fine-tune domain-specific có thể cải thiện. Input: cặp (câu hỏi, Khoản đúng) từ `data_retrieve` + nhãn B0 + citation labels (nguồn sạch hơn). Phương pháp: contrastive learning, cân nhắc semi-hard negative mining. Đánh giá: trên tập giữ kín, không dùng lại `data_retrieve`/`warmup` đã "nhiễm".

**3.2 Model cho span selection — độ cấp thiết giảm sau quyết định III.10.** Vì context giờ mặc định trả cả Điều (không chỉ Khoản hẹp), nhu cầu "cắt gọn xuống đúng đoạn" giảm bớt so với trước — cân nhắc lại có còn cần thiết không trước khi đầu tư.

**3.3 Fine-tune reranker (Layer 3)** — có thể giúp giảm rủi ro "làm tệ hơn" (IV.4) nếu train trên đúng domain thay vì dùng model gốc tổng quát. Làm sau 3.1.

### 4. Thứ tự ưu tiên tổng hợp

1. Chạy `public-official.json` (VIII.1.1) — chặn khả năng nộp bài, làm trước tiên.
2. 2 việc phối hợp (VIII.2) — song song, không phụ thuộc kỹ thuật.
3. Điều tra rủi ro Layer 3 + cải thiện B1/B2 retrieval (VIII.1.2-3) — ảnh hưởng trực tiếp chất lượng trước khi đầu tư fine-tune.
4. Fine-tune bi-encoder (3.1) — ưu tiên trong nhóm fine-tune vì giải đúng nguyên nhân lỗi lớn nhất đã đo được.
5. Fine-tune reranker (3.3), đánh giá lại nhu cầu span selection (3.2) — sau cùng.

---

## Nguồn tham khảo (mô hình, kỹ thuật — cho mục VIII.3)

- [bkai-foundation-models/vietnamese-bi-encoder](https://huggingface.co/bkai-foundation-models/vietnamese-bi-encoder) — bi-encoder tiếng Việt, fine-tune sẵn trên Legal Text Retrieval Zalo 2021.
- [BAAI/bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3) — reranker đang dùng cho Layer 3.
- [Optimizing Legal Document Retrieval in Vietnamese with Semi-Hard Negative Mining](https://arxiv.org/html/2507.14619) — kỹ thuật fine-tune bi-encoder/cross-encoder pháp lý tiếng Việt.
- [Multi-stage Information Retrieval for Vietnamese Legal Texts](https://arxiv.org/pdf/2209.14494) — kiến trúc multi-stage cùng hướng với B.
