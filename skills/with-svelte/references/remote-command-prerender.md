# Remote Functions — command(), prerender() and gotchas

Commands, single-flight mutations, prerendering, `getRequestEvent()`, devalue serialization, and gotchas.


**Contents**

- command()
- Single-flight mutations
- prerender()
- getRequestEvent()
- Serialization (devalue)
- Common Gotchas

---

## command()

Use `command()` for mutations from event handlers or other imperative code. Prefer `form()` when progressive enhancement matters.

```ts
import { command } from '$app/server';
import * as v from 'valibot';

export const addLike = command(v.string(), async (postId) => {
	await db.posts.incrementLikes(postId);
});
```

```svelte
<button onclick={() => addLike(post.id).updates(getLikes(post.id))}>
	Like
</button>
```

Commands cannot be called during render.

## Single-flight mutations

Use single-flight mutations to refresh query data in the same request as a `form()` submission or `command()` invocation. This avoids an extra round-trip and prevents stale UI. Two sides:

1. **Client requests updates** with `.updates(...)`
2. **Server accepts updates** with `requested(queryFn, limit)`

### Client: .updates()

`.updates()` accepts query functions, query instances, and optimistic overrides.

```ts
// Refresh all active getPosts instances
await createPost(data).updates(getPosts);

// Refresh one instance
await addLike(post.id).updates(getLikes(post.id));

// Optimistic update
await addLike(post.id).updates(getLikes(post.id).withOverride((n) => n + 1));
```

Inside enhanced forms:

```svelte
<form {...createPost.enhance(async ({ submit }) => {
	await submit().updates(getPosts);
})}>
	<!-- fields -->
</form>
```

### Server: requested(queryFn, limit)

`requested` is required for client-requested refreshes. The `limit` argument is required as of SvelteKit 2.58 because the list is client-controlled and each item can cause validation and data fetching.

```ts
import { command, query, requested } from '$app/server';

export const getPosts = query(filterSchema, async (filter) => {
	return db.posts.find(filter);
});

export const createPost = command(createSchema, async (data) => {
	await db.posts.create(data);

	for (const { arg, query } of requested(getPosts, 5)) {
		// arg is the validated/transformed argument
		// query is bound to the original client cache key
		void query.refresh();
	}
});
```

Important current behavior:

- `requested(queryFn, limit)` yields `{ arg, query }` objects, not raw args.
- Use the yielded `query` instance for `.refresh()`/`.set(...)`; it is bound to the original client cache key even if validation transformed `arg`.
- `limit` is required. Choose the maximum refreshes you are willing to process per mutation. `Infinity` is possible but usually a DoS footgun.
- If parsing one requested argument fails, that query errors, but the whole mutation does not fail.

Shorthand (equivalent to looping and calling `void query.refresh()` for each requested query):

```ts
await requested(getPosts, 5).refreshAll();
```

### Server-driven updates

If the server already knows exactly what changed, call `.refresh()` or `.set()` inside the handler without waiting for a client request.

```ts
export const updatePost = command(updateSchema, async ({ id, title }) => {
	const post = await db.posts.update(id, { title });
	void getPost(id).set(post);     // send known value back
	void getPosts().refresh();      // refetch list in same response
});
```

Use `void` rather than `await`; SvelteKit awaits and serializes these updates for the response.

## prerender()

Use `prerender()` for data that changes at most once per deployment. Results are computed during prerendering and can be cached on a CDN.

```ts
import { prerender } from '$app/server';
import * as v from 'valibot';

export const getPosts = prerender(async () => {
	return db.posts.allPublished();
});

export const getPost = prerender(v.string(), async (slug) => {
	return db.posts.findBySlug(slug);
});
```

SvelteKit's crawler automatically saves calls it discovers while prerendering. Use the `inputs` option when you need to enumerate values explicitly.

## getRequestEvent()

Use `getRequestEvent()` inside remote functions for cookies, headers, locals, and other request context.

```ts
import { getRequestEvent, query } from '$app/server';

export const getMe = query(async () => {
	const event = getRequestEvent();
	return event.locals.user;
});
```

## Serialization (devalue)

Remote arguments and return values are serialized with `devalue`.

**Can serialize:** primitives, arrays, plain objects, `Date`, `Map`, `Set`, typed arrays.

**Avoid:** functions · class instances without serialization support · symbols · circular references · `RegExp` as remote function arguments.

Query cache keys are based on serialized arguments. Object keys are normalized, so `{ limit: 10, offset: 20 }` and `{ offset: 20, limit: 10 }` refer to the same query cache entry.

## Common Gotchas

- Remote functions are HTTP endpoints; validate every exposed input. Use Standard Schema (`valibot`, `zod`, `arktype`, etc.) or use `.unchecked`/`'unchecked'` deliberately.
- `form()` invalid schema submissions do not run the handler.
- `command()` does not auto-refresh anything; use `.updates()` or server-driven query updates.
- `form()` auto-invalidates broadly after successful submissions unless you use more targeted single-flight updates.
- `requested()` must name each query function the server is willing to refresh; this protects bundle size and avoids unbounded client-controlled work.
- Use `form()` instead of `command()` when no-JS behavior matters.
- Use `prerender()` for data that changes at most once per deployment.
- Do not export shared schemas from `.remote.ts`; put them in a shared module or component module script.
