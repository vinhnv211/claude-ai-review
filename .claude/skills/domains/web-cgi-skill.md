---
name: web-cgi-skill
description: >
  Skill chuyên biệt cho review CGI handlers (C, shell) trong embedded webgui,
  bao gồm POST parsing, buffer bounds, command injection, authentication,
  response escaping. Dùng bởi web-review-agent khi scope có CGI binary source
  hoặc shell CGI.
version: 0.1.0
---

# SKILL: Web CGI (C / Shell)

## 1. PURPOSE
Tri thức và checklist cho review CGI handlers backend của webgui thiết bị nhúng.
CGI thường viết bằng C (biên dịch thành binary trong `/www/cgi-bin/`) hoặc shell script.

## 2. WHEN TO USE
✅ Khi review file C nằm trong path CGI (`cgi-bin/`, `www/`, `webs/`), hoặc shell CGI `.sh`
✅ Khi scope có file đọc/parse `CONTENT_LENGTH`, `QUERY_STRING`, POST body

## 3. COMMON BUG PATTERNS

### Pattern 1: Command injection (CRITICAL)
```c
// BUG: user input trực tiếp vào system()
char cmd[256];
char *ssid = get_post_param("ssid");
snprintf(cmd, sizeof(cmd), "iwconfig wlan0 essid %s", ssid);
system(cmd); // ssid = "; rm -rf /" → đánh sập thiết bị
```
**Fix:**
- Whitelist/sanitize: chỉ cho phép `[A-Za-z0-9_-]`
- Dùng `execv()` với argv array thay vì `system()`
- Escape shell metacharacters

### Pattern 2: Buffer overflow trong POST parsing
```c
// BUG: không check CONTENT_LENGTH
char buf[512];
int len = atoi(getenv("CONTENT_LENGTH"));
fread(buf, 1, len, stdin); // overflow nếu len > 512
```
**Fix:**
```c
int max = sizeof(buf) - 1;
int len = atoi(getenv("CONTENT_LENGTH") ?: "0");
if (len > max) len = max;
fread(buf, 1, len, stdin);
buf[len] = '\0';
```

### Pattern 3: Missing auth check
```c
// BUG: endpoint thực thi admin action không check session
int main() {
  char *action = get_post_param("action");
  if (strcmp(action, "reboot") == 0)
    system("reboot"); // anyone can trigger
}
```
**Fix:** check session cookie / token trước mọi write operation.

### Pattern 4: Response không escape
```c
// BUG: echo user input vào HTML response
printf("Content-Type: text/html\n\n");
printf("<p>Searched: %s</p>", search_term); // XSS
```
**Fix:** HTML escape `<`, `>`, `&`, `"`, `'`.

### Pattern 5: Method không được check
```c
// BUG: chấp nhận cả GET cho write
char *method = getenv("REQUEST_METHOD");
// Thiếu: if (strcmp(method, "POST") != 0) error_out();
do_write_config();
```

### Pattern 6: Shell CGI với biến không escape
```sh
#!/bin/sh
SSID=$(echo "$QUERY_STRING" | sed 's/.*ssid=//')
uci set wireless.default.ssid="$SSID" # nếu SSID='$(reboot)' → command sub
```
**Fix:** Tránh shell CGI khi có thể. Nếu phải: validate kỹ input và dùng `printf %q`.

### Pattern 7: Path traversal
```c
// BUG: serve file theo user input
char *file = get_post_param("file");
char path[256];
snprintf(path, sizeof(path), "/www/%s", file); // file="../../etc/passwd"
FILE *f = fopen(path, "r");
```
**Fix:** validate `file` không chứa `..`, `/`, absolute path.

### Pattern 8: Integer overflow khi allocate
```c
// BUG: multiplier overflow
int n = atoi(get_post_param("count"));
char *buf = malloc(n * sizeof(struct item)); // n rất lớn → overflow → small alloc
for (i = 0; i < n; i++) ... // OOB write
```

## 4. REVIEW HEURISTICS
- `system(`, `popen(`, `execl(...sh...)` → trace input source
- `getenv("CONTENT_LENGTH")` không có bounds check → buffer overflow
- `printf("%s"` với data từ POST → response XSS
- Main CGI không check `REQUEST_METHOD` / session → auth bypass
- Shell CGI với `$QUERY_STRING` vào command → injection

## 5. CGI CHECKLIST
- [ ] Input từ `QUERY_STRING` / POST body được validate/sanitize trước khi dùng
- [ ] Buffer bounds check khi đọc `CONTENT_LENGTH`
- [ ] Không truyền user input thẳng vào `system()` / `popen()` / shell
- [ ] Authentication/session check trước write operations
- [ ] `REQUEST_METHOD` validation (reject GET cho write)
- [ ] Response escape HTML/JSON characters
- [ ] Path traversal check khi file operations
- [ ] Integer overflow trong size calculations
- [ ] Error path cleanup (không leak fd/memory)

## 6. SEVERITY GUIDELINE
- **critical**: Command injection, auth bypass, buffer overflow có thể exploit
- **major**: Missing method check, path traversal, response XSS
- **minor**: Weak validation, missing error cleanup
- **info**: Suboptimal error messages, logging noise

## 7. DEPENDENCIES
Luôn dùng kết hợp với `embedded-c-code-review-skill` vì CGI viết bằng C.
