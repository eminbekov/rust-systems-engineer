<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.10 Cryptography II: encryption and signatures](10-cryptography-encryption-and-signatures.md) · [📋 Phase 10](README.md) · [10.12 Memory safety and reviewing unsafe code →](12-memory-safety-and-reviewing-unsafe.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/11-tls-and-certificates.md) <!-- nav -->

# Lesson 10.11 — TLS and certificates

> **Phase 10 — Security fundamentals** · Lesson 11 of 15 · ⏱ about 120 minutes

## What you will learn

- What TLS protects (privacy, integrity, identity) and how the handshake uses the tools from Lessons 10.9 and 10.10.
- What a certificate contains and how a chain of trust leads from a website to a root certificate authority.
- How to look at a real TLS connection with `curl -v` and `openssl s_client`.
- How to create a local self-signed certificate, serve it, and see exactly why clients reject it.
- How a Rust client with `rustls` verifies a server, and why you must never turn verification off.

## Before you start

- You finished [Lesson 10.10 — Cryptography II: encryption and signatures](10-cryptography-encryption-and-signatures.md).
- You know what a TCP (Transmission Control Protocol) connection is: a reliable stream of bytes between two programs. TLS runs on top of it.
- Tools: `curl` (already on macOS) and OpenSSL 3. The `openssl` that ships with macOS is an older fork (LibreSSL), so the lesson uses the Homebrew OpenSSL; if you do not have it, `brew install openssl` installs it. The commands below write it as `openssl` after you put it first in `PATH`.

```bash
export PATH="/opt/homebrew/opt/openssl/bin:$PATH"
openssl version
```

```text
OpenSSL 3.6.2 7 Apr 2026 (Library: OpenSSL 3.6.2 7 Apr 2026)
```

- Create the practice package and a folder for local certificates that Git must ignore:

```bash
cd ~/projects/rust
cargo new code/p10-tls
cd code/p10-tls
mkdir certs
echo "certs/" >> .gitignore
cargo add rustls@0.23 --no-default-features --features ring,std,tls12
cargo add webpki-roots@1
```

> 🔒 **Ethics and safety:** you connect only to a public website (reading a public page is fine) and to your own server on `localhost`. You will **not** add the test certificate to the macOS keychain or to any trust store: it is used only through command-line options.

## New words

| Word | Meaning |
|---|---|
| **TLS** (Transport Layer Security) | The protocol that makes a connection private, tamper-proof and authenticated. Version 1.3 is current; version 1.2 is still in wide use. It replaced the older SSL (Secure Sockets Layer). |
| **HTTPS** (HTTP Secure) | HTTP (HyperText Transfer Protocol, the web protocol) carried inside TLS. |
| **handshake** | The first messages of a TLS connection, in which both sides agree on algorithms and keys. |
| **certificate** | A signed document that says "this public key belongs to this name". Its format is called X.509. |
| **CA** (Certificate Authority) | An organisation that checks who you are and signs certificates. |
| **root certificate** | A CA certificate that your system already trusts. Operating systems and browsers ship a list of them, the *trust store*. |
| **intermediate certificate** | A certificate that a root signed, which in turn signs ordinary website certificates. |
| **chain of trust** | The sequence leaf → intermediate → root. Each certificate is signed by the next one. |
| **SAN** (Subject Alternative Name) | The field of a certificate that lists the host names (DNS, Domain Name System, names) it is valid for. |
| **self-signed certificate** | A certificate signed by its own key. Nobody vouches for it, so clients reject it unless you explicitly trust it. |
| **forward secrecy** | A property of the key exchange: if the server's long-term key leaks later, old recorded traffic still cannot be decrypted. |
| **man-in-the-middle** (MITM) | An attacker who sits between client and server, reads or changes the traffic, and pretends to be each side. |
| **PEM** (Privacy-Enhanced Mail) | A text format for keys and certificates, between lines such as `-----BEGIN CERTIFICATE-----`. |

## 1. What TLS gives you

Plain HTTP over TCP is a postcard: anyone on the path (a Wi-Fi router, a provider, a compromised machine) can read it and change it,
and you cannot know who is really at the other end. TLS fixes all three problems at once. Two of them are goals of the CIA triad from Lesson 10.1 (confidentiality and integrity; TLS does nothing for availability), and the third is authentication:

| Promise | How |
|---|---|
| **Confidentiality** (nobody can read) | Data is encrypted with an AEAD (Authenticated Encryption with Associated Data) cipher (Lesson 10.10) under a key that only the two sides know. |
| **Integrity** (nobody can change) | The AEAD tag detects every modification. |
| **Authentication** (you talk to the right server) | The server proves its identity with a certificate and a signature. |

TLS does **not** hide that you talk to a server (the IP, Internet Protocol, address is visible, and usually the host name too, because the client sends it in the handshake; see SNI in Section 4), and it does not make the server itself trustworthy or its code bug-free.

## 2. The handshake in plain words

```text
 Client                                                   Server
   │ ── ClientHello: my TLS versions, my ciphers,           │
   │    a fresh public key share (key exchange) ──────────► │
   │                                                        │  picks a version and cipher,
   │ ◄── ServerHello: my key share ─────────────────────────│  computes the shared secret
   │                                                        │
   │   ═══ from here on everything is encrypted ═══         │
   │ ◄── Certificate: my certificate chain ─────────────────│
   │ ◄── CertificateVerify: a signature made with my        │
   │     private key over the whole handshake so far ───────│
   │ ◄── Finished (a MAC over the handshake) ───────────────│
   │ ── Finished ─────────────────────────────────────────► │
   │ ◄═══════════ application data (HTTP), AEAD ═══════════►│
```

Every step uses a tool you already know:

1. **Key exchange.** Each side sends a fresh temporary public key (for example X25519: a key exchange on the elliptic curve Curve25519, with 32-byte keys). From its own private key and the other side's public key, both compute the **same shared secret**, which never travels over the network. The keys are temporary, which gives **forward secrecy**. Modern servers also mix in a post-quantum algorithm (you will see `X25519MLKEM768` in the output below), so recorded traffic stays safe even against future quantum computers.
2. **Key derivation.** The shared secret is fed through hash-based functions (HKDF, HMAC-based Key Derivation Function, built from HMAC, the hash-based message authentication code of Lesson 10.9) to produce the encryption keys.
3. **Authentication.** The server sends its certificate and signs the handshake with the matching private key (Lesson 10.10). The client checks the certificate (Section 3) and the signature. Only the real owner of the private key can produce that signature.
4. **Data.** Every record is encrypted and authenticated by AEAD (`AES-GCM` or `ChaCha20-Poly1305`).

## 3. Certificates and the chain of trust

A certificate is a small document, signed by a CA, that contains:

| Field | Meaning |
|---|---|
| Subject | Who this is (for example `CN=rust-lang.org`; CN means Common Name). |
| SAN | The host names it is valid for. **Clients check this**, not the subject. |
| Public key | The key that goes with the private key on the server. |
| Validity | `notBefore` and `notAfter` dates. Outside them the certificate is invalid. |
| Issuer | Who signed it. |
| Basic constraints | `CA:TRUE` for CAs, `CA:FALSE` for ordinary certificates. |
| Signature | The issuer's signature over all of the above. |

You cannot know every website's key in advance, so trust is delegated through a chain:

```text
   Root CA certificate      ← already in your system's trust store (you trust it)
        │ signs
        ▼
   Intermediate certificate ← sent by the server
        │ signs
        ▼
   Leaf certificate         ← the website's own certificate (CA:FALSE)
```

A client accepts a server only if **all** of these are true:

1. Every certificate in the chain has a valid signature from the next one up, and the chain ends at a root in the trust store.
2. Every certificate is inside its validity dates.
3. The leaf certificate's SAN contains the host name the client asked for.
4. The leaf is allowed to be used as a server certificate (it is not a CA).

If any check fails, the client must refuse. A man-in-the-middle cannot get a CA to sign a certificate for a host name they do not control, and they cannot sign it themselves, because the client does not trust them.

> 🔒 **Security:** certificates are short-lived on purpose. The one you will see below is valid for about 90 days. Short life limits the damage of a stolen key, and tools such as ACME (Automatic Certificate Management Environment, used by Let's Encrypt) renew them automatically. An expired certificate is a very common cause of outages.

## 4. Looking at a real connection

Ask `curl` to show the TLS part of a request to a public site. We keep only the interesting lines:

```bash
curl -sv -o /dev/null https://www.rust-lang.org 2>&1 | grep -E "TLS handshake|SSL connection|subject|issuer|expire date|start date|verify ok"
```

```text
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
*  subject: CN=rust-lang.org
*  start date: Sep 21 20:02:52 2026 GMT
*  expire date: Dec 20 20:02:51 2026 GMT
*  subjectAltName: host "www.rust-lang.org" matched cert's "www.rust-lang.org"
*  issuer: C=US; O=Let's Encrypt; CN=YR1
*  SSL certificate verify ok.
```

**What happens:**

1. `-s` hides the progress meter, `-v` shows details, `-o /dev/null` throws the page away, and `2>&1` joins the details (which `curl` prints on its error stream) into the pipe for `grep`.
2. The lines "Client hello", "Server hello", "Certificate", "CERT verify" and "Finished" are the handshake from Section 2. The cipher is an AEAD (`CHACHA20-POLY1305`).
3. `subjectAltName … matched` is check 3 of Section 3, and `SSL certificate verify ok` means that all checks passed.

Your output will show other dates (and maybe another cipher): the certificate is renewed regularly.

`openssl s_client` is the more detailed tool. Run it, send it no input (`</dev/null`), and filter the output:

```bash
openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org </dev/null 2>/dev/null | grep -E "^ *[0-9] s:|^ +i:|Verification|Protocol|^New|group"
```

```text
 0 s:CN=rust-lang.org
   i:C=US, O=Let's Encrypt, CN=YR1
 1 s:C=US, O=Let's Encrypt, CN=YR1
   i:C=US, O=ISRG, CN=Root YR
 2 s:C=US, O=ISRG, CN=Root YR
   i:C=US, O=Internet Security Research Group, CN=ISRG Root X1
Negotiated TLS1.3 group: X25519MLKEM768
Verification: OK
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Protocol: TLSv1.3
    Protocol  : TLSv1.3
```

**What happens:**

1. `-connect host:port` opens the connection, and `-servername` sends the host name in the handshake (SNI, Server Name Indication), so a server that hosts many sites knows which certificate to show.
2. `s:` is the *subject* and `i:` is the *issuer* of each certificate. Certificate 0 (the leaf, `rust-lang.org`) was issued by `YR1`; `YR1` was issued by `Root YR`; `Root YR` was issued by `ISRG Root X1`, a root that is in the trust store. That is the chain from Section 3.
3. `Negotiated TLS1.3 group` is the key exchange of Section 2: X25519 combined with the post-quantum algorithm ML-KEM (Module-Lattice Key Encapsulation Mechanism).
4. `Verification: OK` is the result of the checks against the trust store of OpenSSL. (The last `Protocol` line comes from the session ticket that the server sends after the handshake.)

To read one certificate in detail, pass it to `openssl x509`:

```bash
openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName,basicConstraints
```

```text
subject=CN=rust-lang.org
issuer=C=US, O=Let's Encrypt, CN=YR1
notBefore=Sep 21 20:02:52 2026 GMT
notAfter=Dec 20 20:02:51 2026 GMT
X509v3 Basic Constraints: critical
    CA:FALSE
X509v3 Subject Alternative Name: 
    DNS:rust-lang.org, DNS:www.rust-lang.org
```

The validity is about 90 days, the certificate is not a CA, and the two names it covers are listed in the SAN.

## 5. A local self-signed certificate

For your own experiments you can make a certificate yourself. It is **self-signed**: its issuer is itself, so no CA vouches for it. Create it in the `certs` folder:

```bash
cd ~/projects/rust/code/p10-tls/certs
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout key.pem -out cert.pem -days 7 -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" \
  -addext "basicConstraints=critical,CA:FALSE"
openssl x509 -in cert.pem -noout -subject -issuer -dates -ext subjectAltName,basicConstraints
```

```text
subject=CN=localhost
issuer=CN=localhost
notBefore=Nov 18 06:39:48 2026 GMT
notAfter=Nov 25 06:39:48 2026 GMT
X509v3 Subject Alternative Name: 
    DNS:localhost, IP Address:127.0.0.1
X509v3 Basic Constraints: critical
    CA:FALSE
```

**What happens:**

1. `req -x509` creates a self-signed certificate directly. `-newkey ec -pkeyopt ec_paramgen_curve:prime256v1` makes a new elliptic-curve key pair. `-nodes` means "no DES" (DES, the Data Encryption Standard, is an old cipher; the option means: do not encrypt the private key file. That is acceptable for a throwaway test key, never for a real one. OpenSSL 3 also accepts the newer name `-noenc`).
2. `-days 7` limits the validity to a week. `-subj "/CN=localhost"` sets the subject.
3. The two `-addext` options add the SAN (clients check it) and say that this is not a CA. `key.pem` is the **private** key; `cert.pem` is the certificate. Subject and issuer are equal: that is what "self-signed" means.

Open a **second terminal** and start a small TLS test server that serves its own status page. It runs until you press `Ctrl+C`:

```bash
cd ~/projects/rust/code/p10-tls/certs
openssl s_server -accept 8443 -cert cert.pem -key key.pem -www -quiet
```

Back in the first terminal, connect with the default trust store:

```bash
curl https://localhost:8443/
```

```text
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.se/docs/sslcerts.html
```

(`curl` also prints a longer explanation. The part that matters is the first line.) The connection is refused: the certificate is valid, but nobody the system trusts signed it. Now tell `curl` that exactly this certificate is the trusted root **for this one command**:

```bash
curl -sS --cacert cert.pem -o /dev/null -w "HTTP %{http_code}\n" https://localhost:8443/
```

```text
HTTP 200
```

The host name check still applies. Ask for a name that is not in the SAN, while pointing it to the same server (`--resolve` maps a name to an address for this command only):

```bash
curl -sS --cacert cert.pem --resolve other.test:8443:127.0.0.1 -o /dev/null https://other.test:8443/
```

```text
curl: (60) SSL: no alternative certificate subject name matches target host name 'other.test'
More details here: https://curl.se/docs/sslcerts.html
```

The same story with `openssl s_client`. The first command (without `-CAfile`) reports a verification error; the second one (with `-CAfile`) says `Verification: OK`:

```bash
openssl s_client -connect localhost:8443 -servername localhost -brief </dev/null 2>&1 | sed -n 1,4p
openssl s_client -connect localhost:8443 -servername localhost -CAfile cert.pem -verify_hostname localhost -brief </dev/null 2>&1 | grep -E "Verif"
```

```text
Connecting to ::1
depth=0 CN=localhost
verify error:num=18:self-signed certificate
CONNECTION ESTABLISHED
Verification: OK
Verified peername: localhost
```

**What happens:** the error `num=18` means "self-signed certificate". With `-CAfile cert.pem` we told OpenSSL to trust this one certificate, and `-verify_hostname localhost` asks it to check the name as well.

> 🔒 **Security:** every tool has a switch that turns the checks off (`curl -k`, `danger_accept_invalid_certs` in HTTP libraries; and `openssl s_client` by default only *reports* a verification error and keeps the connection open, as you saw above, unless you add `-verify_return_error`). With verification off, the connection is encrypted but you do not know who you are
> talking to, so a man-in-the-middle can read everything. "It works with `-k`" is the same as "it is broken". Trust a specific certificate instead (`--cacert`), as above.
> And never import a test certificate into the macOS keychain: it would make your whole computer trust it.

Stop the server with `Ctrl+C` in the second terminal.

## 6. A TLS client in Rust with rustls

`rustls` is a TLS library written in Rust. It does the whole handshake and all checks of Section 3. Your program supplies the **list of trusted roots** (`webpki-roots` is a copy of the Mozilla list) and the **host name**. Replace `src/main.rs` of `code/p10-tls`:

```rust
use std::io::{Read, Write};
use std::net::TcpStream;
use std::sync::Arc;

use rustls::pki_types::ServerName;
use rustls::{ClientConfig, ClientConnection, RootCertStore, Stream};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let host = std::env::args().nth(1).unwrap_or_else(|| "www.rust-lang.org".to_string());
    let port = std::env::args().nth(2).unwrap_or_else(|| "443".to_string());

    // 1. The list of certificate authorities we trust (a copy of Mozilla's list).
    let roots = RootCertStore::from_iter(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
    let config = ClientConfig::builder().with_root_certificates(roots).with_no_client_auth();

    // 2. A TLS connection object that expects a certificate for `host`.
    let server_name = ServerName::try_from(host.clone())?;
    let mut tls = ClientConnection::new(Arc::new(config), server_name)?;
    let mut tcp = TcpStream::connect(format!("{host}:{port}"))?;
    let mut stream = Stream::new(&mut tls, &mut tcp);

    // 3. Send a plain HTTP request. The handshake happens automatically on the first write.
    write!(stream, "GET / HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n")?;
    let mut first = [0u8; 40];
    stream.read_exact(&mut first)?;
    println!("version : {:?}", stream.conn.protocol_version().unwrap());
    println!("cipher  : {:?}", stream.conn.negotiated_cipher_suite().unwrap().suite());
    println!("reply   : {}", String::from_utf8_lossy(&first).lines().next().unwrap_or(""));
    Ok(())
}
```

**What happens:**

1. The host and the port come from the command line, with defaults. `RootCertStore` holds the trusted roots. `ClientConfig::builder()` starts a configuration: it needs the roots and says that we do not present a client certificate.
2. `ServerName` is the name that must appear in the certificate's SAN. `ClientConnection` is the TLS state machine; `TcpStream` is the underlying connection. `Stream` ties them together so that ordinary `Read` and `Write` calls encrypt and decrypt for you.
3. The first `write!` triggers the handshake. If the certificate is wrong, the `write!` or `read_exact` returns an error, and `?` ends the program before any data is sent in the clear.
4. We print the negotiated TLS version and cipher suite and the first line of the HTTP reply (40 bytes are enough for that).

Run `cargo run` (the first run compiles `rustls`, which takes a minute):

```text
version : TLSv1_3
cipher  : TLS13_AES_128_GCM_SHA256
reply   : HTTP/1.1 301 Moved Permanently
```

(`301` is a redirect from `www.rust-lang.org` to `rust-lang.org`: the request went through, which is all we wanted to see.) Now start the local server again (second terminal, same command as in Section 5) and point the client at it:

```bash
cargo run --quiet -- localhost 8443
```

```text
Error: Custom { kind: InvalidData, error: InvalidCertificate(UnknownIssuer) }
```

**What happens:** the local certificate is not signed by any root in the list, so `rustls` refuses (`UnknownIssuer`), exactly as `curl` did. The error is raised during the handshake, before the program sends its request. Stop the server with `Ctrl+C`.

> 📏 **Convention:** use a TLS library, never your own TLS. In real services the TLS part is usually done by a reverse proxy or a framework that uses `rustls`. You configure *certificate and key files* and *which roots to trust*. You do not write the protocol.

## Common mistakes

**1. Disabling verification to "make it work".** `curl -k`, `danger_accept_invalid_certs(true)`, or a custom verifier that accepts everything. The fix is to trust the right certificate (`--cacert`, or add it to the `RootCertStore`) or to fix the real problem (wrong name, expired date, missing intermediate).

**2. A certificate without a SAN.** Modern clients such as browsers and `rustls` ignore the `CN` and look only at the SAN, so a certificate with `-subj "/CN=localhost"` and no `subjectAltName` is rejected by them (some older tools still fall back to the `CN`, which hides the mistake). Always add the SAN.

**3. Connecting by a different name than the certificate has** (an IP address when only `DNS:` names are in the SAN, or `127.0.0.1` when only `localhost` is listed). Add the name or the `IP:` entry, or connect by the right name.

**4. Forgetting that certificates expire.** Automate renewal and monitor the expiry date. `openssl x509 -noout -enddate` shows it.

**5. Committing `key.pem` to Git.** A private key in a repository is public forever (Phase 01, Lesson 10.8). That is why `certs/` is in `.gitignore`.

## Try it yourself

1. Run `openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org -tls1_2 -brief </dev/null`. Which protocol and cipher are used now?
2. Print the whole certificate with `openssl x509 -in cert.pem -noout -text` and find the issuer, the validity dates, the public key and the signature algorithm.
3. Run the Rust client against `www.rust-lang.org` and `localhost` (server on) and compare the two results.

## Exercises

### Exercise 1 — When does it expire? (easy)

Write one command that prints only the expiry date of the certificate of `www.rust-lang.org`.

<details>
<summary>Hint</summary>

Pipe the output of `openssl s_client` (with `</dev/null 2>/dev/null`) into `openssl x509 -noout -enddate`.

</details>

<details>
<summary>Answer</summary>

```bash
openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org </dev/null 2>/dev/null | openssl x509 -noout -enddate
```

```text
notAfter=Dec 20 20:02:51 2026 GMT
```

Why it works: `s_client` prints the certificate in PEM form; `x509` reads it from the pipe and `-enddate` prints only the `notAfter` field. You can use this in a monitoring script.

</details>

### Exercise 2 — A certificate for your own name (medium)

Create a self-signed certificate for the name `myapp.test` (the `.test` domain is reserved for tests), serve it on port 8444 and make three `curl` calls:
one that succeeds, one without `--cacert` and one with the name `other.test`. Use `--resolve` to map the names to `127.0.0.1`.

<details>
<summary>Hint</summary>

Same `openssl req` command as in Section 5, with `/CN=myapp.test` and `subjectAltName=DNS:myapp.test`. The `curl` option is `--resolve myapp.test:8444:127.0.0.1`.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~/projects/rust/code/p10-tls/certs
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout key2.pem -out cert2.pem -days 7 -subj "/CN=myapp.test" \
  -addext "subjectAltName=DNS:myapp.test" -addext "basicConstraints=critical,CA:FALSE"
# second terminal: openssl s_server -accept 8444 -cert cert2.pem -key key2.pem -www -quiet
curl -sS --resolve myapp.test:8444:127.0.0.1 --cacert cert2.pem -o /dev/null -w "HTTP %{http_code}\n" https://myapp.test:8444/
curl -sS --resolve myapp.test:8444:127.0.0.1 -o /dev/null https://myapp.test:8444/
curl -sS --resolve other.test:8444:127.0.0.1 --cacert cert2.pem -o /dev/null https://other.test:8444/
```

```text
HTTP 200
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.se/docs/sslcerts.html
curl: (60) SSL: no alternative certificate subject name matches target host name 'other.test'
More details here: https://curl.se/docs/sslcerts.html
```

Why it works: call 1 trusts exactly this certificate and the name matches the SAN. Call 2 has no trust for the self-signed certificate. Call 3 trusts the certificate but the name is not in the SAN. Stop the server with `Ctrl+C`.

</details>

### Exercise 3 — Host name matching, with tests (hard)

Certificates may contain a wildcard such as `*.example.com`. Write `pub fn host_matches(pattern: &str, host: &str) -> bool` for a new library package with these rules:
names are compared without regard to upper or lower case; a wildcard is allowed only as the whole first label (`*.example.com`) and matches **exactly one** label.
So `www.example.com` matches, but `example.com`, `a.b.example.com`, `.example.com` and `www.example.com.evil.test` do not. Write tests for all of these.
(In real code you never write this yourself: the TLS library does it. The exercise shows how small mistakes in such a rule become security holes.)

<details>
<summary>Hint</summary>

`pattern.strip_prefix("*.")` returns `Some(suffix)` for wildcards. For the host, `split_once('.')` gives the first label and the rest; the rest must equal the suffix **exactly**, and the first label must not be empty.

</details>

<details>
<summary>Answer</summary>

```rust
/// Does a certificate name (`pattern`) match the host the client wants to reach?
/// Rules: names are compared case-insensitively; a wildcard is allowed only as the whole
/// left-most label (`*.example.com`) and matches exactly one label.
pub fn host_matches(pattern: &str, host: &str) -> bool {
    let pattern = pattern.to_ascii_lowercase();
    let host = host.to_ascii_lowercase();
    match pattern.strip_prefix("*.") {
        None => pattern == host,
        Some(suffix) => match host.split_once('.') {
            Some((label, rest)) => !label.is_empty() && rest == suffix,
            None => false,
        },
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn exact_names_match_ignoring_case() {
        assert!(host_matches("www.example.com", "WWW.Example.com"));
        assert!(!host_matches("www.example.com", "example.com"));
    }

    #[test]
    fn wildcard_matches_exactly_one_label() {
        assert!(host_matches("*.example.com", "www.example.com"));
        assert!(!host_matches("*.example.com", "example.com"));
        assert!(!host_matches("*.example.com", "a.b.example.com"));
        assert!(!host_matches("*.example.com", ".example.com"));
    }

    #[test]
    fn lookalike_suffixes_do_not_match() {
        assert!(!host_matches("*.example.com", "www.example.com.evil.test"));
        assert!(!host_matches("*.example.com", "www.badexample.com"));
    }
}
```

`cargo test` shows 3 passing tests.

Why it works: the rest of the host after the first dot must equal the suffix exactly, so `a.b.example.com` (rest `b.example.com`) and `www.example.com.evil.test` both fail. A sloppy `host.ends_with("example.com")` would accept `www.badexample.com`.

</details>

## Quiz

1. Name the three things TLS gives you.

<details>
<summary>Answer</summary>

Confidentiality (encryption), integrity (tamper detection) and authentication of the server (and optionally the client) through certificates.

</details>

2. Why does the client trust the certificate of `www.rust-lang.org`?

<details>
<summary>Answer</summary>

Because it is signed by an intermediate CA, which is signed by a root CA that is in the trust store of the client, because the dates are valid and because the host name is in the SAN. The chain of signatures leads to a root that the client already trusts.

</details>

3. What is forward secrecy?

<details>
<summary>Answer</summary>

The session keys come from temporary key exchange keys, so even if the server's long-term private key is stolen later, old recorded traffic cannot be decrypted.

</details>

4. A colleague says "it only works with `curl -k`". What does that tell you?

<details>
<summary>Answer</summary>

Certificate verification is failing (unknown issuer, wrong name, expired, or missing intermediate). `-k` hides the problem and removes the protection against man-in-the-middle attacks. Find and fix the cause, or trust the specific certificate with `--cacert`.

</details>

5. What does a self-signed certificate lack?

<details>
<summary>Answer</summary>

A third party that vouches for it. Its issuer is itself, so no chain leads to a trusted root and clients reject it unless they are told to trust that exact certificate.

</details>

## Summary

- TLS gives confidentiality, integrity and server authentication on top of TCP; HTTPS is HTTP over TLS.
- The handshake uses a temporary key exchange (forward secrecy), key derivation, a certificate plus signature for identity, and an AEAD cipher for the data.
- A certificate binds a public key to host names (SAN) for a period of time. Trust flows from a root in the trust store through intermediates to the leaf.
- A client must check the chain, the dates and the host name. If any check fails, it must stop.
- `curl -v` and `openssl s_client` show what really happens; `openssl req -x509` makes a local test certificate that you trust only through explicit options.
- Never disable verification and never put a test certificate in a system trust store. Never commit private keys.
- In Rust, `rustls` does the protocol and the checks; you supply the trusted roots and the name.

## Sources and further reading

- [RFC 8446 — TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446) — the TLS 1.3 specification, with the handshake diagram (an RFC, Request for Comments, is a document of the official internet standards series).
- [rustls documentation](https://docs.rs/rustls/0.23.46/rustls/) — the API used in this lesson.
- [Let's Encrypt — How It Works](https://letsencrypt.org/how-it-works/) — how certificates are issued and renewed automatically.
- [OpenSSL manual: s_client](https://docs.openssl.org/3.6/man1/openssl-s_client/) — every option of the command used above.
- [curl: SSL certificate verification](https://curl.se/docs/sslcerts.html) — how `curl` checks certificates and what its options do.
- [Mozilla — Web Security Guidelines](https://infosec.mozilla.org/guidelines/web_security) — recommended TLS settings for services.

---

[🏠 Start](../START-HERE.md) · [← 10.10 Cryptography II: encryption and signatures](10-cryptography-encryption-and-signatures.md) · [📋 Phase 10](README.md) · [10.12 Memory safety and reviewing unsafe code →](12-memory-safety-and-reviewing-unsafe.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
