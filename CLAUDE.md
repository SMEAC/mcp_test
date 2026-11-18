# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

This is not a code project. It is a workspace for **using** two sibling MCP servers together on a hardware bench. It contains [.mcp.json](.mcp.json) and per-scenario test data in [TestData/](TestData/). The `.mcp.json` launches:

| Server | Source | Controls |
|---|---|---|
| `ur3e` | `C:\dev\mcp_stuff\mcp_ur3e` | A UR3e robot arm that holds an oscilloscope probe and places it on named PCB test points |
| `msox2014a` | `C:\dev\mcp_stuff\mcp_msox2014a` | A Keysight MSO-X 2014A oscilloscope over VISA (USB resource set in `MSOX_RESOURCE`) |

The servers work as a pair: `ur3e` puts the probe on a test point, then `msox2014a` measures there. Code changes belong in the sibling repos, not here. Each one has its own README, `pyproject.toml` and tests.

## Commands (run in the sibling server repos)

Both are Python 3.11+ projects managed with `uv`:

```powershell
cd C:\dev\mcp_stuff\mcp_ur3e          # or mcp_msox2014a
uv sync
uv run pytest                          # all tests
uv run pytest tests/test_x.py::test_y  # single test
uv run ur3e-mcp --simulate --board example_board   # stdio smoke run, Ctrl+C to exit
uv run msox2014a-mcp --simulate
```

`msox2014a` pins uv to a uv-managed Python. The Microsoft Store Python cannot load vendor VISA DLLs and fails with `VI_ERROR_LIBRARY_NFOUND`.

## Measurement workflow

1. `robot_device(operation='connect', host=...)`: use `'SIM'` to simulate. The real robot is at `UR_HOST` (192.168.1.20).
2. `robot_board(operation='load', file=...)`: each scenario's board file (test points in board mm) is kept with its PCB and schematic files in `TestData/<Scenario>/`, e.g. `TestData/Scenario1/scenario1_opamp_board.json`. `UR_BOARDS_DIR` in `.mcp.json` points the server at `TestData`, and sub-folders are searched, so `file='scenario1_opamp_board'` is enough. `robot_board(operation='list_files')` lists them all. The default is `example_board`, from `mcp_ur3e/boards/`, which is always searched last. Teach updates (`record_point`, `compute_frame`) are written back to the file that was loaded, in its scenario folder.
3. `robot_move(operation='to_point', point='TP4')`: this returns only after the robot has stopped with the probe on the pad, so a scope capture can follow at once.
4. Measure with the `scope_*` tools, then move to the next point, and finally run `robot_move(operation='home')`.

Each move retracts to the clearance plane, travels, descends and then probes slowly. All four steps go to the robot as one URScript program.

## Safety with the real robot

- Before moving the physical robot to a point or board that has not been used yet, run `dry_run=True` first and show the planned motion to the user.
- `robot_move(operation='stop')` is **not** a safety function. The e-stop on the teach pendant is.
- Do not move the real robot or change scope/WaveGen output unless the user asks for it. Both are physical actions on bench hardware.
