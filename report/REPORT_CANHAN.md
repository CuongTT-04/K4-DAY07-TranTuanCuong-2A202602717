# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Trần Tuấn Cường
**Nhóm:** G41
**Ngày:** 20/09/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> Hai đoạn văn bản có độ tương tự cosine cao (tiến gần về 1.0) nghĩa là hai vector embedding của chúng chỉ về cùng một hướng trong không gian đa chiều, thể hiện sự tương đồng lớn về mặt ngữ nghĩa và chủ đề bất kể độ dài ngắn của văn bản.

**Ví dụ có độ tương tự CAO:**
- Câu A: Chính sách bảo hành điện thoại áp dụng 1 đổi 1 trong 30 ngày đầu tiên.
- Câu B: Khách hàng được quyền đổi mới thiết bị miễn phí trong vòng một tháng nếu có lỗi từ nhà sản xuất.
- Tại sao tương đồng: Cả hai câu đều nói về cùng một quy định hỗ trợ khách hàng đổi thiết bị mới khi gặp lỗi trong thời hạn 30 ngày (1 tháng), chỉ khác biệt về cách dùng từ ngữ.

**Ví dụ có độ tương tự THẤP:**
- Câu A: Thời gian bảo hành sản phẩm trung bình là 7 ngày làm việc.
- Câu B: Cách làm món phở bò Hà Nội thơm ngon đúng điệu truyền thống.
- Tại sao khác: Hai câu thuộc hai miền ngữ nghĩa hoàn toàn độc lập (một câu về quy định dịch vụ kỹ thuật, một câu về công thức ẩm thực) nên vector gần như vuông góc nhau trong không gian embedding.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> Khoảng cách Euclid bị chi phối bởi độ dài (độ lớn vector) của văn bản: một đoạn văn dài và một câu ngắn cùng nội dung có thể có khoảng cách Euclid rất xa nhau. Trong khi đó, độ tương tự cosine chuẩn hóa độ dài và chỉ đo góc giữa hai vector, giúp đánh giá chính xác mức độ trùng khớp ngữ nghĩa mà không bị ảnh hưởng bởi độ dài câu từ.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:*
> Áp dụng công thức: $\text{Số lượng chunk} = \lceil (\text{độ\_dài} - \text{độ\_chồng\_chéo}) / (\text{kích\_thước\_chunk} - \text{độ\_chồng\_chéo}) \rceil$
> $= \lceil (10000 - 50) / (500 - 50) \rceil = \lceil 9950 / 450 \rceil = \lceil 22.11 \rceil = 23$
> *Đáp án:* **23 chunks**

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> Khi overlap tăng lên 100: $\lceil (10000 - 100) / (500 - 100) \rceil = \lceil 9900 / 400 \rceil = \lceil 24.75 \rceil = 25$ chunks (tăng thêm 2 chunks). Tăng độ chồng chéo giúp bảo toàn ngữ cảnh liền mạch giữa các câu/đoạn nằm ngay tại ranh giới bị cắt, đảm bảo thông tin quan trọng không bị đứt đoạn giữa chừng.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
> Sử dụng biểu thức chính quy Lookbehind `(?<=[.!?])(?:\s+|\n+)` để nhận diện ranh giới kết thúc câu mà không làm mất dấu chấm câu. Sau đó nhóm tuần tự các câu hoàn chỉnh thành từng chunk tối đa `max_sentences_per_chunk` câu, đồng thời `strip()` để loại bỏ khoảng trắng thừa đầu và cuối mỗi đoạn.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
> Thuật toán chia đệ quy theo danh sách phân cách có độ ưu tiên giảm dần: `["\n\n", "\n", ". ", " ", ""]`. Base case là khi đoạn văn bản có độ dài $\le \text{chunk\_size}$ hoặc danh sách phân cách rỗng. Ở mỗi tầng, văn bản được tách ra thành các mảnh nhỏ và gộp lại cho đến ngưỡng `chunk_size`, mảnh nào vượt kích thước sẽ được gọi đệ quy với dấu phân cách tiếp theo.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
> Tài liệu được chuẩn hóa thành bản ghi lưu trong danh sách nội bộ `_store` gồm `id`, `content`, `metadata` và vector `embedding` được tạo từ `_embedding_fn`. Khi tìm kiếm, hàm tính tích vô hướng (dot product) giữa embedding của truy vấn và từng tài liệu trong kho, sắp xếp kết quả giảm dần theo điểm số (`score`) và trả về `top_k` bản ghi cao nhất.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
> `search_with_filter` thực hiện tiền lọc (pre-filtering) bằng cách duyệt qua `_store` và chọn ra các bản ghi thỏa mãn tất cả cặp key-value trong `metadata_filter` trước, sau đó mới tính độ tương đồng. `delete_document` so khớp `doc_id` với trường `id` hoặc `metadata['doc_id']`, lọc loại bỏ toàn bộ bản ghi trùng khớp và trả về `True` nếu có ít nhất một bản ghi bị xóa.

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
> Tác tử nhận câu hỏi, gọi `store.search(question, top_k)` để trích xuất các chunk có độ tương đồng cao nhất. Sau đó nối các chunk lại thành một khối ngữ cảnh và định dạng vào template prompt: `Context:\n{context}\n\nQuestion: {question}\n\nAnswer:`, rồi truyền prompt hoàn chỉnh này vào hàm `llm_fn` để sinh câu trả lời có căn cứ.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
============================= test session starts =============================
platform win32 -- Python 3.14.5, pytest-9.1.1, pluggy-1.6.0
rootdir: D:\VinAI\K4-DAY07-TranTuanCuong-2A202602717
plugins: anyio-4.14.1, langsmith-0.9.8
collected 42 items

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED   [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED    [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED   [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]

============================= 42 passed in 0.07s ==============================
```

**Số lượng bài test vượt qua (pass):** 42 / 42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|:---:|:------|:------|:-------:|:------------:|:-----:|
| 1 | Chính sách bảo hành sản phẩm trong 12 tháng. | Thời hạn bảo hành thiết bị là một năm. | cao | 0.0194 | Sai |
| 2 | Quy trình đổi trả hàng và hoàn tiền cho người mua. | Hướng dẫn trả lại sản phẩm và nhận lại tiền. | cao | 0.0343 | Sai |
| 3 | Khách hàng được đổi mới 1 đổi 1 trong 30 ngày. | Thời gian bảo hành trung bình tại cửa hàng là 7 ngày làm việc. | thấp | 0.1841 | Đúng |
| 4 | Sản phẩm bị rơi vỡ vào nước sẽ bị từ chối bảo hành. | Công thức nấu món phở bò truyền thống thơm ngon. | thấp | 0.0573 | Đúng |
| 5 | Người bán có trách nhiệm phản hồi khiếu nại trong 24 giờ. | Người bán phải trả lời tin nhắn thắc mắc trong vòng 1 ngày. | cao | -0.0274 | Sai |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
> Điểm số thực tế của Cặp 1 và Cặp 5 rất thấp (gần 0 và âm) dù về mặt ngữ nghĩa con người nhận thấy chúng gần như trùng khớp hoàn toàn. Điều này xảy ra do bài kiểm thử sử dụng `MockEmbedder` (sinh vector giả lập dựa trên hàm băm MD5 của từng chuỗi ký tự thay vì mô hình học máy ngữ nghĩa). Điều này cho thấy thuật toán băm bề mặt (lexical/hash) không thể nhận diện được từ đồng nghĩa; để xây dựng RAG thực sự cần các mô hình Dense Embedding đa ngôn ngữ (như Multilingual-MiniLM, OpenAI text-embedding-3) để phản ánh đúng không gian ngữ nghĩa tiềm ẩn.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân với chiến lược **`RecursiveChunker`** (chia tách theo khối cấu trúc Markdown và đoạn văn, `chunk_size=500`):

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|-----------------|--------------------------------------|:----------:|:------------------------------:|---------------------------------|
| 1 | Tại Thế Giới Di Động, chính sách Bảo hành có cam kết trong 12 tháng quy định thời gian xử lý tối đa là bao nhiêu ngày? | `thegioididong-warranty-policy.md`: Bảo hành có cam kết trong 12 tháng. Thời gian xử lý cam kết trong vòng 15 ngày... | 0.3616 | Có | Bảo hành trong vòng 15 ngày; nếu quá hạn hoặc lỗi lại trong 30 ngày sẽ đổi máy tương đương hoặc hoàn tiền. |
| 2 | Thời gian bảo hành trung bình tại TTG Shop là bao nhiêu ngày và có chính sách hỗ trợ gì? | `ttgshop-warranty-policy.md`: Thời gian bảo hành trung bình là 07 ngày làm việc; chính sách cho mượn sản phẩm thay thế miễn phí... | 0.2845 | Có | Thời gian bảo hành trung bình 7 ngày làm việc, có chính sách cho mượn thiết bị thay thế miễn phí. |
| 3 | Tại CellphoneS, mức phí nhập lại đối với điện thoại mới khi khách hàng đổi ý trong 30 ngày đầu là bao nhiêu? | `cellphones-warranty-policy.md`: Điện thoại, Máy tính bảng trong 30 ngày đầu thu phí 20% đối với máy mới... | 0.3120 | Có | Trong 30 ngày đầu tiên thu phí 20% đối với máy mới tính trên giá niêm yết hiện tại hoặc giá mua hóa đơn. |
| 4 | FPT Shop áp dụng chính sách 1 đổi 1 trong thời gian bao lâu đối với sản phẩm lỗi nhà sản xuất? | `fptshop-return-policy.md`: Chính sách 1 đổi 1 trong 30 ngày đầu tiên nếu sản phẩm phát sinh lỗi phần cứng từ nhà sản xuất... | 0.3450 | Có | Khách hàng được áp dụng chính sách 1 đổi 1 máy mới 100% trong vòng 30 ngày đầu tiên. |
| 5 | Đối với đơn hàng Shopee có quyết định Hoàn tiền ngay, Người bán có bao nhiêu ngày để gửi khiếu nại? *(Lọc: `audience: seller`)* | `seller-warranty-policy.md`: Trường hợp Shopee quyết định Hoàn tiền ngay, Người bán phải khiếu nại lại trong vòng 02 ngày... | 0.2980 | Có | Người bán bắt buộc phải gửi yêu cầu khiếu nại lại trong vòng 02 ngày kể từ khi nhận thông báo. |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 5 / 5 (100%)

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> Khi so sánh với `FixedSizeChunker`, chiến lược `RecursiveChunker` của tôi vượt trội hoàn toàn vì không bao giờ bị cắt xén câu chữ hay mất ranh giới điều khoản quan trọng (như con số thời hạn bảo hành). Tuy nhiên, tôi cũng học được từ thành viên dùng `SentenceChunker` rằng với các tài liệu văn bản ngắn hoặc dạng hỏi đáp FAQ, chia theo câu đơn lẻ giúp giảm nhiễu và tăng độ tập trung của vector embedding tốt hơn.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|:----------------:|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 10 / 10 |
| **Tổng phần cá nhân** | **60 / 60** |
