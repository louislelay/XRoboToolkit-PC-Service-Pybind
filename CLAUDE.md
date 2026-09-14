# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This project provides Python bindings for the XRoboToolkit PC Service SDK, enabling Python applications to extract XR state data including controller poses, hand tracking, and body motion capture from XR devices (primarily PICO headsets).

## Architecture

The project consists of:

- **Core C++ Bindings** (`bindings/py_bindings.cpp`): Pybind11-based C++ module that wraps the PXREARobotSDK
- **SDK Integration**: Uses the XRoboToolkit-PC-Service SDK (cloned from external repository)
- **Build System**: CMake-based build with Python setuptools integration
- **Multi-platform Support**: Linux (x86_64/aarch64) and Windows

Key components:
- `PXREARobotSDK.h`: Main SDK header providing device connectivity and data parsing
- `py_bindings.cpp`: Thread-safe C++ wrapper with mutex-protected global state variables
- JSON parsing using nlohmann/json for device state data
- Callback-based data updates from the SDK

## Build Commands

The project is uv-first and uses scikit-build-core as its build backend.
Prebuilt PXREARobotSDK binaries are vendored in `vendor/` (Linux x86_64,
Linux aarch64, Windows amd64), so no SDK download or build step is needed.
Building the extension is only possible on Linux and Windows; macOS is
unsupported (no vendor binaries exist).

```bash
# Build and install into the current environment
uv pip install .   # or: pip install .

# Build sdist + wheel
uv build

# Uninstall
uv pip uninstall xrobotoolkit-sdk
```

Release wheels are built by cibuildwheel in `.github/workflows/wheels.yml`
(manylinux_2_34 for both Linux arches, dictated by the vendored libraries'
glibc requirements — see `vendor/README.md`).

## Data Flow and Threading

The SDK uses a callback-based architecture:
- `OnPXREAClientCallback`: Main callback function that receives JSON data from connected devices
- Global state variables (poses, button states, etc.) are updated in real-time
- Thread-safe access via mutex locks for each data category
- Data parsing from comma-separated pose strings to arrays

## Key Functions and Data Types

### Controller Data
- Poses: `std::array<double, 7>` (x,y,z,qx,qy,qz,qw)
- Buttons: Menu, Primary, Secondary, Axis Click
- Analog: Trigger, Grip, Axis (x,y)

### Hand Tracking
- 26 joints per hand with 7 values each (position + quaternion)
- Hand scale factor

### Body Tracking
- 24 body joints with pose, velocity, acceleration data
- IMU timestamps for each joint
- Availability flag for body tracking system

## Dependencies

### Required (build time)
- CMake >= 3.18 and a C++17 compiler
- pybind11 and scikit-build-core (fetched automatically by the build backend)

### Platform-specific Libraries (vendored in `vendor/`)
- Linux: `libPXREARobotSDK.so` (x86_64 and aarch64 variants)
- Windows: `PXREARobotSDK.dll` and `PXREARobotSDK.lib`

## Testing

No formal test suite is included. Test functionality using the example scripts in `examples/`:
- `example.py`: Basic controller and headset pose testing
- `example_body_tracking.py`: Body tracking functionality
- `run_binding_continuous.py`: Continuous data capture

## Important Notes

- The SDK requires active XR device connection (PICO headset)
- Body tracking requires at least two Pico Swift devices
- All data access is thread-safe but real-time dependent on device connectivity
- The project builds a Python extension module that must be installed to site-packages