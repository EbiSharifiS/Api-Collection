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

## Adding a collection

1. Create `collections/<project-name>/` with `opencollection.yml`, `environments/` and `.env.example`.
2. Add it to `workspace.yml` under `collections`.
3. Add a row to the table above.
