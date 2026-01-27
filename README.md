> **Note:** This repository is closed to public contributions.

# Snyk Secure at Inception Power for Kiro

This repo is the public repo for installing the "Snyk Secure at Inception" Power into the Kiro (Amazon's AI agent) IDE. A Kiro Power is a packaged extension that adds specialized capabilities to the Kiro AI development environment. Extensive docs on what powers are, how to install them, and how to use them can be found here: https://kiro.dev/docs/powers/. Below is a guide on how to install and use the Snyk power, assuming you download the Kiro IDE and have node JS on your machine.

## How to Use

### Prerequisites
- **Kiro IDE** (Free tier available at [kiro.ai](https://kiro.ai)). Snyk currently doesn't have any of the paid tiers for Kiro, so use the free tier for now. It comes with 50 tokens.
- **Node.js** installed on your machine with npm/npx

### Installation

1. **Set up Node.js System Links (Required for MCP Server):**
   
   The Snyk Power uses an MCP (Model Context Protocol) server that requires `node` and `npx` to be available in the system PATH. If you're using a Node.js version manager like nvm, fnm, or volta, you'll need to create system-wide symlinks.

   **For nvm users:**
   ```bash
   # Find your current Node.js path
   which node
   which npx
   
   # Create symlinks (replace the path with your actual Node.js path)
   sudo ln -sf $(which node) /usr/local/bin/node
   sudo ln -sf $(which npx) /usr/local/bin/npx
   ```

   **For Homebrew users:**
   ```bash
   # If you don't have Node.js via Homebrew, install it:
   brew install node
   # Homebrew automatically creates the necessary symlinks
   ```

   **Verify the setup:**
   ```bash
   /usr/local/bin/node --version
   /usr/local/bin/npx --version
   ```

2. **Install the Power in Kiro:**
   - Open Kiro IDE
   - Access the Powers panel (Ctrl/Cmd + Shift + P → "Configure Powers", or clicking on the ghost with a lightning bolt next to it in the left menu)
   - Click on "Add Custom Power"
   - Select the "Import power from Github" option
   - Use the link to the snyk-power folder inside this repo as the power: https://github.com/snyk/kiro-power-snyk/tree/598bab69c81607c17445fbeefa9c75c405e91ec3/snyk-power

   **IMPORTANT:** make sure to specifically use that link above which includes the snyk-power folder; if you try to use the overall repo link the installation will fail due to having extra folders/files in the repo.
   - The power will be installed and ready to use

3. **Authenticate with Snyk:**
   In your terminal in Kiro run
   ```
   npx snyk auth
   ```
   This opens your browser for one-time Snyk authentication.

## Using once Installed

Once the Snyk Power is installed in the Kiro IDE, you'll be able to use all of the Snyk MCP commands inside Kiro. In your chat with the Kiro agent, you can reference any of the snyk-fix commands, as defined in the snyk-fix.md file in this repo (kiro-power-snyk/snyk-power/steering/snyk-fix.md). 

## Configuration

The power automatically configures the Snyk MCP server to use the system-wide Node.js installation at `/usr/local/bin/npx`. This approach ensures compatibility across different Node.js installation methods.

### Troubleshooting

#### MCP Connection Issues

If you see `spawn npx ENOENT` errors, it means the MCP server can't find `npx`. This usually happens when:

1. **Node.js symlinks are missing** - Follow step 1 in the installation guide above
2. **Node.js is not installed** - Install Node.js via Homebrew (`brew install node`) or download from [nodejs.org](https://nodejs.org)

#### Verifying Your Setup

Run these commands to verify everything is working:

```bash
# Check if symlinks exist and work
ls -la /usr/local/bin/node /usr/local/bin/npx
/usr/local/bin/node --version
/usr/local/bin/npx --version

# Test Snyk CLI
npx snyk --version
npx snyk auth --check
```

#### Alternative Configuration (Advanced Users)

If you prefer not to create system symlinks, you can manually configure the MCP server path in `~/.kiro/settings/mcp.json`:

**Steps:**
1. Find your npx path: `which npx`
2. Open `~/.kiro/settings/mcp.json` in a text editor
3. Navigate to `powers` → `mcpServers` → `power-snyk-secure-at-inception-Snyk`
4. Update the `command` field with your full npx path
5. Update the `PATH` in the `env` field to include your Node.js bin directory
6. Save the file and restart Kiro IDE

**Example for nvm users:**
```json
{
  "powers": {
    "mcpServers": {
      "power-snyk-secure-at-inception-Snyk": {
        "command": "/Users/yourusername/.nvm/versions/node/v20.19.6/bin/npx",
        "args": ["-y", "snyk@latest", "mcp", "-t", "stdio"],
        "env": {
          "PATH": "/Users/yourusername/.nvm/versions/node/v20.19.6/bin:/usr/local/bin:/usr/bin:/bin"
        }
      }
    }
  }
}
```

#### Common Issues

| Error | Solution |
|-------|----------|
| `spawn npx ENOENT` | Create Node.js symlinks (step 1) or install Node.js |
| `Unauthorized` | Run `npx snyk auth` to authenticate |
| `Command not found: npx` | Install Node.js or check your Node.js installation |
| MCP server won't connect | Restart Kiro IDE after creating symlinks |

#### Why Symlinks?

Node.js version managers (nvm, fnm, volta) install Node.js in user-specific directories that aren't in the system PATH by default. The MCP server process needs access to `npx` from a standard system location (`/usr/local/bin/`) to work reliably across different environments.