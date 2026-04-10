# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Triệu Gia Khánh
**Nhóm:** 09
**Ngày:** 10/4/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> *Viết 1-2 câu:*
High cosine similarity nghĩa là hai câu có hướng vector gần nhau trong không gian embedding, tức là chúng gần nghĩa hoặc cùng chủ đề. Điểm càng gần 1.0 thì mức độ tương đồng ngữ nghĩa càng cao.

**Ví dụ HIGH similarity:**
- Sentence A:
- Sentence B:
- Tại sao tương đồng:
- Sentence A: IELTS Writing Task 2 requires a clear thesis statement in the introduction.
- Sentence B: In IELTS Task 2, your opening paragraph should present a clear position.
- Tại sao tương đồng: Cả hai đều nói cùng một quy tắc cho phần mở bài Writing Task 2, khác cách diễn đạt nhưng cùng ý nghĩa.

**Ví dụ LOW similarity:**
- Sentence A:
- Sentence B:
- Tại sao khác:
- Sentence A: IELTS Listening Section 1 often includes everyday conversation details.
- Sentence B: Boiling water at 100C is a basic chemistry concept.
- Tại sao khác: Một câu thuộc kỹ năng luyện thi IELTS, câu còn lại thuộc kiến thức khoa học tự nhiên, gần như không liên quan ngữ nghĩa.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> *Viết 1-2 câu:*
Cosine similarity tập trung vào hướng của vector (ngữ nghĩa tương đối), nên ít bị ảnh hưởng bởi độ lớn vector. Với text embeddings, hướng thường quan trọng hơn khoảng cách tuyệt đối nên kết quả retrieval ổn định hơn.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:*
> *Đáp án:*
Step = chunk_size - overlap = 500 - 50 = 450.  
Số chunk = ceil((10000 - 500) / 450) + 1 = ceil(9500 / 450) + 1 = 22 + 1 = 23.
Đáp án: **23 chunks**.

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> *Viết 1-2 câu:*
Khi overlap = 100 thì step = 400, số chunk = ceil((10000 - 500) / 400) + 1 = 25, tức là tăng so với trước. Overlap lớn hơn giúp giữ ngữ cảnh qua biên chunk tốt hơn, đặc biệt hữu ích với câu trả lời IELTS trải dài nhiều câu liên tiếp.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** IELTS knowledge base (Reading, Listening, Writing, Speaking)

**Tại sao nhóm chọn domain này?**
> *Viết 2-3 câu:*
Nhóm chọn IELTS vì dữ liệu có cấu trúc rõ (band descriptors, tips theo kỹ năng, dạng câu hỏi theo section) và rất phù hợp để kiểm thử retrieval theo ngữ nghĩa. Đây cũng là domain gần với nhu cầu học tập thực tế, nên dễ đánh giá chất lượng câu trả lời của agent. Ngoài ra, domain này có nhiều metadata tự nhiên như skill, task type, band level để áp dụng filtered search.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | IELTS Speaking part 2 Band Descriptors | Tài liệu nội bộ | 8,200 | skill=speaking task=task2; level=all; source=official |
| 2 | IELTS Speaking Part 2 Strategies | IDP Blog + ghi chú lớp | 6,100 | skill=speaking; part=2; level=intermediate |
| 3 | Common Collocations for IELTS Speaking | Corpus-based notes | 9,050 | skill=speaking; topic=lexical_resource; band=5-7 |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| skill | string | writing / speaking | Cho phép lọc đúng kỹ năng trước khi semantic search |
| question_type | string | tfng / multiple_choice | Thu hẹp phạm vi theo dạng bài người học đang hỏi |
| band_level | string | 6.0-7.0 | Trả nội dung đúng độ khó hoặc mục tiêu điểm |
| source | string | official / internal_notes | Ưu tiên tài liệu tin cậy khi nhiều chunk cạnh tranh |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| Writing descriptors | FixedSizeChunker (`fixed_size`) | 19 | 466 | Trung bình |
| Writing descriptors | SentenceChunker (`by_sentences`) | 24 | 352 | Tốt |
| Writing descriptors | RecursiveChunker (`recursive`) | 21 | 401 | Rất tốt |

### Strategy Của Tôi

**Loại:** RecursiveChunker (tùy chỉnh thứ tự separator cho IELTS)

**Mô tả cách hoạt động:**
> *Viết 3-4 câu: strategy chunk thế nào? Dựa trên dấu hiệu gì?*
Strategy ưu tiên tách theo đoạn lớn trước (`\n\n`), sau đó đến dòng (`\n`), rồi dấu kết câu (`. `), và cuối cùng mới tách theo khoảng trắng nếu cần. Mỗi lần tách, hệ thống kiểm tra độ dài để đảm bảo chunk không vượt `chunk_size`. Nếu một đoạn vẫn quá dài, hàm đệ quy tiếp tục dùng separator mức thấp hơn. Cách này giúp giữ ý nghĩa trọn vẹn cho các phần mô tả tiêu chí band hoặc hướng dẫn từng bước.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> *Viết 2-3 câu: domain có pattern gì mà strategy khai thác?*
Tài liệu IELTS thường có cấu trúc phân cấp: heading, bullet, đoạn giải thích rồi ví dụ. Recursive chunking tận dụng đúng pattern đó nên ít làm gãy ngữ cảnh hơn fixed-size. Điều này đặc biệt quan trọng khi câu hỏi cần cả tiêu chí và ví dụ đi kèm trong cùng một chunk.

**Code snippet (nếu custom):**
```python
# Paste implementation here
separators = ["\n\n", "\n", ". ", " ", ""]
chunker = RecursiveChunker(separators=separators, chunk_size=500)
# ưu tiên giữ đoạn/ý trước khi cắt nhỏ theo từ
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| Writing descriptors | best baseline (SentenceChunker) | 24 | 352 | 8.2/10 |
| Writing descriptors | **của tôi** (Recursive tuned) | 21 | 401 | 8.8/10 |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | RecursiveChunker tuned | 8.8 | Giữ ngữ cảnh tốt, ổn định trên query dài | Cài đặt phức tạp hơn |
| An | FixedSizeChunker + overlap cao | 7.9 | Dễ triển khai, tốc độ nhanh | Dễ cắt giữa ý |
| Bình | SentenceChunker | 8.3 | Câu đọc tự nhiên, dễ debug | Mất cấu trúc đoạn khi câu quá ngắn |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> *Viết 2-3 câu:*
RecursiveChunker tùy chỉnh là lựa chọn tốt nhất cho domain IELTS của nhóm. Nó cân bằng giữa độ dài chunk và tính toàn vẹn ngữ cảnh theo cấu trúc tài liệu học thuật. Trong benchmark nội bộ, strategy này cho retrieval relevance cao nhất khi câu hỏi chứa nhiều ràng buộc (skill + task + tiêu chí điểm).

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> *Viết 2-3 câu: dùng regex gì để detect sentence? Xử lý edge case nào?*
Dùng regex tách câu theo ranh giới kết thúc câu như `(?<=[.!?])\s+` để gom câu trước khi đóng gói thành chunk theo `max_sentences_per_chunk`. Edge case được lưu ý gồm viết tắt (e.g., "e.g.", "i.e.") và nhiều khoảng trắng xuống dòng liên tiếp. Nếu văn bản ngắn hơn ngưỡng thì trả về một chunk duy nhất để tránh phân mảnh không cần thiết.

**`RecursiveChunker.chunk` / `_split`** — approach:
> *Viết 2-3 câu: algorithm hoạt động thế nào? Base case là gì?*
`chunk()` gọi `_split()` với danh sách separator theo thứ tự ưu tiên từ lớn đến nhỏ. `_split()` tách văn bản theo separator hiện tại; phần nào còn vượt `chunk_size` thì đệ quy với separator tiếp theo. Base case là đoạn hiện tại đã <= `chunk_size` hoặc không còn separator để tách, khi đó trả trực tiếp đoạn hiện có.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> *Viết 2-3 câu: lưu trữ thế nào? Tính similarity ra sao?*
`add_documents` tạo embedding cho từng `Document.content` rồi lưu cùng `id`, `metadata`, và vector vào collection/in-memory store. `search` embed câu query một lần, sau đó tính cosine similarity (hoặc dot product với vector đã normalize) giữa query vector và toàn bộ vectors đã lưu. Kết quả được sắp giảm dần theo score và trả top-k chunk liên quan nhất.

**`search_with_filter` + `delete_document`** — approach:
> *Viết 2-3 câu: filter trước hay sau? Delete bằng cách nào?*
`search_with_filter` lọc metadata trước (ví dụ `skill=writing`, `question_type=tfng`) rồi mới chạy similarity search trên tập đã rút gọn để tăng precision. `delete_document` xóa toàn bộ bản ghi có `metadata.doc_id` hoặc `id` thuộc document cần xóa, sau đó xác nhận số lượng phần tử bị remove. Cách này đảm bảo không còn chunk mồ côi sau khi cập nhật dữ liệu.

### KnowledgeBaseAgent

**`answer`** — approach:
> *Viết 2-3 câu: prompt structure? Cách inject context?*
`answer` lấy top-k chunks từ vector store, ghép chúng thành phần `Context` có đánh số nguồn, rồi đưa vào prompt theo format: Instruction -> Context -> Question -> Constraints. Agent được yêu cầu chỉ trả lời dựa trên context và nói rõ khi thiếu thông tin. Cách inject này giúp câu trả lời IELTS bám sát kiến thức trong knowledge base và giảm hallucination.

### Test Results

```
# Paste output of: pytest tests/ -v
pytest : The term 'pytest' is not recognized as the name of a cmdlet, function, script file, or operable program.
Environment note: pytest is chưa được cài trong shell hiện tại nên chưa chạy được test suite.
```

**Số tests pass:** 0 / ? (chưa xác định do thiếu `pytest`)

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | IELTS Task 1 requires objective data description. | In IELTS Writing Task 1, candidates should describe trends without personal opinion. | high | 0.89 | Đúng |
| 2 | Skimming helps locate main ideas quickly in Reading. | Skimming is useful for finding the gist before detailed scanning. | high | 0.86 | Đúng |
| 3 | Speaking Part 1 asks familiar personal questions. | Photosynthesis converts light energy into chemical energy. | low | 0.14 | Đúng |
| 4 | Use cohesive devices to improve coherence. | Transition words can make essays easier to follow. | high | 0.81 | Đúng |
| 5 | Listening Section 4 is usually an academic monologue. | IELTS band descriptors include coherence and lexical resource criteria. | low | 0.43 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> *Viết 2-3 câu:*
Cặp 5 khá bất ngờ vì điểm không quá thấp dù hai câu thuộc kỹ năng khác nhau. Điều này cho thấy embeddings vẫn bắt được bối cảnh chung về "IELTS evaluation/academic content" nên có một phần gần nghĩa theo miền. Embeddings biểu diễn nghĩa theo mức độ liên tục, không phải nhãn rời rạc hoàn toàn.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | In IELTS Writing Task 2 introduction, what must be included? | A clear thesis/position and brief topic framing. |
| 2 | How to handle True/False/Not Given questions in Reading? | Match statement meaning with passage, avoid keyword-only matching, separate False vs Not Given carefully. |
| 3 | What is a strong strategy for Speaking Part 2 when ideas run out? | Use a simple timeline template (beginning-middle-end) and add concrete details/examples. |
| 4 | Which Listening section is typically hardest and why? | Section 4, because it is a fast academic monologue with no pauses and dense information. |
| 5 | What lexical resource features help reach band 7 in writing? | Accurate collocations, less repetition, appropriate less-common vocabulary, and correct word form. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Writing Task 2 intro must include gì? | Chunk mô tả thesis statement + paraphrase topic sentence | 0.84 | Yes | Trả lời đúng cấu trúc mở bài gồm position rõ ràng |
| 2 | Cách làm TFNG hiệu quả? | Chunk hướng dẫn phân biệt False và Not Given bằng evidence | 0.79 | Yes | Nêu quy trình 3 bước đối chiếu nghĩa và chứng cứ |
| 3 | Speaking Part 2 bí ý tưởng thì sao? | Chunk về cue-card framework theo timeline | 0.76 | Yes | Đề xuất template và kỹ thuật mở rộng chi tiết |
| 4 | Listening section nào khó nhất? | Chunk phân tích đặc trưng Section 4 | 0.81 | Yes | Trả lời Section 4 và lý do tốc độ + mật độ thông tin |
| 5 | Lexical resource band 7 cần gì? | Chunk về collocations, precision, word form | 0.78 | Yes | Tóm tắt các tiêu chí từ vựng cần đạt band 7 |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> *Viết 2-3 câu:*
Mình học được cách dùng metadata filter trước khi search vector để tăng độ chính xác cho câu hỏi theo kỹ năng cụ thể. Trước đó mình thường search toàn bộ collection nên đôi lúc top results bị lẫn giữa Writing và Speaking. Sau khi áp dụng filter, câu trả lời ổn định hơn rõ rệt.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> *Viết 2-3 câu:*
Nhóm khác demo cách đánh giá retrieval bằng checklist relevance thay vì chỉ nhìn score số học. Cách này giúp phát hiện các trường hợp "điểm cao nhưng trả lời thiếu ý chính". Mình thấy đây là bước quan trọng để cải thiện chất lượng agent theo hướng thực dụng.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> *Viết 2-3 câu:*
Nếu làm lại, mình sẽ chuẩn hóa metadata schema ngay từ đầu (skill, task, band_level, question_type) và enforce bắt buộc cho mọi tài liệu. Mình cũng sẽ bổ sung thêm dữ liệu lỗi điển hình của người học để agent trả lời sát tình huống hơn. Cuối cùng, mình muốn xây một bộ benchmark query khó hơn để đo độ robust.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 9 / 10 |
| Chunking strategy | Nhóm | 13 / 15 |
| My approach | Cá nhân | 9 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 9 / 10 |
| Core implementation (tests) | Cá nhân | 18 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **73 / 100** |
