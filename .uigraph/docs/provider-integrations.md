# Provider Integrations

Mysubs collects usage from six providers through a common interface: each
provider module registers a display name, color, config schemas, a detection
function, and a fetch function in `src/providers/index.ts`. This document
records, per provider, how accounts are found, how credentials are obtained,
which endpoints are called, and what usage data is mapped.

All providers return the same result contract: a `provider` name, a `cached`
flag, optional `accountInfo` and `accountPlan`, a `usage` map of resources, and
an `error` string when the fetch failed. Usage resources are either
*consumption* (`used`, `limit`, `remaining`, `utilization`, `resetsAt`,
`windowSeconds`, in `percent` or `usd`) or *balance* (`available`, in `usd` or
`credits`). Failures are returned as error results per account; they are never
thrown across accounts, and secrets or email addresses are never stored, logged,
or displayed (verbose output is redacted).

## Codex

- **Detection** — the first of `$CODEX_HOME`, `~/.config/codex`, `~/.codex`
  that contains `auth.json`; plus the OpenAI OAuth entry inside the opencode
  CLI's auth file when present.
- **Credentials** — `auth.json` in the account's `configDir`, or on macOS the
  `Codex Auth` keychain item read through `/usr/bin/security` when the file is
  missing. The `opencode-oauth` adapter reads `openai` from opencode's
  `auth.json` (`$XDG_DATA_HOME/opencode/auth.json`, default
  `~/.local/share/opencode/auth.json`).
- **Token refresh** — `POST https://auth.openai.com/oauth/token`
  (`refresh_token` grant) when the access token is near expiry; refreshed
  tokens are written back to the source file (mode 0600) or keychain, and to
  opencode's `auth.json` for the adapter.
- **Usage** — `GET https://chatgpt.com/backend-api/wham/usage` with the bearer
  access token and optional `ChatGPT-Account-Id`. Primary and secondary rate
  limit windows map to `session` and `weekly` (exact 5-hour and weekly windows
  are preferred), `additional_rate_limits` map to named usage keys, and credits
  balance maps to a USD balance at $0.04 per credit. The plan comes from
  `plan_type` (`prolite` is shown as `Pro 5x`, `pro` as `Pro 20x`); the account
  name is decoded from the `id_token` JWT.
- **Not supported** — accounts authenticated by `OPENAI_API_KEY` report that
  usage is unavailable for API-key auth.

## Claude

- **Detection** — `.credentials.json` in `CLAUDE_CONFIG_DIR` or `~/.claude`
  (also honoring `CLAUDE_SECURESTORAGE_CONFIG_DIR`), or the macOS keychain item
  `Claude Code-credentials` (plus a config-dir-scoped service name when
  `CLAUDE_CONFIG_DIR` overrides the default).
- **Credentials** — the `claudeAiOauth` object from `.credentials.json` or the
  keychain, read through `/usr/bin/security`.
- **Token refresh** — `POST https://platform.claude.com/v1/oauth/token` when
  `expiresAt` is within five minutes; refreshed credentials are written back
  atomically (temp file + rename) or to the keychain.
- **Usage** — `GET https://api.anthropic.com/api/oauth/usage` with the
  `anthropic-beta: oauth-2025-04-20` header and a `claude-code` user agent.
  `five_hour` maps to `session`, `seven_day` to `weekly`, `seven_day_opus` to
  `opus`, `seven_day_sonnet` to `sonnet`, and `weekly_scoped` limits map to
  keys named after the model display name. The plan combines
  `subscriptionType` with the multiplier in `rateLimitTier`; the account label
  comes from `oauthAccount.displayName` in the Claude config JSON files. HTTP
  429 reports the `retry-after` delay.

## GitHub Copilot

- **Detection** — `GITHUB_TOKEN`, then `GH_TOKEN`; otherwise a `gh` signed-in
  state via the GitHub CLI.
- **Credentials** — the `env:` token reference, or the output of
  `gh auth token` (5 second timeout).
- **Usage** — `GET https://api.github.com/copilot_internal/user` with editor
  and plugin version headers and `X-Github-Api-Version: 2025-04-01`. Quota
  snapshots map to `credits` (premium interactions), `chat`, and `completions`
  as percent consumed; unlimited or zero entitlements are hidden. When no
  snapshot yields data, `limited_user_quotas` against `monthly_quotas` is used
  instead, and accounts without token-based billing report that usage data is
  unavailable. The plan comes from `copilot_plan`, account info from `login`,
  and reset times from the quota reset dates.

## OpenRouter

- **Detection** — `OPENROUTER_API_KEY` set in the environment.
- **Credentials** — `env:` or `key:` secret reference on the account.
- **Usage** — two parallel `GET` calls with the bearer key:
  `https://openrouter.ai/api/v1/credits` and `https://openrouter.ai/api/v1/key`.
  Credits yield a USD balance (`total_credits - total_usage`). The key endpoint
  yields `today`, `week`, and `month` spend from the daily/weekly/monthly usage
  fields, plus a limit consumption resource (`keyLimit`, `dailyLimit`,
  `weeklyLimit`, or `monthlyLimit` depending on `limit_reset`) with limit,
  remaining, and utilization. `is_free_tier` decides `Free tier` vs
  `Pay as you go`, and the key `label` becomes the account info. When both
  endpoints reject the key, the account reports an invalid key.

## OpenCode

Two products share the provider block:

- **Go** — API key from a secret reference. Usage comes from
  `GET https://opencode.ai/zen/go/v1/usage`: `rolling`, `weekly`, and
  `monthly` percent windows with optional reset times.
- **Zen** — authenticated `Cookie` from a secret reference. Usage comes from
  `GET https://opencode.ai/_server` server functions with browser-like
  headers (`Origin: https://opencode.ai`, a Chrome user agent, and a workspace
  billing `Referer`): the workspaces function auto-detects a `wrk_…` workspace
  id unless `workspaceID` is configured, then the subscription function is
  scraped for `rollingUsage` and `weeklyUsage` percentages and `resetInSec`
  values. HTTP 401/403 report an invalid or expired cookie.

## Antigravity

- **Detection** — requires the `antigravity` or `agy` binary (found via
  `ANTIGRAVITY_CLI_PATH`, `PATH`, or the well-known bin directories) and the
  `~/.gemini/antigravity-cli` directory to exist.
- **How it reads usage** — no stored token is used. If a matching CLI process
  is already running (found via `ps`), mysubs talks to its local language
  server directly. Otherwise it spawns the CLI under `script` (macOS and
  Linux), discovers its listening ports with `lsof`, polls the loopback
  endpoint for up to 30 seconds, reads the quota, and kills the process tree.
- **Endpoint** — `POST` to `127.0.0.1` on the discovered port, scheme
  `https` then `http` with self-signed certificates accepted, calling
  `exa.language_server_pb.LanguageServerService/RetrieveUserQuotaSummary` and
  `/GetUserStatus`.
- **Mapping** — quota buckets map to `session` (`gemini-5h`), `weekly`
  (`gemini-weekly`), `other` (`3p-5h`), and `otherWeekly` (`3p-weekly`);
  `remainingFraction` becomes percent used with the bucket's window seconds
  and reset time. Account info and plan come from the user status response.
