# Snippets — {#snippet} / {@render}

Snippets, render tags, and replacing slots.

**Last verified:** 2026-03-12

**Contents**

- Snippets — `{#snippet}` / `{@render}`

---

## Snippets — `{#snippet}` / `{@render}`

Snippets are reusable chunks of markup instantiated with `{@render ...}` or passed to components as props. They must be declared within the template. Instead of duplicative code:

```svelte
{#snippet figure(image)}
	<figure>
		<img src={image.src} alt={image.caption} width={image.width} height={image.height} />
		<figcaption>{image.caption}</figcaption>
	</figure>
{/snippet}

{#each images as image}
	{#if image.href}
		<a href={image.href}>{@render figure(image)}</a>
	{:else}
		{@render figure(image)}
	{/if}
{/each}
```

Like functions, snippets can have any number of parameters, with default values and destructuring. **You cannot use rest parameters.**

### `{@render}`

```svelte
{#snippet sum(a, b)}<p>{a} + {b} = {a + b}</p>{/snippet}
{@render sum(1, 2)}
```

The expression can be an identifier or any JS expression: `{@render (cool ? coolSnippet : lameSnippet)()}`.

**Optional snippets** — optional chaining, or `{#if}`/`{:else}` for fallback:

```svelte
{@render children?.()}
{#if children}
	{@render children()}
{:else}
	<p>fallback content</p>
{/if}
```

### Snippet scope

Snippets can be declared anywhere and can reference values from `<script>` or `{#each}` blocks. They are 'visible' to everything in the same lexical scope (siblings and their children). They can reference themselves and each other (recursion):

```svelte
{#snippet blastoff()}<span>🚀</span>{/snippet}
{#snippet countdown(n)}
	{#if n > 0}
		<span>{n}...</span>
		{@render countdown(n - 1)}
	{:else}
		{@render blastoff()}
	{/if}
{/snippet}
{@render countdown(10)}
```

### Passing snippets to components

**Explicit props** — snippets are values like any other:

```svelte
{#snippet header()}<th>fruit</th><th>qty</th>{/snippet}
{#snippet row(d)}<td>{d.name}</td><td>{d.qty}</td>{/snippet}
<Table data={fruits} {header} {row} />
```

**Implicit props** — snippets declared directly inside a component become props on it:

```svelte
<Table data={fruits}>
	{#snippet header()}<th>fruit</th>{/snippet}
	{#snippet row(d)}<td>{d.name}</td>{/snippet}
</Table>
```

**Implicit `children` snippet** — any non-snippet content inside the component tags becomes the `children` snippet:

```svelte
<!-- App.svelte --> <Button>click me</Button>
<!-- Button.svelte -->
<script>
	let { children } = $props();
</script>
<button>{@render children()}</button>
```

> You cannot have a prop called `children` if you also have content inside the component — avoid props with that name.

### Snippets with parameters

```svelte
<!-- List.svelte -->
<script>
	let { items, children } = $props();
</script>
<ul>
	{#each items as item, i}
		<li>{@render children(item, i)}</li>
	{/each}
</ul>
<!-- Usage -->
<List items={users}>
	{#snippet children(item, index)}{index}: {item.name}{/snippet}
</List>
```

### Typing snippets

Snippets implement the `Snippet` interface from `'svelte'`. The type argument is a tuple (snippets can have multiple parameters).

```svelte
<script lang="ts">
	import type { Snippet } from 'svelte';
	interface Props {
		children: Snippet;
		header?: Snippet;
		item?: Snippet<[{ name: string; age: number }]>; // with params
	}
	let { children, header, item }: Props = $props();
</script>
```

Tighten with a generic so two props share a type:

```svelte
<script lang="ts" generics="T">
	import type { Snippet } from 'svelte';
	let { data, row }: { data: T[]; row: Snippet<[T]> } = $props();
</script>
```

### Exporting snippets

Top-level snippets can be exported from a `<script module>` for use in other components, provided they don't reference non-module `<script>` declarations (directly or indirectly). _Requires Svelte 5.5.0+._

```svelte
<script module>
	export { add };
</script>
{#snippet add(a, b)}{a} + {b} = {a + b}{/snippet}
```

Snippets can also be created with `createRawSnippet` (advanced use cases).

### Snippets vs slots

In Svelte 4, content was passed via slots; snippets are more powerful and flexible, and slots are **deprecated** in Svelte 5.

| Feature          | Svelte 4 (Slots)               | Svelte 5 (Snippets + Children)         |
| ---------------- | ------------------------------ | -------------------------------------- |
| Default content  | `<slot />`                     | `{@render children()}`                 |
| Named content    | `<slot name="header" />`       | `{@render header()}`                   |
| Provide content  | `<div slot="header">...</div>` | `{#snippet header()}...{/snippet}`     |
| Slot props       | `<slot item={data} />`         | `{@render item(data)}`                 |
| Fallback content | `<slot>Fallback</slot>`        | `{@render children?.() ?? 'Fallback'}` |

Why snippets are better: more explicit (props show what content exists); better TS support (typed parameters); composable (passed around like functions); cleaner (no `let:prop`); more powerful (reusable within components); consistent (everything is a prop).

**Slot prop / `$$slots` migration:**

```svelte
<!-- Before -->
{#each items as item}<slot {item} />{/each}
{#if $$slots.header}<slot name="header" />{:else}<h1>Default</h1>{/if}
<!-- After -->
{#each items as item}{@render children(item)}{/each}
{#if header}{@render header()}{:else}<h1>Default</h1>{/if}
```

### Snippet common mistakes

```svelte
{@render children}      <!-- ❌ Missing () -->
{@render children()}    <!-- ✅ -->
<slot />                <!-- ❌ Don't mix slot with snippet syntax -->
{@render header()}      <!-- ❌ Errors if header not provided; use {#if} or ?.() -->
```

Always declare `children` in `$props()` before rendering it.
