# Project Status

**Project:** shiny-robotics  
**Last updated:** 2026-11-27  
**Status:** Early development / architecture definition

## 1. Project overview

shiny-robotics is an educational robotics platform intended to let children assemble simple robots and program their behaviour through an approachable visual programming environment.

The initial platform is based on Raspberry Pi Pico running MicroPython, with a Raspberry Pi Zero acting as a central programming and communication hub. The longer-term goal is to support multiple independently controlled robots.

## 2. Goals

- Build an affordable, modular and accessible educational robotics platform.
- Keep hardware control, program execution and communication separate.
- Make the system understandable and maintainable.
- Provide a visual programming interface, eventually based on Blockly.
- Support multiple programs and parallel execution branches.
- Use non-blocking waits so one task does not stop other work.
- Keep dependencies lightweight and prefer incremental prototypes.

## 3. Hardware direction

### Robot

- Original Raspberry Pi Pico (non-wireless model).
- DC motors and motor driver circuitry.
- nRF24L01+ radio module for wireless communication.
- Battery-powered operation; the current battery concept is a four-cell rechargeable AA pack.

### Central hub

- Raspberry Pi Zero.
- Intended to host the programming interface and coordinate communication with robots.

### Development equipment and tests

- MicroPython firmware has been installed on the Pico.
- The Pico has been accessed from Linux over USB serial (`/dev/ttyACM0`).
- `mpremote` is installed and working in the CLI-based development workflow.
- Basic MicroPython execution and serial output have been tested.
- Discrete H-bridge motor-control circuitry has been explored on a breadboard. This is experimental and should not be treated as a finalized motor-driver design.

## 4. Software status

| Area | Current status | Notes |
|---|---|---|
| Project repository | Started | GitHub repository: https://github.com/wiseby/shiny-robotics |
| Blockly application | Present in repository | `blockly-app/` exists; its current functionality has not yet been assessed in this status document. |
| Pico MicroPython setup | Working baseline | USB serial access and `mpremote` workflow established. |
| Hardware abstraction | Not yet specified | Keep hardware-specific code isolated behind small interfaces. |
| Canonical program format | Decision pending detailed specification | JSON is the proposed initial format. |
| UART message protocol | Planned | JSONL is the proposed development/debugging transport. |
| Radio protocol | Planned | Compact binary encoding is expected because nRF24L01+ payloads are limited to 32 bytes. |
| Command manager | Not yet implemented/specified | Must support independent tasks, parallel branches and non-blocking waits. |
| Blockly integration | Early / not yet specified | Blockly should generate the canonical program representation rather than directly control hardware. |
| Testing and integration | Early stage | Continue with small independently testable components. |

## 5. Agreed architectural direction

The following are current design intentions, not claims that the implementation is complete:

1. **Separate program representation from transport.** The same logical program and commands should work whether the robot is connected over USB/UART or radio.
2. **Use versioned JSON as the initial canonical program format.** The exact schema is still to be designed and agreed.
3. **Use JSONL over UART during early development.** One JSON message per line makes serial debugging straightforward.
4. **Plan a compact binary radio encoding.** Preserve the logical message model while changing the wire encoding for the radio.
5. **Include message identifiers and acknowledgements.** Responses should be correlatable with requests and report success or failure.
6. **Keep execution non-blocking.** A wait should suspend only the relevant task/branch, not the whole command manager.
7. **Keep hardware details behind abstractions.** Program commands should describe intent (for example, setting motor speed), not GPIO pin operations.

## 6. Immediate next steps

### Architecture and format

- [ ] Define the canonical program document structure and versioning rules.
- [ ] Define identifiers for programs, tasks, steps and parallel branches.
- [ ] Define the initial command vocabulary and argument types.
- [ ] Define parallel-branch completion semantics (for example, wait for all branches).
- [ ] Define non-blocking wait behaviour.
- [ ] Define execution states, cancellation and error reporting.
- [ ] Define JSONL request/response messages, sequence identifiers and ACK/NACK behaviour.
- [ ] Record the agreed specification in the GitHub repository.

### Firmware

- [ ] Establish a minimal, reproducible Pico project layout.
- [ ] Create and test small hardware drivers independently.
- [ ] Select and validate the motor driver and power arrangement before connecting motors to the final robot.
- [ ] Implement the command manager only after its behaviour and data contract are agreed.

### Hub and visual programming

- [ ] Review the existing `blockly-app/` implementation.
- [ ] Define how the hub stores and sends programs.
- [ ] Define how Blockly output maps to the canonical program format.
- [ ] Prototype communication with one robot before expanding to multiple robots.

## 7. Development workflow

- Develop locally on Linux using a CLI-first workflow.
- Use `mpremote` for Pico interaction and deployment.
- Test components independently before integration.
- Use Git commits to preserve working milestones.
- Update this file when meaningful project status changes occur.
- Record durable architectural decisions and specifications in the repository, not only in ChatGPT conversations.

## 8. Source of truth

The GitHub repository is the authoritative home for source code, specifications and project documentation.

ChatGPT is used as a discussion, design and troubleshooting workspace. Important decisions made in conversations should be transferred into the repository once agreed.

Repository: https://github.com/wiseby/shiny-robotics
