# AGENTS.md — instructions for AI agents working in `~/gitcache`

This repo is a small personal tool. It keeps local bare mirrors of the git
submodules used by `~/projects/fancy-player`, so new worktrees can clone their
submodules locally instead of from GitHub. `README.md` is the user-facing
documentation; read it first.

## What is tracked

Only `.gitignore`, `AGENTS.md`, `README.md`, `bin/` and `sources` are
tracked. Everything else is runtime data and is ignored:

| Path | What it is | Rules |
|---|---|---|
| `bin/gitcache` | The whole tool (bash) | The main thing you edit. |
| `sources` | Absolute paths of superprojects whose submodules are mirrored, one per line | Edit through `gitcache add-from`, not by hand. |
| `github.com/<owner>/<repo>.git` | Bare mirrors (~900 MB) | **Never delete, move, or `gc --prune=now` these.** Worktree submodules borrow objects from them through git alternates. Deleting objects corrupts those worktrees. |
| `logs/sync.log` | LaunchAgent output | Ignored. |
| `.sync.lock/` | Lock directory, present while a sync runs | Remove it only if the PID inside is dead. |

When you add a new tracked file at the top level, add a matching
`!/<name>` line to `.gitignore`. Its first rule (`/*`) ignores everything else.

## Related files outside this repo

- `~/Library/LaunchAgents/com.user.gitcache-sync.plist` runs `gitcache sync`
  hourly and at load, with `ProcessType` set to `Standard`. **Don't change it
  to `Background`:** macOS then throttles the network and a sync goes from
  ~8 s to ~5 min.
- `~/projects/fancy-player/.git/config` holds the alias
  `alias.subinit = !/Users/pavel.malyshev/gitcache/bin/gitcache submodule-update`.
  All fancy-player worktrees share this config.

If you change the CLI, the paths or the scheduling, update `README.md`, and
update these external files where they're affected.

## Design decisions (keep them unless the user asks otherwise)

1. **Mirror layout:** `$GITCACHE_ROOT/<host>/<owner>/<repo>.git`, computed by
   `mirror_path()`. SSH and HTTPS URLs, with or without `.git`, map to the same
   path. Changing this function moves every mirror and breaks the
   `objects/info/alternates` files of existing worktrees.
2. **Branches and tags only:** fetch refspecs are `+refs/heads/*:refs/heads/*`
   and `+refs/tags/*:refs/tags/*`, not `--mirror`'s `refs/*`. This avoids
   GitHub's `refs/pull/*`.
3. **`reftable` ref format** (`git init --ref-format=reftable`). Some upstream
   repos (Catch2, BuddyTV/platform) have branch names that differ only in
   case, which the default `files` format can't store on macOS's
   case-insensitive filesystem.
4. **Objects are never deleted:** mirrors set `gc.pruneExpire=never` and
   `gc.cruftPacks=true`.
5. **`--reference`, not `insteadOf`:** `submodule-update` runs
   `git submodule update --reference <mirror> -- <path>` for each submodule.
   Each submodule's `origin` stays the real GitHub URL, and git fetches only
   what the mirror lacks. `git submodule update` accepts `--reference` and
   `--dissociate`, but **not** `--reference-if-able`.
6. **Serial init, parallel update:** `git submodule init` writes the shared
   `.git/config`, so it runs once, serially. The per-path updates then run
   in parallel through `xargs -P`.
7. **Never block:** `GIT_TERMINAL_PROMPT=0`, SSH with `BatchMode=yes` and
   timeouts, git's HTTP low-speed limits, and 3 fetch attempts with backoff.
   The LaunchAgent has no TTY, so a prompt would hang forever.

## Coding rules for `bin/gitcache`

- **Target `/bin/bash` 3.2** (the macOS system bash). No associative arrays,
  `mapfile` or `readarray`, `${var,,}`, `|&`, or `wait -n`.
- **macOS userland:** BSD `sed` (`sed -i ''`), BSD `find` and `xargs`; there
  is no `timeout`, `flock` or `realpath -m`. The LaunchAgent sets its own
  `PATH`, so don't assume Homebrew tools are on it.
- **Address bare repos with `git --git-dir=<mirror>`**, never with
  `git -C <mirror>`. With `safe.bareRepository=explicit`, git refuses
  `git -C <mirror>`.
- `git submodule`'s `--quiet` goes **before** the subcommand
  (`git submodule --quiet update ...`).
- Keep `set -u`-safe: initialize variables, and use `${1:-}` for optional
  arguments.
- `gitcache help` prints the `#` comment header at the top of the file
  (`sed -n '2,15s/^# \{0,1\}//p'`). Keep the header in sync with the
  commands, and if it grows past line 15, widen that range.
- Write output in one line per event, prefixed with a timestamp (`log()`),
  so `logs/sync.log` stays easy to scan.
- `shellcheck bin/gitcache` must be clean. Suppress a check with a
  `# shellcheck disable=SCxxxx` comment placed before the command it covers,
  with the reason on the line above.

## Validating changes

Run whichever of these apply to your change, smallest first:

```sh
cd ~/gitcache
bash -n bin/gitcache && shellcheck bin/gitcache
./bin/gitcache help
# URL mapping (expect identical paths for the first two):
./bin/gitcache path git@github.com:fmtlib/fmt.git
./bin/gitcache path https://github.com/fmtlib/fmt
./bin/gitcache list | wc -l            # number of mirrors (72 as of creation)
./bin/gitcache sync                    # expect "sync done: N mirrors, ~10s, ok"
```

End-to-end test, for changes to `submodule-update` or `mirror_path`. It uses a
throwaway worktree; always clean it up:

```sh
cd ~/projects/fancy-player
git worktree add -q --detach ../fancy-player.worktrees/_gitcache-test HEAD
cd ../fancy-player.worktrees/_gitcache-test
time git subinit                                   # ~12 s for 64 submodules
git submodule status | grep -v '^ ' || echo "all submodules clean"
cat "$(git -C third_party/fmt rev-parse --git-dir)/objects/info/alternates"
git -C third_party/fmt remote get-url origin       # must be the GitHub URL
cd ~/projects/fancy-player
git worktree remove --force --force ../fancy-player.worktrees/_gitcache-test
git worktree prune
```

To test the LaunchAgent:

```sh
launchctl kickstart -k gui/$(id -u)/com.user.gitcache-sync
sleep 20; tail -n 3 ~/gitcache/logs/sync.log
launchctl print gui/$(id -u)/com.user.gitcache-sync | grep -E 'state|last exit'
```

## Safety

- Don't touch the user's real fancy-player worktrees. Only the throwaway
  `_gitcache-test` worktree may be created and removed.
- Don't run `git gc --prune=now`, `git repack -a -d` with pruning, or
  `rm -rf` on mirrors, and don't migrate them in a way that drops objects.
  Two operations are safe: `git --git-dir=<mirror> refs migrate --ref-format=reftable`
  and adding new mirrors.
- Don't add `url.*.insteadOf` rules to global git config unless the user asks.
- The mirrors include private BuddyTV repos. This repo's `origin` is the
  **public** GitHub repo `pavel-m-vizio/gitcache`. Only the tracked files go
  there: never commit mirrors, logs, credentials, or private source code, and
  keep `.gitignore`'s `/*` rule.

## Commits

Make small, focused commits with imperative subject lines. End every
commit message with this trailer:

```
Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```
