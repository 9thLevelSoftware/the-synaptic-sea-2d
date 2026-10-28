# Test Infrastructure

**Engine**: Godot 4.7.2 (GDScript)
**Test Framework**: GdUnit4
**CI**: `.github/workflows/tests.yml`
**Setup date**: 2026-10-28

## Directory Layout

```
tests/
  unit/           # Isolated unit tests (formulas, state machines, logic)
  integration/    # Cross-system and save/load tests
  smoke/          # Critical path test list for /smoke-check gate
```

```
production/qa/
  evidence/       # Screenshot logs and manual test sign-off records
```

> **Manual evidence lives under `production/qa/evidence/`, not `tests/`** — that
> is where every consumer reads it.

## Installing GdUnit4

Godot 4.7 replaced the Asset Library with the Asset Store. Either route works:

1. Editor: Asset Store → search "GdUnit4" → install; or download the latest
   release from https://github.com/godot-gdunit-labs/gdUnit4/releases and copy
   its `addons/gdUnit4/` folder into this project's `addons/`
2. Enable the plugin: Project → Project Settings → Plugins → GdUnit4 ✓
3. Restart the editor
4. Verify: `res://addons/gdUnit4/bin/GdUnitCmdTool.gd` exists (capital U)
5. Commit `addons/gdUnit4/` — CI runs the committed copy (`version: installed`)

## Running Tests

From the project root (the `commands.test` entry in `project.yaml` is the same
command with the local editor path):

```bash
godot --headless --path . --import
godot --headless -s -d --remote-debug tcp://127.0.0.1:0 res://addons/gdUnit4/bin/GdUnitCmdTool.gd -a res://tests --ignoreHeadlessMode
```

- Run the import first: a fresh clone has no `.godot/` class cache, and the run
  exits **1** (`Failed to load script`) without it.
- Exit codes: **0** all pass · **100** a test failed · **101** passed but a test
  leaked nodes (warning) · **103/104** gdUnit4 could not run · **105** a test
  script does not parse.
- `No test cases found` with exit 0 is **not** a pass.
- The two `ERROR:` lines about the remote port `127.0.0.1:0` are expected.

## Test Naming

Names follow the engine's language (`.claude/rules/test-standards.md`):
- File `[system]_[feature]_test.gd`, function `test_[scenario]_[expected]`
  — `combat_damage_test.gd` → `test_base_attack_returns_expected_damage()`

## Story Type → Test Evidence

| Story Type | Required Evidence | Location |
|---|---|---|
| Logic | Automated unit test — must pass | `tests/unit/[system]/` |
| Integration | Integration test OR playtest doc | `tests/integration/[system]/` |
| Visual/Feel | Screenshot + lead sign-off | `production/qa/evidence/` |
| UI | Retained screenshot of each screen touched | `production/qa/evidence/` |
| Config/Data | Smoke check pass | `production/qa/smoke-*.md` |

## CI

Tests run automatically on every push to `main` and on every pull request.
A failed test suite blocks merging.
