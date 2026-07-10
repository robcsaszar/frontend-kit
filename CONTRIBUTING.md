# Contributing

Thanks for considering a contribution to `frontend-kit`. This is a small collection of independent frontend skills. Contributions are welcome, but the bar is "does this fit the collection's shape and stay generically useful," not "is this a good idea in general."

## Before you start

- **Bug in an existing skill's guidance?** Open an issue with a concrete example: what the skill got wrong or missed.
- **New skill idea?** Open an issue first. Each skill in this collection is self-contained and covers one frontend implementation area; a new skill needs a clear, non-overlapping scope.
- **Design questions** (should this be a new skill vs. folded into an existing one, does this belong in `references/` vs. the main `SKILL.md`) are worth raising as an issue before writing code.

## Making a change

1. Fork and branch from `main`.
2. Keep each skill self-contained; don't introduce cross-skill dependencies.
3. If you're adding a new skill, make sure it reads as generic guidance, not tied to one project's stack or conventions, before opening a PR.
4. Update `README.md`'s skill table in the same change.

## Quality bar

Every skill in this collection should stay useful outside the project it was originally built for. Keeping parity with how the skill is actually used day to day is the practical test: guidance that's gone stale relative to current library APIs or best practices is a bug, same as an outright error.

## What won't be merged

- Project-specific assumptions leaking into a skill's guidance: these skills need to read as generic frontend advice, not one project's dogfood notes.
- New skills without a prior issue discussing scope and overlap with existing ones.

## Questions

Open an issue. There's no separate chat or forum for this project.
