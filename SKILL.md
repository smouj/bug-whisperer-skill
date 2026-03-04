---
name: bug-whisperer
version: 2.1.0
description: Analyzes error patterns, commit history, and code quality metrics to predict potential bug hotspots and suggest preventive fixes before they manifest in production.
author: OpenClaw Collective
tags:
  - debugging
  - prediction
  - prevention
  - static-analysis
  - regression
dependencies:
  - git>=2.30
  - python>=3.9
  - nodejs>=16.0
  - rubocop:optional
  - eslint:optional
  - bandit:optional
  - radon:optional
  - semgrep:optional
required_env:
  - GIT_REPO_PATH
  - LOG_LEVEL (default: INFO)
  - PREDICTION_THRESHOLD (default: 0.7)
  - EXCLUDE_PATTERNS (default: "vendor/,node_modules/,*.min.js")
capabilities:
  - pattern_analysis
  - regression_prediction
  - anti_pattern_detection
  - heatmap_generation
  - fix_suggestion
---

# Bug Whisperer Skill

## Purpose

Bug Whisperer serves three core real-world use cases:

1. **Pre-deployment risk assessment**: Before merging a feature branch, run `bug-whisperer analyze` to identify high-risk files with high cyclomatic complexity, duplicated code, or historical regression patterns. CLI usage: `bug-whisperer --path ./src --output risk-report.json`

2. **Regression hotspot prediction**: Analyze Git history to find which files change together frequently and cause breakage. Example: `bug-whisperer history --days 90 --threshold 3` shows files that, when modified simultaneously, have caused bugs 3+ times in the last 90 days.

3. **Code health trend tracking**: Generate weekly heatmaps of code quality metrics (complexity, test coverage, churn) to guide refactoring efforts. Usage: `bug-whisperer trends --weeks 12 --output heatmap.html`

The skill runs without modifying any code, providing actionable reports with specific line numbers and suggested fixes.

## Scope

Bug Whisperer provides these concrete CLI commands (all under `bug-whisperer <command>`):

### `analyze [--path <dir>] [--output <file>] [--min-risk <float>]`
Performs static analysis on specified directory. Defaults to current working directory. Generates JSON report by default, HTML if `--output` ends with `.html`. `--min-risk` filters reports to only show predictions with confidence ≥ threshold (default 0.5).

### `history [--days <int>] [--threshold <int>] [--max-commits <int>]`
Analyzes Git commit history to find regression patterns. `--days` looks back this many days (default 30). `--threshold` sets minimum recurrence count to report (default 2). `--max-commits` limits analysis to most recent N commits for performance.

### `trends [--weeks <int>] [--output <file>] [--metrics <list>]`
Generates code health trends over time. `--weeks` analyzes this many weeks of history (default 8). `--metrics` accepts comma-separated list: `complexity,coverage,churn` (default: all). `--output` creates HTML heatmap or CSV.

### `predict [--files <paths>] [--output <file>] [--format <json|md>]`
Given a list of files (or `--files .` for all), predicts likelihood of bugs based on current state and historical patterns. `--format` controls output format (default: md for terminal, json for scripting).

### `suggest [--file <path> [--line <int>]]`
For a specific file (and optionally line), generates targeted refactoring or testing suggestions. Without `--line`, analyzes entire file.

### `scan [--linter <tool>] [--rules <file>] [--fix]`
Runs integrated linters (eslint, rubocop, semgrep) with bug-pattern-specific rule sets. `--fix` attempts safe automated fixes. `--rules` points to custom rule configuration.

## Work Process

When invoked, Bug Whisperer executes this exact sequence:

1. **Environment validation**:
   ```bash
   validate_git_repo() {
     [ -d "$GIT_REPO_PATH/.git" ] || die "Missing .git directory";
     git rev-parse --is-inside-work-tree >/dev/null 2>&1 || die "Not a git repo";
   }
   ```

2. **File discovery** (respecting `.gitignore` and `EXCLUDE_PATTERNS`):
   - Run: `git ls-files '*.py' '*.js' '*.rb' '*.go' '*.java'`
   - Filter out patterns from `EXCLUDE_PATTERNS` env var
   - Build file list for analysis

3. **Static analysis pipeline** (parallelized by file extension):
   ```
   For each file:
     - Compute cyclomatic complexity (radon for Python, eslint complexity for JS)
     - Detect code duplication (simian or custom token-based detector)
     - Count lines of code, comment ratio
     - Identify TODO/FIXME comments with urgency keywords
   Output: metrics.json
   ```

4. **History correlation**:
   ```bash
   git log --since="$START_DATE" --format="%H %s" --name-only | \
   awk '/^[0-9a-f]/ {commit=$1} /\.(py|js|rb|go|java)$/ {files[$2]++}'
   ```
   Build co-change matrix: files modified together frequently. Then:
   - Filter pairs with joint occurrence ≥ `--threshold`
   - Cross-reference with bug fixes: `git log --grep="bug\|fix\|regression"` to find which co-changes introduced bugs

5. **Prediction model** (simple Bayesian scoring):
   ```
   BugScore(file) = 
     0.4 * normalized_complexity +
     0.3 * normalized_churn +
     0.2 * historical_breakage_rate +
     0.1 * test_coverage_gap
   ```
   Files with score > `PREDICTION_THRESHOLD` (default 0.7) are flagged as high-risk.

6. **Report generation**:
   - JSON format: array of `{file, line, risk_score, factors, suggested_fixes}`
   - Markdown: human-readable table with color codes (red >0.8, orange 0.6-0.8, yellow 0.4-0.6)
   - HTML: interactive heatmap with file tree

7. **Fix suggestion engine**:
   For each high-risk item, load language-specific templates:
   ```
   if complexity > 10: suggest "Extract Method" refactoring
   if duplicate code found: suggest "Extract Class/Module"
   if high churn + no tests: suggest "Add unit tests for X function"
   ```
   Provide exact code snippets with before/after.

## Golden Rules

- **Never modify source files**: Bug Whisperer is read-only. All suggestions are advisory.
- **Always respect `.gitignore`**: Never analyze ignored files, even if they exist in working tree.
- **Historical accuracy**: Only consider commits that are merged to main/master branch for history analysis. Ignore feature branches unless explicitly requested with `--include-branches`.
- **Threshold transparency**: Document exact `PREDICTION_THRESHOLD` used in report. Never silently change thresholds.
- **No false positives for tests**: Do not flag test files as bug sources; instead, flag missing tests for risky code.
- **Performance guardrail**: If repository has >10k files, default to sampling (top 1000 by churn) unless `--full-scan` is specified.
- **CI/CD friendliness**: Exit code 0 if no high-risk bugs found, 1 if any found, 2 on error. No interactive prompts.
- **Report reproducibility**: Include git commit hash, timestamp, and tool versions in every report.

## Examples

### Example 1: Pre-commit check
User runs:
```bash
bug-whisperer analyze --path ./src --min-risk 0.6 --output risk.json
```
Skill outputs:
```json
{
  "meta": {
    "git_commit": "a1b2c3d",
    "timestamp": "2026-03-04T10:30:00Z",
    "threshold": 0.6,
    "files_scanned": 142
  },
  "high_risk": [
    {
      "file": "src/auth/login.py",
      "lines": [45, 112],
      "risk_score": 0.82,
      "factors": {
        "complexity": 15,
        "historical_breakage": 4,
        "test_coverage": 0.35
      },
      "suggestions": [
        "Extract method 'validate_token' starting at line 45",
        "Add unit tests covering edge cases in login flow"
      ]
    }
  ]
}
```

### Example 2: Regression pattern discovery
User runs:
```bash
bug-whisperer history --days 60 --threshold 3
```
Skill outputs:
```
Regression Patterns (co-changed files causing bugs ≥3 times):
1. src/models/user.js + src/api/auth.js
   Occurrences: 5
   Sample bugs: "Login fails after password reset", "User session lost"
2. src/components/Form.js + src/utils/validation.js
   Occurrences: 3
   Sample bugs: "Validation bypassed on Safari", "Form submits twice"
Run: bug-whisperer suggest --file src/models/user.js for detailed fixes.
```

### Example 3: CI/CD integration (GitHub Actions)
```yaml
- name: Predict Bugs
  run: |
    bug-whisperer predict --files . --format json > predictions.json
    if jq -e '.high_risk | length > 0' predictions.json > /dev/null; then
      echo "::error::High-risk files detected. See predictions.json"
      exit 1
    fi
```

## Rollback Commands

Bug Whisperer is read-only and produces only reports. No system state is modified. To "rollback":

1. **Discard generated report files**:
   ```bash
   rm -f risk-report.json heatmap.html predictions.md
   ```

2. **Undo any accidental changes** (should never happen, but if `--fix` was mistakenly used with a linter):
   ```bash
   git reset --hard HEAD
   git clean -fd
   ```

3. **Revert environment changes** (if user manually set `EXCLUDE_PATTERNS` or `PREDICTION_THRESHOLD` in shell):
   ```bash
   unset EXCLUDE_PATTERNS PREDICTION_THRESHOLD
   # Or restore from backup: source ~/.openclaw/env-backup.sh
   ```

4. **Remove temporary analysis cache** (stored in `.bugwhisperer_cache/`):
   ```bash
   rm -rf .bugwhisperer_cache
   ```

Since the skill only reads code and produces files, the rollback is simply deleting its outputs.
```