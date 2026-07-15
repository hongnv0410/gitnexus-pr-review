# gitnexus-pr-review

Plugin Claude Code: **review PR / nhánh / diff local** theo 3 tầng bằng GitNexus.
Kiến trúc tổng thể → thiết kế module/data-flow → correctness + trùng lặp. Validate chính
thiết kế, ra **verdict merge-readiness**, **read-only**. Báo cáo tiếng Việt kèm sơ đồ mermaid
và bảng findings quét nhanh.

Plugin **tự bật GitNexus MCP** khi được kích hoạt — không phải khai MCP bằng tay.

## Cần có sẵn

- **Node.js** (để chạy `npx`). GitNexus tự tải qua `npx -y gitnexus@latest`.
- **Git**.

## Cài đặt (2 bước)

```
/plugin marketplace add <owner>/gitnexus-pr-review
/plugin install gitnexus-pr-review@gitnexus-tools
```
(`<owner>` = tài khoản/GitHub org chứa repo này.)

Cài xong, khởi động lại Claude Code để MCP `gitnexus` được nạp.

## Dùng (1 lần dựng index → review thoải mái)

1. **Dựng index cho repo của bạn** (chạy 1 lần, làm mới khi code đổi nhiều):
   ```
   /gitnexus-index
   ```
   (tương đương `npx gitnexus analyze` ở gốc repo — plugin không dựng index thay bạn được
   vì nó cần chính code của bạn.)

2. **Review** — gõ tự nhiên:
   - "review PR này"
   - "soi diff nhánh này trước khi merge"
   - "review branch X so với main"

## Ghi chú

- Read-only tuyệt đối: không sửa file / commit / comment PR (trừ khi bạn chủ động chọn xuất `.md`).
- Base mặc định `main`; ngưỡng cảnh báo HIGH.
- Báo cáo bằng **tiếng Việt**.
- Index lỗi thời (code đổi nhiều) → chạy lại `/gitnexus-index`.
- Nhiều repo cùng gốc (git worktree) → khi review truyền `repo` cho tool GitNexus để tránh
  `Multiple repositories indexed`.
