---
description: Dựng / làm mới index GitNexus cho repo hiện tại (chạy 1 lần trước khi review)
---

Người dùng muốn dựng hoặc làm mới index GitNexus cho repo đang mở — điều kiện tiên quyết
để skill `gitnexus-pr-review` chạy được.

Làm theo thứ tự:

1. Kiểm đã có index chưa: nếu tồn tại `.gitnexus/run.cjs` thì chạy
   `node .gitnexus/run.cjs status` để xem index còn khớp HEAD không.
   Không có `.gitnexus/run.cjs` → đây là lần đầu.
2. Dựng / làm mới index ở gốc repo:
   - Có `.gitnexus/run.cjs`: `node .gitnexus/run.cjs analyze`
   - Chưa có: `npx gitnexus analyze`
   - Nhiều repo cùng gốc (git worktree) → thêm `--name <tên>` để giữ alias, tránh nhập nhằng
     `Multiple repositories indexed`.
3. Báo lại cho user: index đã sẵn sàng chưa, bao nhiêu symbol/flow, còn khớp HEAD không.

Sau bước này, user có thể gõ "review PR này" / "review nhánh X so với main" để chạy skill.
