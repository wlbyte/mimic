# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development & Build Commands

### Build Targets
- **Full build (CLI + kmod + tools)**: `make` or `make build`
- **CLI binary only (`out/mimic`)**: `make build-cli`
- **Kernel module only (`out/mimic.ko`)**: `make build-kmod`
- **Tools only (`out/*`)**: `make build-tools`
- **Clean build artifacts**: `make clean`

### Build Options
- **Debug build**: `make MODE=debug`
- **Release build**: `make MODE=release`
- **Static binary**: `make STATIC=1`
- **Checksum hack selection**:
  - `make CHECKSUM_HACK=kfunc` (default, uses BTF kfunc from kernel module)
  - `make CHECKSUM_HACK=kprobe` (optional kprobe-based fallback)
  - `make CHECKSUM_HACK=kprobe STRIP_BTF_EXT=1` (strips `.BTF.ext` section for compatibility)

### Code Generators & Intellisense
- **Generate BPF skeleton (`src/bpf_skel.h`)**: `make generate-skel`
- **Generate compile commands for LSP (`compile_commands.json`)**: `make generate-compile-commands`

### Testing & Benchmarking
- **Run integration tests (requires root)**: `make test` or `bats tests`
- **Run a single test file**: `bats tests/general.bats` or `bats tests/conntrack.bats`
- **Run a specific test case**: `bats tests/general.bats --filter "IPv4"`
- **Run benchmarks**: `make bench` or `tests/bench.bash [clean|veth|wg|mimic] <iperf3_args>`

## High-Level Architecture & Structure

Mimic is a high-performance UDP-to-TCP obfuscator designed to bypass UDP QoS throttling and port blocking without user-space packet copying overhead.

```
                  +-------------------------------------------------------+
                  |                      Linux Kernel                     |
                  |                                                       |
  Egress (UDP)    |  +---------------+  TC Egress   +------------------+  |  Egress (TCP)
 ----------------->  | Traffic Ctrl  | -----------> | bpf/egress.c     | ----------------->
                  |  | (TC)          |              | (Mangles UDP->TCP|  |
                  |  +---------------+              |  Fixes csum via  |  |
                  |                                 |  kmod kfunc)     |  |
                  |                                 +------------------+  |
                  |                                                       |
  Ingress (TCP)   |  +---------------+  XDP Ingress +------------------+  |  Ingress (UDP)
 ----------------->  | Network Driver| -----------> | bpf/ingress.c    | ----------------->
                  |  | (XDP)         |              | (Restores TCP    |  |
                  |  +---------------+              |  headers->UDP)   |  |
                  |                                 +--------+---------+  |
                  |                                          |            |
                  +------------------------------------------|------------+
                                                             | BPF RingBuffer (mimic_rb)
                                                             v
                                                    +------------------+
                                                    | src/ (User Space)|
                                                    | - run.c          |
                                                    | - Handshake/Timer|
                                                    | - Control plane  |
                                                    +------------------+
```

### Core Subsystems

1. **Kernel Space eBPF Programs (`bpf/`)**:
   - `egress.c`: Attached to Traffic Control (TC) egress hook. Transforms UDP headers into TCP headers in-place and appends reserved header payload at the packet tail.
   - `ingress.c`: Attached to XDP (eXpress Data Path) ingress hook. Restores TCP packets back to original UDP headers in-place at early packet receipt before kernel stack overhead.
   - `main.c` / `main.h`: Maps definition (`mimic_whitelist`, `mimic_conns`, `mimic_rb`), shared BPF ringbuffer events, and packet buffering helpers.

2. **User-Space Control Daemon (`src/`)**:
   - Compiles to `out/mimic`.
   - `run.c`: Loads BPF bytecode using `libbpf` skeleton (`src/bpf_skel.h`), attaches TC egress and XDP ingress programs to network interfaces, and runs an `epoll` event loop consuming BPF ringbuffer events.
   - Handles TCP handshake emulation, window scale negotiation, keepalives, timers, and packet retransmission prior to connection establishment.
   - `config.c` & `args.c`: Configuration parser and CLI option processing (`argp`).
   - `show.c`: Subcommand `mimic show` for inspecting active connections and loaded BPF state from runtime lock files in `/run/mimic`.

3. **Kernel Module Extension (`kmod/`)**:
   - Compiles to `out/mimic.ko` (also supports DKMS via `generate-dkms`).
   - When mangling UDP to TCP in TC egress, `skb->csum_offset` points to the UDP checksum position. The kernel module registers BTF kfuncs (`mimic_change_csum_offset`, `mimic_inspect_skb`) so eBPF can fix `skb->csum_offset` for hardware/driver checksum offloading.

4. **Common Header Abstractions (`common/`)**:
   - Shared between BPF and C user-space (`defs.h`, `checksum.h`, `try.h`, `log.h`).
   - `MIMIC_BPF` preprocessor symbol distinguishes eBPF compilation target from user-space target.
   - Contains `raii(f)` cleanup attributes (`__attribute__((cleanup(f)))`) and `try()` macro helpers for error handling.

## Code Style & Guidelines

- **C Standard**: C99 with GNU extensions (`-std=gnu99`).
- **Formatting**: Google C style with 2-space indentation and 100-character line limit (configured in `.clang-format`).
- **Header Guards**: `MIMIC_<DIR>_<NAME>_H` format (e.g., `MIMIC_COMMON_DEFS_H`).
