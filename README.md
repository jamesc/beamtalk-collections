# collections

Specialized, purely-functional data structures for [Beamtalk](https://github.com/jamesc/beamtalk) that
don't belong in core stdlib.

Core stdlib is always-present, always-compiled, and owns the unqualified namespace, so it stays limited
to structures every program needs. `collections` is the opt-in library for the rest: structures that want
prime names that would otherwise collide with user code (`Heap`, `Deque`, `SortedMap`, `Zipper`), and that
are built demand-driven — one structure per issue, only once a real use case appears, not speculatively.
See the tracking epic ([BT-2697](https://linear.app/beamtalk/issue/BT-2697)) for the full rationale and the
list of structures planned or already shipped.

This repo is a standalone first-party package, following the same extraction shape as
[`beamtalk-http`](https://github.com/jamesc/beamtalk-http) per ADR 0073 (package distribution and
discovery). It dogfoods the package system introduced by ADR 0070 (namespaces) and ADR 0073 (registry).

## Structures

### `PriorityQueue` — immutable pairing heap ([BT-3013](https://linear.app/beamtalk/issue/BT-3013))

A persistent priority queue. `add:` and `merge:` are O(1) (they link two trees and stop), `removeMin`
is O(log n) amortised, and `size` is O(1) because the count is carried in the queue's state. Every
operation returns a new queue — the receiver is never modified.

```beamtalk
q := collections@PriorityQueue withAll: #(5, 1, 3).
q peek                    // => 1
q size                    // => 3
q asSortedList            // => #(1, 3, 5)

rest := q removeMin.      // just the remaining queue — no pair to unpack
rest peek                 // => 3
rest size                 // => 2
```

`peek` and `removeMin` are deliberately separate: `peek` answers the minimum element, `removeMin` answers
the queue without it. That follows the `List first` / `List rest` idiom used throughout the stdlib rather
than returning an element-plus-remainder pair. Everything here is immutable, so there is no atomicity to
preserve by fusing them, and neither half gets more expensive for being split — `peek` is O(1) and
`removeMin` is O(log n) amortised either way.

**Ordering.** A two-argument comparator block returning a Boolean, the same convention as `List>>sort:`
— it answers "should `a` come out before `b`?". The default is `[:a :b | a <= b]`, a min-heap.

```beamtalk
maxHeap := collections@PriorityQueue sortedBy: [:a :b | a >= b].
(maxHeap addAll: #(5, 1, 9)) peek     // => 9
```

**`merge:` and comparator identity.** Merging two queues that order elements differently would produce a
heap whose `peek` is not the minimum, so `merge:` refuses it. Each queue carries an *ordering token*
(readable via `ordering`) that `merge:` compares with `=:=`: `new` uses `#natural`, `sortedBy:` uses the
comparator block itself, and `sortedBy:labelled:` uses a label you supply. Blocks compare by identity, so
two separately written literals never match even when they read the same — use `sortedBy:labelled:` to
declare independently built comparators equivalent.

`#natural` is reserved for the default ordering and is rejected as an explicit label. Allowing it would
collide with the token `new` and `withAll:` assign, so `merge:` would accept a foreign comparator and
splice its subtree in whole — `peek` would still answer correctly while a full drain came out unsorted.

```beamtalk
a := collections@PriorityQueue sortedBy: [:x :y | x >= y] labelled: #descending.
b := collections@PriorityQueue sortedBy: [:x :y | x >= y] labelled: #descending.
((a add: 1) merge: (b add: 9)) peek   // => 9

// different tokens — raises
(collections@PriorityQueue new) merge: (collections@PriorityQueue sortedBy: [:x :y | x >= y])
```

**Iteration is unordered.** `do:` — and everything built on it (`collect:`, `select:`, `asList`,
`includes:`, …) — walks the heap, not a sorted sequence. Only the minimum is guaranteed to come first.
Use `asSortedList` (O(n log n), a repeated `removeMin` drain) when you want ordered output.

**Rebuilding operations keep your ordering.** `collect:`, `select:`, and `reject:` all answer a queue
that carries the receiver's comparator *and* its ordering token, so a max-heap stays a max-heap and the
result can still be merged with the queue it came from. These are overridden here rather than inherited:
`Collection` rebuilds via `self species withAll:`, and `species` answers a *class*, which cannot carry
per-queue state — the inherited versions would silently hand back a `#natural` min-heap.

```beamtalk
maxHeap := (collections@PriorityQueue sortedBy: [:a :b | a >= b]) addAll: #(1, 4, 2, 5).
(maxHeap select: [:x | x > 2]) asSortedList     // => #(5, 4)
(maxHeap reject: [:x | x > 2]) asSortedList     // => #(2, 1)
(maxHeap collect: [:x | x * 10]) peek           // => 50
```

Note that `collect:` maps `E -> R`, so a comparator written against `E` may not be meaningful for `R`.
It is preserved anyway: you explicitly chose that ordering, and silently reverting to natural order is a
worse failure mode than a comparator that raises. Mapping to a type your comparator cannot handle is
caller error — go through `asSortedList` or `asList` first if you want out of the ordering.

**Empty queues.** `peek` and `removeMin` raise on an empty queue; `peekIfEmpty:` and `removeMinIfEmpty:`
take a block and return its value instead. These raise rather than answering a `Result` on purpose:
per [ADR 0060](https://github.com/jamesc/beamtalk/blob/main/docs/ADR/0060-result-type-hybrid-error-handling.md),
`Result` is for environmental failures (I/O, parsing, network) while exceptions signal caller misuse, and
asking an empty heap for its minimum is caller misuse.

**No decrease-key.** There is deliberately no `decreaseKey:`, and no efficient one is possible in a
persistent structure: there are no stable node handles to decrease, and finding an element costs O(n).
For Dijkstra or A*, use the standard workaround — push a *duplicate* entry at the lower priority and
discard stale entries as you pop them:

```beamtalk
entry := queue peek.
queue := queue removeMin.
(entry at: 1) =:= (best at: (entry at: 2))
  ifTrue: [ "…entry is live, relax its neighbours…" ]
  ifFalse: [ "…stale duplicate, skip it…" ]
```

Duplicate elements are fully supported: a `PriorityQueue` is a heap, not a set.

## Usage

Everything this library exports is reached through its qualified package name, `collections`, using the
`package@Class` syntax (ADR 0070 §4). For example, once `collections@Deque` exists:

```beamtalk
Object subclass: MyApp
  run =>
    queue := collections@Deque new.
    queue := queue pushFront: 1.
    queue := queue pushBack: 2.
    queue first
```

Qualifying with `collections@` is required whenever the plain class name would collide with another
dependency's export, and is always accepted even when it wouldn't be ambiguous — see
[Qualified Names](https://github.com/jamesc/beamtalk/blob/main/docs/beamtalk-packages.md#qualified-names-packageclass)
in the main repo's docs for the full rules.

## Adding this as a dependency

Once a version has been published to the registry (starting with `collections@Deque` in BT-3011), add it
to a project's `beamtalk.toml` with:

```bash
beamtalk deps add collections
```

or pin an exact version:

```bash
beamtalk deps add collections --version 0.1.0
```

Before a version is published, or to track an unreleased commit, add it as a git dependency instead:

```bash
beamtalk deps add collections --git https://github.com/jamesc/beamtalk-collections --branch main
```

`deps add` writes the resolved entry to `beamtalk.toml` and pins it in `beamtalk.lock`. See
[Dependency Management](https://github.com/jamesc/beamtalk/blob/main/docs/beamtalk-packages.md#dependency-management-cli)
in the main repo's docs for the full `deps` CLI reference, including git tags/revs and registry details.

## Development

```bash
just build   # compile
just test    # run the test suite
just fmt     # check formatting
just lint    # run style/redundancy checks
just ci      # fmt + lint + build + test
```

## License

Apache-2.0. See [LICENSE](LICENSE).
