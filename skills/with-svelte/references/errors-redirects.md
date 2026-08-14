# SvelteKit fail(), redirect() and error()

Control-flow helpers and the rethrow rule.


**Contents**

- fail(), redirect(), error()
- ❌ Catching redirect Without Rethrowing

---

## fail(), redirect(), error()

| Function     | When             | Must Throw? | Use Case                              |
| ------------ | ---------------- | ----------- | ------------------------------------- |
| `fail()`     | Validation error | No (return) | Form validation errors                |
| `redirect()` | Navigate user    | **YES**     | After successful action               |
| `error()`    | Fatal error      | **YES**     | Unauthorized, not found, server error |

### fail() — Validation Errors

**Return** (don't throw) from form actions to show validation errors:

```typescript
import { fail } from '@sveltejs/kit';

export const actions = {
	default: async ({ request }) => {
		const data = await request.formData();
		const email = data.get('email');
		if (!email || !email.includes('@')) {
			return fail(400, { email, error: 'Invalid email', missing: !email });
		}
		await processEmail(email);
		throw redirect(303, '/success');
	},
};
```

**Key points:** Return (don't throw) · status code (400 = bad request) · return validation errors + form data to repopulate fields · accessible via `form` prop · page stays on same URL.

### redirect() — Navigation

**Throw** redirect() to navigate user to another page:

```typescript
import { redirect } from '@sveltejs/kit';

export const actions = {
	login: async ({ request, cookies }) => {
		const data = await request.formData();
		const user = await authenticate(data);
		if (!user) return fail(401, { error: 'Invalid credentials' });
		cookies.set('session', user.sessionToken, { path: '/' });
		throw redirect(303, '/dashboard'); // MUST throw
	},
};
```

**Status codes:** `303` See Other (recommended POST → GET) · `301` Moved Permanently · `302` Found (temporary) · `307` Temporary (preserves method) · `308` Permanent (preserves method). **Use 303** for most cases (especially after form submission).

**Key points:** MUST throw · use 303 for form actions · can redirect to external URLs · can use relative paths `throw redirect(303, '..')`.

### error() — Fatal Errors

**Throw** error() for unrecoverable errors (auth, not found, server error):

```typescript
import { error } from '@sveltejs/kit';

export const load = async ({ params, locals }) => {
	const post = await db.query.posts.findFirst({ where: eq(posts.id, params.id) });
	if (!post) throw error(404, 'Post not found'); // MUST throw
	if (post.authorId !== locals.userId) throw error(403, 'Forbidden'); // MUST throw
	return { post };
};
```

**Common status codes:** 400 Bad Request · 401 Unauthorized · 403 Forbidden · 404 Not Found · 500 Internal Server Error.

With custom error data:

```typescript
// src/routes/posts/[id]/+page.server.ts
if (!post) {
	throw error(404, { message: 'Post not found', postId: params.id });
}
```

```svelte
<!-- src/routes/posts/[id]/+error.svelte -->
<script>
	import { page } from '$app/stores';
</script>

<h1>{$page.status}: {$page.error.message}</h1>
{#if $page.error.postId}
	<p>Could not find post with ID: {$page.error.postId}</p>
{/if}
```

**Key points:** MUST throw · renders closest `+error.svelte` · accessible via `$page.status` and `$page.error` · stops load function execution · use for authorization, not found, server errors.

### Common Mistakes

**❌ Not Throwing redirect()** — `redirect(303, '/home')` DOESN'T WORK; use `throw redirect(303, '/home')`.

**❌ Not Throwing error()** — `error(404, 'Not found')` DOESN'T WORK; use `throw error(404, 'Not found')`.

**❌ Throwing fail()** — `throw fail(...)` is WRONG; use `return fail(400, { error: 'Bad' })`.

## ❌ Catching redirect Without Rethrowing

```typescript
// WRONG
try {
	throw redirect(303, '/success');
} catch (e) {
	console.error(e); // Catches redirect - it won't work!
	return fail(500, { error: 'Failed' });
}

// RIGHT
import { isRedirect } from '@sveltejs/kit';
try {
	throw redirect(303, '/success');
} catch (e) {
	if (isRedirect(e)) throw e; // Rethrow redirect
	console.error(e);
	return fail(500, { error: 'Failed' });
}
```

### Decision Tree

```text
Problem in form action?
├─ Validation error (show to user) → return fail(400, { errors })
├─ Success (navigate) → throw redirect(303, '/success')
└─ Fatal error (auth, not found) → throw error(403, 'Forbidden')

Problem in load function?
├─ Data not found → throw error(404, 'Not found')
├─ Unauthorized → throw error(401, 'Unauthorized')
├─ Forbidden → throw error(403, 'Forbidden')
└─ Server error → throw error(500, 'Server error')
```

### Summary Table

|                      | fail()            | redirect()      | error()         |
| -------------------- | ----------------- | --------------- | --------------- |
| **Throw or return?** | Return            | **Throw**       | **Throw**       |
| **Use in**           | Form actions      | Actions & load  | Actions & load  |
| **Purpose**          | Validation errors | Navigate        | Fatal errors    |
| **Status codes**     | 400-499           | 301-308         | 400-599         |
| **Accessible via**   | `form` prop       | N/A (navigates) | `+error.svelte` |
| **Stays on page?**   | Yes               | No (navigates)  | No (error page) |
