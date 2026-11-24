# Api-Collection

Bruno workspace holding the API collections of several projects (OpenCollection YAML format).

## Open in Bruno

Open this folder as a workspace in Bruno. Every collection listed in `workspace.yml` loads.

## Collections

| Collection | Path | Source project | Env keys |
|---|---|---|---|
| apicollection | `collections/apicollection` | Kotlin Spring Boot greeting API | `API_KEY` (`X-API-KEY` header on POST /api/greetings) |

## Secrets

Real secrets never get committed. Each collection has a `.env.example`:

```bash
cp collections/apicollection/.env.example collections/apicollection/.env
# fill in real values; .env is gitignored
```

Environment files reference them as `{{process.env.KEY}}`.

## Secret scanning before each commit (pre-commit + gitleaks)

A pre-commit hook runs [gitleaks](https://github.com/gitleaks/gitleaks) on your staged changes and blocks the commit if it finds a secret. It is configured in `.pre-commit-config.yaml`. Every team member runs the setup once per clone.

### One-time setup

```bash
# 1. Install the pre-commit framework (macOS; elsewhere: pipx install pre-commit)
brew install pre-commit

# 2. From the repo root, install the git hook into .git/hooks
pre-commit install

# 3. Download and build gitleaks now instead of during your first commit (takes a minute or two)
pre-commit install-hooks
```

You don't need to install Go or gitleaks yourself. pre-commit downloads Go and builds the gitleaks version pinned in `rev:`.

### Check that it works

```bash
pre-commit run gitleaks
```

`Passed`, `Failed` or `(no files to check)Skipped` means the hook is installed. An error means it is not.

### Good to know

- Only **staged** content is scanned. If you edit a file after `git add`, stage it again before committing.
- If a commit is blocked, replace the literal value with a `{{variable}}` and put the real value in `.env`.
- Upgrade gitleaks with `pre-commit autoupdate` and commit the updated `.pre-commit-config.yaml`. Teammates get the new version on their next commit. `pre-commit gc` removes old cached versions.
- The hook can be skipped with `git commit --no-verify`. Don't do that to get a secret past it.

## Contributing: protected `main` and GitHub-side checks

`main` is protected by a branch ruleset. Direct pushes to `main` are rejected, so every change goes through a pull request.

### Workflow

```bash
git switch -c <short-description>      # work on a branch
git add <files> && git commit          # pre-commit runs gitleaks
git push -u origin <short-description> # open a PR from the link git prints
```

A PR can be merged into `main` only when:
- the repo owner has approved it (`.github/CODEOWNERS` assigns every file to the owner)
- the approval is still current (new commits dismiss earlier approvals)
- the `gitleaks` check passes

Force pushes to `main` and deleting `main` are blocked.

### Secret scanning on GitHub

| Layer | When | Effect |
|---|---|---|
| Push protection (GitHub secret scanning) | During `git push` | Rejects pushes containing known token formats |
| `gitleaks` workflow (`.github/workflows/gitleaks.yml`) | After every push to any branch | Fails the run and emails the owner. The push is already on GitHub at this point. |
| `gitleaks` required check | On every PR to `main` | Blocks the merge until the leak is removed |

Every commit in a push or PR is scanned, not only the latest one. Removing a secret in a later commit does **not** make the check pass, and the secret stays in history.

### If gitleaks or push protection flags your change

1. **Rotate the secret immediately.** Once pushed, it's public, even on a branch that never merges.
2. Replace the literal value with a `{{variable}}` and put the real value in `.env`.
3. Ask the repo owner how to clean up the branch history before opening or updating the PR.

## Adding a collection

1. Create `collections/<project-name>/` with `opencollection.yml`, `environments/` and `.env.example`.
2. Add it to `workspace.yml` under `collections`.
3. Add a row to the table above.
