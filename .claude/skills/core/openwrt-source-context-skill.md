---
name: openwrt-source-context-skill
description: >
  Skill này nên được dùng khi code của package OpenWrt nằm trong tarball (dl/*.tar.gz)
  thay vì trực tiếp trong git. Trigger khi commit đụng vào Makefile với PKG_SOURCE/PKG_HASH,
  hoặc patches/*.patch, hoặc khi review một package OpenWrt mà source là tarball.
  Output là "synthetic diff" + scope_files thực (extract từ tarball) để orchestrator
  và sub-agents review như trên source thật. Dùng bởi review-orchestrator ở Bước 0.
  KHÔNG dùng cho: source đã nằm trực tiếp trong git (dùng git-context-skill), source
  theo PKG_SOURCE_PROTO=git (xử lý riêng).
version: 0.1.1
---

# SKILL: OpenWrt Source Context Extraction

<!--
Changelog:
  0.1.1 (2026-06-13):
    - §3 Detection: thêm pattern "pseudo-tarball" (PKG_SOURCE có nhưng PKG_HASH
      và/hoặc PKG_SOURCE_URL không có — source committed in-tree dưới dl/).
      Skill không extract tarball; emit finding về thiếu hash + flag trong
      missing_context để orchestrator treat dl/ files như git-tracked.
-->


## 1. PURPOSE
Xử lý pattern đặc thù của OpenWrt: source code nằm trong tarball `dl/*.tar.gz`, git chỉ
track `Makefile` + `patches/` + `files/`. Khi có commit đụng vào Makefile (bump version)
hoặc patches, `git-context-skill` một mình không đủ — cần extract tarball + apply patches
để biết thay đổi thực về code.

Skill này bổ sung (không thay thế) `git-context-skill`. Orchestrator gọi skill này ở
Bước 0 khi detect OpenWrt package pattern.

## 2. WHEN TO USE

✅ Dùng khi:
- Commit đụng vào `package/**/Makefile` và Makefile có `PKG_SOURCE` + `PKG_HASH`
- Commit đụng vào `package/**/patches/*.patch`
- User yêu cầu review một OpenWrt package có tarball source
- Mode `module-review` với `module_path` trỏ tới OpenWrt package

❌ KHÔNG dùng:
- Package OpenWrt có `PKG_SOURCE_PROTO:=git` (source là git clone, không tarball) —
  case này cần skill riêng `openwrt-git-source-skill` (chưa implement)
- Feed ipkg/opkg binary (không có source)
- Thay đổi chỉ trong `files/` overlay (xử lý như git file bình thường)

## 3. DETECTION LOGIC

Một directory là **OpenWrt package** nếu thoả:
1. Có file `Makefile` ngay trong thư mục đó
2. Makefile chứa ≥ 1 trong: `include $(INCLUDE_DIR)/package.mk`, `define Package/`, `PKG_NAME:=`

### 3.1 Classification matrix (v0.1.1)

Sau khi parse Makefile, phân loại theo bảng dưới:

| Pattern | `PKG_SOURCE` | `PKG_HASH` | `PKG_SOURCE_URL` | `PKG_SOURCE_PROTO` | Hành động |
|---------|-------------|-----------|-------------------|---------------------|-----------|
| **A. Standard tarball** | ✓ `.tar.*` | ✓ | ✓ | ∅ | Extract tarball, verify SHA256 (workflow §7) |
| **B. Git-protocol pkg** | ✓ | ∅ hoặc ✓ | ✓ | `git`/`svn`/`hg` | Trả error "dùng git-context-skill" — out-of-scope skill này |
| **C. Pseudo-tarball (in-tree)** ⚠️ | ✓ | ∅ | ∅ hoặc rỗng | ∅ | **NEW v0.1.1** — Không extract. Treat `dl/<name>-<ver>/` files như git-tracked. Emit finding `severity=major category=supply-chain` về thiếu hash. Flag `missing_context[]`. |
| **D. Files-only overlay** | ∅ | ∅ | ∅ | ∅ | Không phải tarball pkg. Chỉ review Makefile + files/ overlay. |

### 3.2 Pattern C — Pseudo-tarball (in-tree source) chi tiết

**Trigger conditions (ALL phải đúng):**
- Makefile có `PKG_SOURCE:=<name>.tar.<gz|xz|bz2>` (hoặc tương đương)
- Makefile **KHÔNG** có `PKG_HASH:=` (cả `PKG_MD5SUM`/`PKG_MIRROR_HASH` cũng vắng)
- `PKG_SOURCE_URL:=` rỗng HOẶC vắng
- Tồn tại thư mục `<repo_root>/dl/<PKG_NAME>-<PKG_VERSION>/` chứa source files (NOT tarball)
- Makefile thường có `PKG_UNPACK` custom (vd `$(CP) $(PKG_SRC_DIR)/* $(PKG_BUILD_DIR)/`)

**Tại sao detect tách:**
- Đây là anti-pattern trong OpenWrt (vendor commit source trực tiếp vào git).
- Không có integrity check (PKG_HASH vắng) — supply-chain risk.
- Skill không thể extract tarball vì tarball không tồn tại — phải bypass §7 workflow.

**Hành động skill phải thực hiện:**
1. **KHÔNG** thử fetch + extract tarball — sẽ fail và mất thời gian.
2. Emit synthetic finding vào `warnings[]`:
   ```json
   {
     "type": "pseudo-tarball-detected",
     "severity": "major",
     "category": "supply-chain",
     "package": "<PKG_NAME>",
     "version": "<PKG_VERSION>",
     "message": "Makefile declares PKG_SOURCE=<tarball> but no PKG_HASH and empty PKG_SOURCE_URL; source committed in-tree under dl/<name>-<ver>/. No integrity check possible.",
     "remediation": "Either (a) add PKG_HASH=$(sha256sum dl/<name>-<ver>.tar.gz | cut -d' ' -f1) + host the tarball at PKG_SOURCE_URL; or (b) drop PKG_SOURCE entirely and treat dl/ as a regular git-tracked subtree."
   }
   ```
3. Set `openwrt_package.pattern = "pseudo-tarball-in-tree"` trong output.
4. Set `openwrt_package.extract_dir = <repo_root>/dl/<PKG_NAME>-<PKG_VERSION>/` (đường dẫn git, không phải cache).
5. Trả về thành công (KHÔNG abort) — orchestrator sẽ treat files trong `dl/<name>-<ver>/`
   như git-tracked file thật (đi qua git-context-skill thông thường).

**Lý do trả về thành công thay vì abort:**
Pattern C là hợp pháp về mặt build (chỉ là anti-pattern). Source vẫn review-able qua
git. Skill chỉ cần đánh dấu rủi ro supply-chain để orchestrator (hoặc reviewer)
xử lý.

## 4. MAKEFILE FIELDS PARSED

| Field | Ý nghĩa | Dùng để |
|-------|---------|---------|
| `PKG_NAME` | Tên package | Cache key, virtual path prefix |
| `PKG_VERSION` | Version upstream | Cache key, diff label |
| `PKG_RELEASE` | Revision của OpenWrt package | Metadata |
| `PKG_SOURCE` | Tên file tarball | Xác định format extract |
| `PKG_SOURCE_URL` | URL download | Fetch khi cache miss |
| `PKG_HASH` | SHA256 verify | Bắt buộc; fail nếu mismatch |
| `PKG_SOURCE_SUBDIR` | Thư mục sau extract | Root thực của source |
| `HOST_BUILD_DIR` / `PKG_BUILD_DIR` | (optional) | Chỉ dùng khi user có sẵn build_dir |

## 5. CACHE LAYOUT

```
.review/.cache/sources/
├── _index.json                       LRU index, size tracking
└── <pkg-name>/
    ├── <version>-<hash7>/
    │   ├── .source/                  Extracted tarball raw
    │   └── .patched/                 Sau khi apply tất cả patches tính đến commit target
    └── <version>-<hash7>-prev/       (khi cần diff 2 version)
        └── .patched/
```

**Cache key:** `(PKG_NAME, PKG_VERSION, PKG_HASH[:7])`
**Policy:** LRU, mặc định cap 2GB. Cấu hình qua `REVIEW_CACHE_MAX_MB`.
**Atomic:** extract vào `.tmp-<pid>/` rồi rename → an toàn khi nhiều review chạy song song.

## 6. FETCH STRATEGY

Thứ tự tìm tarball (skill thử tuần tự, dừng khi thành công):
1. `<repo_root>/dl/<PKG_SOURCE>` — nếu user đã download sẵn
2. `<REVIEW_DL_MIRROR>/<PKG_SOURCE>` — local mirror
3. `<PKG_SOURCE_URL>/<PKG_SOURCE>` — network
4. `@APACHE/<PKG_SOURCE>`, `@GNU/<PKG_SOURCE>` — resolve OpenWrt macro
5. Fail — report rõ "tarball not available offline"

Sau khi có tarball: verify SHA256 khớp `PKG_HASH`. Mismatch → abort, không extract.

## 7. WORKFLOW

### 7.1 Case A — Commit chỉ bump version (Makefile changed)

**Trigger:** changed_files chỉ gồm Makefile, và diff có đổi `PKG_VERSION` hoặc `PKG_HASH`.

**Bước:**
1. Parse Makefile cũ (`git show <ref>~1:path/Makefile`) → `(ver_old, hash_old)`
2. Parse Makefile mới (`git show <ref>:path/Makefile`) → `(ver_new, hash_new)`
3. Fetch + extract cả hai vào cache
4. Apply `patches/` hiện tại (từ git) lên cả hai (nếu patches không đổi)
   - Nếu patches đổi cùng commit → áp patches version cũ lên tarball cũ, patches mới lên tarball mới
5. Generate synthetic diff:
   ```bash
   diff -urN .cache/sources/<pkg>/<ver_old>-<hash_old>/.patched \
             .cache/sources/<pkg>/<ver_new>-<hash_new>/.patched
   ```
6. Filter diff theo extension review-able (`.c`, `.h`, `.cpp`, `.lua`, `.sh`, `.py`...)
7. Build `changed_files` từ synthetic diff
8. Đánh dấu mỗi file với `virtual_source` metadata

### 7.2 Case B — Commit sửa/thêm patch

**Trigger:** changed_files có `patches/*.patch` (không phải Makefile).

**Bước:**
1. Parse Makefile hiện tại → `(ver, hash)`
2. Fetch + extract tarball vào `.source/`
3. Lấy danh sách patches trước commit (`git show <ref>~1:patches/`) — gọi là `patches_before`
4. Lấy danh sách patches sau commit — `patches_after`
5. Apply `patches_before` lên copy của `.source/` → `state_before/`
6. Apply `patches_after` lên copy khác của `.source/` → `state_after/`
7. Diff `state_before → state_after` = synthetic diff ngữ nghĩa
8. Đánh dấu mỗi file với `via_patch` pointing tới patch file gây đổi

**Phát hiện patch conflict:** nếu `patch -p1 --dry-run` fail → report finding severity `major`
category `correctness` về patch không apply được, không tạo synthetic diff cho patch đó.

### 7.3 Case C — Mixed (Makefile + patches cùng commit)

1. Parse Makefile cũ + mới như Case A
2. Extract cả hai tarball
3. Apply patches cũ lên tarball cũ, patches mới lên tarball mới
4. Diff `.patched_old → .patched_new`
5. Changed files có thể do: tarball đổi, patch đổi, hoặc cả hai

### 7.4 Case D — Module review OpenWrt package

**Trigger:** user yêu cầu `"review module <pkg>"` và pkg là OpenWrt package với tarball.

**Bước:**
1. Parse Makefile hiện tại → `(ver, hash)`
2. Extract + apply tất cả patches → `state/`
3. `scope_files` = liệt kê file trong `state/` theo glob review-able
4. Không có diff; orchestrator dùng strategy `full-review`
5. Optional: thêm Makefile + patches + files/ overlay vào scope để review build quality
   (sẽ được orchestrator route tới `openwrt-build-review-agent`)

## 8. OUTPUT SCHEMA

```json
{
  "mode": "commit-review | module-review",
  "openwrt_package": {
    "pkg_name": "apps",
    "pkg_version": "1.2.4",
    "pkg_hash": "abc...",
    "pkg_source_url": "https://...",
    "makefile_path": "package/network/services/apps/Makefile",
    "package_dir": "package/network/services/apps",
    "extract_dir": ".review/.cache/sources/apps/1.2.4-abc1234/.patched",
    "prev_version": "1.2.3",
    "prev_extract_dir": ".review/.cache/sources/apps/1.2.3-def5678/.patched",
    "patches_applied": ["001-fix-x.patch", "002-feature-y.patch"],
    "patches_changed": ["003-new.patch"],
    "patches_failed": []
  },
  "diff_content": "<synthetic diff>",
  "changed_files": [
    {
      "path": "apps-1.2.4/src/main.c",
      "status": "M",
      "virtual_source": {
        "package": "apps",
        "version": "1.2.4",
        "tarball": "apps-1.2.4.tar.gz",
        "via_patch": null,
        "extract_path": ".review/.cache/sources/apps/1.2.4-abc1234/.patched/apps-1.2.4/src/main.c"
      }
    },
    {
      "path": "apps-1.2.4/src/new_feature.c",
      "status": "A",
      "virtual_source": {
        "package": "apps",
        "version": "1.2.4",
        "via_patch": "patches/003-new.patch"
      }
    }
  ],
  "scope_files": ["...see changed_files[].path..."],
  "warnings": [
    "patch 004-xxx.patch conflicts on line 42 of src/foo.c, skipped"
  ]
}
```

## 9. VIRTUAL PATH CONVENTION

Path trong `changed_files` và findings dùng **virtual path** = `<tarball_subdir>/<rel_path>`.
Ví dụ: `apps-1.2.4/src/main.c`, KHÔNG phải full path của extract dir.

Lý do:
- Ổn định qua nhiều lần review (cache dir có thể đổi)
- Dễ đọc trong report
- Reviewer có thể reproduce bằng `tar xzf apps-1.2.4.tar.gz && cd apps-1.2.4 && vim src/main.c`

Field `extract_path` trong `virtual_source` giữ full path để agent/tool thực tế đọc file.

## 10. DOMAIN MAPPING AFTER EXTRACTION

Sau khi có `changed_files` với virtual path, orchestrator vẫn dùng `review-domain-mapping-skill`
bình thường — nhưng path match áp dụng lên **tên thực trong tarball**, không phải path git.

Ví dụ:
- `apps-1.2.4/src/dhcp_client.c` → match trigger `dhcp` → `dhcp-review-agent`
- `apps-1.2.4/lua/apps.lua` → extension `.lua` → `web-review-agent`
- `apps-1.2.4/util/strhelper.c` → không match → `generic-review-agent`

Ngoài ra, **file gốc trong git** (`Makefile`, `patches/*.patch`, `files/**`) được route
tới `openwrt-build-review-agent` (nếu có trong registry).

## 11. ERROR HANDLING

| Tình huống | Xử lý |
|-----------|-------|
| Makefile không parse được | Fail skill, báo lỗi cụ thể field nào thiếu |
| `PKG_HASH` mismatch với tarball | Abort, không extract (bảo mật) |
| Network fail + không có tarball cache | Return warning, fallback về review chỉ Makefile + patches |
| Patch conflict | Skip patch đó, add vào `patches_failed`, tiếp tục |
| Tarball > 500MB | Warning, chỉ extract file theo extension review-able |
| Cache full | LRU evict tới khi đủ chỗ |
| `PKG_SOURCE_PROTO:=git` | Báo lỗi "dùng git-context-skill thay vì skill này" |
| **Pseudo-tarball (Pattern C, v0.1.1)** | KHÔNG abort. Emit warning + set `pattern=pseudo-tarball-in-tree` + trỏ `extract_dir` vào `dl/<name>-<ver>/`. Orchestrator treat files như git-tracked. Xem §3.2. |

## 12. INTEGRATION VỚI ORCHESTRATOR

Orchestrator ở Bước 0 (Context Extraction):
```
if git-context-skill detect changed_files trong OpenWrt package:
    call openwrt-source-context-skill với (package_dir, commit_ref)
    → nhận synthetic diff + virtual changed_files
    → merge vào context: changed_files (git) + changed_files (virtual)
    → tiếp tục Bước 1 bình thường
```

Các file git-level (Makefile/patches/files/) giữ nguyên path git của chúng.
Các file virtual (trong tarball) dùng virtual path + `virtual_source` metadata.

## 13. GUARDRAILS

- KHÔNG fetch tarball nếu hash không verify được (bảo mật supply chain)
- KHÔNG extract file symlink trỏ ra ngoài extract dir (zip-slip / tar-slip)
- KHÔNG run code từ tarball (build script, configure, …) — chỉ extract
- KHÔNG cache tarball quá size cap
- PHẢI cleanup tmp dir nếu skill fail giữa chừng
- PHẢI log rõ mọi fetch + apply patch để reproduce được

## 14. DEPENDENCIES

- `git-context-skill` (chạy trước để biết changed_files)
- Unix tools: `tar`, `xz`, `bzip2`, `patch`, `sha256sum`, `diff`
- Optional: `curl` hoặc `wget` cho fetch network

## 15. FINAL CHECK

□ Đã detect đúng OpenWrt package (Makefile có `PKG_SOURCE` + `PKG_HASH`)?
□ Đã verify SHA256 khớp?
□ Đã handle đúng 4 case (A/B/C/D)?
□ Virtual path sạch (không chứa cache dir)?
□ `virtual_source` metadata đầy đủ cho mỗi changed file?
□ Patches failed đã log vào `warnings`?
□ Cache không leak khi fail giữa chừng?
