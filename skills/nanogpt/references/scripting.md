# Writing NanoGPT scripts dynamically

There are no bundled scripts. For every stage, write a small, purpose-built Python script, run it, and inspect its printed summary. Use only the Python standard library (`urllib`, `json`, `base64`, `mimetypes`, `time`) so nothing needs installing.

## 1. Where and how

- Write scripts named `ng_<stage>.py` into the working directory with the file-creation tool, then run them with `python3` (`python` on Windows). The working directory is `/home/claude/` on claude.ai; elsewhere, use a `nanogpt-work/` folder in the current directory.
- One script per stage (or per small group of tightly coupled stages). This keeps user checkpoints possible between stages.
- Scripts print compact, human-readable summaries: model used, job ID, status, cost, saved file paths. Never dump large JSON into the conversation - save it to a file and print a filtered view.
- Final deliverables go to the outputs directory and are presented to the user; intermediates stay in the working directory. The outputs directory is `/mnt/user-data/outputs/` on claude.ai; elsewhere, it is the current directory unless the user names another.

## 2. Key rules (non-negotiable)

- The key file is `nanogpt.key`, looked up first in the directory that contains this skill's `SKILL.md`, then at `~/.config/nanogpt/nanogpt.key`. Put the skill directory's absolute path (known from where the skill was loaded) in the script; the template checks both places.
- Only the script opens the file. Never read, print, search, copy or move it with any other tool or command.
- Load it inside a function, use it only to build the auth header, and never let it reach stdout, logs, exception text, filenames, URLs, command-line arguments or files you write.
- Never print request headers. When reporting an error, print the HTTP status and the response body only.
- Unauthenticated calls (catalogs, quotes) must not load the key at all.

## 3. Core template

Copy this into each script and adapt only the stage-specific part.

```python
import base64, json, mimetypes, os, sys, time, urllib.error, urllib.request

SKILL_DIR = "/ABSOLUTE/PATH/TO/SKILL/DIR"      # directory containing SKILL.md
WORK_DIR = "/ABSOLUTE/PATH/TO/WORKING/DIR"     # see section 1
KEY_PATHS = [os.path.join(SKILL_DIR, "nanogpt.key"),
             os.path.join(os.path.expanduser("~"), ".config", "nanogpt", "nanogpt.key")]
PLACEHOLDER = "REPLACE_WITH_YOUR_NANOGPT_API_KEY"
BASE = "https://api.nano-gpt.com"

def _auth_header():
    for path in KEY_PATHS:                       # first usable file wins
        try:
            with open(path, encoding="utf-8-sig") as f:
                k = f.read().strip()
        except (OSError, UnicodeError):
            continue
        if k and k != PLACEHOLDER:
            return {"x-api-key": k}
    sys.exit("ERROR: no usable nanogpt.key (missing, unreadable, empty or placeholder). Checked: "
             + ", ".join(KEY_PATHS) + ". The user must save their key in one of these files.")

def call(method, path, body=None, auth=True, headers=None, timeout=300, raw=False, form=None):
    """Return (status, parsed_json_or_bytes, content_type). Never prints headers."""
    h = {}
    data = None
    if form is not None:                         # (bytes, content_type) for multipart
        data, h["Content-Type"] = form
    elif body is not None:
        data = json.dumps(body).encode()
        h["Content-Type"] = "application/json"
    if auth:
        h.update(_auth_header())
    if headers:
        h.update(headers)
    url = path if path.startswith("http") else BASE + path
    req = urllib.request.Request(url, data=data, method=method, headers=h)
    try:
        with urllib.request.urlopen(req, timeout=timeout) as r:
            status, ctype, payload = r.status, r.headers.get("content-type", ""), r.read()
    except urllib.error.HTTPError as e:
        status, ctype, payload = e.code, e.headers.get("content-type", ""), e.read()
    if raw:
        return status, payload, ctype
    try:
        return status, json.loads(payload.decode()), ctype
    except ValueError:
        return status, {"_text": payload[:800].decode(errors="replace")}, ctype

def data_url(path):
    mt = mimetypes.guess_type(path)[0] or "application/octet-stream"
    with open(path, "rb") as f:
        return f"data:{mt};base64,{base64.b64encode(f.read()).decode()}"

def download(url, out_path):
    os.makedirs(os.path.dirname(out_path) or ".", exist_ok=True)
    with urllib.request.urlopen(url, timeout=900) as r, open(out_path, "wb") as f:
        f.write(r.read())
    return out_path

DAILY_CAP_CODES = {"daily_rpd_limit_exceeded", "daily_usd_limit_exceeded"}

def error_code(j):
    e = j.get("error") if isinstance(j, dict) else None
    return e.get("code") if isinstance(e, dict) else None

def poll(fetch, is_done, is_failed, max_seconds=1800):
    """fetch() -> (status, json, ctype). Backoff: 4s first minute, 8s to 5 min, then 15s."""
    t0 = time.time()
    while time.time() - t0 < max_seconds:
        el = time.time() - t0
        time.sleep(4 if el < 60 else 8 if el < 300 else 15)
        st, j, _ = fetch()
        if st == 429 and error_code(j) in DAILY_CAP_CODES:
            return st, j                     # key's daily cap: stop, never retry
        if st == 429 or st >= 500:
            continue                         # throughput limit / transient server error
        if st >= 400 or is_done(j) or is_failed(j):
            return st, j
    return None, {"error": "timeout"}
```

429 handling everywhere (submit calls included): read `error_code(j)`.
- `rate_limit_exceeded` → throughput limit; wait (use the `Retry-After` response header when present) and retry a bounded number of times.
- `daily_rpd_limit_exceeded` / `daily_usd_limit_exceeded` → the key's daily cap; do not retry. Print `error.message` (it states the count and the midnight-UTC reset) and exit so the pipeline stops for the user. To read `Retry-After`, call with `raw=True` or extend `call()` to also return response headers.

Multipart uploads (only where an endpoint requires a file upload): build the body with a random boundary - one part per form field plus one file part with its filename and content type - and pass it as `form=(body_bytes, "multipart/form-data; boundary=...")`.

## 4. Catalog queries (free, no key)

```python
st, cat, _ = call("GET", "/api/v1/<catalog>?detailed=true", auth=False)
json.dump(cat, open(os.path.join(WORK_DIR, "catalog_<domain>.json"), "w"))
rows = [m for m in cat["data"] if <capability / modality filter>]
for m in sorted(rows, key=<price sort key>)[:15]:
    print(m["id"], "|", m.get("name"), "|", json.dumps(m.get("pricing"))[:160], "|", (m.get("description") or "")[:140])
```

Each domain reference names its catalog path, the capability flags to filter on, and how to read its pricing. Catalogs can be hundreds of kilobytes - always filter before printing.

## 5. Exact price quotes (free, no key, no charge)

Many endpoints return an exact or estimated price when called **without any auth header** and with `x-x402: true`:

```python
st, q, _ = call("POST", "<endpoint path>", body=<the exact request body you intend to send>,
                auth=False, headers={"x-x402": "true"})
if st == 402:
    amount = (q.get("payment") or {}).get("amountUsd") or (q.get("accepts") or [{}])[0].get("maxAmountRequiredUSD")
```

- **The quote call must never carry the key.** With auth it becomes a real, paid call. Keep `auth=False`.
- Never send funds to any address in the response; the unpaid quote simply expires.
- Officially quote-capable endpoints and their quote type: `GET /api/v1/x402/endpoints` (`deterministic` = exact price; `estimated-with-reconciliation` = estimate). Other endpoints, including the asynchronous job submission endpoint, have also returned quotes in testing - use a quote whenever a `402` with an amount comes back.
- A `401` (missing key) or a validation error means no quote is available for that request; fall back to catalog arithmetic.
- An error such as "unsupported model" in the quote response is also a free validation that the request is wrong - fix it before any paid call.

## 6. Balance and actual cost

- Balance check: see `references/account.md`. Run it once at the start of an approved plan.
- Generation responses usually include `cost` and often `remainingBalance`; asynchronous jobs report a pre-charge on submit and a final cost on completion. Record both in the manifest.

## 7. Manifest

Keep `ng_manifest.json` in the working directory:

```json
{"pipeline": "...", "approved_total_usd": [low, high],
 "stages": [{"name": "...", "model": "...", "params": {}, "job_id": null, "status": "planned",
             "inputs": [], "outputs": [], "est_usd": 0, "actual_usd": null, "updated": "..."}]}
```

- Write the job ID into the manifest immediately after submission, before polling starts.
- On resume, read the manifest first: completed stages are skipped; pending job IDs are polled, not re-submitted.
