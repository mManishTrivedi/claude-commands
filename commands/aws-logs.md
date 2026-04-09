---
description: "Download AWS CloudWatch logs and extract errors using awslogs CLI. Supports time ranges, profiles, live tail, and error summaries."
---

## AWS CloudWatch Log Downloader (via awslogs)

You are downloading AWS CloudWatch logs and extracting errors using **only bash commands** (no AI token consumption for log parsing). Parse the user's arguments from: $ARGUMENTS

### Prerequisites Check

**FIRST**, verify `awslogs` is installed:
```bash
command -v awslogs >/dev/null 2>&1 && echo "AWSLOGS_OK" || echo "AWSLOGS_NOT_FOUND"
```
If `AWSLOGS_NOT_FOUND`, install it:
```bash
pip3 install awslogs
```
If pip fails, try:
```bash
pip install awslogs
```
If both fail, stop and tell the user to install Python 3 and pip first.

Then verify AWS credentials work with the given profile:
```bash
AWS_PROFILE=<profile> AWS_REGION=<region> awslogs groups 2>&1 | head -5
```
If auth fails, stop and tell the user to check their AWS credentials/profile with `aws configure --profile <profile>`.

### Argument Parsing

Arguments are key-value flags. All are optional with sensible defaults:

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--log-group` | `-g` | (auto-discover) | Full CloudWatch log group path (e.g. `/ecs/my-api-prod`) |
| `--prefix` | | `/ecs` | Log group name prefix — used for auto-discovery if `--log-group` is not set |
| `--last` | `-l` | `2h` | Relative time: `30m`, `2h`, `5d` (shorthand for `--start`) |
| `--from` | | | Start time: `YYYY-MM-DD HH:MM:SS`, ISO 8601, or relative like `2h ago` |
| `--to` | | | End time: same formats as `--from` (defaults to now) |
| `--profile` | `-p` | `default` | AWS CLI named profile |
| `--region` | `-r` | `us-east-1` | AWS region |
| `--filter` | `-f` | | Grep filter pattern applied after download (e.g. `"ERROR"`, `"status=500"`) |
| `--errors-only` | | `false` | Only extract error/exception lines |
| `--raw` | | `false` | Skip error extraction, just download raw logs |
| `--output` | `-o` | `./aws-logs/` | Output directory |
| `--tail` | `-t` | | Live tail mode — stream logs in real time (Ctrl+C to stop) |
| `--watch` | `-w` | | Alias for `--tail` |
| `--streams` | | | Filter to specific log stream(s) — passed to `awslogs get --log-stream-name` |

**Examples:**
- `/aws-logs -g /ecs/my-api-prod` — last 2h of logs + error summary
- `/aws-logs -g /ecs/my-api-prod -l 5d` — last 5 days of logs
- `/aws-logs -g /ecs/my-store-staging -l 1h` — staging logs, last hour
- `/aws-logs -g /ecs/my-api-prod --from '2026-04-03 00:00:00' --to '2026-04-03 23:59:59'` — specific date range
- `/aws-logs -g /ecs/my-api-prod -p my-profile -r us-east-2 -l 2d --errors-only` — errors only
- `/aws-logs -g /ecs/my-api-prod -t` — live tail (real-time streaming)
- `/aws-logs -g /aws/lambda/my-func -f "OutOfMemory" -l 6h` — download then grep for pattern

### Log Group Resolution

If `--log-group` is provided, use it directly.

If `--log-group` is NOT provided, auto-discover available log groups:
```bash
AWS_PROFILE=<profile> AWS_REGION=<region> awslogs groups | grep "<prefix>"
```
- Show the user the list of discovered log groups
- Ask which one they want to download logs from
- If only one group matches, use it automatically

### Step-by-Step Execution

Run ALL of the following using **bash commands only**. Do NOT use AI to parse or summarize logs.

#### Step 1: Build the awslogs time arguments

`awslogs` accepts human-readable time strings for `--start` and `--end`:

**If `--last` is provided** (e.g., `2h`, `5d`, `30m`):
```bash
# Convert shorthand to awslogs format
# 2h  → --start='2h ago'
# 5d  → --start='5d ago'
# 30m → --start='30m ago'
START_ARG="--start='${LAST} ago'"
# No --end needed (defaults to now)
```

**If `--from`/`--to` are provided**:
```bash
# awslogs accepts these directly
START_ARG="--start='2026-04-03 00:00:00'"
END_ARG="--end='2026-04-03 23:59:59'"
```

**Default** (no time args): use `--start='2h ago'`.

#### Step 2: Create output directory & set filenames

```bash
OUTPUT_DIR="./aws-logs"
mkdir -p "$OUTPUT_DIR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
# Derive safe filename from log group (replace / with -)
SAFE_NAME=$(echo "$LOG_GROUP" | sed 's|^/||; s|/|-|g')
RAW_FILE="${OUTPUT_DIR}/${SAFE_NAME}-${TIMESTAMP}-raw.log"
ERRORS_FILE="${OUTPUT_DIR}/${SAFE_NAME}-${TIMESTAMP}-errors.log"
SUMMARY_FILE="${OUTPUT_DIR}/${SAFE_NAME}-${TIMESTAMP}-summary.txt"
```

#### Step 3: Live tail mode (if `--tail` or `--watch` flag)

If `--tail`/`--watch` is set, stream logs and skip all other steps:
```bash
AWS_PROFILE=<profile> AWS_REGION=<region> awslogs get <log-group> --watch --start='<last> ago'
```
Tell the user to press `Ctrl+C` to stop. Then exit — do not proceed to other steps.

#### Step 4: Download logs

Use `awslogs get` to download and save to file:

```bash
AWS_PROFILE=<profile> AWS_REGION=<region> awslogs get <log-group> \
  --start='<start-time>' \
  --end='<end-time>' \
  --no-group \
  --no-stream \
  --timestamp \
  > "$RAW_FILE" 2>&1
```

**Flags explained:**
- `--no-group` — omit log group name from output (cleaner)
- `--no-stream` — omit log stream name from output (cleaner)
- `--timestamp` — include event timestamp

**If `--streams` is provided**, add `--log-stream-name <stream>` to filter specific streams.

**IMPORTANT**: For large time ranges (>3 days of production), warn the user it may take several minutes and suggest `--errors-only` or a narrower range.

After download, check file size:
```bash
ls -lh "$RAW_FILE"
TOTAL_LINES=$(wc -l < "$RAW_FILE" | tr -d ' ')
echo "Downloaded $TOTAL_LINES lines"
```

If file is empty (0 lines), tell user:
- Check if the log group exists: `AWS_PROFILE=<profile> AWS_REGION=<region> awslogs groups | grep "<search>"`
- Check if the time range is correct
- Check if the region/profile is correct

#### Step 5: Extract errors (unless `--raw` flag)

Use grep/awk — NO AI processing:

```bash
# Extract error lines (case-insensitive match for common error patterns)
grep -inE \
  "(error|exception|fatal|panic|critical|fail|ECONNREFUSED|ETIMEDOUT|timeout|OOM|OutOfMemory|killed|segfault|status[=: ]5[0-9]{2}|\"statusCode\":(5[0-9]{2})|unhandled|uncaught|rejected)" \
  "$RAW_FILE" > "$ERRORS_FILE" 2>/dev/null

ERROR_COUNT=$(wc -l < "$ERRORS_FILE" | tr -d ' ')
TOTAL_LINES=$(wc -l < "$RAW_FILE" | tr -d ' ')
```

**If `--filter` is provided**, apply additional grep:
```bash
grep -i "<filter-pattern>" "$RAW_FILE" >> "$ERRORS_FILE"
```

#### Step 6: Generate error summary (bash only)

```bash
{
  echo "============================================"
  echo "  AWS LOG SUMMARY"
  echo "============================================"
  echo "  Log Group:   $LOG_GROUP"
  echo "  Profile:     $PROFILE"
  echo "  Region:      $REGION"
  echo "  Time Range:  $START_TIME → $END_TIME"
  echo "  Total Lines: $TOTAL_LINES"
  echo "  Error Lines: $ERROR_COUNT"
  echo "============================================"
  echo ""
  echo "--- TOP ERROR PATTERNS (by frequency) ---"
  echo ""

  # Group and count error patterns
  grep -oiE "(Error|Exception|Fatal|Panic|CRITICAL|FAIL|TIMEOUT|OOM|status[=: ]5[0-9]{2})[^\"]*" \
    "$ERRORS_FILE" 2>/dev/null \
    | sed 's/[0-9a-f]\{8,\}/XXX/g' \
    | sed 's/[0-9]\{4,\}/NNN/g' \
    | sort | uniq -c | sort -rn | head -20

  echo ""
  echo "--- ERROR TIMELINE (errors per hour) ---"
  echo ""

  # Errors per hour
  grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}' "$ERRORS_FILE" 2>/dev/null \
    | sort | uniq -c | sort -k2

  echo ""
  echo "--- HTTP 5xx STATUS CODES ---"
  echo ""

  # Extract HTTP status codes
  grep -oiE "(status[=: \"]+5[0-9]{2}|\"statusCode\"[=: ]+5[0-9]{2}|HTTP [0-9]{3})" "$ERRORS_FILE" 2>/dev/null \
    | sort | uniq -c | sort -rn | head -10

  echo ""
  echo "============================================"
  echo "  FILES"
  echo "============================================"
  echo "  Raw logs:      $RAW_FILE"
  echo "  Errors only:   $ERRORS_FILE"
  echo "  This summary:  $SUMMARY_FILE"
  echo "============================================"
} > "$SUMMARY_FILE"

# Print summary to console
cat "$SUMMARY_FILE"
```

#### Step 7: Quick sanity output

After everything, print:
```bash
echo ""
ls -lh "$OUTPUT_DIR"/*${TIMESTAMP}*
```

### If `--errors-only` Flag

Pipe awslogs output directly through error grep (skip saving full raw file):
```bash
AWS_PROFILE=<profile> AWS_REGION=<region> awslogs get <log-group> \
  --start='<start>' --end='<end>' --no-group --no-stream --timestamp 2>&1 \
  | grep -inE "(error|exception|fatal|panic|critical|fail|ECONNREFUSED|ETIMEDOUT|timeout|OOM|OutOfMemory|killed|segfault|status[=: ]5[0-9]{2}|unhandled|uncaught|rejected)" \
  > "$ERRORS_FILE"
```
- Still generate the summary file from errors file
- Much faster for large log volumes — less disk I/O

### Error Handling

- **awslogs not found**: Install with `pip3 install awslogs`
- **Log group not found / empty results**: List available log groups:
  ```bash
  AWS_PROFILE=<profile> AWS_REGION=<region> awslogs groups
  ```
- **Access denied**: Tell user to check IAM permissions (`logs:GetLogEvents`, `logs:DescribeLogGroups`, `logs:FilterLogEvents`)
- **Throttling**: awslogs handles pagination automatically — but for very large ranges, suggest narrowing the window or using `--errors-only`
- **Timeout**: For multi-day ranges, run in background with `&` and check file size periodically

### Important Rules

- Run ALL log downloading and parsing via **bash commands only** — never read log contents into the conversation
- Do NOT summarize or analyze log contents with AI — only use grep/awk/sort/uniq
- Show the summary file output to the user so they can see error counts and patterns
- For large downloads (>3d of production logs), warn the user it may take a while and suggest `--errors-only`
- Always use `AWS_PROFILE=x AWS_REGION=y` env vars prefix style (not --profile flags) — awslogs reads from env vars
- awslogs handles pagination internally — no need for manual next-token loops
