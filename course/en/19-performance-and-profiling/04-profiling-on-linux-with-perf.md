<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.3 Profilers and flame graphs](03-profilers-and-flame-graphs.md) · [📋 Phase 19](README.md) · [19.5 Memory profiling and fewer allocations →](05-memory-profiling-and-allocations.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/04-profiling-on-linux-with-perf.md) <!-- nav -->

# Lesson 19.4 — Profiling on Linux with perf

> **Phase 19 — Performance and profiling** · Lesson 4 of 11 · ⏱ about 90 minutes

## What you will learn

- What `perf` is and why it is the standard profiler on Linux servers.
- How to run `perf` safely: in the VM (virtual machine) `dev` or in a privileged container, never as root on your Mac.
- How to read `perf stat` (counters), `perf record` and `perf report` (sampled call stacks).
- The two ways to walk the call stack, frame pointers and DWARF, and what each one costs.
- How to turn a Linux profile into a flame graph with `inferno`, and why the same fix gives a different speed-up on Linux.

## Before you start

- You finished [Lesson 19.3 — Profilers and flame graphs](03-profilers-and-flame-graphs.md) and have the package `code/p19-wordfreq` with `src/main.rs` (the fast version), `src/bin/slow.rs` and `src/bin/gen.rs`.
- You know system calls ([Lesson 13.3 — System calls](../13-operating-systems/03-system-calls.md)) and how to run Rust in a container ([Lesson 11.13 — Dockerfiles for Rust programs](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md)).
- Docker Desktop is running, or your VM `dev` from Phase 11 is started (`limactl shell dev`).

## New words

| Word | Meaning |
|---|---|
| **perf** | The profiler that comes with the Linux kernel. It can count events and sample call stacks of programs and of the kernel itself. |
| **event** | Something the kernel or the CPU (Central Processing Unit) can count: time on the CPU (`task-clock`), context switches, page faults, CPU cycles, instructions. |
| **software event** | An event counted by the kernel, for example `task-clock` or `page-faults`. It works everywhere, including in virtual machines. |
| **hardware event** | An event counted by the CPU itself in its **PMU** (Performance Monitoring Unit), for example `cycles` or `cache-misses`. Virtual machines often do not offer it. |
| **IPC** (Instructions Per Cycle) | How many instructions the CPU finishes per clock cycle on average. Low IPC usually means the CPU is waiting, for example for memory. |
| **privileged container** | A container started with `--privileged`. It has almost all the rights of root on the (virtual) machine that runs it. Use it only for local experiments. |
| **frame pointer** | A register that points to the current function's stack frame. If every function keeps it, the profiler can walk from frame to frame quickly. |
| **DWARF** | The standard format for debug info on Linux (the name is a pun, not an abbreviation). It also describes how to unwind the stack without frame pointers. |
| **unwinding** | Walking the stack from the current function back to `main` to find the call chain. |
| **DSO** (Dynamic Shared Object) | A file that holds code: your program, a shared library such as `libc.so.6`, or the kernel (`[kernel.kallsyms]` in perf). |
| **libc** | The C standard library of the system. On Linux it provides `malloc` and `free`, which Rust's default allocator uses. |

## 1. Why profile on Linux too

Your services run on Linux, not on macOS. The Linux kernel, the C library and its memory allocator are all different from the ones on your Mac, so the same program can spend its time in different places. And `perf` can do something `samply` on macOS cannot: it sees **into the kernel**, so a profile shows time spent inside system calls such as `read` next to your own functions.

`perf` talks to the kernel through a system call named `perf_event_open`. Because it can observe other programs and the kernel, Linux restricts it. Two safe ways to run it for this lesson:

- **In the VM `dev`** (Phase 11). Install it with `sudo apt-get install -y linux-perf` and run `perf` with `sudo`, because Ubuntu restricts `perf` to root by default.
- **In a privileged container** from the official `rust:1.100` image (Debian). This is what the outputs in this lesson come from.

> 🔒 **Security:** `--privileged` gives the container root power over Docker's Linux virtual machine. Use it only for local experiments with your own code, never with images you do not trust, and never on a shared or production server. Never run `perf` as root on your own Mac.

## 2. A Linux shell with perf

From your package folder on the Mac:

```bash
cd ~/projects/rust/code/p19-wordfreq
docker run --rm -it --privileged -v "$PWD":/work -w /work \
  -v course-cargo-registry:/usr/local/cargo/registry rust:1.100 bash
```

**What happens:**

1. `--rm -it` starts an interactive container that is deleted when you leave it.
2. `--privileged` allows `perf_event_open`. Section "Common mistakes" shows what happens without it.
3. `-v "$PWD":/work` makes your package folder visible inside as `/work`. Files you create there appear on your Mac too.
4. The named volume `course-cargo-registry` keeps downloaded crates between containers.

Inside the container, install `perf` and `rustfilt`, and build. Linux binaries must not mix with the macOS binaries in `target/`, so we send them to `target/linux`:

```bash
apt-get update && apt-get install -y linux-perf
cargo install --locked rustfilt
export CARGO_TARGET_DIR=target/linux
cargo build --profile profiling
perf --version
```

```text
perf version 6.12.117
```

`words.txt` from Lesson 19.3 is already there, because the folder is shared. The Linux binaries are now in `target/linux/profiling/`: `slow` (the original version) and `p19-wordfreq` (the fast one).

## 3. Counting with perf stat

`perf stat` runs a program and counts events while it runs:

```bash
perf stat ./target/linux/profiling/slow words.txt > /dev/null
```

```text
 Performance counter stats for './target/linux/profiling/slow words.txt':

           1425.51 msec task-clock                       #    0.937 CPUs utilized
              1044      context-switches                 #  732.371 /sec
                 0      cpu-migrations                   #    0.000 /sec
             29742      page-faults                      #   20.864 K/sec
   <not supported>      cycles
   <not supported>      instructions
   <not supported>      branches
   <not supported>      branch-misses

       1.520992709 seconds time elapsed

       1.289571000 seconds user
       0.136636000 seconds sys
```

**What happens** (line by line):

1. `> /dev/null` throws away the program's own output; `perf` prints its report to the error stream, so you still see it.
2. **task-clock:** 1425 ms of CPU time. "0.937 CPUs utilized" means the single thread was running 94% of the elapsed time.
3. **context-switches:** how often the kernel took the CPU away from our thread (Lesson 13.9). About 1000 in 1.5 s is normal for a busy machine.
4. **page-faults:** 29 742 times the program touched memory that the kernel had not mapped yet (Lesson 13.10). Reading a 130 MB (megabyte) file into a `String` needs many new pages.
5. **cycles, instructions, branches, branch-misses: `<not supported>`.** These are **hardware events** from the CPU's PMU. Docker Desktop runs Linux in a virtual machine that does not pass the PMU through, so these counters are not available here (the VM `dev` on a Mac behaves the same way). On a real Linux machine you would see numbers, and `perf` would also print **IPC**, instructions per cycle: a value well below 1 often means the CPU is waiting for memory, a value of 3 or 4 means it is busy computing.
6. **user / sys:** time in our code and libraries versus time inside the kernel (system calls, page faults).

> 💡 **Tip:** `perf stat -r 5 <command>` runs the command five times and prints the mean and the variation (`+- 3.7%`), like a small `hyperfine`.

## 4. Sampling with perf record and perf report

`perf record` samples the program like `samply` does; `-g` also records the call stack of each sample:

```bash
perf record -g ./target/linux/profiling/slow words.txt > /dev/null
```

```text
[ perf record: Woken up 5 times to write data ]
[ perf record: Captured and wrote 1.092 MB perf.data (6889 samples) ]
```

The samples are in the file `perf.data` (about 4000 samples per second by default). `perf report` reads it. Without options it opens an interactive text screen: arrow keys move, `Enter` opens a menu to zoom into or annotate the selected function, `+` expands its call chain, `/` filters by name, `h` shows all keys and `q` quits. For a printable report, use `--stdio`. The Rust names in the report are mangled, so we pipe it through `rustfilt`:

```bash
perf report --stdio --no-children --sort symbol --percent-limit 2 -g none | rustfilt
```

```text
# Samples: 6K of event 'task-clock:ppp'
# Event count (approx.): 1722250000
#
# Overhead  Symbol
# ........  ..........................................................
#
    20.31%  [.] slow::count_words
    13.05%  [.] slow::is_stop_word
    11.31%  [.] <alloc::string::String as core::iter::traits::collect::FromIterator<char>>::from_iter::<...>
    10.70%  [.] cfree
     7.71%  [.] malloc
     6.60%  [.] <std::hash::random::RandomState as core::hash::BuildHasher>::hash_one::<&alloc::string::String>
     5.17%  [.] <hashbrown::map::HashMap<alloc::string::String, usize, std::hash::random::RandomState>>::rustc_entry
     4.59%  [.] <str>::to_lowercase
     4.22%  [.] <core::hash::sip::Hasher<core::hash::sip::Sip13Rounds> as core::hash::Hasher>::write
     2.28%  [.] __rustc::__rdl_alloc
     2.02%  [.] <alloc::raw_vec::RawVecInner>::finish_grow
```

(Header shortened and one long name cut.) **What happens:**

1. `--no-children` sorts by **self time** ("Overhead" here means self samples). `--children` would sort by total time instead.
2. `--sort symbol` groups by function; `--percent-limit 2` hides everything below 2%; `-g none` hides the call chains under each line.
3. `[.]` marks code running in user space; `[k]` would mark kernel code.
4. `cfree` and `malloc` are the allocator functions of glibc, the C library on Debian. `cfree` is glibc's name for `free`.

Now with total time (`--children`), only the lines above 10%:

```bash
perf report --stdio --children --sort symbol --percent-limit 10 -g none | rustfilt
```

```text
    99.99%     0.00%  [.] slow::main
    ...
    93.90%    20.31%  [.] slow::count_words
    30.21%     0.17%  [.] slow::normalize
    20.13%    11.31%  [.] <alloc::string::String as core::iter::traits::collect::FromIterator<char>>::from_iter::<...>
    16.36%     5.17%  [.] <hashbrown::map::HashMap<alloc::string::String, usize, std::hash::random::RandomState>>::rustc_entry
    13.12%    13.05%  [.] slow::is_stop_word
    11.31%    10.70%  [.] cfree
    10.86%     6.60%  [.] <std::hash::random::RandomState as core::hash::BuildHasher>::hash_one::<&alloc::string::String>
```

The first column is total time ("Children"), the second is self time. Compare with the macOS profile from Lesson 19.3:

```text
                          macOS (samply)     Linux (perf)
normalize, total             ~50%               30%
malloc + free + growing      ~45%               ~23%
HashMap (entry + hashing)     ~4%               16%
is_stop_word                  <1%               13%
```

Same program, same input, very different pictures. glibc's `malloc` and `free` are much cheaper for these tiny strings than the macOS allocator, so on Linux the allocation problem is smaller and other parts become visible.

## 5. Frame pointers or DWARF

To find the call chain of a sample, `perf` must **unwind** the stack. `-g` uses **frame pointers**: it follows a chain of saved frame addresses, which is fast and makes small files. It only works where code keeps frame pointers. Rust on Linux does not force them, and neither do many system libraries, so some chains break. In one of our runs, a few samples had a chain that ended in a raw address such as `0x9aaaadeb5c4eb` instead of a function name.

The alternative is `--call-graph dwarf`. For each sample, `perf` copies a piece of the stack (8 KB, kilobytes, by default) into the file, and `perf report` later uses the DWARF debug info to unwind it. Try it:

```bash
perf record --call-graph dwarf -o perf-dwarf.data ./target/linux/profiling/slow words.txt > /dev/null
perf report -i perf-dwarf.data --stdio --children --sort symbol --percent-limit 15 -g none | rustfilt
```

```text
[ perf record: Captured and wrote 60.797 MB perf-dwarf.data (7450 samples) ]
    79.89%     0.01%  [.] slow::main
    ...
    79.06%    17.68%  [.] slow::count_words
    26.55%     0.21%  [.] slow::normalize
    19.28%    13.05%  [.] <alloc::string::String as core::iter::traits::collect::FromIterator<char>>::from_iter::<...>
    17.70%     0.00%  [.] <std::collections::hash::map::HashMap<alloc::string::String, usize>>::entry (inlined)
    ...
    15.57%     0.00%  [.] <core::str::iter::SplitWhitespace as core::iter::traits::iterator::Iterator>::next (inlined)
```

(Shortened: the full list has many `(inlined)` lines.) The trade-off is clear:

| | `-g` (frame pointers) | `--call-graph dwarf` |
|---|---|---|
| File size for our run | 1.1 MB | 61 MB |
| Inlined functions | not shown | shown as `(inlined)` |
| Broken chains | where code has no frame pointers | where a stack is deeper than the copied 8 KB |

With DWARF we suddenly see that `split_whitespace` (inlined into `count_words`) costs about 15%: invisible with `-g`. But only 80% of the samples reach `main`; for the other 20%, the copied stack piece was not enough. Start with `-g` because it is cheap, and switch to DWARF when you need the inlined details.

> 💡 **Tip:** a third way is to compile your code with frame pointers: `RUSTFLAGS="-C force-frame-pointers=yes" cargo build --profile profiling`. Then `-g` works well for your own functions with small files. The standard library you link is precompiled, so its own functions keep their original settings.

## 6. A flame graph from perf

`perf script` prints every sample with its stack as text. That text is the input for `inferno-collapse-perf`. You can install `inferno` in the container too, but it is already on your Mac (Lesson 19.3), and the folder is shared. In the container:

```bash
perf script > perf.txt
```

Then on the Mac, in a second terminal in the same folder:

```bash
inferno-collapse-perf perf.txt | rustfilt | inferno-flamegraph > flame-linux.svg
open -a Safari flame-linux.svg
```

**What happens:**

1. `perf script` writes about 7 MB of text: for every sample a header line (program, thread ID, time, event) and one line per stack frame, including kernel frames such as `do_page_fault`.
2. `inferno-collapse-perf` turns this into one line per distinct stack with a count; `rustfilt` demangles; `inferno-flamegraph` draws.
3. In the picture, `count_words` is the wide tower in the middle; `is_stop_word` and the `HashMap` boxes are clearly wider than in the macOS graph; at the left edge a thin tower shows the program loading and the kernel handling **page faults**.

Add `perf.data*`, `perf.txt` and `*.svg` to `.gitignore`: profiles are output, not source code.

## 7. Same fix, different speed-up

In Lesson 19.3 the fast path for ASCII words (ASCII: American Standard Code for Information Interchange, the basic Latin letters, digits and symbols) made the program 1.5 times faster on macOS. On Linux, with `perf stat -r 5`:

```bash
perf stat -r 5 ./target/linux/profiling/slow words.txt > /dev/null
perf stat -r 5 ./target/linux/profiling/p19-wordfreq words.txt > /dev/null
```

```text
           1347.59 msec task-clock                       #    0.979 CPUs utilized               ( +-  3.71% )
            1.3768 +- 0.0514 seconds time elapsed  ( +-  3.74% )
           1139.72 msec task-clock                       #    0.978 CPUs utilized               ( +-  3.08% )
            1.1658 +- 0.0357 seconds time elapsed  ( +-  3.06% )
```

(Only the two key lines of each report.) 1.38 s → 1.17 s: about **1.18 times** faster, not 1.5. The fix removed allocations, and allocations were simply cheaper on Linux. Neither number is wrong. The lesson: **measure on the platform you deploy to**, and treat speed-ups measured on a laptop as a first hint.

> 🧪 **Testing:** a profile tells you which part to change next (on Linux: `is_stop_word`, hashing, `split_whitespace`). Before you touch `count_words` or `top`, pin down their behaviour with a small test (Exercise 2). Counting code is easy to break in quiet ways, for example by changing the order of words with equal counts.

## Common mistakes

1. **Forgetting `--privileged`** (or `sudo` in the VM). Without it, `perf` cannot open its events:

   ```text
   Error:
   No permission to enable task-clock event.
   ```

2. **Mixing macOS and Linux builds in one `target/`.** Running `cargo build` inside the container without `CARGO_TARGET_DIR` puts Linux files into the same `target/release` that your Mac uses, and both sides keep rebuilding or get confused. Use `target/linux` (or another folder) inside the container.
3. **Profiling a stripped binary.** `strip` removes the symbol table. A profile then shows only addresses:

   ```text
    11.98%  stripped           [.] 0x00000000000099e0
    10.48%  libc.so.6          [.] cfree
     9.46%  stripped           [.] 0x0000000000009d78
   ```

   Profile the `profiling` build, and do not set `strip = true` in the profile you profile.
4. **Reading mangled names.** Without `rustfilt`, `perf` 6.12 shows Rust names like `_RNvCs5bwcD7iyBfD_4slow11count_words`. Pipe `perf report` and the collapsed stacks through `rustfilt`.
5. **Believing `<not supported>` means zero.** It means "this machine cannot count it", not "it did not happen". Hardware counters need real hardware (or a VM that passes the PMU through).

## Try it yourself

1. Run `perf report` without `--stdio`, select `slow::count_words` and press `e` to expand its call chains. Which paths are the biggest? Press `h` to see the other keys.
2. Record the fast version: `perf record -g ./target/linux/profiling/p19-wordfreq words.txt > /dev/null`, and print the self-time list. Which function is now at the top?
3. Run `perf stat -e task-clock,page-faults ./target/linux/profiling/slow words.txt > /dev/null`. `-e` chooses which events to count.

## Exercises

### Exercise 1 — perf stat as a stopwatch (easy)

Use `perf stat -r 5` to compare `slow` and `p19-wordfreq` on Linux. Which lines of the report do you compare, and how big is the speed-up?

<details>
<summary>Hint</summary>

Look at the `seconds time elapsed` line with its `+-` part, and at `task-clock`. Divide the two elapsed times.

</details>

<details>
<summary>Answer</summary>

The two runs from Section 7: `1.3768 +- 0.0514` and `1.1658 +- 0.0357` seconds elapsed, so 1.3768 ÷ 1.1658 ≈ 1.18. The `+-` values (about 3–4%) are much smaller than the 18% difference, so the difference is real. `task-clock` (1348 ms vs 1140 ms) tells the same story in CPU time. Your numbers will differ; the ratio should be similar.

</details>

### Exercise 2 — Pin down the behaviour before the next change (medium)

The Linux profile points at `count_words` and the `HashMap`. Before anyone optimizes them, write a test in `src/main.rs` that calls `count_words` on the text `"z y, Z! the Y x X. The w"` and checks that (a) there are 4 different words (`"the"` is a stop word), (b) `"w"` was counted once, (c) `top(&counts, 2)` returns `x` and `y` with count 2, in this order.

<details>
<summary>Hint</summary>

`counts["w"]` reads a value from a `HashMap` by key (it panics if the key is missing, which is fine in a test). `x`, `y` and `z` all have count 2; `top` sorts equal counts alphabetically.

</details>

<details>
<summary>Answer</summary>

Add this test to the `tests` module from Lesson 19.3:

```rust
    #[test]
    fn counts_and_top_on_a_small_text() {
        let mut counts = HashMap::new();
        count_words("z y, Z! the Y x X. The w", &mut counts);
        assert_eq!(counts.len(), 4); // "the" is a stop word
        assert_eq!(counts["w"], 1);
        // Equal counts come out in alphabetical order.
        let expected = vec![("x".to_string(), 2), ("y".to_string(), 2)];
        assert_eq!(top(&counts, 2), expected);
    }
```

```text
test tests::counts_and_top_on_a_small_text ... ok
test tests::fast_normalize_matches_the_reference ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the text has `z`, `y`, `x` twice each (in different cases and with punctuation), `w` once, and `the` twice, which is skipped. The tie between `x`, `y` and `z` checks the alphabetical rule in `top`, which an optimized version could easily lose.

</details>

### Exercise 3 — Who spends the time: our code, libc or the kernel? (hard)

Use `perf report --sort dso` on the `perf.data` of the slow version to split the time by DSO. Then list the biggest functions inside libc and inside the kernel (`--dsos libc.so.6` and `--dsos '[kernel.kallsyms]'`). Explain each part.

<details>
<summary>Hint</summary>

```bash
perf report --stdio --no-children --sort dso -g none
perf report --stdio --no-children --sort symbol --dsos libc.so.6 -g none --percent-limit 0.3
```

</details>

<details>
<summary>Answer</summary>

```text
    75.18%  slow
    20.03%  libc.so.6
     4.79%  [kernel.kallsyms]
```

Inside libc: `cfree` 10.70%, `malloc` 7.71%, `memcmp` 1.00%, `realloc` 0.58%. Inside the kernel: `__pi_clear_page` 1.29%, `__arch_copy_to_user` 0.62%, `charge_memcg` 0.36%, and many small ones.

Why it works: three quarters of the time is our own Rust code (including the inlined standard library). Almost all of libc's 20% is the allocator: the cost of the short-lived strings from `normalize`. The kernel's 5% is the price of the input: `clear_page` fills new memory pages with zeros when the 130 MB `String` grows (page faults), `copy_to_user` copies file data into our buffer during `read`, and `charge_memcg` accounts the memory to the container's cgroup (Lesson 13.16). The kernel part would only matter for a program that reads much more than it computes.

</details>

## Quiz

1. Why can a profile from macOS and a profile from Linux of the same program look different?

<details>
<summary>Answer</summary>

The operating system, the C library and especially the memory allocator are different, so the same operations cost different amounts. In our program, allocation was about half the time on macOS and about a quarter on Linux.

</details>

2. What does `<not supported>` next to `cycles` mean in `perf stat`?

<details>
<summary>Answer</summary>

The machine cannot count that hardware event, here because Docker's virtual machine does not expose the CPU's performance counters. It does not mean the value is zero.

</details>

3. In `perf report`, what is the difference between `--children` and `--no-children`?

<details>
<summary>Answer</summary>

`--children` sorts by total time (the function plus everything it called) and shows both columns; `--no-children` sorts by self time only.

</details>

4. Name one advantage and one disadvantage of `--call-graph dwarf` compared with `-g`.

<details>
<summary>Answer</summary>

Advantage: it works without frame pointers and shows inlined functions. Disadvantage: much bigger files (61 MB instead of 1.1 MB here) and more work to record; very deep stacks can still be cut off.

</details>

5. Why do we run `perf` in the VM or in a privileged container instead of directly on the Mac?

<details>
<summary>Answer</summary>

`perf` is a Linux tool and needs the Linux kernel's `perf_event_open`. It also needs elevated rights, which we only give in a disposable Linux environment, never as root on our own Mac.

</details>

6. A fix gave 1.5× on your Mac and 1.18× on a Linux server. Which number do you report?

<details>
<summary>Answer</summary>

The Linux number, if the program runs on Linux in production, together with the setup it was measured on. The Mac number is still useful as a hint, but users get the Linux speed.

</details>

## Summary

- `perf` is the standard Linux profiler; it sees your code, the libraries **and the kernel**.
- Run it in the VM `dev` (with `sudo`) or in a `--privileged` container; set `CARGO_TARGET_DIR=target/linux` so Linux and macOS builds do not mix.
- `perf stat` counts events: `task-clock`, context switches, page faults; hardware counters such as cycles and IPC need real hardware.
- `perf record -g` + `perf report --stdio | rustfilt` gives self and total time per function; `[k]` marks kernel code.
- `-g` (frame pointers) is cheap; `--call-graph dwarf` shows inlined functions but makes large files.
- `perf script | inferno-collapse-perf | rustfilt | inferno-flamegraph` makes a flame graph.
- The same change can give different speed-ups on different platforms. Measure where you deploy.

## Sources and further reading

- [perf wiki: Tutorial](https://perfwiki.github.io/main/tutorial/) — the official tutorial for `perf stat`, `perf record` and `perf report`.
- [perf-record(1) manual](https://man7.org/linux/man-pages/man1/perf-record.1.html) — all options, including `-g` and `--call-graph dwarf`.
- [Brendan Gregg — perf Examples](https://www.brendangregg.com/perf.html) — many practical one-liners and explanations.
- [The Rust Performance Book — Profiling](https://nnethercote.github.io/perf-book/profiling.html) — notes on `perf` with Rust, debug info and frame pointers.
- [inferno on GitHub](https://github.com/jonhoo/inferno) — `inferno-collapse-perf` and the flame graph renderer.
- [rustc codegen options: force-frame-pointers](https://doc.rust-lang.org/rustc/codegen-options/index.html) — the flag that keeps frame pointers in your code.

---

[🏠 Start](../START-HERE.md) · [← 19.3 Profilers and flame graphs](03-profilers-and-flame-graphs.md) · [📋 Phase 19](README.md) · [19.5 Memory profiling and fewer allocations →](05-memory-profiling-and-allocations.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
