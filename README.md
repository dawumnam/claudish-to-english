# claudish-to-english

A Claude Code plugin that shows a **plain-English rewrite** of each assistant
message, produced by a **headless CLI model** — the
[Gemini CLI](https://github.com/google-gemini/gemini-cli) by default, or
headless Claude Code (`claude -p`). It is **display-only**: Claude's own
reasoning and the saved transcript keep the original text — only what you read
on screen changes.

> This is a fork of [gvzdv/claudish-to-english](https://github.com/gvzdv/claudish-to-english).
> Differences from upstream: the ollama backend is removed, `gemini` (default)
> and `claude` CLI backends are added, and the display mode defaults to
> `replace` instead of `append`.

An optional second hook rewrites **Markdown files** into plain English when they
are written or edited (opt-in, off by default).

> Status: working prototype. Every hook fails **open** — if anything goes wrong
> (CLI missing, not authenticated, timeout), you simply see Claude's original
> text. The plugin can never swallow or corrupt an answer.

---

## Requirements

| Requirement | Why | Install |
|---|---|---|
| `gemini` CLI, authenticated | The default rewriter | `brew install gemini-cli`, then log in or set `GEMINI_API_KEY` |
| `jq` | Parses hook JSON | ships with macOS; else `brew install jq` |
| `claude` CLI | Only for `CLAUDISH_BACKEND=claude` | you already have it |

Sanity-check the rewriter once:

```bash
echo hi | gemini -m gemini-3.6-flash
```

**If the rewriter isn't ready, the plugin does nothing to your text** — Claude's
output shows normally, unchanged. That is by design, not a bug. The first time a
rewrite is skipped in a session it tells you why: the display hook shows a
one-line notice, the Markdown hook a `systemMessage` (once per session; set
`CLAUDISH_NOTICE=0` to silence).

Every rewrite is a normal Gemini/Claude API or subscription call — it costs
usage, and the message text is sent to Google (gemini backend) or Anthropic
(claude backend).

---

## Install

From the local checkout (serves as its own marketplace):

```shell
/plugin marketplace add /Users/dawumnam/projects/claudish-to-english
/plugin install claudish-to-english@dawumnam-plugins
```

Or from the GitHub fork:

```shell
/plugin marketplace add dawumnam/claudish-to-english
/plugin install claudish-to-english@dawumnam-plugins
```

If the install summary says `Run /reload-plugins to activate.`, run that command.

**Try before installing** (loads it for one session, no install):

```bash
claude --plugin-dir /Users/dawumnam/projects/claudish-to-english
```

Run `/reload-plugins` after edits; if it doesn't load, check the `/plugin`
**Errors** tab.

---

## Configuring the plugin

All behavior is controlled by `CLAUDISH_*` environment variables (full list in
[Configuration](#configuration-env-vars) below). When you install from a
marketplace, set them in Claude Code's **`env` block in `settings.json`** — do
**not** edit the plugin's own `hooks/hooks.json`, which lives in the read-only
plugin cache (`~/.claude/plugins/cache/…`) and is overwritten on every update.

For a personal, all-projects setup, use `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDISH_GEMINI_MODEL": "gemini-3.6-flash",
    "CLAUDISH_MODE": "replace"
  }
}
```

The hooks are subprocesses Claude Code spawns, so they inherit these. A few
things to know:

- **Restart Claude Code after editing `env`.** The value is captured at launch,
  so a running session keeps the old one.
- **`env` does not merge across scopes.** The highest-precedence settings file
  that defines `env` supplies the *entire* block — it isn't combined with lower
  scopes. Precedence: managed → local → project → user. Keep all your
  `CLAUDISH_*` vars in whichever file wins.
- **Scopes:** `~/.claude/settings.json` (all your projects) ·
  `.claude/settings.json` (shared with a repo, checked in) ·
  `.claude/settings.local.json` (just you, just this repo).

Quick one-off without editing a file — hooks inherit the launching shell:

```bash
CLAUDISH_BACKEND=claude claude
```

To confirm the hook is firing, set `CLAUDISH_DEBUG=1` and watch
`"$TMPDIR"/claudish-to-english/debug.log`.

---

## How the display hook works

Claude Code fires the `MessageDisplay` event **once per streamed chunk**, not
once per message. Each fire is a separate process carrying `message_id`,
`index`, a `final` flag, and this chunk's `delta` (a text fragment, not the
whole message). So the hook **buffers every delta** to a temp file (keyed by
`message_id`) and only calls the model on the **final** chunk, once the whole
message is known:

```
chunk 0 (final:false) ─┐
chunk 1 (final:false) ─┤ append each delta to $TMPDIR/claudish-to-english/<session>/<message>/<index>.part
chunk 2 (final:false) ─┘  → emit "" (replace) or nothing (append)
chunk 3 (final:true)  ──► reconstruct full message → call the CLI once → show the rewrite
                          → delete the buffer
```

On that final chunk it also reads the **original user question** from the
transcript and passes it to the model as **context only** — to keep the rewrite
on-topic. The model is told never to answer or repeat the question; it only
rewrites the assistant's message.

### Display modes

| `CLAUDISH_MODE` | On screen | Notes |
|---|---|---|
| `replace` (default) | Only the simplified version (original chunks suppressed while streaming). | Appears all at once after LLM latency; on failure it re-shows the full original. If the hook is hard-killed at its 60s `hooks.json` timeout, that message's original may be lost from the display (never from the transcript). |
| `append` | Original streams normally, then a `💬 In plain English:` block is appended. | Safest. No streaming loss; if the LLM fails you just don't get the extra block. |

---

## Markdown file rewrite (optional second hook)

A `PostToolUse` hook (`rewrite-md.sh`) rewrites Markdown **files** into plain
English when they are written or edited. Unlike the display hook, this changes
bytes on disk.

**Opt-in by directory.** It does nothing unless `CLAUDISH_MD_DIR` is set, and it
only touches `*.md` files whose resolved path is inside that directory. Every
other `README`, `CLAUDE.md`, or doc you edit is left alone.

| `CLAUDISH_MD_MODE` | Result | Notes |
|---|---|---|
| `sibling` (default) | Writes `NAME.plain.md` next to `NAME.md`. | Non-destructive; the original is never touched. |
| `overwrite` | Replaces `NAME.md` in place. | Adds a `<!-- claudish-to-english:rewritten -->` marker so a re-write is skipped (idempotent). A weak model can degrade real docs — use with care. |

In both modes: YAML frontmatter is split off and re-attached **verbatim**, fenced
code is left to the model instruction, short files are skipped, and the write is
atomic. Fail-open here means the file is left **exactly as the agent wrote it**.

Long documents can be slow to rewrite. This hook allows up to
`CLAUDISH_MD_TIMEOUT` (150s) inside a 180s `PostToolUse` hook budget; if a
rewrite still times out you get the one-time notice above — raise those limits,
or use a faster model.

Enable it for one directory, in sibling mode (the safe default), the same way
as every other setting — the `env` block of your `settings.json`:

```json
{
  "env": {
    "CLAUDISH_MD_DIR": "/ABS/PATH/docs/plain",
    "CLAUDISH_MD_MODE": "sibling"
  }
}
```

In `overwrite` mode the marker comment is written **after** any YAML
frontmatter, so the frontmatter stays on line 1 where parsers expect it.

---

## Configuration (env vars)

| Var | Default | Meaning |
|---|---|---|
| `CLAUDISH_ENABLED` | `1` | Master switch. `0` = pass everything through. |
| `CLAUDISH_BACKEND` | `gemini` | `gemini` (headless Gemini CLI) or `claude` (headless `claude -p`). |
| `CLAUDISH_GEMINI_MODEL` | `gemini-3.6-flash` | `gemini` backend only: model passed to the Gemini CLI (`-m`). |
| `CLAUDISH_CLAUDE_MODEL` | *(unset)* | `claude` backend only: model passed to `claude -p`. Unset = the session's model if the hook payload carries one, else your CLI default. |
| `CLAUDISH_MODE` | `replace` | `replace` or `append` (display hook). |
| `CLAUDISH_MIN_CHARS` | `0` (display) / `200` (Markdown hook) | Skip messages/files whose prose (code stripped) is shorter than this. The display hook defaults to `0` — every message is rewritten. |
| `CLAUDISH_STUB` | `0` | `1` = deterministic stub instead of the model (for testing display mechanics). |
| `CLAUDISH_TIMEOUT` | `45` | LLM client timeout for the **display** hook (seconds; needs GNU `timeout`/`gtimeout` on PATH, else the hook's own 60s ceiling applies). |
| `CLAUDISH_MD_TIMEOUT` | `150` | LLM client timeout for the **Markdown file** hook (seconds). Higher on purpose — rewriting a long doc is slow. Keep it below the `PostToolUse` hook `timeout` (180s). |
| `CLAUDISH_DEBUG` | `0` | `1` = write a debug log to `$TMPDIR/claudish-to-english/`. |
| `CLAUDISH_NOTICE` | `1` | `1` = show a one-time, once-per-session notice when a rewrite is skipped because the CLI failed or timed out (display hook shows it on screen; Markdown hook uses a `systemMessage`). `0` = stay fully silent (pure fail-open). |
| `CLAUDISH_MD_DIR` | *(unset)* | **Markdown hook opt-in.** Only `*.md` under this directory is rewritten. Unset = the Markdown hook does nothing. |
| `CLAUDISH_MD_MODE` | `sibling` | `sibling` (`NAME.plain.md`) or `overwrite` (in place). |
| `CLAUDISH_MD_SUFFIX` | `plain` | Sibling infix: `NAME.<suffix>.md`. |

In `hooks/hooks.json` the display hook (`MessageDisplay`) has a 60s `timeout` and
the Markdown hook (`PostToolUse`) has a 180s `timeout` — the file hook is higher
because rewriting a long document can take a couple of minutes.
`CLAUDISH_TIMEOUT` and `CLAUDISH_MD_TIMEOUT` keep the LLM call itself bounded
below those ceilings, so it fails open cleanly instead of being killed mid-write.

**Quick kill switch:** set `CLAUDISH_ENABLED=0`, or disable the plugin.

---

## Privacy / egress

The rewriter calls a **hosted model**: each rewritten message (and, for the
Markdown hook, file contents) is sent to Google (gemini backend) or Anthropic
(claude backend). The claude backend sends text to Anthropic that was already
part of your Claude conversation; the gemini backend sends it to a second
provider — make sure that's acceptable for what you're working on.

---

## Layout

```
claudish-to-english/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # so the repo can be added as a marketplace directly
├── hooks/
│   └── hooks.json          # MessageDisplay -> rewrite.sh ; PostToolUse -> rewrite-md.sh
├── rewrite.sh              # display-rewrite hook
├── rewrite-md.sh           # markdown-file rewrite hook (opt-in)
├── LICENSE
└── README.md
```

## License

MIT — see [LICENSE](./LICENSE).
