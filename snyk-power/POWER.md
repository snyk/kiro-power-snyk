---
name: "snyk-secure-at-inception"
displayName: "Snyk Secure at Inception"
description: "Security scanning and remediation for your codebase using Snyk. Automatically scans generated code for vulnerabilities and provides tools to fix them."
author: "Snyk"
keywords: ["snyk", "security", "scan", "sast", "sca", "vulnerability", "vuln", "cve", "cwe", "appsec", "devsecops", "code-scan", "dependency-scan", "container-scan", "iac", "remediation", "fix"]
---

# Snyk Secure at Inception

Security scanning and remediation for your codebase using Snyk. Automatically scans generated code for vulnerabilities and provides tools to fix them.

## Activation Keywords

This power activates when you mention:
- security, vulnerability, vuln, CVE, CWE
- snyk, scan, SAST, SCA
- dependency vulnerability, code vulnerability
- fix security, remediate

## What This Power Provides

1. **Snyk MCP Server** - Access to Snyk's security scanning tools
2. **Secure at Inception Rules** - Automatic scanning of generated code
3. **`/snyk-fix` Command** - Interactive vulnerability remediation

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| **Node.js** | Required for npx to run Snyk |
| **Snyk Account** | Free at [snyk.io](https://snyk.io) |

On first use, Snyk will prompt you to authenticate (opens browser).

## Available MCP Tools

This power uses the **Snyk MCP Server** which provides:

| Tool | Purpose |
|------|---------|
| `snyk_code_scan` | Static Application Security Testing (SAST) - finds vulnerabilities in your code |
| `snyk_sca_scan` | Software Composition Analysis - finds vulnerable dependencies |
| `snyk_container_scan` | Container image vulnerability scanning |
| `snyk_iac_scan` | Infrastructure as Code security scanning |
| `snyk_sbom_scan` | Scan existing SBOM files for vulnerabilities |
| `snyk_auth` | Authenticate with Snyk |
| `snyk_send_feedback` | Report remediation metrics |

## Steering Files

### `steering/security-rules.md` - Secure at Inception

Auto-applied to all files (`applyTo: "**"`). Instructs the agent to:
- Automatically scan new code with `snyk_code_scan`
- Fix any security issues found
- Re-scan after fixes to validate
- Iterate until no new issues remain

This implements Snyk's "Secure at Inception" pattern for agentic workflows.

### `steering/snyk-fix.md` - `/snyk-fix` Command

Complete security remediation workflow. Scans for vulnerabilities, fixes them, and validates the fixes.

**Usage:**
```
/snyk-fix                                    # Auto-detect and fix highest priority issue
/snyk-fix code                               # SAST scan only
/snyk-fix sca                                # SCA scan only  
/snyk-fix javascript/PT@files.ts:45          # Fix specific code vulnerability
/snyk-fix sca:lodash, sca:express            # Fix multiple dependency issues
/snyk-fix javascript/XSS@app.ts:20, sca:lodash  # Fix mixed code + dependency issues
```

See `steering/snyk-fix.md` for full documentation.

## MCP Server Configuration

The `mcp.json` configures the Snyk MCP server to run via npx:

```json
{
  "mcpServers": {
    "Snyk": {
      "command": "npx",
      "args": ["-y", "snyk@latest", "mcp", "-t", "stdio"],
      "env": {}
    }
  }
}
```

No global Snyk CLI installation required - npx handles it automatically.

This power integrates with Snyk MCP Server (Apache-2.0 license)
