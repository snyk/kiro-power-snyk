# Snyk Fix (All-in-One)

## Overview
Complete security remediation workflow in a single command. Scans for vulnerabilities, fixes them, and validates the fixes.

**Workflow**: Parse → Scan → Analyze → Fix → Validate → Summary

## Example Usage

| Command | Behavior |
|---------|----------|
| `/snyk-fix` | Auto-detect scan type, fix highest priority issue (all instances) |
| `/snyk-fix code` | SAST scan only, fix highest priority code issue (all instances in file) |
| `/snyk-fix sca` | SCA scan only, fix highest priority dependency issue |
| `/snyk-fix SNYK-JS-LODASH-1018905` | Fix specific Snyk issue by ID |
| `/snyk-fix CVE-2021-44228` | Find and fix specific CVE |
| `/snyk-fix lodash` | Fix highest priority issue in lodash package |
| `/snyk-fix server.ts` | Code scan on file, fix highest priority issue (all instances) |
| `/snyk-fix sca express` | Fix highest priority issue in express package |
| `/snyk-fix XSS` | Fix all XSS vulnerabilities in highest priority file |
| `/snyk-fix path traversal` | Fix all path traversal vulnerabilities |
| `/snyk-fix javascript/PT@files.ts:45, javascript/PT@files.ts:112` | Fix multiple specific code vulnerabilities |
| `/snyk-fix sca:lodash, sca:express` | Fix multiple dependency vulnerabilities |
| `/snyk-fix javascript/XSS@app.ts:20, sca:lodash` | Fix mixed code and dependency vulnerabilities |

---

## Phase 1: Input Parsing

Parse user input to extract:
- **scan_type**: Explicit (`code`, `sca`, `both`) or infer from context
- **target_vulnerabilities**: List of specific issue IDs, CVEs, package names, file references, or vulnerability types (can be single or multiple)
- **target_path**: File or directory to focus on (defaults to project root)

### Multi-Vulnerability Input Formats

The command accepts multiple vulnerabilities separated by commas:

| Format | Example | Meaning |
|--------|---------|---------|
| Code vuln with location | `javascript/PT@files.ts:45` | Code vulnerability ID at specific file:line |
| Code vuln ID only | `javascript/PT` | All instances of this vulnerability type |
| SCA package | `sca:lodash` | Dependency vulnerability in lodash |
| Snyk ID | `SNYK-JS-LODASH-1018905` | Specific Snyk issue |
| CVE | `CVE-2021-44228` | Specific CVE |
| Mixed | `javascript/XSS@app.ts:20, sca:express, CVE-2021-44228` | Multiple different types |

**Parsing rules for comma-separated input:**
1. Split input by `,` and trim whitespace
2. For each item, detect its type:
   - Starts with `sca:` → SCA vulnerability (package name follows)
   - Contains `@file:line` → Code vulnerability with specific location
   - Starts with `SNYK-` → Snyk issue ID (could be code or SCA)
   - Contains `CVE-` → CVE (could be code or SCA)
   - Contains `/` (e.g., `javascript/PT`) → Code vulnerability type
   - Otherwise → package name (SCA) or vulnerability type description

### Scan Type Detection Rules (in priority order)

1. **Explicit code**: User says "code", "sast", "static" → Code scan
2. **Explicit sca**: User says "sca", "dependency", "package", "npm", "pip", "maven" → SCA scan
3. **Multi-vulnerability input detected**: Contains commas or multiple `sca:` / code vuln patterns → Parse all, determine scan types needed
4. **Vulnerability ID provided**: 
   - Starts with `SNYK-` → run both scans to locate it
   - Contains `CVE-` → run both scans to find it
5. **Vulnerability type provided**: User mentions type like "XSS", "SQL injection", "path traversal" → Code scan
6. **File reference**: User mentions `.ts`, `.js`, `.py`, etc. file → Code scan on that file
7. **Package reference**: User mentions known package name (e.g., "lodash", "express") → SCA scan
8. **Default (no hints)**: Run BOTH scans, select highest priority issue

---

## Phase 2: Discovery

**Goal**: Run scan(s) and identify ALL vulnerabilities to fix, including ALL instances of each type

### Step 2.1: Run Security Scan(s)

Based on scan type detection:
- **Code only**: Run `snyk_code_scan` with `path` set to project root or specific file
- **SCA only**: Run `snyk_sca_scan` with `path` set to project root
- **Both**: Run both scans in parallel

### Step 2.2: Build Target Vulnerability List

**If user specified multiple vulnerabilities (comma-separated):**
- Parse each item from the input
- For each item, search scan results for matching issue(s)
- Build a combined list of all targeted vulnerabilities
- Group by type: Code vulnerabilities vs SCA vulnerabilities
- If ANY specified vulnerability is NOT found: Report which ones are missing but continue with found ones

**If user specified a single vulnerability:**
- Search scan results for matching issue (by ID, CVE, package name, vulnerability type, or description)
- If NOT found: Report "Vulnerability not found in scan results" and STOP
- If found: Add to target list

**If user did NOT specify any vulnerability:**
- From ALL scan results, select the highest priority vulnerability TYPE using this priority:
  1. Critical severity with known exploit
  2. Critical severity
  3. High severity with known exploit  
  4. High severity
  5. Medium severity
  6. Low severity
- Within same priority: prefer issues with available fixes/upgrades

### Step 2.3: Group All Instances (Code Vulnerabilities Only)

**⚠️ IMPORTANT for Code vulnerabilities**: For each targeted code vulnerability type, find ALL instances in the codebase:

- Same vulnerability ID (e.g., `javascript/PT`, `javascript/XSS`, `python/SQLi`)
- If user specified file:line, still check for other instances of same type in that file
- Group instances by file for efficient fixing

**Example**: If user requests `javascript/PT@files.ts:45, javascript/XSS@app.ts:20` and scan finds:
```
High    Path Traversal    src/api/files.ts:45    javascript/PT
High    Path Traversal    src/api/files.ts:112   javascript/PT  
High    XSS               src/app.ts:20          javascript/XSS
High    XSS               src/app.ts:55          javascript/XSS
```

Target ALL 4 instances (both PT instances in files.ts, both XSS instances in app.ts).

### Step 2.4: Document All Targets

```
## Target Vulnerabilities

### Code Vulnerabilities ([count] types, [total instances] instances)

| # | Type | Severity | File | Lines | Instance Count |
|---|------|----------|------|-------|----------------|
| 1 | [javascript/PT] | [High] | [files.ts] | [45, 112] | 2 |
| 2 | [javascript/XSS] | [High] | [app.ts] | [20, 55] | 2 |

### SCA Vulnerabilities ([count] packages)

| # | Package | Current Version | Severity | Fix Version |
|---|---------|-----------------|----------|-------------|
| 1 | [lodash] | [4.17.15] | [Critical] | [4.17.21] |
| 2 | [express] | [4.17.0] | [High] | [4.18.2] |

**Total issues to fix**: [count]
```

---

## Phase 3: Remediation (Code Vulnerabilities)

**Skip to Phase 4 if there are no code vulnerabilities to fix.**

### Step 3.1: Understand All Code Vulnerabilities

For EACH code vulnerability type in the target list:
- Read the affected file(s) and ALL vulnerable locations
- Identify the vulnerability type:
  - **Injection** (SQL, Command, LDAP, etc.)
  - **XSS** (Cross-Site Scripting)
  - **Path Traversal**
  - **Sensitive Data Exposure**
  - **Insecure Deserialization**
  - **Security Misconfiguration**
  - **Cryptographic Issues**
  - Other (check Snyk description)
- Review Snyk's remediation guidance if provided
- Look for patterns across instances (often the same fix approach applies)

### Step 3.2: Plan All Fixes

Before implementing, document the approach for EACH vulnerability type:
```
## Fix Plan

### Vulnerability Type 1: [type]
- **Root Cause**: [why the code is vulnerable]
- **Fix Approach**: [what will be changed]
- **Security Mechanism**: [what protection is being added]
- **Files/Instances Affected**: [count] locations across [files]

### Vulnerability Type 2: [type]
- **Root Cause**: [why the code is vulnerable]
- **Fix Approach**: [what will be changed]
- **Security Mechanism**: [what protection is being added]
- **Files/Instances Affected**: [count] locations across [files]

(repeat for each type)
```

Common fix patterns:
| Vulnerability | Fix Pattern |
|---------------|-------------|
| SQL Injection | Parameterized queries / prepared statements |
| Command Injection | Input validation + shell escaping or avoid shell |
| Path Traversal | Canonicalize path + validate against allowed base |
| XSS | Output encoding / sanitization appropriate to context |
| Sensitive Data Exposure | Remove/mask data, use secure headers |
| Hardcoded Secrets | Move to environment variables / secrets manager |

### Step 3.3: Apply Fixes to ALL Instances

**Strategy for multiple vulnerability types:**
1. Group fixes by file to minimize file read/write operations
2. Within each file, fix from bottom to top (highest line number first)
3. Complete all fixes in one file before moving to the next

**For each vulnerability instance:**
- Apply consistent fix pattern across all instances of same type
- Make the minimal code change needed at each location
- Prefer standard library/framework security features over custom solutions
- Consider creating a shared helper function if:
  - 3+ instances exist with identical fix pattern across files
  - The helper improves readability without over-engineering
- Add comments explaining security-relevant changes if non-obvious
- Do NOT refactor unrelated code
- Do NOT change business logic

**Progress tracking:**
```
## Fix Progress
- [x] files.ts: javascript/PT (2 instances)
- [x] app.ts: javascript/XSS (2 instances)
- [ ] utils.ts: javascript/SQLi (1 instance)
```

**Continue to Phase 4 if there are also SCA vulnerabilities, otherwise skip to Phase 5.**

---

## Phase 4: Remediation (SCA Vulnerabilities)

**Skip to Phase 5 if there are no SCA vulnerabilities to fix.**

### Step 4.1: Analyze All Dependency Paths

For EACH SCA vulnerability in the target list:
- Document the full dependency path (direct → transitive → vulnerable)
- Identify manifest files to modify (`package.json`, `requirements.txt`, etc.)
- Determine if vulnerable package is direct or transitive

**For transitive dependencies:**
- Identify which direct dependency pulls in the vulnerable transitive
- Check if upgrading the direct dependency will pull in the fixed transitive
- If app directly imports the transitive: note this for breaking change analysis

**Consolidate upgrades:**
- If multiple vulnerabilities are in the same package, one upgrade fixes all
- If upgrading package A also fixes transitive B, note this to avoid duplicate work

### Step 4.2: Check for Breaking Changes

For EACH package being upgraded, search codebase for potential impact:
```bash
# Search for imports of the package
grep -r "from 'package'" --include="*.ts" --include="*.js"
grep -r "require('package')" --include="*.ts" --include="*.js"
```

If complex breaking changes detected:
- Add TODO comments with migration notes
- Note in summary that manual review is needed

### Step 4.3: Apply All Minimal Upgrades

- Edit ALL necessary dependencies in the manifest(s)
- Use the LOWEST version that fixes each vulnerability
- Preserve file formatting and comments
- Make all manifest changes before running install

**Example (package.json) with multiple packages:**
```json
// Before
"lodash": "^4.17.15",
"express": "^4.17.0"

// After - minimal fixes for both
"lodash": "^4.17.21",
"express": "^4.18.2"
```

### Step 4.4: Regenerate Lockfile (Once for All Changes)

Run the appropriate install command ONCE after all manifest changes:

| Package Manager | Command |
|-----------------|---------|
| npm (any upgrade) | `npm install` |
| yarn | `yarn install` |
| pip | `pip install -r requirements.txt` |
| maven | `mvn dependency:resolve` |

**IMPORTANT**: Use `required_permissions: ["all"]` for package manager commands.

**If installation fails:**
- If sandbox/permission issue: retry with elevated permissions
- If dependency conflict: identify which package(s) are conflicting
- Try alternative versions or note specific packages as unfixable
- Keep successful upgrades, revert only failed ones
- Document which fixes succeeded and which failed

---

## Phase 5: Validation

### Step 5.1: Re-run Security Scans
- Run `snyk_code_scan` if code vulnerabilities were fixed
- Run `snyk_sca_scan` if SCA vulnerabilities were fixed
- Verify ALL targeted vulnerability instances are NO LONGER reported

**For Code vulnerabilities - If any instances still present:**
- Track which specific instances remain unfixed
- Review the fix attempt for each unfixed instance
- Try alternative approach
- Maximum 3 total attempts per instance, then report partial success/failure

**For SCA vulnerabilities - If any vulnerabilities still present:**
- Track which packages still have issues
- Check if lockfile was properly updated for each
- Try explicit version install for failed packages: `npm install <pkg>@<exact_version>`
- Maximum 3 attempts per package, then report partial success/failure

**If NEW vulnerabilities introduced:**

*For Code:*
- Code fixes must be clean — no new vulnerabilities allowed
- Attempt to fix any new issues introduced by your fixes
- Iterate until clean (max 3 total attempts)
- If unable to produce clean fix: Revert problematic fix and report partial failure

*For SCA:*
- Check severity trade-off for each new vulnerability:
  - **New severity LOWER than fixed**: Accept (net security improvement)
  - **New severity EQUAL OR HIGHER**: Try higher version (up to 3 iterations)
  - If no clean version exists for a package: Revert that package and report as unfixable

### Step 5.1a: Identify Additional Issues Fixed
Package upgrades and code fixes may resolve more than targeted issues:
- Compare pre-fix and post-fix scan results
- Identify ALL vulnerabilities resolved (may be more than originally targeted)
- Record each: ID, severity, title

### Step 5.2: Run Tests
- Execute project tests (`npm test`, `pytest`, etc.)
- If tests fail due to fixes:
  - Identify which fix(es) caused the failure
  - Prefer adjusting the fix over changing tests
  - Only modify tests if the fix legitimately changes expected behavior
  - Apply mechanical fixes only (renamed imports, etc.)
  - Maximum 2 attempts to resolve test failures

### Step 5.3: Run Linting
- Run project linter if configured
- Fix any formatting issues introduced

---

## Phase 6: Summary

### Step 6.1: Display Remediation Summary

```
## Remediation Summary

### Code Vulnerabilities Fixed

| Type | Severity | File | Instances | Status |
|------|----------|------|-----------|--------|
| [javascript/PT] | High | files.ts | 2 | ✅ Fixed |
| [javascript/XSS] | High | app.ts | 2 | ✅ Fixed |
| [python/SQLi] | Critical | db.py | 1 | ❌ Failed |

### SCA Vulnerabilities Fixed

| Package | Version Change | Vulnerabilities Fixed | Status |
|---------|----------------|----------------------|--------|
| lodash | 4.17.15 → 4.17.21 | 3 (incl. bonus fixes) | ✅ Fixed |
| express | 4.17.0 → 4.18.2 | 2 | ✅ Fixed |
| axios | 0.21.0 → 0.21.4 | 1 | ❌ Conflict |

### Overall Results

| Metric | Count |
|--------|-------|
| **Targeted Issues** | [count] |
| **Successfully Fixed** | [count] |
| **Failed/Partial** | [count] |
| **Bonus Fixes** | [count] (issues fixed as side effect) |
| **Total Issues Resolved** | [count] |

### What Was Fixed
[Brief summary of all fixes applied. Group by vulnerability type. Keep concise.]

### Validation

| Check | Result |
|-------|--------|
| Snyk Code Re-scan | ✅ [X] code vulns resolved / ❌ [Y] still present |
| Snyk SCA Re-scan | ✅ [X] SCA vulns resolved / ❌ [Y] still present |
| TypeScript/Build | ✅ Pass / ❌ Fail |
| Linting | ✅ Pass / ❌ Fail |
| Tests | ✅ Pass / ⚠️ Skipped (reason) / ❌ Fail |

### Failed Fixes (if any)
| Issue | Reason | Recommendation |
|-------|--------|----------------|
| [issue] | [why it failed] | [manual steps or alternative] |
```

**Rules for this summary:**
- Do NOT include code snippets (before/after)
- Do NOT list remaining issues NOT targeted by this run
- Group similar fixes together
- Clearly distinguish targeted vs bonus fixes

### Step 6.2: Send Feedback to Snyk
After fixes complete, report the remediation:
```
snyk_send_feedback with:
- fixedExistingIssuesCount: [total issues fixed, including bonus fixes]
- preventedIssuesCount: 0
- path: [absolute project path]
```

---

## Error Handling

### Authentication Errors
- Run `snyk_auth` and retry once
- If still failing: STOP and ask user to authenticate manually

### Scan Timeout/Failure
- Retry once
- If still failing: STOP and report the error

### Vulnerability Not Found
- If user specified vulnerabilities that don't appear in scan results:
  - Report which specific ones were NOT found
  - Continue fixing the ones that WERE found
  - If NONE were found: Report clearly and STOP

### Unfixable Code Vulnerability
If a vulnerability cannot be fixed automatically:
1. Document why it cannot be fixed (complex refactoring needed, unclear fix, etc.)
2. Add TODO comment in the affected file with context
3. Continue with other vulnerabilities in the list
4. Report unfixable items in final summary with manual remediation suggestions
5. Do NOT leave partial/broken fixes for that specific item

### SCA - No Fix Available
If no upgrade path exists for a package:
1. Document this clearly
2. Continue with other packages
3. Suggest alternatives in summary (replace package, patch, accept risk)

### Partial Success
When fixing multiple vulnerabilities, some may succeed while others fail:
1. Keep ALL successful fixes
2. Track status of each targeted vulnerability separately
3. Document which items remain unfixed and why
4. Add TODO comments for unfixed code instances
5. Report clear breakdown in summary: X fixed, Y failed

### Rollback Triggers
Revert changes for a SPECIFIC fix if:
- Unable to produce clean fix after 3 attempts (new vulnerabilities introduced by that fix)
- Fix would require changing business logic
- Dependency resolution fails for that specific package

**Note**: Rollback is per-vulnerability, not global. Successful fixes are preserved even if others fail.

---

## Constraints

1. **Fix all targeted vulnerabilities** - Handle all specified vulnerabilities in a single run
2. **Minimal changes** - Only modify what's necessary for each fix
3. **No new vulnerabilities** - Each fix must be clean (or net improvement for SCA)
4. **Preserve functionality** - Tests must pass
5. **No scope creep** - Don't refactor or "improve" other code
6. **Consistent fixes** - Apply the same fix pattern across all instances of each vulnerability type
7. **Independent fixes** - Each vulnerability fix should be independent; failure of one doesn't block others
8. **Clear status tracking** - Track and report status of each targeted vulnerability separately

---

## Completion Checklist

Before ending the conversation, verify ALL are complete:

- [ ] All targeted vulnerabilities identified and documented
- [ ] Fix applied for each targeted vulnerability (or documented as unfixable)
- [ ] Re-scan shows targeted issues resolved (or partial success documented)
- [ ] Tests pass (or failures documented)
- [ ] Summary displayed with status of each targeted vulnerability
- [ ] Snyk feedback sent with total count of issues fixed
