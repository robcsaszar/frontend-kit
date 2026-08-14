# Svelte 5 — Async Reactivity

Async Svelte (`await` in components) and `hydratable`.

**Last verified:** 2026-03-12

**Contents**

- Async Svelte
- `hydratable`

---

## Async Svelte

If using Svelte 5.36+, you can use `await` directly in three places previously unavailable: at the top level of `<script>`, inside `$derived(...)`, and inside markup. **Experimental** — opt in via `experimental.async` in `svelte.config.js`; the flag will be removed in Svelte 6.

```js
// svelte.config.js
export default {
	compilerOptions: { experimental: { async: true } },
};
```

### Synchronized updates

When an `await` expression depends on state, changes are **not** reflected in the UI until the async work completes (UI never left inconsistent):

```svelte
<script>
	let a = $state(1);
	let b = $state(2);
	async function add(a, b) {
		await new Promise((f) => setTimeout(f, 500));
		return a + b;
	}
</script>
<input type="number" bind:value={a} />
<input type="number" bind:value={b} />
<p>{a} + {b} = {await add(a, b)}</p>
```

Incrementing `a` does **not** immediately show `2 + 2 = 3`; the text updates to `2 + 2 = 4` when `add` resolves. Updates can overlap — a fast update shows while an earlier slow one is ongoing.

### Concurrency

Independent `await` expressions in markup run in parallel:

```svelte
<p>{await one()}</p><p>{await two()}</p>
```

Sequential `await`s inside `<script>` / async functions run like normal async JS. Independent `$derived` expressions update independently (but run sequentially the first time):

```js
let a = $derived(await one());
let b = $derived(await two());
```

> Code like this triggers an `await_waterfall` warning.

### Loading states

Wrap content in `<svelte:boundary>` with a `pending` snippet (shown on first creation, not subsequent updates). After first resolution, detect subsequent async work with `$effect.pending()` (e.g. async-validation spinner). Use `settled()` for a promise that resolves when the current update completes:

```js
import { tick, settled } from 'svelte';
async function onclick() {
	updating = true;
	await tick(); // else change to `updating` is grouped with others, not reflected
	color = 'octarine';
	answer = 42;
	await settled();
	updating = false;
}
```

### Error handling / SSR / Forking

- Errors in `await` expressions bubble to the nearest `<svelte:boundary>` error boundary.
- SSR: `await render(App)` (`svelte/server`). SvelteKit does this for you. A `<svelte:boundary>` `pending` snippet renders during SSR while the rest is ignored; all `await`s outside such boundaries resolve before `render` returns.
- `fork(...)` (added 5.42) runs `await` expressions you _expect_ to happen soon (preloading). Mainly for frameworks.

```svelte
<script>
	import { fork } from 'svelte';
	let open = $state(false);
	/** @type {import('svelte').Fork | null} */
	let pending = null;
	function preload() { pending ??= fork(() => { open = true; }); }
	function discard() { pending?.discard(); pending = null; }
</script>
<button
	onpointerenter={preload}
	onpointerleave={discard}
	onclick={() => { pending?.commit(); pending = null; open = true; }}>open menu</button>
```

**Caveat:** as experimental, details (and `$effect.pending()`) may change outside a semver major. With `experimental.async` true, block effects (`{#if}`, `{#each}`) now run before an `$effect.pre`/`beforeUpdate` in the same component.

## `hydratable`

Solves the pitfall where awaited server data is re-fetched during client hydration (blocking it). Low-level API (usually used behind the scenes by data-fetching libraries; powers SvelteKit remote functions).

```svelte
<script>
	import { hydratable } from 'svelte';
	import { getUser } from 'my-database-library';
	// SSR: serializes & stashes result under the key, baked into `head` content.
	// Hydration: returns the serialized version instead of running getUser.
	// Post-hydration: subsequent calls just invoke getUser.
	const user = await hydratable('user', () => getUser());
</script>
<h1>{user.name}</h1>
```

Also for stable random/time values across SSR + hydration:

```ts
const rand = hydratable('random', () => Math.random());
```

Library authors: prefix keys with the library name to avoid conflicts.

**Serialization:** all returned data must be serializable. Uses [`devalue`](https://npmjs.com/package/devalue) — supports `Map`, `Set`, `URL`, `BigInt`, and (via Svelte magic) promises:

```svelte
<script>
	import { hydratable } from 'svelte';
	const promises = hydratable('random', () => ({
		one: Promise.resolve(1),
		two: Promise.resolve(2),
	}));
</script>
{await promises.one}
{await promises.two}
```

**CSP:** `hydratable` adds an inline `<script>` to the `head`. Provide a `nonce` to `render`:

```js
const nonce = crypto.randomUUID();
const { head, body } = await render(App, { csp: { nonce } });
response.headers.set('Content-Security-Policy', `script-src 'nonce-${nonce}'`);
```

A `nonce` must only be used when dynamically server-rendering one response. For static HTML use hashes:

```js
const { head, body, hashes } = await render(App, { csp: { hash: true } });
response.headers.set(
	'Content-Security-Policy',
	`script-src ${hashes.script.map((h) => `'${h}'`).join(' ')}`,
);
```

Prefer `nonce` over `hash` — `hash` will interfere with future streaming SSR.
