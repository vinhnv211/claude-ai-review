---
name: python-tools-domain-skill
description: >
  Tri thức domain cho Python build/tooling scripts trong repo embedded. Bug patterns
  + code examples cho subprocess, input validation, path traversal, code-gen escape,
  Dockerfile hardening. Dùng bởi python-tools-review-agent.
version: 0.1.0
---

# SKILL: Python Build Tools Domain

## 1. PURPOSE
Cung cấp checklist + bug patterns cụ thể cho Python scripts chạy tại **build-time**
(không phải on-device runtime). Trọng tâm: build-host RCE, supply-chain, và code
injection từ Python sinh ra C/H artifact.

## 2. ARCHITECTURE
Trong repo embedded (OpenWrt + vendor), Python thường xuất hiện ở 3 chỗ:

1. **Code generators** — sinh C/H source từ data model JSON/XML/Excel.
   Ví dụ: `convert_dm_json_to_c.py`, `generate_dm.py`. Output là `.c`/`.h` sau đó
   compile vào binary deploy lên thiết bị. **Bug ở đây = bug trên-device**.

2. **Plugin validators** — validate JSON schema, check naming convention, lint
   parameter list. Output là pass/fail. Bug ít blast radius, nhưng silent-pass
   trên malformed input có thể slip qua tới production.

3. **Build helpers** — wrap subprocess calls (gcc, git, make, jq) với
   configuration. Bug ở đây = build host RCE / supply-chain.

**Trust boundary:** input của Python tools thường đến từ:
- `tools_input.json` (developer-controlled — usually trusted)
- Plugin git clone (semi-trusted — third-party repo)
- BBF data model XML (downloaded từ BBF — trusted nhưng có thể bị MitM)
- CLI args do shell script wrapper truyền vào (file paths, có thể chứa user input)

## 3. COMMON BUG PATTERNS

### 3.1 Shell injection via subprocess(shell=True)

**Anti-pattern:**
```python
cmd = ['gcc', '-shared', '-o', output_lib, '-fPIC',
       f'-DBBF_VENDOR_PREFIX="{vendor_prefix}"'] + source_files
cmdstr = ' '.join(str(e) for e in cmd)   # ← rebuild thành 1 string
subprocess.run(cmdstr, shell=True, check=True)  # ← shell parse lại
```
Vấn đề: list đã sẵn cho `exec`-form, tác giả join thành string + `shell=True`.
`vendor_prefix` chứa `;`/`$(...)`/backtick sẽ bẻ shell.

**Fix:**
```python
cmd = ['gcc', '-shared', '-o', output_lib, '-fPIC',
       f'-DBBF_VENDOR_PREFIX="{vendor_prefix}"'] + source_files
subprocess.run(cmd, check=True)  # list = exec, không qua shell
# Validate vendor_prefix nếu nó từ input:
if not re.match(r'^[A-Za-z0-9_]+$', vendor_prefix):
    sys.exit("Invalid vendor prefix")
```

**Severity:** CRITICAL nếu input untrusted, MAJOR nếu literal/internal.
CWE-78 (Command Injection), CWE-88 (Argument Injection).

### 3.2 Path traversal

**Anti-pattern:**
```python
os.chdir(repo_path)
for dm_file in dm_files:  # từ tools_input.json
    if filename.endswith('.json'):
        shutil.move(filename, "/usr/share/bbfdm/plugins")
    elif os.path.isfile(filename):
        gcc(filename, "-o", "/usr/share/bbfdm/plugins/lib.so")
```
`filename = "../../etc/passwd"` hoặc `"/etc/shadow"` đều pass.

**Fix:**
```python
base = pathlib.Path(repo_path).resolve()
for dm_file in dm_files:
    candidate = (base / dm_file).resolve()
    if not str(candidate).startswith(str(base) + os.sep):
        raise ValueError(f"Path traversal: {dm_file}")
    # Use candidate (absolute path) downstream
```

CWE-22.

### 3.3 Code injection in C generator (NGHIÊM TRỌNG)

**Anti-pattern:**
```python
str_enum = f"char *{enum_name}[] = {{"
for v in enum_values:
    str_enum += f'"{v}", '   # ← v từ data model JSON; có thể chứa " hoặc \\
str_enum += "NULL};"
```
Một enum value `'"; system("rm -rf /"); //'` sẽ compile thành C code thực thi
`system(...)` khi data model load.

**Fix:**
```python
def c_escape(s: str) -> str:
    """Return C string literal including surrounding quotes."""
    out = '"'
    for ch in s:
        if ch == '"':   out += r'\"'
        elif ch == '\\': out += r'\\'
        elif ch == '\n': out += r'\n'
        elif ch == '\r': out += r'\r'
        elif ch == '\t': out += r'\t'
        elif 0x20 <= ord(ch) < 0x7f: out += ch
        else: out += f'\\x{ord(ch):02x}'
    out += '"'
    return out

str_enum = f"char *{enum_name}[] = {{"
for v in enum_values:
    str_enum += f'{c_escape(v)}, '
str_enum += "NULL};"

# Validate enum_name là C identifier:
assert re.match(r'^[A-Za-z_][A-Za-z0-9_]*$', enum_name)
```

CWE-94, CWE-116. **Luôn ≥ major, leo lên critical nếu input không kiểm soát.**

### 3.4 Regex pitfalls

**Anti-pattern:**
```python
text = re.sub('{{datatype|expand}}', detail, text)
# `|` là alternation → match '{{datatype' OR 'expand}}'
```

**Fix:**
```python
text = text.replace('{{datatype|expand}}', detail)
# Hoặc:
text = re.sub(re.escape('{{datatype|expand}}'), detail, text)
```

`re.sub` còn vấn đề khác: backslash trong replacement bị interpret là back-reference.
Khi `detail` có thể chứa `\`, dùng `str.replace` an toàn hơn.

### 3.5 Resource leak (open without context manager)

**Anti-pattern:**
```python
json_file = open(sys.argv[1], "r")
data = json.loads(json_file.read())
# ... 200 dòng code, có nhánh raise
# json_file.close() có thể không bao giờ chạy
```

**Fix:**
```python
with open(sys.argv[1], encoding='utf-8') as f:
    data = json.load(f)
```

Trên long-running script (vd validator chạy trên hàng nghìn file), FD leak có thể
chạm `ulimit`.

### 3.6 Argparse fields not declared

**Anti-pattern:**
```python
parser = argparse.ArgumentParser()
parser.add_argument('--format')
parser.add_argument('--output')
args = parser.parse_args()
generate_xml(args.format, args.dm_json_files, args.output)  # ← args.dm_json_files chưa add
```
`AttributeError: 'Namespace' object has no attribute 'dm_json_files'`.

**Fix:** mọi attribute access vào `args` phải có `add_argument` tương ứng. Dùng
linter `argparse-manpage` hoặc bandit để catch.

### 3.7 Unsafe deserialization

```python
# CRITICAL
import pickle
data = pickle.loads(open(input_file, 'rb').read())   # RCE
import yaml
data = yaml.load(stream)                              # RCE if Loader missing

# OK
data = json.load(open(input_file))                    # safe
import yaml
data = yaml.safe_load(stream)                         # safe
data = yaml.load(stream, Loader=yaml.SafeLoader)     # safe
```

CWE-502.

## 4. REVIEW HEURISTICS

- Mọi `subprocess.run(..., shell=True)` → đọc kỹ. Hơn 90% có thể bỏ `shell=True`.
- Mọi `open(...)` không trong `with` → minor finding.
- Mọi `os.path.join(base, user_input)` không kèm `commonpath` check → major.
- Mọi script ghi vào `.c`/`.h` → phải có escape helper. Nếu không có → critical.
- Mọi `re.sub(pattern, ...)` có ký tự `|`/`.`/`*`/`(` trong pattern → kiểm
  xem có cần `re.escape`.
- Mọi `print(...)` trong code-gen → kiểm output có là source code không (xem
  next char là `"` không).

## 5. DOMAIN-SPECIFIC RISKS

| Risk | Mitigation |
|------|-----------|
| Build-host compromise via input JSON | Validate JSON schema trước khi process; sandbox script run |
| Supply-chain: plugin git clone | Pin commit SHA; verify GPG signature; mirror trong internal registry |
| Generated C code injection | Escape helper; identifier whitelist; CI check generated artifact diff |
| FD/cwd leak in long script | Context manager; pathlib instead of os.chdir |
| Dockerfile drift | Pin base image SHA256; lock dependency manifest |

## 6. CHECKLIST

Per file:
- [ ] Mọi `subprocess` không `shell=True` hoặc đã validate args
- [ ] `open()` luôn trong `with`
- [ ] Path từ input đã `resolve() + startswith(base)`
- [ ] Regex literal đã escape `|.*()[]{}`
- [ ] Code-gen có escape helper cho C string literal
- [ ] argparse fields đầy đủ
- [ ] Không dùng `eval`/`exec`/`pickle`/`yaml.load` unsafe
- [ ] Có try/except quanh I/O quan trọng
- [ ] Exit code non-zero trên failure (cho CI catch được)

## 7. SEVERITY GUIDELINE

| Severity | Examples |
|---------|----------|
| **critical** | `pickle.loads(file)`, `eval(file_content)`, code-gen vào C/H không escape khi input untrusted |
| **major** | `subprocess(shell=True)` với input, path traversal, code-gen escape thiếu (input semi-trusted), Dockerfile chạy root + mount volume host |
| **minor** | `open()` không context manager, regex literal bug, argparse field thiếu (chỉ block standalone run), Dockerfile tag-not-digest |
| **info** | Bashisms trong shell wrapper, style issues, missing type hints |

## 8. TEST GAPS TO FLAG

- Không có unit test cho `c_escape` helper (nếu có generator)
- Không có golden-file test cho generated `.c`/`.h` (diff against checked-in expected)
- Không có schema validation test cho `tools_input.json`
- Không có CI lint chạy bandit / ruff / mypy
- Không có Dockerfile lint (`hadolint`)
- Không có dependency vuln scan (`pip-audit` / `safety`)
