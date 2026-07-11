+++
title = "I Braced for the Hard Part; the Easy Part Broke: A Jagged-Frontier Story from an HLS FIFO"
date = 2026-07-11
draft = false
tags = ["jagged-frontier", "HLS", "FPGA", "AI-collaboration", "hardware-design"]
+++

Back in 2024 I read Ethan Mollick's *Co-Intelligence*, hoping to get better at actually
using AI tools. One idea stuck with me: the **jagged frontier**. Draw a map of what an AI
can do, and the edge of it isn't a smooth line — it's a ragged coastline. On one task the
model beats an expert; on the task right next to it, one that looks no harder to us, it
fails in a way that's almost funny. The hard part is that this edge doesn't match human
intuition. Tasks that look easy to us are sometimes where the model breaks, and tasks
that look hard are sometimes where it shines — and you can't tell which is which from the
outside. Mollick's advice was to find out by hand: poke at the model until you learn the
shape of its coastline.

I hit that edge recently, and what makes it worth telling isn't that the model failed —
it's *where* it failed. I had braced for the hard part, and it sailed through; the part
I'd waved off as trivial is the one that broke. That flip is the whole post.

## The task

A thread on a hot path was doing too much, so the plan was to split it in two and connect
the halves with a small FIFO. Splitting a thread is the part that can break — state has to
land on the right side of the cut — so that's where I put my attention. The FIFO was an
afterthought: connect the halves with a shallow, narrow FIFO (say 16 deep, 14 bits wide),
then run synthesis and place-and-route.

The timing report came back much worse. My first guess was the thread split — the tricky
part I'd braced for. But the split was clean: no logic error, no corner case, no bug. The
problem was entirely in the part I'd called trivial. Instead of instantiating one of the
FIFO primitives already in our library, the model had written its own — a register array
with hand-rolled pointers. That was what wrecked the timing.

## Why the hand-rolled FIFO is expensive

The code the model wrote was, in effect, a write into an indexed array:

```cpp
mem[tail] = din;
```

Harmless on the page. But an indexed write into a register array, with a variable index,
doesn't stay compact. The tool flattens it into a per-entry decode:

```cpp
if (push && tail == 0)  entry0  <= din;
if (push && tail == 1)  entry1  <= din;
// ...
if (push && tail == 15) entry15 <= din;
```

Every entry becomes its own bank of flip-flops, and each bank gets its own clock-enable:
`CE_k = push AND (tail == k)`. For a 16-deep, 14-bit FIFO, that's 224 flip-flops driven
by **16 distinct enable nets** — one per entry, each feeding that entry's 14 flip-flops.

```
  tail --->[ decode ]---> 16 one-hot lines
                          |
  push -------------------|  (AND'd into each line)
                          v
             CE0   CE1   CE2   ...   CE15     16 distinct clock-enable nets
              |     |     |           |
              v     v     v           v
            [e0]  [e1]  [e2]   ...  [e15]     each entry = 14 FFs on one shared CE

  16 entries x 14 FF = 224 FF  ->  16 control sets
```

Two costs follow, and neither is visible in the source.

**First, control sets.** On a Xilinx FPGA, the flip-flops packed into one slice have to
share a *control set*: the same clock, clock-enable, and set/reset. Sixteen distinct
clock-enables means the 224 flip-flops split into at least sixteen control-set groups
that can't share slices freely. The array spreads out across the fabric — more routing,
half-empty slices. It's a textbook control-set problem, and it's invisible in the C++.

**Second, the enable network.** The enable signals (CEs) come from a decode of `tail`,
ANDed with `push`. So each CE sits at the *output* of a decode cone — a stack of logic
that has to resolve before the enable is even valid — and it then has to fan out across
its entry to drive every flip-flop. A signal that comes off a deep logic cone and then
fans out widely is one of the worst patterns you can hand to place-and-route, and here it
landed in a region that was already congested. Vivado's usual fix for a high-fanout net
is to replicate its driver, so each copy drives fewer loads — but that trades one problem
for another:

```
  before:  drv ---------------> many loads across the array   (wide fanout)

  after :  drv -+- copy1 -> ...
                |- copy2 -> ...    fanout per net down, but
                +- copy3 -> ...    total LUT/net count up
                                   -> net loss inside a congested region
```

Per-net fanout drops, but the total LUT and net count goes up — right where the fabric
could least afford it. A rewrite that looked free on the page pushed the timing of
everything around it over the edge.

## What the primitive does instead

The FIFO in our library maps to distributed RAM (LUTRAM): the same LUTs, opened up as
small writable memories. The key point is that the write-address decoder lives *inside*
the primitive, as dedicated silicon — not as fabric logic you synthesized:

```
  wr_addr (tail) --->+--------------------------------+
  wr_en  (1 net) --->|   write-address decoder         |  <- hardened inside
  din            --->|   (dedicated silicon)           |     the primitive
                     |             |                  |
                     |             v                  |
                     |   word0 word1 ... word15        |   distributed-RAM cells
                     |             |                  |
  rd_addr (head) --->|             v                  |
                     +---------- dout ----------------+

  one write-enable, one address  ->  no per-entry CE, no control-set split
```

You give it one write-enable and an address, and it puts the data in the right word. No
sixteen-way enable decode, no sixteen control sets, no fanout cone for the tools to
fight. It packs tight and routes cheap — which, in a congested region, is the whole game.

The fix was small: delete the hand-rolled FIFO, instantiate the primitive. The timing
went back to the baseline.

## Why did it hand-roll it at all?

It wasn't a weak setup. The model was a current, top-tier one with full access to the
codebase — this wasn't a snippet handed to it in isolation. So take the tidy explanations
one by one, and watch each one fail.

- *Maybe implementing a FIFO is just trivial for the model, so it never thought to look
  for a primitive.* This is my best guess, and it might be part of it — a human finds
  hand-rolling a FIFO tedious, and that tedium is a quiet push to go find the ready-made
  version, a push the model never feels. But it doesn't fully hold up. The model didn't
  even have to go looking: a FIFO instantiation was sitting in the code right next to what
  it was editing, and "too lazy to search" doesn't explain ignoring an answer already in
  view.
- *Maybe it didn't know the convention — nobody told it we instantiate from the library.*
  True, I didn't say it out loud. But the convention was right there in the neighboring
  code, so the context wasn't missing; it just went unread.
- *Maybe it simply didn't check its own work.* This one is true: I don't think the model
  ever opened the timing report it had generated, so it had no way to know anything was
  wrong. But this explains the wrong thing. It explains why the model didn't *notice* the
  bad result — not why it *chose* to hand-roll the FIFO to begin with.

Every explanation has a hole, and I've stopped trying to force one — because the fact
that no clean cause fits is itself the point. This is what the jagged frontier looks like
up close. The line between "the model does this brilliantly" and "the model does this the
wrong way" didn't fall where any reasoning from the outside would have put it. It just sat
where I wasn't looking.

## What stayed human

There are two honest readings, and I don't get to keep only the flattering one.

**One: my map was wrong, and fixing it is the human's job.** I braced for the thread
split and waved off the FIFO, and I had it backwards. The lesson worth keeping is that in
hardware an easy-*looking* task isn't a safe one — the better the model is at building
something, the more easily it buries a real design decision inside a job that reads as
trivial, and I only learned this because my guess failed first.

**Two, with equal weight: this is a tooling gap, not human knowledge.** A strong agentic
model with the whole codebase in reach, and the answer sitting in adjacent code, still
shipped the worse design and never read its own report. That's not something a person
knows and a model can't — it's a workflow that didn't check its neighbors or its output,
and better defaults would close it. On this reading my "edge" is temporary, and shrinking.

I can't pick between them, and forcing a winner would be the tidy story I'm trying to
avoid. Either way, the model drew a coastline I'd mis-mapped — and for now, noticing the
mis-map is still my line of work.
