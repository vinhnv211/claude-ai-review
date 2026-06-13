---
name: shell-script-review-skill
description: >
  Core skill cho review shell script chung (POSIX sh, ash, bash). Cover portability,
  quoting, error handling, TOCTOU, injection, exit code. Dùng bởi mọi agent review
  file `.sh` hoặc script không extension có shebang shell.
version: 0.1.0
---

# SKILL: Shell Script Review (Generic)

## 1. PURPOSE
Cung cấp checklist và bug patterns cho shell script nói chung. Áp dụng được cho cả
POSIX `sh`, BusyBox `ash`, và `bash`. Không cover OpenWrt-specific idioms — phần đó
thuộc `openwrt-shell-domain-skill`.

## 2. WHEN TO USE
- File `.sh` bất kỳ trong scope review
- File không extension có shebang `#!/bin/sh`, `#!/bin/ash`, `#!/bin/bash`, `#!/usr/bin/env bash`
- Fragment shell inline trong `Makefile` recipe, `postinst`, `prerm`

## 3. SHEBANG & PORTABILITY

### 3.1 Shebang mismatch với syntax (MAJOR)
```sh
#!/bin/sh
arr=(a b c)           # bashism, ash/dash fail silent
[[ "$x" == "y" ]]     # bashism
```
**Check:** Nếu shebang `/bin/sh` → cấm bashism (liệt kê §3.2).

### 3.2 Bashism cần flag khi shebang là sh/ash
| Syntax | Bashism | POSIX thay thế |
|--------|---------|----------------|
| `[[ ... ]]` | yes | `[ ... ]` |
| `==` trong test | yes | `=` |
| Array `arr=(a b)` / `${arr[i]}` | yes | không có — dùng set positional |
| `${var^^}`, `${var,,}` | yes | `tr` |
| Here-string `<<<` | yes | `echo ... \|` |
| `$'...'` | yes | printf |
| `function foo { }` | yes | `foo() { }` |
| `source file` | yes | `. file` |
| `local` | yes (ash có) / không POSIX | OK trong ash/bash, không dùng trong dash pure |
| `declare`, `typeset` | yes | không có |
| `pushd/popd` | yes | cd + OLDPWD |
| `read -p` | yes | `printf; read` |
| `$(< file)` | yes | `$(cat file)` |

### 3.3 BusyBox ash limitations (MINOR)
Một số feature bash có trong ash nhưng cần enable:
- `{1..10}` brace expansion → ash KHÔNG có
- `<(...)` process substitution → ash KHÔNG có
- `trap ... ERR` → ash KHÔNG có (chỉ có EXIT, INT, TERM, HUP)

## 4. QUOTING BUG PATTERNS

### 4.1 Unquoted variable với whitespace/glob (CRITICAL)
```sh
FILE=$1
rm $FILE              # $1 = "a b.txt" → rm a b.txt (2 files)
rm $FILE              # $1 = "*" → rm tất cả file
```
**Fix:** `rm -- "$FILE"` — luôn quote + dùng `--` chặn option injection.

### 4.2 Unquoted trong test (MAJOR)
```sh
[ -n $var ]           # $var rỗng → [ -n ] → luôn true
[ $x = "y" ]          # $x rỗng → [ = "y" ] → syntax error
```
**Fix:** `[ -n "$var" ]`, `[ "$x" = "y" ]`.

### 4.3 Word splitting trong command substitution (MAJOR)
```sh
for f in $(ls *.conf); do ...    # fail với filename có space
```
**Fix:**
```sh
for f in *.conf; do              # glob trực tiếp
    [ -e "$f" ] || continue      # xử lý trường hợp không match
    ...
done
```

### 4.4 IFS không restore (MAJOR)
```sh
IFS=:
for p in $PATH; do ...; done
# IFS vẫn là : cho phần còn lại → bug khó debug
```
**Fix:**
```sh
OLD_IFS=$IFS; IFS=:
for p in $PATH; do ...; done
IFS=$OLD_IFS
```

### 4.5 printf format injection (MAJOR)
```sh
printf "$user_input\n"            # user input chứa %s → crash hoặc info leak
```
**Fix:** `printf '%s\n' "$user_input"`.

## 5. ERROR HANDLING

### 5.1 Không check exit code (MAJOR)
```sh
cd /some/dir
rm -rf ./*            # nếu cd fail → xóa CWD!
```
**Fix:**
```sh
cd /some/dir || exit 1
rm -rf ./*
```

### 5.2 `set -e` pitfalls (MAJOR)
`set -e` không trap trong:
- Command trong `if`, `while`, `until`, `&&`, `||`
- Pipeline (chỉ check last command, trừ khi có `set -o pipefail`)
- Subshell chỉ trap subshell đó
- Function return dùng làm condition

**Fix:** Kết hợp `set -euo pipefail` (nếu shell hỗ trợ) và explicit check chỗ quan trọng.

### 5.3 Pipeline không pipefail (MAJOR)
```sh
curl http://example.com/data | grep foo | wc -l
# curl fail → grep thấy rỗng → wc trả 0 → script tưởng thành công
```
**Fix:**
- bash/ash: `set -o pipefail`
- POSIX pure: check `${PIPESTATUS[@]}` hoặc dùng temp file

### 5.4 Trap không cleanup (MINOR)
Tạo tempfile không có trap → file rác khi interrupt.
```sh
TMP=$(mktemp)
trap 'rm -f "$TMP"' EXIT INT TERM HUP
```

## 6. TOCTOU / TEMP FILE

### 6.1 Predictable tempfile (CRITICAL security)
```sh
TMP=/tmp/myapp.$$
echo data > $TMP      # symlink attack: attacker tạo /tmp/myapp.<pid> → /etc/passwd
```
**Fix:** `mktemp` hoặc `mktemp -d`.

### 6.2 Check-then-use race (MAJOR)
```sh
if [ ! -e "$file" ]; then
    touch "$file"     # race: ai đó tạo giữa check và touch
fi
```
**Fix:** Dùng atomic primitive (`mkdir`, `ln`, `set -C` với `>`).

### 6.3 `rm -rf "$var/"` khi var rỗng (CRITICAL)
```sh
rm -rf "$BUILD_DIR/"   # $BUILD_DIR rỗng → rm -rf / (!!!)
```
**Fix:**
```sh
: "${BUILD_DIR:?BUILD_DIR must be set}"
rm -rf "$BUILD_DIR/"
```

## 7. INJECTION

### 7.1 `eval` với input không trusted (CRITICAL)
```sh
eval "echo $user_input"   # user_input = "; rm -rf /" → boom
```
**Fix:** Tránh `eval`. Nếu bắt buộc → validate whitelist strict.

### 7.2 Command substitution trong SQL/shell exec (CRITICAL)
```sh
sqlite3 db.sql "SELECT * FROM users WHERE name='$name'"  # ' trong name → SQLi
ssh host "cmd $arg"                                       # remote shell injection
```
**Fix:** Escape/parameterize hoặc dùng heredoc `<<'EOF'` (quote để không expand).

### 7.3 Path traversal từ input (MAJOR)
```sh
cat "/var/log/$user_file"    # user_file = "../../../etc/shadow"
```
**Fix:** Validate với `case` hoặc check `[[ "$user_file" = */* ]]` → reject.

### 7.4 `curl | sh` anti-pattern (MAJOR)
```sh
curl http://example.com/install.sh | sh   # no hash verify, no TLS pinning
```
**Fix:** Download → verify SHA256 → exec.

## 8. COMMON ANTI-PATTERNS

### 8.1 UUOC (Useless Use of Cat) (MINOR)
```sh
cat file | grep pattern        # → grep pattern file
cat file | wc -l               # → wc -l < file
```

### 8.2 `ls` parse (MAJOR)
```sh
for f in $(ls); do ...         # gãy với space, newline, special char
count=$(ls | wc -l)            # sai nếu tên file có newline
```
**Fix:** Dùng glob hoặc `find ... -print0 | xargs -0`.

### 8.3 `echo` với flag (MINOR)
```sh
echo "-n is flag"              # BusyBox hiểu -n là flag → output empty
```
**Fix:** `printf '%s\n' "$var"`.

### 8.4 `test -a` / `test -o` (MINOR)
Deprecated, ambiguous precedence.
**Fix:** Dùng `&&` / `||` giữa các `[ ]` riêng.

### 8.5 Subshell vô ý (MINOR)
```sh
cat file | while read line; do
    COUNT=$((COUNT+1))         # subshell → COUNT ngoài không đổi
done
echo "$COUNT"                  # vẫn 0
```
**Fix:** `while read line; do ...; done < file`.

## 9. PERMISSION & SECURITY

### 9.1 Executable bit (MINOR)
Script shebang `#!/bin/sh` nhưng không `chmod +x` → phải gọi bằng `sh script.sh`.

### 9.2 `umask` không set (MAJOR khi tạo file nhạy cảm)
```sh
echo "$password" > /etc/myapp/secret   # mode mặc định có thể 644
```
**Fix:** `umask 077` trước khi tạo hoặc `chmod 600` sau khi tạo.

### 9.3 World-writable directory cho state (MAJOR)
```sh
mkdir -p /tmp/myapp && chmod 777 /tmp/myapp
```
**Fix:** 700 hoặc 755, dùng subdir riêng per-user.

## 10. SEVERITY GUIDELINE

| Bug | Severity |
|-----|----------|
| `eval` với untrusted input | critical |
| `rm -rf "$var/"` không check empty | critical |
| Predictable tempfile (symlink attack) | critical |
| Command injection qua shell exec | critical |
| Unquoted var trong rm/mv/cp | major |
| Pipeline không pipefail khi exit code quan trọng | major |
| `cd` không check + lệnh destructive sau | major |
| `set -e` pitfall trong pipeline/function | major |
| TOCTOU race | major |
| `ls` parse trong loop | major |
| Bashism với shebang sh | major |
| umask không set khi ghi file nhạy cảm | major |
| Subshell vô ý làm mất biến | minor |
| UUOC | minor |
| `echo` không portable | minor |
| Thiếu trap cleanup | minor |

## 11. TOOL CROSS-REFERENCE
Các pattern trên overlap lớn với ShellCheck (https://www.shellcheck.net).
Mapping tham khảo:
- §4.1, §4.2 → SC2086, SC2248
- §4.3 → SC2045, SC2035
- §5.1 → SC2164
- §5.3 → SC2181, SC2143
- §6.3 → SC2115
- §8.1 → SC2002
- §8.2 → SC2045
- §8.5 → SC2030, SC2031

Agent có thể reference SC code để reviewer đối chiếu nhanh.

## 12. REVIEW OUTPUT HINT
Khi report finding, agent nên điền:
- `category`: `security` (injection/TOCTOU) | `correctness` (quoting/exit code) | `compat` (bashism) | `reliability` (error handling) | `style` (UUOC)
- `references`: link ShellCheck rule (nếu có)
- `code_snippet`: trích 1-3 dòng xung quanh bug

## 13. DEPENDENCIES
- Không có skill dependency
- Kiến thức tham khảo: POSIX shell spec, ShellCheck wiki, BusyBox ash docs
