# multitask

Helper functions for concurrent programming in FANUC Karel.

## Overview

Karel programs can run concurrently as separate controller tasks, but the native built-ins (`RUN_TASK`, `GET_TSK_INFO`, `PAUSE_TASK`, `ABORT_TASK`) have unwieldy signatures and no error reporting. This module wraps them in a clean, consistently named API with automatic `karelError` posting on failures.

Reach for this module whenever you need to:
- Spawn a background Karel program (e.g., a sensor reader, a motion task)
- Poll whether a concurrent task has started or finished
- Join a task (blocking wait with timeout)
- Abort a task, with or without cancelling its in-progress robot motion

---

## Files

| File | Purpose |
|------|---------|
| `src/multitask.kl` | Full implementation — 8 routines wrapping `RUN_TASK`, `GET_TSK_INFO`, `PAUSE_TASK`, `ABORT_TASK` |
| `include/multitask.klh` | Function declarations via `declare_function` macros; include this in any caller |
| `include/multitask.klt` | Constants: `MAX_CHECK_DELAY` (100 ms), `MAX_STATUS_CHECKS` (20 polls = 2s timeout) |
| `package.json` | rossum manifest: `multitask` v0.0.1, depends on `errors` and `ktransw-macros` |

---

## API Reference

Include `multitask.klh` in any file that calls these routines.

### Spawning Tasks

```karel
task__thread(task_name : STRING) : BOOLEAN
```
Spawns `task_name` as a concurrent Karel task with no motion group access. Returns `TRUE` on success. On failure, posts a `ER_WARN` error via `karelError` and returns `FALSE`.

```karel
task__thread_motion(task_name : STRING; grp_mask : INTEGER) : BOOLEAN
```
Same as `task__thread`, but grants motion rights to the specified robot groups. `grp_mask` is a decimal integer encoding binary group bits:

| `grp_mask` | Groups |
|-----------|--------|
| `1` | GP1 only |
| `2` | GP2 only |
| `3` | GP1 + GP2 |
| `4` | GP3 only |
| `5` | GP1 + GP3 |
| `7` | GP1 + GP2 + GP3 |
| `8` | GP4 only |

### Querying Task Status

```karel
task__is_task_running(task_name : STRING) : BOOLEAN
```
Returns `TRUE` if the task's status is `PG_RUNNING`. Use this to guard against double-spawning.

```karel
task__is_task_done(task_name : STRING) : BOOLEAN
```
Returns `TRUE` if the task's status is `PG_ABORTED` or `PG_ABORTING`. Note: this returns `TRUE` on abnormal termination, not on normal completion. For polling completion in a loop, use this in combination with `task__is_task_running`.

```karel
task__is_parent_of(task_name : STRING) : BOOLEAN
```
Returns `TRUE` if `task_name` has a parent task (i.e., it was spawned by another task). Useful for detecting whether code is running as a spawned child or as the top-level program.

### Joining (Waiting)

```karel
task__wait(task_name : STRING) : BOOLEAN
```
Polls the task every `MAX_CHECK_DELAY` (100 ms) until it stops running, up to `MAX_STATUS_CHECKS` (20) times — a **2-second total timeout**. Once the task finishes, calls `PAUSE_TASK` to clean up. On timeout, posts an `ER_ABORT` error with `PROGRAM_TIMEDOUT`. Returns `TRUE` on success.

To increase the timeout, change `MAX_STATUS_CHECKS` in `include/multitask.klt` and rebuild.

### Aborting Tasks

```karel
task__abort(task_name : STRING) : BOOLEAN
```
Terminates `task_name`. Does not affect any robot motion the task may have issued. Returns `TRUE` on success.

```karel
task__abort_motion(task_name : STRING) : BOOLEAN
```
Terminates `task_name` **and** cancels any in-progress robot motion it was executing. Use this instead of `task__abort` when the task controls a motion group.

---

## Common Patterns

### 1. Spawn-and-Forget

Run a background task and continue without waiting for it.

```karel
%include multitask.klh

VAR started : BOOLEAN

started = task__thread('MY_WORKER')
-- continue; MY_WORKER runs concurrently
```

### 2. Guard Against Double-Start

Always check before spawning if the task might already be running.

```karel
%include multitask.klh

VAR tof_task : BOOLEAN

IF NOT task__is_task_running('SENSOR_CLASS') THEN
  tof_task = task__thread('SENSOR_CLASS')
ENDIF

-- ... do work while sensor task runs in background ...

-- clean up
tof_task = task__abort('SENSOR_CLASS')
```

### 3. Spawn Multiple Tasks, Poll One, Abort Another

Pattern used in test harnesses to run concurrent operations.

```karel
%include multitask.klh

VAR
  strt : BOOLEAN
  done : BOOLEAN

strt = task__thread('TASK_WRITER')
strt = task__thread('TASK_PIPE')

-- Wait for pipe task to complete
REPEAT
  done = task__is_task_done('TASK_PIPE')
UNTIL(done)

-- Clean up writer
strt = task__abort('TASK_WRITER')
```

### 4. Spawn + Join (Blocking Wait)

Run a subtask to completion before continuing. Blocks for up to 2 seconds.

```karel
%include multitask.klh

VAR ok : BOOLEAN

ok = task__thread('MY_SUBTASK')
IF ok THEN
  ok = task__wait('MY_SUBTASK')
  -- resumes here after MY_SUBTASK finishes (or times out)
ENDIF
```

### 5. Motion Task for a Specific Robot Group

Spawn a task that will issue motion commands to GP1.

```karel
%include multitask.klh

VAR ok : BOOLEAN

ok = task__thread_motion('MY_MOTION_TASK', 1)
-- MY_MOTION_TASK can now call motion instructions on GP1
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Spawning a task that is already running | `RUN_TASK` fails; warning posted | Call `task__is_task_running` before `task__thread` |
| Using `task__abort` on a motion task | Robot motion continues after task terminates | Use `task__abort_motion` to also cancel in-progress moves |
| `task__wait` timing out too early | `PROGRAM_TIMEDOUT` abort error after 2s | Increase `MAX_STATUS_CHECKS` in `multitask.klt` and rebuild |
| Confusing `is_task_done` with "finished normally" | Logic inverted — `is_task_done` returns TRUE only on `PG_ABORTED` state | Use `is_task_running` to check active state; `is_task_done` signals abort/error |
| Task name too long | Karel silently truncates the name; task not found | FANUC program names max 12 characters |
| Spawned task missing `%NOLOCKGROUP` | Task fails to acquire groups not in its mask | Non-motion tasks need `%NOLOCKGROUP`; motion tasks need the correct group pragma |

---

## Build Flow

`multitask` is a plain Karel source module with no GPP templates.

```
rossum .. -w -o    # generates build.ninja with multitask as dependency
ninja              # compiles src/multitask.kl → multitask.pc
kpush              # deploys multitask.pc to the controller
```

Any module that calls multitask routines must list `"multitask"` in its `package.json` `depends` array and include `multitask.klh` in each file that uses it.

See the [Ka-Boost readme](../../readme.md) for full build and deployment instructions.
