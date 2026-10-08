# Known Issues & Fixes — ns3-DEFIANCE, ns3-ai, ns3-gym on ns-3.45

Findings from installing and testing three RL-for-networking frameworks on a
clean, verified ns-3.45 build: [ns3-DEFIANCE](https://github.com/DEFIANCE-project/ns3-DEFIANCE),
its dependency [ns3-ai](https://github.com/DEFIANCE-project/ns3-ai), and the
older [ns3-gym](https://github.com/tkn-tub/ns3-gym) (app-ns-3.36+ branch).

---

## Part 1 — ns3-DEFIANCE example scenarios

### 1.1 `defiance-pendulum` — hardcoded output path is wrong (FIXED)

**Symptom:** `./ns3 run defiance-pendulum` crashes with
`NS_FATAL: Output stream is not valid for writing` (SIGABRT).

**Cause:** `examples/scenario/pendulum-cart/test-cart.cc` hardcodes its
default output path missing a `/scenario` directory segment.

**Fix:** pass the correct path explicitly:
```bash
./ns3 run "defiance-pendulum --OutFilePath=$NS3_HOME/contrib/defiance/examples/scenario/pendulum-cart/log.txt"
```
Produces a valid 5000-row CSV log. Proper upstream fix: correct the
hardcoded default path in `test-cart.cc`.

### 1.2 `defiance-balance1` / `defiance-balance2` — RLlib rejects integer Box action space (UNRESOLVED)

**Symptom:** `ValueError: Box(..., int) action spaces are not supported.
Use MultiDiscrete or Box(..., float).`

**Cause:** DEFIANCE's environment code defines integer-typed
`gymnasium.spaces.Box` action spaces; `ray==2.49.1`'s RLlib "new API stack"
(default in this version) rejects those.

### 1.3 `defiance-handover-example` — SIGABRT crash (UNRESOLVED)

Crashes ~30s into training with a native core dump. Separately, an
untimed run completed 50 PPO iterations with
`num_env_steps_sampled_lifetime` stuck at 0 the whole time — the env
runner never delivered real samples even when it doesn't crash outright.

### 1.4 `defiance-lte-learning` — standalone demo misclassified as trainable (reclassified, not a bug)

Fails with `Invalid command-line argument: --seed=1` when run via
`run-agent train`. Same pattern as 1.1 — likely a standalone `./ns3 run`
demo, not an RL-trainable scenario. Not fully investigated.

---

## Part 2 — ns3-gym (tkn-tub) on ns-3.45

ns3-gym was never officially tested against ns-3.45. Getting it to build at
all required several real fixes to its CMake/build system and example code.

### 2.1 Build target naming collision (FIXED)

`contrib/opengym/examples/CMakeLists.txt` defined an example literally
named `opengym`, colliding with the `opengym` module library itself.
**Fix:** renamed the example target to `opengym-basic-example`.

### 2.2 Broken protobuf object-library linkage (FIXED)

ns-3.45 removed the old `${module}-obj` auto-generated object-library
convention that ns3-gym's `CMakeLists.txt` relied on
(`protobuf_generate(TARGET ${libopengym-obj} ...)` resolved to nothing).

**Fix:** explicitly created an object library and linked it into the
module, mirroring how `contrib/ai` does it:
```cmake
add_library(opengym-proto-objects OBJECT "${CMAKE_CURRENT_SOURCE_DIR}/model/messages.proto")
protobuf_generate(TARGET opengym-proto-objects ...)
# in source_files, replace ${proto_source_files} with:
$<TARGET_OBJECTS:opengym-proto-objects>
```

### 2.3 Four bundled examples use removed/deprecated ns-3 APIs (DROPPED)

`interference-pattern`, `linear-mesh`, `linear-mesh-2` call
`InetSocketAddress::SetTos()` (removed in modern ns-3); `rl-tcp` overrides
`TcpSocketDerived::GetInstanceTypeId()`, which is now `final` on the base
`Object` class. All four were removed from the build
(`examples/CMakeLists.txt`) rather than patched — out of scope for this
evaluation. `opengym-basic-example` and `opengym-2` remain and both run
successfully.

### 2.4 contrib/ai and contrib/opengym cannot coexist in one build (STRUCTURAL)

Both modules define similarly-named classes
(`OpenGymBoxContainer`, `container.h`, etc.) that collide in ns-3's merged
`build/include/ns3/` header namespace — whichever module builds first
"wins" and the other's example code silently includes the wrong headers,
causing undefined-reference linker errors.

**Workaround used throughout this evaluation:** keep `contrib/ai` and
`contrib/defiance` in `~/disabled-modules/` while testing `opengym`, and
vice versa; swap directories + full clean rebuild (`rm -rf build
cmake-cache`) when switching focus.

---

## Part 3 — ns3-ai's own bundled examples

Two recurring bug patterns accounted for nearly every failure below.

**Pattern A — segment name mismatch.** `ns3ai_utils.Experiment` (Python)
defaults to shared-memory segment name `"ns3-ai"`, but
`Ns3AiMsgInterface::BuildSegmentName()` (C++) hardcodes
`"ns3-ai_" + m_trailName`, defaulting to `"ns3-ai_single_trial"`. Every
message-interface example that didn't explicitly pass a matching
`segName=` crashed with
`boost::interprocess::interprocess_exception: No such file or directory`.
**General fix:** add `segName="ns3-ai_single_trial"` to the `Experiment(...)`
call.

**Pattern B — uint action/observation spaces.** `gymnasium`'s Gym-interface
wrapper (`ns3ai_gym_env`) rejects unsigned-int `Box` spaces with
`NotImplementedError: uint is not supported by all rl frameworks. Use int
instead!`. **General fix:** change `TypeNameGet<uintN_t>()` to
`TypeNameGet<intN_t>()` and `OpenGymBoxContainer<uintN_t>` to
`OpenGymBoxContainer<intN_t>` in both the space-definition and the matching
data-container code.

### 3.1 `ns3ai_apb_gym` — FIXED (Pattern B)
Action space (`uint32_t` to `int32_t`) and observation space, plus their
`OpenGymBoxContainer` types, patched in `apb.cc`. Runs cleanly, matches
README's `set:`/`get:` output exactly.

### 3.2 `ns3ai_apb_msg_stru` / `ns3ai_apb_msg_vec` — FIXED (two separate bugs)
- **Missing link libraries:** the `pybind11_add_module(...)` targets for
  both variants never linked `${libai} ${libcore}`, causing
  `undefined symbol: _ZN3ns34Time10StaticInitEv` on import. Fixed by
  adding `target_link_libraries(... PRIVATE ${libai} ${libcore})` to each.
- **Pattern A.** Both scripts needed the `segName` fix. Both ran to
  completion (10000 iterations).

### 3.3 `ns3ai_ratecontrol_constant` — FIXED (two separate bugs)
- **Synchronization-order bug:** `ai_constant_rate.py`'s loop called
  `msgInterface.PySendBegin()` before checking
  `msgInterface.PyGetFinished()`. When the C++ side signalled completion,
  Python had already entered a send-begin state that never resolved,
  causing an infinite CPU-spinning hang. **Fix:** move the `PyGetFinished()`
  check to immediately after `PyRecvBegin()`, before `PySendBegin()`.
- Pattern A (`segName`).
- Confirmed: ran to completion, output matched README's expected
  throughput/delay numbers and final summary exactly.

### 3.4 `ns3ai_ratecontrol_ts` (Thompson Sampling) — FIXED
Same two fixes as 3.3. Ran to completion with ~42 Mbps average throughput
(vs ~3.6 Mbps for constant rate), matching the README's claim of
significantly better performance.

### 3.5 `ns3ai_rltcp_msg` — FIXED (Pattern A only)
Its `PyGetFinished()` ordering was already correct. Only needed the
`segName` fix. Ran a full 10000-step Deep-Q-learning TCP congestion
control simulation to completion.

### 3.6 `ns3ai_rltcp_gym` — PARTIALLY FIXED
Pattern B present in both the action space (`uint32_t`) and two
observation-space implementations (`uint64_t`, in `TcpTimeStepEnv` and
`TcpEventBasedEnv`), plus their data containers — all four patched. After
fixing, the simulation ran through all 10000 steps with correctly-formatted
observation/action logs matching the README. However, the final
`env.step()` call that should return `done=True` hung indefinitely
(26+ minutes, still consuming CPU) instead of completing — killed
manually. Root cause not confirmed; plausibly a leftover effect of the
dtype change on the termination-signal message, not investigated further.

### 3.7 `ns3ai_ltecqi_msg` — BLOCKED BY ENVIRONMENT, not a code bug
Needed the standard Pattern A fix. After resolving a VM disk-space crisis
(see Part 4) to install its `tensorflow`/`keras` dependency, the script
crashes with a **fatal glibc error**:
```
Fatal glibc error: tpp.c:83 (__pthread_tpp_change_priority): assertion failed
```
This is TensorFlow's threading code trying to set a real-time thread
priority that this VM's kernel/cgroup configuration doesn't permit — not
something fixable by patching the example's Python code. Likely resolved
on a host with proper RT-scheduling permissions, or a different TensorFlow
build.

---

## Part 4 — VM disk-space crisis (environment note)

Installing `tensorflow`/`keras` for 3.7 exhausted the VM's 39GB disk (down
to 671MB free), aborting the pip install mid-way and leaving a broken
TensorFlow install. Recovered by:
- Removing GNU Radio (`sudo apt remove --purge gnuradio* && sudo apt
  autoremove`) — freed ~1GB, but **also silently removed `pybind11-dev`**
  as a shared dependency, which caused `contrib/ai` to be entirely skipped
  on the next `./ns3 configure` (`-- Skipping contrib/ai: pybind11 not
  found`) — every `ai` example target disappeared, not just the one being
  tested. Fixed by `sudo apt install -y pybind11-dev`.
- Clearing pip and ccache caches (`pip cache purge`, `ccache -C`) — freed
  ~1.3GB.
- Removing old disabled snap revisions (`snap list --all` showed several
  `disabled` duplicates of core22/core24/firefox/gnome/mesa/snapd — each
  `snap remove <name> --revision=<rev>`) — freed ~2.3GB, plus
  `sudo snap set system refresh.retain=2` to stop future buildup.

**Lesson:** after any `apt autoremove` on this VM, re-check
`./ns3 configure` output for unexpected "Skipping contrib/X" lines before
assuming a build failure is a code problem.

---

## Summary table

| Component | Target | Status |
|---|---|---|
| DEFIANCE | `defiance-pendulum` | Fixed |
| DEFIANCE | `defiance-balance1` / `balance2` | Unresolved — RLlib incompatibility |
| DEFIANCE | `defiance-handover-example` | Unresolved — SIGABRT + zero-sample training |
| DEFIANCE | `defiance-lte-learning` | Likely standalone demo, not confirmed |
| ns3-gym | `opengym-basic-example` | Fixed (build system) |
| ns3-gym | `opengym-2` | Fixed (build system) |
| ns3-gym | `interference-pattern`, `linear-mesh`, `linear-mesh-2`, `rl-tcp` | Dropped — deprecated API calls |
| ns3-ai | `ns3ai_apb_gym` | Fixed |
| ns3-ai | `ns3ai_apb_msg_stru` | Fixed |
| ns3-ai | `ns3ai_apb_msg_vec` | Fixed |
| ns3-ai | `ns3ai_ratecontrol_constant` | Fixed |
| ns3-ai | `ns3ai_ratecontrol_ts` | Fixed |
| ns3-ai | `ns3ai_rltcp_msg` | Fixed |
| ns3-ai | `ns3ai_rltcp_gym` | Partially fixed — hangs on final step |
| ns3-ai | `ns3ai_ltecqi_msg` | Blocked — environment (glibc/RT-priority), not code |
