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

## Adding a collection

1. Create `collections/<project-name>/` with `opencollection.yml`, `environments/` and `.env.example`.
2. Add it to `workspace.yml` under `collections`.
3. Add a row to the table above.
