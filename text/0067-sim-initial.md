- Start Date: 2024-06-03
- RFC PR: [amaranth-lang/rfcs#67](https://github.com/amaranth-lang/rfcs/pull/67)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Combinable simulation triggers with `initial()`

## Summary
[summary]: #summary

A new method, `initial()`, is added to combinable triggers in the simulator. It makes the trigger fire once right after creation.

## Motivation
[motivation]: #motivation

Consider the following process, meant to replace a bit of combinational logic:

```
a = Signal(16)
b = Signal(16, init=1)
o = Signal(32)

async def process(ctx):
    async for a_val, b_val in ctx.changed(a, b):
        ctx.set(o, a_val * b_val)
```

It creates a trigger object that fires on every change of `a` and `b`, ensuring that `o = a * b`. However, if the initial state of `a`, `b`, and `o` is not consistent (eg. the first line is changed to `a = Signal(16, init=2)`), the trigger will not fire, and the process will fail to perform its function.

It is not easily possible to fix this problem — `ctx.get()` cannot be used in a process and the only way to obtain signal values involves waiting on a trigger object. Further, even if `ctx.get()` was allowed in processes, solving the problem in the obvious way requires code duplication:

```
async def process(ctx):
    ctx.set(o, ctx.get(a) * ctx.get(b))
    async for a_val, b_val in ctx.changed(a, b):
        ctx.set(o, a_val * b_val)
```

A similar problem also appears for flip-flops when asynchronous resets are involved. Consider the following code:

```
d = Signal()
q = Signal()

async def ff_process(ctx):
    async for clk_edge, rst_val, d_val in ctx.tick().sample(d):
        if rst_val:
            ctx.set(q, 1)
        elif clk_hit:
            ctx.set(q, d_val)
```

While the code is meant to implement an asynchronous-set flip-flop, it fails to do so correctly when the `sync` domain has an asynchronous reset and the reset is initially asserted.


## Guide-level and reference-level explanation
[guide-level-explanation]: #guide-level-explanation

When a trigger object is used in an `async for` loop, the `.initial()` method can be used to ensure the loop body executes once immediately, in addition to any other triggers:

```
async def process(ctx):
    async for a_val, b_val in ctx.changed(a, b).initial():
        # Executed immediately after starting the process, then executed again every time `a` or `b` changes.
        ctx.set(o, a_val * b_val)
```

```
async def ff_process(ctx):
    async for clk_edge, rst_val, d_val in ctx.tick().sample(d).initial():
        # Executed immediately after starting the process if reset is asynchronous and initially asserted;
        # then executed on every clock rising edge and (if reset is asynchronous) every reset rising edge
        if rst_val:
            ctx.set(q, 1)
        elif clk_hit:
            ctx.set(q, d_val)
```

The `.initial()` method is meant only for use with `async for` — passing the result of `.initial()` to plain `await` is an error.


## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Both combinable and initial trigger objects are amended with one new method:

- `initial()` 
  - Create a new trigger object of the same type by copying the current object and appending an initial trigger. This causes the resulting trigger object to fire once immediately after first starting iteration by an `async for` loop. Awaiting such a trigger with plain `await` is an error. The `.initial()` trigger doesn't contribute any components to the yielded value.

## Drawbacks
[drawbacks]: #drawbacks

Making a combinational process that works correctly is more typing than doing it incorrectly (skipping `.initial()`).

An increase in language complexity.

The `.tick()` part is an obscure edge case.


## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This RFC proposes to leave current semantics unchanged and add `.initial()` as something that has to be explicitly requested. Arguably, this can be seen as the wrong default. An alternative would be to make the "initial" behavior default whenever `async for` is involved. However, this in turn can be a very wrong default for other cases, and the flexibility of combinable triggers means that choosing a good default would require horrifying heuristics.

## Prior art
[prior-art]: #prior-art

The Verilog `always @*` construct has the same problem, resulting in pain and horrible workarounds whenever Verilog has to be emitted, including Amaranth's Verilog backend.

The SystemVerilog `always_comb` construct implicitly performs an equivalent of `.initial()`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- is `.tick().initial()` worth the complexity budget?
- should we make writing the wrong implementation of combinational logic harder, somehow?

## Future possibilities
[future-possibilities]: #future-possibilities

None.
