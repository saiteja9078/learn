# Git & GitHub Learning Guide

These notes are built around our Git learning conversation: two GitHub accounts, SSH access, Git's local state, branches, HEAD, stash, reset/revert, rebase, and merge conflicts. They also teach the remaining topics: pull vs fetch, interactive rebase, amend, force-with-lease, cherry-pick, bisect, and blame.

The goal is to understand what Git is storing, where a change currently lives, and which commands are safe for shared work.

---

## 1. Git and GitHub are different things

- **Git** is version-control software on your computer. It records commits, manages branches, and works without a network connection.
- **GitHub** hosts Git repositories online. It lets people collaborate and controls access to repositories.
- A **repository** is a project’s files plus its commit history.
- A **commit** is a saved snapshot with a message and a link to its parent commit.

~~~text
Your computer                          GitHub
--------------                         ------
working files -> Git commits  <---->   remote repository
                 push / fetch
~~~

Git does not automatically put every file change on GitHub. You must create a commit, then push it.

---

## 2. Using two GitHub accounts: keep the layers separate

The initial setup had one GitHub account open in a browser and another account apparently being used by Terminal. The important correction is that these are separate layers:

| Layer | What it controls | Example |
|---|---|---|
| Browser sign-in | The GitHub account visible on the website | Safari has a personal account; Brave has a work account |
| Commit identity | Name and email written into new commits | Your Name <work@example.com> |
| Git authentication | Which GitHub identity Git uses for fetch/push | An SSH key accepted by the work account |
| Repository permission | Whether that account can read/write a repo | A collaborator or team role |

Changing browser accounts does not change which GitHub account Terminal uses. Setting Git user.name and user.email changes commit attribution; it does not grant GitHub access.

### One SSH key per account

A clean approach is one SSH key for each account, then SSH host aliases that choose the intended key. Replace the names and email addresses with yours.

~~~bash
ssh-keygen -t ed25519 -C "personal-email@example.com" -f ~/.ssh/id_ed25519_personal
ssh-keygen -t ed25519 -C "work-email@example.com" -f ~/.ssh/id_ed25519_work
~~~

Add the content of each public .pub file to the matching GitHub account in Settings → SSH and GPG keys. Then add aliases to ~/.ssh/config:

~~~sshconfig
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  IdentitiesOnly yes

Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
~~~

Use the alias in the remote URL:

~~~bash
git clone git@github-work:WORK-OWNER/REPOSITORY.git

# Or change an existing repository:
git remote set-url origin git@github-work:WORK-OWNER/REPOSITORY.git
~~~

Verify the account each key reaches:

~~~bash
ssh -T git@github-personal
ssh -T git@github-work
~~~

Set the author identity inside each repository where needed:

~~~bash
git config user.name "Your Work Name"
git config user.email "work-email@example.com"
git config --get-regexp '^user\.'
~~~

### Read/write access for an SSH key

For a normal personal SSH key, the key only proves your GitHub identity. The GitHub account’s role on a repository grants its read/write access. You do not make a personal key “writable” by changing the key.

A repository **deploy key** is different: GitHub may offer an Allow write access setting for it. It is intended for automation and should be tightly scoped. Never share a private SSH key. If it is exposed, remove it from GitHub and create a replacement.

---

## 3. The three places a change can live

This is the central Git mental model:

~~~text
HEAD / last commit          staging area (index)          working directory
saved snapshot       ->     next commit draft       ->    files you edit now
~~~

- **Working directory**: the real files you are editing.
- **Staging area / index**: the exact version selected for the next commit.
- **HEAD**: normally the latest commit on the current branch; it is Git’s reference point for your checked-out snapshot.

Normal workflow:

~~~bash
git status
git add path/to/file
git commit -m "Explain the change"
~~~

Git add does not create a permanent commit. It selects the file’s current contents for the next commit. If you edit that file again, the staging area and working directory can hold different versions.

Useful inspection commands:

~~~bash
git status                 # summary of staged and unstaged changes
git diff                   # unstaged changes
git diff --staged          # changes selected for the next commit
git log --oneline -10      # recent commits
~~~

---

## 4. What “working tree clean” means

Your understanding was correct in the normal case. When Git says:

~~~text
nothing to commit, working tree clean
~~~

there are no uncommitted changes that Git sees.

For a fully clean repository:

~~~text
HEAD              C1
staging area      C1
working directory C1
~~~

So, in the simple model:

> Working tree clean means the working directory is effectively the same as HEAD.

The correction is that **clean does not mean synchronized with GitHub**. Your files can be clean while your branch is behind, ahead of, or diverged from the remote:

~~~text
Local HEAD       C1
origin/main      C2
Working tree     C1

The working tree is clean, but local main is behind origin/main.
~~~

Check both local and remote context with:

~~~bash
git status
git branch -vv
git log --oneline --decorate --graph --all
~~~

---

## 5. Branches and HEAD

A branch is a movable label pointing to a commit. HEAD normally points to the current branch.

~~~text
main -> C1 <- HEAD
~~~

After a new commit on main, main and HEAD move to that commit.

Create and switch branches:

~~~bash
git switch -c feature/login  # create a branch and switch to it
git switch main              # switch back
git branch                   # list branches; * marks the current one
~~~

Branches are cheap. Create one before independent work or experiments.

---

## 6. Remotes, origin, and origin/main

origin is usually just a nickname for the remote GitHub repository; it is conventional, not magic.

~~~bash
git remote -v
~~~

origin/main is your **local record** of the last-known main branch on origin. It is a remote-tracking branch. It updates after a fetch.

~~~text
main          your local branch
origin/main   your locally stored view of GitHub's main
~~~

---

## 7. Fetch vs pull in practice

### Fetch: download information without changing project files

~~~bash
git fetch origin
~~~

Fetch contacts GitHub and updates remote-tracking names such as origin/main. It does not merge, rebase, switch branches, or edit your working directory.

A safe inspect-first workflow:

~~~bash
git fetch origin
git log --oneline HEAD..origin/main       # commits GitHub has that you do not
git log --oneline origin/main..HEAD       # local commits GitHub does not
git diff HEAD..origin/main                # incoming file changes
~~~

### Pull: fetch, then integrate

By default, pull means fetch followed by merge, unless the repository is configured to rebase on pull.

~~~bash
git pull origin main
~~~

Pull can change the current branch and working directory. It can create a merge commit or stop for a merge conflict.

A cautious daily pattern:

~~~bash
git status
git fetch origin
git log --oneline HEAD..@{u}  # @{u} means the configured upstream branch
git pull --ff-only
~~~

The --ff-only form integrates only when Git can move the branch pointer straight forward. It avoids an unexpected merge commit.

If your team intentionally rebases unpublished local work:

~~~bash
git pull --rebase
~~~

Do not use rebase casually on shared history or while you do not understand your uncommitted changes.

---

## 8. Merge and merge conflicts

A merge combines histories. Git can usually merge automatically if the changes do not overlap.

~~~text
        C2  (Safari branch)
       /
C1 --- 
       \
        C3  (Brave branch)
~~~

If both branches modify the same line or nearby lines differently, Git cannot decide the final content. It pauses and marks the file:

~~~text
<<<<<<< HEAD
current-branch version
=======
incoming-branch version
>>>>>>> other-branch
~~~

Resolve a conflict carefully:

1. Read the surrounding code or text and decide the intended final result.
2. Edit the file, remove every conflict-marker line, and retain the correct content.
3. Run git status to see each unresolved file.
4. Stage the resolution: git add path/to/file.
5. Finish the operation:
   - after a merge: git commit
   - after a rebase: git rebase --continue
   - after a cherry-pick: git cherry-pick --continue

To abandon the current integration attempt:

~~~bash
git merge --abort
git rebase --abort
git cherry-pick --abort
~~~

Use the command that matches the operation in progress.

---

## 9. Rebase: replay work on a new base

Rebase takes commits unique to your branch and recreates them on a new starting point.

Before:

~~~text
main:    A---B---C
feature:      \
               D---E
~~~

After running git rebase main on feature:

~~~text
main:    A---B---C
feature:          \
                   D'---E'
~~~

D' and E' contain similar changes to D and E but are new commits with new IDs. Therefore, rebase rewrites history.

Good use: update and clean up your own unpublished feature branch before review.

Avoid rebasing a branch other people have built work on unless everyone has coordinated.

~~~bash
git switch feature/login
git fetch origin
git rebase origin/main
~~~

If a conflict happens, resolve it, stage the resolution, and continue:

~~~bash
git add path/to/resolved-file
git rebase --continue
~~~

Use git rebase --abort to return to the state from before the rebase.

---

## 10. Stash: temporarily park uncommitted work

Use stash when you need a clean working directory but are not ready to make a commit.

~~~bash
git stash push -m "WIP: investigate parser bug"
~~~

Afterward, normally:

~~~text
HEAD              C1
staging area      C1
working directory C1
stash             previous uncommitted changes
~~~

Useful commands:

~~~bash
git stash list
git stash show -p stash@{0}
git stash apply stash@{0}  # restore and keep the stash entry
git stash pop             # restore, then remove it if successful
~~~

By default, untracked files are not stashed. Include them with -u when appropriate:

~~~bash
git stash push -u -m "WIP including untracked files"
~~~

Stash is temporary parking, not a substitute for meaningful commits or backup.

---

## 11. Restore, reset, and revert: different kinds of undo

### Discard an unstaged file change

~~~bash
git restore path/to/file
~~~

This replaces the working-copy version with the staged/HEAD version. Inspect git diff first because uncommitted work can be lost.

### Unstage a file but keep its edits

~~~bash
git restore --staged path/to/file
~~~

The change moves from the staging area back to ordinary working-directory changes.

### Reset: move local history

Reset moves the current branch pointer and can also change the staging area and working directory.

| Command | Branch / HEAD | Staging area | Working directory | Common use |
|---|---|---|---|---|
| git reset --soft HEAD~1 | moves back | keeps changes staged | keeps changes | redo the last commit |
| git reset HEAD~1 | moves back | unstages changes | keeps changes | turn last commit into edits |
| git reset --hard HEAD~1 | moves back | resets | resets | discard last local commit and work |

The --hard form can destroy uncommitted work. Do not use it as a first response to confusion.

### Revert: undo a shared commit safely

~~~bash
git revert COMMIT_ID
~~~

Revert makes a new commit that reverses an older commit. It preserves published history, making it the usual safe choice on shared branches such as main.

Rule of thumb: use reset to reshape your own local/unpublished history; use revert to undo an already-shared commit.

---

## 12. Interactive rebase

Interactive rebase lets you edit a sequence of your commits: reorder, rename, combine, edit, or drop them. It is very useful before sharing a feature branch.

~~~bash
git log --oneline
git rebase -i HEAD~3
~~~

Git opens a todo list similar to:

~~~text
pick a1b2c3d Add resume parser
pick d4e5f6a Fix typo
pick 123abcd Add tests
~~~

Common actions:

| Action | Meaning |
|---|---|
| pick | keep the commit |
| reword | keep it but change the message |
| edit | pause after applying it so you can amend it |
| squash | combine it with the previous commit and edit the result message |
| fixup | combine it with the previous commit and discard this message |
| drop | remove the commit |

For example, if “Fix typo” only corrects the previous commit, change its action to fixup. The branch ends with one cleaner commit.

Interactive rebase creates new commit IDs. Use it on your own, unshared branch. If you already pushed that personal branch, the update afterward may need:

~~~bash
git push --force-with-lease
~~~

---

## 13. git commit --amend

Amend replaces the most recent commit.

Change only its message:

~~~bash
git commit --amend -m "Better commit message"
~~~

Add a forgotten file to the previous commit without changing its message:

~~~bash
git add forgotten-file.py
git commit --amend --no-edit
~~~

The prior commit gets a new ID. This is excellent immediately after making your own unpushed commit. If it was pushed, coordinate before rewriting that remote branch.

---

## 14. git push --force-with-lease

After a rebase or amend, local history has new commit IDs. A normal push is rejected because it is no longer a fast-forward update.

~~~bash
git push --force-with-lease origin feature/login
~~~

Force-with-lease checks that the remote branch still points where your local repository last saw it. If somebody pushed to it in the meantime, it refuses rather than overwriting their work.

It is safer than bare --force, but it still replaces remote history. Before using it:

1. Confirm that the branch is your feature branch, never main by accident.
2. Fetch and inspect the current remote state.
3. Make sure teammates are not using the history you are replacing.
4. Prefer --force-with-lease; do not normalize bare --force.

Protected main branches usually forbid force-pushing. That is a healthy default.

---

## 15. git cherry-pick

Cherry-pick copies one specific commit’s change to the branch you are currently on, creating a new commit.

~~~bash
git switch release/1.2
git cherry-pick a1b2c3d
~~~

Use it when a focused bug fix needs to go to a release branch without merging an entire unrelated feature branch.

If there is a conflict:

~~~bash
# Resolve the file, then:
git add path/to/resolved-file
git cherry-pick --continue

# Or abandon the operation:
git cherry-pick --abort
~~~

Cherry-pick duplicates changes rather than connecting branch histories. Note why it was used, and use normal merge instead when the goal is to bring over a broader line of work.

---

## 16. git bisect

Bisect finds the commit that introduced a bug using a binary search. You identify one known-good point and one known-bad point; Git checks out middle commits until it finds the first bad one.

~~~bash
git bisect start
git bisect bad                 # current commit is broken
git bisect good v1.4.0         # this tag or commit worked
~~~

At each checked-out commit, test the behavior and report the result:

~~~bash
git bisect good
# or
git bisect bad
~~~

When Git names the first bad commit:

~~~bash
git show COMMIT_ID
~~~

Always exit bisect mode when finished:

~~~bash
git bisect reset
~~~

For a reliable automated test, Git can run the command each step:

~~~bash
git bisect run python -m pytest tests/test_parser.py
~~~

Use automated bisect only if the test is valid across every commit in the range.

---

## 17. git blame

Blame shows the most recent commit responsible for each line in a current file.

~~~bash
git blame path/to/file.py
git blame -L 25,45 path/to/file.py
~~~

Use it to discover context: when a line appeared, what the commit message was, and which related change explains it. Then inspect the commit:

~~~bash
git show COMMIT_ID
~~~

Blame is historical attribution, not a reason to blame a person. Refactors can obscure original authorship; git log -p -- path/to/file.py is often useful alongside it.

---

## 18. Everyday safe workflow

Before an operation that may integrate, rewrite, or discard work:

~~~bash
git status
git branch --show-current
git log --oneline --decorate -10
git remote -v
~~~

Ask these questions:

1. Which branch am I on?
2. Do I have local edits that should be committed or stashed?
3. Is this personal local work or a shared branch?
4. Am I fetching information, integrating history, or rewriting history?
5. Can I inspect a diff or make a backup branch first?

A quick, low-risk checkpoint before experimenting:

~~~bash
git branch backup/before-experiment
~~~

---

## 19. Suggested learning order from here

You have already worked through the core idea of a repository’s state, branches, HEAD, reset/revert, rebase, stash, and clean working trees. Continue in this order:

1. Create a deliberate merge conflict in a throwaway practice repo and resolve it.
2. Fetch first, inspect incoming commits, then compare pull --ff-only and pull --rebase on a personal branch.
3. Amend one unpushed commit, then use interactive rebase to squash a short sequence.
4. Learn exactly when a rewritten personal branch needs push --force-with-lease.
5. Cherry-pick one small bug-fix commit to a practice release branch.
6. Use bisect to find a deliberately introduced bug.
7. Use blame and show to trace why a line exists.

The guiding rule is simple: inspect state first, make the smallest change that achieves your goal, and do not rewrite shared history without coordination.

