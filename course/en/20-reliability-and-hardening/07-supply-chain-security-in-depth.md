<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.6 Formal verification basics with Kani](06-formal-verification-with-kani.md) · [📋 Phase 20](README.md) · [20.8 Overload, limits and backpressure →](08-overload-limits-and-backpressure.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/07-supply-chain-security-in-depth.md) <!-- nav -->

# Lesson 20.7 — Supply-chain security in depth

> **Phase 20 — Reliability and hardening** · Lesson 7 of 11 · ⏱ about 135 minutes

## What you will learn

- Which supply-chain questions `cargo audit` and `cargo deny` do not answer, and which tools do.
- How to record that a person really read the code of a dependency, with `cargo vet`.
- How to put the list of dependencies inside the binary (`cargo auditable`) and write an SBOM (`cargo cyclonedx`).
- What a reproducible build is, how to check one, and how to fix the most common difference.
- How to publish checksums and a signature, so users can check that a file really comes from you.

## Before you start

- You finished [Lesson 20.6 — Formal verification basics with Kani](06-formal-verification-with-kani.md).
- You know `cargo tree`, `Cargo.lock`, `cargo audit` and `cargo deny` from [Lesson 10.13 — Supply-chain security](../10-security-fundamentals/13-supply-chain-security.md). This lesson starts where that one ended.
- You know hashes ([Lesson 10.9](../10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md)), signatures ([Lesson 10.10](../10-security-fundamentals/10-cryptography-encryption-and-signatures.md)), Docker ([Lesson 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)) and GitHub Actions ([Lesson 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md)).
- Install three more tools. It compiles for several minutes:

```bash
cargo install --locked cargo-vet cargo-auditable cargo-cyclonedx
cargo vet --version
cargo cyclonedx --version
```

```text
cargo-vet 0.10.3
cargo-cyclonedx-cyclonedx 0.5.10
```

(`cargo auditable --version` prints the version of Cargo itself, because `cargo auditable` passes every command on to Cargo.)

- Create a practice package. Like in Lesson 10.13, it starts with an **old** `regex` that has a known vulnerability:

```bash
cd ~/projects/rust
cargo new code/p20-supply-demo
cd code/p20-supply-demo
cargo add regex@1.5
cargo update -p regex --precise 1.5.4
```

Replace `src/main.rs`:

```rust
use regex::Regex;

fn main() {
    let date = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    for text in ["2026-11-18", "18.11.2026"] {
        println!("{text}: date? {}", date.is_match(text));
    }
}
```

`cargo run` prints:

```text
2026-11-18: date? true
18.11.2026: date? false
```

## New words

| Word | Meaning |
|---|---|
| **audit** (of a crate) | A person reads the code of one crate version and writes down what they checked. |
| **criteria** | In `cargo vet`, the promise an audit makes, for example `safe-to-deploy`: "no serious security hole for software that faces untrusted input". |
| **exemption** | In `cargo vet`, a crate version that is accepted *without* an audit, for now. Exemptions are the backlog. |
| **delta audit** | An audit of the *difference* between two versions, when the older version was already audited. |
| **SBOM** (Software Bill of Materials) | A machine-readable list of every component in a program: names, versions, licenses, hashes. Like the list of ingredients on food. |
| **CycloneDX / SPDX** | The two common standard formats for SBOMs (SPDX: Software Package Data Exchange). |
| **purl** (package URL; URL = Uniform Resource Locator, a web address) | A standard text name for a package, for example `pkg:cargo/regex@1.13.2`. Tools use it to match SBOM entries with advisories. |
| **reproducible build** | A build where the same source, tools and settings always produce a byte-for-byte identical file. |
| **checksum** | A hash of a file published next to it, so a download can be checked. Here: SHA-256 (Secure Hash Algorithm, 256 bits). |
| **provenance** | Signed information about *how* a file was built: from which repository, which commit, by which build system. |
| **SLSA** (Supply-chain Levels for Software Artifacts) | A framework that describes levels of trust in how software is built, from "no guarantees" to "built on a hardened system with signed provenance". |
| **artifact** | A file that a build produces and you ship: a binary, an archive, an SBOM. |

## 1. Four questions that are still open

After Lesson 10.13 your CI (Continuous Integration: the server that runs your checks on every push) knows that no *known* vulnerability is in `Cargo.lock` and that licenses and sources follow your rules. Four questions are still open:

| Question | Why it matters | Tool in this lesson |
|---|---|---|
| Has a person read the code we depend on? | A new malicious crate has no advisory yet. | `cargo vet` |
| What exactly is inside the binary we shipped last year? | The lock file of an old release may be lost; the binary is still running on servers. | `cargo auditable`, SBOM |
| Was this binary really built from this source? | A compromised build machine can add code that is not in Git. | reproducible builds |
| Does this file really come from us? | A download mirror or a man-in-the-middle can replace it. | checksums, signatures, provenance |

You will answer them in a different order: first what is inside a binary, because the demo has a vulnerable `regex` to find.

## 2. What is inside the binary: `cargo auditable`

`cargo audit` reads `Cargo.lock`. But on a server you have only the binary. Build the demo the normal way and ask `cargo audit` to look at the **binary**:

```bash
cargo build --release
cargo audit bin target/release/p20-supply-demo
```

```text
warning: target/release/p20-supply-demo was not built with 'cargo auditable', the report will be incomplete (4 dependencies recovered)
Crate:     regex
Version:   1.5.4
Title:     Regexes with large repetitions on empty sub-expressions take a very long time to parse
ID:        RUSTSEC-2022-0013
Solution:  Upgrade to >=1.5.5 (patched)

error: 1 vulnerability found in target/release/p20-supply-demo
```

(Shortened: the lines where the advisory database is loaded, and some fields of the advisory, are left out.) The tool had to **guess** the dependencies from text left in the binary, such as file paths in panic messages. It found 4 crates; here that happens to be all of them, but a crate that leaves no such text in the binary stays invisible, so the report may be incomplete. Now build with `cargo auditable` (the output is shortened):

```bash
cargo auditable build --release
cargo audit bin target/release/p20-supply-demo
```

```text
       Found 'cargo auditable' data in target/release/p20-supply-demo (5 dependencies)
Crate:     regex
Version:   1.5.4
...
error: 1 vulnerability found in target/release/p20-supply-demo
```

**What happens:**

1. `cargo auditable build` runs a normal `cargo build`, but adds one small extra: the dependency tree (names, versions, sources) as compressed JSON (JavaScript Object Notation, a text data format) in a special section of the binary. The code does not change and does not run any differently.
2. `cargo audit bin` finds that section and checks every listed crate against the RustSec database, exactly like it checks `Cargo.lock`. Now the list is guaranteed to be complete: 5 packages (the demo and its 4 dependencies).
3. The exit code is `1` because of the vulnerability, so a script can use it.

Now fix the problem, rebuild and check again:

```bash
cargo update -p regex
cargo auditable build --release
cargo audit bin target/release/p20-supply-demo
```

```text
       Found 'cargo auditable' data in target/release/p20-supply-demo (6 dependencies)
```

No findings and exit code `0`. (The new `regex` has one more dependency, `regex-automata`, so there are 6 packages.)

> 🔒 **Security:** advisories are published for old versions every week. With the dependency list inside each binary, you can scan the binaries that **run in production today**, even years after the build, even if nobody knows which commit they came from. Some Linux distributions build all their Rust packages with `cargo auditable` for this reason.

> ⚡ **Performance:** the extra data is a few hundred bytes to a few kilobytes, and the program never reads it. There is no runtime cost.

## 3. A bill of materials: SBOM

The data inside the binary is for tools that understand `cargo auditable`. Customers, security teams and scanners for other languages expect a **standard** document: an SBOM. Create one:

```bash
cargo cyclonedx --format json
```

It prints nothing and writes `p20-supply-demo.cdx.json` next to `Cargo.toml`. The file is long (about 250 lines). This is one component from its `components` list, shortened:

```json
{
  "type": "library",
  "name": "regex",
  "version": "1.13.2",
  "scope": "required",
  "hashes": [
    {
      "alg": "SHA-256",
      "content": "9807af0e638c12e753aa1b4e601791bf2a8f71274023c8bbeec12fc149950857"
    }
  ],
  "licenses": [
    {
      "expression": "MIT OR Apache-2.0"
    }
  ],
  "purl": "pkg:cargo/regex@1.13.2"
}
```

**What happens:**

1. `cargo cyclonedx` reads the same information that `cargo tree` shows and writes it in the **CycloneDX** format (the other common format is **SPDX**; many tools read both).
2. Each component has a name, a version, a license expression and the SHA-256 hash of the downloaded crate file (the same checksum that is in `Cargo.lock`).
3. `purl` is a name that works across languages: a scanner that knows nothing about Cargo can still match `pkg:cargo/regex@1.13.2` against a vulnerability database.
4. A `dependencies` section at the end of the file records who depends on whom, so the SBOM keeps the tree, not only a flat list.
5. Dev-dependencies (for tests only) are not listed: they are not part of what you ship.

When a new vulnerability is announced, an organisation with SBOMs for every release can answer "are we affected, and where?" in minutes, by searching the documents. You publish the SBOM next to the release (Section 6), and you create a new one for every release.

> 📏 **Convention:** generate the SBOM in CI from the same commit and lock file as the binary, never by hand. An SBOM that is older than the binary is wrong.

## 4. Has anybody read this code? `cargo vet`

`cargo audit` only knows about problems that somebody already found and reported. A brand-new malicious version has no advisory. The only defence is that a **person** looks at the code before it goes into your product. `cargo vet` does not read code for you; it keeps a record of **who read what**, and fails CI when something unread comes in.

Start it in the demo (with `regex` already updated):

```bash
cargo vet init
cargo vet
```

```text
Vetting Succeeded (5 exempted)
```

`init` created a folder `supply-chain/` with three files:

| File | Content |
|---|---|
| `config.toml` | Settings, and the **exemptions**: one entry for every dependency you have today. |
| `audits.toml` | The audits that your team writes. Empty at first. |
| `imports.lock` | Audits imported from other organisations, locked like `Cargo.lock`. |

One exemption from `config.toml` looks like this: `[[exemptions.regex]]`, then `version = "1.13.2"` and `criteria = "safe-to-deploy"`.

**What happens:**

1. You cannot read every existing dependency on the first day. So `init` accepts everything you have **now** as exemptions, and the check passes.
2. From now on, every **new** crate or new version needs an audit. The exemptions are a backlog that you reduce over time.
3. `criteria = "safe-to-deploy"` is the stronger of the two built-in criteria. It means: "this code will not add a serious security hole to software that handles untrusted input". The weaker one, `safe-to-run`, only says the code does not harm the machine where you run it (good enough for a tool you run on your own computer, not for production code).

Now add a new dependency and run the check again:

```bash
cargo add itoa@1
cargo vet
```

```text
Vetting Failed!

1 unvetted dependencies:
  itoa:1.0.19 missing ["safe-to-deploy"]

recommended audits for safe-to-deploy:
    Command                        Publisher  Used By          Audit Size
    cargo vet inspect itoa 1.0.19  dtolnay    p20-supply-demo  1074 lines
      NOTE: cargo vet import google would reduce this to a 733-line diff

estimated audit backlog: 1074 lines

Use |cargo vet certify| to record the audits.
```

The exit code is not `0`, so CI stops. The tool says how much reading is needed (1,074 lines) and gives a hint: Google publishes its own audits, and they cover an older version of `itoa`.

**Import the audits of a team you trust:**

```bash
cargo vet import google
cargo vet
```

```text
Vetting Failed!
...
    cargo vet diff itoa 1.0.15 1.0.19  dtolnay    p20-supply-demo  9 files changed, 441 insertions(+), 292 deletions(-)

estimated audit backlog: 733 lines
```

Google's auditors already read `itoa` up to version `1.0.15`. You only need to read the **difference** to `1.0.19`. Now you have two honest choices:

1. **Use the audited version.** `cargo update -p itoa --precise 1.0.15`, and `cargo vet` prints `Vetting Succeeded (1 fully audited, 5 exempted)`.
2. **Read the difference and record it.** `cargo vet diff itoa 1.0.15 1.0.19` shows the diff (in the browser, or in the terminal with `--mode local`). Look for new `unsafe` code, a new `build.rs`, new dependencies, file, network or process access. When you are satisfied, record the audit:

```bash
cargo vet certify itoa 1.0.15 1.0.19 --criteria safe-to-deploy
cargo vet
```

`certify` shows the criteria and asks you to confirm them, takes your name from your Git settings, and writes this to `audits.toml`:

```toml
[[audits.itoa]]
who = "Ada Lovelace <ada@example.com>"
criteria = "safe-to-deploy"
delta = "1.0.15 -> 1.0.19"
notes = "Read the whole diff. The unsafe blocks moved into one helper; no build script, no new dependencies, no file or network access."
```

```text
Vetting Succeeded (1 fully audited, 5 exempted)
```

(Add a note with `--notes "..."`; it is the most useful part for the next reader.) The `supply-chain/` folder goes into Git. Then the audit is reviewed in a pull request like code, and other teams can import **your** audits too.

> ⚠️ **Common mistake:** running `cargo vet certify` without reading the code, only to make CI green. The record then says "a person checked this", which is false, and other teams may rely on it. If you have no time, keep an exemption: it is honest.

Remove `itoa` again (`cargo remove itoa`); the demo does not use it.

## 5. Reproducible builds

A binary is **reproducible** if anybody who takes the same source, the same toolchain and the same settings gets a byte-for-byte identical file. Then a second, independent machine can rebuild a release and compare hashes. If an attacker changed the official build machine, the hashes differ, and the attack is visible.

Rust is close to reproducible by default, but small details leak into a binary. The most common one is **file paths**: panic messages contain the path of the source file, and for dependencies that path is inside `CARGO_HOME` (the folder where Cargo keeps downloaded crates, usually `~/.cargo`).

Check it in a clean Linux container. Save this script as `repro-check.sh` in the project folder and make it executable (`chmod +x repro-check.sh`):

```bash
#!/usr/bin/env bash
# Builds the project in different places and prints a short hash of each binary.
set -euo pipefail

build() {
    local dir="$1" cargo_home="$2" flags="$3"
    mkdir -p "$dir"
    cp -r /src/Cargo.toml /src/Cargo.lock /src/src "$dir"/
    cd "$dir"
    CARGO_HOME="$cargo_home" RUSTFLAGS="$flags" cargo build --release --locked --quiet
    echo "$(sha256sum target/release/p20-supply-demo | cut -c1-16)  $dir (CARGO_HOME=$cargo_home)"
}

echo "== same tools, two folders"
build /build/one /usr/local/cargo ""
build /build/two /usr/local/cargo ""
echo "== another CARGO_HOME"
build /build/three /opt/other-cargo ""
echo "== another CARGO_HOME, paths remapped"
build /build/four /usr/local/cargo "--remap-path-prefix=/usr/local/cargo=/cargo"
build /build/five /opt/other-cargo "--remap-path-prefix=/opt/other-cargo=/cargo"
```

Run it in the official Rust image. The project folder is mounted read-only (`:ro`), so the container cannot change it:

```bash
docker run --rm -v "$PWD":/src:ro rust:1.100 /src/repro-check.sh
```

```text
== same tools, two folders
2ed3df37d40119d4  /build/one (CARGO_HOME=/usr/local/cargo)
2ed3df37d40119d4  /build/two (CARGO_HOME=/usr/local/cargo)
== another CARGO_HOME
cfa62cdcf4bef64a  /build/three (CARGO_HOME=/opt/other-cargo)
== another CARGO_HOME, paths remapped
8e54709e3a080b86  /build/four (CARGO_HOME=/usr/local/cargo)
8e54709e3a080b86  /build/five (CARGO_HOME=/opt/other-cargo)
```

**What happens:**

1. `build` copies only the source (`Cargo.toml`, `Cargo.lock`, `src`) into a new folder, builds a release with `--locked` (exactly the versions in the lock file) and prints the first 16 hex digits of the SHA-256 hash.
2. **Two folders, same tools: identical.** The project folder does not leak into a release binary, because Cargo writes your own source paths relative to the project (`src/main.rs`).
3. **Another `CARGO_HOME`: different.** The binary contains texts like `/usr/local/cargo/registry/src/index.crates.io-…/regex-1.13.2/src/…` (you can see them with `strings`). Another download folder gives other bytes.
4. **Remapped: identical again.** `--remap-path-prefix=FROM=TO` tells the compiler to write `TO` wherever a path starts with `FROM`. Both builds now contain `/cargo/registry/…`, so they match each other (but not the first two, which still have the old paths).
5. Your hashes will not be the same as these: they depend on the exact toolchain and the target. That is the point: reproducible means **the same inputs** give the same output.

The inputs you must fix for a reproducible release:

| Input | How to fix it |
|---|---|
| The toolchain version | A `rust-toolchain.toml` file with `channel = "1.100.0"`, or a pinned Docker image tag. |
| The dependency versions | Commit `Cargo.lock`, build with `--locked`. |
| The target and settings | Same `--target`, same profile, same `RUSTFLAGS`. |
| Paths | `--remap-path-prefix` for `CARGO_HOME` and the project folder. |
| Time stamps | Rust adds none itself; for tools that do, set `SOURCE_DATE_EPOCH` (a fixed time, in seconds since 1970). |

Debug builds (`cargo build` without `--release`) are not reproducible across folders: their debug information contains full paths. Compare release builds.

## 6. Checksums and signatures

Users download your binary from a release page or a mirror. Two things protect them:

- A **checksum file** (`SHA256SUMS`) shows that the download is complete and unchanged.
- A **signature** on the checksum file shows that the checksums were published by the owner of a key, not by whoever controls the mirror.

You know signatures from Lesson 10.10. A practical tool that is already on every Mac and Linux machine is `ssh-keygen`: it can sign any file, not only log in to servers. Create a key **only for releases**, inside the project, and keep it out of Git:

```bash
mkdir -p dist keys
echo "keys/" >> .gitignore
cp target/release/p20-supply-demo dist/
ssh-keygen -t ed25519 -N "" -C "p20-supply-demo release key" -f keys/release_key
cd dist
shasum -a 256 p20-supply-demo > SHA256SUMS
ssh-keygen -Y sign -f ../keys/release_key -n file SHA256SUMS
```

```text
Signing file SHA256SUMS
Write signature to SHA256SUMS.sig
```

**What happens:**

1. `ssh-keygen -t ed25519 -f keys/release_key` creates a key pair: the private key `release_key` and the public key `release_key.pub`. `-N ""` means "no passphrase", which is acceptable only for this exercise.
2. `shasum -a 256` (on Linux: `sha256sum`) writes one line per file: the hash and the file name.
3. `ssh-keygen -Y sign -n file` signs `SHA256SUMS` and writes `SHA256SUMS.sig`. `-n file` is a **namespace**: a label inside the signature, so that a signature made for files cannot be reused as, for example, a signature for Git commits.

A user needs your public key, which you publish (on the website, in the README). They write it into an `allowed_signers` file with a name, then check the signature and the checksums:

```bash
echo "release@p20-supply-demo $(cat ../keys/release_key.pub)" > allowed_signers
ssh-keygen -Y verify -f allowed_signers -I release@p20-supply-demo -n file -s SHA256SUMS.sig < SHA256SUMS
shasum -a 256 -c SHA256SUMS
```

```text
Good "file" signature for release@p20-supply-demo with ED25519 key SHA256:h3w2qWMyzIHM51bP6ydIqxdpAhWCs9vuqS8+y8aSNTQ
p20-supply-demo: OK
```

Change one character in `SHA256SUMS` and check the signature again:

```text
Signature verification failed: incorrect signature
Could not verify signature.
```

The exit code is `255`. An attacker who replaces the binary must also change the checksum, and then the signature no longer matches.

> 🔒 **Security:** the private key is now the most valuable secret of the project (Lesson 10.8). In real projects it lives in the CI secret store or in hardware, has a passphrase, and never sits in a developer's project folder. Many projects avoid long-lived keys completely with **keyless** signing (the Sigstore project): the build system proves its identity for a few minutes and gets a short-lived certificate.

**Provenance.** A signature says *who* published a file. **Provenance** says *how* it was built: repository, commit, workflow, build machine, all signed by the build system. The **SLSA** framework describes levels of such guarantees. On GitHub, the action `actions/attest-build-provenance` creates signed provenance for an artifact, and users check it with `gh attestation verify <file> --repo <owner>/<repo>`. crates.io supports **trusted publishing** in the same spirit: a GitHub workflow publishes a crate without any long-lived API (Application Programming Interface) token that could be stolen.

## 7. All of it in one release workflow

The pieces belong in CI, so they happen on every release without anybody remembering them. Save this as `.github/workflows/release.yml`:

```yaml
name: release

on:
  push:
    tags: ["v*"]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write      # lets the job prove its identity for the attestation
      attestations: write  # lets the job store the attestation
    steps:
      - uses: actions/checkout@v7
      - name: Install the tools
        run: cargo install --locked cargo-vet cargo-auditable cargo-cyclonedx
      - name: Every dependency is vetted
        run: cargo vet --locked
      - name: Build with the dependency list inside
        run: cargo auditable build --release --locked
        env:
          RUSTFLAGS: --remap-path-prefix=${{ github.workspace }}=/src --remap-path-prefix=/home/runner/.cargo=/cargo
      - name: Bill of materials
        run: cargo cyclonedx --format json
      - name: Checksums
        run: |
          cp target/release/p20-supply-demo .
          sha256sum p20-supply-demo p20-supply-demo.cdx.json > SHA256SUMS
      - name: Signed provenance
        uses: actions/attest-build-provenance@v4
        with:
          subject-path: p20-supply-demo
      - uses: actions/upload-artifact@v7
        with:
          name: release
          path: |
            p20-supply-demo
            p20-supply-demo.cdx.json
            SHA256SUMS
```

`actionlint .github/workflows/release.yml` prints nothing and exits with `0`. GitHub runners cannot be started from your Mac, so the run itself is not shown here.

**What happens:**

1. The workflow runs when you push a tag such as `v0.2.0` (tags: Lesson 01.9). Only this job gets the extra permissions `id-token: write` (a short-lived identity token for keyless signing) and `attestations: write`.
2. `cargo vet --locked` stops the release if any dependency is unaudited; `--locked` means it does not fetch new imported audits during the build.
3. The build embeds the dependency list and remaps both paths: the project folder (`github.workspace`) and the runner's Cargo folder. The SBOM, the checksums and the provenance all come from the same checkout.

## Common mistakes

**1. An SBOM made once, by hand.** It describes an old state. Generate it in CI for every release.

**2. Certifying without reading.** `cargo vet` records a human promise. A false record is worse than an exemption.

**3. Comparing debug builds** when testing reproducibility. Use release builds with `--locked`, the same toolchain and remapped paths.

**4. Checksums without a signature.** If the attacker controls the download page, they replace the binary **and** `SHA256SUMS`. The checksum only helps against broken downloads; the signature helps against attackers.

**5. The signing key in the repository.** Anybody who can read the repository could sign releases. Keep it in a secret store, or use keyless signing.

## Try it yourself

1. Run `cargo audit bin` on a release binary of one of your earlier services (for example from Phase 16), first built normally, then with `cargo auditable build --release`. How many dependencies does it recover each time?
2. Run `cargo cyclonedx --format json` in the same project and count the components: `jq '.components | length' <name>.cdx.json`.
3. Run `cargo vet init` there and look at how many exemptions you start with. That number is your audit backlog.

## Exercises

### Exercise 1 — Read the SBOM (easy)

Print one line per component of the demo's SBOM: `name@version`, two spaces, and the license expression.

<details>
<summary>Hint</summary>

`jq -r` prints raw text. Inside a string, `\(.name)` inserts a field. The license is in `.licenses[0].expression`.

</details>

<details>
<summary>Answer</summary>

```bash
jq -r '.components[] | "\(.name)@\(.version)  \(.licenses[0].expression)"' p20-supply-demo.cdx.json
```

```text
aho-corasick@1.1.5  Unlicense OR MIT
memchr@2.8.3  Unlicense OR MIT
regex-automata@0.4.19  MIT OR Apache-2.0
regex-syntax@0.8.12  MIT OR Apache-2.0
regex@1.13.2  MIT OR Apache-2.0
```

Why it works: `.components[]` goes through the array one element at a time, and the string template builds one line from each element. The same one-line query works on any CycloneDX SBOM, from any language.

</details>

### Exercise 2 — A changed binary (medium)

In `dist/`, append one byte to the binary (`printf 'x' >> p20-supply-demo`). Then run both checks from Section 6. Which one fails, and why does the other one still pass?

<details>
<summary>Hint</summary>

The signature covers `SHA256SUMS`, not the binary itself.

</details>

<details>
<summary>Answer</summary>

```text
p20-supply-demo: FAILED
shasum: WARNING: 1 computed checksum did NOT match
```

The signature check still prints `Good "file" signature …`.

Why it works: the signature proves that `SHA256SUMS` is exactly what the key owner signed, and it is. The checksum proves that the binary matches the signed hash, and it does not. You need **both** checks: signature first (is the list genuine?), then checksums (do the files match the list?). Rebuild or copy the binary again afterwards.

</details>

### Exercise 3 — An SBOM policy test (hard)

Write an integration test `tests/sbom_policy.rs` that reads `p20-supply-demo.cdx.json` and fails if a component has no SHA-256 hash, has no license expression, or has a name from a banned list (`atty`, `openssl`). Check that it fails when you put `memchr` into the banned list.

<details>
<summary>Hint</summary>

Add `serde_json` as a dev-dependency (`cargo add --dev serde_json@1`) and parse the file into a `serde_json::Value`. `value["components"].as_array()` gives the list; indexing a missing key gives `Value::Null` instead of panicking, so `c["licenses"][0]["expression"].as_str()` is `None` when anything is missing.

</details>

<details>
<summary>Answer</summary>

```rust
//! Checks the bill of materials that `cargo cyclonedx` wrote next to Cargo.toml.

use serde_json::Value;

const BANNED: &[&str] = &["atty", "openssl"];

#[test]
fn every_component_has_a_hash_a_license_and_is_allowed() {
    let path = concat!(env!("CARGO_MANIFEST_DIR"), "/p20-supply-demo.cdx.json");
    let text = std::fs::read_to_string(path).expect("run `cargo cyclonedx --format json` first");
    let sbom: Value = serde_json::from_str(&text).unwrap();

    let components = sbom["components"].as_array().expect("no components list");
    assert!(!components.is_empty(), "the SBOM lists no components");

    for c in components {
        let name = c["name"].as_str().unwrap();
        assert!(!BANNED.contains(&name), "banned crate `{name}` is in the SBOM");

        let has_sha256 = c["hashes"]
            .as_array()
            .is_some_and(|hashes| hashes.iter().any(|h| h["alg"] == "SHA-256"));
        assert!(has_sha256, "`{name}` has no SHA-256 hash");

        let license = c["licenses"][0]["expression"].as_str();
        assert!(license.is_some(), "`{name}` has no license expression");
    }
}
```

`cargo test --test sbom_policy` passes. With `"memchr"` in `BANNED`:

```text
thread 'every_component_has_a_hash_a_license_and_is_allowed' (5994724) panicked at tests/sbom_policy.rs:18:9:
banned crate `memchr` is in the SBOM
```

Why it works: the SBOM is plain JSON, so a normal test can enforce rules on it. `serde_json` does not appear in the SBOM even though the test uses it: dev-dependencies are not part of the shipped program. In CI, run `cargo cyclonedx` before `cargo test`.

</details>

## Quiz

1. `cargo audit` is green. Why can a malicious crate still be in your build?

<details>
<summary>Answer</summary>

`cargo audit` only knows **reported** problems. A new malicious version has no advisory yet. Only a review by people (recorded with `cargo vet`) or a very small dependency list helps against it.

</details>

2. What does `cargo vet init` put into `config.toml`, and why?

<details>
<summary>Answer</summary>

An exemption for every dependency you have now. You cannot audit everything on day one, so today's state is accepted, and every new crate or version needs an audit from now on. The exemptions are a backlog to reduce.

</details>

3. What is the difference between `cargo auditable` and an SBOM?

<details>
<summary>Answer</summary>

`cargo auditable` stores the dependency list **inside** the binary, for Rust-aware tools such as `cargo audit bin`. An SBOM is a separate document in a standard format (CycloneDX, SPDX) that tools for any language can read. Both answer "what is inside this program?".

</details>

4. Two release builds of the same commit on two machines have different hashes. Name two common reasons.

<details>
<summary>Answer</summary>

A different `CARGO_HOME` or project path that ends up in the binary (fix with `--remap-path-prefix`); a different toolchain version (fix with `rust-toolchain.toml` or a pinned image); different dependency versions (commit `Cargo.lock`, build with `--locked`); a different target or different `RUSTFLAGS`.

</details>

5. Why sign `SHA256SUMS` instead of only publishing it?

<details>
<summary>Answer</summary>

Whoever can replace the binary on the download page can also replace the checksum file. A signature can only be made with the private key, so a changed checksum file is detected.

</details>

6. What does provenance add to a signature?

<details>
<summary>Answer</summary>

A signature says who published the file. Provenance says how it was built: from which repository and commit, by which workflow, on which build system, and the build system signs that statement.

</details>

## Summary

- `cargo audit` and `cargo deny` check known problems and rules; they cannot tell whether anybody read the code.
- `cargo vet` records audits by people (yours and imported ones) and fails on new unaudited crates; exemptions are an honest backlog.
- `cargo auditable build` puts the dependency list into the binary, so `cargo audit bin` can scan binaries that are already deployed.
- `cargo cyclonedx` writes an SBOM: a standard list of components, versions, licenses and hashes, made in CI for every release.
- A reproducible build gives identical bytes from identical inputs: pin the toolchain, use `--locked`, remap paths, compare release builds.
- Publish `SHA256SUMS` and a signature over it; check the signature first, then the checksums. Provenance and keyless signing add "how and where was it built".

## Sources and further reading

- [cargo-vet book](https://mozilla.github.io/cargo-vet/) — concepts, criteria, importing audits and every command.
- [cargo-auditable on GitHub](https://github.com/rust-secure-code/cargo-auditable) — how the dependency list is embedded and which tools read it.
- [cargo-cyclonedx on GitHub](https://github.com/CycloneDX/cyclonedx-rust-cargo) — options of `cargo cyclonedx`.
- [CycloneDX](https://cyclonedx.org/) and [SPDX](https://spdx.dev/) — the two SBOM standards.
- [Reproducible Builds](https://reproducible-builds.org/) — why reproducibility matters, and `SOURCE_DATE_EPOCH`.
- [rustc command-line arguments](https://doc.rust-lang.org/rustc/command-line-arguments.html) — `--remap-path-prefix` and the other compiler flags.
- [ssh-keygen manual](https://man.openbsd.org/ssh-keygen) — `-Y sign`, `-Y verify` and the `allowed_signers` format.
- [SLSA](https://slsa.dev/) and [Sigstore](https://www.sigstore.dev/) — build provenance levels and keyless signing.
- [GitHub: artifact attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds) — signed provenance in GitHub Actions.
- [RFC 3691: trusted publishing for crates.io](https://rust-lang.github.io/rfcs/3691-trusted-publishing-cratesio.html) — the design proposal (RFC: Request for Comments) for publishing crates without long-lived tokens.

---

[🏠 Start](../START-HERE.md) · [← 20.6 Formal verification basics with Kani](06-formal-verification-with-kani.md) · [📋 Phase 20](README.md) · [20.8 Overload, limits and backpressure →](08-overload-limits-and-backpressure.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
