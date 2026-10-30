# Project structure

Read this file first when working in this checkout, then `AGENTS.md` and the
documentation for the affected backend or feature.

## Main paths and entry points

- `src/`, `include/`: C++ engine and GPU interfaces. `src/program/generate.cpp`
  contains engine argument parsing, generation, serving and conversation caching.
- `metal/`: macOS Metal backend, its CMake project, kernels and tests;
  `docs/MAC.md` describes setup and limitations.
- `sycl/`: Intel/SYCL backend; see `docs/INTEL.md` and `docs/INTEL_ARC.md`.
- `src/dsv41/`: DeepSeek V4.1 engine; Metal also builds `strata_ds4`.
- `apps/desktop/`, `apps/host/`: Tauri desktop shell, Node host and app web UI.
- `strata.py`, `strata_cli/`: Terminal entry point and commands; release setup
  uses `strata launch`. Source installation is in `docs/SOURCE_INSTALL.md`.
- `packaging/`: release packages and update feeds.
- `serve/server.py`: Python HTTP server, OpenAI/Anthropic APIs, engine bridge and
  metrics. `serve/web/` is the browser UI, including the Monitor tab.
- `setup.py`, `setup.sh`, `START-HERE.bat`: installer entry points;
  `serve/runconfig.py` handles run configuration. `strata_control/` provides
  control tooling; `chat.py` is a chat client.
- `tools/`: setup tests, utilities and diagnostics. `tests/` and backend-local
  tests cover the engine. `bench/` contains benchmarks and recorded evidence.
- `docs/`: installation, engine/API details, backend guides and experiments.
  Start with `AI_SETUP.md` for installation and `DETAILS.md` for runtime behavior.
- `third_party/`, `ref/`, `data/`: dependencies, reference material and model data.
  `build-metal/` and `build-vision/` are local build outputs in this checkout.

## Build and checks

- Setup checks without GPU/downloads: `python tools/test_setup_choices.py` or
  another relevant `tools/test_setup_<name>.py`.
- Python server checks: `python -m pytest serve/test_<feature>.py`.
  Root `conftest.py` isolates settings to avoid touching real model/config files.
- Terminal/control checks: `python -m pytest tests/terminal`; app checks:
  `npm test` in `apps/host/`.
- Main engine: root `CMakeLists.txt`; follow the relevant backend guide before
  configuring. Build every backend affected by an engine change.
- Metal: `cmake -S metal -B build-metal -G Ninja -DCMAKE_BUILD_TYPE=Release
  -DMETAL_CPP_DIR=/path/to/metal-cpp`, then `cmake --build build-metal`.
  See `docs/MAC.md` and `metal/CMakeLists.txt` for dependencies and targets.

## Conventions and cache investigation

- Keep new features opt-in. Support performance claims with measurements and
  their hardware/build context; preserve unrelated worktree changes.
- Custom includes main through `870aedd9` (0.2.8 / engine 0.1.54), while keeping
  the local Metal conversation-cache changes. In the CUDA compatibility layer,
  unavailable capability/priority queries initialize outputs before failing;
  main's peer-copy synchronization and peer-disable behavior are retained.
  Optional-feature stubs in `stubs_0145.cpp` must not also appear in `stubs.cpp`.
- Prompt-prefix reuse, parked conversation snapshots and VRAM expert hits are
  separate measurements. The Monitor's conversation park/restore counts come
  from parsed engine stderr, not from inspecting snapshots directly.
- For alternating agent requests, inspect backend support, effective
  `--conversation-cache-*` arguments and engine skip logs before increasing
  cache budgets. Configuration alone does not establish successful parking.
- `docs/PROMPT_CACHE_TAIL.md` and `docs/MESSAGE_BOUNDARY_CACHE.md` describe
  optional checkpoints for prompts sharing a prefix; these differ from keeping
  independent conversations across switches.
- `.agents/skills/strata-cache-diagnosis/SKILL.md` records how to match Monitor
  misses to engine log events and distinguish snapshot errors from RAM limits.
- Verify the actual engine build before changing cache code. This checkout's
  `f4bca935` fixes Metal checkpoint tail/position metadata, exposes the main and
  draft K/V buffers to snapshots and adds macOS RAM telemetry. An executable
  built before that commit can still run while reporting snapshot errors.
  `metal.setup_mac.engine_current()` checks the setup build's source digest;
  `metal.setup_mac.build_engine()` rebuilds and updates `build-metal/BUILD.json`.
- Live parking regression: `.venv/bin/python tools/parking_test.py --exe
  "$PWD/build-metal/strata" --config /path/to/run-config.json --extra
  "--max-context 16384 --pcie-frac 0 --adapt-every 1000000 --greedy"`.
  It starts two engines sequentially, compares follow-up tokens with/without a
  conversation switch, and checks restoration and retained K/V reuse. It uses
  the engine helper's temporary log path, including macOS's per-user temp dir.
