---
description: Security-scan, push to GitHub, deploy GitHub Pages, and update README + About section
argument-hint: "[github repo url]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, mcp__playwright__browser_navigate, mcp__playwright__browser_resize, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_close
---

# Publish this project to GitHub

Repo URL argument (may be empty): `$ARGUMENTS`

Run the steps **in order**. Step 1 is a gate: nothing is pushed until it passes.
If any step fails, stop, report exactly what failed, and do not continue to the next step.

## Step 0 — Resolve the target repo

1. `git remote -v` and `gh auth status`.
2. Decide the target:
   - If `$ARGUMENTS` contains a URL, that is the target. If `origin` exists and differs,
     **ask the user** whether to repoint `origin` (`git remote set-url origin <url>`) or add a new remote — do not silently overwrite.
   - If `$ARGUMENTS` is empty and `origin` exists, use `origin` and say so.
   - If both are missing, ask the user for the repo URL and stop until they answer.
3. Confirm the repo exists: `gh repo view <owner>/<repo>`. If it does not, ask whether to create it
   (`gh repo create <owner>/<repo> --public --source=. --remote=origin`) and whether it should be public or private.
   **A private repo cannot serve GitHub Pages on a free plan** — if the user wants Pages, it must be public.

## Step 1 — Security scan (blocking, before anything leaves the machine)

This code is going to the public internet. Scan the **working tree and the git history**:

```bash
# Secrets and credentials in tracked files
git ls-files -z | xargs -0 grep -inE "api[_-]?key|secret|passwd|password|token|bearer |authorization:|private[_-]?key|BEGIN (RSA|OPENSSH|EC|PGP) PRIVATE KEY|AKIA[0-9A-Z]{16}|gh[pousr]_[A-Za-z0-9]{20,}|sk-[A-Za-z0-9]{20,}|xox[baprs]-|AIza[0-9A-Za-z_-]{35}"

# Files that should never be published
git ls-files | grep -inE "\.env|\.pem$|\.key$|\.pfx$|\.p12$|id_rsa|credentials|\.pypirc|\.npmrc|secrets?\.(json|ya?ml|txt)"

# Personal data: emails, internal hosts, absolute local paths
git ls-files -z | xargs -0 grep -inE "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}|\b10\.[0-9]+\.[0-9]+\.[0-9]+\b|192\.168\.|\.internal\b|\.corp\b|C:\\Users\\\\"

# History, in case something was committed and later removed
git log --oneline -- $(git ls-files | grep -iE "\.env|\.pem$|id_rsa|credentials" || echo /dev/null)
```

Also read this project's `CLAUDE.md` constraints and verify they still hold before publishing:

```bash
grep -inE "localStorage|sessionStorage|indexedDB|document\.cookie|!important" index.html
grep -inE "https?://|<script src|<link |@import" index.html   # only FORMSUBMIT_ENDPOINT may match
```

Then judge the hits — grep is noisy, so classify each one rather than dumping the output:

- **Real secret / private key / credential file** → STOP. Report it, remove it, add it to `.gitignore`, and if it is in history tell the user the history must be rewritten (`git filter-repo`) and the credential rotated. Do not push.
- **Personal or internal data** (real email addresses, internal hostnames, employee names, customer data, absolute `C:\Users\...` paths) → STOP and ask before pushing.
- **Placeholder / example values** (`YOUR_EMAIL@example.com`, docs, this command file itself) → fine, note it and continue.

Print a short verdict line: `Security scan: PASS — N matches reviewed, all placeholders/docs` (or the failure and what to do).
Never push on a scan you did not actually read.

## Step 2 — Commit and push

1. `git status --short` and `git diff --stat`. Review what is being published.
2. Ensure a `.gitignore` exists covering at minimum `.env*`, `*.pem`, `*.key`, `node_modules/`, OS cruft.
3. If on a non-`main` branch, ask before pushing to `main`.
4. Stage, commit with a descriptive message, push:
   ```bash
   git add -A && git commit -m "<message>" && git push -u origin main
   ```
   Follow the repo's commit attribution convention.

## Step 3 — GitHub Pages via Actions

1. If `.github/workflows/deploy-pages.yml` already exists, read it and only edit it if it is broken or stale.
   Otherwise create it — for a static site, publish the repo root:

   ```yaml
   name: Deploy to GitHub Pages
   on:
     push:
       branches: [main]
     workflow_dispatch:
   permissions:
     contents: read
     pages: write
     id-token: write
   concurrency:
     group: pages
     cancel-in-progress: true
   jobs:
     deploy:
       environment:
         name: github-pages
         url: ${{ steps.deployment.outputs.page_url }}
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/configure-pages@v5
         - uses: actions/upload-pages-artifact@v3
           with:
             path: .
         - id: deployment
           uses: actions/deploy-pages@v4
   ```

2. Set the Pages source to Actions (idempotent; ignore "already enabled"):
   ```bash
   gh api -X POST repos/{owner}/{repo}/pages -f build_type=workflow || \
   gh api -X PUT  repos/{owner}/{repo}/pages -f build_type=workflow
   ```
3. Commit and push the workflow if it changed, then watch the run:
   ```bash
   gh run list --workflow=deploy-pages.yml --limit 1
   gh run watch $(gh run list --workflow=deploy-pages.yml --limit 1 --json databaseId -q '.[0].databaseId')
   ```
   If it fails, read the logs (`gh run view <id> --log-failed`), fix, and re-run — do not report success on a red run.
4. Capture the live URL: `gh api repos/{owner}/{repo}/pages -q .html_url`
   (normally `https://<owner>.github.io/<repo>/`). For a single-file app whose entry is not `index.html`,
   append the filename to the URL.

## Step 4 — Screenshot with Playwright

Capture a current screenshot of the app for the README. The Playwright MCP browser **blocks the
`file:` protocol**, so serve the directory over HTTP first rather than opening `index.html` directly:

```bash
python -m http.server 8765 &   # from the repo root; then curl -sI http://localhost:8765/index.html
```

Then, with the Playwright MCP tools:

1. `browser_resize` to 1440x900 — a desktop viewport, so the columns render side by side rather
   than stacked at the mobile breakpoint.
2. `browser_navigate` to `http://localhost:8765/index.html`.
3. `browser_take_screenshot` with `fullPage: true` and `filename` set to an **absolute** path at
   `<repo>/docs/screenshot.png`.
4. **Read the PNG back and actually look at it** before committing it. It must show the real,
   populated UI — not a blank page, an error, or a half-loaded layout. If it is wrong, fix and retake.
5. `browser_close`, stop the HTTP server, and delete the `.playwright-mcp/` scratch directory
   Playwright leaves in the repo root (or add it to `.gitignore`).

If Playwright is unavailable, say so and continue without the screenshot — do not block the publish
on it, and do not leave a broken image link in the README.

## Step 5 — README

Create or update `README.md`. Do not blow away a good existing README — merge into it.
It should contain:

- Project title and a one-or-two-sentence description of what it actually does (read the source; do not guess).
- **Live demo:** the Pages URL from Step 3, as a link near the top.
- **The screenshot from Step 4**, just under the live-demo link:
  `![<description of what is on screen>](docs/screenshot.png)` — a relative path, so it renders both
  on GitHub and on Pages. Give it real alt text, not "screenshot".
- How to run it locally (for this project: open `index.html` by double-clicking — no build step, no server).
- Key features, taken from the real code.
- Notable constraints/caveats worth knowing (e.g. state resets on refresh, no persistence, any placeholder config the user must fill in).
- A short "Tech" line and, if applicable, a license/internal-use note.

Keep it honest and concise — no invented benchmarks, no features that do not exist. Commit and push it.

## Step 6 — About section

Set the repo's description, homepage (the Pages URL) and topics:

```bash
gh repo edit <owner>/<repo> \
  --description "<one-line description, <=350 chars>" \
  --homepage "<pages url>" \
  --add-topic <topic1> --add-topic <topic2> --add-topic <topic3>
```

Also enable the Pages link on the repo home:
```bash
gh api -X PATCH repos/{owner}/{repo} -F has_pages=true 2>/dev/null || true
```

Verify: `gh repo view <owner>/<repo> --json description,homepageUrl,repositoryTopics`

## Step 7 — Report

Finish with a compact summary:

- Security scan verdict (and anything you fixed or excluded)
- Commit pushed (sha + message) and branch
- Workflow run status and link
- **Live site URL** (verify it responds: `curl -sI <url> | head -1` — a fresh Pages deploy can take a minute or two; if it 404s, say so and tell the user to retry shortly rather than claiming success)
- Screenshot captured (and confirmed it renders in the README)
- README and About section changes

Flag anything you could not complete and why.
