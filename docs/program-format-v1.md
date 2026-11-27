# Shiny Robotics Program JSON — Version 1

**Status:** v1 specification  
**Format identifier:** `shiny-robotics`  
**Version:** `1`

This document defines the canonical representation of a Shiny Robotics robot program. It is the contract between the visual editor, the hub and robot firmware.

The format describes **robot behaviour**, not hardware wiring or editor state. It is independent of Blockly, MicroPython, the Raspberry Pi Pico and the communication transport.

## 1. Goals

The v1 format must:

- Be readable and straightforward to debug.
- Be lightweight to parse and validate on MicroPython.
- Represent multiple independent tasks.
- Represent sequential instructions and parallel branches.
- Support non-blocking waits.
- Be validated before execution.
- Remain independent of the editor and transport protocol.

The format should not encode GPIO pins, motor-driver details, radio addresses or other hardware implementation details.

## 2. Blockly workspace data is separate

There are two distinct JSON documents in the system:

1. **Blockly workspace JSON** stores the visual editor state: blocks, fields, connections and positions. It is used to save and restore the workspace.
2. **Shiny Robotics Program JSON** stores the robot's actual instructions. It is the canonical program representation consumed by the hub and execution system.

Blockly workspace serialization is **not** the robot program format and must not be sent to the robot as executable instructions. Blockly generates Shiny Robotics Program JSON from the workspace.

## 3. Top-level structure

A program has this structure:

```json
{
  "format": "shiny-robotics",
  "version": 1,
  "program_id": "first-robot",
  "tasks": []
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `format` | string | Must be `"shiny-robotics"`. |
| `version` | integer | Program format version. For this document, it must be `1`. |
| `program_id` | string | Identifier for this program. |
| `tasks` | array | One or more task definitions. |

A v1 program must contain all four fields. A receiver must reject an unknown format or unsupported version; it must not guess how to interpret it.

The format version describes the program schema. It is not the Blockly application version, firmware version or communication protocol version.

## 4. Tasks

A task is an independently scheduled sequence of steps.

```json
{
  "id": "main",
  "steps": []
}
```

- Each task has a non-empty string `id`.
- Task IDs must be unique within the program.
- `steps` is an ordered array of operations.
- An empty `steps` array is valid and means that the task completes immediately.

### Concurrent task execution

**Agreed v1 behaviour:** all tasks start when the program starts. Tasks progress independently under the execution scheduler. A task completing does not stop or wait for the other tasks.

The scheduler is cooperative: an operation such as `wait` must yield control rather than block the scheduler. Concurrency here describes program behaviour; it does not promise simultaneous CPU execution or hard real-time timing.

The program completes when all its tasks have completed.

## 5. Operations and sequential execution

Each step is an object with an `op` string identifying the operation. Operation-specific parameters are properties of the same object; v1 does not wrap them in a generic `parameters` object.

Steps in a task execute in array order. The next step is started when the current step has completed. Operations that start continuous actuator behaviour, such as a motor direction command, complete when the command has been issued; the motor may continue running until a later stop or direction command.

### v1 operation set

The initial v1 operation set is:

| Operation | Required properties | Meaning |
| --- | --- | --- |
| `motor.set_speed` | `motor`, `speed` | Set the configured speed of a logical motor. |
| `motor.forward` | `motor` | Set a logical motor to forward at its configured speed. |
| `motor.backward` | `motor` | Set a logical motor to backward at its configured speed. |
| `motor.stop` | `motor` | Stop a logical motor. |
| `wait` | `duration_ms` | Suspend the current execution context for a duration without blocking other contexts. |

For the motor operations:

- `motor` is a non-negative integer logical motor identifier. It is not a GPIO pin number.
- `speed` is an integer from 0 through 100 inclusive.
- Each motor's configured speed is initially 0.
- `motor.set_speed` updates the configured speed. If the motor is already running, the new speed takes effect immediately.
- Setting speed to 0 stops the motor's drive output. Its configured speed becomes 0, so a later direction command will not make it move until a positive speed is set.
- `motor.forward` and `motor.backward` select the direction and apply the motor's currently configured speed. They do not change that speed. If the configured speed is 0, the motor remains stopped.
- `motor.stop` stops the motor's drive output but does not change its configured speed. A later direction command therefore resumes at the previously configured speed.
- The exact electrical stop behaviour (for example, coasting or active braking) is hardware-specific and is not prescribed by the canonical program format.

For `wait`:

- `duration_ms` is a non-negative integer duration in milliseconds.
- A zero-duration wait completes immediately.

Direction and motor-to-pin mapping are handled by the hardware abstraction layer. The program does not contain pin assignments.

The initial example uses motor identifiers `0` and `1`. The available identifiers depend on the robot configuration and must be checked by the relevant validator/runtime.

Example:

```json
{
  "id": "main",
  "steps": [
    { "op": "motor.set_speed", "motor": 0, "speed": 60 },
    { "op": "motor.forward", "motor": 0 },
    { "op": "wait", "duration_ms": 1000 },
    { "op": "motor.stop", "motor": 0 }
  ]
}
```

### Motor operation sequence

The separate operations make speed and direction independently controllable. For example, a program can set a speed once, start the motor, change its speed while it is running, and then stop it:

```json
[
  { "op": "motor.set_speed", "motor": 0, "speed": 40 },
  { "op": "motor.forward", "motor": 0 },
  { "op": "wait", "duration_ms": 1000 },
  { "op": "motor.set_speed", "motor": 0, "speed": 75 },
  { "op": "wait", "duration_ms": 1000 },
  { "op": "motor.stop", "motor": 0 }
]
```

## 6. Parallel execution

The `parallel` operation starts multiple branches within the current task.

```json
{
  "op": "parallel",
  "branches": [
    { "steps": [] },
    { "steps": [] }
  ]
}
```

Each branch contains an ordered `steps` array and follows the same sequential execution rules as a task.

**Agreed v1 behaviour: fork and join.**

1. All branches start concurrently.
2. Each branch executes its own steps sequentially.
3. A waiting branch does not block other branches or tasks.
4. When a branch completes, the remaining branches continue.
5. The parent task resumes after **all** branches have completed.

For v1, a `parallel` operation must have at least two branches. Nested `parallel` operations are not supported.

## 7. Non-blocking waits

A `wait` operation suspends only the execution context that reaches it. It must not call a blocking sleep that prevents other tasks or parallel branches from progressing.

The execution engine should record a wake-up deadline using a monotonic tick source and return to the scheduler. It resumes the waiting context when the deadline is reached. On MicroPython, the `time.ticks_ms()`, `time.ticks_add()` and `time.ticks_diff()` family is suitable for this purpose.

The wait's JSON representation is the same in a task and in a parallel branch.

## 8. Validation

The complete program must be parsed and validated before any program instruction is executed. Validation should reject malformed or unsupported programs rather than silently ignoring unknown data.

At minimum, v1 validation checks:

- The top-level value is an object.
- Required top-level fields are present and have the expected types.
- `format` and `version` are supported.
- `program_id` is a non-empty string.
- `tasks` is an array containing at least one task.
- Every task has a non-empty string ID, a `steps` array and a unique ID.
- Every step is a recognised operation object.
- Required operation properties are present and have the correct types and ranges.
- Parallel operations have at least two branches, each with a valid `steps` array.
- Nested parallel operations are rejected.
- Logical motor identifiers are valid for the target robot configuration.

Unknown operations must be rejected. The validator should report useful context, including the program ID, task ID, step/branch location and a stable error code where possible.

The hub and firmware must use compatible operation definitions and validation rules. The hub should validate before sending a program; the execution endpoint must still reject invalid or unsupported input rather than trusting transport data.

## 9. Runtime errors and program termination

**Agreed v1 behaviour: runtime errors are global.** If any task or parallel branch encounters a runtime error, the entire program fails.

The runtime must:

1. Mark the program as failed.
2. Stop scheduling new instructions.
3. Cancel remaining task and branch execution.
4. Make a best-effort attempt to stop all motor outputs, independently of normal program instructions.
5. Report the failure to the hub with useful context, including program ID, task ID, step/branch location and error code.

A failure to stop one output must not prevent attempts to stop the others.

Suggested externally visible program states:

| State | Meaning |
| --- | --- |
| `READY` | Validated and ready to start. |
| `RUNNING` | Program is executing. |
| `COMPLETED` | All tasks completed successfully. |
| `ERROR` | A runtime error caused global failure. |
| `CANCELLED` | Program was explicitly stopped externally. |

An explicit stop/cancel request is a runtime control action, not a program operation in this v1 JSON format.

## 10. Shared hardware and actuator ownership

Tasks and parallel branches can run concurrently, but v1 does not define locks, priorities or automatic arbitration for conflicting commands to the same actuator.

A program author must avoid having concurrent execution contexts issue conflicting commands to the same motor or other actuator. Programs that do so have undefined behaviour for that actuator.

This keeps the v1 scheduler small. Resource arbitration can be considered later if actual programs require it.

## 11. Transport independence

The canonical program JSON is independent of how it is delivered.

During development, JSON Lines (JSONL) over USB serial/UART can carry program messages. Later, the hub may package the same canonical program representation into a compact binary nRF24L01+ protocol.

Transport framing, message identifiers, acknowledgements, retries and communication error codes belong to the communication protocol, not to this program schema.

## 12. Deliberately out of scope for v1

The following features are not part of this initial specification:

- Loops and conditional statements.
- Variables, expressions and user-defined functions.
- Sensor operations and event handlers.
- Nested parallel operations.
- Dynamic task creation or task priorities.
- Individual branch cancellation or error recovery.
- Automatic actuator locks or resource arbitration.

They may be added in a later format version when there is a concrete requirement.

## 13. Complete illustrative example

This example demonstrates concurrent tasks, sequential steps, a fork-and-join parallel operation and non-blocking waits. The operation set is limited to the v1 operations above.

```json
{
  "format": "shiny-robotics",
  "version": 1,
  "program_id": "first-robot",
  "tasks": [
    {
      "id": "main",
      "steps": [
        {
          "op": "parallel",
          "branches": [
            {
              "steps": [
                { "op": "motor.set_speed", "motor": 0, "speed": 60 },
                { "op": "motor.forward", "motor": 0 },
                { "op": "wait", "duration_ms": 2000 },
                { "op": "motor.stop", "motor": 0 }
              ]
            },
            {
              "steps": [
                { "op": "wait", "duration_ms": 500 },
                { "op": "wait", "duration_ms": 500 }
              ]
            }
          ]
        }
      ]
    },
    {
      "id": "secondary",
      "steps": [
        { "op": "wait", "duration_ms": 1000 },
        { "op": "motor.stop", "motor": 1 }
      ]
    }
  ]
}
```

In this example, both tasks start together. The `main` task waits at its parallel operation until both branches finish. The `secondary` task progresses independently. The program completes when both tasks have completed.

---

## Change policy

This document is the reference for the v1 program format. Changes that alter the meaning or structure of existing v1 programs require careful compatibility consideration and may require a new format version. Implementation-specific details belong in architecture or firmware documentation, not in the canonical program document.
