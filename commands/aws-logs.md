---
description: "Download AWS CloudWatch logs and extract errors. Supports time ranges, profiles, services, and environments."
---

## AWS CloudWatch Log Downloader

You are downloading AWS CloudWatch logs and extracting errors using **only bash commands** (no AI token consumption for log parsing). Parse the user's arguments from: $ARGUMENTS

### Prerequisites Check

**FIRST**, verify AWS CLI is installed:
```bash
command -v aws >/dev/null 2>&1 && aws --version || echo "AWS_CLI_NOT_FOUND"
```
If `AWS_CLI_NOT_FOUND`, stop immediately and tell the user:
> AWS CLI is not installed. Install it with:
> - macOS: `brew install awscli`
> - Linux: `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && sudo ./aws/install`

Also verify the profile works:
```bash
aws sts get-caller-identity --profile <profile> 2>&1 || echo "AWS_AUTH_FAILED"
```
If auth fails, stop and tell the user to check their AWS credentials/profile.

### Argument Parsing

Arguments are key-value flags. All are optional with sensible defaults:

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--service` | `-s` | `api` | Service to fetch logs for: `api`, `store`, `webapp`, or a custom log group path |
| `--env` | `-e` | `production` | Environment: `production`, `staging` |
| `--last` | `-l` | `2h` | Time window: `30m`, `2h`, `5d`, etc. (m=minutes, h=hours, d=days) |
| `--from` | | | Start time: ISO 8601 (`2025-04-01T00:00:00Z`) or epoch ms |
| `--to` | | | End time: ISO 8601 or epoch ms (defaults to now) |
| `--profile` | `-p` | `default` | AWS CLI named profile |
| `--region` | `-r` | `us-east-1` | AWS region |
| `--filter` | `-f` | | CloudWatch filter pattern (e.g. `"ERROR"`, `"status=500"`) |
| `--errors-only` | | `false` | Only download error/exception lines |
| `--raw` | | `false` | Skip error extraction, just download raw logs |
| `--output` | `-o` | `./aws-logs/` | Output directory |
| `--tail` | `-t` | | Live tail mode — stream logs in real time (Ctrl+C to stop) |

**Examples:**
- `/aws-logs` — last 2h of production API logs + error summary
- `/aws-logs -s api -l 5d` — last 5 days of API logs
- `/aws-logs -s store -e staging -l 1h` — staging store logs, last hour
- `/aws-logs -s webapp --from 2025-04-01T00:00:00Z --to 2025-04-02T00:00:00Z` — specific date range
- `/aws-logs -p nea-prod -l 2d --errors-only` — only errors, last 2 days
- `/aws-logs -s api -t` — live tail production API logs
- `/aws-logs -f "OutOfMemory"` — filter for specific pattern

### Log Group Resolution

Map the `--service` + `--env` to CloudWatch log group:

| Service | Log Group Pattern |
|---------|-------------------|
| `api` | `/ecs/nea-api-{env}` |
| `store` | `/ecs/nea-store-{env}` |
| `webapp` | `/ecs/nea-webapp-{env}` |
| Custom path (starts with `/`) | Use as-is |

If the user provides a service name not in the table and it doesn't start with `/`, try `/ecs/nea-{service}-{env}`.

### Step-by-Step Execution

Run ALL of the following using **bash commands only**. Do NOT use AI to parse or summarize logs.

#### Step 1: Compute time range

Convert `--last` to epoch milliseconds:
```bash
# For --last flag (e.g., 2h, 5d, 30m)
UNIT="${LAST: -1}"    # last char: m, h, or d
VALUE="${LAST%?}"     # everything except last char

case "$UNIT" in
  m) START_MS=$(( ($(date +%s) - VALUE * 60) * 1000 )) ;;
  h) START_MS=$(( ($(date +%s) - VALUE * 3600) * 1000 )) ;;
  d) START_MS=$(( ($(date +%s) - VALUE * 86400) * 1000 )) ;;
esac
END_MS=$(( $(date +%s) * 1000 ))
```

If `--from`/`--to` are provided, convert ISO 8601 to epoch ms:
```bash
# macOS
START_MS=$(( $(date -j -f "%Y-%m-%dT%H:%M:%SZ" "2025-04-01T00:00:00Z" +%s) * 1000 ))
# Linux
START_MS=$(( $(date -d "2025-04-01T00:00:00Z" +%s) * 1000 ))
```

#### Step 2: Create output directory & set filenames

```bash
OUTPUT_DIR="./aws-logs"
mkdir -p "$OUTPUT_DIR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SERVICE="api"  # resolved service name
ENV="production"
RAW_FILE="${OUTPUT_DIR}/${SERVICE}-${ENV}-${TIMESTAMP}-raw.log"
ERRORS_FILE="${OUTPUT_DIR}/${SERVICE}-${ENV}-${TIMESTAMP}-errors.log"
SUMMARY_FILE="${OUTPUT_DIR}/${SERVICE}-${ENV}-${TIMESTAMP}-summary.txt"
```

#### Step 3: Live tail mode (if `--tail` flag)

If `--tail` is set, use live tail and skip all other steps:
```bash
aws logs tail "/ecs/nea-api-production" --follow --since "2h" --profile <profile> --region <region>
```
Tell the user to press `Ctrl+C` to stop. Then exit — do not proceed to other steps.

#### Step 4: Download logs

Use `aws logs filter-log-events` with pagination to handle large result sets:

```bash
LOG_GROUP="/ecs/nea-api-production"
PROFILE="default"
REGION="us-east-1"

# Build filter pattern
FILTER_PATTERN=""  # or user-provided --filter value

# Download with pagination — write raw log messages to file
NEXT_TOKEN=""
while true; do
  if [ -z "$NEXT_TOKEN" ]; then
    RESPONSE=$(aws logs filter-log-events \
      --log-group-name "$LOG_GROUP" \
      --start-time "$START_MS" \
      --end-time "$END_MS" \
      ${FILTER_PATTERN:+--filter-pattern "$FILTER_PATTERN"} \
      --output json \
      --profile "$PROFILE" \
      --region "$REGION" \
      --max-items 10000 2>&1)
  else
    RESPONSE=$(aws logs filter-log-events \
      --log-group-name "$LOG_GROUP" \
      --start-time "$START_MS" \
      --end-time "$END_MS" \
      ${FILTER_PATTERN:+--filter-pattern "$FILTER_PATTERN"} \
      --output json \
      --profile "$PROFILE" \
      --region "$REGION" \
      --max-items 10000 \
      --next-token "$NEXT_TOKEN" 2>&1)
  fi

  # Check for errors
  if echo "$RESPONSE" | grep -q "ResourceNotFoundException"; then
    echo "ERROR: Log group $LOG_GROUP not found"
    break
  fi

  # Extract and append log messages with timestamps
  echo "$RESPONSE" | python3 -c "
import json, sys, datetime
data = json.load(sys.stdin)
for event in data.get('events', []):
    ts = datetime.datetime.fromtimestamp(event['timestamp']/1000).strftime('%Y-%m-%d %H:%M:%S')
    print(f'[{ts}] {event[\"message\"]}')
" >> "$RAW_FILE"

  # Check for next page
  NEXT_TOKEN=$(echo "$RESPONSE" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('nextToken',''))" 2>/dev/null)
  [ -z "$NEXT_TOKEN" ] && break

  echo "  Fetching next page..."
done
```

**IMPORTANT**: If the raw file is very large (>50MB), warn the user and suggest narrowing the time window.

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

#### Step 6: Generate error summary (bash only)

```bash
{
  echo "============================================"
  echo "  AWS LOG SUMMARY"
  echo "============================================"
  echo "  Service:     $SERVICE"
  echo "  Environment: $ENV"
  echo "  Log Group:   $LOG_GROUP"
  echo "  Profile:     $PROFILE"
  echo "  Region:      $REGION"
  echo "  Time Range:  $(date -r $((START_MS/1000)) '+%Y-%m-%d %H:%M:%S') → $(date -r $((END_MS/1000)) '+%Y-%m-%d %H:%M:%S')"
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
  grep -oP '^\[\K[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}' "$ERRORS_FILE" 2>/dev/null \
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

After everything, print to the user:
```bash
echo ""
echo "📁 Files saved to $OUTPUT_DIR/"
ls -lh "$OUTPUT_DIR"/*${TIMESTAMP}*
```

### If `--errors-only` Flag

Skip saving raw logs to file. Instead, pipe directly through the error grep:
- Still generate the summary file
- Only save the errors file
- Faster for large log volumes

### Error Handling

- **Log group not found**: List available log groups and suggest the closest match:
  ```bash
  aws logs describe-log-groups --log-group-name-prefix "/ecs/nea-" --profile <profile> --region <region> --query 'logGroups[].logGroupName' --output text
  ```
- **Access denied**: Tell user to check IAM permissions (`logs:FilterLogEvents`, `logs:DescribeLogGroups`, `logs:Tail`)
- **Empty results**: Suggest widening the time range or checking the environment
- **Timeout/throttle**: Add `--limit` or suggest a smaller time window

### Important Rules

- Run ALL log downloading and parsing via **bash commands only** — never read log contents into the conversation
- Do NOT summarize or analyze log contents with AI — only use grep/awk/sort/uniq
- Show the summary file output to the user so they can see error counts and patterns
- For large downloads (>5d of production logs), warn the user it may take a while and suggest `--errors-only`
- Always confirm the log group exists before attempting download
