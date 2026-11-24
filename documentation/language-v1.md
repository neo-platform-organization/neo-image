# Language v1 contract

Status: implementation contract and target specification, revised 2026-11-25.
The implemented baseline below is testable today. Requirements explicitly marked
as v1 targets are not implemented merely because they appear in this document.
The v1 acceptance gate remains open until all targets pass conformance tests.
This document supersedes descriptions of the retired implicit-primitive VM.
It does not change the read-only whitepaper.

## One world, three representations

Warp is the serialized object graph. Weft is an object-oriented authoring view.
Knight describes pure calculations within that graph. They share identity,
values and persistence; there are not three independent heaps or runtimes.
The compiler translates executable behavior into native code. Parsing,
validation, inspection and projection are inert host operations.

A consumer chooses a supported interpretation. Inspection never grants authority
or executes a handler. An alternate executable interpretation must satisfy its
own validation and effect rules. Unsupported interpretations fail explicitly;
arbitrary code cannot become pure merely by attaching different metadata.

## Implemented baseline

### Graph and object identity

A document has one `(warp 1 ...)` root, an entry receiver/selector and explicitly
named objects. Parentheses group graph forms; their first atom identifies the
form, followed by ordered operands. Empty forms are invalid. Quoted text is data.
`@name` denotes a graph-local reference; `self` denotes the actual message receiver.
Host addresses are never graph identities.

```lisp
(warp 1
  (entry counter start)
  (object counter
    (slot value 0)
    (knight add x y (+ x y))
    (handler start (set self value (call self add 19 23)))
    (handler reading (get self value))))
```

Names are unique where the current graph lookup expects a named object or member.
Ambiguous duplicates fail validation. Named links can form reference cycles.
An object has at most one delegate; delegation cycles are rejected. Delegation
supplies behavior, while the actual receiver supplies `self` and receiver-local
slots. It does not create a class or copy the delegate's mutable state.

Packages bound access to object references, imports and resource grants. An
export declaration makes a selector eligible for an authorized import. An import
declaration alone grants nothing. Absent package metadata uses the current root
package. Package checks are compiler-enforced within a session, not independent
hardware compartments for each package.

### Values and conditions

Implemented kinds are checked signed-32-bit integers, finite binary64 values,
text, bytes, integer vectors, binary64 vectors, and graph-local references.
Binary64 is unavailable on the standalone integer profile. Rich values have
explicit operations such as `set-text`, `set-bytes` and `set-ref`; the current
implementation does not support arbitrary slot-kind changes.

Integer addition, subtraction and multiplication reject overflow. Comparisons
return 0 or 1. A condition is an integer: zero selects false and any nonzero
value selects true. A false `if` without an alternative returns zero. Operands
are positional; an operator does not require artificial left/right wrappers.

Text literals contain UTF-8 scalar sequences. They reject malformed continuation
bytes, overlong encodings, surrogate encodings and values beyond U+10FFFF.
Escapes are `\"`, `\\`, `\n`, `\r` and `\t`. Raw ASCII controls are rejected.
There is no implicit Unicode normalization. Byte values use hexadecimal syntax
and may contain arbitrary bytes, including zero and malformed UTF-8.

For compatibility, `text-length` counts decoded UTF-8 bytes and `text-byte`
indexes those bytes. They do not count characters or grapheme clusters.
Sequence indexes are zero-based. Text-byte, bytes-at and integer vector-at return
-1 outside their bounds, including negative indexes. This is ambiguous for
integer vectors containing -1 and is a compatibility limitation, not the v1
error design. Binary64 vector indexing instead reports a bounds failure.
Text is not a ready-to-use editor string system yet.

### Constructing and inspecting text

The following operations compile to native loops. Their operands are evaluated
left to right; each text operand is captured before evaluating a later operand.
Later handler effects therefore cannot change an earlier operand's value.

| Form | Result |
| --- | --- |
| `(text-concat first second)` | A new text value containing both inputs |
| `(text-slice text start count)` | A new text value containing count Unicode scalars starting at start |
| `(text-scalar-length text)` | Number of Unicode scalar values |
| `(text-scalar-at text index)` | Unicode scalar number at the zero-based index |

For example, `(text-slice "aλ🙂z" 1 2)` produces `"λ🙂"`, and
`(text-scalar-at "aλ🙂z" 2)` returns 128578. A combining mark is a separate
scalar; these are not grapheme or cursor-movement operations. Text-length and
text-byte continue to use bytes. The older byte/integer-vector out-of-range
sentinel behavior is unchanged.

Slice start/count must be nonnegative integers, start must be at most the scalar
length, and count must fit the remaining suffix. An empty slice at the end is
valid. Invalid scalar indexes/ranges fail explicitly. Constructors must fit the
serialized value-token capacity, including quote/backslash/control escaping.
Capacity or range failure rolls back the send's staged graph changes.

Text constructors are rich values used by text consumers such as return-text,
set-text, text-length, let, and text arguments. A text-returning Knight function
is called with call-text, which checks the returned kind:

```lisp
(knight greeting name (return-text (text-concat "Hello, " name)))
(handler start (text-scalar-length (call-text self greeting "λ")))
```

Pure-to-pure calls use an explicit target, for example
`(return-text (call-text @formatter greeting name))`. These operations do not
grant effects or permit pure code to read mutable receiver state. Knight calls
are still expanded during compilation; this does not add native recursion.

Constructed values carry a native-generated serialized token as well as decoded
bytes, so messages, live slots, snapshots and Weft projections use the existing
validated value contract. No host text evaluator runs these loops. CPU encoding
of generic byte loads/stores, masks and shifts remains in the depth CPU backend;
text semantics and lowering remain in surface. Value and platform ABIs are
4 and 6 respectively; matching source components must be rebuilt together.

### Execution and pure calls

Byte and integer-vector construction also runs as native code:

| Form | Result |
| --- | --- |
| `(bytes element ...)` | Bytes from integer operands in the inclusive range 0 to 255 |
| `(bytes-concat first second)` | Both byte sequences, in order |
| `(bytes-slice source start count)` | A byte-indexed slice |
| `(vector element ...)` | An immutable integer vector from computed operands |
| `(vector-concat first second)` | Both integer vectors, in order |
| `(vector-slice source start count)` | An element-indexed integer-vector slice |

Empty constructors and end-position slices with count zero are valid. Slice
ranges must fit completely; negative positions/counts fail. Byte elements outside
0 through 255 fail rather than wrapping. Existing token, vector and native-code
quotas still apply: bytes fit 511 elements, integer vectors fit 31. These bounded
operations do not establish a general dynamically allocated collection heap.

Operands are captured left to right. A later message cannot change an already
captured earlier operand. Constructors never expose writable aliases. A failed
range, element or capacity check discards the send's staged changes.

Use `return-bytes` and `call-bytes` for pure byte-returning Knight functions;
`return-vector` and `call-vector` do the same for integer vectors. Existing
set-bytes/set-vector operations accept constructed values. Weft projections
preserve these operations and their call kinds.

```lisp
(knight packet first second
  (return-bytes (bytes-concat first (bytes-slice second 0 2))))

(handler start (bytes-length (call-bytes self packet #ff #0080aa)))
```

Native code computes the elements and ranges. At the worker boundary, surface
serializes raw constructed bytes/vectors for messages, snapshots and source
views, as it already does for binary64 vectors. This encoding is not expression
evaluation. All lengths are checked before reading or publishing returned data.

`send` invokes a handler; `call` invokes a Knight function. Both accept up to
four positional arguments in the current implementation. Parameters and `let`
bindings are scoped to their activation. Calls compose, but the compiler expands
their bodies during compilation. Recursive calls consequently reach a compilation
limit; there is no recursive native-call implementation today.
Knight has no mutable `self` receiver context: current pure-to-pure calls use an
explicit graph target such as `(call @math add x y)` with explicit input values.

Handlers can stage receiver state changes, send permitted messages and request
explicitly granted effects. `do` sequences a handler's expressions. `while`
is a handler construct with an integer condition. A pure Knight calculation
cannot mutate slots, send effectful messages, perform I/O, draw, or schedule
work. Effects cannot be acquired through a different source view.

Calls are compiled before entering native execution. There is no tree-walking
execution fallback. Failure to compile is a reported failure, not a switch to
another execution model. Compilation currently occurs per prepared execution;
general compiled-code caching and dependency invalidation are unfinished.

### Publication and failure

A send prepares native code and executes against staged graph values. Native
failure, bounds errors and checked overflow must not publish partial graph writes.
Successful synchronous sends publish their resulting graph state. Host callbacks
and their returned data are checked before publication.

Asynchronous requests require explicit grants. A sink accepts an entire staged
request batch or rejects it. External I/O happens later during polling; a
completed external write cannot be undone by a subsequent handler failure.
Graph rollback is not a distributed transaction over external devices.

The hosted worker has timeout/crash containment. Standalone currently runs
trusted code without equivalent process containment or active-code preemption.
These profiles must not claim identical security guarantees.

### Live state and views

Snapshots preserve graph definitions and current graph values. They do not
preserve host handles, grants, native pages, active frames or pending queues.
Reopening requires fresh authority. Graph edits use expected revisions; stale
edits fail. Failed validation or replacement preserves the active graph.

Weft projection and lowering must preserve the supported graph. Unsupported
projection fails rather than discarding members. Clone, deletion, live replacement
and bounded reflective views are supported mechanisms, not a claim of a general
object heap or complete reflective activations.

## Current engineering limits

| Resource | Current bound or behavior |
| --- | --- |
| Image input through hosted frontend | 64 KiB |
| Graph allocation | 2048 cells, including the reserved zero cell; forms and atoms consume cells |
| Names / non-value atoms | 63 bytes plus terminator; references include their @ prefix |
| Serialized rich-value token | 1023 bytes plus terminator; text/byte/vector capacity is independent of names |
| Text / byte value | At most 1021 unescaped UTF-8 bytes / 511 hexadecimal-encoded bytes; escapes reduce usable text capacity |
| Compilation and parser nesting | 48 frames; some forms consume additional frames |
| Handler / Knight arguments | 4 |
| Program arguments | 16, subject to text-token limits |
| Byte literal / graph I/O read | 31 bytes |
| Numeric vector | At most 31 elements and must also fit the value token |
| Native body | 64 KiB |
| Staged requests per send | 8 |
| Pending event queue | 32 entries |
| Lifetime | Bounded session allocation; each send owns a heap-backed staging frame, released on success or failure; no general garbage collector |

These are implementation limits, not the desired capacity of an OS. They must
be reported rather than silently truncating values. Passing tests below these
limits does not establish practical text, collection or application support.
Native language ABI 4 and platform ABI 6 carry compact execution frames and their
explicit word counts. Matching components must be rebuilt together. Compilation
assigns records to live rich slots, supplied arguments and required temporaries.
Unused method bodies and ordinary graph atoms reserve no rich execution record.
The compilation-local map is neither object identity nor authority and is never
serialized. I/O payloads reserve eight contiguous records when needed.

`neo compile-info IMAGE` reports compilation without running the entry. The
canonical exercise entry uses 22,900 staging bytes rather than the former
4,786,612 bytes. This measures staging only, not total memory or execution speed.
Persistent source and live-value arrays remain dense; a general heap and garbage
collector are still unfinished.

Hosted workers retain capacities rounded to 4,096 words, restarting on growth.
Every request clears the retained capacity and transfers only the actual frame.
Counts and packet metadata are checked before publishing results. These transfer
bounds do not provide hardware bounds checks for individual native instructions.
Graph I/O retains its smaller limits, and saved launcher-profile arguments remain
bounded separately by their configuration format.

## v1 targets still required

The object graph remains authoritative. The following extend the baseline
without adding application-specific behavior to C:

1. Values: practical separately sized text/byte/collection storage with explicit
   quotas; general collection/value construction beyond the implemented text
   scalar operations; immutable value sharing or copy semantics that do not expose native pointers;
   indexing with explicit absence/failure instead of an ambiguous integer sentinel.
2. Objects: explicit creation, cloning, deletion and delegation semantics across
   rich values, references, live activations and granted resources; safe dynamic
   value-kind changes with errors at incompatible operations.
3. Knight: native composition and bounded recursion over explicit immutable
   inputs. State needed by a pure calculation is supplied as an input. Contracts
   may validate values but cannot add effects. Numeric errors stay explicit.
4. Reflection: inert, lossless supported views; revision-checked edits and
   alternate interpretations with effect checks. No view manufactures authority.
5. Execution: dependency-aware compilation, invalidation and safe reclamation of
   graph/value/native memory. Active code cannot observe freed or replaced data.
6. Services: graph-visible operations and capability lifecycle for documented
   platform resources. Granting, delegation, cancellation, shutdown, unsupported
   operations and publication ordering have portable contracts.
7. Acceptance: independent black-box tests for those contracts, including failure
   cases and equivalent views, on each claimed target/profile.

Representation details such as heap layout, GC algorithm and native call ABI
remain implementation choices. They must preserve these observable rules.
Type-contract syntax, richer collection syntax and general reflective activation
editing need concrete designs during F2-F4; they are not accepted syntax today.

## Evidence and reading order

The independent [conformance suite](../../neo/tools/conformance.mjs) checks the
implemented baseline through the public CLI. It does not import compiler
internals or the older regression fixtures. Its passing banner explicitly leaves
the full v1 gate open. Existing regressions add package, lifecycle, scheduling,
reflection and platform integration evidence.

Use [foundation milestones](../../neo/documentation/development/foundation-milestones.md)
for the implementation order and [current format](runtime-format.md) for syntax.
The source of truth for current constraints is the checked-in implementation:
`neo-vm/jit.h`, `neo-vm/language/jit.c`, `literal.h` and `weft.c` in the VM repo.
