# lib/multitask — AI Context

## Purpose

Thin wrapper over FANUC Karel's built-in task management system calls (`RUN_TASK`, `GET_TSK_INFO`, `PAUSE_TASK`, `ABORT_TASK`). Provides a named, error-checked API for spawning concurrent Karel programs, querying their status, joining them, and terminating them. Sits at Layer 5 in the Ka-Boost stack — depends only on `errors` and `ktransw-macros`. Used by `sensors`, `csv`, and `paths` modules.

---

## Repository Layout

```
lib/multitask/
├── package.json               # rossum manifest: name=multitask, depends=[errors, ktransw-macros]
├── readme.md                  # minimal stub (replaced by this)
├── include/
│   ├── multitask.klh          # function declarations via declare_function macros
│   └── multitask.klt          # constants: MAX_CHECK_DELAY=100, MAX_STATUS_CHECKS=20
└── src/
    └── multitask.kl           # full implementation, 162 lines
```

No test folder. Tests exercising this module live in `lib/csv/test/write_test/` and `lib/sensors/tof/test/`.

---

## Full API Reference

### Header: `include/multitask.klh`

All routines are declared with `declare_function` using `prog_name=task`, `prog_name_alias=task`. Both long and short aliases resolve to the same symbol. Short aliases are: `thread`, `thdmtn`, `wait`, `isdone`, `isrun`, `ispar`, `abort`, `abrtmtn`.

| Routine | Signature | Returns | FANUC Built-in |
|---------|-----------|---------|----------------|
| `task__thread` | `(task_name: STRING): BOOLEAN` | TRUE = spawned OK | `RUN_TASK(..., 0, FALSE, FALSE, 0, status)` |
| `task__thread_motion` | `(task_name: STRING; grp_mask: INTEGER): BOOLEAN` | TRUE = spawned OK | `RUN_TASK(..., 0, FALSE, TRUE, grp_mask, status)` |
| `task__wait` | `(task_name: STRING): BOOLEAN` | TRUE = completed within timeout | `GET_TSK_INFO` + `PAUSE_TASK` |
| `task__is_task_done` | `(task_name: STRING): BOOLEAN` | TRUE if `PG_ABORTED` or `PG_ABORTING` | `GET_TSK_INFO(..., TSK_STATUS, ...)` |
| `task__is_task_running` | `(task_name: STRING): BOOLEAN` | TRUE if `PG_RUNNING` | `GET_TSK_INFO(..., TSK_STATUS, ...)` |
| `task__is_parent_of` | `(task_name: STRING): BOOLEAN` | TRUE if task has a parent (is a child task) | `GET_TSK_INFO(..., TSK_PARENT, ...)` |
| `task__abort` | `(task_name: STRING): BOOLEAN` | TRUE = abort OK | `ABORT_TASK(..., TRUE, FALSE, status)` |
| `task__abort_motion` | `(task_name: STRING): BOOLEAN` | TRUE = abort OK | `ABORT_TASK(..., TRUE, TRUE, status)` |

### Constants: `include/multitask.klt`

```
MAX_CHECK_DELAY   = 100    -- poll interval in ms for task__wait
MAX_STATUS_CHECKS = 20     -- max polls before PROGRAM_TIMEDOUT error
                           -- total timeout = 100 * 20 = 2000 ms = 2 seconds
```

---

## Core Patterns

### Pattern 1: Spawn-and-Forget

Spawn a background task with no motion rights. Fire and forget — no join.

```karel
%include multitask.klh

VAR started : BOOLEAN

started = task__thread('MY_WORKER')
IF NOT started THEN
  -- karelError already posted a warning; handle or continue
ENDIF
```

Real usage: `lib/paths/pathforms/lib/pathform.blocks.klt` line 253:
```karel
IF NOT task__thread('SCANNING_OBJECT_NAME') THEN
```

---

### Pattern 2: Spawn-Check-Abort (Guard Against Double-Start)

Check if task is already running before spawning it. Abort at cleanup.

```karel
%include multitask.klh

VAR tof_task : BOOLEAN

IF NOT task__is_task_running('SENSOR_CLASS') THEN
  tof_task = task__thread('SENSOR_CLASS')
ENDIF

-- ... work ...

tof_task = task__abort('SENSOR_CLASS')
```

Real usage: `lib/sensors/tof/include/scan_class/scan_part_dyn.klc` lines 140–142, 337.

---

### Pattern 3: Spawn Multiple Tasks, Poll Completion, Abort All

Used in test harnesses to run concurrent operations and synchronize.

```karel
%include multitask.klh

VAR
  strt : BOOLEAN
  done : BOOLEAN

strt = task__thread('tst_csv_writ')
strt = task__thread('tst_csv_pipe')

-- poll until pipe task is done
REPEAT
  done = task__is_task_done('tst_csv_pipe')
UNTIL(done)

strt = task__abort('tst_csv_writ')
```

Real usage: `lib/csv/test/write_test/tst_csv_main.kl` lines 60–78.

---

### Pattern 4: Spawn with Motion Rights (Multi-Group)

Spawn a task that can issue robot motion commands. `grp_mask` selects which motion groups the task controls.

```karel
%include multitask.klh

VAR ok : BOOLEAN

-- Spawn task with access to GP1 only (mask = 1)
ok = task__thread_motion('MY_MOTION_TASK', 1)

-- Spawn task with access to GP1 + GP2 (mask = 3)
ok = task__thread_motion('MY_DUAL_TASK', 3)
```

Group mask encoding (binary bits, rightmost = GP1):
```
1  = GP1 only        (0001)
2  = GP2 only        (0100 per comments, decimal 2)
3  = GP1 + GP2
4  = GP3 only
5  = GP1 + GP3
7  = GP1 + GP2 + GP3
8  = GP4 only
```

---

### Pattern 5: Spawn + Join (Blocking Wait)

`task__wait` polls every 100ms up to 2 seconds then aborts if the task hasn't finished.

```karel
%include multitask.klh

VAR ok : BOOLEAN

ok = task__thread('MY_SUBTASK')
IF ok THEN
  ok = task__wait('MY_SUBTASK')
  -- execution resumes here only after MY_SUBTASK finishes or times out
ENDIF
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Spawning an already-running task | `RUN_TASK` returns non-zero status; `karelError` posts warning | Call `task__is_task_running` before spawning |
| Using `task__abort` when task is controlling motion | Robot motion continues orphaned after task abort | Use `task__abort_motion` to also cancel in-progress moves |
| `task__wait` timeout too short | `PROGRAM_TIMEDOUT` error after 2s even though task is still valid | Increase `MAX_STATUS_CHECKS` in `multitask.klt` and rebuild |
| Calling `task__is_task_done` instead of `task__is_task_running` | Logic inverted — done returns TRUE on `PG_ABORTED`, not on clean finish | Use `task__is_task_running` to check active state; `is_task_done` checks aborted state only |
| Task name longer than STRING buffer | Karel string truncation silently corrupts the name | Keep task names ≤ 12 chars (FANUC program name limit) |
| Forgetting `%NOLOCKGROUP` in spawned task | Task acquisition fails for groups not in mask | All non-motion tasks need `%NOLOCKGROUP`; motion tasks need correct group config |

---

## Dependencies

**This module depends on:**
- `errors` (layer 1) — `karelError()`, `CHK_STAT`, error code constants (`ER_WARN`, `ER_ABORT`, `PROGRAM_TIMEDOUT`)
- `ktransw-macros` (layer 0) — `declare_function`, `namespace.m`

**Modules that depend on this:**
- `sensors` (layer 6) — TOF scan classes spawn/abort sensor reading tasks
- `csv` (layer 4) — test harness uses concurrent task spawning
- `paths` (layer 7) — `pathforms` spawns scanning tasks during path setup

---

## Build / Integration Notes

- Single source file: `src/multitask.kl`. No templates, no GPP expansion required.
- Header guards in `.klh` use `%ifndef source_h` — if including multiple modules that also use `source_h` guard name, conflicts are possible. In practice only `multitask.klh` uses this name.
- `%NOLOCKGROUP` in `multitask.kl` — the controller library itself holds no motion groups. Spawned tasks are responsible for their own group declarations.
- `%NOPAUSE = COMMAND + TPENABLE + ERROR` — the multitask program cannot be paused from the teach pendant.
- Compiled output: `multitask.pc`. No TP programs generated by this module.
- No `.tpp` or TP-interfaces defined. All calls are Karel-to-Karel only.
