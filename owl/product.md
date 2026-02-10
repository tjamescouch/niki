# niki

Niki is a deterministic process supervisor that wraps AI agent commands and enforces token budgets, wall-clock timeouts, and tool-call rate limits.

## components

- [supervisor](components.md#supervisor)
- [token-parser](components.md#token-parser)
- [rate-limiter](components.md#rate-limiter)
- [kill-controller](components.md#kill-controller)
- [diagnostics-logger](components.md#diagnostics-logger)

## behaviors

- the supervisor spawns a child command, passes through stdin and stdout, and captures stderr for monitoring.
- when total tokens exceed the budget, the supervisor terminates the child.
- when wall-clock time exceeds the timeout, the supervisor terminates the child.
- when tool calls or agentchat sends exceed per-minute limits, the supervisor terminates the child.
- on exit (normal or forced), the supervisor writes a state summary and exits with the child's exit code.
- unix signals received by niki are forwarded to the child process.

## constraints

- [constraints](constraints.md)
