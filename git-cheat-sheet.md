# Git & Terminal Cheat Sheet

---

## The one rule everything hangs on

**A repo is a folder with a `.git` inside it.**

Git commands search *upward* from where you're standing until they find a `.git`:

```
  /Users/fahad/                     ← if a .git is HERE, everything below
    └── projects/                      is treated as one giant repo (BAD)
        └── santander-cycles-forecast/
            └── .git                ← this is the repo root (GOOD)
```

If `git status` lists your whole home directory, there's a stray `.git` at `~`. Fix: `rm -rf ~/.git`

---

## The three boxes

```
  WORKING DIR          STAGING AREA          REPOSITORY
  (files on disk)      (the "index")         (saved history)
       |                     |                     |
  edit a file                |                     |
       |                     |                     |
       └──── git add ───────►│                     │
                             └──── git commit ────►│
                                                   │
                                            commit 7a57329
                                            commit f894a51
                                            commit e2ff175
```

- Saving a file = box 1 only
- `git add` = box 1 → box 2 ("queued")
- `git commit` = box 2 → box 3 ("saved forever")
- **Staging is not saving.** Only commit puts it in history.
- Git tracks **files, not folders**. An empty folder is invisible.

---

## The everyday loop

| Command | Does |
|---|---|
| `git init` | Make this folder a repo |
| `git status` | Show the state of the three boxes |
| `git add <file>` | Stage one file |
| `git add .` | Stage everything here (safe — obeys `.gitignore`) |
| `git commit -m "msg"` | Save staged changes |
| `git log --oneline` | See history, one line each |

Commit messages: imperative — finish the sentence *"This commit will…"*
`Add pandas dependency` ✅  `updated stuff` ❌

---

## Reading `git status`

| It says | It means |
|---|---|
| `Untracked files:` | Git has never seen this file |
| `Changes not staged` | Tracked file edited, not added yet |
| `Changes to be committed` | Staged, ready to commit |
| `nothing to commit, working tree clean` | All saved ✅ |

---

## Commits

```
  47df740 ──► a24e759 ──► 6357fa3 ──► 7a57329
  (root)                               ▲
  no parent                            │
                                    HEAD -> main
```

- Each commit points back to its **parent**
- The very first has none — Git labels it `(root-commit)`
- The hash is a fingerprint of the contents. Change anything → new hash
- `git commit --amend` doesn't edit — it **replaces** with a new hash
- **Rewrite freely before pushing. Think hard after.**

**Pointers:**
- `HEAD` = where I am now
- `main` = a movable label on a commit
- `origin/main` = my local note of where the remote was, last time I checked

---

## Branching

A branch is **just a label on a commit**. Not a copy.

```
                      ┌──► c74e066  (add-pr-notes)
                      │
  7a57329 ──► f894a51 ┤
                      │
                      └──► b84fc17  (main) ◄── HEAD
```

| Command | Does |
|---|---|
| `git branch` | List branches (`*` = current) |
| `git switch -c <name>` | Create **and** switch |
| `git switch <name>` | Switch only |
| `git merge <name>` | Fold that branch **into the one I'm on** |
| `git branch -d <name>` | Delete label (refuses if unmerged) |
| `git branch -D <name>` | **Force** delete — discard unmerged work |

⚠️ Switching **rewrites your files on disk** to match that branch's commit. Committed work is safe — the files just reflect wherever the label points.

⚠️ Merge folds into the branch you're **on**, not the one you meant. Sync `main` by standing on `main`.

---

## Fast-forward vs merge commit

**Fast-forward** — receiving branch has no work of its own. The label just slides:

```
  before:   A ──► B           after:   A ──► B ──► C
            ▲     ▲                          ▲     ▲
           main  feature                   (nothing new made)
                                           main  feature
```

**Merge commit** — both branches moved. A new commit with **two parents**:

```
          ┌──► C ──┐
  A ──► B ┤        ├──► M   ◄── merge commit (2 parents)
          └──► D ──┘
```

---

## Merge conflicts

Both branches changed **the same lines**. Git won't guess.

```
  <<<<<<< HEAD
  my version (the branch I'm on)
  =======
  their version (the incoming branch)
  >>>>>>> other-branch
```

**Five steps:**

```
  1. git merge x      → CONFLICT
  2. git status       → "both modified: notes.md"
  3. edit the file    → delete ALL markers, leave what you want
  4. git add notes.md → this is how you say "resolved"
  5. git commit       → no -m, accept the prefilled message
```

- `git add` does double duty: stages **and** marks resolved
- Editing alone isn't enough — step 4 is required
- Stuck? `git merge --abort` returns you to before

---

## Remotes

```
  MY MAC                          GITHUB
  ┌──────────────┐                ┌──────────────┐
  │ main         │ ── push ─────► │ main         │
  │ origin/main  │ ◄── fetch ──── │              │
  └──────────────┘                └──────────────┘
         ▲
    my local note of
    where GitHub was
```

| Command | Does |
|---|---|
| `git remote add origin <url>` | Save the URL. **Sends nothing** |
| `git push -u origin main` | Send commits **and** pair the branches (`-u`, once) |
| `git push` | After `-u`, no arguments needed |
| `git fetch` | Download + move `origin/main`. **Doesn't touch my files** |
| `git merge origin/main` | Apply the fetched commits to my branch |
| `git pull` | = fetch + merge |
| `git clone <url>` | Copy a whole repo to a new machine |

**Two separate steps people confuse:**
- `remote add` = **wiring** (writes a URL into config)
- `-u` on first push = **pairing** (links main ↔ origin/main)

**Setup order:** create an **EMPTY** repo on GitHub (no README / licence / gitignore ticked), then `remote add`, then `push -u`.

Why empty matters:

```
  MY REPO:      517eb39 (root) ──► ...
  THEIR REPO:   9fa1c22 (root) ──► ...   ← README created this

  Two roots, no shared ancestor
  → "refusing to merge unrelated histories"
```

---

## Pull requests

A PR is **a GitHub feature, not a git command**. A proposal to merge, with review before it lands.

```
  1. git switch -c my-feature     ─┐
  2. edit, add, commit             │ local — same as always
  3. git push -u origin my-feature ─┘

  4. Open PR on GitHub            ─┐
  5. Check direction               │ on the website
  6. Merge pull request           ─┘

  7. git switch main              ─┐
  8. git fetch                     │ sync back down
  9. git merge origin/main        ─┘

  10. git branch -d my-feature              ─┐
  11. git push origin --delete my-feature    │ tidy up
  12. git remote prune origin               ─┘
```

**The direction box — people get this backwards:**

```
   base: main   ◄──── compare: my-feature
   ▲                   ▲
   destination         source
   (gets changed)      (changes come from)
```

Read it as a sentence: *into base, from compare.*

**Why bother:** review before `main` changes · permanent record of why · the hook CI tests hang on · `main` stays protected.

**On the PR page:** green `+` = added lines, red `-` = removed. "No conflicts with base branch" = GitHub ran the conflict check for you, in advance.

**Prune is a diagnostic:** if `git remote prune origin` doesn't remove `origin/x`, that branch **still exists on GitHub**. Delete it there first.

---

## Shell commands

| Command | Does | Notes |
|---|---|---|
| `pwd` | Where am I | First move when unsure |
| `ls` | List files | `-a` all (incl. dotfiles), `-l` long, `-la` both |
| `cd <dir>` | Change directory | `cd ..` up, `cd` alone → home |
| `mkdir <dir>` | Make a folder | `-p` makes parents too |
| `touch <file>` | Make an empty file | |
| `rm <file>` | Delete | `-r` into folders, `-f` no prompt. **No Trash** |
| `mv a b` | Move **or** rename | |
| `cat <file>` | Print a file | |
| `tail -3 <file>` | Last 3 lines | |
| `code .` | Open this folder in VS Code | `code <file>` for one file |
| `nano <file>` | Edit in terminal | Save `^O`+Enter, exit `^X` |

## Shell symbols

```
  A && B    run B only if A SUCCEEDED
  A ;  B    run B regardless
  A || B    run B only if A FAILED
  A |  B    feed A's OUTPUT into B    ← different idea: data, not success
```

| Symbol | Does |
|---|---|
| `>` | Write to file — **overwrites** |
| `>>` | Write to file — **appends** |
| `2>/dev/null` | Throw away error messages |
| `~` `.` `..` | Home · here · one up |

Example: `mkdir x && touch x/f` — won't try the file if the folder failed.

---

## The pager — when the terminal "freezes" on a `:`

Long output opens in `less`. It's **waiting, not frozen**.

```
  q         quit back to prompt
  space/b   page down / up
  j/k       line down / up
  /text     search
```

`--oneline` is usually short enough to skip it. Or pipe it: `git log | cat`

---

## Gotchas

| Error | Cause & fix |
|---|---|
| `zsh: command not found: add` | Bad **first** word. It's `git add`, not `add` |
| `zsh: no matches found: [~/x]$` | Pasted the prompt prefix. Only type what comes after `%` |
| `zsh: command not found: code` | VS Code → `Cmd+Shift+P` → "Install 'code' command in PATH" → **new terminal** |
| `git status` shows all of `~` | Stray `.git` at home → `rm -rf ~/.git` |
| `refusing to merge unrelated histories` | Remote was created with a README. Create remotes empty |
| `401 Unauthorized` | "Don't know who you are" — bad/expired token |
| `403 Forbidden` | "Know you, not allowed" — missing scope |
| `...without 'workflow' scope` | Pushing `.github/workflows/*.yml` needs the **`workflow`** scope too |
| Push fails after months away | **Token expired.** New token (tick `repo` + `workflow`) **and** delete the `github.com` entry in Keychain Access |

**Token notes:** GitHub's "Sign in with Google" works in the browser only. Use a Personal Access Token as the CLI password. Nothing appears while pasting it — that's deliberate. Long-term fix: `brew install --cask git-credential-manager`.

---

## Habits

1. `git status` constantly
2. `pwd` before any `init` command
3. `working tree clean` = everything saved
4. Amend only before pushing
5. Never merge a red cross
