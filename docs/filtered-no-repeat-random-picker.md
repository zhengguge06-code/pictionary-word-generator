# Build a Filtered, No-Repeat Random Picker in TypeScript

A standalone companion example: pick a random item from a filtered pool without repeating within a session, and handle pool exhaustion explicitly.

## The three invariants

Before writing any implementation, define what "correct" means as testable contracts:

1. **Never leave the active filter.** If the user selects `hard` + `animals`, every pick must match both constraints. When the matching pool is empty, return an explicit `no-match` result — never silently fall back to a looser filter.

2. **Never repeat a used ID in the session.** Track history by stable identifier, not display text. Two items that display the same word but carry different IDs are different picks. Two items that normalize to the same string but have different IDs still count as separate entries in the pool.

3. **Make empty and exhausted different states.** `empty` means no item in the dataset matches the current filter at all — a data problem. `exhausted` means every matching item has already been shown this session — a usage milestone. The caller needs to respond differently to each.

## Return a result the caller cannot misread

A picker that returns only an item or `undefined` loses useful information. Instead, make failure states part of the return type. This example stays generic: it knows nothing about Pictionary, difficulty names, React, or browser storage.

```typescript
type PickResult<T> =
  | { kind: 'picked'; item: T; remaining: number }
  | { kind: 'no-match' }
  | { kind: 'exhausted'; matching: number };

type PickRequest<T> = {
  catalog: readonly T[];
  accepts: (item: T) => boolean;
  keyOf: (item: T) => string;
  seen: ReadonlySet<string>;
  random?: () => number;
};
```

The caller supplies the filter as a predicate and the identity rule as `keyOf`. That keeps the selection utility reusable for flashcards, quiz questions, recommendations, or game prompts.

## Make one function own the selection boundary

```typescript
export function selectNext<T>({
  catalog,
  accepts,
  keyOf,
  seen,
  random = Math.random,
}: PickRequest<T>): PickResult<T> {
  const matching = catalog.filter(accepts);
  if (matching.length === 0) return { kind: 'no-match' };

  const unseen = matching.filter((item) => !seen.has(keyOf(item)));
  if (unseen.length === 0) {
    return { kind: 'exhausted', matching: matching.length };
  }

  const sample = random();
  const unit = Number.isFinite(sample)
    ? Math.min(Math.max(sample, 0), 1 - Number.EPSILON)
    : 0;
  const item = unseen[Math.floor(unit * unseen.length)]!;

  return {
    kind: 'picked',
    item,
    remaining: unseen.length - 1,
  };
}
```

The function first proves that the filter has data, then removes IDs already seen. It never broadens `accepts`, so an empty or exhausted request cannot leak an item from another level or topic. The small clamp keeps a faulty injected random function from producing an out-of-range array index; normal `Math.random()` values already fall between 0 inclusive and 1 exclusive.

## Adapt it to your own domain

```typescript
type Card = {
  id: string;
  label: string;
  level: 'starter' | 'advanced';
  topics: string[];
};

const cards: Card[] = [
  { id: 'c-1', label: 'Cat', level: 'starter', topics: ['animals'] },
  { id: 'c-2', label: 'Pizza', level: 'starter', topics: ['food'] },
  { id: 'c-3', label: 'Gravity', level: 'advanced', topics: ['concepts'] },
];

const seen = new Set<string>();
const level: Card['level'] = 'starter';
const topic: string = 'animals';

const result = selectNext({
  catalog: cards,
  accepts: (card) =>
    card.level === level && (topic === 'all' || card.topics.includes(topic)),
  keyOf: (card) => card.id,
  seen,
});

if (result.kind === 'picked') {
  seen.add(result.item.id);
  console.log(result.item.label, result.remaining);
}
```

History belongs to the caller. A browser UI might keep the `Set` for one tab, while a command-line quiz could keep it for one process. Changing filters does not have to erase history: the next call simply applies a different predicate against the same `seen` set. Resetting is an explicit `seen.clear()` operation.

## Define the lifetime of “no repeat”

“No repeat” is incomplete until you define its scope. A picker could avoid repeats within one request, one filter, one signed-in account, one browser tab, or forever. Those are different product promises and require different storage choices.

For a short game or study session, a session-scoped `Set` is usually enough. Keep it outside `selectNext`, pass it into every call, and clear it only after an explicit reset. If the user switches from one topic to another and later switches back, the original topic's seen IDs are still excluded because history is independent of the current predicate.

Longer persistence changes the problem. Saving IDs to local storage or a database introduces schema migration, expired entries, multiple-device conflicts, and IDs that no longer exist after a catalog update. Do not add that machinery unless the product actually promises cross-session history.

Stable identity matters here. Use an immutable ID rather than visible text or array position. Labels can be edited, localized, or duplicated intentionally; positions change whenever the catalog is sorted. The key function makes that decision explicit and keeps the generic selector out of domain-specific identity rules.

Also decide what exhaustion means for the interface. Resetting automatically is convenient but hides the fact that every matching entry was used. Returning an explicit `exhausted` result lets the caller ask the user whether to reset, change the filter, or end the activity.

## Test the invariants

| Test case | Setup | Expected result |
|---|---|---|
| Filter mismatch | Predicate matches no catalog entries | Returns `{ kind: 'no-match' }` |
| Used ID exclusion | Mark the first of two matching IDs as seen | Returns the second ID, never the first |
| Full exhaustion | Mark every matching ID as seen | Returns `{ kind: 'exhausted', matching: n }` |
| Broad topic | Predicate accepts all topics at one level | Candidate set includes every topic at that level |
| Deterministic RNG | Inject `() => 0` and `() => 0.999` | Returns the first and last unseen entries |
| No fallback | Predicate matches zero entries while unrelated entries exist | Still returns `no-match`; never broadens the predicate |
| Invalid RNG | Inject `() => Number.NaN` | Falls back to the first unseen entry without indexing outside the array |

For executable tests, assert the discriminated `kind` before reading `item`. TypeScript will then prevent a test — or a UI — from treating `no-match` and `exhausted` as if they contained a valid selection.

## Where this pattern is useful

This isn't specific to word games. The same three invariants apply anywhere you pick from a constrained set without repetition:

- Flashcard apps that cycle through a filtered deck
- Quiz generators that exhaust a topic before recycling
- Recommendation rotations that respect user-selected categories
- Any randomized UI that needs to distinguish "nothing configured" from "everything shown"

The key insight is that the eligible pool must be the intersection of “accepted by the current predicate” and “not yet seen.” A discriminated result forces the caller to handle unavailable states explicitly rather than silently degrading the filter.

## Live case and boundaries

I used this pattern in the round logic of a browser-based Pictionary tool I built. You can [see the no-repeat picker in a live Pictionary round tool](https://pictionarywordgenerator.io/) — select a difficulty, pick a category, and generate words without session repeats.

The production implementation lives in a private repository. The generic, caller-configured selector above was designed separately for this guide: it uses a different API and result model while demonstrating the same public behavioral invariants. It does not include the production word library, application state hook, UI components, or template code. The project uses TanStarter under a license that prohibits redistribution of its templates and components, so the production repository is not public.
