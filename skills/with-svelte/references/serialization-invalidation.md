# SvelteKit Serialization & Invalidation

What crosses the server-client boundary, and invalidating client-side auth state.


**Contents**

- Serialization: What Can/Can't Be Returned
- Client-Side Auth Invalidation

---

## Serialization: What Can/Can't Be Returned

**The Rule:** Server load functions and form actions must return JSON-serializable data. Data travels server → client as JSON; non-JSON types break.

### ✅ Serializable (Safe)

String · Number · Boolean · `null` · Array · Plain Object · Nested (if all values serializable).

### ❌ NOT Serializable (Breaks)

| Type           | Example        | Why                        | Fix                                          |
| -------------- | -------------- | -------------------------- | -------------------------------------------- |
| Date           | `new Date()`   | Becomes string             | Use `.toISOString()`                         |
| undefined      | `undefined`    | Removed from JSON          | Use `null`                                   |
| Function       | `() => {}`     | Can't serialize            | Remove or convert to data                    |
| Class instance | `new User()`   | Only serializes properties | Convert to plain object                      |
| Map            | `new Map()`    | Becomes `{}`               | Convert to object: `Object.fromEntries(map)` |
| Set            | `new Set()`    | Becomes `{}`               | Convert to array: `Array.from(set)`          |
| BigInt         | `123n`         | Error                      | Convert to string                            |
| Symbol         | `Symbol('id')` | Removed                    | Don't use                                    |
| RegExp         | `/test/`       | Becomes `{}`               | Convert to string                            |
| Error          | `new Error()`  | Loses stack                | Extract message/code                         |

### Examples

**Date** — convert to ISO string in load, parse back in component:

```typescript
// +page.server.ts - RIGHT
return {
	user: { id: user.id, name: user.name, createdAt: user.createdAt.toISOString() },
};
```

```svelte
<script>
	export let data;
	const createdAt = new Date(data.user.createdAt); // Parse back to Date
</script>
```

**Class instance** — methods are lost during serialization; return a plain object instead.

**undefined** — removed during `JSON.stringify` (key disappears); use `null` to preserve.

**Map/Set** — become `{}`; convert with `Array.from(set)` / `Object.fromEntries(map)`.

**BigInt** — can't serialize; return as a string.

### ORM Returns (Drizzle, Prisma)

Most ORMs return plain objects with Date fields:

```typescript
return {
	user: { ...user, createdAt: user.createdAt.toISOString() },
};
```

Or use a helper:

```typescript
function serialize<T extends Record<string, any>>(obj: T): T {
	return JSON.parse(JSON.stringify(obj)); // Forces serialization
}
```

### Detecting Issues

SvelteKit throws if you return non-serializable data:

```text
Error: Data returned from `load` while rendering / is not serializable:
  - Cannot stringify arbitrary non-POJOs
```

### Quick Checklist

Before returning from server load or form action: all values string/number/boolean/null/array/plain object? · No Date (use `.toISOString()`)? · No undefined (use null)? · No class instances? · No Map/Set? · No functions? · No BigInt (convert to string)?

## Client-Side Auth Invalidation

When using client-side auth libraries (Better Auth, Firebase, Supabase client), layout server data doesn't auto-refresh after login/logout.

### The Problem

```typescript
// signin/+page.svelte - BROKEN
async function handle_signin() {
  await auth_client.signIn.email({ email, password });
  goto('/');  // Layout still shows "logged out"
}
```

**Why?** Client-side navigation (`goto()`) doesn't re-run server load functions. The session cookie is set, but `+layout.server.ts` data is stale.

### ✅ Correct Pattern

Use `goto()` with `invalidateAll: true` in a single call to ensure layout data refreshes:

```typescript
// RIGHT: Single call with invalidateAll option
await goto('/dashboard', { invalidateAll: true });
```

**Why this matters:** After client-side auth, cookies are set but the root layout's `load` function (which typically checks `auth.api.getSession()`) has cached data. `invalidateAll: true` forces all load functions to re-run with the new session cookie.

### Solution A: Inline Invalidation (Simple)

```typescript
// signin/+page.svelte
import { goto, invalidateAll } from '$app/navigation';

async function handle_signin() {
  const result = await auth_client.signIn.email({ email, password });
  if (result.error) return;
  await invalidateAll();  // Re-runs ALL load functions
  goto('/');
}

async function handle_signout() {
  await auth_client.signOut();
  await invalidateAll();
}
```

**Use when:** Single auth entry point, simple apps.

### Solution B: Auth State Listener (Robust)

```svelte
<!-- +layout.svelte -->
<script>
  import { invalidateAll } from '$app/navigation';
  import { onMount } from 'svelte';
  import { auth_client } from '$lib/auth-client';

  onMount(() => {
    const unsubscribe = auth_client.onAuthStateChange(() => {
      invalidateAll();
    });
    return unsubscribe;
  });
</script>
```

**Use when:** Multiple auth flows (OAuth, magic links, etc.), complex apps.

### Common Mistakes

**❌ Separate invalidateAll + goto** — race condition; data might not refresh before navigation:

```typescript
// WRONG
await invalidateAll();
goto('/dashboard');
// ALSO WRONG: goto doesn't wait for invalidation
await invalidateAll();
await goto('/dashboard');
```

Prefer the single `await goto(url, { invalidateAll: true })`. If using `invalidateAll()` separately, always `await` it before `goto()`.

**❌ Destructuring layout data** — static snapshot never updates after `invalidateAll()`:

```svelte
<!-- WRONG -->
<script>
  let { data } = $props();
  const { user } = data;  // Never updates
</script>

<!-- RIGHT - reactive access -->
<script>
  let { data } = $props();
</script>
{data.user?.email}
```

### When invalidateAll() Runs

`invalidateAll()` is the nuclear option — it re-runs ALL load functions for the current page regardless of dependencies: `+layout.server.ts` (all levels), `+layout.ts` (all levels), `+page.server.ts`, `+page.ts`.

### Comparison with Server-Side Auth

| Approach     | Auth Location              | Invalidation               |
| ------------ | -------------------------- | -------------------------- |
| Form actions | Server (`+page.server.ts`) | Automatic (page reload)    |
| Client auth  | Browser (auth_client)      | Manual (`invalidateAll()`) |

Form actions with `throw redirect()` cause a full navigation, which naturally re-runs load functions. Client-side auth with `goto()` does not.
