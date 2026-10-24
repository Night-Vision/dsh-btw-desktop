# Windows and Linux Compatibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the POSIX/macOS assumptions in dsh-btw-desktop so `/btw` and temporary chat can run on Windows and Linux DSH Desktop, with every platform fact backed by the DSH docs (Context7, `/deepseek-ai/deepseek-harness`).

**Architecture:** Put the three path decisions (DSH home, usable cwd, session directory lookup) into one pure module `lib/paths.js` that takes `path`/env as arguments so it can be unit-tested for Windows from macOS. Stop computing DSH's session folder name (`projectKey`); find the session directory by scanning `<home>/sessions/*/<sessionId>`. Mirror the shipped Standard preset by choosing the shell tool per platform in `cordis.yml`.

**Tech Stack:** Node.js ESM (no dependencies), `node:test` for unit tests, the DSH app CLI plus a mock OpenAI-compatible model for end-to-end probes.

## Global Constraints

- Work only in `/Users/ilyaashirov/Documents/GitHub/dsh-btw-desktop`. Nothing outside the repo is touched except an isolated scratch `DSH_HOME` for probes.
- Zero runtime dependencies: Node builtins and injected DSH services only.
- No test framework beyond `node --test`. No build step; `lib/client.js` stays hand-written.
- macOS behaviour must not change. Every task ends with a macOS check.
- Do not commit or push unless the user asks. The user gives each commit instruction separately.
- The user has a Windows machine to test on. Claude cannot reach it: for every Windows step the plan gives exact PowerShell commands, the user runs them and pastes the output, and Claude adjusts from the real output. Task 0 runs first, so facts that are only assumed from the docs get confirmed before code changes.
- Linux stays untested (no Linux machine). The README keeps saying so until someone runs `test/manual` there.
- The README may claim "tested on Windows" only after the user pastes a passing Task 6 run.

## Verified facts (Context7, `/deepseek-ai/deepseek-harness`)

| Fact | Source |
|---|---|
| Home is `resolveDshHome()`: explicit path, then `$DSH_HOME`, then `join(homedir(), ".dsh")`, on every OS including Windows. A blank or whitespace-only `$DSH_HOME` counts as unset. | `packages/util/home-paths` |
| Sessions are stored as `<root>/--<normalized-cwd>--/<encoded-id>/…`. The id is escaped to one safe path segment (a UUID stays unchanged). `cwd` strings that normalize alike share a project directory. | `session-persistence-jsonl` README |
| The persistence service has `create/open/stat/list/flush` and no delete, so removing the directory stays the only way to burn a session. | `session-persistence` README |
| The bundled CLI is `resources/runtime/cli/bin/dsh` (`dsh.cmd` on Windows). To manage Desktop plugins: launch Desktop once, fully quit it, then run `dsh plugin --profile desktop add <package>`. | `apps/desktop` README |
| Shell tools are per platform: `tool-bash` on POSIX, `tool-pwsh` on Windows. The shipped Standard preset uses `disabled: !!js process.platform === 'win32'` on `tool-bash` and `!== 'win32'` on `tool-pwsh`. | `packages/shell` README, `standard.patch.yml` |
| Absolute `cwd` is required by the session store; "non-absolute path" throws. | `ctx.sessions.create` docs |

## File Structure

- Create `lib/paths.js`: pure helpers `resolveHome`, `isUsableCwd`, `sessionDirsFor`. One responsibility: platform-dependent path decisions.
- Create `test/paths.test.mjs`: unit tests for those helpers, including Windows paths through `path.win32`.
- Modify `lib/index.js`: use the helpers, delete `encodeSegmentChar`, `projectKey`, `sessionDir`, make `burnSession` retry on errors.
- Modify `lib/client.js:574`: platform-neutral folder-name display.
- Modify `cordis.yml`: shell tool per platform in the `btw-temp` preset.
- Modify `package.json`: add a `test` script.
- Modify `README.md`: install steps (quit the app first, Windows CLI name) and platform status.
- Create `test/manual/probe-temp.mjs` and `test/manual/mock-llm.mjs`: copies of the end-to-end probe and mock model so a Windows/Linux tester can run the same check.

---

### Task 0: Windows discovery (the user runs, read-only)

**Purpose:** confirm on the real Windows machine the facts the later tasks assume, before any code changes. Nothing here modifies anything.

**Files:** none.

**Interfaces:**
- Produces: a pasted output that settles four decisions: (a) `DSH_HOME` default and whether it is set, (b) the session layout under `<home>\sessions`, (c) the bundled CLI path, (d) which shell tool and Node are available.

- [ ] **Step 1: User runs this in PowerShell (read-only)**

```powershell
"DSH_HOME env: [$env:DSH_HOME]"
"default home exists: $(Test-Path "$HOME\.dsh")"
Get-ChildItem "$HOME\.dsh" -Force | Select-Object Name
"--- sessions layout (first two project folders, first two entries each)"
Get-ChildItem "$HOME\.dsh\sessions" -ErrorAction SilentlyContinue | Select-Object -First 2 | ForEach-Object {
  $_.Name; Get-ChildItem $_.FullName | Select-Object -First 2 -ExpandProperty Name
}
"--- profiles"
Get-ChildItem "$HOME\.dsh\profiles" -ErrorAction SilentlyContinue | Select-Object Name
"--- tools"
where.exe node; node --version
where.exe pwsh; pwsh --version
where.exe dsh
"--- bundled CLI (searches the usual Electron install folders)"
Get-ChildItem "$env:LOCALAPPDATA\Programs","$env:ProgramFiles" -Recurse -Filter dsh.cmd -ErrorAction SilentlyContinue | Select-Object -First 3 -ExpandProperty FullName
```

If a command prints an error, paste it anyway; it is information.

- [ ] **Step 2: Compare against the assumptions and decide**

| Assumption in this plan | If the output disagrees |
|---|---|
| Default home is `C:\Users\<name>\.dsh` (or `DSH_HOME` when set and non-blank) | Change `resolveHome` in Task 1 to match what the app really uses |
| `<home>\sessions\<project folder>\<uuid>\` | Rewrite `sessionDirsFor` for the real layout before Task 2 |
| Bundled CLI is `...\resources\runtime\cli\bin\dsh.cmd` | Use the real path in Task 6 and the README |
| `pwsh` is installed (PowerShell 7) | Note it as a Windows requirement in the README; if only Windows PowerShell 5 exists, check in Task 6 whether the `pwsh` tool still mounts |
| Node is available for `test/manual` | Use the app's bundled Node (Electron) or install Node LTS; record which one worked |

Stop and re-plan Tasks 1 and 2 if the layout row disagrees; the rest of the plan holds either way.

---

### Task 1: Pure path helpers with tests

**Files:**
- Create: `lib/paths.js`
- Create: `test/paths.test.mjs`
- Modify: `package.json` (scripts)

**Interfaces:**
- Produces: `resolveHome(env?: Record<string,string|undefined>, home?: string): string`, `isUsableCwd(p: unknown, pathApi?: typeof import("node:path")): boolean`, `sessionDirsFor(sessionId: string, home: string): string[]`.

- [ ] **Step 1: Write the failing tests**

Create `test/paths.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import path from "node:path";
import { mkdtempSync, mkdirSync, rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { resolveHome, isUsableCwd, sessionDirsFor } from "../lib/paths.js";

test("resolveHome: explicit DSH_HOME wins, blank falls back to ~/.dsh", () => {
	assert.equal(resolveHome({ DSH_HOME: "/data/dsh" }, "/home/u"), "/data/dsh");
	assert.equal(resolveHome({}, "/home/u"), path.join("/home/u", ".dsh"));
	assert.equal(resolveHome({ DSH_HOME: "" }, "/home/u"), path.join("/home/u", ".dsh"));
	assert.equal(resolveHome({ DSH_HOME: "   " }, "/home/u"), path.join("/home/u", ".dsh"));
});

test("isUsableCwd: absolute only, no parent segments, per platform", () => {
	assert.equal(isUsableCwd("/Users/a/proj", path.posix), true);
	assert.equal(isUsableCwd("C:\\Users\\a\\proj", path.win32), true);
	assert.equal(isUsableCwd("C:\\Users\\a\\proj", path.posix), false);
	assert.equal(isUsableCwd("relative/dir", path.posix), false);
	assert.equal(isUsableCwd("/Users/a/../b", path.posix), false);
	assert.equal(isUsableCwd("C:\\Users\\..\\b", path.win32), false);
	assert.equal(isUsableCwd("/Users/a/my..dir", path.posix), true);
	assert.equal(isUsableCwd(undefined, path.posix), false);
});

test("sessionDirsFor: finds the id under any project directory, ignores bad ids", () => {
	const home = mkdtempSync(path.join(tmpdir(), "btw-paths-"));
	try {
		const id = "6f1c2d3e-aaaa-4bbb-8ccc-1234567890ab";
		mkdirSync(path.join(home, "sessions", "--C~003A-Users-a-proj--", id), { recursive: true });
		mkdirSync(path.join(home, "sessions", "--other--", "different-id"), { recursive: true });
		assert.deepEqual(sessionDirsFor(id, home), [path.join(home, "sessions", "--C~003A-Users-a-proj--", id)]);
		assert.deepEqual(sessionDirsFor("../../etc", home), []);
		assert.deepEqual(sessionDirsFor(id, path.join(home, "nope")), []);
	} finally {
		rmSync(home, { recursive: true, force: true });
	}
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /Users/ilyaashirov/Documents/GitHub/dsh-btw-desktop && node --test test/`
Expected: FAIL, `Cannot find module '../lib/paths.js'`.

- [ ] **Step 3: Write the implementation**

Create `lib/paths.js`:

```js
/**
 * Platform-dependent path decisions, kept pure so they can be tested for
 * Windows from any OS (`path.win32`). Zero dependencies.
 */
import { existsSync, readdirSync } from "node:fs";
import { homedir } from "node:os";
import path from "node:path";

const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

/** DSH home: `$DSH_HOME` unless blank, else `~/.dsh` (same rule as DSH's own home-paths package). */
export function resolveHome(env = process.env, home = homedir()) {
	const configured = env.DSH_HOME;
	return typeof configured === "string" && configured.trim() !== "" ? configured : path.join(home, ".dsh");
}

/** An absolute directory path without `..` segments, judged by the given platform's path rules. */
export function isUsableCwd(p, pathApi = path) {
	return typeof p === "string" && pathApi.isAbsolute(p) && !p.split(/[\\/]/).includes("..");
}

/**
 * Directories holding one session: `<home>/sessions/<project>/<sessionId>`.
 * Scans the project directories instead of re-deriving DSH's per-cwd folder name,
 * which differs per platform. Only UUID ids are accepted (they are never escaped by DSH).
 */
export function sessionDirsFor(sessionId, home) {
	if (!UUID.test(sessionId)) return [];
	const root = path.join(home, "sessions");
	let projects;
	try {
		projects = readdirSync(root);
	} catch {
		return [];
	}
	return projects.map((project) => path.join(root, project, sessionId)).filter((dir) => existsSync(dir));
}
```

In `package.json` add the script next to `check`:

```json
"test": "node --test test/"
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: 3 tests pass.

- [ ] **Step 5: Commit (only when the user says so)**

```bash
git add lib/paths.js test/paths.test.mjs package.json
git commit -m "add pure path helpers with Windows-aware tests"
```

---

### Task 2: Use the helpers; stop deriving DSH's folder name

**Files:**
- Modify: `lib/index.js:31-45` (imports, `DSH_HOME`), `lib/index.js:59-106` (delete `encodeSegmentChar`, `projectKey`, `sessionDir`; rewrite `burnSession`), `lib/index.js:142` and `:357` (call sites), `lib/index.js:372` (cwd check)

**Interfaces:**
- Consumes: `resolveHome`, `isUsableCwd`, `sessionDirsFor` from `lib/paths.js`.
- Produces: `burnSession(sessionId: string): void` (the `cwd` parameter is gone).

- [ ] **Step 1: Switch imports and home**

Replace lines 31-34 and 44 so the top reads:

```js
import { existsSync, readFileSync, rmSync, statSync, writeFileSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";
import { randomUUID } from "node:crypto";
import { isUsableCwd, resolveHome, sessionDirsFor } from "./paths.js";
```

```js
const DSH_HOME = resolveHome();
```

If `existsSync` is no longer used after Step 2, remove it from the import.

- [ ] **Step 2: Replace the session-directory block**

Delete `encodeSegmentChar`, `projectKey`, `sessionDir` and the old `burnSession` (lines 59-106) and put this in their place:

```js
// ── session-directory cleanup ──

/**
 * Remove one temp session's own directory, retrying: the persistence layer may
 * still be flushing events after dispose, and on Windows a held lock file makes
 * the delete fail (EBUSY).
 */
function burnSession(sessionId) {
	let attempts = 0;
	const tryBurn = () => {
		try {
			for (const dir of sessionDirsFor(sessionId, DSH_HOME)) {
				rmSync(dir, { recursive: true, force: true, maxRetries: 5, retryDelay: 100 });
			}
		} catch {
			// retried below; never block the flow on a stubborn directory
		}
		if (attempts < 3 && sessionDirsFor(sessionId, DSH_HOME).length > 0) {
			attempts += 1;
			setTimeout(tryBurn, 400);
		}
	};
	setTimeout(tryBurn, 300);
}
```

- [ ] **Step 3: Update the call sites and the cwd check**

`sweepMarker`: `for (const e of stale) burnSession(e.id);`
`burnTemp`: `burnSession(sessionId);`
`handleOpen`: replace `payload.cwd.startsWith("/") && !payload.cwd.includes("..")` with `isUsableCwd(payload?.cwd)`:

```js
if (isUsableCwd(payload?.cwd)) {
	try {
		if (statSync(payload.cwd).isDirectory()) cwd = payload.cwd;
	} catch { /* fall back to home */ }
}
```

Update the comment at the top of `lib/index.js` that mentions "projectKey mirrored from dsh persistence" if present.

- [ ] **Step 4: Syntax check and unit tests**

Run: `npm run check && npm test`
Expected: both pass; `grep -n "projectKey\|sessionDir(" lib/index.js` prints nothing.

- [ ] **Step 5: macOS end-to-end probe (cleanup must still work)**

Use the isolated scratch home, never the user's profile:

```bash
S=/private/tmp/claude-501/-Users-ilyaashirov-Documents-GitHub-dsh-btw-desktop/e6266124-62fa-41c3-88d3-d6d2113d94bd/scratchpad/dshhome
P=/private/tmp/claude-501/-Users-ilyaashirov-Documents-GitHub-dsh-btw-desktop/85608db6-bcbc-4d7b-9110-c651d23122e5/scratchpad
node $P/mock-llm.mjs $P/mock-requests.log & $P/runprobe.sh 8840
```

Expected in the probe output: `open -> ok`, `send -> MOCK REPLY from the child.`, `close -> burned: true`, `after close (+4s): marker []`, and `ls "$S/sessions"/*/` shows no leftover session id. Stop the mock afterwards (`pkill -f mock-llm.mjs`).

- [ ] **Step 6: Commit (only when the user says so)**

```bash
git add lib/index.js
git commit -m "find temp session directories by scanning instead of mirroring DSH's folder naming"
```

---

### Task 3: Platform-neutral folder name in the tab header

**Files:**
- Modify: `lib/client.js:574`

- [ ] **Step 1: Change the split**

Replace

```js
const cwdName = s.cwd !== null ? s.cwd.split("/").filter(Boolean).pop() : null;
```

with

```js
const cwdName = s.cwd !== null ? s.cwd.split(/[\\/]/).filter(Boolean).pop() : null;
```

- [ ] **Step 2: Check**

Run: `npm run check`
Expected: passes. In a node REPL: `"C:\\Users\\a\\proj".split(/[\\/]/).filter(Boolean).pop()` returns `proj`, and `"/Users/a/proj"` returns `proj`.

- [ ] **Step 3: Commit (only when the user says so)**

```bash
git add lib/client.js
git commit -m "show the project folder name for Windows paths too"
```

---

### Task 4: Shell tool per platform in the preset

**Files:**
- Modify: `cordis.yml` (the `tool-bash` row of the `btw-temp` preset)

- [ ] **Step 1: Replace the tool-bash row**

```yaml
          - id: tool-bash
            name: '@deepseek-ai/dsh-tool-bash'
            disabled: !!js process.platform === 'win32'
          - id: tool-pwsh
            name: '@deepseek-ai/dsh-tool-pwsh'
            disabled: !!js process.platform !== 'win32'
```

This is the same pair the shipped Standard preset uses, so the child gets exactly one shell tool.

- [ ] **Step 2: macOS check that nothing changed**

Run the Task 2 Step 5 probe, then inspect the first request the mock logged:

```bash
node -e 'const t=require("fs").readFileSync(process.argv[1],"utf8").split("\n\n")[0].split("\n").slice(1).join("\n");console.log(JSON.parse(t).tools.map(x=>x.function.name))' $P/mock-requests.log
```

Expected: a list that contains `bash` and not `pwsh`.

- [ ] **Step 3: Commit (only when the user says so)**

```bash
git add cordis.yml
git commit -m "pick bash or pwsh for the temporary chat preset by platform"
```

---

### Task 5: Manual cross-platform test kit

**Files:**
- Create: `test/manual/probe-temp.mjs` (copy of the scratchpad probe, with the home path and token as arguments)
- Create: `test/manual/mock-llm.mjs` (copy of the scratchpad mock)
- Create: `test/manual/README.md`

- [ ] **Step 1: Copy the scripts**

```bash
mkdir -p test/manual
cp /private/tmp/claude-501/-Users-ilyaashirov-Documents-GitHub-dsh-btw-desktop/85608db6-bcbc-4d7b-9110-c651d23122e5/scratchpad/probe-temp.mjs test/manual/probe-temp.mjs
cp /private/tmp/claude-501/-Users-ilyaashirov-Documents-GitHub-dsh-btw-desktop/85608db6-bcbc-4d7b-9110-c651d23122e5/scratchpad/mock-llm.mjs test/manual/mock-llm.mjs
```

Edit `probe-temp.mjs` so the working directory sent to `open` is `process.cwd()` (already so) and the mock provider id and model are `mock` and `mock-1`.

- [ ] **Step 2: Write `test/manual/README.md`**

The file holds the steps a tester runs, written for PowerShell first (Windows is the machine we have), with the POSIX equivalent beside each. Contents, in order:

1. Isolated home: `$env:DSH_HOME = "$env:TEMP\dsh-btw-test"` (POSIX: `export DSH_HOME=$(mktemp -d)`). This never touches the real `~/.dsh`.
2. Install into a throwaway profile with the app's CLI (`<path>\dsh.cmd` from Task 0): `& $dsh plugin --profile web add file:<absolute path to this repo>` (Windows path written with forward slashes if the CLI rejects backslashes; record which one worked).
3. Append the mock provider and default model to `$env:DSH_HOME\profiles\web\cordis.patch.yml`:

```yaml
- id: llm-pi-ai
  name: "@deepseek-ai/dsh-llm-pi-ai"
  config:
    providers:
      mock:
        displayName: Mock
        api: openai-completions
        baseURL: http://127.0.0.1:8899/v1
        models:
          - id: mock-1
            name: Mock
            contextWindow: 32000
            maxTokens: 2000
            input:
              - text
        apiKeyEnv: MOCK_KEY
- id: agent-default-model
  name: "@deepseek-ai/dsh-agent-default-model"
  config:
    provider: mock
    model: mock-1
```

4. Start the mock in one window: `node test\manual\mock-llm.mjs $env:TEMP\mock-requests.log`.
5. Start DSH in another: `$env:MOCK_KEY = "x"; & $dsh --profile web --no-open --port 8850`, and copy the `?token=` value from the printed URL.
6. Run the probe in a third window: `node test\manual\probe-temp.mjs http://127.0.0.1:8850 <token> $env:DSH_HOME`.
7. Expected output: `open -> ok`, `send -> {"output":"MOCK REPLY from the child."}`, `close -> {"burned":true}`, and after about 4 seconds `marker []` with no session folder left under `$env:DSH_HOME\sessions`.
8. Failure signatures: `open -> btw.openFailed|...` (preset or shell tool did not mount), an empty `output` (the mock was not reached; check the log file), a leftover session folder or a non-empty marker (the burn failed, check for a locked file).
9. Optional UI check in a browser at the printed URL: open a session, click Temporary chat, send a message, close the tab.

- [ ] **Step 3: Check**

Run: `node --check test/manual/probe-temp.mjs && node --check test/manual/mock-llm.mjs`
Expected: both pass. `npm pack --dry-run` lists no `test/` files.

- [ ] **Step 4: Commit (only when the user says so)**

```bash
git add test/manual
git commit -m "add the end-to-end probe and mock model for other platforms"
```

---

### Task 6: Windows verification run (the user runs, Claude reads the output)

**Files:** none changed unless a step fails.

**Interfaces:**
- Consumes: Tasks 0-5 merged in the working tree; `test/manual/README.md` steps 1-8.
- Produces: a pass/fail table per check, pasted by the user, that Task 7 turns into README claims.

- [ ] **Step 1: Baseline before the changes (optional, 5 minutes)**

On the unchanged `main` (commit `abf0127`), run steps 1-8 of `test/manual/README.md`. Expected: `open` works only if the paths assumptions hold; a failure here (for example `btw.openFailed|...tool-bash...` or a leftover session folder) is the bug the plan fixes, and proves Task 2 and Task 4 were needed. Skip it if the user prefers to test once, after the changes.

- [ ] **Step 2: Run with the changes**

Run steps 1-8 of `test/manual/README.md` again with the working tree of this plan. The user pastes the probe output and, if anything failed, the DSH console window text.

- [ ] **Step 3: Fill in the result table**

| Check | Expected on Windows | Result (user) |
|---|---|---|
| `npm test` (path rules, `path.win32` cases) | pass | |
| Plugin installs with `dsh.cmd` | `+ dsh-btw` line | |
| `open` | ok, `cwd` shows a `C:\...` path | |
| Child tools | the mock's logged request lists `pwsh`, not `bash` | |
| `send` | `MOCK REPLY from the child.` | |
| `close` then +4 s | marker `[]`, no folder named with the session id under `<home>\sessions` | |
| UI: Start-page card and header button | both open the side tab, header shows the folder name (not the full path) | |
| UI: language switch to 中文 and back | strings and tab title follow | |
| App-level: plugin loads inside the real Desktop app after `plugin --profile desktop add` (app fully quit first) | `/btw` panel and the side tab work | |

- [ ] **Step 4: Decide**

Every row passes: go to Task 7 and claim Windows. A row fails: use the pasted error, fix with one change at a time (systematic debugging), repeat Step 2. Do not edit the README until then.

---

### Task 7: README install steps and platform status

**Files:**
- Modify: `README.md` (Install and Status sections)

- [ ] **Step 1: Fix the install instructions**

State, from the Desktop docs: launch Desktop once so the profile exists, fully quit the app, then run the bundled CLI (`resources/runtime/cli/bin/dsh`, `dsh.cmd` on Windows; use the real path found in Task 0) with `plugin --profile desktop add ...`, then reopen Desktop. Keep the macOS app-bundle path as the macOS example only.

- [ ] **Step 2: Write the platform status from the Task 6 table**

Replace "Tested on macOS only. The CLI path above is the macOS bundle path; Windows and Linux are untested." with exactly what Task 6 showed: macOS tested end to end; Windows tested on the user's machine for the rows that passed (name the Windows version the user reports); Linux not run, same code paths, `test/manual/README.md` explains how to run the probe. Add the requirement found in Task 0 (for example PowerShell 7 for the child's shell tool).

- [ ] **Step 3: Check**

Run: `grep -n "macOS\|Windows\|Linux" README.md`
Expected: every claim matches a passing row in the Task 6 table, and no sentence claims Linux was run.

- [ ] **Step 4: Commit (only when the user says so)**

```bash
git add README.md
git commit -m "document install order and verified platforms"
```

---

## Self-Review

- **Spec coverage:** the eight items from the earlier answer map to tasks: path check (T2), session-folder naming (T2, confirmed first in T0), DSH home (T1, T2, confirmed in T0), file deletion on Windows (T2 `maxRetries` plus retry on error, exercised in T6), orphan pid check (no change, observed in T6), project-name display (T3), bash vs pwsh (T4, exercised in T6), install and CLI path (T0 finds it, T7 documents it). The user's Windows machine is used in T0 (discovery), T6 (verification) and T7 (claims only after a pass).
- **Placeholder scan:** no TBD or "handle edge cases" steps; every code step shows the code. T0, T5 Step 2 and T6 are procedures with exact commands; the Result column in T6 is intentionally blank for the user to fill.
- **Type consistency:** `sessionDirsFor(sessionId, home)`, `resolveHome(env, home)`, `isUsableCwd(p, pathApi)` and `burnSession(sessionId)` match across T1 and T2.
- **Known gaps, deliberately left:** `process.kill(pid, 0)` for the orphan check is documented by Node as an existence test on all platforms, so no change; T6 observes it only indirectly (a crash-orphan test would need killing DSH mid-chat, add it if T6 reveals marker problems). The sessions root is assumed to be `<home>\sessions` until T0 confirms it. Linux stays untested.
