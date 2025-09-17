# FireFoundry CLI Setup

The FireFoundry CLI (`ff-cli`) is essential for creating and managing agent projects. This guide will help you set up the CLI, which requires a GitHub token to access private repositories.

## Prerequisites

Before setting up the FireFoundry CLI, ensure you have completed:

1. **[Prerequisites](01-prerequisites.md)** - Core tools installed
2. **[Environment Setup](03-environment-setup.md)** - minikube cluster running
3. **[Deploy Services](04-deployment.md)** - FireFoundry infrastructure deployed

## Step 1: Obtain a GitHub Token

**If you have a Firebrand GitHub account:**

- Create a classic personal access token with `read:packages` scope
- Visit: https://github.com/settings/tokens

**If you don't have a Firebrand GitHub account:**

- Request a token from doug@firebrand.ai or augustus@firebrand.ai

## Step 2: Configure NPM for Private Packages

The ff-cli is published to GitHub Packages, so you need to configure npm to authenticate.

Create or edit the `.npmrc` file in your home directory:

**File location:**

- **macOS/Linux**: `~/.npmrc`
- **Windows**: `C:\Users\YourUsername\.npmrc`

**File contents:**

```
@firebrandanalytics:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=your_github_token_here
```

Replace `your_github_token_here` with your actual GitHub token.

## Step 3: Install FireFoundry CLI

With the token configured, install the CLI:

```bash
# Install the unified FireFoundry CLI
npm install -g @firebrandanalytics/ff-cli@latest
```

## Step 4: Set Environment Variable (Optional)

For additional GitHub operations, you may also want to set the token as an environment variable:

**For macOS/Linux** (add to your shell profile):

```bash
# Add to ~/.zshrc (for zsh) or ~/.bash_profile (for bash)
echo 'export GITHUB_TOKEN="your_token_here"' >> ~/.zshrc
source ~/.zshrc

# Verify it's set
echo $GITHUB_TOKEN
```

**For Windows** (Git Bash):

```bash
# Add to ~/.bashrc
echo 'export GITHUB_TOKEN="your_token_here"' >> ~/.bashrc
source ~/.bashrc

# Verify it's set
echo $GITHUB_TOKEN
```

## Step 5: Verify Setup

Test that your CLI installation is working:

```bash
# Test ff-cli installation
ff-cli --version

# Test access to examples (requires GitHub token)
ff-cli project create --list-examples
```

**Expected output:**

- `ff-cli --version` should show the CLI version
- `ff-cli project create --list-examples` should list available project templates

## What the FireFoundry CLI Does

The `ff-cli` tool provides:

- **Project Scaffolding**: Creates new agent projects with best practices and proper structure
- **Template Access**: Provides access to example repositories and starter templates
- **Development Workflows**: Manages agent bundle development and testing processes
- **Example Discovery**: Helps you find and clone FireFoundry examples for learning

## Next Steps

With the FireFoundry CLI installed, you're ready to start agent development:

1. **[Agent Development](05-agent-development.md)** - Create your first agent bundle
2. **[Operations & Maintenance](06-operations.md)** - Monitor and debug your agents
3. **[Troubleshooting](07-troubleshooting.md)** - Common issues and solutions

## Troubleshooting

**If npm install fails:**

- Verify your `.npmrc` file has the correct token
- Check that your GitHub token has `read:packages` scope
- Try clearing npm cache: `npm cache clean --force`

**If ff-cli commands fail:**

- Verify the token is correctly set in `.npmrc`
- Check your internet connection
- Ensure you have access to the Firebrand GitHub organization
