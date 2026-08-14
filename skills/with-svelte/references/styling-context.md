# Styling & Context

CSS from JavaScript, styling child components, and `createContext`.

**Last verified:** 2025-01-14

**Contents**

- Using JavaScript Variables in CSS
- Styling Child Components
- Context

---

## Using JavaScript Variables in CSS

To use a JS variable inside CSS, set a CSS custom property with the `style:` directive, then reference `var(--…)` in `<style>`.

```svelte
<div style:--columns={columns}>...</div>
<style>
	/* var(--columns) available here */
</style>
```

## Styling Child Components

Component `<style>` is scoped to that component. For a parent to control a child's styles, the **preferred** way is CSS custom properties:

```svelte
<!-- Parent.svelte -->
<Child --color="red" />

<!-- Child.svelte -->
<h1>Hello</h1>
<style>
	h1 { color: var(--color); }
</style>
```

If impossible (e.g. the child comes from a library), use `:global` to override:

```svelte
<div>
	<Child />
</div>
<style>
	div :global {
		h1 { color: red; }
	}
</style>
```

## Context

Consider using context instead of declaring state in a shared module. This scopes the state to the part of the app that needs it, and eliminates the possibility of it leaking between users when server-side rendering.

Use `createContext` rather than `setContext`/`getContext`, as it provides type safety.

```ts
// context.ts
import { createContext } from 'svelte';
const [get_theme, set_theme] = createContext<{ current: string }>('theme');
export { get_theme, set_theme };
```

```svelte
<!-- Provider.svelte -->
<script>
	import { set_theme } from './context';
	let theme = $state('dark');
	set_theme({
		get current() { return theme; },
		set current(value) { theme = value; },
	});
</script>
{@render children()}
```

```svelte
<!-- Consumer.svelte -->
<script>
	import { get_theme } from './context';
	const theme = get_theme();
</script>
<p>Theme: {theme.current}</p>
<button onclick={() => theme.current = 'light'}>Light mode</button>
```

### Why `createContext` over `set`/`getContext`

| Feature        | `setContext`/`getContext` | `createContext`     |
| -------------- | ------------------------- | ------------------- |
| Type safety    | Manual casting            | **Automatic**       |
| Key management | String keys (typo-prone)  | **Module-scoped**   |
| Default values | Manual check              | **Built-in support**|

### Context vs shared module state

Context scopes state to the component tree, preventing leaks between users during SSR. Use it for deeply nested components instead of prop drilling.

```ts
// BAD - shared module state leaks between SSR requests
export let theme = $state('dark');
// GOOD - context is scoped per component tree
const [get_theme, set_theme] = createContext<string>('theme');
```
