# Svelte 4 to 5 Migration

Mapping legacy Svelte 4 constructs to their runes-mode replacements.

**Last verified:** 2026-03-12

**Contents**

- Svelte 4 → 5 Migration

---

## Svelte 4 → 5 Migration

### Quick translation table

| Svelte 4                      | Svelte 5                                       | Notes                  |
| ----------------------------- | ---------------------------------------------- | ---------------------- |
| `let count = 0`               | `let count = $state(0)`                        | Make reactive          |
| `$: doubled = count * 2`      | `let doubled = $derived(count * 2)`            | Computed value         |
| `$: { console.log(count); }`  | `$effect(() => { console.log(count); })`       | Side effect            |
| `$: if (count > 10) { ... }`  | `$effect(() => { if (count > 10) { ... } })`   | Conditional effect     |
| `export let name`             | `let { name } = $props()`                      | Props                  |
| `export let value` (bindable) | `let { value = $bindable() } = $props()`       | Two-way binding        |
| `on:click={handler}`          | `onclick={handler}`                            | Event handler          |
| `on:click\|preventDefault`    | `onclick={(e) => { e.preventDefault(); ... }}` | Event modifier         |
| `<slot />`                    | `{@render children()}`                         | Default slot           |
| `<slot name="header" />`      | `{@render header()}`                           | Named slot             |
| N/A                           | `{#snippet name()}...{/snippet}`               | Define reusable markup |

### TypeScript props

```svelte
<!-- Svelte 4 -->
<script lang="ts">
	export let count: number;
</script>

<!-- Svelte 5 -->
<script lang="ts">
	interface Props { count: number; }
	let { count }: Props = $props();
</script>
```

### Lifecycle

`onMount` still works (return value = cleanup). For most `onDestroy` cleanup, use `$effect` with a cleanup function:

```svelte
<script>
	import { onMount } from 'svelte';
	onMount(() => { console.log('mounted'); return () => console.log('cleanup'); });
	$effect(() => {
		const interval = setInterval(() => {...}, 1000);
		return () => clearInterval(interval);
	});
</script>
```

### Stores still work

```svelte
<script>
	import { writable } from 'svelte/store';
	const count = writable(0);
</script>
<button onclick={() => $count++}>{$count}</button>
```

**Runes:** component-local state. **Stores:** global/shared state. (Prefer classes with `$state` for shared reactivity.)

### Avoid legacy features

- `$state` instead of implicit reactivity (`let count = 0; count += 1`)
- `$derived`/`$effect` instead of `$:` (use effects only when no better solution)
- `$props` instead of `export let`, `$$props`, `$$restProps`
- `onclick={...}` instead of `on:click={...}`
- `{#snippet}`/`{@render}` instead of `<slot>`, `$$slots`, `<svelte:fragment>`
- `<DynamicComponent>` instead of `<svelte:component this={...}>`
- `import Self from './ThisComponent.svelte'` + `<Self>` instead of `<svelte:self>`
- classes with `$state` fields instead of stores
- `{@attach ...}` instead of `use:action`
- clsx-style arrays/objects in `class` instead of the `class:` directive

### Migration strategy

1. Don't mix syntaxes — migrate one component at a time fully.
2. Start with leaf components (bottom up).
3. Test incrementally.
4. Use TypeScript to catch binding/prop errors.
5. Read the [migration guide](https://svelte.dev/docs/svelte/v5-migration-guide).

**Feature detection** (if supporting both): `import { VERSION } from 'svelte/compiler'; const isSvelte5 = VERSION.startsWith('5');` — but full migration is generally better.
