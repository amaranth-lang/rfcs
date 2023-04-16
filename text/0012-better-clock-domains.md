- Start Date: 2023-04-10
- RFC PR: [amaranth-lang/rfcs#0012](https://github.com/amaranth-lang/rfcs/pull/0012)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Better clock domains

## Summary
[summary]: #summary

(Re)define what clock domains exactly are and what one can do with
them, making them more explicit and flexible for a better abstraction
of clocks in the designs.

## Motivation
[motivation]: #motivation

Clock domains in amaranth are supposed to be first-level concepts,
except they're not entirely at this point.  Reset is a little tacked
on, and enable has not even entirely managed to be tacked.  OTOH they
are critical to the implemention at least on fpgas, and the
*m.d.\<domain\> += assignment* syntax is one of the key points of the
readability of the language.

The interaction with enables in particular could be a lot better.  A
lot of it is inheritance of nmigen, but in any case most of the
abstraction promised by EnableInserter is in reality lost when trying
to actually rely on it.  The purpose of this RFC is to propose useful
and consistent semantics for clock domains and their interactions with
modules and their elaboration.  They try to be compatible with
existing code, even if in some specific places some deprecations
and/or default changes could be considered pertinent but are not
required.

After some testing, it was noticed that the way to use EnableInserter
changes, an example is given at the end.

The main difficulty is that implementing it cleanly will require some
fundamental modifications to the way the implementation works.  But at
least having a clear target can help.


## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Clock domain

A clock domain is a tuple **(*clock signal*, *reset signal*, *enable
signal*, *default name*, *async_reset*)**.

It defines, for every signal driven by that domain, when the state of
the signal can change (posedge on the clock signal + enable = 1) and
when it is reset (reset = 1).  If async_reset is set and reset signal
is handled asynchronously, e.g. all driven signals are set to their
reset value immediately on a reset = 1.  If async_reset is unset they
are set on the next clock signal.

> **Note**
> Is the sync reset done on the next clk edge or on the next clk edge with enable = 1?

> **Note**
> Should we have both sync and async reset signals instead of a toggle?


### Global clock domain

A global clock domain is a clock domain that has been put in the
global environment of the design under a name and can be retrieved by
any module under that name.

A domain is current made global at its creation, see the ClockDomain
documentation.


### Bound clock domain

A bound clock domain is a triplet **(*module*, *name*, *domain*)**.
They allow to add assignments to a given module which are clocked
through the domain.  The signal assigned to is called *driven* by
the domain.

For a given module *m*, the bound clock domain with name *name* is
reached through **m.d.*name* **.

The bound clock domains for a given module come from three sources:

- Global clock domains
- Inherited clock domains
- Locally-bound clock domains

A given domain can be bound to multiple modules and multiple times to
the same module.

Anywhere one can use a domain one can provide a bound domain instead.


#### Binding a domain locally

A domain is bound locally to a module *m* by writing either:

- **m.domains.*name* = domain** which binds the domain under the name *name*
- **m.domains += domain** which binds the domain under its default name

For a given module it is not allowed to bind locally to the same name twice.

> **Note**
> Should we alias m.d and m.domains at that point?  That would allow m.d.name = domain

> **Note**
> Do we want to remove default names entirely?  There, and
> when creating global clock domains, are the two places where they're
> used.


#### Inheriting domains

When binding a submodule to a module through **m.submodules +=
submodule** (or the explictely named version) all the bound clocks of
the top module are inherited by the submodule.

> **Note**
> That means the global clock domains end up inherited too.  That's probably required to have renaming make sense.

> **Note**
> We could also define the global clock domains as the domains inherited by the top module (which has none otherwise) and remove them from the bound domains.  That could give a better-defined escape hatch.

Bound clock domains can be renamed in the submodule by wrapping them in a DomainRenamer object:

- **m.submodules += *DomainRenamer(mapping)(*submodule*)* **

The mapping is a *dict* with string keys and either strings or bound
domains as values.  The bound domains either as given or retrived by
name from teh top module are renamed for the sake of inheritance to
the name in the key.  As a special shortcut **DomainRenamer(value)**
is equivalent to **DomainRenamer({'sync': value})**.

> **Note**
> Should we allow DomainRenamer to add to the inheritance domains that are not bound in the top module, e.g. DomainRenamer({'toto': domain)(submodule) where domain is not bound but possibly just created?

> **Note**
> Should DomainRenamer only rename domains, or should it be capable of removing domains from the inheritance too?


### Creating a domain

A domain is created through building a *ClockDomain* object:

- **ClockDomain(*name*, *reset_less*, *clk_edge*, *async_reset*, *local*, clk=*signal*, en=*signal*, rst=*signal*)**
- **ClockDomain(*domain*, en=*signal*, rst=*signal*)**

The first variant create a domain from scratch.  The *name* gives the
default name, it defaults to None which means take it from the name of
the variable it is store in (optional prefix cd_ removed).  When true,
*reset_less* is equivalent to setting the reset signal to a fixed 0,
which happens to be the default. The parameter *clk_edge* defaults to
"pos", the other possible value "neg" means the clock signal is
inverted before use.

> **Note**
> Ideally, reset_less and clk_edge should go, and local should be
> replaced by an explicit function to push a domain to global scope,
> whatever global scope ends up meaning.  Also, name would also go if
> the concept of default name goes.

> **Note**
> Correctly implemeting clk_edge requires storing its value and if
> needed inverting the clk signal if it is set through domain.clk =
> signal

The second variant allows to derive from an existing domain.  For the
new domain the enable signal is the logical *and* of the initial
domain enable and the parameter, reset is the logical *or* of the
initial domain reset and the parameter.  Default name and async reset
flag are kept as-is.


### Accessing parameters on a domain

A *ClockDomain* object has three read/write members:

- *clk*: the clock signal
- *en*: the enable signal
- *rst*: the reset signal

They should be written only once, and only if they haven't been set to
non-default in the constructor.

> **Note**
> An error should probably be generated there.

These parameters are also reachable as members of a bound clock domain
object, e.g. one can write **m.d.sync.en** for instance.

The member *name* can be accessed on clock domains and bound clock
domains, and return the default name and the binding name
respectively.


In addition **ClockSignal(*name*)** and **ResetSignal(*name*)** can be
used to reach the clock and reset signal of a bound domain for the
module the expression is found in a statement it is added to.

> **Note**
> I'd love to see these two deprecated, their weird
> late-contextual-name-lookup is kind of tricky.  If they're not, then
> EnableSignal should be added too, probably.


### Creating a new domain with derived enable or reset signals

EnableInserter and ResetInserter can be used to create a new domain
with a changed enable or reset.  Specifically:

- **EnableInserter(*domain*, *signal*)** is equivalent to **ClockDomain(*domain*, en=*signal)**
- **ResetInserter(*domain*, *signal*)** is equivalent to **ClockDomain(*domain*, reset=*signal)**

Remember than the new domain does a *and* between old and new enable,
and a *or* between old and new reset.

> **Note**
> An EnableReplacer and ResetReplacer could be imagined to replace
> without the *and* or the *or*, but that already can be done through
> member setting.


### Compatible domains and assignments

A signal can be driven (e.g. assigned to) from domains that are
compatible.  Two domains are said to be compatible iff they have the
same clock signal.  Note that a domain with a given clock signal is
*not* compatible with another domain which has the same signal but
inverted.

> **Note**
> The verilog implementation is not particularly a problem.  It's an
> always @(posedge clk) if(enable1 | enable2) signal <= (expr1 &
> (enable1 & ~enable2)) | (expr2 & enable2); end which should
> trivially be recognized as a dffe by the synth.

> **Note**
> Technically we could go further and accept any clock signal
> combination that ensures no metastability, but there the
> implementation would be a problem.  In fact the fpgas would just
> hate that.


### New uses made possible by the definitions

Issue
[amaranth-lang/amaranth#0762](https://github.com/amaranth-lang/amaranth/issues/762)
would be made obsolete.


#### Instances-wrapping modules running on sync

See issue [amaranth-lang/amaranth#0285](https://github.com/amaranth-lang/amaranth/issues/285).

If you have a Instance of something with clock, enable and reset
inputs, which a lot of on-die blocks in fpga have, you can just wrap
it in a Module that hooks the Instance to m.d.sync.clk, m.d.sync.en
and m.d.sync.rst.  Previously there was no way to reach the enable
signal.


#### Clock-structure-agnostic modules

See issue [amaranth-lang/amaranth#0749](https://github.com/amaranth-lang/amaranth/issues/749).

Imagine one wants to create a sync signal generator for video. 

A clock divider module can take sync and run a counter from it.  It
then outputs a signal that's one on reaching the maximum and zero the
rest of the time, e.g. an enable signal but also the hsync pulse.

The caller can take that signal, create a derived domain from using
using the new enable, and the automatic *and* will ensure it stays a
valid enable signal for the underlying clock signal.

And then that derived domain can be used as sync of a second instance
of the same clock divider (with different parameters) and have it
generate another enable signal that's also the vsync pulse.

The clock divider module does not need to care whether sync uses an
enable signal or not, it doesn't ever need to look at it.  It just
counts and outputs a carry.

The whole sync generator module also doesn't need to care whether its
input clock domain is "pure" or is an enable on a faster clock.  It
just works transparently whichever the case is.


### Conversion example for EnableInserter

As a example, let's take a MainModule which takes a *sync* clock
domain as input and wants to give a submodule to clock domains *s1*
and *s0* corresponding to alternative pos_edges of *sync*.  IOW, the
raising and dropping edges of a divided-by-two clock.

In the original code, we assume that *sync* has itself not been
EnableInserter-ed.

The submodule:

```python
class SubModule(Elaboratable):
    def __init__(self):
        ...

    def elaborate(self, platform):
        m = Module()
        m.d.s1 += ...
        m.d.s0 += ...
        return m
```

The main module, original code (line wrapped for readability, order
between DomainRenamer and EnableInserter is critical):

```python
class MainModule(Elaboratable):
    def __init__(self):
        self.phase = Signal()
        self.s1_en = Signal()
        self.s0_en = Signal()

    def elaborate(self, platform):
        m = Module()

        m.d.sync += self.phase.eq(~self.phase)
        m.d.comb += self.s1_en.eq(self.phase)
        m.d.comb += self.s0_en.eq(~self.phase)

        m.submodules.sub = sub = DomainRenamer({'s0': 'sync', 's1': 'sync'})
                                  (EnableInserter({'s0': self.s0_en, 's1': self.s1_en})
                                   (SubModule()))

        return m
```

New version, with EnableInserter:

```python
class MainModule(Elaboratable):
    def __init__(self):
        self.phase = Signal()
        self.s1_en = Signal()
        self.s0_en = Signal()

    def elaborate(self, platform):
        m = Module()

        m.d.sync += self.phase.eq(~self.phase)
        m.d.comb += self.s1_en.eq(self.phase)
        m.d.comb += self.s0_en.eq(~self.phase)

        s0 = EnableInserter(m.d.sync, self.s0_en)
        s1 = EnableInserter(m.d.sync, self.s1_en)

        m.submodules.sub = sub = DomainRenamer({'s0': s0, 's1': s1})
                                  (SubModule())

        return m
```

New version, with the new constructor:

```python
class MainModule(Elaboratable):
    def __init__(self):
        self.phase = Signal()
        self.s1_en = Signal()
        self.s0_en = Signal()

    def elaborate(self, platform):
        m = Module()

        m.d.sync += self.phase.eq(~self.phase)
        m.d.comb += self.s1_en.eq(self.phase)
        m.d.comb += self.s0_en.eq(~self.phase)

        s0 = ClockDomain(m.d.sync, en = self.s0_en)
        s1 = ClockDomain(m.d.sync, en = self.s1_en)

        m.submodules.sub = sub = DomainRenamer({'s0': s0, 's1': s1})
                                  (SubModule())

        return m
```

New version, if one wants s0/s1 accessible to MainModule and other
submodules if others exist (can be done alternatively with
EnableInserter):

```python
class MainModule(Elaboratable):
    def __init__(self):
        self.phase = Signal()
        self.s1_en = Signal()
        self.s0_en = Signal()

    def elaborate(self, platform):
        m = Module()

        m.d.sync += self.phase.eq(~self.phase)
        m.d.comb += self.s1_en.eq(self.phase)
        m.d.comb += self.s0_en.eq(~self.phase)

        m.domains.s0 = ClockDomain(m.d.sync, en = self.s0_en)
        m.domains,s1 = ClockDomain(m.d.sync, en = self.s1_en)

        m.submodules.sub = sub = SubModule()

        return m
```

An interesting detail is that the new code works whether or not sync
comes with an enable on it.  Transformation is identical for
ResetInserter.


## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This proposal deliberately does not say how the semantics should
actually be implemented.  It probably could be done by slightly
extending the current fragment-transform model.  It's probably better
to rethink that model, but the global consequences would probably be
way larger than just clock domains.  The semantics and syntax proposed
do not and should not constrain that, and the larger-scale
implementations decisions could go in a specific RFC if/once we agree
on a variant of this proposal as a target.


## Drawbacks
[drawbacks]: #drawbacks

That requires big, big changes in the current implementation.  That's
always a problem.  Plus some changes in existing code around the
inserters.


## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This proposal is first about a better, transparent support of enables
so that modules can be written pretty much independently from the
structure of the inbound clock domains.  The
systematization/clarification of domain transmission is a bonus.

The idea was to try to not require changes in existing code, and just
generalize what the existence of EnableInserter implies.  Breaking
changes may allow to go further, but I'm not sure in which direction.
Possibly in the ClockDomain constructor prototype, but that may not be
enough to justify a breaking change by itself.


## Prior art
[prior-art]: #prior-art

The prior art is 
Discuss prior art, both the good and the bad, in relation to this proposal. Does this feature exist in other programming languages and what experience have their community had?

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that Amaranth sometimes intentionally diverges from common language features.


## Unresolved questions
[unresolved-questions]: #unresolved-questions

Clock gating (e.g. creating a new clock signal built from a previous
clock signal and an enable) is out of scope.  It can be an explicit
block, it can be an under-the-cover optimization by the synth for when
the resources are available in the device (some models of fpga have
one or two, some have 80+) and the optimizer notices that a given
clock/enable is used a lot.


## Future possibilities
[future-possibilities]: #future-possibilities

Not sure.  It's a lot about consolidating what's there, not creating
new things.  There may be some work to do around reset management,
where this definition of clock domains could make things easier to
think about.

Also, maybe a "is_compatible" cross-domain check can be made public,
to automatically decide in multi-domain modules if CDC structures are
needed.
