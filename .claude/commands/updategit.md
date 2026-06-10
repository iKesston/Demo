# /updategit — Safe GitHub publish workflow

Run all six steps below in order. Stop and report to the user if any step fails or finds a problem. Do not skip steps.

---

## Step 1 — Security scan (MUST pass before anything is pushed)

Scan every tracked and staged file for secrets and sensitive data before touching git.

Run the following checks using Grep across the working tree (exclude `.git/`):

**Patterns to flag as BLOCKERS — do not push if any are found:**
- Private key headers: `-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY`
- AWS access key IDs: `AKIA[0-9A-Z]{16}`
- Generic high-confidence tokens: `ghp_[A-Za-z0-9]{36}`, `gho_`, `github_pat_`, `xoxb-`, `xoxp-`, `sk-[A-Za-z0-9]{32,}`
- Hardcoded passwords in assignments: `password\s*=\s*['"][^'"]{4,}['"]`, `passwd\s*=\s*['"][^'"]{4,}['"]`
- Connection strings: `(mongodb|mysql|postgres|redis):\/\/[^:]+:[^@]+@`
- `.env` files that are not in `.gitignore`: check if any `.env` file exists and would be committed

**Patterns to flag as WARNINGS (report but do not block):**
- `api_key`, `apikey`, `secret_key`, `access_token`, `auth_token` appearing in non-comment JS/HTML
- Any file matching `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.crt` in the working tree
- `Authorization: Bearer` or `Authorization: Basic` hardcoded in source files

If any BLOCKER pattern is found: print the file name, line number, and matched text. Tell the user to remove the secret and use an environment variable or server-side proxy instead. **Do not proceed to Step 2.**

If only warnings are found: show them, ask the user to confirm they are intentional, then continue.

If the scan is clean: report "Security scan passed — no secrets detected." and continue.

---

## Step 2 — Create or update README.md

Read the current state of `index.html` and any other source files. Then create or overwrite `README.md` with:

1. **Project title and one-line description** (derive from the page `<title>` and hero heading)
2. **Live site link** — `https://ikesston.github.io/learn/` (update if the repo name differs)
3. **Stack** — list the technologies used (HTML/CSS/JS, any CDN fonts/libs, FormSubmit, etc.)
4. **Running locally** — how to open or serve the file
5. **Page sections** — a brief table of sections and their purpose
6. **Form setup** — how to change the recipient email
7. **Deployment** — note that pushing to `main` triggers GitHub Pages via the Actions workflow at `.github/workflows/pages.yml`

Keep it concise — no filler paragraphs.

---

## Step 3 — Stage, commit, and push

1. Check `git status` to see what has changed.
2. Stage only source files. Specifically:
   - `index.html`
   - `README.md`
   - `.github/workflows/pages.yml`
   - Any other HTML/CSS/JS/asset files
   - Do **not** stage: `.env`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `node_modules/`, any file flagged in Step 1
3. Write a concise commit message summarising what changed.
4. Push to `origin main`.

---

## Step 4 — Enable GitHub Pages and update repo About

Run these `gh` CLI commands (check `gh auth status` first; if not authenticated, print the commands for the user to run manually):

```bash
# Enable GitHub Pages from the Actions source
gh api repos/iKesston/learn/pages \
  --method POST \
  --field source='{"branch":"main","path":"/"}' \
  2>/dev/null || echo "Pages already enabled or use Settings > Pages > Source: GitHub Actions"

# Update repo description and homepage
gh repo edit iKesston/learn \
  --description "Investment strategy consultancy landing page — static HTML/CSS/JS" \
  --homepage "https://ikesston.github.io/learn/"
```

If `gh` is not available, print both commands and tell the user to run them, or explain how to do it in the GitHub UI (Settings → Pages → Source: GitHub Actions; About section on the main repo page).

---

## Step 5 — Confirm GitHub Actions workflow is valid

Read `.github/workflows/pages.yml` and verify:
- The `on.push.branches` includes `main`
- The `permissions` block grants `pages: write` and `id-token: write`
- The deploy job uses `actions/deploy-pages@v4` or later
- The `path` in `upload-pages-artifact` points to `.` (repo root, since this is a single-file site)

Report any problems found and fix them before finishing.

---

## Step 6 — Summary

Print a short summary:
- Security scan result
- Files committed and pushed
- Live URL: `https://ikesston.github.io/learn/` (note: first deployment takes ~1–2 minutes after the Actions run completes)
- GitHub Actions workflow status URL: `https://github.com/iKesston/learn/actions`
- Any manual steps the user still needs to take (e.g. enabling Pages in Settings if the API call failed, or authenticating `gh`)
