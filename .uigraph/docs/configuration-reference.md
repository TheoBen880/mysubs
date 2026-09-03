# Mysubs Configuration Reference

Mysubs is configured by a single JSON file. Every field below is enforced by the
zod schemas in `src/core/config.ts`, `src/core/schema.ts`, and the per-provider
config modules in `src/providers/*/config.ts`.

## Config file location

The file is read from `$XDG_CONFIG_HOME/mysubs/config.json`, falling back to
`~/.config/mysubs/config.json`. The file is optional: when it is missing the
tool runs with defaults and relies entirely on automatic account detection.

## Root fields

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `cacheTTL` | number (milliseconds) or `ms` duration string such as `"5m"` | `"1m"` | How long a fetched account result stays in the cache. Parsed by the `ms` package; negative or non-finite values are rejected. |
| `detect` | boolean | `true` | Master switch for automatic account detection. `false` disables detection for every provider. |
| `contrast` | number 0–1 | `0.4` | Saturation of the usage bar colors. |
| `nerdFont` | boolean | `false` | Use the Nerd Font glyph for bars instead of the plain block character. |
| `maxWidth` | number ≥ 80 | `120` | Upper bound for the rendered line width; the actual width also honors the terminal columns. |
| `$schema` | string | — | Optional JSON Schema pointer; mysubs publishes the schema on its `schema` branch. |

Unknown root keys are ignored, but a value that violates a field's type or range
makes the whole config invalid and the run exits 1 with
`<path> is invalid`.

## Provider blocks

Each provider accepts a block keyed by its name: `codex`, `claude`,
`antigravity`, `copilot`, `openrouter`, `opencode`. A block holds:

- `accounts` — a map of account keys to account objects. The key is what
  `mysubs <provider>:<key>` selects.
- Provider options — every provider supports `cache` (default `true`, cache this
  provider's results) and `detect` (default `true`, run detection for this
  provider even when the root `detect` is on).

### Account fields

Common optional fields on every account:

- `name` — display name shown next to the provider. Selection still uses the
  account key.
- `info` — replaces the account info reported by the provider. A string
  replaces it, `false` hides it.

Provider-specific fields:

| Provider | Account shape |
| --- | --- |
| `codex` | `{ "configDir": "…" }` for a native OpenAI OAuth login directory, or `{ "adapter": "opencode-oauth", "authPath": "…" }` to read the OpenAI login stored by the opencode CLI. `authPath` overrides the default opencode auth file. |
| `claude` | `{ "configDir": "…" }` — directory holding Claude credentials (default `~/.claude`). |
| `antigravity` | `{ "cliPath": "…" }` — explicit path to the `antigravity`/`agy` binary. |
| `copilot` | `{ "source": "token", "token": "<secret-ref>" }` or `{ "source": "gh" }` to read the token from the GitHub CLI. |
| `openrouter` | `{ "apiKey": "<secret-ref>" }` |
| `opencode` | `{ "product": "go", "apiKey": "<secret-ref>" }` or `{ "product": "zen", "cookie": "<secret-ref>", "workspaceID": "wrk_…" }`. `workspaceID` skips workspace auto-detection. |

`~` at the start of a path field is expanded to the home directory.

## Secret references

API keys and cookies are never written into the config in plain form. The
schema rejects values that are not references; both forms resolve at run time:

- `env:NAME` — read environment variable `NAME`. Unset or empty variables are
  an error for that account.
- `key:NAME` — read keyring entry `NAME` of service `mysubs`, stored with
  `mysubs key set NAME`.

## Environment variables read for behavior

| Variable | Purpose |
| --- | --- |
| `XDG_CONFIG_HOME` | Moves the config file location. |
| `XDG_CACHE_HOME` | Moves the cache file location. |
| `XDG_DATA_HOME` | Moves the opencode `auth.json` location used by the codex opencode-oauth adapter. |
| `CODEX_HOME` | Primary Codex config directory checked during detection. |
| `CLAUDE_CONFIG_DIR` | Primary Claude config directory checked during detection. |
| `CLAUDE_SECURESTORAGE_CONFIG_DIR` | Alternative directory holding Claude's `.credentials.json`. |
| `GITHUB_TOKEN`, `GH_TOKEN` | Copilot token sources, checked in this order. |
| `OPENROUTER_API_KEY` | OpenRouter account detected when set. |
| `OPENCODE_API_KEY` | OpenCode Go account detected when set. |
| `ANTIGRAVITY_CLI_PATH` | Overrides the antigravity binary lookup. |
| `NO_COLOR` | Disables colored output. |

## Cache

Successful per-account results are stored in
`$XDG_CACHE_HOME/mysubs/cache.json` (default `~/.cache/mysubs/cache.json`).
Cache keys are HMAC-SHA256 digests of the provider name and account object,
keyed by a random secret kept in the `cache-secret` keyring entry; when the
keyring is unavailable, caching is skipped for that run. Entries expire after
`cacheTTL`, only results without an error are written, and `--force` (or
`--verbose`) bypasses reads. Corrupt or stale entries are ignored, and writes
are atomic through a temporary file.
