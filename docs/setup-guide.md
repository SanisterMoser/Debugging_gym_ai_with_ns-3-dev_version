# ns3-DEFIANCE + ns3-ai Setup Guide

Verified working install on: Ubuntu (noble/24.04), VirtualBox VM, ns-3.45

This guide documents a confirmed, end-to-end working installation of
[ns3-DEFIANCE](https://github.com/DEFIANCE-project/ns3-DEFIANCE) (a
multi-agent reinforcement learning framework for ns-3 network simulations)
and its dependency [ns3-ai](https://github.com/DEFIANCE-project/ns3-ai).

The full build succeeds (2052/2052 targets). Running the bundled example
RL scenarios currently fails due to an upstream bug — documented at the
end of this guide, not an installation problem.

---

## 1. System dependencies

```bash
sudo apt update
sudo apt install -y git cmake ninja-build build-essential python3 python3-pip python3-venv \
    pkg-config libboost-all-dev protobuf-compiler libprotobuf-dev \
    gcc g++ ccache curl
```

All of these were already present on a standard Ubuntu 24.04 install except
`curl`, which had to be installed separately.

## 2. Install Poetry

DEFIANCE manages its Python dependencies (Ray, Gymnasium, PyTorch, SUMO
bindings, etc.) via Poetry.

```bash
curl -sSL https://install.python-poetry.org | python3 -
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Verify:
```bash
poetry --version
```

## 3. Clone ns-3.45

DEFIANCE's latest release targets **ns-3.45** specifically.

```bash
git clone https://gitlab.com/nsnam/ns-3-dev.git -b ns-3.45
cd ns-3-dev
export NS3_HOME=$(pwd)
echo "export NS3_HOME=$NS3_HOME" >> ~/.bashrc
```

Confirm the version:
```bash
cat VERSION
# should print: 3.45
```

## 4. Clone ns3-ai and ns3-DEFIANCE into contrib/

```bash
git clone https://github.com/DEFIANCE-project/ns3-ai contrib/ai
git clone https://github.com/DEFIANCE-project/ns3-DEFIANCE contrib/defiance
```

Note the folder name is `contrib/defiance` (lowercase) even though the repo
is named `ns3-DEFIANCE` — ns-3's module system expects the lowercase name.

Sanity check both cloned correctly:
```bash
ls contrib/ai/        # CMakeLists.txt docs examples LICENSE model python_utils README.md
ls contrib/defiance/  # CMakeLists.txt doc Dockerfile examples experiments helper ...
```

## 5. Install Python dependencies (without local package)

```bash
poetry -C contrib/defiance install --without local
```

This creates a dedicated virtualenv (`ns-defiance-<hash>-py3.12`) and installs
~80 packages including `ray[rllib]`, `gymnasium`, `torch`, `sumolib`,
`geopandas`, and `wandb`. This step takes several minutes — most of the time
goes to downloading `eclipse-sumo` and `opencv-python`.

## 6. Activate the virtualenv

```bash
source $(poetry -C contrib/defiance env info --path)/bin/activate
```

Your prompt should now show `(ns-defiance-py3.12)`. Do this in every new
terminal session before running `./ns3` or `run-agent`.

## 7. Configure ns-3

```bash
./ns3 configure --enable-python --enable-examples --enable-tests
```

Watch for missing dependency warnings here (boost, protobuf, python
bindings) — better to catch them now than after a 30+ minute build.

## 8. Build ns3-ai first

ns3-ai generates protobuf message types that other modules depend on, so
build it before the full build:

```bash
./ns3 build ai
```

You may see:
```
lto-wrapper: warning: using serial compilation of 3 LTRANS jobs
```
This is harmless.

## 9. Install the local DEFIANCE Python package

```bash
poetry -C contrib/defiance install --with local
```

This installs `ns3ai-gym-env` and `ns3ai-python-utils` from the local
`contrib/ai` path, plus the `ns-defiance` project itself.

## 10. Full build

```bash
./ns3 build
```

On a 2-3 core VirtualBox VM this took roughly 20-30 minutes and compiled
2052 targets with zero errors, including all `contrib/ai` example modules
(rl-tcp, lte-cqi, rate-control) and all `contrib/defiance` example modules.

## 11. List available DEFIANCE example targets

```bash
./ns3 show targets | grep -i defiance
```

Confirmed targets from this build:
```
defiance                          defiance-test
defiance-addition-example         defiance-balance1
defiance-balance2                 defiance-handover-example
defiance-lte-animation            defiance-lte-learning
defiance-lte-test                 defiance-pendulum
defiance-sumo-test                defiance-agent-communication-example
defiance-app-communication-example
defiance-channel-interface-example
defiance-environment-creator-example
defiance-observation-sharing-example
defiance-sumo-topology-creator
```

Note: the `examples/` folder name (e.g. `handover-example`) doesn't always
match the built target name (`defiance-handover-example`) — always check
`./ns3 show targets` rather than guessing from the folder listing.

## 12. Run a training scenario

```bash
run-agent --help
run-agent train -n <target-name>
```

This launches ns-3 as a subprocess, connects it to a Ray/RLlib PPO trainer
over the ns3-ai message interface, and begins training. A running instance
prints periodic status tables:

```
Trial status: 1 RUNNING
Current time: ...  Total running time: Nm Ns
Logical resource usage: 2.0/3 CPUs, 0/0 GPUs
```

and creates checkpoints under `~/ray_results/PPO_<timestamp>/`.

---

## Known issue: example scenarios fail with an action-space error

**Status as of this build: unresolved, upstream bug — not a setup problem.**

Both `defiance-handover-example` and `defiance-balance1` fail with:

```
ValueError: Box(..., `int`) action spaces are not supported. Use MultiDiscrete or Box(..., `float`).
```

**Root cause:** DEFIANCE's environment code defines certain action spaces as
an integer-typed `gymnasium.spaces.Box`. Ray's RLlib, on the version this
project's `poetry.lock` resolves to (`ray==2.49.1`), now uses the "new API
stack" by default, which explicitly rejects integer-typed `Box` action
spaces in favor of `MultiDiscrete` or float `Box`. This is a real
compatibility gap in DEFIANCE's own code against its pinned Ray version,
not a symptom of a bad local install.

**Symptoms observed:**
- `defiance-handover-example`: ran 50 PPO iterations without error, but
  `num_env_steps_sampled_lifetime` stayed at 0 the entire time (env runner
  never returned samples), then crashed at checkpoint-selection time with
  `Invalid metric name episode_reward_mean!` (that metric never appeared
  because no real training occurred).
- `defiance-balance1`: failed immediately (~43s) with the `Box(..., int)`
  error above, killing the env runner actor before any iterations completed.

**Suggested next steps (not yet attempted):**
- Try `defiance-pendulum` or `defiance-lte-learning` — may use a different
  action space type that isn't affected.
- Pin an older `ray`/`rllib` version compatible with the "old API stack"
  (`config.api_stack(enable_rl_module_and_learner=False, enable_env_runner_and_connector_v2=False)`)
  and see if that unblocks the integer Box space.
- File an issue upstream at
  `github.com/DEFIANCE-project/ns3-DEFIANCE/issues` with this exact
  traceback — worth doing since the repro is clean and versioned.

---

## Summary

| Stage | Status |
|---|---|
| System deps, Poetry install | ✅ Working |
| ns-3.45 clone | ✅ Working |
| contrib/ai + contrib/defiance clone | ✅ Working |
| Poetry deps install (without local) | ✅ Working |
| `./ns3 configure` | ✅ Working |
| `./ns3 build ai` | ✅ Working |
| Poetry deps install (with local) | ✅ Working |
| Full `./ns3 build` (2052/2052) | ✅ Working |
| `run-agent train -n defiance-handover-example` | ⚠️ Runs, but 0 real training samples, crashes at checkpoint step |
| `run-agent train -n defiance-balance1` | ❌ Fails immediately — action space error |
