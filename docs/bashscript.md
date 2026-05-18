# Bash Script Standards (RFC-PLAT-001)

Every Bash script must follow this template:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

SCRIPT_NAME="$(basename "$0")"

log() {
  local level="$1"; shift
  echo "$(date -u +'%Y-%m-%dT%H:%M:%SZ') [$level] [$SCRIPT_NAME] $*"
}

error_exit() { log ERROR "$1"; exit "${2:-1}"; }

trap 'error_exit "Failure at line $LINENO"' ERR
trap 'error_exit "Interrupted"' INT TERM

main() {
  log INFO "Starting execution"
  # logic here
  log INFO "Completed successfully"
}

main "$@"
```

Key rules:
- `set -euo pipefail` is mandatory — no exceptions
- Structured log format: `YYYY-MM-DDTHH:MM:SSZ [LEVEL] [SCRIPT] message`
- Exit codes: 0=success, 1=generic failure, 2=validation error, 3=dependency failure, 4=timeout, 5=permission denied
- `eval`, hardcoded secrets, and `|| true` silent failures are forbidden
- All inputs validated before use: `: "${ENV:?ENV is required}"`
- Scripts must be idempotent and support `--dry-run` for CI use
- Debug mode: `[[ "$DEBUG" == "true" ]] && set -x`
- Lint with `shellcheck` and format with `shfmt` before committing
