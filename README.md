# record-demo

A Claude Code skill that records video demos of web app features using Playwright browser automation.

## What it does

Invoke `/record-demo <description>` and Claude will:

1. Explore your frontend codebase to find real selectors
2. Generate a Playwright script tailored to your app
3. Record a headless browser session as video
4. Let you iterate with "adjust" feedback until it looks right

## Use cases

- **Enable a feedback loop for how your features actually work.** Validate LLM-generated code and apply your sense of taste as part of your normal workflow.
- **Share a feature with your team.** Just built something? Record a 30-second walkthrough and drop it in Slack or a PR description. Faster than writing steps to reproduce, and people actually watch it.
- **Demo for a stakeholder.** Product review coming up? Record the flow ahead of time so you're not fumbling through it live.
- **Document a bug.** `/record-demo` the broken flow, then fix it and record again. Attach both to the ticket.
- **Onboard new devs.** Record walkthroughs of key flows so new teammates can see how the app actually works.
- **Before/after comparisons.** Record the current behavior, make your changes, record again.

## Install

Clone (or fork) this repo, then symlink the skill into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/record-demo
ln -sf <local checkout location>/skills/record-demo/SKILL.md ~/.claude/skills/record-demo/SKILL.md
```

That's it. Claude Code picks up skills from `~/.claude/skills/` automatically — no restart needed.

To update later, just `git pull` in the repo. The symlink means Claude always reads the latest version.

## First-time setup

On first use, the skill will:

1. Create a data directory at `~/.claude/record-demo/` for runtime state
2. Install Playwright and Chromium there
3. If your app requires authentication, open a browser for you to log in once

## Configuration

The skill supports layered config (later wins):

1. **Shared config**: `demo.config.json` in your project root (committed to repo)
2. **Local config**: `~/.claude/record-demo/projects/<project-name>.json` (personal overrides)

### Config schema

```json
{
  "baseUrl": "http://localhost:3000",
  "auth": {
    "strategy": "saved-state",
    "loginUrl": "http://localhost:3000/login",
    "captureDetect": {
      "type": "localStorage",
      "match": "authToken"
    },
    "readySelector": null,
    "readyTimeout": 15000
  },
  "viewport": { "width": 1280, "height": 720 },
  "browser": "chromium",
  "timeoutMs": 120000,
  "hints": {
    "routesDir": null,
    "componentsDir": null,
    "selectorStrategy": "prefer text and role selectors",
    "waitStrategy": "networkidle",
    "notes": null
  }
}
```

### Auth strategies

- **`saved-state`** (recommended): Captures browser state once, reuses for recordings. Supports `captureDetect` to auto-detect login completion and `readySelector` to wait for auth SDK initialization.
- **`form-login`**: Fills login form programmatically.
- **`none`**: No authentication.
- **`custom`**: Use a custom login script.

### `captureDetect` types

| Type | Match example | Use case |
|------|--------------|----------|
| `localStorage` | `@@auth0spajs@@` | Auth0, Firebase, Supabase |
| `cookie` | `session_id` | Cookie-based auth |
| `selector` | `[data-testid='avatar']` | Wait for a DOM element |
| `url` | `^http://localhost:3000(?!/login)` | Wait for redirect back |

### `readySelector`

CSS selector for an element that only appears when the auth SDK has fully initialized (e.g., `a[href='/dashboard']`). Critical for SPAs where the auth SDK hydrates asynchronously from localStorage.

## Data directory

All runtime state lives in `~/.claude/record-demo/` (never in your project):

```
~/.claude/record-demo/
  package.json              # Playwright dependency
  node_modules/             # Installed here
  projects/                 # Per-project config overrides
    <project-name>.json
  auth/                     # Saved browser auth state
    <project-name>.json
  recordings/               # All recordings
    <project-name>/
      <timestamp>-<slug>/
        script.mjs
        *.webm / *.mp4
```

## A note on auth state

The `saved-state` strategy stores browser cookies and localStorage to a JSON file on your machine. **This is designed for local development credentials only** — the kind of throwaway session tokens you get from logging into `localhost:3000`. Don't use this with production credentials or anything you'd be nervous about having on disk. If your auth state expires, the skill will detect it and ask you to log in again.
