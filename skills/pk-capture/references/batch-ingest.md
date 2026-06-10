# Batch Ingest: Capture nguồn lớn

Reference cho pk-capture chế độ "Từ nguồn ngoài" khi nguồn vượt cỡ một lần đọc thường: meeting transcript, export chat, loạt issue/PR, postmortem, tài liệu dài.

## Khi nào dùng batch

- Một file > 500 dòng, HOẶC >= 3 file trong cùng một lần capture.
- Dưới ngưỡng: dùng flow capture thường.

## Bất biến giữ nguyên

Batch KHÔNG phá bất biến nào: vẫn chỉ ghi `inbox/`, vẫn user duyệt trước khi ghi. Khác biệt duy nhất: đọc cuốn chiếu và duyệt theo nhóm.

## Bước 1: Lưu raw TRƯỚC khi đọc

Copy nguyên văn nguồn vào `raw/YYYY-MM-DD-<slug-nguồn>.md`. Một file nguồn = một file raw. Mọi inbox item sinh từ nguồn này trỏ chung `source_file` về file raw đó.

## Bước 2: Đọc theo bản đồ loại nguồn

| Nguồn | Ưu tiên tìm | Bỏ qua |
|---|---|---|
| Meeting transcript | decision, concept, lesson | action item thuần, small talk |
| Chat / Slack export | troubleshooting, pattern, decision | tin điều phối, link không ngữ cảnh |
| Issue / PR / postmortem | troubleshooting, lesson, decision | diễn biến comment |
| Tài liệu / báo cáo | concept, pattern, decision | số liệu chỉ đúng một thời điểm |

## Bước 3: Đọc cuốn chiếu, gom xong mới trình

- Nguồn dài: đọc theo đoạn, gom candidates vào MỘT danh sách. KHÔNG trình từng đoạn.
- Dedupe trong batch: cùng quyết định nhắc 3 lần = 1 candidate.
- Candidate có vẻ trùng tri thức đã có: VẪN GIỮ, đánh dấu "(có thể trùng [[slug]], distill sẽ phân xử)".

## Bước 4: Trình duyệt theo nhóm

- Nhóm theo type, mỗi nhóm tối đa ~10 item.
- User thao tác theo nhóm: "giữ cả nhóm", "bỏ cả nhóm", "giữ trừ item N".
- Trên 30 candidates: trình đợt 1 gồm loại giá trị cao (decision, lesson, troubleshooting), hỏi user có xem tiếp đợt 2 (concept, pattern, candidate-skill/workflow) không.

## Bước 5: Ghi inbox + log

- Mỗi item: `inbox/YYYY-MM-DD-HHmm-slug.md`, `source_file` trỏ file raw chung.
- Log một dòng cho cả batch:
  ```
  Capture batch <tên nguồn>: N items (M decision, K lesson, ...)
  ```
- Nguồn chia nhiều phiên: ghi rõ phạm vi đã xử lý ("phần 1/2, đến dòng 800") để phiên sau tiếp đúng chỗ.
