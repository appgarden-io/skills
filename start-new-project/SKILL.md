---
name: start-new-project
description: Scaffolds a new prototype project from the appgarden template. Use this skill whenever the user wants to start a new project, create a new prototype, spin up a new app, or bootstrap a new repo in the appgarden workspace. Even if they say something casual like "let's start something new" or "new project" — this is the skill to use.
---

# Start New Project

This skill walks the user through creating a new prototype project from the appgarden template repo. It handles navigation, repo creation, cloning, and dependency installation so the user can start building immediately.

## Steps

### 1. Navigate to the appgarden root directory

The appgarden root is the directory that contains all appgarden prototype projects (it typically contains repos like `prototype-template`, `appgarden`, and other project folders).

Check if the current working directory is the appgarden root:
- Look for sibling directories like `prototype-template/` or `appgarden/` as indicators
- If you're already inside a project subdirectory of appgarden, move up to the parent

If you can't determine the root, ask the user for the path.

### 2. Check that the GitHub CLI is installed and authenticated

Before proceeding, verify that the `gh` CLI is available and authenticated:

```bash
gh auth status
```

- If `gh` is **not installed**, install it with `brew install gh`.
- If `gh` is installed but **not authenticated**, tell the user they need to authenticate and run `gh auth login` for them. Wait for them to complete the interactive login flow before continuing.
- If `gh auth status` succeeds, move on to the next step.

### 3. Ask for the project name

Ask the user what they'd like to name the new project. The name should be kebab-case (lowercase with hyphens) since it becomes both the GitHub repo name and the local directory name.

### 4. Create the repo from the template

Use the GitHub CLI to create a new **private** repo under the `appgarden-io` org, using the prototype template:

```bash
gh repo create appgarden-io/<project-name> --template appgarden-io/prototype-template --private --clone
```

This creates the repo on GitHub and clones it locally in one step.

### 5. Install dependencies and start the dev server

Navigate into the newly cloned project directory, install packages, and start the dev server in the background:

```bash
cd <project-name>
pnpm i
pnpm run dev  # run this in the background so you can continue
```

If `pnpm` is not installed, suggest installing it with `npm install -g pnpm` or `corepack enable`.

### 6. Check that agent-browser is installed

Before verifying the dev server, ensure the `agent-browser` CLI is available:

```bash
agent-browser --version
```

- If the command succeeds, move on to the next step.
- If `agent-browser` is **not installed**, install it globally and download Chromium:
  ```bash
  npm install -g agent-browser
  agent-browser install
  ```
  On Linux, if Chromium dependencies are missing, run `agent-browser install --with-deps`.
- See https://github.com/vercel-labs/agent-browser for more details.

### 7. Verify the dev server with agent-browser

After the dev server starts, use the `agent-browser` CLI to verify it's actually running and rendering correctly. Wait a few seconds for the server to be ready, then:

1. **Open the dev server URL** (default `http://localhost:5173`):
   ```bash
   agent-browser open http://localhost:5173
   ```

2. **Take an accessibility snapshot** to verify the page rendered:
   ```bash
   agent-browser snapshot
   ```

3. **Take a screenshot** to visually confirm the page looks correct:
   ```bash
   agent-browser screenshot
   ```

4. **Close the browser**:
   ```bash
   agent-browser close
   ```

Check the snapshot output for expected content (e.g. an "Home" heading, interactive elements). If the page loaded and has the expected content, the server is working. If the snapshot shows an error page or is empty, debug the issue before continuing.

### 8. Confirm and hand off

Let the user know the project is ready. Mention:
- The repo URL: `https://github.com/appgarden-io/<project-name>`
- The dev server is running and was verified with agent-browser
- The project follows the conventions in `.claude/CLAUDE.md`
