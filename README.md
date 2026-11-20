
# Java4Shell — OpenJDK Base64 Decoder Reproduction Lab

<p align="center">
  <strong>CVE-2026-114938 | OpenJDK AVX-512 VBMI Base64 Decoder Memory Corruption</strong>
</p>

<p align="center">
  <img width="1280" height="720" alt="Java4Shell reproduction overview" src="https://github.com/user-attachments/assets/03310df6-b27d-4bdc-8118-6a00b2e06b41" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/CVE-2026--114938-red?style=flat-square" alt="CVE">
  <img src="https://img.shields.io/badge/CVSS%204.0-9.2%20Critical-darkred?style=flat-square" alt="CVSS">
  <img src="https://img.shields.io/badge/Platform-Linux-blue?style=flat-square" alt="Platform">
  <img src="https://img.shields.io/badge/Architecture-x86__64-orange?style=flat-square" alt="Architecture">
  <img src="https://img.shields.io/badge/CPU-AVX--512%20VBMI-blue?style=flat-square" alt="CPU">
  <img src="https://img.shields.io/badge/Result-Root%20Shell-brightgreen?style=flat-square" alt="Result">
</p>

<p align="center">
  <a href="#overview">Overview</a> ·
  <a href="#observed-results">Observed Results</a> ·
  <a href="#system-requirements">System Requirements</a> ·
  <a href="#hardware-compatibility">Hardware Compatibility</a> ·
  <a href="#software-dependencies">Dependencies</a> ·
  <a href="#lab-architecture">Lab Architecture</a> ·
  <a href="#reproduction-stages">Reproduction Stages</a> ·
  <a href="#negative-controls">Negative Controls</a> ·
  <a href="#limitations">Limitations</a>
</p>

---

## Overview

This repository documents a controlled reproduction and investigation of **Java4Shell (CVE-2026-114938)**, a memory-corruption vulnerability in the optimized AVX-512 VBMI Base64 decoder implementation in OpenJDK.

The affected decoder implementation is reachable through `java.util.Base64` on supported x86-64 hardware. Under vulnerable conditions, specially crafted input causes an out-of-bounds write past the end of the decode buffer, before the decoder rejects the input and throws `IllegalArgumentException`.

The chain observed in this lab, in order:

malformed Base64 over HTTP → heap corruption → array header clobber → chosen-value out-of-bounds write → heap disclosure → native read/write → Code Cache shellcode → root shell.

**This is not a new vulnerability.** CVE-2026-114938 was disclosed by Calif on 17 November 2026 and patched the same day. The contribution here is the reproduction record: offsets, commands, negative controls and raw output.

> A successful shell in a lab is not, by itself, proof that every preceding stage caused that shell. The causal relationship is validated separately, by the negative controls in this repository.

## Vulnerability Information

| Property | Details |
|---|---|
| Vulnerability | Java4Shell |
| CVE | CVE-2026-114938 |
| Component | OpenJDK HotSpot JVM, `StubGenerator::generate_base64_decodeBlock()` |
| Affected functionality | Optimized Base64 decoder intrinsic |
| Root cause | Integer underflow in the padded-tail output mask |
| Weakness | CWE-191 → CWE-122 (heap buffer overflow) |
| CPU architecture | x86-64 only |
| Hardware requirement | AVX-512 VBMI |
| Public disclosure | 17 November 2026 |
| Researcher-reported severity | CVSS 4.0: 9.2 Critical |

Severity scores differ between vendors and methodologies. Refer to the original advisory and vendor assessments when reporting severity.

## Observed Results

Every row below was executed and produced the stated output on the environment in the next section.

| Stage | Path | Activity | Observed result |
|---|---|---|---|
| A | `poc/` | trigger | `REPRODUCED` 3/3, `changed=62 first=50 last=111` |
| A control | `poc/` | mitigation | `NOT REPRODUCED`, `changed=0` with `-XX:-UseBASE64Intrinsics` |
| B | `lab/stageB/` | header clobber | `len=64 → 0x9AA6699A`, mark and klass destroyed, 5/5 |
| C | `lab/stageC/` | chosen-value out-of-bounds write | `reader = c0c1c2c3c4c5c6c7`, 3/3 |
| D | `lab/stageD/` | heap disclosure | canary leaked, `leak.length=4096`, 3/3 |
| E | `lab/stageE/` | Code Cache shellcode | `PWNED`, shell as `kali` |
| E (no Unsafe) | `lab/stageE/PureJava.java` | memory-safety bypass | `w.length` negative, `w[100]` writes into neighbour array |
| Remote | `lab/remote/` | full chain over HTTP | reverse shell, `uid=0(root)` |

Recorded environment:

| Item | Value |
|---|---|
| Host | Kali Linux, x86-64 |
| CPU flags | `avx512f avx512bw avx512vbmi bmi2` |
| Local runtime | OpenJDK 21.0.13+9-1-Debian |
| Container runtime | Eclipse Temurin 17.0.9+9 |
| JVM flags | `UseAVX=3`, `UseBASE64Intrinsics=true` |

Measured layout constants, codegen specific, reference only:

- decode buffer 98 bytes, `dec` → `writer` 120, `writer` → `reader` 80, `writer` → `dbuf` 160
- patch site at `hit - 47` (`55 48 83 ec` prologue) or `hit - 14` (`48 81 ec` prologue)
- the marker constant is branch-gated by the invocation counter and is not a valid patch site

## System Requirements

The reproduction environment has four requirements: compatible CPU hardware, a suitable operating system, an appropriate Java runtime, and a working container environment.

### 1. Host hardware

| Component | Requirement |
|---|---|
| CPU architecture | x86-64 |
| CPU instruction support | AVX-512 VBMI |
| Additional CPU flags | AVX-512F, AVX-512BW, BMI2 |
| RAM | 4 GB minimum; 8 GB recommended |
| Storage | 5 GB free space |
| Virtualization | Feature passthrough required if using a VM |

Having an AVX-512-capable processor does not automatically mean the required instruction set is exposed to the OS or VM.

### 2. Operating system

Kali Linux, Debian 12 or later, or Ubuntu 22.04 or later. Linux is required because the workflow uses `/proc`, native binaries and Docker.

### 3. Java runtime

The lab must use a build that actually contains the affected decoder.

| Release line | Affected |
|---|---|
| Java 17 | 17.0.6 and later, before the applicable fixed build |
| Java 21 | before the applicable fixed build |
| Java 25 | before the applicable fixed build |
| Java 27 | before the applicable fixed build |

Pinned builds used here:

- `eclipse-temurin:17.0.9_9-jdk` (container)
- OpenJDK `21.0.13+9-1-Debian` (host, distro build)

Record the vendor, full runtime version, JVM build, architecture and intrinsic configuration for every run.

### 4. Container requirements

| Component | Requirement |
|---|---|
| Docker Engine | Installed |
| Network | Dedicated isolated Docker network |
| Containers | Separate attacker and victim environments |
| Privileges | No privileged containers |

Containers share the host kernel and see only the CPU capabilities the host exposes.

## Hardware Compatibility

The vulnerable path depends on CPU capabilities.

```bash
grep -m1 '^flags' /proc/cpuinfo | tr ' ' '\n' | \
grep -E '^(avx512f|avx512bw|avx512vbmi|bmi2)$'
````

All four flags must appear. Then:

**bash**

```bash
uname -m
java -XX:+PrintFlagsFinal -version 2>&1 | grep -E 'UseBASE64Intrinsics|UseAVX '
```

Required: `x86_64`, `UseAVX = 3`, `UseBASE64Intrinsics = true`.

Notes:

- AVX-512 VBMI is mandatory for the affected path.
- ARM64 never executes this implementation.
- Many hypervisors hide AVX-512; enable passthrough explicitly (`-cpu host` on KVM).
- Cloud CPU exposure depends on instance family.

## Software Dependencies

**bash**

```bash
sudo apt update
sudo apt install -y \
    git curl wget binutils \
    netcat-openbsd \
    docker.io docker-compose \
    procps file strace gdb
```

Optional, PDF documentation only:

**bash**

```bash
sudo apt install -y poppler-utils pandoc chromium
```

Docker:

**bash**

```bash
sudo systemctl enable --now docker
docker --version
sudo docker info
```

Non-root access:

**bash**

```bash
sudo usermod -aG docker "$USER"
```

> Docker group membership is effectively root on the host. Use a dedicated research machine.

Java verification:

**bash**

```bash
java -version
javac -version
java -XshowSettings:properties -version 2>&1
```

## Lab Architecture

Two containers on a dedicated network.

| **Component**      | **Responsibility**                     |
| ------------------ | -------------------------------------- |
| Attacker container | request driver, offset arithmetic      |
| Victim container   | vulnerable JVM and decode endpoint     |
| Docker network     | controlled communication, no egress    |
| Host               | CPU capabilities and container runtime |

The victim runs a single-threaded pool so allocation order stays deterministic. This is easier than a real application, where TLAB grooming across threads and TCP request streaming are required.

## Repository Structure

**text**

```
java4shell-poc/
├── poc/                          trigger and mitigation control
├── exploit/                      original community toolkit
├── scanner/                      original detection utilities
├── detection/                    original rule sets
├── docs/                         original analysis
│
├── lab/
│   ├── stageB/                   header clobber harness
│   ├── stageC/                   chosen-value out-of-bounds write
│   ├── stageD/                   heap disclosure
│   ├── stageE/                   Code Cache shellcode, local, PureJava demo
│   └── remote/
│       ├── victim/               Dockerfile, Victim.java
│       └── attacker/             exploit.py
│
├── evidence/                     captured output per stage
├── java4shell-reproduction.md    full ordered record
├── README.md
└── LICENSE
```

## Reproduction Stages

Each stage is assessed independently. A result from one stage is not proof of the next.

| **Stage** | **Activity**                      | **Evidence**                 |
| --------- | --------------------------------- | ---------------------------- |
| A         | trigger and mitigation control    | stdout, hash manifest        |
| B         | adjacent array header corruption  | mark, klass, length dump     |
| C         | chosen-value write across objects | neighbour array contents     |
| D         | out-of-bounds read                | leaked heap bytes            |
| E         | Code Cache shellcode, local       | `PWNED`, process identity    |
| Remote    | same chain over HTTP              | reverse shell, `uid=0(root)` |

Run order, exact commands and expected output: `java4shell-reproduction.md`.

Operational rules:

- one fresh victim container per exploit run — the first run patches live code, a second run writes over the patched prologue and crashes the JVM
- `-XX:CompileThreshold=1000` on the victim, `dontinline` on `poke`, `pull`, `probe`, `dbput` — an interpreted writer path throws on the negative length
- `--add-modules jdk.httpserver` for compilation and execution
- keep `hs_err_pid*.log` files; they are evidence

## Negative Controls

| **Control** | **Change**                    | **Expected**                 |
| ----------- | ----------------------------- | ---------------------------- |
| 1           | valid Base64, no padding      | `writer.len=64`, no shell    |
| 2           | decode request skipped        | `addr bytes ok: 0`, no shell |
| 3           | `-XX:-UseBASE64Intrinsics`    | `writer.len=64`, no shell    |
| 4           | `PureJava` with intrinsic off | `NO BYPASS`                  |
| 5           | CPU without AVX-512 VBMI      | no corruption                |

All four executed controls fail as expected. That is what makes the chain causal rather than incidental.

## Verification and Evidence

Preserve for every experiment: full command and configuration, runtime and vendor, CPU flags, image identifiers, application logs, crash logs, process identity and privileges, failures, and control results.

`evidence/` holds the per-stage logs, `evidence/remote/` the victim log, `evidence/MANIFEST.sha256` the hashes.

## Security Considerations

- Authorized research only.
- Keep the victim off public networks.
- Disposable containers and test data.
- No production secrets in logs.
- Crash logs and dumps may be sensitive.

## Limitations

- Not a new CVE. Public and patched at disclosure.
- Single-threaded victim. No TLAB grooming across threads, no TCP request streaming. A real application is harder.
- The victim exposes helper endpoints (`/write`, `/read`, `/scan`, `/dbput`). A real target has none; the attacker would drive everything through the application's own code paths.
- Offsets are JDK 17 C1 codegen specific and must be re-derived per build.
- Exploitation uses `sun.misc.Unsafe` for address bookkeeping and patching. The corruption itself does not — `PureJava.java` demonstrates that without importing `Unsafe`.
- Root is a container artefact. A real target yields the service account.
- Reliability is not measured. No N-of-M statistics exist.
- No product chain: Keycloak, Jira and Jenkins are out of scope here.

## References

- Calif — original research: https://calif.io/java4shell/
- OpenJDK security advisories: https://openjdk.org/groups/vulnerability/advisories/2026-11-17
- Oracle CPU: https://www.oracle.com/security-alerts/cspunov2026.html
- Red Hat CVE-2026-114938: https://access.redhat.com/security/cve/cve-2026-114938

## Disclaimer

Educational purposes, vulnerability research and authorized security testing only.

The original vulnerability and the independent experiments documented here are distinct. No claim of independent discovery or universal exploitability is made.
