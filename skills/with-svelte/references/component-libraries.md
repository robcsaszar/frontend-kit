# Component Libraries & Web Components

Bits/Ark/Melt UI, and authoring custom elements.

**Last verified:** 2025-01-14

**Contents**

- Component Libraries
- Web Components (customElement)

---

## Component Libraries

**Bits UI** (headless) | **Ark UI** | **Melt UI** (primitives). All three work with Svelte 5 runes.

| Library | Style    | Approach   | Best For            |
| ------- | -------- | ---------- | ------------------- |
| Bits UI | Unstyled | Components | Quick accessible UI |
| Ark UI  | Unstyled | Components | Feature-rich apps   |
| Melt UI | Unstyled | Builders   | Maximum control     |

### Bits UI

Headless, unstyled, accessible (ARIA, keyboard nav) components. Composable compound components; bring your own CSS. [bits-ui.com](https://bits-ui.com)

```bash
pnpm add bits-ui
```

```svelte
<script>
	import { Button } from 'bits-ui';
</script>
<Button.Root class="my-button">Click me</Button.Root>
```

### Ark UI

Full-featured component library. [ark-ui.com](https://ark-ui.com)

```bash
pnpm add @ark-ui/svelte
```

```svelte
<script>
	import { Dialog } from '@ark-ui/svelte';
</script>
<Dialog.Root>
	<Dialog.Trigger>Open</Dialog.Trigger>
	<Dialog.Backdrop />
	<Dialog.Positioner>
		<Dialog.Content>
			<Dialog.Title>Title</Dialog.Title>
			<Dialog.Description>Description</Dialog.Description>
			<Dialog.CloseTrigger>Close</Dialog.CloseTrigger>
		</Dialog.Content>
	</Dialog.Positioner>
</Dialog.Root>
```

### Melt UI

Low-level primitives (**builders** — functions, not components) for maximum flexibility. [melt-ui.com](https://melt-ui.com)

```bash
pnpm add @melt-ui/svelte
```

```svelte
<script>
	import { createDialog } from '@melt-ui/svelte';
	const {
		elements: { trigger, portalled, overlay, content, title, close },
		states: { open },
	} = createDialog();
</script>
<button use:melt={$trigger}>Open</button>
{#if $open}
	<div use:melt={$portalled}>
		<div use:melt={$overlay} />
		<div use:melt={$content}>
			<h2 use:melt={$title}>Title</h2>
			<button use:melt={$close}>Close</button>
		</div>
	</div>
{/if}
```

## Web Components (customElement)

```javascript
// svelte.config.js — enable for entire project
export default {
	compilerOptions: { customElement: true },
};
```

Or per-component with `<svelte:options>`:

```svelte
<svelte:options customElement="my-element" />
<script>
	let { name = 'World' } = $props();
</script>
<p>Hello {name}!</p>
```

### Gotchas

**1. Self-closing tags** — Svelte 5 requires closing tags for custom elements:

```svelte
<my-element />          <!-- WRONG -->
<my-element></my-element> <!-- RIGHT -->
```

**2. Nested HTML in `<option>`** — causes compiler errors; use snippets:

```svelte
<!-- WRONG - compiler error -->
<select><option><div>Rich content</div></option></select>
<!-- WORKAROUND -->
{#snippet optionContent()}<div>Rich content</div>{/snippet}
<select><option>{@render optionContent()}</option></select>
```

**3. Shadow DOM styling** — styles are scoped to the shadow DOM by default:

```svelte
<svelte:options customElement="styled-button" />
<button><slot /></button>
<style>
	button { background: blue; } /* Only affects this component's shadow DOM */
</style>
```

### Exposing props as attributes

```svelte
<svelte:options
	customElement={{
		tag: 'my-counter',
		props: { count: { reflect: true, type: 'Number' } },
	}}
/>
<script>
	let { count = 0 } = $props();
</script>
<button onclick={() => count++}>{count}</button>
```

### Events

```svelte
<svelte:options customElement="event-button" />
<script>
	import { createEventDispatcher } from 'svelte';
	const dispatch = createEventDispatcher();
</script>
<button onclick={() => dispatch('clicked', { time: Date.now() })}>Click me</button>
```

```html
<event-button></event-button>
<script>
	document.querySelector('event-button')
		.addEventListener('clicked', (e) => console.log(e.detail));
</script>
```

### Library distribution

```json
// package.json — always include svelte in keywords + peerDependencies
{
	"svelte": "./dist/index.js",
	"exports": { ".": { "svelte": "./dist/index.js" } },
	"keywords": ["svelte"],
	"peerDependencies": { "svelte": "^5.0.0" }
}
```
