# Runes — Common Mistakes & Anti-Patterns

Reactivity mistakes that compile fine and behave wrongly.

**Last verified:** 2026-03-12

**Contents**

- Common Mistakes / Anti-Patterns
- Notes

---

## Common Mistakes / Anti-Patterns

### ❌ Using `$effect` for derived state

```svelte
<!-- WRONG -->
<script>
	let count = $state(0);
	let doubled = $state(0);
	$effect(() => { doubled = count * 2; });
</script>
<!-- RIGHT -->
<script>
	let count = $state(0);
	let doubled = $derived(count * 2);
</script>
```

### ❌ Using `$effect` when an event handler works

Per Svelte docs: "If you can put your side effects in an event handler, that's almost always preferable." Event handlers are predictable and run once per action.

```svelte
<!-- RIGHT -->
<script>
	let count = $state(0);
	function increment() {
		count++;
		console.log(`Count is now ${count}`); // side effect in handler
	}
</script>
<button onclick={increment}>Increment</button>
```

### ❌ Using `$effect` to sync linked values

Avoid effects for "connecting one value to another". Use `oninput` callbacks / function bindings:

```svelte
<script>
	let celsius = $state(0);
	let fahrenheit = $state(32);
	function updateFromCelsius(e) {
		celsius = +e.target.value;
		fahrenheit = (celsius * 9) / 5 + 32;
	}
	function updateFromFahrenheit(e) {
		fahrenheit = +e.target.value;
		celsius = ((fahrenheit - 32) * 5) / 9;
	}
</script>
<input type="number" value={celsius} oninput={updateFromCelsius} />
<input type="number" value={fahrenheit} oninput={updateFromFahrenheit} />
```

### ❌ Using `$effect` to sync async data into form state

```svelte
<!-- WRONG — $effect as escape hatch to sync query → form state -->
<script>
	let query = $derived(get_item({ id }))
	let name = $state('')
	$effect(() => { if (query.ready) name = query.current.name })
</script>
<input bind:value={name} />
```

**RIGHT — gate child behind `.ready`; child inits `$state` from prop once at mount:**

```svelte
<!-- Parent.svelte -->
<script>
	let query = $derived(get_item({ id }))
</script>
{#if !query.ready}
	<Skeleton />
{:else}
	<EditForm item={query.current} />
{/if}
```

```svelte
<!-- EditForm.svelte -->
<script>
	let { item } = $props()
	// svelte-ignore state_referenced_locally
	let form = $state({ name: item.name }) // init from prop at mount
</script>
<input bind:value={form.name} />
```

No `$effect` needed, no `state_unsafe_mutation` warning. Standard pattern for editable forms backed by async data.

### ❌ Optional chaining breaks effect reactivity

```svelte
<!-- WRONG — if particles is undefined, `scheme` is NEVER read, so no dependency -->
<script>
	$effect(() => { particles?.updateScheme(scheme); });
</script>
<!-- RIGHT — read scheme first to create dependency -->
<script>
	$effect(() => {
		const currentScheme = scheme;
		if (particles) particles.updateScheme(currentScheme);
	});
</script>
```

JS short-circuits optional chaining; if `particles` is nullish, `scheme` is never evaluated.

### ❌ Infinite loops in `$effect`

```svelte
<!-- WRONG -->
<script>
	let count = $state(0);
	$effect(() => { count++; }); // effect updates count → triggers effect…
</script>
```

Fixes: update _different_ state (`log.push(count)`), or read without subscribing via `untrack`:

```svelte
<script>
	import { untrack } from 'svelte';
	let count = $state(0);
	$effect(() => {
		const current = untrack(() => count); // read without creating dependency
	});
</script>
```

### ❌ Using `$effect` to sync state with DOM elements

```svelte
<!-- WRONG - Dialog sync via effect -->
<script>
	let is_open = $state(false);
	let dialog_element = $state<HTMLDialogElement>();
	$effect(() => {
		if (is_open) dialog_element?.showModal();
		else dialog_element?.close(); // fires 'close' event → handler → loop!
	});
</script>
<dialog bind:this={dialog_element} onclose={() => is_open = false}>
```

`dialog.close()` fires the native `close` event → your handler → loops/double-firing. **RIGHT** — state class with `@attach` register + call DOM methods directly:

```ts
// state.svelte.ts
class DialogState {
	dialog: HTMLDialogElement | null = null;
	is_open = $state(false);
	register = (el: HTMLDialogElement) => { this.dialog = el; return () => { this.dialog = null; }; };
	open() { if (!this.dialog?.open) { this.is_open = true; this.dialog?.showModal(); } }
	close() { this.is_open = false; this.dialog?.close(); }
}
```

```svelte
<dialog {@attach dialog_state.register} onclose={dialog_state.close}>
```

### ❌ Using runes inside functions

```svelte
<!-- WRONG -->
<script>
	function createCounter() {
		let count = $state(0); // ERROR - runes must be top-level
		return count;
	}
</script>
```

Fix: top-level runes, or reactive class fields (`class Counter { count = $state(0); }`). Runes must be statically analyzable at compile time.

### ❌ Mixing Svelte 4 and 5 syntax

```svelte
<!-- WRONG -->
<script>
	let count = $state(0);
	$: doubled = count * 2; // Mixing runes with reactive statements!
</script>
```

### ❌ Forgetting `$state`

```svelte
<script>
	let count = 0; // Not reactive in Svelte 5! UI won't update
</script>
<button onclick={() => count++}>{count}</button>
```

### ❌ Trying to bind without `$bindable`

```svelte
<!-- Child.svelte WRONG -->
<script>
	let { value } = $props(); // not bindable
</script>
<!-- Parent: <Child bind:value={text} /> → errors -->
<!-- RIGHT -->
<script>
	let { value = $bindable() } = $props();
</script>
```

### ❌ Mutating non-bindable props

```svelte
<!-- WRONG -->
<script>
	let { count } = $props(); // not bindable
	function increment() { count++; } // BAD - mutating parent's prop
</script>
<!-- RIGHT: use a callback (onIncrement) OR make it $bindable -->
```

### ❌ Not providing default for bindable / unnecessary bindable

```svelte
let { value = $bindable() } = $props();      // RISKY - undefined if parent omits
let { value = $bindable('default') } = $props(); // SAFER
let { label = 'Submit' } = $props();          // RIGHT - label needn't be bindable
```

### ❌ Forgetting `{@render}` for children

```svelte
<div>{children}</div>      <!-- WRONG - shows [object Object] -->
<div>{@render children()}</div> <!-- RIGHT -->
```

Children is a snippet, not a value.

### Performance / TS mistakes

- Don't wrap non-changing values in `$state` — use a plain `const` (`const API_URL = '...'`).
- Don't `$derived` a value used only once — inline it (`{count * 2}`).
- Always type props with an interface; bindable props must be **optional** in the type (`value?: string` with `$bindable('')`).

### Error messages

- **"Cannot access 'count' before initialization"** — rune used out of order / inside a function. Declare `$state` before `$derived` that uses it.
- **"Cannot read properties of undefined (reading '$effect')"** — rune used outside component scope (e.g. `<script context="module">`). Move to instance `<script>`.
- **"bind:value is not available on this component"** — forgot `$bindable()`.

## Notes

- Use `onclick` not `on:click`; `{@render children()}` in layouts.
- `$derived` can be reassigned (5.25+) — use `const` for read-only.
- Use `createContext` over `setContext`/`getContext` for type safety.
- Use `$inspect.trace` to debug reactivity issues.
- `$effect` doesn't run during SSR.
