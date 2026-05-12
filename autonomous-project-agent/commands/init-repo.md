---
description: Initialize a GitHub repo for the current project. Runs all the pre-flight safety checks (gitignore enforcement, credential scan) before any push. Defaults to private. Useful when retrofitting an existing project the skill didn't scaffold from scratch.
---

# /init-repo

Bootstrap a GitHub repository for the project in the current working directory. Follows the same pre-flight safety checks defined in §26 of the autonomous-project-agent skill — no push happens unless `.gitignore` is verified and no credentials are detected in tracked files.

## Steps

1. **Verify `gh` CLI is installed and authenticated.**
   - Run `gh --version`. If missing, surface install command (`brew install gh` on macOS) and stop.
   - Run `gh auth status`. If not authenticated, surface `gh auth login` and pause for that single OAuth flow.

2. **Check for existing remote.**
   - Run `git remote get-url origin 2>/dev/null`. If a remote already exists, surface it and ask whether to proceed (BLOCK if not confirmed). Never overwrite an existing remote.

3. **Pre-flight `.gitignore` check (MANDATORY).**
   - Verify `.gitignore` exists and contains: `.env`, `.env.local`, `.env.*`, `.env.*.local`, `secrets/`, `*.pem`, `*.key`. Append any missing entries.
   - For each existing `.env*` file in the project, run `git check-ignore <file>`. If any returns non-zero, **abort and surface the offending file**. Do not push.

4. **Pre-flight credential scan (MANDATORY).**
   - Scan the working tree (excluding `node_modules/`, `.git/`, `dist/`, `build/`, `.next/`, `.cache/`) for credential patterns OUTSIDE `.env*` files:
     - `sk-ant-[A-Za-z0-9_-]{20,}` (Anthropic)
     - `sk-[A-Za-z0-9]{40,}` (OpenAI-style)
     - `AIza[0-9A-Za-z_-]{35}` (Google API keys — flag if in tracked files outside `.env*`)
   - If any match outside `.env*`, **abort and surface the offending file + line**. Do not push.

5. **Ask for repo name and visibility** (skipped in autopilot — uses intake values).
   - Default repo name: current directory basename.
   - Default visibility: `private`.

6. **Create the repo:**
   ```
   gh repo create <user>/<repo-name> --private --source=. --remote=origin --description "<one-line description>"
   ```

7. **Initial commit (if no commits yet):**
   ```
   git add -A
   git commit -m "Initial scaffold from autonomous-project-agent"
   ```

8. **Push `main`:**
   ```
   git push -u origin main
   ```

9. **Enable branch protection on `main` (PR-only):**
   ```
   gh api -X PUT repos/<user>/<repo>/branches/main/protection \
     -f required_pull_request_reviews=null \
     -f enforce_admins=true \
     -f required_status_checks=null \
     -f restrictions=null
   ```

10. **Write `.github/dependabot.yml`** for the detected package manager. Commit and push.

11. **Create `dev` integration branch** for subsequent feature work:
    ```
    git checkout -b dev && git push -u origin dev
    ```

12. **Log to `ai_docs/links.md`** under Projects: repo URL, GitHub Actions URL, Dependabot URL, secret scanning URL.

13. **Report success:**
    ```
    Repository created: https://github.com/<user>/<repo>
    Visibility: private
    Branch protection: enabled on main (PR-only)
    Integration branch: dev
    ```

## Failure modes

- `gh` CLI missing → BLOCK with install command.
- Not authenticated → BLOCK with `gh auth login`.
- `.gitignore` enforcement fails → CRITICAL BLOCK, list offending files.
- Credential scan finds keys outside `.env*` → CRITICAL BLOCK, list offending files and lines.
- Repo already exists on GitHub → BLOCK, ask for a different name.
- Push fails → retry once, then BLOCK with the underlying error.

## Notes

- This command is identical to the §26 bootstrap sequence in the autonomous-project-agent skill. The skill calls it automatically during a full run; this slash command exists for manual invocation on existing projects the skill didn't scaffold.
- Defaults to **private**. Public requires explicit `--public` flag (not implemented as a parameter here — edit visibility in the GitHub UI if needed).
- **Never auto-merges PRs.** Even if you later open PRs via the skill, they stay open until you merge them.
