# Template Tags & Bindings

`{@html}`, `{@const}`, `{@debug}`, keyed each, function bindings, and global event elements.

**Last verified:** 2026-03-12

**Contents**

- `{@html ...}`
- `{@const ...}`
- `{@debug ...}`
- Each blocks — always keyed, never index
- `bind:` — function bindings
- `<svelte:window>` / `<svelte:document>` — global events

---

## `{@html ...}`

Renders raw HTML strings. **Use with caution — never render untrusted content.** Always sanitize user-provided HTML (e.g. DOMPurify).

```svelte
<script>
	import DOMPurify from 'dompurify';
	let userContent = $state('');
	const sanitized = $derived(DOMPurify.sanitize(userContent));
</script>
{@html sanitized}
```

Common uses: markdown→HTML, CMS content, syntax-highlighted code blocks.

## `{@const ...}`

Declares local constants within template blocks (`{#each}`, `{#if}`). Avoids recalculating values, improves readability, scoped to the block.

```svelte
{#each items as item}
	{@const fullName = `${item.firstName} ${item.lastName}`}
	{@const isLongName = fullName.length > 20}
	<div class:truncate={isLongName}>{fullName}</div>
{/each}
```

## `{@debug ...}`

Pauses execution and opens devtools when the specified values change. Remove before production; use specific variables, not entire objects. `{@debug}` with no args pauses on every update.

```svelte
{@debug count, items}
```

## Each blocks — always keyed, never index

Prefer keyed each blocks for performance — Svelte can surgically insert/remove/reorder items rather than updating existing DOM in place.

```svelte
{#each items as item (item.id)}
	<li>{item.name} x {item.qty}</li>
{/each}
<!-- with index -->
{#each items as item, i (item.id)}
	<li>{i + 1}: {item.name} x {item.qty}</li>
{/each}
```

The key **must uniquely identify the object** — do NOT use the index. Strings/numbers are recommended (identity persists when objects change).

```svelte
{#each items as item, i (i)}    <!-- WRONG - index as key -->
{#each items as item (item.id)} <!-- RIGHT - unique identifier -->
```

**Without key:** removing item B from [A,B,C] updates node 2 to show C's data and removes the last node. **With key:** B's DOM node is actually removed, leaving A and C untouched.

You can use destructuring/rest patterns in each blocks, **but avoid destructuring if you need to mutate the item** — the destructured value is disconnected from the original.

```svelte
{#each items as { count } (item.id)}<input bind:value={count} />{/each}     <!-- WRONG -->
{#each items as item (item.id)}<input bind:value={item.count} />{/each}      <!-- RIGHT -->
```

## `bind:` — function bindings

Use `bind:property={get, set}`, where `get`/`set` are functions, to perform validation/transformation. _Available in Svelte 5.9.0+._

```svelte
<input bind:value={() => value, (v) => (value = v.toLowerCase())} />
```

For readonly bindings (e.g. dimensions), the `get` value should be `null`:

```svelte
<div bind:clientWidth={null, redraw} bind:clientHeight={null, redraw}>...</div>
```

## `<svelte:window>` / `<svelte:document>` — global events

Use these for window/document event listeners. Avoid `onMount` or `$effect` — they auto-clean-up listeners when the component is destroyed.

```svelte
<svelte:window onkeydown={handleKeydown} onscroll={handleScroll} />
<svelte:document onvisibilitychange={handleVisibility} />
```

Common patterns:

```svelte
<!-- Keyboard shortcuts -->
<svelte:window onkeydown={(e) => {
	if (e.key === 'Escape') closeModal();
	if (e.ctrlKey && e.key === 's') { e.preventDefault(); save(); }
}} />
<!-- Online/offline -->
<svelte:window ononline={() => status = 'online'} onoffline={() => status = 'offline'} />
<!-- Bindable window properties -->
<svelte:window bind:innerWidth bind:innerHeight bind:scrollX bind:scrollY bind:online />
```
