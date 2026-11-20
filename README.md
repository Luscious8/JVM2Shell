# Java4Shell — OpenJDK Base64 Decoder Reproduction Lab

<p align="center">
  <strong>CVE-2026-114938 | OpenJDK AVX-512 VBMI Base64 Decoder Memory Corruption</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/CVE-2026--114938-red?style=flat-square" alt="CVE">
  <img src="https://img.shields.io/badge/CVSS%204.0-9.2%20Critical-darkred?style=flat-square" alt="CVSS">
  <img src="https://img.shields.io/badge/Platform-Linux-blue?style=flat-square" alt="Platform">
  <img src="https://img.shields.io/badge/Architecture-x86__64-orange?style=flat-square" alt="Architecture">
  <img src="https://img.shields.io/badge/CPU-AVX--512%20VBMI-blue?style=flat-square" alt="CPU">
  <img src="https://img.shields.io/badge/Environment-Isolated%20Lab-brightgreen?style=flat-square" alt="Environment">
</p>

<p align="center">
  <a href="#overview">Overview</a> ·
  <a href="#system-requirements">System Requirements</a> ·
  <a href="#hardware-compatibility">Hardware Compatibility</a> ·
  <a href="#software-dependencies">Dependencies</a> ·
  <a href="#lab-architecture">Lab Architecture</a> ·
  <a href="#reproduction-stages">Reproduction Stages</a> ·
  <a href="#verification">Verification</a> ·
  <a href="#limitations">Limitations</a>
</p>

---

## Overview

This repository documents a controlled reproduction and investigation of **Java4Shell (CVE-2026-114938)**, a memory-corruption vulnerability in the optimized AVX-512 VBMI Base64 decoder implementation in OpenJDK.

The affected decoder implementation can be reached through Java's Base64 decoding functionality on supported x86-64 hardware. Under vulnerable conditions, specially crafted input can cause an out-of-bounds memory write.

The vulnerability can potentially result in:

- JVM process crashes.
- Heap memory corruption.
- Unauthorized memory disclosure.
- Arbitrary code execution in affected application processes.

This project focuses on understanding the vulnerability, documenting the reproduction environment, examining memory-corruption behaviour, and recording experimental observations.

**Research scope:** The repository contains the original community toolkit alongside a separate experimental lab. Results from the lab are documented independently from the original vulnerability advisory.

> Important: A successful shell in a lab is not, by itself, proof that every preceding stage of the vulnerability chain caused that shell. The exact causal relationship must be independently validated.

## Vulnerability Information

| Property | Details |
|---|---|
| Vulnerability | Java4Shell |
| CVE | CVE-2026-114938 |
| Component | OpenJDK HotSpot JVM |
| Affected functionality | Optimized Base64 decoder |
| Weakness | Out-of-bounds memory write |
| CWE | CWE-787 |
| CPU architecture | x86-64 |
| Hardware requirement | AVX-512 VBMI support |
| Public disclosure | 17 November 2026 |
| Researcher-reported severity | CVSS 4.0: 9.2 Critical |

Severity scores can differ between vendors and scoring methodologies. Refer to the original advisory and vendor assessments when reporting severity.

## System Requirements

The reproduction environment has four important requirements: compatible CPU hardware, a suitable operating system, an appropriate Java runtime, and a working container environment.

### 1. Host hardware

| Component | Requirement |
|---|---|
| CPU architecture | x86-64 |
| CPU instruction support | AVX-512 VBMI |
| Additional CPU flags | AVX-512F, AVX-512BW and BMI2 |
| RAM | 8 GB minimum; 16 GB recommended |
| Storage | 20 GB free space recommended |
| Virtualization | Hardware feature passthrough must be available if using a VM |

**Important:** Having an AVX-512-capable processor does not automatically mean the required instruction set is exposed to the operating system or virtual machine.

### 2. Operating system

Recommended environment:

- Kali Linux
- Debian 12 or later
- Ubuntu 22.04 or later

A Linux environment is recommended because the research workflow uses Linux process information, `/proc`, native binaries and Docker containers.

### 3. Java runtime

The lab must use a Java build that actually contains the affected decoder implementation.

The relevant upstream affected release lines include:

| Java release line | Affected range |
|---|---|
| Java 17 | 17.0.6 and later, before the applicable fixed build |
| Java 21 | Before the applicable fixed build |
| Java 25 | Before the applicable fixed build |
| Java 27 | Before the applicable fixed build |

Exact affected and fixed builds depend on the Java distribution. Do not assume that every vendor uses identical version numbering.

For a reproducible experiment, record:

- Java vendor and distribution.
- Full Java runtime version.
- JVM build information.
- CPU architecture.
- Enabled Base64 intrinsic configuration.

### 4. Container requirements

The lab uses an isolated Docker environment.

| Component | Requirement |
|---|---|
| Docker Engine | Installed |
| Docker Compose | Recommended |
| Network | Dedicated isolated Docker network |
| Containers | Separate attacker and victim environments |
| Privileges | Avoid privileged containers unless strictly required for a documented experiment |

Docker containers share the host kernel and normally see the CPU capabilities exposed by the host. Containerization alone does not provide AVX-512 support if the host or hypervisor does not expose it.

## Hardware Compatibility

The vulnerable optimized decoder path depends on CPU capabilities.

Check the CPU flags on the Linux host:

```bash
grep -m1 '^flags' /proc/cpuinfo | tr ' ' '\n' | \
grep -E '^(avx512f|avx512bw|avx512vbmi|bmi2)$'
```

Expected capabilities:

```text
avx512f
avx512bw
avx512vbmi
bmi2
```

Check the processor architecture:

```bash
uname -m
```

Expected:

```text
x86_64
```

Check CPU information:

```bash
lscpu
```

**Hardware compatibility notes:**

- AVX-512 VBMI is a required capability for the affected optimized decoder path.
- ARM64 systems do not execute this x86-64 AVX-512 implementation.
- Some virtual machines hide AVX-512 CPU features.
- Cloud instances may expose different CPU capabilities depending on the instance family and host configuration.

A missing CPU flag means the corresponding optimized execution path is not available in that environment.

## Software Dependencies

### Required packages

For Debian and Kali Linux:

```bash
sudo apt update

sudo apt install -y \
    git \
    curl \
    wget \
    build-essential \
    binutils \
    netcat-openbsd \
    docker.io \
    docker-compose-v2 \
    procps \
    file \
    strace \
    gdb
```

Optional documentation dependency:

```bash
sudo apt install -y poppler-utils
```

`poppler-utils` is only needed when working with PDF documentation.

### Docker configuration

Start Docker:

```bash
sudo systemctl enable --now docker
```

Check Docker:

```bash
docker --version
docker compose version
```

Check daemon connectivity:

```bash
sudo docker info
```

Optional non-root Docker access:

```bash
sudo usermod -aG docker "$USER"
```

Log out and log in again after changing group membership.

> Security warning: Membership in the Docker group effectively grants root-level control over the host. Use this only on a dedicated research machine.

### Java installation and verification

Install a suitable JDK distribution separately, selecting the exact build required for the experiment.

Verify the runtime:

```bash
java -version
```

Verify the compiler:

```bash
javac -version
```

Record the Java runtime properties:

```bash
java -XshowSettings:properties -version 2>&1
```

The runtime version alone does not establish vulnerability. CPU compatibility and the actual decoder implementation must also be verified.

## Lab Architecture

The reproduction environment uses two separate containers connected through a dedicated Docker network.

| Component | Responsibility |
|---|---|
| Attacker container | Research client and test harness |
| Victim container | Java application and decoder test environment |
| Docker network | Controlled communication between containers |
| Host system | Provides CPU capabilities and container runtime |

The purpose of this separation is to distinguish the remote application environment from the testing environment and to make observations easier to reproduce.

## Repository Structure

```text
java4shell/
│
├── poc/                 # Original proof-of-concept material
├── exploit/             # Original exploit-related material
├── scanner/             # Detection and scanning utilities
├── detection/           # Detection resources
├── docs/                # Research documentation
│
├── lab/                 # Independent reproduction environment
│   ├── attacker/        # Attacker-side test environment
│   ├── victim/          # Java victim environment
│   ├── docker/          # Container configuration
│   └── evidence/        # Experiment logs and observations
│
├── java4shell-reproduction.md
├── README.md
└── LICENSE
```

Directory names describe the intended organization. The actual repository should be updated to match the files that are present.

## Reproduction Stages

The research investigation is organized into separate stages so that each observation can be assessed independently.

| Stage | Research activity | Evidence to record |
|---|---|---|
| 01 | Environment validation | OS, CPU, Docker and Java versions |
| 02 | Hardware compatibility | CPU architecture and instruction flags |
| 03 | Decoder investigation | Java Base64 decoder behaviour |
| 04 | Memory-corruption observation | Crash logs and controlled test output |
| 05 | Memory behaviour analysis | Address and memory observations |
| 06 | Application-level testing | HTTP request and response records |
| 07 | Execution analysis | Process identity and execution evidence |
| 08 | Negative controls | Results with incompatible hardware or mitigations |

Each stage should have its own logs and reproducible test record. A result from one stage should not automatically be treated as proof of the next stage.

## Verification and Evidence

For every experiment, preserve:

- Complete command and configuration.
- Java runtime and vendor details.
- CPU flags and architecture.
- Container image identifiers.
- Application logs.
- JVM crash logs, if generated.
- Process identity and privilege information.
- Successful and unsuccessful test cases.
- Negative-control results.

### Negative controls

Useful controls include:

1. Testing on a CPU without AVX-512 VBMI support.
2. Testing with the relevant JVM intrinsic disabled.
3. Comparing affected and patched Java builds.
4. Comparing decoder behaviour across supported Java distributions.
5. Confirming whether the application actually reaches the relevant Base64 decoder.

Negative controls help establish whether the observed result depends on the vulnerability rather than another component of the test harness.

## Security Considerations

This repository is intended for authorized research in isolated environments.

- Do not run experimental tests against systems without explicit permission.
- Do not expose the victim container or its test endpoints to public networks.
- Use disposable containers and test data.
- Avoid storing production credentials or secrets in experiment logs.
- Treat crash logs and memory dumps as potentially sensitive.
- Keep the vulnerable runtime isolated from normal workloads.

## Limitations

This project distinguishes between the published vulnerability and the results of the independent lab.

The following claims require separate supporting evidence:

- That the Base64 decoder caused the observed memory corruption.
- That the corruption provided the specific memory disclosure or write primitive claimed.
- That the resulting control-flow change was caused by the decoder flaw.
- That remote code execution was achieved through the vulnerable decoder rather than another test mechanism.

A root shell observed in a container establishes the shell's reported identity and privileges. It does not independently establish the vulnerability's complete exploit chain.

<img width="1280" height="720" alt="conve" src="https://github.com/user-attachments/assets/03310df6-b27d-4bdc-8118-6a00b2e06b41" />

## References

- Calif — Original Java4Shell research and advisory: https://calif.io/java4shell/
- OpenJDK security advisories: https://openjdk.org/groups/vulnerability/advisories/
- Red Hat CVE-2026-114938: https://access.redhat.com/security/cve/cve-2026-114938
- Sophos security advisory: https://www.sophos.com/en-us/security-advisories/sophos-sa-20261118-java4shell

## Disclaimer

This repository is intended for educational purposes, vulnerability research and authorized security testing only.

The original vulnerability and the independent experiments documented here are distinct. No claim of independent vulnerability discovery, universal exploitability or successful end-to-end exploitation should be inferred without supporting evidence.

---

<p align="center">
  <strong>Java4Shell — OpenJDK Security Research</strong>
</p>
