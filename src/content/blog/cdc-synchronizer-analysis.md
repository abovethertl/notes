---
title: "CDC Synchronizer Analysis: Multiple Requesters Through a Single-Bit Synchronizer"
description: "Reference note on the four OR-before-sync configurations, runt-pulse physics, MTBF regime analysis, and correct-by-construction vs safe-under-assumptions. Companion to Above the RTL Post 5."
pubDate: 2026-04-23
---
*Reference note accompanying the Above the RTL post on frontier-model behavior across CDC review questions. The post covers what two LLM versions did with this material; this note covers the material itself.*

## Setup

A level-based request signal is carried from a source domain to a destination domain through a standard 2-flop synchronizer. The request line is held high for multiple destination clock cycles so the receiver can capture it reliably. Multiple *requesters* share the line — any one of them being high must be reflected as `request = 1` at the destination. The OR that combines the requesters is placed *before* the synchronizer.

This note walks through four configurations of that OR, explains why one of them is broken at the methodology level, why another is always broken at the functional level, and why the remaining two are safe in specific frequency regimes but still wrong for new design. Along the way it digs into the deeper questions those cases raise — why brief combinational glitches are tolerated by a level protocol, what we can and cannot prove analytically about runt pulses and metastability resolution, what "marginal MTBF" actually means, and why the rule is absolute for new code even when the physics of an individual case is defensible.

---

## Part 1 — Four configurations of the OR

All four configurations are CDC rule violations: each has combinational logic or signal merging in front of the synchronizer. They differ in *what kind* of failure that violation produces, which matters for understanding the physics and for prioritizing what to fix first in shipped silicon — but it does not matter for the methodology question on new design, where the rule applies uniformly.

### (a) Same clock domain, flop outputs ORed → synchronizer — **rule violation; failure mode contained under level protocol with sufficient frequency headroom**

```
req_flop[0] ─┐
req_flop[1] ─┤─OR─► [2FF sync] ─► dst domain
req_flop[N] ─┘
```

All OR inputs are register outputs in the same source clock domain. The strict rule flags this as a violation — combinational logic (the OR gate) sits directly between a flop and the synchronizer. The specific failure mode the rule protects against here is a brief combinational glitch when two requesters transition in opposite directions on the same source edge with unequal path delays (e.g., req₀ 1→0 and req₁ 0→1, producing a momentary dip on the OR output). Under a level protocol with hold-high across multiple destination cycles, a single dropped sample is recovered on the next destination edge, and at low destination frequencies the MTBF exponent cushion absorbs the runt-induced denominator degradation (see Parts 2 and 4). This is why (a) configurations have historically shipped at 100-MHz-class destinations without visible field failures. It does not make the violation correct. Part 5 addresses why the rule holds for new design anyway, and why the cushion evaporates at modern frequencies.

### (b) Same clock domain, state-machine decode → synchronizer — **rule violation; silent phantom requests**

```
state_reg ─► [comb decode: (state == XYZ)] ─► [2FF sync]
```

The decode is combinational logic on multiple state bits. Path-delay skew between those bits causes the decode to glitch through intermediate encodings during state transitions (binary `011 → 100` can momentarily look like `001`, `101`, `111`). Any transient that matches `XYZ` produces a spurious 1 on `request` — **including when the state machine never actually reaches `XYZ`**.

The failure mode here is categorically different from the other configurations. In (a) and (c), the OR of stable flop outputs can only go high when some source flop is genuinely 1; every high at the synchronizer input corresponds to a real request somewhere. Here, the decode can assert `request` from a state sequence that was never supposed to produce one, on every transition whose intermediate encoding happens to match — systematically, not rarely — and the synchronizer faithfully delivers the lie downstream. A phantom request that the source FSM never intended.

**A synchronizer is a metastability filter, not a glitch filter.** Narrow glitches that land inside the destination aperture can be captured as legitimate pulses, and sub-cycle glitches can drive the first flop metastable into either resolved value — including a false 1.

**Fix:** register the decode in the source domain before the synchronizer, or use a one-hot state encoding.

```verilog
always @(posedge src_clk) req_r <= (state == XYZ);
// feed req_r (not the raw decode) to the synchronizer
```

### (c) Different clock domains, flops ORed → synchronizer — **rule violation; MTBF degradation, no phantom requests**

Three failure-mode issues versus (a):

1. **Asynchronous-OR glitching.** When req_A and req_B transition on unrelated clock edges that happen to fall within a few hundred ps of each other, the OR output produces glitches whose width is set by the phase relationship of the source clocks — effectively random, and unlike (a), not confined to the edges of a single clock.
2. **Handoff runt at the synchronizer input.** When A's 1 ends just before B's 1 begins, the union signal has a brief gap. That gap is a property of the requests themselves and is present in any implementation — it is not something the correct structure eliminates. What (c) adds is that the gap reaches the synchronizer input as a runt pulse of uncontrolled sub-cycle width, set by the phase relationship between the source clocks and by gate propagation delays. In the correct structure (each requester synchronized into the destination first, OR performed in the destination), the same handoff produces an integer-cycle dip on clean synchronous signals: the destination may see a one-cycle zero before 1 returns, recovered on the next edge under a level protocol, with no sampling flop ever presented with a runt. What (c) gets wrong is not the dip; it is where and how the dip is sampled.
3. **MTBF model violated at the input.** The formula assumes a well-behaved synchronizer input: a level signal from a single source clock, transitioning at most once per source cycle. The OR of independently-clocked signals isn't that — `f_data` sums multiple source rates, and `T_w` effectively widens because transitions arrive at arbitrary phase relative to the destination clock. Both hit MTBF linearly.

The OR of stable flop outputs from any domain cannot produce a 1 from steady-state zero, so every high on the synchronizer input corresponds to some source flop that really was 1. There is no "phantom request from a state that was never asserted" failure here — that is unique to (b) and (d). The failure modes in (c) are narrow dips (missed samples, recoverable on the next destination edge under a level protocol, same mechanism as the glitch in (a)) and MTBF degradation (budgetable and analyzable per Part 4).

That is a distinct failure mode from (b), but it is not a weaker violation of the rule. (c) is still combinational merging in front of a synchronizer, and the MTBF degradation is real. The clean design synchronizes each request into the destination domain first, then ORs in the destination domain:

```verilog
req_A_dst <= sync2(req_A); // in destination clock
req_B_dst <= sync2(req_B);
request = req_A_dst | req_B_dst;
```

### (d) Different domains with combinational logic before the OR — **rule violation; (b) and (c) failure modes stacked**

This is (b) and (c) stacked. The phantom-request failure mode from (b) is present in each source domain's combinational output, and the async-OR merging of (c) is layered on top. Fix is the union of the fixes: register each per-domain combinational output in its own source clock, synchronize each registered signal into the destination domain, OR in the destination.

---

## Part 2 — Why the combinational glitch in (a) is absorbed under a level protocol

*A note on framing: this section explains the physics of why the (a) glitch does not reach the destination as a corrupted value, given specific conditions. It is not an argument that (a) is a methodologically acceptable design. The strict CDC rule flags (a) as a violation and is correct to do so; Part 5 addresses why the rule holds for new design even though the physics of an individual (a) case can be defended.*

The glitch in question: on a source edge, req_i goes 1→0 while req_j goes 0→1 with unequal path delays, producing a brief 1→0→1 transient on the OR output.

"Safe" rests on three conditions:

### 1. The steady state before and after is identical

Before the source edge: OR = 1 (at least one requester was high). After: OR = 1 (at least one is high). The glitch is a pure transient — a dip in the middle of a continuous 1. It doesn't change the level the destination is supposed to see; it just momentarily lies about it.

### 2. The dip is narrow compared to the hold time

Glitch width is bounded by the path-delay skew at the OR gate's inputs — typically hundreds of picoseconds in a sane layout. The hold-high guarantee is typically multiple destination cycles. Worst case: one destination edge happens to fall in the dip and samples 0; the next edge (one T_dst later) samples 1 again. One cycle of latency, not a lost request.

### 3. The destination protocol is level-sensitive

"The line stays high while any requester asks" is a level protocol. The destination reads *is it high now?*, not *was there a new edge?*. Missing a 1 for one cycle is fully recovered the next cycle.

### A stronger guarantee: inputs that cannot produce opposing transitions

The three conditions above describe a regime in which the runt is generated but absorbed. There is a stronger regime worth calling out: when the requesters are guaranteed by the surrounding system never to have opposing transitions on the same source edge, no runt is generated in the first place, and the (a) failure mode cannot trigger. The cases that actually qualify:

- **Requesters from independent logic with temporally separated activity.** The N inputs to the OR come from unrelated control paths whose operating conditions do not produce simultaneous transitions — e.g., a DMA completion, a timer interrupt, and a user-initiated event, each issued by its own block and quiet between issuances. This is not a structural guarantee from the RTL; it is an operating-conditions assumption that has to be stated explicitly and reviewed when the block is integrated. In practice this is the configuration most (a) designs rely on, because requesters ORed at this level are usually drawn from physically and logically separated sources to begin with.
- **Handshake protocols with gap cycles.** The current owner is required to hold its output low for at least one full source cycle before a new owner asserts; opposing-transition-on-same-edge is eliminated by construction.
- **Single active requester.** The degenerate case: N = 1, no OR gate, nothing to glitch.

What is *not* a safe case, and is worth naming explicitly because it is the intuitive-sounding mistake: a one-hot state machine whose state bits are ORed together. A one-hot FSM has exactly one bit high at any time, which sounds like mutual exclusion — but every state transition produces exactly one bit going 1→0 and another going 0→1 on the same source edge. That is the canonical opposing-transition scenario and a worst case for runt generation at the OR, not a safe case. If the requesters share a source FSM, the correct fix is trivially within reach (add a gap state between the two active states, or simply register the OR output in the source clock before the synchronizer), and the one-hot-without-mitigation configuration should not be what the design is relying on.

Under the protocols that do qualify, the OR output transitions only at the rate and in the direction of the genuine union change. The rule violation (combinational OR in front of a synchronizer) is methodologically still present, and the reuse-and-regime arguments in Part 5 still apply if the block is later instantiated in a context where the temporal-separation assumption is relaxed. Within the original operating conditions, the specific silicon-level failure mode the rule is there to prevent does not occur — and alongside the low-frequency exponent cushion from Part 4, this structural explanation covers most of the (a) configurations that have historically shipped without visible field failures.

### Where "safe" stops being safe

- **Edge-sensitive destination logic.** If someone writes `req_pulse = req_sync & ~req_sync_d1` to count new requests, the glitch-induced 1→0→1 creates a phantom rising edge. This is the classic reason CDC guidance insists on level handshakes, not pulse counts, across a synchronizer.
- **Pathological skew.** The argument assumes glitch width ≪ hold duration. If path delays from requesters to the OR are wildly unbalanced, the dip can widen enough to be missed over multiple destination cycles. This is a timing-closure issue, not a protocol issue.
- **Non-flop inputs (cases b and d).** If any OR input is a combinational decode, glitches are bigger, wider, and can occur at any time, not just on source edges. Now you are not safe.

**Belt-and-suspenders mitigation:** add a source-domain flop after the OR before feeding the synchronizer. Converts any runt into a clean full-swing transition synchronous to the source clock.

```verilog
always @(posedge src_clk) req_src <= |req_vec;
// feed req_src to the synchronizer
```

Note: this adds one source-clock cycle of latency on the request path. For new design the cost is a single flop; for late-cycle RTL the change can invalidate downstream timing assumptions and STA closure already completed against the current path, which is why Part 4's closing caveat on restructuring existing RTL matters here.

---

## Part 3 — Does a runt pulse cause deep metastability?

A reasonable worry: if the glitch is a narrow runt that rises only partway and falls back, doesn't that drive the synchronizer's first flop into a particularly deep metastable state?

The answer splits into three separate questions, and it is important to be honest about which of them we can answer from first principles and which require simulation.

### 1. Probability of entering metastability — yes, a runt is clearly worse

A clean full-swing transition spends only its rise/fall time near Vdd/2. A runt that rises partway and falls back spends *most* of its duration near mid-rail. If the flop's aperture closes while the input is in that uncertain zone, metastability is triggered. The fraction of input time near balance is higher for a runt, so the per-event probability of metastability is higher. This is straightforward to argue from the geometry of the input waveform and is not in dispute.

### 2. Intrinsic τ of the regenerative loop — unchanged by input waveform

The flop's intrinsic metastable resolution follows `V(t) = V(0) · exp(-t/τ)`, where τ is the regenerative time constant of the cross-coupled storage node — a small-signal property of the loop's gain, node capacitance, and supply voltage. τ is measured and characterized by the cell vendor under clean-input conditions, and in that small-signal picture it is a property of the flop, not of the input waveform. By this characterization, the runt does not change τ.

### 3. Effective resolution time under runt input — a concern, not a proof

The subtlety, and the part worth being honest about: the characterization in (2) assumes the input transistor has decoupled from the storage node by the time regenerative resolution begins. A clean full-swing transition that happens to land near Vdd/2 delivers its charge and gets out of the way; the regenerative loop then evolves autonomously with time constant τ from whatever initial condition that charge left behind.

A runt pulse is not guaranteed to behave that way. During the runt's rise-partway-and-fall-back, and potentially past it, the input transistor remains partially conducting, continuing to inject (or draw) charge at the storage node — precisely during the period when the regenerative loop is trying to pull away from balance. Whether that extended input influence measurably changes the *effective* resolution time seen at the Q output is a question about the interaction between input impedance, node capacitance, and the regenerative dynamics during the aperture and early resolution window. The small-signal linear analysis that gives us τ does not fully cover it.

The honest position: the intrinsic τ is unchanged; the effective resolution time under runt input may be longer, and without a Monte Carlo transient simulation on the actual cell under representative runt waveforms, I cannot prove it isn't. Treat it as a concern rather than a proof. That concern does not flip the direction of the methodology conclusion — the rule holds either way — but it tightens the methodology argument: we are uncertain about the magnitude of the runt's effect on resolution, which is all the more reason not to be operating in a regime where the formula's clean-input assumption is violated.

### What we *can* prove directly from the MTBF formula

Leaving resolution-time questions aside, the linear-denominator degradation is provable without simulation:

- **T_w widens.** The effective metastability aperture for a runt input is the portion of the runt's duration where the input sits near the balance threshold — which is most of the runt. For MTBF accounting this acts as a much wider aperture than a clean edge crossing.
- **f_data rises.** A runt is a rise *and* a fall. Each is a metastability opportunity.

Both are linear penalties in the MTBF denominator. They compound with the uncertain effective-resolution-time effect from (3) to push MTBF below the clean-input prediction by an amount that is bounded below by these linear factors and upper-bounded by simulation results nobody runs on waived blocks.

### Why the level protocol still recovers — even when metastability happens

Two structural properties save the level-protocol case even when the first flop does go metastable:

1. **Either resolution is recoverable.** The first flop resolves to 0 or 1 before the second samples. If it resolves to 1, the destination captures correctly. If it resolves to 0, the destination misses this cycle — but the OR's steady-state value is 1, the signal is held high for multiple cycles, and the next destination edge captures it. Indistinguishable from a non-metastable miss.
2. **No metastability propagates past the second flop.** The whole point of 2FF is that the second flop sees a signal that has had ≈ T_dst to resolve. Whatever resolution time the first flop needed, the second flop sees a settled value.

These structural properties are what let legacy designs at low frequencies survive rule violations — not the rule being wrong, but the structure being forgiving in a specific regime. More on that in Part 4.

### Where this whole discussion stops applying

- **Edge-counted destination logic** (see Part 2): the recovery argument above evaporates the moment the destination interprets rising edges rather than levels.
- **Sub-aperture runts**: if the runt is so narrow that it rises *and* falls entirely inside the flop's aperture, the flop sees a transient mid-rail voltage without a clean transition at all. The resolved value is essentially random. Recoverable under the level protocol, but not something to rely on.
- **High event rates.** The "safe" argument assumed simultaneous one-up-one-down transitions are rare. If requesters toggle often and frequently produce runts, the MTBF denominator inflates linearly with the event rate, and a marginal-MTBF 2FF can start missing its budget.

---

## Part 4 — What "marginal MTBF" actually means

Standard 2FF synchronizer MTBF:

```
           exp(T_r / τ)
MTBF = ─────────────────────
        f_clk · f_data · T_w
```

- `T_dst` — destination clock period (1 / f_clk)
- `T_setup2` — setup time of the second flop in the 2FF synchronizer
- `T_r` — time the first flop has to resolve before the second samples it ≈ `T_dst − T_setup2`
- `τ` — regenerative time constant of the flop (physical property of the cell)
- `T_w` — metastability window (aperture width)
- `f_clk` — destination clock rate
- `f_data` — rate of transitions on the synchronizer input

Everything in the denominator is linear. The numerator is exponential in `T_r/τ`. That exponent is where designs live or die: drop `T_r/τ` from 30 to 15 and MTBF drops by `e¹⁵ ≈ 3×10⁶`.

### When the exponent gets small

"Marginal" means `T_r/τ` has shrunk to the point where the exponential no longer swamps the linear denominator. The knob that shrinks it at design time is the destination clock frequency, because `T_r ≈ T_dst − T_setup2`.

Rough illustration with τ ≈ 40 ps, T_w ≈ 20 ps, f_data = 10 MHz:

| f_dst   | T_dst  | T_r / τ | exp(T_r/τ) | MTBF (rough)       |
|---------|--------|---------|------------|--------------------|
| 100 MHz | 10 ns  | ~240    | ~10¹⁰⁴     | astronomical       |
| 500 MHz | 2 ns   | ~47     | ~10²⁰      | ~10⁸ years         |
| 1 GHz   | 1 ns   | ~22     | ~4×10⁹     | ~hours             |
| 2 GHz   | 500 ps | ~10     | ~22 000    | ~ms                |

At 100 MHz, the exponent is so large the linear terms are irrelevant and you can be sloppy. At 1 GHz+, the exponent drops into the range where every factor in the linear denominator starts to actually matter.

### How runt pulses hit the formula

Assuming τ is unchanged by input waveform — which is the small-signal-characterization story from Part 3, and a working assumption rather than something the linear analysis can fully guarantee for runt inputs — runts hit the linear factors in the denominator:

1. **Effective `f_data` scales combinatorially with fan-in, not linearly.** With a single registered signal feeding a synchronizer, `f_data` is that signal's toggle rate — at most one transition event per source cycle. With an OR of N flops feeding the synchronizer directly (configuration a), each source cycle is an opportunity for any of the up-to-`N(N−1)/2` opposing-transition pairs to produce a runt, and the effective event rate at the synchronizer input is set by the pairwise activity across the OR inputs rather than by any individual signal's toggle rate. A wide fan-in in a busy system can generate runt events at a rate orders of magnitude above what a single registered input would produce. (Each runt also contains both a rise and a fall, each a separate metastability opportunity, but that factor of two is small compared to the combinatorial scaling from fan-in.) MTBF scales as 1/f_data.
2. **`T_w` effectively widens per runt.** A runt that dwells near Vdd/2 for 100 ps acts, for MTBF accounting, like a transition with an aperture as wide as that dwell — a per-runt property, independent of how many runts there are. MTBF scales as 1/T_w.

Both are linear. When the exponent is 10¹⁰⁰, a 10× or 100× linear hit is noise. When it's 10⁹ at hours-MTBF, a 100× linear hit takes you to minutes-MTBF — a real bug.

### Baseline: use SYNC-library cells when available

Modern standard-cell libraries ship dedicated synchronizer-variant flops — cells with intentionally small τ. These should be the default for every CDC synchronizer, not a mitigation reserved for marginal cases. There is no reason to send a CDC path through a general-purpose flop when the library offers a SYNC variant.

Many libraries further package a full 2FF synchronizer as a single characterized cell: the two flops are placed directly adjacent inside the cell, and no routing sits between Q of the first flop and D of the second. That construction pushes T_r to almost the full destination clock period (only the second flop's setup time eats into it — no buffer delay, no interconnect parasitic on the Q-to-D path), which maximizes the exponent `T_r / τ` the silicon makes available. Whenever the library provides it, the paired-cell synchronizer is the right primitive to reach for.

### Mitigations when the exponent is still marginal

When the SYNC-cell baseline is in place and the exponent is still in the range where linear factors matter (roughly destination clock rates of 1 GHz and above, per the table above), additional mitigations become relevant:

1. **Kill the runt at the source** — a source-domain flop between the OR and the synchronizer. Turns the synchronizer input into the idealized clean full-swing signal the MTBF formula assumes, restoring the linear factors (`f_data`, `T_w`) to their nominal values. Note: adds one source-clock cycle of latency on the request path.
2. **3FF synchronizer** — gain another full T_dst of resolution time. The effective exponent becomes ≈ `2·T_r/τ`, so MTBF roughly squares. Turns "hours" into "years." Standard move at ≥ 1 GHz. Also adds one destination cycle of latency on the captured signal.
3. **Slow the destination sample** — clock the synchronizer from a slower derived clock and transition to full rate after capture. Not always architecturally available.

### Caveat: late-cycle RTL changes carry their own risk

Each of the restructuring mitigations above — adding a source-side flop, moving from 2FF to 3FF, re-timing the OR — changes the latency of the request path. For new design that is a cost to budget once. For late-cycle RTL it is a change that can invalidate downstream timing assumptions, functional-verification coverage built against the current arrival time of the signal, and STA closure already completed on the existing path. The further a design is into its tapeout cycle, the more expensive a late mitigation becomes relative to the empirical safety the existing structure already demonstrates.

Near-tapeout RTL deserves the same posture Part 5 describes for shipped silicon: characterize the regime the design is actually operating in, document the assumptions the existing structure relies on, write assertions that would fire if any of those assumptions were violated by a future reuse, and leave the RTL alone. Introducing an extra flop at that stage to satisfy a rule the silicon will empirically handle at the current destination frequency is trading a known-good timing path for an unquantified regression risk on a schedule that no longer has room to re-verify downstream behavior. The absolutism of the rule is for the design phase where re-timing is cheap. Once the block's timing is committed, the calculus is different — for both shipped silicon and late-cycle RTL.

---

## Part 5 — The rule is absolute; why shipped silicon sometimes appears to disagree

Parts 1 through 4 establish that (b) and (d) are always broken, and that (a) and (c) sit in a category where physics makes them tolerable under specific conditions — level-sensitive destination, hold-high multiple destination cycles, MTBF exponent dominant over linear-denominator degradation. The strict CDC rule flags all four as violations and does not carve exceptions. That rule is correct, and it is the position any new design should be reviewed against. What needs an honest accounting is why rule-violating (a) and (c) configurations appear, repeatedly, in working shipped silicon — and why that empirical observation does not rehabilitate the violation.

### The rule and the apparent counter-evidence

The strict rule: no combinational logic before a synchronizer, no merging of signals before a synchronizer. Full stop.

The counter-evidence engineers raise, correctly: they have shipped (a) and (c) configurations, repeatedly, across many designs, without field failures. The designs went through CDC review with waivers, through STA, through post-silicon validation, and the field failure rate was zero or indistinguishable from zero across the deployed fleet.

Both of these statements are true. The question is whether the second one is an argument against the first.

### The frequency-regime explanation

It is not. The MTBF formula tells us why. At a destination frequency of 100 MHz, with τ on the order of 40 ps and T_setup small, `T_r / τ ≈ 240`, and the exponential term `exp(T_r/τ)` is a number too large to write without scientific notation. The linear denominator — `f_clk · f_data · T_w` — is dwarfed by orders of magnitude. A rule violation that degrades the linear denominator by a factor of 100 (wider `T_w`, higher `f_data` from runts, async-OR effects in the (c) case) moves MTBF from `10¹⁰⁴ years` to `10¹⁰² years`. The violation is real; the consequence is numerically invisible.

Design that fleet. Ship it. Field returns come in. Investigate. Find nothing attributable to CDC. Conclude: "we do this all the time and it works."

The conclusion is accurate for that frequency regime. It is not a methodology result. It is a side-effect of the MTBF formula's exponential term dominating everything else at frequencies where `T_dst` is large compared to τ. The rule was not disproved; it was hidden by a safety margin so large that the violation's cost rounded to zero.

### Why modern frequencies remove the cushion

At 1 GHz with the same τ and a typical T_setup budget, `T_r / τ` drops to roughly 22. `exp(T_r/τ)` is now around `4 × 10⁹`. MTBF for a clean-input synchronizer is hours to days, not geological time. A 100× linear-denominator hit from a rule violation moves MTBF to minutes. The same (a) or (c) block that shipped cleanly at 100 MHz becomes a field-failure generator at GHz rates.

The rule did not become more important. The regime became less forgiving. What was numerically invisible at 100 MHz is the failure mode at 1 GHz.

And here is the trap: IP blocks do not come stamped with the frequency regime they are safe in. The (a) block an engineer designed, verified, and shipped at 100 MHz in 2015 — with an informal mental caveat of "works because hold-high is long enough and MTBF margin is huge" — gets reused in a 2026 SoC at 1.2 GHz, possibly by a different team, possibly in a different physical design, possibly without the original designer being consulted. The block has not changed. The regime has. The silent cushion the block was relying on is now gone, and the rule violation that was always there now has failure-mode consequences.

This is why the rule holds absolutely for new design. Not because the physics of (a) and (c) cannot be defended — under the right conditions, Part 2 shows they can — but because the design-time check cannot anticipate the range of regimes the block will be asked to operate in over its lifetime. Correct-by-construction blocks are regime-independent. Safe-under-assumptions blocks are regime-specific, and the engineer verifying the assumptions is often not the engineer who instantiated the block.

### The reuse trap

Every reuse of an (a) or (c) block requires re-verifying the regime assumptions the original design made. The questions that have to be answered, every time:

- Is the destination still level-sensitive, or has a downstream change introduced edge-counting on `request`?
- Is the hold-high duration still multiple destination cycles at the new clock frequencies?
- Has the destination clock rate moved into a regime where the exponent cushion no longer dominates?
- Have the switching rates stayed within the MTBF budget the original analysis assumed?
- Has the fan-in on the OR grown such that the runt-width bound no longer holds?
- Has anyone touched the output-side consumer in a way that invalidates the level-protocol assumption?

Each of these is a review. Each takes engineering hours. Each is a place where the assumption can quietly fail — most commonly when the block is dropped into a new SoC, a new clock tree, or a new team and nobody remembers what the original analysis assumed.

Correct-by-construction is free on reuse. A crossing that satisfies the rule without exception requires no condition-checking, in any context, ever. The analysis is closed at design time, by construction. The block travels with its own proof. Break-even against safe-under-assumptions is one or two reuses — and reusable CDC primitives typically reuse more than that.

For new code, this is not an aesthetic preference. It is a cost argument, and it is independent of the physics defense of (a) and (c). Even if the level-protocol argument for (a) is fully valid at the frequency the block is being designed for, the block's reuse cost over its lifetime is higher than the cost of making it correct-by-construction today. The rule wins on economics before any argument about review discipline or sign-off rigor is made.

### What to do with shipped silicon — and with late-cycle RTL

The above applies to new design. For a block that has already shipped in silicon and worked in the field, redesign trades a known-good implementation for regression risk in exchange for an analytical gain the field has already closed out. The rule's absolutism does not extend to "tear out working silicon to satisfy a rule whose failure mode the silicon has empirically demonstrated it is not experiencing."

The right move for existing (a) and (c) blocks:

1. Leave the RTL alone.
2. Document the assumptions explicitly — destination frequency range, hold-high duration, fan-in bound, expected switching-rate range, level-protocol requirement on the downstream consumer.
3. Write the assertions that fire if any of those assumptions is violated in a reuse context.

That documentation is what makes the next reuse review tractable. It is also the artifact that protects the decision on audit: the block ships because the assumptions are explicit and checked, not because nobody remembered to look.

The same posture extends to RTL that is not yet taped out but is far enough into its verification cycle that restructuring the signal path carries its own risk. Every mitigation that fixes a rule violation on the (a)/(c)/(d) pattern — adding a source-domain flop, moving from 2FF to 3FF, re-timing the OR — adds latency to the request path, and the further a design is into its tapeout cycle the more that latency change costs in re-verification, STA re-closure, and downstream functional-coverage invalidation. Weeks before tapeout, that cost can exceed the empirical safety the existing structure already demonstrates for the regime it is operating in. At that stage the same posture is the right one: characterize the regime, document the assumptions, write the assertions, and do not re-open the signal path for a rule whose failure mode the design is empirically handling.

Absolutism on the rule is for the phase where re-timing is cheap. Once timing is committed — in silicon or in late-cycle RTL — the calculus is different, and "document, don't redesign" is the posture that keeps known-good timing paths intact while still closing the audit trail on why the violation is being waived.

### Why (b) and (d) do not get this treatment

None of the above applies to (b) or (d). Those configurations are not conditionally safe at any frequency, and no amount of exponent cushion saves them. Their failure mode is a phantom request asserted from a state the source FSM never actually reached — a silent functional lie, not a tolerated glitch. The silicon-works evidence that explains (a) and (c)'s apparent invisibility at low frequencies is worth nothing here: (b) and (d) failures do not present as CDC bugs. They present as intermittent functional oddities, marginal corner cases, flaky tests that pass on re-run, wrong interrupts fired occasionally, state machines that wedge once every few hours. A design with (b) or (d) that "hasn't had failures" is often a design with undiagnosed failures that have been triaged to other piles.

Existing (b) or (d) code gets fixed regardless of reuse context or field history. There is no frequency-regime defense to fall back on.

---

## Summary: design rules that fall out of this

1. **Never feed combinational logic through a synchronizer.** This is the most dangerous configuration. Comb outputs glitch through intermediate encodings and can assert a 1 from a state that was never actually reached — a functional false positive that the synchronizer faithfully forwards. Register in the source clock first, then synchronize.
2. **One source clock per synchronizer input.** Merging independently-clocked signals before the synchronizer degrades MTBF and breaks the formula's input assumptions, but — unlike comb logic — can only distort real requests, not invent them. Still wrong: synchronize each separately, then OR in the destination domain.
3. **Use level protocols across synchronizers, not edge-counted pulses.** Level protocols tolerate brief glitches, missed samples, and metastable resolutions; edge protocols amplify them into phantom events.
4. **When in doubt, put a flop between combinational logic and a synchronizer.** Cheap insurance that turns any glitch into a clean synchronous transition and restores the MTBF formula to its ideal assumptions.
5. **Use SYNC-library cells as a baseline, not a mitigation.** Whenever the standard-cell library provides a synchronizer-variant flop (or a paired-cell 2FF synchronizer), use it for every CDC crossing. These cells are characterized for small τ, and the paired-cell variants place the two flops directly adjacent with no routing between them, which maximizes T_r and pushes the exponent as high as silicon allows. There is no reason to send a CDC path through a general-purpose flop when a SYNC cell is available. Additional mitigations — runt suppression at the source, 3FF, slower destination sampling — are for when the exponent is still marginal after the baseline is in place.
6. **Know your `T_r / τ`.** At low frequencies the exponent buries everything. At high frequencies the exponent gets small and linear factors — including glitch-induced `f_data` and `T_w` degradation — start showing up in your failure budget. That is when the additional mitigations from rule 5 move from optional to required.
7. **The rule is absolute for new design.** The physics of (a) and (c) can be defended under specific conditions (level protocol, sufficient hold-high, MTBF exponent dominant over linear degradation). That defense is accurate and does not rehabilitate the violation. IP blocks do not come stamped with the frequency regime they are safe in, and the exponent cushion that makes low-frequency rule violations invisible is exactly the thing that evaporates when the block is reused at GHz rates. Correct-by-construction is regime-independent; safe-under-assumptions is regime-specific, and reuse crosses regimes.
8. **Once timing is committed, the calculus changes — for shipped silicon and for late-cycle RTL.** Working silicon has already closed out the MTBF question empirically for its deployed regime; redesigning it trades known-good silicon for regression risk. The same posture applies to RTL that is far enough into its verification cycle that restructuring the signal path invalidates downstream timing and coverage. In both cases: document the assumptions the existing structure relies on (frequency range, hold-high, fan-in, level-protocol dependency), write the assertions that fire if any assumption is violated by a future reuse, and leave the RTL alone. That applies to (a) and (c). (b) and (d) get fixed regardless, because their failure mode is a silent functional lie that never surfaced as a CDC bug in the first place.
