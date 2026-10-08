# ns3-DEFIANCE / ns3-ai / ns3-gym Evaluation

An independent setup, testing, and bug-fixing evaluation of three
reinforcement-learning-for-network-simulation frameworks on
[ns-3.45](https://www.nsnam.org/):
[ns3-DEFIANCE](https://github.com/DEFIANCE-project/ns3-DEFIANCE),
[ns3-ai](https://github.com/DEFIANCE-project/ns3-ai), and
[ns3-gym](https://github.com/tkn-tub/ns3-gym).

## Goal

Document a clean, reproducible install of all three frameworks, then
systematically test every bundled example scenario — fixing what's
fixable, and clearly documenting what isn't — to produce an honest,
evidence-based picture of how well these frameworks actually work out of
the box on a current ns-3 release.

## Headline results

- **Install and build: fully working for all three frameworks.** ns-3.45
  builds cleanly (2000+ targets) with `ai`, `defiance`, and (separately)
  `opengym` all compiling without errors.
- **9 example scenarios fixed and confirmed working**, each with a real,
  identified root cause and a working patch — not just "it runs," but
  output verified against each example's documented expected results.
- **4 issues documented as genuinely unresolved** (upstream RLlib
  incompatibility, a native crash, an environment-level TensorFlow/glibc
  conflict, a partial fix with a remaining hang) — included because a
  fair evaluation reports failures as rigorously as successes.
- Along the way: found and fixed two real, previously-undocumented build
  bugs in ns3-gym's CMake system (never officially updated for ns-3.45),
  and a disk-space/dependency crisis that silently broke an entire module
  via a transitive `apt autoremove`.

## Results at a glance

| Framework | Fixed & confirmed | Unresolved / blocked |
|---|---|---|
| ns3-DEFIANCE | 1 (`defiance-pendulum`) | 3 |
| ns3-gym | 2 (`opengym-basic-example`, `opengym-2`) | 4 dropped (deprecated APIs, out of scope) |
| ns3-ai | 6 | 1 partial, 1 environment-blocked |

Full details, error traces, root causes, and exact patches for every item
are in [`notes/known-issues.md`](notes/known-issues.md).

## Contents

- [`docs/setup-guide.md`](docs/setup-guide.md) — step-by-step, verified
  install instructions for ns-3.45 + ns3-ai + ns3-DEFIANCE.
- [`notes/known-issues.md`](notes/known-issues.md) — the full writeup:
  every bug found across all three frameworks, with root causes,
  reproduction steps, and fixes (or clear documentation of why something
  can't be fixed).
- [`scripts/run-all-scenarios.sh`](scripts/run-all-scenarios.sh) — a
  time-limited test harness for the DEFIANCE scenario targets.
- [`results/`](results/) — raw logs and summary tables from test runs.

## Environment

- Ubuntu 24.04 (noble), VirtualBox VM, 39GB disk, ~4GB RAM
- ns-3.45
- Python 3.12, managed via Poetry
- ray 2.49.1 / RLlib (new API stack) for DEFIANCE
- Note: `contrib/ai`/`contrib/defiance` and `contrib/opengym` cannot
  coexist in the same ns-3 build (header namespace collision) — see
  known-issues.md Part 2.4 for the workaround used throughout testing.

## Status

Complete for this phase. Remaining open items (not pursued further):
`defiance-lte-learning` and several other DEFIANCE bundled examples,
and the four ns3-gym examples dropped for using deprecated ns-3 APIs.
