# SkillManager Workspace Install Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `SkillManager({ action: "install_from_workspace", path })` so the agent can install a complete VFS skill directory or `SKILL.md` file into the Skills panel.

**Architecture:** Keep this as a focused extension to the existing single-file skill system in `onepagent.html`. Add one new SkillManager action, one workspace importer helper that reuses `_collectVfsFiles` and `_putSkillAsset`, then route the result through the existing validation, risk, confirmation, persistence, render, and mount pipeline.

**Tech Stack:** Plain browser JavaScript in `onepagent.html`, OnePagent VFS helpers, localStorage skill metadata, IndexedDB skill file storage, manual browser validation.

---

## File structure

- Modify: `onepagent.html`
  - `SKILL_MANAGER_WRITE_ACTIONS`: add `install_from_workspace`.
  - `BASE_TOOLS_ANTHROPIC` SkillManager schema: add the action enum and clarify the `path` field supports workspace path for this action.
  - Skill import helper area near `fetchSkillFromGithubOptions`: add `skillFromWorkspacePath(path)`.
  - `toolSkillManager`: add the `install_from_workspace` branch.
- No automated test files exist for this single-file app. Validation is manual in the browser through the existing Agent tools and VFS panels.

---

### Task 1: Register the new SkillManager action

**Files:**
- Modify: `onepagent.html` around `SKILL_MANAGER_WRITE_ACTIONS` near line 6450.
- Modify: `onepagent.html` SkillManager tool schema near line 10410.

- [ ] **Step 1: Update the write-action set**

Change the SkillManager write actions from:

```js
const SKILL_MANAGER_WRITE_ACTIONS = new Set(['create', 'install_from_markdown', 'install_from_json', 'install_from_github', 'update', 'set_active', 'remove']);
```

to:

```js
const SKILL_MANAGER_WRITE_ACTIONS = new Set(['create', 'install_from_markdown', 'install_from_json', 'install_from_github', 'install_from_workspace', 'update', 'set_active', 'remove']);
```

- [ ] **Step 2: Update the tool schema enum and descriptions**

In the `SkillManager` entry inside `BASE_TOOLS_ANTHROPIC`, replace the current schema text for `action` and `path` with the following complete versions:

```js
action: { type: 'string', enum: ['list', 'inspect', 'create', 'install_from_markdown', 'install_from_json', 'install_from_github', 'install_from_workspace', 'update', 'set_active', 'remove'], description: 'Skill management action.' }
```

```js
path: { type: 'string', description: 'For install_from_github: path to the skill folder in the repository. For install_from_workspace: VFS path to a skill directory or its SKILL.md file.' }
```

Also update the SkillManager description from:

```js
'Manage installed skills in the left Skills panel. Supports listing, inspecting, creating, installing from SKILL.md, JSON, or a GitHub skill folder URL, updating, enabling/disabling, and removing skills. Write actions modify persisted skills and may require confirmed=true for risky operations such as delete, overwrite, executable handlers, tools, scripts, or remote install.'
```

to:

```js
'Manage installed skills in the left Skills panel. Supports listing, inspecting, creating, installing from SKILL.md text, JSON, GitHub skill folder URLs, or current workspace VFS paths, updating, enabling/disabling, and removing skills. Write actions modify persisted skills and may require confirmed=true for risky operations such as delete, overwrite, executable handlers, tools, scripts, remote install, or workspace install.'
```

- [ ] **Step 3: Smoke-check syntax by searching the schema**

Run:

```bash
grep -n "install_from_workspace" onepagent.html
```

Expected: at least two matches: one in `SKILL_MANAGER_WRITE_ACTIONS`, one in the SkillManager action enum.

- [ ] **Step 4: Commit Task 1**

```bash
git add onepagent.html
git commit -m "feat(skills): register workspace install action"
```

---

### Task 2: Add the workspace-to-skill importer helper

**Files:**
- Modify: `onepagent.html` near `_putSkillAsset` / GitHub skill import helpers, around line 7080.

- [ ] **Step 1: Add helper code after `_putSkillAsset`**

Insert this function immediately after `_putSkillAsset`:

```js
async function skillFromWorkspacePath(path) {
  const requestedPath = String(path || '').trim();
  if (!requestedPath) throw new Error('path is required.');

  const resolvedPath = normPath(requestedPath);
  const node = vfsResolve(resolvedPath);
  if (!node) throw new Error(`Workspace path not found: ${resolvedPath}`);

  let rootPath = resolvedPath;
  let manifestPath = resolvedPath;
  if (node.type === 'dir') {
    manifestPath = normPath(`${resolvedPath}/SKILL.md`);
    const manifestNode = vfsResolve(manifestPath);
    if (!manifestNode) throw new Error(`No SKILL.md found in workspace directory: ${resolvedPath}`);
    if (manifestNode.type !== 'file') throw new Error(`SKILL.md is not a file: ${manifestPath}`);
  } else if (node.type === 'file') {
    if (_pathBaseName(resolvedPath) !== 'SKILL.md') throw new Error('Workspace file path must point to SKILL.md.');
    rootPath = _pathDirName(resolvedPath);
  } else {
    throw new Error(`Unsupported workspace path type: ${resolvedPath}`);
  }

  const manifestStat = vfsStat(manifestPath);
  if (!manifestStat || manifestStat.type !== 'file') throw new Error(`Workspace manifest not found: ${manifestPath}`);
  if (manifestStat.binary) throw new Error(`Workspace manifest must be text: ${manifestPath}`);

  const manifest = vfsRead(manifestPath);
  if (manifest.error) throw new Error(`Failed to read workspace manifest: ${manifest.error}`);

  const skill = skillFromMd(manifest.content, `workspace:${rootPath}`);
  if (!skill.references) skill.references = {};
  if (!skill.scripts) skill.scripts = {};
  if (!skill.files) skill.files = {};
  if (!skill.binaryFiles) skill.binaryFiles = {};

  const { files } = await _collectVfsFiles(rootPath);
  for (const file of files.sort((a, b) => a.rel.localeCompare(b.rel))) {
    if (file.path === manifestPath || file.rel === 'SKILL.md') continue;
    if (file.binary) {
      _putSkillAsset(skill, file.rel, file.bytes, true);
    } else {
      _putSkillAsset(skill, file.rel, new TextEncoder().encode(file.content || ''), false);
    }
  }

  skill.source = `workspace:${rootPath}`;
  return { skill, root: rootPath, manifest: manifestPath };
}
```

- [ ] **Step 2: Verify helper uses existing utilities only**

Check these functions already exist in `onepagent.html` before the new helper is called at runtime:

```bash
grep -n "function _pathBaseName\|function _pathDirName\|function normPath\|function vfsResolve\|async function _collectVfsFiles\|function _putSkillAsset" onepagent.html
```

Expected: all six function names are present.

- [ ] **Step 3: Commit Task 2**

```bash
git add onepagent.html
git commit -m "feat(skills): collect workspace skill assets"
```

---

### Task 3: Wire the importer into SkillManager

**Files:**
- Modify: `onepagent.html` inside `toolSkillManager`, after the `install_from_github` branch and before `update`.

- [ ] **Step 1: Add the `install_from_workspace` branch**

Insert this branch after the existing `install_from_github` block:

```js
  if (action === 'install_from_workspace') {
    let imported;
    try {
      imported = await skillFromWorkspacePath(input.path);
    } catch (e) {
      return skillManagerError(action, e?.message || String(e));
    }
    const { skill, root, manifest } = imported;
    normalizeSkill(skill, { source: `workspace:${root}` });
    const oldSkill = skills.find(s => s.name === skill.name);
    const errors = validateSkillInput(skill, { operation: 'install' });
    if (errors.length) return skillManagerError(action, 'Skill validation failed.', { errors, source: { root, manifest } });
    const risk = assessSkillRisk(skill, 'install', oldSkill);
    if (input.dry_run) return skillManagerResult({ ok: true, action, dry_run: true, source: { root, manifest }, skill: serializeSkillForTool(skill, { detail: true, includeBody: true }), overwrite: !!oldSkill, risk });
    if (requiresSkillManagerConfirmation(risk, 'install') && input.confirmed !== true) return skillManagerResult({ ok: false, action, needs_confirmation: true, source: { root, manifest }, overwrite: !!oldSkill, risk, summary: `Installing workspace skill "${skill.name}" requires confirmation.` });
    installSkill(skill);
    return skillManagerResult({ ok: true, action, summary: `Installed skill "${skill.name}" from workspace.`, source: { root, manifest }, skill: serializeSkillForTool(skill), warnings: risk.reasons });
  }
```

- [ ] **Step 2: Check branch placement**

Run:

```bash
grep -n "install_from_github\|install_from_workspace\|if (action === 'update')" onepagent.html
```

Expected: `install_from_workspace` appears between the GitHub install branch and the update branch.

- [ ] **Step 3: Commit Task 3**

```bash
git add onepagent.html
git commit -m "feat(skills): install workspace skills via SkillManager"
```

---

### Task 4: Manual browser validation

**Files:**
- Modify only if validation reveals bugs: `onepagent.html`.
- No commit if no code changes are needed.

- [ ] **Step 1: Start a local static server**

Run:

```bash
python -m http.server 8000
```

Expected: server starts and serves the repository root. Leave it running while testing.

- [ ] **Step 2: Open OnePagent**

Open:

```text
http://localhost:8000/onepagent.html
```

Expected: OnePagent loads without a console syntax error.

- [ ] **Step 3: Create a workspace skill fixture through Agent tools or JSExec**

Use the app's agent tools or browser console-equivalent JS execution to create these VFS files:

```js
vfsWrite('/drafts/demo/SKILL.md', `---
name: demo
summary: Demo workspace skill
description: Demo workspace skill
---

Use this skill for workspace install validation.`, true);
vfsWrite('/drafts/demo/references/readme.md', '# Demo reference\n', true);
vfsWrite('/drafts/demo/scripts/run.py', 'print("demo")\n', true);
vfsWrite('/drafts/demo/assets/icon.svg', '<svg xmlns="http://www.w3.org/2000/svg"></svg>');
renderFileTree();
```

Expected: `/drafts/demo` appears in the Files tree.

- [ ] **Step 4: Dry-run install from directory path**

Ask the agent to call SkillManager with:

```json
{
  "action": "install_from_workspace",
  "path": "/drafts/demo",
  "dry_run": true
}
```

Expected output includes:

```json
{
  "ok": true,
  "action": "install_from_workspace",
  "dry_run": true,
  "source": {
    "root": "/drafts/demo",
    "manifest": "/drafts/demo/SKILL.md"
  },
  "overwrite": false
}
```

Also verify the serialized skill has `name: "demo"`, `referenceCount: 1`, `scriptCount: 1`, and `fileCount` at least `3`.

- [ ] **Step 5: Install from directory path**

Ask the agent to call SkillManager with:

```json
{
  "action": "install_from_workspace",
  "path": "/drafts/demo",
  "confirmed": true
}
```

Expected: output says `Installed skill "demo" from workspace.` and the left Skills panel shows `demo`.

- [ ] **Step 6: Verify mounted files**

Use Read or Bash inside the app to verify these paths:

```text
/skills/demo/SKILL.md
/skills/demo/references/readme.md
/skills/demo/scripts/run.py
/skills/demo/assets/icon.svg
```

Expected: all four paths exist. `SKILL.md` contains the parsed skill body; the other files contain the fixture contents.

- [ ] **Step 7: Dry-run install from SKILL.md file path**

Ask the agent to call SkillManager with:

```json
{
  "action": "install_from_workspace",
  "path": "/drafts/demo/SKILL.md",
  "dry_run": true
}
```

Expected: `source.root` is `/drafts/demo`, `source.manifest` is `/drafts/demo/SKILL.md`, and `overwrite` is `true` because `demo` is now installed.

- [ ] **Step 8: Verify non-SKILL.md file input fails**

Ask the agent to call SkillManager with:

```json
{
  "action": "install_from_workspace",
  "path": "/drafts/demo/references/readme.md",
  "dry_run": true
}
```

Expected: output includes:

```json
{
  "ok": false,
  "action": "install_from_workspace",
  "error": "Workspace file path must point to SKILL.md."
}
```

- [ ] **Step 9: Verify missing manifest fails**

Create a directory without a manifest:

```js
vfsWrite('/drafts/no-manifest/readme.md', 'no manifest');
```

Ask the agent to call SkillManager with:

```json
{
  "action": "install_from_workspace",
  "path": "/drafts/no-manifest",
  "dry_run": true
}
```

Expected: output includes `No SKILL.md found in workspace directory: /drafts/no-manifest`.

- [ ] **Step 10: Verify overwrite confirmation**

Ask the agent to call SkillManager with:

```json
{
  "action": "install_from_workspace",
  "path": "/drafts/demo"
}
```

Expected: output includes `needs_confirmation: true` and `overwrite: true`.

- [ ] **Step 11: Stop the local server**

Stop the `python -m http.server 8000` process with Ctrl+C.

- [ ] **Step 12: Commit any validation fixes**

If validation required code fixes, commit them:

```bash
git add onepagent.html
git commit -m "fix(skills): handle workspace install validation"
```

If no fixes were needed, do not create an empty commit.

---

### Task 5: Final review

**Files:**
- Inspect: `onepagent.html`
- Inspect: `docs/superpowers/specs/2026-05-13-skillmanager-workspace-install-design.md`

- [ ] **Step 1: Confirm implementation matches spec**

Check the spec requirements against the changed code:

```bash
grep -n "install_from_workspace\|skillFromWorkspacePath\|workspace:" onepagent.html
```

Expected: matches cover action registration, helper, schema, and dispatch.

- [ ] **Step 2: Confirm git state**

Run:

```bash
git status --short
```

Expected: no unexpected uncommitted files. If `docs/superpowers/plans/2026-05-13-skillmanager-workspace-install.md` is uncommitted, commit it separately with:

```bash
git add docs/superpowers/plans/2026-05-13-skillmanager-workspace-install.md
git commit -m "docs: add SkillManager workspace install plan"
```

- [ ] **Step 3: Report results**

Final report should include:

```text
Implemented SkillManager install_from_workspace.
Validated directory path install, SKILL.md file path install, missing manifest failure, non-SKILL.md failure, overwrite confirmation, and mounted /skills/<name>/ resources.
```

---

## Self-review notes

- Spec coverage: The plan covers the new action, directory and `SKILL.md` path inputs, full asset collection, references/scripts mirroring, dry-run output, confirmation behavior, persistence through existing install flow, and manual browser validation.
- Placeholder scan: The plan contains concrete code, commands, and expected outcomes for every step.
- Type consistency: The action name is consistently `install_from_workspace`; the helper returns `{ skill, root, manifest }`; SkillManager responses use `source: { root, manifest }`.