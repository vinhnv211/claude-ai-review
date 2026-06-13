---
name: git-context-skill
description: >
  Skill này nên được dùng khi cần trích xuất context từ git để phục vụ review,
  ví dụ: "review commit gần nhất", "review HEAD", "review 3 commit mới nhất",
  "review commit thứ 2 gần nhất", "review commit <hash>", "review module X".
  Output là git context (diff, changed_files, commit message) làm đầu vào cho review workflow.
  KHÔNG dùng cho: phân tích lịch sử dài hạn, git blame, thống kê metrics.
version: 0.1.1
---

# SKILL: Git Context Extraction

## 1. PURPOSE
Trích xuất ngữ cảnh từ git repository để cung cấp đầu vào chuẩn hóa cho review workflow.
Skill này là bước đầu tiên khi user yêu cầu `commit-review` hoặc `module-review`.

## 2. WHEN TO USE
✅ Dùng khi:
- "review commit gần nhất" / "review HEAD"
- "review 3 commit gần nhất" / "review N commit mới nhất" (N ≤ 10)
- "review commit thứ 2 gần nhất" / "review HEAD~1"
- "review commit <hash>" (commit ID cụ thể)
- "review khoảng commit từ A đến B"
- "review toàn bộ module Y"

❌ KHÔNG dùng:
- Phân tích git history toàn bộ project
- Thống kê số lượng commit, contributor
- Git blame để tìm author

## 3. COMMIT SELECTION MODES

Skill hỗ trợ 4 cách chọn commit để review. Mỗi mode mapping vào một `commit_spec` chuẩn:

### 3.1 Mode A: Commit gần nhất (HEAD)
**Trigger:** "review commit gần nhất", "review HEAD", "review commit mới nhất"
```json
{
  "ref": "HEAD",
  "count": 1
}
```
Git command:
```bash
git show HEAD --unified=5
```

### 3.2 Mode B: N commit gần nhất (1 ≤ N ≤ 10)
**Trigger:** "review 3 commit gần nhất", "review 5 commit mới nhất"
```json
{
  "ref": "HEAD",
  "count": 3,
  "aggregation": "combined"
}
```
Git command (combined diff — aggregation mặc định):
```bash
git log -3 --oneline          # list commits
git diff HEAD~3 HEAD --unified=5   # combined diff
```
**Giới hạn:** `count` phải ∈ [1, 10]. Nếu user yêu cầu > 10, **reject** và yêu cầu giảm.

Nếu `aggregation == "per-commit"` (optional): loop qua từng commit HEAD, HEAD~1, ... HEAD~N-1 và review riêng.

### 3.3 Mode C: Commit thứ K gần nhất (HEAD~K-1)
**Trigger:** "review commit thứ 2 gần nhất" → HEAD~1 (vì HEAD là thứ 1)
```json
{
  "ref": "HEAD~1",
  "count": 1
}
```
Git command:
```bash
git show HEAD~1 --unified=5
```

**Quy ước index:**
- "commit thứ 1 gần nhất" = `HEAD`
- "commit thứ 2 gần nhất" = `HEAD~1`
- "commit thứ K gần nhất" = `HEAD~(K-1)`
- Giới hạn: K ≤ 10

### 3.4 Mode D: Commit theo ID (hash)
**Trigger:** "review commit abc123def", "review commit <hash>"
```json
{
  "ref": "abc123def",
  "count": 1
}
```
Git command:
```bash
git show abc123def --unified=5
```
**Validation:** verify hash tồn tại bằng `git cat-file -e <hash>^{commit}` trước khi review.

## 4. WORKFLOW: COMMIT-REVIEW MODE

**Bước 1:** Parse user request → xác định `commit_spec`:
- `ref`: `HEAD` | `HEAD~N` | `<hash>`
- `count`: 1 ≤ count ≤ 10
- `aggregation`: `combined` (default) | `per-commit`

**Bước 2:** Validate:
- `count` trong [1, 10]
- `ref` resolve được (nếu là hash thì tồn tại)
- Repo có commit history đủ sâu (ví dụ yêu cầu HEAD~5 nhưng repo chỉ có 3 commit → error)

**Bước 3:** Lấy commit metadata (cho từng commit trong range):
```bash
git log <ref>~<count-1>..<ref> --format="%H|%s|%an|%ai"
```

**Bước 4:** Lấy changed files:
```bash
# Combined (aggregation mặc định)
git diff <ref>~<count> <ref> --name-status

# Per-commit
for c in $(git log <ref>~<count-1>..<ref> --format=%H); do
  git show $c --name-status
done
```

**Bước 5:** Lấy diff content:
```bash
# Combined
git diff <ref>~<count> <ref> --unified=5

# Per-commit
git show <commit_hash> --unified=5
```

**Bước 6:** Tạo review request:
```json
{
  "review_id": "RV-<DOMAIN>-<DATE>-<SEQ>",
  "mode": "commit-review",
  "commit_spec": {
    "ref": "HEAD",
    "count": 3,
    "aggregation": "combined"
  },
  "commits": [
    {"hash": "...", "subject": "...", "author": "..."}
  ],
  "changed_files": [
    {"path": "src/pppoe.c", "status": "M"}
  ],
  "change_summary": "<aggregated từ commit messages>",
  "diff_content": "<combined diff hoặc list of per-commit diffs>"
}
```

**Bước 7:** Chuyển cho `review-orchestrator`

## 5. WORKFLOW: MODULE-REVIEW MODE

**Bước 1:** Xác định `module_path` (hỏi user nếu chưa rõ)

**Bước 2:** Liệt kê file. PHẢI dùng fallback chain dưới — vì repo có thể đang ở
trạng thái working-tree D (toàn bộ file staged-delete, `git ls-files` rỗng),
hoặc shallow clone:

```bash
# 2.a — Primary: index-based (working tree)
files=$(git ls-files -- "<module_path>")

# 2.b — Fallback 1: empty index → đọc từ HEAD tree
if [ -z "$files" ]; then
    files=$(git ls-tree -r HEAD --name-only -- "<module_path>")
fi

# 2.c — Fallback 2: HEAD detached / orphan → tìm ref khả dụng
if [ -z "$files" ]; then
    main_ref=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null \
               || echo refs/heads/master)
    files=$(git ls-tree -r "$main_ref" --name-only -- "<module_path>")
fi

# 2.d — Cuối cùng: filesystem walk (chỉ khi git hoàn toàn không khả dụng)
if [ -z "$files" ]; then
    files=$(find "<module_path>" -type f \( -name '*.c' -o -name '*.h' \
              -o -name '*.lua' -o -name '*.sh' \) 2>/dev/null)
    # Warning: filesystem walk includes untracked files — log vào missing_context
fi
```

Sau khi có `files`, lọc theo extension:
```bash
echo "$files" | grep -E '\.(c|h|cpp|hpp)$'                # C/C++ code
echo "$files" | grep -E '\.(html|js|css|lua|htm)$'        # Frontend
echo "$files" | grep -E '\.(sh|init|user)$|^.*/etc/(init.d|uci-defaults|hotplug.d)/'  # Shell
```

**Bước 3:** Tạo review request với `mode: module-review` và `scope_files`.
Ghi rõ phương pháp đã dùng vào `missing_context[]` nếu fallback 2.c hoặc 2.d
được kích hoạt — để orchestrator biết source-of-truth là HEAD tree (không phải
working tree) hoặc filesystem (có thể chứa file untracked).

**Bước 4:** Chuyển cho orchestrator với strategy `full-review`

## 6. ERROR HANDLING
| Tình huống | Xử lý |
|-----------|-------|
| `count > 10` | Reject, yêu cầu user giảm count |
| `HEAD~N` không tồn tại (shallow repo / không đủ history) | Report rõ "repo chỉ có N commits" |
| Hash không hợp lệ | Báo lỗi + gợi ý `git log` để tìm hash đúng |
| Module path không tồn tại | Báo lỗi + list subdirs gần nhất |
| Diff quá lớn (>5000 dòng) | Báo warning + gợi ý chia nhỏ hoặc dùng `aggregation: per-commit` |
| `git ls-files` rỗng nhưng HEAD có content (working tree D-state, ví dụ user đang staged-delete) | Tự động fallback sang `git ls-tree -r HEAD` (xem §5 Bước 2.b). Log vào `missing_context[]`: "scope_files lấy từ HEAD tree do index rỗng". |
| HEAD detached / orphan branch | Resolve `refs/remotes/origin/HEAD` hoặc `refs/heads/master` (§5 Bước 2.c) |
| Repo không phải git hoặc git binary thiếu | Fallback filesystem walk (§5 Bước 2.d). Cảnh báo về untracked files. |

## 7. OUTPUT SCHEMA
```json
{
  "mode": "commit-review | module-review",
  "commit_spec": {
    "ref": "HEAD | HEAD~N | <hash>",
    "count": 1,
    "aggregation": "combined | per-commit"
  },
  "commits": [
    {"hash": "...", "subject": "...", "author": "...", "date": "..."}
  ],
  "module_path": null,
  "changed_files": [{"path": "...", "status": "M|A|D|R"}],
  "scope_files": ["..."],
  "diff_summary": "...",
  "diff_content": "..."
}
```

## 8. NOTES
- Default aggregation là `combined` — đơn giản hơn, một pass review
- `per-commit` dùng khi: muốn findings tách theo commit, hoặc từng commit có intent khác nhau
- Commit quá lớn (>500 dòng): ưu tiên hunks liên quan, bỏ formatting-only changes
- Luôn lấy commit message trước khi review diff để hiểu intent
