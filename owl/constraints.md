# constraints

## runtime

- node.js >= 18 (esm modules, `node:util` parseArgs).
- single-file executable at `bin/niki`, no runtime dependencies beyond node standard library.
- the `--` separator is mandatory in the cli invocation.

## defaults

- token budget: 1,000,000 tokens.
- wall-clock timeout: 3,600 seconds (1 hour).
- max agentchat_send calls: 10 per minute.
- max total tool calls: 30 per minute.
- sigterm-to-sigkill cooldown: 5 seconds.

## security

- never log or expose api tokens, environment variables, or message content.
- inherit the parent environment as-is so credentials flow through env, never through cli arguments.
- state json files contain only counters, timestamps, pids, and kill reasons.
- diagnostics logs contain only operational events with counters, never payload data.

## process model

- stdin and stdout are inherited (passed through to the child process unchanged).
- stderr is piped and captured for monitoring, then forwarded to the supervisor's stderr.
- the child process receives the full parent environment.
- signals (SIGINT, SIGTERM) received by niki are forwarded to the child.

## termination

- always attempt SIGTERM before SIGKILL.
- never send SIGKILL without a preceding SIGTERM and cooldown grace period.
- exit with the child's exit code, or 1 if unavailable.

## rate limiting

- use a sliding 60-second window, not fixed calendar minutes.
- prune stale timestamps before every limit check.

## packaging

- published as `@tjamescouch/niki` on npm.
- the `bin` field maps `niki` to `./bin/niki`.
- licensed under MIT.
