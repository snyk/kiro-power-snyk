# Snyk Secure at Inception Power for Kiro

This repo is the public repo for installing the "Snyk Secure at Inception" Power into the Kiro (Amazon's AI agent) IDE. A Kiro Power is a packaged extension that adds specialized capabilities to the Kiro AI development environment. Extensive docs on what powers are, how to install them, and how to use them can be found here: https://kiro.dev/docs/powers/. Below is a guide on how to install and use the Snyk power, assuming you download the Kiro IDE and have node JS on your machine.

## How to Use

### Prerequisites

- **Kiro IDE** (Free tier available at [kiro.ai](https://kiro.ai)). Snyk currently doesn't have any of the paid tiers for Kiro, so use the free tier for now. It comes with 50 tokens.
- **Node.js** installed on your machine with npm/npx

### Installation

1. **Install the Power in Kiro:**
   - Open Kiro IDE
   - Access the Powers panel (Ctrl/Cmd + Shift + P → "Configure Powers", or clicking on the ghost with a lightning bolt next to it in the left menu)
   - Click on "Add Custom Power"
   - Select the "Import power from Github" option
   - Use the link to the snyk-power folder inside this repo as the power: https://github.com/snyk/kiro-power-snyk/tree/598bab69c81607c17445fbeefa9c75c405e91ec3/snyk-power
   **IMPORTANT:** make sure to specifically use that link above which includes the snyk-power folder; if you try to use the overall repo link the installation will fail due to having extra folders/files in the repo.
   - The power will be installed and ready to use

2. **Authenticate with Snyk:**
   In your terminal in Kiro run
   ```
   npx snyk auth
   ```
   This opens your browser for one-time Snyk authentication.

## Using once Installed
Once the Snyk Power is installed in the Kiro IDE, you'll be able to use all of the Snyk MCP commands inside Kiro. In your chat with the Kiro agent, you can reference any of the snyk-fix commands, as defined in the snyk-fix.md file in this repo (kiro-power-snyk/snyk-power/steering/snyk-fix.md). 

## Configuration

The power automatically configures the Snyk MCP server. If you encounter connection issues, ensure:

1. **npx is available** in your PATH
2. **Node.js environment** is properly set up
3. **Snyk authentication** is completed

### Troubleshooting MCP Connection

If you see `spawn npx ENOENT` errors:

1. Find your npx path: `which npx`
2. Update your MCP configuration in `~/.kiro/settings/mcp.json`:

```json
{
  "powers": {
    "mcpServers": {
      "power-snyk-secure-at-inception-Snyk": {
        "command": "/full/path/to/npx",
        "args": ["-y", "snyk@latest", "mcp", "-t", "stdio"],
        "env": {
          "PATH": "/your/node/bin:/usr/local/bin:/usr/bin:/bin"
        }
      }
    }
  }
}
```
