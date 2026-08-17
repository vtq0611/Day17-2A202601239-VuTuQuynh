# README_submission — Lab 17 (Minh, minh-lab17)

Kết quả: `reports/benchmark.md` = **11/11 PASS (100%)**; `reports/benchmark_no_memory.md` = 2/11 (18.2%); chi tiết ở `reports/comparison.md`.

## 3 câu bắt buộc

**1) Layer quan trọng nhất trong bộ test này:** `long_term`, chiếm 4/11 case (E02, E03, E08, E09) — nhiều nhất — và duy nhất xử lý cả conflict/recency (E08: BLUEBIRD-42 đổi từ Python sang TypeScript/NestJS) lẫn user isolation (E09: Lan không thấy `ORCHID-27` của Minh). Sai `retrieve_long_term` rớt cả cụm, ảnh hưởng nhiều điểm nhất trong 56đ auto.

**2) Trade-off Context Block (Zep) vs tự build Redis+Qdrant:** Zep tự lo trích fact/entity, theo dõi validity range, merge context theo relevance — đổi lại phụ thuộc network/latency (~660-1400ms/case) và ít kiểm soát schema. Redis+Qdrant cho toàn quyền schema, latency thấp, chạy local, nhưng phải tự làm TTL, rerank, xử lý conflict — hợp khi cần kiểm soát chặt/offline.

**3) Guardrail chống memory poisoning:** (a) `data/consent.json` chặn ingest nếu chưa `memory_opt_in`; (b) `privacy_guard.minimize_pii` redact PII trước khi ghi; (c) `heartbeat.py` chỉ de-duplicate/đánh dấu stale, **không** tự thêm instruction/quyền mới vào durable memory; (d) fact có `valid_at/invalid_at` nên fact sai vẫn truy vết được thay vì bị ghi đè âm thầm.

## 4 câu phân tích benchmark

1. **Layer hit rate thấp nhất:** cả 4 layer đều 100% ở lần chạy này. Layer rủi ro cấu hình cao nhất là `semantic` — chọn sai `scope` (`"auto"` thay vì `"episodes"`) hoặc nhầm `user_id` thay `graph_id` sẽ rớt cả E06+E11 cùng lúc.
2. **Query tốn token nhất:** E02 và E08 (long_term), cùng 861 token — Context Block gộp toàn bộ user summary xuyên nhiều session.
3. **E07 (mixed) cần:** `long_term` (Python — preference cá nhân) + `semantic` (`Idempotency-Key` — quy tắc retry dùng chung); thiếu 1 trong 2 là fail.
4. **Token reduction vs hit rate:** memory-enabled giảm 14.2% token nhưng đạt 100% hit rate; no-memory "giảm" 81.8% token (gần như không trả gì) nhưng chỉ 18.2% hit rate. Token reduction cao chỉ có ý nghĩa khi đi cùng hit rate cao — nếu không, đó là dấu hiệu bỏ sót bằng chứng chứ không phải hiệu quả.

## Ghi chú thêm

- **E08 (recency):** BLUEBIRD-42 đổi sang TypeScript/NestJS, Context Block trả bản mới nhất, không lẫn Python cũ — đúng "recency wins".
- **E10 (compaction):** giảm `max_recent_messages` 6→4 vẫn PASS vì `sliding` giữ `REVIEW-DEADLINE-1600` trong `DURABLE_NOTES` dù raw turn đã evict.
