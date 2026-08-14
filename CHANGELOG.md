# Changelog

All notable changes to this project are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/).

## [0.6.0] - 2026-08-14

### Changed

- `with-svg-animation`: rewritten from a technique tutorial into a pre-flight trap-surfacing router, 145 lines down to 58. Seven baseline probes (agents given SVG animation tasks without the skill) showed the model already knows every technique the old body explained, and on two points the old body was worse than no skill — its `transform-origin` entry omitted `transform-box: fill-box`, and its `<use>` warning gave no mechanism. What the probes did establish is that the model does not raise these conditions when the request is vague, so the body now asks what the request leaves unstated (root vs inner element, subpath count, stroked vs filled, how many elements animate) and documents only traps that a naive prompt provably misses.
- `with-svg-animation`: added the CSS-first escalation ladder to SMIL, WAAPI, and libraries, with the note that GSAP DrawSVG and MorphSVG are paid Club plugins and need licence confirmation before use.
- `with-svg-animation`: `references/techniques.md` gained a contents table with anchors for all 15 techniques, replacing the fragile "items 8-10" pointer in the body.

- `with-svelte`: split the seven oversized reference files into 25 topic files of roughly 50–270 lines each. References load whole, so a question about `$effect` previously pulled all 933 lines of `runes-reactivity.md`, including Svelte 4 migration and every anti-pattern; it now loads `runes-core.md` alone. No content was removed — all section headings are preserved verbatim.
- `with-svelte`: each reference now opens with a title, a one-line scope statement, and a contents list of its own sections, so a file's fit can be judged before reading it.
- `with-svelte`: the routing table now has one row per topic file, grouped by area (Runes, Template, Components, Routing, Data, Remote, Deploy, Tooling), replacing seven broad rows that each mapped several unrelated tasks onto one large file.

## [0.5.0] - 2026-07-10

### Added

- Initial release: with-canvas, with-drag-drop, with-svelte, and with-svg-animation skills.

[0.5.0]: https://github.com/robcsaszar/frontend-kit/releases/tag/v0.5.0
