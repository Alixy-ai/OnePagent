# SkillManager workspace install design

## Goal

Allow the agent to install a skill from the current OnePagent virtual workspace into the left Skills panel by calling `SkillManager` with a workspace path. The path may point to a skill directory or directly to its `SKILL.md`; in both cases, `SKILL.md` is required.

## Chosen approach

Add a dedicated SkillManager action: `install_from_workspace`.

Example:

```json
{
  "action": "install_from_workspace",
  "path": "/drafts/my-skill",
  "dry_run": true,
  "confirmed": true
}
```

This keeps workspace installs separate from GitHub installs, where `path` already means repository subpath. Existing actions keep their current behavior.

## Architecture

Add `install_from_workspace` to the SkillManager write actions and tool schema in `onepagent.html`.

Add a helper such as `skillFromWorkspacePath(path)` that:

1. Normalizes and resolves the VFS path.
2. If the path is a directory, reads `<path>/SKILL.md`.
3. If the path is a file, accepts it only when the basename is `SKILL.md` and uses its parent as the skill root.
4. Parses `SKILL.md` with the existing `parseSkillMd` / `skillFromMd` flow.
5. Collects every file under the skill root using the existing VFS traversal helper.
6. Skips the root `SKILL.md` as an asset because it becomes the manifest/body.
7. Copies text assets to `skill.files` and binary assets to `skill.binaryFiles`.
8. Mirrors `references/*` into `skill.references` and `scripts/*` into `skill.scripts` for existing resource lookup compatibility.
9. Sets `source` to `workspace:<root>`.

The resulting skill then goes through the existing pipeline:

`normalizeSkill → validateSkillInput → assessSkillRisk → installSkill`

Installed skills remain persistent records, not live links to the source workspace directory. The original draft directory is left unchanged.

## Data flow

1. The agent calls `SkillManager({ action: "install_from_workspace", path })`.
2. `toolSkillManager` dispatches to the workspace importer.
3. The importer locates the root and manifest.
4. `SKILL.md` frontmatter becomes skill metadata; its Markdown body becomes `skill.body`.
5. Root assets are copied by relative path.
6. `dry_run` returns the parsed skill summary, resource counts, overwrite status, source root, and risk.
7. A real install validates, asks for confirmation when needed, saves metadata to localStorage, saves assets to IndexedDB, rebuilds tools, renders the Skills panel, and mounts the skill at `/skills/<skill-name>/`.

## Error handling

- Empty path: return `path is required.`
- Missing path: return `Workspace path not found: <path>`.
- File path not named `SKILL.md`: return `Workspace file path must point to SKILL.md.`
- Directory without `SKILL.md`: return `No SKILL.md found in workspace directory: <path>`.
- Binary or unreadable `SKILL.md`: return a clear manifest read error.
- Invalid skill name, resource path, tool definition, or conflicting tool name: reuse existing validation errors.
- Existing skill with the same name: require `confirmed: true` unless `dry_run` is set.
- Script files or executable handlers: reuse existing high-risk confirmation.

## Testing

Manual browser tests should cover:

1. Create `/drafts/demo/SKILL.md`, `/drafts/demo/references/readme.md`, `/drafts/demo/scripts/run.py`, and `/drafts/demo/assets/icon.svg` in the VFS.
2. Run `SkillManager` dry-run install from `/drafts/demo`; verify source, counts, overwrite flag, and risk.
3. Install for real; verify the left Skills panel shows the skill.
4. Verify `/skills/demo/SKILL.md`, `/skills/demo/references/readme.md`, `/skills/demo/scripts/run.py`, and `/skills/demo/assets/icon.svg` exist.
5. Inspect the skill and verify `referenceCount`, `scriptCount`, `fileCount`, and `binaryFileCount`.
6. Repeat using `/drafts/demo/SKILL.md` as the input path.
7. Verify failure paths for missing `SKILL.md`, non-`SKILL.md` file input, overwrite without confirmation, and script-risk confirmation.

## Scope

This design only adds Agent-driven workspace path installation through `SkillManager`. It does not add a Files-panel button or automatic install from selected files.