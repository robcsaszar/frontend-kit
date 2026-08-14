# Component Form Patterns

Form composition and validation inside components.

**Last verified:** 2025-01-14

**Contents**

- Form Patterns

---

## Form Patterns

### Form attribute trick

When you can't nest a form (e.g. inside tables), use the `form` attribute to associate inputs with a form anywhere in the document. Submit-with-Enter, FormData collection, and accessibility all work.

```svelte
<form id="add-item" action="?/add" method="POST"></form>
<table>
	<tbody>
		{#each items as item}
			<tr><td>{item.name}</td><td>{item.price}</td></tr>
		{/each}
		<tr>
			<td><input form="add-item" name="name" required /></td>
			<td><input form="add-item" name="price" type="number" required /></td>
			<td><button form="add-item">Add</button></td>
		</tr>
	</tbody>
</table>
```

### Default values and reset

Forms support `defaultValue` for easy resets:

```svelte
<script>
	let name = $state('');
</script>
<form onreset={() => (name = '')}>
	<input bind:value={name} defaultValue="" />
	<button type="submit">Save</button>
	<button type="reset">Reset</button>
</form>
```

### Progressive enhancement

```svelte
<script>
	import { enhance } from '$app/forms';
	let submitting = $state(false);
</script>
<form
	method="POST"
	use:enhance={() => {
		submitting = true;
		return async ({ update }) => { await update(); submitting = false; };
	}}
>
	<input name="email" type="email" required />
	<button disabled={submitting}>{submitting ? 'Saving...' : 'Save'}</button>
</form>
```

### Form validation with Valibot

```typescript
// +page.server.ts
import * as v from 'valibot';
import { fail } from '@sveltejs/kit';

const ContactSchema = v.object({
	email: v.pipe(v.string(), v.email()),
	message: v.pipe(v.string(), v.minLength(10)),
});

export const actions = {
	default: async ({ request }) => {
		const formData = await request.formData();
		const data = Object.fromEntries(formData);
		const result = v.safeParse(ContactSchema, data);
		if (!result.success) {
			return fail(400, { data, errors: v.flatten(result.issues) });
		}
		await saveContact(result.output);
	},
};
```

```svelte
<!-- +page.svelte -->
<script>
	let { form } = $props();
</script>
<form method="POST">
	<label>
		Email
		<input name="email" type="email" value={form?.data?.email ?? ''} />
		{#if form?.errors?.nested?.email}
			<span class="error">{form.errors.nested.email[0]}</span>
		{/if}
	</label>
	<label>
		Message
		<textarea name="message">{form?.data?.message ?? ''}</textarea>
		{#if form?.errors?.nested?.message}
			<span class="error">{form.errors.nested.message[0]}</span>
		{/if}
	</label>
	<button>Send</button>
</form>
```

### Multiple forms on one page

Use named actions (`?/subscribe`, `?/contact`) mapping to keys in `export const actions`:

```svelte
<form action="?/subscribe" method="POST"><input name="email" type="email" /><button>Subscribe</button></form>
<form action="?/contact" method="POST"><input name="message" /><button>Send</button></form>
```

```typescript
// +page.server.ts
export const actions = {
	subscribe: async ({ request }) => { /* … */ },
	contact: async ({ request }) => { /* … */ },
};
```
