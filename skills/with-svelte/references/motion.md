# Motion — Springs, Tweens, Transitions

`svelte/motion` (`Spring`, `Tween`, `prefersReducedMotion`), `svelte/transition`, `animate:`, `svelte/easing`.

**Last verified:** 2026-09-15

**Contents**

- Choosing: CSS / `Tween` / `Spring`
- `Spring` — physics-driven values
- `Tween` — duration-driven values
- Deprecated: `spring()` / `tweened()` stores
- `prefersReducedMotion`
- SSR and cleanup
- Recipe: animating height
- Recipe: asymmetric springs
- Trap: `precision` on elements that unmount
- Springs inside Svelte transitions
- Beyond the built-in spring
- `transition:` / `in:` / `out:`
- `animate:` — reordering keyed lists
- `svelte/easing`

---

## Choosing: CSS / `Tween` / `Spring`

| Situation | Use |
|---|---|
| Element enters/leaves the DOM | `transition:` (`svelte/transition`) |
| Hover, focus, or a two-state toggle | plain CSS `transition` — no JS value needed |
| A value must reach a target in a known time | `Tween` |
| Target changes mid-flight (drag, gesture, live data) | `Spring` |
| Items reorder inside a keyed `{#each}` | `animate:flip` |

A spring has no duration. It is defined by how it reacts to a moving target, so it absorbs interruptions without restarting — that is the only reason to pick it over `Tween`. For a value that animates once from A to B, `Tween` is simpler and lands predictably.

## `Spring` — physics-driven values

_Since 5.8.0._ Wraps a value. Writing `spring.target` moves `spring.current` toward it over time.

```svelte
<script>
	import { Spring } from 'svelte/motion';

	const spring = new Spring(0, { stiffness: 0.1, damping: 0.4 });
</script>

<input type="range" bind:value={spring.target} />
<div style:translate="{spring.current}px 0">●</div>
```

**Options** (all clamped or defaulted at construction, and writable afterwards as `spring.stiffness` / `spring.damping` / `spring.precision`):

| Option | Default | Range | Effect |
|---|---|---|---|
| `stiffness` | `0.15` | 0–1 (clamped) | higher = snappier pull toward target |
| `damping` | `0.8` | 0–1 (clamped) | lower = more overshoot and wobble |
| `precision` | `0.01` | any | distance at which motion is declared settled |

Both `stiffness` and `damping` are clamped to 0–1 — they are unitless ratios, not the `170`/`26` physical constants used by Framer Motion or React Spring. Porting those numbers directly gives an instant jump, not a spring.

**Reading a source with `Spring.of`** — binds the spring to a reactive expression. Must run inside an effect root (component init, or `$effect.root`):

```svelte
<script>
	import { Spring } from 'svelte/motion';

	let { number } = $props();
	const spring = Spring.of(() => number);
</script>
```

**`spring.set(value, options)`** returns a promise that resolves when `current` catches up:

- `{ instant: true }` — `current` jumps to `target`, aborting any running animation. Use on first paint or after a reset, so nothing animates from a meaningless start value.
- `{ preserveMomentum: ms }` — keep the current trajectory for `ms` before the spring takes over. This is the 'fling' gesture case: release a drag, let it coast, then settle.

`spring.target = value` is shorthand for `spring.set(value)`.

**Springable types:** numbers, `Date`s, and arrays or plain objects whose leaves are numbers or `Date`s. The shapes of `current` and `target` must match. Anything else throws `Cannot spring <type> values` at the first tick — strings and colours included, so interpolate colour channels as an object or array of numbers.

```js
const coords = new Spring({ x: 0, y: 0 });
coords.target = { x: 100, y: 50 }; // both keys animate independently
```

Elapsed time per tick is clamped to 1/30s, so a blocked thread or a backgrounded tab resumes smoothly instead of launching the value across the screen.

## `Tween` — duration-driven values

_Since 5.8.0._ Same `current` / `target` / `set()` / `.of()` shape as `Spring`, driven by a duration and an easing function.

```svelte
<script>
	import { Tween } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';

	const progress = new Tween(0, { duration: 600, easing: cubicOut });
</script>

<progress value={progress.current}></progress>
<button onclick={() => (progress.target = 1)}>fill</button>
```

| Option | Default | Notes |
|---|---|---|
| `delay` | `0` | ms before the tween starts |
| `duration` | `400` | ms, or `(from, to) => ms` to scale with distance travelled |
| `easing` | `linear` | from `svelte/easing`, or any `(t: number) => number` |
| `interpolate` | inferred | custom interpolator for types Svelte can't handle |

Options passed to `set()` override the constructor defaults for that call. `duration: 0` sets `current` immediately and aborts the running task.

A per-call `duration` function keeps velocity constant across different distances:

```js
const x = new Tween(0, { duration: (from, to) => Math.abs(to - from) * 2 });
```

## Deprecated: `spring()` / `tweened()` stores

The `spring()` and `tweened()` store factories are deprecated. Replace them with the classes — the store's `$value` reads become `.current`, and `set`/`update` become `.set()` or a `.target` assignment.

```js
// legacy
const size = spring(0, { stiffness: 0.1 });
size.set(100);    // read as $size

// runes mode
const size = new Spring(0, { stiffness: 0.1 });
size.target = 100; // read as size.current
```

The legacy `set()` options map across as `{ hard: true }` → `{ instant: true }` and `{ soft: n }` → `{ preserveMomentum: n }`, but **the two sets are not interchangeable and fail silently**. `SpringUpdateOptions` declares all four, so `spring.set(v, { hard: true })` on a `Spring` type-checks and does nothing — the value animates when you asked it to snap. Likewise `instant`/`preserveMomentum` are inert on the legacy store.

## `prefersReducedMotion`

_Since 5.7.0._ A `MediaQuery` from `svelte/motion` matching `(prefers-reduced-motion: reduce)`. Read `.current` — it is reactive, so the value re-evaluates if the user changes the setting mid-session.

```svelte
<script>
	import { prefersReducedMotion } from 'svelte/motion';
	import { fly } from 'svelte/transition';
</script>

{#if visible}
	<p transition:fly={{ y: prefersReducedMotion.current ? 0 : 200 }}>…</p>
{/if}
```

Honour it for any motion that travels, scales, or spins. Opacity-only fades are generally safe to keep. For a `Spring`, gate the animation rather than the value: `spring.set(next, { instant: prefersReducedMotion.current })`.

## SSR and cleanup

Springs and tweens do not animate on the server: `current` renders as the initial constructor value, then animates from there once hydrated. If that first frame matters, construct with the settled value and animate on interaction instead of on mount.

Both classes cancel their own animation loop when the value settles, so a `Spring` or `Tween` created in component init needs no teardown. `Spring.of` / `Tween.of` register a render effect, which is why they must be called during init — calling them in an event handler or a plain `.svelte.ts` module leaks the effect or throws. A module-level `new Spring(0)` is fine; a module-level `Spring.of(...)` is not, unless wrapped in `$effect.root`.

## Recipe: animating height

A spring interpolates numbers, so there is no springing to `height: auto` — you must measure. Observe the content with a `ResizeObserver`, spring the measured number, and clip with `overflow: hidden` on the element whose height you animate.

```svelte
<script>
	import { Spring } from 'svelte/motion';

	let { open } = $props();

	const height = new Spring(0, { stiffness: 0.1, damping: 0.3 });
	let measured = $state(0);
	let settled = false; // deliberately not $state — writing it must not re-run the effect

	function measure(node) {
		const ro = new ResizeObserver(() => (measured = node.offsetHeight));
		ro.observe(node);
		return () => ro.disconnect();
	}

	// one of the few legitimate effects: the set needs `instant`, which `.of()` cannot pass
	$effect(() => {
		height.set(open ? measured : 0, { instant: !settled });
		if (measured > 0) settled = true;
	});
</script>

<div style="overflow: hidden; height: {height.current}px">
	<div {@attach measure}>…</div>
</div>
```

Three things this handles:

- **Toggling `open`.** The effect reads `open` and `measured`, so it re-runs on either. Putting the `set` inside the observer callback instead is a dead path — the observer fires on content resize, and toggling `open` resizes the clipping wrapper, not the content.
- **First render.** The observer fires only after layout, so `measured` is 0 on the first pass and a panel that starts open would spring up from nothing. Snap until the first real measurement lands, animate after.
- **Content that changes size while open.** The observer writes `measured` again and the effect follows — no extra wiring.

Drop the effect when you need no `instant`: `Spring.of(() => (open ? measured : 0))` covers the steady state on its own.

## Recipe: asymmetric springs

`stiffness`, `damping` and `precision` are writable at any time, so one spring can carry different physics per direction. Collapsing with the same bounce used for opening makes content flicker as it clips; damp the close.

```js
const OPEN = { stiffness: 0.1, damping: 0.3 };
const CLOSE = { stiffness: 0.1, damping: 0.5 };

Object.assign(height, open ? OPEN : CLOSE);
height.target = open ? measured : 0;
```

Apply the same split to growing vs shrinking, not just open vs closed — compare the new measurement against the previous one and pick the config before setting the target.

## Trap: `precision` on elements that unmount

`precision` is the distance at which the spring declares itself settled, default `0.01`. If something unmounts when the spring's promise resolves, that default spends a visible stretch of time creeping imperceptibly toward the target before resolving, and the element reads as stuck.

For an exiting element, raise precision far above the default — a value like `3` for a pixel translation finishes where the eye already believes it arrived.

## Springs inside Svelte transitions

Svelte transitions are duration-based: you return `{ duration, css: (t) => '…' }` and Svelte samples `css` ahead of time, compiles the result to a CSS keyframe animation, and runs it off the main thread. A spring has no duration, so it cannot be plugged in directly.

To get spring motion in a transition, pre-run the spring, collect its values, and index into them:

- duration = `frames * 1000 / 60`, from the number of sampled frames
- `css: (t) => …` maps `t` (0→1) onto that array, interpolating between neighbours when there's no exact hit

Keep the `css` form rather than `tick` — `tick` runs per frame on the main thread and gives up the keyframe compilation.

Two behaviours to design around:

- **An out transition should start from where the element currently is.** Read the live transform and opacity (`getComputedStyle`) at the start of the out function, so a fast toggle leaves from the current position instead of teleporting to the resting value first.
- **Re-mounting mid-exit reuses the original in-transition**, rather than building a fresh one from the current position. Rarely matters for a modal dismissed by an overlay click, but it rules out rapid bidirectional toggling.

Use the spring for the property that *moves* and an easing function for the rest. Opacity springing looks wrong — bouncing transparency has no physical analogue — so drive transform from the spring and fade with `quintOut` / `quadIn` in the same `css` callback.

## Beyond the built-in spring

`stiffness` and `damping` as 0–1 ratios are hard to reason about when a designer asks for "settles in 400ms with a small bounce". Third-party springs (and Apple's `SwiftUI` model) expose `duration` + `bounce` instead, deriving the physics: stiffness `k = (2π / duration)²`, damping `c = 2ζ√k` with the damping ratio ζ from `bounce` — `0` critically damped, positive underdamped (bouncy), negative overdamped (sluggish). Such implementations also tend to expose `velocity`, explicit `restDelta`/`restSpeed` thresholds, and `start`/`change`/`end` events.

Two routes when you need that:

- **Motion** (`motion`, motion.dev) — framework-agnostic, actively maintained, and its spring generator takes `duration` + `bounce` directly. The heaviest option, but the one to reach for if the project already animates with it.
- **A bespoke class** — the physics is ~100 lines of closed-form integration over the three damping regimes (under, critical, over), wrapped in `$state` fields so `current` stays reactive. Worth it only if you need `velocity` read-back or `start`/`change`/`end` hooks that no library gives you in the shape you want.

Reach for either only when you need velocity read-back, event hooks, or duration/bounce authoring. For everything else the built-in `Spring` is one import with no dependency, and `Tween` already covers "settles in 400ms".

## `transition:` / `in:` / `out:`

From `svelte/transition`: `fade`, `blur`, `fly`, `slide`, `scale`, `draw`, `crossfade`. They run when an element enters or leaves the DOM — they are not a way to animate a value that stays mounted.

```svelte
{#if visible}
	<div transition:fly={{ y: 20, duration: 200 }}>…</div>
{/if}
```

- `transition:` is bidirectional and reversible mid-flight; `in:` / `out:` are separate and do **not** reverse.
- Transitions are local by default — they play only when their own block is added or removed. Add `|global` to play when a parent block changes too.
- Prefer transitions that return a `css` function over ones that return `tick`: CSS transitions run off the main thread, JS ones do not.

## `animate:` — reordering keyed lists

`animate:flip` (from `svelte/animate`) tweens elements to their new positions when a keyed `{#each}` reorders. It requires a keyed each block and applies only to reordering — not to items being added or removed.

```svelte
{#each items as item (item.id)}
	<li animate:flip={{ duration: 200 }}>{item.name}</li>
{/each}
```

## `svelte/easing`

Standard easings: `linear`, plus `back` / `bounce` / `circ` / `cubic` / `elastic` / `expo` / `quad` / `quart` / `quint` / `sine`, each with `In`, `Out`, and `InOut` variants.

Default to `*Out` for anything responding to user input — it moves fastest at the start, which reads as immediate. Reserve `*In` for elements leaving.
