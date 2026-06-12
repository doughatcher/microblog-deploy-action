# Deploy Hugo theme to Micro.blog — GitHub Action

> **Status: early release.** Seeded from the battle-tested deploy scripts that
> ship the `dougie-theme` sites. Validating against the proving grounds before a
> Marketplace listing.

A composite GitHub Action that authenticates to [Micro.blog](https://micro.blog) via
its email magic-link flow and deploys a Hugo theme to a hosted Micro.blog site —
caching the session cookie so it only emails a sign-in link when the cookie has
expired.

Micro.blog has no deploy API token; the only automated path is "request a sign-in
email → click the magic link." This action wraps that reliably (poll Gmail over
IMAP, follow the link, push the theme, poll `/posts/check`) so any Hugo project that
publishes through Micro.blog can deploy on push.

## Usage

```yaml
- uses: actions/checkout@v4
- uses: doughatcher/microblog-deploy-action@v1
  with:
    microblog-email:    ${{ vars.MICROBLOG_EMAIL }}
    gmail-email:        ${{ vars.GMAIL_EMAIL }}
    gmail-app-password: ${{ secrets.GMAIL_APP_PASSWORD }}
    site-id:            ${{ vars.MICROBLOG_SITE_ID }}
    theme-id:           ${{ vars.MICROBLOG_THEME_ID }}
```

See [`examples/deploy.yml`](examples/deploy.yml) for a full workflow including the
`actions/cache` step that persists `.session-cookie` between runs.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `microblog-email` | yes | — | Micro.blog account email (sign-in link target). |
| `gmail-email` | yes | — | Gmail address that receives the email (IMAP). |
| `gmail-app-password` | yes | — | Gmail **App Password** for IMAP. Use a secret. |
| `site-id` | yes | — | Micro.blog site ID. |
| `theme-id` | yes | — | Micro.blog theme ID. |
| `deploy-args` | no | `--all --timeout 120` | Extra args for `microblog_deploy.py`. |
| `python-version` | no | `3.12` | Python to set up. |

## Outputs

| Output | Description |
|--------|-------------|
| `deployed` | `"true"` when the deploy step completed. |

## How sign-in works (and why you need a Gmail App Password)

1. The action POSTs your `microblog-email` to Micro.blog's sign-in endpoint.
2. Micro.blog emails a magic link to that address.
3. The action reads that inbox over IMAP (`gmail-email` + `gmail-app-password`),
   extracts the link, and follows it to obtain a session cookie.
4. The cookie is written to `.session-cookie` and reused on later runs until it
   expires — cache it (see the example) to avoid emailing a link every run.

Generate an App Password at <https://myaccount.google.com/apppasswords> (requires
2-Step Verification). **Never commit it** — pass it as a GitHub Actions secret.

## Proving grounds

This stub is exercised against the `dougie-theme` sites before extraction:

| Site | `site-id` | Notes |
|------|-----------|-------|
| leaning.blue | `267931` | dougie-theme via Micro.blog plugin |
| superterran.net | `337434` | dougie-theme (starfield variant) |

## Roadmap to a community action

1. ✅ **Extracted** to its own public repo, `doughatcher/microblog-deploy-action`,
   with `action.yml` at the root (required for `uses:` from other repos + Marketplace).
2. **Validate** against the proving grounds above (caller workflow in `dougie-theme`).
3. **Tag** `v1` and **publish to the GitHub Marketplace**.
4. **Retire duplication**: once stable, the per-site `.github/deploy/` copies
   (the micro.blog workspace, `experience-digest`, `waccamaw`) call the action
   instead of vendoring their own scripts — killing the deploy-script drift this
   action was created to solve.

## Files

| File | Role |
|------|------|
| `action.yml` | Composite action definition. |
| `microblog_auth.py` | Email magic-link login → `.session-cookie`. |
| `microblog_deploy.py` | Theme reload + rebuild + `/posts/check` poll. `--validate-only` checks the cached cookie. |
| `requirements.txt` | `requests`, `beautifulsoup4`, `pyyaml`, `python-dotenv`. |
| `examples/deploy.yml` | Drop-in caller workflow with session caching. |

> These Python scripts are the **canonical source**. The per-site `.github/deploy/`
> copies across the doughatcher Micro.blog repos retire as those sites adopt this action.
