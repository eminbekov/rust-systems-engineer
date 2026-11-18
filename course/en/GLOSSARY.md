<!-- nav:top -->
[🏠 Start](START-HERE.md) · [🗺 Course map](README.md) · [✅ Progress](PROGRESS.md) · [📖 Glossary](GLOSSARY.md) · [🇷🇺 Русский](../ru/GLOSSARY.md) <!-- nav -->

# Glossary

Every abbreviation and new word in the course, explained simply. Each entry links to the lesson where it first appears. Use Ctrl/Cmd+F to search this page.

## Abbreviations

| Abbreviation | Stands for | What it means | First used in |
|---|---|---|---|
| 0-RTT | zero round-trip time | Sending request data in the very first packet to a server you talked to before; fast, but the data can be replayed. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| 2FA | Two-Factor Authentication | A second proof of identity (for example a code from an app) in addition to the password. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| 2PC | two-phase commit | An atomic commit protocol: a coordinator first asks every participant whether it can commit, then tells all of them the one decision. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| AA | Authoritative Answer | A DNS header flag: the answer comes from a server that is responsible for the zone, not from a cache. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| AAA | Arrange, Act, Assert | The shape of a good test: prepare the data, run the code, check the result. | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| AAAA | quad-A (an address record four times the size of A) | The DNS record type that holds an IPv6 address (16 bytes). | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| AAD | Additional Authenticated Data | Data that AEAD does not encrypt but covers with the tag, for example a file header. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| AAPCS64 | Procedure Call Standard for the Arm 64-bit Architecture | The calling convention of arm64: arguments in x0–x7, result in x0, return address in x30. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| AArch64 / arm64 | Arm 64-bit architecture | The 64-bit instruction set of Arm processors, used by Apple M-series Macs; both names mean the same. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| ABA | (the values A, B, A) | A CAS sees value A, does not notice that it changed to B and back to A, and succeeds when it should fail. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| ABAC | Attribute-Based Access Control | Deciding access from many facts (department, time of day, data label), not only a role. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| ABI | Application Binary Interface | Low-level rules for calling functions in compiled code: where arguments go, how results are returned, how types are laid out. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| ACID | Atomicity, Consistency, Isolation, Durability | The four guarantees of database transactions: all or nothing, rules always hold, unfinished work is invisible to others, committed data survives crashes. | [Lesson 16.7](16-backend-services/07-migrations-and-transactions.md) |
| ACK | acknowledgement | A TCP message or flag meaning 'I have received every byte before number N'. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| ACL | Access Control List | A list on an object that says which users may do what with it. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| ACME | Automatic Certificate Management Environment | The protocol (used by Let's Encrypt) that issues and renews certificates automatically. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| AcqRel | Acquire and Release | The ordering for read-modify-write operations whose read part is Acquire and write part is Release. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| AD | Authenticated Data | A DNSSEC flag in the DNS header: in an answer the resolver checked the signatures; in a query it asks for that information. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| ADR | Architecture Decision Record | A short file that records one important decision: its context, the decision and its consequences. ADRs are numbered and never edited, only superseded. | [Lesson 23.2](23-capstone-projects/02-writing-a-design-document.md) |
| AEAD | Authenticated Encryption with Associated Data | Encryption that also produces an authentication tag, so it hides the data and detects any change. XChaCha20-Poly1305 and AES-GCM are AEAD ciphers. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| AES | Advanced Encryption Standard | A widely used symmetric block cipher with 128-, 192- or 256-bit keys. AES-256-GCM is an AEAD mode of it. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| AF | Access Flag | Bit 10 of an arm64 page table entry; it must be 1, or the first access to the page faults. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| AI | Artificial Intelligence | Software that imitates human intelligence. An AI assistant such as Claude answers questions in plain language; use it as a tutor, not a replacement for your own work. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| ALPN | Application-Layer Protocol Negotiation | A TLS extension in which client and server agree which application protocol to speak, for example h2 (HTTP/2) or http/1.1. | [Lesson 15.9](15-networking/09-http-with-hyper-and-reqwest.md) |
| Alt-Svc | Alternative Services | An HTTP response header that advertises another protocol for the same site, for example h3 on UDP port 443. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| AMD | Advanced Micro Devices | A company that makes x86-64 processors; x86-64 is also called amd64. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| AMD64 | AMD 64-bit architecture | AMD's name for the x86-64 instruction set; x86-64, amd64 and x64 mean the same. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| ANSI | American National Standards Institute | The organisation that publishes the SQL standard in the USA; "ANSI SQL" means standard SQL. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| AoS | Array of Structs | A data layout that stores whole records one after another, such as Vec<Particle>. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| AP | Available under Partition | A CAP label for a system that keeps answering every request during a network partition, at the price of possibly stale answers. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| APFS | Apple File System | The file system used by macOS on the Mac's SSD. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| API | Application Programming Interface | The functions, types and rules that a library or program offers to other code that uses it. The public API is what is marked pub. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| APT | Advanced Package Tool | The package manager of Debian and Ubuntu; it downloads packages, solves dependencies and calls dpkg. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| Arc | Atomically Reference Counted | Like Rc, but its counter is safe to use from many threads. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| ARIES | Algorithms for Recovery and Isolation Exploiting Semantics | A classic database recovery method that uses both redo and undo information in the log. | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| ARM | (name of a CPU design family) | A family of CPU designs. The 64-bit form is arm64 / AArch64, used by Apple M-series chips. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| arm64 / aarch64 | 64-bit ARM architecture | The CPU family of Apple Silicon chips such as the M4. Linux calls it aarch64. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| ARP | Address Resolution Protocol | The protocol that finds the MAC address for an IPv4 address on a local network. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| ASan | AddressSanitizer | A sanitizer that adds checks to a running program and stops it when it uses memory in an invalid way. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| ASCII | American Standard Code for Information Interchange | An old table that gives numbers 0-127 to English letters, digits and some symbols ('A' is 65). Unicode includes it. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| ASLR | Address Space Layout Randomization | The operating system loads a program, its libraries and its stack at new random addresses on every run, which makes attacks harder. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| AST | Abstract Syntax Tree | The tree the compiler builds from source code, describing its meaning (for example, a struct with two fields). | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| AT&T | American Telephone and Telegraph | The company whose Bell Labs created Unix; the AT&T assembly syntax for x86-64 is named after it. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| ATS | Applicant Tracking System | Software that companies use to collect and search job applications; people often search the stored resumes by words. | [Lesson 24.6](24-career-and-open-source/06-resume-and-job-search.md) |
| AVL | Adelson-Velsky and Landis (AVL tree) | A classic self-balancing binary search tree, named after its two inventors; the heights of the two subtrees of every node differ by at most 1. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| AVX | Advanced Vector Extensions | The wider (256-bit and 512-bit) SIMD instruction sets of newer x86-64 CPUs. The compiler uses them only when told the CPU has them. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| AVX2 | Advanced Vector Extensions 2 | The 256-bit SIMD instruction set of newer x86-64 CPUs (8 f32 values per instruction). It is used only when the target CPU allows it, for example with -C target-cpu=x86-64-v3. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| AWS | Amazon Web Services | The cloud platform of Amazon; its engineering articles are a common source on retries and timeouts. | [Lesson 18.3](18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| bash | Bourne Again SHell | The default shell of Ubuntu and the language of shell scripts in this course. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| BFS | Breadth-First Search | A graph search that visits all neighbours first, then their neighbours, level by level. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| Big-O | big-O (order of growth) notation | A short way to write how the number of steps or the memory of an algorithm grows with the input size, for example O(n) or O(n²). | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| BOLT | Binary Optimization and Layout Tool | An LLVM tool that reorders the machine code of a finished binary using a profile; used mainly on Linux. | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| BOM | Byte Order Mark | A few invisible bytes at the start of some text files that say how the text is encoded. UTF-8 files usually do not need one. | [Lesson 04.8](04-structs-enums-collections-errors/08-strings-in-depth.md) |
| bool | Boolean | The type with two values, true and false. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| BOOTSEL | boot select | The button on a Raspberry Pi Pico; held at power-on, it makes the board appear as a USB drive for flashing. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| BPF | Berkeley Packet Filter | The older, smaller in-kernel packet filter that eBPF grew from; the kernel still uses the name (the bpf system call). | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| BSD | Berkeley Software Distribution | The name of a family of permissive licenses, for example BSD-3-Clause, used by the redis crate. | [Lesson 20.10](20-reliability-and-hardening/10-project-harden-a-service.md) |
| BSL | Boost Software License | A short permissive license (BSL-1.0); not to be confused with the Business Source License. | [Lesson 20.10](20-reliability-and-hardening/10-project-harden-a-service.md) |
| BSS | Block Started by Symbol | The section of zero-initialised global data; the kernel must fill it with zeros itself at boot. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| BST | Binary Search Tree | A binary tree in which every node's left subtree holds smaller values and its right subtree holds bigger values. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| BTF | BPF Type Format | Type information about the kernel's own structures, in /sys/kernel/btf/vmlinux. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| CA | Certificate Authority | An organisation that checks who you are and signs certificates. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| CAP | Consistency, Availability, Partition tolerance | The CAP theorem: during a network partition a system cannot be both linearizable and available to every request. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| CAP_SYS_ADMIN | capability: system administration | The broad Linux capability needed, for example, to mount file systems and create most namespaces. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| CAS | compare-and-swap | An atomic operation: if the value is still the expected one, replace it with a new one; otherwise report the actual value. In Rust: compare_exchange. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| CBC | Cipher Block Chaining | An older block-cipher mode that hides data but, without a MAC, does not detect changes; prefer an AEAD cipher. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| CBMC | C Bounded Model Checker | The model-checking engine that Kani uses under the hood; Kani translates Rust code into a form CBMC can check. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| CD | Continuous Delivery / Deployment | Automatically delivering or deploying the result of CI. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| CEL | Common Expression Language | A small rule language that the Kubernetes API server uses to validate fields of custom resources. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| cfg | configuration | As in #[cfg(test)]: an attribute that compiles the next item only under a given configuration (here, when testing). | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| CFS | Completely Fair Scheduler | The Linux scheduler before kernel 6.6, replaced by EEVDF. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| cgroup | control group | A Linux feature that groups processes so the system can track them, limit their resources and stop them together; systemd gives every service its own cgroup. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| char | character | The type of one Unicode character, written in single quotes. It takes 4 bytes. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| CI | Continuous Integration | A service that automatically builds and tests your code on every push or pull request. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| CI/CD | Continuous Integration / Continuous Delivery | The whole automated pipeline: build and test every change (CI), then deliver or deploy it (CD). | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| CIA | Confidentiality, Integrity, Availability | The three basic goals of security: secrets stay secret, data is not changed illegally, and the service works when needed. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| CIDR | Classless Inter-Domain Routing | The address/prefix-length notation such as 192.168.1.0/24, and the rule that a subnet can have any prefix length. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| CISA | Cybersecurity and Infrastructure Security Agency | The United States agency that publishes security alerts, for example about UDP amplification attacks. | [Lesson 15.5](15-networking/05-udp-sockets-in-rust.md) |
| CLA | Contributor License Agreement | A legal agreement, signed once, that gives a project the rights it needs to use your contributions. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| clap | Command Line Argument Parser | The standard crate for parsing command-line arguments. It generates --help and error messages from a struct. | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| CLI | Command-Line Interface | Using a program by typing commands and options in a terminal. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| CMake | Cross-platform Make | A tool that generates and drives builds of C and C++ code; some Rust crates need it to build bundled C libraries. | [Lesson 22.20](22-specializations/20-networking-track-production-proxies.md) |
| CMU | Carnegie Mellon University | A university in the USA whose free database course (15-445/645) is a common next step after this phase. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| CN | Common Name | The name field of a certificate subject (for example CN=rust-lang.org). Modern clients check the SAN instead. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| CNAME | Canonical NAME | A DNS record that says a name is an alias of another (canonical) name. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| CNCF | Cloud Native Computing Foundation | A non-profit foundation that hosts open-source cloud-native projects such as Kubernetes, Prometheus, Helm and OpenTelemetry. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| CNI | Container Network Interface | The standard for plug-ins that give Kubernetes pods their network addresses; kind installs one called kindnet. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| CORS | Cross-Origin Resource Sharing | Browser rules, set by response headers, that decide which other websites may call an HTTP API from JavaScript. | [Lesson 16.4](16-backend-services/04-errors-validation-and-middleware.md) |
| COW | Copy-On-Write | After fork, parent and child share memory pages until one writes; only then is the page copied. | [Lesson 13.5](13-operating-systems/05-fork-exec-and-wait.md) |
| CP | Consistent under Partition | A CAP label for a system that stays linearizable during a network partition and refuses some requests instead (for example a Raft cluster). | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| CPU | Central Processing Unit | The chip (processor) that runs the instructions of programs. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| CQ | Completion Queue | The io_uring ring where the kernel puts the results of operations. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| CQE | Completion Queue Entry | One io_uring result: the user_data number and a result (bytes, or a negative error number). | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| CRC | Cyclic Redundancy Check | A family of fast checksums that are very good at catching damaged bytes; CRC-32 gives a 32-bit number. | [Lesson 17.3](17-storage-and-databases/03-binary-file-formats.md) |
| CRC-32C | Cyclic Redundancy Check, 32 bits, Castagnoli polynomial | A 32-bit checksum used to detect corrupted batches on disk and in the network. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| CRC32C | Cyclic Redundancy Check, 32-bit, Castagnoli polynomial | A 32-bit checksum that many storage engines and logs use. Modern CPUs compute it in hardware. | [Lesson 18.15](18-distributed-systems/15-checkpoint-stage-4-complete.md) |
| CRD | CustomResourceDefinition | A Kubernetes object that adds a new object type to the API, with its names and a schema of allowed fields. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| CRI | Container Runtime Interface | The interface between the kubelet and the container runtime. It also defines the line format of pod log files: time, stream, flag and text. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| CRLF | Carriage Return, Line Feed | The two bytes \r\n (13 and 10) that end every line in the head of an HTTP message. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| CRT | C runtime | The startup code and C library that run before main and provide functions like malloc; -C target-feature=+crt-static links it statically. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| CS:APP | Computer Systems: A Programmer's Perspective | A classic textbook by Bryant and O'Hallaron about how programs run on real hardware. It uses C and x86-64. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| CSPRNG | Cryptographically Secure Pseudo-Random Number Generator | A source of random numbers that an attacker cannot predict; the operating system provides one. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| CSR | Compressed Sparse Row | A layout for graphs and sparse matrices: all neighbour lists in one array plus an array of start offsets. | [Lesson 19.6](19-performance-and-profiling/06-data-layout-for-speed.md) |
| CSRF | Cross-Site Request Forgery | An attack that tricks a logged-in user's browser into sending an unwanted request to a site. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| CSV | Comma-Separated Values | A simple text format for tables: one row per line, columns separated by commas. | [Lesson 08.2](08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) |
| CV | curriculum vitae | Latin for 'course of life'. In many countries the same as a resume; in some a longer, complete record of your work and education. | [Lesson 24.6](24-career-and-open-source/06-resume-and-job-search.md) |
| CVE | Common Vulnerabilities and Exposures | A public ID (like CVE-2024-12345) for one known security hole. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| CVS | Concurrent Versions System | A very old centralized version control system, the ancestor of Subversion. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| DAG | Directed Acyclic Graph | A directed graph without cycles; dependencies between tasks normally form a DAG. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| DAIF | Debug, SError, IRQ, FIQ | The arm64 register with the four mask bits that switch kinds of exceptions off and on. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| DB | database | Short form of 'database' in diagrams and names. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| DCO | Developer Certificate of Origin | A short statement that you wrote a change or have the right to submit it under the project's licence. You agree to it with a Signed-off-by line (git commit -s). | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| DDL | Data Definition Language | The part of SQL that changes the structure of a database: CREATE TABLE, ALTER TABLE, DROP TABLE. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| DER | Distinguished Encoding Rules | The binary encoding of certificates and keys; PEM is DER in base64 text. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| DES | Data Encryption Standard | An old, broken cipher; it survives in the name of the OpenSSL option -nodes ('no DES': do not encrypt the key file). | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| DFD | Data-Flow Diagram | A simple picture of a system: who talks to it, which programs run, where data is stored and how data moves. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| DFS | Depth-First Search | A graph search that goes as deep as possible along one path before backing up. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| DHAT | Dynamic Heap Analysis Tool | Valgrind's heap profiler. The dhat crate and its web viewer (dh_view) are named after it and use its report format. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| DI | Dependency Injection | Giving a type its dependencies from outside instead of letting it create them. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| DMA | Direct Memory Access | Hardware that copies data between memory and peripherals without using the CPU. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| DML | Data Manipulation Language | The part of SQL that changes the data itself: INSERT, UPDATE, DELETE. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| DNS | Domain Name System | The internet's phone book: it turns a name like example.com into an IP address. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| DNSSEC | DNS Security Extensions | Digital signatures on DNS records, so that a resolver can check that an answer is genuine. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| DoH | DNS over HTTPS | DNS messages sent as HTTPS requests, so they are encrypted and look like web traffic. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| DoS | Denial of Service | An attack or bug that makes a service unusable, for example by making it loop forever or flooding it with data. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| DoT | DNS over TLS | DNS messages sent inside an encrypted TLS connection. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| DP | Dynamic Programming | Solving a big problem by solving smaller overlapping problems once and remembering the answers. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| dpkg | Debian package | The low-level tool that installs one .deb file and records it in the database of installed packages. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| DRAM | Dynamic Random Access Memory | Cheap, dense memory that stores each bit in a tiny capacitor. The main memory (RAM) is made of it. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| DRY | Don't Repeat Yourself | Principle: one idea or number should live in one place (a function or a constant), not be copied in many places. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| DSL | Domain-Specific Language | A tiny language made for one job, such as calc!(add 2, 3). | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| DSO | Dynamic Shared Object | A file that holds code in perf's reports: your program, a shared library such as libc, or the kernel. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| DST | Dynamically Sized Type | A type whose size is not known at compile time: str, [T] and dyn Trait. It always lives behind a pointer. | [Lesson 06.2](06-rust-advanced/02-advanced-traits-and-types.md) |
| DTB | Device Tree Blob | A file that describes the hardware of a machine; QEMU puts one at the start of RAM. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| E2E | End-to-End | A test that starts the real finished program and checks what a user would see, from input to output. | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| e2e test | end-to-end test | A test that runs the whole system the way a user would, for example against a real cluster with the operator running. | [Lesson 22.28](22-specializations/28-cloud-native-track-testing-and-running-operators.md) |
| EABI | Embedded Application Binary Interface | The rules for calling functions and laying out data on embedded Arm chips. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| EAGAIN | Error: try AGAIN | The error a non-blocking call returns when it would have to wait; in Rust io::ErrorKind::WouldBlock. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| eBPF | extended Berkeley Packet Filter | A small virtual machine inside the Linux kernel that runs checked programs when events happen. The name is historical. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| ECB | Electronic Codebook | An old, insecure way to apply a block cipher: equal input blocks give equal output blocks, and nothing detects changes. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| ECDSA | Elliptic Curve Digital Signature Algorithm | A signature algorithm with small keys; rcgen uses it with the P-256 curve by default. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| EDNS | Extension Mechanisms for DNS | An OPT pseudo-record in a DNS message that allows bigger UDP answers and new options. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| EEVDF | Earliest Eligible Virtual Deadline First | The algorithm of Linux's normal scheduler since kernel 6.6; it shares the CPU fairly by weight. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| EL | Exception Level | The privilege level of an arm64 CPU: EL0 for normal programs, EL1 for the kernel. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| ELF | Executable and Linkable Format | The file format of executables, shared libraries and object files on Linux and most Unix systems. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| ELR | Exception Link Register | The address where the interrupted code continues after the eret instruction. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| enum | enumeration | A type with a fixed list of possible values (variants), such as Ordering. | [Lesson 02.10](02-programming-basics/10-project-guessing-game.md) |
| EOF | End Of File | The signal that there is no more input. read_line returns Ok(0). Ctrl+D sends it in a terminal. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| EOIR | End Of Interrupt Register | A GIC register; writing an INTID to it marks that interrupt as handled. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| errno | error number | A per-thread variable where libc stores why the last call failed, for example 2 = ENOENT. | [Lesson 13.3](13-operating-systems/03-system-calls.md) |
| ESR | Exception Syndrome Register | A system register that says why an exception happened: the exception class and details. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| FAQ | Frequently Asked Questions | A page with answers to the questions people ask most often. | [Lesson 04.8](04-structs-enums-collections-errors/08-strings-in-depth.md) |
| FAR | Fault Address Register | A system register with the address that a faulting memory access tried to use. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| FD | File Descriptor | A small number the OS gives to an open file, socket or pipe. 0, 1, 2 are stdin, stdout, stderr. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| FFI | Foreign Function Interface | A way for Rust code to call code written in another language, such as C (and the reverse). #[repr(C)] fixes struct layout for it. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| FHS | Filesystem Hierarchy Standard | The agreement about what the top-level directories of a Linux file system (/etc, /var, /usr and others) are used for. | [Lesson 11.2](11-linux-shell-and-containers/02-the-linux-file-system.md) |
| FIFO | First In, First Out | The rule of a queue: the item added first is removed first. | [Lesson 07.5](07-algorithms-and-data-structures/05-stacks-and-queues.md) |
| FIN | finish | The TCP flag that says 'I will send no more data'; each direction is closed with its own FIN. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| FIQ | Fast Interrupt Request | A second, high-priority kind of interrupt on Arm CPUs; normal kernels rarely use it. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| FLP | Fischer, Lynch and Paterson | The FLP result (1985): if messages may be delayed without limit, no algorithm can guarantee that consensus always finishes when even one node may crash. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| FNV | Fowler–Noll–Vo | A family of simple, fast, non-cryptographic hash functions; FNV-1a is the variant used in this course for Bloom filters. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| FQDN | Fully Qualified Domain Name | A complete DNS name that ends with the root dot, such as rust-lang.org., so nothing is appended to it. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| FSM | Finite State Machine | A model of something that is always in exactly one of a fixed list of states and moves between them on events. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| FTL | Flash Translation Layer | Software inside an SSD that maps the block numbers the computer uses to real places in the flash chips. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| futex | fast user-space mutex | A Linux system call that lets a thread sleep until a number in memory changes, and wakes it later. Rust's locks use it on Linux. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| GB | gigabyte | About one billion bytes (1000 MB); GiB is the binary version, 1024 MiB. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| GCM | Galois/Counter Mode | The mode in which AES is used as an AEAD cipher (AES-GCM). | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| GCRA | Generic Cell Rate Algorithm | A rate-limiting algorithm, a variant of the token bucket, used by the governor crate. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| GHCR | GitHub Container Registry | GitHub's image registry at ghcr.io; workflows can push to it with the built-in GITHUB_TOKEN. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| GHz | gigahertz | A billion cycles per second; CPU clock speeds are a few GHz. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| GiB | gibibyte | A unit of data size equal to 1,073,741,824 bytes; GiB/s is gibibytes per second. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| GIC | Generic Interrupt Controller | The Arm chip that collects interrupts from devices and delivers them to CPU cores. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| GID | Group ID | The number that identifies a group to the kernel. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| glibc | GNU C Library | The C library of most Linux distributions; normal Rust programs on Linux use it, so the runtime image needs a compatible version. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| GND | ground | The 0-volt reference side of an electric circuit. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| GNU | GNU's Not Unix | A large free-software project (tools such as bash and GCC). The name is a joke that refers to itself. GPG (GNU Privacy Guard) is one of its programs. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| GPG | GNU Privacy Guard | Another kind of key and tool for signing data and commits. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| GPIO | General-Purpose Input/Output | A chip pin that a program can switch on and off or read. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| GPL | General Public License | A family of open-source licenses that place conditions on the code that uses the licensed code. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| GPS | Global Positioning System | A satellite system; its satellites carry atomic clocks, so a GPS receiver can also provide very exact time. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| grep | Global Regular Expression Print | A classic Unix program that prints the lines of a file that match a pattern. minigrep is a small version. | [Lesson 05.13](05-rust-intermediate/13-project-minigrep.md) |
| gRPC | gRPC Remote Procedure Calls | A fast RPC framework from Google that uses HTTP/2 and Protocol Buffers. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| GUI | Graphical User Interface | Using a program with windows, buttons and the mouse. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| HAL | Hardware Abstraction Layer | A library that gives safe, convenient types and functions for the hardware of one chip family. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| HCL | HashiCorp Configuration Language | The language of OpenTofu and Terraform files (.tf): named blocks with key = value lines. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| HDD | Hard Disk Drive | A storage device with spinning magnetic platters and a moving read/write arm; every random access needs a slow mechanical seek. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| HDR | High Dynamic Range | In hdrhistogram: one histogram covers both tiny and huge values with fixed precision. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| HIR | High-level Intermediate Representation | The compiler's tree form of Rust code after macro expansion and name resolution; type checking works on it. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| HKDF | HMAC-based Key Derivation Function | A function built from HMAC that turns a shared secret into several encryption keys. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| HMAC | Hash-based Message Authentication Code | A way to sign data with a shared secret key; HS256 means HMAC with SHA-256. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| HOT | Heap-Only Tuple | A PostgreSQL optimization: an update that changes no indexed column puts the new row version on the same page, without new index entries. The old slot becomes a redirect. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| HPA | HorizontalPodAutoscaler | A Kubernetes object that changes the number of replicas of a Deployment automatically, for example by CPU usage. | [Lesson 21.13](21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) |
| HRTB | Higher-Ranked Trait Bound | A bound written with `for<'a>` that must hold for every lifetime 'a, not for one chosen lifetime. | [Lesson 06.5](06-rust-advanced/05-lifetimes-in-depth.md) |
| HS256 | HMAC with SHA-256 | A JWT signature algorithm that uses one shared secret key with HMAC-SHA-256. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| HTML | HyperText Markup Language | The language of web pages. rustdoc generates documentation as HTML. | [Lesson 05.7](05-rust-intermediate/07-documentation.md) |
| HTTP | HyperText Transfer Protocol | The text-based language browsers and web servers speak on top of TCP. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| HTTP/2 | Hypertext Transfer Protocol version 2 | A binary version of HTTP in which one TCP connection carries many requests at the same time. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| HTTPS | HyperText Transfer Protocol Secure | The web protocol with encryption. Addresses start with `https://`. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| I/O | Input/Output | Everything that moves data into or out of a program or computer: keyboard, screen, disk, network. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| I2C | Inter-Integrated Circuit | A simple two-wire bus that connects a microcontroller to sensors and other chips. | [Lesson 22.30](22-specializations/30-checkpoint-your-specialization.md) |
| IaC | Infrastructure as Code | Describing servers and cloud resources in files that are versioned and reviewed like code. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| IANA | Internet Assigned Numbers Authority | The organisation that manages the DNS root zone, protocol numbers and address ranges. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| IAR | Interrupt Acknowledge Register | A GIC register; reading it returns the INTID of the pending interrupt and marks it active. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| IC | Incident Commander | The person who leads an incident response: keeps the overview, decides and assigns tasks, but does not fix things personally. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| ICMP | Internet Control Message Protocol | The small control messages used by ping and traceroute. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| ID | Identifier | A name or number that points to exactly one thing, for example a user ID or a commit ID (its hash). | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| IDE | Integrated Development Environment | A program for writing code that combines an editor, run buttons, error marks and other helpers. The course uses RustRover. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| IDL | Interface Definition Language | A small language that only describes an interface (functions and data), not how it works. The .proto language is one. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| IDOR | Insecure Direct Object Reference | A flaw where the program uses an identifier from the request without checking that this user may access that object. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| IEC | International Electrotechnical Commission | An international standards body. It defined the binary size units KiB, MiB and GiB (powers of 1024). | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| IEEE | Institute of Electrical and Electronics Engineers | A professional organisation that publishes technical standards, including IEEE 754 for floating-point numbers. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| IEEE 754 | Institute of Electrical and Electronics Engineers, standard 754 | The international standard describing how computers store floating-point numbers. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| IETF | Internet Engineering Task Force | The organisation that writes internet standards, published as RFC documents. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| IHL | Internet Header Length | The 4-bit field of the IPv4 header that gives the header length in 32-bit words. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| ILP | Instruction-Level Parallelism | Running several independent instructions of one program at the same time inside one CPU core. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| IMSC | Interrupt Mask Set/Clear register | The PL011 UART register that enables its interrupts, for example 'byte received'. | [Lesson 22.5](22-specializations/05-os-track-project-and-next-steps.md) |
| INI | initialization (file format) | A simple settings-file format with [sections] and key = value lines. ~/.gitconfig uses it. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| inode | index node | The kernel's record of one file: type, size, owner, permissions and where its data lives, but not its name. | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| INTID | interrupt ID | The number that the GIC gives each interrupt source, for example 30 for the EL1 physical timer. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| IOPS | Input/Output Operations Per Second | How many separate reads or writes a storage device finishes per second. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| IP | Internet Protocol (IP address) | The number that names a computer on a network, for example 127.0.0.1. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| IPC | Inter-Process Communication | Ways for processes to exchange data; the IPC namespace isolates System V shared memory and message queues. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| IPv4 | Internet Protocol version 4 | The older and most common kind of IP address, four numbers such as 172.30.0.2. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| IPv4 / IPv6 | Internet Protocol version 4 / version 6 | The two versions of IP in use: IPv4 addresses are 32 bits long, IPv6 addresses 128 bits. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| IPv6 | Internet Protocol version 6 | The newer, longer kind of IP address, for example ::1 for the loopback. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| IR | Intermediate Representation | A form of a program between source code and machine code, made for analysis and optimisation. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| IRQ | Interrupt Request | The normal kind of hardware interrupt on arm64, for example from a timer. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| ISA | Instruction Set Architecture | The instructions, registers and rules of a CPU family; the contract between machine code and hardware. arm64 and x86-64 are ISAs. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| ISR | Interrupt Service Routine | The function that runs when a hardware interrupt happens; also called an interrupt handler. | [Lesson 22.12](22-specializations/12-embedded-track-async-with-embassy.md) |
| ITU | International Telecommunication Union | The United Nations agency that publishes radio and telecom standards, including the official Morse code. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| I²C | Inter-Integrated Circuit | A two-wire bus for talking to sensors and other small chips. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| JIT | Just-In-Time | A compiler that turns code into machine code while the system runs; the kernel's eBPF JIT does this for checked programs. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| JNZ | Jump if Not Zero | A machine instruction: "if the last result was not zero, continue at another place in the program". | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| JSON | JavaScript Object Notation | A popular text format for data, for example {"name": "Ana", "age": 30}. | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| JWT | JSON Web Token | A signed text token that carries claims about a user, often used to stay logged in to an API. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| K8s | Kubernetes (K, 8 letters, s) | A short way to write Kubernetes, the system that runs and manages containers across many machines. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| KB | kilobyte | 1000 bytes in the SI meaning. MB = 1000 KB, GB = 1000 MB. Some tools use KB for 1024 bytes, so check. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| KB / MB / GB | kilobyte / megabyte / gigabyte | About a thousand / a million / a billion bytes. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| KDF | Key Derivation Function | A function that turns a password (or another secret) into a key of the right length. Slow ones such as Argon2 are used for passwords. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| KEM | Key Encapsulation Mechanism | A way to agree on a shared secret by wrapping it with the other side's public key; ML-KEM is the post-quantum one used in TLS. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| KiB | kibibyte | 1024 bytes. Argon2 memory cost is measured in KiB. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| KiB / MiB | kibibyte / mebibyte | 1 KiB = 1,024 bytes; 1 MiB = 1,024 KiB = 1,048,576 bytes. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| KiB / MiB / GiB | kibibyte / mebibyte / gibibyte | Units of memory size based on 1,024: 1 KiB = 1,024 bytes, 1 MiB = 1,024 KiB, 1 GiB = 1,024 MiB. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| kind | Kubernetes IN Docker | A tool from the Kubernetes project that creates a whole Kubernetes cluster whose nodes are Docker containers. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| KIP | Kafka Improvement Proposal | Kafka's public design document for one change, such as KIP-98 for idempotent producers and transactions. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| KISS | Keep It Simple, Stupid | Principle: prefer the simplest solution that works. Keep functions small, with one job. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| km | kilometre | A unit of distance equal to 1,000 metres; the unit of road lengths in the route finder. | [Lesson 07.19](07-algorithms-and-data-structures/19-project-route-finder.md) |
| KV | key-value | Storage that keeps values under keys; also the name of the example protocol of Lesson 15.7. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| L1 / L2 / L3 | layer 1 / layer 2 / layer 3 | OSI layer numbers for the physical cable, the link layer (Ethernet, MAC addresses) and the internet layer (IP). | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| L1, L2, L3 | Level 1, Level 2, Level 3 (cache) | The cache levels of a CPU. L1 is the smallest and fastest and sits next to the core; each next level is bigger and slower. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| L4 / L7 | Layer 4 / Layer 7 | Short names for the transport layer (ports, TCP, UDP) and the application layer (HTTP, DNS) from the OSI model. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| LB | load balancer | A component that spreads incoming requests over several copies of a service. It is written as LB in diagrams. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| LCOV | (name of the lcov coverage tool) | A plain-text file format for code-coverage data (usually lcov.info) that many editors and coverage services read. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| LCS | Longest Common Subsequence | The longest sequence of characters that is a subsequence of both of two texts. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| LEB128 | Little-Endian Base 128 | The varint encoding that stores 7 bits of a number per byte, lowest bits first, with the top bit meaning 'more bytes follow'. | [Lesson 17.3](17-storage-and-databases/03-binary-file-formats.md) |
| LED | Light-Emitting Diode | A small electronic lamp that lights up when current flows through it. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| LGTM | Looks Good To Me | A common short approval comment in a code review. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| libc | C standard library | The standard C library with functions such as strlen, abs, malloc, free. It is already linked into Rust programs. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| LIFO | Last In, First Out | The rule of a pile: the last item put on is the first one taken off. This is how the stack works. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| LLDB | (the debugger of the LLVM project) | The debugger used on macOS and inside RustRover: breakpoints, stepping, watching variables. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| LLVM | (once: Low Level Virtual Machine) | The code-generating engine under the Rust compiler. It turns the compiler's output into machine code and optimizes it. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| LRU | Least Recently Used | A cache that, when full, throws away the item that has not been used for the longest time. | [Lesson 07.4](07-algorithms-and-data-structures/04-linked-lists.md) |
| LSan | LeakSanitizer | A sanitizer that reports memory that was allocated and never freed when the program ends. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| LSB | Least Significant Byte first | In ELF headers and file output, LSB means the file stores numbers little-endian. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| LSE | Large System Extensions | A part of the arm64 instruction set (since ARMv8.1) with single instructions for atomic operations, such as ldadd. Apple M chips have it. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| LSM | Log-Structured Merge-tree | A storage design that buffers writes in memory and merges sorted files on disk. Fast for writes. | [Lesson 17.4](17-storage-and-databases/04-write-ahead-log.md) |
| LT / ET | level-triggered / edge-triggered | Two epoll modes: report a descriptor as long as it is ready (LT), or only when it becomes ready (ET). | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| LTO | Link-Time Optimisation | Optimising the whole program across crates when it is linked, at the cost of longer build times. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| LTS | Long-Term Support | A version of software that gets fixes for a long time. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| LWN | Linux Weekly News (now LWN.net) | A long-running news site about the Linux kernel; its article series explain kernel features such as namespaces. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| LWP | Light-Weight Process | An old name for a thread; ps -L shows TIDs in the LWP column. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| LWW | Last Write Wins | A conflict rule that keeps the value with the latest timestamp and throws the others away. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| MAC | Message Authentication Code | A short value that proves a message was not changed and comes from someone who knows the secret key. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| MAC (network) | Media Access Control | The link-layer address of a network interface, used for the next hop on a local network. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| Mach-O | Mach object | The file format of executables, libraries and object files on macOS. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| MAIR | Memory Attribute Indirection Register | The arm64 system register that defines memory types (device, normal) that page table entries refer to by index. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| MB | megabyte | One million bytes, a little less than one MiB. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| MB / GB / TB | megabyte / gigabyte / terabyte | Decimal size units: 1 MB = 1,000,000 bytes, 1 GB = 10^9 bytes, 1 TB = 10^12 bytes. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| MCU | Microcontroller Unit | A microcontroller: one small chip with a CPU core, flash memory, RAM and peripherals. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| MD5 | Message-Digest algorithm 5 | An old hash function that is broken for security use (collisions can be produced). Fine only as a checksum against accidents. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| MDN | Mozilla Developer Network | A widely used web documentation site (developer.mozilla.org). | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| MFA | Multi-Factor Authentication | Asking for two or more different kinds of proof of identity, for example a password and a code from a phone. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| Mi | mebibyte | 1,048,576 bytes (2 to the power 20); Kubernetes writes memory sizes as 16Mi or 1Gi. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| MiB | mebibyte | A unit of data size equal to 1,048,576 bytes; MiB/s is mebibytes per second. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| MiB / GiB / TiB | mebibyte / gibibyte / tebibyte | Binary size units: 1 MiB = 2^20 bytes, 1 GiB = 2^30 bytes, 1 TiB = 2^40 bytes. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| MIR | Mid-level Intermediate Representation | A simplified form of a Rust program inside the compiler. Miri is an interpreter for it. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| MIT | Massachusetts Institute of Technology (MIT License) | A short permissive open-source licence. Rust projects often use MIT OR Apache-2.0. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| MIT (licence) | Massachusetts Institute of Technology | A short, permissive open-source licence named after the university where it was written. Rust projects often offer it together with Apache-2.0. | [Lesson 24.2](24-career-and-open-source/02-building-your-portfolio.md) |
| MITM | man-in-the-middle | An attacker who sits between client and server, reads or changes the traffic and pretends to be each side. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| ML-KEM | Module-Lattice Key Encapsulation Mechanism | A post-quantum key exchange algorithm, combined with X25519 in modern TLS. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| MMIO | Memory-Mapped I/O | Devices that appear as memory addresses: reading or writing such an address talks to the device. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| MMU | Memory Management Unit | The part of the CPU that translates virtual addresses into physical addresses using the page table. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| MPMC | multi-producer, multi-consumer | A channel that may have many senders and many receivers, such as crossbeam::channel. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| mpsc | multiple producer, single consumer | The kind of channel in std::sync::mpsc: many senders, one receiver. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| MQTT | Message Queuing Telemetry Transport | A lightweight publish/subscribe messaging protocol over TCP, common for small devices. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| MR | Merge Request | The GitLab name for a pull request. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| MRE | Minimal Reproducible Example | The smallest complete program that still shows a problem. It lets a maintainer see a bug in seconds. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| ms | millisecond | One thousandth of a second. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| MSan | MemorySanitizer | A sanitizer that finds reads of uninitialized memory; it works only on Linux. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| MSRV | Minimum Supported Rust Version | The oldest Rust compiler version a crate promises to work with (the rust-version field). | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| MSS | Maximum Segment Size | The largest amount of data one TCP segment may carry, announced during the handshake. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| MTBF | Mean Time Between Failures | The average time a system works between two failures. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| mTLS | mutual TLS | TLS in which the client also presents a certificate and the server verifies it. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| MTTR | Mean Time To Recovery | The average time from a failure until the system works again; some teams read the R as "repair". | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| MTU | Maximum Transmission Unit | The largest packet one network link can carry; 1500 bytes on Ethernet. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| MVCC | Multi-Version Concurrency Control | A database technique that keeps several versions of a row so readers and writers do not block each other. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| MVP | Minimum Viable Product | The smallest version of a product that is already useful to a real user. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| MX | Mail eXchange | A DNS record that names a mail server for a domain, with a preference number (lower is preferred). | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| NaN | Not a Number | A special float value for an impossible result such as 0.0 / 0.0. It is not equal to itself, so floats cannot be HashMap keys. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| NAT | Network Address Translation | A router rewrites private source addresses into its own public address on the way out and back on the way in, so many devices share one address. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| NATS | (a name, not a short form) | A fast system that sends messages between programs. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| NFC / NFD | Normalization Form C / Normalization Form D | Two standard ways to write the same Unicode text: NFC prefers composed characters (é as one code point), NFD splits them (e plus an accent). | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| NIST | National Institute of Standards and Technology | The US standards institute that publishes widely used security guidance, for example on passwords. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| NLL | Non-Lexical Lifetimes | The compiler rule that a borrow ends at the last use of the reference, not at the end of the block. | [Lesson 03.4](03-ownership-and-borrowing/04-references-and-borrowing.md) |
| ns | nanosecond | One billionth of a second. A modern CPU core does about 4–5 clock cycles in 1 ns. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| ns / µs / ms / s | nanosecond / microsecond / millisecond / second | Time units: one billionth, one millionth, one thousandth of a second, and a second. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| NTP | Network Time Protocol | The protocol computers use to set their wall clocks from time servers over the network. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| NUL | NUL (the ASCII null character, byte value 0) | The byte with value 0 that ends every C string. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| NVMe | Non-Volatile Memory Express | The fast protocol that modern SSDs use to talk to the computer over the PCIe bus. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| OCI | Open Container Initiative | The group that writes the open standards for container images and container runtimes; runc follows its runtime specification. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| OIDC | OpenID Connect | A standard for proving identity with short-lived tokens; GitHub Actions uses it to get temporary cloud credentials without stored secrets. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| OLAP | Online Analytical Processing | Database work that scans a few columns of very many rows to compute reports. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| OLTP | Online Transaction Processing | Database work made of many small reads and writes of whole rows, such as orders in a shop. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| OOD | Object-Oriented Design | Designing software around classes and objects; the SOLID principles were first written for it. | [Lesson 08.3](08-software-design/03-solid-principles.md) |
| OOM | Out Of Memory | The situation where an allocator cannot provide the requested memory; a GlobalAlloc signals it by returning a null pointer. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| ORM | Object-Relational Mapper | A library that generates SQL for you from method calls on objects; sqlx deliberately is not one. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| OS | Operating System | The main program (for example macOS or Linux) that starts other programs and gives them memory, files and the screen. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| OSI | Open Systems Interconnection | A seven-layer model of networks; its numbers are still used, for example L4 for transport and L7 for application. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| OSS | Open Source Software | Software whose source code is public and free to use. OSS-Fuzz is Google's free fuzzing service for such projects. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| OSS-Fuzz | Open Source Software Fuzz | Google's free service that fuzzes important open-source projects around the clock and reports bugs privately to their maintainers. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| OSTEP | Operating Systems: Three Easy Pieces | A free textbook about the ideas behind every operating system. | [Lesson 22.5](22-specializations/05-os-track-project-and-next-steps.md) |
| OTel | OpenTelemetry | Short name of OpenTelemetry, the open standard and libraries for creating and sending telemetry. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| OTLP | OpenTelemetry Protocol | The standard format for sending logs, metrics and traces from programs to monitoring tools. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| OWASP | Open Worldwide Application Security Project | A community that publishes security guides and the well-known "Top 10" list of web risks. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| P-core / E-core | performance core / efficiency core | Fast cores and slower, power-saving cores in the same CPU. Apple M chips have both, so equal work can finish at different times. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| p95 | 95th percentile | The value that 95% of measurements stay under, for example the latency that 95% of requests are faster than. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| p99 | 99th percentile | The value that 99 % of measurements do not exceed, for example the latency of all but the slowest 1 % of requests. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| PAC | Peripheral Access Crate | A crate generated from a chip's register description, with one type per register and no high-level logic. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| PACELC | if Partition: Availability or Consistency; Else: Latency or Consistency | An extension of CAP that also names the trade-off between latency and consistency when there is no partition. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| PAT | Personal Access Token | A long random string that works like a password for one purpose, created in GitHub or GitLab settings. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| PATH | (environment variable name, not an acronym) | The list of directories where the shell looks for commands when you type a command name. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| PC | Personal Computer | An ordinary desktop or laptop computer, for example a Windows PC. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| PCIe | Peripheral Component Interconnect Express | The internal high-speed bus that connects devices such as SSDs and graphics cards to the processor. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| PDF | Portable Document Format | A file format for documents that look the same on every computer. | [Lesson 22.17](22-specializations/17-databases-track-project-and-next-steps.md) |
| PEM | Privacy-Enhanced Mail | A text format for keys and certificates between lines such as -----BEGIN CERTIFICATE-----. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| PGID | Process Group IDentifier | The ID shared by all processes of one process group, usually one shell job. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| PGO | Profile-Guided Optimization | Building a program with counters, running it on typical work, and building it again so the compiler optimizes the paths that really run often. | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| PHC | Password Hashing Competition | The competition whose winner was Argon2; the PHC string format stores algorithm, settings, salt and hash in one text. | [Lesson 10.15](10-security-fundamentals/15-checkpoint.md) |
| PiB | pebibyte | 1024 x 1024 GiB, about 1.1 quadrillion bytes. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| PID | Process ID | The number the OS gives to each running process. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| PIE | Position-Independent Executable | An executable whose code works at any load address, so ASLR can move it; Rust builds PIE by default. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| PIN | Personal Identification Number | A short secret number or code that identifies a person to a system. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |
| PIO | Programmable Input/Output | Small state machines in the RP2040/RP2350 that drive pins on their own with exact timing. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| PKI | public key infrastructure | Everything around certificates: CAs, key pairs, certificates and the rules of trust. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| PL/pgSQL | Procedural Language/PostgreSQL | PostgreSQL's built-in programming language for small programs inside the database, such as DO blocks with IF, RAISE and EXCEPTION. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| PMR | Priority Mask Register | A GIC CPU-interface register: only interrupts with a higher priority than its value are delivered. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| PMU | Performance Monitoring Unit | The part of a CPU that counts hardware events such as cycles, instructions and cache misses. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| POSIX | Portable Operating System Interface | A standard that describes the system calls and tools a Unix-like OS should have. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| PP | program point | In dhat's report, one place in the code (with its call stack) that allocates memory. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| PPA | Personal Package Archive | A package repository run by one person or team on Launchpad, outside the official Ubuntu archive. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| PPID | Parent Process ID | The PID of the process that started this process. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| ppm | parts per million | A unit for small ratios; a clock that is off by 1 ppm gains or loses 1 µs every second. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| PR | Pull Request | A GitHub request to merge your branch into another, with a place for review, discussion and checks. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| procfs | process file system | The folder /proc, whose files the kernel creates on every read to show its internal state. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| PromQL | Prometheus Query Language | The language for asking Prometheus questions about metrics, for example request rates per status over the last minute. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| protobuf | Protocol Buffers | A compact binary format for structured messages, defined by a schema; used by gRPC and Prometheus remote write. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| PRR | Production Readiness Review | A review before launch or after a big change in which the team goes through a checklist and shows evidence for each item. | [Lesson 20.9](20-reliability-and-hardening/09-production-readiness-checklist.md) |
| PSCI | Power State Coordination Interface | An Arm standard for asking the firmware to switch CPUs or the whole machine on and off. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| PSH | push | The TCP flag asking the receiver to hand the data to the program right away. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| pthread | POSIX threads | The standard thread and lock library of Unix-like systems. On macOS, Rust's Mutex uses the pthread mutex. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| pub/sub | publish/subscribe | A messaging style where one program publishes a message to a subject and every subscriber of that subject gets a copy. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| purl | package URL | A standard text name for a package, such as pkg:cargo/regex@1.13.2, used to match SBOM entries with advisories. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| PVC | PersistentVolumeClaim | A request for storage in Kubernetes that survives pod restarts. The cluster binds it to a PersistentVolume, for example on the node's disk in kind. | [Lesson 21.13](21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) |
| QEMU | Quick Emulator | A free program that pretends to be a whole computer, so that you can boot your own kernel safely. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| QPACK | QPACK (header compression for HTTP/3) | The header compression of HTTP/3, designed so that QUIC streams do not block each other. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| QPS | Queries Per Second | The rate at which a load generator sends requests; the same idea as req/s. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| QR / RD / RA / AA / TC | Query/Response, Recursion Desired, Recursion Available, Authoritative Answer, TrunCated | Single-bit flags in the DNS header that describe the message. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| QUIC | (first "Quick UDP Internet Connections"; today just a name) | A modern network protocol used by HTTP/3. It is fast and always encrypted. | [Lesson 22.21](22-specializations/21-networking-track-project-and-next-steps.md) |
| RAII | Resource Acquisition Is Initialization | A value that owns a resource (memory, file, lock) takes it when created and releases it automatically when dropped. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| RAM | Random Access Memory | Fast working memory where running programs and their data live. It is cleared when the power goes off. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| RBAC | Role-Based Access Control | Deciding access from the user's role. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| Rc | Reference Counted | A smart pointer that lets several owners in one thread share a value. The value is freed when the count reaches zero. | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| RCODE | Response CODE | The 4-bit result code in a DNS header: 0 no error, 2 server failure, 3 NXDOMAIN. | [Lesson 15.11](15-networking/11-project-dns-resolver.md) |
| README | read me (file name) | A text file at the top of a project that says what it is and how to use it. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| Redis | REmote DIctionary Server | A popular server that keeps keys and values in memory; used for caches, counters, sessions and queues. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| ReDoS | Regular expression Denial of Service | An attack with a text that makes a slow regular-expression engine run for a very long time. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| regex | regular expression | A short pattern language for text, such as [0-9]+ for one or more digits. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| REPL | Read-Eval-Print Loop | A program that repeats: read a command, run it, print the result. | [Lesson 04.11](04-structs-enums-collections-errors/11-project-phone-book.md) |
| req/s | requests per second | The usual unit of throughput for a service. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| RESP | REdis Serialization Protocol | The wire protocol of Redis, which mixes delimiters and length prefixes. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| REST | Representational State Transfer | A style of web API where URLs name resources and HTTP methods (GET, POST, PUT, DELETE) act on them. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| RFC | Request for Comments | The document series where internet standards are published, for example RFC 9112 for HTTP/1.1. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| RISC | Reduced Instruction Set Computer | A CPU design with few, simple instructions; RISC-V is an open instruction set of this kind. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| RLE | Run-length encoding | A way to shorten text with repeated letters by writing a count and a letter, for example aaabcc becomes 3a1b2c. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| rlib | Rust library | The archive format in which Cargo stores a compiled Rust crate: object files plus metadata. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| RMW | read-modify-write | One atomic operation that reads, computes and writes with no gap, such as swap, fetch_add or compare_exchange. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| RNG | Random Number Generator | Code that produces numbers that look random. A seeded RNG gives the same sequence for the same seed, which makes tests repeatable. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| ROM | Read-Only Memory | Memory whose content is fixed at the factory; a chip's boot ROM holds the first code that runs. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| rootfs | root file system | The directory tree a container sees as /, usually unpacked from an image. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| RPC | Remote Procedure Call | Calling a function that runs on another computer as if it were a local call. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| RPITIT | Return-Position impl Trait In Traits | Writing -> impl Future<…> (or another impl Trait) as the return type of a trait method; it lets a trait method be async without boxing. | [Lesson 14.15](14-concurrency-and-async/15-project-concurrent-job-queue.md) |
| RPN | Reverse Polish Notation | A way of writing calculations where the operator comes after the numbers, so 3 4 + means 3 + 4. | [Lesson 09.6](09-testing-debugging-benchmarking/06-tdd-in-practice.md) |
| RR | resource record | One piece of DNS data: name, type, class, TTL and value. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| RS256 | RSA signature with SHA-256 | A JWT signature algorithm with a key pair: a private key signs, a public key verifies. RSA is named after Rivest, Shamir and Adleman. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| RSA | Rivest–Shamir–Adleman | An older, popular type of SSH key, named after its inventors. Ed25519 is the modern choice. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| RSS | Resident Set Size | How much of a process's memory is really in RAM right now (VmRSS in /proc). | [Lesson 13.2](13-operating-systems/02-processes-from-the-inside.md) |
| RST | reset | The TCP flag that aborts a connection at once; a SYN answered by RST means 'connection refused'. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| RTT | Round-Trip Time | The time a packet needs to reach the other side plus the time the answer needs to come back. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| RUSTSEC | Rust Security Advisory Database | The database of security advisories for crates on crates.io; entries have IDs such as RUSTSEC-2022-0013. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| S3 | Simple Storage Service | A popular protocol for storing files in buckets over the network. Many storage systems speak it, and OpenTofu can keep its state there. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| SAN | Subject Alternative Name | The certificate field that lists the host names the certificate is valid for. Clients check it. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| SAT | satisfiability | The question whether a logical formula can be made true; a SAT solver answers it and finds values that make it true. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| SBOM | Software Bill of Materials | A list of all components and dependencies inside a program, used to check for known vulnerabilities. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| scp | secure copy | A command that copies files between computers over SSH. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| SCTLR | System Control Register | The arm64 system register whose bits switch on the MMU and the caches. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| SDK | Software Development Kit | A set of libraries that implements something for programmers; in OpenTelemetry, the part that collects, batches and exports data. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| seccomp | secure computing mode | A Linux feature that lets a process or container runtime forbid certain system calls. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| SemVer | Semantic Versioning | A rule for version numbers MAJOR.MINOR.PATCH: MAJOR for breaking changes, MINOR for new features, PATCH for bug fixes. For 0.x crates the minor number is the breaking one. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| SeqCst | sequentially consistent | The strongest memory ordering: all SeqCst operations of all threads appear in one order that every thread agrees on. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| serde | SERialization / DEserialization | The standard Rust framework for converting data to and from text or binary formats (traits Serialize and Deserialize). | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| SError | System Error | An arm64 exception kind raised by the memory system, for example a broken bus transfer. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| SEV | severity (level) | Short for the severity level of an incident: SEV1 critical, SEV2 major, SEV3 minor. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| SHA | Secure Hash Algorithm | A family of hash functions that turn data into a short fingerprint. Git names commits with SHA-1 (and can use SHA-256). | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| SHA-1 | Secure Hash Algorithm 1 | The hash function Git uses by default. Its output is a 40-character hexadecimal string. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| SHA-256 | Secure Hash Algorithm, 256-bit | A very common fast hash function whose result always has 32 bytes. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| SI | International System of Units | The standard system of measurement units. Its prefixes kilo, mega and giga mean powers of 1000. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| SID | Session IDentifier | The ID of a session: all process groups started from one terminal login. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| SIG | Special Interest Group | A Kubernetes community group that owns one area, such as SIG API Machinery, with open meetings. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| SIGABRT | signal abort | The signal a program sends to itself to stop at once (exit code 134). Rust uses it when a panic tries to unwind out of an extern "C" function. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| SIGINT | signal interrupt | The signal the terminal sends when you press Ctrl-C. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| SIGKILL | signal kill | The signal that ends a process at once; it cannot be caught, so nothing is cleaned up. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| SIGTERM | signal terminate | The polite "please terminate" signal that service managers such as systemd and Kubernetes send to stop a program. | [Lesson 14.14](14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| SIMD | Single Instruction, Multiple Data | A CPU feature that does the same operation on several values at once. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| SIO | Single-cycle I/O | A block of the RP2350 that switches and reads GPIO pins in one CPU cycle. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| SKB | socket buffer (sk_buff) | The Linux kernel's structure for one network packet; 'SKB mode' is generic XDP, which runs after this structure is built. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| SLA | Service Level Agreement | A promise in a contract with customers, with consequences if it is broken. It is looser than the internal SLO. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| SLC | System Level Cache | The last cache level on Apple chips, shared by the CPU cores, the graphics chip and other parts of the chip. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| SLI | Service Level Indicator | A measured number that tells how well a service is doing, such as the share of successful requests. SLOs are set on SLIs. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| SLO | Service Level Objective | A target for how reliable a service should be, for example 99.9% of requests succeed. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| SLSA | Supply-chain Levels for Software Artifacts | A framework that describes levels of trust in how software is built, up to signed build provenance. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| SMP | Symmetric MultiProcessing | A kernel that can use many CPU cores of the same kind at the same time. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| SMTP | Simple Mail Transfer Protocol | The protocol used to send email between servers. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| SNI | Server Name Indication | A TLS field in which the client sends the host name it wants, so a server hosting many sites shows the right certificate. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| SoA | Struct of Arrays | A data layout with one array per field, so all values of one field lie next to each other. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| SOLID | Single responsibility, Open-closed, Liskov, Interface segregation, Dependency inversion | Five classic design principles for keeping code easy to change. | [Lesson 07.20](07-algorithms-and-data-structures/20-checkpoint.md) |
| SOPS | Secrets OPerationS | A tool that keeps secret files in Git only in encrypted form. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| SP | Special Publication | The NIST series of security guides, for example SP 800-63B on passwords. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| SPDX | Software Package Data Exchange | A standard list of short license names such as MIT and Apache-2.0. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| SPI | Shared Peripheral Interrupt | A GIC interrupt from a device that any core could handle, such as the UART on the virt machine. | [Lesson 22.5](22-specializations/05-os-track-project-and-next-steps.md) |
| SPI | Serial Peripheral Interface | A fast four-wire bus between a microcontroller and peripheral chips. Not the same as the Shared Peripheral Interrupt above. | [Lesson 22.30](22-specializations/30-checkpoint-your-specialization.md) |
| SPSC | single-producer, single-consumer | A queue with exactly one thread that adds and exactly one thread that removes; it can be built without compare-and-swap. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| SPSR | Saved Program Status Register | Holds the processor state (flags, interrupt masks) of the interrupted code during an exception. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| SQ | Submission Queue | The io_uring ring where a program puts operations for the kernel. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| SQE | Submission Queue Entry | One operation in the io_uring submission queue: what to do, on which descriptor, with which buffer. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| SQL | Structured Query Language | The language for asking questions of and changing data in relational databases. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| SQLSTATE | SQL state (standard error code) | A five-character error code defined by the SQL standard, for example 40001 for a serialization failure. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| SRAM | Static Random Access Memory | Fast, expensive memory that needs several transistors per bit. Caches are made of it. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| SRE | Site Reliability Engineering | The practice (and job) of running services reliably using software engineering. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| ss | socket statistics | The Linux command that lists sockets and their states, for example ss -tan. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| SSA | Server-Side Apply | Writing a Kubernetes object by sending the whole desired object; the API server merges it and records which fields each writer owns. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| SSD | Solid-State Drive | Storage that keeps files when the power is off. It is slower than RAM. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| SSE | Streaming SIMD Extensions | The 128-bit SIMD instruction sets of x86-64 CPUs. SSE2 is present on every x86-64 CPU. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| SSE2 | Streaming SIMD Extensions 2 | The 128-bit SIMD instruction set that every x86-64 CPU has, so the compiler uses it by default. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| SSH | Secure Shell | A protocol for secure connections to another computer. Git can log in to GitHub or GitLab with an SSH key. | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| SSI | Serializable Snapshot Isolation | PostgreSQL's method for the SERIALIZABLE level: snapshots plus tracking of read/write dependencies; a transaction is aborted when a dangerous pattern appears. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| SSL | Secure Sockets Layer | The old name and predecessor of TLS. People still say "SSL" when they mean TLS. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| SSRF | Server-Side Request Forgery | An attack that makes your server send a request to an address chosen by the attacker, often an internal one. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| SSTable | Sorted String Table | An immutable file of key-value records sorted by key, with an index; the on-disk part of an LSM tree. | [Lesson 17.7](17-storage-and-databases/07-lsm-trees.md) |
| STAR | Situation, Task, Action, Result | A structure for telling a story about your past work in a behavioural interview; most words go to the Action. | [Lesson 24.7](24-career-and-open-source/07-coding-and-system-design-interviews.md) |
| std | standard library | The library that ships with Rust. Its modules are written std::io, std::cmp, and so on. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| stderr | standard error | A second output stream for errors and diagnostics. eprintln! and dbg! write here. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| stdin | standard input | The normal input stream of a program: the keyboard in a terminal, or the output of another command through a pipe. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| stdout | standard output | The normal output stream of a program. println! writes here. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| strace | system call trace | A tool that runs a program and prints every system call it makes, with arguments and results. | [Lesson 13.3](13-operating-systems/03-system-calls.md) |
| STRIDE | Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege | A checklist of six kinds of threats used when thinking about how a system can be attacked. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| svc | SuperVisor Call | The arm64 instruction that enters the kernel to make a system call; on x86-64 it is syscall. | [Lesson 13.3](13-operating-systems/03-system-calls.md) |
| SVG | Scalable Vector Graphics | An image format made of shapes and text; flame graph files are SVG and can be zoomed in a browser. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| SVN | Subversion | An older centralized version control system. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| SWD | Serial Wire Debug | The two-wire connection Arm chips use for flashing and debugging. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| SYN | synchronise | The TCP flag that opens a connection and carries the starting sequence number. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| System V AMD64 ABI | System V Application Binary Interface for AMD64 (x86-64) | The calling convention of x86-64 on Linux and macOS: arguments in rdi, rsi, rdx, rcx, r8, r9, result in rax. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| tc | traffic control | The Linux subsystem that shapes and filters network traffic; eBPF programs can attach to it. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| TCP | Transmission Control Protocol | Rules for a reliable, ordered byte stream between two programs, implemented by the OS. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| TCR | Translation Control Register | The arm64 system register that configures address translation: address size, page size, caching of table walks. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| TDD | Test-Driven Development | Working method: write a failing test first, then the simplest code that passes, then refactor (red, green, refactor). | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| TGID | Thread Group IDentifier | The kernel's name for the PID: the ID shared by all threads of one process. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| TID | Thread IDentifier | The kernel's number for one thread; the main thread's TID equals the PID. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| TLA+ | Temporal Logic of Actions | A language for writing precise models of systems and checking them with a model checker; used for Raft and many industrial protocols. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| TLB | Translation Lookaside Buffer | A small cache inside the CPU that remembers recent translations from virtual to physical addresses. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| TLD | Top-Level Domain | The label right under the DNS root, such as org, com or uz. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| TLS | Transport Layer Security | Encryption for network connections (the S in HTTPS). | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| TOCTOU | Time Of Check To Time Of Use | A check-then-act bug in security: something is checked, then changed by someone else before it is used. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| TOML | Tom's Obvious, Minimal Language | A simple text format for settings. Cargo.toml is written in it. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| TOS | Type of Service | A byte of the IPv4 header that can mark a packet's priority. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| TPGID | Terminal Process Group IDentifier | The ID of the process group that is in the foreground of a terminal and receives Ctrl-C. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| TSan | ThreadSanitizer | A sanitizer that finds data races: two threads using the same memory at the same time without synchronization, at least one of them writing. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| TSDB | Time-Series Database | A database for values with timestamps, such as metrics. | [Lesson 23.7](23-capstone-projects/07-capstone-your-own-idea.md) |
| tt | token tree | Macro fragment specifier that matches one token or one bracketed group of tokens: the "anything" fragment. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| TTAS | test-and-test-and-set | A spin lock that waits with plain loads and tries the expensive swap only when the lock looks free, so waiting cores do not steal the cache line. | [Lesson 14.7](14-concurrency-and-async/07-building-a-spin-lock.md) |
| TTBR | Translation Table Base Register | The arm64 system register (TTBR0, TTBR1) that holds the physical address of the top-level page table. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| TTL | Time To Live | A counter or time limit: in DNS how long an answer may be cached, in ping how many routers a packet may still pass. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| TUI | Text User Interface | A screen inside the terminal that redraws itself, with bars and numbers. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| TXT | text | A DNS record type that holds free text, often used to prove domain ownership. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| UART | Universal Asynchronous Receiver-Transmitter | A simple chip that sends and receives bytes one bit at a time over a serial line. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| UB | Undefined Behaviour | A program that breaks the language rules. The compiler assumes this never happens, so the result can be anything: a crash, garbage, or apparently fine. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| UDP | User Datagram Protocol | A simple network protocol that sends separate packets with no guarantee of delivery or order. Faster and lighter than TCP. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| UI | User Interface | The part of a program that people look at and click, for example Jaeger's web pages. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| UID | User ID | The number that identifies a user to the kernel. root has UID 0. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| ULP | Unit in the Last Place | The gap between a float and the next larger float. It grows with the size of the number. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| URG | urgent | A TCP flag for urgent data; it exists in the header but is almost never used. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| URI | Uniform Resource Identifier | A name or address of a resource; in an HTTP request, the path such as /orders/7. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| URL | Uniform Resource Locator | A web address such as https://rustup.rs. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| USB | Universal Serial Bus | The common connector and bus for external devices such as flash drives. | [Lesson 11.2](11-linux-shell-and-containers/02-the-linux-file-system.md) |
| USE | Utilization, Saturation, Errors | Brendan Gregg's checklist method: for every resource (CPU, memory, disk, network), check how busy it is, how much work waits for it, and how many errors it reports. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| UTC | Coordinated Universal Time | The world reference time that other time zones are measured from. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| UTF-16 | Unicode Transformation Format, 16-bit | An encoding with 16-bit units: one unit for most characters, a surrogate pair for the others. Used by Windows and some file formats. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| UTF-8 | Unicode Transformation Format, 8-bit | The standard way to store Unicode text (all world languages) as bytes: 1 to 4 bytes per character. Rust text is always valid UTF-8. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| UTS | UNIX Time-Sharing | The namespace kind that isolates the host name and domain name; the name comes from the old kernel structure utsname. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| UUID | Universally Unique Identifier | A 128-bit number written as 36 characters that is practically unique in the world. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| V and E | number of vertices and number of edges | The size of a graph; running times of graph algorithms are written with them, for example O(V + E). | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| VBAR | Vector Base Address Register | The system register that holds the address of the exception vector table. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| VC | Version Control | Keeping every version of your files so you can see the history and undo mistakes. Git is the tool. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| VCS | Version Control System | A program that records every version of your files so you can compare, share and go back. Git is a VCS. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| vDSO | virtual Dynamic Shared Object | A tiny library the kernel maps into every process so that calls like reading the clock need not enter the kernel. | [Lesson 13.3](13-operating-systems/03-system-calls.md) |
| Vec | vector | The growable-list type in Rust. `Vec<i32>` is a vector of i32 numbers. | [Lesson 04.7](04-structs-enums-collections-errors/07-vectors.md) |
| veth | virtual Ethernet | A pair of virtual network interfaces connected like the two ends of a cable. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| VFS | Virtual File System | The OS layer that gives all file systems one common interface. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| VIP | Very Important Person | A customer who gets special treatment, such as an extra discount. | [Lesson 08.13](08-software-design/13-refactoring-and-code-review.md) |
| VM | Virtual Machine | A computer simulated by software. CI runners are often VMs. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| VPN | Virtual Private Network | A service that routes your internet traffic through another server. It can block some downloads. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| VSZ | virtual size | The total size of all mappings of a process, as shown by ps; much bigger than the memory really used. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| W3C | World Wide Web Consortium | The organization that publishes web standards, including Trace Context (the traceparent header). | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| WAL | Write-Ahead Log | A database file where each change is written first, so the data can be recovered after a crash. | [Lesson 17.4](17-storage-and-databases/04-write-ahead-log.md) |
| WASI | WebAssembly System Interface | A standard set of imports (files, clocks, standard output) that lets WebAssembly programs run outside the browser. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| Wasm | WebAssembly | A portable binary format for programs that runs in browsers and in separate runtimes on servers. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| WAT | WebAssembly Text format | The human-readable text form of a WebAssembly module. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| wc | word count | A classic Unix program that prints line, word and byte counts. The lesson project is a small cousin of it. | [Lesson 03.8](03-ownership-and-borrowing/08-project-word-counter-and-checkpoint.md) |
| WIP | Work In Progress | Unfinished work. Also the default label of a git stash entry. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| X25519 | key exchange on Curve25519 | A fast elliptic-curve key exchange with 32-byte keys, used in the TLS handshake (often combined with ML-KEM). | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| x86-64 | (name of a CPU family) | The 64-bit CPU family of most PCs and many servers (Intel, AMD). It is strongly ordered, so many ordering bugs stay hidden there. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| XDP | eXpress Data Path | An eBPF hook in the network driver that sees each received packet before the rest of the kernel. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| XID | transaction ID | The number PostgreSQL gives every transaction that writes; a bigger number means the transaction started later. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| XML | Extensible Markup Language | A text format for data that uses tags, for example `<user>ana</user>`. | [Lesson 06.2](06-rust-advanced/02-advanced-traits-and-types.md) |
| XOR | exclusive OR | The bitwise operator ^: a result bit is 1 if the two input bits are different. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| XSS | Cross-Site Scripting | An attack where a site shows attacker-supplied script to other users, and the script runs in their browser. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| YAGNI | You Aren't Gonna Need It | Principle: do not build a feature until you really need it. | [Lesson 07.20](07-algorithms-and-data-structures/20-checkpoint.md) |
| YAML | YAML Ain't Markup Language | An indentation-based text format for configuration. CI files are written in it. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| ZST | Zero-Sized Type | A type whose values take 0 bytes, such as () or a struct without fields. It exists only for the type checker. | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| µs | microsecond | One millionth of a second; 1 ms = 1000 µs. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| µs / ms | microsecond / millisecond | A millionth and a thousandth of a second. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| σ | sigma (standard deviation) | A number saying how much measurements spread around the mean. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |

## Words and terms

### A

| Word | What it means | First used in |
|---|---|---|
| abort | Stopping a process immediately, without unwinding; Rust does this when the global allocator returns null. | [Lesson 12.14](12-computer-systems/14-project-your-own-allocator.md) |
| abort (a task) | Cancel a task from outside with handle.abort(); its future is dropped at the next .await. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| abstract (paper) | The short summary at the start of a research paper. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| abstraction | A name (a function, type or trait) that hides details so an idea can be used without repeating its steps. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |
| accept | To take the next finished connection from a listening socket's backlog, getting a new connected socket. | [Lesson 15.4](15-networking/04-tcp-sockets-in-rust.md) |
| accept / reject | Accept makes the new output the saved snapshot; reject throws it away. | [Lesson 09.5](09-testing-debugging-benchmarking/05-snapshot-testing.md) |
| accept loop | The loop that accepts connections and starts one task or thread for each. | [Lesson 15.6](15-networking/06-async-networking-with-tokio.md) |
| accept queue | The kernel queue of completed connections; accept takes connections from it. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| acceptance script | A script that checks with commands that a system does what its contract says. | [Lesson 11.15](11-linux-shell-and-containers/15-project-deploy-a-rust-service.md) |
| acceptance test | A check that the finished program matches its contract: a command and its expected output (the "Done when" lines). | [Lesson 01.10](01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) |
| acceptor / connector (TLS) | The tokio-rustls objects that turn a TcpStream into a TLS stream on the server side / on the client side. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| access control | The rules that decide who may do what; also called authorization. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| access log | One log line per answered request: ip, method and path, status, body bytes, milliseconds. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| accumulator | The running state of one aggregate for one group, such as a count or a sum. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| acknowledge | To tell the caller that its data is saved; a storage engine must do this only after the data is durable. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| acknowledged write | A write for which the client received OK; a correct replicated store never loses it. | [Lesson 18.14](18-distributed-systems/14-project-replicated-key-value-store.md) |
| acks | The producer's choice of how long to wait for a write: 0 (not at all), 1 (leader wrote it), all (every in-sync replica has it). | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| action (GitHub Actions) | A reusable step written by somebody else, such as actions/checkout. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| action item | A concrete task from a postmortem with an owner and a due date. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| active / passive health check | A check the load balancer makes on its own at an interval / a conclusion drawn from real traffic that failed. | [Lesson 15.15](15-networking/15-project-tcp-load-balancer.md) |
| active health check | A check in which the proxy itself sends requests to each backend at intervals. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| active low | A signal that means 'on' when it is low, like a button wired to ground with a pull-up. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| active recall | Trying to remember without looking. It trains memory better than re-reading. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| actor | A task that owns some state and changes it only in reaction to messages from a channel. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| actual state | What is really running right now. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| adapter | An iterator method returning a new iterator (map, filter, take). Lazy. | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| adaptive sort | A sort that runs faster when the input is already (almost) sorted. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| address | The number of a memory byte, like a house number on a street. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| address space | All the memory addresses a process can use; each process has its own. | [Lesson 13.2](13-operating-systems/02-processes-from-the-inside.md) |
| adjacency list | A way to store a graph: for each vertex, a list of its neighbours. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| adjacency matrix | A V x V table that marks which pairs of vertices are joined by an edge. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| admission control | Deciding at the door whether a new request may come in, instead of letting it wait inside. | [Lesson 20.11](20-reliability-and-hardening/11-checkpoint.md) |
| advisory | A published security notice about a crate version: what is wrong, which versions are affected, which version fixes it. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| aggregate function | A SQL function that turns many rows into one value, such as count, sum or max. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| aggregation | Turning many rows into one value per group, for example COUNT or SUM with GROUP BY. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| alert fatigue | When people get so many unimportant or false alerts that they stop reacting to them, including the real ones. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| alerting rule | A Prometheus rule with a condition; while it stays true (for the time in for), Prometheus raises an alert. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| Alertmanager | The Prometheus component that receives alerts, groups them and sends notifications to pagers, chat or email. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| algorithm | A finite list of steps that solves a problem. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| alias | A short custom name for a longer command. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| aliasing | Having two or more names (references) for the same value at the same time. | [Lesson 03.5](03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| alignment | The rule that an address must be a multiple of a power of two, for example 8; rounding up to it is called aligning. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| alloc (library) | The part of the standard library with heap types; it needs an allocator but no OS. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| allocate / free | To allocate is to ask for a piece of heap memory; to free is to give it back. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| allocation rate | How many heap blocks a program allocates and frees in total; many short-lived blocks cost CPU time. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| allocator | The system part that hands out and takes back pieces of heap memory. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| allow-list | A list of what is allowed; everything else is rejected. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| allowed failure (CI) | A CI job whose red result does not turn the whole pipeline red (continue-on-error). | [Lesson 09.14](09-testing-debugging-benchmarking/14-checkpoint.md) |
| Amdahl's law | The speed-up of a whole program is limited by the part you did not improve. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| amend | Replace the latest commit with a new, corrected one (git commit --amend). Never use on pushed commits. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| amortized | The cost averaged over the whole run: one step may be expensive, but the total is small. | [Lesson 07.15](07-algorithms-and-data-structures/15-two-pointers-and-sliding-window.md) |
| amortized cost | The cost per operation over a long series of operations; a rare expensive operation is paid for by many cheap ones. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| amplification attack | A small request with a forged source address makes a server send a large answer to the victim. | [Lesson 15.5](15-networking/05-udp-sockets-in-rust.md) |
| analyzer (SQL) | The stage that looks up every name in the parse tree (tables, columns, functions) and checks types. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| annotated tag | A Git label with a message attached to one commit, used here to mark a finished milestone. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| annotated tag / merge --no-ff | A tag with message, author and date (git tag -a), and a merge that always creates a merge commit. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| anomaly (transactions) | A result of concurrent transactions that could never happen if they ran one after another. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| anonymous lifetime '_ | A placeholder meaning "a lifetime deduced by the elision rules". | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| anonymous mapping | A mapping with no file behind it: fresh zeroed memory, used by allocators for large blocks. | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| anti-pattern | A common solution that looks fine but causes more problems than it solves. | [Lesson 08.4](08-software-design/04-composition-over-inheritance.md) |
| Apache Arrow | A standard columnar in-memory data format with libraries for many languages. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| Apache DataFusion | A query engine written in Rust on top of Arrow, embedded by many data systems. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| Apache-2.0 | The Apache License 2.0: a permissive open-source licence that also covers patents. | [Lesson 24.2](24-career-and-open-source/02-building-your-portfolio.md) |
| API contract | The exact, agreed description of an API: paths, methods, bodies and status codes that both sides rely on. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| API group | The family name of a Kubernetes object type, the part before the slash in apiVersion (for example course.example.com). | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| API server | The central Kubernetes program. Every tool and controller reads and writes cluster objects only through its HTTP API. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| API server (kube-apiserver) | The front door of a Kubernetes cluster; every tool and component reads and writes objects only through it. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| AppendEntries | The Raft leader's message that carries new log entries plus the index and term of the entry before them; with no entries it is a heartbeat. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| appetite | How much time you are willing to spend on something, decided before the design; the design must fit it. | [Lesson 23.7](23-capstone-projects/07-capstone-your-own-idea.md) |
| application state | Data shared by all requests of a service, such as a store or a connection pool; in axum it is attached with with_state and read with State. | [Lesson 16.3](16-backend-services/03-web-services-with-axum.md) |
| apply (an entry) | Give a committed command to the state machine, in log order, exactly once. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| architecture | The big structure of a program: its parts and the rules about which part may depend on which. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| architecture (instruction set) | The "language" a CPU family understands: arm64 (AArch64) on Apple M chips, x86-64 on many PCs. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| arena | A big region of memory that an allocator cuts into blocks. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| Argon2 | A deliberately slow, memory-hungry password hashing function; Argon2id is the recommended variant. | [Lesson 16.8](16-backend-services/08-authentication-in-a-web-service.md) |
| Argon2id | A deliberately slow, memory-hungry password-hashing algorithm that won an international competition in 2015. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| argument | Extra information given to a command, or the actual value passed to a function in a call. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| argument injection | Input passed as an argument starts with a dash, so the program treats it as an option. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| arithmetic shift | A right shift of a signed number that fills the new high bits with copies of the sign bit, so the sign is kept. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| arm | One line of a match: pattern =&gt; code. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| array | A fixed-length list of values of one type: [i32; 4]. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| array (bash) | A numbered list of values in one variable, for example files=(a b c). | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| artifact | A file that a build produces and you ship, such as a binary, an archive or an SBOM. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| artifact (CI) | A file that a CI job saves so that people or later jobs can download it. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| as (cast) | Converts a value to another type with as. For primitives it can silently lose information. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| assembler directive | A line starting with a dot, such as .globl; it instructs the assembler program, not the CPU. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| assembly language | Machine code written in readable short words such as add and ret. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| assert_cmd | A crate for integration tests of command-line programs: run the binary, feed input, check output and exit code. | [Lesson 13.8](13-operating-systems/08-project-your-own-shell.md) |
| assertion | A check in code that says "this must be true". It panics if false (assert!, assert_eq!). | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| asset | Anything worth protecting: data, a service that must stay online, a server, money, a reputation. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| associated constant | A const defined inside an impl block, used as Type::NAME. | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| associated function | A function in an impl block without self, called as Type::name(). | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| associated type | A type name inside a trait that each implementation chooses (type Output = …, type Item = …). | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| associativity | The direction in which operators of the same precedence are evaluated. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| assumption (Kani) | A condition given with kani::assume; Kani only considers inputs for which it is true. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| asymmetric cryptography | Cryptography with a key pair: a public key that anyone may know and a private key that stays secret. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| async fn / .await | async fn returns a future instead of running its body; .await pauses the current function until a future is ready. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| async runtime | A library that polls futures and provides timers and network I/O. Rust's standard library has none; Tokio is the most used. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| async-signal-safe | A function that may be called in a signal handler or in a child right after fork in a threaded program, such as write or _exit. | [Lesson 13.5](13-operating-systems/05-fork-exec-and-wait.md) |
| asynchronous (async) | Code that can pause while it waits and let other code run meanwhile. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| asynchronous replication | The leader confirms a write at once, before followers have it; confirmed writes can be lost on failover. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| at-least-once delivery | A delivery promise: each message arrives one or more times, so receivers must handle duplicates. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| at-most-once delivery | A delivery promise: each message arrives zero or one time, so messages can be lost. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| atomic | A number type (AtomicUsize) whose operations are one indivisible step, usable by threads without a lock. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| atomic commit | Making all participants of a transaction either commit or abort, never a mix. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| atomic operation | An action that is either fully done or not done at all; renaming a file in one folder is atomic. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| atomic replace | Writing a temporary file, syncing it and renaming it over the target, so readers see the old or the new file, never a mix. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| atomic write | Writing a file so that a crash leaves either the complete old file or the complete new one, never half of each; done with a temporary file and rename. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| AtomicPtr | An atomic raw pointer *mut T with load, store, swap and compare_exchange. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| attack surface | All the places where an attacker can reach a program: every input, port, file, argument and dependency. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| attack tree | A diagram that starts with the attacker's goal and splits it into the different ways to reach it. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| attacker | A person or a program that tries to make the system do something its owner did not intend. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| attribute | A #[name] label giving the compiler extra information about the next item. | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| attribute (telemetry) | A key-value pair attached to a span or a metric value, such as outcome=found. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| attribute macro | A procedural macro used as #[name] on an item that replaces the item with new code. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| audit (of a crate) | A person reads the code of one crate version and writes down what was checked. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| audit criteria | In cargo vet, the promise an audit makes, for example safe-to-deploy or safe-to-run. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| audit log | A record of important events kept so that attacks and mistakes can be found later. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| authentication | Proving who you are. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| authentication factor | A kind of proof: something you know, something you have, or something you are. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| authentication tag | A 16-byte check value that AEAD appends to the ciphertext. Decryption fails if the data or the tag changed. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| authoritative server | A DNS server that holds the real records of a zone; its answers carry the aa flag. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| authorization | Deciding what you may do once we know who you are. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| authorized_keys | The file on the server with the public keys that may log in as that user. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| auto trait | A trait the compiler implements automatically for a type when all its fields have it, like Send, Sync and Unpin. | [Lesson 14.12](14-concurrency-and-async/12-pin-and-self-referential-futures.md) |
| auto-referencing | The compiler adds &, &mut or \* to the receiver so value.method() fits the method's self form. | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| auto-vectorization | The compiler turns a normal loop into SIMD code by itself, without changes in the source code. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| auxiliary space | The extra memory an algorithm needs on top of its input. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| availability | The system works and answers when authorised users need it. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| available_parallelism | A standard function that says how many threads the machine can really run at the same time, usually the number of cores. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| avalanche | The property of a good hash function that changing one letter of the key changes the whole hash. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| avalanche effect | A tiny change in the input changes about half of the bits of a hash. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| average case | The cost for a typical input, averaged over many inputs. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| Aya | A Rust library for eBPF with crates for the kernel side and the user side. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |

### B

| Word | What it means | First used in |
|---|---|---|
| B+ tree | A B-tree variant where all values are in linked leaves and internal nodes hold only keys and child page numbers. | [Lesson 17.6](17-storage-and-databases/06-b-trees.md) |
| B-tree | A balanced tree whose nodes hold many values; the standard BTreeMap and BTreeSet are built on it. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| baby step | A very small change in TDD: one test, then the least code that makes it pass. | [Lesson 09.6](09-testing-debugging-benchmarking/06-tdd-in-practice.md) |
| back end | The part of a language implementation that runs or translates the checked program, such as an interpreter or a code generator. | [Lesson 22.25](22-specializations/25-compilers-track-project-and-next-steps.md) |
| back-of-the-envelope estimate | A quick, rough calculation of load, storage or time that is good to within about a factor of ten. | [Lesson 23.2](23-capstone-projects/02-writing-a-design-document.md) |
| backend | The part of a software system that runs on servers: it keeps the data, applies the rules and answers requests. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| backend (OpenTofu state) | The shared place where a team keeps the OpenTofu state, for example an S3 bucket or a PostgreSQL database. A good backend also locks the state. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| backend / upstream | One of the real servers behind a proxy. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| backing service | A service that a program uses over the network: a database, a message queue, an email server. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| backlog | The length of the queue of connections that the kernel has completed but the program has not accepted yet. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| backlog (listen queue) | The kernel queue of finished handshakes that the program has not accepted yet. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| backoff | A waiting time between retries of a failed operation that usually grows with each attempt. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| balanced tree | A tree whose height is close to log₂ n; a self-balancing tree repairs its shape after each change. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| bandwidth | How many bytes per second can flow when many accesses are in progress at once. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| bare metal | Running directly on the hardware, with no operating system below your code. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| base (Kustomize) | The shared set of complete manifests that every environment uses. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| base (radix) | How many different digits a number system uses: 10 for decimal, 2 for binary, 16 for hexadecimal, 8 for octal. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| base case | The input small enough to answer directly, which stops the recursion. | [Lesson 07.7](07-algorithms-and-data-structures/07-recursion.md) |
| base image | The image named in FROM, on top of which a new image is built. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| base64 | Writing bytes with 64 printable characters. The base64url variant uses - and _ so it is safe in URLs. It is an encoding, not encryption. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| basic block | A straight sequence of statements with no jumps inside, ending with one jump; MIR and LLVM IR are made of them. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| batching (group commit) | Writing many proposals to a log with a single fsync instead of one fsync each. | [Lesson 23.3](23-capstone-projects/03-capstone-distributed-key-value-store.md) |
| beacon | A device that repeats the same message again and again. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| bearer token | A token that gives access to whoever holds it. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| behavioural interview | An interview about how you worked in the past: conflicts, mistakes, decisions. | [Lesson 24.7](24-career-and-open-source/07-coding-and-system-design-interviews.md) |
| benchmark methodology | The written description of how a measurement was made: hardware, versions, workload, warm-up, repetitions, commands. | [Lesson 23.8](23-capstone-projects/08-testing-benchmarking-and-hardening-capstones.md) |
| best case / worst case | The input of a given size that makes the algorithm finish the fastest / the slowest. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| big-endian (network byte order) | Byte order where the most significant byte comes first. Network protocols and many file formats use it. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| Big-O notation | A short way to write complexity that keeps only the fastest-growing part, for example O(n²). | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| binary format | A way to store data as raw bytes with a fixed meaning, not as readable text. | [Lesson 17.3](17-storage-and-databases/03-binary-file-formats.md) |
| binary heap | A heap in which every node has at most two children and the tree is stored in a vector. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| binary search | Finding an item in a sorted slice by comparing with the middle item and discarding the half that cannot contain it; O(log n). | [Lesson 07.10](07-algorithms-and-data-structures/10-binary-search.md) |
| binary tree | A tree in which every node has at most two children, a left and a right one. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| bind | To attach a socket to a local address and port. | [Lesson 15.4](15-networking/04-tcp-sockets-in-rust.md) |
| bind address | The address a server listens on; it decides which networks can reach the program. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| bind mount | A folder of the host made visible inside the container. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| @ binding | name @ pattern: test a value against a pattern and keep it under a name. | [Lesson 06.3](06-rust-advanced/03-patterns-in-depth.md) |
| backpressure | A way to say "slow down" when the system is full, e.g. a bounded channel makes send wait or a server answer 503. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| backtrace | The list of functions that were running when a panic happened (RUST_BACKTRACE=1). | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| backup | A second copy of your files in another place. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| bare repository | A repository without working files, only history. Used by servers. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| base (zdiff3) | The text in the common ancestor before either side changed it; shown with merge.conflictStyle zdiff3. | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| baseline | Saved results to compare against later (criterion --save-baseline). | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| benchmark | A program or test measuring speed of code. A micro-benchmark measures one small function. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| binary | Counting with two digits, 0 and 1. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| bindgen / cbindgen / cc | bindgen writes Rust declarations from C headers; cbindgen writes a C header from Rust; cc compiles C code from build.rs. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| binding | The link between a name and a value; let creates a binding. Also giving a name to part of a matched value (n in Some(n)). | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| binding (query plan) | Replacing column names in a plan with column positions once, before execution. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| binding mode | How a pattern variable gets its value: by move/copy or as a reference. Matching a reference makes bindings references. | [Lesson 06.3](06-rust-advanced/03-patterns-in-depth.md) |
| binding power | The number a Pratt parser gives an operator; higher numbers bind tighter. | [Lesson 22.22](22-specializations/22-compilers-track-lexing-and-parsing.md) |
| bipartite graph | A graph whose vertices can be split into two groups so that every edge joins different groups. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| bisect | Finding the commit that introduced a problem by binary search over the history (git bisect, git bisect run). | [Lesson 01.4](01-git-and-version-control/04-viewing-history-and-changes.md) |
| bit | The smallest piece of information: 0 or 1. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| bit field | A group of neighbouring bits inside an integer that stores one small value; read it with shift then mask. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| bit set | A set of small numbers stored as the bits of an integer: bit n is 1 when n is in the set. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| Bitcask | A storage design (from the Riak database) with an append-only data file and an in-memory hash index of all keys. | [Lesson 17.5](17-storage-and-databases/05-hash-indexes.md) |
| bitwise copy | Copying the bytes of a value exactly as they are, with no extra work. | [Lesson 03.3](03-ownership-and-borrowing/03-move-copy-and-clone.md) |
| bitwise operator | Operators acting on individual bits: & \| ^ ! &lt;&lt; &gt;&gt;. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| blame | Showing for each line the last commit and author that changed it (git blame). | [Lesson 01.4](01-git-and-version-control/04-viewing-history-and-changes.md) |
| blameless postmortem | A written analysis after an incident that looks for causes in systems and processes, never for a guilty person, and ends with action items. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| blanket implementation | An implementation for every type that meets a bound, e.g. impl&lt;T: Display&gt; Shout for T. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| blast radius | How much damage one leaked secret can do; smaller is better. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| bloat | A table or index that is much bigger than its live data because of dead versions and empty space. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| block | Code between { and }. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| block (heap) | One piece of heap memory handed out by one allocation. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| block device | A device that stores data in fixed-size blocks, like a disk. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| block entry | A page table entry that maps a whole large region directly, without a next-level table. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| block expression | A { } block whose last expression without ; is its value. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| blocking | Keeping a thread busy or asleep without letting the runtime run other tasks (e.g. std::thread::sleep in async code). | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| blocking (algorithm) | If one thread is stopped in the middle, for example holding a lock, other threads can be stopped forever too. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| blocking call | A call that puts the process to sleep until it can finish, like read on an empty pipe. | [Lesson 13.6](13-operating-systems/06-pipes-and-redirection-inside.md) |
| blocking pool | Extra threads that Tokio keeps for spawn_blocking, so blocking work does not stop the worker threads. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| blocklist | A list of addresses whose packets must be dropped. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| Bloom filter | A small bit array that answers 'definitely not in the set' or 'maybe in the set' and never says no for an added key. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| blue-green deployment | Running the new version next to the old one and switching all traffic at once. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| body (HTTP) | The bytes of a request or response after the headers. In hyper a body is a stream of frames. | [Lesson 15.9](15-networking/09-http-with-hyper-and-reqwest.md) |
| boolean blindness | A bool argument whose meaning you cannot see at the call site, such as render(true, false). | [Lesson 08.10](08-software-design/10-anti-patterns-and-code-smells.md) |
| borrow / borrowing | Creating a reference. While it exists the owner still owns the value. | [Lesson 03.4](03-ownership-and-borrowing/04-references-and-borrowing.md) |
| borrow checker | The part of the compiler that enforces the borrowing and lifetime rules. | [Lesson 03.5](03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| bottleneck | The part of an algorithm that costs most of the time; make it faster first. | [Lesson 07.18](07-algorithms-and-data-structures/18-problem-solving-practice.md) |
| bounce | Restart a crashed simulated host: its program starts from the beginning with empty memory. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| bounce (button) | The fast random on-off flicker of a mechanical contact during the first milliseconds of a press. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| boundary | The place where data enters the program from outside; the best place to parse it. | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| boundary value | A value exactly on the edge between two cases, for example weight 500 when the limit is up to 500. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| bounded (proof) | Checked only up to a limit, such as a number of loop rounds or an array length; it says nothing about bigger inputs. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| bounded queue | A channel with fixed capacity; try_send fails at once when full so the server can answer 503. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| bounds check | The test index < len before indexing; if it fails, safe Rust panics instead of reading wrong memory. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| Box&lt;dyn Error&gt; | A catch-all error type: a box that can hold any error. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| `Box<T>` | Smart pointer storing a T on the heap with a single owner. Needed for recursive types and cheap moves. | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| bpf-linker | The linker that turns Rust code compiled for bpfel-unknown-none into an eBPF object file. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| bpftrace | A one-liner tracing language that compiles to eBPF. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| branch | A named, movable pointer to a commit; a line of work. In Rust, also one path of an if. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| branch misprediction | A wrong guess about a branch; the CPU discards the work on the wrong path, which costs many cycles. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| branch prediction | The CPU guesses which way a branch will go and keeps working on that guess before the condition is known. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| breaking change | A change to the public API that can break users' code; requires a new MAJOR version. | [Lesson 05.7](05-rust-intermediate/07-documentation.md) |
| breakpoint | A mark on a line where the debugger pauses the program. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| broadcast address | The last address of an IPv4 subnet, with all host bits one; a packet to it reaches every computer in the subnet. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| broadcast channel | A channel where every receiver gets every message; a too slow receiver gets a Lagged error. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| broker | A server that receives messages from producers, stores them and hands them to consumers. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| brute force | The simplest correct solution: try every possibility; slow but easy to get right, so it is a good judge for faster solutions. | [Lesson 07.18](07-algorithms-and-data-structures/18-problem-solving-practice.md) |
| BTreeMap | A map that keeps entries sorted by key; keys need Ord. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| bucket | One slot of the map's internal table where entries with a given hash are kept. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| buffer | A piece of memory holding data on its way in or out. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| buffer (planning) | Time in a plan that is reserved for surprises, not for new features. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| buffer overflow | Reading or writing past the end of an array or buffer. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| buffer pool | A database's own cache of disk pages in its own memory. The database decides which pages stay and when changed pages are written. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| buffer reuse | Keeping one String or Vec and clearing it for each item instead of creating a new one every time. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| buffered I/O | Input and output through a buffer, for example with BufWriter and BufReader. | [Lesson 19.9](19-performance-and-profiling/09-async-and-io-performance.md) |
| buffering | Collecting many small writes in memory and sending them with one big write. | [Lesson 13.3](13-operating-systems/03-system-calls.md) |
| BufWriter | A buffered writer that makes fewer system calls (16x faster output in minigrep). Remember flush(). | [Lesson 05.13](05-rust-intermediate/13-project-minigrep.md) |
| bug | A mistake that makes a program do something other than intended. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| bug report | An issue that describes a problem so precisely that someone else can see it happen: what I did, what I expected, what happened, environment. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| bug-compatible | Keeping a known bug exactly as it is during a refactoring, so that the structure change and the bug fix are two separate steps. | [Lesson 08.16](08-software-design/16-project-refactor-a-messy-program.md) |
| build | Compiling the code into an executable file. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| build cache | Docker's memory of each build step; an unchanged step is reused. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| build context | The folder whose files docker build sends to the builder. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| build matrix | A CI feature that runs the same job several times with different values, for example once per sanitizer. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| build profile (dev / release) | A named set of compiler settings in Cargo, such as opt-level and overflow-checks; cargo build uses dev, --release uses release. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| build script | Code that runs on your computer while you compile. Procedural macros do the same, so dependencies can run code at build time. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| build script (build.rs) | A Rust program Cargo runs before compiling the crate; it compiles C code (cc), generates bindings (bindgen) and prints cargo:: instructions. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| builder | A helper type that collects settings one by one and creates the final value in a last step. | [Lesson 08.6](08-software-design/06-builder-pattern.md) |
| builder pattern | Small with_\* methods that take and return self so calls can be chained. | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| built-in (command) | A command the shell runs inside its own process, such as cd, because it changes the shell itself. | [Lesson 13.8](13-operating-systems/08-project-your-own-shell.md) |
| bullet point | One line in a list, starting with a dash or a dot; a resume consists mostly of them. | [Lesson 24.6](24-career-and-open-source/06-resume-and-job-search.md) |
| bump allocator | An allocator that hands out blocks by moving one pointer forward and can only free everything at once. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| burn rate | How fast the error budget is used: the current error ratio divided by the budget. Burn rate 1 uses the budget exactly by the end of the window. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| burnout | Exhaustion from too much work or pressure for too long; common among maintainers. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| busy polling | Calling poll again and again in a loop without sleeping; it works but burns a whole CPU core. | [Lesson 14.10](14-concurrency-and-async/10-how-futures-work.md) |
| busy waiting | Checking again and again in a loop whether something is ready, wasting CPU time. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| byte | 8 bits; holds a value from 0 to 255. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| byte order (endianness) | The order in which the bytes of a multi-byte number are stored in memory, in a file or in a network packet. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| byte stream | Data seen as one long row of bytes with no message borders; TCP delivers a byte stream. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| bytecode | Instructions for a virtual machine, not for a real CPU. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| BytesMut / Bytes | Byte buffers from the bytes crate: BytesMut grows and splits cheaply, Bytes is a read-only, cheaply shared piece. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| Byzantine failure | A failure in which a node sends wrong or even malicious messages. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| Byzantine fault | A node behaves arbitrarily or maliciously and may lie. Raft does not handle these. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |

### C

| Word | What it means | First used in |
|---|---|---|
| C ABI | The ABI of C: the common language between almost all programming languages. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| cache | A small, very fast memory inside the CPU that keeps recently used data close to the chip. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| cache (CI) | Files saved between CI runs to make the next run faster; it must never be required for correctness. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| cache coherence | The hardware rule that all cores see one consistent value for each address, although each core has its own cache. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| cache hit / cache miss | A hit means the data is already in the cache; a miss means it must be fetched from a slower level. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| cache line | The fixed-size, aligned block (64 or 128 bytes) that a cache copies as one unit. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| cache poisoning | An attack in which a resolver accepts and caches a fake DNS answer, so all its clients are sent to the wrong address. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| cache stampede | Many requests miss the same expired key at once and all hit the database together. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| cache-aside | A caching pattern: read the cache first; on a miss read the database and store the result in the cache with a TTL. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| cache-friendly code | Code that touches memory in an order caches handle well: in order, close together, with a small working set. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| call frame | The variables of one running function call; every call gets a fresh frame that disappears when it returns. | [Lesson 22.25](22-specializations/25-compilers-track-project-and-next-steps.md) |
| call stack | Memory holding the data of currently running functions; one stack frame per call. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| callback | A function or closure given to other code so it calls you back at the right moment. | [Lesson 05.4](05-rust-intermediate/04-closures.md) |
| calling convention | The rule for where function arguments and results are placed (which registers, the stack). | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| canary release | Sending a small share of traffic to a new version first and more only if it behaves well. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| cancel safety | A future is cancel-safe if dropping it at any .await loses nothing: no data it already took, no half-done change. | [Lesson 14.14](14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| cancellation | Stopping a future before it finishes. In Rust you cancel by dropping it (timeout and select! do this). | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| CancellationToken | A shareable flag from tokio-util: one place calls cancel(), every holder can await cancelled(). | [Lesson 14.14](14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| canonical form | The one standard way to write a value, for example 1h30m and never 90m for the same duration. | [Lesson 09.13](09-testing-debugging-benchmarking/13-project-tested-library-with-ci.md) |
| capability (Linux) | One separate piece of root's power, such as CAP_SYS_ADMIN, that can be given or dropped on its own. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| capacity | How many bytes or items a growable value such as String or Vec can hold before it must reallocate. Always at least the length. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| capstone project | A large final project that combines many skills from a whole course; Phase 23 asks for two. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| capture | A closure using an outside variable by borrowing it, mutably borrowing it, or taking ownership. | [Lesson 05.4](05-rust-intermediate/04-closures.md) |
| cardinality | The number of different values an attribute can have; high-cardinality metric attributes create too many time series. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| Cargo | Rust's build tool and package manager. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| Cargo alias | A short custom command defined in .cargo/config.toml, such as cargo host-test. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| Cargo example | A small extra program in the examples/ folder, run with cargo run --example name. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| cargo expand | A Cargo subcommand (needs nightly installed) that prints the crate after all macros are expanded. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| Cargo runner | A program that cargo run starts instead of your binary, with the binary's path as argument. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| Cargo.lock | A file written by Cargo recording the exact versions used in the last build; commit it for programs. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| Cargo.toml | The settings file of a project: name, version, edition, dependencies. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| carry | The bit that falls out on the left of an addition. For unsigned numbers a carry means the result did not fit. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| CAS loop | Read the value, compute a new one, try to store it with compare-and-swap, and retry if another thread changed it. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| case | One generated set of inputs and one run of the test body with it; proptest runs 256 cases per test by default. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| cast | Converting a value to another type with as. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| catch_unwind | A std::panic function that runs a closure and returns Err(payload) instead of letting a panic leave the closure. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| caught mutant | A mutant that makes a test fail, which is good: your tests noticed the change. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| causal consistency | Causally related operations are seen in their happens-before order by everybody; concurrent ones may be seen in different orders. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| causality | The relation of cause and effect between events; in distributed systems, the same as happens-before. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| causality violation | loom's name for two accesses to an UnsafeCell that are not ordered by any synchronisation: a data race. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| cdylib | A crate type for libraries loaded by other languages or hosts; for WebAssembly it produces a module with exports. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| `Cell<T>` | A box that copies a value out (get) and puts a new one in (set) through &. Never gives a reference to the inside. | [Lesson 06.4](06-rust-advanced/04-interior-mutability.md) |
| centralized VCS | A VCS where one server holds the only real history (CVS, Subversion). | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| certificate | A signed document that says this public key belongs to this name (format X.509). | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| certificate chain | The server's leaf certificate followed by intermediate certificates, sent during the TLS handshake. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| cfg flag | A name switched on at compile time (for example --cfg loom); code marked #[cfg(name)] is compiled only when it is on. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| cfg(miri) | A condition that is true only when code runs under Miri; used to skip or shrink tests that are too slow for it. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| cgroup | A kernel feature that limits and counts the resources of a group of processes. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| cgroup v2 | The current cgroup version: one tree for all resources, mounted at /sys/fs/cgroup. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| cgroup.procs | The file that lists the processes of a cgroup; writing a PID into it moves the process into the group. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| chain of trust | The sequence leaf, intermediate, root, in which each certificate is signed by the next one. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| chaining | Collision strategy where each bucket holds a small list of all entries that landed in it. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| changelog | CHANGELOG.md: a file listing what changed in each version, for users. | [Lesson 01.10](01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) |
| channel | A pipe between threads (sender tx, receiver rx). send moves the value; the receiver loop ends when all senders are dropped. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| chaos testing (chaos engineering) | Running experiments that inject faults into a whole running system to learn whether it stays healthy. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |
| char boundary | A byte position where one character ends and the next begins. Slicing is allowed only there. | [Lesson 03.6](03-ownership-and-borrowing/06-slices-and-string-types.md) |
| character encoding | A rule that turns characters into bytes and back, for example ASCII or UTF-8. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| characterization test (golden test) | A test that records what the code does now, including odd results, so that you notice when a change alters it. | [Lesson 08.13](08-software-design/13-refactoring-and-code-review.md) |
| chart (Helm) | A Helm package: a folder with Chart.yaml, values.yaml and templates of manifests. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| check-then-act | A race condition: a thread checks a condition and acts on it later, but the condition changed in between. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| checked arithmetic | Integer methods such as checked_add and checked_mul that return None on overflow instead of wrapping or panicking. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| checkpoint | A self-test lesson at the end of a phase with questions and practical tasks. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| checkpoint (database) | The moment when a database writes all dirty pages to its data files, so that crash recovery can start from there. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| checkpoint (file position) | A saved file position that lets an agent continue reading after a restart. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| checkpoint / self-test | A stop to check you know the material; a test you give yourself. | [Lesson 02.11](02-programming-basics/11-checkpoint.md) |
| checksum | A small number computed from data and sent with it so that the receiver can detect damaged bytes. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| cherry-pick | Copy one commit from another branch onto the current branch as a new commit. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| child and parent | A child is a node directly below another node; the node above it is its parent. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| child object | An object an operator creates for a custom resource and owns through an owner reference, such as a ConfigMap or Deployment. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| child token | A token made with child_token(): cancelling the parent cancels it, cancelling it does not cancel the parent. | [Lesson 14.14](14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| chroot | A system call that changes the root directory of one process; the old file system stays mounted and can be reached again by a root process. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| chunked transfer coding | Sending an HTTP body in pieces, each with its hexadecimal size in front, ending with a piece of size 0. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| CI badge | A small picture in a README that shows whether the last automatic build and test run passed. | [Lesson 24.2](24-career-and-open-source/02-building-your-portfolio.md) |
| cipher | An encryption algorithm, for example ChaCha20 or AES. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| ciphertext | The unreadable result of encryption. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| circuit breaker | A guard on the calling side that stops calling a failing service for a while and fails fast instead. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| claim | One statement inside a token: sub (the user), exp (expiry time), role. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| cleanup (operator) | The work an operator does for a terminating object before it removes its finalizer, such as deleting outside resources. | [Lesson 22.28](22-specializations/28-cloud-native-track-testing-and-running-operators.md) |
| client (turmoil) | A simulated machine that runs the test code; the simulation ends when all clients finish. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| client / server (SSH) | The computer you type on is the client (ssh); the computer you log in to is the server (sshd). | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| client stub | The generated client type whose methods send the RPC requests for you. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| Clippy | Rust's linter: gives advice about mistakes and bad style (cargo clippy). | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| clock algorithm | A cheap approximation of LRU: each frame has a "used" bit, and a hand goes round the frames giving used pages a second chance. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| clock cycle | One tick of the CPU clock, a fraction of a nanosecond; the basic unit of time inside a CPU. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| clock drift | A clock running slightly too fast or too slow, measured in ppm. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| clock skew | The difference between the clocks of two machines at the same moment. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| clone | Make a full copy of a remote repository, including its whole history. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| close-on-exec (FD_CLOEXEC) | A flag that makes the kernel close a descriptor automatically when the process starts a new program. | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| CLOSE-WAIT | The state of a socket whose peer has closed while the local program has not closed yet; many of them point to a bug. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| close_notify | The TLS alert that says 'I am done sending'; without it a client cannot tell a clean end from a cut connection. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| closed book | Answering from memory only, without looking anything up. | [Lesson 07.20](07-algorithms-and-data-structures/20-checkpoint.md) |
| closed model | A load generator with a fixed number of users, each waiting for its answer before the next request. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| closure | A small unnamed function \|x\| x + 1 that can use variables from where it was created. | [Lesson 04.4](04-structs-enums-collections-errors/04-option-and-match.md) |
| cloud native | A way of building and running services so that automation, not people, starts, copies, replaces and updates them. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| cluster | A group of nodes that work together as one system. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| ClusterIP | The default Service type: a virtual IP address reachable only inside the cluster. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| ClusterRole / ClusterRoleBinding | A set of RBAC rules valid in all namespaces / the object that gives such a role to a subject; Role and RoleBinding are the one-namespace versions. | [Lesson 22.28](22-specializations/28-cloud-native-track-testing-and-running-operators.md) |
| CMD / ENTRYPOINT | The command a container runs when it starts. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| code block | Lines shown in a fixed-width font, fenced by three backticks in Markdown. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| code coverage | A measurement of which parts of your code were executed while the tests ran. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| code generation | A tool writes source code for you from a description, for example Rust types from a .proto file. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| code generator | The part of a compiler that turns the syntax tree into instructions for the target machine. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| code map | Your own short notes that list the main parts of a codebase, what each does and where it lives. | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| code of conduct | The rules for how people treat each other in a project's spaces, and how problems are reported. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| code point | One number from the Unicode table, written like U+044F. | [Lesson 04.8](04-structs-enums-collections-errors/08-strings-in-depth.md) |
| code review | Other people read your changes and comment before they are merged. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| code smell | A surface sign in the code that may point to a deeper design problem. The code works, but something feels wrong. | [Lesson 08.10](08-software-design/10-anti-patterns-and-code-smells.md) |
| codebase | All the source code of one project, together with its tests, scripts and documents. | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| codec (decoder / encoder) | Code that turns bytes into messages (decoder) and messages into bytes (encoder). | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| codegen unit | A piece of a crate that LLVM optimizes separately and in parallel. More units build faster; fewer units let LLVM optimize more code together. | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| coding interview | An interview where you solve a programming problem while the interviewer watches and asks questions. | [Lesson 24.7](24-career-and-open-source/07-coding-and-system-design-interviews.md) |
| Coffman conditions | The four conditions a deadlock needs at once: mutual exclusion, hold and wait, no preemption, circular wait. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| cohesion | How closely the things inside one part (function, struct, module) belong together; high cohesion is the goal. | [Lesson 08.2](08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) |
| collect | Consumer that gathers items into a Vec, String, HashMap or `Result<Vec<_>,_>`; the target type decides. | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| collection | A value that holds many other values; its data lives on the heap so it can grow and shrink. | [Lesson 04.7](04-structs-enums-collections-errors/07-vectors.md) |
| collision | Two different keys that land in the same bucket. The map handles it. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| columnar format | Storing each column's values next to each other instead of each row's values. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| command | A word telling the shell which program to run. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| command injection | Injection into a text that a shell will run. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| command language | The small set of words a user types to control a program. | [Lesson 04.11](04-structs-enums-collections-errors/11-project-phone-book.md) |
| command substitution | $(command): the shell runs the command and puts its output into the line. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| comment | Text for humans starting with // that the compiler ignores. /// and //! comments become documentation. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| commit | One saved version in a repository's history, with an author, a date and a message. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| commit / rollback | Ending a transaction by keeping all its changes (commit) or throwing them all away (rollback). | [Lesson 16.7](16-backend-services/07-migrations-and-transactions.md) |
| commit index | The highest log index known to be committed. A node applies entries up to it. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| commit message | A short text explaining why a commit was made: an imperative subject and an optional body explaining why. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| commit point | The exact moment a transaction becomes permanent: before it a crash undoes the transaction, after it a crash cannot. | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| commit record | The log entry whose durable presence means "this transaction committed". | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| commit timestamp | A number given to every commit in order; a snapshot can be just the last commit timestamp when a transaction began. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| committed (entry) | A log entry stored on a majority. It will never be lost or replaced, so nodes may apply it. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| committed offset | The offset a consumer group saved as processed; reading continues from it after a restart. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| common ancestor (merge base) | The latest commit that two branches share. | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| community | The people around a project: users, contributors, maintainers, and the places where they talk. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| compaction | Rewriting a log or data files so that only the latest live data remains and old versions and deleted keys are dropped. | [Lesson 17.4](17-storage-and-databases/04-write-ahead-log.md) |
| comparison sort | A sorting algorithm that learns the order only by comparing two items at a time. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| compensation | An action that semantically undoes an earlier saga step, such as a refund for a charge. It must be idempotent and retried until it succeeds. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| compile error | A mistake the compiler finds before the program runs. No executable is produced. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| compile time | When cargo build runs the compiler, before the program runs. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| compile-time checked query | A query written with a sqlx macro (query!, query_as!) that the compiler checks against the real database: syntax, columns and types. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| compile_error! | Built-in macro that stops compilation with a custom message. Used in a catch-all macro arm. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| compiler | A program that translates source code into machine code and checks it for mistakes (rustc for Rust). | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| compiler front end / back end | The front end understands the language (parsing, types, borrow checking); the back end produces machine code for one CPU. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| compiler pass | One walk over the syntax tree that does one job, such as type checking or code generation. | [Lesson 22.25](22-specializations/25-compilers-track-project-and-next-steps.md) |
| complete binary tree | A binary tree where every level is full except maybe the last, which is filled from the left. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| completion model | The kernel performs an operation and later reports that it is finished, with the result. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| component | A small type that does one job and is used as a field of bigger types. | [Lesson 08.4](08-software-design/04-composition-over-inheritance.md) |
| Compose project | One running instance of a Compose file; its name prefixes all objects it creates. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| compose.yaml | The Compose description file with services, volumes, networks and secrets. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| composition | Building a bigger type by putting smaller types inside it as fields. | [Lesson 08.4](08-software-design/04-composition-over-inheritance.md) |
| composition root | The one place, usually main, that chooses the real implementations and connects them. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| compound assignment | Operators like += that change a variable using its own value. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| compound type | A type grouping several values: tuple, array. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| compute kernel | An Arrow function that works on whole arrays at once, such as compare, filter or sum. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| concatenation | Joining two pieces of text into one longer piece (+, format!). | [Lesson 04.8](04-structs-enums-collections-errors/08-strings-in-depth.md) |
| concern | One separate topic a program must handle, such as reading input, applying rules or showing results. | [Lesson 08.2](08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) |
| concrete playback | A Kani option that prints a counterexample as a unit test. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| concrete type | A real type such as i32 or String, as opposed to a blank like T. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| concurrency | Several jobs in progress at the same time, even if only one runs at a given moment. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| concurrency limit | The highest number of operations allowed to run at the same time, for example with buffer_unordered(n) or a Semaphore. | [Lesson 19.9](19-performance-and-profiling/09-async-and-io-performance.md) |
| concurrent events | Two events where neither happened before the other, so no information could flow between them. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| condition | A yes/no (bool) question inside a program that decides what happens next. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| condition code | A suffix such as eq, gt or hi that tells a branch or select instruction which flag combination to test. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| condition variable (Condvar) | A place where threads sleep until another thread says the state changed. It always works together with a mutex. | [Lesson 14.3](14-concurrency-and-async/03-condition-variables.md) |
| conditional breakpoint | A breakpoint that stops only when a given expression is true. | [Lesson 09.9](09-testing-debugging-benchmarking/09-debugging-with-lldb-and-rustrover.md) |
| conditional select (csel / cmov) | An instruction that picks one of two values depending on the flags, without a branch. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| confidence | How sure a passing test makes you that the program really works for a user. | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| confidentiality | Only the people who are allowed can read the data. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| config level | Where a Git setting is stored: all users (system), you (~/.gitconfig, global), or one repository (.git/config, local). The most specific wins. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| ConfigMap | A Kubernetes object that holds non-secret configuration as key-value pairs, read by pods as environment variables or files. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| configuration | Settings that change between environments (laptop, test, production) but not between runs, such as addresses, ports and passwords. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| configuration (config) | A set of settings that change how a program behaves. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| conflict markers | The lines &lt;&lt;&lt;&lt;&lt;&lt;&lt;, =======, &gt;&gt;&gt;&gt;&gt;&gt;&gt; (and \|\|\|\|\|\|\|) that Git writes into a conflicted file. | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| congestion control | Keeping the sender from sending more than the network can carry; it backs off when packets are lost. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| connected component | A maximal group of vertices that can reach each other. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| connected socket | A socket for one connection through which data flows; in Rust a TcpStream. | [Lesson 15.4](15-networking/04-tcp-sockets-in-rust.md) |
| connection (TCP) | An agreement between two programs to exchange a byte stream, with state kept on both sides. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| connection ID | A number in every QUIC packet that identifies the connection, so it survives a change of address or port. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| connection limit | The maximum number of client connections served at once; extra clients are refused instead of overloading the system. | [Lesson 15.15](15-networking/15-project-tcp-load-balancer.md) |
| connection manager (redis) | A Redis connection from the redis crate that reconnects by itself and can be shared by cloning. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| connection migration | Keeping a QUIC connection alive when the client's network address changes, for example from Wi-Fi to mobile data. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| connection pool | A set of open connections that a client keeps and reuses, so it does not open a new connection for every request. | [Lesson 15.9](15-networking/09-http-with-hyper-and-reqwest.md) |
| connection string | One URL with everything needed to connect to a database: postgres://user:password@host:port/database. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| consensus | Several machines agree on one value or one sequence of values and never change their minds, even when some crash or messages are lost. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| consistency check | A follower's test before appending: does my log have an entry at prev_log_index with term prev_log_term? If not, it refuses. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| consistency model | A contract between a data store and its clients about which values reads may return. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| consistent hashing | Assigning keys to nodes on a ring of hash values so that adding or removing a node moves only about 1/N of the keys. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| const assertion | An assert! inside a const item, checked while compiling, so a wrong size or alignment stops the build. | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| const generic | A generic parameter that is a value (a number), such as the length N of [i32; N]. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| constant (const) | A fixed value known at compile time, with a required type, named in SCREAMING_SNAKE_CASE. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| constant folding | Computing at compile time what is already known at compile time, such as 24 * 60 * 60. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| constant-time | Code whose running time does not depend on secret values, so timing reveals nothing. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| constant-time comparison | A comparison that takes the same time whatever the data is. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| constraint | A limit on the input given with a problem, for example at most 100,000 numbers; it tells which complexity is fast enough. | [Lesson 07.18](07-algorithms-and-data-structures/18-problem-solving-practice.md) |
| constraint (database) | A rule the database checks on every change, such as NOT NULL, CHECK, UNIQUE or REFERENCES; a change that breaks it fails. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| constructor | A function that builds a new instance. In Rust a convention (usually Type::new). | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| consumer | An iterator method that uses up the iterator and gives a final result (sum, count, collect, fold). | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| consumer (JetStream) | A reader of a stream that remembers which messages it has already processed; a durable one keeps its position across restarts. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| consumer group | A set of consumers that share a topic's partitions; each partition is read by exactly one member. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| consumer lag | How far a consumer group is behind: log end offset minus committed offset. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| consuming builder | A builder whose methods take self by value and return it. | [Lesson 08.6](08-software-design/06-builder-pattern.md) |
| container | An isolated Linux process with its own view of files, processes and network, sharing the host kernel. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| container runtime | The program that creates a container: it sets up namespaces, the root file system and cgroups, then starts the program inside (for example runc). | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| containerd | A container runtime: the program that actually runs containers on a node. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| contention | Several threads want the same lock at the same time. A contended lock is slow; an uncontended one is cheap. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| context (eBPF) | The value the kernel passes to an eBPF program, giving access to the event and the current process. | [Lesson 22.7](22-specializations/07-ebpf-track-a-first-program-with-aya.md) |
| context (kubectl) | A named pair of a cluster and a user in a kubeconfig; the current context decides where kubectl sends commands. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| Context (std::task) | The second argument of poll; it carries the Waker of the task that is polling. | [Lesson 14.10](14-concurrency-and-async/10-how-futures-work.md) |
| context propagation | Passing the trace id and current span id to the next service (for example in the traceparent header) so its spans join the same trace. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| context switch | Saving the registers of one task and loading another's, so the CPU continues the other one. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| contiguous | Placed one right after another in memory, with no gaps. | [Lesson 07.3](07-algorithms-and-data-structures/03-arrays-and-vectors-under-the-hood.md) |
| continuation byte | A byte of a multi-byte UTF-8 character after the first one; it always has the form 10xxxxxx. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| continuous delivery | Every change that passes the pipeline becomes a release that can go to production at any time; a person decides when. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| continuous deployment | Every change that passes the pipeline goes to production automatically. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| continuous fuzzing | Running fuzzers automatically and regularly, for example on every pull request and every night, while keeping the corpus between runs. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| continuous integration vocabulary | A workflow file contains jobs; a job is a group of steps run on a runner (a machine, often a VM). | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| contract | The exact description of what a program must do: input, output and rules. | [Lesson 02.10](02-programming-basics/10-project-guessing-game.md) |
| contract (of a trait) | What a trait promises to its users beyond its types: for example never panics, keeps the order. | [Lesson 08.3](08-software-design/03-solid-principles.md) |
| contract test | A test function written against a trait, so that every implementation of the trait must pass the same test. | [Lesson 23.3](23-capstone-projects/03-capstone-distributed-key-value-store.md) |
| CONTRIBUTING file | CONTRIBUTING.md: a project's rules for contributors: building, testing, formatting, commits and how to send changes. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| contribution | Any useful work given to a project: code, tests, documentation, a clear bug report, answers to other users. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| control character | An ASCII code that is an instruction rather than a visible symbol, such as newline (0x0a) or tab (0x09). | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| control flow | The order in which lines of a program run; changed by conditions and loops. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| control loop | Running reconcile again and again, every time something may have changed, like a thermostat. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| control plane | The programs of a Kubernetes cluster that store the desired state and make decisions: API server, etcd, scheduler and controller manager. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| controller | A program that runs a reconciliation loop for one kind of object, for example Deployments. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| controller (cgroup) | The kernel part that manages one resource for cgroups, such as memory, cpu, io or pids. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| controller (Kubernetes) | A program that watches some kinds of objects and keeps changing the cluster until it matches what those objects ask for. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| controlling terminal | The terminal of a session; it sends Ctrl-C and Ctrl-Z as signals to its foreground process group. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| convention | An agreed way of doing something that the language does not enforce but everyone follows, so that code looks familiar. | [Lesson 08.11](08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) |
| Conventional Commits | A commit message convention: type(scope): description, with feat, fix, docs, refactor, ci and so on. ! marks a breaking change. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| cookie | A small piece of text that the server asks the browser to store and to send back with later requests to the same site. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| cooperative scheduling | Tasks give control back by themselves at .await points; the runtime cannot stop a task that never awaits. | [Lesson 14.11](14-concurrency-and-async/11-build-a-tiny-async-runtime.md) |
| coordinated omission | A measuring mistake: a tester that waits for slow answers sends fewer requests and so hides most of the slow ones. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| coordinator / participant | In 2PC, the process that collects votes and decides, and the services or databases that vote and obey the decision. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| Copy | A trait marking a type as cheap to duplicate bytewise. Such values are copied instead of moved. | [Lesson 03.3](03-ownership-and-borrowing/03-move-copy-and-clone.md) |
| core (library) | The part of the Rust standard library that needs nothing from an operating system. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| core.hooksPath | A Git setting pointing to a folder of hooks that can be committed to the repository. | [Lesson 01.10](01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) |
| corpus | The folder of interesting inputs that a fuzzer has collected; it is the fuzzer's memory. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| corpus minimization | Removing corpus files that add no new coverage, done with cargo fuzz cmin. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| cost (planner) | The planner's estimate of the work for a plan node, in arbitrary units where reading one page in order costs about 1. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| counter (metric) | A metric that only goes up, such as the number of requests served. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| counterexample | One concrete input on which a rule gives a wrong answer; one is enough to show that the rule is wrong. | [Lesson 07.17](07-algorithms-and-data-structures/17-greedy-algorithms.md) |
| coupling | How much one part depends on the details of another; low (loose) coupling is the goal. | [Lesson 08.2](08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) |
| course | The whole set of lessons, from zero to a systems engineer. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| cover letter | A short letter sent with the resume that says why you apply for this job. | [Lesson 24.6](24-career-and-open-source/06-resume-and-job-search.md) |
| coverage-guided | A fuzzer is coverage-guided when it keeps the inputs that reach new code and mutates them further. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| CPU affinity | The set of CPU cores a thread is allowed to run on (taskset, sched_setaffinity). | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| crash artifact | The file that stores the exact input that made a fuzz target fail. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| crash fault | A node simply stops working; it does not send wrong messages. Raft handles crash faults. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| crash-only design | A style where a program never tries to repair broken state in memory: it crashes and a supervisor restarts it from durable data. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| crash-recovery | A failure in which a node stops and later restarts with only the data it had saved durably to disk. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| crash-stop | A failure in which a node stops and never comes back. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| CrashLoopBackOff | A pod status: the container keeps crashing and the kubelet waits longer and longer before each restart. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| crate | The unit the compiler builds: a program (binary crate) or a library. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| crate root | Source file where compilation of a crate starts: src/main.rs or src/lib.rs. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| crate type (proc-macro = true) | What a crate builds into; a proc-macro library can export only macros and is built before code that uses it. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| crates.io | The public registry where Rust crates are published. | [Lesson 02.10](02-programming-basics/10-project-guessing-game.md) |
| credential | The proof a user gives: a password, a key, a code. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| credential stuffing | Trying passwords stolen from one website on many other websites, because many people reuse passwords. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| credentials | What a user proves their identity with, for example an email and a password. | [Lesson 16.8](16-backend-services/08-authentication-in-a-web-service.md) |
| critical section | The code that runs while a lock is held. Keep it short, but make it cover the whole invariant. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| cron | A short notation for repeating times, for example 17 3 * * 1 means 03:17 every Monday. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| cross-compilation | Building a program on one kind of computer for another kind, for example on a Mac for a microcontroller. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| cross-compiling | Building on one platform a program for another platform. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| crossover point | The input size at which one algorithm stops being faster and another takes over. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| crypto provider | The library that performs the cryptography for rustls, either ring or aws-lc-rs; a program should use exactly one. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| cryptography | The science of keeping information secret, unchanged and attributable, using mathematics. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| CString and CStr | CString is an owned NUL-terminated string to pass to C; CStr is a borrowed view of a C string to read in Rust. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| ctid | The physical address of a PostgreSQL row version: page number and slot number in the page. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| ctrlc crate | Small crate: ctrlc::set_handler runs a closure on Ctrl-C; the closure should only set an AtomicBool flag. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| current-thread runtime | A Tokio runtime that runs every task on the thread that called block_on; the default for #[tokio::test]. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| cursor | A variable that points at a place in a data structure and moves along it step by step. | [Lesson 07.4](07-algorithms-and-data-structures/04-linked-lists.md) |
| custom resource | One object of a type that a CRD added to the Kubernetes API. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| cut list | The ordered list of features to drop first if a project runs late, written at the start. | [Lesson 23.7](23-capstone-projects/07-capstone-your-own-idea.md) |
| cycle | A path that starts and ends at the same vertex. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| CycloneDX | One of the two common standard formats for SBOMs; SPDX is the other. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |

### D

| Word | What it means | First used in |
|---|---|---|
| daemon | A process that runs in the background for a long time, usually a service. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| DaemonSet | A Kubernetes object that runs exactly one copy of a pod on every node, also on nodes added later. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| dangling pointer | A pointer to memory that has already been freed (use after free) or never existed. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| dangling reference | A reference to a value that no longer exists. Rust makes it impossible in safe code. | [Lesson 03.4](03-ownership-and-borrowing/04-references-and-borrowing.md) |
| dashboard / panel | A Grafana page / one graph or number on that page, driven by one query. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| data flow | Data moving between the parts of a system: a network request, a file read, an argument. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| data layout | How values are arranged in memory: sizes, order, and whether they are adjacent or behind pointers. | [Lesson 19.6](19-performance-and-profiling/06-data-layout-for-speed.md) |
| data parallelism | Doing the same operation on many pieces of data at the same time on several cores. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| data plane | The part of a network system that moves the actual bytes, such as proxies and load balancers. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| data race | Two parts of a program use the same data at once, at least one writes, and nothing coordinates them. Impossible in safe Rust. | [Lesson 03.5](03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| data race detector (Miri) | The part of Miri that reports two accesses to the same memory without happens-before, at least one a write, as undefined behaviour. | [Lesson 14.7](14-concurrency-and-async/07-building-a-spin-lock.md) |
| data source (Grafana) | A connection in Grafana to a database it reads from, for example Prometheus. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| data store | A place where data rests: a file, a database. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| data structure | A way of organising data in memory so that the operations on it are cheap, such as a vector, a hash map or a tree. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| database driver | The code that speaks a database's network protocol. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| datagram | One UDP message, delivered whole or not at all. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| day number | In Lesson 24.8: the number of whole days since 1970-01-01, easy to compare and subtract. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| dbg! | Macro that prints an expression, its value, file and line to stderr and returns the value. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| dead tuple | A row version that no current or future snapshot can see; it only takes space until VACUUM removes it. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| dead-code elimination / black_box | The optimizer removes code whose result is unused; std::hint::black_box hides values from it so benchmarks stay honest. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| deadline | The moment (an Instant) when a timer is due. | [Lesson 14.11](14-concurrency-and-async/11-build-a-tiny-async-runtime.md) |
| deadlock | Threads waiting for each other's locks forever. Avoid by locking in the same order and holding locks briefly. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| .deb package | The file format of a Debian and Ubuntu package. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| debouncing | Ignoring short changes of a button signal until it has been stable for some time. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| Debug | Trait behind {:?}: developer-friendly printing. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| debug assertion | debug_assert!: a check that exists only in debug builds. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| debug info | Data in a binary that maps machine code to function names, files and line numbers. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| debug information | Extra data in a compiled program that maps machine code back to file names, lines and variable names. | [Lesson 09.9](09-testing-debugging-benchmarking/09-debugging-with-lldb-and-rustrover.md) |
| debug probe | A small adapter that connects a computer to a board to flash and debug firmware. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| debugger | A tool that runs a program under control: pause, inspect variables, step. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| debugging | Finding and fixing bugs. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| declarative | Describing what: the end result, while a program finds the steps. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| declarative macro | A macro written with macro_rules! as patterns of input and code to produce. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| deduplication table | The part of the state machine that remembers, per client, the last applied sequence number and its result. | [Lesson 23.3](23-capstone-projects/03-capstone-distributed-key-value-store.md) |
| deep copy | A duplicate that also duplicates everything the value owns on the heap. | [Lesson 03.3](03-ownership-and-borrowing/03-move-copy-and-clone.md) |
| Default | Trait with default() that returns a sensible empty value; #[derive(Default)] and ..Default::default(). | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| default method | A trait method with a body. Implementing types get it for free and may override it. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| default network | The private network Compose creates; services reach each other by service name. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| default route | The route 0.0.0.0/0 that matches every address and is used when nothing more specific matches. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| default type parameter | A generic parameter with a default value, e.g. Add&lt;Rhs = Self&gt;. Rhs is the type of the right-hand operand. | [Lesson 06.2](06-rust-advanced/02-advanced-traits-and-types.md) |
| defense in depth | Use several independent protections, so that when one fails the next one still stops the attack. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| definition of done | A short list of conditions that must all be true before a piece of work counts as finished. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| defmt | A logging crate for microcontrollers that sends compact binary messages and formats them on the computer. | [Lesson 22.12](22-specializations/12-embedded-track-async-with-embassy.md) |
| delayed ACK | The receiver waits a short time before acknowledging, hoping to send the acknowledgement together with a reply. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| delegation | A method of the outer type that does its work by calling the same job on an inner value. | [Lesson 08.4](08-software-design/04-composition-over-inheritance.md) |
| deletionTimestamp / terminating | The field set when an object with finalizers is deleted; from then on the object is terminating until its finalizers are removed. | [Lesson 22.28](22-specializations/28-cloud-native-track-testing-and-running-operators.md) |
| delimiter | A character or text that separates parts in a longer text. | [Lesson 04.8](04-structs-enums-collections-errors/08-strings-in-depth.md) |
| delta audit | An audit of the difference between two versions of a crate when the older one was already audited. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| demand paging | The kernel gives RAM to a page only when the program touches it for the first time. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| demo | A short live or recorded showing of a system doing its job, usually about two minutes. | [Lesson 23.9](23-capstone-projects/09-presenting-your-work.md) |
| denial of service | An attack or bug that makes a service unusable, for example by using up all memory or CPU. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| dense keys | Small integer keys without big gaps, which can be used directly as Vec indexes. | [Lesson 19.6](19-performance-and-profiling/06-data-layout-for-speed.md) |
| deny by default | If no rule allows an action, it is refused. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| deny-list | A list of what is not allowed; everything else passes. Weak, because attackers find what you forgot. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| dependency | A crate (code by someone else) that your project uses, listed in Cargo.toml. | [Lesson 02.10](02-programming-basics/10-project-guessing-game.md) |
| dependency chain | A series of instructions where each needs the result of the previous one, so they cannot run in parallel. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| dependency cycle | A situation where package A needs B and B needs A (directly or through others); Cargo refuses to build it. | [Lesson 08.12](08-software-design/12-project-layout-and-workspaces.md) |
| dependency injection | Giving a type its dependencies from outside, through new or a parameter, instead of letting it create them. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| dependency lock file (.terraform.lock.hcl) | The file in which tofu init records the exact provider versions and checksums. It is committed to Git, like Cargo.lock. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| dependency rule | Source-code dependencies point towards the domain; the domain never mentions adapters or technology. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| dependency tree | All crates a project needs, including the dependencies of its dependencies. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| depends_on | A Compose key that says a service needs another first; with service_healthy it waits for the health check. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| deploy | One running copy of the program in one place: a laptop, a test server or production. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| Deployment | A Kubernetes object that manages ReplicaSets so pods can be updated step by step and rolled back. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| deque | A queue where items can be added and removed at both ends. | [Lesson 07.5](07-algorithms-and-data-structures/05-stacks-and-queues.md) |
| Deref | Trait that defines what \*p does for a pointer-like type; deref coercion automatically turns `&Box<String>` into &str. | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| dereference | Following a reference or pointer to the value it points to, written \*r. Deref coercion lets &String be used where &str is expected. | [Lesson 03.4](03-ownership-and-borrowing/04-references-and-borrowing.md) |
| derive | #[derive(Debug)] asks the compiler to write standard trait code for a type. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| derive macro | #[derive(...)] makes the compiler write a trait implementation. An attribute #[...] gives extra information to the compiler or a macro. | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| descriptor leak | Forgetting to close descriptors until the process reaches its limit (EMFILE). | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| design (of software) | The decisions about how code is split into parts, what the parts are called and how they work together. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |
| design decision record | A short note that says what was decided, why, and what was given up. | [Lesson 16.15](16-backend-services/15-project-url-shortener-service.md) |
| design document | A short text, written before the code, that describes the problem, the chosen design, the alternatives and the risks so that others can review it. | [Lesson 23.2](23-capstone-projects/02-writing-a-design-document.md) |
| design principle | A rule of thumb that helps choose between two ways of writing code; advice from experience, not a law. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |
| desired state | The written description of what the system should look like. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| desired state / actual state | What an object asks for (its spec) / what really exists in the cluster right now. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| destructive command | A command that can permanently delete work not saved in a commit. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| destructuring | Taking a tuple, struct, enum or array apart into pieces with a pattern and naming the pieces. | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| detach | Let a task run without keeping its handle; dropping a JoinHandle detaches the task and does not stop it. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| detached HEAD | HEAD points directly to a commit, not to a branch. New commits are easy to lose. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| deterministic | Always gives the same result for the same input. A test must be deterministic. | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| dev-dependency | A dependency used only by tests, examples and benchmarks, not by the final program. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| dev/prod parity | Keeping the development, test and production environments as similar as possible. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| device register | One small control or data cell of a device at a fixed address. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| dhat | A heap profiler for Rust published as a crate; it replaces the global allocator with a counting one. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| diagnosis | Finding the cause of a failure from its symptoms, for example from error messages. | [Lesson 15.16](15-networking/16-checkpoint.md) |
| diagnostic | The compiler's full message about a problem: code, snippet, labels and suggestions. | [Lesson 03.7](03-ownership-and-borrowing/07-fixing-common-ownership-errors.md) |
| dictionary (fuzzing) | A file of byte strings, such as keywords or magic numbers, that the fuzzer may insert into inputs. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| diff | The lines added, removed or changed between two versions (git diff prints it). | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| differential fuzzing | Fuzzing two implementations with the same input and failing when their answers differ. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| differential testing | Testing a function by comparing its results with a trusted version, such as the standard library, on many inputs. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| dig | A command-line tool that sends DNS queries and prints the answers in detail. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| digest | The exact, unchanging fingerprint sha256:… of one image. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| digital signature | A value made with a private key that anyone can check with the matching public key. It proves who signed and that the data did not change. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| Dijkstra's algorithm | An algorithm that finds shortest paths from one start vertex to all others when no weight is negative; O((V + E) log V) with a binary heap. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| directed graph | A graph whose edges have a direction, a -> b. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| directory (folder) | A container for files and other directories. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| directory entry | One line of a directory: a name and the inode number it points to. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| dirty page | A page changed in memory and not yet written back to the file. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| dirty read | Reading a change of another transaction that has not committed and may still roll back. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| discriminant | The number identifying an enum variant. Settable (Low = 1), read with as. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| discriminant (tag) | The hidden number inside an enum value that says which variant it is. | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| Display | Trait behind {}: user-facing printing, written by hand. Also gives .to_string(). | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| Display / Debug | The human-readable ({}) and programmer-oriented ({:?}) ways to show a value. Display is written by hand; Debug can be derived. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| disposition (signal) | What a process does when a signal arrives: the default action, ignore it, or run a handler. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| disruptive server | A node that was cut off and raised its term in lonely elections; when it returns it forces a needless new election. Pre-Vote prevents this. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| distributed system | Several programs on several computers that work together on one job and communicate only by sending messages over a network. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| distributed transaction | A transaction whose changes live on several machines but must still be all-or-nothing. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| distributed VCS | A VCS where every developer has a full copy of the history (Git). | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| distribution | A complete Linux system: the kernel plus a package manager, tools and settings, such as Ubuntu or Debian. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| distroless image | A minimal image without a shell or package manager, only the files a program needs to run. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| diverge | Never continue normally: return, break, continue, panic! (type !). | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| divide and conquer | Splitting a problem into parts, usually two halves, solving each recursively and combining the answers. | [Lesson 07.7](07-algorithms-and-data-structures/07-recursion.md) |
| doc comment | A /// (next item) or //! (enclosing item) comment that becomes generated documentation. | [Lesson 05.7](05-rust-intermediate/07-documentation.md) |
| doc test | An example in a doc comment that cargo test compiles and runs. | [Lesson 05.6](05-rust-intermediate/06-testing-in-depth.md) |
| Docker Compose | A tool that starts several containers together from one description file. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| Docker Desktop | The app for Mac and Windows that runs the Docker Engine inside a hidden Linux VM. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| Docker Engine | The background program that builds images and runs containers; the docker command is its client. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| docker export | A Docker command that writes all files of a container into one tar archive. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| Dockerfile | A text file with the steps to build an image. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| .dockerignore | A file that lists names that must not be sent in the build context. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| docs.rs | The website that builds and hosts documentation of every crate on crates.io. | [Lesson 05.7](05-rust-intermediate/07-documentation.md) |
| documentation (docs) | The official description of a language, library or tool; text explaining what code is for and how to use it. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| domain | The part of the program that holds the business rules and concepts; it knows nothing about screens, files or networks. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| domain name | A name made of labels separated by dots, read from right to left as a path in the DNS tree. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| domain type | A type named after an idea of the problem being solved, like Email or Percent. | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| dotfile | A file or folder whose name starts with a dot; hidden by default. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| double caching | The same page is kept in RAM twice: in the database's buffer pool and in the kernel's page cache. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| double free | Freeing the same memory twice, which corrupts the allocator's bookkeeping. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| double hashing | Building k hash positions from two hashes as h1 + i·h2, instead of computing k separate hashes. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| downstream | The client side of a proxy: whoever sends it requests. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| Downward API | A Kubernetes feature that gives a container facts about its own pod, such as its name and namespace, as environment variables or files. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| drain timeout | The longest time a server waits for open connections during shutdown before cutting them. | [Lesson 15.15](15-networking/15-project-tcp-load-balancer.md) |
| draining | Reading from a non-blocking source until it reports WouldBlock, required with edge-triggered epoll. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| drift | A difference between the infrastructure description and reality, usually caused by a change made by hand. tofu plan shows it. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| drift detection | Checking automatically that what runs in a system still matches the desired state stored in Git. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| driven adapter | An adapter that the core calls through a port: storage, email, the clock. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| driver | Kernel code that knows how to talk to one kind of device. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| driver (embedded) | Code that controls one device, written against embedded-hal traits so it works on any chip. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| driver (sans-IO) | The code around a sans-IO core that ticks it, delivers messages, sends its outbox, saves its state and applies its results. | [Lesson 18.14](18-distributed-systems/14-project-replicated-key-value-store.md) |
| driving adapter | An adapter that calls the core: a command-line interface, a web handler or a test. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| drop | To free a value and everything it owns. Rust drops a value when its owner goes out of scope. Also the Drop trait. | [Lesson 03.2](03-ownership-and-borrowing/02-ownership-rules.md) |
| Drop (trait) | Trait whose drop method runs when a value goes out of scope. Values drop in reverse order of creation. Call drop(x), never x.drop(). | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| dry run | A mode in which a program says what it would do and changes nothing. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| dummy | A test double that only fills a parameter and is never really used. | [Lesson 09.3](09-testing-debugging-benchmarking/03-test-doubles.md) |
| dup | A system call that makes a second descriptor for the same open file description. | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| dup2 | A system call that makes descriptor number B point where descriptor A points; used for redirection. | [Lesson 13.6](13-operating-systems/06-pipes-and-redirection-inside.md) |
| duplication | The same idea written in more than one place. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |
| durability | The promise that saved data survives a crash or power loss. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| durable | Saved so that it survives a crash or a power loss. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| DWARF | The standard debug info format on Linux; it also tells how to unwind the stack without frame pointers. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| dyn | Keyword that marks a trait object type, e.g. dyn Shape. | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| dyn compatible (object safe) | A trait that may be used as dyn Trait: no methods returning Self, no generic methods, methods have a receiver. | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| dynamic dispatch | The program chooses which function a call runs at run time, through a vtable (dyn Trait). | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| dynamic programming | Solving a problem by solving its smaller versions once, saving the answers and combining them. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |

### E

| Word | What it means | First used in |
|---|---|---|
| EAGAIN | "Try again": the error a non-blocking call returns instead of waiting. | [Lesson 13.6](13-operating-systems/06-pipes-and-redirection-inside.md) |
| eBPF program | A small function compiled to eBPF bytecode that the kernel runs at a hook. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| echo server | A server that sends back every byte it receives. | [Lesson 15.4](15-networking/04-tcp-sockets-in-rust.md) |
| Ed25519 | A modern, fast signature algorithm with 32-byte keys and 64-byte signatures. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| edge | A link between two vertices of a graph. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| edge (signal) | The moment a digital signal changes: rising (low to high) or falling (high to low). | [Lesson 22.12](22-specializations/12-embedded-track-async-with-embassy.md) |
| edge case | An unusual but valid input at the border of what is allowed: zero, empty, the maximum. | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| edge list | A way to store a graph as a list of its edges, one per line. | [Lesson 07.19](07-algorithms-and-data-structures/19-project-route-finder.md) |
| edge-triggered | epoll reports a descriptor only when new data arrives, so the program must read until WouldBlock. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| edition | A named version of the language rules (2015, 2018, 2021, 2024). The course uses 2024. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| edition 2024 FFI syntax | unsafe extern block declares foreign functions (safe fn marks safe-to-call ones); #[unsafe(no_mangle)] keeps a Rust function's plain name for C. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| editor | A program for writing and changing text files. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| EINTR | "Interrupted system call": the error a blocking call returns when a signal handler ran while it waited. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| election timeout | How long a Raft follower waits without hearing from a leader before it starts an election. It is chosen at random from a range. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| elevation of privilege | Getting more rights than you were given. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| Embassy | A framework of crates for async embedded Rust: executor, timers, synchronization and HALs. | [Lesson 22.12](22-specializations/12-embedded-track-async-with-embassy.md) |
| embedded repository | A Git repository inside the working tree of another one; the outer repository stores only a pointer to it. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| embedded system | A small computer built into a device that runs one program, usually without an operating system. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| embedded-hal | A crate of traits (output pin, input pin, delay, buses) that HALs implement and drivers use. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| embedded-hal-async | The async version of the embedded-hal traits, with async fn methods such as delay_ms. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| emulator | A program that pretends to be another computer. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| enable (service) | Make a service start automatically at boot, by a symbolic link; start only starts it now. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| encapsulation | Hiding a type's inside (private fields) and exposing a small public API so outside code cannot break its rules. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| encode / decode | Turn a value into bytes / turn bytes back into a value. | [Lesson 17.3](17-storage-and-databases/03-binary-file-formats.md) |
| encoding | A rule for turning code points into bytes and back. | [Lesson 04.8](04-structs-enums-collections-errors/08-strings-in-depth.md) |
| encryption | Turning plaintext into ciphertext with a key. Decryption turns it back with the right key. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| end-to-end acknowledgement | A message from a sink to a source that a batch was delivered, so that the source may advance its checkpoint. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| end-to-end test | A test that starts the real finished program, gives it input like a user would, and checks the output and the exit code. | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| endpoint | One kind of request a service answers, written as an HTTP method plus a path, for example GET /health. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| endpoint (quinn) | One UDP socket in quinn that can act as client, server or both, with all its QUIC connections. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| EndpointSlice | A Kubernetes object listing the current addresses of the ready pods behind a Service. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| entropy | A measure of unpredictability counted in bits. A key with 128 bits of entropy has 2^128 equally likely values. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| entry | One key with its value; also the type returned by map.entry(key), a handle to the place for a key. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| entry point | The address of the first instruction the program runs, usually _start of the C runtime, not main. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| enum (type) | A type whose value is exactly one of a fixed list of named variants. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| .env file | A file with NAME=value lines that tools and programs load as environment variables during development; it must not be committed when it holds secrets. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| environment | The interpreter's record of which variables exist and what values they hold, scope by scope. | [Lesson 22.23](22-specializations/23-compilers-track-an-interpreter.md) |
| environment (bug report) | The facts about where code ran: versions of Rust and the library, operating system, processor type. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| environment (deployment) | One place where a service runs with its own settings, such as dev, staging or production. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| environment file | A file of NAME=value lines that systemd reads to set a service's environment. | [Lesson 11.15](11-linux-shell-and-containers/15-project-deploy-a-rust-service.md) |
| environment of a closure | The variables around a closure that it may capture. | [Lesson 05.4](05-rust-intermediate/04-closures.md) |
| environment variable | A named setting readable by programs started from the shell (HOME, PATH). | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| ephemeral port | A temporary port the operating system picks for the client side of a connection. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| EPIPE / SIGPIPE | The error and the signal you get when writing to a pipe that has no readers left. | [Lesson 13.6](13-operating-systems/06-pipes-and-redirection-inside.md) |
| epoch-based reclamation | Memory is freed only after every thread has left the epoch in which it could have seen it; crossbeam-epoch implements it. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| epoll | The Linux interface to register many descriptors and wait until some of them are ready. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| equivalent mutant | A mutant that changes the code but not its behaviour, so no test can catch it. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| erase block | A large group of flash pages that an SSD can only erase all together. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| error | Something that went wrong while the program runs (missing file, bad input, network down). Different from a bug. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| error budget | The amount of failure an SLO allows, 1 minus the SLO. A team spends it on risk and agrees in advance what happens when it is used up. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| error budget policy | A short agreed document that says what the team does when the error budget is spent, for example stop feature releases until it recovers. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| error chain | An error with a context message plus the errors that caused it ({e:#}). | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| error code | A name like E0425 identifying a kind of compile error, explained by rustc --explain. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| error policy | The function a kube-rs controller calls when reconcile fails; it decides when to try again. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| Error trait | std::error::Error, the standard trait that marks a type as an error. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| error type | One enum that lists every way an operation can fail, such as ApiError; with IntoResponse it becomes one consistent error response. | [Lesson 16.4](16-backend-services/04-errors-validation-and-middleware.md) |
| escape sequence | A backslash plus a letter for a special character: \n, \t. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| escaping | Writing a value so that the place that receives it treats it as plain data, not as a command. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| etcd | A replicated key-value database that stores all objects of a Kubernetes cluster. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| Ethernet | The most common link-layer technology for network cables. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| Ethernet header | The first 14 bytes of a frame: destination and source MAC address and the EtherType. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| EtherType | A 2-byte number that says what follows the Ethernet header, for example 0x0800 for IPv4. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| evaluate | Compute the value of an expression. | [Lesson 22.23](22-specializations/23-compilers-track-an-interpreter.md) |
| evaluation (expression) | Computing the value of an expression for one row, for example age >= 30 gives true. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| event | Something that happens and may change the state. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| event (Kubernetes) | A short record that a component writes about an object, such as Pulling image or Failed; kubectl describe shows them. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| event (perf) | Something the kernel or the CPU can count, such as CPU time, context switches or cycles. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| event (pipeline) | One unit of data in a pipeline: a log line with its fields, or one metric value. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| event (tracing) | One thing that happened at one moment; info!(...) creates an event. | [Lesson 09.10](09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| event loop | The main loop of a program that waits for events and handles them one after another. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| eventual consistency | If no new writes happen, all replicas eventually return the same value; nothing is promised in the meantime. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| eviction | Removing an old line from a full cache to make room for a new one. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| evidence | A test, a command with its output, a dashboard or a document that proves a checklist item is done. | [Lesson 20.9](20-reliability-and-hardening/09-production-readiness-checklist.md) |
| example (Cargo) | A small program in the examples folder of a package, run with cargo run --example name. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| example-based test | An ordinary test with one hand-chosen input and one expected output. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| exception | An event that makes the CPU stop the current code and jump to a handler. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| exception (CPU) | An event caused by the running instruction, such as a bad memory address, handled by the kernel. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| exchange argument | A proof technique for greedy rules: swap one choice of a best answer for the greedy choice and show that the answer does not get worse. | [Lesson 07.17](07-algorithms-and-data-structures/17-greedy-algorithms.md) |
| exec (execve) | The system call that replaces the program running in a process; the PID stays the same. | [Lesson 13.5](13-operating-systems/05-fork-exec-and-wait.md) |
| exec form / shell form | CMD ["/app"] runs the program directly; CMD /app runs it through /bin/sh -c. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| executable | A file with a ready-to-run program in machine code (Mach-O on macOS). | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| execute | Run a statement for its effect, such as printing or changing a variable. | [Lesson 22.23](22-specializations/23-compilers-track-an-interpreter.md) |
| executor | The part of a runtime that decides which future to poll next. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| executor (query engine) | The part of a query engine that runs a plan and produces the result rows. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| executor (SQL) | The part of a database that runs the plan and produces the rows. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| exemption | In cargo vet, a crate version accepted without an audit for now; the exemptions are the audit backlog. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| exhaustive | Covering every possible value. A match must be exhaustive (error E0004). | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| exit code | A number a program returns when it ends. 0 is success, anything else is failure. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| exit status | The number a program returns to the shell when it ends; 0 means success, any other number means failure. | [Lesson 07.19](07-algorithms-and-data-structures/19-project-route-finder.md) |
| expect | Result method: gives the value on Ok, panics with your message on Err. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| expected error | A problem that is a normal part of the job, such as bad input or a timeout; it is returned as Err and handled. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| experiment branch | A throw-away Git branch for risky tries, deleted with git branch -D. | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| expiry | The moment after which a token or session stops working. | [Lesson 16.8](16-backend-services/08-authentication-in-a-web-service.md) |
| exploit | A way of using a vulnerability, or the program or input that does it. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| exponent bias | A constant added to the real exponent so it can be stored without a sign: 127 for f32, 1023 for f64. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| exponential | Growing like 2ⁿ: each extra step of n doubles the work, so only small n are usable. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| exponential backoff | Waiting longer before each new retry (100 ms, 200 ms, 400 ms, …) so that a struggling service can recover. | [Lesson 14.15](14-concurrency-and-async/15-project-concurrent-job-queue.md) |
| exporter | The component that sends telemetry somewhere: to a collector, a terminal or a file. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| expression | Code that produces a value. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| extension | Adding new behaviour to a program, for example a new kind of shape. | [Lesson 08.3](08-software-design/03-solid-principles.md) |
| external entity | Something outside your system that talks to it: a user, another service. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| extractor (axum) | A type in a handler's parameter list that axum fills from the request, such as Path, Query, Json or State. | [Lesson 16.3](16-backend-services/03-web-services-with-axum.md) |

### F

| Word | What it means | First used in |
|---|---|---|
| F_FULLFSYNC | A macOS fcntl command that makes the drive write its own cache to permanent storage; Rust's sync_all and sync_data use it on Apple systems. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| fail fast | Stop at start-up with a clear message when something is wrong, instead of failing later in a request. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| fail open | When a protective component (like a rate limiter) fails, requests are allowed instead of blocked. | [Lesson 16.15](16-backend-services/15-project-url-shortener-service.md) |
| fail open / fail closed | What a check does when it cannot decide: fail open allows the action, fail closed refuses it. Security checks must fail closed. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| failover | Sending a client to another backend when the chosen one fails. | [Lesson 15.15](15-networking/15-project-tcp-load-balancer.md) |
| failure | The moment when users see wrong behaviour or get no answer. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| failure detector | A component that watches heartbeats and suspects nodes that have been silent for too long; it can be wrong. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| failure mode | One typical way of failing, such as a crash, a hang, a slow answer, a wrong answer or data loss. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| failure model | A description of the ways in which parts of a system are allowed to fail; an algorithm is designed for one failure model. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| fairness | Whether a lock is given to waiting threads in the order they came. Rust's standard locks do not promise it. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| fake | A test double with a simple but working implementation, such as an in-memory store. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| fake it | Making a test pass in the simplest, even silly, way such as returning a constant; the next test forces the real code. | [Lesson 09.6](09-testing-debugging-benchmarking/06-tdd-in-practice.md) |
| fallacies of distributed computing | Eight false beliefs about networks, such as "the network is reliable" and "latency is zero", that cause real bugs. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| false positive | A 'maybe' from a Bloom filter for a key that was never added; it costs a useless read but never a wrong answer. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| false sharing | Threads write different variables that lie in the same cache line, so the line keeps moving between their cores. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| fan-out | Starting many independent operations at the same time and then collecting their results. | [Lesson 19.9](19-performance-and-profiling/09-async-and-io-performance.md) |
| fanout | How many children a tree node has; B-tree nodes can have hundreds. | [Lesson 17.6](17-storage-and-databases/06-b-trees.md) |
| fast lane / slow lane | CI jobs that run on every push in minutes, and longer jobs that run after them or at night. | [Lesson 23.8](23-capstone-projects/08-testing-benchmarking-and-hardening-capstones.md) |
| fast path / slow path | The quick case of an operation (a free lock is taken with one atomic operation) and the expensive case (a busy lock: the OS puts the thread to sleep). | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| fast-forward | A merge that only moves the branch pointer forward. No new commit is created. | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| fat LTO / thin LTO | Fat LTO optimizes all code as one unit (slow build); thin LTO works in parallel with a summary of the whole program (faster build, often similar results). | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| fat pointer | A pointer twice as big as normal: an address plus a vtable address (trait objects) or a length (slices). | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| fault | A cause of trouble: a bug, a full disk, a lost packet, a slow dependency. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| fault injection | Making failures happen on purpose in a test, such as a crash after N bytes, to check how the code reacts. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| fault tolerance | The ability to keep working correctly while some parts have failed. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| fd table | The per-process table in the kernel; a file descriptor is an index into it. | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| fdatasync | A system call like fsync that may skip metadata not needed to read the data back; Rust's File::sync_data on Linux. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| feature (Cargo) | An optional part of a crate switched on in Cargo.toml. Features are additive. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| feature branch | A short-lived branch for one task, merged back into main when done. | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| feature envy | A method that mostly uses the fields of another type, so the logic sits far from its data. | [Lesson 08.10](08-software-design/10-anti-patterns-and-code-smells.md) |
| feature flag | A configuration switch that turns a feature on or off without a new deployment. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| feedback time | How long you wait to learn that something broke. Short feedback time makes a developer faster. | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| fence | std::sync::atomic::fence: an ordering rule without a memory access of its own; fence(SeqCst) is a full barrier. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| fencing token | A number that grows with every new lease; the storage rejects writes carrying an older number than one it has already seen. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| fetch | Download new commits from a remote without changing your branches or files. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| fetch, decode, execute | The three steps the CPU repeats for every instruction. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| field | One named part of a struct. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| field (tracing) | A named value attached to an event or a span, such as total=2500; fields make logs searchable. | [Lesson 09.10](09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| field / delimiter | A field is one column of a line; a delimiter is the character that separates columns. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| field init shorthand | Writing name instead of name: name when a variable has the same name as the field. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| field manager | The name a writer signs its changes with in server-side apply, so the API server knows who owns which field. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| field number (tag) | The number after each field in a .proto message; it identifies the field in the encoded bytes, so it must never change. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| file descriptor limit | The maximum number of open files and sockets per process, shown by ulimit -n. | [Lesson 15.6](15-networking/06-async-networking-with-tokio.md) |
| file header | The first bytes of a file, describing its format (magic bytes, version, parameters, salt). | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| file mode | The numbers that say who may read, write or run a file; 0o600 means only the owner can read and write. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| file permissions 0600 | A Unix file mode in which only the owner may read and write the file. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| file system | A way to organise files on a storage device: data blocks, names, owners and permissions; ext4 is the usual one on Linux. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| filter | A program that reads text from stdin, changes or selects some of it and writes the result to stdout. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| finalizer | A string in an object's metadata.finalizers list; while the list is not empty, the API server only marks the object for deletion and waits. | [Lesson 22.28](22-specializations/28-cloud-native-track-testing-and-running-operators.md) |
| fingerprint | A short hash of a public key used to identify it. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| firmware | The program that runs on a microcontroller, stored in its flash memory. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| first committer wins | A snapshot-isolation rule: if two concurrent transactions write the same key, the first to commit wins and the other is aborted. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| first fit / best fit | Search strategies of a free-list allocator: take the first block that is big enough, or the smallest one that is big enough. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| fixed window (rate limit) | A rate-limit method that counts requests in time windows of fixed length and resets the count when a window ends. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| fixture | The prepared data a test needs before it runs: files, folders, input text. | [Lesson 09.2](09-testing-debugging-benchmarking/02-testing-command-line-programs.md) |
| flag (bit flag) | One bit that stores a yes/no answer for one option, such as a read or write permission. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| flag (CPU) | A one-bit note the CPU sets after an instruction, such as carry or overflow. On arm64 the flags are N, Z, C and V. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| flaky test | A test that sometimes passes and sometimes fails without code changes. | [Lesson 05.6](05-rust-intermediate/06-testing-in-depth.md) |
| flame graph | A picture of many stacks: each function is a box whose width is its share of samples. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| flash memory | Memory that keeps its content without power; firmware lives there. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| flash page | The unit in which an SSD writes flash memory; a page can be written only when it is empty. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| flashcard | A card with a question on one side and a short answer on the other, used for spaced repetition. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| flashcards | Cards with a question on the front and a short answer on the back, e.g. in Anki. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| float | A number with a fractional part (f32, f64), stored approximately. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| floating input | An input pin connected to nothing; it reads random levels because of electrical noise. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| floating point | A way to store real numbers as a few binary digits times a power of two, so that both tiny and huge numbers fit. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| floating-point register | A register for f32 and f64 values (d0, s0 and so on on arm64); the calling convention passes floats in them. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| flow control | Keeping the sender from sending more than the receiver can store, using the receive window. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| flush | Force a buffer to be sent out now (stdout().flush()). | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| Fn / FnMut / FnOnce | Closure traits: only reads captured values (many calls) / changes them (many calls, one by one) / may move them out (one call). | [Lesson 05.4](05-rust-intermediate/04-closures.md) |
| fold / reduce (rayon) | Each thread folds its items into its own partial result, then the partial results are reduced into one, without locks. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| follower | A replica that copies the leader's changes and may serve reads; also called secondary or standby. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| follower / candidate / leader | The three Raft roles: a passive node, a node asking for votes, and the node that handles all writes and sends heartbeats. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| force-with-lease | git push --force-with-lease: a safer force that refuses if the remote branch changed since you last saw it. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| foreground / background job | A foreground job holds the terminal until it ends; a background job runs while you keep typing. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| foreign key | A column that points to the primary key of another table; the database makes sure the target row exists. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| fork | Your own copy of someone else's repository on GitHub or GitLab. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| fork bomb | A program that copies itself again and again until the machine cannot start any new process. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| formal verification | Proving with mathematical methods that a program has a property for every possible input, instead of trying examples. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| format specifier | The part after the colon in a placeholder, e.g. :&gt;8.2. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| format string / placeholder | The quoted text of println! and its {} slots replaced by values. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| forward proxy | A proxy that works for clients: they send their requests through it. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| forward secrecy | If the server's long-term key leaks later, old recorded traffic still cannot be decrypted. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| four-tuple | Source IP, source port, destination IP and destination port: the values that identify one TCP connection. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| fragment specifier | The part after the colon in $x:expr saying what kind of code the macro variable matches (expr, ident, ty, literal, tt, ...). | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| fragmentation | Splitting an IP packet bigger than the MTU into smaller packets; losing one fragment loses the whole datagram. | [Lesson 15.5](15-networking/05-udp-sockets-in-rust.md) |
| fragmentation (internal, external) | Wasted memory inside blocks (padding, rounding) or between blocks (free memory split into pieces too small to use). | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| frame | A page-sized block of physical memory that holds one page. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| frame (buffer pool) | One slot of a buffer pool, big enough for exactly one page. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| frame (protocol) | One complete message as it travels on the wire, including its framing such as a length prefix. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| frame / segment / datagram | Names of one unit of data at different layers: Ethernet frame, TCP segment, UDP datagram (an IP unit is a packet). | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| frame pointer | A register (x29 on arm64, rbp on x86-64) that points to the current stack frame; debuggers use it to walk the stack. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| Framed | A tokio_util type that combines a socket and a codec into a stream and a sink of messages. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| framing | Deciding where one message ends and the next begins in a byte stream. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| free list | A linked list of free blocks, stored inside the free memory itself, that lets an allocator reuse freed blocks. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| freestanding binary | A program that needs no operating system and brings everything it uses; a kernel is one. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| fresh-clone test | Cloning a repository into an empty folder and building and testing it in a clean environment, to prove nothing depends on your own machine. | [Lesson 23.10](23-capstone-projects/10-checkpoint-capstone-review.md) |
| From trait | The conversion trait. impl `From<A>` for B tells how to turn A into B; used by ?. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| FromStr | The standard trait behind text.parse::<T>(). | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| frontend | The part of a system the user sees: a web page, a phone app or a desktop program that sends requests to the backend. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| fsync | The system call that waits until a file's data and metadata are on the storage device; File::sync_all in Rust. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| fuel (step limit) | A budget of steps an interpreter or sandbox may run before it stops the program with an error. | [Lesson 22.23](22-specializations/23-compilers-track-an-interpreter.md) |
| full jitter | A jitter strategy that waits a random time between zero and the exponential backoff. | [Lesson 18.3](18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| full scan | Finding data by reading all of it from start to end. | [Lesson 17.5](17-storage-and-databases/05-hash-indexes.md) |
| fully qualified syntax | &lt;Type as Trait&gt;::function(...), which says exactly which trait's function to call. | [Lesson 06.2](06-rust-advanced/02-advanced-traits-and-types.md) |
| function | A named group of steps that can be reused. main is where Rust programs start. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| function pointer | A value of type fn(i32) -&gt; i32 pointing to a function (or a non-capturing closure). | [Lesson 05.4](05-rust-intermediate/04-closures.md) |
| function-like macro | A procedural macro called like name!(...), marked #[proc_macro]. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| functional requirement | What a system must do, for example redirect a short link to the long address. | [Lesson 24.7](24-career-and-open-source/07-coding-and-system-design-interviews.md) |
| fundamentals | Ideas that change very slowly: how memory, processes, networks, storage and agreement between machines work. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| future | A value standing for work that finishes later; anything implementing the Future trait. Futures are lazy. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| fuzz target | A small function that takes the bytes made by the fuzzer and passes them to the code under test. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| fuzzer | The program that generates inputs, runs the code under test and watches for failures. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| fuzzing | Running code again and again with automatically generated, often strange input to find crashes and hangs. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| FxHash | A very fast, unkeyed hash function from the Rust compiler (crate rustc-hash); not safe for untrusted keys. | [Lesson 19.6](19-performance-and-profiling/06-data-layout-for-speed.md) |

### G

| Word | What it means | First used in |
|---|---|---|
| game day | A planned session in which a team injects faults into a system on purpose and practises the reaction. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |
| garbage collection (Kubernetes) | The Kubernetes controller that deletes objects whose owner no longer exists. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| garbage collector | A runtime part that searches for unused memory from time to time. Rust has none. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| gateway | The computer that forwards packets to other networks; the default route points to it. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| gateway (next hop) | The router a packet is handed to when its destination is not in a directly connected subnet. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| gauge | A metric that can go up and down, such as requests in progress or memory in use. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| general-purpose register | A register for ordinary integers and addresses; arm64 has 31 (x0–x30), x86-64 has 16. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| generation (data files) | A number that grows each time compaction produces new data files and appears in their names. | [Lesson 17.14](17-storage-and-databases/14-project-key-value-store.md) |
| generation number | A number that grows with every rebalance, so that commits from members of an older assignment can be rejected. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| generator (Kustomize) | A Kustomize feature that creates a ConfigMap or Secret and adds a hash of its content to its name. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| generic | Code written once that works for many types; the type is a blank the compiler fills in. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| Git | The version control tool used by almost all teams. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| git blame | A Git command that shows, for every line of a file, the last commit that changed it. | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| Git dependency | A dependency Cargo clones from a Git repository (use tag or rev, not branch). | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| .git folder | The hidden folder holding the whole history. Deleting it deletes the history. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| git grep | A Git command that searches the files of a repository quickly and skips files that Git ignores. | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| git init | The command that turns a folder into a repository by creating the hidden .git folder. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| GitHub / GitLab | Services that host Git repositories and add teamwork tools. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| .gitignore | A file listing what Git must not track, such as /target. It affects only untracked files. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| GitOps | Deploying by keeping the desired state in Git and letting an agent inside the cluster pull and reconcile it. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| glob | A pattern such as *.txt that the shell expands into the list of matching file names. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| glob import | use path::\*; imports all public items of a module (avoid except use super::\* in tests). | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| global allocator | The one allocator used by Box, Vec, String and all standard collections in a program; chosen with #[global_allocator]. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| global ignore file | A file with patterns Git ignores in every repository on your computer (core.excludesFile). | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| #[global_allocator] | The attribute that tells Rust which allocator Box, Vec and String use. | [Lesson 22.5](22-specializations/05-os-track-project-and-next-steps.md) |
| global_asm! | A Rust macro that puts assembly code into the program outside of any function. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| glue record | An address sent together with a referral, so the named server can be reached without another lookup. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| god type | A struct that knows and does too much: many fields and methods about unrelated topics. | [Lesson 08.10](08-software-design/10-anti-patterns-and-code-smells.md) |
| gold-plating | Polishing a part far beyond what the project needs while important work waits. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| golden file | The stored expected output that a golden test compares with. | [Lesson 08.16](08-software-design/16-project-refactor-a-messy-program.md) |
| golden master | The saved output of the old program, used as the golden file while you rewrite it; it freezes the current behaviour. | [Lesson 19.10](19-performance-and-profiling/10-project-make-a-slow-program-fast.md) |
| golden test | A test that runs a program on a fixed input and compares the whole output with a stored file. | [Lesson 08.16](08-software-design/16-project-refactor-a-messy-program.md) |
| good first issue | A label on issues that are suitable for newcomers; some projects use E-easy or help wanted instead. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| goodput | The useful part of the throughput: correct answers that arrive in time. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| grace period | The time a shutdown waits for work to finish normally before it cancels what is left. | [Lesson 14.15](14-concurrency-and-async/15-project-concurrent-job-queue.md) |
| graceful shutdown | Stopping after finishing the work in progress (drop the sender, join the workers). | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| graceful upgrade | Replacing a running server process with a new version without closing its listening sockets or dropping requests. | [Lesson 22.20](22-specializations/20-networking-track-production-proxies.md) |
| Grafana | An open-source web tool that draws graphs and dashboards from data sources such as Prometheus. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| grammar | The rules that say which sequences of tokens form valid statements. | [Lesson 22.15](22-specializations/15-databases-track-parsing-sql.md) |
| graph | A set of vertices and edges that link pairs of vertices. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| grapheme cluster | What a human reads as one letter on screen; may be several code points. | [Lesson 04.8](04-structs-enums-collections-errors/08-strings-in-depth.md) |
| gray failure | A node that is not dead but works badly, for example very slowly or only for some requests; it is hard to detect. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| greedy algorithm | An algorithm that builds the answer step by step and at each step takes the choice that looks best now, without looking ahead or undoing it. | [Lesson 07.17](07-algorithms-and-data-structures/17-greedy-algorithms.md) |
| greedy-choice property | The fact that a locally best choice can always be extended to a best answer overall; a greedy algorithm is correct exactly when the problem has it. | [Lesson 07.17](07-algorithms-and-data-structures/17-greedy-algorithms.md) |
| group | A named set of users; a file belongs to one user and one group. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| group commit | Making many waiting writes durable with one fsync, so that each pays only part of its cost. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| group key | The values of the GROUP BY columns for one row; rows with equal keys form one group. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| guard | A value whose life means it holds something; when it is dropped the resource is released. | [Lesson 08.7](08-software-design/07-raii-and-guards.md) |
| guard page | A page with no permissions placed next to a stack so that a stack overflow crashes instead of overwriting memory. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |

### H

| Word | What it means | First used in |
|---|---|---|
| half-close | Closing only the sending direction with shutdown(Shutdown::Write) while still reading. | [Lesson 15.4](15-networking/04-tcp-sockets-in-rust.md) |
| half-open range | A range lo..hi that includes lo but not hi; it is empty when lo == hi. | [Lesson 07.10](07-algorithms-and-data-structures/10-binary-search.md) |
| handle | The small fixed-size part of a value on the stack that points to the real data (for String: pointer, length, capacity). | [Lesson 03.3](03-ownership-and-borrowing/03-move-copy-and-clone.md) |
| handler | The function in a service that runs for one endpoint and builds the response. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| handshake | The first messages of a TLS connection, in which both sides agree on algorithms and keys. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| happens-before | A guarantee between two operations: everything the first one did is visible to the second one. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| hard link | A second name for the same file (the same inode). | [Lesson 11.2](11-linux-shell-and-containers/02-the-linux-file-system.md) |
| hard state | The part of a Raft node that must survive a crash: the current term, the vote and the log. | [Lesson 18.14](18-distributed-systems/14-project-replicated-key-value-store.md) |
| hardening | Settings that reduce what a program is allowed to do, so that a bug or an attacker can do less damage. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| hardship | Problems injected into a simulation on purpose: delays, lost messages, partitions, crashes. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| hash | A short fingerprint computed from content: the same content gives the same hash, any change gives a different one. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| Hash / Hasher | The Rust trait for types that can be hashed, and the object that receives the bytes and produces the hash number. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| hash aggregation | Grouping rows by putting each one into a map entry for its group key and updating running totals there. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| hash flooding | A DoS attack that chooses many keys with the same hash to slow a map. Rust's default hasher resists it. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| hash function | A function that turns any value into a number (the hash). | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| hash index | An index kept in a hash table that maps each key to the position of its latest record in a file. | [Lesson 17.5](17-storage-and-databases/05-hash-indexes.md) |
| hash join | A join method: build a hash table from the smaller input, then look up every row of the bigger input in it. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| hash map | A collection of key → value pairs. Lookup takes about the same time for any size. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| hash partitioning | Choosing the partition by a hash of the key; spreads keys evenly but loses their order. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| hasher | The algorithm that turns a key into a hash number for a HashMap; the default is SipHash. | [Lesson 19.6](19-performance-and-profiling/06-data-layout-for-speed.md) |
| hazard pointers | Each thread publishes the pointers it is reading right now, and nobody frees a node that is in that list. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| HEAD | A pointer to the commit or branch you are on now. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| head-of-line blocking | One lost or slow item holds back everything behind it, as in TCP's in-order delivery. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| header (HTTP) | A Name: value line after the request line or status line. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| header (network) | Bytes at the front of a packet with information for one layer, such as addresses and lengths. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| header field (HTTP) | One Name: value line after the start line of an HTTP message; names are case-insensitive. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| header file | A .h file listing the functions and types of a C library without their code. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| heading | A title line in Markdown starting with one or more #. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| headroom | The distance between the load a system runs at and the load at which it starts to miss its goals. | [Lesson 23.8](23-capstone-projects/08-testing-benchmarking-and-hardening-capstones.md) |
| HEAD~N | The commit N steps before HEAD. | [Lesson 01.4](01-git-and-version-control/04-viewing-history-and-changes.md) |
| health check | A command run regularly inside a container whose exit status says whether the service is healthy. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| heap | A large pool of memory for data of unknown or changing size, or data that must outlive a function call. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| heap (data structure) | A tree of values in which every parent is at least as big as its children (max-heap) or at most as big (min-heap); not the heap memory of Phase 03. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| heap profiler | A tool that records every allocation: where it happened, its size and its lifetime. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| heap property | The rule of a max-heap: every parent is greater than or equal to both of its children. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| heapify | Turning an unordered vector into a heap in O(n). | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| heapsort | A sorting algorithm that builds a heap and takes the biggest item out again and again; O(n log n), in place, not stable. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| heartbeat | A small "I am alive" message that a node sends regularly so that others know it is running. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| height (of a tree) | The number of levels of a tree: the longest path from the root down to a leaf, counted in nodes. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| heisenbug | A bug that disappears or changes when you try to observe it, for example by adding a println!. Concurrency bugs often are. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| Helm | A package manager for Kubernetes that installs a set of manifests as one release and keeps its history. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| helper function | A function that the kernel offers to eBPF programs, for example to read the current process ID. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| heuristic | A rule that is right most of the time but not always, for example 'a file with a zero byte is binary'. | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| hex | Writing bytes as two characters from 0-9a-f each. 32 bytes become 64 characters. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| hexadecimal (hex) | Counting with 16 digits (0-9, a-f). Two hex digits are one byte; numbers start with 0x. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| hexagonal architecture (ports and adapters) | A way of building a program with the domain and use cases in the middle and ports and adapters around them, so that dependencies point inwards. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| hidden bit | The leading 1 before the binary point of a normal float. It is always 1, so it is not stored. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| hidden file | A file whose name starts with a dot (a dotfile). | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| high watermark | The highest offset that every in-sync replica has; consumers may read only below it. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| high-level / low-level code | High-level code decides what the program does; low-level code does technical details such as files and screen. | [Lesson 08.3](08-software-design/03-solid-principles.md) |
| higher-ranked trait bound | A bound that must hold for every lifetime 'a; Fn(&str) -&gt; &str is shorthand for it. | [Lesson 06.5](06-rust-advanced/05-lifetimes-in-depth.md) |
| histogram | A metric that records many measured values and groups them into ranges (buckets), with a count, sum, minimum and maximum. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| histogram bucket | A range of a histogram with an upper limit le; it counts how many values were at most that limit. Buckets are cumulative. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| history | The list of all commits from the first to the latest. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| history (test) | The recorded list of operations a test performed, with start and end times and results, checked afterwards for correctness. | [Lesson 18.14](18-distributed-systems/14-project-replicated-key-value-store.md) |
| hit ratio | Hits divided by all accesses of a cache; 0.99 means 99 of 100 accesses were found in memory. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| home directory | Your personal directory, written ~ (tilde). | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| Homebrew | A popular package manager for macOS (brew); optional. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| homoglyph | A character that looks like another one, such as the Cyrillic а and the Latin a. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| hook | A script Git runs automatically at certain moments, e.g. pre-commit. It lives in .git/hooks and is not part of the repository. A non-zero exit cancels the commit. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| hook (filter) | A method that a framework calls at a fixed moment, so your code can inspect or change what happens. | [Lesson 22.20](22-specializations/20-networking-track-production-proxies.md) |
| hop-by-hop header | An HTTP header meant for one connection only, such as Connection or Transfer-Encoding; a proxy removes it. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| horizontal operation | An operation between the lanes of one vector register, for example adding its four lanes together. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| horizontal scaling | Handling more load by running more instances of a service instead of using a bigger machine. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| host (network) | Any computer or container with an IP address; host bits are the bits after the prefix. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| host (turmoil) | A simulated machine that runs a server program and can be crashed and restarted. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| host (WebAssembly) | The program that loads and runs a WebAssembly module and provides its imports. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| host / guest | The real computer (your Mac) is the host; the operating system inside the virtual machine is the guest. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| host function | A function that a WebAssembly module imports and the host provides, such as print_int. | [Lesson 22.25](22-specializations/25-compilers-track-project-and-next-steps.md) |
| host key | The key pair of the server itself, which proves to the client that it talks to the right machine. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| hostPath volume | A volume that mounts a folder of the node itself into a pod. Powerful and risky, so it is given only to trusted agents and mounted read-only where possible. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| hot / cold data | Hot data is used on every pass of a frequent loop, cold data rarely; keeping them apart keeps the working set small. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| hot data | The data that a program's busy loop reads again and again; its layout decides the speed. | [Lesson 19.6](19-performance-and-profiling/06-data-layout-for-speed.md) |
| hot reload | Applying a new configuration while a program runs, without dropping connections or requests. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| hot spot | A partition that gets much more load than the others. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| HTTP method | What the client wants to do, such as GET to read or POST to send data. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| HTTP status code | The number in an HTTP response: 200 ok, 404 not found, 500 server error. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| HTTP/3 | HTTP carried over QUIC; every request uses its own QUIC stream. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| hunk | One continuous block of changes inside a diff, introduced by an @@ header. | [Lesson 01.4](01-git-and-version-control/04-viewing-history-and-changes.md) |
| hybrid logical clock | A timestamp made of a wall-clock time and a small logical counter; close to real time but respecting causality. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| hygiene | Local names created inside a macro cannot clash with names in the calling code (items are not hygienic). | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| hyper | A fast, low-level Rust HTTP library that parses and writes HTTP/1.1 and HTTP/2 messages and manages connections, but does no routing. | [Lesson 15.9](15-networking/09-http-with-hyper-and-reqwest.md) |
| hypervisor | The part of the host that runs virtual machines; on a Mac it is Apple's Virtualization framework. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| hypothesis | The expected result of an experiment, written down before it runs. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |

### I

| Word | What it means | First used in |
|---|---|---|
| I/O multiplexing | Watching many descriptors with one thread and handling whichever is ready. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| I/O-bound vs CPU-bound | I/O-bound work mostly waits (async helps); CPU-bound work mostly calculates (threads help). | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| ice-cream cone | The upside-down pyramid: many slow end-to-end tests and few unit tests. It makes a test suite slow and fragile. | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| idempotency key | A unique value a client sends with a non-idempotent request so the server can recognise and ignore a repeat. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| idempotent | An operation that gives the same result when applied twice as when applied once, such as sorting a sorted list. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| idempotent method | An HTTP method whose effect is the same whether it is sent once or many times, such as GET, PUT and DELETE; it can be retried safely. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| idempotent producer | A producer whose retried batches the broker recognises by producer ID and sequence number and stores only once. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| identifier | A name in a program or query: a table, column or function name. | [Lesson 22.15](22-specializations/15-databases-track-parsing-sql.md) |
| identity mapping | A mapping where the virtual address equals the physical address. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| idle timeout | Closing a connection that has sent nothing for a certain time. | [Lesson 15.6](15-networking/06-async-networking-with-tokio.md) |
| if let | if let PATTERN = VALUE { } runs the block only if the value fits the pattern. | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| image (container) | A read-only package of layers with everything a container needs to start. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| image digest | The SHA-256 hash of an image's content; unlike a tag, it can never point to different content. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| ImagePullBackOff | A pod status meaning the node could not download the image and waits longer and longer before trying again. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| imagePullPolicy | The field that says when the kubelet downloads an image: IfNotPresent, Always or Never. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| immediate (value) | A constant written inside an instruction, such as #3 on arm64. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| immutable | Cannot be changed after it is set. The default for Rust variables. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| immutable infrastructure | Never changing a running server or container; instead you build a new version and replace the old one. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| imperative | Describing how: a list of steps to run one after another. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| impl block | impl Type { ... } holds the functions belonging to a type, or implements a trait for it (impl Trait for Type). | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| impl Trait | Writing a parameter or return type as "some type implementing this trait" without naming it. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| implement a trait | Write impl Trait for Type { … }, giving the type the trait's methods. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| in doubt | The state of a 2PC participant that voted yes and has not heard the decision; it may neither commit nor abort on its own. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| in flight | Submitted but not yet completed. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| in place | Working inside the memory of the input with only a few extra variables. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| in-degree | The number of edges that lead into a vertex. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| in-memory | Data that lives in RAM instead of on disk: very fast, but it may be lost when the server stops. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| in-order traversal | Visit the left subtree, then the node, then the right subtree; on a binary search tree it gives sorted order. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| in-process test | A test that calls the application's code directly in the same process instead of over a real network connection. | [Lesson 16.14](16-backend-services/14-testing-a-backend-service.md) |
| incident | An unplanned event that hurts users or puts the service at risk and needs a coordinated response now. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| index | The position of an item in an array or vector; counting starts at 0. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| index (database) | An extra sorted structure that lets the database find rows by a column quickly, at the cost of space and slower writes. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| indirection | Reaching data through a pointer instead of directly; each indirection can cost one more cache miss. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| infinite loop | A loop that never ends; stop it with Ctrl+C. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| information disclosure | Letting data reach someone who must not see it. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| infrastructure | Code that talks to the outside world: files, databases, networks, the clock. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| inheritance | A feature of some languages where a child type receives the fields and methods of a parent type; Rust has none between structs. | [Lesson 08.4](08-software-design/04-composition-over-inheritance.md) |
| init process | PID 1 of a system or PID namespace: it adopts orphans, is protected from signals without handlers, and its exit ends the namespace. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| injection | An attack where input is placed into a command, query or document so that the receiver treats data as code. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| inline snapshot | A snapshot written inside the Rust source file, in the test itself, instead of a separate file. | [Lesson 09.5](09-testing-debugging-benchmarking/05-snapshot-testing.md) |
| inlining | Copying the body of a called function into the caller, which removes the call and enables more optimisations. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| inode | The record in a file system that stores everything about a file except its name: size, owner, permissions, times and the place of the data. | [Lesson 11.2](11-linux-shell-and-containers/02-the-linux-file-system.md) |
| input / output | Data a program receives / data it produces. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| input size | The number, usually called n, that says how big the input is: the length of a vector or the number of words in a text. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| instance | One concrete value of a struct (or other) type. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| instance (Lima) | One virtual machine managed by Lima, such as dev. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| instruction | One small exact step, for example "add these two numbers". | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| instrumentation | Extra counting code that the compiler adds to a program so that it records what was executed. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| instrumented binary | A special build that counts how often each branch and function runs and writes the counts to a file when it exits. | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| integer | A whole number without a fractional part (i8..i128, u8..u128, isize, usize). | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| integration test | A test in tests/ that uses the crate from outside as a separate crate; it sees only the public API. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| integrity | Data and programs are changed only in allowed ways, by allowed people. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| Intel syntax / AT&T syntax | Two ways to write x86-64 assembly: Intel puts the destination first, AT&T puts it last and marks registers with %. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| interaction verification | Checking which calls were made to a dependency, how many times and with which arguments. | [Lesson 09.3](09-testing-debugging-benchmarking/03-test-doubles.md) |
| interactive rebase | git rebase -i: edit the list of commits (pick, reword, squash, fixup, drop, reorder). | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| interactive staging | Staging only some hunks of a file so each commit holds one idea. | [Lesson 05.4](05-rust-intermediate/04-closures.md) |
| interest list | The set of descriptors and events that an epoll instance watches. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| interface file (cgroup) | A file in a cgroup directory that shows a value or sets a limit, such as memory.max. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| interior mutability | Changing a value through a shared reference &T; the type itself (Cell, RefCell, Mutex) makes this safe. | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| interleaving | One possible order in which the steps of several threads are mixed together in time. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| intermediate certificate | A certificate signed by a root that in turn signs ordinary website certificates. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| internal node | A tree node that has children, as opposed to a leaf. | [Lesson 17.6](17-storage-and-databases/06-b-trees.md) |
| internal platform | The shared tools and services a company builds for its own developers: deployment, clusters, logs, metrics. | [Lesson 24.1](24-career-and-open-source/01-roles-for-systems-engineers.md) |
| interpreter | A helper program that translates code line by line while the program runs (contrast with a compiler). | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| interrupt | A signal from a device (timer, disk, network card) that makes the CPU run kernel code. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| interview loop | The series of interviews for one job, often on one day or over a few weeks. | [Lesson 24.7](24-career-and-open-source/07-coding-and-system-design-interviews.md) |
| interview-style problem | A short, self-contained question with a clear input and output, of the kind asked in job interviews. | [Lesson 07.18](07-algorithms-and-data-structures/18-problem-solving-practice.md) |
| IntoIterator | Trait meaning "can be turned into an iterator". for loops use it. | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| IntoResponse | The axum trait for values that can be turned into an HTTP response: text, Json, StatusCode, tuples with headers, Result. | [Lesson 16.3](16-backend-services/03-web-services-with-axum.md) |
| intra-doc link | A link written [`Name`] pointing to another item; rustdoc checks it exists. | [Lesson 05.7](05-rust-intermediate/07-documentation.md) |
| intrinsic | A function that the compiler replaces by one specific CPU instruction, such as vaddq_f32 for a NEON addition of four floats. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| invalidation (cache) | Removing or updating a cached copy because the real data changed. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| invariant | A statement that is true every time a loop repeats; it helps to prove that an algorithm is correct. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| invariant (data) | A rule about your data that must always be true, for example "the total amount of money never changes". | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| inversion | A pair of items in the wrong order: the left one is bigger than the right one. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| io_uring | A Linux interface for asynchronous I/O: operations are queued in shared memory and the kernel reports when each one is done. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| IP address | The number that identifies a network interface, such as 172.30.0.2. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| IP spoofing | Sending packets with a false source address; easy with UDP because there is no handshake. | [Lesson 15.5](15-networking/05-udp-sockets-in-rust.md) |
| is-a / has-a | Two kinds of relationship: a dog is an animal (inheritance thinking), a player has a position (composition). | [Lesson 08.4](08-software-design/04-composition-over-inheritance.md) |
| isolation | Keeping parts of a program apart, so that a failure in one part does not spread to the others. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| isolation level | A setting that says which anomalies a transaction accepts in exchange for speed. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| issue | A numbered discussion in a project's tracker about one bug, idea or question, for example #12. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| Item | The associated type of Iterator: the type of produced items. | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| iterator | Something that gives items one at a time via next. A range is an iterator. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| iterator invalidation | Changing a collection while looping over it so the loop reads memory that has moved. | [Lesson 03.5](03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| iterator pipeline | A chain of adapters ending in a consumer. | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |

### J

| Word | What it means | First used in |
|---|---|---|
| Jaeger | An open-source tool that stores traces and shows them as timelines in a web UI. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| JetStream | The part of NATS that stores messages so they can be delivered later and again (at-least-once). | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| jitter | Randomness added to a wait so that many clients do not act at the same moment. | [Lesson 18.3](18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| job | A group of steps that run one after another on one runner. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| job control | How an interactive shell moves jobs between foreground and background with process groups and tcsetpgrp. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| job description | The text of a job advertisement: what the team does, what you will do and what they expect you to know. | [Lesson 24.1](24-career-and-open-source/01-roles-for-systems-engineers.md) |
| job queue | A part of a program that accepts units of work, keeps them in order and lets workers run them. | [Lesson 14.15](14-concurrency-and-async/15-project-concurrent-job-queue.md) |
| job title | The name written on a job offer. The same title can mean different work in different companies. | [Lesson 24.1](24-career-and-open-source/01-roles-for-systems-engineers.md) |
| join | Combining rows of two tables that belong together in one query result; LEFT JOIN also keeps rows without a partner. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| join! and spawn | join! runs futures concurrently inside one task and can borrow locals; spawn creates independent tasks needing Send + 'static. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| JoinError | The error from a task that panicked or was aborted. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| JoinHandle | What tokio::spawn returns; awaiting it gives Result<T, JoinError>. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| JoinHandle / join | Value returned by thread::spawn. join() waits for the thread and returns a Result with its value (Err if it panicked). | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| JoinSet | A Tokio collection of tasks whose results come in finishing order; dropping it aborts all its tasks. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| journal | The central log of systemd, read with journalctl; everything a service writes to stdout and stderr goes there. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| journal (file system) | An area where the file system first records planned changes, so a crash cannot leave its structures half-changed. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| JSON Schema | A standard way to describe which fields a JSON or YAML document may have and of which types. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| judge (test) | A slow, obviously correct function that tests use to check a fast function on many inputs. | [Lesson 07.18](07-algorithms-and-data-structures/18-problem-solving-practice.md) |

### K

| Word | What it means | First used in |
|---|---|---|
| k-way merge | Merging k sorted runs into one in a single pass, using a heap to find the smallest next key. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| Kahan summation | A way to add many floats that remembers the rounding error of each step and adds it back, so the total is much more accurate. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| Kahn's algorithm | A topological sort that repeatedly takes a vertex with in-degree 0; it also detects cycles; O(V + E). | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| Kani | A model checker for Rust, open source from Amazon Web Services, that checks for panics, overflow, invalid memory use and failed assertions. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| KDF parameters | The cost settings of Argon2: memory (m_cost), passes (t_cost) and parallelism (p_cost). | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| keep-alive | Reusing one network connection for many HTTP requests instead of opening a new one each time. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| kernel | The central part of an operating system; it talks to the hardware and gives programs memory, files and network. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| kernel module | A piece of kernel code, usually a driver, that can be loaded while the kernel runs. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| kernel shell | A tiny command loop inside the kernel itself, reading lines from the serial console. | [Lesson 22.5](22-specializations/05-os-track-project-and-next-steps.md) |
| kernel side / user side | The eBPF program that runs in the kernel, and the program that loads it and reads its results. | [Lesson 22.7](22-specializations/07-ebpf-track-a-first-program-with-aya.md) |
| kernel space | The memory and CPU mode where the kernel runs; user programs cannot read or change it. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| key | The label used to find a value in a map; appears at most once. Also a cryptographic key. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| key exchange | A way for two sides to compute the same shared secret without sending it over the network. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| key pair | Two linked files: the secret private key stays on your computer; the public key is given to the service. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| key-range partitioning | Each partition owns a continuous range of keys; good for range queries, but prone to hot spots. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| keyless signing | Signing with a short-lived certificate that the build system gets by proving its identity, instead of a long-lived private key. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| keyring | A file with the public keys that apt trusts for a repository. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| keystream | The long secret byte stream that a stream cipher creates from a key and a nonce and XORs with the plaintext. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| keyword | A reserved word of the language (fn, let, mut, const) that cannot be used as a name. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| knee | The point on a load-to-latency curve where latency starts to grow fast. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| known_hosts | The file on the client with the host keys of servers it has connected to before. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| kprobe | A hook at the start of almost any kernel function; powerful but not stable between versions. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| kube-proxy | A component on every node that makes Service addresses work by programming network rules. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| kube-rs | The Rust project for Kubernetes: a client, a controller runtime and derive macros, published as the crate kube. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| kubeconfig | A file, by default ~/.kube/config, that lists clusters, users with their credentials and contexts; keep it out of Git. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| kubeconfig / context | The file that tells kubectl and client libraries which clusters exist and how to log in / the cluster and user selected right now. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| kubectl | The command-line client for Kubernetes; it sends your requests to the API server. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| kubelet | The agent on every Kubernetes node that starts and stops the containers of its pods and reports their state. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| Kubernetes | An open-source orchestrator that runs containers on a cluster and keeps the actual state equal to the declared desired state. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| Kustomize | A tool built into kubectl that builds manifests from a base plus overlays that patch it, without templates. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |

### L

| Word | What it means | First used in |
|---|---|---|
| label | A short note the compiler prints beside your code, such as "value moved here". | [Lesson 03.7](03-ownership-and-borrowing/07-fixing-common-ownership-errors.md) |
| label (assembly) | A name for a position in the code, ending with a colon; branches jump to labels. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| label (DNS) | One part of a domain name between dots, at most 63 bytes long. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| label (metrics) | A key-value pair such as status="404" that splits one metric into several time series. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| label selector | A rule such as "all pods with app.kubernetes.io/name: hello"; Deployments and Services use it to find their pods. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| lagging | A broadcast receiver that is so slow that old messages were overwritten before it read them. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| lagging (broadcast) | A broadcast receiver so slow that the channel dropped messages it had not read yet; recv returns Lagged(n). | [Lesson 15.6](15-networking/06-async-networking-with-tokio.md) |
| Lamport clock | A logical clock with one counter per node: tick on every event, jump to max(own, received) + 1 on receive. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| lane | One slot of a vector register. A 128-bit register has 4 lanes of 32 bits or 16 lanes of 8 bits. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| latch | A one-time gate: threads wait until a counter reaches zero, then all of them go on. | [Lesson 14.3](14-concurrency-and-async/03-condition-variables.md) |
| latency | How long one operation takes from start to result. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| layer | A group of code with one kind of job, placed in an order, where code may use the layers below it. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| layer (network) | One level of the network design; each layer solves one problem and uses the layer below. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| layer (tower) | A value that wraps a service in a middleware; layers nest like the skins of an onion, and the last .layer() is the outermost. | [Lesson 16.4](16-backend-services/04-errors-validation-and-middleware.md) |
| Layout | The std::alloc type that describes a memory block by its size and alignment. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| layout (of a type) | Where the bytes of a value go: its size, its alignment and the position of each field. | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| lazy | Does no work until the result is needed. | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| lazy (future) | A future does no work until it is polled. | [Lesson 14.10](14-concurrency-and-async/10-how-futures-work.md) |
| lazy unmount (MNT_DETACH) | Removing a mount from the tree at once; the kernel frees it when nobody uses it any more. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| LazyCell / LazyLock | A value computed by a closure on first access. LazyLock works between threads and replaces lazy_static!. | [Lesson 06.4](06-rust-advanced/04-interior-mutability.md) |
| leader | The one replica that accepts writes and decides their order; also called primary. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| leaderless replication | Clients write to and read from several replicas directly; no replica is special. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| leaf | A tree node that has no children. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| leaf future | A future that waits for something outside (a timer, a socket, another thread), not for other futures; written by hand. | [Lesson 14.10](14-concurrency-and-async/10-how-futures-work.md) |
| leak | A value that is never dropped, so its cleanup never runs; safe in Rust but rarely wanted. | [Lesson 08.7](08-software-design/07-raii-and-guards.md) |
| leap second | An extra second sometimes added to UTC to keep it in line with the Earth's rotation; systems handle it differently. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| learner (Raft) | A non-voting node that receives the log to catch up before it becomes a voter. | [Lesson 23.3](23-capstone-projects/03-capstone-distributed-key-value-store.md) |
| learning journal | A short written log after each lesson: learned, confusing, remember, question, next. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| learning plan | A written plan for the next months: what you will build, contribute, read and write, and when you review it. | [Lesson 24.9](24-career-and-open-source/09-checkpoint-course-complete.md) |
| learning repository | The single Git repository ~/projects/rust that holds all course code and notes. | [Lesson 01.10](01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) |
| lease | A lock that is valid only for a limited time, so that it frees itself if its holder dies. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| least connections | Choosing the backend with the fewest open connections. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| least privilege | Give every person and program only the permissions it needs, nothing more. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| left recursion | A grammar rule that starts with itself; written as recursive descent, it recurses forever. | [Lesson 22.22](22-specializations/22-compilers-track-lexing-and-parsing.md) |
| Leitner system | Spaced repetition with boxes: a remembered card moves to a box reviewed less often, a forgotten card goes back to box 1. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| length (len) | How many items a collection holds now. | [Lesson 04.7](04-structs-enums-collections-errors/07-vectors.md) |
| length extension | A weakness of plain SHA-256: from hash(key + message) an attacker can compute hash(key + message + extra) without the key. HMAC avoids it. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| length prefix | A number in front of a message that says how many bytes follow. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| length-prefixed format | A format where the first bytes say how many bytes of data follow. The length must be limited before allocating. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| lesson | One text file studied in one session, numbered like 02.4. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| let chain | Several conditions and let patterns joined with && in one if/while (edition 2024, Rust 1.88+). | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| let else | let PATTERN = VALUE else { ... }; the else block must diverge. | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| let-else | A Rust statement `let pattern = value else { ... };` whose else block must leave the function when the pattern does not match. | [Lesson 16.8](16-backend-services/08-authentication-in-a-web-service.md) |
| level-triggered | epoll keeps reporting a descriptor while it is ready; the default mode. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| level-triggered / edge-triggered | Acting on the current state / acting only on single change events. Kubernetes controllers are level-triggered, so a lost event does no harm. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| leveled compaction | A compaction strategy with levels that grow about ten times each; inside a level the tables do not overlap. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| lexer | The first step of a compiler: it cuts source text into tokens. | [Lesson 22.22](22-specializations/22-compilers-track-lexing-and-parsing.md) |
| libc | The C standard library of the system; on Linux it provides malloc and free, which Rust's default allocator uses. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| libFuzzer | The fuzzing engine of the LLVM project that cargo-fuzz drives. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| library crate / binary crate | A crate built from src/lib.rs (reusable code, no main) / from src/main.rs (a runnable program). | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| library error / application error | A precise enum for callers to match (thiserror) / a catch-all error shown to a human with context (anyhow). | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| licence | The legal text that says what others may do with your code; without one, nobody may legally reuse it. | [Lesson 23.9](23-capstone-projects/09-presenting-your-work.md) |
| licence (LICENSE) | A file saying what others may do with your code. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| license | The legal terms for using code. Tools such as cargo-deny check that all dependencies use licenses you allow. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| lifetime | The period during which a reference is valid, checked by the compiler. | [Lesson 03.4](03-ownership-and-borrowing/04-references-and-borrowing.md) |
| lifetime annotation | A name like 'a written on references to describe how their lifetimes relate. | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| lifetime elision | Compiler rules that fill in obvious lifetimes so you need not write them. | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| lifetime parameter | The declaration of a lifetime name in angle brackets, e.g. fn `longest<'a>`. | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| lightning talk | A very short talk, usually five minutes, about one idea. | [Lesson 24.2](24-career-and-open-source/02-building-your-portfolio.md) |
| Lima | A free tool that creates and starts Linux virtual machines on a Mac; its command is limactl. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| line buffering | A buffer that is flushed at every newline; Rust's standard output works this way. | [Lesson 19.9](19-performance-and-profiling/09-async-and-io-performance.md) |
| line coverage | The share of source lines that were executed at least once, in percent. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| linear | O(n): when the input doubles, the work doubles. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| linear memory | The single byte array a WebAssembly module can read and write; it cannot reach any other memory. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| linear search | Looking at the items one by one from the start until the wanted one is found; O(n). | [Lesson 07.10](07-algorithms-and-data-structures/10-binary-search.md) |
| linearizability | The system behaves as if there were one copy of the data and every operation took effect at one instant between its start and end. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| linearizable read | A read that returns the value of the latest acknowledged write, as if there were only one copy of the data. | [Lesson 18.14](18-distributed-systems/14-project-replicated-key-value-store.md) |
| link count | How many directory entries point to an inode; the data is freed when it is 0 and the file is not open. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| link register | On arm64, register x30: bl stores the return address there and ret jumps back to it. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| link-local address | An address that works only on one local link and never through a router, such as 169.254.0.0/16 or fe80::/10. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| linked list | A chain of nodes, each holding a value and a link to the next node. | [Lesson 07.4](07-algorithms-and-data-structures/04-linked-lists.md) |
| linker | A program that joins compiler output into one executable. The step is called linking. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| Linker (wasmtime) | The wasmtime type that connects a module's imports to host functions before instantiation. | [Lesson 22.25](22-specializations/25-compilers-track-project-and-next-steps.md) |
| linker script | A file that tells the linker where to put code and data and which symbols to define. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| lint | An automatic check for suspicious or unidiomatic code. Clippy is a collection of lints. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| lint group | A named set of lints that you switch on or off together, like clippy::pedantic. | [Lesson 08.11](08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) |
| lint level | What happens when a lint fires: allow, warn, deny, forbid or expect. | [Lesson 08.11](08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) |
| linter | A tool that warns about suspicious or unidiomatic code. Rust's linter is Clippy. | [Lesson 04.11](04-structs-enums-collections-errors/11-project-phone-book.md) |
| linter / lint | A tool that reads code without running it and warns about suspicious or unidiomatic code; one warning rule is a lint. | [Lesson 08.10](08-software-design/10-anti-patterns-and-code-smells.md) |
| Linux | The kernel used by most servers and by Android; also used for a whole system built around it. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| listening socket | A socket that only accepts new connections; in Rust a TcpListener. | [Lesson 15.4](15-networking/04-tcp-sockets-in-rust.md) |
| literal | A value written directly in code, such as 1_000, 0xff, b'A'. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| litmus test | A tiny program of two or three threads that checks whether one particular reordering can happen. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| Little's law | Requests in progress = throughput × latency. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| little-endian | Byte order where the least significant byte comes first. Apple M-series Macs and x86-64 computers use it. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| little-endian / big-endian | Byte orders: the lowest byte of a number comes first / the highest byte comes first. | [Lesson 17.3](17-storage-and-databases/03-binary-file-formats.md) |
| livelock | Threads keep reacting to each other and stay busy, but none of them makes progress. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| liveness | "Something good eventually happens", for example the cluster eventually elects a leader. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| liveness check | A health check that asks whether the process is alive and can answer; if it fails, the service is restarted. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| liveness probe | Asks whether a program is still working; after repeated failures the kubelet restarts the container. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| llvm-tools | A rustup component with LLVM tools (llvm-objdump, llvm-readobj, llvm-nm, llvm-size) to look inside binaries. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| load / store instruction | An instruction that copies data from memory into a register (load) or from a register into memory (store). | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| load balancer | A reverse proxy that spreads connections or requests over several equal backends and avoids those that are down. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| load factor | The number of entries divided by the number of buckets; it is the average chain length. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| load generator | The program that sends the requests in a load test, for example oha. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| load imbalance | Some threads finish early and sit idle while others still work, so the whole job waits for the slowest thread. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| load shedding | Refusing extra work at once with a cheap error, so that the accepted work stays fast. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| load test | Running a service under many concurrent requests to measure throughput, percentiles and errors. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| loader (dynamic linker) | The program (dyld on macOS, ld.so on Linux) that maps shared libraries into a new process, fills in addresses and jumps to the entry point. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| localhost / loopback | The address 127.0.0.1: this same computer. Only local programs can connect. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| locality (temporal, spatial) | Programs tend to reuse the same data soon (temporal) and to use data next to it (spatial). Caches work because of this. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| lock file | File made by Cargo recording the exact version and checksum of every crate in a build. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| lock granularity | How much data one lock protects: coarse (one lock for everything) or fine (many locks for small parts). | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| lock guard | A value that proves a lock is held and releases the lock when it is dropped. | [Lesson 12.14](12-computer-systems/14-project-your-own-allocator.md) |
| lock ordering | A rule that every thread takes several locks in the same fixed order. It prevents deadlocks. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| lock-free | A progress guarantee: at any moment some thread makes progress, even if other threads are stopped anywhere. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| log | A record of what a program did, written while it runs so that it can be read later. | [Lesson 09.10](09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| log compaction | Deleting old log entries after a snapshot so that the log does not grow forever. | [Lesson 23.3](23-capstone-projects/03-capstone-distributed-key-value-store.md) |
| log conflict | A follower's entry at some index has a different term than the leader's; it was never committed, so the follower deletes it and everything after it. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| log end offset | The offset that the next appended record will get. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| log entry | One change in a log, for example 'put apple = red' or 'delete kiwi'. | [Lesson 17.4](17-storage-and-databases/04-write-ahead-log.md) |
| log file | A text file where a program writes what happened, one event per line. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| log injection | An attack where text typed by the attacker contains a newline and so creates fake lines in a log file. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| log level | How important a log message is: ERROR, WARN, INFO, DEBUG or TRACE; you choose which levels to show. | [Lesson 09.10](09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| log matching | Raft's promise: if two logs have an entry with the same index and term, they are identical up to that index. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| log rotation | Limiting how much log is kept by dropping the oldest part when a size is reached. | [Lesson 11.15](11-linux-shell-and-containers/15-project-deploy-a-rust-service.md) |
| log-structured storage | A design where the data file itself is an append-only log and nothing is changed in place. | [Lesson 17.5](17-storage-and-databases/05-hash-indexes.md) |
| logarithm | Here: how many times n can be cut in half before it reaches 1. It grows very slowly. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| logfmt | A simple log format of key=value pairs on one line. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| logic bug | The program runs without complaint but gives a wrong result. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| logical clock | A counter that orders events by happens-before instead of by physical time. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| logical plan | A tree of relational operators that describes what a query computes. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| logical shift | A right shift that fills the new high bits with zeros; Rust uses it for unsigned types. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| long polling | A request that waits on the server until data arrives or a time limit passes, instead of returning empty at once. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| longest prefix match | Of all matching routes, the one with the longest (most specific) prefix is chosen. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| loom | A model checker for concurrent Rust: it runs a small test under every interleaving and allowed reordering. It treats SeqCst accesses as AcqRel. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| loop | A way to repeat steps many times; one run of the body is an iteration. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| loop device | A kernel device that makes an ordinary file look like a disk, so a file system can be created and mounted in it. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| loop invariant | A fact that is true before and after every turn of a loop and helps to prove the loop correct. | [Lesson 07.10](07-algorithms-and-data-structures/10-binary-search.md) |
| loop label | A name like 'outer that break and continue can target. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| loopback | The address 127.0.0.1 (::1), which always means this same computer. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| loose coupling | A design where parts know little about each other, so one can change or fail without breaking the others. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| lost update | A race condition: two threads read the same old value, both write a new value, and one of the writes is lost. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| lost wake-up | A notification sent when nobody waits is lost, and a thread that starts waiting later sleeps forever unless it checks a condition first. | [Lesson 14.3](14-concurrency-and-async/03-condition-variables.md) |
| lower bound | The index of the first item that is not smaller than the target. | [Lesson 07.10](07-algorithms-and-data-structures/10-binary-search.md) |
| LSM tree | A log-structured merge tree: a memtable plus immutable sorted SSTables that are merged by compaction. | [Lesson 17.7](17-storage-and-databases/07-lsm-trees.md) |

### M

| Word | What it means | First used in |
|---|---|---|
| machine epsilon | The gap between 1.0 and the next larger float: about 1.19e-7 for f32 and 2.22e-16 for f64. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| #[macro_export] and $crate | #[macro_export] puts a macro at the crate root so other crates can use it; $crate inside refers to the defining crate. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| machine code | Instructions as numbers in the exact form the CPU understands. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| macro | Code that writes code. A macro call ends its name with ! (println!). | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| macro expansion | The step where the compiler replaces a macro call with the code the macro produces. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| magic bytes | A fixed sequence at the start of a file that says what type of file it is. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| magic number | A fixed signature at the start of a file that tells the OS its format. Also a bare number in code with no name (see DRY). | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| maintainability | How easy it is to read, change and fix a program months later, often by someone else. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |
| maintainer | A person who looks after a project: reviews and merges changes, makes releases, decides the direction. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| majority | More than half of the nodes of a cluster: 2 of 3, 3 of 5. Any two majorities share at least one node. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| making illegal states unrepresentable | Design habit: shape types so invalid combinations cannot be written (enums, private fields). | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| man-in-the-middle | An attack where someone sits between you and the server and pretends to be the server. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| man-in-the-middle attack | An attack in which someone between client and server reads or changes the traffic. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| mangled name | The encoded form of a function name that the compiler puts into the binary; tools show it in reports, and the readable part is near its end. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| mangled name / demangling | The encoded form of a Rust function name in a binary, and turning it back into a readable path. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| Manhattan distance | The distance on a grid when you can only move up, down, left or right: the difference of rows plus the difference of columns. | [Lesson 07.20](07-algorithms-and-data-structures/20-checkpoint.md) |
| manifest | A small file that lists which table files of a storage engine are valid; replacing it atomically changes the set of files in one step. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| MANIFEST (storage engine) | A small file that names the data files of the current state; replacing it atomically is the commit point of compaction. | [Lesson 17.14](17-storage-and-databases/14-project-key-value-store.md) |
| map (eBPF) | A data structure shared by eBPF programs and user-space programs. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| mapping (memory) | One continuous range of virtual addresses in use, with its permissions and an optional file behind it. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| Markdown | A way to write formatted text in a plain .md file with marks like # and -. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| marker type | An empty struct used only as a name for a state or a property. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| mask (bit mask) | An integer whose 1 bits mark the bits you want to work with; x & mask keeps only those bits. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| mask (filter) | An array of true/false values, one per row, that says which rows to keep. | [Lesson 22.16](22-specializations/16-databases-track-executing-queries.md) |
| master password | The one password from which the encryption key of the vault is derived. If it is forgotten, the data cannot be recovered. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| match | An expression comparing a value with patterns and running the first that fits. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| match guard | An extra if condition after a pattern in a match arm that must also hold for the arm to run. | [Lesson 03.8](03-ownership-and-borrowing/08-project-word-counter-and-checkpoint.md) |
| matches! | Macro returning true/false: does a value fit a pattern (with optional guard). | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| matrix | A way to run the same CI job with several values, for example on two operating systems. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| maxSurge / maxUnavailable | How many pods a rolling update may add above the desired count, and how many may be missing below it. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| mean / median / standard deviation | Average / middle value after sorting (robust to outliers) / spread around the mean. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| median | The middle value of a group after sorting it. | [Lesson 07.9](07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) |
| median (p50) | The 50th percentile: half of the values are smaller, half are larger. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| median-of-three | A pivot rule: use the median of the first, middle and last item. | [Lesson 07.9](07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) |
| member (of a workspace) | A package that belongs to a Cargo workspace. | [Lesson 08.12](08-software-design/12-project-layout-and-workspaces.md) |
| membership change | Adding a node to or removing a node from a running cluster without stopping it. | [Lesson 23.3](23-capstone-projects/03-capstone-distributed-key-value-store.md) |
| memoization | Remembering the results of a function (in a map) so they are not computed twice. | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| memory | The place where a running program keeps its data; a long row of numbered boxes (bytes). | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| memory address | The number of a byte in RAM, like a house number on a street. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| memory bandwidth | How many bytes per second can move between RAM and the CPU. | [Lesson 19.6](19-performance-and-profiling/06-data-layout-for-speed.md) |
| memory hierarchy | The layers of memory in a computer, from tiny and very fast (registers, caches) to huge and slow (RAM, disk). | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| memory leak | Heap memory that was allocated but never freed. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| memory map | The list of address ranges of a machine and what is behind each: RAM or a device. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| memory model | The rules of a language that say which values a load may return when threads share memory. Rust follows the C++ model. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| memory reclamation | Freeing memory that other threads may still be reading; the hard part of lock-free structures with pointers. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| memory region (mapping) | A continuous range of addresses with the same permissions and source, listed in /proc/<pid>/maps. | [Lesson 13.2](13-operating-systems/02-processes-from-the-inside.md) |
| memory safety | A program only reads and writes memory it owns and that is still valid. Safe Rust guarantees it. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| memory-mapped file | A file that the kernel makes visible as a range of memory in a process; reading the memory reads the file. | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| memory-mapped I/O | Controlling hardware by reading and writing fixed memory addresses. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| memtable | The sorted in-memory table that receives all new writes of an LSM tree before they are flushed to an SSTable. | [Lesson 17.7](17-storage-and-databases/07-lsm-trees.md) |
| merge | Combine the work of one branch into the current branch. | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| merge commit | A commit with two parents that records the joining of two lines of work. | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| merge conflict | Git cannot combine two changes because both branches changed the same lines (or one changed a file the other deleted). | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| merge join | A join method: sort both inputs by the join key, then walk through them side by side. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| merge patch | A partial update of an object: you send only the fields you want to change, as JSON. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| merge tool | A program with a visual view of a conflict (RustRover has one). | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| message | A piece of data that one node sends to another over the network; the only way nodes share information. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| message (protobuf) | A record type in a .proto file, similar to a Rust struct. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| message body | The bytes after the empty line of an HTTP message, such as a form or a web page. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| message broker | A program that receives messages from one service and delivers them to others, so services do not call each other directly. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| method | A function attached to a value with a dot (x.abs()). In an impl block, a function whose first parameter is self. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| method chaining | Calling several methods in a row, each on the previous result. | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| metric | A number measured over time, such as requests per second. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| metrics facade / recorder | The metrics crate gives macros such as counter! (the facade); a separately installed recorder decides where the values go, for example to a Prometheus page. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| microcontroller | A whole small computer on one chip: CPU, memory and pins to talk to the outside world. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| middleware | Code that runs around every request, before and after the handler: logging, timeouts, limits, authentication. | [Lesson 16.4](16-backend-services/04-errors-validation-and-middleware.md) |
| migration | One numbered SQL file that changes the database schema one step; migrations run in order, once per database. | [Lesson 16.7](16-backend-services/07-migrations-and-transactions.md) |
| milestone | A step of a project with a clear goal and a way to check it is done (M1, M2, ...). | [Lesson 01.10](01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) |
| millicore | One thousandth of a CPU core, written as m: 50m is 5% of one core. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| min-heap | A heap whose pop gives the smallest item; in Rust a BinaryHeap of Reverse values. | [Lesson 07.17](07-algorithms-and-data-structures/17-greedy-algorithms.md) |
| minimal failing input | The simplest input found that still makes the test fail. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| minimal reproducible example | The smallest complete program that still shows your problem. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| minimizing a crash (tmin) | Making a crashing input as small as possible so that it is easy to understand. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| minimum degree | The number t that fixes node size in a B-tree: every node except the root has between t-1 and 2t-1 keys. | [Lesson 17.6](17-storage-and-databases/06-b-trees.md) |
| minor / major page fault | A minor fault is fixed without the disk; a major fault needs to read data from the disk first. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| Miri | An interpreter for Rust MIR that runs a program step by step and reports undefined behaviour (cargo +nightly miri run/test). | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| misconfiguration | A wrong or unsafe setting: a default password, debug mode in production, a service open to the whole network. | [Lesson 10.3](10-security-fundamentals/03-the-owasp-top-10.md) |
| missed mutant | A mutant for which all tests still pass, which is bad: your tests do not notice this change. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| mitigation | An action that reduces a threat: a check, an encryption, a limit. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| mmap / munmap | The system calls that create and remove a memory mapping. | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| mnemonic | The short name of an instruction, such as add, ldr or ret. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| mock | A test double that also checks how it was called: which methods, which arguments, how many times. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| mock interview | A practice interview with a friend, a colleague or a study partner, under real time limits. | [Lesson 24.9](24-career-and-open-source/09-checkpoint-course-complete.md) |
| model (test oracle) | A second, very simple and obviously correct implementation used only to check the real one. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| model checker | A tool that tries every possible schedule (or every one inside a limit) and checks your assertions in each. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| model checking | A technique in which a tool explores every possible order of events of a small model of a system to find any broken property. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| model-based test | A test that runs random operations on your code and on a simple, obviously correct model, and checks that both always agree. | [Lesson 17.14](17-storage-and-databases/14-project-key-value-store.md) |
| modular arithmetic | Arithmetic on a clock: after the largest value the count starts again from zero. Unsigned integers with n bits work modulo 2 to the power n. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| module | A named container of code, mod name { }; a namespace and privacy boundary inside a crate. | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| module (OpenTofu) | A folder of .tf files with its own variables and outputs that you can use several times with different inputs, like a function. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| module tree | All modules of a crate as a tree rooted at crate. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| monolithic kernel | A kernel where the scheduler, memory manager, file systems and most drivers live in one big program, like Linux. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| monomorphization | The compiler makes a specialised copy of generic code for every concrete type used. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| monotone predicate | A predicate that is true for a prefix of a sequence and false for the rest, so it changes only once. | [Lesson 07.10](07-algorithms-and-data-structures/10-binary-search.md) |
| monotonic clock | A clock that only moves forward at a steady rate (Instant in Rust); good for durations, meaningless on another machine. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| monotonic reads | A session guarantee: once a client has seen a value, it never sees an older one. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| Morse code | A way to send letters as short (dot) and long (dash) signals with fixed timing rules. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| most / least significant bit | The bit with the largest place value (leftmost) / the smallest place value (rightmost, worth 1). Bits are numbered from 0 on the right. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| mount | Making a folder or a disk visible at a place inside another file system; Lima mounts your Mac home folder into the VM. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| mount / mount point | Attaching a file system to a directory (the mount point) so that its files appear there. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| mount namespace | A namespace with its own copy of the mount table. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| mount point | A directory where a disk or another file system is attached to the tree. | [Lesson 11.2](11-linux-shell-and-containers/02-the-linux-file-system.md) |
| mount propagation | The rule that decides whether a mount in one mount namespace also appears in others; private means it does not. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| move | Transferring ownership to a new owner. Only the small handle is copied and the old variable becomes unusable. | [Lesson 03.2](03-ownership-and-borrowing/02-ownership-rules.md) |
| move closure | A closure written move \|…\| … that takes ownership of everything it captures. | [Lesson 05.4](05-rust-intermediate/04-closures.md) |
| msync / flush | Asking the kernel to write the changed pages of a mapping to the disk now. | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| multi-stage build | A Dockerfile with several FROM lines: early stages build the program, the last stage is the small image that is shipped. | [Lesson 11.13](11-linux-shell-and-containers/13-dockerfiles-for-rust.md) |
| multi-thread runtime | Tokio's default runtime: a pool of worker threads that share the tasks and steal work from each other. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| must-have / nice-to-have | Lines in a job description that the team really needs / lines that would be a bonus. | [Lesson 24.1](24-career-and-open-source/01-roles-for-systems-engineers.md) |
| #[must_use] | An attribute that makes the compiler warn when a caller ignores the result. | [Lesson 08.11](08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) |
| mutable (mut) | Can be changed; requested with the keyword mut. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| mutable reference | A reference &mut T that may change the value. At most one can exist at a time (also called exclusive). | [Lesson 03.5](03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| mutant | One changed copy of your program, for example with + replaced by -. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| mutation (fuzzing) | A small random change of a stored input, such as flipping a bit or inserting a byte. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| mutation testing | Changing your code on purpose in small ways and checking that at least one test fails. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| mutex | A lock allowing one thread at a time into the data. MutexGuard unlocks on drop; the mutex is poisoned if a thread panics while holding it. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |

### N

| Word | What it means | First used in |
|---|---|---|
| Nagle's algorithm | A TCP feature that holds back small writes to send them together; set_nodelay(true) turns it off. | [Lesson 15.4](15-networking/04-tcp-sockets-in-rust.md) |
| name compression | A DNS trick: a name can end with a 2-byte pointer to an earlier position in the same message instead of repeating its labels. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| name mangling | Encoding the full path of a Rust function into one unique symbol name; #[unsafe(no_mangle)] turns it off. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| named argument | An argument given as name: value at the call; Rust has none, builder methods read like them. | [Lesson 08.6](08-software-design/06-builder-pattern.md) |
| namespace | A named area in which names cannot clash with names elsewhere. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| namespace (Kubernetes) | A named group of objects inside one cluster, such as default or kube-system; not the same as a Linux namespace. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| namespace (Linux) | A kernel feature that gives a process its own view of one kind of resource. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| native / generic XDP | XDP run by the network driver (fast) / XDP run later by the kernel for any interface. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| NATS | A fast, simple, open-source message broker that routes messages by subject. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| neighbour | A vertex that can be reached from another vertex by one edge. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| NEON | The SIMD instruction set of arm64 CPUs, including Apple M-series chips. It is always present on 64-bit Arm. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| nested loop join | A join method: for every row of one input, look through the other input for matching rows. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| nested repository | A repository inside another repository's folder. Almost always a mistake. | [Lesson 01.10](01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) |
| network | Computers connected so that they can send data to each other; the internet is a network of networks. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| network address | The first address of a subnet, with all host bits zero. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| network byte order | Big-endian order (most significant byte first), used for numbers in network headers. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| network interface | A network connection of the computer, such as eth0 or the loopback lo. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| network namespace | A namespace with its own network interfaces, addresses and routes. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| network partition | A cut in the network so that some nodes cannot reach others, although all of them are still running. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| never type (!) | The type of expressions that never produce a value (panic!, continue, exit). It converts to any type. | [Lesson 06.2](06-rust-advanced/02-advanced-traits-and-types.md) |
| newtype | A one-field tuple struct that gives an existing type a new meaning or lets you implement traits for a foreign type, e.g. struct Meters(f64). | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| newtype pattern | A tuple struct with one field that wraps another type to give it a new meaning or new trait implementations. | [Lesson 06.2](06-rust-advanced/02-advanced-traits-and-types.md) |
| next_index / match_index | A Raft leader's bookkeeping per follower: the next entry to send, and the highest entry known to be the same on that follower. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| nibble | Half a byte: 4 bits, which is one hexadecimal digit. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| nice value | A number from -20 to 19 that sets a task's weight; higher means a smaller CPU share. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| niche | A bit pattern a type never uses (such as 0 for a reference); the compiler can store an enum tag there, so Option<&T> is as small as &T. | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| niche optimization | Rust's trick of storing None as a value the type can never have, so Option<Box<T>> needs no extra space. | [Lesson 07.4](07-algorithms-and-data-structures/04-linked-lists.md) |
| nightly | The Rust toolchain built every night; it has unstable features some tracks need. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| nines | A short way to write availability: three nines is 99.9 %, four nines is 99.99 %. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| nit | A tiny optional comment in a review, for example about naming. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| --no-ff | An option that forces a merge commit even when a fast-forward is possible. | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| no-op entry | A log entry with no command that a new Raft leader appends so it can commit entries from earlier terms. | [Lesson 18.11](18-distributed-systems/11-raft-log-replication.md) |
| no_std | A crate attribute that removes the standard library; the crate uses only core (and maybe alloc). | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| #![no_std] | A crate attribute that links only core, not std. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| node | One element of a linked structure: a value plus links to other nodes, usually a separate heap allocation. | [Lesson 07.4](07-algorithms-and-data-structures/04-linked-lists.md) |
| node (Kubernetes) | One machine, real or virtual, in a Kubernetes cluster that runs containers. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| node agent | A program that runs once on every node and works for all pods there, for example a log collector. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| noise (measurement) | Random variation in a measurement that is not caused by your code, such as other programs or the heat of the chip. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| noise threshold | The size of a change (2% by default in criterion) below which a difference is treated as noise. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| non-blocking mode | The O_NONBLOCK flag: calls that would have to wait return an error at once instead. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| non-determinism | Anything that can differ between two runs of the same program: thread scheduling, real time, random numbers, packet order. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| non-fast-forward | The remote has commits you lack, so your push is rejected. Fetch and integrate first. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| non-functional requirement | How well a system must do its job: speed, availability, durability, cost. | [Lesson 24.7](24-career-and-open-source/07-coding-and-system-design-interviews.md) |
| non-goal | Something you decide, in writing, not to build. It protects the scope. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| non-repeatable read | Reading the same row twice in one transaction and getting different values because another transaction committed in between. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| non-root user | A normal user without administrator rights; running containers as non-root limits the damage of a break-in. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| #[non_exhaustive] | An attribute telling other crates that variants or fields may be added later, so they must handle the unknown. | [Lesson 08.11](08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) |
| nonce | A value that must be unique for every message encrypted with the same key. It is not secret and is stored next to the ciphertext. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| normalisation (Unicode) | Converting text into one standard form (such as NFC) so that texts that look the same also compare equal. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| notify (notify_one, notify_all) | Wake up one thread, or all threads, that wait on a condition variable. | [Lesson 14.3](14-concurrency-and-async/03-condition-variables.md) |
| nsenter | A command that runs a program inside the namespaces of another process, with the setns system call; docker exec works the same way. | [Lesson 13.18](13-operating-systems/18-checkpoint.md) |
| NUL byte | The byte with the value 0. Normal text almost never contains it; binary files often do. | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| null | A "no value" marker in other languages. Rust has none. | [Lesson 04.4](04-structs-enums-collections-errors/04-option-and-match.md) |
| NULL (SQL) | SQL's marker for a missing or unknown value; in a CSV file, an empty field. | [Lesson 22.17](22-specializations/17-databases-track-project-and-next-steps.md) |
| null ordering | The rule that says whether NULL values come first or last when sorting. | [Lesson 22.17](22-specializations/17-databases-track-project-and-next-steps.md) |
| null pointer | A pointer that points nowhere (address 0). C uses it to mean "nothing". | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| NXDOMAIN | The DNS response code 3: the asked name does not exist. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |

### O

| Word | What it means | First used in |
|---|---|---|
| O_DIRECT | A Linux flag for open that skips the page cache, so data moves directly between the disk and the program's buffer. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| object file | Compiled machine code and data of one piece of a program, with a symbol table of defined and still-needed names; not runnable before linking. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| object file (eBPF) | The compiled eBPF program: an ELF file with code sections, map definitions and a license. | [Lesson 22.7](22-specializations/07-ebpf-track-a-first-program-with-aya.md) |
| observability | How well you can understand what a running system is doing from the data it sends out. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| observability pipeline | A program that collects logs, metrics and traces, changes them on the way and sends them to storage; often an agent on every machine. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| observable behaviour | Everything that callers can see: return values, output, errors and side effects. | [Lesson 08.13](08-software-design/13-refactoring-and-code-review.md) |
| obvious implementation | Writing the real code at once when you are sure of it, instead of faking it first. | [Lesson 09.6](09-testing-debugging-benchmarking/06-tdd-in-practice.md) |
| octal | Base 8, with digits 0–7. One octal digit is 3 bits; Unix file permissions such as 755 are written in octal. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| octal mode | The permission bits written as three digits such as 755 or 640 (read 4, write 2, execute 1). | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| octet | One byte of an IPv4 address; 192.168.1.37 has four octets. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| offline mode (sqlx) | Building with saved query descriptions in the .sqlx folder instead of a running database, enabled with SQLX_OFFLINE=true. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| offset | The position in a file where the next read or write starts. | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| offset (of a field) | The distance in bytes from the start of a struct to one of its fields; std::mem::offset_of! returns it. | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| omission | A failure in which a message is sent but never delivered. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| on-call | The duty of being reachable and ready to respond to pages during a shift. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| oneshot | A tower helper that sends exactly one request to a service and returns the response; used to test a router in memory. | [Lesson 16.3](16-backend-services/03-web-services-with-axum.md) |
| oneshot channel | A channel for exactly one value from one sender to one receiver; often used for a reply. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| OOM kill | Killing a process because its cgroup reached the memory limit and the kernel could not free enough memory. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| OOM killer | The part of the Linux kernel that kills a process when memory runs out. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| open addressing | Collision strategy where all entries live in the table itself and a collision sends you to another slot. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| open file description | The kernel record created by one open() call; it holds the offset and flags and can be shared by several descriptors. | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| open model | A load generator that sends requests at a fixed rate, whether or not earlier answers have arrived. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| open question | A part of a design that is not decided yet and is written down honestly as such. | [Lesson 23.2](23-capstone-projects/02-writing-a-design-document.md) |
| open redirect | A service that redirects to any address given to it; attackers use it to make phishing links look trustworthy. | [Lesson 16.15](16-backend-services/15-project-url-shortener-service.md) |
| open source | Software whose source code is published under a licence that lets anyone use, study, change and share it. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| OpenAPI | A standard machine-readable format for describing REST APIs, from which tools generate documentation and client code. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| OpenAPI schema | A standard description of fields and their types; CRDs use its v3 form so the API server can validate custom resources. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| OpenTelemetry Collector | A separate program that receives telemetry, processes it (receivers, processors, exporters, pipelines) and passes it on to back ends. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| OpenTofu | An open-source infrastructure-as-code tool run by the Linux Foundation; its command is tofu. It began as a fork of Terraform and reads the same kind of files. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| ? operator | Gives the value on Ok; on Err returns Err(e) from the current function (converting with From). | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| off-by-one error | A bug where a loop or index is one too many or too few. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| OnceCell / OnceLock | A box filled exactly once, then read-only. OnceLock also works between threads and in statics. | [Lesson 06.4](06-rust-advanced/04-interior-mutability.md) |
| operand | A value an operator works on. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| operator | A symbol that calculates or compares: +, ==, &&. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| operator (Kubernetes) | A program in a cluster that manages one kind of application automatically. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| operator overloading | Making +, \*, == etc. work for your own types by implementing std::ops traits. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| operator precedence | The rule that decides which operator binds stronger, for example comparisons before AND. | [Lesson 22.15](22-specializations/15-databases-track-parsing-sql.md) |
| optimal substructure | The best answer for a problem is built from the best answers of its subproblems. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| optimiser | The part of the compiler that rewrites the program into a faster program with the same behaviour. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| optimization level | The opt-level setting: how hard the compiler works to make code fast (0 to 3) or small ("s", "z"). | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| optimizer | The stage that rewrites a query plan into a cheaper one with the same result. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| Option | A type that is either Some(value) or None. Used when a value may be missing. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| option (command line) | An argument that starts with a dash and changes how a command works, such as -k 3. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| option (flag) | A switch that changes how a command works, written with a dash. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| option / flag / subcommand | CLI parts: named option (--count 3), value-less flag (--shout), command inside a command (git commit). | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| `Option<T>` | Enum Some(T) / None for a value that may be missing. Replaces null. | [Lesson 04.4](04-structs-enums-collections-errors/04-option-and-match.md) |
| optional dependency | A dependency compiled only when a feature of your package switches it on (dep:name). | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| optional parameter | A setting with a sensible default that the caller may leave out; Rust has none for functions, so builders and Default fill the gap. | [Lesson 08.6](08-software-design/06-builder-pattern.md) |
| or-pattern | A pattern with \| that matches any of several alternatives. | [Lesson 06.3](06-rust-advanced/03-patterns-in-depth.md) |
| oracle | A trusted, simpler or slower way to get the right answer, used to compare against, for example the standard library's sort. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| oracle (testing) | A trusted way to compute the right answer, such as a slow simple version of a function, used to judge the fast version. | [Lesson 09.14](09-testing-debugging-benchmarking/14-checkpoint.md) |
| orchestration / choreography | Two saga styles: one component directs every step, or each service reacts to events from the others. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| orchestrator | A program that starts, watches and restarts containers, such as Docker Compose or Kubernetes. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| Ord | The standard trait for types with a total order, where any two values can be compared. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| order of magnitude | A rough size given as a power of ten. | [Lesson 23.2](23-capstone-projects/02-writing-a-design-document.md) |
| Ordering | Standard enum Less / Equal / Greater, the result of cmp. | [Lesson 02.10](02-programming-basics/10-project-guessing-game.md) |
| Ordering (atomics) | The argument of every atomic method that says which other memory operations are ordered around it: Relaxed, Acquire, Release, AcqRel, SeqCst. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| origin | The default name of the remote you cloned from or first added. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| orphan | A process whose parent has ended; the kernel gives it a new parent, normally PID 1. | [Lesson 13.5](13-operating-systems/05-fork-exec-and-wait.md) |
| orphan process | A process whose parent has ended; PID 1 adopts it. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| orphan rule | A trait may be implemented for a type only if the trait or the type belongs to your crate. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| OS crash | The kernel stops or the power goes off, so everything in RAM, including the page cache, is lost. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| ours / theirs | In a merge: the version from your current branch / from the branch being merged in (swapped in a rebase). | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| out-of-order execution | The CPU runs instructions as soon as their inputs are ready, not strictly in program order, and then makes the results appear in order. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| OUT_DIR | A folder Cargo gives each build script for generated files. | [Lesson 22.7](22-specializations/07-ebpf-track-a-first-program-with-aya.md) |
| outbox | A list inside a node where it puts the messages it wants to send; the caller takes them out and delivers them. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| outlier | One value far away from the others; in a service, a real request that someone waited for. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| outlier detection | Removing a backend for a while after it fails several real requests in a row. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| outlives bound | 'long: 'short means 'long lasts at least as long as 'short. | [Lesson 06.5](06-rust-advanced/05-lifetimes-in-depth.md) |
| overcommit | Linux lets programs request more memory than exists, because most of it is never touched. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| overflow | A result that does not fit the type. Panics in debug builds, wraps in release builds. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| overflow check | Code the compiler adds to detect integer overflow and panic. It is on in debug builds and off in release builds unless overflow-checks = true. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| overlapping subproblems | The same subproblem is needed again and again by different parts of the solution; this makes a plain recursion slow and DP useful. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| overlay (Kustomize) | A folder for one environment that points to the base and lists the changes for it. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| overload | More work arrives than the system can finish in the same time. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| overlong encoding | A forbidden UTF-8 form that uses more bytes than needed for a character; valid UTF-8 rejects it. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| oversubscription | More runnable threads than CPU cores, so some threads always wait for a core. | [Lesson 14.7](14-concurrency-and-async/07-building-a-spin-lock.md) |
| OwnedFd / BorrowedFd / RawFd | Rust types for descriptors: owned and closed on drop / borrowed for a lifetime / a plain i32 number. | [Lesson 13.4](13-operating-systems/04-files-and-file-descriptors.md) |
| owner | The variable responsible for a value; only the owner may free it. | [Lesson 03.2](03-ownership-and-borrowing/02-ownership-rules.md) |
| owner reference | A metadata field of a child object that names its owner; when the owner is deleted, Kubernetes deletes the child. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| ownership | The rules saying who owns each value and when it is freed: one owner per value, one at a time, drop when the owner leaves scope. | [Lesson 03.2](03-ownership-and-borrowing/02-ownership-rules.md) |

### P

| Word | What it means | First used in |
|---|---|---|
| p-value | A number from statistics; below 0.05 it means a difference is unlikely to be luck, but it does not say the difference is big. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| package | A folder with Cargo.toml and source code that Cargo can build; holds one or more crates (at most one library). | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| package index | The list of packages and versions of a repository that apt downloads with apt update. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| package manager | A program that downloads, installs and updates other programs. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| packet | A small piece of data with a header that says where it goes. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| packet capture | Recording the packets that pass through a network interface, for example with tcpdump. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| padding | Unused bytes the compiler adds inside a struct for alignment. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| page | A fixed-size block of virtual memory, usually 4 KiB on Linux. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| page cache | RAM in which the kernel keeps recently used file data; read and mmap both use it. | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| page fault | A CPU exception when a page is not mapped or not allowed; the kernel then maps a frame or kills the program. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| page table | A per-process table kept by the kernel that maps pages to frames and stores their permissions. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| pager | A program (less) that shows long output one screen at a time; q quits. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| pagination | Returning a long list in pages, for example 50 items at a time, with limit/offset or a cursor. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| panic | A controlled crash at run time: the program stops itself with a message (exit code 101). | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| panic handler | The function marked #[panic_handler] that runs when code panics; required without std. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| panic hook | A function that Rust calls on every panic before unwinding starts; set it with std::panic::set_hook. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| panic payload | The value a panic carries, usually its message as &str or String. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| panic strategy | What a panic does: unwind (go back up the stack and run Drop code) or abort (end the process at once). | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| parallel iterator | A rayon iterator (par_iter, into_par_iter) whose items are processed by several threads; collect keeps the original order. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| parallelism | Several jobs actually running at the same instant on different CPU cores. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| parameter | A variable in a function definition that receives a value. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| parameterized query | A SQL command with placeholders for the values; the values are sent separately and never read as SQL. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| park / unpark | thread::park puts the current thread to sleep; unpark from another thread wakes it. | [Lesson 14.10](14-concurrency-and-async/10-how-futures-work.md) |
| parse | Turn text into a value of a type, e.g. "42" into 42. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| parser | Code that turns text into structured data, e.g. a line into a Command enum. | [Lesson 04.11](04-structs-enums-collections-errors/11-project-phone-book.md) |
| parser combinator | A small parser that is combined with others into bigger parsers; nom is a library of them. | [Lesson 22.19](22-specializations/19-networking-track-zero-copy-parsing.md) |
| partial clone | A Git clone that downloads the history but fetches file contents only when needed (git clone --filter=blob:none). | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| partial failure | A situation where some parts of a system have failed while other parts still work. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| partial index | An index that contains only the rows matching a condition, such as WHERE NOT done. | [Lesson 16.7](16-backend-services/07-migrations-and-transactions.md) |
| partial input | Bytes that are a correct beginning of a message but not all of it yet; the reader must read more instead of failing. | [Lesson 22.19](22-specializations/19-networking-track-zero-copy-parsing.md) |
| partial move | Moving a part of a value (field/payload) out, leaving the whole unusable (E0382). | [Lesson 04.4](04-structs-enums-collections-errors/04-option-and-match.md) |
| partial order | An order in which some pairs of items cannot be compared, like concurrent events. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| PartialEq | Trait that makes == work for a type (needed with Debug for assert_eq!). | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| PartialOrd | Standard trait for types that can be ordered with &lt;, &gt;, &lt;=, &gt;=. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| partition | Rearranging a slice around a pivot so that it falls into groups of smaller, equal and bigger items. | [Lesson 07.9](07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) |
| partition (broker) | One ordered, append-only log inside a topic; order is kept only within a partition. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| partition key | The value that decides which partition a record belongs to, for example a user id. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| partitioning | Splitting data into partitions so that each node stores and serves only part of it; also called sharding. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| pass | One full walk over a slice, or a part of it, inside an algorithm. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| passive health checking | Learning that a backend is down from failed real requests instead of separate test connections. | [Lesson 22.21](22-specializations/21-networking-track-project-and-next-steps.md) |
| passphrase | A password that protects a private key file. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| path | The address of a file or directory. Absolute paths start at /; relative paths start from the current directory. In a URL, the part after the host. In Rust, the address of an item in the module tree. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| path dependency | A dependency that lives in a folder on your computer. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| path parameter | A part of a URL path that changes from request to request, such as {id} in /tasks/{id}. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| path traversal | An attack where a path like /static/../secret tries to leave the allowed folder; stopped by checking canonical paths. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| pattern | A shape a value is compared with, e.g. 0, 2 \| 3, 4..=9, Ok(n), Some(x). It can also name parts of the value. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| paused clock | In Tokio tests with start_paused = true, time stands still and jumps to the next timer when all tasks wait, so sleeps take no real time. | [Lesson 14.14](14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| Paxos | An older consensus algorithm by Leslie Lamport, correct but hard to understand and implement. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| payload | The data a variant carries. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| peak factor | How many times the busiest period exceeds the average load; used in estimates. | [Lesson 23.2](23-capstone-projects/02-writing-a-design-document.md) |
| peak memory | The largest amount of heap memory a program holds at one moment. | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| peek | Look at the next item to be removed without removing it. | [Lesson 07.5](07-algorithms-and-data-structures/05-stacks-and-queues.md) |
| peer | The program on the other side of a conversation; with UDP every datagram can come from a different peer. | [Lesson 15.5](15-networking/05-udp-sockets-in-rust.md) |
| peer review | A review of your work by another person who did not build it. | [Lesson 23.10](23-capstone-projects/10-checkpoint-capstone-review.md) |
| pending signal | A signal that was sent but not yet delivered. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| per-CPU map | A map in which every CPU has its own copy of each value; user space adds the copies. | [Lesson 22.8](22-specializations/08-ebpf-track-maps-and-tracing.md) |
| percentile | The value below which a given share of measurements falls; p99 is the value that 99 % of measurements do not exceed. | [Lesson 18.2](18-distributed-systems/02-failures-and-timeouts.md) |
| perf | The profiler that comes with the Linux kernel; it counts events and samples call stacks, including kernel code. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| performance core / efficiency core | The two kinds of cores in Apple M-series chips: fast cores and slower cores that use less power. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| performance regression | A change that makes a program slower. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| peripheral | A hardware block inside a chip that does one job: pins, timers, serial ports, converters. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| permanent failure | A failure that happens again every time, such as a wrong password; a retry never helps. | [Lesson 18.3](18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| permission | A flag on a file saying who may read, change or run it. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| permission bits | Nine switches on every file: read, write and execute for the owner, the group and everybody else. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| permission fault | A page fault because the address is mapped but the access (for example a write) is not allowed. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| persistent connection (keep-alive) | One TCP connection used for several HTTP requests one after another; the default in HTTP/1.1. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| PersistentVolume | A piece of storage in the cluster that lives independently of pods; a PVC claims it. | [Lesson 21.13](21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) |
| pets and cattle | Two ways to treat servers: a pet is unique and repaired by hand, cattle are identical and replaced when one breaks. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| phantom read | Running the same WHERE query twice in one transaction and getting a different set of rows. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| PhantomData | A zero-sized standard type that ties a struct to a type parameter without storing data of that type. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| PhantomPinned | A marker type with no data; a struct that contains it is not Unpin. | [Lesson 14.12](14-concurrency-and-async/12-pin-and-self-referential-futures.md) |
| phase | A part of the course about one topic, numbered 00 to 24. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| phase (status) | A short word in an object's status that sums up its state, for example Pending or Ready. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| physical address | The real location of a byte in the RAM chips. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| physical plan | A tree of concrete algorithms that describes how a query is computed. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| pickaxe | Searching the history for commits where the number of occurrences of a text changed (git log -S). | [Lesson 01.4](01-git-and-version-control/04-viewing-history-and-changes.md) |
| PID 1 | The first process that the kernel starts and the ancestor of all others; on Ubuntu it is systemd. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| PID namespace | A namespace with its own process numbering; its first process is PID 1. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| Pin | A wrapper around a pointer that promises the pointed-to value will not be moved again until it is dropped. | [Lesson 14.12](14-concurrency-and-async/12-pin-and-self-referential-futures.md) |
| pin projection | Getting a pinned reference to one field from a pinned reference to the whole struct. | [Lesson 14.12](14-concurrency-and-async/12-pin-and-self-referential-futures.md) |
| ping | A short, polite reminder in a discussion that has been quiet for a while. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| Pingora | A Rust framework from Cloudflare for building proxies and network services. | [Lesson 22.20](22-specializations/20-networking-track-production-proxies.md) |
| pinning | Putting a value behind Pin so that it stays at one memory address until it is dropped. | [Lesson 14.12](14-concurrency-and-async/12-pin-and-self-referential-futures.md) |
| pipe | The symbol \| that sends the output of one command into the input of the next. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| pipe buffer | The kernel memory holding bytes written to a pipe but not yet read; 64 KiB by default on Linux. | [Lesson 13.6](13-operating-systems/06-pipes-and-redirection-inside.md) |
| pipeline | The whole automatic process that runs for one change: all the jobs in order. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| pipeline (CPU) | Splitting the work on an instruction into stages so that several instructions are in different stages at the same time. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| pipeline (delivery) | The ordered stages a change passes through from commit to production. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| pipeline (Redis) | Several commands sent to Redis together; with MULTI/EXEC they run atomically. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| pipeline breaker | An operator, such as sort or aggregate, that must read all its input before producing output. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| pipelining | Sending the next request before the answer to the previous one has arrived; answers come back in order. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| pitch | One sentence that says what you build and for whom. | [Lesson 23.7](23-capstone-projects/07-capstone-your-own-idea.md) |
| pivot | The item quicksort uses as a dividing line: smaller items go to its left, bigger ones to its right. | [Lesson 07.9](07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) |
| pivot_root | A system call that swaps the root mount of a mount namespace: a new directory becomes /, and the old root can be unmounted. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| plaintext | The readable data before encryption. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| plan / apply | A plan is the preview of the changes OpenTofu would make; apply makes them for real after you confirm. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| planner statistics | Facts about each column (distinct values, most common values, histogram) that ANALYZE collects from a sample and the planner uses for estimates. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| pod | The smallest unit Kubernetes runs: one or more containers that share a network address and can share volumes. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| pod template annotation | A note in spec.template.metadata.annotations of a Deployment; changing it changes the template and starts a rollout. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| pointer | A value that contains an address and points to other data. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| pointer (index) | In this phase, a plain number (a usize index) that says at which position of the data you are; not a memory address and not a Rust reference. | [Lesson 07.15](07-algorithms-and-data-structures/15-two-pointers-and-sliding-window.md) |
| pointer authentication | An arm64 feature that signs return addresses and pointers so that a changed value is detected before use. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| pointer chasing | Following a link to a node that may be anywhere in memory; it is slow because the processor cannot predict where to look next. | [Lesson 07.4](07-algorithms-and-data-structures/04-linked-lists.md) |
| pointer loop | A broken or malicious DNS compression pointer that points to itself or forward, so a naive parser follows it forever. | [Lesson 15.11](15-networking/11-project-dns-resolver.md) |
| pointer provenance | The information about which allocation a pointer belongs to; pointers must be derived from that allocation, not rebuilt from plain numbers. | [Lesson 12.14](12-computer-systems/14-project-your-own-allocator.md) |
| poisoned lock | A Mutex whose holder panicked while holding it; later lock() calls return Err(PoisonError) because the data may be half-changed. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| poll | Asking a future if it can progress; it answers Poll::Ready(value) or Poll::Pending. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| Pomodoro technique | Working in 25-minute blocks with 5-minute breaks. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| pool (upstream) | A group of equal backends among which a proxy balances requests. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| population count | The number of 1 bits in an integer; count_ones computes it, often with one CPU instruction. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| port | A number 0-65535 that picks one program on a computer (the server uses 7878). | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| port binding | The program opens a network port by itself and listens on it, instead of being plugged into another server. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| port forwarding | Carrying the traffic of another program through an SSH connection, from a port on one computer to a port on the other. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| port-forward | A kubectl feature that opens a local port and tunnels it through the API server to a pod or Service; a debugging tool. | [Lesson 21.3](21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) |
| portfolio | A collection of your work you can show to others: repository with projects, READMEs and tests. | [Lesson 06.12](06-rust-advanced/12-checkpoint-stage-1-complete.md) |
| positional number system | A way to write numbers where the value of a digit depends on its place; each place is worth base times the place to its right. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| positional parameters | $1, $2 and so on: the arguments given to a script; $# is their count and $@ is all of them. | [Lesson 11.8](11-linux-shell-and-containers/08-bash-scripting-basics.md) |
| post-order traversal | Visit the left subtree, then the right subtree, then the node. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| PostgreSQL | A popular open-source database server; its client program is psql. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| power of two | A number with exactly one 1 bit, such as 1, 2, 4, 8 or 4096. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| power of two choices | A balancing method: pick two backends at random and send the request to the less busy one. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| Pratt parsing | An expression-parsing method in which each operator has a binding power and one loop handles precedence and associativity. | [Lesson 22.22](22-specializations/22-compilers-track-lexing-and-parsing.md) |
| pre-order traversal | Visit the node first, then the left subtree, then the right subtree. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| Pre-Vote | An extra Raft phase in which a node asks whether others would vote for it before it raises its term. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| precedence | Rules for which operator is evaluated first. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| precise capturing | A bound on impl Trait return types listing which lifetimes are captured. In edition 2024 all in-scope lifetimes are captured by default. | [Lesson 06.5](06-rust-advanced/05-lifetimes-in-depth.md) |
| precomputation | Work done once, before the questions come, so that every question becomes cheap. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| predicate | A function that answers a yes/no question and returns bool (is_even). | [Lesson 02.4](02-programming-basics/04-functions.md) |
| predicate pushdown | An optimization that moves filters down to or into the scan so fewer rows flow through the plan. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| preemption | The kernel taking the CPU away from a running task with the help of the timer interrupt. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| preemption (loom) | A point where loom lets another thread run before the current one reaches a natural stop; LOOM_MAX_PREEMPTIONS limits how many there may be. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| preemption bound | A limit on how many preemptions loom tries in one schedule; a smaller bound means fewer iterations. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| prefetching | The CPU noticing a regular access pattern and loading the next cache lines before the program asks for them. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| prefix (network) | How many bits at the start of an address name the network, for example /24. | [Lesson 11.11](11-linux-shell-and-containers/11-networking-commands.md) |
| prefix sums | A vector whose item k is the sum of the first k items of the input; it answers range-sum questions with one subtraction. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| prelude | The small set of names every Rust program can use without a use line. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| primary key | The column that identifies each row of a table; it is unique and never NULL. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| primitive obsession | Using plain types such as String and u32 for everything, so the compiler cannot tell different meanings apart. | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| print debugging | Printing intermediate values to find where a result goes wrong. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| priority queue | A collection where you add items and always take out the item with the highest priority first. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| privacy | Rule of which code may see which item: private items are visible to their module and its descendants. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| private address | An address from 10.0.0.0/8, 172.16.0.0/12 or 192.168.0.0/16, which is not used on the public internet. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| private key | The secret half of a key pair that proves ownership of a certificate; it must never leave its owner. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| private mapping | A copy-on-write mapping of a file; changes stay in the process and never reach the file (MAP_PRIVATE). | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| privilege level | A CPU mode that decides which instructions and which memory are allowed (EL0/EL1 on arm64, ring 3/ring 0 on x86-64). | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| privileged container | A container started with --privileged, with almost all the rights of root on its host; only for local experiments. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| probe | A regular check that the kubelet runs against a container: an HTTP request, a TCP connection or a command. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| probe read | Copying memory with a bpf_probe_read helper, which fails safely on a bad address. | [Lesson 22.8](22-specializations/08-ebpf-track-maps-and-tracing.md) |
| probing | Looking at the next candidate slot in open addressing; linear probing tries the slots one after another. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| proc-macro2 | Crate with its own token types that work outside the compiler, so macro logic can be unit-tested. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| procedural macro | A macro written as a Rust function that receives and returns a TokenStream at compile time. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| process | A program that is currently running, with its own memory. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| process (in a data-flow diagram) | A part of your system that runs code and handles data. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| process crash | Only one program dies (abort, kill -9, a bug); the kernel and its page cache keep running. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| process descriptor (task_struct) | The record the kernel keeps for each process or thread: PID, state, memory map, open files and more. | [Lesson 13.2](13-operating-systems/02-processes-from-the-inside.md) |
| process group | A set of processes, usually one shell job, that receives terminal signals together. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| process state | What a process is doing: running (R), sleeping (S), waiting for a device (D), stopped (T) or zombie (Z). | [Lesson 13.2](13-operating-systems/02-processes-from-the-inside.md) |
| process tree | Every process except the first has a parent, so processes form a tree. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| producer / consumer | A thread that puts work items into a queue / a thread that takes them out and handles them. | [Lesson 14.3](14-concurrency-and-async/03-condition-variables.md) |
| profile (Cargo) | A named set of compiler settings in Cargo.toml, such as dev and release; you can define your own with [profile.name]. | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| profile (debug / release) | A set of compiler settings: debug builds fast, release is optimised and runs faster. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| profile page | Your personal page on GitHub or GitLab, which can show a README from a special repository. | [Lesson 24.2](24-career-and-open-source/02-building-your-portfolio.md) |
| profiler | A tool that shows where a program spends its time (taught in Phase 19). | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| program | A list of instructions telling a computer what to do. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| program counter | The CPU register that remembers which instruction comes next. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| programming language | A strict small language in which people write programs. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| project | A lesson where you build a program yourself, with a contract but no ready solution. Also a Cargo package folder. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| projection | Keeping or computing only some columns of a table. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| projection pushdown | An optimization that makes the scan read only the columns the query uses. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| prologue / epilogue | Instructions at the start of a function that save registers and reserve stack space, and at the end that restore them. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| Prometheus | An open-source monitoring system that regularly reads metrics from programs, stores them as time series, answers queries and can raise alerts. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| promotion | Moving the same artifact from one environment to the next without rebuilding it. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| prompt | The text at the start of the line showing the shell waits for a command. Also a mark like "&gt; " printed by a program that waits for input. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| promtool | The command-line tool of Prometheus that checks configuration and rule files and runs unit tests for rules. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| proof harness | A function marked #[kani::proof] that sets up inputs and states what must be true; Kani checks it for all inputs. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| propagate an error | To pass an error up to the caller instead of handling it where it happened. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| property | A statement that must be true for every valid input, for example that the shares always add up to the total. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| property (testing) | A statement that must be true for all inputs of some kind; a property test checks it on many random inputs. | [Lesson 11.16](11-linux-shell-and-containers/16-checkpoint-stage-2-complete.md) |
| property-based / snapshot / mutation testing | Advanced test styles (proptest, insta, cargo-mutants), named in Lesson 05.6 and taught in Phase 09. | [Lesson 05.6](05-rust-intermediate/06-testing-in-depth.md) |
| property-based test | A test that checks a property on many automatically generated inputs instead of a few chosen by hand. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| proposal | A one-page document that describes an idea before the design document: pitch, contract sketch, milestones, risks, non-goals. | [Lesson 23.7](23-capstone-projects/07-capstone-your-own-idea.md) |
| protected branch | A branch (e.g. main) that the server does not allow to be pushed to directly or force-pushed. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| .proto file | A text file with message and service definitions in the Protocol Buffers language. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| protoc | The Protocol Buffers compiler program that reads .proto files. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| protocol | A set of rules both sides follow: which bytes to send, in which order, and what they mean. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| Protocol Buffers (protobuf) | A language-neutral way to describe messages and to encode them into compact binary bytes. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| provenance | Signed information about how a file was built: repository, commit, workflow and build system. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| provider (OpenTofu) | A plugin that teaches OpenTofu to talk to one system, such as Kubernetes, Docker or a cloud, by turning resources into API calls. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| provisioning (Grafana) | Configuring Grafana data sources and dashboards from files at start-up instead of clicking them together, so they can live in Git. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| proxy | A program between a client and a server that passes their traffic on, possibly looking at or changing it. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| pseudo-random | Numbers that look random but come from a fixed formula and a starting number (the seed), so the same seed gives the same numbers. | [Lesson 07.15](07-algorithms-and-data-structures/15-two-pointers-and-sliding-window.md) |
| pseudo-random number generator | A formula that produces numbers that look random but are fully decided by the starting number (the seed). | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| pseudo-random numbers | Numbers that look random but come from a fixed formula; the same seed gives the same numbers, so a failing test can be repeated. | [Lesson 07.3](07-algorithms-and-data-structures/03-arrays-and-vectors-under-the-hood.md) |
| pseudocode | An algorithm written in plain words without strict language rules. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| psql | The command-line client of PostgreSQL: you type SQL and backslash commands, it shows the results. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| pub / pub(crate) / pub(super) | Visibility keywords: everyone / whole crate / parent module. Default is private. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| public API | The pub items other code may use; what you must document. | [Lesson 05.7](05-rust-intermediate/07-documentation.md) |
| publish a port | Connecting a port of the host to a port of a container with -p. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| pull | fetch followed by a merge into the current branch. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| pull request / merge request | A request to merge your branch into another, with review, discussion and automatic checks (GitHub / GitLab). | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| pull-up resistor | A resistor that connects an input to the high level so it reads high when nothing else drives it. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| pure function | A function that only reads its arguments and returns a value, with no side effects. | [Lesson 08.2](08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) |
| push | Send your new commits to a remote. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| push / pop | Add an item / remove an item at the working end of a stack. | [Lesson 07.5](07-algorithms-and-data-structures/05-stacks-and-queues.md) |

### Q

| Word | What it means | First used in |
|---|---|---|
| quadratic | O(n²): when the input doubles, the work becomes four times bigger. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| quality bar | The minimum quality a capstone must reach: tests, lints, benchmarks, documentation, CI and observability. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| quality gate | Commands that must pass before accepting work: cargo fmt --check, cargo clippy -- -D warnings, cargo test. | [Lesson 05.6](05-rust-intermediate/06-testing-in-depth.md) |
| quantile | The same idea as a percentile, written as a fraction: 0.99 instead of 99. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| query | One SQL command sent to a database, such as SELECT * FROM tasks. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| query engine | The part of a database that turns a query into steps and runs them over the data. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| query parameter (bind parameter) | A placeholder like $1 in SQL whose value is sent separately from the text; the defence against SQL injection. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| query plan | The steps the database will use to run a query, shown by EXPLAIN, for example an index scan or a sequential scan. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| query planner (optimizer) | The part of a database that chooses how to run a query: which scans, which join methods and in which order. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| queue | A collection where items are added at the back and removed at the front, first in first out. | [Lesson 07.5](07-algorithms-and-data-structures/05-stacks-and-queues.md) |
| queue group | NATS subscribers with the same group name that share the work: each message goes to only one of them. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| QUIC | A transport protocol over UDP with built-in TLS 1.3 encryption, many independent streams and connection IDs (RFC 9000). | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| QUIC datagram | An encrypted but unreliable message inside a QUIC connection; it may be lost and is never resent. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| quick start | The shortest list of commands that takes a stranger from git clone to a working system. | [Lesson 23.9](23-capstone-projects/09-presenting-your-work.md) |
| quickselect | An algorithm that finds the k-th smallest item by partitioning, without sorting everything; average time O(n). | [Lesson 07.9](07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) |
| quirk | A surprising but existing behaviour, such as a rounding rule or a formatting oddity. | [Lesson 08.16](08-software-design/16-project-refactor-a-messy-program.md) |
| quorum | The minimum number of replicas that must answer for an operation to count (W for writes, R for reads, out of N). | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| quorum index | The highest log index stored on a majority of nodes; a Raft leader commits it when its entry is from the current term. | [Lesson 18.15](18-distributed-systems/15-checkpoint-stage-4-complete.md) |
| quote | Crate whose quote! macro builds a TokenStream from code with #variable holes and #(#items),\* repetitions. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| quoting | Putting text in double or single quotes so that the shell does not split or change it. | [Lesson 11.8](11-linux-shell-and-containers/08-bash-scripting-basics.md) |

### R

| Word | What it means | First used in |
|---|---|---|
| Raft | A consensus algorithm designed to be easy to understand, built on leader election and log replication with terms and majorities. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| rainbow table | A big pre-computed list of password-to-hash pairs that reverses fast, unsalted hashes by lookup. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| random access | Reading any item directly by its position, in the same time for every position. | [Lesson 07.3](07-algorithms-and-data-structures/03-arrays-and-vectors-under-the-hood.md) |
| range query | A question about a range of keys, such as all keys from user:000100 to user:000199. | [Lesson 17.5](17-storage-and-databases/05-hash-indexes.md) |
| rate limit | The most requests one client may send per time unit; over it, the answer is 429. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| rate limiting | Allowing a client only a certain number of requests per time window, to protect a service. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| rayon | A crate for data parallelism with parallel iterators and join on a global work-stealing thread pool. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| re-entrant lock | A lock that the same thread may take again while it already holds it. std::sync::Mutex is not re-entrant. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| reactor | The part of an async runtime that waits for I/O events from the operating system (epoll, kqueue) and wakes the tasks that wait for them. | [Lesson 14.11](14-concurrency-and-async/11-build-a-tiny-async-runtime.md) |
| read amplification | Reading more places or more bytes than needed to answer one lookup. | [Lesson 17.7](17-storage-and-databases/07-lsm-trees.md) |
| read half / write half | The two parts of a split stream: one can only read, the other only write, so they can be used in different places. | [Lesson 15.6](15-networking/06-async-networking-with-tokio.md) |
| read index | A way to serve a linearizable read without a log entry: the leader confirms with one heartbeat round that it is still leader, waits until its commit index is applied, then reads. | [Lesson 23.3](23-capstone-projects/03-capstone-distributed-key-value-store.md) |
| read repair | When a read finds a replica with an old value, it writes the newest value back to it. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| read your writes | A session guarantee: after a client writes, its own reads see that write. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| read-modify-write | An operation in three steps: read a value, compute a new one, write it back. x += 1 is one. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| read-write lock (RwLock) | A lock that allows many readers at once or one writer, never both. It helps when reads are long and writes are rare. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| reader funnel | The order in which people look at a project: most read only the top of the README, few read the design and the code. | [Lesson 23.9](23-capstone-projects/09-presenting-your-work.md) |
| readiness | The state in which a call on a descriptor would not block: data to read or space to write. | [Lesson 13.13](13-operating-systems/13-non-blocking-io-and-epoll.md) |
| readiness check | A health check that asks whether the service can do real work now (for example, its database answers); if it fails, it gets no traffic but is not restarted. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| readiness model | The kernel reports when an operation could be done without waiting; the program then does it. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |
| readiness probe | Asks whether a pod can take traffic now; on failure the pod is removed from the Service's endpoints but keeps running. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| README | The file README.md at the top of a repository; code hosting sites show it on the project's front page. | [Lesson 24.2](24-career-and-open-source/02-building-your-portfolio.md) |
| real-time policy (SCHED_FIFO, SCHED_RR) | Scheduling policies whose tasks always run before normal tasks; they need root. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| reap | To collect a zombie's exit status with wait, after which it disappears. | [Lesson 13.5](13-operating-systems/05-fork-exec-and-wait.md) |
| rebalance | Assigning partitions to group members again after a member joins or leaves. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| rebalancing | Moving data between nodes when nodes are added or removed or the load is uneven. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| receive window | The number in each TCP segment that says how many more bytes the receiver can store. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| recommended labels | The app.kubernetes.io/... label keys suggested by the Kubernetes documentation, such as name and managed-by. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| reconcile / reconciliation | To compare desired and actual state and make the changes that remove the difference; one such run is a reconciliation. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| reconciliation loop | A loop that repeatedly observes the actual state, compares it with the desired state and acts to close the gap; also called a control loop. | [Lesson 21.1](21-cloud-native-infrastructure/01-what-cloud-native-means.md) |
| record | One unit of data in a file, for example one key and one value with a header. | [Lesson 17.3](17-storage-and-databases/03-binary-file-formats.md) |
| record batch | Arrow's unit of data: a schema plus one array per column with the same number of rows. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| recording rule | A Prometheus rule that computes a query regularly and stores the result as a new time series with its own name. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| recovery | What a storage engine does at start-up to return to a correct state after a crash, such as replaying the log and cutting a torn tail. | [Lesson 17.4](17-storage-and-databases/04-write-ahead-log.md) |
| recruiter | A person whose job is to find and contact candidates, inside a company or for an agency. | [Lesson 24.6](24-career-and-open-source/06-resume-and-job-search.md) |
| recurrence | The formula that gives the answer of a state from the answers of smaller states. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| recursion | A function calling itself, each call solving a smaller version of the same problem. | [Lesson 07.7](07-algorithms-and-data-structures/07-recursion.md) |
| recursion depth | How many calls are waiting on the call stack at the same time; it is the space cost of a recursion. | [Lesson 07.7](07-algorithms-and-data-structures/07-recursion.md) |
| recursive case | The part of a recursive function that reduces the problem and calls the function again. | [Lesson 07.7](07-algorithms-and-data-structures/07-recursion.md) |
| recursive descent | A parsing method with one function per grammar rule; the functions call each other as the rules do. | [Lesson 22.22](22-specializations/22-compilers-track-lexing-and-parsing.md) |
| recursive resolver | A DNS server that finds answers for its clients by asking root, TLD and authoritative servers, and caches the results. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| redact | Replace secret text with a placeholder such as <redacted> before printing or logging it. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| redaction | Replacing a value that changes on every run, such as an id or a time, with a fixed placeholder before comparing. | [Lesson 09.5](09-testing-debugging-benchmarking/05-snapshot-testing.md) |
| redirect (HTTP) | An HTTP response with a 3xx status and a Location header that tells the client to go to another address. | [Lesson 16.15](16-backend-services/15-project-url-shortener-service.md) |
| redirection | Sending a stream to a file, or reading it from a file, instead of the terminal, with >, >>, < and 2>. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| redo information | What recovery needs to repeat a committed change after a crash: the new value. | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| reduction | Combining all values into one result, such as a sum, a maximum or a count. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| redundancy | Having more than one copy of a part, so the system keeps working when one copy fails. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| reference implementation | A simple, obviously correct version of a function that tests compare a faster or parallel version against. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| referral | A DNS answer that does not contain the record but names the servers responsible for that part of the tree. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| reflector / store | A local, always up-to-date copy of watched objects inside an operator, so it does not ask the API server each time. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| region | A piece of code that the compiler counts separately for coverage, for example one branch of an if. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| register (peripheral) | A small piece of memory inside a peripheral; writing a number to it changes what the hardware does. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| registry (images) | A server that stores container images, such as Docker Hub or ghcr.io; nodes pull images from it. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| regression file | A file in which proptest saves a failing input so that it is tried first on the next run. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| regression file (proptest) | The file in which proptest saves seeds of failing cases so that every later run tries them first. | [Lesson 23.8](23-capstone-projects/08-testing-benchmarking-and-hardening-capstones.md) |
| regular expression | A short pattern that describes a set of texts, such as ^[a-z]+$. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| rehashing | Building a bigger table and moving every entry to its new bucket. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| rejection (axum) | The error response axum sends by itself when an extractor fails, for example 400 for broken JSON; the handler does not run. | [Lesson 16.3](16-backend-services/03-web-services-with-axum.md) |
| relational database | A database that keeps data in tables with typed columns and lets queries combine them; PostgreSQL is one. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| relational operator | One basic step on tables, such as scan, filter, projection, aggregate, sort, limit or join. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| Relaxed | The weakest memory ordering: only the operation on this one variable is atomic. Enough for counters, IDs and stop flags. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| relaxing an edge | Checking whether going through a vertex gives a shorter way to its neighbour than the best known one, and updating it if so. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| release (Helm) | One named, installed copy of a chart in a cluster. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| Release / Acquire | A Release store makes everything before it visible to an Acquire load that reads its value; together they create happens-before between threads. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| release notes | The list of changes in a new version of a program; Rust publishes them on its blog for every release. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| reliability | The ability of a system to keep doing the right job for a long time, also when something goes wrong. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| relocation | A note in a binary file that says where the final address of a symbol must be written; the linker or the loader processes it. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| rendezvous hashing | Every node gets a score hash(node + key) and the node with the highest score owns the key; also called highest random weight. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| reordering | Memory operations become visible to other threads in a different order than the program wrote them; both the compiler and the CPU may do it. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| replacement character | U+FFFD, shown as �, which stands in for bytes that could not be decoded. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| replay | An attack that sends an old, valid message or token again. Expiry times and one-time values defend against it. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| replica | One copy of the data, usually on its own machine. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| ReplicaSet | A Kubernetes object that keeps a given number of identical pods running. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| replicated state machine | A program on every node that changes its state only by applying commands from a shared log, in order. Same log, same state. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| replication | Keeping several copies (replicas) of the same data up to date. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| replication lag | How far a follower is behind the leader, in changes it has not applied yet. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| replication log | The ordered list of changes that the leader sends to its followers. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| repository (packages) | A server with many packages and an index file that lists them. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| #[repr(C)] | Attribute that gives a Rust struct the same memory layout as in C. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| race condition | A bug that depends on thread timing (still possible in safe Rust, unlike a data race). | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| random number | An unpredictable number. Computers produce pseudo-random numbers (rand crate). | [Lesson 02.10](02-programming-basics/10-project-guessing-game.md) |
| range | A sequence of integers: 1..5 excludes 5, 1..=5 includes 5. A range is itself an iterator. | [Lesson 02.5](02-programming-basics/05-control-flow.md) |
| range pattern | Pattern matching a span of values: 1..=9. | [Lesson 04.4](04-structs-enums-collections-errors/04-option-and-match.md) |
| raw pointer | A pointer without reference safety rules; may be null, dangling or misaligned. Creating is safe, dereferencing is unsafe. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| raw string | Text written r"..." where backslashes are not special. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| re-export (pub use) | Making an item available under a new, usually shorter, public path. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| reallocation | Moving items to a new, bigger piece of heap memory because the old one is full. | [Lesson 04.7](04-structs-enums-collections-errors/07-vectors.md) |
| rebase | Replay your commits on top of another base commit, creating new commits. Never rebase shared history. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| reborrow | Making a short new reference from an existing &mut so the original can be used again. | [Lesson 03.5](03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| receiver | The value a method is called on (left of the dot), called self inside. | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| recoverable error | An error the program can reasonably handle; modelled with Result. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| recursion / base case | A function calling itself; the base case is the simple case that stops it. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| recursive type / indirection | A type containing a value of its own type; needs indirection (Box) to have a finite size (error E0072). | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| red / green / refactor | The three TDD steps: failing test, passing test, improve code with tests green. | [Lesson 05.6](05-rust-intermediate/06-testing-in-depth.md) |
| redirect | &gt; and &gt;&gt; send command output into a file. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| Ref / RefMut | Guard values returned by borrow() and borrow_mut(). The borrow lasts while the guard lives. | [Lesson 06.4](06-rust-advanced/04-interior-mutability.md) |
| refactoring | Changing how code is written without changing what it does; tests pass before and after. | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| `RefCell<T>` | A box that lends &T and &mut T and checks the borrowing rules at run time; a violation panics. | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| reference | A value that points to another value without owning it, written &value with type &T. Lets code use a value without taking it away. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| reference count | Number of owners of an Rc value (strong count). Rc::clone only increments it; the value is freed at zero. | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| reference cycle | Values owning each other in a circle so counts never reach zero; memory is never freed (a memory leak). | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| reflog | A private diary of where HEAD has been. It lets you rescue commits no branch points to. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| refutable / irrefutable pattern | A pattern that can fail to match (Some(x)) vs one that always matches (x, (a, b)). | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| register | A tiny, very fast storage place inside the CPU. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| registry | A server that stores published crates and versions. The default registry is crates.io. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| regression | A bug that comes back after a fix, or something that worked and is now broken. | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| regression test | A test written to capture a fixed bug so it cannot return unnoticed. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| release | A named, tagged version of the project, e.g. v0.1.0. | [Lesson 01.10](01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) |
| release channel | One of three kinds of Rust release: stable, beta, nightly. The course uses stable (a new version every 6 weeks). | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| remainder | What is left after integer division (%). | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| remote | A copy of a repository on another computer, such as GitHub or GitLab. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| remote-tracking branch | Your local note of where a branch was on the remote at the last contact, e.g. origin/main. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| repetition (macro) | Macro syntax $( ... ),\* / + / ? that matches or produces a piece of code many times. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| repository | A folder tracked by Git together with its history (the files plus the hidden .git folder). | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| representation | The form in which a resource travels in a request or a response, usually JSON. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| representation (repr) | The set of layout rules for a type: repr(Rust) by default, or repr(C), repr(packed), repr(align(N)), repr(transparent). | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| reproduce | Make a bug happen again reliably with the same steps. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| reproducible | Another person can follow your steps and get the same result. | [Lesson 24.2](24-career-and-open-source/02-building-your-portfolio.md) |
| reproducible build | A build where the same source, toolchain and settings always give a byte-for-byte identical file. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| repudiation | Denying that you did something when the system cannot prove otherwise. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| request context | A value created for each request and passed to every hook of that request so they can share data. | [Lesson 22.20](22-specializations/20-networking-track-production-proxies.md) |
| request head | The request line and headers of an HTTP/1.1 request, ending with an empty line; the body follows it. | [Lesson 22.19](22-specializations/19-networking-track-zero-copy-parsing.md) |
| request ID | A number a client puts into each request and the server copies into the answer, so answers can be matched to requests. | [Lesson 15.5](15-networking/05-udp-sockets-in-rust.md) |
| request line | First line of an HTTP request: method, path, version. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| request smuggling | An attack in which two HTTP programs disagree on where a request ends, so a hidden second request slips through. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| request/reply | A messaging pattern where a message expects one answer, sent back to a private reply subject (an inbox). | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| requeue | Asking the controller to reconcile an object again after some time, even if nothing changed. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| requirement | Something a system must do (functional) or a property it must have, such as speed or durability (non-functional); a good one has a number, a unit and a test. | [Lesson 23.2](23-capstone-projects/02-writing-a-design-document.md) |
| reqwest | A high-level HTTP client for Rust built on hyper, with a connection pool, redirects, timeouts and HTTPS. | [Lesson 15.9](15-networking/09-http-with-hyper-and-reqwest.md) |
| research paper | An article in which researchers describe a new idea or system and the evidence for it, usually reviewed by experts. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| reset | Move the current branch to another commit (--soft, --mixed, --hard). The mode decides whether the staging area and files change too. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| reset handler | The first function that runs after power-on; it prepares RAM and calls the program's entry function. | [Lesson 22.10](22-specializations/10-embedded-track-microcontrollers-and-no-std.md) |
| resolve a conflict | Edit the file to its final form, remove the markers and tell Git with git add. | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| resolver | The part of Cargo that picks the versions and features of dependencies. | [Lesson 08.12](08-software-design/12-project-layout-and-workspaces.md) |
| resource | Something taken that must be given back: memory, an open file, a lock, a connection. | [Lesson 08.7](08-software-design/07-raii-and-guards.md) |
| resource (OpenTelemetry) | The description of who sends telemetry, most importantly service.name. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| resource (OpenTofu) | One thing that OpenTofu creates and manages, such as a namespace or a Deployment. Its address is type.name. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| resource (REST) | A thing an API lets you work with, such as a task or a list of tasks; it has its own URL. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| resource attributes (OpenTelemetry) | Key-value pairs that describe who sends telemetry, such as service.name and k8s.pod.name. They can be set with OTEL_RESOURCE_ATTRIBUTES. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| resource exhaustion | A DoS in which the attacker makes the program use up memory, CPU time or disk. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| resource limit | The most CPU or memory a container may use; above its memory limit it is killed. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| resource request | The CPU and memory reserved for a container; the scheduler places pods by their requests. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| resource requests / limits | The CPU and memory a container is guaranteed (requests, used for scheduling) and may use at most (limits). | [Lesson 21.13](21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) |
| responsibility | A reason for a piece of code to change. | [Lesson 08.3](08-software-design/03-solid-principles.md) |
| responsible disclosure | Telling the owner privately about a vulnerability and giving them time to fix it before speaking about it publicly. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| restart count | How many times the kubelet restarted a container of a pod; the RESTARTS column. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| restart policy | The rule that says when systemd starts a service again after it ended: never, on failure or always. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| restore | Bring a file back to the state of a commit or the staging area (git restore). | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| Result | A type that is Ok(value) or Err(error). Used when an operation can fail. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| Result&lt;T, E&gt; | An enum with Ok(T) for success and Err(E) for an error. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| resume | A short document (one or two pages) that summarises your skills, projects and experience for a job application. | [Lesson 24.6](24-career-and-open-source/06-resume-and-job-search.md) |
| retention | The rule for how long a broker keeps records, by age or total size. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| retransmission | Sending data again because no acknowledgement came back in time. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| retrieval practice | Learning by trying to recall an answer from memory instead of rereading. | [Lesson 04.12](04-structs-enums-collections-errors/12-checkpoint.md) |
| retrospective | A short look back: what went well, what was hard, what to change. | [Lesson 06.12](06-rust-advanced/12-checkpoint-stage-1-complete.md) |
| retry | Sending the same request again after it failed or timed out. | [Lesson 18.3](18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| retry budget | A rule that retries may be at most a fixed share of all requests, so retries cannot multiply an overload. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| retry storm | Many clients retry failed requests at the same time and multiply the load on an overloaded service. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| retryable / fatal error | A retryable error (for example a timeout) may succeed on another try; a fatal error (for example bad input) never will. | [Lesson 14.15](14-concurrency-and-async/15-project-concurrent-job-queue.md) |
| return value / return type | The value a function gives back and its type, written after -&gt;. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| Reverse | The wrapper std::cmp::Reverse<T> that flips the order of T, so a max-heap acts as a min-heap. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| reverse Polish notation | A way to write expressions with the operator after its two numbers, for example 3 4 + for 3 + 4; evaluated with a stack. | [Lesson 07.20](07-algorithms-and-data-structures/20-checkpoint.md) |
| reverse proxy | A proxy in front of servers: clients think it is the service, and it passes requests to the real servers. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| revert | Create a new commit that undoes an older commit, without rewriting history. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| review (snapshot) | Looking at the difference between the old and the new snapshot and deciding whether the change is intended. | [Lesson 09.5](09-testing-debugging-benchmarking/05-snapshot-testing.md) |
| review comment | A remark that a reviewer attaches to a line or to a whole pull request. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| reviewer / author | The person who checks a change / the person who wrote it. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| revision | Any way to name a commit: hash, HEAD, HEAD~2, a branch or a tag. | [Lesson 01.4](01-git-and-version-control/04-viewing-history-and-changes.md) |
| revision (Deployment) | A numbered version of a Deployment's pod template; every template change creates a new one. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| revision (Helm) | A numbered version of a Helm release; every install, upgrade and rollback creates one. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| revoke | Cancel a secret at the service that issued it, so it stops working. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| rewriter (SQL) | The stage that applies rules to the query tree, for example replaces a view by its definition. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| ring buffer | A fixed block of slots used as a circle: after the last slot comes the first one again. | [Lesson 07.5](07-algorithms-and-data-structures/05-stacks-and-queues.md) |
| ring buffer (eBPF) | A map that works as a queue of records from the kernel to user space. | [Lesson 22.8](22-specializations/08-ebpf-track-maps-and-tracing.md) |
| risk | How bad a threat is for you: roughly how likely it is multiplied by how much damage it does. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| role | A named set of permissions given to users, for example viewer, editor, admin. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| role fit | How well your current skills cover the skills a role needs, as a percentage (Lesson 24.1's program). | [Lesson 24.1](24-career-and-open-source/01-roles-for-systems-engineers.md) |
| rollback | Undoing the changes of an unfinished operation so the data returns to its earlier state. | [Lesson 08.7](08-software-design/07-raii-and-guards.md) |
| rollback (transaction) | Ending a transaction by throwing its changes away. | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| rolling update | Replacing the pods of a Deployment a few at a time so that the service keeps running. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| rolling window | A time window that always ends now, for example the last 30 days, so old events leave it day by day. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| rollout | Replacing the pods of a Deployment step by step with new ones after its pod template changed. | [Lesson 22.29](22-specializations/29-cloud-native-track-project-and-next-steps.md) |
| rollout restart | A kubectl command that replaces all pods of a Deployment with new ones through a rolling update. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| root | The top node of a tree; it has no parent. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| root (DNS) | The top of the DNS name tree, written as the final dot of a name. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| root (user) | The administrator, the user with UID 0, for whom the kernel does not check permissions. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| root certificate | A CA certificate that the system already trusts; shipped in the trust store. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| root commit | The very first commit of a repository, which has no parent. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| root directory | The top of the file tree, written /. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| root file system | The directory tree a process or container sees as /: its own /bin, /etc, /lib and so on. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| Rosetta | macOS layer that emulates Intel programs on Apple silicon. Installing Rust under it gives the wrong host. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| rotation | Replacing a secret with a new one, on a schedule or after a leak, and cancelling the old one. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| rotation (bit rotation) | A shift where bits that fall out on one side come back on the other side, as in rotate_left. | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| rotation (on-call) | The schedule that passes on-call duty from person to person, for example one week each. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| round robin | Choosing backends in turn: 1, 2, 3, 1, 2, 3, ... | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| round trip | Doing an operation and its reverse, such as encode then decode, and expecting the original value back. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| round trip (test) | Encoding a value and decoding it again; a correct codec gives back exactly the original. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| round-trip property | The rule decode(encode(x)) == x for every x; one of the most useful properties for formats and protocols. | [Lesson 23.8](23-capstone-projects/08-testing-benchmarking-and-hardening-capstones.md) |
| round-trip test | A test that converts a value into another form and back and checks that the result equals the original value. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| rounding error | The small difference between the exact result of an operation and the nearest float the computer can store. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| route | A path from the start city to the goal city: the list of cities in order. | [Lesson 07.19](07-algorithms-and-data-structures/19-project-route-finder.md) |
| router | A device that receives packets and forwards each one towards its destination. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| routing table | The list of rules a computer uses to decide where to send each packet. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| row lock | A lock on one row: others that want to change or lock it wait until the holder commits or rolls back. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| row version (tuple) | One stored copy of a row, with notes about which transaction created it and which deleted it. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| row-major order | Storing a grid row by row in one flat buffer; element (row, col) is at index row * width + col. | [Lesson 12.10](12-computer-systems/10-cache-friendly-code.md) |
| rsync | A program that copies only the parts of files that changed. | [Lesson 11.10](11-linux-shell-and-containers/10-ssh-and-remote-servers.md) |
| rubber duck debugging | Explaining your code aloud line by line to find the mistake. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| rubric | A table of criteria with a description of each level, so that a review gives the same result whoever does it. | [Lesson 23.10](23-capstone-projects/10-checkpoint-capstone-review.md) |
| rule of three | Copy once if you must, but extract a shared abstraction when the same code appears a third time. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |
| run queue | The list of runnable tasks waiting for one CPU core; each core has its own. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| run-time check | A check done while the program runs instead of at compile time; failure causes a panic. | [Lesson 06.4](06-rust-advanced/04-interior-mutability.md) |
| run-time error | An error found while a program runs, such as division by zero, as opposed to a syntax error. | [Lesson 22.23](22-specializations/23-compilers-track-an-interpreter.md) |
| runbook | A short document for the person on call: how to recognise a problem, what to check and what to do. | [Lesson 20.9](20-reliability-and-hardening/09-production-readiness-checklist.md) |
| runner | The machine, a virtual machine or a container, that runs a CI job. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| Rust | A programming language for fast and safe programs. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| Rust API Guidelines | A checklist from the Rust library team for designing public APIs that feel idiomatic. | [Lesson 08.11](08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) |
| rust-lldb | A wrapper script that starts LLDB with helper files so that Rust types print in a readable way. | [Lesson 09.9](09-testing-debugging-benchmarking/09-debugging-with-lldb-and-rustrover.md) |
| RUST_LOG | An environment variable that tells env_logger which log levels to print. | [Lesson 22.7](22-specializations/07-ebpf-track-a-first-program-with-aya.md) |
| rustc | The Rust compiler. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| rustdoc | The tool shipped with Rust that turns doc comments into HTML; cargo doc runs it. | [Lesson 05.7](05-rust-intermediate/07-documentation.md) |
| RUSTFLAGS | An environment variable with extra options that Cargo passes to the Rust compiler. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| rustfmt | The official formatter that formats code in the standard style (cargo fmt). | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| RustRover | JetBrains' IDE for Rust, free for non-commercial use. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| rustup | The installer and updater of Rust toolchains. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |

### S

| Word | What it means | First used in |
|---|---|---|
| S-expression | A bracketed way to write a tree with the operator first, such as (+ 1 (* 2 3)). | [Lesson 22.22](22-specializations/22-compilers-track-lexing-and-parsing.md) |
| SA_RESTART | A sigaction flag: restart an interrupted system call after the handler instead of failing with EINTR. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| safe abstraction | A function with a safe signature whose body uses unsafe internally and checks everything needed for soundness. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| safe method | An HTTP method that only reads and never changes data on the server, such as GET and HEAD. | [Lesson 16.2](16-backend-services/02-designing-http-apis.md) |
| safe wrapper | A small safe Rust API (checks, Result, Drop) that hides unsafe FFI calls. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| safety (property) | "Nothing bad ever happens", for example two nodes never decide different values. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| SAFETY comment | Convention: every unsafe block explains why it is valid; every unsafe fn documents what the caller must guarantee. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| safety net (tests) | A set of fast tests that tell you within seconds whether a change broke something. | [Lesson 08.13](08-software-design/13-refactoring-and-code-review.md) |
| saga | A long operation split into local transactions; if one fails, the finished ones are undone by compensations in reverse order. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| salt | A random value, different for every password, mixed in before hashing. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| sample | One snapshot of what a thread was running at one moment. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| sampling profiler | A profiler that records the call stack many times per second; functions in many samples are where time goes. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| sandbox | An environment where code can only do what the host explicitly allows. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| sanitization | Changing input to remove the dangerous parts; easy to get wrong, so prefer rejecting. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| sanitizer | A compiler feature that adds runtime checks to a program, for example for invalid memory use. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| sans-IO | A style of protocol code with no input/output inside: it receives events and returns messages to send, and other code does the networking. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| saturating cast | A conversion that clamps a value to the smallest or largest value of the target type instead of wrapping; float-to-integer `as` works like this. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| saturation | The state in which a resource is busy all the time, so extra load only makes the queue longer. | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| scalability | The ability of a system to handle more work by adding more machines. | [Lesson 18.1](18-distributed-systems/01-what-a-distributed-system-is.md) |
| scalar | One single value, or code that works on one value at a time. The opposite of vector. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| scalar type | A type holding one value: integer, float, bool, char. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| schedule | The decisions about which thread runs next that produce one interleaving. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| scheduler | The part of the kernel that decides which task runs on which CPU core and for how long. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| scheduler (Kubernetes) | The Kubernetes component that chooses a node for each new pod. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| schema (database) | The structure of a database: its tables, columns, types, constraints and indexes, not the data in them. | [Lesson 16.7](16-backend-services/07-migrations-and-transactions.md) |
| schema drift | When databases that should have the same schema slowly become different; migrations prevent it. | [Lesson 16.7](16-backend-services/07-migrations-and-transactions.md) |
| schema inference | Guessing the type of each column of a data file by looking at its values. | [Lesson 22.17](22-specializations/17-databases-track-project-and-next-steps.md) |
| scope | The part of the code where a name can be used; a block creates a new scope. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| scope creep | When a project slowly grows unplanned features, so it never finishes. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| scope guard | A guard that runs code at the end of a scope however the scope ends. | [Lesson 08.7](08-software-design/07-raii-and-guards.md) |
| scoped thread | Thread from thread::scope guaranteed to end before the scope ends, so it may borrow local data without move. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| scrape | One visit of Prometheus to a program's /metrics page to read the current values. It repeats every scrape interval. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| scratch buffer | A temporary block of memory that an algorithm uses for its work. | [Lesson 07.9](07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) |
| script | A text file with commands the shell runs one after another. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| scrutinee | The expression that match or if let looks at. Its temporaries live until the end of the whole match. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| secret | A value that gives access or power to whoever knows it: a password, an API key, a token, a private key. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| secret (CI) | A value such as a token that the CI system stores and gives to jobs without showing it in the logs. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| secret (Compose) | A password or key given to a container as a file under /run/secrets/. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| Secret (Kubernetes) | A Kubernetes object for small secret values; base64-encoded, not encrypted by default. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| secret file | A file, mounted by the orchestrator, that contains one secret such as a password, so the secret is not in an environment variable. | [Lesson 16.13](16-backend-services/13-configuration-health-checks-and-containers.md) |
| secret manager | A service that stores secrets encrypted, controls who may read them and records every access. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| sector | The smallest unit a storage device reads or writes, usually 512 bytes or 4 KiB. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| security | Keeping a system working correctly and its data safe even when someone is trying to make it fail. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| security context | Pod or container settings that restrict what it may do, such as running as non-root or a read-only file system. | [Lesson 21.4](21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) |
| security review | Reading code or a design with the question how could this be abused instead of does this work. | [Lesson 10.15](10-security-fundamentals/15-checkpoint.md) |
| seed | The starting number of a pseudo-random generator; the same seed gives the same sequence. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| seed (fuzzing) | A hand-written input put into the corpus so that the fuzzer starts from something meaningful. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| segment | One file of a partition's log; old segments are deleted as a whole. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| segment / section | Parts of a binary file: sections (.text, .data) are what the linker works with, segments are what the loader maps into memory. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| segmentation fault | The SIGSEGV signal, sent when a program touches an address it may not use; by default the program dies. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| select! (Tokio) | A macro that waits for several futures at once and runs the branch of the first one that finishes. | [Lesson 15.6](15-networking/06-async-networking-with-tokio.md) |
| selective staging | Staging only some changed files (or hunks, git add -p) so that a commit contains one logical change. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| selectivity | The fraction of rows that a condition keeps; a very selective condition keeps only a few rows. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| selector | A rule that picks objects by their labels; every rule must hold, extra labels do not matter. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| Self / self | Self is the type of the impl block; self is the receiver value (&self, &mut self, self). | [Lesson 04.2](04-structs-enums-collections-errors/02-methods-and-associated-functions.md) |
| self time | Samples in which a function itself was running (top of the stack). | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| self-assessment | Rating your own skills honestly, with evidence, to decide what to learn next. | [Lesson 24.9](24-career-and-open-source/09-checkpoint-course-complete.md) |
| self-healing | A system noticing that the actual state drifted (for example an object was deleted) and fixing it by itself. | [Lesson 22.26](22-specializations/26-cloud-native-track-kubernetes-operators.md) |
| self-referential | A value that contains a pointer or reference to another part of itself. | [Lesson 14.12](14-concurrency-and-async/12-pin-and-self-referential-futures.md) |
| self-referential struct | A struct with a field pointing into another field of itself. Safe Rust cannot express it. | [Lesson 06.5](06-rust-advanced/05-lifetimes-in-depth.md) |
| self-review | Reviewing your own work with a fixed checklist and some distance, as if someone else had written it. | [Lesson 23.10](23-capstone-projects/10-checkpoint-capstone-review.md) |
| self-signed certificate | A certificate signed by its own key. Nobody vouches for it, so clients reject it unless explicitly told to trust it. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| self-synchronising encoding | An encoding where you can find the start of a character from any byte position; UTF-8 is one because continuation bytes start with 10. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| self-test | A test that you give yourself and mark yourself, to find gaps. | [Lesson 07.20](07-algorithms-and-data-structures/20-checkpoint.md) |
| semantics | What programs of a language mean: what each operation computes and what happens on errors. | [Lesson 22.25](22-specializations/25-compilers-track-project-and-next-steps.md) |
| semicolon | The mark ; that ends a statement in Rust. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| semihosting | A way for a program inside QEMU to ask QEMU itself for services, such as exiting with a code. | [Lesson 22.5](22-specializations/05-os-track-project-and-next-steps.md) |
| Send / Sync | Marker traits: Send = value can move to another thread; Sync = &T can be shared between threads. The compiler adds them automatically. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| separation of concerns | Keeping each concern in its own part of the code. | [Lesson 08.2](08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) |
| sequence number | The number of the first byte in a TCP segment, used to order bytes and find gaps. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| sequential access | Reading or writing neighbouring blocks one after another; storage devices are fastest this way. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| sequential consistency | All clients see the operations in one order that keeps each client's own order, but not necessarily real time. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| sequential flooding | A big scan that pushes every useful page out of an LRU cache, so the cache gets no hits. | [Lesson 17.9](17-storage-and-databases/09-page-cache-and-buffer-pool.md) |
| serial console | Text that a machine sends over its UART; QEMU shows it in your terminal. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| serial execution | Running transactions one at a time with no overlap: always correct, but slow. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| serializable | A way of running transactions concurrently whose result equals some serial order of the same transactions. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| serialization | Turning a value into text or bytes in a known format, for example JSON. | [Lesson 09.5](09-testing-debugging-benchmarking/05-snapshot-testing.md) |
| serialization / deserialization | Turning a value into text/bytes and back. | [Lesson 05.11](05-rust-intermediate/11-practical-crates-serde-clap-errors.md) |
| serialization failure | The database aborts a transaction because committing it would cause an anomaly; the application should retry the whole transaction. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| server / client | A server waits for connections and answers requests; a client (curl, browser) connects and sends requests. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| server streaming | An RPC where one request gets many responses that arrive one by one. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| service | A program that runs for a long time and answers requests from other programs, such as a web server. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| service (backend service) | A long-running program that waits for requests over the network and answers many clients at the same time. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| service (Compose) | One part of an application, run as one or more containers from one image. | [Lesson 11.14](11-linux-shell-and-containers/14-docker-compose.md) |
| service (hyper) | A function or object that turns a request into a future of a response; hyper calls it once per request. | [Lesson 15.9](15-networking/09-http-with-hyper-and-reqwest.md) |
| Service (Kubernetes) | A Kubernetes object that gives a group of pods one stable virtual IP address and DNS name. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| ServiceAccount | A Kubernetes identity for programs that run in pods; RBAC roles are bound to it. | [Lesson 22.28](22-specializations/28-cloud-native-track-testing-and-running-operators.md) |
| session | The server's memory that this browser belongs to a user who logged in a while ago. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| session fixation | An attack where the attacker gives the victim a session token that the attacker already knows and waits for the victim to log in. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| session guarantee | A promise about the calls of one client, such as read-your-writes or monotonic reads. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| session resumption | A TLS feature that lets a returning client skip most of the handshake work. | [Lesson 15.12](15-networking/12-tls-in-rust-servers.md) |
| session token | A random secret text that the browser sends with each request so the server can find the session. | [Lesson 10.7](10-security-fundamentals/07-authorization-sessions-and-tokens.md) |
| set | A collection of unique values `(HashSet<T>`, `BTreeSet<T>`). | [Lesson 04.9](04-structs-enums-collections-errors/09-hash-maps.md) |
| set / clear / toggle a bit | Make a bit 1 (with \|), make it 0 (with & and an inverted mask), or flip it (with ^). | [Lesson 12.4](12-computer-systems/04-bitwise-operations.md) |
| setuid / setgid | A bit that makes a program run as the owner or group of the file; on a directory, setgid makes new files inherit its group. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| setup (benchmark) | Work needed before each measured call, such as copying the input, which should not be timed. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| severity (incident) | How bad an incident is; it decides who is called, how fast and how often updates are sent. | [Lesson 21.12](21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) |
| shadowing | Declaring a new variable with the same name as an old one. The new one hides the old one and may have another type. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| sharding (lock striping) | Splitting data into parts (shards), each with its own lock, so threads that touch different parts do not wait for each other. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| shared library | A library file (.so on Linux, .dylib on macOS) used by many programs at once and loaded at run time. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| shared mapping | A mapping whose changes go to the file and are seen by every process that maps it (MAP_SHARED). | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| shared reference | A read-only reference &T. Any number can exist at once. | [Lesson 03.4](03-ownership-and-borrowing/04-references-and-borrowing.md) |
| shared state | Data that more than one thread can read or change. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| shebang | The first line of a script (#!/bin/zsh) naming the program that runs it. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| shell | The program inside the terminal that reads and runs your commands; zsh on macOS. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| shell array | A shell variable that holds a list of values. | [Lesson 11.8](11-linux-shell-and-containers/08-bash-scripting-basics.md) |
| shell function | A named group of commands inside a script that can be called again and again. | [Lesson 11.8](11-linux-shell-and-containers/08-bash-scripting-basics.md) |
| shell option | A switch of the shell itself, turned on with set -o name and off with set +o name. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| shell variable | A named piece of text such as name="Ann"; read it with $name. | [Lesson 11.8](11-linux-shell-and-containers/08-bash-scripting-basics.md) |
| ShellCheck | A program that reads a shell script and warns about typical mistakes. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| shifting | Moving a group of items one place to the left or right to open or close a gap. | [Lesson 07.3](07-algorithms-and-data-structures/03-arrays-and-vectors-under-the-hood.md) |
| short code | The short part of a short link, such as learn in http://127.0.0.1:8080/learn. | [Lesson 16.15](16-backend-services/15-project-url-shortener-service.md) |
| short read | A read call that returns fewer bytes than the buffer could hold; normal for sockets, pipes and slow devices. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |
| short-circuit evaluation | && and \|\| skip the right side when the left side already decides the result. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| short-circuiting | Stopping as soon as the answer is known (any, all, find). | [Lesson 05.5](05-rust-intermediate/05-iterators.md) |
| shortest path | A path whose total weight is the smallest; in an unweighted graph, the path with the fewest edges. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| shotgun surgery | A smell where one small change requires edits in many different places. | [Lesson 08.10](08-software-design/10-anti-patterns-and-code-smells.md) |
| shrinking | After a failure, trying smaller and simpler inputs to find the minimal failing input that is easy to read. | [Lesson 09.4](09-testing-debugging-benchmarking/04-property-based-testing.md) |
| sibling | One of several concurrent versions of the same value that a database keeps because it cannot know which is newer. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| side effect | Anything a function does besides returning a value, such as printing or reading a file. | [Lesson 08.2](08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) |
| sidecar | A helper container that runs in the same pod as the main container, for example to ship logs. | [Lesson 21.2](21-cloud-native-infrastructure/02-kubernetes-concepts.md) |
| sift up / sift down | Moving an item up (swapping with its parent) or down (swapping with its bigger child) until the heap property holds. | [Lesson 07.12](07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) |
| SIGBUS | The bus error signal; for a mapping it means the program touched a page that has no file data behind it anymore. | [Lesson 13.11](13-operating-systems/11-memory-mapped-files.md) |
| sign / exponent / fraction (mantissa) | The three fields of a float: one sign bit, the power of two, and the binary digits after the point. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| sign bit | The top bit of a signed integer; in two's complement it is 1 for every negative number. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| sign extension | Making a signed number wider by copying its sign bit into the new high bits, so the value does not change. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| sign-magnitude | An old way to write negative numbers: one bit for the sign and the rest for the size. It has two zeros and is not used for integers today. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| sign-off | The line Signed-off-by: Name <email> at the end of a commit message; a statement, not a cryptographic signature. | [Lesson 24.4](24-career-and-open-source/04-your-first-open-source-contribution.md) |
| signal | A small message that the kernel or another process sends to a process, such as please stop. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| Signal (embassy-sync) | A small type that holds the latest value; one task waits until another task signals a new one. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| signal (SIGTERM) | A small message from the operating system to a process; SIGTERM means please stop, SIGKILL stops the process at once. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| signal (telemetry) | One kind of telemetry: traces, metrics or logs. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| signal handler | A function the kernel calls when a signal arrives, interrupting the program at any point. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| signal mask | The set of signals a thread does not accept right now; they stay pending until unblocked. | [Lesson 13.7](13-operating-systems/07-signals-in-your-programs.md) |
| signature | The first line of a function: name, parameters, return type. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| signed / unsigned | A signed type (i32) can be negative; an unsigned type (u32) never is. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| signed overflow | The true result of a signed operation is outside the range of the type, for example two positive numbers whose sum looks negative. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| signed token | Text of the form payload.tag, where the tag is a MAC of the payload. The server can trust it later without a database if the tag verifies. | [Lesson 10.9](10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) |
| significant digits | How many leading digits of a value are kept exactly; hdrhistogram's precision setting. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| SIGTERM / SIGKILL | Signal 15, a polite request to terminate that the program may handle, and signal 9, which the kernel enforces and no program can catch. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| silent failure | A failure that is not noticed at once because the program does not crash or report it; it just gives a wrong result. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| simulated time | A clock that the simulation moves forward itself, so long waits take almost no real time. | [Lesson 18.12](18-distributed-systems/12-deterministic-simulation-testing.md) |
| single point of failure | A part without a copy: when it fails, the whole system fails. | [Lesson 20.1](20-reliability-and-hardening/01-what-reliability-means.md) |
| singly linked | Each node knows only the next node, not the previous one. | [Lesson 07.4](07-algorithms-and-data-structures/04-linked-lists.md) |
| sink | Something you can send values into one after another; the opposite of a stream. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| size class | A fixed block size to which real allocators round small requests, keeping one free list per class. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| size-tiered compaction | A compaction strategy that merges several tables of similar size into one bigger table. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| Sized | Marker trait for types whose size is known at compile time. dyn Trait is not Sized; ?Sized allows a type without known size, such as str. | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| slice | A reference to a continuous part of a sequence: a pointer and a length (&str, &[T]). | [Lesson 03.6](03-ownership-and-borrowing/06-slices-and-string-types.md) |
| slice / scope (systemd) | systemd names for cgroups: a slice groups services or users, a scope holds processes started outside systemd. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| slice pattern | Pattern for arrays/slices such as [first, .., last] or [first, rest @ ..]. | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| sliding window | A block of neighbouring items described by a left and a right index; the right index moves forward to add an item, the left index moves forward to drop one. | [Lesson 07.15](07-algorithms-and-data-structures/15-two-pointers-and-sliding-window.md) |
| slotted page | A page layout with an array of offsets at the start that point to variable-size records stored from the end of the page. | [Lesson 17.6](17-storage-and-databases/06-b-trees.md) |
| slow-burn alert | An SLO alert for a moderate burn rate over long windows, for example 6 over 6 hours and 30 minutes; it creates a ticket instead of a page. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| slowloris | An attack that keeps many connections open by sending headers very slowly; a header read timeout stops it. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| smart constructor | The only function that creates a value of a type and checks its invariant, returning a Result or Option. | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| smart pointer | A pointer that also owns the data and has extra abilities (Box, Rc, RefCell). | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| smoke / stress / soak test | Load tests with a little load (does it work?), beyond the expected load (where is the limit?), and normal load for hours (slow leaks). | [Lesson 19.2](19-performance-and-profiling/02-load-testing.md) |
| smoke test | A quick check that something basic works, for example that a service is active and logs a line. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| snake_case | Naming style: lowercase words separated by underscores. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| snapshot | A picture of all tracked project files at one moment. A commit is a snapshot. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| snapshot (database) | A frozen picture of the committed data at one moment; a transaction that reads from it does not see later commits. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| snapshot test | A test that produces some output and compares it with the saved snapshot; any difference fails the test. | [Lesson 09.5](09-testing-debugging-benchmarking/05-snapshot-testing.md) |
| SO_REUSEPORT | A socket option that lets several sockets listen on the same port; on Linux the kernel spreads new connections over them. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| socket | The handle a program uses to send and receive data over the network (TcpListener, TcpStream). | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| socket option | A setting on a socket, read and written with getsockopt and setsockopt, for example its buffer size. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| softfloat | A target variant that never uses the floating-point and SIMD registers; kernels use it. | [Lesson 22.2](22-specializations/02-os-track-a-freestanding-binary.md) |
| software / hardware event | An event counted by the kernel (works everywhere) or by the CPU's PMU (often missing in virtual machines). | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| sorted run | A sequence of key-value pairs in key order, such as the contents of one SSTable. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| sorting | Putting the items of a list in order, for example numbers from small to large. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| sound / unsound | A safe function is sound if no input can cause undefined behaviour; if some input can, it is unsound, which is a bug. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| soundness | A safe API is sound if no safe code can use it to cause UB. One that can be misused from safe code is unsound. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| source / transform / sink | The three kinds of pipeline components: produce events, change events, send events away. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| source code | The text of a program written by a person. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| source of truth | The one place where the desired state is kept, usually a Git repository. The running system is always changed to match it, not the other way round. | [Lesson 21.13](21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) |
| space amplification | Using more disk space than the live data needs, because of old versions and tombstones. | [Lesson 17.8](17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| space complexity | How the extra memory an algorithm needs grows when the input size grows. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| spaced repetition | Reviewing after growing gaps (1, 3, 7, 30 days) so material stays in long-term memory. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| span | The part of the code a compiler message points to, shown with ^^^^. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| span (macro) | The place in source code where a token came from; errors use spans to point at the right line. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| sparse file | A file with holes that were never written and use no disk blocks. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| sparse index | An index with one entry per block of records instead of one per record, so that it stays small. | [Lesson 17.7](17-storage-and-databases/07-lsm-trees.md) |
| spawn token | The value an Embassy task function returns; passing it to the spawner starts the task. | [Lesson 22.12](22-specializations/12-embedded-track-async-with-embassy.md) |
| spawn_blocking | Tokio function that runs blocking or heavy CPU work on a separate thread pool. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| specialization | One area of engineering that you know much deeper than the rest, for example kernels or databases. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| specification | A precise description of a format or protocol; code must produce and accept exactly what it says. | [Lesson 15.11](15-networking/11-project-dns-resolver.md) |
| speed-up | Old time divided by new time; 6.8 s to 0.68 s is a speed-up of 10. | [Lesson 19.10](19-performance-and-profiling/10-project-make-a-slow-program-fast.md) |
| spike | A quick experiment, written to learn how something works and then thrown away. | [Lesson 09.6](09-testing-debugging-benchmarking/06-tdd-in-practice.md) |
| spin lock | A lock that waits by checking an atomic flag again and again instead of sleeping; it never allocates, so allocators can use it. | [Lesson 12.14](12-computer-systems/14-project-your-own-allocator.md) |
| spin_loop | std::hint::spin_loop: a hint to the CPU that the thread is busy-waiting in a loop. | [Lesson 14.7](14-concurrency-and-async/07-building-a-spin-lock.md) |
| split | Cutting a full B-tree node into two half-full nodes and moving its middle key up into the parent. | [Lesson 17.6](17-storage-and-databases/06-b-trees.md) |
| split brain | Two nodes both believe they are the leader and both accept writes, so the data diverges. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| split vote | An election in which no candidate gets a majority because several candidates divided the votes; a new term follows. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| split_for_impl | syn method that splits generics into the three pieces an impl block needs. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| splitting / coalescing | Cutting a big free block into the needed part and a free rest / joining neighbouring free blocks into one. | [Lesson 12.13](12-computer-systems/13-how-memory-allocators-work.md) |
| spoofing | Pretending to be someone or something else. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| spurious failure | compare_exchange_weak may fail although the value was equal; inside a retry loop this only costs one more round. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| spurious wake-up | A thread returns from wait although nobody notified it. It is allowed, so you always wait in a loop. | [Lesson 14.3](14-concurrency-and-async/03-condition-variables.md) |
| spy | A test double that works normally and also records how it was called, so the test can inspect the record. | [Lesson 09.3](09-testing-debugging-benchmarking/03-test-doubles.md) |
| SQL dialect | One variant of SQL, such as PostgreSQL, MySQL or SQLite, with slightly different syntax. | [Lesson 22.15](22-specializations/15-databases-track-parsing-sql.md) |
| SQL injection | Injection into a SQL command; the attacker can read, change or delete data. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| SQLite | A small database engine that lives inside your program and stores data in a file or in memory. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| sqlx | An async Rust crate for SQL databases: you write plain SQL, it sends it with parameters and turns rows into Rust values. | [Lesson 16.6](16-backend-services/06-databases-with-sqlx.md) |
| #[sqlx::test] | An sqlx attribute that creates a fresh database for one test, runs the migrations and passes a pool to the test. | [Lesson 16.14](16-backend-services/14-testing-a-backend-service.md) |
| squash | Combine several commits into one. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| ssh-agent | A helper program that holds unlocked SSH keys in memory. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| stable sort | A sort in which items that compare as equal keep their original relative order. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| stack | A region of memory for local variables that works like a pile of plates (last in, first out). Fast but small. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| stack (data structure) | A pile where items are added (push) and removed (pop) only at the top (last in, first out). Not the memory stack. | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| stack frame | The block of the stack that belongs to one function call and holds its local variables. | [Lesson 03.1](03-ownership-and-borrowing/01-memory-stack-and-heap.md) |
| stack machine | A machine whose instructions take operands from a stack and push results onto it. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| stack overflow | A crash when the stack runs out of room, e.g. from endless recursion. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| stack pointer | The register holding the address of the top of the stack: sp on arm64, rsp on x86-64. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| Stacked Borrows / Tree Borrows | Two experimental rule sets Miri uses to decide which pointer uses are allowed (aliasing models). | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| stage | A big group of phases with one common goal (5 stages). | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| stage (verb) | To put a change into the staging area with git add. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| staging | An environment as close to production as possible, used for the last checks before a release. | [Lesson 21.7](21-cloud-native-infrastructure/07-ci-cd-from-commit-to-production.md) |
| staging area (index) | The place where you collect the changes that go into the next commit. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| stale data | A cached copy that is older than the real data. | [Lesson 16.10](16-backend-services/10-caching-with-redis.md) |
| stale entry | A heap entry that was valid when pushed but was made out of date by a later, shorter result; it is skipped when popped. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| stale read | A read that returns a value that was already overwritten. | [Lesson 18.7](18-distributed-systems/07-consistency-models.md) |
| stall (async runtime) | A moment when the async runtime cannot run ready tasks because a worker thread is busy with a task that does not yield. | [Lesson 19.9](19-performance-and-profiling/09-async-and-io-performance.md) |
| standard deviation (sigma) | How spread out repeated measurements are; a large value means unreliable results. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| standard stream | One of the three channels (standard input, output and error) that every program gets at start. | [Lesson 11.3](11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) |
| starter code | Code that a lesson gives as a starting point; in a project it may contain weak points on purpose. | [Lesson 20.10](20-reliability-and-hardening/10-project-harden-a-service.md) |
| startup probe | Asks whether a program has finished starting; until it succeeds, the other probes wait. | [Lesson 21.5](21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) |
| starvation | One thread waits for a very long time (or forever) because other threads always get the resource first. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| stash | A shelf where unfinished changes are put away (git stash, git stash pop). | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| state | A situation the thing can be in, such as Created, Paid or Shipped. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| state (DP) | The few numbers that completely describe a subproblem; the table has one cell per state. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| state (OpenTofu) | The file terraform.tfstate in which OpenTofu remembers which real objects belong to which resources. It can contain secrets, so it is protected and never committed. | [Lesson 21.8](21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) |
| state explosion | The number of possible schedules grows extremely fast with every extra thread or step. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| state machine | A model of something always in exactly one of a few states, moving between them by rules. The compiler turns every async function into one. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| state verification | Checking the result or the final state after the code ran, for example that a user is in the store. | [Lesson 09.3](09-testing-debugging-benchmarking/03-test-doubles.md) |
| stateful fuzzing | Fuzzing a generated sequence of operations on an object instead of a single function call. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| stateless | Keeping no important data between requests inside the process; whatever must survive is stored in a backing service. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| stateless service | A service that keeps no important data in its own memory, so any instance can answer any request and instances can be restarted freely. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| statement | An instruction that does something but produces no value; usually ends with ;. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| 'static | The lifetime of the whole program run; string literals have it. T: 'static means T holds no borrowed data that could expire. | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| static / dynamic linking | Static linking copies library code into the executable; dynamic linking leaves it in a shared library that is connected at start-up. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| static dispatch | The call target is decided at compile time (generics, impl Trait), with no run-time lookup. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| static library | A .a file with compiled code that the linker copies into the final program; crate-type = ["staticlib"]. | [Lesson 06.10](06-rust-advanced/10-ffi-calling-c.md) |
| static type checker | A compiler pass that gives every expression a type and reports mismatches before the program runs. | [Lesson 22.25](22-specializations/25-compilers-track-project-and-next-steps.md) |
| static_mut_refs | Edition 2024 deny-by-default lint that forbids taking a reference to a static mut variable. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| statically typed | The type of every variable is known at compile time, before the program runs. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| status code | The number in a response telling what happened: 200, 404, 503 ... | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| status code (gRPC) | The kind of error a gRPC call returned, such as NOT_FOUND or INVALID_ARGUMENT. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| status line | The first line of an HTTP response: version, status code and reason phrase, for example HTTP/1.1 404 Not Found. | [Lesson 15.8](15-networking/08-http-from-scratch.md) |
| status subresource | A separate part (/status) of a custom resource that the operator writes to report what it did; users write the spec. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| stdin / stdout / stderr | Standard input, standard output and standard error: the three streams every program has. | [Lesson 08.16](08-software-design/16-project-refactor-a-messy-program.md) |
| steady state | The normal, measurable behaviour of a system that an experiment compares against. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |
| step | One command or one ready-made action inside a job. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| step down | A leader or candidate becomes a follower again because it saw a higher term or a valid leader. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| step over / into / out | Debugger commands: run the line / enter the called function / run until the function returns. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| step over / step into / step out | Debugger commands: run the current line whole, enter the called function, or run until the current function returns. | [Lesson 09.9](09-testing-debugging-benchmarking/09-debugging-with-lldb-and-rustrover.md) |
| sticky bit | On a directory: only the owner of a file may delete it, even if others can write into the directory. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| sticky routing | Sending all requests of one user to the same backend, chosen by hashing a key. | [Lesson 22.20](22-specializations/20-networking-track-production-proxies.md) |
| storage | Memory that keeps data when the power is off (SSD). | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| storage device | Hardware that keeps data when the power is off, such as a hard disk or an SSD. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| storage engine | The part of a database that stores and finds bytes on disk: records, indexes, logs, compaction and recovery. | [Lesson 17.14](17-storage-and-databases/14-project-key-value-store.md) |
| StorageClass | Describes how volumes are created in a cluster. kind has a default class called standard that stores data on the node. | [Lesson 21.13](21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) |
| store buffer | A small queue inside each CPU core where writes wait before they reach the cache that other cores see. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| store buffering (pattern) | Two threads each store to their own variable and then load the other's; with weaker orderings than SeqCst both can read the old value. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| strace | A Linux tool that runs a program and shows or counts every system call it makes. | [Lesson 19.9](19-performance-and-profiling/09-async-and-io-performance.md) |
| strategy | One of several interchangeable ways to do the same job; the pattern lets code swap it without changing its users. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| stream | A flow of bytes a program writes to or reads from. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| stream (JetStream) | A stored, ordered log of all messages whose subjects match a pattern. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| stream (QUIC) | An ordered byte stream inside a QUIC connection; a loss in one stream does not stop the others. | [Lesson 15.13](15-networking/13-quic-and-http3.md) |
| strength reduction | Replacing an expensive operation with a cheaper one that gives the same result, such as x * 8 with a shift. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| stress test | A test that performs thousands of random operations and checks after each one that nothing broke. | [Lesson 12.14](12-computer-systems/14-project-your-own-allocator.md) |
| strict mode | The three settings set -e, set -u and set -o pipefail that make a bash script stop at the first error. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| string | A piece of text in double quotes. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| string literal | Text written in code between double quotes, stored in the program file, type &str. | [Lesson 03.6](03-ownership-and-borrowing/06-slices-and-string-types.md) |
| string slice (&str) | A borrowed view of UTF-8 text: pointer and length. String literals have this type. | [Lesson 03.6](03-ownership-and-borrowing/06-slices-and-string-types.md) |
| stringify! | Built-in macro that turns code into a string literal. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| stringly-typed | Using String or &str for data that has only a few valid values, instead of an enum. | [Lesson 08.10](08-software-design/10-anti-patterns-and-code-smells.md) |
| struct | A user-defined type that groups several named values (fields) into one value. | [Lesson 03.2](03-ownership-and-borrowing/02-ownership-rules.md) |
| struct update syntax | ..other at the end of a struct value takes the remaining fields from another instance (moves non-Copy fields). | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| structured fuzzing | Fuzzing with typed input, such as a struct, built from the fuzzer's bytes, instead of raw bytes. | [Lesson 09.8](09-testing-debugging-benchmarking/08-fuzzing.md) |
| structured log | A log line made of named fields, for example one JSON object per line, so that programs can filter and count it. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| structured logging | Logging in which each event is data with fields, not only text, so tools can filter and count events. | [Lesson 09.10](09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| stub | A test double that returns fixed answers or does nothing. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| stub resolver | The small resolver inside the operating system that asks one recursive resolver and waits for the final answer. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |
| subject (NATS) | The dotted name of a message, such as orders.created; receivers choose messages by subject. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| subnet | A group of addresses that share the same first bits and can reach each other directly, without a router. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| subnet mask | The prefix written as an address, with ones for the prefix bits, for example 255.255.255.0 for /24. | [Lesson 15.2](15-networking/02-ip-addresses-and-routing.md) |
| subnormal number | A tiny float below the smallest normal value, stored without the hidden bit, so values fade to zero gradually. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| subproblem | A smaller version of the same problem, for example fib(3) inside fib(5). | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| subscriber | The part of a program that receives tracing events and spans and prints, stores or counts them. | [Lesson 09.10](09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| subsequence | Some characters of a text in the same order, not necessarily next to each other. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| substitution | Replacing one implementation of a trait by another without the using code noticing. | [Lesson 08.3](08-software-design/03-solid-principles.md) |
| subtree | A node together with everything below it. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| subtype | A type usable where another is expected; a longer lifetime is a subtype of a shorter one. | [Lesson 06.5](06-rust-advanced/05-lifetimes-in-depth.md) |
| sudo | A program that runs one command as another user, usually as root, if you are allowed to. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| suite / component | A suite is a release line of a distribution such as resolute-updates; a component is a part of it such as main or universe. | [Lesson 11.7](11-linux-shell-and-containers/07-package-managers.md) |
| super | In a path: the parent module. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| super loop | Firmware structure with one endless loop that checks and does everything in turn. | [Lesson 22.11](22-specializations/11-embedded-track-gpio-and-embedded-hal.md) |
| superscalar CPU | A CPU core that can start and finish several instructions in the same clock cycle. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |
| supertrait | A trait that requires another trait: trait Greeter: Named. | [Lesson 05.2](05-rust-intermediate/02-traits.md) |
| supervisor | A program that starts another program and restarts it when it exits, such as systemd or Kubernetes. | [Lesson 20.2](20-reliability-and-hardening/02-errors-and-panics-in-production.md) |
| supply chain | All code from other people that ends up inside your program. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| surrogate | A code point from 0xD800 to 0xDFFF reserved for UTF-16 pairs; it is never a character by itself. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| swap | Exchanging the places of two items; the slice method is `swap(i, j)`. | [Lesson 07.8](07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) |
| switch | Move HEAD to another branch and update the files (git switch). | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| symbol (binary) | A function or variable name stored in a binary for debuggers, profilers and crash reports; stripping removes symbols. | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| symbol / symbol table | A symbol is the name and address of a function or static in a binary file; the symbol table lists them all, and nm prints it. | [Lesson 12.12](12-computer-systems/12-linking-and-loading.md) |
| symbolic link | A special file that points to another file or folder. | [Lesson 10.5](10-security-fundamentals/05-injection-attacks.md) |
| symbolic value | A value from kani::any() that stands for every possible value of its type at once. | [Lesson 20.6](20-reliability-and-hardening/06-formal-verification-with-kani.md) |
| symbolication | Turning raw code addresses from a profile into function names. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| symmetric encryption | Encryption where the same secret key encrypts and decrypts. Fast; used for the data itself. | [Lesson 10.10](10-security-fundamentals/10-cryptography-encryption-and-signatures.md) |
| syn | Crate that parses a TokenStream into an AST (LitStr, DeriveInput, ItemFn). | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| synchronizes-with | The link created when an Acquire load reads the value of a Release store; it gives happens-before between two threads. | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| synchronous replication | The leader confirms a write only after followers have it. | [Lesson 18.6](18-distributed-systems/06-replication.md) |
| system call | A request from a program to the kernel, made with one special CPU instruction. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| system call (syscall) | A request from a program to the OS, for example to print or open a file. Each has a cost. | [Lesson 00.3](00-getting-started/03-how-a-computer-runs-a-program.md) |
| system call number | The number that tells the kernel which service is requested; it differs between CPU types. | [Lesson 13.3](13-operating-systems/03-system-calls.md) |
| system design interview | An interview where you design a larger system and discuss its trade-offs, numbers and failures. | [Lesson 24.7](24-career-and-open-source/07-coding-and-system-design-interviews.md) |
| system register | A CPU register for control and status, read with mrs and written with msr. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| system user | A user without a login, with a UID below 1000, created to run one service. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| systemd | The first process (PID 1) on most modern Linux systems; it starts the system, supervises services and collects their logs. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| systems engineer | A programmer who builds the lower layers of software (servers, databases, tools, OS parts) where speed, memory and reliability matter. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |

### T

| Word | What it means | First used in |
|---|---|---|
| 4-tuple | Source address, source port, destination address and destination port: the four values that identify a TCP connection. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| T (type parameter) | Placeholder for any type, as in `Option<T>`. | [Lesson 04.4](04-structs-enums-collections-errors/04-option-and-match.md) |
| t-gmax / t-end | In dhat: the moment the heap was biggest, and the end of the program (blocks alive then were never freed). | [Lesson 19.5](19-performance-and-profiling/05-memory-profiling-and-allocations.md) |
| T-shaped engineer | An engineer with broad basics (the bar of the T) and one deep area (the vertical line). | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| table (bottom-up) | Filling an array of answers from the smallest subproblem to the largest, in an order where everything needed is already filled. | [Lesson 07.16](07-algorithms-and-data-structures/16-dynamic-programming.md) |
| table / row / column | A table holds records as rows; each column is one field with a fixed type. | [Lesson 16.5](16-backend-services/05-sql-and-postgresql-basics.md) |
| table-driven test | One test that loops over a list of (input, expected) cases. | [Lesson 05.6](05-rust-intermediate/06-testing-in-depth.md) |
| tag | The part of an enum value in memory that records the current variant. (Not to be confused with a Git tag.) | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| tag (Git) | A permanent name for one commit, usually marking a release. Annotated tags (git tag -a) also store a message and tagger. | [Lesson 01.9](01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| tag (image) | A label on an image version, written after a colon, such as ubuntu:26.04. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| tail call | A call that is the very last thing a function does; Rust does not promise to optimize it. | [Lesson 07.7](07-algorithms-and-data-structures/07-recursion.md) |
| tail expression | The last expression of a body without ;. It is the return value. | [Lesson 02.4](02-programming-basics/04-functions.md) |
| tail latency | The slowest part of the latency distribution: p99, p99.9 and the maximum. | [Lesson 19.1](19-performance-and-profiling/01-latency-throughput-and-percentiles.md) |
| tailing | Reading a growing file line by line as new lines are written, like tail -f. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| tailoring (resume) | Adjusting your resume to one job description: which projects come first, which words you use. | [Lesson 24.6](24-career-and-open-source/06-resume-and-job-search.md) |
| tampering | Changing data or code without permission. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| tar archive | One file that contains many files and folders with their owners and permissions; tar -xf unpacks it. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| target (Cargo) | One thing Cargo can build from a package: a library, a binary, an example, a test or a benchmark. | [Lesson 08.12](08-software-design/12-project-layout-and-workspaces.md) |
| target (Prometheus) | One address that Prometheus scrapes, for example one pod. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| target (systemd) | A group of units that stands for a state of the system, such as multi-user.target. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| target (tracing) | The module that wrote a tracing event; RUST_LOG directives can select levels per target. | [Lesson 09.10](09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| target CPU | The CPU model the compiler assumes; it decides which optional instructions the code may use. target-cpu=native means the CPU of the build machine. | [Lesson 19.8](19-performance-and-profiling/08-build-settings-for-speed.md) |
| target feature | An optional part of an instruction set, such as neon or avx2, that the compiler may use only when it knows the CPU has it. | [Lesson 19.7](19-performance-and-profiling/07-simd.md) |
| target folder | The folder where Cargo puts all build results; ignored by Git. | [Lesson 00.6](00-getting-started/06-your-first-rust-program.md) |
| target triple | A platform name cpu-vendor-os, e.g. aarch64-apple-darwin. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| task | A future given to the runtime (tokio::spawn) to run on its own. Like a very light thread. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| task (Linux) | The kernel's word for anything it schedules; every thread is a task. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| task granularity | How big one piece of parallel work is: too small and scheduling eats the gain, too big and the balance is bad. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| TaskTracker | A tokio-util type that counts spawned tasks and lets you wait until all of them have finished. | [Lesson 14.14](14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| TCP/IP model | The four-layer description of the internet: link, internet, transport and application. | [Lesson 15.1](15-networking/01-how-the-internet-works.md) |
| TCP_NODELAY | The socket option that turns Nagle's algorithm off, so small writes are sent at once. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| telemetry | The data a program sends about itself: logs, metrics and traces. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| template (Helm) | A manifest with placeholders such as {{ .Values.replicaCount }} that Helm fills in. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| template (Lima) | A ready-made description of a virtual machine, such as ubuntu-lts. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| temporary | A value the compiler creates for part of an expression and drops at the end of the statement. | [Lesson 03.7](03-ownership-and-borrowing/07-fixing-common-ownership-errors.md) |
| temporary (value) | A value created in the middle of an expression and never stored in a variable; dropped at the end of the statement. | [Lesson 08.7](08-software-design/07-raii-and-guards.md) |
| temporary directory | A folder created just for one test in the system's temporary location and removed when the test ends. | [Lesson 09.2](09-testing-debugging-benchmarking/02-testing-command-line-programs.md) |
| temporary file | A file that exists only while a program works; mktemp creates one with an unguessable name. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| term (Raft) | Raft's epoch number. Every message carries it; each term has at most one leader, and a higher term always wins. | [Lesson 18.9](18-distributed-systems/09-consensus-and-raft.md) |
| terminal | A window where you type commands and see text answers. | [Lesson 00.1](00-getting-started/01-how-to-use-this-course.md) |
| terminal vs. pipe | Standard input is a terminal when a person types, not a terminal when fed by a pipe; is_terminal() tells which. | [Lesson 04.11](04-structs-enums-collections-errors/11-project-phone-book.md) |
| test / unit test | Code that runs your code and checks the result. A unit test checks one small part and sits next to the code in #[cfg(test)] mod tests. | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| test and test-and-set lock | A spin lock that waits with plain loads and tries the atomic compare-and-swap only when the lock looks free, so it does not write while waiting. | [Lesson 14.9](14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) |
| test double | A stand-in for a real dependency used in a test. | [Lesson 08.9](08-software-design/09-strategy-and-dependency-injection.md) |
| test double / mock | A fake object used in a test instead of the real one, often recording how it was used. | [Lesson 06.4](06-rust-advanced/04-interior-mutability.md) |
| test fixture | Everything a test needs prepared before it runs: a database, tables, some rows. | [Lesson 16.14](16-backend-services/14-testing-a-backend-service.md) |
| test isolation | Each test has its own data, so tests cannot change each other's results, even in parallel. | [Lesson 16.14](16-backend-services/14-testing-a-backend-service.md) |
| test level | The scope of a test: how much of the program it runs at once (unit, integration, end-to-end). | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| test list | A short list, written before the first test, of the behaviours you want; you take one item at a time. | [Lesson 09.6](09-testing-debugging-benchmarking/06-tdd-in-practice.md) |
| test matrix | A table that maps every requirement and failure mode of a design document to the test that proves it, with its status. | [Lesson 23.8](23-capstone-projects/08-testing-benchmarking-and-hardening-capstones.md) |
| test oracle | A trusted source of correct answers that a test compares your code with, for example the standard library. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| test pyramid | A rule of thumb: many fast unit tests at the bottom, fewer integration tests in the middle, a few end-to-end tests at the top. | [Lesson 09.1](09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| test runner | The program (cargo test) that finds, runs and reports tests. | [Lesson 02.8](02-programming-basics/08-your-first-tests.md) |
| test runner (Cargo runner) | A program that Cargo puts in front of every test or binary it starts, set with target.<triple>.runner, for example sudo. | [Lesson 13.17](13-operating-systems/17-project-mini-container-runtime.md) |
| test timeout | A time limit for a test run; a mutant that makes the program loop for ever is stopped by it. | [Lesson 09.7](09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| test vector | A fixed input with its exact expected bytes, written in a specification and checked by a test. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| test-and-set | An atomic operation that sets a flag and returns its old value in one step, such as swap(true) on an AtomicBool. | [Lesson 14.7](14-concurrency-and-async/07-building-a-spin-lock.md) |
| testcontainers | A library that starts Docker containers from test code and removes them when the test ends. | [Lesson 16.14](16-backend-services/14-testing-a-backend-service.md) |
| thin / fat pointer | A thin pointer is just an address; a fat pointer also carries a length (slices, str) or a vtable pointer (dyn Trait). | [Lesson 12.11](12-computer-systems/11-memory-layout-of-rust-types.md) |
| thread | One line of execution inside a process; threads share the process memory. | [Lesson 05.10](05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| thread pool | A fixed group of worker threads taking jobs from a queue instead of one thread per job. | [Lesson 06.11](06-rust-advanced/11-project-multithreaded-web-server.md) |
| thread-local | A variable with a separate copy in every thread, made with the thread_local! macro. | [Lesson 14.11](14-concurrency-and-async/11-build-a-tiny-async-runtime.md) |
| thread-per-core | A server design with one thread per CPU core, each with its own event loop and listening socket, sharing almost nothing. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| threat | Something bad that could happen to an asset, for example someone reads all user passwords. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| threat model | A short list of what you protect, against whom, and what you do not protect against. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| threat modeling | A structured way to find what can go wrong with a system before it goes wrong, and to decide what to do about it. | [Lesson 10.2](10-security-fundamentals/02-threat-modeling-with-stride.md) |
| three-pass method | Reading a paper three times, each time deeper: the idea, the content, the details. | [Lesson 24.8](24-career-and-open-source/08-keep-learning.md) |
| three-valued logic | Logic with three results: true, false and unknown (NULL); used by SQL conditions. | [Lesson 22.17](22-specializations/17-databases-track-project-and-next-steps.md) |
| three-way handshake | The SYN, SYN+ACK, ACK exchange that opens a TCP connection. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| three-way merge | A merge comparing both branch tips with their common ancestor. | [Lesson 01.6](01-git-and-version-control/06-branches-and-merging.md) |
| three-way partition | A partition into three groups (smaller, equal, bigger), which handles many equal items well. | [Lesson 07.9](07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) |
| throttling (CPU) | Making a cgroup wait because it used up its CPU quota for the current period. | [Lesson 13.16](13-operating-systems/16-cgroups.md) |
| throughput | How much work is done per second, for example megabytes of text per second. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| thundering herd | Many clients acting at exactly the same moment, for example all retrying together after an outage, and overloading the server again. | [Lesson 18.3](18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| tick | One unit of simulated time. A sans-IO node does not read a clock; the caller tells it that one tick passed. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| ticket | A task in the team's tracker that is handled in working hours; the opposite of a page. | [Lesson 21.11](21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) |
| ticket lock | A fair spin lock: each thread takes a number with fetch_add and waits until that number is served. | [Lesson 14.7](14-concurrency-and-async/07-building-a-spin-lock.md) |
| tilde (~) | A short name for your home folder. | [Lesson 01.2](01-git-and-version-control/02-install-and-configure-git.md) |
| time box | A fixed amount of time for a task; when it is over, you stop and decide what to do next. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| time complexity | How the number of steps of an algorithm grows when the input size grows. | [Lesson 07.1](07-algorithms-and-data-structures/01-algorithms-and-big-o.md) |
| Time Machine | The backup program built into macOS. | [Lesson 00.8](00-getting-started/08-study-routine-notes-and-checkpoint.md) |
| time series | One metric name with one exact set of labels, and its values over time. | [Lesson 21.9](21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) |
| time slice | How long a task may run before the scheduler checks whether another should run. | [Lesson 13.9](13-operating-systems/09-threads-and-scheduling.md) |
| TIME-WAIT | The state the side that closed first keeps for a while, so that late segments of the old connection die out. | [Lesson 15.3](15-networking/03-tcp-and-udp.md) |
| timeout | A limit on how long you wait for something. | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| timer (runtime) | A note "wake this waker at this time" that a sleep future registers in the runtime. | [Lesson 14.11](14-concurrency-and-async/11-build-a-tiny-async-runtime.md) |
| timer wheel | A data structure that sorts timers into buckets by time, so millions of timers stay cheap. | [Lesson 14.11](14-concurrency-and-async/11-build-a-tiny-async-runtime.md) |
| timing attack | Learning a secret by measuring how long the program takes to answer. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| TLS passthrough | The proxy forwards encrypted bytes without decrypting, choosing a backend at most by the SNI name. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| TLS termination | The proxy holds the certificate, decrypts the traffic and can look into requests. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| tmpfs | A file system that keeps its files in RAM; its content disappears when it is unmounted. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| token | The smallest piece of source code the compiler understands: a name, number or symbol. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| token bucket | A rate-limit algorithm: requests take tokens from a bucket that refills at a fixed rate up to a capacity. | [Lesson 20.8](20-reliability-and-hardening/08-overload-limits-and-backpressure.md) |
| token tree | One token, or a group of tokens inside matching brackets. | [Lesson 06.7](06-rust-advanced/07-declarative-macros.md) |
| tokenizer (lexer) | The function that cuts a line into tokens, handling spaces and quotes. | [Lesson 13.8](13-operating-systems/08-project-your-own-shell.md) |
| TokenStream | The type for a list of tokens that a procedural macro receives and returns. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| tolerance (absolute / relative) | How big a difference you accept when comparing floats: a fixed amount, or a fraction of the size of the numbers. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| tombstone | A marker left in a slot of an open-addressing table after a removal so that lookups do not stop too early. | [Lesson 07.6](07-algorithms-and-data-structures/06-hash-tables.md) |
| toolchain | The set of programs needed to build Rust code (compiler, standard library, helpers). | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| topic | A named stream of messages in a broker, such as orders. | [Lesson 23.5](23-capstone-projects/05-capstone-message-broker.md) |
| topological order | An order of the vertices of a DAG in which every edge a -> b has a before b. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| topology | The graph of sources, transforms and sinks connected by their inputs; it must have no cycles. | [Lesson 23.6](23-capstone-projects/06-capstone-observability-pipeline.md) |
| torn tail | Incomplete or damaged bytes at the end of a log after a crash in the middle of an append. | [Lesson 17.4](17-storage-and-databases/04-write-ahead-log.md) |
| torn write | A write interrupted by a crash, so that only its first part reached the storage. | [Lesson 17.2](17-storage-and-databases/02-durability-and-fsync.md) |
| total order | An order in which any two different items can be compared. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| total order (total_cmp) | An ordering in which every pair of values can be compared. f64::total_cmp gives one for floats, including NaN and -0.0. | [Lesson 12.3](12-computer-systems/03-floating-point-numbers.md) |
| total time | Samples in which a function was anywhere on the stack, including time in the functions it called. | [Lesson 19.3](19-performance-and-profiling/03-profilers-and-flame-graphs.md) |
| tower Service | The trait from the tower crate for something that takes a request and asynchronously returns a response; an axum Router is one. | [Lesson 16.3](16-backend-services/03-web-services-with-axum.md) |
| tower::Service | The trait behind axum: something that takes a request and returns a future with a response; an axum Router is one. | [Lesson 16.14](16-backend-services/14-testing-a-backend-service.md) |
| toxic | One kind of fault in toxiproxy, such as latency, reset_peer, slicer or timeout. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |
| toxiproxy | A TCP proxy for testing, made by Shopify, controlled through an HTTP API, that adds faults (toxics) to connections. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |
| trace | A table of the values of each name after each step, made by hand. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| trace id / span id | A 16-byte id shared by all spans of one trace / an 8-byte id of one span. | [Lesson 16.12](16-backend-services/12-logs-metrics-and-traces.md) |
| trace sampling | Sending only a share of all traces, for example 10%, to save cost in busy services. | [Lesson 21.10](21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) |
| tracefs | A special file system at /sys/kernel/tracing that lists tracepoints and their formats. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| tracepoint | A fixed, documented hook placed in the kernel code; stable between kernel versions. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| tracing a path (code reading) | Following one behaviour through the code, file by file, from where it starts to where it happens. | [Lesson 24.3](24-career-and-open-source/03-reading-large-codebases.md) |
| track | In Phase 22: a group of four lessons about one specialization; you study one track. | [Lesson 22.1](22-specializations/01-choosing-a-specialization.md) |
| track project | The project in the last lesson of a Phase 22 track: a kernel, an eBPF tool, a firmware, a query engine, a proxy, an interpreter or an operator. | [Lesson 22.30](22-specializations/30-checkpoint-your-specialization.md) |
| tracked / untracked file | A file Git already follows / a new file Git has not been told to follow. | [Lesson 01.3](01-git-and-version-control/03-your-first-repository-and-commits.md) |
| trade-off | Gaining one good thing by paying with another, for example speed for memory. | [Lesson 07.2](07-algorithms-and-data-structures/02-analyzing-time-and-space.md) |
| trait | A set of abilities a type can have (Drop, Copy, Clone, Display). A named list of method signatures that a type can implement. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| trait bound | A rule saying which traits a type parameter must implement, e.g. T: PartialOrd. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| trait object | A value of some type that implements a trait, with the exact type known only at run time; written dyn Trait. | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| transaction | A group of SQL statements the database treats as one unit: either all their changes are kept, or none. | [Lesson 16.7](16-backend-services/07-migrations-and-transactions.md) |
| transactional outbox | Writing outgoing messages into a table in the same local transaction as the data change; a relay sends them later, so nothing is lost. | [Lesson 18.13](18-distributed-systems/13-distributed-transactions-and-sagas.md) |
| transient failure | A failure that goes away by itself after a while, such as a busy server; worth a retry. | [Lesson 18.3](18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| transition | A move from one state to another caused by an event. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| transitive dependency | A dependency of your dependency. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| translation fault | A page fault because no mapping exists for the address. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| transparent huge pages | A Linux feature that maps 2 MiB pages instead of 4 KiB ones automatically for large memory areas. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| transport | The code that moves protocol messages between nodes, for example over TCP with length-prefixed frames. | [Lesson 18.14](18-distributed-systems/14-project-replicated-key-value-store.md) |
| trap | A shell command that runs given code when an event happens, for example when the script ends or gets a signal. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| traversal | Visiting every node of a tree in some order. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| tree | A data structure of nodes connected by links, with one top node (the root) and no cycles. | [Lesson 07.11](07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) |
| tree-walking interpreter | An interpreter that runs a program by visiting the nodes of its syntax tree and computing each node from its children. | [Lesson 22.23](22-specializations/23-compilers-track-an-interpreter.md) |
| Treiber stack | The classic lock-free stack: a linked list whose head pointer is changed with compare-and-swap. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| triage | Sorting new issues: checking that they can be reproduced, adding labels, asking for missing facts. | [Lesson 24.5](24-career-and-open-source/05-working-with-maintainers-and-communities.md) |
| triangulation | Adding a second test with different data so that returning a constant no longer works and the general code must appear. | [Lesson 09.6](09-testing-debugging-benchmarking/06-tdd-in-practice.md) |
| trigger | The event that starts a workflow: a push, a pull request, a schedule or a manual button. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| trim | Remove white space from both ends of text. | [Lesson 02.7](02-programming-basics/07-reading-input-and-parsing.md) |
| troubleshooting | Finding the cause of a failing system step by step, from the symptom to the first thing that is wrong. | [Lesson 11.16](11-linux-shell-and-containers/16-checkpoint-stage-2-complete.md) |
| truncate | Cut off the fractional part without rounding. | [Lesson 02.3](02-programming-basics/03-operators-and-expressions.md) |
| truncation | Making a number narrower by dropping its high bits; `as` does this silently. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| trust boundary | The line where data passes from a place you do not control into code that you do control. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| trust store | The list of root certificates that an operating system or browser trusts. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| trusted publishing | Publishing a crate from a CI workflow that proves its identity, without a long-lived API token. | [Lesson 20.7](20-reliability-and-hardening/07-supply-chain-security-in-depth.md) |
| try_lock | A method that takes a lock only if it is free right now; otherwise it returns an error immediately instead of waiting. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| try_update / update | Methods of atomic integers that contain the CAS loop; they replace the deprecated fetch_update. The closure may run several times. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| trybuild | Crate for compile-fail tests: compares compiler output with saved .stderr files. | [Lesson 06.8](06-rust-advanced/08-procedural-macros.md) |
| TryFrom | The standard trait for a conversion that can fail and returns a Result. | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| tuple | A fixed group of values of possibly different types: (1, "a", 2.5). | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| tuple struct | A struct with unnamed, positional fields: Color(255, 0, 0). | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| tuple variant / struct variant / unit variant | Variants with unnamed payload, named-field payload, or no payload. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| turbofish | The syntax `::<Type>` that names a type parameter explicitly, e.g. `"5".parse::<i32>(`). | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| twelve-factor app | A set of twelve rules for building programs that run as services and are easy to deploy, scale and maintain. | [Lesson 08.15](08-software-design/15-the-twelve-factor-app.md) |
| two pointers | A technique with two indexes into the same data; each step moves one of them and they never move back, so the work is O(n). | [Lesson 07.15](07-algorithms-and-data-structures/15-two-pointers-and-sliding-window.md) |
| two's complement | The way modern CPUs store signed integers: the top bit has a negative place value. To negate a number, invert all bits and add 1. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| type | The kind of a value; decides its size in memory and the allowed operations. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| type alias | A new name for an existing type (type Name = ...). Not a new type. | [Lesson 06.2](06-rust-advanced/02-advanced-traits-and-types.md) |
| type annotation | A type written by the programmer, e.g. let x: u8 = 5. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| type coercion | Converting a value to another type so that two values can be compared, for example Int 7 to Float 7.0. | [Lesson 22.17](22-specializations/17-databases-track-project-and-next-steps.md) |
| type inference | The compiler works out types by itself from how values are used. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| type parameter | The name of the blank in angle brackets, such as T in fn `largest<T>` or i32 in `Vec<i32>`. | [Lesson 04.7](04-structs-enums-collections-errors/07-vectors.md) |
| typestate | A pattern where the state is part of the type of a value, so operations not allowed in a state do not exist for that type. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| typosquatting | Publishing a malicious crate with a name close to a popular one and waiting for typing mistakes. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |

### U

| Word | What it means | First used in |
|---|---|---|
| Ubuntu | A popular Linux distribution; this course uses Ubuntu 26.04 LTS. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| umask | A mask that removes permission bits from every new file and directory. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| unary RPC | An RPC with one request and one response. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| unbound variable | A variable that was never given a value; with set -u using it is an error. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| undirected edge | An edge that works in both directions, like a two-way road. | [Lesson 07.19](07-algorithms-and-data-structures/19-project-route-finder.md) |
| undirected graph | A graph whose edges work in both directions. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| undo information | What recovery needs to take back an uncommitted change after a crash: the old value. | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| Unicode | The world standard that gives a number (code point) to every character of every language. | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| Unicode scalar value | A code point from 0 to 0x10FFFF that is not a surrogate; a Rust char is exactly one such value. | [Lesson 12.5](12-computer-systems/05-text-encoding-ascii-and-utf8.md) |
| unified diff | The standard diff format with - and + lines and @@ markers. | [Lesson 01.4](01-git-and-version-control/04-viewing-history-and-changes.md) |
| union | A type whose fields share the same memory. Reading a field is unsafe. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| unique violation | The database error raised when an INSERT or UPDATE would create a duplicate value in a UNIQUE column. | [Lesson 16.8](16-backend-services/08-authentication-in-a-web-service.md) |
| unit | Anything systemd manages, described by a unit file: a service, a timer, a target and so on. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| unit (Morse) | The length of one dot; all other Morse timings are multiples of it. | [Lesson 22.13](22-specializations/13-embedded-track-project-and-next-steps.md) |
| unit file | A small text file in INI format that tells systemd what to run and how. | [Lesson 11.6](11-linux-shell-and-containers/06-services-with-systemd.md) |
| unit type () | The empty tuple, meaning "no value". | [Lesson 02.2](02-programming-basics/02-data-types.md) |
| unit-like struct | A struct with no fields. It takes zero bytes. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| Unix epoch | The moment 1970-01-01 00:00:00 UTC, from which wall-clock time is often counted in seconds. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| Unix time | A moment written as the number of seconds since 1970-01-01 00:00 UTC. | [Lesson 09.3](09-testing-debugging-benchmarking/03-test-doubles.md) |
| unlink | The system call behind deleting a file: it removes one directory entry. | [Lesson 13.12](13-operating-systems/12-file-systems-inside.md) |
| unmaintained | A crate whose authors stopped working on it; it may keep working but gets no security fixes. | [Lesson 10.13](10-security-fundamentals/13-supply-chain-security.md) |
| unmerged path | A file with an unresolved conflict. | [Lesson 01.7](01-git-and-version-control/07-merge-conflicts.md) |
| Unpin | An auto trait meaning that moving the value is safe even when it is pinned; compiler-made futures do not have it. | [Lesson 14.12](14-concurrency-and-async/12-pin-and-self-referential-futures.md) |
| unrepresentable state | A combination of values that the types make impossible to write. | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| unsafe | A keyword by which the programmer, not the compiler, guarantees Rust's rules hold. It allows five extra abilities. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| unsafe impl | Implementing a trait such as Sync whose rules the compiler cannot check; you promise that the rules hold. | [Lesson 14.7](14-concurrency-and-async/07-building-a-spin-lock.md) |
| unsafe_op_in_unsafe_fn | Edition 2024 lint: the body of an unsafe fn is not an implicit unsafe block; unsafe operations inside need their own unsafe { }. | [Lesson 06.6](06-rust-advanced/06-unsafe-rust-and-miri.md) |
| `UnsafeCell<T>` | The low-level building block under all cell types; the only legal way to mutate data behind &. | [Lesson 06.4](06-rust-advanced/04-interior-mutability.md) |
| unshare | The system call and command that move a process into new namespaces. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| unsized coercion | Automatic conversion such as `Box<Circle>` to Box&lt;dyn Shape&gt;, where the compiler forgets the exact type. | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| unstage | Take a change out of the staging area but keep it in the file. | [Lesson 01.5](01-git-and-version-control/05-undoing-changes-safely.md) |
| Unstructured | The type from the arbitrary crate that wraps the fuzzer's raw bytes and hands them out piece by piece to build values. | [Lesson 20.3](20-reliability-and-hardening/03-fuzzing-in-depth.md) |
| unwinding | What a panic does by default: walk back out of the running functions, freeing their values, then end the thread. | [Lesson 04.10](04-structs-enums-collections-errors/10-error-handling.md) |
| unwinding (stack) | Walking the stack from the current function back to main to find the call chain. | [Lesson 19.4](19-performance-and-profiling/04-profiling-on-linux-with-perf.md) |
| unwrap / expect | Return the inner value of Some/Ok, or panic (expect with a custom message). | [Lesson 04.4](04-structs-enums-collections-errors/04-option-and-match.md) |
| up-to-date log | A log whose last entry has a higher term, or the same term and at least the same index. Raft nodes vote only for candidates with an up-to-date log. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| upper bound | The index of the first item that is bigger than the target. | [Lesson 07.10](07-algorithms-and-data-structures/10-binary-search.md) |
| UpperCamelCase | Naming style for types: each word capitalised, no underscores. | [Lesson 04.1](04-structs-enums-collections-errors/01-structs.md) |
| upstream | The remote branch a local branch is linked to; used by plain git push and git pull. | [Lesson 01.8](01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| upstream / backend | A server behind a proxy that really answers the request. | [Lesson 23.4](23-capstone-projects/04-capstone-http-reverse-proxy.md) |
| upstream / downstream | Directions of traffic through a proxy: upstream from the client to the server, downstream from the server back to the client. | [Lesson 20.5](20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) |
| URL shortener | A service that turns a long web address into a short one and redirects visitors of the short one to the long one. | [Lesson 16.15](16-backend-services/15-project-url-shortener-service.md) |
| usage message | The short help text a script prints when it is called wrongly or with -h. | [Lesson 11.9](11-linux-shell-and-containers/09-robust-bash-scripts.md) |
| use | Declaration that brings a path into scope under a short name (as, {a, b}, self). | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| use after free | Using memory after it was released. | [Lesson 10.12](10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) |
| use case | One thing a user wants to do with the program, such as adding a task; it uses the domain to do it. | [Lesson 08.14](08-software-design/14-layers-and-hexagonal-architecture.md) |
| use-after-free | A bug (and security hole) of using memory after it was freed. Prevented by lifetimes. | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| user | An account that owns files and runs programs; every process runs as exactly one user. | [Lesson 11.4](11-linux-shell-and-containers/04-users-groups-and-permissions.md) |
| user enumeration | Finding out which user names exist from small differences in the answers. | [Lesson 10.6](10-security-fundamentals/06-authentication-and-password-hashing.md) |
| user memory / kernel memory | Memory of a process / memory of the kernel itself; eBPF reads them with different helpers. | [Lesson 22.8](22-specializations/08-ebpf-track-maps-and-tracing.md) |
| user namespace | A namespace that maps user IDs, so a process can be root inside while being an ordinary user outside. | [Lesson 13.15](13-operating-systems/15-namespaces.md) |
| user space | Where normal programs run, with limited power; they must ask the kernel for anything privileged. | [Lesson 13.1](13-operating-systems/01-what-an-operating-system-does.md) |
| user_data | A 64-bit number attached to an io_uring submission and copied into its completion to match them. | [Lesson 13.14](13-operating-systems/14-io-uring.md) |

### V

| Word | What it means | First used in |
|---|---|---|
| VACUUM / autovacuum | The PostgreSQL command and background worker that remove dead row versions and make their space reusable. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| validate vs parse | Validate checks a value and keeps its type; parse checks it and returns a new, stricter type that proves the check. | [Lesson 08.5](08-software-design/05-type-driven-design.md) |
| validation | Checking that input has the right form and meaning, and rejecting it when it does not. | [Lesson 10.4](10-security-fundamentals/04-input-validation.md) |
| validity bitmap | One bit per row in an Arrow array that says whether the value is present or NULL. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| value | A piece of data such as 10 or "hello". In a map, the data stored under a key. | [Lesson 02.1](02-programming-basics/01-variables-and-mutability.md) |
| values (Helm) | The settings of a chart, overridable at install time with --set or -f. | [Lesson 21.6](21-cloud-native-infrastructure/06-helm-and-kustomize.md) |
| variable | A named place that holds a value; in Rust it is created with let. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| variance | How a subtype relation passes through a type: same direction, not at all, or reversed. | [Lesson 06.5](06-rust-advanced/05-lifetimes-in-depth.md) |
| variant | One choice of an enum. | [Lesson 04.3](04-structs-enums-collections-errors/03-enums.md) |
| varint | A way to encode an integer in a variable number of bytes: small numbers take one byte, bigger ones take more. | [Lesson 16.9](16-backend-services/09-grpc-with-tonic.md) |
| vault | A file that stores named secrets in encrypted form. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| vDSO | A tiny library the kernel maps into every process so that calls like reading the clock need no real system call. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| vector | A growable list of values of the same type stored one after another `(Vec<T>`). | [Lesson 04.7](04-structs-enums-collections-errors/07-vectors.md) |
| vector clock | A logical clock with one counter per node in every timestamp; it can tell concurrent events from ordered ones. | [Lesson 18.5](18-distributed-systems/05-logical-clocks.md) |
| vector register | A wide register (v0–v31 on arm64, 128 bits) that holds several numbers for SIMD instructions. | [Lesson 12.7](12-computer-systems/07-reading-assembly.md) |
| vector table | A table of small code blocks that the CPU jumps into when an exception happens. | [Lesson 22.4](22-specializations/04-os-track-interrupts-and-paging.md) |
| vectorisation | Rewriting a loop so that SIMD instructions process several elements at once. | [Lesson 12.8](12-computer-systems/08-from-rust-to-machine-code.md) |
| vectorized execution | Query execution where operators process batches of many values at once. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| verb (RBAC) | An action in an RBAC rule, such as get, list, watch, create, patch or delete. | [Lesson 22.28](22-specializations/28-cloud-native-track-testing-and-running-operators.md) |
| verifier | The kernel part that checks every eBPF program before it may run. | [Lesson 22.6](22-specializations/06-ebpf-track-what-ebpf-is.md) |
| version | One state of your files at one moment in time. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| version control | Recording every version of your files so you can see what changed, who changed it and why, and go back. | [Lesson 00.2](00-getting-started/02-what-is-programming.md) |
| version requirement | The rule after the crate name in Cargo.toml (e.g. "0.15") saying which versions are allowed (caret rule by default). | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| vertex | One thing (node) in a graph. | [Lesson 07.13](07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| virtual address | The address a program uses in pointers; it is translated to a physical address on every access. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| virtual file system | Files that do not live on a disk but are made by the kernel when read, such as /proc and /dev/null. | [Lesson 11.2](11-linux-shell-and-containers/02-the-linux-file-system.md) |
| virtual machine | A whole computer simulated by a program; the course VM is called dev. | [Lesson 11.1](11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) |
| virtual manifest | A root Cargo.toml that has a [workspace] table but no [package], so the root itself builds nothing. | [Lesson 08.12](08-software-design/12-project-layout-and-workspaces.md) |
| virtual memory | A system where each process uses its own virtual addresses, which the CPU and kernel translate to real RAM. | [Lesson 13.10](13-operating-systems/10-virtual-memory.md) |
| virtual node | One of many points that one real node owns on a consistent-hashing ring; more points give a more even share. | [Lesson 18.8](18-distributed-systems/08-partitioning-and-consistent-hashing.md) |
| visibility rule | The rule that decides whether a row version exists for a given snapshot. | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| volatile access | A memory read or write that the compiler must do exactly as written; needed for devices. | [Lesson 22.3](22-specializations/03-os-track-booting-a-kernel-in-qemu.md) |
| Volcano model | Query execution where each operator's next() pulls one row from its child. | [Lesson 22.14](22-specializations/14-databases-track-how-query-engines-work.md) |
| Volcano model (iterator model) | An executor design where every node has one operation, "give me the next row", and pulls rows from its children. | [Lesson 17.13](17-storage-and-databases/13-how-a-sql-query-runs.md) |
| volume | Storage managed by Docker that lives outside the container and survives its removal. | [Lesson 11.12](11-linux-shell-and-containers/12-containers-and-docker-basics.md) |
| vote | A node's answer to a vote request: granted or not. Each node gives at most one vote per term. | [Lesson 18.10](18-distributed-systems/10-raft-leader-election.md) |
| vtable | A table of function addresses (plus drop, size, alignment) for one type and one trait, used by trait objects. | [Lesson 06.1](06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |
| vulnerability | A weakness in a system (a bug, a bad setting, a missing check) that an attacker can use. | [Lesson 10.1](10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |

### W

| Word | What it means | First used in |
|---|---|---|
| wait / waitpid | System calls with which a parent waits for a child to end and collects its exit status. | [Lesson 13.5](13-operating-systems/05-fork-exec-and-wait.md) |
| wait-free | A progress guarantee: every thread finishes its operation in a bounded number of its own steps. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |
| Wake trait | A standard trait: implement wake for your type in an Arc, and Waker::from turns it into a Waker without unsafe code. | [Lesson 14.10](14-concurrency-and-async/10-how-futures-work.md) |
| waker | A handle a future gives to the runtime to say "poll me again". | [Lesson 06.9](06-rust-advanced/09-async-await-introduction.md) |
| walking skeleton | The smallest version of a whole system that builds, runs and is tested end to end; it does almost nothing, but every part is connected. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| wall clock | The clock that tells the time of day and date (SystemTime in Rust); it can be changed and can jump. | [Lesson 18.4](18-distributed-systems/04-time-and-clocks.md) |
| wall-clock time / latency / throughput | Elapsed time as on a wall clock / time of one operation / operations per unit of time. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| warm-up | Runs made and thrown away before measuring so that caches and CPU speed settle. | [Lesson 09.11](09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) |
| warm-up / noise / outlier | Runs before measuring / random variation between measurements / a measurement far from the others. | [Lesson 05.12](05-rust-intermediate/12-benchmarking-basics.md) |
| warning | A compiler message about code that compiles but looks suspicious. | [Lesson 00.7](00-getting-started/07-reading-errors-and-getting-help.md) |
| Wasmtime | A WebAssembly runtime written in Rust, used as a command-line tool and as a library. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| watch | Follow a variable or expression while stepping. | [Lesson 02.9](02-programming-basics/09-debugging-basics.md) |
| watch / watcher | A long-running API request that streams every change of some objects; watcher is the kube-runtime helper for it. | [Lesson 22.27](22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) |
| watch channel | A channel that keeps only the latest value; receivers are told when it changes. | [Lesson 14.13](14-concurrency-and-async/13-tokio-tasks-and-channels.md) |
| watchdog | A small piece of code that watches for progress and reports when there is none, for example to detect a deadlock. | [Lesson 14.1](14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| watchpoint | A marker on a memory location; the program stops when the value stored there changes. | [Lesson 09.9](09-testing-debugging-benchmarking/09-debugging-with-lldb-and-rustrover.md) |
| weak point | A place where a service can crash, hang, be abused or fail without anybody noticing. | [Lesson 20.10](20-reliability-and-hardening/10-project-harden-a-service.md) |
| `Weak<T>` | Non-owning pointer to an Rc value; upgrade() returns `Option<Rc<T>>`. Used for back-links to avoid cycles. | [Lesson 05.9](05-rust-intermediate/09-smart-pointers.md) |
| weakly / strongly ordered | A weakly ordered CPU (arm64) reorders memory operations much more than a strongly ordered one (x86-64). | [Lesson 14.6](14-concurrency-and-async/06-memory-ordering.md) |
| web framework | A library that does the common work of an HTTP server (parsing, routing, building responses) so you only write handlers; axum is one. | [Lesson 16.1](16-backend-services/01-what-a-backend-service-is.md) |
| WebAssembly module | One WebAssembly file with functions, memory, imports and exports. | [Lesson 22.24](22-specializations/24-compilers-track-webassembly.md) |
| weighted graph | A graph whose edges have a number (a weight), for example the length of a road. | [Lesson 07.14](07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) |
| where clause | Trait bounds moved below the function signature for readability. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| while let | Loop that repeats while the pattern keeps matching (e.g. pop()). | [Lesson 04.5](04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) |
| width / precision | Minimum characters of a printed value and digits after the decimal point. | [Lesson 02.6](02-programming-basics/06-printing-formatting-and-comments.md) |
| wildcard | A symbol like \* standing for any characters in a name. In Rust, the pattern _ that matches anything. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| wildcard (NATS) | A special token in a subscription: * matches exactly one token, > matches one or more tokens at the end. | [Lesson 16.11](16-backend-services/11-messaging-with-nats.md) |
| wildcard certificate | A certificate name such as *.example.com that matches exactly one label in front of example.com. | [Lesson 10.11](10-security-fundamentals/11-tls-and-certificates.md) |
| wire format | The exact bytes a protocol sends: field order, sizes and byte order. | [Lesson 15.7](15-networking/07-designing-a-protocol.md) |
| word (machine word) | The natural size of a number for a CPU. On a 64-bit CPU it is 64 bits, the size of usize. | [Lesson 12.1](12-computer-systems/01-bits-bytes-and-number-systems.md) |
| word splitting | The shell cuts an unquoted value into separate words at spaces; the source of many scripting bugs. | [Lesson 11.8](11-linux-shell-and-containers/08-bash-scripting-basics.md) |
| work log | A file where you write, after every session, what you did, how long it took and what is next. | [Lesson 23.1](23-capstone-projects/01-how-capstone-projects-work.md) |
| work stealing | Each worker has its own task queue; an idle worker steals tasks from the other end of a busy worker's queue. | [Lesson 14.4](14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) |
| work-stealing | A scheduling method in which an idle worker thread takes tasks from the queue of a busy one. | [Lesson 22.18](22-specializations/18-networking-track-high-performance-servers.md) |
| worker | A task or thread that takes jobs from a queue and runs them one after another. | [Lesson 14.15](14-concurrency-and-async/15-project-concurrent-job-queue.md) |
| workflow (GitHub Actions) | A GitHub Actions file in .github/workflows/ that says when and what to run. | [Lesson 09.12](09-testing-debugging-benchmarking/12-continuous-integration.md) |
| working directory | The directory where you currently are in the terminal. | [Lesson 00.4](00-getting-started/04-terminal-basics.md) |
| working set | The memory a piece of code touches during some period. Code is fast when its working set fits in a cache. | [Lesson 12.9](12-computer-systems/09-memory-hierarchy-and-caches.md) |
| working tree | The files and folders you see and edit. | [Lesson 01.1](01-git-and-version-control/01-what-is-version-control.md) |
| workspace | A group of Cargo packages in one folder tree built together, sharing one Cargo.lock and target folder. | [Lesson 04.6](04-structs-enums-collections-errors/06-modules-crates-and-packages.md) |
| wrap around | A number goes past its largest value and starts again from the smallest; atomic fetch_add does this silently. | [Lesson 14.5](14-concurrency-and-async/05-atomics.md) |
| wrapper | A small function around another one that makes it easier or safer to use. | [Lesson 13.3](13-operating-systems/03-system-calls.md) |
| wrapper (decorator) | A type that holds another value, offers the same interface and adds behaviour around it. | [Lesson 08.4](08-software-design/04-composition-over-inheritance.md) |
| write amplification | When a device or engine writes more bytes than the program asked for, for example a whole block for one changed byte. | [Lesson 17.1](17-storage-and-databases/01-how-disks-and-files-work.md) |
| write set | The changes a transaction has made but not yet committed, kept aside in memory. | [Lesson 17.10](17-storage-and-databases/10-transactions-and-acid.md) |
| write skew | Two transactions read the same data, decide from it and change different rows; together they break a rule each one kept alone. | [Lesson 17.11](17-storage-and-databases/11-isolation-levels.md) |
| write-up | A longer text about a project, such as a blog post, that tells its story: problem, key decision, hardest bug, numbers. | [Lesson 23.9](23-capstone-projects/09-presenting-your-work.md) |
| writer starvation | Readers keep a read-write lock busy all the time, so a writer waits for a very long time. | [Lesson 14.2](14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) |
| wrong abstraction | A shared piece of code made from things that only look alike and change for different reasons. | [Lesson 08.1](08-software-design/01-good-design-kiss-dry-yagni.md) |

### X

| Word | What it means | First used in |
|---|---|---|
| X-Forwarded-For | An HTTP header in which proxies write the original client's IP address; trust it only from your own proxy. | [Lesson 15.14](15-networking/14-proxies-and-load-balancers.md) |
| Xcode Command Line Tools | Apple's free package with a C compiler, linker, git and other developer tools. | [Lesson 00.5](00-getting-started/05-install-rust-and-rustrover.md) |
| XDP action | The answer an XDP program returns for a packet: XDP_PASS, XDP_DROP, XDP_TX, XDP_REDIRECT or XDP_ABORTED. | [Lesson 22.9](22-specializations/09-ebpf-track-project-and-next-steps.md) |
| xmin / xmax | Hidden columns of every PostgreSQL row version: the transaction ID that created it and the one that deleted or replaced it (0 = nobody). | [Lesson 17.12](17-storage-and-databases/12-mvcc.md) |
| xorshift | A tiny pseudo-random number generator that mixes its state with shifts and XOR; fast and repeatable, but not secure. | [Lesson 12.6](12-computer-systems/06-how-the-cpu-runs-instructions.md) |

### Y

| Word | What it means | First used in |
|---|---|---|
| yanked version | A published version its author marked "do not use for new projects". Existing lock files keep working. | [Lesson 05.8](05-rust-intermediate/08-cargo-dependencies-and-versions.md) |
| yield (async) | To give the thread back to the runtime voluntarily so that other tasks can run, with tokio::task::yield_now().await. | [Lesson 19.9](19-performance-and-profiling/09-async-and-io-performance.md) |
| yield_now | std::thread::yield_now: tells the OS that this thread has nothing to do right now, so another thread may run. | [Lesson 14.8](14-concurrency-and-async/08-lock-free-data-structures.md) |

### Z

| Word | What it means | First used in |
|---|---|---|
| -Zbuild-std | A nightly Cargo option that compiles the standard library from source together with your code, using the same compiler flags. | [Lesson 20.4](20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) |
| zero extension | Making an unsigned number wider by filling the new high bits with zeros. | [Lesson 12.2](12-computer-systems/02-integers-in-memory.md) |
| zero-copy | Reading data by pointing into the original instead of copying it into new strings. | [Lesson 05.3](05-rust-intermediate/03-lifetimes.md) |
| zero-copy parsing | Parsing whose results are borrowed views into the input bytes instead of new copies. | [Lesson 22.19](22-specializations/19-networking-track-zero-copy-parsing.md) |
| zero-cost abstraction | A convenient feature (generics, iterators) that adds no run-time cost. | [Lesson 05.1](05-rust-intermediate/01-generics.md) |
| zero-sized type | A type whose values take no memory, such as an empty struct. | [Lesson 08.8](08-software-design/08-state-machines-and-typestate.md) |
| zeroize | To overwrite memory with zeros so that a secret does not stay there after use. | [Lesson 10.8](10-security-fundamentals/08-secrets-management.md) |
| Zeroizing | A wrapper from the zeroize crate that overwrites the value with zeros when it is dropped. | [Lesson 10.14](10-security-fundamentals/14-project-encrypted-secrets-vault.md) |
| zombie | A process that has ended but whose parent has not yet collected its exit status. | [Lesson 13.2](13-operating-systems/02-processes-from-the-inside.md) |
| zombie process | A process that has ended but whose parent has not yet read its exit status. | [Lesson 11.5](11-linux-shell-and-containers/05-processes-and-signals.md) |
| zone | The part of the DNS tree that one organisation manages, such as org or rust-lang.org. | [Lesson 15.10](15-networking/10-dns-how-names-become-addresses.md) |

---

[🏠 Start](START-HERE.md) · [🗺 Course map](README.md) · [✅ Progress](PROGRESS.md) · [📖 Glossary](GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
