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

### 2. Ask for the project name

Ask the user what they'd like to name the new project. The name should be kebab-case (lowercase with hyphens) since it becomes both the GitHub repo name and the local directory name.

### 3. Create the repo from the template

Use the GitHub CLI to create a new **private** repo under the `appgarden-io` org, using the prototype template:

```bash
gh repo create appgarden-io/<project-name> --template appgarden-io/prototype-template --private --clone
```

This creates the repo on GitHub and clones it locally in one step.

If the `gh` CLI isn't authenticated or not installed, let the user know and help them resolve it (`gh auth login`).

### 4. Install dependencies and start the dev server

Navigate into the newly cloned project directory, install packages, and immediately start the dev server:

```bash
cd <project-name>
pnpm i && pnpm run dev
```

If `pnpm` is not installed, suggest installing it with `npm install -g pnpm` or `corepack enable`.

### 5. Confirm and hand off

Let the user know the project is ready. Mention:
- The repo URL: `https://github.com/appgarden-io/<project-name>`
- The dev server is running
- The project follows the conventions in `.claude/CLAUDE.md`
