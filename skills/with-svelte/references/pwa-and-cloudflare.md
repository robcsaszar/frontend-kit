# PWA & Cloudflare

PWA setup, plus Cloudflare/streaming gotchas.

**Last verified:** 2025-01-14

**Contents**

- PWA Setup with SvelteKit
- Cloudflare Gotchas

---

## PWA Setup with SvelteKit

### Basic PWA with Workbox

```bash
pnpm add -D workbox-precaching
```

#### Service Worker

```typescript
// src/service-worker.ts
/// <reference lib="webworker" />
import { precacheAndRoute } from 'workbox-precaching';

declare const self: ServiceWorkerGlobalScope;

precacheAndRoute(self.__WB_MANIFEST);
```

#### Manifest

```json
// static/manifest.json
{
	"name": "My App",
	"short_name": "App",
	"start_url": "/",
	"display": "standalone",
	"background_color": "#ffffff",
	"theme_color": "#ff3e00",
	"icons": [
		{
			"src": "/icon-192.png",
			"sizes": "192x192",
			"type": "image/png"
		},
		{
			"src": "/icon-512.png",
			"sizes": "512x512",
			"type": "image/png"
		}
	]
}
```

#### HTML Head

```svelte
<!-- src/app.html -->
<head>
	<link rel="manifest" href="/manifest.json" />
	<meta name="theme-color" content="#ff3e00" />
	<link rel="apple-touch-icon" href="/icon-192.png" />
</head>
```

#### SvelteKit Config

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-static';

export default {
	kit: {
		adapter: adapter({
			fallback: 'index.html',
		}),
		serviceWorker: {
			register: true,
		},
	},
};
```

### Offline-First Strategy

```typescript
// src/service-worker.ts
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { NetworkFirst, CacheFirst } from 'workbox-strategies';

declare const self: ServiceWorkerGlobalScope;

// Precache static assets
precacheAndRoute(self.__WB_MANIFEST);

// Cache API responses
registerRoute(
	({ url }) => url.pathname.startsWith('/api/'),
	new NetworkFirst({
		cacheName: 'api-cache',
	}),
);

// Cache images
registerRoute(
	({ request }) => request.destination === 'image',
	new CacheFirst({
		cacheName: 'image-cache',
	}),
);
```

### Testing PWA

1. Build: `pnpm build`
2. Preview: `pnpm preview`
3. Open DevTools → Application → Service Workers
4. Check "Offline" to test offline behavior

---

## Cloudflare Gotchas

### Streaming Broken

**Problem:** Cloudflare may remove `Transfer-Encoding: chunked` header, breaking HTML streaming in SvelteKit.

**Symptoms:**

- Page loads as one chunk instead of streaming
- `await` blocks don't show progressive loading
- Longer initial page load times

**Workarounds:**

1. **Disable compression** in Cloudflare dashboard (temporary)
2. **Use page rules** to bypass caching for dynamic routes
3. **Switch to Cloudflare Workers** adapter for more control

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare';

export default {
	kit: {
		adapter: adapter({
			routes: {
				include: ['/*'],
				exclude: ['<all>'],
			},
		}),
	},
};
```

### View Transitions Bug

**Problem:** Same-page view transitions may not work correctly.

**Fix:** Update BOTH SvelteKit AND Svelte to latest versions:

```bash
pnpm add @sveltejs/kit@latest svelte@latest
```

Both packages had related bugs that needed fixing together.

### Environment Variables

Cloudflare uses different env variable handling:

```typescript
// +page.server.ts
export const load = async ({ platform }) => {
	// Access via platform.env, not process.env
	const apiKey = platform?.env?.API_KEY;

	return { data };
};
```

### WebSocket Issues with Bun

**Problem:** Bun's WebSocket module incomplete for Cloudflare Workers.

**Workaround:** Use Node.js or tsx runner instead of Bun for WebSocket-heavy apps.

### DNS Proxy Settings

If using Cloudflare for DNS only (gray cloud), streaming works normally. Issues occur when traffic is proxied through Cloudflare (orange cloud).
