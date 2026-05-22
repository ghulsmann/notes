# Promoting `release` to `main` After Go-Live

**A short proposal for the cleanest, safest way to do this**

---

## The ask, in plain English

> "We've gone live. The `release` branch *is* the live code. I want `main` to become `release` — exactly, byte for byte. I do **not** want GitHub trying to merge anything. I don't want any automatic conflict resolution. I just want the swap."

That's the whole brief. Everything that follows is built around honouring that, with the right safety net.

---

## Why we're not doing a merge

A merge sounds like the obvious answer, but in this scenario it's the wrong tool:


- **`release` is already the source of truth.** It's what's running in production. We're not trying to combine work — we're trying to acknowledge reality.
- **A merge invites conflicts.** Where `main` has diverged from `release`, Git (or worse, a GitHub auto-resolver) will try to reconcile. Every one of those decisions is a chance to silently corrupt the production codebase.
- **A merge produces a commit that is neither branch.** We'd end up with a new commit on `main` that doesn't match what's actually live. That defeats the point.

We want `main` to **point at the same commit as `release`**. No new commits, no reconciliation, no drama.

---

## The recommended approach

**Reset `main` to match `release` and force-push, with branch protection lifted briefly and a backup taken first.**

Six steps. Twenty minutes if nothing surprising happens.

---

## Step 1 — Pre-flight checks

Before touching anything:

- Confirm `release` is genuinely what's deployed. Check the deployment record, the running version, whatever your source of truth is.
- Note the SHA of `release` HEAD and the SHA of `main` HEAD. Write them down.
- List open PRs targeting `main`. We'll need to flag these to authors afterwards.
- Tag the release point so it's preserved as an immutable reference:

```bash
git fetch origin
git tag -a v<version> origin/release -m "Production release <date>"
git push origin v<version>
```

---

## Step 2 — Create a safety net

A backup branch. Cheap, fast, and means we can never truly lose `main`'s history.

```bash
git checkout main
git pull origin main
git branch main-backup-$(date +%Y-%m-%d)
git push origin main-backup-$(date +%Y-%m-%d)
```

If anything goes wrong, we can restore `main` from this branch in seconds.

---

## Step 3 — Temporarily lift branch protection on `main`

This is the bit that needs a repo admin. In GitHub:

**Settings → Branches → Branch protection rules → Edit the rule for `main`**

Two options:

- **Cleanest:** Tick *"Allow force pushes"* and *"Allow specified actors to bypass required pull requests"*, then add the person doing the swap.
- **Simplest:** Disable the rule entirely for the duration of the swap.

**Take a screenshot of the current rules before you change anything.** That's how we make sure they get restored precisely.

---

## Step 4 — The swap itself

```bash
git fetch origin
git checkout main
git reset --hard origin/release
git push --force-with-lease origin main
```

A note on `--force-with-lease` instead of `--force`: it refuses to push if someone else has pushed to `main` since we last fetched. It's the difference between "make it so" and "make it so, unless someone got there first" — and the latter is what we want.

After the push, verify:

```bash
git rev-parse origin/main
git rev-parse origin/release
```

These two SHAs should be **identical**. If they are, the swap worked.

---

## Step 5 — Restore branch protection

Back to GitHub settings. Restore the rule exactly as it was, using the screenshot from Step 3 as the reference.

Don't skip this. The window where `main` is unprotected should be as short as we can make it.

---

## Step 6 — Communicate to the team

Everyone with a local clone of the repo needs to reset their `main`, because we've rewritten its history. Send something like:

> Hey all — `main` has been reset to match `release` as part of go-live. If you have a local clone, please run:
>
> ```bash
> git fetch origin
> git checkout main
> git reset --hard origin/main
> ```
>
> Don't try to pull `main` normally — it will look like a divergence and Git will suggest a merge. Reset, don't pull.

---

## Things worth considering

These aren't blockers, but they're worth thinking about before the swap, not after.

### Open pull requests targeting `main`

PRs that were opened against the old `main` are still valid in GitHub's eyes, but the diff will look enormous and weird, because the base has effectively jumped forward. Authors will likely need to:

- Rebase the PR branch on top of the new `main`, or
- Close and reopen against the new state.

Flag this to PR authors in advance.

### CI/CD pipelines

A force-push to `main` will trigger anything that's configured to run on push to `main`. Worth asking:

- Will this re-trigger a deploy? (Probably undesirable — we already deployed.)
- Will it kick off long-running test suites? (Usually harmless, just noisy.)
- Are there tag-based or release-based pipelines that need disabling for the window?

If there's a "skip CI" mechanism available, this is a reasonable time to use it.

### GitHub Releases and tags

These point to specific commits, not branches, so they're unaffected. Worth confirming the tag from Step 1 was created successfully — that's our immutable reference to "this is what we shipped."

### Audit trail

The force-push is recorded in GitHub's audit log. For regulated environments, it's worth attaching a short note to the change record explaining why: "Promotion of `release` to `main` post go-live, in lieu of merge, to preserve byte-for-byte parity with deployed code."

### Local feature branches

Anyone with a feature branch based on the old `main` will find their branch is now based on something that no longer exists in the same form. They'll need to rebase onto the new `main` when they're next working on it. Not urgent, but worth mentioning.

---

## A safer-feeling alternative: rename instead of force-push

If the idea of a force-push on `main` is unsettling, there's a no-force-push alternative:

1. Rename `main` to `main-archive-<date>` (GitHub UI: branch settings → rename).
2. Rename `release` to `main`.
3. GitHub automatically retargets open PRs and updates the default branch setting.

**Trade-offs versus the force-push approach:**

- **Pro:** No force-push. No history rewrite on `main`.
- **Pro:** GitHub handles PR retargeting for you.
- **Con:** `release` ceases to exist as a branch (it's now `main`), which may break tooling, scripts, or workflows that expect `release` to be present.
- **Con:** Anyone with `release` checked out locally is in the same boat — they need to rename their local branch.

If `release` is a long-lived branch in your workflow (which it sounds like it is), the force-push approach is probably cleaner. If `release` was just for this go-live and won't be used again as `release`, the rename approach is arguably tidier.

---

## What we are explicitly *not* doing

To be unambiguous about the brief:

- ❌ No merge of `release` into `main`
- ❌ No cherry-picking
- ❌ No relying on GitHub to auto-resolve anything
- ❌ No rebasing `release` onto `main`

The goal is parity, not reconciliation.

---

## Risk summary

| Risk | Mitigation |
|---|---|
| `main` history lost | Backup branch created and pushed in Step 2 |
| Branch protection left off | Screenshot taken; restore is a tracked step |
| Force-push to wrong branch | Use `--force-with-lease`; verify SHAs after |
| Open PRs break | Identified in pre-flight; authors notified |
| CI re-deploys production | Reviewed before swap; pipelines paused if needed |
| Local clones diverge | Team comms with explicit reset instructions |
| Race condition (someone pushes during swap) | `--force-with-lease` refuses the push if so |

---

## Sign-off checklist

- [ ] `release` confirmed as the deployed code
- [ ] Pre-swap SHAs recorded
- [ ] Open PRs against `main` identified
- [ ] Tag created at `release` HEAD
- [ ] Backup branch created and pushed
- [ ] Branch protection rules screenshotted
- [ ] Branch protection lifted
- [ ] `main` reset to `release` and force-pushed
- [ ] Post-swap SHAs verified identical
- [ ] Branch protection restored and verified
- [ ] Team notified with reset instructions
- [ ] CI/CD status checked

---

## The one-line summary, if anyone asks

We're pointing `main` at the same commit as `release`, with branch protection briefly lifted and a backup taken — no merge, no auto-resolution, no risk of `main` containing anything that isn't already in production.
