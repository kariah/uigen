Audit the project's dependencies for security vulnerabilities, outdated packages, and licensing issues. Run the following checks and summarize findings:

## 1. Security Vulnerabilities

Run a security audit:
```bash
npm audit
```

Report:
- Total vulnerabilities by severity (critical, high, moderate, low)
- Any critical or high severity issues with package name, vulnerability description, and whether a fix is available

## 2. Outdated Packages

Check for outdated dependencies:
```bash
npm outdated
```

Categorize results:
- **Major updates** (breaking changes possible) — list package, current, and latest versions
- **Minor/patch updates** (safe to update) — summarize count, list any that are significantly behind

## 3. Unused Dependencies

Identify packages that may no longer be used:
```bash
npx depcheck --ignores="@types/*,eslint-*,prettier" 2>/dev/null || echo "depcheck not available, skipping"
```

List any unused dependencies found in `package.json`.

## 4. Duplicate Packages

Check for duplicate versions of the same package in the dependency tree:
```bash
npm ls --all 2>/dev/null | grep -E "WARN|deduped" | head -30 || true
```

## 5. License Check

Review licenses of direct dependencies by reading `package.json` and checking for any non-permissive licenses (GPL, AGPL, LGPL, CC-BY-SA) that could affect distribution:
```bash
cat package.json | grep -A 200 '"dependencies"'
```

Cross-reference notable packages against known license types.

## Summary Report

After running all checks, provide a structured summary:

### Security
- Vulnerability counts by severity
- Top issues requiring immediate action

### Maintenance
- Count of outdated packages
- Recommended updates (prioritize packages with security fixes)

### Risks
- Any licensing concerns
- Unused dependencies that could be removed to reduce attack surface

### Recommended Actions
Ordered list of actions from highest to lowest priority.
