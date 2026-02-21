# autoclaude

Automated Claude Code runner designed for use with Claude Code subscription plans where usage limits are frequently hit. Runs Claude in headless, fully autonomous mode and automatically waits for the rate limit window to reset before resuming — so you can walk away and let it work.

> ⚠️ **Security advisory:** This tool runs Claude with all permission prompts disabled. Claude can read and write files, run shell commands, and make network requests without confirmation. Read the [Security Considerations](#security-considerations) section before use.

## How It Works

`autoclaude` launches `claude` with `--dangerously-skip-permissions` (yolo mode), streams its JSON output, and monitors for rate limit events. When a usage limit is hit, it reads the reset timestamp from the event stream, sleeps until 2 minutes after the reset, then resumes the same session automatically. This repeats until Claude exits cleanly.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed (default path: `/usr/local/bin/claude`)
- `jq`
- `bash` 4+

## Usage

```bash
./autoclaude                        # Start a new session
                                    # Uses `autoclaude/prompt.md` as the prompt

./autoclaude --continue             # Continue the most recent Claude session
./autoclaude --resume               # Resume the session ID saved from a previous run
./autoclaude --prompt my-prompt.md  # Use a specific prompt file
```

### Flags

| Flag | Description |
|------|-------------|
| *(none)* | Start a fresh session |
| `--continue` | Pass `--continue` to Claude to resume the most recent conversation |
| `--resume` | Resume using the session ID saved in `.autoclaude/session_id` |
| `--prompt <file>` | Load the prompt from the given file instead of the default location |

## Configuration

### Prompt file

By default, `autoclaude` looks for a prompt in:

1. `autoclaude/prompt.md`
2. `autoclaude/prompt.txt`

Create one of these files with the task you want Claude to work on. You can override the path at runtime with `--prompt`.

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_BIN` | `/usr/local/bin/claude` | Path to the `claude` binary |

### Model

The model is set to `sonnet`. Edit the `MODEL` variable near the top of the script to change it.

## The `.autoclaude` directory

`autoclaude` creates a `.autoclaude/` directory in the project root to store runtime state and logs. **Add it to your `.gitignore`.**

| File | Description |
|------|-------------|
| `autoclaude.log` | Timestamped log of all runs, tool calls, rate limit events, and session activity |
| `session_id` | The session ID from the most recent completed run, used by `--resume` |
| `last_session.jsonl` | Raw stream-json event lines from the last Claude invocation |
| `run_state` | Transient file written during a run to pass session ID and reset timestamps out of the stream processor subshell |

```gitignore
.autoclaude/
```

## Security Considerations

`autoclaude` passes `--dangerously-skip-permissions` to Claude on every run. This disables all permission prompts, meaning Claude will read files, write files, execute arbitrary shell commands, and make network requests without asking. It also disables trust verification for new codebases and MCP servers (a side effect of running with `-p` in non-interactive mode).

**Do not run this against a codebase or on a machine where unreviewed, autonomous actions could cause damage.**

### Risks

- **Destructive commands** — Claude can run any shell command, including `rm`, `git reset --hard`, database migrations, deployments, etc.
- **Credential exposure** — Without restrictions, Claude can read `.env` files, AWS credential files, SSH keys, and anything else on disk.
- **Network access** — Claude can make arbitrary web requests (`curl`, `wget`, `WebFetch`) which could exfiltrate data or download and execute remote content.
- **Prompt injection** — If Claude reads files containing adversarial instructions (e.g., from fetched web content or untrusted dependencies), those instructions execute without a human in the loop.

### Recommended `.claude/settings.json`

Lock down what Claude can touch by adding a project-level settings file. Three example configs are included at increasing levels of strictness — copy the one that fits your situation to `.claude/settings.json`:

| File | What it does |
|------|-------------|
| [claude-settings.json.minimal](claude-settings.json.minimal) | Denies access to credential files only. Claude can still run arbitrary shell commands and make network requests. |
| [claude-settings.json.standard](claude-settings.json.standard) | Denies credential file access and blocks outbound network tools (`curl`, `wget`, `WebFetch`, `WebSearch`). |
| [claude-settings.json.strict](claude-settings.json.strict) | Enables bash sandboxing with an explicit network domain allowlist. The most isolated option. |

If you want to prevent `--dangerously-skip-permissions` from being used at all in a shared or managed environment, set this in your managed settings:

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  }
}
```

Note that setting `disableBypassPermissionsMode` will break `autoclaude` entirely — it is intended for environments where you want to prevent autonomous runs altogether.
