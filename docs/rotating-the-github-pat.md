# Rotating the GitHub Personal Access Token

This repository authenticates its Upptime workflows with a GitHub Personal Access Token (PAT).
The token has an expiration date and **must be rotated before it expires**.

## What the token is

| | |
|---|---|
| **Token name** (in GitHub Developer settings) | `GH_PAT_KUFLOW_STATUS_PAGE` |
| **Repository secret name** | `GH_PAT` |
| **Repository** | `kuflow/kuflow-status` |
| **Required scopes** (classic PAT) | `repo`, `workflow` |

The names differ: the token is called `GH_PAT_KUFLOW_STATUS_PAGE` in the account's Developer
settings, but it is stored in this repository as the Actions secret `GH_PAT`. Rotating means
generating a new token value and updating that secret.

## Where it is used

Every Upptime workflow reads it, both to check out the repository and to run the monitor:

- [`uptime.yml`](../.github/workflows/uptime.yml)
- [`response-time.yml`](../.github/workflows/response-time.yml)
- [`graphs.yml`](../.github/workflows/graphs.yml)
- [`summary.yml`](../.github/workflows/summary.yml)
- [`updates.yml`](../.github/workflows/updates.yml)
- [`site.yml`](../.github/workflows/site.yml)
- [`setup.yml`](../.github/workflows/setup.yml)
- [`update-template.yml`](../.github/workflows/update-template.yml)

They all reference it as `${{ secrets.GH_PAT || github.token }}`.

> [!WARNING]
> Because of that `|| github.token` fallback, **an expired token does not produce a failing
> build**. The workflows silently fall back to the default `GITHUB_TOKEN`, and commits pushed
> with it [do not trigger other workflows](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#using-the-github_token-in-a-workflow).
> The monitoring commits would keep landing, but `Static Site CI` would stop being triggered by
> them and the published status page would go stale. Rotate *before* the expiration date.

## Rotation steps

### 1. Generate a new token value

The simplest option is to regenerate the existing token, which preserves its name and scopes:

1. Go to <https://github.com/settings/tokens>.
2. Open `GH_PAT_KUFLOW_STATUS_PAGE`.
3. Click **Regenerate token** and pick a new expiration.
4. Copy the new value (`ghp_…`). It is shown only once.

If you create a brand new token instead, give it the `repo` and `workflow` scopes. The
`workflow` scope is required because `update-template.yml` commits files under
`.github/workflows/`.

> [!IMPORTANT]
> Generate the token from **the same account that owns the current one** — an account with write
> access to `kuflow/kuflow-status`. Using a different account changes the identity that pushes
> the monitoring commits.

### 2. Update the repository secret

```bash
gh secret set GH_PAT --repo kuflow/kuflow-status
```

The command prompts for the value with hidden input, so the token never reaches your shell
history. Alternatively, update it through the web UI at
**Settings → Secrets and variables → Actions → `GH_PAT`**.

### 3. Verify

Trigger a run manually and confirm it succeeds:

```bash
gh workflow run "Uptime CI" --repo kuflow/kuflow-status
gh run list --repo kuflow/kuflow-status --limit 5
```

A green run is sufficient proof that the token is valid: `actions/checkout` fails outright when
given an expired token. For a stronger check, confirm that the commit produced by `Uptime CI`
triggers a `Static Site CI` run — that chaining is exactly what the default `GITHUB_TOKEN`
cannot do.

You can also confirm the secret's update timestamp:

```bash
gh api repos/kuflow/kuflow-status/actions/secrets/GH_PAT --jq '{name,updated_at}'
```

### 4. Revoke the old token

If you created a new token rather than regenerating the existing one, delete the old one from
<https://github.com/settings/tokens>.

## Notes

- **Check for other copies of the token.** If the same PAT is also stored as an organization
  secret or in another repository, it must be updated there too. Listing organization secrets
  requires org admin rights: `gh secret list --org kuflow`.
- **Consider migrating to a fine-grained PAT** scoped to `kuflow-status` only, with
  `Contents: write`, `Workflows: write`, `Pages: write` and `Issues: write` (Upptime opens and
  closes issues as incident reports). This narrows the blast radius considerably compared to a
  classic token with full `repo` scope. Note that fine-grained tokens owned by an organization
  may require an org owner to approve them.
- **GitHub Pages configuration is unrelated to this token.** The site is published from the
  `gh-pages` branch (custom domain `status.kuflow.com`), deployed by `site.yml`.
