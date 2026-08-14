# SvelteKit Form Actions

Default and named actions, progressive enhancement, and validation returns.


**Contents**

- Form Actions

---

## Form Actions

Form actions live in `+page.server.ts` and handle form submissions:

```typescript
// +page.server.ts
import { fail, redirect } from '@sveltejs/kit';
import type { Actions } from './$types';

export const actions: Actions = {
	default: async ({ request }) => {
		const data = await request.formData();
		const email = data.get('email');
		const password = data.get('password');

		if (!email) {
			return fail(400, { email, missing: true });
		}

		await login(email, password);
		throw redirect(303, '/dashboard');
	},
};
```

```svelte
<!-- +page.svelte -->
<script>
	export let form; // Contains return value from action
</script>

<form method="POST">
	<input name="email" value={form?.email ?? ''} />
	{#if form?.missing}
		<p class="error">Email is required</p>
	{/if}
	<button>Login</button>
</form>
```

### Named Actions

```typescript
export const actions: Actions = {
	login: async ({ request }) => { /* Handle login */ },
	register: async ({ request }) => { /* Handle registration */ },
};
```

```svelte
<form method="POST" action="?/login">...</form>
<form method="POST" action="?/register">...</form>
```

### Progressive Enhancement

Form works without JavaScript:

```svelte
<script>
	import { enhance } from '$app/forms';
</script>

<form method="POST" use:enhance>
	<!-- Works with or without JS -->
</form>
```

With custom handling:

```svelte
<form
	method="POST"
	use:enhance={({ formData, cancel }) => {
		// Before submit
		formData.append('timestamp', Date.now().toString());

		return async ({ result, update }) => {
			// After response
			if (result.type === 'success') {
				await update(); // Update form prop
			}
		};
	}}
>
	...
</form>
```

### Validation Pattern

```typescript
import { fail } from '@sveltejs/kit';
import { z } from 'zod';

const schema = z.object({
	email: z.string().email(),
	password: z.string().min(8),
});

export const actions = {
	default: async ({ request }) => {
		const data = await request.formData();
		const rawData = {
			email: data.get('email'),
			password: data.get('password'),
		};
		const result = schema.safeParse(rawData);
		if (!result.success) {
			return fail(400, {
				errors: result.error.flatten().fieldErrors,
				data: rawData,
			});
		}
		await createUser(result.data);
		throw redirect(303, '/welcome');
	},
};
```

### File Upload

```typescript
export const actions = {
	default: async ({ request }) => {
		const data = await request.formData();
		const file = data.get('file') as File;
		if (!file || file.size === 0) {
			return fail(400, { error: 'No file uploaded' });
		}
		const bytes = await file.arrayBuffer();
		const buffer = Buffer.from(bytes);
		await saveFile(buffer, file.name);
		throw redirect(303, '/uploads');
	},
};
```

```svelte
<form method="POST" enctype="multipart/form-data">
	<input type="file" name="file" required />
	<button>Upload</button>
</form>
```

### Key Rules

1. ✅ Actions must be in `+page.server.ts` (not `+page.ts`)
2. ✅ ALWAYS throw `redirect()` and `error()` (not return)
3. ✅ Return `fail()` for validation errors
4. ✅ Return only serializable data
5. ✅ Don't catch redirects/errors without rethrowing
6. ✅ Use `enhance` for progressive enhancement
7. ✅ Access FormData with `data.get('fieldName')`
