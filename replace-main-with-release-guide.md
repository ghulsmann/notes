# Post-Release Guide: Replacing `main` with `release`

A complete, step-by-step procedure for pointing the `main` branch at the exact same commit as the `release` branch after a successful go-live — without merging, without conflict resolution, and without risk of corrupting `main`.

---dddd445

## Table of Contents

1. [Overview & Goal](#overview--goal)
2. [Why This Approach Is Safer Than Merging](#why-this-approach-is-safer-than-merging)
3. [Prerequisites](#prerequisites)
4. [Pre-Flight Checks (Do Not Skip)](#pre-flight-checks-do-not-skip)
5. [Communication Plan](#communication-plan)
6. [Execution: Step-by-Step](#execution-step-by-step)
7. [Post-Operation Verification](#post-operation-verification)
8. [Team Follow-Up Instructions](#team-follow-up-instructions)
9. [Rollback Plan](#rollback-plan)
10. [Troubleshooting](#troubleshooting)
11. [Appendix: Branch Protection Reference](#appendix-branch-protection-reference)

---

## Overview & Goal

**Goal:** After a successful release, make the `main` branch *become* the `release` branch — i.e. point `main` at the exact same commit `release` is on. No merge commit. No three-way merge. No auto-resolution.

**The technique:** A force-update of `main` to match `release`. This is a *pointer change*, not a merge. After the operation, `main` and `release` will share the same commit SHA and have identical working trees.

**Why this works for a post-release scenario:** You've just released from `release`, so you know that commit is the production-correct code. By definition, that's what `main` should reflect. There is nothing to "merge" — you simply want `main` to point at the known-good commit.

---

## Why This Approach Is Safer Than Merging

| Concern | Merging `release` → `main` | Force-update `main` to `release` |
| --- | --- | --- |
| Merge conflicts | Possible; auto-resolution may pick wrong side silently | None — no merge occurs |
| Resulting tree | May differ from `release` if conflicts resolved poorly | Bit-for-bit identical to `release` |
| History shape | Adds a merge commit | Clean — `main` simply points at the release commit |
| Risk of "corrupting" main | Real | Effectively zero (with the pre-flight check below) |
| Reproducibility | Depends on merge resolution | Deterministic |

The only real risk in the force-update approach is **losing commits that exist on `main` but not on `release`**. The pre-flight check below catches this before you do anything destructive.

---

## Prerequisites

Before you start, confirm all of the following:

- [ ] You have **admin** rights on the GitHub repository (needed to modify branch protection / rulesets).
- [ ] You have **push access** to `main` (or can grant yourself a temporary bypass).
- [ ] You have a local clone of the repository with both `main` and `release` available as remote-tracking branches.
- [ ] Git is installed and you are reasonably comfortable with `git reset --hard` and `git push --force-with-lease`.
- [ ] You know the current state of branch protection rules on `main` (you will be temporarily modifying and then restoring them).
- [ ] The release has already been deployed and verified in production. **Do not run this procedure on an unverified release.**

---

## Pre-Flight Checks (Do Not Skip)

These checks confirm the operation is safe. Run them before touching anything.

### Check 1: Fetch the latest state of the remote

```bash
git fetch origin --prune
```

### Check 2: Are there any commits on `main` that are NOT on `release`?

This is the single most important check.

```bash
git log origin/release..origin/main --oneline
```

- **Empty output** → safe to proceed. Every commit on `main` is already on `release`.
- **Non-empty output** → **STOP**. There are commits on `main` (likely hotfixes pushed directly, or merges from another branch) that would be discarded by the force-update. Investigate each one:
  - If they are already incorporated into `release` via some other path (e.g. cherry-picked), the comparison should be empty — re-check.
  - If they are genuinely missing from `release`, you must decide whether to cherry-pick them onto `release` first, or to abort this approach entirely.

### Check 3: Note the current `main` SHA (for rollback)

```bash
git rev-parse origin/main
```

**Copy this SHA somewhere safe.** This is your rollback anchor. Save it in a sticky note, a chat message to yourself, anywhere you won't lose it for the next hour.

### Check 4: Note the target `release` SHA

```bash
git rev-parse origin/release
```

Both SHAs should be different at this point (that's the whole reason you're doing this). Note this one too.

### Check 5: List open PRs targeting `main`

In the GitHub web UI:

```
https://github.com/<org>/<repo>/pulls?q=is%3Aopen+is%3Apr+base%3Amain
```

Or via the GitHub CLI:

```bash
gh pr list --base main
```

Note these down. After the force-update, these PRs may show unusual diffs and may need to be rebased onto the new `main`. You don't need to do anything to them *now*, but their authors will need a heads-up.

### Check 6: List tags pointing at the current `main`

```bash
git tag --points-at origin/main
```

This is just informational — tags are not affected by branch pointer changes — but it's good to know what was tagged at the old `main` tip in case anyone asks.

---

## Communication Plan

Force-updating a protected branch is an event everyone with the repo cloned needs to know about. Send a message in your team's channel **before** you start and **after** you finish.

### Before the operation

> **Heads-up:** I'm going to force-update `main` to match `release` at approximately `<time>` as part of post-release cleanup. This is a pointer change — no merge — so the new `main` will be bit-for-bit identical to the release we just deployed.
>
> **After it lands, you will need to re-sync your local `main`** (instructions to follow). If you have a branch based on `main`, you may need to rebase it. Open PRs against `main` may show odd diffs and may need rebasing too.
>
> Branch protection on `main` will be briefly relaxed and restored immediately after.

### After the operation

> `main` has been force-updated to match `release`. New `main` SHA is `<sha>`.
>
> **To re-sync your local `main`:**
> ```
> git fetch origin
> git checkout main
> git reset --hard origin/main
> ```
> Branch protection is back in place. If you had a PR open against `main` and the diff looks weird now, rebase your branch onto the new `main`.

---

## Execution: Step-by-Step

### Step 1: Capture the current branch protection configuration

You're going to relax protection briefly. You need to be able to put it back exactly. Choose one approach:

**Option A — Screenshots (simplest):** Go to **Settings → Branches** (for classic branch protection) or **Settings → Rules → Rulesets** (for the newer ruleset system). Open the rule(s) covering `main` and screenshot every page. Capture every checkbox, every dropdown, every entry in lists like "Restrict who can push" or "Bypass list".

**Option B — Export via API (more rigorous):** Using the GitHub CLI or a tool like `gh api`:

```bash
# For classic branch protection:
gh api repos/<org>/<repo>/branches/main/protection > main-protection-backup.json

# For rulesets, list and export them:
gh api repos/<org>/<repo>/rulesets > rulesets-list.json
# Then for each ruleset ID:
gh api repos/<org>/<repo>/rulesets/<id> > ruleset-<id>-backup.json
```

Keep these files until you've verified protection is fully restored at the end.

### Step 2: Relax protection on `main`

You need to allow yourself to force-push to `main` for the next few minutes. There are several ways to achieve this, in rough order of preference:

1. **Add yourself to a bypass list** on the existing ruleset (GitHub Team/Enterprise plans). This is the cleanest — protection stays "on" for everyone else, and you don't have to remember to flip switches back.
2. **Toggle "Allow force pushes"** on the branch protection rule, restricted to specific people (you). Re-disable after.
3. **Temporarily disable the entire rule.** Use this only if the above options aren't available. It's the riskiest because you must remember to re-enable everything correctly.

You will also need:

- Force pushes allowed (for you).
- Any "Require a pull request before merging" rule bypassed or relaxed.
- Any required status checks bypassed (since you are not opening a PR).
- Any "Restrict who can push" rule must include you.

> **Practical tip:** GitHub's UI lets a repo admin tick "Do not allow bypassing the above settings" — make sure that is **unchecked** for the duration of this operation if you intend to bypass, otherwise admins can't bypass either.

### Step 3: Sync your local repository

```bash
git fetch origin --prune
```

### Step 4: Switch to `main` locally

```bash
git checkout main
```

If you don't have a local `main` branch yet:

```bash
git checkout -b main origin/main
```

### Step 5: Reset local `main` to the `release` commit

```bash
git reset --hard origin/release
```

After this, your local `main` is identical to `origin/release`. Verify:

```bash
git rev-parse HEAD
git rev-parse origin/release
```

These two SHAs must match. If they don't, stop and investigate.

### Step 6: Force-push `main` to the remote

```bash
git push --force-with-lease origin main
```

**Why `--force-with-lease` and not `--force`:** `--force-with-lease` refuses the push if someone else has updated `main` on the remote since your last fetch. This prevents you from accidentally clobbering a commit you didn't know about. If it fails, fetch again and re-run the pre-flight checks — something changed on the remote.

If the push is rejected because of branch protection, return to Step 2 — protection wasn't sufficiently relaxed.

### Step 7: Restore branch protection immediately

Do this **before** doing anything else. Don't go for coffee first.

- If you used a bypass list, remove yourself.
- If you toggled "Allow force pushes", toggle it back off.
- If you disabled the rule entirely, re-enable it and compare every setting to your screenshots / JSON backup.
- Re-tick "Do not allow bypassing the above settings" if it was ticked before.

### Step 8: Verify protection is back

Attempt (and expect to fail) a dry-run force push to confirm protection is restored:

```bash
git push --force-with-lease --dry-run origin main
```

This should be rejected by branch protection. If it *succeeds*, protection is not fully restored — go back to Step 7.

---

## Post-Operation Verification

Run all of these. All should pass.

### Verification 1: `main` and `release` point to the same commit

```bash
git fetch origin
git rev-parse origin/main
git rev-parse origin/release
```

The two SHAs must be identical.

### Verification 2: No commits differ between them

```bash
git log origin/main..origin/release --oneline
git log origin/release..origin/main --oneline
```

Both must produce empty output.

### Verification 3: Trees are identical

```bash
git diff origin/main origin/release
```

Must produce no output.

### Verification 4: GitHub UI shows the expected state

- Navigate to the repository's main page on GitHub.
- The default branch (`main`) should now show the same latest commit message as `release`.
- Open the branches list — `main` and `release` should both show "0 ahead, 0 behind" of each other.

### Verification 5: Branch protection is restored

- Re-open the branch protection rule / ruleset for `main`.
- Compare every setting to your pre-operation backup.
- Confirm you can no longer push directly to `main` from your normal account.

---

## Team Follow-Up Instructions

Send this to the team after the operation:

> ### Action required: re-sync your local `main`
>
> `main` was force-updated to match `release` after the release. Your local `main` is now out of sync with the remote. Run:
>
> ```bash
> git fetch origin
> git checkout main
> git reset --hard origin/main
> ```
>
> **If you have a feature branch based on the old `main`:** rebase it onto the new `main`:
>
> ```bash
> git checkout your-feature-branch
> git rebase origin/main
> ```
>
> **If you have a pull request open against `main`:** the diff may look unusual. Rebase the branch onto the new `main` (as above) and force-push your feature branch with `git push --force-with-lease`.
>
> **If `git status` on a branch you care about shows commits you don't recognise**, stop and ask before resetting — you may have local work that needs preserving.

---

## Rollback Plan

If something goes wrong and you need to put `main` back the way it was, you saved the old SHA in **Check 3** of the pre-flight section.

### Step 1: Relax protection on `main` again

Same procedure as **Execution Step 2**.

### Step 2: Reset and force-push to the old SHA

Replace `<OLD_MAIN_SHA>` with the SHA you captured.

```bash
git fetch origin
git checkout main
git reset --hard <OLD_MAIN_SHA>
git push --force-with-lease origin main
```

### Step 3: Restore protection

Same as **Execution Step 7**.

### Step 4: Notify the team

> Heads-up — I reverted the `main` force-update. `main` is now back at its previous commit. If you already ran the re-sync instructions, run them again to pick up the rollback:
> ```
> git fetch origin
> git checkout main
> git reset --hard origin/main
> ```

> **Important:** A rollback is only fully safe if nobody has based new work on the new `main` yet. If teammates have already pulled and built on top of it, coordinate with them before rolling back.

---

## Troubleshooting

### "Updates were rejected because the tip of your current branch is behind"

You ran `git push` without `--force-with-lease` (or `--force`). Use `--force-with-lease`. This is expected because you are intentionally rewriting `main`.

### "Updates were rejected because the remote contains work that you do not have locally"

`--force-with-lease` is doing its job — someone updated `main` on the remote since you last fetched. Stop, run `git fetch origin --prune`, redo the pre-flight checks, and decide whether you still want to proceed.

### "Protected branch update failed" / "required status checks are expected"

Branch protection on `main` isn't sufficiently relaxed. Go back to **Execution Step 2** and check:

- Is force pushing allowed for you?
- Are required PRs / required reviews bypassed for you?
- Are required status checks bypassed for you?
- Are you in any "restrict who can push" list?

If the repo uses the new ruleset system, check **all rulesets** — there may be more than one applying to `main`.

### "You don't have permission to bypass these rules"

Even repo admins can be locked out if "Do not allow bypassing the above settings" is enabled. An admin needs to untick that option first (or you need to disable the rule entirely for the duration of the operation).

### A teammate ran `git pull` on `main` after the force-update and got conflicts / weird state

Their local `main` had diverged. Have them run:

```bash
git fetch origin
git checkout main
git reset --hard origin/main
```

**Warning them first:** `git reset --hard` discards uncommitted local changes. If they have uncommitted work on `main`, they should stash it (`git stash`) before resetting.

### Open PR against `main` now shows hundreds of unexpected changed files

That's expected if the PR branch was based on old `main`. The author should rebase their branch onto the new `main`:

```bash
git fetch origin
git checkout their-branch
git rebase origin/main
# resolve any conflicts
git push --force-with-lease origin their-branch
```

### CI is now failing on `main` even though it passed on `release`

The branches are identical now, so CI should pass on `main` if it passed on `release`. If it doesn't, the failure is likely flaky CI, an environment-specific config (e.g. a workflow that only runs on `main`), or branch-name-coupled secrets/variables. Investigate the workflow logs.

---

## Appendix: Branch Protection Reference

GitHub has two systems for protecting branches; you may be using one or both.

### Classic branch protection rules

Found under **Settings → Branches → Branch protection rules**. Common settings that interfere with a force-update:

| Setting | Effect | What to do |
| --- | --- | --- |
| Require a pull request before merging | Blocks direct pushes | Bypass or temporarily disable |
| Require status checks to pass before merging | Blocks pushes failing checks | Bypass or temporarily disable |
| Require linear history | Blocks merge commits (not your issue here, but worth noting) | Usually fine — leave alone |
| Restrict who can push to matching branches | Limits who can push | Ensure you are listed |
| Allow force pushes | Off by default | Enable, restricted to you, for the operation |
| Allow deletions | Off by default | **Leave off** — you are not deleting |
| Do not allow bypassing the above settings | Blocks admin bypass | Untick for the operation |

### Repository rulesets (newer system)

Found under **Settings → Rules → Rulesets**. Rulesets can stack — multiple rulesets can apply to `main` simultaneously, and **all of them** must allow your push. Common rules to check:

- Restrict deletions
- Restrict force pushes ← this is what you're working around
- Require a pull request before merging
- Require status checks to pass
- Block force pushes

For each ruleset that applies to `main`, check the **Bypass list** — adding yourself there is generally the cleanest way to perform this operation without weakening protection for anyone else.

### Verifying which rules apply

In the GitHub UI, go to **Settings → Rules → Rulesets** and use the "View rule insights" or "Rule insights" feature, or use:

```bash
gh api repos/<org>/<repo>/rules/branches/main
```

This lists every rule currently applied to `main`, regardless of which ruleset it came from.

---

*End of guide.*
