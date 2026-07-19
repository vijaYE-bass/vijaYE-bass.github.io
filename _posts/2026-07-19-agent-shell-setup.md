---
layout: post
title: "Setting Up agent-shell: an ACP Client for Claude (and friends) in Emacs"
date: 2026-07-19
---

[agent-shell](https://github.com/xenodium/agent-shell) is a native Emacs shell
for talking to LLM coding agents over [ACP](https://agentclientprotocol.com/)
(Agent Client Protocol) — the same open protocol behind Zed's agent panel.
Unlike a plain terminal wrapper around a CLI, it speaks ACP directly, which
gets you structured permission modes, richer session state, and a UI that
isn't just scraping terminal output.

It works with more than Claude — Codex, Gemini CLI, Goose, Cursor, Qwen Code,
and a growing list of others all speak ACP too — but this post covers getting
Claude Code running end to end, since that's the one most people reach for
first.

This is a from-scratch setup guide. If you want to see the specific
customizations, keybindings, and gotchas I ran into after installing it, see
the [companion post]({% post_url 2026-07-19-claude-in-emacs %}).

## Prerequisites

- Emacs 29+
- Node.js (for the ACP adapter below)
- A Claude Code subscription or Anthropic API key

## 1. Install the ACP adapter for Claude

agent-shell doesn't talk to the `claude` CLI directly — it talks to a small
ACP adapter that wraps it:

```bash
npm install -g @agentclientprotocol/claude-agent-acp
```

The `-g` flag matters: the binary needs to be on your `PATH`. Verify it:

```bash
which claude-agent-acp
```

If you don't already have the Claude Code CLI itself, install it too (see
[Claude Code's own docs](https://code.claude.com/docs/en/overview)) and run it
once outside Emacs to log in.

## 2. Install agent-shell

Both `agent-shell` and its dependency `acp.el` are on MELPA now, so a plain
`use-package` block is all you need — no custom recipes required:

```elisp
(use-package agent-shell
  :ensure t)
```

`use-package` will pull in `acp.el` and
[shell-maker](https://github.com/xenodium/shell-maker) (the comint-based shell
engine agent-shell is built on) automatically as dependencies.

If you're on Doom Emacs, use `package!` instead:

```elisp
(package! shell-maker)
(package! acp)
(package! agent-shell)
```

then `doom sync` and restart.

## 3. Authenticate

Pick one, depending on how you use Claude Code:

```elisp
;; Browser-based login (default, works with a Claude subscription)
(setq agent-shell-anthropic-authentication
      (agent-shell-anthropic-make-authentication :login t))

;; Or an API key
(setq agent-shell-anthropic-authentication
      (agent-shell-anthropic-make-authentication :api-key "your-api-key-here"))

;; Or an OAuth token from `claude setup-token`
(setq agent-shell-anthropic-authentication
      (agent-shell-anthropic-make-authentication :oauth "your-oauth-token-here"))
```

If you'd rather not put a literal key in your config, both `:api-key` and
`:oauth` accept a function instead of a string — read it from
`auth-source-pass` or wherever you keep secrets.

## 4. Start a session

- `M-x agent-shell` — a generic picker across every agent you've configured;
  reuses an existing shell if one's already running, or `C-u M-x agent-shell`
  to force a new one.
- `M-x agent-shell-anthropic-start-claude-code` — jumps straight to a new
  Claude Code session, skipping the picker.

Either way you'll land in a normal Emacs buffer with agent-shell's own
comint-based prompt — send a message, and you're off.

## macOS: the full case

The steps above are OS-agnostic, but macOS has enough of its own texture —
installing Emacs itself, a PATH gotcha that silently breaks step 1 for a lot
of people — that it's worth walking through end to end.

### Installing Emacs

Three common routes, roughly in order of how much you want to fuss with it:

- **Prebuilt app** — [Emacs for Mac OS
  X](https://emacsformacosx.com/) — download, drag to `/Applications`, done.
- **Homebrew core** — `brew install emacs` — quick, but the formula's default
  build has historically lagged behind on GUI polish (native comp, better
  window-system integration).
- **[emacs-plus](https://github.com/d12frosted/homebrew-emacs-plus)** — a
  Homebrew tap most macOS Emacs users end up on for a fully native,
  native-comp build with more up-to-date options:

  ```bash
  brew tap d12frosted/emacs-plus
  brew install emacs-plus@30
  ```

  Check the tap's README for the current recommended flags (icon variants,
  native full-screen, etc.) — they change between Emacs versions.

### Node, for the npm step

Step 1 above needs `npm`. If you don't already have Node:

```bash
brew install node
```

### The GUI PATH gotcha (this is the one that bites people)

Emacs.app launched from Finder, Spotlight, or the Dock is **not** a login
shell — it never sources your `.zshrc`/`.zprofile`, so it doesn't inherit
whatever `PATH` your terminal has. Homebrew's bin directory (`/opt/homebrew/bin`
on Apple Silicon, `/usr/local/bin` on Intel) and npm's global bin directory
both live outside Emacs's PATH in that case.

Concretely: `which claude-agent-acp` works fine in Terminal right after step 1,
but inside a GUI-launched Emacs, `(executable-find "claude-agent-acp")` comes
back `nil` — and agent-shell just silently can't find the agent, with no
obvious error pointing at PATH as the cause.

Fix: [exec-path-from-shell](https://github.com/purcell/exec-path-from-shell).
It launches your shell once at startup, captures its environment, and copies
it into Emacs:

```elisp
(use-package exec-path-from-shell
  :ensure t
  :config
  (when (memq window-system '(mac ns))
    (exec-path-from-shell-initialize)))
```

Put this near the top of your config, before the `agent-shell` block from
step 2, so `claude-agent-acp` (and anything else from Homebrew or npm) is
already on Emacs's `exec-path` by the time you start a session.

If you always launch Emacs from a terminal instead (`emacs &` or similar) you
won't hit this at all — the inherited shell PATH carries over naturally. It's
specifically the double-click-the-Dock-icon launch path that needs the fix.

### Screenshot capture permission

The first time you run `M-x agent-shell-send-screenshot`, macOS will prompt
for Screen Recording permission under **System Settings → Privacy & Security
→ Screen Recording** for whichever binary is invoking `screencapture` (Emacs,
or Emacs.app). Grant it and re-run the command; on older macOS versions you
may need to relaunch Emacs for the permission to take effect.

With Emacs, Node, and `exec-path-from-shell` in place, steps 1–4 above work
exactly as written.

## Optional: pasting images

If you'll be pasting screenshots into a session, the clipboard-paste and
screenshot-capture commands depend on a small external tool per platform:

| Platform        | Clipboard paste                             | Screenshot capture              |
|------------------|----------------------------------------------|-----------------------------------|
| macOS            | `brew install pngpaste`                       | `screencapture` (preinstalled)    |
| Linux (Wayland)  | `wl-paste` (e.g. `apt install wl-clipboard`)  | `import` (from ImageMagick)      |
| Linux (X11)      | `apt install xclip`                           | `import` (from ImageMagick)      |
| Windows          | `powershell`                                  | not bundled                      |

Everything's detected at runtime via `executable-find`, so just install the
one for your platform and it starts working — no config needed.

## Where to go from here

That's the whole install. From here it's just `M-x agent-shell` whenever you
want Claude in a buffer. The [companion post]({% post_url
2026-07-19-claude-in-emacs %}) covers the sharper edges I hit next: a
keybinding collision with `tab-bar-mode`, getting new sessions to respect
window splits instead of hijacking whatever pane they feel like, and pasting
images from the clipboard.
