# Git & Terminal Cheat Sheet

My working notes from learning Git from scratch. Ordered by mental model first, then commands, then the gotchas I actually hit.

---

## The mental model (everything hangs on this)

**A repo is a folder with a `.git` in it.** The `.git` folder *is* the repository — delete it and you're left with plain files, no history.

**Files live in one of three places, and move through them in order:**

```
  Working directory  ──git add──▶  Staging area  ──git commit──▶  Repository
  (files on disk)                  (the "index",                 (permanent
                                    a drafting area)              snapshots/history)
```

- Committing is **two steps**: stage (`git add`), then commit (`git commit`). This gives me precise control over what goes into each snapshot.
- **Staging is not saving.** A staged file is queued, not yet in history.
- **Git tracks files, not folders.** An empty folder is invisible to Git — it only ever sees the files inside (e.g. `src/` shows up only once there's an `__init__.py` in it).

**Commits are immutable, content-addressed snapshots.**
- A commit's **hash** (e.g. `a24e759…`) is a SHA-1 fingerprint of its *contents*: files + message + author + timestamp + parent commit.
- Change *anything* → different hash. This is why history is tamper-evident.
- You never edit a commit; you replace it with a new one.
- The **very first commit has no parent** — Git labels it `(root-commit)`. Only one per repo; it's the base of the whole tree.

**Pointers:**
- `HEAD` = "where I am right now".
- `main` = the default branch (a movable pointer along the commit chain).
- `HEAD -> main` = HEAD is on the main branch, main is on this commit.
- `origin/main` = my local record of "where main was on the remote, last time I checked".

---

## The everyday loop

| Command | What it does | Box it touches |
|---|---|---|
| `git init` | Create a repo (makes `.git`) | — |
| `git status` | Show state of the three boxes | reads all |
| `git add <file>` | Stage a change | working → staging |
| `git add .` | Stage **everything** under here (`.` = current dir & below) — respects `.gitignore`, never grabs `.git` | working → staging |
| `git commit -m "msg"` | Save staged changes as a snapshot | staging → repo |
| `git log` | Show the commit history | reads repo |

**Commit messages:** write in the imperative — "Add…", "Fix…", "Update…" — completing *"This commit will…"*. Describe what the snapshot **does**, so `git log` is useful months later.

---

## Reading `git status` (my most-used command)

| Message / section | Means |
|---|---|
| `No commits yet` | Empty repo, no history |
| `Untracked files:` | File exists on disk but Git isn't managing it |
| `Changes not staged for commit` + `modified:` | Tracked file edited, but change not staged yet |
| `Changes to be committed` + `new file:` / `modified:` | Staged, ready for next commit |
| `nothing to commit, working tree clean` | Disk matches repo — all saved ✅ |

- `new file:` = Git has never seen it. `modified:` = tracked file whose contents changed.
- Editing a tracked file does **not** auto-stage it — `git add` every time.
- **`git status` listing your whole home directory (with `../../` paths)** = there's an accidental `.git` at `~`. Tools search *upward* for their root marker; a stray one poisons everything beneath it. (See Gotchas.)

---

## Inspecting things

| Command | Shows |
|---|---|
| `git log` | Commit chain (newest on top), hashes, pointers, author, message |
| `git log --oneline` | Compact one-line-per-commit view (short enough to skip the pager) |
| `git log --oneline --all` | Every branch's pointer at once — reveals splits between `main` and `origin/main` |
| `git log --oneline --graph` | Draws branch structure as ASCII — forks and merges as lines |
| `git ls-files -s` | What's in the staging area — mode, blob hash, stage, filename |
| `git hash-object <file>` | The blob hash of a file's contents |
| `git remote -v` | Remote bookmarks and their URLs (empty = none set) |

- File mode `100644` = regular file; `100755` = executable. Git only records the executable bit, not full Unix permissions.
- A **blob** is how Git stores file contents, named by the hash of that content — same content-addressing idea as commits, one level down.

---

## The pager (`less`) — when the terminal "freezes" on a `:`

Long output (`git log`, `git diff`) opens in a scrollable pager. A `:` or coloured bar at the bottom means it's **waiting for a navigation command, not frozen**. Keystrokes are navigation, not text.

| Key | Does |
|---|---|
| `q` | Quit back to the prompt |
| Space / `b` | Page down / page up |
| `j` / `k` or arrows | Line down / line up |
| `/text` | Search forward for "text" |

`--oneline` output is usually short enough to skip the pager entirely. Piping to `cat` (`git log | cat`) also dodges it.

---

## Amending & rewriting history

```
git commit --amend -m "new message"
```
- **Replaces** the last commit with a new one (new hash) — it doesn't edit in place.
- Safe **before** a commit is shared/pushed. Risky **after**, because your history then diverges from the copy others have.
- Rule of thumb: **rewrite freely before you push; think hard after.**

---

## Remotes & GitHub

A **remote** = another copy of the repo on a server. `origin` = the conventional name for the main one.

| Command | What it does |
|---|---|
| `git remote add origin <url>` | Save the URL under the nickname `origin` — **local bookkeeping only, sends nothing, pairs nothing** |
| `git push -u origin main` | Send local `main` to the remote; `-u` links `main` ↔ `origin/main` (the pairing) |
| `git push` / `git pull` | After `-u`, no arguments needed |
| `git fetch` | Download remote commits into `origin/main` **without** touching my files |
| `git pull` | fetch **+** merge into working files |
| `git clone <url>` | Copy an entire remote repo to a new machine (sets up `origin` automatically) |

**Two distinct steps, often confused:**
- `git remote add origin <url>` = **wiring** (writes a bookmark into config). Nothing is transmitted; no branch is paired.
- The `-u` on the first `git push` = **pairing** (links `main` to `origin/main`, once). The first byte of the project travels on this push.

**The push/pull logic:**
- Local `main` ahead of `origin/main` → I have commits the server doesn't → **push**.
- `origin/main` ahead of local `main` → server has commits I don't → **pull**.
- All three pointers on the same commit = perfectly in sync.

**Setup order that avoids pain:** create an **empty** repo on GitHub (no README/licence/gitignore) → `git remote add origin` → `git push -u origin main`.
- **Why empty matters:** ticking "Add a README" makes GitHub create its *own* first commit — a rival `root-commit`. My local repo already has *its* own root-commit. Two independent trees with **no shared ancestor** → push rejected, and `git pull` errors with `refusing to merge unrelated histories`. An empty remote has no rival root-commit, so my history slots straight on.

---

## Config

```
git config --get user.name       # check
git config --get user.email
git config --global user.name  "Your Name"    # set for every repo on this machine
git config --global user.email "you@email"
```
Identity is baked into every commit's hash, so it's worth getting right.

---

## Branching & merging

A branch is just a **named, movable pointer to a commit** — not a copy of files or history. One label on one commit, which slides forward as I commit on that branch.

| Command | What it does |
|---|---|
| `git branch` | List branches (`*` marks the one I'm on) |
| `git branch <name>` | Create a branch — makes a pointer, does **not** switch to it |
| `git switch <name>` | Move `HEAD` onto a branch (older equivalent: `git checkout`) |
| `git switch -c <name>` | Create a branch **and** switch to it in one step |
| `git merge <name>` | Fold the named branch **into** the branch I'm currently on |
| `git branch -d <name>` | Safe delete — removes only the label, and only if already merged |
| `git push -u origin <branch>` | First push of a *new* branch — sets its upstream. Bare `git push` fails until this is done once |
| `git merge --abort` | Bail out of an in-progress merge, back to the pre-merge state |

**Key ideas:**
- Creating a branch ≠ switching to it — two separate actions.
- Switching branches **rewrites my working files** to match that branch's commit — files physically change on disk. Work already committed is safe; the files just reflect wherever the branch points.
- To merge: stand on the **receiving** branch first, then merge the other one in. (Want `experiment`'s work on `main` → switch to `main`, then `git merge experiment`.) **Merging folds into the branch I'm *on* — not the one I *meant*.** Sync `main` by standing on `main`, not on a feature branch.
- Deleting a merged branch removes only the pointer; every commit is safe on the branch it was merged into.
- A merge done locally doesn't touch the remote — `origin/main` stays behind until I `git push`.

**Fast-forward vs merge commit:**
- **Fast-forward** — when the receiving branch has no work of its own since the split, Git just slides its pointer forward. No new commit, straight-line history.
- **Merge commit** — when both branches have diverged (each has commits the other lacks), Git creates a commit with **two parents**, one reaching back to each branch. The join point where two lines of history knit together. A real merge of divergent work always makes one.

**Editing files properly:** `echo "..." >> file` was a teaching device for single lines. For real multi-line edits use a **text editor** — `code <file>` (VS Code) or `nano <file>`. Saving writes to box 1 (working directory) — then it's the usual `add` → `commit`.

**Everyday loop:** `git switch -c feature` → edit → `git add` → `git commit` → `git switch main` → `git merge feature` → `git push` → `git branch -d feature`.

---

## Merge conflicts

**What causes one:** two branches change the **same lines of the same file**, starting from the same point. Git can't fast-forward and **won't guess** which version wins — so it pauses the merge and hands the decision to me. A conflict is Git refusing to silently discard work, not Git breaking.

**The five-step workflow:**

1. **Merge** triggers it → `CONFLICT (content): Merge conflict in <file>` / `Automatic merge failed`.
2. **Read** it → `git status` shows `Unmerged paths` / `both modified: <file>`. The repo is now paused mid-merge.
3. **Resolve** it → open the file, edit it into the final version I want **by hand**, delete the marker lines.
4. **Mark resolved** → `git add <file>`. This is how I *tell Git* the conflict is settled — editing the file alone isn't enough.
5. **Complete the merge** → `git commit` (leave off `-m`; Git pre-fills a merge message — just save & close the editor). Creates the two-parent merge commit.

**Conflict markers** — Git rewrites the file to show both versions:
```
<<<<<<< HEAD
my current branch's version
=======
the incoming branch's version
>>>>>>> other-branch
```
- `<<<<<<< HEAD` to `=======` → **my** side (the branch I'm on).
- `=======` → the divider (not content).
- `=======` to `>>>>>>> other-branch` → the **incoming** side.
- Resolving = deleting the markers and leaving exactly the content I want. Final file has **no markers**. Free to keep either side or write a blend.

**Key ideas:**
- `git add` does double duty in a conflict: it stages the file *and* declares the conflict resolved.
- While mid-merge, Git blocks other actions (`cannot switch branch while merging`) — it holds me in place until I finish or `git merge --abort`.
- Fast-forward = straight line, no merge commit. A conflict resolution = a diamond in `--graph`: split, then rejoin at the merge commit.
- `-m` needs a value right after it — a bare `-m` errors with `switch 'm' requires a value`.

---

## Pull requests (PRs) — the team workflow

**What a PR is:** a *proposal* to merge one branch into another, opened **on GitHub**, that others can review and discuss before the merge happens. It is **not a git command** — there's no `git pull-request`. It's a GitHub feature wrapped around the ordinary branch-and-merge I already know. The Git half (branch, commit, push) is identical; the PR is what happens *next*, on the website.

**Why bother (vs merging locally):** review catches problems before they hit `main`; the PR page is a permanent record of *why* a change was made; it's the hook automated tests hang on (Phase 2 — tests run on every PR); `main` stays protected — nothing lands except through a deliberate, reviewed merge.

**The full round trip:**

1. **Branch + work** — `git switch -c <branch>`, edit, `add`, `commit`. (Same as always.)
2. **Push the branch** — `git push -u origin <branch>`. The key break from local merging: a PR needs the branch to **exist on GitHub** to open a request about. After pushing a *new* branch, GitHub prints a `Create a pull request... visiting: <url>` line it never shows for existing branches.
3. **Open the PR** — that URL, or the "Compare & pull request" banner on the repo page.
4. **Check the direction** — `base: main ← compare: <branch>`. Changes flow **from compare into base**. `base` = destination (gets modified), `compare` = source. Reversed = nothing to merge, or the wrong merge. Read it as a sentence every time; never memorise.
5. **Review the diff** — green `+` = added lines, red `-` = removed. This *is* the content of the request. "Able to merge / No conflicts with base branch" = GitHub ran the same conflict-check I did by hand, pre-emptively, on the server.
6. **Create + Merge** — title + description, "Create pull request", then "Merge pull request" → "Confirm merge". (Solo = I approve my own; on a team this may be locked until someone reviews / tests pass.)

**Syncing local main afterwards** — the merge happened on GitHub's servers; **local `main` knows nothing** (Git is distributed — nothing syncs automatically; the `-u` pairing just records *where* to sync, it doesn't auto-sync):

| Command | What it does |
|---|---|
| `git fetch` | Download new commits from `origin` and move the **remote-tracking pointer** (`origin/main`) — but **leave local `main` untouched**. |
| `git log --oneline --all` | Show every branch's pointer at once — reveals `main` and `origin/main` split after fetch. |
| `git merge origin/main` | Merge the fetched commits into local `main`. **Must be standing on `main`** — merge folds into the current branch. Purely behind + no local work = **fast-forward**. |

(`git pull` = `git fetch` + `git merge` in one step. Doing them separately lets me *see* the two halves.)

**A trap I hit:** running `git merge origin/main` while still on a feature branch fast-forwards the *wrong* pointer (the feature branch), leaving local `main` stranded. The commands succeed but update the branch I'm *on*, not the one I meant. Fix: `git switch main` first, *then* merge.

**Tidying up a merged branch:**

| Command | What it does |
|---|---|
| `git branch -d <branch>` | Safe-delete the **local** branch label. Refuses if unmerged; fine once merged. Prints `was <hash>` as a receipt. |
| `git push origin --delete <branch>` | Delete the branch **on the remote** (GitHub) — or use the "Delete branch" button on the merged PR. |
| `git remote prune origin` | Remove local **remote-tracking pointers** (`origin/<branch>`) for branches that **no longer exist on the remote**. |

**Prune is a diagnostic, not just cleanup:** if `git remote prune origin` *doesn't* remove `origin/<branch>`, that branch **still exists on GitHub**. Delete it on the remote first, *then* prune has something to sweep.

---

## Shell & terminal (zsh on macOS)

The **prompt** is context, never typed. bash shows `$`, zsh (default on modern macOS) shows `%`.

**Commands (the verbs — they *do* something):**

| Command | What it does | Notes |
|---|---|---|
| `pwd` | Print working directory — "where am I" | First move whenever unsure |
| `ls` | List directory contents | `-l` long (perms/owner/size), `-a` all incl. dotfiles, `-d` the folder itself not its contents. Combine: `ls -la` |
| `cd <dir>` | Change directory | `cd ..` up one, `cd ~` or `cd` alone → home |
| `mkdir <dir>` | Make a directory | `-p` also creates missing parents and doesn't error if it exists |
| `touch <file>` | Create an empty file (or bump its modified-time if it exists) | The empty-file trick is what it's mostly used for |
| `rm <file>` | Remove (delete) | `-r` recursive (into folders), `-f` force (no prompt). `rm -rf` deletes instantly, no Trash — powerful, unforgiving |
| `cat <file>` | Print a file's contents | Short for "concatenate" |
| `echo "text"` | Print text | Alone → screen; with `>`/`>>` → into a file |
| `code <file>` / `code .` | Open a file / the whole current folder in VS Code | Enable once via VS Code palette → "Shell Command: Install 'code' command in PATH", then reopen the terminal. `code .` (`.` = here) is the everyday one |
| `nano <file>` | Edit a file in-terminal | Save `^O`+Enter, exit `^X`. Always available when `code` isn't |

**Operators & redirection (the glue — they connect or redirect):**

| Symbol | Meaning | Mental model |
|---|---|---|
| `&&` | "and then, **only if that succeeded**" | Every command returns a hidden pass/fail. `A && B` runs `B` only if `A` passed — stops me acting on a broken state |
| `;` | "and then, **regardless**" | `A ; B` runs `B` whatever `A` did — chaining independent steps |
| `\|\|` | "**or else**, only if that failed" | Mirror of `&&` |
| `\|` | **Pipe** — feed one command's *output* into the next's *input* | About *data flowing*, not success/failure. `git log \| cat` pipes the log into cat to dodge the pager |
| `>` | Redirect output into a file — **overwrite** | `echo "x" > f` replaces f's contents |
| `>>` | Redirect into a file — **append** | Adds a line; big difference from `>` |
| `2>/dev/null` | Throw away *error* messages | Streams: `1` = normal, `2` = errors; `/dev/null` = the void. "Send errors nowhere" |
| `~` / `.` / `..` | Home dir / here / parent dir | `~` = `/Users/<me>`; `.` = current folder; `..` = one up |

**Notes:**
- Hidden files start with `.` (`.git`, `.gitignore`, `.zshrc`) — invisible to plain `ls`, need `ls -a`.
- No output usually means success with nothing to report, not an error.
- A stray `%` right after `cat` output = the file has no trailing newline. Harmless; convention is to end text files with one.
- `command not found` almost always = a bad *first* word (a typo, or a tool not on `PATH` yet — e.g. `code` before it's installed).

---

## Gotchas I actually hit

- **`zsh: no matches found: [~/gitrepo]$`** — I pasted the guide's prompt prefix. In zsh, `[...]` is a filename pattern; matching nothing aborts the command. Fix: only type what comes *after* the prompt.
- **`zsh: command not found: add`** — typo in the *first* word. Git commands are two words: `git add`, not `add`. `command not found` almost always = bad first word.
- **`zsh: command not found: code`** — the VS Code launcher isn't on `PATH` yet. Fix: VS Code → `Cmd+Shift+P` → "Shell Command: Install 'code' command in PATH", then **open a fresh terminal** (PATH is read at shell start).
- **`git status` shows my entire home directory** (paths like `../../.DS_Store`) — an accidental `.git` at `~`. Tools search *upward* for their root marker and latch onto strays. Fix: `rm -rf ~/.git` (see the Python-setup sheet for the full clean-up).
- **`refusing to merge unrelated histories`** — pushed a local repo (with its own root-commit) at a GitHub repo that was created *with* a README (its own root-commit). Two trees, no shared ancestor. Fix: create remotes empty.
- **GitHub "Sign in with Google" only works in the browser** — the command line can't use it, and GitHub doesn't accept account passwords for Git operations at all.
- **Use a Personal Access Token (PAT)** as the password on the CLI. Generate under Settings → Developer settings → Tokens (classic). **Must tick the `repo` scope** to push. Copy it immediately — shown once.
- **Terminal shows nothing while pasting a token** — no dots or stars. That's deliberate, not frozen.
- **`403 Forbidden` ≠ `401 Unauthorized`.** 401 = "I don't know who you are" (bad credentials). 403 = "I know who you are, you're not allowed" (permission/scope). A 403 on push usually means the token lacks `repo` scope.
- **Stale credentials in Keychain** — macOS caches the first token. After making a new one, delete the `github.com` entry in Keychain Access or Git keeps reusing the old broken one.
- **Git Credential Manager** (`brew install --cask git-credential-manager`) stores credentials in Keychain and allows a browser (Google) login — authenticate once, no repeated prompts.

---

## Habits worth keeping

1. Run `git status` constantly — it's how Git talks back about the three boxes.
2. `pwd` before `git init` or `uv init` — never initialise in the wrong (or home) directory.
3. Meaningful, imperative commit messages.
4. `working tree clean` = green light that everything's saved.
5. Read the *absence* of output as information (empty `git remote -v` = no remotes).
6. Amend/rewrite only before pushing.