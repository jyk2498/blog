+++
title = "When the Ground Truth Isn't There: A Hardware Bug the AI Couldn't Catch"
date = 2026-07-07
draft = false
tags = ["rdma", "hardware-debugging", "ai-collaboration", "ground-truth", "skepticism"]
+++

I spend most of my days building an RDMA datapath in hardware, and most of that
work now happens with an AI coding agent sitting next to me. It writes a lot of
the code. It reads specs faster than I do. On a good day it feels like pairing
with a senior engineer who never gets tired.

And then there are the bugs it can't touch.

This is a story about one of them. Not to dunk on the model — it's genuinely
good — but because *why* it failed turned out to be more interesting than the
bug itself. The failure had a shape, and that shape has been showing up over and
over in my work. It's the clearest example I have of where the human still has
to stand.

## The setup

Two RDMA endpoints, a requester and a responder, doing an RDMA READ. The
requester asks for a range of data; the responder streams back READ RESPONSE
packets, each carrying a Packet Sequence Number (PSN). Standard stuff.

Here's the part I'd glossed over. The InfiniBand Architecture Specification
allows the responder to send something called an **unsolicited acknowledge**:

> The responder may send credits to the requester asynchronously by using an
> Unsolicited acknowledge packet. An unsolicited acknowledge packet is created by
> re-sending the most recently sent acknowledge packet.
>
> From a PSN perspective, an unsolicited acknowledge message appears to the
> requester like a duplicate of the most recent positive acknowledge message.
> However, it always has an opcode of Acknowledge even if the most recent positive
> acknowledge was an RDMA READ Response or Atomic Response.

Read that carefully. The responder is allowed to, at essentially any moment,
re-send its most recent acknowledgment purely to hand back flow-control credits
and refresh the Message Sequence Number (MSN). And crucially: even if the most
recent "acknowledgment" was actually a READ RESPONSE, this credit-only packet
carries the **same PSN** as that READ RESPONSE, just with an Acknowledge opcode.

So during a READ, you can legally see an ACK whose PSN sits *inside the range of
the READ responses currently in flight*. It looks like a duplicate. It isn't an
error. The spec is explicit that the requester must be ready for it at any time:

> Since the responder's receive queue may generate an unsolicited acknowledge
> message at any time, the requester shall be prepared to receive an unsolicited
> acknowledge message from the responder at any time...

One more detail that matters later: this trick only works for ACKs, never for
NAKs. The spec prohibits duplicate NAKs, so a negative acknowledge that arrives
mid-range is always a *real* NAK and must be handled, never dropped.

![Normal case: the unsolicited ACK arrives after the READ has already completed, and is safely absorbed](/img/unsolicited-ack-normal.svg)

In the clean case above, the READ has already completed by the time the
unsolicited ACK shows up. The requester's expected-PSN has already moved past the
whole range, so the ACK reads as a harmless duplicate: recover the credit, refresh
the MSN, absorb it. My design handled this exactly right.

## The bug

The unsolicited-ACK scenario on its own was never the problem. What my design
*didn't* handle was that same scenario with packet loss layered on top of it —
an unsolicited ACK arriving while responses were still being dropped.

Here's the sequence that broke things. Suppose the requester asks for PSNs 100
through 103:

1. Requester → Responder: READ request for range 100–103.
2. Responder → Requester: READ RESPONSEs for 100–103 — but 102 and 103 get
   dropped on the wire. The requester has only committed through 101, so its
   expected-PSN is now 102.
3. Responder → Requester: an **unsolicited ACK** carrying PSN 103.

![Bug case: responses are dropped, and an unsolicited ACK carrying a PSN ahead of expected-PSN and inside the READ range is misread as an implied NAK](/img/unsolicited-ack-bug.svg)

Now look at it from inside the requester's completion logic. It's expecting PSN
102 next. Instead a packet shows up announcing PSN 103 — a PSN *ahead* of what
it expected, and still inside the outstanding READ range. My logic misread this
as an "implied NAK," triggered a spurious retransmission, and — worse — advanced
the completion count off the packet's MSN field when it shouldn't have. The queue
pair wedged.

The fix, once I understood it, was almost embarrassingly small: teach the
requester to recognize this credit-only packet for what it is, rather than
mistaking it for a gap in the data that needed recovering.

The necessary conditions for the bug are worth stating precisely, because this is
the kind of thing that only fires in the field: an unsolicited ACK whose PSN is
(a) inside the READ range and (b) ahead of the current expected-PSN — which is
exactly the window that opens when responses get dropped. No drops, no bug. It
needed the unsolicited ACK *and* packet loss *and* a particular internal state to
line up all at once.

## Why the AI couldn't catch it

This is the part I actually want to talk about.

I threw a lot at this. Not just a quick prompt — I used the strongest model
available to me at the time, at its highest reasoning-effort setting, and let it
run across two full extended sessions. It burned through the equivalent of two
hundred percent of a session's token budget and then most of another, and it
still didn't find this. I didn't send it in empty-handed either — I gave it a
packet dump, waveforms exported from the logic analyzer as CSV, and a clear
description of the symptom. And this is a model that, handed a codebase and a
spec and asked to point out where a bug is likely hiding, runs circles around me.

So why did it whiff on this one? It clearly wasn't a matter of raw capability or
compute. Something else was missing.

I've come to think the answer is a single idea: **there was no ground truth for
it to stand on.** And once I saw it that way, the model's strengths and blind
spots stopped looking random.

The model is superb wherever the ground truth is externalized — written down in a
codebase, a spec, a log, a stack trace. Hard debugging against existing code,
corner-case hunting, checking an implementation against a written spec: this is
its home turf, and it's better than me there. But this bug lived in two places
where the ground truth simply wasn't available to it.

**First, the spec context wasn't in the room.** The unsolicited-ACK behavior is
public — it's right there in the IB spec — but I never handed the model the spec,
and I never told it *which chapter mattered*. I couldn't have, because I didn't
know myself. To point it at the flow-control-and-acknowledgment section, I would
have needed a conceptual grasp of why that mechanism exists in the first place.
The model can read a spec brilliantly. It can't guess which twenty pages of a
thousand-page document are the relevant ones when nobody tells it the symptom
lives there.

**Second, in hardware, deciding what even counts as ground truth is a human
job — and I got it wrong.** Look again at what I handed the model. The ILA
capture didn't contain the internal register that mattered — the requester's
expected-PSN, the state my completion logic was actually branching on, was never
routed out to the analyzer, so it appears in no artifact at all. The window I'd
captured was aimed at a symptom rather than the actual race. And the packet dump
was all ordinary, healthy traffic; the one situation that mattered — an
unsolicited ACK landing while responses were being dropped — wasn't in it. Three
artifacts, not one of them holding the moment the bug lived in. The model wasn't
failing to reason over the evidence; there was no evidence. I'd collected the
wrong things.

And in hardware, that's the human's problem to fix, not the model's. In software
it could have closed the gap itself — add a log line, re-run, narrow in, almost
for free. On real hardware that loop barely exists: you see a fixed set of
signals you *chose in advance* to route out, in a narrow window around a trigger
you *chose in advance* to arm, and each capture is slow and costly to pull, not
something you casually redo twenty times. (Simulation gives you total visibility
instead, but running the real workload takes days and dumps hundreds of
gigabytes — the opposite bad trade.) Either way the same question lands back on a
person, before any model sees anything: of everything I *could* record, what do I
actually make visible? No amount of intelligence reconstructs a signal that was
never recorded.

None of this is really about the software/hardware label — plenty of software
bugs are their own kind of hell to reproduce. The axis that actually matters is
whether a fresh observation is cheap and repeatable enough for the model to keep
producing its own ground truth, or whether each one is fixed and expensive and a
person has to decide, up front, what will even be visible. Software usually sits
at one end of that axis; real hardware usually sits at the other.

![The real axis is observation cost: when a fresh observation is cheap the AI closes the loop itself; when it is fixed and costly a human must decide what to expose next](/img/observation-cost-loop.svg)

## What actually stayed human

If I line those failures up, none of them is "the model isn't smart enough." I
proved that much by throwing my most capable tools at it. Every one of them is a
place where the ground truth was missing, un-externalized, or un-recordable — and
closing that gap was a human job:

- **Knowing the spec conceptually, not line by line.** The model out-reads me on
  detail. But knowing *that a mechanism like unsolicited ACK exists and where to
  look for it* is what lets me aim the model at the right pages. Conceptual
  understanding is how you generate the context the model needs; it isn't
  something you can offload to the thing that needs it.

- **Deciding what becomes ground truth in the first place.** In hardware, the
  most important debugging decision happens before any tool runs: which trigger
  to arm, which internal signals to expose, what to record. That framing act — "I
  need to make *this* observable so *someone* can reason about it" — is upstream
  of any analysis the model can do, and it's where hardware debugging demands far
  more human involvement than software.

I don't take this to mean the AI is bad at hardware. It means the boundary
between what it can and can't do isn't about difficulty — it's about whether the
truth of the situation has been written down somewhere it can reach. The model
lives on externalized ground truth. My job, increasingly, is everything that
happens *before* the ground truth exists: understanding the system well enough to
know what to look for, and doing the physical and conceptual work of making the
invisible observable.

The fix was three lines. Getting to the point where three lines was obviously the
answer was the whole job. That gap is where I still live.
