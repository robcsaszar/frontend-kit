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

## Releasing

Releases are cut by the **Release** workflow (`.github/workflows/release.yml`), never by hand. It is a manual `workflow_dispatch` with one input, `tag`, and it releases the commit at the tip of the branch it is run on. Run it on `main`.

The workflow, in order:
1. Rejects a tag that is not `vX.Y.Z`, or that already exists.
2. Fails unless `version` in `.claude-plugin/plugin.json` and `plugins[0].version` in `.claude-plugin/marketplace.json` both equal `X.Y.Z`.
3. Takes the `## [X.Y.Z]` block from `CHANGELOG.md` as the release notes, and fails if there is none.
4. Creates the tag at the checked-out commit and publishes the GitHub release with those notes.

Nothing is created until every check passes, so a failed run leaves nothing to clean up.

To prepare a release, in one PR:
- Set the same new version in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.
- Add a `## [X.Y.Z] - YYYY-MM-DD` block at the top of `CHANGELOG.md`, and its `[X.Y.Z]: …` link reference at the bottom.
- Merge to `main`.

Then run the workflow on `main` with `tag=vX.Y.Z`, from the Actions tab (**Release → Run workflow**) or from a shell:

```sh
gh workflow run release.yml --ref main -f tag=vX.Y.Z
```

Afterwards, confirm the release exists and its notes match the changelog block.

NEVER:
- Never push a tag or create a release outside the workflow. A hand-made tag makes the workflow refuse that version, and a hand-written release skips the changelog and version checks.
- Never work around a failed run by hand-writing notes or skipping a check. Fix the changelog or the manifests, merge, and re-run.
