# Setup Log & Next Steps

## Team member setup (once per machine)

```bash
# 1. Clone and enter the repo
git clone https://github.com/EbiSharifiS/Api-Collection.git
cd Api-Collection

# 2. Create local secret files and fill in the real values (never committed)
cp collections/apicollection/.env.example collections/apicollection/.env

# 3. Install pre-commit (macOS; elsewhere: pipx install pre-commit)
brew install pre-commit

# 4. Install the git hook and build gitleaks (takes a minute or two)
pre-commit install
pre-commit install-hooks

# 5. Verify: Passed / Failed / (no files to check)Skipped = installed; an error = not installed
pre-commit run gitleaks
```

Then open the repo folder as a workspace in Bruno.

## Done

### 1. Collection format
- **Bruno OpenCollection YAML** (`.yml`), Bruno's recommended format over legacy `.bru`.
- One file per request: small diffs, few merge conflicts, readable PRs.
- Not Postman JSON: Bruno can import it but not keep it in sync.

### 2. Repo structure (Bruno workspace)

```
Api-Collection/
├── workspace.yml              # lists all collections
├── README.md                  # usage, secrets, pre-commit setup
├── SETUP.md                   # this file
├── FUTURE_IMPROVEMENTS.md     # postponed ideas
├── .pre-commit-config.yaml    # gitleaks hook
├── .github/
│   ├── CODEOWNERS             # owner review required on every file
│   └── workflows/gitleaks.yml # gitleaks on every push + PRs to main
├── .gitignore                 # .env, .idea/
└── collections/
    └── apicollection/         # Kotlin Spring Boot greeting API
        ├── opencollection.yml # collection root, default env = local
        ├── .env.example       # required secret keys, empty values (committed)
        ├── .env               # real secrets (gitignored)
        ├── environments/local.yml
        └── greetings/         # get-greeting, create-greeting, create-greeting-without-key
```

Conventions: one collection per project under `collections/<project>/`, registered in `workspace.yml`; each collection has its own `environments/`, `.env.example`, `.env`; folders mirror API resources; file names are kebab-case actions.

### 3. Secrets via `.env`
- Real values in `collections/<project>/.env` (gitignored); `.env.example` committed with the same keys.
- Environments reference secrets as `{{process.env.KEY}}`; requests only use variables (`{{apiKey}}`).
- Verified end to end with `API_KEY` (`X-API-KEY` on `POST /api/greetings`): GET 200, POST without or with a wrong key 401, POST with the right key 200.
- Run the source app with the same key:
  ```bash
  set -a; source ~/Workspace/Practice/Api-Collection/collections/apicollection/.env; set +a
  ./gradlew bootRun
  ```

### 4. Decisions
- [x] **Request tests** removed from all requests. The Collection Runner still sends every request.
- [x] **Production safety:** never commit a production environment or production keys. Pre-request guard script blocking writes on production is postponed.

### 5. Pre-commit secret scanning (layer 1)
- `.pre-commit-config.yaml` runs the `gitleaks` hook (`rev: v8.30.1`). pre-commit builds gitleaks itself, so everyone runs the same version with no Go or gitleaks install.
- Team setup commands: see "Team member setup" above (also in the README).
- Verified: a staged GitHub token is blocked (`github-pat`).
- Known gaps:
  - Only **staged** content is scanned. Re-run `git add` after editing.
  - Literal custom keys in Bruno `value:` lines and short passwords are **not** detected. The fix is postponed: see FUTURE_IMPROVEMENTS.md #1.
  - `git commit --no-verify` skips the hook.

### 6. GitHub-side protection (layer 2)

**Repo visibility: public.** This makes rulesets, secret scanning and push protection free. Before switching, a full history scan (`gitleaks git -v`, 4 commits) found no leaks. Commit author emails are now public.

**CODEOWNERS** (`.github/CODEOWNERS`, commit `974ed2f`):
```
* @EbiSharifiS
```
Every file is owned by the repo owner, so only the owner's approval satisfies the code-owner rule. Validated: `GET /repos/EbiSharifiS/Api-Collection/codeowners/errors` returns no errors.

**Branch ruleset "require a pull request"** (Settings → Rules → Rulesets, id `28018995`):

| Setting | Value |
|---|---|
| Enforcement | Active |
| Target | Default branch (`main`) |
| Bypass list | Repository admin. Use **For pull requests only** so the owner also goes through PRs; with **Always** the owner can push straight to `main` (this is how `974ed2f` got in). |
| Restrict deletions | On |
| Block force pushes | On |
| Require a pull request | On: 1 approval, dismiss stale approvals, require review from Code Owners |
| Require approval of most recent push | Off |
| Require status checks | **Pending:** add `gitleaks` (the workflow is on `main` now) |

Check the active rules without logging in:
```bash
curl -s https://api.github.com/repos/EbiSharifiS/Api-Collection/rules/branches/main
```

**gitleaks workflow** (`.github/workflows/gitleaks.yml`, merged via PR #1, merge commit `b6be35d`). Runs verified green on `push` and `pull_request` for `ci/gitleaks`, `main` and `test-1`:
- `push` with no filter: scans every push to every branch and tag. This is an **alarm**: it runs after the push is accepted, so a hit means **rotate the secret**.
- `pull_request` to `main`: the **gate**. The `gitleaks` check will be required by the ruleset, so a PR with a leak can't merge. It also covers PRs from forks.
- `workflow_dispatch`: manual run from the Actions tab.
- Uses `gitleaks/gitleaks-action@v3`, `fetch-depth: 0` (every commit is scanned), `GITLEAKS_VERSION: 8.30.1` (same as pre-commit), and `GITLEAKS_NOTIFY_USER_LIST: "@EbiSharifiS"` (the owner is emailed on any leak). No license is needed for a personal-account repo.

**Push protection: pending.** Settings → Advanced Security → enable Secret Protection, then Push protection. It's the only layer that **rejects** a push containing a secret (known token formats only). Pushers can bypass it with a reason, which raises an alert in the Security tab. Locking bypass down requires an organization.

### Who stops what

| Situation | Pre-commit | Push protection | Workflow (push) | Required check (PR) |
|---|---|---|---|---|
| Hook installed | Blocks commit | — | — | — |
| Hook skipped or missing | — | Rejects push (known formats) | Alerts owner | Blocks merge |
| Push protection bypassed | — | Allowed, alert raised | Alerts owner | Blocks merge |
| Custom key format (`X-API-KEY` value) | Not caught | Not caught | Not caught | Not caught (FUTURE_IMPROVEMENTS #1) |

## Next steps

1. **Require the check:** ruleset → Require status checks to pass → add `gitleaks`.
2. **Enable push protection** (see above).
3. **Set the admin bypass** to "For pull requests only", if not done yet.
4. **Test end to end:** push a branch with a fake token (random-looking, e.g. `ghp_` + 36 random characters; an alphabetical one is ignored by gitleaks' stopword list). Expect the run to fail, an email to arrive, and the PR to be blocked. Then close the PR and delete the branch.
5. **Commit SETUP.md** (not yet in the repo).
6. **CI collection runs without committed secrets:** store `API_KEY` as a GitHub Actions secret, start the API in the job (or use a test environment, never production), then:
   ```bash
   cd collections/apicollection
   bru run --env local --env-var apiKey=$API_KEY
   ```

## If a secret leaks

Rotate it immediately. Deleting the file or rewriting history is not enough, because forks, clones and caches may already have it.
