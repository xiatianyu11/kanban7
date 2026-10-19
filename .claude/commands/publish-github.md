---
description: Security-scan the project, then push it to GitHub with README, About section, CI/CD and GitHub Pages
argument-hint: <owner/repo or https://github.com/owner/repo> [public|private]
allowed-tools: Bash, PowerShell, Read, Write, Edit, Glob, Grep, mcp__playwright__browser_navigate, mcp__playwright__browser_resize, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_close
---

# Publish this project to GitHub

Target repository: `$ARGUMENTS`

If no repository was given above, stop and ask the user for:
- the GitHub owner and repository name (or the full repo URL)
- whether a new repo should be **public** or **private** (default: public, because free GitHub Pages needs it)

Work through the steps below in order. **Step 1 is a gate:** if it finds a problem, stop, report it and don't push anything until the user decides. Report the result of every step at the end, including steps that were skipped or failed.

Respect `CLAUDE.md` throughout: the board is a single self-contained `index.html` for a **fictitious** bank. Don't add a real bank's name or branding to any file, the README or the repo About section.

---

## Step 0 — Preflight

1. Check the tools: `git --version`, `gh --version`, `gh auth status`. If `gh` is missing or not logged in, stop and tell the user to run `gh auth login`. Don't ask for or handle tokens yourself.
2. Parse `$ARGUMENTS` into `OWNER` and `REPO`, and work out the Pages URL: `https://OWNER.github.io/REPO/`. If `REPO` is `OWNER.github.io`, the URL is `https://OWNER.github.io/` instead.
3. Run `git status` and `git log --oneline -5`. If there are uncommitted changes, list them and include them in the commit only if they belong to the project.

## Step 1 — Security scan (gate: must pass before anything leaves this machine)

The goal is to make sure nothing sensitive gets uploaded, and that the app doesn't send data anywhere it shouldn't.

1. **Review what will be published.** List every tracked file and every file that would be committed (`git ls-files`, plus `git status --porcelain`). Flag anything that shouldn't be public: `.env*`, `*.pem`, `*.key`, `*.pfx`, `id_rsa*`, credentials or config dumps, `.claude/settings.local.json`, logs, OS or editor junk (`Thumbs.db`, `.DS_Store`, `.vscode/`), archives and large binaries.
2. **Check `.gitignore`.** Create or update it so it covers the patterns above, including `.claude/settings.local.json` and `.playwright-mcp/` (Playwright MCP's snapshot and console-log output). Don't ignore `.claude/commands/`: that folder is meant to be shared.
3. **Scan for secrets.** Scan the working tree **and the full git history** (`git log -p --all`):
   - If `gitleaks` is installed, run `gitleaks detect --source . --log-opts="--all" --redact`.
   - Always also grep for common patterns: `ghp_`, `github_pat_`, `gho_`, `AKIA[0-9A-Z]{16}`, `-----BEGIN .*PRIVATE KEY-----`, `xox[baprs]-`, `sk-[A-Za-z0-9]{20,}`, `AIza[0-9A-Za-z_-]{35}`, and `password|passwd|secret|api[_-]?key|token` followed by `=` or `:` and a value.
   - If a secret is in the **history**, removing it from the current files isn't enough. Stop and tell the user. The secret must be rotated, and the history rewritten (for example with `git filter-repo`), before pushing.
4. **Check personal data.** Look for real email addresses, phone numbers, names and internal hostnames or IPs. `FORMSUBMIT_ENDPOINT` in `index.html` must stay the placeholder `https://formsubmit.co/ajax/YOUR_EMAIL@example.com` in the published copy. If it holds a real address, stop and ask the user before publishing it. Also check what the commit author email will expose (`git config user.email`), and mention GitHub's `noreply` address as an option.
5. **Check the app's outbound traffic.** Grep `index.html` for `http://`, `https://`, `fetch(`, `XMLHttpRequest`, `navigator.sendBeacon`, `WebSocket`, `<script src`, `<link href`, `@import`, `url(` and `<img src`. The **only** allowed network call is the FormSubmit `fetch` in `notifyNewTask()`. Anything else (CDNs, fonts, analytics, trackers) is a finding. Also confirm there's no `localStorage`, `sessionStorage`, `indexedDB` or `document.cookie`, so no board data is stored or leaked.
6. **Check XSS hygiene.** Every user-supplied value in `renderCard()` must go through `escapeHtml()`. Flag any `innerHTML` assignment that takes unescaped user input.

Print a short **Security scan report**: each check marked PASS, FIXED or FAIL, with file:line for findings. Continue only if nothing is FAIL.

## Step 2 — Screenshot with Playwright MCP

Capture the board so the README shows what it looks like. Use the Playwright MCP tools (`mcp__playwright__*`, configured in `.mcp.json`). If they aren't available, skip this step, keep any existing `docs/screenshot.png`, and report it as skipped.

1. **Serve the folder locally.** Playwright MCP blocks `file://` URLs, so start a throwaway server in the background from the project root: `python -m http.server 8765 --bind 127.0.0.1` (or `npx --yes http-server -p 8765 -a 127.0.0.1` if Python is missing). Capturing locally shows the version you're about to publish, not the old live one.
2. **Capture.**
   - `browser_resize` to 1440 × 900, so all four columns sit side by side.
   - `browser_navigate` to `http://127.0.0.1:8765/index.html`.
   - `browser_take_screenshot` with `filename: "docs/screenshot.png"`, `type: "png"`, `scale: "css"` (viewport only, not full page).
   - `browser_close`.
3. **Check it.** Read `docs/screenshot.png` and confirm it shows the header, the summary tiles and all four columns with seed cards, with no error state. The only expected console error is the dev server's `favicon.ico` 404.
4. **Clean up.** Stop the background server. Don't commit `.playwright-mcp/`.

The screenshot is for the README only. `index.html` must never reference it, because the board itself stays free of image files (see `CLAUDE.md`). `pages.yml` copies only `index.html`, so `docs/` isn't deployed to Pages.

## Step 3 — README.md

Create `README.md`, or update it if it exists (keep any sections the user wrote). Include:

- Title and a one-paragraph description: a single-file IT PMO Kanban board for a fictitious bank, used as a demo and training tool.
- **Screenshot** from Step 2, right under the live demo link: `![IT PMO Kanban board showing the Backlog, In Progress, Blocked and Done columns](docs/screenshot.png)`. Use the relative path so it renders on GitHub.
- **Badges** (shields.io), on one line:
  - HTML5, CSS3 and JavaScript (vanilla), for example `https://img.shields.io/badge/HTML5-E34F26?logo=html5&logoColor=white`
  - No dependencies / zero build
  - GitHub Pages deploy status: `https://github.com/OWNER/REPO/actions/workflows/pages.yml/badge.svg`
  - CI status: `https://github.com/OWNER/REPO/actions/workflows/ci.yml/badge.svg`
  - License, if a LICENSE file exists
- **Live demo** link to the Pages URL, and next to it a **Source on GitHub** link back to `https://github.com/OWNER/REPO`. Repeat the repo link at the end of "How to run" for issues and contributions.
- **Features**: four columns, drag and drop, Move menu, filters, overdue badges, in-card delete confirm, toasts, accessibility, and the optional FormSubmit email notification.
- **Tech stack**: vanilla HTML/CSS/JS in one file, with no frameworks, CDNs or build step.
- **How to install**: `git clone https://github.com/OWNER/REPO.git`. Nothing else to install.
- **How to run**:
  - double-click `index.html`, or
  - on Windows: `Start-Process index.html`, or
  - on macOS: `open index.html`, or
  - on Linux: `xdg-open index.html`, or
  - open the live GitHub Pages link.
- **Configuration**: how to set `FORMSUBMIT_ENDPOINT`, and the one-time FormSubmit activation email.
- **Notes**: there's no persistence (a refresh resets to the seed data), and all names are fictitious.
- **CI/CD**: what each workflow does.

## Step 4 — CI/CD with GitHub Actions

1. Keep `.github/workflows/pages.yml`, which deploys `index.html` to Pages on push to `main`. Check it still matches the current file layout.
2. Create or update `.github/workflows/ci.yml`. It runs on `push` and `pull_request` to `main`, with `permissions: contents: read`, and has:
   - **secret-scan**: `actions/checkout@v4` with `fetch-depth: 0`, then `gitleaks/gitleaks-action@v2` (env `GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}`).
   - **constraints**: shell checks that fail the build if `index.html` breaks the rules in `CLAUDE.md`: more than one `<style>` or `<script>` block; `!important`; `localStorage`, `sessionStorage`, `indexedDB` or `document.cookie`; `alert(` or `confirm(`; or any external `src=`/`href=` resource (http/https), with `<a href>` links and the FormSubmit endpoint as the only allowed exceptions.
   - **html-validate** (optional, non-blocking): `npx --yes html-validate index.html` with `continue-on-error: true`.
3. Optionally make `pages.yml` deploy only after CI passes (for example `workflow_run` on CI completing successfully). Ask the user first, because it changes how deploys trigger.

## Step 5 — Commit and push

1. Check that the git author name and email are what the user wants public (see Step 1.4).
2. Stage only the project files: `index.html`, `README.md`, `docs/screenshot.png`, `CLAUDE.md`, `.gitignore`, `.github/`, `.claude/commands/`, `.mcp.json` and any LICENSE. Never use `git add -A` blindly. Show `git status` before committing.
3. Commit with a clear message, then add the attribution lines the session requires.
4. Create or connect the remote:
   - If `gh repo view OWNER/REPO` fails, create it with `gh repo create OWNER/REPO --public|--private --source . --remote origin`. Confirm with the user first, because this creates a public resource.
   - If it exists, set `origin` to `https://github.com/OWNER/REPO.git` (`git remote add` or `git remote set-url`).
5. Push with `git push -u origin main`. Never force-push. If the remote has history this branch doesn't have, stop and ask the user.

## Step 6 — Enable GitHub Pages

1. Enable Pages with GitHub Actions as the source:
   `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`
   If Pages already exists (HTTP 409), switch it to Actions instead:
   `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`
2. Start the deploy if the push didn't already: `gh workflow run pages.yml`.
3. Watch it with `gh run list --workflow=pages.yml -L 1`, then `gh run watch <id> --exit-status`. If it fails, show `gh run view <id> --log-failed`, fix the cause and push again.
4. Confirm the site responds: `curl -sI https://OWNER.github.io/REPO/` should return HTTP 200. The first deploy can take a minute or two.

## Step 7 — GitHub About section

Update the repo's About panel. It must include the Pages link.

```
gh repo edit OWNER/REPO \
  --description "Single-file IT PMO Kanban board (vanilla HTML/CSS/JS) for a fictitious bank — demo & training tool" \
  --homepage "https://OWNER.github.io/REPO/" \
  --add-topic kanban --add-topic html --add-topic css --add-topic javascript \
  --add-topic vanilla-js --add-topic github-pages --add-topic no-dependencies
```

Check the result with `gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`.

## Step 8 — Final check and report

1. Run `gh run list -L 5` and confirm both CI and Pages are green.
2. Run one last check on what is now public: `git ls-files` on the pushed commit should match the intended file list, with nothing from the Step 1 deny list.
3. Report to the user:
   - Repo URL and live Pages URL
   - The security scan report from Step 1
   - Files created or changed
   - Workflow run status
   - Anything skipped, failed or needing their action (for example, FormSubmit activation, or rotating a secret)
