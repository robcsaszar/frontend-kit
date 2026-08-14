# Svelte 5 Runes — Core Reactivity

`$state`, `$derived`, `$effect`, and when to reach for each.

**Last verified:** 2026-03-12

**Contents**

- Quick Start
- Decision Matrix
- `$state`
- `$derived`
- `$effect`
- `$effect` vs `$derived`

---

## Quick Start

**Which rune?** Props: `$props()` | Bindable: `$bindable()` | Computed: `$derived()` | Side effect: `$effect()` | State: `$state()`

**Key rules:** Runes are top-level only. `$derived` can be overridden (use `const` for read-only). Don't mix Svelte 4/5 syntax. Objects/arrays are deeply reactive by default.

```svelte
<script>
	let count = $state(0); // Mutable state
	const doubled = $derived(count * 2); // Computed (const = read-only)

	$effect(() => {
		console.log(`Count is ${count}`); // Side effect
	});
</script>

<button onclick={() => count++}>
	{count} (doubled: {doubled})
</button>
```

## Decision Matrix

| Need                 | Use                    | Why                                    |
| -------------------- | ---------------------- | -------------------------------------- |
| Mutable state        | `$state()`             | Base reactive variable                 |
| Computed value       | `$derived()`           | Auto-updates when dependencies change  |
| Complex computation  | `$derived.by()`        | Use function body for multi-line logic |
| Large immutable data | `$state.raw()`         | Skip deep reactivity for performance   |
| Read-only snapshot   | `$state.snapshot()`    | Get plain JS value, no proxy           |
| Side effect          | `$effect()`            | Run code when dependencies change      |
| Pre-DOM effect       | `$effect.pre()`        | Run before DOM updates                 |
| Accept props         | `$props()`             | Declare component props                |
| Bindable prop        | `$bindable()`          | Allow parent to bind to prop           |
| Reactive class field | `$state` (class field) | Reactive property in class             |

## `$state`

Only use `$state` for variables that should be _reactive_ — i.e. that cause an `$effect`, `$derived` or template expression to update. Everything else can be a normal variable.

```svelte
<script>
	let count = $state(0); // Primitive
	let user = $state({ name: 'Alex', profile: { age: 30 } }); // Object (DEEP reactive)
	let items = $state([1, 2, 3]); // Array (DEEP reactive)
</script>
```

- Must be top-level in component (or a class field).
- Objects/arrays (`$state({...})` / `$state([...])`) are made **deeply reactive** — nested mutations trigger updates. The trade-off: objects must be proxied, which has performance overhead.
- Mutate nested properties directly: `user.profile.age = 31` ✅ (works!)
- Reassigning also works: `user = { ...user, name: 'Bo' }` ✅

### Deep reactivity works by default

```svelte
<script>
	let user = $state({ profile: { name: 'Alex' } });
	function updateName() {
		user.profile.name = 'Bo'; // This DOES trigger reactivity!
	}
</script>
<p>{user.profile.name}</p> <!-- Will update correctly -->
```

Array methods and nested mutations all work because `$state()` creates deep proxies:

```svelte
<script>
	let items = $state([1, 2, 3]);
	function addItem() {
		items.push(4);                 // ✅ Triggers reactivity
		items[items.length] = 5;       // ✅ Also works
		items = [...items, 6];         // ✅ Also works
	}
	let data = $state({ items: [1, 2, 3], nested: { arr: [10, 20] } });
	data.items.push(4);        // ✅ Deep reactivity
	data.nested.arr.push(30);  // ✅ Deeply reactive
</script>
```

### `$state.raw` (performance)

For large objects that are only ever **reassigned** (not mutated) — e.g. API responses — use `$state.raw` to skip proxy overhead.

```svelte
<script>
	let config = $state.raw(hugeConfigObject); // No proxy overhead
	let apiData = $state.raw(data);
	// Later: apiData = newData; (full replacement)

	// If you WILL mutate nested properties, use $state:
	let user = $state({ profile: { name: 'Alex' } });
	user.profile.name = 'Bo'; // Works with deep reactivity
</script>
```

**Use `$state.raw()` when:** data is large/immutable; you'll fully replace not mutate; performance-critical. **Don't when:** you need to mutate nested properties and see UI updates; data is small/medium.

### `$state.snapshot`

Extract plain JS values from proxies:

```svelte
<script>
	let user = $state({ name: 'Alex', age: 30 });
	function saveToAPI() {
		const plain = $state.snapshot(user); // Get plain object
		fetch('/api/users', { body: JSON.stringify(plain) });
	}
</script>
```

## `$derived`

To compute something from state, use `$derived` rather than `$effect`:

```js
// do this
let square = $derived(num * num);

// don't do this
let square;
$effect(() => {
	square = num * num;
});
```

> `$derived` is given an expression, _not_ a function. If you need a function (complex expression) use `$derived.by`.

```svelte
<script>
	let count = $state(0);
	let doubled = $derived(count * 2); // Simple
	let message = $derived.by(() => {
		if (count === 0) return 'Zero';
		return count > 10 ? 'High' : 'Low';
	});
</script>
```

- **Deriveds are writable** — as of Svelte 5.25+ you can reassign them, but they re-evaluate when their expression changes. Use `const` to make truly read-only.
- Auto-tracks dependencies. **Lazy** — only computes when accessed; garbage-collectable.
- If the derived expression is an object/array, it is returned as-is — **not** made deeply reactive. You can use `$state` inside `$derived.by` in the rare cases you need this.

## `$effect`

**Effects are an escape hatch and should mostly be avoided.** In particular, avoid updating state inside effects.

```text
Need to react to state change?
├─ Can use event handler? → USE EVENT HANDLER (preferred)
├─ Is it a computed value? → USE $derived
├─ Is it DOM-specific? → USE @attach
└─ External side effect? → USE $effect (with cleanup)
```

- If you need to sync state to an external library (e.g. D3), it is often neater to use `{@attach ...}`.
- If you need code in response to user interaction, put it directly in an event handler or use a function binding.
- If you need to log values for debugging, use `$inspect`.
- If you need to observe something external to Svelte, use `createSubscriber`.

Never wrap effect contents in `if (browser) {...}` — **effects do not run on the server**.

```svelte
<script>
	let count = $state(0);
	$effect(() => {
		console.log(`Count changed to ${count}`);
		document.title = `Count: ${count}`;
	});

	// $effect with cleanup
	$effect(() => {
		const interval = setInterval(() => {...}, 1000);
		return () => clearInterval(interval); // runs on re-run or unmount
	});
</script>
```

**Legitimate uses:** logging/analytics; updating external state (localStorage, document.title); setting up/tearing down subscriptions; third-party library integration (when @attach isn't suitable).

**Key points:** eager execution (runs whenever deps change until destroyed); lifecycle-bound (only in effect roots / components); runs after DOM (use `$effect.pre` for pre-DOM); no SSR; return cleanup function; don't update state the effect depends on (infinite loop!).

**Why `$derived` is preferred for computed values:** `$derived` is lazy + garbage-collectable with no lifecycle management; `$effect` is eager and keeps running until destroyed.

### `$effect.pre`

Runs BEFORE DOM updates. Useful for measuring DOM before changes.

```svelte
<script>
	let element = $state(null);
	$effect.pre(() => { /* Runs BEFORE DOM updates */ });
</script>
```

## `$effect` vs `$derived`

- **`$derived`** — transforming data; computing from other state; value used in template; read-only computed property.
- **`$effect`** — logging/analytics; updating external state (localStorage, DOM); fetching data; subscriptions (intervals, listeners); any operation with side effects.
