# Remote Functions — query() and form()

Status, function types, reading data, and form mutations in `*.remote.ts`.


**Contents**

- Current Status
- Function Types
- query()
- form()

---

## Current Status

Remote functions are **experimental** in SvelteKit 2.58. They are exported from `.remote.ts`/`.remote.js` files and can be called anywhere in the app. They always execute on the server; on the client, SvelteKit transforms calls into generated `fetch` requests. Enable them explicitly:

```js
// svelte.config.js
export default {
	kit: { experimental: { remoteFunctions: true } },
	compilerOptions: { experimental: { async: true } } // only for await in components
};
```

Remote files can live anywhere under `src` except `src/lib/server`.

## Function Types

| Function      | Use for                            | Notes                                                  |
| ------------- | ---------------------------------- | ------------------------------------------------------ |
| `query()`     | Dynamic server reads               | Cached while rendered; supports refresh and batching   |
| `form()`      | Progressive form mutations         | Works without JS; supports fields, validation, enhance |
| `command()`   | Imperative/event-handler mutations | Cannot be called during render                         |
| `prerender()` | Static/build-time reads            | For data that changes at most once per deployment      |

**Which function?** Dynamic reads → `query()` · Progressive forms → `form()` · Event-handler mutations → `command()` · Build-time/static reads → `prerender()`.

## query()

Use `query()` for dynamic server reads.

```ts
import { query } from '$app/server';
import * as v from 'valibot';

export const getPost = query(v.string(), async (slug) => {
	return db.posts.findBySlug(slug);
});
```

In components with async enabled:

```svelte
<script lang="ts">
	import { getPost } from './posts.remote';
	let { slug } = $props();
	const post = $derived(await getPost(slug));
</script>

<h1>{post.title}</h1>
```

Without async in components, use `{#await}`:

```svelte
{#await getPost(slug) then post}
	<h1>{post.title}</h1>
{/await}
```

### Query refresh

```svelte
<button onclick={() => getPost(slug).refresh()}>
	Refresh
</button>
```

Queries are cached while on the page. Calling `getPost(slug)` repeatedly returns the same active query instance for that argument.

### query.batch()

Use `query.batch()` for n+1 reads. Calls made in the same macrotask are grouped into one server request. The handler receives all inputs and returns a resolver.

```ts
import { query } from '$app/server';
import * as v from 'valibot';

export const getWeather = query.batch(v.string(), async (cityIds) => {
	const rows = await db.weather.findMany(cityIds);
	const byId = new Map(rows.map((row) => [row.city_id, row]));
	return (cityId) => byId.get(cityId);
});
```

## form()

Use `form()` for mutations that should gracefully degrade when JavaScript is disabled.

```ts
import { form } from '$app/server';
import * as v from 'valibot';

export const createPost = form(
	v.object({
		title: v.pipe(v.string(), v.nonEmpty()),
		content: v.pipe(v.string(), v.nonEmpty())
	}),
	async ({ title, content }) => {
		await db.posts.create({ title, content });
	}
);
```

Use `.fields.<name>.as(type)` to bind typed inputs:

```svelte
<form {...createPost}>
	<input {...createPost.fields.title.as('text')} />
	<textarea {...createPost.fields.content.as('text')}></textarea>
	<button>Publish</button>
</form>
```

`.as(...)` supplies `name`, type-specific attributes, validation state, and repopulation behavior after failed validation.

### Sensitive fields

Prefix sensitive field names with `_` to prevent repopulation after invalid non-enhanced submissions. Use for passwords, credit card numbers, tokens, and similar secrets.

```svelte
<input {...login.fields._password.as('password')} />
```

### Validation

If schema validation fails, the handler does not run. Issues are exposed through field helpers:

```svelte
{#each createPost.fields.title.issues() as issue}
	<p class="error">{issue.message}</p>
{/each}
```

Programmatic validation inside a handler uses `invalid` from `@sveltejs/kit`:

```ts
import { invalid } from '@sveltejs/kit';
import { form } from '$app/server';

export const login = form(schema, async (data, issue) => {
	if (!(await auth.check(data))) {
		invalid(issue.email('Invalid credentials'));
	}
});
```

Client-side preflight validation can prevent invalid submissions before they hit the server:

```svelte
<form {...createPost.preflight(schema)}>
	<!-- fields -->
</form>
```

### enhance()

`enhance` customizes JS-enabled submission. Since SvelteKit 2.57, `submit()` returns a boolean: `true` means the submission completed without validation failure; `false` means validation failed and the handler did not run.

```svelte
<form {...createPost.enhance(async ({ form, submit }) => {
	try {
		if (await submit()) {
			form.reset();
			showToast('Published');
		}
	} catch (error) {
		showToast('Something went wrong');
	}
})}>
	<!-- fields -->
</form>
```

With `enhance`, the form is not automatically reset. Call `form.reset()` when you want to clear inputs.

### Multiple form instances

Use `.for(id)` when rendering the same form many times and each instance needs isolated state.

```svelte
{#each todos as todo}
	{@const modify = modifyTodo.for(todo.id)}
	<form {...modify}>
		<input {...modify.fields.description.as('text', todo.description)} />
		<button disabled={!!modify.pending}>Save</button>
	</form>
{/each}
```

### Multiple submit buttons

Model the clicked submit button as a schema field and bind buttons with `.as('submit', value)`.

```svelte
<form {...loginOrRegister}>
	<input {...loginOrRegister.fields.email.as('email')} />
	<input {...loginOrRegister.fields._password.as('password')} />
	<button {...loginOrRegister.fields.action.as('submit', 'login')}>Login</button>
	<button {...loginOrRegister.fields.action.as('submit', 'register')}>Register</button>
</form>
```

### File uploads

Remote `form()` supports file uploads. Since SvelteKit 2.49, enhanced forms use a streaming binary upload format so server-side form handlers can access form data before large files finish uploading. SvelteKit 2.52 tightened file metadata and offset-table validation.

Practical rules:

- Keep file schemas explicit and validate type/size server-side.
- Do not trust client-provided file metadata.
- Prefer streaming processing/storage for large files.
- Handle upload errors as normal form validation or request errors.
