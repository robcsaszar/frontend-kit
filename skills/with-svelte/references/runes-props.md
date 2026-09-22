# Svelte 5 Runes — Props & Class Fields

`$props`, `$bindable`, reactive class fields, `svelte/reactivity` built-ins, `createSubscriber`, and `$inspect`.

**Last verified:** 2026-09-15

**Contents**

- `$props`
- `$bindable`
- Reactive class fields
- Reactive built-ins from `svelte/reactivity`
- `createSubscriber` — external observables
- `$inspect`

---

## `$props`

Treat props as though they will change. Values that depend on props should usually use `$derived`:

```js
let { type } = $props();
let color = $derived(type === 'danger' ? 'red' : 'green'); // do this
let color = type === 'danger' ? 'red' : 'green'; // don't — won't update if `type` changes
```

```svelte
<script>
	let { name, age = 18, ...rest } = $props(); // Destructure with defaults + rest
	// OR
	let props = $props(); // props.name, props.age
</script>
```

- Replaces `export let`. Props are reactive automatically.

### TypeScript

```svelte
<script lang="ts">
	interface Props {
		name: string;
		age?: number; // Optional with default
	}
	let { name, age = 18 }: Props = $props();
</script>
```

### Generic components

```svelte
<script lang="ts" generics="T">
	interface Props<T> {
		items: T[];
		selected?: T;
		onSelect?: (item: T) => void;
	}
	let { items, selected, onSelect }: Props<T> = $props();
</script>
```

### Props are reactive — don't over-derive

```svelte
<!-- UNNECESSARY -->
<script>
	let { count } = $props();
	let doubled = $derived(count * 2); // Overkill if used once
</script>
<p>{doubled}</p>

<!-- SIMPLER -->
<script>
	let { count } = $props();
</script>
<p>{count * 2}</p>
```

Use `$derived` when the value is used multiple times, computation is expensive, or you derive from multiple props.

## `$bindable`

Makes a prop two-way bindable so the parent can use `bind:propName`.

```svelte
<!-- Child.svelte -->
<script>
	let { value = $bindable() } = $props();
</script>
<input bind:value />

<!-- Parent.svelte -->
<script>
	let text = $state('');
</script>
<Child bind:value={text} />
<p>You typed: {text}</p>
```

- Provide a default: `$bindable('default')`.
- Multiple bindable props OK: `let { min = $bindable(0), max = $bindable(100) } = $props();`

### Props vs Bindable decision tree

```text
Parent needs to read child state?
├─ No  → Just pass callbacks (controlled component)
└─ Yes → Parent needs to UPDATE child state?
    ├─ No  → Callback to notify parent (onChange pattern)
    └─ Yes → Use $bindable (two-way binding)
```

**Controlled component (no $bindable)** — parent fully controls state:

```svelte
<!-- Counter.svelte -->
<script>
	let { count, onIncrement } = $props();
</script>
<button onclick={onIncrement}>Count: {count}</button>
<!-- Usage: <Counter {count} onIncrement={() => count++} /> -->
```

**Hybrid: bindable with callback:**

```svelte
<script>
	let { value = $bindable(50), onChange } = $props();
	function handleChange() { onChange?.(value); }
</script>
<input type="range" bind:value oninput={handleChange} />
```

**Rule of thumb:** Only use `$bindable` when the parent _needs_ to update the prop value.

## Reactive class fields

```svelte
<script>
	class Counter {
		count = $state(0);
		doubled = $derived(this.count * 2);
		increment() { this.count++; }
	}
	const counter = new Counter();
</script>
<button onclick={() => counter.increment()}>
	{counter.count} (doubled: {counter.doubled})
</button>
```

Use classes with `$state` fields to share reactivity between components, instead of stores.

## Reactive built-ins from `svelte/reactivity`

Native `Map`, `Set`, `Date`, and `URL` are **not** made reactive by `$state` — the proxy cannot see through their internal slots, so `map.set(k, v)` updates nothing. Use the drop-in replacements, which are reactive per-key:

```js
import { SvelteMap, SvelteSet, SvelteDate, SvelteURL } from 'svelte/reactivity';

const selected = new SvelteSet();     // not: $state(new Set())
selected.add(id);                     // re-runs only what read this key
```

`MediaQuery` (from the same module) wraps `window.matchMedia` with a reactive `.current`. `prefersReducedMotion` in `svelte/motion` is one. The hand-rolled version below is how `MediaQuery` is built, and the pattern to copy for any other external event source.

## `createSubscriber` — external observables

_Available since 5.7.0._ From `svelte/reactivity`. Integrates external event-based systems (MediaQuery, IntersectionObserver, WebSocket) with Svelte reactivity **without `$effect`**.

If `subscribe` is called inside an effect (incl. via a getter), the `start` callback is called with an `update` function; calling `update` re-runs the effect. If `start` returns a cleanup function, it's called when the effect is destroyed. With multiple effects, `start` runs once and teardown runs when all are destroyed.

```js
import { createSubscriber } from 'svelte/reactivity';
import { on } from 'svelte/events';

export class MediaQuery {
	#query;
	#subscribe;
	constructor(query) {
		this.#query = window.matchMedia(`(${query})`);
		this.#subscribe = createSubscriber((update) => {
			const off = on(this.#query, 'change', update);
			return () => off();
		});
	}
	get current() {
		this.#subscribe(); // makes the getter reactive if read in an effect
		return this.#query.matches;
	}
}
```

```dts
function createSubscriber(
	start: (update: () => void) => (() => void) | void
): () => void;
```

Another example (browser location):

```ts
function createLocationStore() {
	let location = window.location.href;
	const subscribe = createSubscriber((update) => {
		const handler = () => { location = window.location.href; update(); };
		window.addEventListener('popstate', handler);
		return () => window.removeEventListener('popstate', handler);
	});
	return { get href() { subscribe(); return location; } };
}
```

**When:** wrapping browser APIs, third-party event emitters, or any external source that doesn't integrate with Svelte's reactivity natively.

## `$inspect`

> `$inspect` only works during development. In a production build it becomes a noop.

Roughly equivalent to `console.log`, but re-runs whenever its argument changes. Tracks reactive state **deeply**.

```svelte
<script>
	let count = $state(0);
	let message = $state('hello');
	$inspect(count, message); // logs when `count` or `message` change
</script>
```

On updates a stack trace is printed (except in the playground).

### `$inspect(...).with`

Returns a `with` property; invoke with a callback used instead of `console.log`. First arg is `"init"` or `"update"`; rest are the inspected values.

```svelte
<script>
	let count = $state(0);
	$inspect(count).with((type, count) => {
		if (type === 'update') {
			debugger; // or console.trace, etc.
		}
	});
</script>
```

### `$inspect.trace`

_Added in 5.14._ Debugging tool for reactivity. If something isn't updating properly or runs more than it should, add `$inspect.trace(label)` as the **first statement** of an `$effect` or `$derived.by` (or any function they call) to trace dependencies and discover which one triggered an update.

```svelte
<script>
	let count = $state(0);
	let name = $state('world');

	$effect(() => {
		$inspect.trace('greeting effect'); // must be first statement
		console.log(`Hello ${name}, count is ${count}`);
	});

	const message = $derived.by(() => {
		$inspect.trace('message derived');
		return `${name}: ${count}`;
	});
</script>
```

`$inspect.trace` takes an optional first argument used as the label. **When:** something not updating when it should; an effect/derived running more than expected; identifying which dependency triggered a re-run. Remove before production.
