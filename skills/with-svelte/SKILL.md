---
name: with-svelte
description: "Modern Svelte 5 + SvelteKit guidance and routing. Use whenever creating, editing, reviewing, or debugging a Svelte component (.svelte) or module (.svelte.ts/.svelte.js), or any SvelteKit project. Covers runes ($state, $derived, $effect, $props, $bindable), template directives ({@attach}, {@render}, {@html}, snippets, keyed each), components (Bits UI, web components, forms), SvelteKit routing/layouts/error boundaries, data flow (load functions, form actions, +page.server.ts vs +page.ts, serialization, invalidateAll), remote functions (query/form/command/prerender in .remote.ts), motion (Spring, Tween, transitions, animate flip, reduced motion), deployment (adapters, Vite, pnpm, PWA, Cloudflare), and the @sveltejs/mcp CLI. Triggers are Svelte, SvelteKit, runes, .svelte, .remote.ts, +page, load function, form action, svelte/motion, svelte transitions, animate flip. Don't use for visual design critique, component-library selection, non-Svelte frameworks, or Spring Boot."
---

# with-svelte

Unified Svelte 5 + SvelteKit skill. This body is a router: apply the always-on core below, then **READ the one reference file** that matches the task before writing code. Load only the file(s) you need — not all of them.

## Always-on core (every Svelte task)

Write runes-mode Svelte 5. Never reach for a legacy feature that has a modern replacement:

- `$state` instead of implicit `let count = 0; count += 1`
- `$derived`/`$effect` instead of `$:` — and prefer `$derived` over `$effect` (effects are an escape hatch; never set state inside one)
- `$props` instead of `export let`, `$$props`, `$$restProps`
- `onclick={...}` instead of `on:click={...}`
- `{#snippet}`/`{@render}` instead of `<slot>`, `$$slots`, `<svelte:fragment>`
- `{@attach ...}` instead of `use:action`
- `<DynamicComponent>` instead of `<svelte:component this={...}>`; `import Self` instead of `<svelte:self>`
- classes with `$state` fields instead of stores; `createContext` instead of `setContext`/`getContext`
- clsx-style class arrays/objects instead of the `class:` directive
- `Spring`/`Tween` classes from `svelte/motion` instead of the deprecated `spring()`/`tweened()` stores
- keyed `{#each}` — never use the index as the key

When unsure of current syntax, do not guess — confirm via the `@sveltejs/mcp` CLI (see `references/tooling.md`) and run `svelte-autofixer` before finalizing any component.

## Routing table — READ the matching reference before writing

| Task involves… | MANDATORY READ |
|---|---|
| **Runes** — `$state`, `$derived`, `$effect`, choosing between them | `references/runes-core.md` |
| **Runes** — `$props`, `$bindable`, reactive class fields, `createSubscriber`, `$inspect` | `references/runes-props.md` |
| **Runes** — `await` in components, async reactivity, `hydratable` | `references/runes-async.md` |
| **Runes** — porting Svelte 4 syntax to runes mode | `references/runes-migration.md` |
| **Runes** — reactivity that compiles but behaves wrongly | `references/runes-antipatterns.md` |
| **Template** — `{@attach}`, migrating `use:` actions | `references/attachments.md` |
| **Template** — `{#snippet}` / `{@render}`, replacing slots | `references/snippets.md` |
| **Template** — `{@html}`, `{@const}`, `{@debug}`, keyed each, `bind:`, `<svelte:window>` | `references/template-tags.md` |
| **Components** — Bits/Ark/Melt UI, web components, custom elements | `references/component-libraries.md` |
| **Components** — forms inside components | `references/forms.md` |
| **Components** — CSS from JS, styling children, context | `references/styling-context.md` |
| **Motion** — `Spring`, `Tween`, `transition:`, `animate:flip`, easing, `prefersReducedMotion` | `references/motion.md` |
| **Routing** — file naming (`+page`/`+layout`/`+error`/`+server`), route groups, params | `references/routing-files.md` |
| **Routing** — nested layouts, layout groups, layout data | `references/layouts.md` |
| **Routing** — `+error.svelte`, expected vs unexpected errors | `references/error-handling.md` |
| **Routing** — `<svelte:boundary>` | `references/error-boundary.md` |
| **Routing** — SSR, hydration mismatches, browser-only work | `references/ssr-hydration.md` |
| **Data** — `load` functions, server vs universal, `depends` | `references/load-functions.md` |
| **Data** — form actions, progressive enhancement | `references/form-actions.md` |
| **Data** — `fail()` / `redirect()` / `error()` | `references/errors-redirects.md` |
| **Data** — serialization across the boundary, `invalidateAll()` | `references/serialization-invalidation.md` |
| **Remote** — `query()` / `form()` in `*.remote.ts`, schema validation | `references/remote-query-form.md` |
| **Remote** — `command()`, single-flight mutations, `prerender()`, `getRequestEvent()` | `references/remote-command-prerender.md` |
| **Deploy** — adapters, Vite/pnpm build setup | `references/deployment-adapters.md` |
| **Deploy** — publishing a Svelte library | `references/library-authoring.md` |
| **Deploy** — PWA setup, Cloudflare/streaming gotchas | `references/pwa-and-cloudflare.md` |
| **Tooling** — confirming syntax, looking up docs, validating/fixing code | `references/tooling.md` |

If a task spans areas (e.g. a form that uses runes + a server action), read each matching file. Do **not** load files outside the task's scope. If no reference covers the case, fetch authoritative docs via `references/tooling.md` rather than guessing.

## NEVER

- **NEVER set `$state` inside an `$effect` to compute a value**
  **Instead:** use `$derived` (or `$derived.by` for complex expressions).
  **Why:** effect-driven assignment creates extra render passes and update loops; `$derived` is glitch-free and runs lazily.

- **NEVER use an `$effect` to mirror a value into a `Spring`/`Tween`**
  **Instead:** `Spring.of(() => value)` / `Tween.of(...)` at component init, or assign `spring.target = value` in the handler that changed it. The exception is a set that needs per-call options `.of()` cannot pass — syncing to a measured size with `{ instant }` on the first measurement (see `references/motion.md`); an effect is correct there.
  **Why:** `.of()` already owns a render effect, so mirroring wraps one effect in another for no gain. But an effect is the only place a reactive read and a per-call option meet, and avoiding it there produces code that never re-runs.

- **NEVER guard effect/lifecycle code with `if (browser) {...}` to make it server-safe**
  **Instead:** effects already don't run on the server; for global listeners use `<svelte:window>`/`<svelte:document>`, and for browser-only setup use the right reference's SSR guidance.
  **Why:** the guard is dead code inside an effect and signals a misunderstanding that hides real hydration bugs.

- **NEVER return non-serializable values (class instances, functions, symbols) from a SvelteKit `load` or remote function**
  **Instead:** return plain JSON-serializable data; see `references/serialization-invalidation.md` / `references/remote-command-prerender.md`.
  **Why:** load uses JSON and remote functions use `devalue`; non-serializable returns fail silently or at runtime across the server→client boundary.

- **NEVER call `redirect()`/`error()` in SvelteKit without `throw`-ing them**
  **Instead:** `throw redirect(303, '/path')` / `throw error(404)`.
  **Why:** without `throw` execution continues and the navigation/error never happens.

- **NEVER guess current Svelte/SvelteKit API surface from memory for unfamiliar features**
  **Instead:** run `npx @sveltejs/mcp list-sections` + `get-documentation`, then `svelte-autofixer` (see `references/tooling.md`).
  **Why:** runes, remote functions, and async Svelte change fast and are version-gated; stale syntax compiles to subtly wrong behavior.
