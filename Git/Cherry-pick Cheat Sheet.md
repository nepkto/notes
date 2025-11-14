# Git Cherry-pick Cheat Sheet

A compact cheat sheet for using `git cherry-pick` and related commands. Use this when you want to apply commits from one branch onto another, control how commit messages are handled, or recover from problems during a cherry-pick.

What I did: I created this README that explains each command, shows examples, and includes the exact list you provided in the summary. What's next: use this file in your repo (copy/paste or save as README.md) and tell me if you want it expanded with examples for conflict resolution, rebase alternatives, or a printable one-page card.

---

## Quick Reference (the list from your summary)

- Single: git cherry-pick <commit>
- Multiple: git cherry-pick <h1> <h2>
- Range: git cherry-pick A^..B
- No commit: git cherry-pick -n <commit>
- Annotate: git cherry-pick -x <commit>
- Edit message: git cherry-pick -e <commit>
- Continue: git cherry-pick --continue
- Abort: git cherry-pick --abort
- Skip: git cherry-pick --skip
- Revert if pushed: git revert <commit-hash>

---

## Basic Usage

- Cherry-pick a single commit:
  - git cherry-pick <commit>
  - Example: git cherry-pick f1a2b3c

- Cherry-pick multiple explicit commits:
  - git cherry-pick <h1> <h2>
  - Example: git cherry-pick a1b2c3d e4f5g6h

- Cherry-pick a range of commits:
  - git cherry-pick A^..B
  - Example: git cherry-pick a1b2c3^..d4e5f6
  - Note: `A^..B` means commits after A (exclusive) up to and including B.

## Useful Options

- Apply without committing (no commit):
  - git cherry-pick -n <commit>
  - Use when you want to combine several cherry-picked changes into a single commit.

- Annotate message with original commit hash:
  - git cherry-pick -x <commit>
  - Appends “(cherry picked from commit <hash>)” to the message for traceability.

- Edit the commit message before committing:
  - git cherry-pick -e <commit>
  - Opens your editor to modify the message.

## Conflict and Flow Control

- After resolving conflicts and staging changes, finish the cherry-pick:
  - git cherry-pick --continue

- Abort the cherry-pick and restore the state before it:
  - git cherry-pick --abort

- Skip the problematic commit and continue with the next:
  - git cherry-pick --skip

## Reverting Changes That Were Already Pushed

- If a commit has been pushed and you must undo it without rewriting history:
  - git revert <commit-hash>
  - This creates a new commit that reverses the changes introduced by the specified commit.

## Tips & Best Practices

- Keep your working tree clean before starting a cherry-pick.
- Prefer `-x` when cherry-picking across long-lived branches so you can trace origins.
- Use `-n` to accumulate multiple cherry-picked changes and commit them together if that fits your workflow.
- If you need to move many commits, consider creating a branch and merging or using `git rebase --onto` as an alternative.
- When resolving conflicts:
  - Use git status to see remaining files with conflicts.
  - Use git add <file> to mark each resolved file.
  - Run git cherry-pick --continue to finish.

## Example Workflow

1. Switch to the target branch:
   - git checkout target-branch

2. Cherry-pick commits:
   - Single commit: git cherry-pick f1a2b3c
   - Multiple commits: git cherry-pick a1b2c3d e4f5g6h
   - Range: git cherry-pick a1b2c3^..d4e5f6

3. If conflicts occur:
   - Resolve conflicts in the affected files.
   - git add <resolved-file>
   - git cherry-pick --continue

4. If you decide to stop:
   - git cherry-pick --abort

5. If the changes are already pushed and you need to undo:
   - git revert <commit-hash>

---

If you'd like, I can:
- Add concrete conflict-resolution examples (with diff snippets).
- Produce a one-page printable cheat sheet.
- Convert this into a GitHub-flavored README with badges and links.

Tell me which of those you'd like next and I'll update the file. 