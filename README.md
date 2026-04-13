# RustHarder

A Rust port of [tryharder](https://github.com/tehstoni/tryharder) — a Windows shellcode loader using APC injection with sandbox evasion techniques, intended for authorized red team engagements.

## Overview

RustHarder fetches a shellcode payload (from HTTP/HTTPS or a raw TCP staging listener), spawns a sacrificial host process in a suspended state, injects the payload via `QueueUserAPC`, and resumes the thread to execute it.

## Techniques

| Technique | Description |
|---|---|
| **APC Injection** | Queues shellcode as an APC routine on the suspended host thread, executed on `ResumeThread` |
| **Process Spawning** | Spawns `wmiprvse.exe` (or configurable) suspended as the shellcode host |
| **Dynamic API Resolution** | `CreateProcessA`, `VirtualAllocEx`, `WriteProcessMemory`, `QueueUserAPC` are resolved at runtime via `GetModuleHandle`/`GetProcAddress` to avoid static IAT entries |
| **String Obfuscation** | API name strings are built from char arrays at runtime to hinder static string analysis |
| **Sleep-Based Sandbox Evasion** | Sleeps 2 seconds and verifies the elapsed time; accelerated sandboxes that skip sleep calls will fail the timing check and exit |
| **Anti-Debug Heuristic** | Allocates a buffer seeded from system time and conditionally loops `GetCurrentThreadId` calls to interfere with debugger-based timing |
| **RAM Check** | Optional memory floor check (currently commented out) — exits if total physical RAM is ≤ 1 GB, targeting sandbox VMs |

## Payload Delivery

The `url` variable in `main()` controls where the shellcode is fetched from. Two schemes are supported:

- **HTTP/HTTPS** — standard GET request via `reqwest`
  ```
  http://10.200.47.86:8443/agent.x64.bin
  https://your.server/payload.bin
  ```
- **TCP** — connects to a raw TCP staging listener; expects a 4-byte little-endian length prefix followed by the payload bytes (compatible with [Sliver](https://github.com/BishopFox/sliver) stagers)
  ```
  tcp://10.200.47.86:4444
  ```

## Building

Requires the MSVC toolchain targeting Windows x86-64.

```bash
# Install Rust with the MSVC target
rustup target add x86_64-pc-windows-msvc

# Debug build
cargo build

# Release build (recommended for ops)
cargo build --release
```

The compiled binary will be at `target/release/RustHarder.exe`.

## Configuration

Edit `src/main.rs` before building:

| Variable / Line | Default | Purpose |
|---|---|---|
| `url` | `http://10.200.47.86:8443/agent.x64.bin` | Shellcode staging URL |
| `target_process` | `C:\Windows\System32\wbem\wmiprvse.exe` | Sacrificial host process |
| RAM check (commented) | `≤ 1 GB → exit` | Re-enable to add memory-based sandbox detection |

## Dependencies

- [`reqwest`](https://crates.io/crates/reqwest) `0.12` — blocking HTTP client for payload staging
- [`winapi`](https://crates.io/crates/winapi) `0.3` — Windows API bindings

## Disclaimer

This tool is intended for use in **authorized penetration testing, red team exercises, and security research only**. Do not use against systems you do not have explicit written permission to test.

## Credits

Based on [tryharder](https://github.com/tehstoni/tryharder) by tehstoni.
