# components

## supervisor

the supervisor is the top-level orchestrator. it parses cli arguments, spawns the child process, and coordinates all other components.

### state

- child process reference and pid
- start timestamp
- killed flag (prevents duplicate kill attempts)

### capabilities

- parses `niki [options] -- <command> [args...]` argument format using a `--` separator.
- spawns the child command with inherited stdin, inherited stdout, and piped stderr.
- inherits the parent environment so api tokens pass through without appearing in cli args.
- forwards SIGINT and SIGTERM to the child process.
- exits with the child's exit code or 1 if the code is unavailable.

### interfaces

- **input**: command-line arguments, child process stderr stream, unix signals.
- **output**: forwarded stderr, diagnostics log, state json file, exit code.

### invariants

- a `--` separator and at least one command token must be present or niki exits with usage help.
- the child process is spawned exactly once per invocation.
- the killed flag ensures termination logic runs at most once.

---

## token-parser

the token parser extracts input and output token counts from the child's stderr output.

### state

- cumulative input tokens (high-water mark)
- cumulative output tokens (high-water mark)
- total tokens (sum of input and output)

### capabilities

- matches multiple token-reporting formats: json-style (`"input_tokens": N`), and human-readable (`Input tokens: N`).
- uses high-water-mark tracking per field (takes the max of each new reading vs. current value).
- processes stderr line by line using a newline-delimited buffer.

### interfaces

- **input**: raw stderr text lines from the child process.
- **output**: updated token counts in the shared state object.

### invariants

- token counts are monotonically non-decreasing.
- total tokens always equals input tokens plus output tokens.

---

## rate-limiter

the rate limiter tracks tool calls and agentchat send calls within a sliding one-minute window.

### state

- ordered list of tool-call timestamps
- ordered list of send-call timestamps
- current-minute counts for each

### capabilities

- detects tool calls from stderr lines matching tool-use patterns.
- separately identifies agentchat_send calls as a subset of tool calls.
- prunes timestamps older than 60 seconds from each window.
- reports a violation type (`rate-sends` or `rate-tools`) when a window exceeds its limit.

### interfaces

- **input**: stderr text lines from the child process.
- **output**: violation type string or null.

### invariants

- the sliding window is always pruned before checking limits.
- a send call increments both the send counter and the tool-call counter.

---

## kill-controller

the kill controller handles graceful and forced termination of the child process.

### state

- reference to the child process
- killed flag (shared with supervisor)
- kill reason string

### capabilities

- sends SIGTERM to the child process.
- after a configurable cooldown period, sends SIGKILL if the child has not exited.
- records the kill reason in the shared state.

### interfaces

- **input**: kill reason from budget check, timeout, or rate-limit check.
- **output**: SIGTERM and SIGKILL signals to the child process.

### invariants

- kill logic executes at most once per supervisor invocation (guarded by the killed flag).
- SIGKILL is always preceded by SIGTERM with a cooldown grace period.

---

## diagnostics-logger

the diagnostics logger writes timestamped operational events and, optionally, a final state json snapshot.

### state

- optional append-mode log file stream
- optional state file path

### capabilities

- writes timestamped lines to a log file (if `--log` is specified).
- writes all log lines to stderr prefixed with `[niki]`.
- writes a json state snapshot on exit (if `--state` is specified), containing counters and metadata only.

### interfaces

- **input**: log messages from all components, shared state object.
- **output**: log file, stderr output, state json file.

### invariants

- diagnostics never contain message content, api tokens, or environment variables.
- state json contains only counters, timestamps, pids, exit codes, and kill reasons.
