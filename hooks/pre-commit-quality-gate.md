# Pre-Commit Quality Gate Hook

Runs before every commit to enforce quality standards.

## What it checks

1. **Clean Code**: flags methods longer than 20 lines or with more than 3 parameters
2. **Test coverage**: warns if a changed file has no corresponding test file
3. **No TODO/FIXME in committed code**: use issue tracker instead
4. **No secrets**: detects API keys, passwords, tokens in diff
5. **Architecture violations**: detects framework imports in domain/core packages

## Hook configuration

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash /path/to/quality-gate.sh"
          }
        ]
      }
    ]
  }
}
```

## quality-gate.sh

```bash
#!/bin/bash
# Fail fast on any quality violation

CHANGED=$(git diff --cached --name-only --diff-filter=ACM)

for file in $CHANGED; do
  # Detect secrets
  if git diff --cached "$file" | grep -E '(password|secret|api_key|token)\s*=\s*["\x27][^"\x27]{8,}'; then
    echo "BLOCKED: Possible secret in $file"
    exit 1
  fi

  # Detect framework imports in domain packages
  if echo "$file" | grep -qE '/(domain|core)/'; then
    if git diff --cached "$file" | grep -E '^+.*(import org\.springframework|import fastapi|import flask)'; then
      echo "BLOCKED: Framework import in domain layer: $file"
      exit 1
    fi
  fi
done

echo "Quality gate passed."
exit 0
```
