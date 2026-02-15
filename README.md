<p align="center">
  <img src="niki.png" width="200" alt="niki — blue and gold macaw" />
</p>

<h1 align="center">niki</h1>

<p align="center">
  Deterministic process supervisor for AI agents.<br/>
  Token budgets, rate limits, and abort control.
</p>

---

Niki wraps any AI agent command and enforces hard limits. When the agent exceeds its budget, timeout, or rate limit, niki kills it. No negotiation.

## Install

```bash
npm install -g @tjamescouch/niki
```

## Usage

```bash
niki [options] -- <command> [args...]
```

The `--` separator is required. Everything before it is niki config, everything after is the command to supervise.

### Examples

```bash
# Basic: 500k token budget, 1 hour timeout
niki --budget 500000 --timeout 3600 -- claude -p "your prompt" --verbose

# Strict: rate-limit sends and tool calls
niki --budget 1000000 --max-sends 5 --max-tool-calls 20 -- claude -p "..." --verbose

# With external abort file (touch this file to kill the agent)
niki --budget 500000 --abort-file /tmp/niki-12345.abort -- claude -p "..." --verbose

# With logging and state output
niki --budget 500000 --log /tmp/niki.log --state /tmp/niki-state.json -- claude -p "..."
```

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `--budget <tokens>` | `1000000` | Max total tokens (input+output) before SIGTERM |
| `--timeout <seconds>` | `3600` | Max wall-clock runtime before SIGTERM |
| `--max-sends <n>` | `10` | Max `agentchat_send` calls per minute |
| `--max-tool-calls <n>` | `30` | Max total tool calls per minute |
| `--log <file>` | none | Append diagnostics to file |
| `--state <file>` | none | Write exit-state JSON on completion |
| `--metrics <file>` | none | Append session metrics as JSONL on exit (cumulative across runs) |
| `--cooldown <seconds>` | `5` | Grace period after SIGTERM before SIGKILL |
| `--abort-file <path>` | none | Poll this file for external abort signal |
| `--poll-interval <ms>` | `1000` | Base poll interval for abort file (±30% jitter) |

## How it works

1. Niki spawns the child command, inheriting stdin and stdout
2. Stderr is captured and parsed for token counts and tool calls
3. Token usage is tracked via high-water-mark (monotonically increasing)
4. Tool calls and sends are rate-limited with a sliding 60-second window
5. When any limit is exceeded, niki sends SIGTERM, waits the cooldown period, then SIGKILL
6. On exit, niki writes a state summary and exits with the child's exit code

## State file

When `--state` is provided, niki writes a JSON snapshot on exit:

```json
{
  "startedAt": "2026-02-09T12:00:00.000Z",
  "pid": 12345,
  "tokensIn": 45000,
  "tokensOut": 12000,
  "tokensTotal": 57000,
  "toolCalls": 42,
  "sendCalls": 8,
  "exitCode": 0,
  "killedBy": null,
  "duration": 1234
}
```


## Metrics file

When `--metrics` is provided, niki **appends** one JSON line per session exit. The file grows across restarts, giving you a full history:

```bash
# View last 5 sessions
tail -5 /tmp/niki-metrics.jsonl | jq .

# Total tokens across all sessions
cat /tmp/niki-metrics.jsonl | jq -s '[.[].tokensTotal] | add'

# Sessions killed by reason
cat /tmp/niki-metrics.jsonl | jq -s 'group_by(.killedBy) | map({reason: .[0].killedBy, count: length})'
```

Each line contains the full session state plus `endedAt`, `budget`, and `timeoutS` for context.

`killedBy` is one of: `"budget"`, `"timeout"`, `"rate-sends"`, `"rate-tools"`, `"abort"`, or `null` (clean exit).

## Security

- Never logs API tokens, environment variables, or message content
- Diagnostics contain only counters and timestamps
- Credentials flow through inherited env, never in CLI args
- State file contains only operational metrics

## Kill reasons

| Reason | Trigger |
|--------|---------|
| `budget` | `tokensTotal > --budget` |
| `timeout` | Wall-clock time exceeds `--timeout` |
| `rate-sends` | More than `--max-sends` agentchat_send calls in 60s |
| `rate-tools` | More than `--max-tool-calls` tool calls in 60s |
| `abort` | External abort file detected at `--abort-file` path |

## License

MIT

---

*Photo: [Unsplash](https://unsplash.com) (stylized)*
