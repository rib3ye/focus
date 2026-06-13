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
focus install        symlink focus onto your PATH + install zsh tab-completion
focus remove         uninstall: remove the symlinks and clear any active block
focus help           print usage
```

`add`/`rm` edit the blocklist and, if focus is currently on, re-apply the block
automatically. URLs are normalized to a bare domain:
`https://www.YouTube.com/feed` → `youtube.com`. Each domain blocks both the apex
and its `www.` host.

## Setup

```
git clone <this repo> focus
cd focus
./bin/focus add reddit.com youtube.com   # build your list
./bin/focus on                           # start focusing
```

Your blocklist lives in a local `blocklist` file. It is **gitignored and never
published** — `blocklist.example` is the shipped starting point, and the script
copies it to `blocklist` automatically the first time you run a command. Edit
your list with `focus add` / `focus rm`, or by hand.

### Put `focus` on your PATH

```
./bin/focus install     # PATH symlink + completion + passwordless on/off
exec zsh                # reload the shell to pick up tab-completion
```

Then run `focus` from anywhere; `focus <Tab>` completes commands and
`focus rm <Tab>` completes domains from your blocklist. The symlink points back
at this repo, so don't move or delete the folder after installing.

To undo it:

```
focus remove            # removes the symlinks and clears any active hosts block
```

This only removes symlinks that point at this repo, and leaves the repo and your
blocklist in place.

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
