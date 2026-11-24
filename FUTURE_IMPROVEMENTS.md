# Future Improvements

Ideas that are deliberately postponed. Each entry says what it is, why we'd want it, and the known trade-offs.

## 1. Custom gitleaks rule for literal values in Bruno YAML

### Problem

The default gitleaks rules detect known token formats (GitHub, AWS, JWT, ...) anywhere, but they miss a custom API key pasted directly into a Bruno request. The built-in generic rule needs a keyword such as `key` on the **same line** as the value, and Bruno YAML puts the name and the value on separate lines:

```yaml
- name: X-API-KEY
  value: "k9F2xQ7vLm3PzR8tWc1N"   # not detected by the default rules
```

In addition, the pre-commit scan only sees the changed lines (`git diff -U0 --staged`). When only the `value:` line is edited, the `name:` line is not visible, so the rule has to judge the `value:` line on its own.

### Proposal

Add a `.gitleaks.toml` at the repo root (gitleaks picks it up automatically, so `.pre-commit-config.yaml` does not change). In any `.yml`/`.yaml` file under `collections/`, flag a `value:` line when all of these hold:

1. it is not a `{{...}}` reference (does not start with `{`)
2. it is at least 12 characters long
3. it looks random (Shannon entropy ≥ 3.5)
4. it is not a URL (`http://` or `https://`)

```toml
title = "Api-Collection gitleaks config"

[extend]
useDefault = true          # keep all built-in rules, add ours on top

[[rules]]
id = "bruno-literal-value"
description = "Literal secret-looking value in a Bruno collection file; use a {{variable}} instead"
path = '''^collections/.*\.ya?ml$'''
regex = '''value:\s*["']?([^"'\s{][^"'\s]{11,})'''
secretGroup = 1
entropy = 3.5
keywords = ["value:"]

    [[rules.allowlists]]
    description = "URLs such as baseUrl are not secrets"
    regexes = ['''^https?://''']
```

### Expected results on the current collection

| Line | Flagged | Reason |
|---|---|---|
| `value: http://localhost:8080` (`environments/local.yml`) | No | URL allowlist |
| `value: "{{process.env.API_KEY}}"` (`environments/local.yml`) | No | `{{...}}` reference |
| `value: "{{apiKey}}"` (`greetings/create-greeting.yml`) | No | `{{...}}` reference |
| `value: "k9F2xQ7vLm3PzR8tWc1N"` | Yes | literal, long, random |
| `value: application/json` | Expected no | low entropy; confirm when testing |

### Limits

- Short or simple passwords (`hunter2`) stay undetected. Lowering the length or entropy threshold would flag ordinary headers like `Accept: text/plain`.
- Secrets inside a request body (`data:` block) are not on a `value:` line. Known token formats there are still caught by the default rules, custom keys are not.
- False positives can be silenced with `# gitleaks:allow` at the end of the line, or by extending the allowlist.

### Steps to implement

1. Add `.gitleaks.toml` as above.
2. Stage a throwaway file with the cases from the table and run `pre-commit run gitleaks` to confirm what is caught.
3. Run `gitleaks dir collections` to confirm the existing collection has no false positives.
4. Remove the test file, then commit `.gitleaks.toml`.
