<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.14 Checkpoint](../09-testing-debugging-benchmarking/14-checkpoint.md) · [10.1 The security mindset and the CIA triad →](01-security-mindset-and-the-cia-triad.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/README.md) <!-- nav -->

# Phase 10 — Security fundamentals

> **Stage 2 — Engineering fundamentals** · ⏱ about 4 weeks at ~10 hours per week

## What this phase is about

Security is the part of engineering that asks "what if someone is trying to break this?". A program can pass
every test and still leak passwords, let a visitor read another person's data, or crash on one strange request. In
this phase you learn to think like an attacker so that you can defend like an engineer.

You start with the ideas: the CIA triad (confidentiality, integrity, availability), threat modeling with STRIDE (a checklist of six
kinds of threat), and the OWASP (Open Worldwide Application Security Project) Top 10 list of the most common web
risks. Then you meet the real attacks and their cures: checking input, injection (SQL, the Structured Query Language
of databases; shell commands; file paths), safe password storage, sessions and tokens, and the care of secrets.
Every attack is shown on your own small program, with a harmless payload, and then fixed.

The second half is about the tools that protect you: the basics of cryptography (hashes, message authentication,
random numbers, encryption, signatures), TLS (Transport Layer Security, the protection of network connections) and certificates, memory safety and the review of `unsafe` code, and
supply-chain security with `cargo audit` and `cargo deny`. One rule runs through everything: **never invent your own
cryptography**. Use audited crates and their high-level APIs (Application Programming Interfaces: the functions a crate offers).

All experiments are done on your own machine, in your own programs and in throw-away containers. Attacking systems
that you do not own is a crime; the first lesson explains the rules.

## What you will be able to do

- [ ] Explain the CIA triad and find which goal a given bug breaks.
- [ ] Draw a data-flow diagram, apply STRIDE and rank threats by risk.
- [ ] Recognise the OWASP Top 10 categories in real code and review code for them.
- [ ] Validate input with an allow-list and parse it into types that can only hold valid values.
- [ ] Prevent SQL, command and path injection with parameters, no shell, and path checks.
- [ ] Store passwords with Argon2id, and build login, authorization, sessions and tokens correctly.
- [ ] Keep secrets out of code, logs and history, and respond correctly when one leaks.
- [ ] Use hashes, MACs (message authentication codes), encryption and signatures from audited crates, understand TLS, review `unsafe` code, and check your dependencies.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 10.1 | [The security mindset and the CIA triad](01-security-mindset-and-the-cia-triad.md) | 120 min |
| 10.2 | [Threat modeling with STRIDE](02-threat-modeling-with-stride.md) | 135 min |
| 10.3 | [The OWASP Top 10](03-the-owasp-top-10.md) | 135 min |
| 10.4 | [Input validation: never trust input](04-input-validation.md) | 135 min |
| 10.5 | [Injection attacks: SQL, commands and paths](05-injection-attacks.md) | 135 min |
| 10.6 | [Authentication and password hashing](06-authentication-and-password-hashing.md) | 135 min |
| 10.7 | [Authorization, sessions and tokens](07-authorization-sessions-and-tokens.md) | 135 min |
| 10.8 | [Secrets management](08-secrets-management.md) | 135 min |
| 10.9 | [Cryptography I: hashes, MACs and randomness](09-cryptography-hashes-macs-and-randomness.md) | 135 min |
| 10.10 | [Cryptography II: encryption and signatures](10-cryptography-encryption-and-signatures.md) | 135 min |
| 10.11 | [TLS and certificates](11-tls-and-certificates.md) | 120 min |
| 10.12 | [Memory safety and reviewing unsafe code](12-memory-safety-and-reviewing-unsafe.md) | 135 min |
| 10.13 | [Supply-chain security: cargo-audit and cargo-deny](13-supply-chain-security.md) | 120 min |
| 10.14 | [Project: an encrypted secrets vault](14-project-encrypted-secrets-vault.md) | 8 h |
| 10.15 | [Checkpoint](15-checkpoint.md) | 105 min |

## Projects in this phase

- **Small programs in every lesson** in `code/p10-<topic>`: each attack is shown, then fixed, then covered by a test.
- **[Lesson 10.14 — Project: an encrypted secrets vault](14-project-encrypted-secrets-vault.md)** in `code/p10-vault`: a command-line program that stores secrets in an encrypted file, unlocked by a master password. It brings together input validation, password-based keys, authenticated encryption, secret handling and tests.
- **[Lesson 10.15 — Checkpoint](15-checkpoint.md)**: a quiz and practical tasks before the next phase.

## How to study this phase

- **Read with the attacker's eyes.** For every piece of code ask "where does this value come from, and what is the worst thing it could be?"
- **Type the examples and run the attacks yourself.** Seeing `INJECTED` appear in your own output teaches more than reading about it. Then run the fixed version.
- **Do the exercises that ask for tests.** A security fix without a test is easy to undo by accident.
- **Keep to the ground rules.** Only your own programs, only harmless payloads, no real secrets. If you find a real vulnerability in someone else's software, report it privately.
- **Use the glossary.** This phase has many new words; look them up in the [Glossary](../GLOSSARY.md) when one slips away.

---

[🏠 Start](../START-HERE.md) · [← 09.14 Checkpoint](../09-testing-debugging-benchmarking/14-checkpoint.md) · [10.1 The security mindset and the CIA triad →](01-security-mindset-and-the-cia-triad.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
