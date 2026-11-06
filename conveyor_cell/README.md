# Conveyor cell — two robots, one conveyor (OmniCore / RobotWare 7)

Two robots share one conveyor and take turns. Robot 1 picks part 1, robot 2 picks part 2, and so on. Each robot puts its parts on its own 3 × 2 pallet. Only the robot whose turn it is enters the pick zone, so the two arms never meet there.

## Files

| File | Task | Content |
| --- | --- | --- |
| `Cell_R1.modx` | `T_ROB1` | Robot 1 program. It also resets the shared data and starts the conveyor |
| `Cell_R2.modx` | `T_ROB2` | Robot 2 program, a twin of Cell_R1 |

## Requirements

- A **MultiMove** system with two motion tasks, `T_ROB1` and `T_ROB2` (independent motion; no `SyncMoveOn` needed).
- These I/O signals, created in **I/O Engineering**:

| Signal | Type | Meaning |
| --- | --- | --- |
| `doConvRun` | DO | 1 = conveyor runs |
| `diPartAtPick` | DI | 1 = a part is at the pick point |
| `doGrip1` | DO | 1 = gripper of R1 closed |
| `doGrip2` | DO | 1 = gripper of R2 closed |

- In simulation, connect these signals to a conveyor Smart Component and a sensor in the station.

## One part, step by step

```
robot whose turn it is
 ├─ wait diPartAtPick = 1 → conveyor OFF
 ├─ approach → pick → close gripper → lift
 ├─ move to pClear (out of the pick zone)
 ├─ hand over: nTurn := other robot, conveyor ON
 │    (or bCellDone := TRUE when nBatch parts are picked)
 └─ place in the next pallet slot → back to pClear
```

At the start, both tasks wait for each other at `WaitSyncTask` (`syncStart`). At the end, when `nBatch` parts are done, both robots go home and wait for each other at `syncEnd`.

## Shared data

These `PERS` variables are declared identically in both modules, so both tasks share them:

- `nTurn`: whose turn it is to pick (1 or 2)
- `nPartsTotal`: parts picked by the whole cell
- `bCellDone`: TRUE when the batch is finished
- `tlRobots`: the two tasks that start together

Only the robot holding the turn writes `nTurn`, `nPartsTotal`, `bCellDone` and `doConvRun`, so the two tasks never write them at the same time.

## Placeholders — teach these before running

These values are only examples:

- `jHome`
- `pPick`, `pClear`, `pPlace0` — **teach them in each robot's own `wobj0`**
- `tGripper1` / `tGripper2` (TCP and load)
- the pallet pitch

`pClear` must be outside the pick zone, so the other robot can safely come in while this robot waits there.

## Run it

1. Load `Cell_R1.modx` into `T_ROB1` and `Cell_R2.modx` into `T_ROB2`, then click **Apply**.
2. Set the program pointer to `main` in **both** tasks, then press **Start**.
3. For the first run on a real robot, use `v200`, manual mode and reduced speed.
