# SSR & Hydration

Server rendering, hydration mismatches, and browser-only work.


**Contents**

- SSR & Hydration

---

## SSR & Hydration

### The Problem

SvelteKit runs on server (SSR) then hydrates in browser. Code using browser APIs (`window`, `document`, `localStorage`) fails on server.

### Solution: Check for Browser

```typescript
import { browser } from '$app/environment';

// In load function
export const load = async () => {
	const theme = browser ? localStorage.getItem('theme') : 'light';
	return { theme };
};
```

```svelte
<!-- In component -->
<script>
	import { browser } from '$app/environment';
	import { onMount } from 'svelte';

	let data = $state(null);

	// Option 1: browser check
	if (browser) {
		data = localStorage.getItem('data');
	}

	// Option 2: onMount (only runs in browser)
	onMount(() => {
		data = localStorage.getItem('data');
	});

	// Option 3: $effect with browser check
	$effect(() => {
		if (browser) {
			data = localStorage.getItem('data');
		}
	});
</script>
```

### Common Mistakes

**❌ Using window without check** — `window.innerWidth` in a load function errors on server. Guard with `browser ? window.innerWidth : 1024`.

**❌ Accessing DOM in load** — `document.getElementById(...)` errors on server. Do DOM work in `onMount` instead.

### Disable SSR (Not Recommended)

```typescript
// +page.ts
export const ssr = false; // Disables SSR for this page
```

Only use when absolutely necessary (e.g., heavy Canvas/WebGL).
