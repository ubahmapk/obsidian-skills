---
name: sync-safe-file-names
description: Work in Obsidian vaults running the sync-safe-file-names plugin — expect automatic file renames, predict the safe name, or create files with safe names to begin with. Use when creating or renaming files or folders in an Obsidian vault, when a file you just created gets renamed or "disappears", when note names must be compatible across Windows, Linux, macOS and mobile sync, or when sanitizing filenames (characters like ? * : " < > |).
---

# Sync-safe file names

Obsidian vaults may run the [sync-safe-file-names](https://github.com/j-maas/sync-safe-file-names) plugin. It automatically renames files and folders so their names can sync across Windows, Linux, macOS and mobile. **Renaming is the plugin working correctly, not an error.**

## What the plugin does

- Listens to the vault's `create` and `rename` events and renames unsafe names ~100ms later — while Obsidian is open. Files created while Obsidian is closed are renamed on next launch.
- Applies to **every** creation path: direct filesystem writes, the `obsidian` CLI, MCP servers, templates. Folders too.
- Never overwrites an existing file: if the safe name collides with an existing file, the rename silently fails and the unsafe name persists.
- Adds the original name as a frontmatter alias on rename — except on fresh notes that have no `aliases` array yet (v1.1.0 bug: the alias is dropped). If the original name matters for linking, add the alias yourself.

When a file you created gets renamed:

1. Do **not** recreate the old-named file — it exists under the new name.
2. Do **not** report the rename as an error or unexpected event.
3. Re-resolve the path (glob the directory for the sanitized name) and continue.

## Detecting the plugin and its per-vault policy

```bash
# installed?
grep sync-safe-file-names .obsidian/community-plugins.json
# exact settings for this vault:
cat .obsidian/plugins/sync-safe-file-names/data.json
```

| Setting | Meaning |
|---|---|
| `renameAutomatically` | auto-rename on create/rename events (default `true`) |
| `forceLowercase` | lowercases the entire name |
| `additionalCharacters` | extra allowed characters beyond the base set below |
| `addOriginalAlias` | add original name as frontmatter alias on rename |

## The safe-name algorithm

Base allowed set: `a-z A-Z 0-9 . _ -` and space.

1. If `forceLowercase`: lowercase the whole name.
2. `–` and `—` → `-`.
3. `’` → `'`, `“` and `”` → `"`.
4. Every character **not** in (base set + `additionalCharacters`) → `-` — one hyphen per character, no collapsing.
5. Trim leading/trailing whitespace.

Only the basename is sanitized; the extension is preserved.

## Create safe names proactively

Before creating any file in a vault with this plugin, sanitize the name yourself so no rename ever happens. Exact port of the plugin's algorithm:

```bash
node -e '
const [raw, extra = "", lower = "false"] = process.argv.slice(1);
const re = new RegExp("[^-a-zA-Z0-9._ " + extra.replace(/[\[\]]/g, "") + "]", "g");
let s = lower === "true" ? raw.toLocaleLowerCase() : raw;
s = s.replace(/[–—]/g, "-").replace(/’/g, "\u0027")
     .replace(/[“”]/g, "\u0022").replace(re, "-").trim();
console.log(s);
' 'My: Unsafe* Name?' '+(),$€' true
```

Pass `extra` and `lower` from the vault's `data.json` (`additionalCharacters`, `forceLowercase`). For ASCII-only names, `tr -c 'a-zA-Z0-9._ -' '-'` approximates step 4 (per-byte, so multi-byte characters yield multiple hyphens).

Note: apostrophes and quotes are only preserved if present in `additionalCharacters` — some vaults exclude them, turning `John's Note` into `John- s Note` (leading space kept, apostrophe → hyphen). Check `data.json` before assuming.