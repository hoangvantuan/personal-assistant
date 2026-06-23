# Data Flows chi tiết (skill-only inline)

Toàn bộ luồng do **một agent** thực thi: orchestrator đọc state, rồi đọc tiếp SKILL.md phù hợp và chạy theo flow. "→ skill X" = cùng agent đọc và thực thi skill X, không spawn sub-agent.

## 1. Dashboard flow

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: /pk-harness
    H->>H: Snapshot (.cockpit/ state)
    H->>H: → pk-analyze: đọc objective/plan/actions/inbox/knowledge
    H->>H: → pk-analyze: tính metrics (execution + knowledge)
    H->>H: → pk-analyze: phát hiện issues
    H->>U: Dashboard hợp nhất (execution + knowledge) + nhắc review
```

## 2. Track light flow

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "track" / "cập nhật"
    H->>H: → pk-analyze (focus: track)
    H->>H: → pk-track light, dùng analysis
    H->>U: Dashboard + hỏi: KR? Action status?
    U->>H: Updates
    H->>U: CONFIRM bảng thay đổi
    U->>H: OK
    H->>H: Ghi SOT + archive + inbox execution
    H->>U: Tóm tắt + next action
```

## 3. Deep review flow

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "review sâu"
    H->>H: → pk-analyze deep: root cause (≥3 "tại sao?")
    H->>H: → pk-track deep: nhận analysis
    H->>U: Root cause + đề xuất
    U->>H: Feedback / chọn
    H->>U: All-changes confirm batch
    U->>H: Approve
    H->>H: → pk-track: ghi progress + log
    opt Thay đổi cấu trúc
        H->>H: → pk-init / pk-plan (pre_confirmed)
    end
    opt Phát hiện tri thức
        H->>H: → pk-capture: tạo inbox knowledge
    end
    H->>U: Tóm tắt toàn bộ
```

## 4. Capture flow

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "ghi nhanh: ý tưởng ABC"
    H->>H: → pk-capture: phân loại type + domain
    H->>U: Trình candidates
    U->>H: Duyệt
    H->>H: Ghi raw/ + inbox/
    H->>U: "Đã ghi vào inbox"
```

## 5. Knowledge distill flow

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "distill" / "đúc kết"
    H->>H: → pk-distill: đọc registries + inbox knowledge
    H->>H: → pk-distill: phân tích + tìm trùng
    H->>U: Đề xuất per item
    U->>H: Duyệt
    H->>H: Tạo/cập nhật pages + registries
    H->>H: Dọn inbox → archive
    H->>U: Báo cáo
```

## 6. Consult flow (query)

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "kho có gì về retry?"
    H->>H: → pk-consult query: grep knowledge
    H->>H: → pk-consult: tổng hợp câu trả lời + [[slug]]
    H->>H: Log usage
    H->>U: Câu trả lời + trích dẫn
```

## 7. Consult flow (run)

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "chạy skill parse-invoice"
    H->>H: → pk-consult run: load skills/parse-invoice.md
    H->>H: → pk-consult: đọc HẾT + làm theo
    H->>H: Log usage
    H->>U: Kết quả
```

## 8. Consult flow (teach)

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "soạn onboarding" / "dạy lại X"
    H->>H: → pk-consult teach: gom page theo chủ đề
    H->>H: → pk-consult: xếp biết/hiểu/làm
    H->>H: → pk-consult: soạn lộ trình (link [[slug]], không copy nguyên văn)
    H->>U: Trình lộ trình
    opt User chỉ định ghi file
        H->>H: Ghi file (NGOÀI .cockpit/)
    end
```

## 10. Reflect flow (light)

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "rút bài học"
    H->>H: → pk-reflect light: quét phiên
    H->>H: → pk-capture: trình candidates
    U->>H: Duyệt
    H->>H: → pk-capture: ghi raw/ + inbox/
    H->>U: Báo cáo + gợi ý distill
```

## 9. Reflect flow (deep AAR)

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "AAR sâu"
    H->>H: → pk-reflect deep: gọi pk-analyze lấy data
    H->>H: → pk-reflect: phỏng vấn 5 câu
    H->>U: Câu 1: Kỳ vọng?
    U->>H: Trả lời
    H->>U: Câu 2: Thực tế?
    U->>H: Trả lời
    H->>U: Câu 3-5: Nguyên nhân? Bài học? Hành động?
    U->>H: Trả lời
    H->>H: → pk-capture: tổng hợp + trình candidates
    U->>H: Duyệt
    H->>H: → pk-capture: ghi
    H->>U: Báo cáo
```

## 11. Lint flow

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "lint" / "kiểm tra kho"
    H->>H: → pk-lint check: quét 6 hạng mục
    H->>U: Report
    opt User muốn fix
        H->>H: → pk-lint fix: consolidation / rebuild / restore
        H->>U: Đề xuất
        U->>H: Duyệt
        H->>H: Thực thi + log
    end
```

## 12. Inbox routing

```mermaid
sequenceDiagram
    participant U as User
    participant H as Harness (1 agent)

    U->>H: "xử lý inbox"
    H->>H: Đếm inbox by domain
    alt Chỉ execution
        H->>H: → pk-track inbox-only
    else Chỉ knowledge
        H->>H: → pk-distill
    else Cả hai
        H->>U: "N execution + M knowledge. Bên nào trước?"
        U->>H: Chọn
        H->>H: Route theo chọn
    end
```

## 4 điểm tích hợp chính

4 điểm tích hợp dưới đây là **bản phái sinh** từ `../../pk-shared/references/cross-call-rules.md` (canonical bảng cross-call hợp lệ). Khi lệch, cross-call-rules.md thắng.

### 1. Plan → Consult (tri thức hỗ trợ lập kế hoạch)

pk-plan (lớp 2) → pk-consult (lớp 3): tìm pattern, decision, troubleshooting liên quan khi tạo action.

### 2. Track → Capture (thực thi sinh tri thức)

pk-track deep (lớp 2) → pk-capture (lớp 3): pattern/lesson mới → inbox knowledge.

### 3. Reflect → Analyze (phân tích nuôi phản tư)

pk-reflect deep (lớp 2) → pk-analyze (lớp 4): metrics, trends → dữ liệu cho AAR.

### 4. Track → Consult (skill hỗ trợ tracking)

pk-track (lớp 2) → pk-consult run (lớp 3): action có skill liên quan → thực thi.
