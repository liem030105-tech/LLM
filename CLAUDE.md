# CLAUDE.md

Thông tin cho Claude Code khi làm việc trong repo này.

## Repository

- **GitHub:** https://github.com/liem030105-tech/LLM.git
- **Thư mục local:** `C:\Users\Admin\Desktop\ShopAsssitant`

## Project

**Shop Assistant** — trợ lý chăm sóc khách hàng cho shop online. Là project
thực hành của lộ trình 12 tuần trở thành AI Engineer (xem
`Lo-trinh-AI-Engineer-12-tuan.pdf`).

Agent làm hai việc cùng lúc:
1. **RAG** — trả lời câu hỏi từ tài liệu chính sách (đổi trả, bảo hành, vận chuyển), bắt buộc kèm trích dẫn nguồn.
2. **Tool calling** — thực hiện hành động thật trên hệ thống: tra đơn hàng, tạo yêu cầu hoàn tiền, chuyển cho nhân viên.

Kế hoạch triển khai chi tiết theo từng tuần: `KE-HOACH-CHI-TIET.md`.

## Tech stack

| Thành phần | Lựa chọn |
|---|---|
| Ngôn ngữ, framework | Python + FastAPI (async) |
| Database | PostgreSQL + pgvector |
| LLM | một LLM API (Claude / GPT / Gemini) |
| Tracing, giám sát | Langfuse |
| Triển khai | Docker, deploy cloud có link demo |
| Giao diện | trang chat đơn giản có streaming (SSE) |

## Cấu trúc thư mục

```
BE/     FastAPI backend (app/, tests/, eval/)
DB/     migrations (Alembic), seed, docs (tài liệu chính sách tiếng Việt)
FE/     trang chat có streaming
```

## Nguyên tắc quan trọng của project

- **Tự viết vòng lặp agent bằng tay**, không dùng LangChain/LangGraph cho tới
  tuần 12. Mục tiêu là hiểu cơ chế bên dưới.
- **Phân quyền ở tầng tool, không ở prompt.** Mọi truy vấn dữ liệu đơn hàng
  phải filter `WHERE user_id = :current_user` lấy từ session, không bao giờ lấy
  từ tham số do LLM sinh ra.
- **Tool không ném exception ra ngoài** — trả về
  `{"error": ..., "hint": ...}` để agent tự xử lý và hỏi lại user.
- **Hành động có hậu quả cần xác nhận 2 pha** (tạo hoàn tiền, chuyển nhân
  viên): dry-run → user xác nhận → mới thực thi.
- **Mọi cải tiến RAG phải bật/tắt được qua cấu hình** (`RAG_MODE=vector |
  hybrid | hybrid_rerank`) để đo so sánh được.
- **Không bỏ phần evaluation.** Mỗi cải tiến đều cần số liệu trước/sau trong
  README.
- Ghi lại token, chi phí và độ trễ cho mỗi lần gọi LLM.

## Quy ước

- Migration bằng **Alembic**, không dùng `create_all`.
- Script seed phải chạy lại được nhiều lần (idempotent).
- Secrets trong `.env` (không commit), kèm `.env.example`.
- Lint/format: `ruff`; type check: `mypy`.
- Trao đổi và tài liệu viết bằng **tiếng Việt**.

## Môi trường

Node 20 đã cài. Cần kiểm tra/cài khi bắt đầu: **Python 3.12**, **Docker Desktop**.
