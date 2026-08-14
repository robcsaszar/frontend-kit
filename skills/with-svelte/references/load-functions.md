# SvelteKit Load Functions

Server vs universal `load`, composition patterns, and `depends`.


**Contents**

- Quick Start
- Load Functions: Server vs Universal
- Pattern 1: Server + Universal
- Pattern 2: Conditional Universal Load
- Pattern 3: Depends for Revalidation
- ❌ Importing Server Code in Universal Load

---

## Quick Start

**Which file?** Server-only (DB/secrets): `+page.server.ts` | Universal (runs both): `+page.ts` | API: `+server.ts`

**Load decision:** Need server resources? → server load | Need client APIs? → universal load

**Form actions:** Always `+page.server.ts`. Return `fail()` for errors, throw `redirect()` to navigate, throw `error()` for failures.

```typescript
// +page.server.ts
import { fail, redirect } from '@sveltejs/kit';

export const load = async ({ locals }) => {
	const user = await db.users.get(locals.userId);
	return { user }; // Must be JSON-serializable
};

export const actions = {
	default: async ({ request }) => {
		const data = await request.formData();
		const email = data.get('email');

		if (!email) return fail(400, { email, missing: true });

		await updateEmail(email);
		throw redirect(303, '/success');
	},
};
```

## Load Functions: Server vs Universal

### Decision Matrix

| Need                                | Use            | File              |
| ----------------------------------- | -------------- | ----------------- |
| Database access                     | Server load    | `+page.server.ts` |
| Secrets/env vars                    | Server load    | `+page.server.ts` |
| Server-only packages                | Server load    | `+page.server.ts` |
| Browser APIs (window, localStorage) | Universal load | `+page.ts`        |
| Client-side fetch                   | Universal load | `+page.ts`        |
| Runs on both                        | Universal load | `+page.ts`        |

### Server Load (+page.server.ts)

**When:** Need server-only resources (DB, secrets, server APIs). **Runs:** Only on server (never in browser).

```typescript
// src/routes/profile/+page.server.ts
import type { PageServerLoad } from './$types';
import { db } from '$lib/server/database';

export const load: PageServerLoad = async ({ locals, params }) => {
	const user = await db.query.users.findFirst({
		where: eq(users.id, locals.userId),
	});
	const posts = await db.query.posts.findMany({
		where: eq(posts.authorId, user.id),
	});
	// Must return serializable data
	return {
		user: { id: user.id, name: user.name, email: user.email },
		posts,
	};
};
```

**Key points:**

- Runs only on server
- Can access `$lib/server/*` imports
- Can use secrets from `env` safely
- Return values must be JSON-serializable
- Output is automatically passed to universal load

### Universal Load (+page.ts)

**When:** Need to run on both server and client, or need browser APIs. **Runs:** Server (during SSR) AND client (during navigation).

```typescript
// src/routes/dashboard/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ data, fetch }) => {
	// `data` comes from +page.server.ts if it exists
	const { user } = data;

	// Fetch additional data (works on both server and client)
	const response = await fetch('/api/stats');
	const stats = await response.json();

	// Can access browser APIs (but check if in browser first)
	const theme =
		typeof window !== 'undefined'
			? localStorage.getItem('theme')
			: null;

	return { user, stats, theme };
};
```

**Key points:**

- Runs on both server AND client
- Receives server load output as `data` parameter
- Use SvelteKit's `fetch` (automatically handles SSR)
- Check `typeof window !== 'undefined'` for browser APIs
- Cannot import from `$lib/server/*`

### Data Flow

```text
Request → Server Load (+page.server.ts)
            ↓ (returns { user })
        Universal Load (+page.ts)
            ↓ (receives data: { user }, returns { user, stats })
        Page Component (+page.svelte)
            ↓ (receives data: { user, stats })
```

```typescript
// +page.server.ts
export const load = async () => {
  return { serverData: 'from server' };
};

// +page.ts
export const load = async ({ data }) => {
  console.log(data.serverData);  // 'from server'
  return { ...data, clientData: 'from universal' };
};

// +page.svelte
<script>
  export let data;  // { serverData, clientData }
</script>
```

### Common Patterns

## Pattern 1: Server + Universal

```typescript
// +page.server.ts - Fetch sensitive data
export const load = async ({ locals }) => {
	const user = await getUser(locals.session);
	return { user };
};

// +page.ts - Fetch public data
export const load = async ({ data, fetch }) => {
	const publicPosts = await fetch('/api/posts').then((r) => r.json());
	return { ...data, publicPosts };
};
```

## Pattern 2: Conditional Universal Load

```typescript
// +page.ts
import { browser } from '$app/environment';

export const load = async ({ fetch }) => {
	const serverData = await fetch('/api/data').then((r) => r.json());
	let clientOnlyData = null;
	if (browser) {
		clientOnlyData = localStorage.getItem('cache');
	}
	return { serverData, clientOnlyData };
};
```

## Pattern 3: Depends for Revalidation

```typescript
// +page.ts
export const load = async ({ fetch, depends }) => {
	depends('app:posts'); // Invalidate with invalidate('app:posts')
	const posts = await fetch('/api/posts').then((r) => r.json());
	return { posts };
};

// Somewhere else:
import { invalidate } from '$app/navigation';
invalidate('app:posts'); // Re-runs load function
```

### Common Mistakes

## ❌ Importing Server Code in Universal Load

```typescript
// +page.ts - WRONG
import { db } from '$lib/server/database'; // ERROR - can't import server code
```

**Fix:** Move to `+page.server.ts`

**❌ Returning Non-Serializable Data** — class instances aren't serializable. **Fix:** Return plain objects (see Serialization below).

**❌ Using window/localStorage Without Check** — fails on server. **Fix:** Check `browser`:

```typescript
import { browser } from '$app/environment';
export const load = async () => {
	const theme = browser ? localStorage.getItem('theme') : 'light';
	return { theme };
};
```

### When to Use Which

**Server Load:** DB access, secrets/env vars, server-only npm packages, hiding implementation details.

**Universal Load:** browser APIs (localStorage, window), public APIs (works on both), client-side navigation without server round-trip, public data.

**Both:** server load fetches sensitive data; universal load fetches public data or adds client-side enhancements.
