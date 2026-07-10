# AGENTS.md

## Mission

This repo publishes the `frontend-kit` skill collection: four independent skills under `skills/<name>/`, each covering one frontend implementation area (canvas, drag-and-drop, Svelte 5 / SvelteKit, SVG animation). There is no build, no tests, no runtime of its own. The deliverable is the contents of `skills/*`. This repo's purpose is publishing copies of skills for others to install into their own projects, not authoring net-new guidance from scratch. Changes should be judged by: would a stranger who installs one of these skills into an unrelated project get correct, generically useful frontend guidance out of it?

## Layout convention

- One directory per skill under `skills/<name>/`, `name:` in frontmatter matching the directory name exactly.
- Each skill is self-contained: its own `SKILL.md` plus any `references/` it needs. No skill depends on another skill in this collection being installed.
- No skill in this collection ships a `scripts/` directory. If one is ever added, it needs a `SAFETY.md` at the repo root documenting what the script does, per the same convention used in this author's other skill repos.

## Judgment boundaries

NEVER:
- Never let a copied skill's content leak repo-specific references back to Orakl (or any other single project it may have been sourced from): file paths, project names, internal conventions. These skills need to read as generic frontend guidance, usable in any codebase.

ASK:
- Ask before removing a skill from the collection.
- Ask before changing the license or copyright holder.

ALWAYS:
- When adding, removing, or renaming a skill: update the table in [`README.md`](README.md) in the same change.

## Adding a skill

1. Copy the skill's full directory (`SKILL.md` plus `references/`, `assets/`, `scripts/` as applicable) into `skills/<name>/`, matching the frontmatter `name:` to the directory name.
2. Grep the copy for any project-specific leakage (source repo name, absolute paths, internal doc references) before publishing; flag anything found rather than silently editing the skill's content.
3. Add a row to the table in `README.md`.
4. If the skill ships a script, add a `SAFETY.md` at the repo root and link it from `README.md`.
