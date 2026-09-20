# Báo Cáo Nhóm — Lab 7: Embedding & Vector Store

**Nhóm:** G41
**Thành viên:** Trần Tuấn Cường, Trần Đình Hinh, Lê Như Ý
**Ngày:** 20/09/2026

> **Nộp 1 bản / nhóm.** Phần cá nhân (hướng tiếp cận, kết quả riêng, dự đoán…) mỗi thành viên nộp riêng trong `REPORT_CANHAN.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần nhóm: 40** = Lựa chọn tài liệu (10) + Thiết kế chiến lược (15) + Chất lượng truy xuất (10) + Thuyết trình (5).

---

## 1. Lựa chọn tài liệu (Document Set Quality) — Nhóm (10 điểm)

### Chủ đề (Domain) & Lý Do Chọn

**Chủ đề:** Chính sách Bảo hành tại các Hệ thống Bán lẻ Điện máy và Thương mại điện tử tại Việt Nam

**Tại sao nhóm chọn chủ đề này?**
> Nhóm chọn chủ đề này vì các quy định bảo hành và đổi trả tại các hệ thống bán lẻ (Thế Giới Di Động, FPT Shop, CellphoneS, Điện Máy Xanh, TTG Shop) và sàn TMĐT (Shopee) có tính pháp lý cao, nhiều điều khoản ràng buộc với các mốc thời gian (15 ngày, 30 ngày, 48 giờ) và tỷ lệ phí cụ thể. Đây là miền dữ liệu thực tế tuyệt vời để đánh giá năng lực trích xuất thông tin chuẩn xác và đối chiếu hiệu quả giữa các kỹ thuật chunking khác nhau.

### Danh sách tài liệu (Data Inventory)

| # | Tên tài liệu | Nguồn (Source URL) | Ngày lấy / Phiên bản | Số ký tự | Metadata đã gán |
|---|--------------|--------------------|----------------------|----------|-----------------|
| 1 | `shopee-warranty-policy.md` | https://help.shopee.vn/portal/4/article/79046 | 2026-09-20 / 2024.1 | 2,885 | `audience: buyer`, `category: warranty`, `platform: Shopee` |
| 2 | `ttgshop-warranty-policy.md` | https://ttgshop.vn/quy-dinh-bao-hanh | 2026-09-20 / 2024.1 | 2,545 | `audience: buyer`, `category: warranty-returns`, `platform: TTGShop` |
| 3 | `thegioididong-warranty-policy.md` | https://www.thegioididong.com/chinh-sach-bao-hanh-san-pham | 2026-09-20 / 2024.1 | 2,903 | `audience: buyer`, `category: warranty-returns`, `platform: TheGioiDiDong` |
| 4 | `cellphones-warranty-policy.md` | https://cellphones.com.vn/chinh-sach-bao-hanh | 2026-09-20 / 2024.1 | 2,868 | `audience: buyer`, `category: warranty-returns`, `platform: CellphoneS` |
| 5 | `dienmayxanh-warranty-policy.md` | https://www.dienmayxanh.com/chinh-sach-bao-hanh-san-pham | 2026-09-20 / 2024.1 | 2,793 | `audience: buyer`, `category: warranty-returns`, `platform: DienMayXanh` |
| 6 | `fptshop-return-policy.md` | https://fptshop.com.vn/ho-tro/chinh-sach-doi-san-pham | 2026-09-20 / 2024.1 | 2,811 | `audience: buyer`, `category: returns-exchange`, `platform: FPTShop` |
| 7 | `seller-warranty-policy.md` | https://banhang.shopee.vn/edu/article/1815 | 2026-09-20 / 2024.2 | 2,748 | `audience: seller`, `category: seller-dispute`, `platform: Shopee` |

**Danh sách kiểm tra quản trị dữ liệu (Data governance checklist):**
- [x] Tập tài liệu (Corpus) chỉ chứa nguồn công khai/được phép dùng và không chứa dữ liệu cá nhân, thông tin đăng nhập hoặc tài liệu nội bộ.
- [x] Mỗi tài liệu có `source_url`, `retrieved_at`, `document_version` (hoặc ngày hiệu lực) trong metadata và đã đồng bộ với `data/ecommerce/sources.csv`.

### Cấu trúc Metadata (Metadata Schema)

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho truy xuất (retrieval)? |
|----------------|------|---------------|-------------------------------------------|
| `audience` | `str` | `"buyer"`, `"seller"` | Phân định đối tượng áp dụng chính sách, phục vụ lọc chính xác theo yêu cầu biến thể K4-L3B. |
| `category` | `str` | `"warranty"`, `"warranty-returns"`, `"seller-dispute"` | Thu hẹp phạm vi tìm kiếm theo nghiệp vụ cụ thể (bảo hành hoặc đổi trả). |
| `platform` | `str` | `"Shopee"`, `"TheGioiDiDong"`, `"CellphoneS"` | Lọc chính sách theo nhà phân phối hoặc hệ thống bán lẻ người dùng quan tâm. |
| `source_url` | `str` | `"https://ttgshop.vn/quy-dinh-bao-hanh"` | Hỗ trợ truy vết nguồn gốc (citation) và kiểm chứng tính xác thực của câu trả lời. |
| `retrieved_at` | `str` | `"2026-09-20"` | Đánh giá độ mới của chính sách và quản lý vòng đời tài liệu khi có điều khoản mới. |

---

## 2. Thiết kế chiến lược (Strategy Design) — Nhóm (15 điểm)

> Mỗi thành viên thử **một chiến lược khác nhau** trên cùng bộ tài liệu; nhóm tổng hợp và so sánh ở đây.

### Phân tích đường cơ sở (Baseline Analysis)

Chạy `ChunkingStrategyComparator().compare()` trên 3 tài liệu tiêu biểu trong kho dữ liệu:

| Tài liệu | Chiến lược (Strategy) | Số lượng Chunk | Độ dài trung bình | Giữ được ngữ cảnh không? |
|-----------|----------|:-------------:|:------------:|-------------------|
| `thegioididong-warranty-policy.md` | FixedSizeChunker (`fixed_size`, 300) | 11 | 291.18 | Không trọn vẹn: Cắt ngang điều khoản tại mốc 300 ký tự, ngắt câu giữa chừng. |
| `thegioididong-warranty-policy.md` | SentenceChunker (`by_sentences`, 3) | 8 | 360.62 | Khá tốt: Giữ trọn vẹn từng câu quy định, nhưng câu ghép dài làm chunk vượt ngưỡng mong muốn. |
| `thegioididong-warranty-policy.md` | RecursiveChunker (`recursive`, 300) | 15 | 191.87 | Rất tốt: Ưu tiên ngắt theo đoạn `\n\n` rồi tới dòng `\n`, giữ nguyên tiêu đề mục và nội dung. |
| `ttgshop-warranty-policy.md` | FixedSizeChunker (`fixed_size`, 300) | 10 | 281.50 | Kém: Cắt ngang số ngày bảo hành (07 ngày) và quyền lợi mượn thiết bị ra 2 chunk khác nhau. |
| `ttgshop-warranty-policy.md` | SentenceChunker (`by_sentences`, 3) | 8 | 316.62 | Tốt: Gom 3 câu liên tiếp giữ nguyên điều kiện bảo hành tem và số serial. |
| `ttgshop-warranty-policy.md` | RecursiveChunker (`recursive`, 300) | 13 | 194.31 | Rất tốt: Phân tách rõ ràng từng điều khoản đổi trả và điều kiện từ chối bảo hành. |
| `cellphones-warranty-policy.md` | FixedSizeChunker (`fixed_size`, 300) | 11 | 288.00 | Không tốt: Cắt đôi bảng tỷ lệ phí đổi trả máy mới 20% và máy cũ 15%. |
| `cellphones-warranty-policy.md` | SentenceChunker (`by_sentences`, 3) | 9 | 317.00 | Tốt: Bảo toàn câu quy định mức phí nhập lại theo giá xuất hóa đơn. |
| `cellphones-warranty-policy.md` | RecursiveChunker (`recursive`, 300) | 15 | 189.67 | Xuất sắc: Giữ cấu trúc tiêu đề cấp `#` và danh sách gạch đầu dòng tính phí. |

### Chiến lược của từng thành viên

**Thành viên 1 — [Trần Tuấn Cường]**
- **Loại chiến lược:** `RecursiveChunker` (chunk_size=300, chunk_overlap=30, separators=["\n\n", "\n", ". ", " ", ""])
- **Mô tả & lý do chọn cho chủ đề này:** Tài liệu chính sách bảo hành thương mại điện tử có cấu trúc Markdown với nhiều cấp bậc (tiêu đề `#`, `##`, danh sách gạch đầu dòng `-`). `RecursiveChunker` ưu tiên chia theo các ranh giới tự nhiên từ đoạn văn xuống dòng rồi mới tới câu và từ, giúp bảo toàn trọn vẹn ngữ cảnh tiêu đề cùng điều khoản quy định mà không bị đứt câu ngẫu nhiên.
- **Code snippet:**
```python
class RecursiveChunker:
    def __init__(self, chunk_size: int = 500, chunk_overlap: int = 50, separators: list[str] | None = None) -> None:
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.separators = separators or ["\n\n", "\n", ". ", " ", ""]

    def chunk(self, text: str) -> list[str]:
        if not text:
            return []
        if len(text) <= self.chunk_size:
            return [text]
        return self._split(text, self.separators)

    def _split(self, text: str, separators: list[str]) -> list[str]:
        if not text:
            return []
        if len(text) <= self.chunk_size or not separators:
            return [text]
        sep = separators[0]
        remaining = separators[1:]
        pieces = text.split(sep) if sep else list(text)
        result: list[str] = []
        current = ""
        for piece in pieces:
            sub = (current + sep + piece) if current else piece
            if len(sub) <= self.chunk_size:
                current = sub
            else:
                if current:
                    result.append(current)
                if len(piece) > self.chunk_size and remaining:
                    result.extend(self._split(piece, remaining))
                    current = ""
                else:
                    current = piece
        if current:
            result.append(current)
        return result
```

**Thành viên 2 — [Trần Đình Hinh]**
- **Loại chiến lược:** `SentenceChunker` (max_sentences_per_chunk=3)
- **Mô tả & lý do chọn:** Trong các văn bản quy định pháp lý và chính sách bảo hành, mỗi câu thường mang một mệnh đề điều kiện hoặc quyền lợi hoàn chỉnh ("nếu... thì...", "trong vòng..."). `SentenceChunker` sử dụng regex lookbehind tách theo dấu chấm kết câu giúp các điều kiện ràng buộc không bao giờ bị cắt vụn ngang xương, đảm bảo tính toàn vẹn của quy định.
- **Code snippet:**
```python
class SentenceChunker:
    def __init__(self, max_sentences_per_chunk: int = 3) -> None:
        self.max_sentences_per_chunk = max(1, max_sentences_per_chunk)

    def chunk(self, text: str) -> list[str]:
        if not text or not text.strip():
            return []
        sentences = [s.strip() for s in re.split(r'(?<=[.!?])\s+', text.strip()) if s.strip()]
        if not sentences:
            return []
        chunks: list[str] = []
        for i in range(0, len(sentences), self.max_sentences_per_chunk):
            group = sentences[i : i + self.max_sentences_per_chunk]
            chunk_str = " ".join(group).strip()
            if chunk_str:
                chunks.append(chunk_str)
        return chunks
```

**Thành viên 3 — [Lê Như Ý]**
- **Loại chiến lược:** `FixedSizeChunker` (chunk_size=500, overlap=50)
- **Mô tả & lý do chọn:** `FixedSizeChunker` chia văn bản thành các khối có độ dài cố định kèm 50 ký tự gối đầu (overlap) để duy trì tính liên tục của thông tin giữa các ranh giới cắt. Phương pháp này có ưu thế là tốc độ xử lý nhanh, kích thước chunk đồng đều tuyệt đối và không phụ thuộc vào cấu trúc ngữ pháp hay dấu phân đoạn của tài liệu.
- **Code snippet:**
```python
class FixedSizeChunker:
    def __init__(self, chunk_size: int = 500, overlap: int = 50) -> None:
        self.chunk_size = chunk_size
        self.overlap = overlap

    def chunk(self, text: str) -> list[str]:
        if not text:
            return []
        if len(text) <= self.chunk_size:
            return [text]
        step = self.chunk_size - self.overlap
        chunks: list[str] = []
        for start in range(0, len(text), step):
            chunk = text[start : start + self.chunk_size]
            chunks.append(chunk)
            if start + self.chunk_size >= len(text):
                break
        return chunks
```

### So Sánh Giữa Các Thành Viên

| Thành viên | Chiến lược (Strategy) | Điểm truy xuất (/10) | Điểm mạnh | Điểm yếu |
|-----------|------------------------|:--------------------:|-----------|----------|
| **Trần Tuấn Cường** | `RecursiveChunker` (size 300, overlap 30) | 10 / 10 (5/5 Top-1) | Bảo toàn cấu trúc thứ bậc ngữ nghĩa (Heading $\rightarrow$ Đoạn $\rightarrow$ Câu); điểm tương đồng cao và ngữ cảnh trích dẫn trọn vẹn nhất. | Thuật toán đệ quy phức tạp hơn, sinh ra nhiều chunk hơn (13-15 chunks/tài liệu). |
| **Trần Đình Hinh** | `SentenceChunker` (max 3 câu) | 10 / 10 (5/5 Top-1) | Giữ trọn vẹn từng câu quy định hoàn chỉnh; không bị ngắt quãng mệnh đề điều kiện hoặc quyền lợi. | Kích thước chunk không đồng đều (câu ghép dài tạo chunk lớn); bỏ qua phân cấp tiêu đề Markdown. |
| **Lê Như Ý** | `FixedSizeChunker` (size 500, overlap 50) | 8 / 10 (4/5 Top-1) | Đơn giản, tốc độ thực thi nhanh nhất; kích thước chunk đồng đều giúp vector embedding ổn định. | Cắt cơ học không theo ngữ nghĩa; dễ cắt rời bảng phí hoặc tên thương hiệu dẫn đến nhầm lẫn ở Câu 3. |

**Chiến lược nào tốt nhất cho chủ đề này? Tại sao?**
> **`RecursiveChunker`** là chiến lược tối ưu nhất cho tập tài liệu chính sách bảo hành thương mại điện tử. Văn bản chính sách có tính phân cấp cao (tên sàn $\rightarrow$ nhóm chính sách $\rightarrow$ từng điều khoản cụ thể), việc phân rã đệ quy theo đoạn `\n\n` rồi mới tới câu giúp giữ trọn vẹn tiêu đề điều khoản đi kèm nội dung chi tiết, không làm đứt đoạn các con số quy định (ngày, mức phí %) và mang lại độ chính xác truy xuất cao nhất (5/5 câu hỏi đều đạt Top-1).

---

## 3. Câu hỏi đánh giá & Chất lượng truy xuất (Retrieval Quality) — Nhóm (10 điểm)

### Câu hỏi đánh giá & Câu trả lời chuẩn (nhóm thống nhất)

> **Đúng 5 câu hỏi**, đa dạng, có thể kiểm chứng; **ít nhất 1 câu** cần lọc metadata mới trả lời tốt. Đây là bộ câu hỏi chung cho mọi thành viên chạy.

| # | Câu hỏi (Query) | Câu trả lời chuẩn (Gold Answer) | Chunk nào chứa thông tin? |
|---|-----------------|---------------------------------|--------------------------|
| 1 | Tại Thế Giới Di Động, chính sách Bảo hành có cam kết trong 12 tháng quy định thời gian xử lý tối đa là bao nhiêu ngày? | Thời gian xử lý cam kết tối đa trong vòng 15 ngày tính từ ngày nhận máy; nếu quá hạn hoặc lỗi lại trong 30 ngày sẽ đổi máy mới tương đương hoặc hoàn tiền 100%. | `thegioididong-warranty-policy.md` (Mục 2: Chính sách bảo hành có cam kết trong 12 tháng) |
| 2 | Thời gian bảo hành trung bình tại TTG Shop là bao nhiêu ngày và có chính sách hỗ trợ gì cho khách hàng? | Thời gian bảo hành trung bình là 07 ngày làm việc; khách hàng được hỗ trợ mượn sản phẩm/thiết bị thay thế miễn phí trong thời gian chờ bảo hành. | `ttgshop-warranty-policy.md` (Mục 2: Thời gian xử lý bảo hành & Mục 3: Chính sách mượn hàng thay thế) |
| 3 | Tại CellphoneS, mức phí nhập lại đối với điện thoại mới khi khách hàng đổi ý trong 30 ngày đầu là bao nhiêu? | Mức phí nhập lại là 20% đối với máy mới (hoặc 15% đối với máy cũ) tính trên giá niêm yết tại thời điểm trả hoặc giá trên hóa đơn (tùy giá nào thấp hơn). | `cellphones-warranty-policy.md` (Mục 2: Bảng phí đổi trả đối với sản phẩm không lỗi / đổi ý) |
| 4 | FPT Shop áp dụng chính sách 1 đổi 1 trong thời gian bao lâu đối với sản phẩm lỗi nhà sản xuất? | FPT Shop áp dụng chính sách 1 đổi 1 máy mới 100% cùng model trong vòng 30 ngày đầu tiên kể từ ngày xuất hóa đơn và nhận hàng thành công. | `fptshop-return-policy.md` (Mục 1: Chính sách 1 đổi 1 trong 30 ngày đầu) |
| 5 | Đối với đơn hàng Shopee có quyết định Hoàn tiền ngay, Người bán có bao nhiêu ngày để gửi khiếu nại? *(Yêu cầu lọc metadata: `audience="seller"`)* | Người bán bắt buộc phải gửi khiếu nại trong vòng 02 ngày kể từ khi nhận được thông báo từ Shopee; quá thời hạn khiếu nại sẽ không được giải quyết. | `seller-warranty-policy.md` (Mục 1: Khiếu nại đối với quyết định Hoàn tiền ngay của Shopee) |

### Tổng hợp chất lượng truy xuất của nhóm

> Cách chấm (theo `docs/SCORING.md`): **2 điểm/câu** — top-3 chứa chunk liên quan + agent trả lời đúng (2), có liên quan nhưng thiếu/không ở top-1 (1), không có trong top-3 (0).

| # | Câu hỏi | Chiến lược tốt nhất cho câu này | Có chunk liên quan trong top-3? | Ghi chú |
|---|---------|--------------------------------|:------------------------------:|---------|
| 1 | TGDD: Thời gian xử lý bảo hành cam kết tối đa? | `RecursiveChunker` / `SentenceChunker` | Có (Top-1 cả 3 thành viên) | Cả 3 chiến lược đều truy xuất chính xác thời hạn cam kết 15 ngày. (2/2 điểm) |
| 2 | TTG Shop: Số ngày bảo hành TB và chính sách hỗ trợ? | `RecursiveChunker` / `SentenceChunker` | Có (Top-1 cả 3 thành viên) | Trả lời đầy đủ 07 ngày làm việc và chính sách mượn máy miễn phí. (2/2 điểm) |
| 3 | CellphoneS: Phí nhập lại máy mới đổi ý 30 ngày đầu? | `RecursiveChunker` / `SentenceChunker` | Có (Top-1 ở Cường & Hinh; Top-3 ở Ý) | FixedSize bị nhiễu do cắt rời bảng phí; Recursive giữ trọn vẹn mức 20%. (2/2 điểm) |
| 4 | FPT Shop: Thời hạn 1 đổi 1 lỗi nhà sản xuất? | `RecursiveChunker` / `SentenceChunker` | Có (Top-1 ở Cường & Hinh; Top-2 ở Ý) | Trả về chính xác thời hạn 30 ngày đầu tiên của FPT Shop. (2/2 điểm) |
| 5 | Shopee Seller: Thời hạn gửi khiếu nại Hoàn tiền ngay? | Cả 3 chiến lược kết hợp Pre-filtering (`audience="seller"`) | Có (Top-1 cả 3 thành viên khi lọc metadata) | Bắt buộc phải áp dụng lọc metadata để loại trừ chính sách hoàn tiền của Buyer. (2/2 điểm) |

**Điểm tổng hợp chất lượng truy xuất: 10 / 10**

**Lọc bằng metadata có giúp ích không? Ở câu hỏi nào?**
> Lọc bằng metadata đóng vai trò quyết định ở **Câu hỏi 5** (`audience="seller"`). Trong kho dữ liệu TMĐT, Shopee có hai bộ chính sách song song dành cho Người mua (Buyer) và Người bán (Seller) với nhiều từ khóa hoàn tiền và khiếu nại trùng lặp. Nếu không lọc theo metadata, mô hình sẽ bị nhiễu bởi các tài liệu của Người mua chiếm ưu thế từ vựng; khi áp dụng lọc `{"audience": "seller"}`, kho vector cô lập chính xác văn bản khiếu nại của Người bán, đưa tỷ lệ trả lời đúng đạt 100% ở vị trí Top-1.

---

## 4. Thuyết trình (Demo) & Bài học nhóm — Nhóm (5 điểm)

**Những phân tích (insights) hay nhất nhóm sẽ trình bày:**
> 1. **Cấu trúc tài liệu quyết định chiến lược chunking:** Văn bản chính sách có tính phân cấp tiêu đề chặt chẽ, do đó các chiến lược tôn trọng ranh giới ngữ nghĩa (`RecursiveChunker`, `SentenceChunker`) cho điểm truy xuất vượt trội và ít hallucination hơn hẳn chia theo kích thước cố định (`FixedSizeChunker`).
> 2. **Sức mạnh của Tiền lọc (Pre-filtering) Metadata:** Trong các hệ thống RAG thực tế phục vụ nhiều đối tượng khách hàng (Buyer vs Seller, sàn A vs sàn B), việc tiền lọc metadata trước khi tính vector similarity là giải pháp then chốt để triệt tiêu nhiễu chéo mà không làm tăng chi phí tính toán.
> 3. **Prompt grounding chống ảo giác:** Cấu trúc ngữ cảnh `[1]`, `[2]` kèm yêu cầu trích dẫn rõ nguồn gốc (source_url, platform) giúp LLM Agent luôn neo chặt vào sự thật trong tài liệu, hoàn toàn tránh được hiện tượng bịa đặt thông tin.

**Bài học rút ra khi so sánh trong nhóm:**
> Khi so sánh 3 chiến lược trên cùng một bộ tài liệu và 5 câu hỏi chuẩn, nhóm nhận thấy `FixedSizeChunker` dễ làm đứt gãy bảng tỷ lệ phí hoặc số liệu quan trọng khi rơi vào ranh giới cắt; `SentenceChunker` giữ trọn vẹn ngữ nghĩa câu nhưng kích thước chunk không đều; còn `RecursiveChunker` tạo ra sự cân bằng hoàn hảo nhất giữa kích thước vector tối ưu và tính toàn vẹn ngữ cảnh.

**Nếu làm lại, nhóm sẽ thay đổi gì trong chiến lược dữ liệu (data strategy)?**
> Nhóm sẽ chuẩn hóa dữ liệu đầu vào bằng parser chuyên biệt cho bảng biểu Markdown (Markdown Table Parser) để các hàng trong bảng phí không bị phân mảnh, đồng thời bổ sung thêm các trường metadata chi tiết như `product_category` (điện thoại, laptop, gia dụng) và `policy_version` để hỗ trợ truy vấn lọc đa tầng chính xác hơn nữa.

---

## Tự Đánh Giá (Phần Nhóm)

| Tiêu chí | Điểm tự đánh giá |
|----------|:-----------------:|
| Lựa chọn tài liệu (Document Set Quality) | 10 / 10 |
| Thiết kế chiến lược (Strategy Design) | 15 / 15 |
| Chất lượng truy xuất (Retrieval Quality) | 10 / 10 |
| Thuyết trình (Demo) | 5 / 5 |
| **Tổng phần nhóm** | **40 / 40** |

