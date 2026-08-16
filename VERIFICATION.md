# How this calculator is verified

This document records what has been checked in this tool, how, and what the
result does and does not entitle anyone to conclude. It exists because the
README carries a disclaimer, and a disclaimer only says what a tool does not
promise. This says what was actually done.

## What this does not establish

Read this part first, because everything below is easy to mistake for something
it is not.

Nothing here has been checked against an aircraft. There is no flight test, no
measured takeoff, no independent review, and no approval of any kind.
Verification here means one thing only: **the figures in the code were compared
against the figures the handbook publishes, and the behaviour built on them was
compared against what those figures imply.** A calculation can be faithful to a
handbook and still be the wrong thing to fly on, because the handbook describes
a new aircraft flown by a test pilot on a prepared surface, in a condition
yours may not be in.

Nor does the tool know anything about the runway you type in. It compares two
numbers it computed against one number you gave it. Whether that number is the
runway you are actually standing on, whether it is the declared distance that
applies to your departure, and whether there is anything past the far end, are
all outside what it can check.

Use the official POH for your aircraft. Ask a qualified instructor. That is not
boilerplate: it follows from what is written below.

## Where the figures come from

One document: **POH-162-00-40-001, rev. A07**, for the Pipistrel ALPHA
Trainer. Section 5.6 for the takeoff distances against temperature and
elevation, section 5.3 for the wind table. The ALPHA Trainer PRO (162A) has
its own handbook, POH-162-00-40-003, which nothing here is checked against;
every figure in this project traces to rev. A07 above and to no other
document.

**Those tables are published for one weight: MTOM, 550 kg.** The handbook states
it as a condition of the figures, and this calculator inherits it whole. It asks
for no weight and applies no correction for one, so every distance it prints is
the distance for an aircraft at its maximum takeoff mass. The handbook publishes
no correction for any other weight, and nothing here establishes what one would
be.

## The suite

166 assertions across seven modules, all running in the browser against the same
code the page uses.

| Module | Assertions | What it is for |
| --- | ---: | --- |
| Readiness & Validity | 50 | What counts as enough input, and when a result is labelled extrapolated or refused |
| Persistence | 37 | Session restore, corrupt payloads, the data age clock |
| getWindFactors (POH wind table) | 21 | Reading between the table rows, and past their ends |
| Wind Components | 18 | Headwind, tailwind and crosswind derived from two headings |
| Core Calculation Logic | 17 | Worked examples end to end, from inputs to distances |
| POH source data (Section 5.6) | 13 | Every figure written out a second time against the handbook |
| Invariants (grid sweeps) | 10 | Relationships that must hold at every point of a grid |

Core Calculation Logic, Wind Components and Persistence are measured together as
one layer of 72 assertions in the experiment below, because they are all worked
examples of one kind or another.

Four ideas shape how those modules are split.

**A figure is checked against its source, not against itself.** Every number
taken from the handbook is written out a second time in the suite, next to the
section it comes from. This is the only kind of check that catches a value which
is correct in form and wrong in fact. A transposed digit in a table cell breaks
no relationship and passes every worked example that does not happen to land on
that row.

**Reading the table and going past it are different things, and the tool has to
say which it did.** The published wind table runs from 6 kt of tailwind to 12 kt
of headwind, and the temperature and elevation tables have ends of their own.
Beyond them the interpolation would clamp to the edge and produce a number that
looks as precise as any other. So the verdict carries a label: within the
published range, extrapolated, or refused outright. That label has its own
module, because a number without it is a different claim.

**A value that is missing is not a value that is wrong.** Every input starts
empty and produces a pending state, not an error and not a zero. The distinction
has its own tests because losing it is how a blank form starts reporting a
takeoff distance computed at 0 °C and sea level.

**Relationships are swept systematically, not sampled at a few points.** Where a
property has to hold everywhere, such as the distance to clear a 50 ft obstacle
never coming out shorter than the ground roll, it is checked over a
deterministic grid spanning the supported ranges and their boundaries. A grid
over continuous inputs is still a finite sample of them, and no number of grid
points proves a property over a continuum. What it buys is which points get
visited, stated in the code and the same on every run.

## Why these techniques

Three choices are worth stating, because each had a plausible alternative.

**The tests run in the browser, against the file that ships.** The calculator is
one self-contained HTML file with its dependencies vendored beside it and no
build step. A test runner under Node would need the logic pulled out into a
module, or a bundler standing in front of it, and either one puts a translation
step between what was tested and what is served. Loading the shipped file in a
browser removes that gap. The price is that running it needs a browser, which
`run_tests.sh` handles headlessly.

**Deterministic grid sweeps rather than property-based testing.** Property-based
testing was evaluated for the sweeps, with fast-check, and passed over for two
reasons.

The smaller one is packaging. Since its 2.x release fast-check publishes no
browser build; the documented options are importing it from a CDN, which would
break the offline guarantee this repository is built around, or bundling it
locally into a vendored file. Workable, and not the deciding factor.

The deciding one is the shape of the input space. This calculator takes a
handful of continuous parameters over known, bounded ranges, and what needs
checking in it is not a scattering of interior points but the edges: the ends of
the handbook tables, where reading past the last row begins, and the boundary of
every limit. A grid can be laid down to land on those edges on purpose, and to
step through the interior at a known spacing on the way. Random generation
distributes its draws instead, spending most of them where the arithmetic is
unremarkable and reaching a narrow band at an edge only by chance and only with
enough draws. Neither approach exhausts a continuum. Over a space this small,
choosing the points deliberately is worth more than drawing more of them.

The coverage is also legible. A grid says in the code which points it visits and
what spacing it visits them at, and that statement does not change with a seed,
a generator or a library version. Property-based testing reproduces perfectly
when the seed is fixed, which it must be for this kind of use, but what a run
actually covered is not something a reader can see without running it.

That reasoning has a boundary, and it is worth naming. Property-based testing
would still be the better instrument for two things this suite does not do:
throwing arbitrary payloads at the session restore, which is covered here by
handwritten ones, and generating random sequences of actions over the unit
switch and the reset. Bugs of that shape do not live in a bounded numeric space
and a grid will not find them.

**Mutation testing rather than coverage.** Coverage reports which lines ran. A
line can run in every single test and still have nothing asserted about it:
every table cell in this calculator is executed by any worked example, so a
coverage report would show the whole file green while a wrong figure sailed
through. Mutation testing asks the question coverage cannot, which is whether a
defect would be noticed.

## Measuring the suite: a mutation experiment

To answer that question, 37 deliberate defects were introduced into the
application code, one at a time, and the suite was run against each. All of it
ships with the repository: the defects in `verification/mutations.json`, the
harness in `verification/run_mutations.py`, the outcome in
`verification/results.json`.

The set is composed to cover the ways this kind of tool goes wrong: 22 edits to
figures taken from the handbook, including transposed digits and values that
stay plausible in their column; 7 to the arithmetic and the flow around it; and
8 that widen a limit, remove a check, or stop the tool from saying it has left
the published range. That last group is written deliberately, because a
calculator that exists to say "this runway is not long enough" fails dangerously
in one direction only.

Each mutation is applied only to the application half of the file, never to the
tests, so that the module which writes the figures out a second time cannot be
edited along with the value it checks. Every layer is then run in isolation, so
a defect can be attributed to the module that catches it rather than to the
suite as a whole.

**All 37 are caught.** Attribution, counting only the mutations a given layer
catches when no other layer does:

| Layer | Assertions | Mutations it catches on its own |
| --- | ---: | ---: |
| Readiness and validity | 50 | 11 |
| POH source data | 13 | 6 |
| The wind table | 21 | 2 |
| Worked examples | 72 | 2 |
| Grid sweeps | 10 | 0 |

## What each layer turned out to be worth

**Writing the figures out against the handbook earns its place.** Six mutations
are caught by nothing else, and four of them are single table cells edited to a
value that keeps its column in order. A transposed digit in a row no worked
example happens to land on breaks no relationship and produces a number that
looks like every other number this tool prints.

**Saying which side of the published range a reading came from is the heaviest
single job in the suite.** Eleven mutations are caught only by the module that
checks it, and they are the ones that matter most: a limit widened, a check
removed, a result presented as a reading when it came from a row this project
invented rather than from the handbook. The distances are only half of what this
calculator outputs. The other half is the label on them.

**The grid sweeps caught nothing on their own.** The measured unique
contribution is zero, over 37 mutations, and there is no honest way to report
that as anything else. They are kept because what they assert is different in
kind: a relationship checked at every point of a grid rules out a whole class of
wrong answers rather than a list of specific ones, and the count above measures
the suite against 37 chosen mutations rather than against that class. That is a
reason to keep them. It is not a demonstration that they are earning their
place, and this document does not have one.

## Known deviations and open questions

**The hot-atmosphere factor is applied always.** The handbook publishes
`L = 1.10 * (Lh + Lt - Lo)` for a hot atmosphere. This tool applies the 1.10
unconditionally, as a buffer, including on a cold day when the handbook does not
call for it. That is a deliberate departure from the published method, in the
direction that asks for more runway than the handbook does. It is recorded here
rather than presented as what section 5.6 says.

**The rows past the ends of the wind table are invented.** The published table
runs from 6 kt of tailwind to 12 kt of headwind. Beyond it the tool carries two
rows that are not handbook data:

- At 40 kt of tailwind, a deliberately aggressive slope, to penalise a condition
  nobody should be departing into.
- At 40 kt of headwind, the ground roll follows the last published slope, but
  the obstacle column does not follow its own. Extending both would make two
  lines with different slopes cross, and past the crossing the distance to clear
  a 50 ft obstacle would come out shorter than the ground roll, which cannot
  happen. The obstacle column holds the ratio published at 12 kt instead, which
  understates how much a headwind helps and therefore asks for more distance.

Both are choices, both are conservative, and the interface labels any result
that depends on them as extrapolated.

**The handbook transcription was done by hand.** The figures are checked against
what a person read off the handbook. That comparison catches drift between the
code and the transcription. It cannot catch an error made while transcribing.

## Limits of what was measured

Mutation testing measures a suite against defects that somebody thought to
introduce. The 37 here were chosen to cover table figures, arithmetic,
boundaries and the direction of each comparison. None of that says anything
about a defect nobody thought of. A count of zero escapes is a statement about
this set of mutations, not about the code.

The suite runs in headless Chromium only. It exercises the calculation and the
state machinery, not the rendering: the scale drawing of the runway is not
inspected by any test, and no test looks at a pixel.

## Reproducing this

The suite:

```bash
./run_tests.sh
```

Or open `index.html?test=true` in a browser to watch it run, and use QUnit's
filter to isolate a module:

```
index.html?test=true&filter=POH source data
index.html?test=true&filter=!Invariants
```

The mutation experiment:

```bash
python3 verification/run_mutations.py          # the whole set
python3 verification/run_mutations.py C04 P01  # or just these
```

It needs nothing beyond Python 3 and the same browser the suite uses. Nothing on
disk is modified while it runs: the mutated page is held in memory and served
from there, so an interrupted run cannot leave a defect behind in `index.html`.

The recorded outcome carries the SHA-256 of the two files it was measured over,
and there is a check that takes milliseconds rather than minutes:

```bash
python3 verification/run_mutations.py --check
```

CI runs it on every push. A count measured over a file that has since changed is
not a weaker count, it is a statement about something that no longer exists, and
this is what keeps the number above from quietly becoming one. The whole of
`index.html` is hashed, tests included, because what each layer catches depends
on the tests as much as on the code. An edit that turns out to change nothing
still has to be measured for anyone to know that.

A count is only worth as much as the set it was measured over, which is why the
set ships too. Reading it is the fastest way to judge whether the number above
means anything, and adding to it is better still: a defect this set does not
cover is a defect nobody has checked for.
