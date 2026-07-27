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

### `collections@SortedMap` — ordered key-value map

An immutable map that keeps its keys in comparator order, filling the `gb_trees` role that core
`Dictionary` (a deliberately unordered BEAM map) cannot. Backed by a persistent weight-balanced tree,
so `at:`, `at:put:` and `removeKey:` are O(log n) and every operation returns a new map.

```beamtalk
m := collections@SortedMap new
m := m at: 3 put: "c"
m := m at: 1 put: "a"
m keys                            // => #(1, 3)
m min                             // => 1
m floor: 2                        // => 1
(m from: 1 to: 2) keys            // => #(1)
```

| Group | Messages |
|---|---|
| Construction | `new`, `sortedBy:`, `withAll:`, `sortedBy:withAll:` |
| Lookup | `at:`, `at:ifAbsent:`, `includesKey:`, `size`, `isEmpty` |
| Update | `at:put:`, `removeKey:`, `merge:` |
| Ordered traversal | `do:`, `keysAndValuesDo:`, `keys`, `values`, `collect:` |
| Ends | `min`, `max`, `first`, `last` |
| Nearest neighbours | `floor:`, `floor:ifAbsent:`, `ceiling:`, `ceiling:ifAbsent:` |
| Range scans | `from:to:`, `from:to:do:` |
| Conversions | `asDictionary`, `asList`, `asSortedMap` |

`min` and `max` answer *keys* — the ordered dimension of a map — while `first` and `last` answer the
values sitting at those keys. `do:` iterates values, matching `Dictionary>>do:`. `withAll:` accepts a
`Dictionary`, another `SortedMap`, or any collection of two-element key-value pairs.

### `collections@SortedSet` — ordered set

The `gb_sets` counterpart, sharing `SortedMap`'s tree: a set is that map with nothing hanging off the
keys.

```beamtalk
s := collections@SortedSet withAll: #(5, 1, 3, 1)
s asList                          // => #(1, 3, 5)
s ceiling: 2                      // => 3
(s from: 2 to: 5) asList          // => #(3, 5)
s union: (collections@SortedSet withAll: #(2))
```

| Group | Messages |
|---|---|
| Construction | `new`, `sortedBy:`, `withAll:`, `sortedBy:withAll:` |
| Membership | `add:`, `remove:`, `includes:`, `size`, `isEmpty` |
| Ordered traversal | `do:`, `asList` |
| Ends | `min`, `max`, `first`, `last` |
| Nearest neighbours | `floor:`, `floor:ifAbsent:`, `ceiling:`, `ceiling:ifAbsent:` |
| Range scans | `from:to:`, `from:to:do:` |
| Set algebra | `union:`, `intersection:`, `difference:` |
| Conversions | `asSet`, `asList`, `asSortedSet` |

### Ordering and comparator identity

Both classes order themselves with a two-argument block returning a Boolean — the same convention as
core `List sort:` — read as "a sorts at or before b". The default is ascending natural order,
`[:a :b | a <= b]`; pass your own to `sortedBy:`.

```beamtalk
byLength := [:a :b | a size <= b size]
collections@SortedSet sortedBy: byLength withAll: #("ccc", "a", "bb")   // => "a", "bb", "ccc"
```

That comparator also decides *identity*: two keys or elements that sort at-or-before each other in both
directions are the same key. A set ordered by `byLength` therefore holds at most one string of each
length.

**The comparator is part of the structure's identity.** Combining two collections that disagree about
ordering could only produce a mis-ordered result, so `SortedMap>>merge:` and `SortedSet>>union:`,
`intersection:` and `difference:` raise an error instead of guessing. Sameness is decided by *block
identity*: everything built with `new` or `withAll:` shares the default ordering and combines freely,
while anything built with `sortedBy:` combines only with structures built from that same block value.
Bind the block to a variable and reuse it when several interoperable collections are needed.

Range scans are inclusive of both bounds, and neither bound has to be present in the collection.
`min`, `max`, `first`, `last` and the `ifAbsent:`-less `at:`, `floor:` and `ceiling:` raise a
structured error rather than answering `nil` when there is no such element.

The balanced tree behind both classes (`SortedTreeNode`) is an implementation detail of this package,
not public API.

## Usage

Everything this library exports is reached through its qualified package name, `collections`, using the
`package@Class` syntax (ADR 0070 §4):

```beamtalk
Object subclass: MyApp
  run =>
    scores := collections@SortedMap new.
    scores := scores at: "carol" put: 3.
    scores := scores at: "alice" put: 1.
    scores keys
```

Qualifying with `collections@` is required whenever the plain class name would collide with another
dependency's export, and is always accepted even when it wouldn't be ambiguous — see
[Qualified Names](https://github.com/jamesc/beamtalk/blob/main/docs/beamtalk-packages.md#qualified-names-packageclass)
in the main repo's docs for the full rules.

## Adding this as a dependency

Once a version has been published to the registry, add it to a project's `beamtalk.toml` with:

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
