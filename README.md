> **Note:** This repository is closed to public contributions.

# Snyk Secure at Inception Power for Kiro

This repo is the public repo for installing the "Snyk Secure at Inception" Power into the Kiro (Amazon's AI agent) IDE. A Kiro Power is a packaged extension that adds specialized capabilities to the Kiro AI development environment. Extensive docs on what powers are, how to install them, and how to use them can be found here: https://kiro.dev/docs/powers/. Below is a guide on how to install and use the Snyk power, assuming you download the Kiro IDE and have node JS on your machine.

## How to Use

### Prerequisites
- **Kiro IDE** (Free tier available at [kiro.ai](https://kiro.ai)). Snyk currently doesn't have any of the paid tiers for Kiro, so use the free tier for now. It comes with 50 tokens.
- **Node.js** installed on your machine with npm/npx:
  -To verify this, you can run "which npx" and "which node" in the terminal and if you see a version number for both, then npx and node are both succesfully installed on your machine.

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

### Troubleshooting

#### MCP Connection Issues

If you see `spawn npx ENOENT` errors, it means the MCP server can't find `npx`. This usually happens when:

1. **Kiro is not seeing npx or node**:
Verify that npx and node are both installed by running "which npx" and "which node" and verifying that you see version numbers for both. If they are and this issue persists, it may be an issue with Kiro itself. Close Kiro and try reopening it thorugh the terminal, by going to your desired directory and running "kiro .", and this will hopefully fix the issue.

2. **Node.js is not installed**:
Install Node.js via Homebrew (`brew install node`) or download from [nodejs.org](https://nodejs.org)

#### Verifying Your Setup

Run these commands to verify everything is working:

```bash
# Check if npx and node exist
which npx
which node

# Test Snyk CLI
npx snyk --version
npx snyk auth --check
```