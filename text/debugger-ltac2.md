- Title: Ltac2 Debugger Support

- Driver: Jim Fehrle

Identifies requirements for supporting Ltac2 debugging and outlines the associated
changes at a high level.

# Requirements

Ltac2 debugging should support all the features currently supported for Ltac debugging.

In addition, we should provide the ability to debug tactics that combine Ltac2
with Ltac.  That’s likely to be a very common use case for Ltac2 for the
foreseeable future:

- The vast majority of packages (all?) are currently written using only Ltac.
  Ltac2 users are likely to use these. Those users should be able to debug
  problems and observe how tactics are processed. Conversion from Ltac to Ltac2
  will happen slowly--and for some packages, never.

- Anyone converting a package from Ltac to Ltac2 is likely to proceed
  piecemeal, converting a small number of Ltac tactics at a time. Having the
  debugger handle combined Ltac/Ltac2 tactics will likely be very helpful to them
  in this process. Also, the conversions may be buggy for some time, so it would
  be good to have debugger support.

@JasonGross and @MSoegtropIMC agree that supporting debugging of combined
tactics is very useful.  (See [here](https://github.com/coq/coq/pull/15737#issuecomment-1062413445) and
[here](https://github.com/coq/coq/pull/15737#issuecomment-1062639786).)

# Protocol

Debugging Ltac2 code requires XML API support for the same operations used to
debug Ltac tactics:
- get the current goals
- set/clear/stop at breakpoints, step, etc.
- get the call stack
- get the variables defined in a specified stack frame

These will be supported through the existing XML calls used for Ltac debugging.
The changes will be in server code.  CoqIDE and other IDEs will not need to
distinguish between Ltac and Ltac2 debugging.  No changes to CoqIDE are expected.

Since Coq is single threaded, the Ltac debugger calls a read loop
`Tactic_debug.Comm.read` when the stopping point is in the Ltac interpreter.
The Ltac2 interpreter must similarly call the read loop when the stopping point
is in the Ltac2 interpreter.  Perhaps the read loop code can be fully or mostly
shared between the two.  In addition, some of the code supporting the XML APIs
will be shared between Ltac and Ltac2 debugger code, which can be done below
the existing DebugHook interface.  For example:
- Breakpoint setting – CoqIDE doesn’t know whether the user is setting a
  breakpoint in an Ltac or Ltac2 tactic.  The existing set/clear breakpoint code
  will be enhanced to make breakpoints available in both environments.
- Getting the call stack – Combined Ltac/Ltac2 tactics will show a single
  stack that combines both Ltac and Ltac2 calls.  This necessarily means
  combining information from both the Ltac and Ltac2 interpreters.  The next
  section discusses how to handle this without overly coupling the internals of
  the two interpreters.
- Getting the variables for a stack frame – Clearly requires combining
  information from both interpreters.

Fetching the current goal may not require communication between Ltac and Ltac2.
Stepping in/out/over code and continue operations may also be independent.

# Getting the Call Stack

Ltac and Ltac2 currently maintain separate stacks in different formats.  We
shouldn’t try to force them to use a common format.  Instead, Coq will maintain
information sufficient to recreate a unified stack from the separate stacks.
This could be done in various ways.  For example, have each Ltac* interpreter,
when entered, create a marker stack frame that indicates whether it was invoked
from the other interpreter.  The Ltac* interpreters then ignore these frames.
The API to get the call stack will use this information (and knowledge of which
interpreter the debugger is stopped in) to stitch together a combined stack for
user display.  For example:

```
Ltac2 stack:		Ltac stack:
Tactic1			Ltac_tactic3
Tactic2			Marker: from Ltac2
Marker: from Ltac1	Ltac_tactic4
```

When stopped in the Ltac interpreter, the stack returned to CoqIDE and shown to
the user is:

```
Ltac_tactic3
Tactic1
Tactic2
Ltac_tactic4
```