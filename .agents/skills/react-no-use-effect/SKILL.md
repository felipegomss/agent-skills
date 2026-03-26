---
name: react-no-use-effect
description: "Enforces a no-direct-useEffect policy in React codebases. Replaces useEffect patterns with derived state, event handlers, data-fetching libraries, useMountEffect, and key-based remounts. Apply when writing, reviewing, or refactoring React components."
version: 1.0.0
license: MIT
---

# No useEffect

Never call `useEffect` directly in components or hooks. Most useEffect usage compensates for something React already gives you better primitives for. This rule prevents race conditions, infinite loops, stale closures, and implicit dependency chains.

## Quick Reference

| Instead of useEffect for... | Use |
|---|---|
| Deriving state from other state/props | Inline computation (Rule 1) |
| Fetching data | `useQuery` / data-fetching library (Rule 2) |
| Responding to user actions | Event handlers (Rule 3) |
| One-time external sync on mount | `useMountEffect` (Rule 4) |
| Resetting state when a prop changes | `key` prop on parent (Rule 5) |

- Lint rule: `no-restricted-syntax` (configured to ban `useEffect`)
- React docs: [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- Origin: https://x.com/alvinsng/status/2034143381530783832

## When to Use This Skill

- Writing new React components
- Refactoring existing `useEffect` calls
- Reviewing PRs that introduce `useEffect`
- An agent adds `useEffect` "just in case"

## The Escape Hatch: useMountEffect

For legitimate external system synchronization on mount, use `useMountEffect`:

```typescript
export function useMountEffect(effect: () => void | (() => void)) {
  // eslint-disable-next-line react-hooks/exhaustive-deps
  useEffect(effect, []);
}
```

This makes intent explicit: "run once on mount." Any other use of `useEffect` is a code smell.

`useMountEffect` failures are binary and loud. Direct `useEffect` failures degrade gradually (flaky behavior, performance degradation, loops) before hard crashing.

---

## Rule 1: Derive state, do not sync it

If you're computing a value from existing state or props, do it inline. Never use an effect to set state derived from other state.

```typescript
// BAD
function ProductList() {
  const [products, setProducts] = useState([]);
  const [filteredProducts, setFilteredProducts] = useState([]);

  useEffect(() => {
    setFilteredProducts(products.filter((p) => p.inStock));
  }, [products]);
}

// GOOD
function ProductList() {
  const [products, setProducts] = useState([]);
  const filteredProducts = products.filter((p) => p.inStock);
}
```

**Smell test:** You're about to write `useEffect(() => setX(deriveFromY(y)), [y])`, or you have state that only mirrors other state or props.

---

## Rule 2: Use data-fetching libraries

Effect-based fetching creates race conditions and re-implements caching logic. Use React Query, SWR, or the framework's data layer instead.

```typescript
// BAD
function ProductPage({ productId }) {
  const [product, setProduct] = useState(null);

  useEffect(() => {
    fetchProduct(productId).then(setProduct);
  }, [productId]);
}

// GOOD
function ProductPage({ productId }) {
  const { data: product } = useQuery(['product', productId], () =>
    fetchProduct(productId)
  );
}
```

**Smell test:** Your effect does `fetch(...)` then `setState(...)`, or you're re-implementing caching, retries, cancellation, or stale handling.

---

## Rule 3: Event handlers, not effects

If something happens because the user did something, put the logic in the event handler, not in an effect watching a state flag.

```typescript
// BAD
function LikeButton() {
  const [liked, setLiked] = useState(false);

  useEffect(() => {
    if (liked) {
      postLike();
      setLiked(false);
    }
  }, [liked]);

  return <button onClick={() => setLiked(true)}>Like</button>;
}

// GOOD
function LikeButton() {
  return <button onClick={() => postLike()}>Like</button>;
}
```

**Smell test:** State is a flag so an effect can run, or you're building "set flag -> effect runs -> reset flag" mechanics.

---

## Rule 4: useMountEffect for one-time external sync

Use `useMountEffect` for DOM integration, third-party widget lifecycles, and browser API subscriptions. Prefer conditional mounting over guards inside effects.

```typescript
// BAD
function VideoPlayer({ isLoading }) {
  useEffect(() => {
    if (!isLoading) playVideo();
  }, [isLoading]);
}

// GOOD
function VideoPlayerWrapper({ isLoading }) {
  if (isLoading) return <LoadingScreen />;
  return <VideoPlayer />;
}

function VideoPlayer() {
  useMountEffect(() => playVideo());
}
```

For stable dependencies:

```typescript
// BAD
useEffect(() => {
  connectionManager.on('connected', handleConnect);
  return () => connectionManager.off('connected', handleConnect);
}, [connectionManager]);

// GOOD
useMountEffect(() => {
  connectionManager.on('connected', handleConnect);
  return () => connectionManager.off('connected', handleConnect);
});
```

**Smell test:** You're synchronizing with an external system, and behavior is naturally "setup on mount, cleanup on unmount."

---

## Rule 5: Reset with key, not dependency choreography

When a component must restart cleanly for a new entity/ID, use React's `key` prop to force a remount instead of managing reset logic inside effects.

```typescript
// BAD
function VideoPlayer({ videoId }) {
  useEffect(() => {
    loadVideo(videoId);
  }, [videoId]);
}

// GOOD
function VideoPlayer({ videoId }) {
  useMountEffect(() => loadVideo(videoId));
}

function VideoPlayerWrapper({ videoId }) {
  return <VideoPlayer key={videoId} videoId={videoId} />;
}
```

**Smell test:** Your effect only resets local state when an ID/prop changes, or you want fresh component instances per entity.

---

## Component Structure Convention

```typescript
export function FeatureComponent({ featureId }: ComponentProps) {
  // Hooks first
  const { data, isLoading } = useQueryFeature(featureId);

  // Local state
  const [isOpen, setIsOpen] = useState(false);

  // Computed values (NOT useEffect + setState)
  const displayName = user?.name ?? 'Unknown';

  // Event handlers
  const handleClick = () => { setIsOpen(true); };

  // Early returns
  if (isLoading) return <Loading />;

  // Render
  return <Flex direction="column" gap="lg">...</Flex>;
}
```

---

## Failure Mode Summary

| Pattern | Common Bug |
|---|---|
| Derived state via effect | Extra render cycles, stale values |
| Fetch in effect | Race conditions, no cancellation |
| State flag + effect action | Missed runs, double-fire, infinite loops |
| Dependency array choreography | Silent regressions after refactors |
| Reset logic in effect | Stale state, reset not firing |

---

## Workflow

### 1. Identify the useEffect
Determine what the effect is doing: deriving state, fetching data, responding to an event, syncing with an external system, or resetting state.

### 2. Apply the correct replacement pattern
Use the five rules above to pick the right replacement.

### 3. Verify

```
npm run lint -- --filter=<package>
npm run typecheck -- --filter=<package>
npm run test -- --filter=<package>
```

## Enforcement

- Add an ESLint rule to ban direct `useEffect` imports in components (`no-restricted-syntax`).
- The `useMountEffect` wrapper lives in `hooks/use-mount-effect.ts` (or equivalent shared location).
- Code review: any `useEffect` call without the `useMountEffect` wrapper is an automatic request-changes.
