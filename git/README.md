# Git Recipes

A collection of useful git commands and configurations picked up along the way — learned from Kiro, AI tools, and daily development. Use as you like! 

## Pre-push Hooks

### Block non-merge commits to the master/main remote repository branch

```bash
#!/bin/sh
#
# pre-push hook: Block direct pushes to main/master.
# Allows pushes that contain ONLY merge commits (from feature branch merges).
#
# Install: cp git-hooks/pre-push .git/hooks/pre-push && chmod +x .git/hooks/pre-push
# Or:      git config core.hooksPath git-hooks

PROTECTED_BRANCHES="main master"

remote="$1"

while read local_ref local_sha remote_ref remote_sha; do
  # Extract branch name from ref
  branch=$(echo "$remote_ref" | sed 's|refs/heads/||')

  # Check if this is a protected branch
  is_protected=0
  for protected in $PROTECTED_BRANCHES; do
    if [ "$branch" = "$protected" ]; then
      is_protected=1
      break
    fi
  done

  if [ "$is_protected" -eq 0 ]; then
    continue
  fi

  # Handle new branch creation (remote_sha is all zeros)
  zero="0000000000000000000000000000000000000000"
  if [ "$remote_sha" = "$zero" ]; then
    echo "ERROR: Cannot create protected branch '$branch' via push."
    echo "       Protected branches: $PROTECTED_BRANCHES"
    exit 1
  fi

  # Check if local_sha is a delete (all zeros) — allow branch deletion? No, block it.
  if [ "$local_sha" = "$zero" ]; then
    echo "ERROR: Cannot delete protected branch '$branch' via push."
    exit 1
  fi

  # Get all commits between remote HEAD and local HEAD
  # These are the commits that would be pushed
  commits=$(git rev-list "$remote_sha..$local_sha")

  if [ -z "$commits" ]; then
    # Nothing to push, allow
    continue
  fi

  # Check each commit — if ANY is not a merge commit, block the push
  non_merge_found=0
  for commit in $commits; do
    parent_count=$(git cat-file -p "$commit" | grep -c '^parent ')
    if [ "$parent_count" -lt 2 ]; then
      non_merge_found=1
      short=$(git log --oneline -1 "$commit")
      echo ""
      echo "BLOCKED: Non-merge commit found in push to '$branch':"
      echo "  $short"
      echo ""
    fi
  done

  if [ "$non_merge_found" -eq 1 ]; then
    echo "═══════════════════════════════════════════════════════════════"
    echo " Push to '$branch' REJECTED"
    echo ""
    echo " Only merge commits are allowed on protected branches."
    echo " To get your changes onto '$branch':"
    echo ""
    echo "   git checkout $branch"
    echo "   git merge --no-ff <your-feature-branch>"
    echo "   git push origin $branch"
    echo ""
    echo " This ensures all changes go through feature branches first."
    echo "═══════════════════════════════════════════════════════════════"
    exit 1
  fi

  echo "✓ Push to '$branch' allowed (all commits are merges from feature branches)"

done

exit 0
```

**What it does:**

- Runs automatically before every `git push` as a [pre-push hook](https://git-scm.com/docs/githooks#_pre_push).
- Inspects each commit about to be pushed to `main` or `master`.
- If any commit has fewer than two parents (i.e., it's not a merge commit), the push is rejected.
- Merge commits (created by `git merge --no-ff`) are allowed through, since they represent completed feature-branch work.

**When it's useful:**

- You want to enforce a "merge only" policy on your primary branch — no direct commits.
- Prevents accidental pushes of WIP or unreviewed work straight to `main`/`master`.
- Encourages a feature-branch workflow where all changes land via merge.
- Works as a local safety net even when the remote doesn't have branch protection rules.

**Install it:**

```bash
# Copy into the repo's hooks directory
cp pre-push .git/hooks/pre-push && chmod +x .git/hooks/pre-push

# Or point the repo at a shared hooks folder
git config core.hooksPath git-hooks
```

**Customise protected branches:**

Edit the `PROTECTED_BRANCHES` variable at the top of the script to add or remove branch names.

**Bypass it (when you really need to):**

```bash
git push --no-verify origin main
```

Use sparingly — the hook exists for a reason.

## SSH

### Use a specific SSH key for a single repo

```bash
git config --local core.sshCommand "ssh -i ~/.ssh/<private-ssh-key> -o IdentitiesOnly=yes"
```

**What it does:**

- `--local` scopes the config to the current repository only — other projects are unaffected.
- `-i ~/.ssh/<private-ssh-file>` tells SSH to use that specific private key.
- `-o IdentitiesOnly=yes` prevents the SSH agent from offering other keys, ensuring only the specified key is tried.

**When it's useful:**

- You have multiple GitHub/GitLab accounts (personal vs work) and need different keys per repo.
- A project lives on a different remote (e.g., personal GitHub) than your default SSH key targets.
- You want to avoid editing `~/.ssh/config` with `Host` blocks for a one-off repo.
- You're pushing from a shared machine and want to isolate credentials per project.

**Verify it's set:**

```bash
git config --local core.sshCommand
# → ssh -i ~/.ssh/id_ed25519_ka -o IdentitiesOnly=yes
```

**Remove it later:**

```bash
git config --local --unset core.sshCommand
```
