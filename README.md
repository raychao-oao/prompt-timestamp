# prompt-timestamp

**Your AI agent doesn't know what time it is. One hook fixes it.**

A minimal Claude Code plugin that stamps every user prompt with the current date and time:

```
[2026-07-08 18:29 +0800]
```

## Why

LLMs have no clock. Claude Code injects today's date **once, at session start** — and that's it. As the session runs on, the model has no idea whether your last message came ten minutes or ten hours later. This causes real problems:

- **Relative time breaks.** "Do this tomorrow", "before end of day", "is this CVE from today?" — the model guesses from a stale date.
- **Long-lived sessions drift.** A session left open overnight still thinks it's yesterday.
- **Freshness judgment fails.** Whether a status check from "earlier" is 5 minutes or 5 hours old changes whether it should be re-verified. The model can't tell.
- **Log correlation is blind.** Paste a log with timestamps and the model can't align it with "now".

This plugin fixes all of that with a single `UserPromptSubmit` hook: every message you send carries a timestamp, so the model always knows the current time — and how much time passed between messages.

Cost: ~10 tokens per message. Effectively free.

## Install

```bash
claude plugin marketplace add raychao-oao/prompt-timestamp
claude plugin install prompt-timestamp@prompt-timestamp
```

Restart Claude Code after installing (hooks load at session start).

## What it does

On every prompt you submit, the hook runs:

```bash
date '+[%Y-%m-%d %H:%M %z]'
```

The output is injected as context alongside your message. The `%z` offset (`+0800`, `-0500`, …) makes the timestamp unambiguous in any timezone — no configuration needed.

That's the whole plugin. No dependencies, no network access, no state.

## Prefer raw settings over a plugin?

Add this to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "date '+[%Y-%m-%d %H:%M %z]'"
          }
        ]
      }
    ]
  }
}
```

> **Note:** don't run both the plugin and the settings.json hook — you'll get double timestamps.

## License

MIT
