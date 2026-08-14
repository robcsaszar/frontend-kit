# Attachments — {@attach}

`{@attach}` (Svelte 5.29+) and migrating from `use:` actions.

**Last verified:** 2026-03-12

**Contents**

- Quick Reference
- `{@attach}` (Svelte 5.29+)

---

## Quick Reference

| Directive          | Purpose                        | Reactive? |
| ------------------ | ------------------------------ | --------- |
| `{@attach}`        | DOM manipulation, 3rd-party    | Yes       |
| `{@html}`          | Render raw HTML strings        | Yes       |
| `{@render}`        | Render snippets                | Yes       |
| `{@const}`         | Local constants in blocks      | N/A       |
| `{@debug}`         | Pause debugger on value change | N/A       |
| `{#each (key)}`    | Keyed iteration (always key!)  | Yes       |
| `<svelte:window>`  | Window event listeners         | N/A       |

## `{@attach}` (Svelte 5.29+)

**The reactive alternative to `use:` actions.** Attachments are functions that run in an effect when an element is mounted to the DOM or when state read inside the function updates. Optionally they return a function called before the attachment re-runs, or after the element is removed from the DOM. An element can have any number of attachments.

```svelte
<script>
	/** @type {import('svelte/attachments').Attachment} */
	function myAttachment(element) {
		console.log(element.nodeName); // 'DIV'
		return () => { console.log('cleaning up'); };
	}
</script>
<div {@attach myAttachment}>...</div>
```

### `@attach` vs `use:` actions

| Feature               | `use:`  | `@attach`           |
| --------------------- | ------- | ------------------- |
| Re-runs on arg change | No      | **Yes**             |
| Composable            | Limited | **Fully**           |
| Pass through props    | Manual  | **Auto via spread** |
| Convert legacy        | N/A     | `fromAction()`      |

Attachments are **fully reactive** — `{@attach foo(bar)}` re-runs on changes to `foo` _or_ `bar` (or any state read inside `foo`). Actions only run once on mount.

```svelte
<!-- use: - runs ONCE, ignores content changes -->
<button use:tooltip={content}>Won't update</button>
<!-- @attach - re-runs when content changes -->
<button {@attach tooltip(content)}>Updates!</button>
```

**Still use `use:` actions when:** legacy code/libraries not yet updated; you specifically DON'T want re-runs on argument change; simple one-time DOM setup with no reactive dependencies.

### Attachment factories

A function that _returns_ an attachment, enabling parameterized behavior. Since `tooltip(content)` runs inside an effect, the attachment is destroyed and recreated whenever `content` changes.

```svelte
<script>
	import tippy from 'tippy.js';
	let content = $state('Hello!');
	/** @returns {import('svelte/attachments').Attachment} */
	function tooltip(content) {
		return (element) => {
			const tooltip = tippy(element, { content });
			return tooltip.destroy;
		};
	}
</script>
<input bind:value={content} />
<button {@attach tooltip(content)}>Hover me</button>
```

### Inline attachments

```svelte
<canvas
	width={32} height={32}
	{@attach (canvas) => {
		const context = canvas.getContext('2d');
		$effect(() => {
			context.fillStyle = color;
			context.fillRect(0, 0, canvas.width, canvas.height);
		});
	}}
></canvas>
```

> The nested effect runs whenever `color` changes; the outer effect (`getContext`) runs only once since it reads no reactive state. Good for canvas where you need reactive updates without recreating the context.

### Conditional attachments

Falsy values (`false`/`undefined`) are treated as no attachment:

```svelte
<div {@attach enabled && myAttachment}>...</div>
```

### Passing attachments through components

On a component, `{@attach ...}` creates a prop keyed by a `Symbol`. If the component spreads props onto an element, the element receives the attachments. Enables "augmented element" wrapper components.

```svelte
<!-- Button.svelte -->
<script>
	/** @type {import('svelte/elements').HTMLButtonAttributes} */
	let { children, ...props } = $props();
</script>
<button {...props}>
	{@render children?.()}
</button>

<!-- App.svelte -->
<Button {@attach tooltip('Click me for help')}>Help</Button>
```

### Controlling when attachments re-run

For expensive/unavoidable setup work, pass data via an accessor function and read it in a child effect, so only the cheap update re-runs:

```js
function foo(getBar) {
	return (node) => {
		veryExpensiveSetupWork(node);
		$effect(() => { update(node, getBar()); }); // cheap, re-runs on data change
	};
}
```

```svelte
<!-- Pass accessor function, not the data directly -->
<div {@attach expensiveChart(() => data)}>Chart</div>
```

### Converting legacy actions — `fromAction`

```svelte
<script>
	import { fromAction } from 'svelte/attachments';
	import { someAction } from 'some-legacy-library';
	const attached = fromAction(someAction);
</script>
<div {@attach attached(options)}>...</div>
```

To add attachments to an object spread onto a component/element programmatically, use `createAttachmentKey` from `svelte/attachments`.

### Multiple attachments

```svelte
<button
	{@attach tooltip('Help text')}
	{@attach trackClicks}
	{@attach highlight(isActive ? 'yellow' : 'transparent')}
>Multi-attached button</button>
```

### DOM-controlling libraries (ProseMirror, etc.)

Combine `@attach` with the imperative `mount`/`unmount` API for libraries that control their own DOM segment:

```svelte
<script>
	import { mount, unmount } from 'svelte';
	import MyComponent from './MyComponent.svelte';
	function proseMirrorNodeView(node) {
		return (dom) => {
			const component = mount(MyComponent, { target: dom, props: { data: node.attrs } });
			return () => unmount(component);
		};
	}
</script>
```

### Registering elements with global state

Use `@attach` to register DOM elements with state classes — avoids `$effect` sync loops and `bind:this` chains, and avoids event loops from `dialog.close()` firing `onclose`.

```ts
// modal-state.svelte.ts
class ModalState {
	dialog: HTMLDialogElement | null = null;
	input: HTMLInputElement | null = null;
	is_open = $state(false);
	register = (el: HTMLDialogElement) => { this.dialog = el; return () => { this.dialog = null; }; };
	register_input = (el: HTMLInputElement) => { this.input = el; return () => { this.input = null; }; };
	open() { if (!this.dialog?.open) { this.is_open = true; this.dialog?.showModal(); this.input?.focus(); } }
	close() { this.is_open = false; this.dialog?.close(); }
	toggle() { this.is_open ? this.close() : this.open(); }
}
export const modal_state = new ModalState();
```

```svelte
<!-- Modal.svelte -->
<dialog {@attach modal_state.register} onclose={modal_state.close}>
	<input {@attach modal_state.register_input} />
</dialog>
<!-- Anywhere else - no component ref needed -->
<button onclick={modal_state.toggle}>Open Modal</button>
```
