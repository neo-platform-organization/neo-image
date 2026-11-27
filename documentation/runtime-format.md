# Current image format

The [v1 contract](language-v1.md) separates the tested baseline, resource limits,
and requirements still needed before language completion. Quoted text now
rejects malformed UTF-8; text-length/text-byte retain byte-based semantics.

This guide describes the executable foundation snapshot. The whitepaper
contains broader design proposals. Retired implicit-primitive syntax and
metadata wrappers are not accepted by this VM.

## Warp

The current native language interface is ABI 4. Value tokens now hold up to
1023 serialized bytes independently of bounded names. Matching native components
must be rebuilt for the larger result buffer/staging layout. Graph-facing managed stores use
explicit package grants; see the
[milestone checkpoint](../../neo-vm/documentation/development/milestone-checkpoint.md)
for store operations, checkpoint/resume, and image-library commands. These are
implemented additions to the foundation subset, not a completed version-1 gate.

Warp is a serialized graph with explicit objects and named members. A document
starts with a version and an entry receiver/selector:

```lisp
(warp 1
  (entry workspace start)
  (object workspace
    (handler start (send @counter increment 21)))
  (object counter
    (slot value 0)
    (knight double value (* value 2))
    (handler increment amount
      (set self value (call self double amount)))
    (handler reading (get self value))))
```

The first atom names each form's role or operation. Its remaining children are
ordered operands. `@counter` is an object reference; `self` is the current
receiver. Quoted text is data, never a reference. Names have bounded lengths.
Object references and mutable state remain subject to package checks.

Objects may declare slots, links, a delegate, imports, handlers, and Knight
functions. A delegate supplies behavior while sends retain the actual receiver.
Slots belong to that receiver. A `package` declaration scopes resource grants;
imports require explicit host authorization. Syntax alone cannot grant access.

## Handlers and Knight

Text now supports text-concat, scalar-indexed text-slice, text-scalar-length and
text-scalar-at. Text-returning Knight functions compose through call-text.
See the [text contract](language-v1.md#constructing-and-inspecting-text) for
evaluation order, capacity errors and Unicode scalar versus byte semantics.

Bytes support `bytes(...)`, `bytes-concat` and byte-indexed `bytes-slice`.
Integer vectors support `vector(...)`, `vector-concat` and element-indexed
`vector-slice`. In Warp, constructors use ordinary forms such as `(bytes 0 255)`
and `(vector 1 2 3)`. Pure byte results compose through `call-bytes`; pure vector
results use `call-vector`. See the [value contract](language-v1.md#execution-and-pure-calls)
for bounds, failure, serialization and ownership rules.

`send` invokes object behavior. `call` invokes pure Knight calculations. Both
use positional arguments, currently at most four. Knight cannot write slots,
access documents, publish pixels, or schedule messages.

| Form | Meaning |
| --- | --- |
| `(get self value)` | Read a slot |
| `(set self value expression)` | Stage a slot write |
| `(do first second)` | Evaluate ordered expressions |
| `(let name value body)` | Bind a local value for the body |
| `(if test yes no)` | Choose a branch |
| `(return value)` | Return from a handler |
| `(+ a b)`, `(- a b)`, `(* a b)` | Checked signed integer arithmetic |
| `(= a b)`, `(< a b)`, `(> a b)` | Comparisons |
| `(post self selector arguments...)` | Stage a later message |
| `(after 20 self selector arguments...)` | Stage a delayed message |

Integer overflow rejects the send. Validated graph writes publish together.
External effects have narrower contracts: rollback cannot undo bytes already
sent, or a storage write whose result is uncertain.

Hosted builds also support bounded text, bytes, integer vectors, references,
binary64 values, and binary64 vectors. Their exact source tokens and operations
are exercised in `../depth/tools/regression-data.json`. Rich values have small fixed
limits; this is not an unbounded general-purpose collection implementation.
Standalone x86_64 currently uses the integer profile and rejects binary64.

## Weft

Weft is a readable source view that lowers to Warp. It does not replace the
canonical object graph or introduce a second runtime:

```text
object counter {
    slot value = 0;
    knight double(value) = value * 2;
    handler increment(amount) = set self.value = call self.double(amount);
    handler reading() = get self.value;
}
```

The complete runnable source also needs its entry and receiver declarations.
To inspect the current image's generated Weft, use:

```text
neo source ../neo-image/neo-os/image.neo
```

Inspection and bounded alternate source views do not imply arbitrary runtime
reinterpretation. Unsupported interpretations return an explicit failure.

## Image services

The canonical image contains `surface-exercise` and `lifecycle-exercise`.
They are graph objects, not C implementations of applications.

`buffer-fill` and `buffer-present` stage commands for an explicitly granted
surface. The host forwards normalized input to graph handlers such as `button`,
`key`, `text`, and `configure`. The image chooses what to draw and how to react.

`read-document` and `write-document` access one explicitly granted scalar.
The lifecycle harness attaches a durable commit callback. A failed commit
rejects graph publication. A durable write must use a separate send from
framebuffer, scheduling, or object lifecycle effects. General graph-facing
asynchronous storage and networking remain unfinished.

See [the lifecycle exercise](../../neo-vm/documentation/development/image-exercise.md)
for executable commands, controls, and failure tests.

## Live sessions and snapshots

```text
neo session ../neo-image/neo-os/image.neo
```

Inside the session:

```text
revision
send instrument sample 21
send instrument reading
snapshot
quit
```

A snapshot serializes graph definitions and values. It does not serialize host
grants, native pages, pending events, or active call frames. Reload validates a
candidate before publication and checks the expected graph revision. Failed
edits preserve the active graph. Matching members retain state under the
implemented reload contract.

The lifecycle test saves a snapshot and scalar document as separate managed
records. It does not provide an atomic whole-image transaction or transparent
resumption of an interrupted native activation.

## Program launch metadata

An explicitly packaged object may declare the ordinary text slot
`entry-selector` to name its parameterless program entry. Optional
`presentation` metadata accepts internal (the default), host, or headless.
`instances` accepts single (the default) or optional. These slots describe
launch intent; they grant no authority and do not instantiate an application.
See [the current milestones](../../neo-vm/documentation/development/launch-milestones.md).
