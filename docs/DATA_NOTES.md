# Data Audit — Corpus vừa được BTC cấp (06/08/2026)

> Khảo sát toàn bộ `train.json`, `warmup.json`, `public-official.json`, và `selected-contexts/` (8,532 file `context_*.json`). Đây là lần đầu có corpus thật — trước đó bị chặn (xem `B_technical_handoff.md` mục 7).

## 1. Phát hiện quan trọng nhất: corpus là tập ĐÃ ĐƯỢC LỌC SẴN, không phải toàn bộ kho luật VN

- Tổng số câu hỏi: `train.json` 7,000 + `warmup.json` 500 + `public-official.json` 1,000 = 8,500.
- Tổng số văn bản trong corpus: **8,532** — gần khớp tuyệt đối với tổng số câu hỏi.
- Đây gần như chắc chắn **không phải trùng hợp**: BTC (đúng như tên `selected-contexts`) đã tự lọc sẵn văn bản liên quan cho toàn bộ câu hỏi train+warmup+public(+có thể cả private), thay vì để đội tự retrieve trên toàn bộ kho luật VN (hàng triệu văn bản).

**Hệ quả trực tiếp cho thiết kế B2 (BM25)**: không gian tìm kiếm bị chặn trên (upper-bounded) ở **8,532 văn bản**, không phải corpus luật mở. Đây là tin tốt về mặt kỹ thuật (bài toán retrieval dễ hơn tưởng tượng nhiều — closed-set thay vì open-domain), nhưng **cần verify lại**: không có `context_id` gắn trực tiếp vào từng câu hỏi trong `train.json`/`warmup.json`/`public-official.json` (mỗi mẫu chỉ có `question` + `answer`), nên giả thuyết "mỗi câu hỏi ứng với đúng 1 (hoặc vài) văn bản trong 8,532 này" **chưa được chứng minh bằng liên kết trực tiếp** — chỉ suy ra từ con số trùng khớp. B0 (gán nhãn) chạy xong sẽ xác nhận chắc chắn tỉ lệ match thực tế.

## 2. Cấu trúc dữ liệu thực đo

| File | Kiểu | Số lượng | Keys mỗi item |
|---|---|---|---|
| `train.json` | dict `{id: {...}}` | 7,000 | `question`, `answer` |
| `warmup.json` | dict `{id: {...}}` | 500 | `question`, `answer` |
| `public-official.json` | dict `{id: {...}}` | 1,000 | `question`, `answer` (= `null` — đây là tập cần dự đoán, không có đáp án) |
| `selected-contexts/selected-contexts/context_*.json` | 1 file/văn bản | 8,532 | `id`, `name`, `link`, `passage` |

Lưu ý cấu trúc thư mục gốc bị lồng đôi: `data/selected-contexts/selected-contexts/context_*.json` (chắc do giải nén zip). Đã thêm symlink `data/corpus` trỏ thẳng vào đây để code không phải quan tâm việc lồng thư mục — xem `data/README.md`.

## 3. Chất lượng corpus — các vấn đề cần xử lý ở bước parse (Phase A / B1)

Đo trên toàn bộ 8,532 file (không phải mẫu):

- **Độ dài `passage`**: trung bình 41,455 ký tự; p50 = 23,112; p90 = 90,141; p99 = 285,366; max = 5,983,358 ký tự. Phân phối lệch phải rất mạnh — một số văn bản là cả bộ luật/quy chuẩn khổng lồ. **B1 (parser Điều/Khoản) phải xử lý được văn bản rất dài, không giả định "1 văn bản = vài Điều".**
- **Loại văn bản** (suy từ tiền tố tên file, ~85% nhận diện được): Thông tư 32.2%, Quyết định 31.1%, Nghị định 12.9%, Nghị quyết 3.4%, Luật 2.9%, còn lại (Chỉ thị, Thông báo, Kế hoạch, Pháp lệnh, Công điện, Bộ luật, Hiến pháp) < 1% mỗi loại. **15% không nhận diện được loại từ tên** — phần lớn trùng với nhóm `name = None` bên dưới, hoặc là văn bản Đảng (Điều lệ, Quy định, Hướng dẫn nội bộ — không theo mẫu số hiệu nhà nước chuẩn).
- **20 văn bản `passage` rỗng hoàn toàn** (0 ký tự) — đều là các trang TCVN/QCVN hoặc `/van-ban/` bị crawl lỗi (link vẫn có, nội dung trống). Cần loại khỏi index hoặc flag riêng, không thể parse Điều/Khoản từ rỗng.
- **1,125 văn bản (13.2%) có `name = None`** (367 thuộc `/tcvn/`, 187 thuộc `/van-ban/`, 571 link dạng khác) — nhưng **`link` luôn có giá trị** và chứa tên văn bản dạng slug (vd `.../Nghi-dinh-16-2023-ND-CP-to-chuc-...aspx`). → **Fallback bắt buộc trong `io_utils`**: khi `name is None`, suy ra tên từ slug cuối `link`.
- **1 văn bản (`context_68843`, QCVN 118:2018) dài bất thường (~6M ký tự)**, chứa lặp lại nhiều lần cụm "Bạn phải đăng nhập hoặc đăng ký Thành Viên TVPL Pro..." — dấu hiệu nội dung sau paywall bị crawl lặp/hỏng, không phải toàn văn thật. Chỉ 5/8,532 văn bản (0.06%) chứa cụm paywall này — hiếm, không cần xử lý đại trà, nhưng **cần 1 bộ lọc câu paywall trong bước làm sạch text** phòng khi gặp lại, và nên loại các văn bản dạng này khỏi B0 alignment (matched_span sẽ vô nghĩa).
- **Không có `context_id` trùng lặp, không có mismatch giữa tên file và field `id`** — phần định danh sạch, đáng tin cậy.

## 4. Việc này mở khoá gì trong handoff

Theo `B_technical_handoff.md` mục 7, corpus là thứ **chặn toàn bộ B0, B1 (Phase A), và mọi thứ downstream**. Giờ đã có → có thể bắt đầu:

1. B1 — parser Điều/Khoản (ưu tiên #2 trong mục 8 của handoff), nhưng cần thêm bước làm sạch (loại 20 văn bản rỗng, fallback tên từ link, lọc câu paywall) trước khi parse.
2. B0 — gán nhãn tự động (spec đã chốt, xem `src/b0_autolabel/README.md`).
3. Đo lại số liệu "~60% token trích nguyên văn" — số quan trọng nhất còn treo trong handoff mục 3 — giờ có thể verify thật bằng B0 thay vì đoán.

## 5. Câu hỏi cần bạn xác nhận thêm (không tự quyết được từ data)

- Có nên loại hẳn 20 văn bản `passage` rỗng khỏi index BM25 luôn, hay giữ lại + gắn cờ để không vô tình bỏ sót nếu BTC sửa lại data sau? → đề xuất: loại khỏi index, log riêng ra file để theo dõi.
- File `private-official.json` (theo `DSC2026_Task2_LegalQA_Data_Overview.md`) sẽ dùng corpus nào — cùng 8,532 file này hay 1 batch corpus mới? Nếu corpus mở rộng thêm ở Private Test, giả thuyết "closed-set 8,532" ở mục 1 sẽ cần review lại lúc đó.

## 6. Số liệu tham chiếu từ tài liệu phân công gốc (KHÔNG phải tự đo — trích dẫn từ "DSC2026 — Phân công nhóm Task 2", Trưởng nhóm, 02/08/2026, đo trên 120 mẫu thật)

Dùng để thiết kế B4 (chọn đoạn trích) và B6 (đóng gói context) sau này — quan hệ giữa tỉ lệ độ dài (t/r = độ dài trả lời hệ thống / độ dài tham chiếu) và điểm METEOR, giả định hệ thống bắt đúng toàn bộ nội dung tham chiếu rồi thêm/bớt:

| Tỉ lệ t/r | METEOR |
|---|---|
| 0.40 | 0.378 |
| 0.60 | 0.568 |
| 0.80 | 0.772 |
| 1.00 | 1.000 |
| 1.25 | 0.944 |
| 1.50 | 0.873 |
| 2.00 | 0.752 |
| 3.00 | 0.611 |

Kịch bản bổ sung: chép y hệt = 1.000; đúng nội dung nhưng viết ngắn gọn (1 câu kết luận) = **0.214** (thua nặng dù "đúng"); dài gấp 2.5 lần với phần thừa là rác hoàn toàn = 0.862 (vẫn khá); đúng đủ từ nhưng xáo thứ tự = 0.500 (hình phạt phân mảnh tối đa, mất nửa điểm).

**Kết luận vận hành (nguyên văn từ tài liệu gốc, áp dụng cho thiết kế B4/B6)**: viết thừa rẻ hơn viết thiếu 3-4 lần (thiếu 20% mất 0.228 điểm, thừa 25% chỉ mất 0.056) → **thiên về lấy dư hơn lấy thiếu** khi B4 chọn đoạn trích, B6 đóng gói context. Trả lời ngắn gọn dù đúng là chiến lược thua.

### Bảng ngân sách tham số (trần chung < 4.0B, tính cả embedding)

| Thành phần | Chủ sở hữu | Hạn mức | Ghi chú |
|---|---|---|---|
| BM25/TF-IDF | B | 0 | Sparse thuần, không tham số |
| Bi-encoder (nếu dùng) | B | ≤ 0.7B | Chỉ dùng nếu B5 chứng minh xứng đáng |
| Reranker (nếu dùng) | B | ≤ 0.6B | |
| Generator | C | ≤ 3.3B | Nới lên ~3.9B nếu B bỏ hẳn dense |

Đếm đúng bằng `sum(p.numel() for p in model.parameters())` trên model đã load — không tin tên model (model gắn nhãn "4B" thường là 4.0xB, đã vượt ngưỡng).

### Lưu ý vận hành nhóm quan trọng cho B

- **Một harness/dev-set/script chấm DUY NHẤT do Trưởng nhóm sở hữu và đóng băng** — `src/common/scoring.py` là công cụ cá nhân để tự kiểm tra nhanh, KHÔNG phải nguồn số liệu chính thức để báo cáo.
- **Tập dev nội bộ 150 câu tách từ `train.json`, đóng băng, không ai được đụng tới** (thuộc T2, Trưởng nhóm phụ trách) — **cần hỏi đã chốt danh sách 150 câu này chưa**, để tránh vô tình dùng chúng khi B tự thử nghiệm trên `train.json`.
- Oracle để tách phụ thuộc: B luôn test với 1 generator CỐ ĐỊNH (không đợi C), C luôn dev với gold context (không đợi B) — không ai chờ ai.
