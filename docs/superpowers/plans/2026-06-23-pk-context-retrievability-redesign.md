# Kế hoạch tái thiết kế bộ pk-* theo lăng kính "AI luôn lấy đủ/đúng context khi thực thi"

> Trạng thái: **đã chốt thiết kế, chưa thực thi**. Bản này là output của một phiên grilling, đủ chi tiết để một phiên khác cầm là làm được.
>
> Ngày: 2026-06-23 · Nguồn: audit đa tác nhân (run `wf_c7c6b753-37a`), 98 phát hiện thô → 77 độc nhất → **65 xác nhận** (12 loại) → 8 nhóm nguyên nhân cấu trúc.

## 1. Lăng kính (ưu tiên số 1 do chủ dự án chốt)

> Mỗi khi AI thực thi một công việc, nó phải **luôn lấy được đủ và đúng context** để xử lý. Để được vậy, **cấu trúc dữ liệu + cách tổ chức phải logic** để AI dễ truy xuất. Áp cho cả 3 loại tài sản: **skill** (instructions), **knowhow/wiki** (knowledge pages), **workflow**.

Thước đo cho mọi thay đổi: *nó giúp AI lấy đủ/đúng context dễ hơn, hay làm khó hơn?*

## 2. Cái lõi (nhân gốc duy nhất)

> Hệ thống có lớp canonical tập trung tốt (`schemas.md`, `snapshot-contract.md`, `sot-ownership.md`, `cross-call-rules.md`, `metrics.md`), nhưng giả định **"một agent chạy inline luôn nhớ đủ context"** khiến **không bất biến nào được *cưỡng chế***. Đặt quy tắc ở pk-shared bị coi là đủ. *Duyên* đẩy nó vỡ: pipeline promote/evolve, teach, deep, candidate-\* được bolt-on ở các commit sau mà canonical + bản phái sinh không đồng bộ theo.

Bốn bất biến vắng mặt, lặp ở **mọi** cluster, là kim chỉ nam của toàn bộ kế hoạch:

1. **Bản phái sinh/subset phải trỏ canonical, canonical thắng.**
2. **File sống phải có format + có mặt trong snapshot.**
3. **Tham chiếu phải phân giải tất định + có chiều ngược.**
4. **Vocabulary type/mode phải ánh xạ liên-lớp + quy tắc phải tới được điểm dùng.**

## 3. Nguyên tắc quyết định (khung đã chốt)

- Đi **hướng structural** (đổi cấu trúc), không vá triệu chứng.
- **(i) Tường minh hoá hợp đồng** làm nền **bắt buộc cho cả 8 cluster**: viết ra 4 bất biến còn thiếu. Giữ kiến trúc inline-1-agent.
- **(ii) Đổi mô hình dữ liệu** chỉ áp **chọn lọc cho C1 và C2** (nơi (i) để lại no-op).
- Căng thẳng "đủ vs loãng": chốt nghiêng hẳn về **đủ**. Snapshot giữ **đồng phục, load đầy đủ** ở tầng chỉ mục/định danh; body on-demand.
- Phân loại lại: spec procedure-block + workflow body (C4 members 7-8-9) thuộc **(i)** (viết format còn thiếu), nên **làm luôn, không defer**.

## 4. Thứ tự thực thi (theo phụ thuộc, KHÔNG theo số cluster)

| Bước | Nội dung | Cluster | Phụ thuộc |
|---|---|---|---|
| 1 | Nền schema (phân vùng + format file sống) | C1 | — |
| 2 | Định danh & tham chiếu (unique-slug + redirect cấu trúc) | C3 | 1 |
| 3 | Snapshot đầy đủ + idempotency (ensure-snapshot block) | C2 + phần idempotency của C5 | 1 |
| 4 | Vocabulary & procedure block | C4 | 1, 2 |
| 5 | Quy tắc tới điểm dùng | C5 (phần còn lại) | 1, 3 |
| 6 | Chuỗi KR→action | C6 | — |
| 7 | Chống drift & định tuyến | C7 + C8 + tách period/constraints | tất cả |

---

## 5. Chi tiết từng cluster

### C1 — Nền schema (P0) · mức (i)+(ii)

**Quyết định: phân vùng sở hữu**, không hợp nhất, không giữ luật thắng-thua.
- [skills/pk-shared/references/schemas.md](../../../skills/pk-shared/references/schemas.md) = **bất biến lõi** mà *không project nào được đổi* (frontmatter bắt buộc, format inbox/action/log/objective/plan, 6 wiki type gốc, promote criteria). Read-only ở runtime.
- `SCHEMA.md` (trong `.cockpit/`) = **phần mở rộng project-specific** do evolve sinh (type mới ngoài 6, section template-hoá, glossary, ngưỡng tuỳ biến).
- Hai vùng **không giao** → bỏ nhu cầu "luật thắng-thua"; writer đọc lõi từ `schemas.md`, mở rộng từ `SCHEMA.md`; evolve chỉ ghi `SCHEMA.md` nên **có hiệu lực ngay** (khép vòng evolve hiện đang no-op).

**Thay đổi kèm (đi cùng gói):**
- Thêm mục **"schema-signals.md format"** vào `schemas.md`: bản ghi key-value cố định (loại phiếu `promote-candidate`/`query-miss`/`no-fit-type`/`adhoc-section`/`term-repeat`, slug, nguồn skill, timestamp), 2 section "Đang chờ xử lý"/"Đã xử lý". Giải [schemas.md:198-206](../../../skills/pk-shared/references/schemas.md), liên quan emit ở [pk-consult:43-47,71,149](../../../skills/pk-consult/SKILL.md), đọc ở [pk-distill:41-46,126](../../../skills/pk-distill/SKILL.md), [pk-lint:101-120](../../../skills/pk-lint/SKILL.md).
- **Gộp một bảng ngưỡng emit/act** duy nhất, sửa lệch: [pk-distill:75](../../../skills/pk-distill/SKILL.md) (`>=2 page`) vs [pk-lint:115](../../../skills/pk-lint/SKILL.md) (`>=3 page`) cho adhoc-section.
- **Cấp nguồn dữ liệu cho ngưỡng glossary**: thêm tín hiệu emit `term-repeat` (thuật-ngữ-lặp) thay vì "quét sống" không bắt được ([pk-lint:116](../../../skills/pk-lint/SKILL.md)).
- Bổ sung mệnh đề canonical: "`SCHEMA.md` = `schemas.md` + delta evolve; trong phạm vi project, vùng mở rộng do `SCHEMA.md` sở hữu."

**Đánh đổi:** vẫn còn hai file, nhưng ranh giới "không đổi được / project tuỳ biến" là một bất biến *có ý nghĩa*, dễ cho AI lẫn người biết "nhồi cái này ở đâu". Rẻ hơn hợp nhất (không bắt mỗi project mang cả tảng schema).

---

### C2 — Snapshot đầy đủ (P0) · mức (i)

**Quyết định: giữ một snapshot đồng phục, load đầy đủ.** Bỏ profile-hoá. "Đầy đủ" = đủ ở **tầng chỉ mục/định danh** (frontmatter scan + index/registry full + pinned body), **body on-demand**.

**Thay đổi:** sửa [snapshot-contract.md](../../../skills/pk-shared/references/snapshot-contract.md):
- **Bổ sung 3 nguồn sống bị sót vào preload:**
  - `schema-signals.md` ([snapshot-contract.md:20-44](../../../skills/pk-shared/references/snapshot-contract.md) thiếu hẳn) — nếu không, chạy lẻ `/pk-distill`, `/pk-lint` sẽ **đếm sót phiếu promote**.
  - Vùng mở rộng `SCHEMA.md` — cần bởi [pk-distill:75](../../../skills/pk-distill/SKILL.md), [pk-lint:97](../../../skills/pk-lint/SKILL.md).
  - `captured_at` từng item inbox ([snapshot-contract.md:26](../../../skills/pk-shared/references/snapshot-contract.md) chỉ count by domain) — nếu không, **inbox aging không chạy được**.
- Đổi bảng "Đọc state" của [pk-analyze:18-33](../../../skills/pk-analyze/SKILL.md) thành **delta** (chỉ liệt cái đọc *thêm* ngoài snapshot, chuẩn hoá nhãn "(đã có trong snapshot)") → xoá nguồn-đôi về "nạp gì".
- Thêm bất biến **"mỗi điểm GHI body phải đọc body hiện tại trước khi ghi"** — vá [pk-track:57-64](../../../skills/pk-track/SKILL.md) ghi `## Output/Deliverable` mà không đọc action cũ.

**Đánh đổi:** chấp nhận nạp thừa (vd body `tools.md` cho skill knowledge — [snapshot-contract.md:23](../../../skills/pk-shared/references/snapshot-contract.md)) để đổi lấy chắc chắn không thiếu. Body on-demand giữ snapshot không sập ở project lớn.

---

### C3 — Định danh & tham chiếu (P0) · mức (i)

**Quyết định: bất biến unique-slug** — slug duy nhất across mọi type, `[[slug]]` phân giải tất định. Không đổi cú pháp link (loại phương án `[[type/slug]]`).

| Member (locus) | Thay đổi |
|---|---|
| `[[slug]]` trần ([schemas.md:291](../../../skills/pk-shared/references/schemas.md), [pk-consult:36,78](../../../skills/pk-consult/SKILL.md)) | Ép unique-slug + thêm hạng mục pk-lint kiểm trùng slug |
| Stub redirect ở body ([pk-distill:94,130](../../../skills/pk-distill/SKILL.md)) | Nâng thành **field frontmatter** `redirect_to: [[đích]]` + `status: stub` + cột index. Snapshot (đã preload index) thấy ngay, không cần mở body. Bao cả →skill **lẫn →workflow** (vá [pk-consult:36,78](../../../skills/pk-consult/SKILL.md) chỉ nhận "Đã nâng thành skill") |
| `skills_used` / `→ Skill: [[X]]` không được lint ([pk-lint:20-23](../../../skills/pk-lint/SKILL.md)) | Thêm hạng mục pk-lint link-integrity: resolve cả hai (bắt skill con đã archive) |
| `related_inbox` cặp `both` 1 chiều ([pk-capture:42,109](../../../skills/pk-capture/SKILL.md) vs [pk-track:66-77](../../../skills/pk-track/SKILL.md)) | Hợp đồng ghi 2 chiều; track/distill **đọc lại** `related_inbox` để xử nốt item cặp |
| `resource` đổi `domain=knowledge` kẹt ([pk-track:74,77](../../../skills/pk-track/SKILL.md)) | Chỉ đổi field `domain`, giữ nguyên trong `inbox/`; distill tự nhặt khi quét `domain=knowledge` (không "di chuyển") |
| Usage nguồn-đôi ([sot-ownership.md:20](../../../skills/pk-shared/references/sot-ownership.md), [metrics.md:94-96](../../../skills/pk-shared/references/metrics.md)) | **Log là SOT**; cột Usage trong index chỉ là *cache-hint* đánh dấu "có thể cũ"; **bắt buộc recompute từ log khi duyệt promote** |
| Rewrite inbound link khi GỘP không có back-link ([pk-distill:86](../../../skills/pk-distill/SKILL.md)) | **KHÔNG thêm back-link index** (tránh đẻ bản phái sinh dễ drift — bệnh C7). Dùng **grep ngược bắt buộc** mỗi khi GỘP/deprecate + pk-lint báo link mồ côi |

**Đánh đổi:** mất tự do đặt trùng slug khác type, nhưng đó là tự do hiếm dùng và chính là nguồn nhập nhằng — bỏ là lãi ròng.

---

### C4 — Vocabulary & procedure block (P1) · mức (i)

**Quyết định 1: hai đường tạo skill có kiểm soát.**
- *Đường phiếu* (`promote-candidate`) = nâng cấp page **đã có**, đã chứng minh qua usage.
- *Đường candidate* (`candidate-skill`/`candidate-workflow`) = ý định tạo mới chủ động → distill **tạo trực tiếp** (qua bước user duyệt) **chỉ khi đủ** `trigger`/`input`/`output`; **thiếu thì tự hạ xuống wiki pattern page** chờ chín. Mô tả ranh giới hai đường rõ trong `schemas.md`. Giải [pk-distill:23-29,55,83](../../../skills/pk-distill/SKILL.md).

**Quyết định 2: một procedure block canonical, dùng chung.**
- Định nghĩa trong `schemas.md`: khối bước chuẩn (bước tuần tự, điều kiện rẽ nhánh, marker `→ Skill: [[X]]`).
- Là **body bắt buộc của workflow** ([schemas.md:219-248](../../../skills/pk-shared/references/schemas.md) hiện chỉ có frontmatter) và **được phép xuất hiện trong `## Cách dùng`** của pattern/troubleshooting/lesson.
- Ràng buộc: **chỉ Run/promote được thứ CÓ procedure block**; thiếu thì [pk-consult Run](../../../skills/pk-consult/SKILL.md) báo "mới mô tả, chưa có quy trình" thay vì bịa ([schemas.md:176-190](../../../skills/pk-shared/references/schemas.md)).
- **Đứt mạch**: gặp `[[X]]` không tồn tại/đã archive hoặc bước lỗi → **dừng, báo bước lỗi** (mượn error-handling harness). Giải [pk-consult:80-95](../../../skills/pk-consult/SKILL.md).

**Gói (i) còn lại:**
- `thought` → distill **re-classify** vào 1 trong 6 type hoặc discard ([pk-distill:55](../../../skills/pk-distill/SKILL.md)).
- `domain=both` thêm **tiêu chí phát hiện** (có động-từ-hành-động-có-output **và** tri-thức-tái-dùng) ([pk-capture:34-42](../../../skills/pk-capture/SKILL.md)).
- reflect Bước 4 **trỏ về `schemas.md`** là nguồn type duy nhất, bỏ liệt song song (vá sót `resource`/`blocker`) ([pk-reflect:67-70](../../../skills/pk-reflect/SKILL.md)).
- `deep` khai báo thành **param độc lập** `mode: light|deep` (tách khỏi `focus`) ([pk-analyze:100-108](../../../skills/pk-analyze/SKILL.md)).
- Tách `period` → `start_date`+`end_date` ([metrics.md:20-21](../../../skills/pk-shared/references/metrics.md) vs [schemas.md:39](../../../skills/pk-shared/references/schemas.md)) và `constraints` → `capacity`/`budget`/`gaps_risks` ([pk-init:72](../../../skills/pk-init/SKILL.md)). **Xếp P2 vì cần migrate.**

---

### C5 — Quy tắc tới điểm dùng (P1) · mức (i)

**Quyết định 1: Quality Gate gắn vào "cổng ghi SOT", không phải "điểm nhận input".**
- Sửa [quality-gate.md:1,6](../../../skills/pk-shared/references/quality-gate.md) từ hard-code "pk-init + pk-plan" thành **"mọi skill ghi SOT từ input user: pk-init, pk-plan, pk-track"** (track ghi KR current — số liệu rủi ro nhất).
- Capture/reflect chỉ ghi inbox (staging) → **không bắt buộc** QG đầy đủ; QG áp khi item vào SOT (lúc track/distill xử).

**Quyết định 2: ensure-snapshot block + marker.**
- Mỗi SKILL.md mở đầu bằng **block chuẩn bắt buộc**: (1) precondition `.cockpit/` tồn tại; (2) "nếu phiên chưa có marker `SNAPSHOT_LOADED` → tự nạp full theo Snapshot Contract; có rồi → skip". Harness Phase 1 đặt marker.
- Biến idempotency từ "agent tự nhớ" → **kiểm được bằng marker**. Giải member 1, 5, 6: [pk-reflect](../../../skills/pk-reflect/SKILL.md) (thiếu precondition + xử lý `deep degraded`), [pk-track inbox-only:113-121](../../../skills/pk-track/SKILL.md).

**Gói (i) còn lại:**
- Định nghĩa payload `pre_confirmed` cho pk-init/pk-plan để biết bỏ qua confirm → giải mâu thuẫn [pk-init:32,106](../../../skills/pk-init/SKILL.md) "BẮT BUỘC confirm" vs [cross-call-rules.md:42-60](../../../skills/pk-shared/references/cross-call-rules.md). Thêm `pre_confirmed` vào [pk-track light Phase 5:72](../../../skills/pk-track/SKILL.md) (lệch inbox-only). Định rõ ánh xạ payload→field action ở [pk-plan:71-76](../../../skills/pk-plan/SKILL.md).
- reflect deep chỉ định `mode/focus` khi gọi pk-analyze ([pk-reflect:45](../../../skills/pk-reflect/SKILL.md)).
- pk-plan tham chiếu [metrics.md:98-107](../../../skills/pk-shared/references/metrics.md) để check capacity/xung đột.
- distill nói rõ "đề xuất hành động hệ thống" bàn giao về đâu ([pk-distill:61](../../../skills/pk-distill/SKILL.md)).
- Sửa 2 viện dẫn ma: [pk-lint:79](../../../skills/pk-lint/SKILL.md) (`frontmatter usage` không tồn tại); [pk-analyze:50](../../../skills/pk-analyze/SKILL.md) viện `metrics.md` cho "index lệch" (thuộc pk-lint). Trỏ Reachability audit [pk-lint:38-39](../../../skills/pk-lint/SKILL.md) về bảng neo SOT [snapshot-contract.md:56-68](../../../skills/pk-shared/references/snapshot-contract.md).

---

### C6 — Chuỗi KR→action (P1) · mức (i)

- Thêm `key_result` + `milestone` vào **bảng SOT pk-plan** ([pk-plan:23](../../../skills/pk-plan/SKILL.md)) *và* [sot-ownership.md:13](../../../skills/pk-shared/references/sot-ownership.md) canonical; flow `mode new` ([pk-plan:53-60](../../../skills/pk-plan/SKILL.md)) thêm bước **gắn action→KR**; pk-lint thêm chốt chặn **báo orphan-action** (action thiếu `key_result`).
- pk-track **deep** ghi `last_review_date` ([pk-track:97-99](../../../skills/pk-track/SKILL.md) hiện tái dùng "giống light Phase 4" mà light cố ý không đụng field này → không phase nào ghi → "nhắc review" sai vĩnh viễn).
- Chuyển quy tắc cứng "verify sequence action ID" từ [pk-track:135](../../../skills/pk-track/SKILL.md) sang **pk-plan** (nơi sinh ID).

---

### C7 — Chống drift (P2, có 1 lỗi P1-ẩn) · mức (i)+(ii)\*

- **Sửa ngay (P1-ẩn):** [README.md:64](../../../README.md) ghi "3+ lần sử dụng được thăng cấp" — **trái canonical** ([schemas.md:198-206](../../../skills/pk-shared/references/schemas.md): trigger là 3 phiếu, `usage_count` KHÔNG phải trigger).
- Bổ sung bản đồ kiến trúc README thiếu file ([README.md:22-36](../../../README.md): thiếu `skills/registry.md`, `workflows/registry.md`, `SCHEMA.md`, `schema-signals.md`).
- **Gắn mệnh đề canonical** ("bản phái sinh, canonical ở X, canonical thắng") vào: [flows.md "4 điểm tích hợp":197-213](../../../skills/pk-harness/references/flows.md), bảng consumer [pk-shared/SKILL.md:14-19](../../../skills/pk-shared/SKILL.md). Bổ sung flow teach còn thiếu trong flows.md.
- **Auto-gen (ii\*):** mở rộng pk-lint `rebuild-index` để **tái sinh** bảng consumer + registry (grep ngược ra được). Prose (README, flows.md Mermaid) → **không** auto-gen, chỉ trỏ canonical + đưa vào **checklist pk-lint thủ công**.

---

### C8 — Định tuyến (P2) · mức (i)

- **Harness chỉ route, không chép flow:** [pk-harness Phase 2:104-111](../../../skills/pk-harness/SKILL.md) bỏ chép thứ tự + delegate của [pk-track deep:83-112](../../../skills/pk-track/SKILL.md); chỉ ghi "→ pk-track deep (theo flow của nó)".
- **Description mang tín hiệu phân biệt** + mệnh đề "KHÔNG dùng khi…":
  - "bài học" → phạm vi nguồn + giai đoạn: capture (1 mẩu) / reflect (cả phiên) / distill (cả inbox).
  - "xử lý inbox" → domain: track (execution) / distill (knowledge).
  - "cập nhật KR" → target (pk-init) vs current (pk-track).
  - reflect light vs capture mặc định: phân ranh "quét toàn bộ hội thoại" ([pk-reflect:25-31](../../../skills/pk-reflect/SKILL.md) vs [pk-capture:46-49](../../../skills/pk-capture/SKILL.md)).
- Routing inbox bổ sung **case "cả hai"** ([pk-harness:68,83-89](../../../skills/pk-harness/SKILL.md)).
- **Loại pk-shared khỏi callee** trong [cross-call-rules.md:13](../../../skills/pk-shared/references/cross-call-rules.md): nó là thư viện, "chỉ đọc", cấm gọi.

---

## 6. Phụ lục — 12 finding đã LOẠI (đã cân nhắc, không nằm trong kế hoạch)

Phản biện độc lập bác vì xuyên tạc nội dung file hoặc hiểu nhầm thiết kế có chủ ý:

- "pk-analyze vs pk-track chồng lấn 'review sâu'": description đã ghi rõ ranh giới read-only.
- "Bảng phân luồng domain thiếu nhánh thought/resource khi chọn knowledge": đây là **thiết kế 2 tầng có chủ đích** (type ở capture, hỏi domain), không phải gap.
- "Enum lặp nhiều nơi không trỏ enum": tiền đề sai (đã có canonical).
- "Flow Capture thiếu nhánh batch-ingest": flow 4 vẽ đúng phạm vi ghi-nhanh.
- "focus=track bỏ KR at-risk": văn bản đúng như thiết kế.
- "capture Bước 4 không liệt field bắt buộc": đã trỏ schema.
- "capture Bước 2 chỉ 3 emoji mẫu": ví dụ minh hoạ, không phải spec.
- "Teach thiếu cổng promote-candidate": có chủ đích (teach chỉ emit query-miss).
- "Description pk-lint thiếu trigger evolve": thực ra đã có "schema cũ".
- "Workflows registry chỉ metadata": đúng vai registry (body ở file workflow).
- "promote/chạy skill chồng lấn distill vs consult": dữ kiện đúng nhưng không gây route sai thực.

## 7. Ghi chú vận hành

> ⚠️ **Hai lưu ý bắt buộc đọc trước khi bắt đầu thực thi:**
>
> 1. **Sửa canonical trước, subset theo sau.** Luôn sửa `schemas.md` + `snapshot-contract.md` + `sot-ownership.md` (lớp canonical) TRƯỚC, rồi mới cập nhật subset tương ứng trong từng SKILL.md. Làm ngược lại sẽ tự sinh nguồn-đôi mới — đúng bệnh mà kế hoạch này đang chữa.
> 2. **Lỗi C7 phải sửa NGAY, bất kể tiến độ các bước khác.** [README.md:64](../../../README.md) đang ghi "promote = 3+ lần sử dụng được thăng cấp", **trái canonical** ([schemas.md:198-206](../../../skills/pk-shared/references/schemas.md): trigger là 3 phiếu `promote-candidate`, `usage_count` KHÔNG phải trigger). Đây là cửa vào tài liệu: AI đọc README làm context sẽ promote sai thời điểm. Không cần chờ tới bước 7 — sửa trước.

- **Migrate cần lưu ý** (thay đổi format dữ liệu đang tồn tại): unique-slug (C3), `redirect_to` field (C3), tách `period`/`constraints` (C4), thêm `key_result`/`milestone` (C6). Nên có 1 pass `pk-lint` migrate sau khi sửa skill.
- **Reversibility**: mọi thay đổi đều ở tầng skill/schema (markdown), không hard-delete dữ liệu `.cockpit/`. Archive trước khi đổi format.
- **Verify mỗi bước**: sau mỗi bước trong §4, chạy `pk-lint check` trên một `.cockpit/` mẫu để bắt link hỏng / registry lệch trước khi sang bước sau.
- Khi thực thi: cân nhắc fan-out theo file (mỗi tác nhân một SKILL.md/reference) nhưng **sửa `schemas.md` + `snapshot-contract.md` + `sot-ownership.md` trước** (canonical), rồi mới sửa subset trong từng SKILL.md theo sau — đúng nguyên tắc "sửa canonical trước, subset theo sau".
