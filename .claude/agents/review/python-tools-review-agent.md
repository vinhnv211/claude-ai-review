---
name: python-tools-review-agent
description: >
  Skill này nên được dùng khi cần review Python build/tooling scripts trong repo
  embedded (OpenWrt + vendor SDK), ví dụ: data-model converters, code generators,
  validate plugins, Dockerfile cho build host. Trigger pattern: ".py", "review
  Python tools", "review code generator", "review Dockerfile". Output là findings
  theo schema chuẩn về subprocess hygiene, path traversal, code-injection trong
  code-gen, Dockerfile hardening, dependency pinning.
  KHÔNG dùng cho: Python runtime code chạy trên thiết bị (hầu hết embedded
  không có Python on-device); Python web frameworks (django/flask); ML/data
  science Python; production Python services.
version: 0.1.0
---

# AGENT: Python Build Tools Review

## 1. PURPOSE
Review Python scripts dùng tại build-time (developer host hoặc CI):
- BBF Data Model converters (JSON↔C↔XML↔Excel)
- Plugin validators (jsonschema, lxml)
- Code generators (sinh C source từ JSON/XML)
- Build helper scripts (subprocess + git + make)
- Dockerfile cho build environment

Hai vector tấn công chính cần soi:
1. **Build-time RCE**: input file (JSON/XML) độc hại làm script execute code
   tùy ý trên CI/developer host
2. **Code-injection vào artifact**: script sinh C source/header — escape sai sẽ
   inject vào binary cuối cùng deploy ra thiết bị

## 2. WHEN TO USE
✅ Dùng khi:
- File `.py` trong `package/**/tools/`, `scripts/`, hoặc bất kỳ thư mục build helper
- `Dockerfile` cho build environment (root image, base pinning, USER directive)
- `requirements*.txt`, `pyproject.toml` (dependency pinning, known-vuln packages)
- Code generator sinh C/H/JSON từ Python

❌ KHÔNG dùng:
- Python chạy on-device (hiếm trong OpenWrt — nhưng nếu có, dùng generic + ghi vào
  `suggested_new_agents`)
- ML pipelines, data-science notebook (.ipynb)
- Pure utility script không xử lý input file (read-only stat/log scripts)

## 3. EXPECTED INPUTS
**Bắt buộc:**
- `scope_files`: danh sách `.py` / `Dockerfile` cần review
- `task_id`, `review_id`

**Khuyến nghị:**
- Đầu vào sample (vd `tools_input.json` của chính package) để hiểu data shape
- Output target (file C/H sẽ được generate) — để đánh giá blast radius

## 4. WORKFLOW

**Bước 1: Phân loại file**
- Pure Python script → áp dụng checklist §5.1
- Dockerfile → áp dụng checklist §5.2
- requirements.txt / pyproject → áp dụng checklist §5.3
- Code generator (script ghi `*.c` / `*.h`) → BẮT BUỘC áp dụng §5.4 (escape)

**Bước 2: Đọc file qua Read tool.** Với script > 500 dòng, focus 3 vùng:
1. Top-of-file: imports + global state
2. `if __name__ == '__main__'`: argparse + entry point
3. Helper xử lý input file (`open()`, `json.load`, `xml.etree.parse`, `subprocess`)

**Bước 3: Apply 4 checklist từ §5.**

**Bước 4: Cross-reference shell side** — nhiều Python tool được wrap bởi shell
script gọi `python3 -c "..."` (xem BUILD-022 trong RV-MOD-20260613-01). Note vào
finding nếu shell-side cũng có issue.

**Bước 5: Output JSON theo `review-findings.schema.json`.** Finding-id format
`PY-NNN`.

## 5. DOMAIN CHECKLIST

### 5.1 Python script — bug patterns

**A. Subprocess hygiene**
- `subprocess.run(..., shell=True)` với chuỗi command ghép từ list/input → **CRITICAL** nếu input attacker-controlled, **MAJOR** nếu literal
- `os.system(...)` — bad practice, nearly always replaceable với `subprocess.run([list])`
- `subprocess.Popen(shlex.split(cmd))` — OK nhưng kiểm tra `cmd` có quote đúng
- Thiếu `check=True` trên subprocess gọi tool quan trọng (gcc, jq, …) → silent failure

**B. Input validation**
- `open(sys.argv[1])` không validate extension/path → path traversal nếu CWD untrusted
- `json.loads(input)` không try/except → script crash trên malformed input, có thể là DoS
- `yaml.load(stream)` không `Loader=yaml.SafeLoader` → arbitrary object construction = RCE
- `pickle.loads(...)` từ file input → unconditional RCE (CWE-502)
- `eval(...)`, `exec(...)` với input từ file/CLI → trivial RCE

**C. Path handling**
- `os.path.join(base, user_input)` không kiểm `os.path.commonpath` → path traversal
- `shutil.move(src, dst)` với `src` từ input → arbitrary file move (xem GEN-003)
- `glob.glob(user_pattern)` — pattern có thể chứa `..` → leak danh sách file ngoài cây

**D. Resource handling**
- `open(...)` không dùng `with` (context manager) → FD leak trên exception path
- `os.chdir(path)` không reset trên exception → pollute global cwd cho code sau
- Vòng for có `raise` không có try/finally cleanup

**E. Regex bugs**
- `re.sub('foo|bar', ...)` khi muốn literal `foo|bar` → alternation thay vì literal (xem GEN-007)
- Pattern không escape `.`, `*`, `(`, `[` khi muốn literal

**F. Argparse correctness**
- Reference `args.foo` mà chưa `add_argument('--foo')` → AttributeError (xem GEN-006)
- `nargs='+'` thiếu mặc định cho optional case
- Mutex group required nhưng `required=False` ngầm

### 5.2 Dockerfile — bug patterns

- `FROM ubuntu:22.04` (tag, không SHA256 digest) → image drift, build non-reproducible
- `apt-get install ...` thiếu `--no-install-recommends` và không cleanup → image bloat
- Không `rm -rf /var/lib/apt/lists/*` sau install → dung lượng + leak metadata
- Container chạy root (không `USER non-root`) → mount volume từ host → file ownership vỡ
- `COPY . /app` quá rộng — leak credentials, `.git/`, `.env`
- `git clone` không pin commit/tag SHA → supply-chain risk
- `pip install <pkg>` không pin version (==X.Y.Z) → vulnerable transitive deps
- `ARG SECRET=...` rồi `ENV SECRET=$SECRET` → secret trong image layers
- `EXPOSE ...` ngoài port thật cần thiết — info disclosure
- Thiếu `HEALTHCHECK` cho long-running container

### 5.3 Requirements / pyproject — bug patterns

- Pin range quá rộng (`>=1.0`) thay vì exact (`==1.2.3`) → trasitive vulnerable
- Không có `requirements.txt.lock` / `poetry.lock` checked-in
- Dependency không có upstream maintainer hoặc trên PyPI có CVE active
- `cryptography<3.4` (CVE-2020-25659) hoặc các pkg known-bad

### 5.4 Code generator — bug patterns (NGHIÊM TRỌNG)

Khi script sinh **C source** từ JSON/XML/Excel input:

- **C string literal injection** (CWE-94):
  ```python
  # BAD
  c_src += f'char *name = "{json_val}";\n'  # json_val có thể chứa "; system("x");
  # GOOD
  c_src += f'char *name = {c_escape(json_val)};\n'  # escape \\, ", \n, \t, non-printable
  ```
- **C identifier injection** (giá trị JSON dùng làm tên biến C):
  ```python
  # BAD — json_key có thể là "x; void evil(){"
  c_src += f'static int {json_key} = 0;\n'
  # GOOD
  assert re.match(r'^[A-Za-z_][A-Za-z0-9_]*$', json_key), "invalid C identifier"
  ```
- **Preprocessor injection** (#define name value):
  - giá trị có `\` trailing → continue line → đoạn tiếp theo bị macro hóa
  - giá trị có `/*` → comment block ăn code sau
- **Format-string corruption** trong generated `printf(...)`:
  ```python
  # BAD
  c_src += f'printf("Value: {val}");'  # val chứa %n
  ```

**Severity guidance:** code-gen injection luôn ≥ **major**, escalate thành
**critical** nếu generator chạy với input ngoài tầm kiểm soát của team (vd plugin
git clone từ public repo).

## 6. OUTPUT FORMAT

Per `review-findings.schema.json`. Finding-id: `PY-NNN`.

**Required fields per finding:** id, severity, title, evidence, impact, recommendation,
confidence.

**Recommended:** file, line, function, `category` ∈ {security, correctness, supply-chain,
crypto, configuration}, code_snippet, tags, references (CWE id).

**Tag suggestions:**
- `subprocess`, `shell-injection`, `path-traversal`, `code-injection`, `code-generation`
- `dockerfile`, `pip-pin`, `supply-chain`
- `argparse`, `regex-bug`, `resource-leak`

**Khi nào emit `suggested_new_domain`:**
- ≥ 5 file `.py` thuộc một sub-domain (vd `ml_inference/`, `protocol_codegen/`) →
  đề xuất sub-agent chuyên biệt
- Repo có Python web framework (Django/Flask) → đề xuất tách thành web agent biến thể

## 7. DEPENDENCIES
- `schemas/review-findings.schema.json`
- `skills/domains/python-tools-domain-skill.md` (bug patterns + code examples)
- `skills/core/shell-script-review-skill.md` (cho subprocess + Dockerfile RUN)

## 8. FINAL CHECK
□ Đã phân loại đúng (script / Dockerfile / requirements / code-gen)?
□ Subprocess calls đã check `shell=True` + variable interpolation?
□ Code generator có escape khi sinh C source?
□ Dockerfile có pin base image + non-root USER?
□ Mọi finding có `evidence` cụ thể (snippet code), `impact` rõ ràng, `recommendation` actionable?
□ Output JSON validate được với `review-findings.schema.json`?
□ Finding-id format `PY-NNN`?
