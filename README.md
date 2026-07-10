# prompt-timestamp

**Your AI agent doesn't know what time it is. One hook fixes it.**

A minimal Claude Code plugin that stamps every user prompt with the current date and time:

```
[2026-07-08 20:53 +0800 | calculate time since previous stamp]
```

Not just "what time is it now" — the trailing hint prompts the model to look back at the previous stamp in the conversation and work out **how much time passed between messages**: how long a session sat idle, whether that status check was 5 minutes or 5 hours ago.

## Why

LLMs have no clock. Claude Code injects today's date **once, at session start** — and that's it. As the session runs on, the model has no idea whether your last message came ten minutes or ten hours later. This causes real problems:

- **Relative time breaks.** "Do this tomorrow", "before end of day", "is this CVE from today?" — the model guesses from a stale date.
- **Long-lived sessions drift.** A session left open overnight still thinks it's yesterday.
- **Freshness judgment fails.** Whether a status check from "earlier" is 5 minutes or 5 hours old changes whether it should be re-verified. The model can't tell.
- **Log correlation is blind.** Paste a log with timestamps and the model can't align it with "now".

This plugin fixes all of that with a `UserPromptSubmit` hook: every message you send carries a timestamp, so the model always knows the current time — and how much time passed between messages. Optional turn-ending stamps make long terminal sessions easier for humans to scan too.

Cost: ~10 tokens per message. Effectively free.

## Install

```bash
claude plugin marketplace add raychao-oao/prompt-timestamp
claude plugin install prompt-timestamp@prompt-timestamp
```

Or skip the plugin entirely and paste this into Claude Code — it will set the hook up for you:

```
Read https://github.com/raychao-oao/prompt-timestamp and set up the hook it describes.
```

Restart Claude Code after installing (hooks load at session start).

## What it does

On every prompt you submit, the hook runs:

```bash
date '+[%Y-%m-%d %H:%M %z | calculate time since previous stamp]'
```

The output is injected as context alongside your message. The `%z` offset (`+0800`, `-0500`, …) makes the timestamp unambiguous in any timezone — no configuration needed.

The `| calculate time since previous stamp` part is a **static hint, not a computed value**: the previous timestamp already lives in the conversation history, so the model can derive the gap itself. No state file, no cross-session bookkeeping, no concurrency issues — the context window *is* the state.

That's the whole plugin. No dependencies, no network access, no state.

Two different timestamps solve two different problems:

| Stamp | Who sees it | Why it exists |
| --- | --- | --- |
| Prompt timestamp | The model | Lets the assistant reason about current time and elapsed time between user messages |
| Turn stamp | You | Gives long terminal sessions a visible "this reply ended here" marker |

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
            "command": "date '+[%Y-%m-%d %H:%M %z | calculate time since previous stamp]'"
          }
        ]
      }
    ]
  }
}
```

> **Note:** don't run both the plugin and the settings.json hook — you'll get double timestamps.

## Bonus: human-visible turn stamps (`turn-stamp`)

`prompt-timestamp` feeds time to the **model**. If you also want a timestamp *you* can see at the end of each turn, this repo ships a second plugin from the same marketplace:

```bash
claude plugin install turn-stamp@prompt-timestamp
```

Each turn then ends with a one-line `⏱ 2026-07-10 10:35` in the UI. The `systemMessage` goes to your screen only — it never enters the model's context, so it costs zero tokens. The two plugins are independent: install either one, or both.

Prefer raw settings? Add this to `~/.claude/settings.json` instead:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "printf '{\"systemMessage\":\"⏱ %s\"}\\n' \"$(date '+%Y-%m-%d %H:%M')\"",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

> **Note:** don't run both the `turn-stamp` plugin and the settings.json hook — you'll get double stamps.

Or let AI do this for you — paste this into Claude Code:

```
Read the Bonus section of https://github.com/raychao-oao/prompt-timestamp
and add the Stop hook it describes to my ~/.claude/settings.json.
```

Related built-ins worth knowing: `"showMessageTimestamps": true` in settings shows per-message arrival times in the transcript view (`Ctrl+O`), and turn durations ("Cooked for 16s") are on by default.

## Other tools: Codex CLI

Codex CLI supports the same model-facing hook, just with a different output schema — `UserPromptSubmit` there requires JSON, not plain stdout. Add this to `~/.codex/hooks.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "date '+{\"hookSpecificOutput\":{\"hookEventName\":\"UserPromptSubmit\",\"additionalContext\":\"[%Y-%m-%d %H:%M %z | calculate time since previous stamp]\"}}'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

After editing, run `/hooks` inside Codex CLI to review and trust the change.

If you also want a human-visible marker when each assistant reply finishes, add a `Stop` hook too:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "date '+{\"hookSpecificOutput\":{\"hookEventName\":\"UserPromptSubmit\",\"additionalContext\":\"[%Y-%m-%d %H:%M %z | calculate time since previous stamp]\"}}'",
            "timeout": 5
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "date '+{\"continue\":false,\"stopReason\":\"reply ended at [%Y-%m-%d %H:%M %z]\"}'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

The `UserPromptSubmit` stamp goes into model context. The `Stop` stamp is for you: Codex renders it after the assistant finishes, like this:

```text
• Stop hook (stopped)
  stop: reply ended at [2026-07-10 18:20 +0800]
```

Restart Codex after adding a new hook event such as `Stop`; existing sessions may not pick up newly added event types. The examples use `date` to emit the whole JSON object, which avoids inline shell variables and quoting problems.

Codex schema notes:

- `UserPromptSubmit` output must be JSON, not raw text.
- `additionalContext` belongs under `hookSpecificOutput`.
- `hookSpecificOutput.hookEventName` must match the event name.
- Do not output `"decision":"allow"`; Codex only accepts blocking decisions there.
- The `Stop` hook example intentionally uses `continue:false` because that is what makes Codex render the visible `stop:` line after the assistant reply has finished.

Both Codex examples are inline commands. No extra script file is required.

## Other tools: Antigravity CLI (`agy`)

`agy` already injects the current local time into every prompt's `<ADDITIONAL_METADATA>` (`The current local time is: ...`), so no `PreInvocation` hook is needed for the model-facing half. And its Stop-hook path is actively hostile to printing: `agy`'s TUI repaints the screen buffer on every turn, so a literal `\n` from a hook scrambles the readline cursor, and an ANSI-positioned line gets wiped by the next repaint anyway.

The fix that works: skip hooks entirely and add two Rules to `~/.gemini/GEMINI.md` (agy's global instructions file), so the model does the timestamping itself as part of its normal reply text:

```markdown
## Time-Awareness
- Every time the user speaks, compare the `The current local time is` timestamp in the current and prior `<ADDITIONAL_METADATA>` blocks.
- Actively compute and stay aware of elapsed time between turns, to judge system state, log freshness, and whether re-verification is needed.

## Turn-Stamp Rule
- So the user can see the time of each turn in the terminal, print a small timestamp on the last line of every reply, formatted as: `⏱ YYYY-MM-DD HH:MM`.
- Base it on the local time from that turn's `<ADDITIONAL_METADATA>`.
```

Because the stamp is emitted as ordinary Markdown in the reply body, `agy` renders it natively — no cursor or input-box interference.

**Trade-off vs. the hook-based versions**: this is instruction-following, not a guaranteed mechanism. A hook always fires; a Rule can be forgotten by the model on a given turn, especially in long sessions or after context compaction. If the `⏱` line stays consistent in practice, leave it as-is — but this is inherently less reliable than `turn-stamp`'s Stop hook.

## License

MIT
