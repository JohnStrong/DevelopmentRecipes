# Git Recipes

A collection of useful git commands and configurations picked up along the way — learned from Kiro, AI tools, and daily development. Use as you like! 

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
