# Mysubs CLI Usage

Mysubs is a command-line tool that aggregates AI subscription usage across
accounts and providers in one run. The commands and flags below come from the
argument parser in `src/index.ts` and the orchestration in `src/app.ts`.

## Commands

### `mysubs [subs...]`

Fetches and prints usage for the configured and detected accounts. Positional
`subs` arguments filter which accounts run; see [Selecting accounts](#selecting-accounts).
Without a filter, every target collected from detection and configuration runs.

### `mysubs key set <name>`

Prompts for a secret on stderr and stores it in the OS keyring under service
`mysubs` with entry `name`. Input is read with hidden characters when stdin is a
TTY, and falls back to plain line reading when it is not. An empty secret is
rejected: nothing is stored and the process exits with code 1.

### `mysubs key get <name>`

Prints the secret stored under entry `name` of the `mysubs` keyring service.
When the entry does not exist, `mysubs: no keyring entry named <name>` is
written to stderr and the process exits with code 1.

## Options

| Flag | Effect |
| --- | --- |
| `-j`, `--json` | Print the results as a JSON array instead of the rendered bars. Progress output and per-account rendering are suppressed. |
| `-f`, `--force` | Ignore the cache and fetch fresh data. Verbose mode implies a fresh fetch as well. |
| `-v`, `--verbose` | Fetch fresh data and print sanitized provider request and response details to stderr. Secrets, cookies, tokens, emails, and password-like fields are redacted before printing. |

## Selecting accounts

Each filter token is either `provider` or `provider:account`:

- `mysubs codex` — every Codex target (detected and configured).
- `mysubs codex:work` — the configured account with the key `work` under `codex`.
- `mysubs codex:` — only automatically detected Codex accounts; the empty
  account name excludes manually configured ones.
- `mysubs codex claude opencode:zen` — several filters in one run.

Account keys are the keys of the `accounts` map in the provider's config block,
not display names. When no target matches a token, the run stops with
`no configured account matches "<token>"` and exits with code 1.

## Output

Text output renders one block per account: the provider name (in the provider's
color), the optional display name, the account info after `›`, the plan, and a
cached marker when the result came from the cache. Usage rows draw a bar per
consumption resource with its percentage and, when known, the time until it
resets; balance resources render as text rows. Bars are colored from the
remaining ratio, and `NO_COLOR` disables all color.

With `--json`, the process prints a JSON array of result objects. Each object
carries `provider`, `cached`, and optionally `sourceName`, `sourceType`,
`accountInfo`, `accountPlan`, `usage`, and `error`.

## Progress and errors

When stderr is a TTY and output is not JSON, a spinner shows the current
account and elapsed time while it is fetched. A failed account never stops the
run: the error is rendered as a row for that account (or an `error` field in
JSON), remaining accounts continue, and the process still exits 0.

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | The run completed. Per-account fetch failures are reported in the output, not through the exit code. |
| 1 | Fatal condition: no accounts were configured or detected, a filter matched nothing, a key command failed, or the command itself threw. When there are no accounts, the message names the config file to create. |
