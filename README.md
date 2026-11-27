# shiny-robotics

An educational robotics platform built around the Raspberry Pi Pico and MicroPython.

shiny-robotics aims to make robotics accessible in educational environments. The goal is for children to assemble simple robots and program their behaviour through an intuitive visual programming interface.

> **Project stage:** Early development. Hardware experiments and the MicroPython development workflow are underway; the program format, execution engine and communication protocols are still being designed.

## Project goals

- Affordable, modular and accessible educational robots.
- Simple hardware that children can assemble and understand.
- A visual programming environment, planned around Blockly.
- Multiple robots controlled by a central programming hub.
- A small, readable MicroPython firmware running on each robot.
- An architecture that separates programs, execution, communication and hardware.

## Planned system

The intended system has three main parts:

1. **Robot controller** – an original Raspberry Pi Pico running MicroPython, connected to motor drivers and other peripherals.
2. **Communication** – USB/UART during development and nRF24L01+ radio for wireless communication.
3. **Central hub** – a Raspberry Pi Zero intended to host the programming environment and coordinate connected robots.

The design is intended to support multiple independent programs and parallel execution branches. Waiting should be non-blocking, so one branch can wait while other work continues.

## Hardware

### Robot

- Raspberry Pi Pico (original, without wireless).
- DC motors and motor drivers.
- nRF24L01+ radio module.
- Battery-powered operation.

### Central hub

- Raspberry Pi Zero.
- Web-based programming interface (planned).

Hardware choices and circuit designs are subject to testing. Experimental breadboard circuits are not necessarily the final robot design.

## Software architecture

The software is being designed around clear boundaries:

- **Program format:** a versioned, canonical representation of what the robot should do.
- **Command manager:** interprets and schedules commands, tasks and parallel branches.
- **Hardware abstraction:** provides a consistent interface to motors, sensors and other hardware.
- **Communication layer:** transports messages between the hub and robots without defining program behaviour.
- **Programming interface:** Blockly will eventually create programs using the canonical format.

### Data and communication direction

The current proposal is:

- JSON as the initial canonical program representation.
- JSONL over UART during development, with one JSON message per line.
- A compact binary encoding for nRF24L01+ radio communication, while retaining the same logical message concepts.
- Message identifiers, acknowledgements and explicit error reporting.

These are design directions; the exact schemas and protocol are not yet finalized. See the project status and specifications as they are added.

## Development status

The project is at an early stage.

- MicroPython is installed and running on the Pico.
- USB serial communication from a Linux development machine has been tested.
- A CLI-based workflow using `mpremote` is established.
- Motor-control circuitry has been explored experimentally.
- The repository contains an initial `blockly-app/` directory; its current state and integration are still to be reviewed.
- The canonical program format, command manager and communication protocols remain to be specified and implemented.

For a more detailed snapshot and current tasks, see [PROJECT_STATUS.md](PROJECT_STATUS.md).

## Development workflow

The project is developed locally using a lightweight, CLI-oriented workflow.

The current Pico workflow uses:

- MicroPython firmware.
- `mpremote` for connecting to and deploying code to the Pico.
- USB serial (`/dev/ttyACM0` on the current Linux development setup).

Development should proceed incrementally: test hardware drivers independently, agree on interfaces and data formats, and integrate components only after their behaviour is understood.

## Repository layout

The repository is evolving. The currently visible top-level application directory is:

```text
shiny-robotics/
├── blockly-app/
├── LICENSE
├── README.md
└── PROJECT_STATUS.md
```

Additional firmware, protocol and documentation directories should be introduced when their contents are ready, rather than creating empty structure prematurely.

## Documentation

Planned documentation includes:

- `PROJECT_STATUS.md` – current development stage and next steps.
- `ARCHITECTURE.md` – system architecture and component boundaries.
- `DECISIONS.md` – significant architectural decisions and their rationale.
- `PROGRAM_FORMAT.md` – canonical program representation and schema.
- `PROTOCOL.md` – UART and radio message formats.
- `COMMANDS.md` – supported robot operations and arguments.

## Contributing and development principles

This project favours:

- Readable, modular and maintainable code.
- Lightweight dependencies.
- Small, testable components.
- Clear separation between program representation, execution, communication and hardware.
- Explicitly documented design decisions.
- Educational simplicity over unnecessary abstraction.

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE).
