# gitcache — local mirror cache for git submodules

Local bare mirrors of every submodule, including nested ones, used by the
superprojects listed in `sources` (`~/projects/fancy-player` and
`~/projects/es-player`). A LaunchAgent keeps them fresh. New worktrees clone
their submodules from these mirrors instead of GitHub. In fancy-player, 64
submodules check out in ~12 s instead of minutes; es-player's 9 submodules,
nested 3 levels deep, take ~5 s.

## TL;DR

```sh
cd ~/projects/fancy-player          # or ~/projects/es-player
git worktree add ../fancy-player.worktrees/my-branch -b my-branch
cd ../fancy-player.worktrees/my-branch
git subinit                  # instead of: git submodule update --init --recursive
```

## Layout

```
~/gitcache/
├── README.md
├── bin/gitcache                     # the tool (bash)
├── sources                          # superprojects whose submodules are mirrored
├── logs/sync.log                    # LaunchAgent output
└── github.com/<owner>/<repo>.git    # bare mirrors, one per upstream repo
```

`~/gitcache` is a git repo that tracks only `README.md`, `AGENTS.md`
(instructions for AI coding agents), `bin/`, `sources` and `.gitignore`. The
mirrors, logs and lock are ignored.

The mirror path comes from the submodule URL, so the URL variants
`git@github.com:Org/Repo.git`, `https://github.com/Org/Repo` and
`ssh://git@github.com/Org/Repo.git` all map to
`~/gitcache/github.com/Org/Repo.git`.

## The `gitcache` command

| Command | What it does |
|---|---|
| `gitcache add <url>...` | Creates mirrors for the given URLs; skips any that already exist. |
| `gitcache add-from <repo>` | Mirrors every submodule URL in every `.gitmodules` of the repo **and all of its worktrees**, including nested `.gitmodules` inside checked-out submodules. Records the repo in `sources`, so `sync` picks up new submodules later. |
| `gitcache sync` | For each repo in `sources`: mirrors any new submodule URLs. Then runs `git remote update --prune` on every mirror (8 in parallel, 3 attempts each). |
| `gitcache list` | Lists all mirrors. |
| `gitcache path <url>` | Prints the mirror directory for a URL. |
| `gitcache submodule-update [--dissociate] [--no-recursive] [path...]` | Run inside a superproject or worktree: `git submodule init`, then clones or updates each submodule using its mirror as `--reference` (8 in parallel). Then does the same inside each submodule that has its own `.gitmodules`, unless you pass `--no-recursive`. |

Environment overrides: `GITCACHE_ROOT` (default `~/gitcache`) and
`GITCACHE_JOBS` (default `8`).

## The custom git command: `git subinit`

`git subinit` is a **git alias** (not a built-in command), defined in the
repo-local config of fancy-player and es-player (`<repo>/.git/config`).
Worktrees share their repo's config, so the alias works in every worktree of
those two repos, but nowhere else.

```ini
[alias]
    subinit = !/Users/pavel.malyshev/gitcache/bin/gitcache submodule-update
```

The leading `!` makes git run it as a shell command from the top of the
worktree. Any arguments are passed through:

```sh
git subinit                          # all submodules, recursively
git subinit third_party/fmt          # only some top-level paths (and their nested submodules)
git subinit --no-recursive           # top-level submodules only
git subinit --dissociate             # copy objects in, don't depend on the cache
```

Managing the alias:

```sh
git config --get alias.subinit       # show
git config --unset alias.subinit     # remove
# add the alias to another repo (after `gitcache add-from <that repo>`):
git -C <repo> config alias.subinit '!/Users/pavel.malyshev/gitcache/bin/gitcache submodule-update'
```

Plain `git submodule update --init` still works as before, but it does
**not** use the cache.

## How it works

1. **Mirrors.** Each mirror is a bare repo, but not a literal `git clone --mirror`:
   - It fetches only branches and tags
     (`+refs/heads/*:refs/heads/*`, `+refs/tags/*:refs/tags/*`). A literal
     `--mirror` clone would also pull GitHub's `refs/pull/*`, which is a lot of
     extra data for nothing.
   - It uses the `reftable` ref format. Some upstream repos (Catch2,
     BuddyTV/platform) have branch names that differ only in case, which the
     default `files` ref storage can't hold on macOS's case-insensitive disk.
   - It sets `gc.pruneExpire=never` and `gc.cruftPacks=true`, so objects are
     never deleted, even after an upstream force-push or branch deletion.
     Clones borrow objects from the mirror (step 3), so deleting objects
     here could break them.

2. **Sync.** The LaunchAgent `com.user.gitcache-sync` runs `gitcache sync` at
   login and then every hour. A lock in `~/gitcache/.sync.lock` stops two
   syncs from running at once. A sync normally takes about 8 s.

3. **Using the mirrors (`--reference` / alternates).**
   `gitcache submodule-update` runs, for each submodule:

   ```sh
   git submodule update --reference ~/gitcache/github.com/<owner>/<repo>.git -- <path>
   ```

   This writes the mirror's `objects` directory into the submodule's
   `objects/info/alternates`. The submodule clone then:
   - stores **no objects of its own**: it reads them from the mirror, so it is
     fast and takes almost no disk space;
   - keeps `origin` pointing at the **real GitHub URL**, so `git fetch`,
     `git pull` and `git push` inside the submodule work as usual;
   - downloads only what the mirror lacks. If the mirror is behind
     (a commit newer than the last sync), git fetches just that commit from
     GitHub.

   Git has no setting to give each submodule a different `--reference`
   repo, which is why the wrapper script and the alias exist.

   Nested submodules are handled by running the same steps inside each
   checked-out submodule that has a `.gitmodules`. Before it starts, the
   script clears the `GIT_DIR`-style variables git exports to aliases
   (`git rev-parse --local-env-vars`). Otherwise, git commands run inside a
   nested submodule would act on the superproject.

## Alternative considered: `url.<base>.insteadOf`

A global rule such as
`git config --global url."$HOME/gitcache/github.com/boostorg/".insteadOf https://github.com/boostorg/`
would make plain `git submodule update --init` use the cache with no
wrapper. It was rejected for these reasons:

- the submodule's `origin` becomes the local mirror, not GitHub;
- a submodule commit newer than the last sync fails to check out until you
  run `gitcache sync`;
- pushes need extra `pushInsteadOf` rules to still reach GitHub;
- it applies to every repo on the machine.

## LaunchAgent

File: `~/Library/LaunchAgents/com.user.gitcache-sync.plist`. It runs hourly
(`StartInterval` 3600) and at load (`RunAtLoad`), with `ProcessType` set to
`Standard`, `Nice` 10 and `LowPriorityIO`.

> Don't set `ProcessType` to `Background`: macOS then throttles the network
> and a sync took 321 s instead of 8 s.

```sh
launchctl print gui/$(id -u)/com.user.gitcache-sync | grep -E 'state|last exit|runs'
launchctl kickstart -k gui/$(id -u)/com.user.gitcache-sync   # run now
launchctl bootout   gui/$(id -u)/com.user.gitcache-sync      # stop / unload
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.gitcache-sync.plist  # load
tail -f ~/gitcache/logs/sync.log
```

The agent runs in your login session and uses the macOS ssh-agent
(`SSH_AUTH_SOCK`), so it can fetch private `git@github.com:BuddyTV/...` repos.
SSH runs with `BatchMode=yes`, so if the key isn't loaded the fetch fails
instead of waiting for a passphrase.

## Common tasks

```sh
gitcache sync                                   # refresh now (e.g. before creating a worktree)
gitcache add-from ~/projects/playback-manager   # cache another superproject's submodules
gitcache add https://github.com/foo/bar.git     # cache a single repo
du -sh ~/gitcache/github.com/*/* | sort -h      # disk usage per mirror
```

New submodules added to a superproject in `sources` are mirrored
automatically by the next sync. This includes submodules added on any branch
that's checked out in a worktree, and nested submodules once their parent is
checked out. If a submodule has no
mirror yet, `git subinit` prints a message and clones it from GitHub.

## Troubleshooting

- **`FETCH FAILED ...` in `sync.log`.** Usually a GitHub or network blip;
  each fetch is already tried 3 times. Run `gitcache sync` by hand to check.
- **`references that only differ in casing`.** The mirror uses the `files`
  ref format. Convert it with
  `git --git-dir=<mirror> refs migrate --ref-format=reftable`.
- **`cannot use bare repository ... (safe.bareRepository is 'explicit')`.**
  Address bare mirrors with `git --git-dir=<mirror> ...`, not `git -C <mirror>`.
- **Is a submodule using the cache?**
  `cat "$(git -C <submodule> rev-parse --git-dir)/objects/info/alternates"`
- **Make a worktree independent of the cache** (before deleting or moving
  `~/gitcache`): in each submodule run `git repack -a -d` and then
  `rm "$(git rev-parse --git-dir)/objects/info/alternates"`. Or create
  worktrees with `git subinit --dissociate` from the start.

## Removing everything

Make existing worktrees independent first (see above), then:

```sh
launchctl bootout gui/$(id -u)/com.user.gitcache-sync
rm ~/Library/LaunchAgents/com.user.gitcache-sync.plist
git -C ~/projects/fancy-player config --unset alias.subinit
git -C ~/projects/es-player config --unset alias.subinit
rm -rf ~/gitcache
```
