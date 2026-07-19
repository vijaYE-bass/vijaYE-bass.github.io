---
layout: post
title: "Wiring Claude Code into Emacs: claude-code-ide and agent-shell"
date: 2026-07-19
---

Claude Code is as much a part of my studio toolchain now as REAPER — it's what
wired up the [reapy MCP bridge]({% post_url 2026-06-24-reaper-mcp-claude %}) in
the first place. This post documents how it's actually plumbed into Emacs,
since it turns out there isn't one integration — there are three, and a
handful of gotchas that only showed up once I actually started using it
split-screen.

## Three separate ways to launch Claude

- **`claude-code-ide.el`** — a full MCP/IDE integration package (session
  resume, diagnostics, an emacs-tools MCP server). Wrapped by my own
  `vxe/claude-code-ide`, bound to `C-c 9`.
- **`claude-code.el`** — a lighter wrapper around the plain `claude` CLI in a
  terminal buffer. Wrapped by `vxe/claude-code`. Loaded, but I never got
  around to binding a key to it.
- **`agent-shell`** — an [ACP](https://agentclientprotocol.com/) client that
  talks to `claude-code-acp` instead of shelling out to the CLI directly. Bound
  to `C-c 0` via `vxe/agent-shell`. No resume prompt, and every call starts
  a fresh, parallel session landing wherever you're currently focused (more on
  that below — it took two attempts to get right).

Three packages that all do roughly the same thing sounds redundant until you
remember they arrived at different times solving different problems, and each
still does something the others don't (MCP tool access vs. ACP's structured
permission modes vs. just wanting a dumb terminal).

## The `C-<tab>` collision

`agent-shell` has its own equivalent of the terminal CLI's Shift-Tab —
switching between a session's permission modes (default → accept-edits → plan
→ bypass-permissions, whatever the agent advertises). Its default keymap binds
*cycling* through them to `C-<tab>`:

```elisp
(defvar-keymap agent-shell-mode-map
  ...
  "C-<tab>" #'agent-shell-cycle-session-mode
  ...)
```

Which is exactly the key `tab-bar-mode` already owns globally for `tab-next`.
Outside an `agent-shell` buffer, `C-<tab>` moves between tabs. Inside one, it
silently did something else — cycled the permission mode instead of moving to
the next tab. Easy to not notice until you're several tabs deep and `C-<tab>`
stops doing what your hands expect.

Fix: unbind it locally so lookup falls through to the global `tab-bar` binding
again, and put mode selection on `C-c <tab>` instead — as a `completing-read`
picker rather than blind cycling, since I'd rather see the list of modes than
guess how many times to hit a key:

```elisp
(with-eval-after-load 'agent-shell
  (define-key agent-shell-mode-map (kbd "C-<tab>") nil)
  (define-key agent-shell-mode-map (kbd "C-c <tab>") #'agent-shell-set-session-mode))
```

`define-key` with a `nil` binding doesn't just no-op — it clears the entry in
that keymap, so the lookup chain continues past it to the parent keymap and
eventually the global map. Confirmed with a throwaway batch instance before
touching the real config:

```
C-<tab> now: nil
```

`C-c <tab>` and the package's own `C-c C-m` (`agent-shell-set-session-mode`)
are now just two keys for the same picker.

## A MOTD so the new binding isn't forgotten

`agent-shell` shows a welcome banner on session start — gated by
`agent-shell-show-welcome-message` (on by default), with the actual text
produced by a per-agent `:welcome-function`. For Claude Code that's
`agent-shell-anthropic--claude-code-welcome-message`. Rather than fork the
function, `:filter-return` advice appends a line to whatever it returns:

```elisp
(defun vxe/agent-shell--motd-append (message)
  (concat message "\n       Pick permission mode: C-c <tab>\n"))
(advice-add 'agent-shell-anthropic--claude-code-welcome-message
            :filter-return #'vxe/agent-shell--motd-append)
```

Every fresh `C-c 0` session now prints the reminder along with the usual ASCII
banner.

## Parallel sessions, and actually respecting the split

My usual flow is to split the frame — code or notes in one pane, Claude in the
other — and I wanted `C-c 0` to just drop a new session into whichever pane
I'm focused in, without disturbing the rest of the layout. First attempt set
`agent-shell-display-action` to a custom function modeled on stock
`display-buffer-same-window`, redirecting away from side windows (helm, etc.)
when needed:

```elisp
(defun vxe/agent-shell--display-buffer (buffer alist)
  (unless (cdr (assq 'inhibit-same-window alist))
    (let ((win (if (window-parameter (selected-window) 'window-side)
                   (seq-find (lambda (w) (not (window-parameter w 'window-side)))
                             (window-list))
                 (selected-window))))
      (when win
        (window--display-buffer buffer win 'reuse alist)))))
```

Looked right in a sandboxed test. Then I actually tried it split-screen and
got a brand new session dropped into a *different* pane than the one I was
looking at. Turns out agent-shell's own startup path doesn't reliably consult
`agent-shell-display-action` for a session's *initial* placement — it can pick
whatever window it wants regardless of that variable, so my fix only ever
covered *later* redisplays, not the launch itself.

The actual fix uses `display-buffer-overriding-action` — the one variable
Emacs checks *before* any package's own display logic, no exceptions:

```elisp
(defun vxe/agent-shell--display-in (window)
  "Return a `display-buffer' action function that only ever targets WINDOW."
  (lambda (buffer alist)
    (when (window-live-p window)
      (window--display-buffer buffer window 'reuse alist))))

(defun vxe/agent-shell ()
  (interactive)
  (let* ((target (selected-window))
         (display-buffer-overriding-action (list (vxe/agent-shell--display-in target))))
    (when (fboundp 'agent-shell-anthropic-start-claude-code)
      (agent-shell-anthropic-start-claude-code))))
(global-set-key (kbd "C-c 0") 'vxe/agent-shell)
```

Capture the window you're in *before* the launch, then pin every display
attempt made during that launch to it. It doesn't matter what agent-shell
tries to do internally — the override wins, so no other window can be
clobbered as a side effect. Confirmed against the actual failure mode in a
sandbox before trusting it: a two-pane split with unrelated buffers in each,
cursor in the bottom one, launch → the bottom pane gets the new session, the
top pane's buffer is provably untouched.

`vxe/agent-shell` dropped its old reuse-an-existing-session logic entirely in
the process — every `C-c 0` now starts an additional, parallel session (no
prefix arg needed), since agent-shell uniquifies buffer names and collisions
were never actually a problem.

## Pasting images into a session

Screenshots come up constantly when debugging anything visual (window
layouts, mixer states, this blog's own rendering). Turns out `agent-shell`
already handles it:

- **`C-y`** just works — it's remapped to `agent-shell-yank-dwim` in
  `agent-shell-mode-map`, which checks the system clipboard for an image and
  attaches it, falling back to normal text yank otherwise.
- **`M-x agent-shell-send-clipboard-image`** does the same explicitly; `C-u`
  on it (`agent-shell-send-clipboard-image-to`) lets you pick *which* running
  session to send it to.
- **`M-x agent-shell-send-screenshot`** captures directly via macOS's
  `screencapture` and inserts it — no clipboard step at all.

The clipboard path depends on an external tool per platform — `pngpaste` on
macOS, `wl-paste`/`xclip` on Linux — detected at runtime via
`executable-find`. Wasn't installed here (`brew install pngpaste` fixed it);
screenshot capture worked out of the box since `screencapture` ships with
macOS.

## Where things ended up

| Key | Command | Package |
|-----|---------|---------|
| `C-c 9` | `vxe/claude-code-ide` | `claude-code-ide.el` |
| `C-c 0` | `vxe/agent-shell` (new, parallel, split-aware) | `agent-shell` |
| `C-<tab>` / `C-S-<tab>` | tab-bar next/previous | stock Emacs |
| `C-c <tab>` / `C-c C-m` | `agent-shell-set-session-mode` (picker) | `agent-shell` |
| `C-c C-c` | `agent-shell-interrupt` | `agent-shell` (stock) |
| `C-y` | `agent-shell-yank-dwim` (image-aware) | `agent-shell` (stock) |
