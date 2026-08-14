# SvelteKit Error Handling

`+error.svelte`, expected vs unexpected errors, and error pages.


**Contents**

- Error Handling

---

## Error Handling

### Error Boundary Placement

**Key rule:** `+error.svelte` must be _above_ the failing route in the hierarchy.

```text
src/routes/
├── +error.svelte           # Catches errors in all routes below
├── +page.svelte            # If this errors → uses +error.svelte above
└── admin/
    ├── +error.svelte       # Catches errors in admin routes
    └── +page.svelte        # If this errors → uses admin/+error.svelte
```

**Wrong:**

```text
src/routes/dashboard/
├── +layout.svelte          # If this errors...
└── +error.svelte           # This won't catch it (too low)
```

**Right:**

```text
src/routes/
├── +error.svelte           # Catches dashboard layout errors
└── dashboard/
    ├── +layout.svelte
    └── +error.svelte       # Catches dashboard page errors
```

### Error Propagation

Errors bubble up to the nearest `+error.svelte`. If no error boundary exists at that level, it goes to the parent.

```text
src/routes/
├── +error.svelte                    # Level 1 (root fallback)
└── blog/
    ├── +error.svelte                # Level 2 (blog fallback)
    └── [slug]/
        ├── +layout.server.ts        # Error here → blog/+error.svelte
        ├── +page.server.ts          # Error here → blog/+error.svelte
        └── +page.svelte             # Error here → blog/+error.svelte
```

### Basic Error Page

```svelte
<!-- +error.svelte -->
<script>
	import { page } from '$app/stores';
</script>

<h1>{$page.status}</h1><p>{$page.error.message}</p>
```

### Custom Error Data

```typescript
// +page.server.ts
import { error } from '@sveltejs/kit';

export const load = async ({ params }) => {
	const post = await getPost(params.id);

	if (!post) {
		throw error(404, {
			message: 'Post not found',
			postId: params.id,
		});
	}

	return { post };
};
```

```svelte
<!-- +error.svelte -->
<script>
	import { page } from '$app/stores';
</script>

<h1>{$page.status}</h1>
<p>{$page.error.message}</p>

{#if $page.error.postId}
	<p>Could not find post with ID: {$page.error.postId}</p>
{/if}
```

### Status Code Specific Errors

```svelte
<!-- +error.svelte -->
<script>
	import { page } from '$app/stores';
</script>

{#if $page.status === 404}
	<h1>Page Not Found</h1>
	<a href="/">Go home</a>
{:else if $page.status === 403}
	<h1>Access Denied</h1>
{:else if $page.status === 401}
	<h1>Unauthorized</h1>
	<a href="/login">Login</a>
{:else if $page.status >= 500}
	<h1>Server Error</h1>
{:else}
	<h1>Error {$page.status}</h1>
	<p>{$page.error.message}</p>
{/if}
```

### Common Status Codes

- **400** Bad Request | **401** Unauthorized (not logged in) | **403** Forbidden (logged in, no permission) | **404** Not Found | **500** Internal Server Error | **503** Service Unavailable

### Expected vs Unexpected Errors

**Expected (use `error()`):**

```typescript
if (!post) throw error(404, 'Post not found');
if (post.authorId !== user.id) throw error(403, 'Not your post');
```

**Unexpected (let it bubble):** Unhandled exceptions (DB connection fails) show generic 500.

### handleError Hook (logging/monitoring)

```typescript
// src/hooks.server.ts
import type { HandleServerError } from '@sveltejs/kit';

export const handleError: HandleServerError = ({ error, event }) => {
	console.error('Error:', error, 'Path:', event.url.pathname);
	// Return user-friendly message (don't expose internals)
	return {
		message: 'An unexpected error occurred',
		code: error?.code ?? 'UNKNOWN',
	};
};
```

### Best Practice: Throw in load, not components

Validate and throw errors in `load` functions, not components. A component reading undefined `data` just renders nothing (or crashes) without triggering an error boundary.

### Fallback Error Handling

Always have a root `src/routes/+error.svelte`. Show details in dev, generic in production:

```svelte
<!-- src/routes/+error.svelte -->
<script>
	import { page } from '$app/stores';
	import { dev } from '$app/environment';
</script>

<h1>Oops! Something went wrong</h1>

{#if dev}
	<pre>{JSON.stringify($page.error, null, 2)}</pre>
{:else}
	<p>We're sorry, but something unexpected happened.</p>
{/if}

<a href="/">Go home</a>
```
