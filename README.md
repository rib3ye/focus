# focus

A tiny CLI that blocks distracting sites at the local DNS level by managing a
delimited block inside `/etc/hosts`. No root prefix to remember, no app to run
in the background — just `focus on` to start a focus session and `focus off` to
end it.

```
focus on            # block the sites in your list
focus off           # unblock
focus add reddit.com
focus status
```

## How it works

Your blocklist is a plain file of bare domains. `focus on` injects a
marker-delimited block into `/etc/hosts` that points each domain — and its
`www.` variant — at `127.0.0.1` and `::1`, so the browser can't reach them:

```
# >>> focus blocklist >>>
127.0.0.1   youtube.com
::1         youtube.com
127.0.0.1   www.youtube.com
::1         www.youtube.com
# <<< focus blocklist <<<
```

`focus off` removes only that block, leaving the rest of `/etc/hosts` (localhost,
etc.) untouched. The DNS cache is flushed after every change so it takes effect
immediately.

Only `on`/`off` write `/etc/hosts`, and they call `sudo` themselves — you never
type `sudo focus`. Everything else runs as you.

## Commands

```
focus on [-f]        block the sites; also clears cached service workers (see
                     below). -f quits running browsers first to clear them.
focus off            unblock
focus add <url...>   add domain(s) to the blocklist (full URLs are normalized)
focus rm  <url...>   remove domain(s) from the blocklist
focus list           print the blocklist
focus status         show enabled/disabled + domain count
focus install        passwordless on/off; from a checkout also ~/.local symlink + completion
focus remove         undo install and clear any active block
focus help           print usage
```

`add`/`rm` edit the blocklist and, if focus is currently on, re-apply the block
automatically. URLs are normalized to a bare domain:
`https://www.YouTube.com/feed` → `youtube.com`. Each domain blocks both the apex
and its `www.` host.

## Install

### Homebrew

```
brew install rib3ye/tap/focus
```

This puts `focus` on your PATH and installs zsh tab-completion. Your blocklist
lives at `~/.config/focus/blocklist`, seeded from a bundled example the first
time you run a command and left untouched by upgrades. Reload your shell
(`exec zsh`) to pick up completion, then `focus <Tab>` completes commands and
`focus rm <Tab>` completes blocked domains.

Optionally make `on`/`off` stop prompting for your password:

```
focus install           # adds a scoped passwordless-sudo rule (see Privileges)
```

Under Homebrew, `install` only sets up that rule — brew already owns the PATH
entry and completion. To undo it and clear any active block before
`brew uninstall focus`, run `focus remove`.

### From source

```
git clone https://github.com/rib3ye/focus
cd focus
./bin/focus add reddit.com youtube.com   # build your list
./bin/focus on                           # start focusing
```

From a checkout your blocklist stays in-tree at `./blocklist` — **gitignored and
never published**; `blocklist.example` is the shipped starting point, copied to
`blocklist` on first run. Edit it with `focus add` / `focus rm`, or by hand.

#### Optional: install onto your PATH (`~/.local`)

So you can run `focus` from anywhere instead of typing `./bin/focus`:

```
./bin/focus install     # per-user install — no sudo for the PATH/completion part
exec zsh                # reload to pick up tab-completion
```

`install` symlinks `focus` into `~/.local/bin` and the zsh completion into
`~/.local/share/zsh/site-functions` (honoring `XDG_DATA_HOME`) — **no `sudo`
needed**. If `~/.local/bin` isn't on your `PATH`, or that completion dir isn't on
your `fpath`, it prints the exact one-line snippet to add to `~/.zshrc`. The same
command also makes `on`/`off` password-free; *that* step does use `sudo`, because
writing `/etc/hosts` needs root (see [Privileges](#privileges)).

The symlink points back at this repo, so don't move or delete the folder after
installing. Undo with `focus remove` — it removes only symlinks that resolve to
this repo (including any left by older `/usr/local` installs) and leaves the repo
and your blocklist in place.

## Service workers (the YouTube problem)

Some sites — YouTube especially — are PWAs that install a **service worker**
caching the whole app on disk and serving it from there, bypassing DNS. A hosts
block alone won't stop them. So on every `focus on`, the tool wipes the cached
service workers for any **closed** Chromium-based browser (Chrome, Brave, Edge,
Arc, Vivaldi, Chromium). A browser that's currently **running** is skipped and
reported, because the wipe only sticks once it's quit — run `focus on -f` to
gracefully quit running browsers first, then clear them.

This deletes only each profile's `Service Worker` cache (re-downloaded on next
visit); cookies, logins, and IndexedDB are left untouched. Firefox and Safari
aren't handled — clear their site data manually if needed.

## Privileges

`/etc/hosts` is owned by root, so writing it always needs root — that part is
unavoidable for any system-wide DNS block. `focus` keeps the footprint tiny:

- Only `on`/`off` touch `/etc/hosts`, and they do it through one small helper
  (`bin/focus-hosts`) that takes a single fixed argument (`on` or `off`), never
  a file path, and always maps domains to `127.0.0.1`/`::1` (IPs hardcoded).
- Out of the box, `on`/`off` prompt for your password (normal `sudo`).
- `focus install` removes that prompt: it copies the helper to a **root-owned**
  `/Library/PrivilegedHelperTools/focus-hosts` and adds a narrowly scoped rule
  to `/etc/sudoers.d/focus` that lets only that exact command run without a
  password. Root ownership of both the file *and its directory* is the security
  boundary — you can't alter, or swap out, what runs as root, and the rule
  grants nothing else. Install refuses if that directory isn't root-owned (so it
  won't, for example, drop a passwordless-root helper into a user-writable
  `/usr/local` on a Homebrew machine). `focus remove` deletes both.

Everything else (`add`, `rm`, `list`, `status`, clearing service workers) runs
entirely as you, no root involved.

## Requirements

macOS, zsh, and an account with `sudo`. No other dependencies.

## License

MIT — see [LICENSE](LICENSE).
