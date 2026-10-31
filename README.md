# ABB RAPID Demo — Path, Circle, Home & Logging

A small RAPID program for an ABB robot on an **OmniCore controller (RobotWare 7)**, built and tested in **RobotStudio**.

Each cycle, the robot:

1. moves to a safe **home position**;
2. follows a **triangular path** P0 → P1 → P2 → P0, stopping 1 s on each point;
3. draws a **full circle** (50 mm radius) around P0;
4. returns **home**.

Every step is **logged** with a timestamp: on screen, in a log file on the controller, and optionally in the event log.

---

## Files

| File | Content |
| --- | --- |
| `Module1.modx` | Main program: settings, targets, path, circle, home position, logging |
| `CalibData.modx` | Tool definition `MyTool` (TCP and load data) |
| `README.md` | This file |

---

## Requirements

- RobotStudio with a **RobotWare 7 (OmniCore)** virtual controller, or a real OmniCore controller.
- Use the **same RobotStudio and RobotWare versions** as your lab. Stations are not forward-compatible: a station opened in an older RobotStudio than the one that created it can lose data.
- The robot station whose targets P0, P1 and P2 were taught in. They are expressed in `wobj0` (robot base) with the tool `MyTool`.

---

## Quick start

1. **Load the modules.** In RobotStudio, open the RAPID tab, then:
   - load `CalibData.modx` and `Module1.modx` into task `T_ROB1`, or paste their contents into the existing modules;
   - click **Apply**.
2. **Teach the home position** (see [Home position](#home-position)). The default `jHome` values are only a placeholder.
3. **Turn on the Operator Window** to see the log: **File → Options → Robotics → Virtual Controller → "Show virtual Operator Window"**.
4. **Run in simulation.** Set the program pointer to `main`, then press **Start** (RAPID tab) or **Play** (Simulation tab).
   - Do **not** use *Move Along Path*: it only moves the 3D model and does not run the RAPID code.
5. Check the log and the 3D motion. Then, and only then, go to the real robot (see [Before running on the real robot](#before-running-on-the-real-robot)).

---

## Program flow

```
main
 ├─ GoHome                     MoveAbsJ to jHome
 ├─ Path_10                    P0 → P1 → P2 → P0, 1 s dwell at each point
 ├─ DrawCircle P0, 50 mm       two MoveC half-circles around P0
 ├─ GoHome
 └─ log cycle time
```

---

## Settings

All settings are at the top of `Module1`.

| Name | Default | Meaning |
| --- | --- | --- |
| `nWaitTime` | `1` | Dwell time at each point, in seconds |
| `vRun` | `v1000` | Speed along the path. Use `v200` for first tests |
| `vHome` | `v300` | Speed when going home from an unknown position |
| `nRadius` | `50` | Circle radius, in mm |
| `vCircle` | `v200` | Speed along the circle |
| `bEchoToEventLog` | `TRUE` | Also write each log line to the event log (Output window) |
| `sLogFile` | `"path_log.txt"` | Log file name, in the controller's `HOME:` folder |
| `jHome` | `[0,0,0,0,30,0]` | Home position as joint angles (**placeholder — teach your own**) |
| `nCycle` | `0` | Cycle counter (`PERS`, kept between runs; set to 0 to reset) |

---

## Routines

| Routine | What it does |
| --- | --- |
| `main` | Runs one cycle, measures cycle time, logs any recoverable error and stops |
| `Path_10` | Triangular path with `fine` stop points and `WaitTime\InPos` dwell |
| `DrawCircle(pCenter, nR, vSpeed)` | Full horizontal circle around `pCenter` with radius `nR` |
| `GoHome` | Moves to `jHome` with `MoveAbsJ` |
| `LogJoints` | Logs the current joint angles, ready to paste into `jHome` |
| `LogPos(sName)` | Logs the actual TCP position (x, y, z) after a move |
| `LogMsg(sMsg)` | Writes one time-stamped line to screen, file and (optionally) event log |

---

## Key techniques

### Dwell at each point

```rapid
MoveL P1,vRun,fine,MyTool\WObj:=wobj0;
WaitTime\InPos,nWaitTime;
```

- `fine` makes the robot stop exactly on the point. With a zone such as `z100`, it would round the corner and never reach the point.
- `\InPos` starts the timer only once the robot has fully stopped.

### Circle

A single `MoveC` cannot draw a full circle, so `DrawCircle` uses two half circles:

- `Offs(pCenter, x, y, z)` computes four points at 0°, 90°, 180° and 270° around the center, in the `wobj0` XY plane (horizontal), keeping the center's tool orientation.
- `MoveC p90deg, p180deg` draws the first half; `MoveC p270deg, p0deg` draws the second.
- `z10` between the two halves keeps the motion continuous.
- `ConfL\Off` lets the robot pick the nearest arm configuration for the computed points. It is switched back `\On` afterwards, including on error.

**Customize it:**
- Other size: change `nRadius`.
- Other place: pass another center, for example `DrawCircle P1,nRadius,vCircle;`.
- Vertical circle (XZ plane): use the radius in the z offset instead of y: `Offs(pCenter,0,0,nR)` and `Offs(pCenter,0,0,-nR)`.

### Home position

`GoHome` uses `MoveAbsJ` with a `jointtarget`. Each axis goes to an exact angle, so the arm always ends in the same pose whatever tool, work object or configuration is active.

**Teach your own home pose:**

1. Jog the robot to a safe pose: clear of the table and fixtures, with axis 5 **not** at 0° (wrist singularity). In RobotStudio, right-click the robot and choose **Mechanism Joint Jog**.
2. Set the program pointer to `LogJoints` and run it. It logs a line such as `Joints [-25.0,10.5,15.2,0.0,64.3,-25.0]`.
3. Paste these values into `jHome`, keeping the `9E+09` part (no external axes), then **Apply**.

### Logging

`LogMsg` writes each line to three places:

| Where | How to see it |
| --- | --- |
| Operator Window (`TPWrite`) | RobotStudio Operator Window (enable it in Options), or the FlexPendant on a real controller |
| Log file `HOME:/path_log.txt` | Download from the controller, or open the virtual controller's `HOME` folder |
| Event log (`ErrWrite\I`) | RobotStudio **Output** window. Set `bEchoToEventLog := FALSE` once you no longer need it |

Example log:

```
2026-10-31 14:32:01 | Cycle 3 started
2026-10-31 14:32:01 | Going home
2026-10-31 14:32:04 | At home
2026-10-31 14:32:06 | At P0 x=403.5 y=-188.7 z=508.5
...
2026-10-31 14:32:21 | Cycle 3 done in 19.84 s
```

If the log file cannot be written, `LogMsg` skips it and the robot keeps running.

---

## Tool data (`CalibData`)

```rapid
PERS tooldata MyTool:=[TRUE,[[31.792631019,0,229.638935148],[0.945518576,0,0.325568154,0]],[1,[0,0,1],[1,0,0,0],0,0,0]];
```

| Part | Value | Meaning |
| --- | --- | --- |
| `robhold` | `TRUE` | The robot holds the tool |
| TCP position | `[31.8, 0, 229.6]` mm | Tool tip, measured from the flange center |
| TCP orientation | quaternion | Tool frame rotated 38° about the flange Y axis |
| Load | 1 kg, CoG at `[0,0,1]` | Placeholder values |

All targets (P0, P1, P2, circle points) are positions of this TCP, not of the flange.

---

## Before running on the real robot

- [ ] Same RobotStudio / RobotWare versions as the lab controller.
- [ ] **TCP measured** on the real tool (FlexPendant tool calibration: 4-point, or TCP & Z, X for orientation). For a pen, the TCP is the pen tip.
- [ ] **Load data set** to the real tool mass and center of gravity (or run load identification). Wrong load data reduces path accuracy and affects collision detection.
- [ ] **`jHome` taught** on the real cell, and the move to it checked for collisions.
- [ ] `vRun` lowered to `v200` (or less) for the first run.
- [ ] First run in **manual mode at reduced speed**, hand near the stop button.

---

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| No log messages visible | Enable **"Show virtual Operator Window"** in Options; check the **Output** window (event log echo); make sure RAPID is actually running, not *Move Along Path*. Messages can also appear late, in a batch |
| Changes have no effect | Click **Apply** in the RAPID editor, then reset the program pointer to `main` |
| "Position outside reach" on the circle | P0 is about 445 mm from the robot base. Reduce `nRadius` or use a center closer to the robot |
| Configuration error on a linear or circular move | Teach the target again with a reachable configuration, or use `ConfL\Off` around computed targets (as in `DrawCircle`) |
| Robot doesn't stop on the points | Use `fine` instead of a zone (`z10`, `z100`...) on that move |
| Error not caught by `main`'s handler | Some motion errors stop execution directly. Check the event log |

---

## Limitations

- `jHome` and the tool load data are placeholders until taught or measured on the real cell.
- The circle is always in a plane of `wobj0`. For drawing on a tilted canvas, define a work object on the canvas and pass it instead of `wobj0`.
- Targets are `CONST`. To retouch them on the FlexPendant, use *Modify Position*, or change them to `PERS`.
