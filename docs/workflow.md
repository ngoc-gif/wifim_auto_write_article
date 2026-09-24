# WIFIM Automation Workflow Documentation

## Tổng quan dự án
Proof of Concept (PoC) tự động hóa quy trình viết bài chuẩn SEO cho WIFIM chạy trên nền tảng n8n và Docker local.

---

## PHASE 3 - Minimal AI Test

### 1. Mục tiêu
Chứng minh tính khả thi của luồng sinh nội dung bài viết tối thiểu sử dụng AI:
- Kích hoạt thủ công (`Manual Trigger`)
- Cung cấp từ khóa đầu vào cố định
- Gửi yêu cầu (System Message & Prompt) tới mô hình AI
- Chuẩn hóa đầu ra (JSON: Title, Meta Description, Content bài viết đầy đủ với H2/H3, Generated At).

### 2. Kiến trúc luồng (Flow)
```text
[Manual Trigger: When clicking 'Test step']
                  ↓
[Set/Edit Fields: Set Initial Input]
  - keyword: "Dịch vụ Digital Marketing cho doanh nghiệp"
  - language: "vi"
                  ↓
[AI Engine: Basic LLM Chain] <──── [Model: OpenAI Chat Model (gpt-4o-mini)]
  - System Instructions: Chuyên gia Content Writer, không bịa số liệu/thông tin doanh nghiệp
  - Prompt: Sinh bài 700-1000 từ chuẩn cấu trúc (Title, Meta, Intro, H2/H3, Conclusion)
                  ↓
[Normalize Output: Code (JavaScript)]
  - Parse JSON & Fallback bảo vệ
  - Output fields: keyword, title, meta_description, content, generated_at
```

### 3. Thông số Input & Output
- **Input data**:
  ```json
  {
    "keyword": "Dịch vụ Digital Marketing cho doanh nghiệp",
    "language": "vi"
  }
  ```
- **Output data chuẩn hóa**:
  ```json
  {
    "keyword": "Dịch vụ Digital Marketing cho doanh nghiệp",
    "title": "Dịch Vụ Digital Marketing Cho Doanh Nghiệp: Giải Pháp Tăng Trưởng Bền Vững",
    "meta_description": "Tìm hiểu giải pháp Digital Marketing toàn diện cho doanh nghiệp giúp tối ưu chi phí và tăng trưởng doanh thu hiệu quả.",
    "content": "# ... [Nội dung markdown gồm Intro, các thẻ H2, H3 và Kết luận] ...",
    "generated_at": "2026-09-22T04:35:00.000Z"
  }
  ```

### 4. AI Provider & Model sử dụng
- **Node loại**: `@n8n/n8n-nodes-langchain.chainLlm` (version 1.5) kết hợp `@n8n/n8n-nodes-langchain.lmChatOpenAi` (version 1.2).
- **Mặc định**: OpenAI `gpt-4o-mini` (hoặc dễ dàng cắm `Google Gemini Chat Model` `gemini-1.5-flash` / `gemini-2.0-flash` bằng credential Google Gemini).
- **Lưu ý bảo mật**: Không hard-code API Key hay Credential ID giả vào workflow JSON. Credential được người dùng trực tiếp gán qua UI n8n.

### 5. Hướng dẫn Test trên Giao diện n8n
1. Truy cập giao diện n8n tại: `http://localhost:5678`
2. Tạo tài khoản Owner (nếu là lần đầu vào).
3. Import workflow:
   - Nhấn biểu tượng menu 3 chấm (góc trên bên phải) -> Chọn **Import from file...**
   - Chọn file `workflows/wifim-ai-minimal-test.json`.
4. Cấu hình Credential cho node AI:
   - Nhấp đúp vào node con **OpenAI Chat Model** (nằm bên dưới node **Basic LLM Chain**).
   - Tại mục **Credential to connect with**, chọn **Create New Credential**.
   - Dán OpenAI API Key của bạn vào ô **API Key**.
   - Bấm **Save** (Lưu) và đóng cửa sổ node.
   *(Ghi chú: Nếu sử dụng Google Gemini, chỉ cần xóa node OpenAI Chat Model và kéo node Google Gemini Chat Model vào vị trí tương ứng).*
5. Chạy thử nghiệm:
   - Bấm nút **Execute workflow** (hoặc bấm **Test step** tại node `When clicking ‘Test step’`).
   - Kiểm tra kết quả trả về tại node **Normalize Output**.

### 6. Trạng thái kiểm thử hiện tại
- **Môi trường & Schema**: PASS (Workflow JSON đã được import và validate thành công 100% trong n8n engine v1.82.2 qua CLI).
- **Thực thi AI thực tế**: Đang chờ người dùng gán API key trên giao diện n8n để hoàn tất lần chạy đầu tiên.
