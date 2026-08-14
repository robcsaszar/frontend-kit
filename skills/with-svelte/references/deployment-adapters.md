# Deployment Adapters

Adapter selection and Vite/pnpm build setup.

**Last verified:** 2025-01-14

**Contents**

- Quick Start
- Adapters
- Key Notes

---

## Quick Start

**pnpm 10+:** Add prepare script (postinstall disabled by default):

```json
{
	"scripts": {
		"prepare": "svelte-kit sync"
	}
}
```

**Vite 7:** Update both packages together:

```bash
pnpm add -D vite@7 @sveltejs/vite-plugin-svelte@6
```

## Adapters

```bash
# Static site
pnpm add -D @sveltejs/adapter-static

# Node server
pnpm add -D @sveltejs/adapter-node

# Cloudflare
pnpm add -D @sveltejs/adapter-cloudflare
```

## Key Notes

- Cloudflare may strip `Transfer-Encoding: chunked` (breaks streaming)
- Library authors: include `svelte` in keywords AND peerDependencies
- Single-file bundle: `kit.output.bundleStrategy: 'single'`

---
