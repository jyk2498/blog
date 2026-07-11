+++
title = "The Feedback Signal Was the Missing Ground Truth: An HLS Optimization the AI Got Wrong, Then Right"
date = 2026-07-07
draft = false
tags = ["HLS", "hardware-design", "AI-collaboration", "ground-truth", "optimization"]
+++

In my [last post](/posts/1_when-the-ground-truth-isnt-there/) I wrote about a bug
the AI never caught, because the ground truth it needed was never recorded. This
one is almost the mirror image: a case where the AI got it *wrong*, I pushed
back, and it fixed the problem itself — without my ever explaining the trick. The
interesting question isn't why it failed. It's why the first round was wasted at
all, and whose job it was to prevent that.

Here's the setup, stripped down to the pure HLS problem.

## The problem

I have a bitmap of `N` bits (say `N = 128`). Given a variable count `k`, I want to
count how many of the lowest `k` bits are set — a windowed popcount. It sits on a
hot path, so I want it to synthesize into something with a very short initiation
interval; ideally the whole thing resolves in a combinational sweep, not a long
sequential grind.

My baseline did it in two moves:

```cpp
// 1. shift the window into place with a variable-length shift
bitmap = bitmap << (N - k);

// 2. unroll and sum every bit
uint_log2N count = 0;
#pragma hls_unroll yes
for (int i = 0; i < N; i++) {
    count += bit(bitmap, i);
}
```

The variable-length shift synthesizes into a **barrel shifter**. The unrolled sum
synthesizes into an **adder tree** — `N` bits reduced pairwise, so the longest
path is only `log2(N)` deep (7 levels for 128). That tree is exactly what I want:
wide, shallow, fast. The barrel shifter in front of it is the part I'd like to
kill, because it's the expensive piece and it isn't really doing counting work —
it's just aligning the window.

```
            N-bit bitmap
                 |
                 v
        +-----------------+
        |  barrel shifter |
        +-----------------+
                 |  N aligned bits
                 v
     b0   b1   b2   b3  ...  b(N-1)      leaves
       \  /      \  /          \  /
        +          +      ...    +        level 1
         \        /
          \      /
             +                            level log2(N)  ->  count
```

The whole thing is combinational: it settles well inside a single clock, so it
folds neatly into the enclosing pipeline without stretching the schedule.

So the question I handed the AI was simple: *get rid of the barrel shifter.* I
gave it the code and nothing else.

## The AI's first answer, and the trap in it

It came back with this shape:

```cpp
uint_log2N count = 0;
#pragma hls_unroll yes
for (int i = 0; i < N; i++) {
    if (i < k) {
        count += bit(bitmap, i);   // conditional accumulate
    }
}
```

Read casually, this looks great. The barrel shifter is gone — no more shift.
Instead of shifting the window into place, we just accumulate the bits whose index
falls inside the window. Logically correct. Cleaner, even.

But look at what it synthesizes into. The condition is wrapped around the
*accumulate*, which means every iteration's output depends on the previous
iteration's running total:

```
c[0] = (0 < k) ? bit[0]        : 0
c[1] = (1 < k) ? c[0] + bit[1] : c[0]
c[2] = (2 < k) ? c[1] + bit[2] : c[1]
...
```

Each stage is an adder plus a mux, and — this is the killer — each stage feeds the
next:

```
       always adds the running total, then muxes on the window test:

  0 -> [+/mux] -> c0 -> [+/mux] -> c1 -> [+/mux] -> c2 -> ... -> [+/mux] -> count
          ^ b0            ^ b1            ^ b2                      ^ b(N-1)
        (0<k)           (1<k)           (2<k)                    (N-1<k)
  -------------------------------------------------------------------
  one stage:
                bit b(i)
                   |
    c(i-1) ------>(+)------> --+
       |                       +--> [2:1 mux] --> c(i)
       +-----------------------+         ^
              (unchanged total)       (i < k)
```

The `if` made the accumulator itself conditional, so the tool can't reorder or
rebalance anything. Instead of a `log2(N)`-deep adder tree, you get an `N`-deep
serial chain of add-and-mux — for `N = 128`, a dependent tail 128 stages long.

The barrel shifter was gone. The thing I actually cared about — the shallow tree —
was gone too. Net loss.

## What I did, and what fixed it

I didn't spot this by reading the code. On the page, the first answer looks like a
clean win. I spotted it because the code shape smelled slightly off, so I opened
the synthesis report — and the initiation interval of the whole thread that
contained this logic had collapsed. That sent me to the place-and-route view to
see why, and the reason was sitting there: what used to settle inside one clock
was now a 128-deep dependent chain that couldn't close in a single cycle, so the
scheduler could no longer pack the thread the way it had before. One local rewrite
had wrecked the timing of everything around it.

Then the part that genuinely surprised me. I told the AI what I'd seen — the
thread's schedule blowing up, the serial chain behind it — and *without my
explaining the fix*, it produced this:

```cpp
uint_log2N count = 0;
#pragma hls_unroll yes
for (int i = 0; i < N; i++) {
    uint1 keep = (i < k);
    count += bit(bitmap, i) & keep;   // condition on the term, not the accumulate
}
```

The change is tiny and the payoff is everything. The condition moved off the
accumulate and onto the *term*. Instead of "conditionally add this bit," it's now
"always add this bit, but zero it out when it's outside the window." The
accumulate is unconditional again.

That one move restores the data flow. Each term — `bit(bitmap, i) & keep` — now
depends only on its own inputs, not on the running total. So the sum is once again
a sum of `N` independent terms, and addition is associative, so the tool is free
to rebalance it back into a balanced `log2(N)`-deep adder tree. The barrel shifter
stays gone; in its place is a row of cheap one-bit AND gates and a comparator per
term. The logic settles inside one clock again, and the thread's schedule went
back to what the baseline had.

|              | condition on | synthesized path            | fits in one clock? |
| :----------: | :----------: | :-------------------------: | :----------------: |
| first answer | the add<br>(`if`) | serial `N`-deep<br>add+mux chain | NO <br>schedule blew up |
| final answer | the term<br>(`& keep`) | independent terms → tree | YES <br>schedule restored |

Same logic, both times. The only difference is where the condition lives — and
that difference is the whole ballgame in hardware, because it decides whether the
tool sees a dependent chain or a bag of independent terms it can rebalance.

## Whose job the first round was

It would be easy to file this under "AI makes mistakes, keep an eye on it." True —
I caught a plausible-looking answer only because I distrusted the shape enough to
open the report. But that's the shallow lesson. The sharper one is *why the first
round happened at all*. The AI produced the correct answer with no conceptual hint
from me — I never mentioned terms or trees or associativity — so it had the
capability the whole time. It didn't need me to teach it the trick. It needed to
see what its first answer *cost*, and it couldn't, because all I'd given it was
source. Source is enough to judge logical correctness, and by that measure the
first answer was perfect; the failure was in the synthesized result, which lives
in the report and the place-and-route view, not the code. Those artifacts were on
my machine. I just didn't put them in the loop. Had I, I suspect it would have
skipped the serial chain and gone straight to the term-masked tree.

That's the same lesson as [last time](/posts/1_when-the-ground-truth-isnt-there/),
from the other side. There, the ground truth for a bug never existed because I
hadn't recorded it. Here it *did* exist — I just didn't route it into the model's
view. Either way, the model is only as good as the ground truth it can see, and
deciding what that should be keeps landing on the engineer. In software the
feedback signal is nearly free — the model runs the code, reads the error, adjusts.
In hardware the signal I needed, *what does this become once synthesized*, takes a
toolchain run to produce and a human who knows to go looking for it. The model acts
on the signal beautifully once it has it. Getting it the signal is still mine.
