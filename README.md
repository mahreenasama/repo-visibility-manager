# Repository Visibility Manager

A GitHub Actions workflow that bulk-changes the visibility of all repositories in your account: 
- Public → Private
- Private → Public
with just a single click. No local scripts, no manual clicking through dozens of repo settings pages.

I built this after ending up with 80+ public repos where only a few of them were actually meant to be seen. Flipping each one by hand wasn't worth, so I automated it.

> 📖 I wrote a full walkthrough of how this works and why: [Read the article on Medium](#) *(add your link here)*

## What it does

- Lists every repository under your account matching a given visibility (`public` or `private`)
- Flips all of them to the opposite visibility in one workflow run
- Skips the repository the workflow itself lives in, so it never locks itself out
- Runs entirely inside GitHub Actions — nothing to install locally

## How it works

```
You trigger the workflow (pick a direction from a dropdown)
        │
        ▼
GitHub Actions spins up a runner and installs GitHub CLI
        │
        ▼
GitHub CLI authenticates using a Personal Access Token (PAT) stored in Secrets
        │
        ▼
gh repo list  →  fetches all repos matching the source visibility
        │
        ▼
gh repo edit  →  updates each one to the target visibility
```

## Setup

### 1. Create a Fine-grained Personal Access Token (PAT)

Go to **GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens** and generate a new one:

- **Repository access:** All repositories
- **Permissions:** `Administration` → Read and write
- Set an expiration date (don't use "No expiration")
- Create a PAT (and do not forgot to copy it)

This is the only permission required — repository visibility is an administrative setting, not a content or metadata one.

### 2. Add it as a repository secret

In this repository, go to **Settings → Secrets and variables → Actions → New repository secret**:

- **Name:** `PAT`
- **Value:** paste the copied token you just generated

### 3. Run the workflow

Go to the **Actions** tab → **Repository Visibility Manager** → **Run workflow**, choose a direction, and confirm:

- `public-to-private` — makes all your public repos private
- `private-to-public` — makes all your private repos public

Watch the run logs to see each repository as it's updated.

## The workflow file

```yaml
name: Repository Visibility Manager
on:
  workflow_dispatch:
    inputs:
      action:
        description: "Select the operation"
        required: true
        type: choice
        options:
          - public-to-private
          - private-to-public
jobs:
  update-visibility:
    runs-on: ubuntu-latest
    steps:
      - name: Install GitHub CLI
        run: |
          type -p gh || (
            curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg |
            sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg &&
            sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg &&
            echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" |
            sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null &&
            sudo apt update &&
            sudo apt install gh -y
          )
      - name: Update repository visibility
        env:
          GH_TOKEN: ${{ secrets.PAT }}
          OWNER: ${{ github.repository_owner }}
          ACTION: ${{ github.event.inputs.action }}
        run: |
          if [ "$ACTION" = "public-to-private" ]; then
            FROM="public"
            TO="private"
          else
            FROM="private"
            TO="public"
          fi
          echo "Changing all $FROM repositories to $TO..."
          gh repo list "$OWNER" \
            --visibility "$FROM" \
            --limit 1000 \
            --json nameWithOwner \
            --jq '.[].nameWithOwner' |
          while read repo; do
            if [ "$repo" = "${{ github.repository }}" ]; then
              echo "Skipping $repo"
              continue
            fi

            echo "Updating $repo..."
            gh repo edit "$repo" \
              --visibility "$TO" \
              --accept-visibility-change-consequences
          done
          echo "Done."
```

## ⚠️ Important notes

- **This is a bulk, account-wide action.** Once you confirm, it affects *every* matching repository — there's no per-repo selection or undo. Double-check the direction before you run it.
- Changing a repo from public to private can break external links, GitHub Pages sites, and any integrations that assumed it was public.
- The workflow always skips the repo it's running in, so you won't accidentally lock yourself out of the Actions tab.
- Keep your PAT's expiration date short and rotate it periodically. If the token expires, just update the `PAT` secret with a new value — no changes needed to the workflow itself.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `403 Resource not accessible by personal access token` | Token is missing the `Administration` permission | Regenerate the PAT with `Administration: Read and write` |
| `use of --visibility flag requires --accept-visibility-change-consequences` | GitHub CLI safety guard | Already included in the workflow above — make sure you haven't removed it |

## Roadmap / ideas

- [ ] Exclude list for repos that should never be touched
- [ ] Regex/pattern filter (e.g. only `archive-*` repos)
- [ ] Dry-run mode to preview changes before applying them
- [ ] Organization-level support
- [ ] Run summary posted to Slack or email

## License

MIT — use it, fork it, adapt it.
