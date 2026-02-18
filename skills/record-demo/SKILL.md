---
description: Record a video demo of a web app feature using browser automation
argument-hint: <feature to demo, e.g. "the search and filter flow">
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep]
---

# Record Demo Skill

You are recording a video demo of a web application feature using Playwright browser automation. Follow these steps exactly.

**Architecture:** This plugin separates distributable code from runtime state:

- **Plugin directory** (distributed): Contains only this SKILL.md prompt. Installed via marketplace or `--plugin-dir`.
- **Data directory** (`~/.claude/record-demo/`): All runtime state — Playwright installation, auth state, recordings, project configs. Created automatically on first use. Never distributed.
- **Shared config** (optional): `demo.config.json` in the project repo — committed, shared with the team.
- **Local config** (optional): `~/.claude/record-demo/projects/<project-name>.json` — personal overrides.

```
~/.claude/record-demo/                    # Per-user runtime data (auto-created)
  package.json                            # Playwright dependency
  node_modules/                           # Installed here
  projects/                               # Local per-project config overrides
    <project-name>.json
  auth/                                   # Saved browser auth state
    <project-name>.json
  recordings/                             # All recordings
    <project-name>/
      <timestamp>-<slug>/
        script.mjs
        recording.webm
        recording.mp4
        screenshot-failure.png

<any-project>/                            # Optional shared config
  demo.config.json                        # Committed to repo
```

The **project name** is the basename of the git repo root (or working directory). For example, working in `~/foo/myCoolProject` → project name is `myCoolProject`.

---

## 1. Read Config and Validate Setup

**Determine the project name:**
- Find the git root of the current working directory (`git rev-parse --show-toplevel`)
- Take the basename as the project name

**Ensure data directory exists:**
```bash
mkdir -p ~/.claude/record-demo
```

If `~/.claude/record-demo/package.json` does not exist, create it:
```json
{
  "name": "record-demo-data",
  "private": true,
  "type": "module",
  "dependencies": {
    "playwright": "^1.50.0"
  }
}
```

**Resolve config (layered, later wins):**
1. Start with defaults (see schema below)
2. Look for `demo.config.json` in the project root (the git root). If found, merge it on top of defaults. This is the **shared config** — teams can commit this to their repo.
3. Look for `~/.claude/record-demo/projects/<project-name>.json`. If found, merge it on top. This is the **local config** — personal overrides that aren't committed.
4. If neither file exists: ask the user for a base URL and auth strategy, then create the local config file at `~/.claude/record-demo/projects/<project-name>.json`.

Merging is shallow per top-level key: if local config has `"auth"`, it replaces the entire `"auth"` object from shared config (not deep-merged).

**Config schema:**
```json
{
  "baseUrl": "http://localhost:3000",
  "auth": {
    "strategy": "saved-state",
    "loginUrl": "http://localhost:3000/login",
    "customScript": null,
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

**`auth` fields explained:**
- **`strategy`**: `"saved-state"` (recommended), `"form-login"`, `"none"`, or `"custom"`.
- **`loginUrl`**: Where to navigate for login.
- **`customScript`**: Path to a custom login script (for `"custom"` strategy).
- **`captureDetect`**: How to detect that the user has finished logging in during auth capture. Replaces interactive stdin (which doesn't work in Claude Code). Types:
  - `"localStorage"` — poll for a localStorage key containing `match` (e.g. `"@@auth0spajs@@"` for Auth0, `"supabase.auth.token"` for Supabase)
  - `"cookie"` — poll for a cookie whose name contains `match`
  - `"selector"` — wait for a CSS selector in `match` to appear in the DOM (e.g. `"[data-testid='user-avatar']"`)
  - `"url"` — wait for the page URL to match the regex in `match` (e.g. `"^http://localhost:3000(?!/login)"` — useful when auth redirects away and back)
- **`readySelector`**: CSS selector that indicates the auth SDK has fully initialized and the UI is in an authenticated state. The recording script waits for this element after page load before interacting. **This is critical for SPAs** — auth SDKs (Auth0, Firebase, Supabase, etc.) need time to hydrate tokens from localStorage/cookies after page load. During that window the UI renders in an unauthenticated state, and clicking protected elements triggers login modals. Example: `"a[href='/dashboard']"` (a link that only renders for logged-in users). Default: `null` (skip this wait).
- **`readyTimeout`**: How long (ms) to wait for `readySelector`. Default: `15000`.

**Validate tooling:**

1. Check that Playwright is installed in the data directory:
   ```bash
   ls ~/.claude/record-demo/node_modules/playwright/package.json
   ```
   If not found, run:
   ```bash
   cd ~/.claude/record-demo && npm install
   ```
   Then check if the Chromium browser is available:
   ```bash
   cd ~/.claude/record-demo && npx playwright install --dry-run chromium
   ```
   If browsers are not installed, tell the user to run:
   ```bash
   cd ~/.claude/record-demo && npx playwright install chromium
   ```
   and stop.

2. Check if auth state file exists (if strategy is `"saved-state"`):
   ```bash
   ls ~/.claude/record-demo/auth/<project-name>.json
   ```

3. Check if `ffmpeg` is available: `which ffmpeg`

---

## 2. Understand the Demo Request

Parse `$ARGUMENTS` to understand what the user wants to demo.

**If `$ARGUMENTS` is empty or vague:**
- This skill runs in the same conversation, so you have full context of what the user has been working on.
- Look at the conversation history to identify what was just built, changed, or discussed.
- Offer a suggestion: "Based on what we just worked on, I can demo [specific feature/flow]. Want me to go with that, or did you have something else in mind?"

**If `$ARGUMENTS` is specific:**
- Use it directly as the demo target

**In both cases, produce a numbered demo plan** — a human-readable list of actions the recording will show. Example:

```
Demo plan:
1. Navigate to the home page
2. Click "Search" in the navigation
3. Type "NVIDIA" in the search bar
4. Wait for results to load
5. Click on the first listing result
6. Scroll down to show the full listing detail
7. Pause 2 seconds on the detail view
```

**Present the plan to the user and wait for confirmation before proceeding.** Let them add, remove, or reorder steps.

---

## 3. Explore the Codebase

Use Glob and Grep to find the frontend routes and components relevant to the demo:

- Use `hints.routesDir` from config if provided (e.g. `src/app/` for Next.js app router)
- Use `hints.componentsDir` from config if provided
- Search for route definitions, page components, and relevant UI elements

**Read the actual component files** to find real selectors. Look for:
- `data-testid` attributes
- aria labels and roles
- Button/link text content
- Form input names and placeholders
- Unique CSS classes (as a last resort)

**IMPORTANT: Read the real frontend code. Do NOT guess selectors.** Every click, fill, or wait target in the generated script must come from an actual selector you found in the source code.

---

## 4. Handle Authentication

Based on `auth.strategy` in the config:

### Strategy: `"saved-state"` (recommended)

Auth state file: `~/.claude/record-demo/auth/<project-name>.json`

If the file exists:
- It will be loaded into the Playwright browser context automatically
- Continue to Step 5

If the file does NOT exist, run the **one-time auth capture** (Step 4a).

### Strategy: `"form-login"`

The generated script will fill the login form. Ask the user for credentials if not provided previously.

### Strategy: `"none"`

Skip authentication entirely.

### Strategy: `"custom"`

Import the user-provided login script specified in `auth.customScript`.

### Step 4a: One-Time Auth Capture

Generate and run a small script at `~/.claude/record-demo/auth/capture-<project-name>.mjs` that:
1. Opens a **visible** (headful) browser
2. Navigates to the app's login URL
3. Tells the user: "Log in manually in the browser window"
4. Polls for successful login using the configured `captureDetect` method (up to 5 minutes)
5. Saves `context.storageState()` to `~/.claude/record-demo/auth/<project-name>.json`

> **Why polling, not readline?** Claude Code's Bash tool does not have interactive stdin, so `readline`-based "press Enter when done" approaches fail with `ERR_USE_AFTER_CLOSE`. Instead, the capture script polls for a signal that login completed.

```javascript
import { chromium } from 'playwright';

const authStatePath = '{authStateFile}';

const browser = await chromium.launch({ headless: false });
const context = await browser.newContext({
  viewport: { width: {viewportWidth}, height: {viewportHeight} }
});
const page = await context.newPage();

console.log('Opening browser — please log in. Will wait up to 5 minutes.');
await page.goto('{loginUrl}');

// Poll for successful login using configured captureDetect method
{captureDetectBlock}

await page.waitForLoadState('networkidle');
await page.waitForTimeout(3000);

console.log('Login detected! Saving auth state...');
await context.storageState({ path: authStatePath });
console.log('Auth state saved to', authStatePath);
await browser.close();
```

**Generate `{captureDetectBlock}` based on `captureDetect.type`:**

- **`"localStorage"`** (default — works for Auth0, Firebase, Supabase, etc.):
  ```javascript
  await page.waitForFunction((match) => {
    for (let i = 0; i < localStorage.length; i++) {
      if (localStorage.key(i)?.includes(match)) return true;
    }
    return false;
  }, '{match}', { timeout: 300000, polling: 2000 });
  ```

- **`"cookie"`**:
  ```javascript
  await page.waitForFunction((match) => document.cookie.includes(match),
    '{match}', { timeout: 300000, polling: 2000 });
  ```

- **`"selector"`** (e.g. wait for a user avatar or logout button):
  ```javascript
  await page.locator('{match}').waitFor({ state: 'visible', timeout: 300000 });
  ```

- **`"url"`** (e.g. wait for redirect back from auth provider):
  ```javascript
  await page.waitForFunction((pattern) => new RegExp(pattern).test(window.location.href),
    '{match}', { timeout: 300000, polling: 2000 });
  ```

Run from the data directory so imports resolve:
```bash
cd ~/.claude/record-demo && node auth/capture-<project-name>.mjs
```

After auth state is captured, continue to Step 5.

---

## 5. Generate the Playwright Script

Create the output directory:
```
~/.claude/record-demo/recordings/<project-name>/<timestamp>-<slug>/
```
Where `timestamp` is `YYYYMMDD-HHmmss` and `slug` is a short kebab-case version of the demo description.

Generate `script.mjs` in that directory. The script must be a **self-contained ESM module** that:

1. Imports from `playwright` (resolves from data dir's node_modules since we run from there)
2. Launches the configured browser
3. Creates a context with:
   - `recordVideo: { dir: '<output-dir>', size: { width, height } }` from config viewport
   - `storageState` loaded from `~/.claude/record-demo/auth/<project-name>.json` (if applicable)
   - `viewport` from config
4. Creates a page
5. Executes each demo plan step as Playwright actions:
   - `page.goto()` for navigation
   - `page.getByRole()`, `page.getByText()`, `page.getByPlaceholder()`, `page.getByTestId()` for element selection
   - `page.click()`, `page.fill()`, `page.hover()` for interactions
   - `page.waitForLoadState('networkidle')` after navigations (or per config `waitStrategy`)
   - `page.waitForTimeout(1500)` between steps for visual pacing
   - `page.waitForTimeout(2500)` for "pause and show" moments
6. Wraps everything in try/catch:
   - On error: takes a screenshot to `screenshot-failure.png`, logs the error
7. In `finally`: calls `context.close()` (flushes video), then `browser.close()`
8. After context close, finds the video file and prints its path to stdout

**Auth readiness wait (if `readySelector` is configured):** After the first `page.goto()` and `waitForLoadState`, wait for the `readySelector` element to become visible (with `readyTimeout`). If it times out, take a failure screenshot and `process.exit(2)` to signal expired auth. This gives the auth SDK time to hydrate tokens from localStorage/cookies before the script interacts with the page. If `readySelector` is `null`, skip this wait.

**Auth expiry detection (fallback when no readySelector):** After the first `page.goto()`, check if the URL was redirected to an auth provider (e.g. contains `auth0.com`, `login`). If so, `process.exit(2)` to signal expired auth.

**Use absolute paths** in the generated script for auth state and output directory, since the script runs from the data directory, not the project directory.

**Example script structure:**
```javascript
import { chromium } from 'playwright';
import path from 'path';
import fs from 'fs';

const outputDir = '/Users/.../.claude/record-demo/recordings/<project>/<dir>';
const authState = '/Users/.../.claude/record-demo/auth/<project>.json';

const browser = await chromium.launch({ headless: true });
const context = await browser.newContext({
  viewport: { width: 1280, height: 720 },
  recordVideo: { dir: outputDir, size: { width: 1280, height: 720 } },
  storageState: authState,
});
const page = await context.newPage();

try {
  await page.goto('{baseUrl}');
  await page.waitForLoadState('networkidle');

  // Auth expiry check (fallback when no readySelector)
  if (page.url().includes('auth0.com') || page.url().includes('/login')) {
    console.error('Auth state expired — re-run auth capture');
    process.exit(2);
  }

  // Wait for auth SDK to initialize (if readySelector configured)
  // Auth SDKs (Auth0, Firebase, etc.) hydrate tokens from localStorage
  // asynchronously. The UI starts unauthenticated and re-renders once
  // the SDK initializes. readySelector targets an element that only
  // appears in the authenticated state.
  // {readySelectorBlock — include only if readySelector is not null}
  console.log('Waiting for auth to initialize...');
  try {
    await page.locator('{readySelector}').waitFor({ state: 'visible', timeout: {readyTimeout} });
  } catch {
    console.error('Auth readySelector not found — auth may have expired');
    await page.screenshot({ path: path.join(outputDir, 'screenshot-failure.png') });
    process.exit(2);
  }

  await page.waitForTimeout(1500);

  // Demo steps...

} catch (err) {
  console.error('Recording failed:', err.message);
  await page.screenshot({ path: path.join(outputDir, 'screenshot-failure.png') });
  process.exitCode = 1;
} finally {
  await context.close();
  await browser.close();
}

// Find the newest video file (multiple takes may exist in the same directory)
const files = fs.readdirSync(outputDir).filter(f => f.endsWith('.webm'));
if (files.length > 0) {
  const sorted = files
    .map(f => ({ name: f, mtime: fs.statSync(path.join(outputDir, f)).mtimeMs }))
    .sort((a, b) => b.mtime - a.mtime);
  const videoPath = path.join(outputDir, sorted[0].name);
  console.log('Video saved:', videoPath);
}
```

**Run from the data directory:**
```bash
cd ~/.claude/record-demo && node recordings/<project>/<dir>/script.mjs
```

---

## 6. Run and Present

Execute the script with a timeout (use `timeoutMs` from config, default 120 seconds):

```bash
cd ~/.claude/record-demo && node recordings/<project>/<dir>/script.mjs
```

**If exit code is 2 (auth expired):**
- Tell the user: "Auth state has expired. Running auth capture so you can log in again..."
- Delete the expired auth state file
- Go back to Step 4a to re-capture auth state
- Then re-run the recording

**If exit code is 1 (script error):**
- Show the error output
- If `screenshot-failure.png` exists, tell the user to look at it for context
- Ask the user what to do — do NOT auto-retry

**If exit code is 0 (success):**
1. Attempt mp4 conversion if ffmpeg is available:
   ```bash
   ffmpeg -i recording.webm -c:v libx264 -preset fast -crf 23 -movflags +faststart recording.mp4
   ```
   If ffmpeg fails, skip silently and use the webm file.
2. Open the video file: `open <video-path>`
3. Tell the user the file path

---

## 7. Feedback Loop

Ask the user:

> **How does the recording look?** Say **adjust** with what to change (e.g. "adjust: slow down the scrolling"), or **done** to finish.

**On "adjust":**
- Modify `script.mjs` based on feedback (edit in place)
- Re-run the script (new recording gets a new video filename; previous takes are preserved)
- Show the new result and ask again

**On "done":**
- Proceed to Step 8

---

## 8. Cleanup and Summary

Print a summary:

```
Demo recording complete!

Final video: <path>
Script: <path>
```

If there were multiple takes, list them all:
```
All takes:
  1. <path-to-first-video>
  2. <path-to-second-video> (final)
```

---

## Important Guidelines

- **Runtime data lives in `~/.claude/record-demo/`** — auth state, recordings, scripts, node_modules. Never write these to the project directory. Shared config (`demo.config.json`) may optionally live in the project root.
- **Auth state is for local dev only** — saved-state stores browser cookies/localStorage to a plain JSON file. This is fine for local dev session tokens (localhost). Don't use it with production credentials.
- **Always read real code for selectors** — never guess CSS selectors, test IDs, or text content
- **Pace the demo for humans** — 1.5s between actions, 2.5s for "look at this" moments
- **Keep scripts self-contained** — each `script.mjs` should run independently
- **Preserve previous takes** — never overwrite a previous recording
- **Headless for recording, headful for auth** — recordings are headless; auth capture must be headful
- **Fail gracefully** — always capture a failure screenshot, always close the browser
- **Use ESM (.mjs)** — all generated scripts use ES module syntax
- **Run from the data directory** — so Playwright imports resolve from the data dir's node_modules
- **Use absolute paths in scripts** — since scripts run from the data dir, not the project
- **Leverage conversation context** — if the user just built something, you know what to demo without them explaining it again
