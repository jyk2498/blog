+++
title = "Ranked Last of Nine: When Branch Order Changes an HLS Schedule"
date = 2026-08-03
draft = true
tags = ["hls", "catapult", "scheduling", "ai-collaboration", "hardware-design"]
+++

The model gave me nine ways to bring a thread's initiation interval (II) back to target.
One was to flip a branch: reorder an `if` so the other case comes first. I put it last,
because as far as I knew, branch order cannot change what code does.

Of the nine, it was the only one that worked.

## What I got wrong about branch order

A non-pipelined `while` loop compiles to a single FSM. Each state is one cstep, one cycle,
and every operation is assigned to one of them at compile time. The loop below is what I
started from.

```cpp
while (1) {
    // common pre-ops
    if (heavy_case) {
        // long dependent chain
        downstream.push();
    }
    // common post-ops
    downstream.push();
}
```

Most traffic takes the light path, and the light path was slow. Its data was ready early and
it still finished late.

My first guess was port contention. Both branches push to the same channel, so I assumed
they were queueing for it. The synthesized RTL says otherwise:

```verilog
output [DATA_WIDTH-1:0] downstream_msg;   // data
output                  downstream_val;   // producer: data is valid now
input                   downstream_rdy;   // consumer: I can take it now
```

Not one set per call site. The two pushes carry different data onto that same bus, so a
one-hot mux sits in front of it:

```verilog
// sel_* = (FSM state == N) && path condition
assign downstream_msg = ({DATA_WIDTH{sel_heavy}} & data_heavy)
                      | ({DATA_WIDTH{sel_light}} & data_light);
```

Two pushes that can never fire together are free to share a cstep, and in the schedule they
often do. Port contention was not the constraint.

What I had missed was a different question. Look at the select signals: a call site fires
only while the machine stands in one particular state. The question was never whether the
port is free. It was how far a path has to walk before it reaches its own push.

## Where a path leaves

In software, a branch you don't take costs you nothing, because there is no machine to walk
through. In hardware there is one. A single iteration walks from the first state to the
last, then jumps back to the top. That jump is the only way out of the loop, and the code
above has exactly one of them — at the bottom, after the final push.

Now look at the light path. It skips the heavy block, so it does none of that work. But
those states still sit between it and the exit, and it has to walk through them to get
there. With one exit, every path takes as long as the longest.

```cpp
while (1) {
    // common pre-ops
    if (!heavy_case) {
        // only the post-ops this path needs
        downstream.push();
        continue;
    }
    // long dependent chain
    downstream.push();
}
```

The early `continue` builds a second exit. The light path stops walking the heavy tail, and
its iteration gets shorter. This is where branch order comes in. A fall-through path cannot
have an early exit, because it is what everything else falls through to. Only a leading
branch can carry a `continue`, so whichever case you put first is the one that finishes
short.

Which is why my rule was almost always right. Reordering branches usually changes nothing.
This time it decided which path got an exit at all.

## The most expensive failure is the quiet one

Deciding what to accept and what to reject has become one of the main things I do. There
are two ways to get it wrong.

1. **Accept a bad suggestion** and you pay the loop: implement, synthesize, place and
   route, simulate, read the report. I've [written about that cost
   before](/posts/feedback-signal-ground-truth/). It is expensive, but it ends with an
   artifact telling you something broke.
2. **Reject a good suggestion** and there is no loop. Nothing happens at all. The option
   leaves the world at the moment of judgment, and no report will ever mention that it
   existed.

What makes this hard is that rejecting is usually right. Eight of my nine candidates
deserved it: some simply wrong, some ignoring context that was never written down anywhere,
some correct in isolation but against how the team builds things. The habit gets rewarded
almost every time. The one time it doesn't, nothing tells you.

I got lucky. I paid the loop cost on the others first, then went back to the one I'd
dismissed. Had the II target been softer, I'd have stopped inside my own ranking. That
suggestion would have stayed rejected.

So the real price of not knowing your tool well enough isn't a wasted synthesis run. Those
are tuition. It's a working solution that was on the table, in writing, and left the table
because I couldn't recognize it.
