# Migration & Implementation Document: WIFIM - 01 - AUTO WRITE ARTICLE

## 1. Bối cảnh & Bảo toàn dữ liệu gốc
- **Workflow gốc**: `Auto viết bài Website.json` đã được lưu trữ bản nguyên gốc tại: [`workflows/reference/Auto-viet-bai-Website-original.json`](file:///d:/xampp/htdocs/VietBai/workflows/reference/Auto-viet-bai-Website-original.json) (không chỉnh sửa).
- **Workflow đăng bài gốc**: Đã lưu tại [`workflows/reference/Auto-dang-bai-original.json`](file:///d:/xampp/htdocs/VietBai/workflows/reference/Auto-dang-bai-original.json) (chỉ dùng tham chiếu, không chạy).
- **Workflow Phase 3**: [`workflows/wifim-ai-minimal-test.json`](file:///d:/xampp/htdocs/VietBai/workflows/wifim-ai-minimal-test.json) giữ nguyên, đã PASS với OpenAI.
- **Workflow chính thức**: [`workflows/wifim-auto-write-article.json`](file:///d:/xampp/htdocs/VietBai/workflows/wifim-auto-write-article.json).

---

## 2. Tiến độ triển khai theo từng bước (Step-by-Step Progress)

- [x] **STEP A: Audit Workflow gốc**: Đã hoàn tất phân tích 15 node, xác định các hạn chế cốt lõi (SERP ảo, mất context ID, hard-code folder/sheet).
- [x] **STEP B: Skeleton Workflow**: Đã hoàn tất và **RUNTIME TEST PASS 100%** (Cả 4 Case A, B, C, D).
- [x] **STEP C: Real Source Research (Đã implement, sẵn sàng test)**:
  - Tách bạch 100% Real Source Fetch -> Clean Data -> AI Analysis.
  - Khôi phục context bằng `.first()` qua node `Restore Task Context`.
  - Fetch URL WIFIM thật, clean HTML, giới hạn 2500-3000 ký tự.
  - Ràng buộc AI không hallucinate URL, không bịa competitor hay People Also Ask.
  - Lưu kết quả `Research_Result` dạng JSON string vào Google Sheet.
- [ ] **STEP D: Tích hợp Generate Outline**: AI Content Architect tạo Outline cấu trúc H2/H3 -> Format Outline.
- [ ] **STEP E: Tích hợp Generate Article**: AI Master Content Writer viết bài hoàn chỉnh HTML body.
- [ ] **STEP F: Tích hợp Google Drive & Google Docs**: Tạo Folder bài viết -> Upload HTML & Convert sang Google Docs -> Lấy `Drive_URL`.
- [ ] **STEP G: Cập nhật kết quả Google Sheet**: Update dòng về trạng thái `DONE` hoặc `ERROR`.
- [ ] **STEP H: End-to-End Testing**: Kiểm thử toàn trình với từ khóa mẫu.

---

## 3. Chi tiết kỹ thuật STEP C (Real Source Research)

### 3.1. Danh sách 7 Node của STEP C
1. **`Restore Task Context`** (`code` v2):
   - **Input**: Output từ node `Update Row: PROCESSING`.
   - **Xử lý**: Dùng `.first()` khôi phục `ID`, `row_number`, `Keyword`, `Created_At` từ node gốc `Filter & Take First PENDING Task` và lấy `website_source_url` từ `Workflow Config`.
   - **Output**: Object context sạch, đầy đủ metadata.
2. **`Fetch WIFIM Source URL`** (`httpRequest` v4.2):
   - **Input**: `website_source_url` (`https://wifim.vn`).
   - **Cấu hình**: GET request, timeout 8000ms, `onError: "continueRegularOutput"`.
   - **Output**: HTML body thô (hoặc error object nếu URL lỗi/timeout).
3. **`Clean Source Data`** (`code` v2):
   - **Input**: Raw response từ HTTP Request và context.
   - **Xử lý**: Lọc bỏ thẻ script, style, noscript, nav, header, footer, svg, comments. Decode HTML entities. Chuẩn hóa khoảng trắng.
   - **Ràng buộc**: Giới hạn tối đa 3000 ký tự. Nếu fetch lỗi -> `source_fetch_status = 'failed'`, `sources_used = []`, `research_mode = 'LLM_INFERENCE_ONLY'`, `brand_context = ''`.
   - **Output**: `cleaned_source_text`, `sources_used`, `source_fetch_status`, `research_mode`, `search_intent_basis`, `source_warning`.
4. **`AI Research Analysis`** (`chainLlm` v1.5):
   - **Input**: Keyword + cleaned_source_text + metadata nguồn.
   - **Ràng buộc Prompt**: Cấm tuyệt đối bịa URL đối thủ, cấm bịa People Also Ask, cấm phát minh số liệu WIFIM.
5. **`OpenAI Model (Research)`** (`lmChatOpenAi` v1.2):
   - Sub-node: `gpt-4o-mini`, temperature 0.3.
6. **`Structured Output Parser (Research)`** (`outputParserStructured` v1.3):
   - Ép output AI theo schema 13 trường chuẩn.
7. **`Format Research Result`** (`code` v2):
   - Serialize kết quả phân tích thành chuỗi JSON sạch gán vào trường `Research_Result`.
   - Bảo toàn `ID` và `row_number` cho node Google Sheets tiếp theo.
8. **`Update Row: Save Research Result`** (`googleSheets` v4.5):
   - **Matching**: `matchingColumns: ["ID"]`.
   - **Cập nhật**: `Research_Result = JSON string`, giữ nguyên `Status = 'PROCESSING'`.

### 3.2. Thông số kiểm thử Fetch thực tế từ `https://wifim.vn`
- **HTTP Status**: `200 OK`
- **Raw HTML Length**: `375,214` bytes (~375 KB)
- **Cleaned Text Length (toàn bộ)**: `5,920` ký tự
- **Bounded Cleaned Text (đưa vào AI)**: `3,000` ký tự
- **Trích đoạn Cleaned Text mẫu (Sample 400 ký tự đầu)**:
  ```text
  WIFIM JSC - Công ty Digital Marketing Online Thuê Ngoài
  WIFIM JSC có 5+ năm kinh nghiệm thực chiến về Công nghệ, Marketing & Media.
  WIFIM JSC – Phủ sóng thương hiệu, hoạt động như một phòng marketing thuê ngoài với chi phí chỉ như 1 nhân sự cho từng gói công việc. Giúp doanh nghiệp đạt HIỆU QUẢ trong tiến độ và chất lượng công việc, Kết quả mong muốn với chi phí hợp lý.
  Dịch vụ: Phòng Marketing Thuê Ngoài, Quảng cáo Google/Facebook/TikTok, Thiết kế Website chuẩn SEO, Sản xuất Media...
  ```
  *(Đánh giá: Nguồn dữ liệu cực kỳ sạch, phản ánh chính xác năng lực và dịch vụ thực tế của WIFIM, sẵn sàng cho AI phân tích).*

---

## 4. Structured Research Output Schema Cuối cùng
```json
{
  "search_intent": "Doanh nghiệp tìm kiếm đơn vị cung cấp giải pháp Marketing tổng thể để tối ưu chi phí và nâng cao doanh thu",
  "search_intent_basis": "keyword_and_source_inference",
  "brand_context": "WIFIM JSC là đơn vị 5+ năm kinh nghiệm cung cấp dịch vụ phòng marketing thuê ngoài, quảng cáo đa kênh và thiết kế web",
  "competitor_analysis": "",
  "people_also_ask": [],
  "top_urls": [],
  "related_keywords": [
    "phòng marketing thuê ngoài",
    "chi phí digital marketing",
    "quảng cáo đa kênh cho doanh nghiệp",
    "chiến lược marketing tổng thể"
  ],
  "related_keywords_source": "llm_suggestion",
  "analysis_summary": "Bài viết cần nhấn mạnh giải pháp tối ưu chi phí, đo lường hiệu quả bằng KPI rõ ràng và cam kết đồng hành cùng doanh nghiệp",
  "sources_used": [
    "https://wifim.vn"
  ],
  "source_fetch_status": "success",
  "source_warning": "",
  "research_mode": "SOURCE_PLUS_LLM_INFERENCE"
}
```

---

## 5. Hướng dẫn Kiểm thử STEP C trên Google Sheet

### Kịch bản 1: URL Fetch Thành công (Default)
1. Trong Google Sheet `Content_Tasks`, đặt dòng test:
   - `ID = 001`
   - `Keyword = Dịch vụ Digital Marketing cho doanh nghiệp`
   - `Status = PENDING`
   - `Research_Result = ` (để trống)
2. Chạy Manual Execute trên n8n.
3. **Kết quả kiểm tra**:
   - `Status = PROCESSING`
   - Cột `Research_Result` được điền chuỗi JSON với:
     - `sources_used: ["https://wifim.vn"]`
     - `source_fetch_status: "success"`
     - `research_mode: "SOURCE_PLUS_LLM_INFERENCE"`
     - `search_intent_basis: "keyword_and_source_inference"`
     - `competitor_analysis: ""`
     - `people_also_ask: []`
     - `top_urls: []`

### Kịch bản 2: URL Fetch Thất bại (Failure Fallback)
1. Mở node **`Workflow Config`**, tạm đổi `website_source_url` thành một URL không tồn tại: `https://domain-khong-ton-tai-xyz123.vn`.
2. Đặt dòng test: `ID = 001`, `Status = PENDING`.
3. Chạy Manual Execute.
4. **Kết quả kiểm tra**:
   - Workflow không bị crash.
   - Cột `Research_Result` được điền chuỗi JSON với:
     - `sources_used: []` (rỗng)
     - `source_fetch_status: "failed"`
     - `research_mode: "LLM_INFERENCE_ONLY"`
     - `search_intent_basis: "keyword_only_inference"`
     - `brand_context: ""` (rỗng)
     - `source_warning: "WIFIM source could not be fetched"`
     - `competitor_analysis: ""`
     - `people_also_ask: []`
     - `top_urls: []`
     - `search_intent` và `related_keywords` vẫn được AI suy luận từ Keyword.
