---
name: gitnexus-pr-review
description: "Use when the user wants to review a PR / branch / local diff on this repo with GitNexus. Reviews top-down in 3 layers (architecture conformance → module/data-flow design → low-level correctness + duplication), validates the design itself (not just doc), scores findings by confidence×impact to cut noise, and supports parallel-subagent swarm mode for big diffs. Read-only; báo cáo tiếng Việt; ends with 3 verdict (Merge State / Branch Hygiene / Final). Examples: \"review PR này\", \"soi diff nhánh này trước khi merge\", \"review branch X so với main\"."
---

# PR Review với GitNexus — 3 tầng, top-down

Review một PR / nhánh / diff **như senior thật**: đi từ trên xuống qua **3 tầng**, **validate cả chính thiết
kế** (doc không phải chân lý), chấm điểm để lọc nhiễu, kết luận bằng **verdict merge-readiness**. **Báo cáo
tiếng Việt.**

> **READ-ONLY tuyệt đối.** Chỉ dùng lệnh đọc: `git log/diff/show/grep/ls-files`, `gh pr view/diff/checks`,
> `grep/cat/find/ls`, và tool GitNexus (đều read-only). KHÔNG sửa file / commit / checkout / cài package.
> Áp cho **mọi subagent** ở Swarm — nhắc lại trong prompt giao việc.
> Ngoại lệ (chỉ khi user chọn ở Q4): "xuất `.md`" (ghi 1 file) hoặc "Comment PR" (`gh pr comment` —
> **hiện nháp cho user duyệt TRƯỚC**, không tự đăng).

## 3 tầng

| Tầng | Câu hỏi cốt lõi | Persona | Công cụ chính |
|---|---|---|---|
| **1. Kiến trúc** | Ăn khớp kiến trúc & data-flow tổng thể? Sinh module/tầng lệch? **Doc còn khớp code?** | Architect | docs + `clusters` + `processes` + `impact` |
| **2. Module/luồng** | Component đúng pattern hiện có? Data flow đi chuẩn? (DB nếu có) | FE (component) · BE (data flow) | `context` + `process/{name}` + đọc pattern anh em |
| **3. Low-level** | Hàm ổn? **Trùng lặp/thừa** với code đã có? Correctness / leak / perf | Senior dev | `query` + `context` + `cypher` + đọc diff |

> **Tầng trên sai → tầng dưới đúng cũng vô nghĩa** (hàm hoàn hảo đặt sai tầng vẫn là nợ). Review từ tầng 1
> xuống; đừng nhảy thẳng vào correctness. Chuẩn "đúng" đọc từ **base `main`** + `CLAUDE.md`/docs, **không** từ
> nhánh PR (PR không tự định chuẩn của nó). Finding lệch chuẩn → nhãn `[lệch quy ước]`.

## Quy tắc GitNexus (đọc trước)

- **Định danh repo:** `list_repos({})` → 1 index thì `repo` tùy chọn; nhiều index cùng gốc thì **truyền `repo`
  cho MỌI tool**. Lỗi `Multiple repositories indexed` / `Repository not found` → dùng **path đầy đủ** tới gốc repo.
- **Resource nhận TÊN, tool nhận PATH:** khi alias mất, tool (`detect_changes`/`query`/`context`/`impact`/
  `cypher`) vẫn chạy với path; nhưng resource `gitnexus://repo/<name>/clusters|processes|context` **chết**.
  Muốn clusters/processes → khôi phục alias TRƯỚC (`node .gitnexus/run.cjs analyze --name <repo>`).
- **No silent source-swap (BẮT BUỘC):** chỉ dán nhãn "GitNexus" khi resource/tool **thực sự trả về**. Dựng
  hình từ git-tree/grep/doc → ghi ĐÚNG nguồn đó. Mỗi artifact (sơ đồ · bảng · finding) mang nguồn thật.
- **`impact(upstream)` báo THIẾU** (chỉ đi cạnh Function/Method). `upstream=0` ≠ an toàn — đối chiếu
  `context({name})` (`incoming.calls`) hoặc `grep` trước khi kết luận LOW.
- **`query` chạy tốt** (BM25+vector) → **nguồn chính** tìm symbol/trùng lặp theo khái niệm. Nó có thể trả
  `processes` rỗng → muốn hiểu LUỒNG thì dùng `processes`/`context`.

## Phase 0 — Tự kiểm tiền đề (KHÔNG hỏi user)

```
1. node .gitnexus/run.cjs status      → index khớp HEAD? (stale → cảnh báo, đề nghị analyze)
2. list_repos({})                     → path index đúng; có PDG chưa; embeddings
2b. thử đọc gitnexus://repo/<repo>/context → "not found" = alias mất (xem quy tắc resource)
3. explain({limit:1})                 → "no taint layer" = KHÔNG có PDG → skip security-taint
4. git rev-parse --abbrev-ref HEAD    → nhánh hiện tại (đối tượng mặc định)
5. git rev-list --left-right --count main...HEAD  → ahead/behind (phạm vi)
6. gh pr view <branch> / git log      → có PR/ticket? lấy mô tả → ticket-compliance
7. gh pr view --json state,mergeable,mergeStateStatus,isDraft + gh pr checks → Merge State + CI
8. git log main..HEAD --oneline       → Branch Hygiene: có merge-from-main / churn lạc đề?
9. grep migrations/schema/*.sql/ORM   → có DB không? không → skip tầng DB
```

Suy ra được base (`main`) + đối tượng → **đừng hỏi lại**. Không PDG/DB → skip & ghi rõ. Không thấy state PR/CI
(không có gh) → Merge State = "visibility incomplete", đẩy sang "Cần verify", KHÔNG đoán mergeable. Diff gộp
nhiều theme / merge-from-main → cờ decomposition + hygiene, đề xuất rebase/split.

## Phase 1 — Hỏi user (AskUserQuestion, gộp 1 lần, ≤4 câu, bỏ câu tự biết)

- **Q1. Đối tượng** *(chỉ nếu không chắc)* — Nhánh vs `main` *(Rec)* · Commit/range · PR số · Diff local.
- **Q2. Tầng & lens** *(multiSelect)* — Kiến trúc *(Rec)* · Tổ chức component · **Trùng lặp/thừa** *(Rec)* ·
  Correctness *(Rec)* · Regression *(Rec)* · Coupling · Performance · Security-taint *(chỉ nếu có PDG)*.
- **Q3. Validate CHÍNH thiết kế?** — Có: đối chiếu doc↔code + phản biện thiết kế *(Rec nếu chạm kiến trúc/BE)*
  · Không: chỉ dùng thiết kế làm chuẩn.
- **Q4. Độ sâu + Đầu ra** — Độ sâu (quyết định đọc code tới đâu, xem bảng Phase 2): **Nhanh** · **Vừa** *(Rec)*
  · **Sâu**. Ra *(multiSelect)*: Báo cáo 3 tầng *(Rec)* · Tóm tắt · Chi tiết+trace · Xuất `.md` · **Comment PR**.

## Phase 1.5 — Describe + sơ đồ *(BẮT BUỘC nếu diff > ~20 file hoặc mixed; nhỏ thì rút gọn)*

Dựng "bức tranh PR" trước khi soi — **người review đọc SƠ ĐỒ trước, chữ sau**. Cần ≥1 sơ đồ kiến trúc + ≥1 sơ đồ luồng.
- **PR type**: feat/fix/refactor/docs/mixed (mixed → cờ decomposition). **Review effort 1–5**.
- **Walkthrough**: bảng `file/cụm → tóm tắt` (tách docs khỏi code).
- **Sơ đồ 1 — kiến trúc & vùng đụng** (`flowchart`) từ `clusters`; **đánh dấu** node/cạnh diff chạm.
- **Sơ đồ 2 — luồng bị đụng** (`sequenceDiagram`) từ `process/{tên}`; đánh dấu bước qua symbol đã đổi.
- ⚠️ `clusters`/`process` gọi hụt → dựng từ git-tree/doc và **ghi rõ nguồn ≠ GitNexus** (no silent swap).

````
```mermaid
flowchart TD
  subgraph EXT["Extension"]; B[Manager]; C[Bridge]; end
  subgraph CORE["Core (KHÔNG đụng)"]; D[(Service)]; end
  B --> C; B -.command.-> D
  classDef touched fill:#fde,stroke:#c39,stroke-width:2px; class B,C touched
```
````

## Phase 2 — Chạy review 3 tầng

### Độ sâu điều khiển "đọc code tới đâu" (GitNexus-first)

GitNexus luôn là nguồn *điều hướng/đồ thị*; đọc code chỉ để *xác nhận đúng/sai trên dòng đã đổi*.

| Độ sâu | Đọc dòng code (delta)? |
|---|---|
| **Nhanh** | ❌ Không hunk. Chạy 2A đầy đủ, 2B mức flow-metadata, **BỎ** 2C correctness |
| **Vừa** *(mặc định)* | ⚠️ Chỉ hunk được flag (symbol MED+): `context(include_content)` → `git diff` hunk. KHÔNG cả file |
| **Sâu** | ✅ Đọc full delta mọi symbol bị đụng trong worklist |

> **Cứng (mọi mức):** worklist = symbol/flow do `detect_changes` trả về. KHÔNG mở file ngoài worklist, KHÔNG
> đọc toàn file. Xem code → `context({name, include_content:true})` TRƯỚC, rồi `git diff <base>..HEAD -- <file>`
> đúng **hunk** (chỉ để thấy *delta*).

### Chế độ: Solo (mặc định) vs Swarm

- **Solo** *(diff nhỏ/vừa)*: chạy tuần tự 2A → 2B → 2C trong 1 context. Ít token.
- **Swarm** *(diff lớn / nhiều tầng / chọn "Sâu")*: fan-out subagent song song rồi tổng hợp. Thứ tự:
  **facts trước → phân tích song song → synthesis-critic gate cuối**.

```
Lane A (facts+hygiene, nhẹ): gom PR facts + branch hygiene + merge state   [Phase 0 đã có]
   ↓ (xong A mới đủ ngữ cảnh)
── song song ── Lane 1: 2A kiến trúc+validate · Lane 2: 2B module/data-flow ·
   Lane 3: 2C low-level+trùng lặp · Lane 4: test/CI (nhẹ) · Lane 5: security (nếu có PDG)
── gate ── Lane 7 (synthesis-critic): GOM → dedup → xử mâu thuẫn → chấm điểm → 3 verdict. BẮT BUỘC.
```
Giao mỗi lane bằng Agent tool, prompt nhắc: **READ-ONLY**, truyền `repo`, trả finding theo format chuẩn
(Risk/Evidence/Fix/Blocks-merge). Lane checklist → tier nhẹ; lane phân tích → tier nặng. **Lane 7 không được bỏ.**

### 2A · Tầng 1 — Kiến trúc + validate thiết kế *(mọi độ sâu — chỉ GitNexus, không hunk)*
```
1. Nạp "chuẩn kiến trúc" từ base `main`: docs (docs/*, ADR, CLAUDE.md) + clusters + processes.
2. VALIDATE thiết kế (nếu Q3=Có): so doc ↔ clusters/processes — doc còn khớp code? tầng nào doc nói có mà
   code không (hoặc ngược)? → comment VỀ CHÍNH thiết kế. Cần sâu → giao subagent architecture-designer.
3. ĐỐI CHIẾU diff: symbol đổi rơi vào cluster nào? file MỚI không thuộc cluster nào → nghi lệch tầng.
   Import/gọi VƯỢT TẦNG? lần bằng context(incoming/outgoing.calls) hoặc cypher; grep chỉ XÁC NHẬN.
   Sinh "module song song" trùng chức năng tầng đã có → chuyển 2C.
   → Verdict tầng 1: ĂN KHỚP / LỆCH (+ điểm lệch cụ thể).
```

### 2B · Tầng 2 — Module / component / data flow
```
1. detect_changes({scope:"compare", base_ref, repo}) → symbol đổi + flow bị đụng (TÁCH docs khỏi code).
   Output lớn → giao subagent đọc file kết quả, giữ context chính gọn.
2. Component (FE): thành phần MỚI theo pattern anh em? (props/handler/command wiring, tách file, đặt tên).
   Lấy thành phần cùng loại qua context/cypher để so; grep chỉ XÁC NHẬN contract giữa tầng.
3. Data flow (BE): READ process/{tên} cho flow bị đụng → bước nào qua symbol đã đổi; đúng thứ tự/tầng?
   state reset đúng ở ranh giới (teardown, đổi ngữ cảnh/phiên)?
4. DB (nếu có): schema/migration/index/transaction vs data flow.
[Nhanh: chỉ metadata. Vừa: dynamic context — đọc hàm/class bao ngoài hunk MED+, soi state TRƯỚC đổi. Sâu: full delta.]
```

### 2C · Tầng 3 — Low-level: correctness + TRÙNG LẶP + leak/perf *(bước 2–3 chỉ Vừa/Sâu; Nhanh BỎ)*
```
1. TRÙNG LẶP/THỪA — MỌI độ sâu (dùng index, không đọc dòng): mỗi hàm/util MỚI → query({search_query:"<chức
   năng>"}) TRƯỚC → có candidate thì context({name}) xác nhận → đề xuất TÁI DÙNG, KHÔNG viết mới. Rỗng → mới là mới.
2. Correctness (Vừa/Sâu): null/undefined deref, nhánh sai, off-by-one, race, missed await, state không reset.
3. Leak/perf (Vừa/Sâu): listener/timer/disposer dọn trong dispose/teardown? throttle/debounce đúng? hot path chậm thêm?
4. impact({target, direction:"upstream", repo}) → blast radius. upstream=0 PHẢI đối chiếu (context/grep).
5. Security taint (nếu chọn & có PDG): explain({target, repo}) → source→sink.
```

## Phase 3 — Synthesis gate → chấm điểm → verdict → báo cáo

**Synthesis-critic gate (bắt buộc, trước khi viết; ở Swarm là Lane 7):**
1. **Gom + dedup**: trùng file:line / cùng nguyên nhân → gộp 1.
2. **Xử mâu thuẫn lane**: ưu tiên cái có evidence code cụ thể; chưa ngã ngũ → "Cần verify", không chọn bừa.

**Chấm điểm để lọc nhiễu — Confidence × Impact (càng nhẹ càng phải chắc):**
Mỗi finding có Confidence (chắc tới đâu) và Impact (hại tới đâu). Chỉ báo cáo finding **vượt ngưỡng**:

| Impact | Ngưỡng confidence để BÁO |
|---|---|
| **Critical** (chặn merge · lệch kiến trúc · security · hot path/auth/thanh toán) | ≥ 50% |
| **High** | ≥ 70% |
| **Medium** | ≥ 85% |
| **Low / nit** | ≥ 95% |

Dưới ngưỡng nhưng còn nghi → **"Cần verify"** kèm lệnh kiểm cụ thể (grep gì / test nào / đọc file nào), KHÔNG
kết luận. Bỏ hẳn finding chỉ dựa `upstream`/suy đoán chưa đối chiếu code.

**Risk (theo CODE, đã tách docs):** <5 symbol → LOW · 5–15 symbol/2–5 flow → MEDIUM · >15 symbol/nhiều flow →
HIGH · hot path/session/auth/thanh toán **hoặc lệch kiến trúc tầng 1** → CRITICAL.

**3 verdict cứng (mỗi cái đúng 1 nhãn):**

| Verdict | Nhãn hợp lệ |
|---|---|
| **Merge State** | mergeable · blocked by conflicts · checks pending/failing · review blocked · draft/WIP · merged · closed · visibility incomplete |
| **Branch Hygiene** | clean feature/fix · merge-from-main harmless · polluted by unrelated merge/churn · rebase/split required |
| **Final Verdict** | production-ready · production-ready w/ minor follow-ups · not production-ready · rebase/split required before review |

> 1 finding chặn merge (CRITICAL / lệch kiến trúc / security) → **Final KHÔNG thể production-ready.** Cảnh báo
> RÕ nếu HIGH/CRITICAL hoặc lệch kiến trúc trước khi nói "ổn để merge".

### Khung báo cáo (tiếng Việt) — sơ đồ trước, bảng quét-nhanh, rồi card MED+
```
## Review: <đối tượng> vs <base>
> Read-only · Base=`<base>` · Chế độ: Solo/Swarm · Ngày: <YYYY-MM-DD>
**Verdict:** Final: <…> · Merge State: <…> · Branch Hygiene: <…>
**Ticket-compliance:** Fully/Partially/Not/Code-Verified — <lý do> (bỏ nếu không ticket)
**Tóm tắt:** <2–3 câu: ăn khớp kiến trúc? blocker? rủi ro chính?> · PR type: <…> · Effort: <1–5>

### Sơ đồ (đọc trước)   — flowchart kiến trúc-vùng-đụng + sequenceDiagram luồng chính
### Phạm vi đã đo        — bảng: commit · file · code vs docs · file core bị đụng · PDG? · DB?
### Findings — bảng quét nhanh
| ID | Tầng | Risk | Blocks | Vị trí | Vấn đề (1 dòng) | Conf×Impact |
(sắp cao→thấp; LOW cũng 1 dòng ở đây, không cần card riêng)

### Tầng 1 — Kiến trúc   — Verdict ĂN KHỚP/LỆCH + nhận xét VỀ CHÍNH thiết kế (nếu validate)
### Tầng 2 — Module/data flow  — bảng: Vùng | Symbol/flow đụng | Nhận xét pattern/luồng | Risk
### Tầng 3 — Chi tiết (chỉ MED+)  — Trùng lặp/thừa + card cố định mỗi finding:
  **F1 — <tiêu đề>** · [MED][Blocks: maybe] · `file.ts:223` · `[lệch quy ước]`(nếu có)
  - **Hiện tượng:** <cái gì sai/thiếu, quan sát được>
  - **Vì sao:** <cơ chế 1 câu, dẫn symbol/flow bị đụng>
  - **Fix:** <việc cụ thể, làm được ngay>

### Cần verify           — <nghi ngờ> → kiểm bằng: <grep/test/đọc file cụ thể>
### Regression cần test  — <execution flow> — vì <lý do>
### Đề xuất tách PR/rebase (nếu gộp theme/churn)   ### Cảnh báo (HIGH/CRITICAL · lệch KT · stale · PDG/DB skip · upstream thiếu · merge-state chưa thấy)
### Nguồn GitNexus thực dùng (BẮT BUỘC — chống overclaim)
  | Tool/Resource | Chạy? ✅/⚠️/❌ | Đóng góp thật vào kết luận |
  (ghi thẳng cái gọi hụt; evidence từ git/grep/doc → nói RÕ, đừng gán GitNexus)
```

### Đầu ra tuỳ chọn — Comment PR *(WRITE — chốt trước khi đăng)*

Chỉ khi user chọn ở Q4. **Rút gọn** báo cáo thành 1 comment quét được trong 30s (không dán cả báo cáo dài).
Nội dung lấy từ synthesis-gate đã chốt — chỉ tái trình bày, đừng review lại.
- Chỉ đăng nếu có PR thật (`gh pr view` ra số). Không có PR → đề nghị xuất `.md`.
- `gh pr comment` là lệnh GHI → **hiện nháp cho user duyệt TRƯỚC**; chỉ `gh pr comment <num> --body-file <path>`
  sau khi user OK. Tuyệt đối không tự đăng.

```
## 🔍 Review tóm tắt — <đối tượng> vs <base>
**Verdict:** Final: <…> · Merge State: <…> · Branch Hygiene: <…> · Ticket: <Fully/Partially/Not>
### ✅ Đã đạt được   — Tầng-1 ĂN KHỚP + ticket Fully + vùng "SẠCH" đã trace
### ❌ Chưa đạt được  — mọi finding Blocks≠no + ticket Partially/Not + "Cần verify" + regression chưa test — [risk][Blocks]
### 🛠 Đề xuất (cao→thấp)  — `file.ts:line` — <fix cụ thể> (Fxx) + tách PR/rebase nếu có
<details><summary>Chi tiết đầy đủ</summary> (dán bảng findings quét-nhanh, hoặc link báo cáo .md) </details>
```
"Chưa đạt được" rỗng → ghi thẳng: "Không có mục chặn — sẵn sàng merge (kèm follow-up nếu có)".
