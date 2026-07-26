+++
title = "Making the Same Mistake a Third Time: The Fix Was Never the Problem, the Price Was"
date = 2026-07-26
draft = false
tags = ["workflow", "AI-collaboration", "tooling"]
+++

Anyone who writes code has lost hours to something stupid — the wrong variable, a type
declared one size too small. I did it again recently, and this time it burned tokens as
well as hours.

What's worth writing down isn't the mistake. It's that I finally closed the loop on it,
and that the fix had been sitting in front of me, fully understood, for years.

## Where the width lives twice

I write HLS in SystemC: write the SystemC, run it through high-level synthesis to generate
RTL, then build the FPGA bitstream from that RTL.

Trouble starts when a module's interface changes — a field added to a struct crossing a
port, say, so the port gets wider. Synthesis handles this perfectly; the generated RTL
follows the source. But when two modules are connected through a FIFO, that FIFO isn't
part of the generated RTL. It lives in the design assembled for the bitstream, and its
width follows nothing. I have to change it by hand.

And I forget — not at random, but exactly when it matters most: after a large change
touching many modules, when there is the most to remember and the least attention left to
remember it with. The dropped field doesn't announce itself either. What I see is a wrong
value downstream, which looks like a logic bug, which is where I go and where I stay for
hours. Then I find the real cause and feel the particular despair of an afternoon lost to
my own carelessness.

## Third time

This was the third time.

I want to be precise here, because it would be easy to tell this as a story about learning
something. I hadn't learned anything. After the first two I knew exactly what would have
prevented it: a check comparing a module's interface against the width of the FIFO it
feeds. I had thought of it. I never built it — building it meant parsing two artifacts and
keeping that parser alive as both formats drifted underneath it, which didn't pay against
a mistake I make once every year or two. That was the right call. So the rule sat where
every engineer keeps a large collection: known, real, and permanently below the threshold
of being worth fixing.

## What changed was the price

I run synthesis through a Claude skill now, so this time the fix was one paragraph: when
synthesizing, if a module's interface has changed, check whether it feeds a FIFO and
change that width, or say so. No parser, nothing to keep alive as the formats drift.
Nothing I know changed — only the cost of encoding a rule, which fell far enough that a
fix I had correctly priced as not-worth-building became worth building. Which makes the
real question not this FIFO but the size of that category: every engineer carries a list
of known-but-not-worth-it fixes, all of them priced under an old cost structure.
