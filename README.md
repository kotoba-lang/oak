# oak

**OaK — the ontology as the kernel.** An agent does not act on the graph. It
hands `oak-call` a typed value naming what it wants, and a schema decides
whether that is a thing the world admits. Written in Kotoba, compiled by amu,
with the graph behind the `dataspace/transact` capability.

The name is a metaphor, so: this is not a library of ontology helpers, and it
is not [OAK, the Ontology Access Kit](https://github.com/INCATools/ontology-access-kit).
It is a **syscall table whose admission rule is an ontology**.

```
       agent                     kernel                        world
  ┌──────────────┐        ┌────────────────────┐        ┌──────────────┐
  │ a value of   │ ─────► │  S   entity types  │ ─────► │  dataspace   │
  │ [:variant    │        │      relations     │        │  assertions  │
  │  :oak/call]  │        │      domain/range  │        │              │
  │              │        │  ─────────────────  │        │              │
  │ never holds  │ ◄───── │  F   declare       │ ◄───── │  bindings    │
  │ the          │        │      relate        │        │              │
  │ capability   │        │      related       │        └──────────────┘
  └──────────────┘        │      census        │
                          └────────────────────┘
                            :ok value+evidence
                            :rejected code+detail
```

## Why this shape, on this stack

The OaK framing — the kernel is `K = (S, F)`, a schema plus the executable
reasoning functions permitted over it — lands on Kotoba almost without
translation, because Kotoba already refuses ambient authority. Three of the
paper's properties are not things this implementation adds; they are things
it cannot avoid:

| OaK wants | Kotoba already gives |
|---|---|
| the agent computes only through `F` | `F` is not exported. `oak-call` is, and it is the only door |
| the agent cannot reach the store directly | the graph is behind `:dataspace/transact`, and a guest that does not hold it cannot name a pattern |
| a refusal is a result, not a crash | no ambient `throw`; every refusal is `[:rejected code detail]` |

The `:capabilities` clause is checked both ways: a declared capability that is
never used is a compile error, and a policy that does not grant id 24 refuses
admission before a byte is emitted.

## S

```clojure
(defn schema-doc [] :document
  [[:entity :agency]
   [:entity :tender]
   [:entity :supplier]
   [:relation :issued-by  :tender :agency]
   [:relation :awarded-to :tender :supplier]])
```

S is an ordinary document, so it has an identity: `document-sha256` of it, and
**every result carries that digest**. A result is therefore attributable to the
schema that admitted it, not merely to the kernel that returned it.

Editing `schema-doc` is how you retarget the kernel at another task. Nothing
below it names `:tender`.

## F

| call | admits when | returns |
|---|---|---|
| `:declare  {entity type}` | `type` is an `:entity` of S | count written |
| `:relate   {subject relation object}` | `relation` is in S, and the subject's and object's **types in the graph** match its domain and range | count written |
| `:related  {subject relation}` | `relation` is in S | the objects, as a document vector |
| `:census   {type}` | `type` is an `:entity` of S | how many entities carry it |

Refusals are named, not boolean: `:oak/unknown-entity-type`,
`:oak/unknown-relation`, `:oak/domain-violation`, `:oak/range-violation`,
`:oak/untyped-subject`, `:oak/untyped-object`, `:oak/graph-at-capacity`.

Note where the type of an entity lives. It is not an attribute of the call and
not a field of the schema — it is **an assertion in the graph**, put there by
an earlier admitted `:declare`. So checking a domain constraint is itself a
graph query, and an entity nothing has typed cannot be related to anything.

## Two ceilings, and only one of them is the kernel's

**The graph holds 32 typed entities and 32 facts**, because a query result is
one document and a document container holds `document-container-item-limit`
items. The kernel enforces this **on the write side**: it refuses a 33rd fact
with `:oak/graph-at-capacity` rather than accepting one it could never report.
It has to be the write side — the provider does not answer "too many", it traps
while encoding its reply, and a Kotoba guest has no `catch`.

**Reading is bounded again by the host's fuel, and that ceiling is not ours.**
Measured 2026-08-31 on the reference interpreter at its default fuel: a census
answers at 24 declared entities and traps `:fuel-exhausted` at 28 — so on that
host the fuel runs out before the document does. A browser wasm host budgets
fuel differently. The conformance suite therefore does not name a size: it
probes for the running host's own ceiling and asserts the invariant on both
sides of it — below, a read answers; above, what comes out is a Kotoba trap
naming `:fuel-exhausted`, never a wrong answer and never a host error object.

Do not copy either number into code. Measure the host you are on.

## Running it

```sh
nbb bin/conformance.cljs        # from the repository root
```

No JVM: `kotoba.sema`, `kotoba.kir`, `provider.dataspace` and the reference
runtime are all `.cljc`, so analyze → lower → instantiate → invoke is Node
throughout. Missing sibling checkouts make the runner **exit 2** and say which
ones — a suite that could not run must not answer with what a suite that ran
and found nothing answers with.

To compile the kernel rather than interpret it:

```sh
amu check   src/oak/kernel.kotoba --jvm-free --policy src/oak/policy.edn
amu compile src/oak/kernel.kotoba --jvm-free --target wasm32-browser \
            --policy src/oak/policy.edn --output oak.wasm
```

`check` passes as written. **`compile` does not yet, and the reason is a
stale pin rather than a missing capability.**

That path did not work at all when this repository was written: `check`
passed and `compile` reported an internal compiler error, at two separate
places where an i64 reached a host operation that cannot take a BigInt —
`uleb` in kotoba-kir, and the capability import key in kotoba-wasm. Both are
fixed and merged (`kotoba-kir` `f0a56e9`, `kotoba-wasm` `34557de`), and both
needed their repository to grow a ClojureScript test runner first, because
every test in them was `.clj` and the `:cljs` branches had never been run.

With those two sources on the classpath the kernel emits an **8,394-byte
`wasm32-browser-kotoba-v1` module with no JVM anywhere** (measured
2026-08-31). But `bin/amu` resolves its own dependency closure from amu's
`deps-lock.edn`, which still pins `kotoba-kir` at `099c627` and
`kotoba-wasm` at `87a5c0b` — both from before the fixes — so the CLI as
invoked above still fails. Until amu's pins advance, emit through the
compiler directly:

```sh
nbb --classpath "<amu>/src:<amu>/resources:<kotoba-kir>/src:<kotoba-sema>/src:..." \
    <amu>/src/kotoba/compiler/nbb/wasm_cli.cljs compile src/oak/kernel.kotoba \
    --target wasm32-browser --policy src/oak/policy.edn --output oak.wasm
```

Advancing amu's pins was left alone on purpose: it would pull in every other
change to those two repositories since, which is a decision about amu and not
about this kernel.

`dataspace-v1` is qualified on `:reference`, `:wasm-aot`, `:native-aot` and
`:jit` — the only kit qualified across all four — which is why the graph plane
is this kit and not something built here. Check
`amu/resources/kotoba/lang/capability-kits/dataspace-v1.edn` for the current
values rather than trusting this sentence.

## What the suite actually discriminates

Green is cheap. These were measured by breaking the kernel one edit at a time
and confirming that what went red was what the check is named after:

| break | what failed |
|---|---|
| domain check made vacuous | *only* "a subject of the wrong type is refused as a domain violation" — and it showed the call falling through to the range check |
| capacity guard removed | the two capacity checks, reporting the provider's own `document-edn-read vector item limit exceeded` |
| evidence digest replaced with a constant | "every result carries the digest of the schema that admitted it" |
| query results discarded | six checks, starting with the ones that depend on a type being readable |

## Boundaries with the nearest repositories

- **`kotoba-lang/ontology`** is a registry of *what kind of thing an ingested
  fact is* — object types and the connector that produced them. It answers a
  vocabulary question about data at rest. This repository answers an authority
  question about a call in flight. Neither is a layer of the other.
- **`kotoba-lang/kgraph`** and **`kotobase`** are the datom plane. This kernel
  does not touch them; the `dataspace-v1` provider may be backed by kgraph
  without the guest knowing.
- **`kotoba-lang/agent`** is one bounded execution. It is a caller here, not a
  component.

## Provenance

The framing is taken from *Toward Effective and Reliable LLM Agents via
Dynamic Ontology* (Zhang, Sun, Yang, Cui, Guo, Hu; arXiv:2608.22974), as it
was described to this repository's author — **the paper itself was not read**,
so treat the correspondence as a reading of the idea and not as a claim of
fidelity to its algorithms. What is here that the paper does not describe is
the capability boundary: OaK argues that the ontology should be the kernel,
and Kotoba is a language in which "kernel" can mean the thing that holds the
authority rather than a naming convention.

The paper also builds `S` and `F` from training data and refines them against
a judge. That loop is **not implemented here**. `S` is hand-written and fixed.
