# AGENTS.md

This file provides guidance to AI coding agents working in this repository.

## Build & Test Commands

**Host application (C# .NET 8.0):**
```bash
dotnet build Host/Host.csproj
dotnet test Host.Tests/Host.Tests.csproj
dotnet test Host.Tests/Host.Tests.csproj --filter "FullyQualifiedName~ClassName.TestName"  # single test
```

**MSVC Instrument (C, Windows only):**
```bash
cd MsvcInstrument
cmake -G "Visual Studio 17 2022" -B build
cmake --build build --config Release
```

**STM32 Instrument (C, ARM cross-compile):**
```bash
cd STM32Instrument
cmake --preset RelWithDebInfo
cmake --build build/RelWithDebInfo
```

## Architecture

This project implements a medical instrument communication protocol using **FlatBuffers** for serialization and **Compact Frame Format (CFF)** for reliable framing over Named Pipes, TCP, or UART.

### Protocol Schema

`instrument_protocol.fbs` defines three message types via a `Message` union:
- **Command** — Start/Stop instrument control
- **Configuration** — sampling rate (`measurements_per_second`) and `samples_per_measurement`
- **Measurement** — variable-length float array of measurement data

### Components

- **Host** (`Host/Program.cs`) — C# .NET console app. Contains all host logic in one file: command-line parsing, `ICommunicator` interface with `NamedPipeCommunicator` and `TcpSocketCommunicator` implementations, and `InstrumentHost` which orchestrates connect → configure → start → receive measurements → stop.

- **MsvcInstrument** (`MsvcInstrument/instrument.c`) — C implementation for Windows. Named pipe server that receives config/commands and generates simulated measurement data at the configured rate.

- **STM32Instrument** (`STM32Instrument/`) — ARM embedded C implementation for STM32F100. Communicates over UART. Uses STM32CubeMX-generated project structure with QEMU emulation support.

- **Tests** (`Host.Tests/`) — xUnit tests covering both communicator implementations (connection, TCP echo) and InstrumentHost (FlatBuffer message creation/parsing, fragmented data handling, corrupted data resilience).

### Message Flow

Host connects → sends Configuration → sends Start command → Instrument sends Measurement messages at configured Hz → Host sends Stop command.

### Key Dependencies

- **FlatBuffers**: Schema compiler generates C# classes into `Host/InstrumentProtocol/` and C headers into `MsvcInstrument/InstrumentProtocol/` and `STM32Instrument/InstrumentProtocol/`. The `flatcc/` and `flatbuffers/` directories are git submodules for the compilers.
- **CompactFrameFormat (CFF)**: NuGet package for C#; vendored C implementation in `MsvcInstrument/cff/` and `STM32Instrument/cff/`. Frames have an 8-byte header (0xFACE preamble, length, counter), payload, and 2-byte checksum footer. Ring buffer-based parsing handles fragmented data.

### CI

GitHub Actions (`.github/workflows/ci.yml`) runs `dotnet test` on pushes to main and PRs.
