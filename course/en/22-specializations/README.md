<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.14 Checkpoint](../21-cloud-native-infrastructure/14-checkpoint.md) · [22.1 Choosing a specialization →](01-choosing-a-specialization.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/README.md) <!-- nav -->

# Phase 22 — Specializations

> **Stage 5 — Mastery and career** · ⏱ about 4 weeks at ~10 hours per week (one track)

## What this phase is about

Phases 00–21 gave you broad foundations: the Rust language, testing and security, how the computer and the operating
system work, networks, services, storage, distributed systems, performance, reliability and Kubernetes. Almost every
systems job expects these basics. This phase adds the second half of a good engineering profile: **one area that you
know in depth**.

You choose **one of seven tracks** after Lesson 22.1: OS and kernel, eBPF, embedded, databases, networking,
compilers and WebAssembly, or cloud-native. Each track has four lessons with the same shape: the field, why Rust is
used there and how to set it up; two lessons of core skills with small, complete examples; and a project with
milestones, followed by a list of books, documentation, open-source projects and communities for the next steps.

You read six lessons in total: 22.1, the four lessons of your track, and the checkpoint 22.30, which has one question
set per track. The other tracks stay here for later; many engineers add a second specialization after a year.
Everything that touches the kernel or hardware runs in a safe place — QEMU, the VM `dev`, a local kind cluster, or a
small board on your desk — never directly on your Mac.

## What you will be able to do

- [ ] Explain what each of the seven specializations is about and choose one with clear reasons.
- [ ] Set up the tools of your track: extra Rust targets, nightly components, an emulator, the VM or a cluster.
- [ ] Build a working system in your field from small pieces: a kernel, an eBPF tool, firmware, a query engine, a
  proxy, an interpreter or a Kubernetes operator.
- [ ] Keep hardware- and kernel-specific code small and test the pure logic with ordinary `cargo test`.
- [ ] Read the official documentation of your field and find the exact facts (registers, offsets, APIs) you need.
- [ ] Finish a track project with milestones, acceptance checks, a README and a clean Git history.
- [ ] Name the books, open-source projects and communities where you can continue and contribute.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 22.1 | [Choosing a specialization](01-choosing-a-specialization.md) | 105 min |
| 22.2 | [OS track: a freestanding Rust binary](02-os-track-a-freestanding-binary.md) | 2.5 h |
| 22.3 | [OS track: booting a kernel in QEMU](03-os-track-booting-a-kernel-in-qemu.md) | 2.5 h |
| 22.4 | [OS track: interrupts and paging](04-os-track-interrupts-and-paging.md) | 3 h |
| 22.5 | [OS track: project and next steps](05-os-track-project-and-next-steps.md) | 20 h |
| 22.6 | [eBPF track: what eBPF is](06-ebpf-track-what-ebpf-is.md) | 105 min |
| 22.7 | [eBPF track: a first program with Aya](07-ebpf-track-a-first-program-with-aya.md) | 135 min |
| 22.8 | [eBPF track: maps and tracing](08-ebpf-track-maps-and-tracing.md) | 135 min |
| 22.9 | [eBPF track: project and next steps](09-ebpf-track-project-and-next-steps.md) | 20 h |
| 22.10 | [Embedded track: microcontrollers and no_std](10-embedded-track-microcontrollers-and-no-std.md) | 3 h |
| 22.11 | [Embedded track: GPIO and embedded-hal](11-embedded-track-gpio-and-embedded-hal.md) | 2.5 h |
| 22.12 | [Embedded track: async with Embassy](12-embedded-track-async-with-embassy.md) | 105 min |
| 22.13 | [Embedded track: project and next steps](13-embedded-track-project-and-next-steps.md) | 20 h |
| 22.14 | [Databases track: how query engines work](14-databases-track-how-query-engines-work.md) | 120 min |
| 22.15 | [Databases track: parsing SQL](15-databases-track-parsing-sql.md) | 2.5 h |
| 22.16 | [Databases track: executing queries](16-databases-track-executing-queries.md) | 2.5 h |
| 22.17 | [Databases track: project and next steps](17-databases-track-project-and-next-steps.md) | 20 h |
| 22.18 | [Networking track: high-performance servers](18-networking-track-high-performance-servers.md) | 2.5 h |
| 22.19 | [Networking track: zero-copy protocol parsing](19-networking-track-zero-copy-parsing.md) | 2.5 h |
| 22.20 | [Networking track: production proxies](20-networking-track-production-proxies.md) | 2.5 h |
| 22.21 | [Networking track: project and next steps](21-networking-track-project-and-next-steps.md) | 20 h |
| 22.22 | [Compilers track: lexing and parsing](22-compilers-track-lexing-and-parsing.md) | 4 h |
| 22.23 | [Compilers track: a tree-walking interpreter](23-compilers-track-an-interpreter.md) | 3 h |
| 22.24 | [Compilers track: WebAssembly with Rust](24-compilers-track-webassembly.md) | 3 h |
| 22.25 | [Compilers track: project and next steps](25-compilers-track-project-and-next-steps.md) | 20 h |
| 22.26 | [Cloud-native track: Kubernetes operators](26-cloud-native-track-kubernetes-operators.md) | 135 min |
| 22.27 | [Cloud-native track: an operator with kube-rs](27-cloud-native-track-an-operator-with-kube-rs.md) | 2.5 h |
| 22.28 | [Cloud-native track: testing and running operators](28-cloud-native-track-testing-and-running-operators.md) | 3 h |
| 22.29 | [Cloud-native track: project and next steps](29-cloud-native-track-project-and-next-steps.md) | 20 h |
| 22.30 | [Checkpoint: your specialization](30-checkpoint-your-specialization.md) | 3.5 h |

## Projects in this phase

- **Setup check** (Lesson 22.1, `code/p22-setup-check`): a small Rust program that checks which tools your track needs.
- **OS track** (`code/p22-kernel`): a tiny arm64 kernel for the QEMU `virt` machine with a serial console, exceptions,
  timer interrupts, paging, a heap and an interactive kernel shell (project in Lesson 22.5).
- **eBPF track** (`code/p22-tracer`, `code/p22-netwatch`): tracing tools with tracepoints, kprobes, maps and a ring
  buffer, and an XDP packet counter with an IP blocklist (project in Lesson 22.9).
- **Embedded, databases, networking, compilers and cloud-native tracks:** a project in the fourth lesson of each
  track — firmware for a small board, a tiny query engine, an HTTP proxy, an interpreter that also runs as
  WebAssembly, and a Kubernetes operator (Lessons 22.13, 22.17, 22.21, 22.25 and 22.29).
- **Checkpoint** (Lesson 22.30): the question set and practical tasks of your track.

## How to study this phase

- **Choose once, then commit.** Read Lesson 22.1, answer its questions in your learning journal, and give your track
  at least two weeks before you judge it. Do not study two tracks at the same time.
- **Get the setup working first.** The first lesson of every track is mostly setup; do not move on until its example
  runs exactly as shown.
- **Keep a facts page.** Every track is full of exact facts — register addresses, tracepoint offsets, API names.
  Write each one down with its source; your own list becomes your fastest reference.
- **Test the pure parts.** Kernel, eBPF and firmware code is hard to test directly; move every calculation into plain
  functions and test them with `cargo test`.
- **Plan the project like Phase 23.** Use the milestones, commit after each one, and read the "next steps" list before
  you finish, so that you know where to continue.

---

[🏠 Start](../START-HERE.md) · [← 21.14 Checkpoint](../21-cloud-native-infrastructure/14-checkpoint.md) · [22.1 Choosing a specialization →](01-choosing-a-specialization.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
