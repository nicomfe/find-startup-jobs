# Find Startup Jobs

Cursor plugin that will check a seeded list of European and remote-friendly tech startups for new jobs and rank them against your profile. This repository can hold more than one plugin. The only plugin right now is `plugins/find-startup-jobs`.

## Add it to Cursor

Cursor loads local plugins from `~/.cursor/plugins/local/<plugin-name>/`, and that folder must contain `.cursor-plugin/plugin.json`. Copy the plugin directory, not the repository root. Cursor skips a symlink that points outside `~/.cursor/plugins/local`.

```bash
mkdir -p ~/.cursor/plugins/local
cp -R plugins/find-startup-jobs ~/.cursor/plugins/local/find-startup-jobs
```

Restart Cursor, or run **Developer: Reload Window** from the Command Palette. Open **Customize** and confirm **Find Startup Jobs** is listed.

After you change the plugin, copy the folder again and reload.

On Enterprise, an admin has to allow local plugin imports. The setting is **Allow Local Plugin Imports** under Dashboard → Settings → Security & Identity → Marketplace and Plugins.

## Run it

In Agent chat:

```text
/find-startup-jobs
```

To limit results to jobs posted on or after a date:

```text
/find-startup-jobs 2026-10-20
```

The command is registered. The scan itself is not built yet, so a run only confirms the plugin is installed.

## Develop the skill (this repo)

While working in this repository, a project skill mirrors the plugin skill:

`.cursor/skills/find-startup-jobs/SKILL.md` → `plugins/find-startup-jobs/skills/find-startup-jobs/SKILL.md`

Edit that file (either path is the same content), then run `/find-startup-jobs` in Agent chat. You do not need to copy into `~/.cursor/plugins/local` for prompt changes. Reload the window if Cursor does not pick up edits.

If you also installed the plugin under `~/.cursor/plugins/local`, remove or rename that copy while iterating here so only one `find-startup-jobs` skill is active.

## Add another plugin

See [docs/add-a-plugin.md](docs/add-a-plugin.md). Check the repo with:

```bash
node scripts/validate-template.mjs
```
