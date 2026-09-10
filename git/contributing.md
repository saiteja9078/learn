# Git & GitHub Mastery — Two-Account Collaboration Simulation

> **Goal:** By the end of this, you'll have personally created and resolved a merge conflict, opened and merged a real pull request between two GitHub accounts you control, and used `revert`, `rebase`, and `stash` in situations that actually needed them — not just as isolated commands you memorized.

We'll use two personas throughout:

| Persona | Role | Represents |
|---|---|---|
| 🧑‍💼 **MAINTAINER** | Owns the repo | Your "company" GitHub account |
| 🧑‍💻 **CONTRIBUTOR** | Sends contributions | Your "employee/OSS contributor" GitHub account |

You already know `commit`, `push`, branching, and basic merging — so we'll move fast through those and spend real time on the parts you asked for: conflicts, revert, rebase, stash, SSH auth, and PR etiquette.

---

## Table of Contents

1. [Prerequisites & Mental Model](#1-prerequisites--mental-model)
2. [Part 1 — SSH Key Auth for Two GitHub Accounts on One Machine](#2-part-1--ssh-key-auth-for-two-github-accounts-on-one-machine)
3. [Part 2 — MAINTAINER Creates the Repo](#3-part-2--maintainer-creates-the-repo)
4. [Part 3 — CONTRIBUTOR Gets Access & Clones](#4-part-3--contributor-gets-access--clones)
5. [Part 4 — Real Branching Workflow](#5-part-4--real-branching-workflow)
6. [Part 5 — Pull Requests (aka "Merge Requests")](#6-part-5--pull-requests-aka-merge-requests)
7. [Part 6 — Creating & Resolving a Real Merge Conflict](#7-part-6--creating--resolving-a-real-merge-conflict)
8. [Part 7 — `git revert`: Undoing Safely](#8-part-7--git-revert-undoing-safely)
9. [Part 8 — `git rebase`: Rewriting History Cleanly](#9-part-8--git-rebase-rewriting-history-cleanly)
10. [Part 9 — `git stash`: The Pocket Save](#10-part-9--git-stash-the-pocket-save)
11. [Part 10 — Full Sprint Simulation (Putting It All Together)](#11-part-10--full-sprint-simulation-putting-it-all-together)
12. [Real-World Workflow Patterns](#12-real-world-workflow-patterns)
13. [Readiness Checklist — Are You Ready for a Company Repo?](#13-readiness-checklist--are-you-ready-for-a-company-repo)
14. [Cheat Sheet](#14-cheat-sheet)

---

## 1. Prerequisites & Mental Model

Before diving in, lock in these mental models — they matter more than memorizing commands.

- **A commit is a snapshot**, not a diff. Each commit points to its parent(s), forming a graph (a DAG).
- **A branch is just a movable label** pointing at a commit. `HEAD` is a label pointing at "where you currently are."
- **`origin` is just a name** for a remote URL — you can have multiple remotes with different names.
- **Merge vs Rebase** both integrate changes, but they rewrite history differently (we'll see this hands-on in Part 8).
- **A Pull Request is a GitHub feature, not a Git feature.** Git has no idea what a PR is — GitHub just compares two branches and gives you a UI to discuss + merge them.

### What you need installed
```bash
git --version        # 2.34+ recommended
ssh -V
```

You'll need **two real GitHub accounts** (you said you already have these) with two different email addresses.

---

## 2. Part 1 — SSH Key Auth for Two GitHub Accounts on One Machine

This is the part most tutorials skip, and it's exactly what real engineers set up on day one when they have a personal + work GitHub account.

### Step 1: Generate two separate SSH key pairs

```bash
# Key for MAINTAINER account
ssh-keygen -t ed25519 -C "maintainer@example.com" -f ~/.ssh/id_ed25519_maintainer

# Key for CONTRIBUTOR account
ssh-keygen -t ed25519 -C "contributor@example.com" -f ~/.ssh/id_ed25519_contributor
```

Press Enter through the passphrase prompt (or set one — recommended for real work, optional here for learning speed).

This creates 4 files:
```
~/.ssh/id_ed25519_maintainer       (private key — never share)
~/.ssh/id_ed25519_maintainer.pub   (public key — goes on GitHub)
~/.ssh/id_ed25519_contributor
~/.ssh/id_ed25519_contributor.pub
```

### Step 2: Add each public key to the matching GitHub account

```bash
cat ~/.ssh/id_ed25519_maintainer.pub
```
Copy the output → GitHub (logged in as MAINTAINER) → **Settings → SSH and GPG keys → New SSH key** → paste.

Repeat with `id_ed25519_contributor.pub` on the CONTRIBUTOR account.

> ⚠️ Common mistake: pasting the `.pub` (public) key is correct. Never paste the private key anywhere.

### Step 3: Create SSH "host aliases" so Git knows which key to use

Edit `~/.ssh/config` (create it if it doesn't exist):

```
# Maintainer account
Host github-maintainer
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_maintainer
    IdentitiesOnly yes

# Contributor account
Host github-contributor
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_contributor
    IdentitiesOnly yes
```

This is the trick: `github.com` doesn't know about "accounts" at the SSH level — so you create fake hostnames (`github-maintainer`, `github-contributor`) that both point to `github.com` but each forces a different key.

### Step 4: Test both connections

```bash
ssh -T git@github-maintainer
ssh -T git@github-contributor
```

You should see:
```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```
If you see the wrong username, double-check the config file and that the right `.pub` key is on the right account.

### Step 5: Set per-repo identity (name/email)

Global config sets a default identity, but each clone should have its own local override so commits are attributed correctly:

```bash
git config --global user.name "Your Global Name"
git config --global user.email "global@example.com"

# Inside a specific repo folder:
git config user.name "Maintainer Name"
git config user.email "maintainer@example.com"
```

`git config` (no `--global`) always writes to `.git/config` of the current repo, overriding the global setting *for that repo only*. This is exactly how real devs keep work vs personal commits properly attributed.

---

## 3. Part 2 — MAINTAINER Creates the Repo

On GitHub, logged in as **MAINTAINER**:

1. Click **New repository** → name it `team-sim-project` → Public → check **Add a README** → Create.
2. Clone it using the SSH alias you built above — **note the hostname swap**:

GitHub gives you:
```
git@github.com:maintainer-username/team-sim-project.git
```
You use it as:
```bash
git clone git@github-maintainer:maintainer-username/team-sim-project.git
cd team-sim-project
```

Because you renamed the host to `github-maintainer` in your SSH config, Git silently uses the maintainer key for every push/pull/fetch on this clone — no extra flags needed, ever.

3. Add the contributor as a collaborator so they can push branches directly (simulating an internal team member, not an external OSS contributor):
   GitHub repo → **Settings → Collaborators → Add people** → search the CONTRIBUTOR username → Add.
   The CONTRIBUTOR must accept the invite (check email or GitHub notifications).

> 📌 We're using the **shared-repo model** (collaborator with push access) here since you want the "internal company team" simulation. Part 12 covers the **fork model** used for open source.

---

## 4. Part 3 — CONTRIBUTOR Gets Access & Clones

After accepting the collaborator invite, on the CONTRIBUTOR's machine (same machine, different folder is fine):

```bash
git clone git@github-contributor:maintainer-username/team-sim-project.git contributor-copy
cd contributor-copy
git config user.name "Contributor Name"
git config user.email "contributor@example.com"
```

Both of you now have independent local clones of the same repo — exactly like two employees at a company.

---

## 5. Part 4 — Real Branching Workflow

**Rule #1 of real repos: nobody commits directly to `main`.**

CONTRIBUTOR creates a feature branch:

```bash
git checkout -b feature/add-greeting
echo "console.log('Hello team');" > greeting.js
git add greeting.js
git commit -m "feat: add greeting script"
git push -u origin feature/add-greeting
```

`-u` sets the **upstream tracking branch**, so future `git push`/`git pull` on this branch don't need the remote+branch name again.

MAINTAINER, meanwhile, works on their own branch:

```bash
git checkout -b feature/add-readme-section
echo "## Setup instructions" >> README.md
git add README.md
git commit -m "docs: add setup section"
git push -u origin feature/add-readme-section
```

You now have two branches diverging from `main`, exactly like two people on a real team.

---

## 6. Part 5 — Pull Requests (aka "Merge Requests")

**Terminology note:** GitHub calls it a **Pull Request (PR)**. GitLab and Bitbucket call the identical concept a **Merge Request (MR)**. Same idea, different name.

### CONTRIBUTOR opens a PR
1. Push the branch (already done above).
2. GitHub → repo → **Pull requests → New pull request**.
3. Base: `main` ← Compare: `feature/add-greeting`.
4. Write a clear title + description (real teams often use a template: *what changed, why, how to test*).
5. Click **Create pull request**.

### MAINTAINER reviews it
- **Files changed** tab shows the diff.
- Leave inline comments by clicking the `+` next to a line.
- Approve, request changes, or just comment.
- Once satisfied: **Merge pull request**.

GitHub gives three merge strategies — know the difference, you'll be asked about this in interviews:

| Strategy | What it does | When teams use it |
|---|---|---|
| **Create a merge commit** | Keeps full branch history + adds a merge commit | When you want full audit trail of feature work |
| **Squash and merge** | Combines all commits in the PR into one clean commit on `main` | Most common in companies — keeps `main` history readable |
| **Rebase and merge** | Replays each commit individually onto `main`, no merge commit | Teams that want linear history but keep individual commits |

Merge the `feature/add-greeting` PR now using **Squash and merge**.

### Sync up locally
Both accounts now pull the updated `main`:

```bash
git checkout main
git pull
```

---

## 7. Part 6 — Creating & Resolving a Real Merge Conflict

This is the part everyone fears and almost nobody practices deliberately. Let's force one on purpose.

### Step 1: Both branches edit the *same line* of the *same file*

MAINTAINER (still has `feature/add-readme-section` branch open):
```bash
git checkout feature/add-readme-section
```
Edit `README.md` line 1 to say:
```
# Team Sim Project (maintained by MAINTAINER)
```
```bash
git add README.md
git commit -m "docs: update project title"
git push
```
Open a PR for this branch and **merge it into `main`** (squash merge).

Now CONTRIBUTOR, **without pulling the latest `main` first** (this is exactly how real conflicts happen — someone forgets to sync):
```bash
git checkout main
git checkout -b feature/rename-project
```
Edit `README.md` line 1 (their local copy is *stale*, still has the old title) to say:
```
# Team Sim Project (community edition)
```
```bash
git add README.md
git commit -m "docs: rename project title"
```

### Step 2: Trigger the conflict

```bash
git checkout main
git pull                      # pulls MAINTAINER's already-merged change
git checkout feature/rename-project
git merge main
```

You'll see:
```
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.
```

### Step 3: Look inside the conflicted file

```bash
cat README.md
```
```
<<<<<<< HEAD
# Team Sim Project (community edition)
=======
# Team Sim Project (maintained by MAINTAINER)
>>>>>>> main
```

Read this literally:
- Everything between `<<<<<<< HEAD` and `=======` is **your current branch's version**.
- Everything between `=======` and `>>>>>>> main` is **the incoming branch's version**.

### Step 4: Resolve it

Manually edit the file to what it *should* say — delete the markers entirely:
```
# Team Sim Project (community edition, maintained by MAINTAINER)
```

Then:
```bash
git add README.md
git commit           # Git pre-fills a "Merge branch 'main' into ..." message — just save it
git push
```

Open the PR — GitHub will now show it as mergeable, since the conflict is resolved *before* the PR merge, not inside GitHub's UI (though GitHub does have a web-based conflict editor for simple cases too, under the PR's "This branch has conflicts" banner).

### Bonus: aborting a conflict you don't want to deal with yet
```bash
git merge --abort
```
This safely returns you to the state right before you ran `merge` — no shame in bailing out and asking a teammate first.

---

## 8. Part 7 — `git revert`: Undoing Safely

**Key idea:** `revert` doesn't delete history — it adds a *new* commit that undoes a previous one. This is why it's the only safe undo method on shared/public branches.

### Scenario: a bad commit already reached `main`

```bash
git log --oneline -5
```
Find the bad commit's hash, e.g. `a1b2c3d`.

```bash
git revert a1b2c3d
```
Git opens an editor with a message like `Revert "feat: add greeting script"`. Save and close — a new commit is created that reverses those exact changes.

```bash
git push
```

### Revert vs Reset — the interview-favorite question

| | `git revert` | `git reset` |
|---|---|---|
| History | Adds a new commit; old commit stays | Moves the branch pointer; can delete commits from history |
| Safe on shared branches? | ✅ Yes, always | ❌ No — rewrites history others may have pulled |
| Use case | Undo something already pushed/public | Undo something still local/private |

```bash
# reset examples (use ONLY on local, unpushed commits)
git reset --soft HEAD~1     # undo commit, keep changes staged
git reset --mixed HEAD~1    # undo commit, keep changes unstaged (default)
git reset --hard HEAD~1     # undo commit, DELETE the changes — dangerous
```

> 🚫 Golden rule: **never `reset --hard` or force-push on a branch other people are also pulling from.** Use `revert` there instead.

---

## 9. Part 8 — `git rebase`: Rewriting History Cleanly

### The core difference from merge

- `git merge feature` → creates a new **merge commit** joining two histories; nothing is rewritten.
- `git rebase main` (run from `feature`) → **replays** your feature commits one by one on top of the latest `main`, as if you'd started your branch just now. History becomes linear, but commit hashes change.

### Try it

```bash
git checkout feature/rename-project
git fetch origin
git rebase origin/main
```

If there's a conflict during rebase (very possible here since we edited the same file earlier), Git pauses mid-replay:
```
CONFLICT (content): Merge conflict in README.md
```
Resolve it exactly like before:
```bash
# edit the file, remove markers
git add README.md
git rebase --continue
```
Repeat for each conflicting commit until you see `Successfully rebased and updated refs/heads/feature/rename-project.`

If it gets messy, bail out completely:
```bash
git rebase --abort
```

### The force-push you'll need after rebasing

Since rebase rewrote your commit hashes, your local branch and the remote branch now disagree. A normal `push` will be rejected. Use:

```bash
git push --force-with-lease
```

**Always use `--force-with-lease`, never plain `--force`.** `--force-with-lease` checks that nobody else pushed to that branch since you last fetched — it refuses to blindly overwrite a teammate's work, while plain `--force` will happily destroy it.

### Interactive rebase — cleaning up messy commits before opening a PR

Say you made 4 small "wip" commits locally and want one clean commit before pushing:

```bash
git rebase -i HEAD~4
```
Git opens an editor listing your last 4 commits:
```
pick e1f2a3b wip
pick b4c5d6e wip fix typo
pick f7g8h9i actually finish feature
pick j0k1l2m fix lint
```
Change `pick` to `squash` (or `s`) on the last three:
```
pick e1f2a3b wip
squash b4c5d6e wip fix typo
squash f7g8h9i actually finish feature
squash j0k1l2m fix lint
```
Save → Git prompts you to write a combined commit message → save again. Your 4 messy commits become 1 clean commit. **This is standard practice before opening a PR at most companies** — nobody wants to review 15 "wip" commits.

> ⚠️ **Never rebase (interactive or otherwise) commits that have already been pushed and that other people have pulled** — you'll rewrite shared history and cause everyone else pain. Rebase is for cleaning up *your own local, not-yet-shared* work, or syncing your feature branch with an updated `main` before it's merged.

---

## 10. Part 9 — `git stash`: The Pocket Save

**Scenario:** you're mid-edit on a feature, uncommitted, and suddenly need to switch branches (urgent bug, code review, whatever) — but your changes aren't ready to commit yet.

```bash
# You have uncommitted changes and try to switch branches:
git checkout main
# error: Your local changes to the following files would be overwritten by checkout

# Stash them instead:
git stash

# Now your working directory is clean:
git checkout main
# ...do the urgent thing...

git checkout feature/rename-project
git stash pop      # brings back your changes AND removes them from the stash
```

### Useful stash variants

```bash
git stash save "wip: mid-refactor of auth module"   # named stash, easier to find later
git stash list                                       # see all stashed sets
git stash apply stash@{1}                            # restore a specific one, KEEP it in the stash list
git stash drop stash@{1}                             # delete a specific stash entry
git stash clear                                      # delete all stashes
git stash -u                                         # also stash untracked (new) files
git stash branch new-branch-name                     # create a new branch FROM a stash — great if you stashed against the wrong branch
```

`apply` vs `pop`: `pop` = apply + immediately delete from the stash list. `apply` = restore but keep a copy in the list, useful if you want to apply the same stash to multiple branches.

---

## 11. Part 10 — Full Sprint Simulation (Putting It All Together)

Run this as a self-contained exercise now that you've learned each piece individually:

1. **MAINTAINER**: create issue "Add a CONTRIBUTING.md" on GitHub (Issues tab).
2. **CONTRIBUTOR**: `git checkout -b feature/contributing-doc`, write the file, commit referencing the issue: `git commit -m "docs: add CONTRIBUTING.md (closes #1)"`.
3. Push, open PR, **MAINTAINER** requests a change in review.
4. **CONTRIBUTOR** amends: 
   ```bash
   git add CONTRIBUTING.md
   git commit --amend --no-edit     # folds new changes into the last commit instead of adding a new one
   git push --force-with-lease
   ```
5. **MAINTAINER** approves + squash-merges. Issue auto-closes (GitHub links `closes #1` to the issue).
6. Both pull `main`.
7. **Deliberately** create one more conflict, resolve it via rebase this time instead of merge.
8. **MAINTAINER** discovers the merged CONTRIBUTING.md has an error already on `main` → `git revert` it live, push, re-fix properly in a new PR.
9. **CONTRIBUTOR** stashes mid-work on something else to quickly fix a typo on `main`, then pops the stash back.

If you can do all 9 steps without opening a tutorial, you're genuinely comfortable — this is a fair simulation of an actual sprint.

---

## 12. Real-World Workflow Patterns

You'll encounter one of these at almost any company. Know the vocabulary even if you don't master all of them yet.

### Trunk-based development
Everyone branches briefly off `main`, merges back within a day or two, feature flags hide unfinished work. Common at fast-moving product companies (favored for CI/CD).

### Git Flow
Long-lived `develop` branch, `feature/*` branches merge into `develop`, `release/*` branches stabilize before merging into `main` + `develop`, `hotfix/*` branches patch production urgently. More common in larger, release-cycle-driven orgs.

### Fork-based (open source model)
Instead of being a collaborator, contributors **fork** the repo into their own account, push to *their fork*, and open a PR from `their-fork:branch` → `original-repo:main`. Try this with your two accounts too:

```bash
# On GitHub, as CONTRIBUTOR: click "Fork" on the maintainer's repo
git clone git@github-contributor:contributor-username/team-sim-project.git
cd team-sim-project
git remote add upstream git@github-contributor:maintainer-username/team-sim-project.git
git fetch upstream
git merge upstream/main      # keep your fork synced with the original
```

### Conventional Commits
Many companies enforce a commit message format, often checked by CI:
```
feat: add login page
fix: correct off-by-one in pagination
docs: update API reference
chore: bump dependency versions
refactor: extract validation logic
```
This enables auto-generated changelogs and semantic versioning.

### Protected branches
Company repos almost always protect `main`: no direct pushes, PR + at least one approval required, CI must pass green before merge is allowed. You'll see this as a red/green checkmark next to your PR.

---

## 13. Readiness Checklist — Are You Ready for a Company Repo?

Go through this honestly. If most boxes are checked from doing Parts 1–11 above, you're ready.

**Core comfort**
- [ ] Comfortable creating branches and knowing which branch you're currently on (`git status`, `git branch`)
- [ ] Comfortable writing a clear commit message and know what makes one "good" (imperative mood, explains *why* not just *what*)
- [ ] Can read `git log --oneline --graph --all` and understand the shape of history
- [ ] Understand the difference between `git pull` and `git fetch` (`pull` = `fetch` + `merge`)

**Collaboration**
- [ ] Have opened a PR and gone through at least one review-comment → fix → re-push cycle
- [ ] Know the difference between merge commit / squash / rebase merge strategies
- [ ] Comfortable resolving a merge conflict manually, reading the `<<<<<<<` / `=======` / `>>>>>>>` markers without panic

**Safety**
- [ ] Know that `revert` is safe on shared branches and `reset --hard` + force-push is not
- [ ] Always use `--force-with-lease`, never bare `--force`
- [ ] Never rebase commits that others have already pulled

**Day-to-day fluency**
- [ ] Can stash and restore work without losing anything
- [ ] Comfortable with interactive rebase to clean up commits before a PR
- [ ] Know how `.gitignore` works and won't accidentally commit `node_modules` or `.env` files
- [ ] Comfortable with SSH-key auth and, ideally, have set up more than one identity before

**Bonus commands you'll see at a real job (worth a quick look, not covered above)**
```bash
git cherry-pick <hash>     # apply one specific commit from another branch onto yours
git bisect                 # binary-search commit history to find which commit introduced a bug
git blame <file>           # see who last changed each line, and in which commit
git tag v1.0.0             # mark a release point
git log -p -- <file>       # see the full diff history of a single file
git diff branch1..branch2  # compare two branches directly
```

If everything above is checked or at least *recognized*, you're at a solid junior-to-mid engineer comfort level with Git — the rest (company-specific CI pipelines, code owners, monorepo tooling) is learned on the job in the first week or two anywhere.

---

## 14. Cheat Sheet

```bash
# Setup
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519_name
git config user.name "Name"
git config user.email "email"

# Everyday
git status
git add <file>            # or git add .
git commit -m "type: message"
git push
git pull
git checkout -b <branch>
git checkout <branch>

# Conflict resolution
git merge <branch>              # may conflict
# edit file, remove <<<<<<< ======= >>>>>>> markers
git add <file>
git commit                      # or git rebase --continue if mid-rebase
git merge --abort                # or git rebase --abort to bail out

# Revert (safe, public history)
git revert <hash>

# Reset (local only, unpushed)
git reset --soft HEAD~1
git reset --hard HEAD~1

# Rebase
git rebase main
git rebase -i HEAD~4            # interactive cleanup
git push --force-with-lease     # after any rebase that changes pushed history

# Stash
git stash
git stash pop
git stash list
git stash apply stash@{0}
git stash drop stash@{0}

# Amend last commit
git commit --amend --no-edit
```

---

**Next step:** actually go do Part 1 through Part 11 with your two real accounts right now, in that order, without skipping the conflict/rebase parts — that's where the real learning happens. Everything else in this file is reference material to come back to when you get stuck mid-exercise.
