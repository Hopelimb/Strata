---
name: strata-cache-diagnosis
description: Diagnose low prompt reuse in Strata with alternating chat or agent requests by matching Monitor data to engine logs and snapshot admission failures.
---

# Strata cache diagnosis

Read `docs/PROJECT_STRUCTURE.md` and the multiple-conversations section of
`docs/DETAILS.md`. Distinguish prompt reuse, conversation parking and expert hits.

- Match screenshot request sizes and speeds to the configured engine log before
  attributing an incident to a process. Logs may contain several server runs;
  locate the relevant `engine started:` line and its actual backend/arguments.
- Check whether the configured executable includes the current source changes.
  In this checkout, `f4bca935` already fixes Metal snapshot tail/position metadata,
  main/draft K/V exposure and macOS memory telemetry. An older binary continued
  running after the source was updated. Use `metal.setup_mac.engine_current()`
  and compare build/source dates before adding another fix; rebuild with
  `metal.setup_mac.build_engine()` to update both executable and `BUILD.json`.
- Alternating prompt lengths suggest different request streams, but do not prove
  they are subagents. Small fixed-output requests can also be client helpers.
- Inspect effective `--prompt-cache`, `--conversation-cache-mib` and
  `--conversation-cache-slots` settings before recommending enabling them.
- Search the matching run for `conversation cache:`. `skip parking` reports why
  switching loses the outgoing state. Budget, physical-RAM admission, allocation
  and invalid snapshot payloads require different remedies. Increasing the budget
  does not fix a payload validation error.
- The Monitor's parked/restore counters are parsed by `ConvCacheLog` in
  `serve/server.py`; zero counters alone do not prove parking was disabled.
  Check the raw log. A disconnected Monitor can show stale measurements.
- For `invalid checkpoint running-state payload`, inspect
  `conversation_checkpoint_validate` in `src/core/conversation_state.cpp` and
  the active backend's checkpoint producer. Metal saves/restores through
  `metal/src/engine/window.cpp`; shared validation assumptions need verification.
  Treat the exact mismatched component as unconfirmed until measured.

For a diagnosis, report the matching raw log lines, enabled settings, observed
failure and remaining uncertainty. Avoid sending inference probes that replace
state being investigated. If a fix is requested, verify a newly built and actually
loaded backend with an A/B/A request sequence, successful park/restore events and
resumed-prefix reuse; source checks alone do not prove recovery.

`tools/parking_test.py` provides a live A/B/A comparison against an uninterrupted
reference. It starts two model engines sequentially; avoid another loaded engine
beside it. Check both token equality and nonzero restore/retained-KV events. Read
`eng.log_path`, never assume `/tmp`: macOS uses a per-user temporary directory.
