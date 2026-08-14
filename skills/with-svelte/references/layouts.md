# SvelteKit Layouts

Nested layouts, layout groups, and layout data inheritance.


**Contents**

- Layouts

---

## Layouts

Layouts apply to all child routes. A `+layout.svelte` at any level wraps its descendants.

```svelte
<!-- src/routes/+layout.svelte -->
<script>
	let { children } = $props();
</script>

<header>Header</header>
<main>{@render children()}</main>
<footer>Footer</footer>
```

**Key points:**

- Must declare `children` in `$props()`
- Use `{@render children()}` to render nested content
- Root layout wraps ALL pages

### Nested Layouts

Layouts inherit from parent layouts. Root layout wraps section layout wraps page.

```text
src/routes/
├── +layout.svelte          # Root layout (all pages)
└── dashboard/
    ├── +layout.svelte      # Dashboard layout (dashboard pages only)
    └── +page.svelte        # Uses both layouts
```

```svelte
<!-- src/routes/+layout.svelte -->
<script>
	let { children } = $props();
</script>

<div class="app">
	<nav>Global Nav</nav>
	{@render children()}
</div>
```

```svelte
<!-- src/routes/dashboard/+layout.svelte -->
<script>
	let { children } = $props();
</script>

<div class="dashboard">
	<aside>Dashboard Sidebar</aside>
	<main>{@render children()}</main>
</div>
```

Rendered HTML nests root → dashboard → page accordingly.

### Layout Groups

Use `(groups)` to organize layouts without affecting URLs — different layouts for different sections, useful for auth boundaries.

```text
src/routes/
├── (marketing)/
│   ├── +layout.svelte      # Marketing layout
│   ├── about/+page.svelte  # /about (uses marketing layout)
│   └── pricing/+page.svelte # /pricing (uses marketing layout)
│
└── (app)/
    ├── +layout.svelte      # App layout
    ├── dashboard/+page.svelte  # /dashboard (uses app layout)
    └── settings/+page.svelte   # /settings (uses app layout)
```

### Reset Layout

`@` in a filename to break layout inheritance is **NOT RECOMMENDED** — deprecated in SvelteKit 2+. Use layout groups to create separate hierarchies instead.

### Layout with Data Loading

```typescript
// src/routes/+layout.server.ts
export const load = async ({ locals }) => {
	// Available to all child routes
	return { user: locals.user };
};
```

```svelte
<!-- src/routes/+layout.svelte -->
<script>
	let { children, data } = $props();
</script>

<header>
	{#if data.user}
		<span>Welcome, {data.user.name}</span>
	{:else}
		<a href="/login">Login</a>
	{/if}
</header>

{@render children()}
```

### Protected Layouts

```typescript
// src/routes/(app)/+layout.server.ts
import { redirect } from '@sveltejs/kit';

export const load = async ({ locals }) => {
	if (!locals.user) {
		throw redirect(303, '/login');
	}
	return { user: locals.user };
};
```

All routes under `(app)` now require authentication.

### Sharing Layout State

```svelte
<!-- src/routes/+layout.svelte -->
<script>
	import { setContext } from 'svelte';
	let { children, data } = $props();
	setContext('user', data.user);
</script>

{@render children()}
```

```svelte
<!-- Any child component -->
<script>
	import { getContext } from 'svelte';
	const user = getContext('user');
</script>

<p>Hello, {user.name}</p>
```

### Layout Slot Props (Snippets)

```svelte
<!-- src/routes/dashboard/+layout.svelte -->
<script>
	let { children, header } = $props();
</script>

<div class="dashboard">
	<aside>Sidebar</aside>
	<div class="content">
		{#if header}
			<header>{@render header()}</header>
		{/if}
		<main>{@render children()}</main>
	</div>
</div>
```

```svelte
<!-- src/routes/dashboard/+page.svelte -->
{#snippet header()}
	<h1>Custom Dashboard Header</h1>
{/snippet}

<p>Dashboard content</p>
```

### Layout Best Practices

1. Keep root layout minimal (shared across ALL pages)
2. Use layout groups for separate sections
3. Load shared data in layout's load function
4. Use context for sharing state with descendants
5. Avoid too many nested layouts (max 2-3 levels)
6. Don't put auth logic in root layout (use groups)
7. Layouts share data DOWN, not UP
8. ❌ Avoid conditionals in layouts (use groups instead)
