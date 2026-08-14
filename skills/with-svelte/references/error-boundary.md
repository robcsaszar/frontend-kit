# svelte:boundary

Component-level error boundaries and their failed/pending snippets.


**Contents**

- svelte:boundary Component

---

## svelte:boundary Component

> Available in Svelte 5.3+

### Two Purposes

1. **Error boundaries** - catch rendering errors
2. **Pending UI** - show loading state while `await` resolves

### Basic Error Boundary

```svelte
<svelte:boundary onerror={(e, reset) => console.error(e)}>
	<RiskyComponent />

	{#snippet failed(error, reset)}
		<p>Error: {error.message}</p>
		<button onclick={reset}>Try again</button>
	{/snippet}
</svelte:boundary>
```

### Pending UI (Loading States)

> **⚠️ Known Bug:** `<svelte:boundary>` + `{@const await}` causes
> infinite navigation loops during client-side page transitions when
> pages share async queries. See
> [sveltejs/svelte#17717](https://github.com/sveltejs/svelte/issues/17717).
> Use `{#await}` blocks until this is fixed.

With `experimental.async: true` (Svelte 5.36+), `{@const await}` is possible but **not safe for pages with shared queries or navigation**:

```svelte
<!-- ⚠️ Causes navigation loops - use {#await} instead -->
<svelte:boundary>
	{#snippet pending()}
		<LoadingSpinner />
	{/snippet}

	{@const data = await loadData()}
	<DataView {data} />
</svelte:boundary>
```

**Safe alternative:**

```svelte
{#await loadData()}
	<LoadingSpinner />
{:then data}
	<DataView {data} />
{:catch error}
	<p>Error: {error.message}</p>
{/await}
```

### Combined Error + Pending

Use `svelte:boundary` for **error catching only**, with `{#await}` for async data:

```svelte
<svelte:boundary onerror={logError}>
	{#snippet failed(error, reset)}
		<p>Failed to load user</p>
		<button onclick={reset}>Retry</button>
	{/snippet}

	{#await fetchUser()}
		<p>Loading user...</p>
	{:then user}
		<UserProfile {user} />
	{:catch error}
		<p>Error: {error.message}</p>
	{/await}
</svelte:boundary>
```

### What Gets Caught

**Caught:** Errors during rendering; errors in `$effect`.

**NOT Caught:** Event handler errors (`onclick`, etc.); errors after `setTimeout`; async errors outside boundary's await.

### vs +error.svelte

| Feature  | svelte:boundary         | +error.svelte |
| -------- | ----------------------- | ------------- |
| Scope    | Component subtree       | Route segment |
| Reset    | Built-in reset function | Navigate away |
| Pending  | Yes (pending snippet)   | No            |
| Use case | Component-level         | Page-level    |

### Nested Boundaries

Inner boundary catches first.

```svelte
<svelte:boundary>
	{#snippet failed(e)}
		<p>Outer caught: {e.message}</p>
	{/snippet}

	<svelte:boundary>
		{#snippet failed(e)}
			<p>Inner caught: {e.message}</p>
		{/snippet}

		<ComponentThatMightFail />
	</svelte:boundary>
</svelte:boundary>
```

### Key Points

- Use `svelte:boundary` for component-level error isolation; `+error.svelte` for route-level error pages
- `pending` snippet shows only on **initial** load (not on refresh - no flicker)
- `failed` snippet replaces content on error; `reset` function lets users retry
- Errors in event handlers are NOT caught
- Requires `experimental.async: true` in svelte.config.js for `{@const await}`
- **⚠️ Bug:** `{@const await}` + navigation causes infinite loops ([#17717](https://github.com/sveltejs/svelte/issues/17717)) — use `{#await}` instead
